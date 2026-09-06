# Migrating a CI Workflow to Self-Hosted ARC Runners on Kubernetes

This document explains, generically, what changes when you move a GitHub
Actions workflow from GitHub-hosted runners (`ubuntu-latest`) to a
self-hosted runner scale set managed by the Actions Runner Controller
(ARC) on Kubernetes — and *why* each change is necessary, not just what to
paste in.

None of this is specific to any one workflow. If your pipeline does the
usual things — checkout, secret/code scanning, build and test, build and
push a container image, update some downstream config — you will hit every
category of change described here, regardless of what the individual steps
are called.

---

## 1. The core difference: a VM vs. a pod you configure yourself

A GitHub-hosted runner is a full virtual machine that GitHub provisions,
pre-loads with a large, curated toolset, and destroys after the job. An ARC
runner is a **pod in your own cluster**, running whatever container image
you told it to run. GitHub only handles job scheduling; everything else —
whether a Docker daemon exists, what tools are installed, where the
workspace lives — is something you now own.

| | GitHub-hosted runner | ARC self-hosted runner |
|---|---|---|
| Compute | Full VM per job | A pod on your own cluster nodes |
| Docker daemon | Present on the VM — `docker build`, service containers, and container actions all just work | **Not present by default.** You must explicitly choose a container mode that provides one (at a security cost) or restructure builds to not need one |
| Tooling (git, language runtimes, build tools) | Pre-installed on a large, GitHub-maintained image | Only whatever is baked into the container image(s) you specify |
| Job containers | Optional | Depends on the container mode — some modes require **every** job to declare one |
| Workspace storage | Local VM disk, destroyed with the VM | A volume you provision (in-memory, ephemeral, or a real block-storage claim), whose lifecycle you're responsible for reasoning about |
| `$HOME` vs. workspace path | Coincidentally nested under each other | Frequently unrelated paths — this mismatch quietly breaks any tool that assumes one is inside the other |
| Isolation & network | GitHub's network, dedicated VM per job | Your own network (can reach internal/private services), pod-level isolation on infrastructure you manage |

Everything below follows from that one shift: nothing is free anymore, and
the pieces that used to be invisible now have to be decided on purpose.

---

## 2. Choosing a container mode (this decision drives everything else)

ARC runner scale sets support a few container modes, and the one you pick
determines which of the changes below actually apply to you.

- **`dind` (Docker-in-Docker)** — the runner pod gets a real, privileged
  Docker daemon as a sidecar container. Anything that needs Docker (image
  builds, service containers, container actions) works exactly as it did
  on a GitHub-hosted runner, with no workflow changes required. The cost is
  a genuinely privileged pod, plus a startup delay caused by an init
  container that copies runner binaries into the sidecar before every job
  — this can add real time (sometimes minutes) to every job's startup on a
  busy cluster.
- **`kubernetes`** — every container a job needs (the job's own container,
  service containers, or container actions) is created as a **separate,
  unprivileged Kubernetes pod**, rather than a Docker container inside a
  privileged sidecar. This is the more secure option, but it comes with
  real constraints (below) and no Docker daemon anywhere.
- **`kubernetes-novolume`** — a newer variant of `kubernetes` mode that
  avoids provisioning a shared persistent volume, at the cost of currently
  being less mature: at the time of writing it has multiple open upstream
  bugs (event payloads not being copied into job containers, jobs failing
  to start, composite actions failing to resolve their own local files).
  Worth revisiting later; not yet reliable enough to build production CI
  on top of.

Once you pick `kubernetes` (or `kubernetes-novolume`), the rest of this
document applies. If you pick `dind`, most of §4 does not apply to you —
but you inherit the privileged-pod security trade-off and the startup
latency instead.

---

## 3. Storage: why a workflow that "just runs" now needs a volume decision

On a GitHub-hosted VM, the workspace is just a directory on local disk that
vanishes with the VM. Under `containerMode: kubernetes`, the workspace has
to be a real, addressable volume, because **multiple separate pods** (the
runner, plus any job/service/container-action pods it spins up mid-job)
all need to see the same checked-out files and build artifacts.

That's provided via a `PersistentVolumeClaim` **template** in your runner
scale set configuration — not a fixed, pre-existing volume. Concretely,
per job:

1. When a job is dispatched, the controller creates a new ephemeral runner
   and, from your PVC template, a **new PVC** requesting dynamic
   provisioning from whatever storage class you specify.
2. The underlying CSI driver for your cloud provisions a real block volume
   to back it, and mounts it into the runner pod's workspace path.
3. Any additional pods that job spins up (a job-level container, a service
   container, a container action) mount that **same** PVC, so files
   persist across steps and across pods.
