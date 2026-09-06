# GitHub ARC on EKS — Runner Scale Set Setup Guide

Deploys the Actions Runner Controller (ARC) with a runner scale set pinned
to a Karpenter spot NodePool, authenticated via PAT, using **Guaranteed
QoS** so Karpenter bin-packs and evicts predictably, and running in
**`containerMode: kubernetes`** — meaning job containers are scheduled as
native, unprivileged Kubernetes pods instead of relying on a Docker daemon.

If you're deciding between container modes at all, read
[§7 "Things to consider"](#7-things-to-consider-before-you-commit-to-this)
before you install anything — several of these decisions are much cheaper
to make up front than to unwind later.

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

## 5. Runner scale set values — field-by-field

```yaml
# values-eks-spot-amd.yaml

githubConfigUrl: "https://github.com/<your-username>/<your-repo>"
githubConfigSecret: eks-spot-amd-gh-secret

runnerScaleSetName: "eks-spot-amd"

minRunners: 0     # scale to zero when idle — no wasted spot node
maxRunners: 3     # single repo, single workflow — keep this tight

containerMode:
  type: "kubernetes"
  kubernetesModeWorkVolumeClaim:
    accessModes:
      - ReadWriteOnce
    storageClassName: gp3
    resources:
      requests:
        storage: 5Gi

template:
  spec:
    securityContext:
      fsGroup: 1001

    nodeSelector:
      node-pool: spot-amd64
      capacity-type: spot
      arch: amd64

    containers:
      - name: runner
        image: ghcr.io/actions/actions-runner:latest
        command: ["/home/runner/run.sh"]
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
```

| Field | What it does | Why it's set this way |
|---|---|---|
| `githubConfigUrl` | The repository (or org) this scale set registers against | Scoped to one repo here — no org-wide runner group, nothing visible outside that repo's **Settings → Actions → Runners** page |
| `githubConfigSecret` | Name of the Kubernetes secret holding the PAT | Must match the secret created in §4 |
| `runnerScaleSetName` | The label workflows target via `runs-on:` | Must match `runs-on` in your workflow **exactly** |
| `minRunners: 0` | No runner is kept warm when idle | No wasted spot node, no idle billing, at the cost of a cold-start delay on the first job after idle |
| `maxRunners: 3` | Upper bound on concurrent runners | Deliberately tight for a single repo/workflow — raise it if you expect real concurrency (parallel PRs, matrix jobs) |
| `containerMode.type: "kubernetes"` | Job containers, service containers, and container actions are each created as their own unprivileged pod instead of running inside a privileged Docker sidecar | The security-preferred mode — see §7 for what this trades away |
| `kubernetesModeWorkVolumeClaim` | A **template** ARC uses to dynamically provision a fresh `PersistentVolumeClaim` per job | Required by `kubernetes` mode: the runner pod and any job/service/container-action pods it spins up all need to see the same checked-out files, and that's only possible via a shared, addressable volume |
| `accessModes: [ReadWriteOnce]` | The volume can only attach to one node at a time | The simplest/cheapest mode most block-storage classes (like EBS-backed `gp3`) support; forces every pod for a given job onto the same node — see §7 |
| `storageClassName: gp3` | Which StorageClass provisions the volume | Standard EBS `gp3` on EKS; confirm its reclaim policy (§7) |
| `resources.requests.storage: 5Gi` | Requested volume size | Size to your actual checkout + build-artifact footprint; too small and builds fail mid-way with no obvious "disk full" signal in some tooling |
| `template.spec.securityContext.fsGroup: 1001` | Sets the filesystem group for all containers in the pod | Makes the PVC mount group-writable for the runner's UID/GID — without this, tools that write to the workspace as a non-root user hit plain permission-denied errors that don't reproduce on a GitHub-hosted runner |
| `nodeSelector` | Pins the runner pod to a specific Karpenter NodePool | Ensures runners actually land on the intended spot/arch pool rather than wherever the scheduler happens to place them |
| `containers[0].resources` (`requests == limits`) | Gives the pod **Guaranteed** QoS | Karpenter sizes/bin-packs instances off `requests`; Guaranteed pods are the *last* class evicted under node memory pressure, so a job isn't killed by kubelet eviction on top of whatever spot interruption already risks |

Adjust the `2 CPU / 4Gi` figures to your actual job's footprint — this also
directly drives which instance type Karpenter's NodePool provisions, since
Karpenter picks the smallest instance that satisfies the pod's requests.

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

## 7. Things to consider before you commit to this

**`kubernetesModeWorkVolumeClaim` is mandatory the moment you set
`containerMode.type: "kubernetes"`.** There's a sibling mode,
`kubernetes-novolume`, that skips provisioning this PVC entirely — it's
tempting, since it removes the per-job volume-provisioning latency this
setup pays. **Don't use it yet.** At the time of writing it has multiple
open, unresolved upstream bugs: job event payloads not being copied into
job containers (breaking any action that reads the triggering event),
jobs failing to start at all in some configurations, and composite actions
failing to resolve their own local files after migrating from `kubernetes`
mode. None of these are configuration mistakes you can work around —
they're regressions in the underlying hook implementation. Revisit it once
those are closed upstream.

**Every job in your workflow needs a job-level `container:`.** This is a
hard requirement of `kubernetes` mode, not something scoped to jobs that
obviously need one — a job with plain `run:` shell steps and no
`container:` key is rejected outright. See §8 for what this looks like in
practice.

**There is no Docker daemon anywhere in this mode.** Any step that
builds/pushes a container image, logs into a registry, or loads an image
locally to scan before pushing needs a real daemon or `--privileged`
access — this mode gives job pods neither, deliberately. If your pipeline
does any of this, plan to either rewrite that job around daemonless
tooling (an image builder that writes to a tarball, a scanner that reads
the tarball directly, a registry-push tool that doesn't need a daemon), or
run just that job on a separate, `dind`-mode scale set.

**Confirm your StorageClass actually deletes volumes on release.**

```bash
kubectl get storageclass gp3 -o jsonpath='{.reclaimPolicy}'
```

If this says `Retain` instead of `Delete`, PVCs (and the EBS volumes
backing them) will not be cleaned up automatically after each ephemeral
runner is torn down, and you'll accumulate billed volumes silently. With
`Delete`, and `minRunners: 0`, there should be no idle pod, PVC, or EBS
volume between workflow runs — verify with `kubectl get pvc -n
arc-runners` after a run completes.

**A spot interruption takes the whole job down, not just one pod.**
Because the PVC is `ReadWriteOnce`, every pod for a given job (the runner,
plus any job/service/container-action pods) is scheduled onto the same
node. If that node is reclaimed mid-job, the runner, the job pods, and the
volume attachment all go down together, and the job is retried from
scratch on a new node with a fresh PVC.

**Container images need their own tooling, and glibc matters.** The
default runner image is far more minimal than GitHub's hosted image — no
`git`, no build tools, nothing beyond the runner binary itself. Every job's
`container:` image needs to actually contain what that job's steps use.
Separately, prefer a glibc-based base image (Debian/Ubuntu-derived) over a
musl-based minimal image for any job using JavaScript-based actions
(most `actions/checkout`-style actions) — musl compatibility issues with
the mounted Node.js runtime are a common, confusing source of failures
that look unrelated to the actual problem.

---

## 8. Point the workflow at it — before and after

**Before** (works on `ubuntu-latest`, and would also work unmodified under
`dind` mode):

```yaml
jobs:
  build:
    runs-on: eks-spot-amd
    steps:
      - uses: actions/checkout@v4
      - run: make build
```

**After** (required under `containerMode: kubernetes`):

```yaml
jobs:
  build:
    runs-on: eks-spot-amd

    # Mandatory under kubernetes mode — a job with no container: is rejected.
    # Pick an image that actually has git + make + whatever else this job's
    # steps assume is already installed, on a glibc base.
    container:
      image: ghcr.io/catthehacker/ubuntu:act-22.04

    defaults:
      run:
        shell: bash   # confirm the chosen image's default shell before skipping this

    steps:
      - uses: actions/checkout@v4
      - run: make build
```

`runs-on` must match `runnerScaleSetName` exactly in both cases — the
`container:` block is the only structural difference `kubernetes` mode
forces on a plain build/test job like this one. Jobs that also build
container images or upload artifacts through actions with home-directory
assumptions need further changes — see the companion migration guide for
those.

---

## 9. Validate

```bash
# Watch for the ephemeral runner pod when a workflow triggers
kubectl get pods -n arc-runners -w

# Confirm it landed on the intended spot node pool
kubectl get pod <runner-pod-name> -n arc-runners -o jsonpath='{.spec.nodeName}'
kubectl get node <node-name> --show-labels | grep -o 'capacity-type=[a-z]*'

# Confirm QoS class actually came out Guaranteed
kubectl get pod <runner-pod-name> -n arc-runners -o jsonpath='{.status.qosClass}'

# Confirm the per-job PVC was created, then cleaned up after the run
kubectl get pvc -n arc-runners
```

Trigger a real workflow run and confirm the full path: listener sees the
job → EphemeralRunnerSet creates a pod and a PVC → the node pool
provisions a node if none exists → job/step pods are created and mount the
same PVC → pod(s), PVC, and eventually the node all terminate after the
job completes.