---
lesson: "04.4"
title: "Container runtime integration — CDI, the runtime hook, and what lands in the container"
module: "04"
concept: "Container runtime integration — CDI, the runtime hook, and what lands in the container"
status: not-started
est_time: "6h"
artifacts: []
---
# 04.4 · Container runtime integration — CDI, the runtime hook, and what lands in the container
> **Concept.** A GPU pod is just a container with the right device nodes and driver libraries mounted in — CDI or the legacy hook is what puts them there.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Why this matters

`Allocate` returned env vars and mounts (lesson 04.3). But *something* has to act on
them — create `/dev/nvidia0` inside the container's namespace, bind-mount `libcuda.so`
from the host, drop in `nvidia-smi`. When a "GPU pod" starts but CUDA can't find a
device, or throws "driver version is insufficient," the bug is almost never in your
code — it is in this injection layer. As the platform engineer who owns GPUs, you are
the person who has to look *inside* a running container, know exactly which files should
be there and where they came from, and say "the host driver is too old" or "the CDI spec
is stale" with confidence. This lesson makes the container's GPU surface legible so your
failure-mode log (and your on-call self) can reason about it.

## What's new here

Modules 02/03 covered the *decision*: the device plugin's `Allocate` chooses devices and
returns env/mounts, and the GPU/driver hardware layer (03) explained the driver stack on
the host. **New here is the *mechanism* that carries those decisions across the container
boundary**, and it comes in two flavors you must be able to tell apart:

- the **legacy `nvidia-container-runtime` hook** — imperative, runs at container create;
- **CDI (Container Device Interface)** — declarative, a static spec the runtime applies.

And the concrete result of either: exactly which `/dev/nvidia*` nodes, `.so` libraries,
and binaries appear inside the container, and which env vars decide that set.

## Core notes

### The two env vars that decide everything

Both injection modes are driven by two environment variables (set by the device plugin's
`Allocate`, or by a CDI device name, or — dangerously — by hand):

- **`NVIDIA_VISIBLE_DEVICES`** — *which GPUs* are exposed. Values: `all`, `none`, `void`
  (or unset), or a comma list of indices (`0,1`), GPU UUIDs (`GPU-<uuid>`), or MIG UUIDs
  (`MIG-<uuid>`). `void`/unset ⇒ the toolkit does nothing and you get a plain container
  with no GPU. `none` ⇒ no device nodes but driver libraries still injected.
- **`NVIDIA_DRIVER_CAPABILITIES`** — *which driver libraries* get mounted. Comma list of:
  `compute` (CUDA + OpenCL: `libcuda`, `libnvidia-ptxjitcompiler`), `utility`
  (`nvidia-smi` + NVML `libnvidia-ml`), `graphics` (OpenGL/GLX), `video`
  (NVENC/NVDEC/`libnvcuvid`), `display`, `compat32`, or `all`. Unset defaults to
  `utility` only — which is why a CUDA program in a hand-rolled image fails with "no CUDA
  driver": without `compute`, `libcuda.so` is never mounted. CUDA base images set
  `compute,utility`.

### Legacy path: nvidia-container-runtime + the hook

Classic mode wires GPUs at container-create time:

```
containerd/CRI ──► nvidia-container-runtime  (a thin wrapper around runc)
                        │ injects an OCI prestart/createRuntime hook into config.json
                        ▼
                   nvidia-container-runtime-hook   (binary from nvidia-container-toolkit)
                        │ reads NVIDIA_VISIBLE_DEVICES / _DRIVER_CAPABILITIES
                        ▼
                   nvidia-container-cli configure   (libnvidia-container)
                        │ enters the container mount namespace and:
                        ├─ mknod /dev/nvidia0, /dev/nvidiactl, /dev/nvidia-uvm ...
                        ├─ bind-mounts host driver libs (libcuda.so.<ver>, libnvidia-ml ...)
                        ├─ bind-mounts binaries (nvidia-smi)
                        └─ updates the ldcache
```

It works, but it is **imperative and runtime-coupled**: you must install a special OCI
runtime, the mutation happens invisibly at start, and the same logic had to be
re-plumbed for Docker, containerd, and CRI-O separately.

### Modern path: CDI

CDI (a CNCF standard, `cncf-tags/container-device-interface`) replaces the hook with a
**declarative spec**. A YAML/JSON file names devices and lists the exact edits — device
nodes, mounts, env, optional hooks — to apply for each. The runtime (containerd ≥ 1.7,
CRI-O, Podman, Docker ≥ 25, or `nvidia-container-runtime` in `cdi` mode) reads the spec
and applies those edits itself, using **stock runc**. Specs live in:

- **`/etc/cdi/`** — persistent, admin-generated (`nvidia-ctk cdi generate`);
- **`/var/run/cdi/`** — transient, generated at runtime (e.g. by the GPU Operator or the
  device plugin), wiped on reboot.

