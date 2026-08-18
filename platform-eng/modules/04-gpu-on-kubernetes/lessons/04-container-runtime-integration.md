---
lesson: "04.4"
title: "Container runtime integration — CDI, the runtime hook, and what lands in the container"
module: "04"
concept: "CDI & container runtime integration"
status: not-started
est_time: "9h"
prev: "03-device-plugin-recap-pod-resources.md"
next: "05-driver-lifecycle-upgrades.md"
artifacts: []
sources: 5
---

# 04.4 · Container runtime integration — CDI, the runtime hook, and what lands in the container

> **Concept.** A GPU pod is just a container with the right device nodes and driver libraries mounted in — CDI or the legacy hook is what puts them there.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Lesson 04.3 ended at the device plugin's `Allocate` RPC returning environment variables, device mounts, and CDI device names to the kubelet, and gave you the pod-resources API to find out which pod holds which device UUID. That lesson deliberately stopped there — it told you *what* gets decided, not *how* it gets carried across the container boundary. This lesson closes that gap: something has to actually create `/dev/nvidia0` inside the container's namespace, bind-mount `libcuda.so` from the host, and drop in `nvidia-smi`. That something is either a legacy imperative hook or the modern declarative CDI mechanism, and telling them apart — and reading either one directly — is what turns "the pod won't see the GPU" from a mystery into a five-minute diagnosis.

## Why this matters

`Allocate` returning the right env vars and mounts is necessary but not sufficient — when a "GPU pod" starts but CUDA can't find a device, or throws "CUDA driver version is insufficient for CUDA runtime version," the bug is almost never in your application code. It's in this injection layer, and it's exactly the kind of failure this module's calibration names as an interview probe: debugging "CUDA driver insufficient" live is table stakes for a GPU-fleet platform role. As the engineer who owns GPUs, you're the person expected to look *inside* a running container, know precisely which files should be there and where they came from, and say "the host driver is too old" or "the CDI spec is stale" with confidence — not guess. There's also a live version-sensitivity trap here: as of GPU Operator v25.10.0, **CDI is the default injection path** out of the box, which means an engineer who learned this stack on an older Operator version and still assumes the legacy hook is running will misdiagnose the very first thing they look at. This lesson makes the container's GPU surface legible enough that your failure-mode log — and your on-call self — can reason about it without guessing.

## What's new here (calibration)

Module 02 covered the *decision*: the device plugin's `Allocate` chooses devices and returns env vars/mounts/CDI names. Module 03 covered the *hardware*: the driver stack on the host, kernel modules, and what a GPU actually is at the silicon level. **Neither is re-taught here.** What's new:

- **The mechanism that carries a device-plugin decision across the container boundary** — the legacy imperative hook vs. the declarative CDI spec, and how to read either one directly instead of inferring from symptoms.
- **The exact runtime support matrix, with precise versions** — which container runtimes support CDI, which need explicit configuration, and which don't yet — confirmed against the CDI spec's own documentation rather than assumed.
- **The GPU Operator 25.10+ default-CDI change** — a genuinely new fact that inverts the "which path am I debugging" default assumption anyone who learned this stack even a year earlier would carry.
- **The concrete file-by-file inventory of what lands inside a GPU container**, and the version-mismatch failure mode that inventory explains — the daily on-call skill, not just the architecture.

## Core concepts

### The two env vars that decide everything