4. When the job finishes — pass, fail, or cancelled — the ephemeral runner
   is deleted. The PVC's owner reference ties its lifecycle to that runner,
   so Kubernetes garbage-collects the PVC automatically, and (provided your
   storage class's reclaim policy is `Delete`, not `Retain`) the backing
   block volume is deleted with it.

**Practical implications this creates:**

- **Nothing should be left dangling** if the reclaim policy is correct —
  but it's worth explicitly verifying (`kubectl get pvc` after a run
  completes; checking the storage class's `reclaimPolicy`), because a
  misconfigured `Retain` policy will silently accumulate billed volumes
  forever.
- **Provisioning a volume takes real time** — attach/format/mount latency
  per job is a genuine cost this architecture adds that didn't exist on a
  local VM disk. It's a different bottleneck than `dind`'s init-container
  delay, not necessarily a faster one; measure it on your own cluster
  rather than assuming.
- **The volume is typically `ReadWriteOnce`**, meaning it can only attach
  to one node at a time. This forces every pod spun up for that job onto
  the same node as the runner — transparent to you, but it means a spot/
  preemptible node interruption takes down the whole job (runner, job
  pods, and the volume attachment together), not just one pod.
- **File ownership/permissions on the mounted volume may not match the
  runner's user by default.** If tooling inside the job container writes
  to the workspace as a non-root UID/GID, and the volume mount doesn't
  match, you'll see plain permission-denied errors on writes that worked
  fine on a GitHub-hosted VM. The fix is setting a pod-level filesystem
  group (`securityContext.fsGroup`) that matches the container's user, so
  the mount is group-writable.

---

## 4. Workflow-level changes `kubernetes` mode forces on you

### 4.1 Every job needs its own container

Under `kubernetes` (and `kubernetes-novolume`) mode, a job that has no
job-level `container:` key is rejected outright — even a job that's just
running shell commands, with nothing that obviously needs "a container."
This is a hard requirement of the mode, not something you can opt out of
per job.

**Why:** the mechanism that lets this mode avoid a privileged Docker
daemon is that *every* container the job needs is created as its own
Kubernetes pod through the same code path. Allowing some jobs to skip that
and run "directly on the runner" would need a different execution path
that the mode doesn't support.

**What to do:** add an explicit `container:` to every job, choosing an
image that has what that job's steps actually need (see 4.2).

### 4.2 The container image has to bring its own tooling

GitHub's hosted image ships an enormous, curated toolset (git, common
language runtimes, build tools, cloud CLIs, and more). The minimal runner
image ARC uses by default ships almost none of that. Once every job needs
a `container:`, the image you choose for each one has to actually contain
whatever that job's steps assume is already there — `git`, `make`,
compilers, a language interpreter, etc. A job that ran fine on a hosted
runner can fail immediately here with nothing more informative than
"command not found," purely because the tool it depends on was never
installed anywhere.

There's a second, less obvious constraint: **JavaScript-based actions
(most `actions/checkout`-style actions) need a Node.js runtime that's
compatible with the container's C library.** Minimal images built on musl
libc (common in very small base images) are frequently incompatible with
the Node.js binary that gets mounted into the container, causing checkout
itself — the very first step of most jobs — to fail. Practically, this
means picking a glibc-based base image (a Debian/Ubuntu derivative, not a
musl-based minimal image) for any job that uses JS-based actions, which is
most jobs.

**What to do:** for each job, inventory what its steps actually require
(a real `git` binary if you need full history, build tools, a specific
runtime), and pick — or build — a container image that has all of it, on
a glibc base.

### 4.3 There is no Docker daemon anywhere

This is the change most likely to break an existing pipeline outright, and
the one most worth planning for deliberately rather than discovering
mid-migration.

Any step that assumes a working Docker CLI against a live daemon —
building an image, logging into a registry, loading a built image locally
so a scanner can inspect it before pushing, or running a container action
that isn't a plain job-level `container:` — will fail, because
`kubernetes` mode never provides `/var/run/docker.sock` or grants
privileged access to job pods. This is a deliberate security property of
the mode, not a bug: giving job pods Docker access is functionally
equivalent to giving them root on the node, which is exactly what this
mode exists to avoid.

**What to do:** anywhere your pipeline builds/scans/pushes a container
image, replace the Docker-CLI-based steps with daemonless tooling that
doesn't need a socket or privileged access — a daemonless image builder
that writes directly to an image tarball, a vulnerability scanner that can
scan that tarball file directly rather than a live Docker image reference,
and a lightweight registry-push tool that can push a tarball without a
daemon. This preserves a "build → scan → only push if clean" gate without
ever touching a Docker socket. Each of these tools can run as its own
per-step container (a `uses: docker://<image>` step), sharing the job's
workspace volume so the artifact one step produces is visible to the next.

If rewriting the build step this way isn't practical right now, the
alternative is accepting `dind` mode's privileged-pod trade-off for just
the jobs that need Docker, and splitting your jobs across two runner scale
sets — one in `kubernetes` mode for everything else, one in `dind` mode
for the build job specifically.

### 4.4 Shell differences

`run:` steps in a job container execute using whatever shell the
container's default is — which is very often `/bin/sh` (a POSIX shell),
not Bash, even on Debian/Ubuntu-based images. Any `run:` step written with
Bash-specific syntax (parameter expansion patterns like `${VAR//x/y}`,
arrays, `[[ ]]` conditionals) will fail with cryptic shell errors like
"bad substitution" the moment it runs under `/bin/sh`, even though it
worked fine on a hosted runner where Bash is the default.