A device is referenced by a fully-qualified name like `nvidia.com/gpu=GPU-<uuid>`,
`nvidia.com/gpu=0`, or `nvidia.com/gpu=all`. Generate and inspect:

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
nvidia-ctk cdi list
# nvidia.com/gpu=all
# nvidia.com/gpu=0
# nvidia.com/gpu=GPU-a1b2c3d4-...
```

A trimmed spec shows the mechanism — every file that lands in the container is *named
here*, no magic:

```yaml
cdiVersion: "0.6.0"
kind: nvidia.com/gpu
devices:
  - name: "0"
    containerEdits:
      deviceNodes:
        - path: /dev/nvidia0
        - path: /dev/nvidiactl
      mounts:
        - hostPath: /usr/lib/x86_64-linux-gnu/libcuda.so.550.90.07
          containerPath: /usr/lib/x86_64-linux-gnu/libcuda.so.550.90.07
containerEdits:            # applied to ALL devices of this kind
  deviceNodes:
    - path: /dev/nvidia-uvm
    - path: /dev/nvidia-uvm-tools
  hooks:
    - hookName: createContainer
      path: /usr/bin/nvidia-ctk
      args: ["nvidia-ctk", "hook", "update-ldcache"]
```

**Why the ecosystem moved to CDI:** it is vendor-neutral and runtime-agnostic (any
CDI-aware runtime, no NVIDIA-specific OCI wrapper), the injection is *declarative and
inspectable* (you can `cat` the spec and diff it, instead of reverse-engineering a hook),
it is reproducible, and — decisively — **Kubernetes Dynamic Resource Allocation (DRA)
and native device management are built on CDI**. The legacy hook was Docker/runc-shaped
and awkward across containerd/CRI-O; CDI is the standard the whole stack is converging on.

### What actually lands inside the container

Regardless of path, a healthy `compute,utility` GPU container contains:

- **Device nodes:** `/dev/nvidia0` (one per visible GPU), `/dev/nvidiactl`,
  `/dev/nvidia-uvm`, `/dev/nvidia-uvm-tools`, and `/dev/nvidia-modeset` (graphics/display).
- **Driver userspace libraries — mounted from the host:** `libcuda.so.<driver-ver>` (the
  CUDA *driver* API), `libnvidia-ml.so.<driver-ver>` (NVML), `libnvidia-ptxjitcompiler`,
  plus video/graphics libs per capability. These match the **host kernel module version**.
- **Binaries:** `nvidia-smi`, `nvidia-debugdump`, `nvidia-cuda-mps-control` (from host).
- **From the image, NOT injected:** the CUDA *runtime* / toolkit — `libcudart.so`,
  `nvcc`, cuDNN. This split is the crux of the version-mismatch failure below.

## Worked example

Exec into a running GPU pod and inventory its GPU surface.

```bash
kubectl exec -it trainer-0 -c cuda -- bash

# 1) Device nodes present?
ls -l /dev/nvidia*
# crw-rw-rw- 1 root root 195,   0 /dev/nvidia0
# crw-rw-rw- 1 root root 195, 255 /dev/nvidiactl
# crw-rw-rw- 1 root root 508,   0 /dev/nvidia-uvm

# 2) Which driver libs were bind-mounted, and from where?
mount | grep -i nvidia
grep nvidia /proc/mounts
# /usr/lib/x86_64-linux-gnu/libcuda.so.550.90.07  (bind from host driver)
ls -l /usr/lib/x86_64-linux-gnu/ | grep -E 'libcuda|libnvidia-ml'

# 3) The driver's own view — note the two version lines:
nvidia-smi
# Driver Version: 550.90.07   CUDA Version: 12.4   <- max CUDA the driver supports
```

Now toggle capabilities to *see the libs change*. Run two pods differing only in one env
var:

```bash
# Pod A: NVIDIA_DRIVER_CAPABILITIES=utility
kubectl exec pod-a -- bash -c 'ls /usr/lib/x86_64-linux-gnu/ | grep -c libcuda'
# 0        <- no libcuda: nvidia-smi works, CUDA programs fail