Both injection modes are driven by two environment variables (set by the device plugin's `Allocate`, by a CDI device name, or — dangerously — by hand):

- **`NVIDIA_VISIBLE_DEVICES`** — *which GPUs* are exposed. Values: `all`, `none`, `void` (or unset), or a comma list of indices (`0,1`), GPU UUIDs (`GPU-<uuid>`), or MIG UUIDs (`MIG-<uuid>`). `void`/unset ⇒ the toolkit does nothing and you get a plain container with no GPU. `none` ⇒ no device nodes but driver libraries still injected.
- **`NVIDIA_DRIVER_CAPABILITIES`** — *which driver libraries* get mounted. Comma list of: `compute` (CUDA + OpenCL: `libcuda`, `libnvidia-ptxjitcompiler`), `utility` (`nvidia-smi` + NVML `libnvidia-ml`), `graphics` (OpenGL/GLX), `video` (NVENC/NVDEC/`libnvcuvid`), `display`, `compat32`, or `all`. Unset defaults to `utility` only — which is why a CUDA program in a hand-rolled image fails with "no CUDA driver": without `compute`, `libcuda.so` is never mounted. CUDA base images set `compute,utility`.

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

It works, but it is **imperative and runtime-coupled**: you must install a special OCI runtime, the mutation happens invisibly at start, and the same logic had to be re-plumbed for Docker, containerd, and CRI-O separately.

### Modern path: CDI

CDI (Container Device Interface) replaces the hook with a **declarative spec**. It's a CNCF-tagged, vendor-neutral standard — confirmed at spec version **v0.6.0** directly from the `cncf-tags/container-device-interface` repository — used by more than just NVIDIA (Intel and AMD publish CDI specs for their own devices too). A YAML/JSON file names devices and lists the exact edits — device nodes, mounts, env, optional hooks — to apply for each. The runtime reads the spec and applies those edits itself, using **stock runc** — no special OCI wrapper required. Specs live in:

- **`/etc/cdi/`** — persistent, admin-generated (`nvidia-ctk cdi generate`);
- **`/var/run/cdi/`** — transient, generated at runtime (e.g. by the GPU Operator or the device plugin), wiped on reboot.

A device is referenced by a fully-qualified name like `nvidia.com/gpu=GPU-<uuid>`, `nvidia.com/gpu=0`, or `nvidia.com/gpu=all`. Generate and inspect:

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
nvidia-ctk cdi list
# nvidia.com/gpu=all
# nvidia.com/gpu=0
# nvidia.com/gpu=GPU-a1b2c3d4-...
```

A trimmed spec shows the mechanism — every file that lands in the container is *named here*, no magic:

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

**The runtime support matrix** — confirmed directly against the CDI spec's own documentation, not assumed from tutorials:

| Runtime | CDI support | Notes |
|---|---|---|
| CRI-O | **Default-on** | No extra config needed to enable CDI. |
| containerd | Supported, **not default** | Requires `enable_cdi = true` in `containerd`'s config — a common source of "CDI spec exists but nothing applies it" confusion. |
| Docker | ≥ 25.0.0 | **Default-on** starting ≥ 28.2.0; earlier 25.x/26.x/27.x versions support it but require opt-in. |
| Podman | ≥ 3.2.0 (partial) | **Full support** from ≥ 4.1.0-rc.1. |

**Why the ecosystem moved to CDI:** it is vendor-neutral and runtime-agnostic (any CDI-aware runtime, no NVIDIA-specific OCI wrapper), the injection is declarative and inspectable (you can `cat` the spec and diff it, instead of reverse-engineering a hook), it is reproducible, and — decisively — **Kubernetes Dynamic Resource Allocation (DRA) and native device management are built on CDI** (lesson 04.9 goes deep on DRA; the spec dependency starts here). The legacy hook was Docker/runc-shaped and awkward across containerd/CRI-O; CDI is the standard the whole stack has converged on.

### GPU Operator 25.10+: CDI becomes the default, not just an option

This is the fact that most changes how you should approach debugging on a current cluster: as of **GPU Operator v25.10.0**, CDI is the default GPU-injection path out of the box, not something you have to opt into via `ClusterPolicy`. If you learned this stack on an older Operator line, or you're reading documentation written before that release, your instinct will be to assume the legacy hook is what's running — and on a 25.10+ install, that assumption is now wrong by default. The practical consequence: before you go hunting for an `nvidia-container-runtime` OCI wrapper and a prestart hook in `config.json`, run `nvidia-ctk cdi list` and check whether a CDI spec actually resolves your device — that single command tells you which path you're debugging, and it should be your first move, not a fallback. (The GPU Operator's own version history has moved further since — v26.3.0 added the `NVIDIADriver` CRD and faster same-version driver-pod restarts — covered in lesson 04.5, but the CDI-default shift is the one that changes *this* lesson's debugging habits.)

### What actually lands inside the container

Regardless of path, a healthy `compute,utility` GPU container contains:

- **Device nodes:** `/dev/nvidia0` (one per visible GPU), `/dev/nvidiactl`, `/dev/nvidia-uvm`, `/dev/nvidia-uvm-tools`, and `/dev/nvidia-modeset` (graphics/display).
- **Driver userspace libraries — mounted from the host:** `libcuda.so.<driver-ver>` (the CUDA *driver* API), `libnvidia-ml.so.<driver-ver>` (NVML), `libnvidia-ptxjitcompiler`, plus video/graphics libs per capability. These match the **host kernel module version**, not anything baked into the image.
- **Binaries:** `nvidia-smi`, `nvidia-debugdump`, `nvidia-cuda-mps-control` (from host).
- **From the image, NOT injected:** the CUDA *runtime*/toolkit — `libcudart.so`, `nvcc`, cuDNN. This split is the crux of the version-mismatch failure below.

## Perspectives

**Container-runtime engineer perspective.** CDI vs. the legacy hook is, at bottom, "declarative spec vs. imperative prestart hook." A spec you can `cat` and diff is a fundamentally different debugging object than a binary that mutates a container invisibly at create time — the entire appeal of CDI to someone who has to support this in production is that the injection stops being a black box.

**Debugging/on-call perspective.** The two-env-var model — `NVIDIA_VISIBLE_DEVICES` and `NVIDIA_DRIVER_CAPABILITIES` — is where "works on my machine" GPU images go to die. An image built against a base that sets `compute,utility` behaves completely differently from one that doesn't, and the failure mode (`nvidia-smi` works, CUDA programs don't) is subtle enough that it wastes real on-call time if you haven't internalized which capability gates which library.

**Security perspective.** `NVIDIA_VISIBLE_DEVICES=all` set by hand in a manifest is a real isolation escape, not a theoretical one — it exposes every GPU on the node into a single container, bypassing whatever the device plugin allocated. The GPU Operator's guard (`accept-nvidia-visible-devices-envvar-when-unprivileged=false`) exists precisely because this pattern has been exploited/discovered as a way for an unprivileged pod to reach hardware it never requested.

**Standards/ecosystem perspective.** CDI isn't an NVIDIA-only convenience — it's a CNCF-tagged, vendor-neutral spec that Intel and AMD also publish device specs against. Betting your mental model on CDI rather than the legacy hook is also a bet on where the broader container ecosystem, not just the NVIDIA stack, is heading — DRA's dependency on CDI (lesson 04.9) is the clearest evidence of that direction inside Kubernetes itself.

## Real-world use cases

- **[CDI specification (cncf-tags/container-device-interface)](https://github.com/cncf-tags/container-device-interface)** — the primary source for everything in this lesson's runtime matrix and spec format; read the `containerEdits` schema directly rather than trusting a summary.
- **[NVIDIA/gpu-operator#1220](https://github.com/NVIDIA/gpu-operator/issues/1220)** — a real practitioner incident ("gpu-operator breaks when upgrading EKS to K8s v1.30") with the log line `failed to get sandbox runtime: no runtime for 'nvidia' is configured` — a runtime-wiring failure directly in this lesson's territory, not a hypothetical. Environment: K8s v1.30, GPU Operator v24.6.2, driver 535.183.01, Ubuntu 22.04, previously working on 1.29.
- **Modal, ["How we achieved truly serverless GPUs"](https://modal.com/blog/truly-serverless-gpus)** — describes CUDA-side checkpoint/restore techniques for fast cold starts, adjacent to this lesson's container/device boundary theme (what has to be re-established when a GPU container resumes). *(Confirmed real via search this session, not independently fetched — treat as high-confidence but unconfirmed.)*
- **Honest gap:** no company engineering blog narrating a CDI migration in production with concrete before/after numbers turned up in research for this lesson. Rather than inventing one, lean on the CDI spec repository itself and the real gpu-operator issue above as the strongest evidence available.

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

Now toggle capabilities to *see the libs change*. Run two pods differing only in one env var:

```bash
# Pod A: NVIDIA_DRIVER_CAPABILITIES=utility
kubectl exec pod-a -- bash -c 'ls /usr/lib/x86_64-linux-gnu/ | grep -c libcuda'
# 0        <- no libcuda: nvidia-smi works, CUDA programs fail

# Pod B: NVIDIA_DRIVER_CAPABILITIES=compute,utility
kubectl exec pod-b -- bash -c 'ls /usr/lib/x86_64-linux-gnu/ | grep -c libcuda'
# 1        <- libcuda.so.<ver> now mounted
```

Finally, inspect the node's CDI spec to tie the mounts back to their source — and confirm which path (CDI or legacy) is actually active on this node before assuming:

```bash
nvidia-ctk cdi list                          # succeeds & lists devices ⇒ CDI is active
sudo cat /etc/cdi/nvidia.yaml | less          # persistent spec
ls /var/run/cdi/                              # runtime-generated specs (operator/plugin)

# If CDI is NOT active, look for the legacy path instead:
grep -A2 nvidia /etc/containerd/config.toml   # containerd: runtime configured as nvidia?
crictl inspect <container-id> | grep -A5 hooks # legacy: prestart hook present in config.json
```

Every path you found inside the container appears as a `mounts:` or `deviceNodes:` entry in that spec when CDI is active — proof there is no magic, just declared edits. On a GPU Operator 25.10+ install, expect `nvidia-ctk cdi list` to succeed by default; if it doesn't, that mismatch is itself worth investigating before you go further.

## Practice — feeds the deliverable

**Task.** Produce a written **"what's in the container and why"** inventory for the module's failure-mode log.

1. `kubectl exec` into a running GPU pod and record: the `/dev/nvidia*` nodes present; the driver `.so` libraries mounted and their host source paths; the two version lines from `nvidia-smi` (Driver Version and CUDA Version); and the `NVIDIA_VISIBLE_DEVICES` / `NVIDIA_DRIVER_CAPABILITIES` values (`env | grep NVIDIA`).
2. Toggle `NVIDIA_DRIVER_CAPABILITIES` between `utility` and `compute,utility` on two otherwise-identical pods and record which libraries appear/disappear.
3. Determine whether your node is using CDI or the legacy hook: run `nvidia-ctk cdi list` and check the runtime config (`enable_cdi` for containerd, or an `nvidia-container-runtime` OCI runtime + prestart hook for the legacy path). Record your GPU Operator version alongside the answer — cross-check against the 25.10+ default-CDI fact above.
4. If CDI is active, `cat` the spec in `/etc/cdi/` (or list `/var/run/cdi/`) and map at least three files from step 1 to their `mounts`/`deviceNodes` entries in the spec.

**Acceptance.** A written inventory in the failure-mode log that, for one real GPU pod, lists the device nodes, driver libraries (with host sources), binaries, and the two `nvidia-smi` version numbers; shows the capability-toggle diff; states which injection path (CDI or legacy) is active and how you determined it; and — if CDI — maps container files to their spec origin. Someone reading it should be able to diagnose a missing-`libcuda` or version-mismatch failure without touching the cluster.

## Common pitfalls

1. **Assuming "CUDA driver version is insufficient" is an image problem.** The container image ships a CUDA runtime/toolkit that's newer than what the host driver supports. `libcuda` and the kernel module come from the **host**, not the image — the fix always lives on the node.
2. **Setting `NVIDIA_VISIBLE_DEVICES=all` by hand.** It bypasses the device plugin's allocation entirely and exposes every GPU on the node — a real isolation escape, not a convenience shortcut. Let `Allocate` set it.
3. **Forgetting `NVIDIA_DRIVER_CAPABILITIES` defaults to `utility` only.** A hand-rolled image without an explicit `compute` capability will run `nvidia-smi` fine and then fail every CUDA call, because `libcuda.so` was never mounted.
4. **Assuming the legacy hook is still what's running.** On GPU Operator v25.10+, CDI is the default injection path out of the box. Check `nvidia-ctk cdi list` before assuming which mechanism you're debugging — guessing wrong wastes the first several minutes of any incident.
5. **Confusing "the toolkit can generate CDI specs" with "the runtime will apply them."** containerd requires explicit `enable_cdi = true` in its config — a generated spec sitting in `/etc/cdi/` that the runtime never reads produces the exact same symptom as no spec at all.

## Self-check

- "CUDA driver version is insufficient for CUDA runtime version" *inside* a container — what's the root cause and where do you look? **Answer:** The container image ships a CUDA **runtime/toolkit** (`libcudart`, from the image) that is newer than what the **driver** supports. The driver userspace (`libcuda`) and kernel module are mounted/loaded from the **host**, so this is a host-driver-too-old problem, not something you fix by rebuilding the image. Look at `nvidia-smi` inside the container: its `CUDA Version:` field is the maximum CUDA the host driver supports; compare it to the toolkit version in the image (`nvcc --version` or the image tag, e.g. `cuda:12.6`). If the image's toolkit version exceeds the driver's max CUDA, you hit this error. Fix: upgrade the host NVIDIA driver, use forward-compat packages, or use an older CUDA image — always a node-side fix.
- `NVIDIA_VISIBLE_DEVICES=all` vs. a specific UUID — what changes? **Answer:** `all` exposes every GPU on the node into the container — all device nodes and all GPUs visible to CUDA/`nvidia-smi` — which bypasses the device plugin's allocation and breaks isolation: a pod could see and use GPUs it never requested. A specific UUID (`GPU-<uuid>`, what `Allocate` normally sets) exposes only that one GPU's device node. Under Kubernetes you want the specific UUID; `all` set by hand in a manifest is a common isolation escape, which is why the GPU Operator provides `accept-nvidia-visible-devices-envvar-when-unprivileged=false` to refuse env-based overrides from unprivileged pods.
- CDI vs. the legacy runtime-hook injection — what's the difference and why did the ecosystem move to CDI? **Answer:** The legacy path uses a special `nvidia-container-runtime` wrapping runc that injects a prestart hook (`nvidia-container-runtime-hook` → `nvidia-container-cli`) which imperatively mutates the container at create time — runtime-coupled and invisible. CDI replaces that with a declarative spec in `/etc/cdi` or `/var/run/cdi` describing named devices and their exact edits, applied by any CDI-aware runtime using stock runc. The ecosystem moved to CDI because it's vendor-neutral and runtime-agnostic, inspectable and reproducible, and it's the substrate Kubernetes DRA and native device support are built on.
- Since GPU Operator 25.10+, how do you tell whether a node is using CDI or the legacy hook, and why does it matter? **Answer:** Run `nvidia-ctk cdi list` — if it succeeds and lists devices, and the runtime is configured for CDI (`enable_cdi = true` for containerd, default-on for CRI-O), CDI is active; if instead you find an `nvidia-container-runtime` OCI runtime entry plus a prestart hook in the container's `config.json`, it's the legacy path. It matters because before 25.10 the safe default assumption was the legacy hook; on 25.10+ CDI is the out-of-the-box default, so debugging by reading `/etc/cdi/`/`/var/run/cdi/` specs is now the correct first move rather than hunting for a prestart hook that may not be running at all.

## Connections & what's next

This lesson is the mechanism layer between two things you already have: lesson 04.3's `Allocate` output (env vars, device names, mounts) and module 03's hardware/driver stack on the host. It's also the substrate lesson 04.9 depends on directly — Dynamic Resource Allocation's device model is built on CDI, so the spec format and runtime matrix here are not a side topic, they're a prerequisite you'll recognize by name later. The GPU Operator's own installation and reconfiguration behavior — including the CDI-default shift covered here — is itself part of the driver/toolkit lifecycle the next lesson manages end to end.

Next: **[04.5 · Driver lifecycle & fleet upgrades](05-driver-lifecycle-upgrades.md)** takes the host-side driver dependency this lesson kept surfacing (the container's `libcuda`/kernel-module version always traces back to the host) and turns it into a managed process — designing safe rolling upgrades across a fleet without breaking every running GPU pod's injection at once.

## References & further reading

**Primary sources**
- [CDI specification (cncf-tags/container-device-interface)](https://github.com/cncf-tags/container-device-interface) — confirmed v0.6.0; the spec format your `/etc/cdi/nvidia.yaml` conforms to, and the source of the runtime support matrix above.
- [NVIDIA k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) (confirmed v0.17.1) — the source of the `Allocate` env-var/mount contract this lesson picks up from lesson 04.3.

**Real-world engineering blogs**
- [NVIDIA/gpu-operator#1220](https://github.com/NVIDIA/gpu-operator/issues/1220) — a real practitioner incident with the exact runtime-wiring failure log line (`no runtime for 'nvidia' is configured`) this lesson's mechanism explains.
- Modal, ["How we achieved truly serverless GPUs"](https://modal.com/blog/truly-serverless-gpus) — CUDA-side checkpoint/restore adjacent to the container/device boundary theme. *(Confirmed real via search, not independently fetched this session — spot-check before citing.)*

**Deeper dives**
- [NVIDIA GPU Operator (repo)](https://github.com/NVIDIA/gpu-operator) — read the `ClusterPolicy` toolkit configuration and release notes for the version history behind the 25.10+ CDI-default change described above.