**What to do:** explicitly set `defaults.run.shell: bash` on any job whose
`run:` steps use Bash syntax, and confirm the chosen container image
actually has Bash installed.

### 4.5 Actions that assume `$HOME` contains the workspace

This is a subtler class of bug worth watching for generally, not just as a
one-off. Some actions — particularly ones that upload their own output as
a workflow artifact — compute a "root directory" for that upload based on
the user's home directory, rather than the actual workspace path
(`GITHUB_WORKSPACE`). On a GitHub-hosted runner, the home directory happens
to be an ancestor of the workspace path, so this assumption holds by
coincidence. Under a self-hosted container, the home directory
(frequently `/root`, since containers commonly run as root with no
explicit `HOME` override) and the workspace (mounted from your PVC at an
unrelated path) usually share nothing in common — so any action relying on
that coincidence throws an error like "X is not a parent directory of Y,"
even though the actual work the action was doing (a scan, a build, a
check) completed successfully just before the crash.

**What to do:** when you hit this class of error, check whether the
underlying tool actually succeeded (it usually did — the crash is in a
separate, cosmetic "upload the result" step) and see if the action offers
a way to disable its own artifact-upload behavior. If so, disable it and
upload the result file yourself with a standard artifact-upload action,
pointing at an explicit path relative to the workspace — since you're
supplying the path directly, you sidestep the broken home-directory
assumption entirely.

---

## 5. Before / after: what the diff actually looks like

### A plain build/test job

**Before** — runs fine on a hosted runner, and would also run unmodified
under `dind` mode:

```yaml
jobs:
  build-test:
    runs-on: <your-runner-scale-set>
    steps:
      - uses: actions/checkout@v4
      - run: make build && make test
```

**After** — required under `kubernetes` mode:

```yaml
jobs:
  build-test:
    runs-on: <your-runner-scale-set>

    # §4.1 — mandatory under kubernetes mode; a job with no container: is rejected.
    container:
      image: <an-image-with-git-and-make-on-a-glibc-base>

    defaults:
      run:
        shell: bash   # §4.4 — only needed if this job's run: steps use Bash syntax

    steps:
      - uses: actions/checkout@v4
      - run: make build && make test
```

Nothing about *what* the job does changed — only the environment it runs
in had to be made explicit.

### A job that builds and pushes a container image

**Before** — assumes a live Docker daemon:

```yaml
jobs:
  build-and-push:
    runs-on: <your-runner-scale-set>
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.REGISTRY_USER }}
          password: ${{ secrets.REGISTRY_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: <registry>/<image>:${{ github.sha }}
```

**After** — no daemon anywhere, so the build/push mechanism itself changes,
not just the container wrapper:

```yaml
jobs:
  build-and-push:
    runs-on: <your-runner-scale-set>
    container:
      image: <a-minimal-image-with-git>

    steps:
      - uses: actions/checkout@v4

      # §4.3 — a daemonless builder writes straight to a tarball; no socket,
      # no privileged pod. Runs as its own pod under kubernetes mode.
      - name: Build image to tarball
        uses: docker://<daemonless-image-builder>
        with:
          args: >-
            --dockerfile=Dockerfile
            --no-push
            --tarPath=image.tar
            --destination=<registry>/<image>:${{ github.sha }}

      # A scanner that can read the tarball directly, before anything ships.
      - name: Scan tarball
        uses: docker://<tarball-capable-scanner>
        with:
          args: image --input image.tar --exit-code 1

      # A daemonless push tool — pushes the tarball, still no daemon.
      - name: Push image
        if: success()
        uses: docker://<daemonless-registry-push-tool>
        with:
          entrypoint: <tool>
          args: push image.tar <registry>/<image>:${{ github.sha }}
```

The "build → scan → only push if clean" *shape* of the pipeline is
unchanged — every tool in it just had to be one that doesn't assume a
Docker daemon is sitting underneath it.

---

## 6. Summary: a checklist for migrating any workflow

- [ ] Decide on a container mode (`dind` vs `kubernetes`) based on whether
      your pipeline needs a real Docker daemon and how you weigh privileged
      pods against pipeline rewrites.
- [ ] If using `kubernetes` mode: provision a PVC template for the shared
      work volume, and verify your storage class's reclaim policy actually
      deletes volumes after use.
- [ ] Add `securityContext.fsGroup` at the pod level so non-root tooling
      can write to the mounted workspace.
- [ ] Add an explicit `container:` to every job.
- [ ] For each job's container image: confirm it has the tools that job's
      steps need, and that it's glibc-based if any step uses a
      JavaScript-based action.
- [ ] Identify every step that assumes a Docker daemon (builds, registry
      logins, "load locally then scan" patterns, non-job-level container
      actions) and replace them with daemonless equivalents, or isolate
      them onto a `dind`-mode scale set.
- [ ] Set `defaults.run.shell: bash` wherever `run:` steps use Bash-only
      syntax, and confirm Bash exists in the chosen image.
- [ ] Watch for actions that fail immediately after apparently succeeding
      — it's often a broken `$HOME`-relative artifact-upload path, not an
      actual failure of the tool's core work.