# Pod B: NVIDIA_DRIVER_CAPABILITIES=compute,utility
kubectl exec pod-b -- bash -c 'ls /usr/lib/x86_64-linux-gnu/ | grep -c libcuda'
# 1        <- libcuda.so.<ver> now mounted
```

Finally, inspect the node's CDI spec to tie the mounts back to their source:

```bash
sudo cat /etc/cdi/nvidia.yaml | less        # persistent spec
ls /var/run/cdi/                            # runtime-generated specs (operator/plugin)
nvidia-ctk cdi list
```

Every path you found inside the container appears as a `mounts:` or `deviceNodes:` entry
in that spec (CDI mode) — proof there is no magic, just declared edits.

## Practice — feeds the deliverable

**Task.** Produce a written **"what's in the container and why"** inventory for the
module's failure-mode log.

1. `kubectl exec` into a running GPU pod and record: the `/dev/nvidia*` nodes present;
   the driver `.so` libraries mounted and their host source paths; the two version lines
   from `nvidia-smi` (Driver Version and CUDA Version); and the `NVIDIA_VISIBLE_DEVICES`
   / `NVIDIA_DRIVER_CAPABILITIES` values (`env | grep NVIDIA`).
2. Toggle `NVIDIA_DRIVER_CAPABILITIES` between `utility` and `compute,utility` on two
   otherwise-identical pods and record which libraries appear/disappear.
3. On the node, `cat` the CDI spec in `/etc/cdi/` (or list `/var/run/cdi/`) and map at
   least three files from step 1 to their `mounts`/`deviceNodes` entries in the spec.
4. State whether your node uses CDI or the legacy hook, and how you can tell
   (`nvidia-ctk cdi list` succeeds and the runtime is configured `cdi`, vs. an
   `nvidia-container-runtime` OCI runtime + prestart hook in `config.json`).

**Acceptance.** A written inventory in the failure-mode log that, for one real GPU pod,
lists the device nodes, driver libraries (with host sources), binaries, and the two
`nvidia-smi` version numbers; shows the capability-toggle diff; and maps container files
to their CDI-spec origin. Someone reading it should be able to diagnose a missing-`libcuda`
or version-mismatch failure without touching the cluster.

## Self-check

**(a) "CUDA driver version is insufficient for CUDA runtime version" *inside* a
container — what's the root cause and where do you look?**

**Answer:** The container image ships a CUDA **runtime/toolkit** (`libcudart`, from the
image) that is newer than what the **driver** supports. The driver userspace (`libcuda`)
and kernel module are **mounted/loaded from the host**, so this is a *host driver too old*
problem, not an image problem you can fix by rebuilding. Look at `nvidia-smi` inside the
container: its `CUDA Version:` field is the *maximum* CUDA the host driver supports;
compare it to the toolkit version in the image (`nvcc --version` or the image tag, e.g.
`cuda:12.6`). If image toolkit > driver's max CUDA, you hit this error. Fix: upgrade the
**host** NVIDIA driver (or use forward-compat packages, or an older CUDA image). Because
`libcuda` comes from the host, the fix always lives on the node, not in the container.

**(b) `NVIDIA_VISIBLE_DEVICES=all` vs a specific UUID — what changes?**

**Answer:** `all` exposes **every GPU on the node** into the container — all device nodes
and all GPUs visible to CUDA/`nvidia-smi` — which **bypasses the device plugin's
allocation and breaks isolation**: a pod could see and use GPUs it never requested. A
specific UUID (`GPU-<uuid>`, what `Allocate` normally sets) exposes **only that one GPU's**
device node and makes only it visible. Under Kubernetes you want the specific UUID; `all`
set by hand in a manifest is a common isolation escape, which is why the GPU Operator
provides `accept-nvidia-visible-devices-envvar-when-unprivileged=false` to refuse
env-based overrides from unprivileged pods.

**(c) CDI vs the legacy runtime-hook injection — what's the difference and why did the
ecosystem move to CDI?**

**Answer:** The legacy path uses a special `nvidia-container-runtime` wrapping runc that
injects a prestart **hook** (`nvidia-container-runtime-hook` → `nvidia-container-cli`)
which *imperatively* mutates the container at create time — runtime-coupled and invisible.
CDI replaces that with a **declarative spec** in `/etc/cdi` or `/var/run/cdi` describing
named devices and their exact edits (device nodes, mounts, env, hooks), applied by any
CDI-aware runtime using stock runc. The ecosystem moved to CDI because it is
vendor-neutral and runtime-agnostic (no NVIDIA-specific OCI runtime), inspectable and
reproducible (you can read/diff the spec), and it is the substrate Kubernetes DRA and
native device support are built on — so the fragile, Docker/runc-shaped hook is being
retired in its favor.

## Resources

1. **NVIDIA Container Toolkit — CDI & specialized config** (modes, env vars, generating
   specs): https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/docker-specialized.html
   — the operational reference for `NVIDIA_VISIBLE_DEVICES`, `NVIDIA_DRIVER_CAPABILITIES`,
   and `nvidia-ctk cdi generate`.
2. **CDI specification** (`cncf-tags/container-device-interface`):
   https://github.com/cncf-tags/container-device-interface — the spec format your
   `/etc/cdi/nvidia.yaml` conforms to; read the `containerEdits` schema.
3. **nvidia-container-runtime-hook internals** (the legacy path, for the failure-mode
   log): https://deepwiki.com/NVIDIA/nvidia-container-toolkit/2.2-nvidia-container-runtime-hook
