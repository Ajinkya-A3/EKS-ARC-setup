# GitHub ARC on EKS — `eks-spot-amd` Setup Guide

Deploys the Actions Runner Controller (ARC) with a runner scale set named
`eks-spot-amd`, pinned to the Karpenter `spot-amd64` NodePool, authenticated
via PAT, using **Guaranteed QoS** so Karpenter bin-packs and evicts predictably.

---

## 1. Create the PAT

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**.
2. **Resource owner**: your personal account (the one that owns the target repo).
3. **Repository access**: *Only select repositories* → pick the one target repo. Nothing org-wide.
4. **Permissions required**:

   | Permission      | Access level      | Why |
   |-----------------|--------------------|-----|
   | Actions         | Read and write     | Register/deregister runners, read job queue |
   | Administration  | Read and write     | Create/remove self-hosted runners on the repo |
   | Metadata        | Read-only          | Auto-required baseline, no action needed |

5. Set an **expiration** (fine-grained tokens can't be "no expiration" in most org policies) and note the renewal date somewhere.
6. Click **Generate token** and copy it immediately — it's shown once.

---

## 2. Namespaces

```bash
kubectl create namespace arc-systems
kubectl create namespace arc-runners
```

---

## 3. Install the controller (once per cluster)

```bash
helm install arc \
  --namespace arc-systems \
  --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

Verify:

```bash
helm list -n arc-systems
kubectl get pods -n arc-systems
# expect: arc-gha-runner-scale-set-controller-xxxx   1/1   Running
```

---

## 4. Create the PAT secret

```bash
kubectl create secret generic eks-spot-amd-gh-secret \
  --namespace arc-runners \
  --from-literal=github_token=<YOUR_PAT>
```

---

## 5. Runner scale set values — `values-eks-spot-amd.yaml`

**QoS note:** setting `requests == limits` on every container gives the pod
**Guaranteed** QoS class. On a spot NodePool this matters for two reasons:
Karpenter sizes and bin-packs instances off your `requests`, and Guaranteed
pods are the *last* class Kubernetes evicts under node memory pressure — so
your CI job isn't killed by kubelet-level eviction on top of whatever spot
already does to it.

```yaml
# values-eks-spot-amd.yaml

githubConfigUrl: "https://github.com/<your-username>/<your-repo>"
githubConfigSecret: eks-spot-amd-gh-secret

runnerScaleSetName: "eks-spot-amd"

minRunners: 0     # scale to zero when idle — no wasted spot node
maxRunners: 3     # single repo, single workflow — keep this tight

template:
  spec:
    nodeSelector:
      node-pool: spot-amd64
      capacity-type: spot
      arch: amd64

    # Only needed if you later add a taint to this NodePool
    # tolerations:
    #   - key: "node-pool"
    #     operator: "Equal"
    #     value: "spot-amd64"
    #     effect: "NoSchedule"

    containers:
      - name: runner
        image: ghcr.io/actions/actions-runner:latest
        command: ["/home/runner/run.sh"]
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
          limits:
            cpu: "2"       # == requests → Guaranteed QoS
            memory: "4Gi"  # == requests → Guaranteed QoS
```

Adjust the `2 CPU / 4Gi` figures to your actual job's footprint — this also
directly drives which instance type Karpenter's `spot-amd64` NodePool
provisions, since Karpenter picks the smallest instance that satisfies the
pod's requests.

---

## 6. Install the runner scale set

```bash
helm install eks-spot-amd \
  --namespace arc-runners \
  -f values-eks-spot-amd.yaml \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

Verify:

```bash
helm list -n arc-runners
kubectl get pods -n arc-runners
# expect: eks-spot-amd-xxxx-listener   1/1   Running   (idle, long-poll)
```

---

## 7. Point the workflow at it

```yaml
jobs:
  build:
    runs-on: eks-spot-amd
    steps:
      - uses: actions/checkout@v4
      - run: echo "running on spot amd64"
```

`runs-on` must match `runnerScaleSetName` exactly.

---

## 8. Validate

```bash
# Watch for the ephemeral runner pod when a workflow triggers
kubectl get pods -n arc-runners -w

# Confirm it landed on a spot-amd64 node
kubectl get pod <runner-pod-name> -n arc-runners -o jsonpath='{.spec.nodeName}'
kubectl get node <node-name> --show-labels | grep -o 'capacity-type=[a-z]*'

# Confirm QoS class actually came out Guaranteed
kubectl get pod <runner-pod-name> -n arc-runners -o jsonpath='{.status.qosClass}'
```

This setup is scoped entirely to the one repo in `githubConfigUrl` — no org
registration, no org-wide runner group, nothing visible outside that repo's
**Settings → Actions → Runners** page.

Trigger a real workflow run in the target repo and confirm the full path:
listener sees the job → EphemeralRunnerSet creates a pod → Karpenter
provisions a `spot-amd64` node if none exists → pod runs with Guaranteed QoS
→ pod (and eventually the node, after `consolidateAfter: 2m`) terminates.
