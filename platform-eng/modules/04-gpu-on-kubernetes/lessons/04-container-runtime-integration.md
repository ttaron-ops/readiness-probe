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
sources: 12
---

# 04.4 · Container runtime integration — CDI, the runtime hook, and what lands in the container

> **Concept.** A GPU pod is just a container with the right device nodes and driver libraries mounted in — CDI or the legacy hook is what puts them there.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Lesson 04.3 ended at the device plugin's `Allocate` RPC returning environment variables, device mounts, and CDI device names to the kubelet, and gave you the pod-resources API to find out which pod holds which device UUID. That lesson deliberately stopped there — it told you *what* gets decided, not *how* it gets carried across the container boundary. This lesson closes that gap: something has to actually create `/dev/nvidia0` inside the container's namespace, bind-mount `libcuda.so` from the host, and drop in `nvidia-smi`. That something is either a legacy imperative hook or the modern declarative CDI mechanism, and telling them apart — and reading either one directly — is what turns "the pod won't see the GPU" from a mystery into a five-minute diagnosis.

## Why this matters

`Allocate` returning the right env vars and mounts is necessary but not sufficient. When a "GPU pod" starts but CUDA cannot find a device, or throws `CUDA driver version is insufficient for CUDA runtime version`, the bug is almost never in your application code. It is in this injection layer, and it is exactly the kind of failure this module's calibration names as an interview probe: debugging "CUDA driver insufficient" live is table stakes for a GPU-fleet platform role.

The structural reason this layer produces so many incidents is that a GPU container is assembled from **two independently versioned halves that meet at container-create time**. The kernel module and its matching userspace driver library (`libcuda.so.<driver-version>`, `libnvidia-ml.so.<driver-version>`) belong to the *host* and are owned by whoever runs your nodes. The CUDA runtime, cuBLAS, cuDNN, NCCL and your framework belong to the *image* and are owned by whoever builds it. Nothing in Kubernetes checks that the two halves are compatible. The injection layer is the seam, and every seam failure looks like an application bug from the outside.

There is a second reason to know this layer cold: **it is moving under you.** The mechanism most blog posts and most people's mental models describe — a special OCI runtime that injects an imperative prestart hook — is no longer the default. As of NVIDIA Container Toolkit v1.20.0 the default runtime mode is `auto`, and on any normal NVML-capable Linux GPU node `auto` resolves to `jit-cdi`: the runtime synthesises a CDI specification *in memory*, per container, and applies it with stock `runc`. There is frequently no CDI file on disk at all. An engineer who opens an incident by grepping `config.json` for a prestart hook, or by running `nvidia-ctk cdi list` and concluding "CDI isn't active because there are no specs", will misread the very first thing they look at.

## What's new here (calibration)

Module 02 covered the *decision*: the device plugin's `Allocate` chooses devices and returns env vars, mounts, and CDI device names. Module 03 covered the *hardware and the compatibility rules*: the driver stack on the host, the kernel-module/userspace match, and the CUDA minor-version-compatibility bands. **Neither is re-taught here.** What is new:

- **The OCI runtime specification as the actual interface.** Everything both injection paths do reduces to four fields in `config.json` — `linux.devices`, `linux.resources.devices`, `mounts`, and `hooks` — plus the hook phases and, decisively, *which namespace each hook phase runs in*. That last detail is the entire mechanical difference between the legacy and CDI designs.
- **The five runtime modes of `nvidia-container-runtime`** (`legacy`, `csv`, `cdi`, `jit-cdi`, `auto`) and the exact resolution rule `auto` uses — including why the modern default leaves no artefact on disk to inspect.
- **The complete CDI specification structure**, field by field, at spec version v1.1.0, with a real generated NVIDIA spec read line by line.
- **The real runtime wiring**: the `/etc/nvidia-container-runtime/config.toml` the toolkit actually writes, the containerd v2 and v3 runtime stanzas, drop-in config files, the CRI-O equivalent, and which containerd version turns `enable_cdi` on by default.
- **A file-by-file inventory of what lands inside a GPU container**, with device majors/minors, provenance, and the MIG capability device nodes — plus the mechanical explanation of the whole version-skew failure family that inventory produces.

## Core concepts

### 1. The problem: a GPU is not one device node

Start from what the container needs, not from what the tooling does.

A CUDA process on bare metal talks to the GPU through a chain: your code calls into `libcudart.so` (the CUDA *runtime* API), which calls into `libcuda.so` (the CUDA *driver* API), which issues `ioctl()` calls on file descriptors opened against character devices under `/dev`. The kernel module behind those devices does the actual work — submitting command buffers, mapping GPU memory into the process address space, handling faults.

So a container that runs CUDA needs, at minimum:

1. **The device nodes**, so `open("/dev/nvidiactl")` succeeds inside the container's mount namespace *and* the container's device cgroup permits the (major, minor) pair. Two separate gates; missing either one produces a different error.
2. **The host's `libcuda.so.<driver-version>`**, because the driver API library must match the loaded kernel module *exactly* — same build, not merely same version band. It therefore cannot be baked into a portable image.
3. **A dynamic-linker path that finds it.** A bind-mounted `libcuda.so.580.65.06` that is not in `/etc/ld.so.cache` and not in a directory listed in `/etc/ld.so.conf.d/` is invisible to `dlopen("libcuda.so.1")`.
4. **The `libcuda.so.1` SONAME symlink**, because that — not the versioned filename — is what applications actually `dlopen`.
5. **Optional extras** gated on what the workload does: `libnvidia-ml.so` + `nvidia-smi` for monitoring, `libnvcuvid`/`libnvidia-encode` for video, GL/EGL/Vulkan libraries for graphics, `libnvidia-ptxjitcompiler` for JIT-compiling PTX.

Everything in this lesson exists to get those five things into a container's namespaces without the image having to know the host's driver version. **The only interface available for doing that is the OCI runtime specification** — because by the time anything NVIDIA-specific runs, the container is already being described as an OCI bundle, and `runc` will only do what `config.json` says.

### 2. The OCI runtime spec is the whole contract

Four parts of `config.json` matter here (OCI Runtime Specification, `config.md`).

**`linux.devices`** — device nodes the runtime should `mknod` inside the container's rootfs:

```json
"linux": {
  "devices": [
    { "path": "/dev/nvidia0",   "type": "c", "major": 195, "minor": 0,
      "fileMode": 438, "uid": 0, "gid": 0 },
    { "path": "/dev/nvidiactl", "type": "c", "major": 195, "minor": 255,
      "fileMode": 438, "uid": 0, "gid": 0 }
  ]
}
```

**`linux.resources.devices`** — the device cgroup allow/deny list. The default entry in every Kubernetes container is `{"allow": false, "access": "rwm"}` — deny everything — so each injected device needs a matching allow rule or `open()` returns `EPERM` even though the node exists:

```json
"resources": {
  "devices": [
    { "allow": false, "access": "rwm" },
    { "allow": true, "type": "c", "major": 195, "minor": 0,   "access": "rw" },
    { "allow": true, "type": "c", "major": 195, "minor": 255, "access": "rw" }
  ]
}
```

This is the mechanical reason "the device node is there but I get permission denied" is a distinct failure from "the device node is missing": the first is a cgroup rule, the second is a `mknod`.

**`mounts`** — the bind mounts that carry host driver libraries in. NVIDIA emits them with `ro,nosuid,nodev,rbind,rprivate`; `rbind` so any submounts come along, `rprivate` so later host mount changes do not propagate into the container.

**`hooks`** — executables the runtime runs at defined points. The OCI spec defines five phases, and the two properties that matter are *when* and *in which namespace*:

| Phase | When | Runs in | Notes |
|---|---|---|---|
| `prestart` | during `create`, after the runtime environment exists, before `pivot_root` | **runtime** namespace | **Deprecated** by the OCI spec in favour of the three phases below |
| `createRuntime` | same point as `prestart`, but after it | **runtime** namespace | Mount namespace exists and mounts are done |
| `createContainer` | during `create`, after `createRuntime`, before `pivot_root` | **container** namespace | Sees the container's mount namespace directly |
| `startContainer` | during `start`, immediately before the user process is exec'd | **container** namespace | Path must resolve *inside* the container |
| `poststart` / `poststop` | after start / after delete | runtime namespace | Not used by this stack |

**Hold on to the runtime-vs-container namespace column.** It is the single fact that explains the shape of both injection designs. A hook that runs in the *runtime* namespace and wants to change the container's filesystem has to find the container's mount namespace and enter it — which means it needs the container's PID, elevated privilege, and a namespace-entering implementation. A hook that runs in the *container* namespace is already there and can just write files. The legacy path is the first kind. CDI's hooks are the second kind.

### 3. Two environment variables decide everything

Both injection paths read the same two variables out of the container's `process.env` in `config.json`. They are normally set by the device plugin's `Allocate` response, but they can also come from the image's `ENV` lines, or from a hand-written pod spec — which is where the security problem lives.

**`NVIDIA_VISIBLE_DEVICES` — which GPUs.** Parsing is exact (`internal/config/image/devices.go`):

| Value | Meaning |
|---|---|
| `all` | every GPU on the node |
| `none` | no device nodes, but driver libraries still injected — a "GPU-capable but GPU-less" container |
| `void` or empty or unset | **not a GPU container at all**; the toolkit makes zero modifications |
| `0`, `0,1` | GPU indices as reported by `nvidia-smi` |
| `GPU-<uuid>` | a full GPU by UUID — what `Allocate` normally sets, with `DEVICE_ID_STRATEGY=uuid` (the default) |
| `MIG-<uuid>` | a MIG device by UUID (lesson 04.6) |
| `nvidia.com/gpu=0`, `nvidia.com/gpu=GPU-<uuid>` | a *fully-qualified CDI device name*; see §5 |

Two behaviours worth internalising. First, `none` and `void` are not synonyms: `none` still mounts `libcuda`, `void` does nothing at all. Second, there is a legacy-image special case: if the image declares `CUDA_VERSION` in its environment but sets no `NVIDIA_VISIBLE_DEVICES`, the toolkit treats it as a pre-2018-style CUDA image and defaults to `all`. That is a compatibility shim, not something to rely on.

**`NVIDIA_DRIVER_CAPABILITIES` — which driver libraries.** The toolkit's supported set is exactly seven values, and you can read the list off the config file it writes: `supported-driver-capabilities = "compat32,compute,display,graphics,ngx,utility,video"`.

| Capability | What it injects | Needed for |
|---|---|---|
| `compute` | `libcuda.so.*`, `libnvidia-ptxjitcompiler.so.*`, `libnvidia-nvvm*.so.*`, `nvidia.icd` | CUDA and OpenCL — anything that calls the driver API |
| `utility` | `libnvidia-ml.so.*` (NVML), `nvidia-smi`, `nvidia-debugdump` | monitoring, `nvidia-smi`, DCGM |
| `video` | `libnvcuvid.so.*`, `libnvidia-encode.so.*` | NVDEC/NVENC — transcoding, video inference |
| `graphics` | GL/EGL/GLX/Vulkan libraries and ICD/JSON config files | OpenGL, Vulkan, rendering |
| `display` | X11/modeset-related libraries, `/dev/nvidia-modeset` | attached displays |
| `compat32` | the 32-bit counterparts of the above | 32-bit binaries |
| `ngx` | `libnvidia-ngx.so.*` | NGX / DLSS-class features |
| `all` | resolves to the full supported set above | convenience |

**The default when the variable is unset is `utility,compute`** — not `utility` alone. This is a correction worth making explicitly, because "unset defaults to `utility`, so CUDA silently breaks" is one of the most widely repeated pieces of stale folklore about this stack. It was true of very early releases; NVIDIA Container Toolkit **v1.4.0** shipped the changelog entry *"Add 'compute' capability to list of defaults"*, and the current source still reads `DefaultDriverCapabilities = NewDriverCapabilities("utility,compute")`. If a hand-rolled image cannot find `libcuda`, the cause is almost certainly something else in this lesson — a `void` device list, a runtime that is not the NVIDIA runtime, a stale ldcache — not the capability default.

Two more mechanical details on capabilities. The requested set is **intersected** with `supported-driver-capabilities`, and if you request something outside the supported set the hook *panics* with `unsupported capabilities found in '<yours>' (allowed '<supported>')` — a loud failure, not a silent drop. And the same legacy-image shim applies: an old-style CUDA image with no explicit capabilities gets the *full* supported set rather than the default two.

**The security consequence, stated precisely.** `NVIDIA_VISIBLE_DEVICES` is read from the container's environment, and a pod spec can set environment variables. Left unguarded, any pod could set `NVIDIA_VISIBLE_DEVICES=all` and reach every GPU on the node regardless of what the scheduler allocated. NVIDIA Container Toolkit **v1.4.1** shipped the fix — changelog entry *"Ignore `NVIDIA_VISIBLE_DEVICES` for containers with insufficient privileges"* — surfaced as the config knob `accept-nvidia-visible-devices-envvar-when-unprivileged`. Its default is `true` for standalone Docker compatibility; the GPU Operator sets it to `false` on Kubernetes nodes, which is why a pod that sets the variable by hand there gets nothing rather than everything.

### 4. The legacy path: an imperative prestart hook

This is the design that shipped with `nvidia-docker2` and still exists as `mode = "legacy"`.

You install a wrapper OCI runtime, `nvidia-container-runtime`, and point containerd at it. On `create`, the wrapper reads `config.json` from stdin, appends one hook, writes it back, and `exec`s the real `runc`. The insertion is about ten lines, and reading them removes all the mystery (`internal/modifier/stable.go`):

```go
spec.Hooks.Prestart = append(spec.Hooks.Prestart, specs.Hook{
    Path: path,                              // /usr/bin/nvidia-container-runtime-hook
    Args: append(args, "prestart"),          // ["nvidia-container-runtime-hook", "prestart"]
})
```

It is idempotent: if an NVIDIA prestart hook is already present, the modifier returns without touching the spec.

`runc` then runs that hook during `create`, in the **runtime** namespace, with the container's state on stdin. The hook re-reads the environment out of the spec, resolves the two variables from §3, and shells out to `nvidia-container-cli configure` (the CLI front-end of `libnvidia-container`) with a `--pid=<container pid>` and a list of devices and capabilities. `nvidia-container-cli` is the component that does the real work, and because it started in the runtime namespace it must:

1. `setns()` into the target PID's mount namespace,
2. `mknod` the required device nodes,
3. bind-mount each host driver library and binary to the same path inside the container,
4. create the SONAME symlinks (`libcuda.so.1 → libcuda.so.580.65.06`),
5. run `ldconfig` inside the container to rebuild `/etc/ld.so.cache`,
6. and — separately, back outside — ensure the device cgroup permits the majors/minors it just created.

Two consequences follow directly from that list, and they are the reason the ecosystem moved on.

**It is invisible.** Nothing in `config.json` describes the mounts or the devices. `crictl inspect` shows one hook path. To know what a container received you have to run the hook's logic in your head, or exec in and look.

**It is coupled to a special runtime and to root.** Every runtime — Docker, containerd, CRI-O, Podman — needed its own wiring to get an NVIDIA-specific `runc` wrapper in the path, and the namespace-entering step requires privilege that a rootless container runtime does not have.

There is also a deprecation problem: `prestart` is the one hook phase the OCI specification marks **DEPRECATED**, in favour of `createRuntime`/`createContainer`/`startContainer`. Building a vendor integration on a deprecated phase is a slow-moving liability.

### 5. The modern path: CDI

The Container Device Interface inverts the design. Instead of *code that mutates a container*, you write *a document that describes the mutation*, and any CDI-aware runtime applies it using stock `runc`. CDI is a CNCF-tagged, vendor-neutral specification (`cncf-tags/container-device-interface`), currently at **spec version v1.1.0**, and NVIDIA is one consumer among several — Intel and AMD publish CDI specs for their devices too.

#### 5.1 The specification, field by field

A CDI file is JSON or YAML with four top-level keys. Required unless marked optional.

| Field | Type | Meaning |
|---|---|---|
| `cdiVersion` | string, **required** | semver of the CDI spec this file uses. Loading fails if the file uses a field newer than the version it declares. |
| `kind` | string, **required** | `<prefix>/<name>`, e.g. `nvidia.com/gpu`. Prefix is a DNS subdomain ≤253 chars; name ≤63 chars, alphanumeric ends, `-`/`_`/`.` inside. Exactly one slash: `vendor.com/foo/bar` is invalid. |
| `devices` | array, **required**, ≥1 entry | the named devices this file provides |
| `containerEdits` | object, optional | edits applied if **any** device of this kind is requested |
| `annotations` | map, optional | free-form metadata; added in spec v0.6.0 |

Each entry in `devices` has a `name` (same character rules; may start with a digit since v0.5.0), an optional `annotations` map, and its own `containerEdits` — **applied only when that specific device is requested.** That two-level structure is the whole design: per-device edits carry the device node for GPU 3; kind-level edits carry the things every GPU container needs (`/dev/nvidiactl`, the driver libraries, the ldcache hook).

`containerEdits` is where the OCI mapping lives:

| Field | Shape | Maps to |
|---|---|---|
| `env` | `["NAME=VALUE", ...]` | appended to `process.env` |
| `deviceNodes` | `[{path, hostPath?, type?, major?, minor?, fileMode?, permissions?, uid?, gid?}]` | `linux.devices` **and** the matching `linux.resources.devices` allow rule. `hostPath` defaults to `path` (v0.5.0+). `permissions` is the cgroup access string from `r`/`w`/`m`; omitted means `rwm`, and the literal `none` requests empty permissions. |
| `mounts` | `[{hostPath, containerPath, type?, options?}]` | `mounts`. `type` added in v0.4.0; for bind mounts it is conventionally `none`. |
| `hooks` | `[{hookName, path, args?, env?, timeout?}]` | `hooks.<phase>`. `hookName` is one of the OCI phases. `path` must be absolute. `timeout`, if set, must be > 0; unset means wait forever. |
| `additionalGIDs` | `[uint32]` | `process.user.additionalGids`; zeros ignored. Added v0.7.0. |
| `intelRdt` | object | Linux `resctrl` settings. Added v0.7.0. |
| `netDevices` | `[{hostInterfaceName, name}]` | moves a host network interface into the container. Added v1.1.0. |

The spec also defines error handling, which matters for diagnosis: a runtime **must** surface an error when an unknown `kind` is requested, when a named device does not exist in any spec, and when a hook fails to execute. A missing *resource* (a `hostPath` that does not exist) is also an error, but only at container-create time — the spec deliberately allows a file to reference paths that do not exist yet when it is written.

**Spec directories and precedence.** CDI-aware runtimes scan two directories, in this order: `/etc/cdi` for persistent, admin-managed specs, and `/var/run/cdi` for transient specs generated at runtime and lost on reboot. Both NVIDIA's toolkit (`spec-dirs = ["/etc/cdi", "/var/run/cdi"]`) and containerd (`cdi_spec_dirs = ["/etc/cdi", "/var/run/cdi"]`) use exactly those defaults.

**Device names are fully qualified**: `<kind>=<device name>`, e.g. `nvidia.com/gpu=0`, `nvidia.com/gpu=GPU-a1b2c3d4-...`, `nvidia.com/gpu=all`. NVIDIA generates device names under two strategies by default — `index` and `uuid` — so both forms resolve; `type-index` (`gpu0`, `mig0:1`) is also available via `--device-name-strategy`.

#### 5.2 A real generated NVIDIA spec, read line by line

`nvidia-ctk cdi generate` walks the driver installation via NVML and writes a spec. The structure below is the toolkit's own golden output from its generator tests, with a realistic driver version substituted for the test placeholder:

```yaml
cdiVersion: 0.5.0
kind: nvidia.com/gpu
devices:
    - name: "0"                                  # per-device edits: only for GPU 0
      containerEdits:
        deviceNodes:
            - path: /dev/nvidia0
              hostPath: /dev/nvidia0
    - name: all                                  # the "all" pseudo-device
      containerEdits:
        deviceNodes:
            - path: /dev/nvidia0
              hostPath: /dev/nvidia0
containerEdits:                                  # applied for ANY nvidia.com/gpu device
    env:
        - NVIDIA_CTK_LIBCUDA_DIR=/lib/x86_64-linux-gnu
        - NVIDIA_VISIBLE_DEVICES=void
    deviceNodes:
        - path: /dev/nvidiactl
          hostPath: /dev/nvidiactl
    hooks:
        - hookName: createContainer
          path: /usr/bin/nvidia-cdi-hook
          args: ["nvidia-cdi-hook", "create-symlinks",
                 "--link", "libcuda.so.1::/lib/x86_64-linux-gnu/libcuda.so"]
          env: ["NVIDIA_CTK_DEBUG=false"]
        - hookName: createContainer
          path: /usr/bin/nvidia-cdi-hook
          args: ["nvidia-cdi-hook", "enable-cuda-compat",
                 "--host-driver-version=580.65.06"]
          env: ["NVIDIA_CTK_DEBUG=false"]
        - hookName: createContainer
          path: /usr/bin/nvidia-cdi-hook
          args: ["nvidia-cdi-hook", "update-ldcache",
                 "--folder", "/lib/x86_64-linux-gnu",
                 "--folder", "/lib/x86_64-linux-gnu/vdpau"]
          env: ["NVIDIA_CTK_DEBUG=false"]
        - hookName: createContainer
          path: /usr/bin/nvidia-cdi-hook
          args: ["nvidia-cdi-hook", "disable-device-node-modification"]
          env: ["NVIDIA_CTK_DEBUG=false"]
        - hookName: createContainer
          path: /usr/bin/nvidia-cdi-hook
          args: ["nvidia-cdi-hook", "update-application-profile"]
          env: ["NVIDIA_CTK_DEBUG=false"]
    mounts:
        - hostPath: /lib/x86_64-linux-gnu/libcuda.so.580.65.06
          containerPath: /lib/x86_64-linux-gnu/libcuda.so.580.65.06
          options: [ro, nosuid, nodev, rbind, rprivate]
        - hostPath: /lib/x86_64-linux-gnu/vdpau/libvdpau_nvidia.so.580.65.06
          containerPath: /lib/x86_64-linux-gnu/vdpau/libvdpau_nvidia.so.580.65.06
          options: [ro, nosuid, nodev, rbind, rprivate]
```

Read the non-obvious lines:

- **`cdiVersion: 0.5.0`, not `1.1.0`.** The generator declares the *minimum required version* implied by the fields it actually used — `hostPath` on `deviceNodes` landed in v0.5.0, and nothing here needs anything newer. So a low `cdiVersion` on a freshly generated spec is correct behaviour, not staleness. The library exposes this as `MinimumRequiredVersion`.
- **`NVIDIA_VISIBLE_DEVICES=void`** is injected *into the container*. That is a deliberate loop-breaker: if something inside this container later runs its own container runtime, it sees `void` and does not try to re-inject GPUs.
- **`NVIDIA_CTK_LIBCUDA_DIR`** tells later hooks where the injected `libcuda` landed, so they do not have to rediscover it.
- **`create-symlinks --link libcuda.so.1::/lib/.../libcuda.so`** creates the SONAME link. The `A::B` syntax is *target::linkpath*. Without this, `dlopen("libcuda.so.1")` fails even though the versioned file is mounted.
- **`update-ldcache --folder ...`** runs `ldconfig` over the named directories inside the container. This is the step that makes the bind mounts visible to the dynamic linker. It is only emitted if libraries were actually discovered (a v1.20.0 change: *"Only generate update-ldcache hook if libraries are discovered"*).
- **`enable-cuda-compat --host-driver-version=...`** is CUDA forward compatibility, covered in §8.
- **`disable-device-node-modification`** blocks the container from creating *new* NVIDIA device nodes itself — a containment measure, because a privileged container that can `mknod` its own `/dev/nvidiaN` could reach GPUs it was not given.
- **Every hook is `createContainer`.** They execute *inside the container's namespace*, which is why they can write `/etc/ld.so.conf.d/` and create symlinks with no `setns` dance and no container PID. This is the concrete payoff of the OCI phase table in §2.

#### 5.3 Runtime modes, and why there may be no spec on disk

`nvidia-container-runtime` supports five values for `mode` (`internal/info/auto.go`):

| Mode | Behaviour |
|---|---|
| `legacy` | inject the `nvidia-container-runtime-hook` prestart hook (§4) |
| `cdi` | resolve the requested fully-qualified CDI devices against the spec directories and apply their edits |
| `jit-cdi` | **generate a CDI spec in memory** for the requested NVIDIA devices and apply it; nothing is read from or written to disk |
| `csv` | build an in-memory CDI spec from CSV mount lists in `/etc/nvidia-container-runtime/host-files-for-container.d` — the Tegra/Jetson path, where the "driver" is a set of files rather than a discoverable NVML installation |
| `auto` | **the default**; resolve at container-create time |

The resolution rule for `auto` is short enough to quote in full:

1. If **every** requested device is a fully-qualified CDI name (`nvidia.com/gpu=...`), use `cdi`.
2. Otherwise detect the platform: `Tegra` → `csv`; `NVML` or `WSL` → the default mode.
3. The default mode is **`jit-cdi`**.

So on an ordinary datacenter GPU node with a device plugin using the default `DEVICE_LIST_STRATEGY=envvar`, `NVIDIA_VISIBLE_DEVICES=GPU-<uuid>` is not a fully-qualified CDI name, the platform is NVML, and you land in `jit-cdi`. The runtime constructs the spec at kind `runtime.nvidia.com/gpu` on the fly and applies it. **`/etc/cdi` and `/var/run/cdi` are empty, `nvidia-ctk cdi list` prints nothing, and CDI is nonetheless exactly what is happening.** Correspondingly, `enable_cdi` in containerd is irrelevant on this path: containerd is not resolving CDI devices, the NVIDIA runtime is.

That is the sentence to carry into an incident: **"is CDI active?" is the wrong question. "Which mode did the runtime resolve to, and who applied the edits?" is the right one** — and the answer is in the runtime's own log, not on the filesystem.

#### 5.4 The container start path, both ways

```
  THE CONTAINER START PATH — legacy prestart hook vs CDI, side by side

  kubelet                                   kubelet
    │ CreateContainerRequest (CRI)            │ CreateContainerRequest (CRI)
    │  envs: NVIDIA_VISIBLE_DEVICES=GPU-abc   │  envs / CDIDevices / annotations
    ▼                                         ▼
  containerd  (io.containerd.cri.v1.runtime)containerd
    │ picks RuntimeHandler "nvidia"           │ enable_cdi=true → resolves
    │ from RuntimeClass                       │ cdi.k8s.io/* annotations or
    │                                         │ CRI CDIDevices field itself
    ▼                                         ▼
  builds OCI bundle: config.json + rootfs   builds OCI bundle, and MERGES the
    │                                         CDI containerEdits INTO config.json
    ▼                                         │  (linux.devices, cgroup allow,
  /usr/bin/nvidia-container-runtime           │   mounts, env, createContainer hooks)
    │  mode=legacy                            ▼
    │  appends hooks.prestart:              /usr/bin/runc      ← STOCK runc
    │    nvidia-container-runtime-hook        │
    ▼                                         │ mknod  linux.devices
  /usr/bin/runc                               │ mount  mounts[]  (rbind,ro)
    │ create: runtime env ready               │ cgroup linux.resources.devices
    │                                         ▼
    ├─▶ hooks.prestart  ── RUNTIME NS ──┐   hooks.createContainer ─ CONTAINER NS ─┐
    │     nvidia-container-runtime-hook  │     nvidia-cdi-hook create-symlinks     │
    │       └─ nvidia-container-cli      │     nvidia-cdi-hook enable-cuda-compat  │
    │            configure --pid=<pid>   │     nvidia-cdi-hook update-ldcache      │
    │              setns(mnt) ───────────┤       (already inside the container's   │
    │              mknod /dev/nvidia*    │        mount namespace — just writes)   │
    │              bind libcuda.so.*     │                                        │
    │              symlink libcuda.so.1  │                                        │
    │              ldconfig              │                                        │
    ▼                                    │                                        │
  pivot_root; exec app                   │   pivot_root; exec app                 │
                                         │                                        │
   ═══════════════ WHERE FILES ENTER THE MOUNT NAMESPACE ══════════════════════════
   LEGACY: inside the hook, invisible     CDI: in runc, from config.json, auditable
           to config.json                     ── plus 3 small in-namespace hooks

   jit-cdi (today's default): same right-hand column, except the CDI spec is
   generated IN MEMORY by nvidia-container-runtime at kind runtime.nvidia.com/gpu
   and merged by it — containerd's enable_cdi is not involved at all.
```

Three things to take from the drawing. Device nodes and libraries enter the container's mount namespace **inside `runc`, driven by `config.json`** on the CDI path, versus **inside a hook process that entered the namespace itself** on the legacy path. The CDI path needs no NVIDIA-specific OCI runtime at all when the caller passes fully-qualified device names. And the CDI hooks that remain are small, single-purpose, and run where they can see what they are changing.

### 6. Wiring it up: the real configuration files

#### 6.1 `/etc/nvidia-container-runtime/config.toml`

This is the toolkit's own config, and this is what a default install writes (commented lines are defaults shown for discoverability; the file is generated, so this is the literal shape):

```toml
#accept-nvidia-visible-devices-as-volume-mounts = false
#accept-nvidia-visible-devices-envvar-when-unprivileged = true
disable-require = false
supported-driver-capabilities = "compat32,compute,display,graphics,ngx,utility,video"
#swarm-resource = "DOCKER_RESOURCE_GPU"

[nvidia-container-cli]
#debug = "/var/log/nvidia-container-toolkit.log"
environment = []
#ldcache = "/etc/ld.so.cache"
ldconfig = "@/sbin/ldconfig"
load-kmods = true
#no-cgroups = false
#path = "/usr/bin/nvidia-container-cli"
#root = "/run/nvidia/driver"
#user = "root:video"

[nvidia-container-runtime]
#debug = "/var/log/nvidia-container-runtime.log"
log-level = "info"
mode = "auto"
runtimes = ["runc", "crun"]

[nvidia-container-runtime.modes]

[nvidia-container-runtime.modes.cdi]
annotation-prefixes = ["cdi.k8s.io/"]
default-kind = "nvidia.com/gpu"
spec-dirs = ["/etc/cdi", "/var/run/cdi"]

[nvidia-container-runtime.modes.csv]
mount-spec-path = "/etc/nvidia-container-runtime/host-files-for-container.d"

[nvidia-container-runtime.modes.legacy]
cuda-compat-mode = "ldconfig"

[nvidia-container-runtime-hook]
path = "nvidia-container-runtime-hook"
skip-mode-detection = false

[nvidia-ctk]
path = "nvidia-ctk"
```

The load-bearing lines:

- **`mode = "auto"`** — §5.3. This is the one line that decides which mechanism you are debugging.
- **`runtimes = ["runc", "crun"]`** — the low-level runtime the wrapper `exec`s, tried in order.
- **`ldconfig = "@/sbin/ldconfig"`** — the leading `@` means "resolve this path in the *host* root, not the container". Running the container's own `ldconfig` would be running an untrusted binary as root; the `@` form is the hardened default, and there is a feature gate (`allow-ldconfig-from-container`) for the cases that genuinely need the other behaviour.
- **`root = "/run/nvidia/driver"`** (commented, but set by the GPU Operator) — the *driver root*. When the driver ships in a container, the host's `/` does not have `libcuda.so`; the driver container bind-mounts its own rootfs at `/run/nvidia/driver` and everything is discovered relative to that. Get this wrong and you get "driver libraries not found" on a node whose GPUs are perfectly healthy.
- **`load-kmods = true`** — the CLI will `modprobe` `nvidia_uvm` etc. if they are not yet loaded.
- **`skip-mode-detection = false`** — with detection on, invoking `nvidia-container-runtime-hook` directly when the resolved mode is not `legacy` fails loudly: *"invoking the NVIDIA Container Runtime Hook directly (e.g. specifying the docker --gpus flag) is not supported. Please use the NVIDIA Container Runtime (e.g. specify the --runtime=nvidia flag) instead"*. If you see that string, someone has mixed the two paths.

#### 6.2 containerd

`nvidia-ctk runtime configure --runtime=containerd` edits `/etc/containerd/config.toml`, or writes a drop-in at `/etc/containerd/conf.d/99-nvidia.toml` referenced from the top-level file by `imports`. It registers **three** runtime handlers, not one:

```toml
version = 2

[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    enable_cdi = true

    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "nvidia"

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
            BinaryName = "/usr/local/nvidia/toolkit/nvidia-container-runtime"

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia-cdi]
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia-cdi.options]
            BinaryName = "/usr/local/nvidia/toolkit/nvidia-container-runtime.cdi"

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia-legacy]
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia-legacy.options]
            BinaryName = "/usr/local/nvidia/toolkit/nvidia-container-runtime.legacy"
```

Field by field:

- **`runtime_type = "io.containerd.runc.v2"`** is the *shim* type. It is unchanged — the NVIDIA integration is not a new shim, it is a different `runc` binary.
- **`options.BinaryName`** is the actual substitution point. `nvidia-container-runtime` (mode from config), `nvidia-container-runtime.cdi` (pinned to `cdi`), `nvidia-container-runtime.legacy` (pinned to `legacy`). Those three handler names map to three `RuntimeClass` objects, which is how you can run one pod through the modern path and one through the legacy path on the same node — a genuinely useful A/B when you are diagnosing whether the injection mechanism is the problem.
- **`privileged_without_host_devices = false`** means a privileged pod also gets the host's `/dev` in the usual containerd way. Setting it `true` is what lets you have privileged GPU pods that only see their allocated devices.
- **`default_runtime_name = "nvidia"`** is set only if you asked for `--set-as-default`. Without it, pods must name a `RuntimeClass`.
- **`enable_cdi`** belongs to the CRI plugin section, *not* to a runtime. It controls whether containerd itself resolves CDI devices from annotations and the CRI `CDIDevices` field.

**Version differences that bite.** Config `version = 3` arrived in containerd v2.0 and renames the plugin from `io.containerd.grpc.v1.cri` to `io.containerd.cri.v1.runtime`; v2 configs are still accepted and converted. And the default changed: containerd 1.7 ships `enable_cdi = false`, containerd 2.x ships `enable_cdi = true` with `cdi_spec_dirs = ['/etc/cdi', '/var/run/cdi']`. A hand-written `enable_cdi = true` under the wrong plugin name for your config version is silently ignored — which produces the exact symptom of not having set it at all.

#### 6.3 CRI-O, Docker, Podman

CRI-O's stanza is smaller because CDI needs no enabling there:

```toml
[crio]
  [crio.runtime]
    default_runtime = "nvidia"
  [crio.runtime.runtimes.nvidia]
    runtime_path = "/usr/bin/nvidia-container-runtime"
    runtime_type = "oci"
```

written to `/etc/crio/crio.conf` or the drop-in `/etc/crio/crio.conf.d/99-nvidia.toml`. **CRI-O has CDI support on by default**, with the same two spec directories; `crio config | grep -A5 cdi_spec_dirs` prints what it is actually using. The toolkit's CRI-O `EnableCDI()` implementation is literally a no-op, with the comment "CRI-O since it always enabled where supported."

The full support matrix, from the CDI project's own configuration guide:

| Runtime | CDI status | What to do |
|---|---|---|
| CRI-O | **on by default** | nothing; drop specs in `/etc/cdi` or `/var/run/cdi` |
| containerd 2.x | **on by default** (`enable_cdi = true`) | verify the plugin key matches your config version |
| containerd 1.7.x | off by default | set `enable_cdi = true` under `plugins."io.containerd.grpc.v1.cri"`, restart containerd |
| Docker ≥ 28.2.0 | **on by default** | pass fully-qualified names to `--device` |
| Docker 25.0.0 – 28.1.1 | opt-in | `{"features": {"cdi": true}}` in `/etc/docker/daemon.json` |
| Docker < 25.0.0 | unsupported | legacy path only |
| Podman ≥ 4.1.0 | supported, no config needed | pass fully-qualified names to `--device` |

#### 6.4 How the device plugin tells the runtime which devices

The device plugin has four ways to express the allocation, selectable with `DEVICE_LIST_STRATEGY` (comma-separated; multiple allowed):

| Strategy | Mechanism | Requires |
|---|---|---|
| `envvar` (default) | sets `NVIDIA_VISIBLE_DEVICES` in the container env | the NVIDIA container runtime in the path |
| `volume-mounts` | encodes the device list as a set of no-op volume mounts under `/var/run/nvidia-container-devices/` | the NVIDIA container runtime; exists because env vars are attacker-settable and mounts are not |
| `cdi-annotations` | sets pod annotations `cdi.k8s.io/nvidia-device-plugin_<claim>: nvidia.com/gpu=<device>` | a CDI-enabled engine; **no** NVIDIA runtime needed |
| `cdi-cri` | populates the CRI `CDIDevices` field directly | CRI + engine support for forwarding it |

`volume-mounts` is the interesting one historically: it exists precisely because `NVIDIA_VISIBLE_DEVICES` is forgeable from a pod spec while a volume mount injected by `Allocate` is not. `cdi-annotations` is the interesting one going forward: it is the strategy under which you can delete the NVIDIA OCI runtime from the node entirely and still get GPUs, because containerd or CRI-O does the injection.

### 7. What actually lands inside the container

Here is the inventory, with provenance. This is the mental model that turns an `ls` into a diagnosis.

```
  A HEALTHY compute,utility GPU CONTAINER — WHAT IS THERE AND WHERE IT CAME FROM

  ┌──────────────────────── CONTAINER MOUNT NAMESPACE ────────────────────────┐
  │                                                                            │
  │  /dev/nvidia0            c 195,  0   ◀── mknod by runc from linux.devices  │
  │  /dev/nvidiactl          c 195,255   ◀──   (+ matching cgroup allow rule)  │
  │  /dev/nvidia-uvm         c <dyn>, 0  ◀──   major from /proc/devices        │
  │  /dev/nvidia-uvm-tools   c <dyn>, 1  ◀──   (nvidia-uvm is dynamic)         │
  │  /dev/nvidia-modeset     c 195,254   ◀── only with `display`/`graphics`    │
  │  /dev/nvidia-caps/                   ◀── MIG only; see lesson 04.6         │
  │      nvidia-cap<gi-minor>                 GI access cap                    │
  │      nvidia-cap<ci-minor>                 CI access cap                    │
  │                                                                            │
  │  /usr/lib/x86_64-linux-gnu/                                                │
  │      libcuda.so.580.65.06        ◀── BIND MOUNT, ro,nosuid,nodev, from HOST│
  │      libcuda.so.1  ─────────────────▶ symlink, made by create-symlinks hook│
  │      libnvidia-ml.so.580.65.06   ◀── HOST  (capability: utility)           │
  │      libnvidia-ptxjitcompiler.so.580.65.06  ◀── HOST (compute)             │
  │      libnvcuvid.so.580.65.06     ◀── HOST  (video)                         │
  │  /usr/bin/nvidia-smi             ◀── HOST  (utility)                       │
  │  /etc/ld.so.cache                ◀── REBUILT in-namespace by update-ldcache│
  │  /etc/ld.so.conf.d/00-compat-*.conf ◀── written by enable-cuda-compat (§8) │
  │                                                                            │
  │  ── everything below is FROM THE IMAGE, never injected ──                  │
  │  /usr/local/cuda/lib64/libcudart.so.13.0   CUDA runtime API                │
  │  /usr/local/cuda/bin/nvcc                  compiler                        │
  │  /usr/lib/.../libcudnn.so.9                cuDNN                           │
  │  /usr/lib/.../libnccl.so.2                 NCCL                            │
  │  /usr/local/cuda/compat/libcuda.so.590.44.01  forward-compat lib (§8)      │
  │                                                                            │
  │  process.env:  NVIDIA_VISIBLE_DEVICES=void   ◀── rewritten on the way in   │
  │                NVIDIA_DRIVER_CAPABILITIES=compute,utility                  │
  │                NVIDIA_CTK_LIBCUDA_DIR=/usr/lib/x86_64-linux-gnu           │
  └────────────────────────────────────────────────────────────────────────────┘

  THE SEAM:  everything above the dashed line is versioned by the HOST DRIVER.
             everything below it is versioned by the IMAGE.
             §8 is what happens when the two disagree.
```

Notes on the device nodes, because the numbers are checkable and people get them wrong:

- The major for the GPU control/render devices is read from `/proc/devices`, conventionally **195**. Do not hard-code it: for **driver 550.40.x and later the entry in `/proc/devices` was renamed from `nvidia-frontend` to `nvidia`**, and the toolkit carries fallback logic for exactly that (`internal/info/proc/devices/devices.go`). If you are parsing `/proc/devices` in your own tooling, handle both names.
- Minors are fixed by convention and are constants in the toolkit: `nvidiactl` = **255**, `nvidia-modeset` = **254**, and GPU *N* gets minor *N*.
- `nvidia-uvm` has a **dynamically allocated** major (it is a separate module), with `nvidia-uvm` at minor 0 and `nvidia-uvm-tools` at minor 1. Read the major from `/proc/devices`, never assume it.
- `nvidia-ctk system create-dev-char-symlinks` populates `/dev/char/<major>:<minor>` symlinks. Some runtimes and some cgroup-v2 setups resolve devices through `/dev/char`, and their absence is a real (and confusing) failure mode on minimal node images.

### 8. The version-skew failure family, mechanically

This is the family the module's checkpoint asks about, so learn it as mechanisms rather than as three error strings.

#### 8.1 `Failed to initialize NVML: Driver/library version mismatch`

`libnvidia-ml.so` and `libcuda.so` talk to the kernel module over an `ioctl` interface that carries a version handshake. The libraries and the module are built together; the handshake rejects a mismatch outright. So this error means: **the userspace half in this container is not the same build as the kernel module currently loaded on the host.**

Three ways to get there, all common:

1. A host package upgrade replaced `/usr/lib/.../libcuda.so.*` while the old `nvidia.ko` was still resident (nothing unloads a module in use). Any *new* container gets the new library and the old module.
2. In the GPU Operator model, the driver container was replaced but the previous driver's rootfs is still bind-mounted at `/run/nvidia/driver`, so the toolkit discovers the *old* libraries against the *new* module — or the reverse. The `k8s-driver-manager` explicitly unmounts that stale rootfs during a rotation for this reason.
3. A CDI spec in `/etc/cdi/nvidia.yaml` was generated against driver 580.65.06 and the host has since moved to 595.71.05. The spec still names `libcuda.so.580.65.06`, which no longer exists. Depending on timing you get either a create-time failure on the missing `hostPath` or a mismatch at runtime. **This is the CDI-specific staleness failure, and it is why the toolkit ships a `nvidia-cdi-refresh` systemd unit to regenerate specs when the driver changes** — and why `jit-cdi`, which generates the spec per container, does not have this failure mode at all.

The tell that distinguishes this from everything else: `nvidia-smi` *inside the container* fails too, with the NVML error, and it names two versions.

#### 8.2 `CUDA driver version is insufficient for CUDA runtime version` (CUDA error 35)

Different mechanism entirely. Here the handshake succeeded — the userspace driver and the kernel module agree — but the **CUDA runtime from the image requires a newer driver API than the host driver provides.** `libcudart.so` from CUDA 13.x asks `libcuda.so` for entry points that an R525 driver never had.

The distinguishing symptom is the one that wastes the most on-call time: **`nvidia-smi` works perfectly and only your framework claims there is no GPU.** `nvidia-smi` talks to NVML directly and cares nothing about the CUDA runtime in your image. `torch.cuda.is_available()` returns `False` with no explanation.

The diagnosis is a two-number comparison, both readable from inside the container:

```
  driver's maximum CUDA:   nvidia-smi | head -3      → "CUDA Version: 13.0"
  image's CUDA runtime:    nvcc --version            → "release 13.1"
                           (or ls /usr/local/cuda*)
  if image > driver's max and the gap crosses a major family → error 35
```

The compatibility bands themselves (CUDA 11.x needs ≥ R450, 12.x needs ≥ R525, 13.x needs ≥ R580, and minor-version compatibility inside a family) are lesson 03.6's territory and are not re-derived here. What belongs *here* is that this is a **node-side** problem: the fix is on the host or in the compat libraries, never in rebuilding the image differently unless you are willing to downgrade CUDA.

#### 8.3 What CUDA forward compatibility actually buys — and what it does not

This is the escape hatch for a fleet frozen on an old driver, and it is almost always described vaguely. The mechanism is concrete.

The `cuda-compat-<major>-<minor>` package installs a **newer `libcuda.so.<version>` into `/usr/local/cuda/compat/`** inside the image. That library speaks the new driver API to your application and the *old* `ioctl` interface to the resident kernel module. The kernel module is untouched. So forward compatibility is: a newer userspace driver library, riding on an older kernel module, for one process tree.

Getting it onto the linker path is what the `enable-cuda-compat` CDI hook does, and its logic is the interesting part. The hook:

1. looks for `/usr/local/cuda/compat/libcuda.so.*.*` in the container;
2. reads that library's **ELF note section `.note.cuda.fwd_compatibility`**, which carries a small JSON blob;
3. decides from the blob whether the compat library may be used;
4. if yes, writes `/etc/ld.so.conf.d/00-compat-<uuid>.conf` containing the compat directory, so it is searched *ahead of* the injected host library.

You can read that blob yourself:

```bash
$ readelf -p .note.cuda.fwd_compatibility \
      /usr/local/cuda/compat/libcuda.so.590.44.01
# decoded payload:
#   Format:       1
#   CUDA Version: "13.1"
#   Driver:       [535, 550, 570, 575, 580, 590]
#   Device:       [1, 2, 7, 8, 9, 10, 11, 12, 13, 14]
```

**The `Driver` array is the answer to "what does forward compatibility buy me".** It is an explicit allowlist of host driver *branches* this compat library will run on. The hook's rule is: use the compat library only if the host driver's major branch appears in that list **and** the compat library's version is strictly greater than the host driver's. Note what the example excludes — 560 and 565 are absent, so an R565 host cannot forward-compat to this CUDA 13.1 library at all, even though 560 < 590. Forward compatibility is a matrix NVIDIA publishes per compat build, not a general "newer runs on older" guarantee.

Three further limits, all mechanical:

- **Datacenter GPUs only.** Attempting it on a consumer card yields CUDA error 804, `cudaErrorCompatNotSupportedOnDevice` — "forward compatibility was attempted on non supported HW". That error is diagnostic gold: it means the compat libraries *were* found and loaded and the hardware refused them. The fix is to remove the compat directory from the search path, not to add more packages.
- **It cannot teach an old kernel module about new silicon.** The `Device` array is an architecture allowlist; a compat library on an R535 host still cannot drive a GPU that R535 never supported, because the *module* is what talks to the hardware.
- **It does not fix an NVML mismatch.** §8.1 is a module/userspace build mismatch; forward compat deliberately pairs a *different* userspace with the module, which is a supported configuration only within the allowlist above.

If the ELF note is absent (older compat builds), the hook falls back to comparing major versions only, and if it cannot parse either version it declines to use the compat libraries — a fail-safe default. There is also a `cuda-compat-mode` setting on the legacy path (`ldconfig`, the default) and `--disable-hook=enable-cuda-compat` on the CDI path, for when you want the host library to win unconditionally.

#### 8.4 The diagnosis ladder

```
  "MY GPU POD DOESN'T SEE THE GPU" — WALK IT IN THIS ORDER
  each step eliminates exactly one layer; do not skip

  1. Did the pod get a device at all?
     kubectl get pod -o jsonpath='{.spec.containers[*].resources}'
     kubectl exec … -- env | grep NVIDIA
        NVIDIA_VISIBLE_DEVICES absent or =void
            ⇒ the toolkit was never asked to do anything.
              Wrong RuntimeClass, or device plugin's Allocate never ran,
              or DEVICE_LIST_STRATEGY is volume-mounts/cdi-* and you are
              looking for the wrong signal. STOP HERE. Go to lesson 04.3.
        NVIDIA_VISIBLE_DEVICES=void but device nodes ARE present
            ⇒ normal: injection happened, then rewrote the var to void.
                                       │
                                       ▼
  2. Are the device NODES present, and PERMITTED?
     kubectl exec … -- ls -l /dev/nvidia*
        missing            ⇒ no injection ran. Check that the container's
                             runtime handler is the nvidia one:
                             crictl inspect <id> | grep -i runtime
        present, EPERM on open
                           ⇒ device cgroup allow rule missing.
                             cat /sys/fs/cgroup/<path>/devices.allow  (v1)
                             or check `no-cgroups` isn't true in config.toml
                                       │
                                       ▼
  3. Is libcuda present AND LINKABLE?
     kubectl exec … -- sh -c 'ls -l /usr/lib/*/libcuda.so*; ldconfig -p | grep libcuda'
        no libcuda.so.<ver>      ⇒ `compute` capability not requested/injected
        libcuda.so.<ver> but no libcuda.so.1
                                 ⇒ create-symlinks hook did not run
        both present, ldconfig -p empty
                                 ⇒ update-ldcache hook did not run,
                                   or ldconfig was blocked (see `ldconfig = "@…"`)
                                       │
                                       ▼
  4. Does NVML initialise?
     kubectl exec … -- nvidia-smi
        "Driver/library version mismatch"  ⇒ §8.1  host-side. Stale libs,
                                             stale /run/nvidia/driver mount,
                                             or a stale /etc/cdi spec.
        prints the GPU normally            ⇒ host driver + module are fine.
                                       │
                                       ▼
  5. Does the CUDA RUNTIME initialise?
     kubectl exec … -- python -c "import torch; print(torch.cuda.is_available())"
        error 35 cudaErrorInsufficientDriver     ⇒ §8.2 image CUDA > driver max
        error 804 cudaErrorCompatNotSupported…   ⇒ §8.3 compat libs on wrong HW
        True                                     ⇒ the injection layer is clean;
                                                   your bug is above this lesson.
```

## Perspectives

**Container-runtime engineer.** The whole story is "imperative hook that enters a namespace" versus "declarative document merged into `config.json` before `runc` runs". The second is not merely tidier: it is *auditable*, it works with stock `runc`, it needs no vendor-specific OCI wrapper when the caller passes fully-qualified device names, and it survives rootless. It is also the thing that made the injection reusable — the same `containerEdits` grammar carries Intel and AMD devices, and `netDevices` (spec v1.1.0) carries network interfaces.

**On-call.** The single highest-value habit from this lesson is refusing to guess which path is active. Read the runtime handler from `crictl inspect`, read `mode` from `/etc/nvidia-container-runtime/config.toml`, and read the runtime's own log line — `Auto-detected mode as 'jit-cdi'` — rather than inferring from the presence or absence of files in `/etc/cdi`. The second habit is knowing that `nvidia-smi` working proves *nothing* about CUDA: it is an NVML client and it skips the entire half of the stack that fails in §8.2.

**Security.** Two real hardening decisions live in this lesson, both with changelog entries behind them. `accept-nvidia-visible-devices-envvar-when-unprivileged=false` exists because a pod-settable environment variable was a path to every GPU on the node. `ldconfig = "@/sbin/ldconfig"` and the `disable-device-node-modification` hook exist because "run a binary from the container image as root" and "let the container create its own device nodes" are both privilege-escalation shapes. When you write your own device plugins or admission policy, these are the two patterns to copy: never trust container-supplied identity for allocation, and never execute container-supplied code in the host's privilege domain.

**Standards / ecosystem.** CDI is the substrate Kubernetes device management is converging on. containerd's own config comments say the `enable_cdi` option is expected to be *deprecated* once Dynamic Resource Allocation or CDI-in-device-plugin-API reach GA — i.e. the future is CDI everywhere, not CDI as an option. Lesson 04.9's DRA driver produces CDI specs; lesson 04.6's MIG devices are CDI devices. The spec structure in §5.1 is a prerequisite you will meet again by name.

## Real-world use cases

- **The `cdi_spec_dirs`/`enable_cdi` defaults are a documented moving target.** containerd's own `docs/cri/config.md` is the primary source and it changed under practitioners' feet: the `release/1.7` branch documents `enable_cdi = false`, `main` documents `enable_cdi = true` with the plugin renamed to `io.containerd.cri.v1.runtime` under config `version = 3`. What it shows: a runbook that says "set `enable_cdi = true`" without naming the containerd version and the plugin key produces a config change that silently does nothing on half your fleet. Verified directly against both branches of the containerd repository.

- **NVIDIA Container Toolkit v1.4.0 and v1.4.1** — two changelog entries that are the origin of facts practitioners still get wrong. v1.4.0: *"Add 'compute' capability to list of defaults"* — the reason the "unset means `utility` only" folklore is wrong today. v1.4.1: *"Ignore `NVIDIA_VISIBLE_DEVICES` for containers with insufficient privileges"* — the reason the env-var escape is closed on any Operator-managed node. What it shows: the fastest way to date a claim about this stack is to find the changelog entry that introduced it. Read directly from the repository's `CHANGELOG.md`.

- **NVIDIA Container Toolkit v1.20.0's CDI-hook changes** — *"Only generate `update-ldcache` hook if libraries are discovered"*, *"Add `enable-cuda-compat` hook to management CDI specs"*, *"Add ability to disable CDI hooks in jit-cdi mode"*, *"feat: drop `nvidia-cdi-hook` shell shim"*. What it shows: the hook list in a generated spec is not fixed across toolkit versions, so "my spec has five hooks and yours has three" is expected rather than alarming — compare toolkit versions before comparing specs. Read directly from `CHANGELOG.md` in the toolkit repository.

- **`nvidia-container-cli: initialization error: nvml error: driver/library version mismatch: unknown`** — filed independently as [NVIDIA/nvidia-container-toolkit#394](https://github.com/NVIDIA/nvidia-container-toolkit/issues/394), [NVIDIA/gpu-operator#898](https://github.com/NVIDIA/gpu-operator/issues/898) and [NVIDIA/k8s-device-plugin#1431](https://github.com/nvidia/k8s-device-plugin/issues/1431). *(Titles and numbers found via search this session; issue bodies were not independently fetched because GitHub API access is restricted in this environment — treat the numbers as pointers, and the mechanism below as the verified part.)* What it shows: the failure surfaces at *container create* time in `nvidia-container-cli`, not inside the workload, because the toolkit itself calls NVML while discovering what to inject. That is the fingerprint of §8.1 rather than §8.2 — the container never starts at all, as opposed to starting and then failing in CUDA.

- **[NVIDIA/gpu-operator#1220 — "gpu-operator breaks when upgrading EKS to K8s v1.30"](https://github.com/NVIDIA/gpu-operator/issues/1220)**, with the log line `failed to get sandbox runtime: no runtime for 'nvidia' is configured`. *(Recorded in an earlier pass of this course; not independently re-fetched this session.)* What it shows: exactly the §6.2 failure. That message comes from containerd's CRI plugin, not from anything NVIDIA — the pod asked for the `nvidia` runtime handler and no such handler exists in the config containerd actually loaded. A managed-Kubernetes upgrade that replaces the node image can wipe `/etc/containerd/config.toml` and take the handler with it, which is precisely why the toolkit prefers a drop-in at `/etc/containerd/conf.d/99-nvidia.toml` over editing the top-level file.

## Worked example

A GPU pod on a node you did not provision is failing. Establish which mechanism is active, prove what the container received, and trace one file back to its source. Transcripts below are representative — the shapes are real, but run these yourself and expect your own paths and versions.

### Step 1 — establish the mechanism, not the folklore

```bash
$ kubectl get pod trainer-0 -o jsonpath='{.spec.runtimeClassName}{"\n"}'
nvidia

$ CID=$(kubectl get pod trainer-0 -o jsonpath='{.status.containerStatuses[0].containerID}' \
        | sed 's|containerd://||')
$ sudo crictl inspect $CID | jq -r '.info.runtimeType, .status.id'
io.containerd.runc.v2
7f3c1e0a...

$ sudo grep -A3 'runtimes.nvidia\]' /etc/containerd/conf.d/99-nvidia.toml
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
            BinaryName = "/usr/local/nvidia/toolkit/nvidia-container-runtime"

$ sudo grep -E '^\s*(mode|spec-dirs|root)' /etc/nvidia-container-runtime/config.toml
mode = "auto"
root = "/run/nvidia/driver"
spec-dirs = ["/etc/cdi", "/var/run/cdi"]

$ ls /etc/cdi/ /var/run/cdi/ 2>&1
/etc/cdi/:
/var/run/cdi/:
```

Read that carefully. `BinaryName` is the un-suffixed `nvidia-container-runtime`, so its mode comes from the config file: `auto`. Both spec directories are **empty**. On the folklore model you would now conclude "CDI is not active, so this is the legacy hook". That is wrong. `auto` on an NVML platform resolves to `jit-cdi`, and the empty directories are exactly what `jit-cdi` looks like. Prove it from the runtime's own log:

```bash
$ sudo sed -n 's/.*\(Auto-detected mode as.*\)/\1/p' \
      /var/log/nvidia-container-runtime.log | tail -2
Auto-detected mode as 'jit-cdi'
Auto-detected mode as 'jit-cdi'
```

(If `debug` is commented out in `[nvidia-container-runtime]`, that log does not exist. Setting `debug = "/var/log/nvidia-container-runtime.log"` and `log-level = "debug"`, then restarting one pod, is a two-minute change that answers this question permanently.)

The confirming negative: if the legacy path were active, the container's `config.json` would carry a prestart hook. Check:

```bash
$ sudo cat /run/containerd/io.containerd.runtime.v2.task/k8s.io/$CID/config.json \
      | jq '.hooks'
{
  "createContainer": [
    { "path": "/usr/local/nvidia/toolkit/nvidia-cdi-hook",
      "args": ["nvidia-cdi-hook","create-symlinks",
               "--link","libcuda.so.1::/usr/lib/x86_64-linux-gnu/libcuda.so"] },
    { "path": "/usr/local/nvidia/toolkit/nvidia-cdi-hook",
      "args": ["nvidia-cdi-hook","update-ldcache",
               "--folder","/usr/lib/x86_64-linux-gnu"] }
  ]
}
```

`createContainer` hooks and no `prestart` array: CDI edits, applied by `runc`. That is the finding, and it took four commands.

### Step 2 — inventory what the container received

```bash
$ kubectl exec trainer-0 -c cuda -- ls -l /dev/nvidia*
crw-rw-rw- 1 root root 195,   0 Aug 17 09:14 /dev/nvidia0
crw-rw-rw- 1 root root 195, 255 Aug 17 09:14 /dev/nvidiactl
crw-rw-rw- 1 root root 235,   0 Aug 17 09:14 /dev/nvidia-uvm
crw-rw-rw- 1 root root 235,   1 Aug 17 09:14 /dev/nvidia-uvm-tools

$ kubectl exec trainer-0 -c cuda -- env | grep NVIDIA
NVIDIA_VISIBLE_DEVICES=void
NVIDIA_DRIVER_CAPABILITIES=compute,utility
NVIDIA_CTK_LIBCUDA_DIR=/usr/lib/x86_64-linux-gnu

$ kubectl exec trainer-0 -c cuda -- sh -c \
    'ls -l /usr/lib/x86_64-linux-gnu/libcuda* /usr/lib/x86_64-linux-gnu/libnvidia-ml*'
lrwxrwxrwx 1 root root       20 libcuda.so.1 -> libcuda.so.580.65.06
-rw-r--r-- 1 root root 28905048 libcuda.so.580.65.06
lrwxrwxrwx 1 root root       25 libnvidia-ml.so.1 -> libnvidia-ml.so.580.65.06
-rw-r--r-- 1 root root  2010568 libnvidia-ml.so.580.65.06

$ kubectl exec trainer-0 -c cuda -- ldconfig -p | grep -E 'libcuda|libnvidia-ml'
        libcuda.so.1 (libc6,x86-64) => /usr/lib/x86_64-linux-gnu/libcuda.so.1
        libnvidia-ml.so.1 (libc6,x86-64) => /usr/lib/x86_64-linux-gnu/libnvidia-ml.so.1
```

Cross-check the three interesting observations against §7. `NVIDIA_VISIBLE_DEVICES=void` is *correct*, not a bug — the spec rewrote it after injection. `/dev/nvidia-uvm` is major **235**, not 195, because `nvidia-uvm` is a separate module with a dynamically allocated major; confirm on the host with `grep nvidia /proc/devices`. And `ldconfig -p` listing both SONAMEs is the proof that both hooks — `create-symlinks` and `update-ldcache` — actually ran; if the second had failed you would see the files but not the cache entries, and every `dlopen` would fail with a file sitting right there.

### Step 3 — read the two version numbers that decide §8

```bash
$ kubectl exec trainer-0 -c cuda -- nvidia-smi | head -4
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 580.65.06    Driver Version: 580.65.06    CUDA Version: 13.0     |
+-----------------------------------------------------------------------------+

$ kubectl exec trainer-0 -c cuda -- sh -c 'nvcc --version | tail -2; ls -d /usr/local/cuda-*'
Cuda compilation tools, release 13.1, V13.1.55
/usr/local/cuda-13.1

$ kubectl exec trainer-0 -c cuda -- python -c \
    "import torch; print(torch.cuda.is_available())"
False
```

Now the arithmetic. The host driver is 580.65.06 and reports a maximum supported CUDA of **13.0**. The image ships CUDA runtime **13.1**. NVML is healthy — `nvidia-smi` printed cleanly, so §8.1 is ruled out. The runtime is newer than the driver's ceiling: this is §8.2, CUDA error 35, and it is a node-side problem.

### Step 4 — decide between the three fixes, with numbers

Check whether forward compatibility is even available before proposing it:

```bash
$ kubectl exec trainer-0 -c cuda -- ls /usr/local/cuda/compat/
libcuda.so.590.44.01

$ kubectl exec trainer-0 -c cuda -- readelf -p .note.cuda.fwd_compatibility \
    /usr/local/cuda/compat/libcuda.so.590.44.01
# decoded:
#   CUDA Version: "13.1"
#   Driver: [535, 550, 570, 575, 580, 590]

$ kubectl exec trainer-0 -c cuda -- cat /etc/ld.so.conf.d/00-compat-*.conf 2>&1
cat: '/etc/ld.so.conf.d/00-compat-*.conf': No such file or directory
```

The compat library targets CUDA 13.1 and its allowlist **includes branch 580**, so this host qualifies — but the conf file is absent, so the compat directory is not on the linker path and the application is using the injected host `libcuda.so.580.65.06`. Two candidate causes: the hook was disabled (`--disable-hook=enable-cuda-compat`, or `cuda-compat-mode = "disabled"`), or the toolkit ran without a `--host-driver-version` and therefore no-oped. Confirm which from the generated spec's hook args or the runtime log; the fix is to re-enable the hook, at which point `ldconfig` prefers `/usr/local/cuda/compat` and `torch.cuda.is_available()` becomes `True` with no driver change on the node.

Compare that against the alternatives, in the units that matter to a fleet:

```
  OPTION A — enable the enable-cuda-compat hook
    blast radius : per-container (an ld.so.conf.d file inside the container)
    node change  : none; no drain, no reboot
    constraint   : requires branch 580 ∈ Driver[] of the compat lib  ✓ here
    fails if     : consumer-class GPU (error 804), or new silicon the
                   R580 kernel module never supported

  OPTION B — pin the image to CUDA 13.0
    blast radius : one workload, rebuilt
    node change  : none
    cost         : you give up whatever 13.1 gave you; every consumer of the
                   image must move together

  OPTION C — upgrade the host driver 580.65.06 → 595.71.05
    blast radius : EVERY container on the node, and every node you roll it to
    node change  : cordon + drain + module unload + install + reload + uncordon
    cost         : the full lesson 04.5 procedure. On a 200-node fleet this is
                   hours of drained capacity, not a config change.
```

**The takeaway to write into the failure-mode log:** the same error string has a per-container fix, a per-image fix, and a per-fleet fix, and the reason to spend five minutes reading an ELF note is to find out whether you are allowed to take the cheap one. Option C is lesson 04.5's subject precisely because it is the expensive one.

## Practice — feeds the deliverable

**Task.** Produce a written **"what's in the container and why"** inventory for the module's failure-mode log, plus a **mechanism determination** you could defend in an interview.

1. **Determine the mechanism, from evidence.** Record, for one real GPU node: the pod's `runtimeClassName`; the `BinaryName` for that handler in `/etc/containerd/config.toml` (or the drop-in, or `/etc/crio/crio.conf.d/`); the `mode` value in `/etc/nvidia-container-runtime/config.toml`; the contents of `/etc/cdi` and `/var/run/cdi`; and the `hooks` object from the container's live `config.json`. State which of the five modes is active and cite which two pieces of evidence prove it. Note your toolkit version (`nvidia-ctk --version`) and containerd version (`containerd --version`) alongside — including whether `enable_cdi` is set and under which plugin key for that config version.
2. **Inventory the container.** `kubectl exec` into a running GPU pod and record: every `/dev/nvidia*` node with its major and minor (and cross-check the majors against `/proc/devices` on the host, noting whether the entry is named `nvidia` or `nvidia-frontend`); the driver `.so` files present with their versioned filenames and SONAME symlinks; whether `ldconfig -p` lists them; the binaries injected; and the values of `NVIDIA_VISIBLE_DEVICES` and `NVIDIA_DRIVER_CAPABILITIES`.
3. **Toggle a capability and observe.** Run two otherwise-identical pods with `NVIDIA_DRIVER_CAPABILITIES=utility` and `NVIDIA_DRIVER_CAPABILITIES=compute,utility`. Record the library diff. Then run a third with the variable *unset* and confirm for yourself that the default is `utility,compute` and not `utility` — this is the folklore-correction step, and doing it once is worth more than reading it.
4. **Trace three files to their source.** For three files from step 2, name the exact mechanism that put them there: a `deviceNodes` entry, a `mounts` entry, or a specific `createContainer` hook. On a `cdi`-mode node read them out of the on-disk spec; on a `jit-cdi` node reconstruct the equivalent with `nvidia-ctk cdi generate --output=/tmp/nvidia.yaml` and diff what it declares against what you observed.
5. **Record the two version numbers and the compat verdict.** Capture `nvidia-smi`'s `Driver Version` and `CUDA Version` lines, the image's CUDA runtime version, whether `/usr/local/cuda/compat/` exists, and — if it does — the `Driver` allowlist from its ELF note. Write one sentence stating whether forward compatibility is available on this host and why.

**Acceptance.** A failure-mode-log entry that, for one real GPU pod: (a) names the active injection mode and the evidence for it, with toolkit and runtime versions; (b) lists device nodes with majors/minors, driver libraries with host provenance, binaries, and both `nvidia-smi` version numbers; (c) shows the capability-toggle diff including the unset case; (d) maps at least three container files to a specific `deviceNodes`/`mounts`/hook entry; and (e) states the forward-compatibility verdict for this host with the branch allowlist that justifies it. Someone reading it should be able to distinguish a missing-`libcuda`, an NVML mismatch, and a CUDA-runtime-too-new failure without touching the cluster.

## Common pitfalls

1. **Concluding "CDI is not active" from empty spec directories.** The default mode is `auto`, which resolves to `jit-cdi` on NVML platforms, and `jit-cdi` generates the spec in memory at kind `runtime.nvidia.com/gpu`. `/etc/cdi` and `/var/run/cdi` are legitimately empty, `nvidia-ctk cdi list` legitimately prints nothing, and CDI edits are legitimately what `runc` applied. Read the resolved mode from the runtime log or the container's `hooks` object; the filesystem is not the evidence.

2. **Believing `NVIDIA_DRIVER_CAPABILITIES` defaults to `utility` alone.** It has defaulted to `utility,compute` since NVIDIA Container Toolkit v1.4.0. Chasing this ghost costs real incident time. When `libcuda` is genuinely missing, the causes are `NVIDIA_VISIBLE_DEVICES=void`, the wrong runtime handler, an explicit `NVIDIA_DRIVER_CAPABILITIES=utility`, or a driver root (`root = "/run/nvidia/driver"`) pointing somewhere the libraries are not.

3. **Setting `enable_cdi = true` under the wrong plugin key.** containerd config `version = 3` (containerd 2.x) renamed the CRI plugin from `io.containerd.grpc.v1.cri` to `io.containerd.cri.v1.runtime`. A `true` written under the old key in a v3 config is inert, and inert is indistinguishable from unset. Match the key to the version — and note that containerd 2.x already defaults it to `true`, so the edit is usually unnecessary there and *always* necessary on 1.7.

4. **Treating `nvidia-smi` working as proof that CUDA works.** `nvidia-smi` is an NVML client. It exercises `libnvidia-ml` against the kernel module and nothing else. It will print a healthy GPU table on a node where every CUDA program fails with error 35, because the CUDA runtime in the image is what is too new. The pair of numbers to compare is `nvidia-smi`'s `CUDA Version:` field against the image's toolkit version.

5. **Leaving a persistent `/etc/cdi/nvidia.yaml` behind after a driver change.** A generated spec names driver libraries by exact versioned filename. Bump the driver and every `hostPath` in that spec is dangling: container create fails on a missing resource, or you get a version mismatch. Either regenerate on every driver change (the toolkit ships an `nvidia-cdi-refresh` systemd unit for exactly this) or stay on `jit-cdi`, which cannot go stale because it never persists anything.

6. **Setting `NVIDIA_VISIBLE_DEVICES` by hand in a manifest.** It bypasses the device plugin's allocation. On an Operator-managed node it is *ignored* for unprivileged pods (`accept-nvidia-visible-devices-envvar-when-unprivileged=false`), so the pod gets nothing and you waste time wondering why — and on a node where that guard is off, it is a genuine isolation escape. Let `Allocate` set it, and prefer `DEVICE_LIST_STRATEGY=volume-mounts` or `cdi-annotations` if you want the allocation carried by something a pod spec cannot forge.

7. **Mixing the two paths.** Registering `nvidia-container-runtime` as the handler *and* invoking `nvidia-container-runtime-hook` directly (for example via Docker's `--gpus` flag on the same host) gets you the loud error `invoking the NVIDIA Container Runtime Hook directly … is not supported`. That message is mode-detection working correctly, not a bug; use the runtime, or pin a handler to `nvidia-legacy` if you genuinely need the legacy path.

## Self-check

- **"CUDA driver version is insufficient for CUDA runtime version" inside a container — root cause, and how do you distinguish it from an NVML mismatch?** **Answer:** Two different seams. Error 35 (`cudaErrorInsufficientDriver`) means the **CUDA runtime from the image** (`libcudart.so`, plus cuBLAS/cuDNN/framework) requires driver-API entry points the host driver does not provide. The handshake between `libcuda.so` and the kernel module succeeded, so `nvidia-smi` prints a healthy GPU table and only your framework says there is no GPU — that asymmetry is the fingerprint. Compare `nvidia-smi`'s `CUDA Version:` field (the maximum CUDA the host driver supports) against the image's toolkit version (`nvcc --version`, or the image tag): if the image is higher and the gap crosses a major family, that is your error. An **NVML mismatch** (`Failed to initialize NVML: Driver/library version mismatch`) is the *other* seam: the userspace driver library in the container is not the same build as the loaded `nvidia.ko`. There `nvidia-smi` itself fails and names two versions, and the container often fails to create at all because `nvidia-container-cli` hits NVML during discovery. Error 35 is fixed node-side (driver upgrade, or enable the CUDA forward-compat libraries if the host branch is in their allowlist) or image-side by pinning to an older CUDA; the NVML mismatch is fixed by reconciling the host — reload the modules, unmount a stale `/run/nvidia/driver`, or regenerate a stale `/etc/cdi` spec.

- **`NVIDIA_VISIBLE_DEVICES=all` vs a specific UUID vs `void` vs `none` — what does each do, and why is `all` a security problem?** **Answer:** `all` exposes every GPU on the node — every device node, every GPU visible to CUDA and `nvidia-smi`. A specific `GPU-<uuid>` (what `Allocate` sets under the default `DEVICE_ID_STRATEGY=uuid`) exposes exactly one GPU's device node. `none` injects **no** device nodes but still mounts the driver libraries, giving a GPU-capable, GPU-less container. `void`, empty, or unset means the toolkit makes **zero** modifications — not a GPU container at all. `all` is a security problem because the variable is read from the container's environment and any pod spec can set environment variables, so an unguarded node lets a pod reach hardware the scheduler never allocated to it. NVIDIA Container Toolkit v1.4.1 added the guard, exposed as `accept-nvidia-visible-devices-envvar-when-unprivileged`; it defaults to `true` for Docker compatibility, and the GPU Operator sets it to `false`. The structurally better fix is not to carry the allocation in a forgeable channel at all — `DEVICE_LIST_STRATEGY=volume-mounts` uses mounts the pod cannot fabricate, and `cdi-annotations`/`cdi-cri` move the decision into the CRI request.

- **CDI vs the legacy runtime hook — what is the mechanical difference, and where do device nodes and libraries actually enter the container?** **Answer:** Legacy: a wrapper OCI runtime (`nvidia-container-runtime`) appends a **`prestart`** hook to `config.json` and execs stock `runc`; `runc` runs `nvidia-container-runtime-hook`, which calls `nvidia-container-cli configure --pid=<container pid>`. Because `prestart` executes in the **runtime** namespace, that process must `setns()` into the container's mount namespace and then `mknod` the device nodes, bind-mount the driver libraries, create the SONAME symlinks and run `ldconfig`. None of that appears in `config.json` — the injection is invisible and requires a vendor-specific runtime plus privilege. CDI: a spec (on disk under `/etc/cdi`/`/var/run/cdi`, or generated in memory in `jit-cdi` mode) declares `deviceNodes`, `mounts`, `env` and `hooks`; those edits are merged into `config.json` *before* `runc` runs, so **`runc` itself** creates the device nodes, adds the matching device-cgroup allow rules, and performs the bind mounts. The only hooks left run at **`createContainer`**, which executes *inside* the container's namespace, so `create-symlinks` and `update-ldcache` just write files with no namespace-entering and no container PID. The practical differences: stock `runc`, an auditable `config.json`, vendor-neutrality (Intel and AMD publish CDI specs too), rootless-friendliness, and — decisively — it is the substrate Kubernetes DRA and the CDI-in-device-plugin-API work are built on. Also note the OCI specification marks `prestart` deprecated in favour of `createRuntime`/`createContainer`/`startContainer`.

- **A node has empty `/etc/cdi` and `/var/run/cdi`, no prestart hook in `config.json`, and GPU pods work fine. What is happening?** **Answer:** The runtime is in `jit-cdi` mode — the default. `/etc/nvidia-container-runtime/config.toml` has `mode = "auto"`; the requested devices arrived as `NVIDIA_VISIBLE_DEVICES=GPU-<uuid>`, which is *not* a fully-qualified CDI name, so rule 1 of the `auto` resolution does not fire; the platform detects as NVML, so the default mode applies; and that default is `jit-cdi`. The runtime discovers the driver installation, builds a CDI spec in memory at kind `runtime.nvidia.com/gpu`, merges its `containerEdits` into `config.json`, and execs `runc`. Nothing is read from or written to the spec directories, and containerd's `enable_cdi` is irrelevant because containerd never resolves a CDI device — the NVIDIA runtime does. You confirm it two ways: the runtime log line `Auto-detected mode as 'jit-cdi'`, and the container's `config.json` showing `createContainer` hooks named `nvidia-cdi-hook` with no `prestart` array. The operational upside is that this mode cannot go stale across a driver upgrade, because the spec is regenerated per container.

- **What does CUDA forward compatibility actually buy you, and what does it not?** **Answer:** The `cuda-compat-<major>-<minor>` package installs a **newer `libcuda.so.<version>` into `/usr/local/cuda/compat/`** inside the image. That library exposes the new driver API to your application while speaking the older `ioctl` interface to the *unchanged* kernel module, and the `enable-cuda-compat` CDI hook puts its directory ahead of the injected host library by writing `/etc/ld.so.conf.d/00-compat-*.conf`. What it buys: running a newer CUDA runtime on a frozen host driver, per container, with no node change, no drain and no reboot. What it does **not** buy: (1) an arbitrary older driver — the compat library carries an ELF note `.note.cuda.fwd_compatibility` with an explicit `Driver` allowlist of host branches (e.g. a CUDA 13.1 compat build listing `[535, 550, 570, 575, 580, 590]`, which pointedly excludes 560 and 565), and the hook refuses unless the host branch is in that list *and* the compat version is strictly greater than the host's; (2) support for GPUs the loaded kernel module never knew about, since the module is what talks to the hardware and the note also carries a `Device` architecture allowlist; (3) anything on consumer-class hardware — that yields CUDA error 804 `cudaErrorCompatNotSupportedOnDevice`, "forward compatibility was attempted on non supported HW", which specifically means the compat libraries were found and loaded and the *hardware* refused them, so the fix is removing the compat path rather than adding packages; (4) a cure for an NVML driver/library mismatch, which is a module-versus-userspace build mismatch, a different failure entirely. Check availability with `readelf -p .note.cuda.fwd_compatibility /usr/local/cuda/compat/libcuda.so.*`.

- **Which four parts of the OCI runtime spec does GPU injection touch, and what breaks if each one is missing?** **Answer:** (1) `linux.devices` — the `mknod` list; missing it means no `/dev/nvidia*` inside the container, and `open()` fails with `ENOENT`. (2) `linux.resources.devices` — the device-cgroup allow rules; Kubernetes containers start from `{"allow": false, "access": "rwm"}`, so a device node with no matching allow rule exists and yet `open()` returns `EPERM`. Those first two are why "node missing" and "permission denied" are different bugs. (3) `mounts` — the `ro,nosuid,nodev,rbind,rprivate` bind mounts that carry `libcuda.so.<ver>`, `libnvidia-ml.so.<ver>` and `nvidia-smi` from the host; missing them means CUDA cannot find a driver library at all. (4) `hooks` — the small in-namespace steps that make the mounts usable: `create-symlinks` produces `libcuda.so.1 → libcuda.so.<ver>` (without it `dlopen("libcuda.so.1")` fails even though the versioned file is mounted) and `update-ldcache` runs `ldconfig` so the linker's cache lists them (without it `ldconfig -p | grep libcuda` is empty while `ls` shows the file — a very confusing pair of observations if you do not know the hook exists).

## Connections & what's next

This lesson is the mechanism layer between two things you already have: lesson 04.3's `Allocate` output (env vars, device names, mounts) and module 03's host driver stack. It is also the substrate lesson 04.9 depends on directly — Dynamic Resource Allocation's device model produces CDI specs, so the `containerEdits` grammar and the runtime matrix here are a prerequisite you will recognise by name, not a side topic. Lesson 04.6's MIG devices are CDI devices too: the `/dev/nvidia-caps/nvidia-cap<minor>` nodes sketched in §7 are how a MIG slice's isolation is expressed at the device-node level, and the profile-to-device-node mapping is that lesson's subject.

The through-line into the next lesson is §8. Every failure in that section traced back to a **host** driver whose version had moved, or needed to move, underneath running containers — the injected `libcuda`, the kernel module, the driver root at `/run/nvidia/driver`, the versioned filenames baked into a persistent CDI spec. This lesson gave you the per-container fixes. The expensive fix, changing the driver itself, is a fleet operation with a state machine, a blocking condition at every step, and a real GPU-hour bill.

Next: **[04.5 · Driver lifecycle & fleet upgrades](05-driver-lifecycle-upgrades.md)** takes that host-side dependency and turns it into a managed process — the upgrade controller's per-node state machine, why the kernel module cannot be unloaded while a process holds the device, and how to design a 200-node rollout that does not break every running job at once.

## References & further reading

**Primary sources**
- [CDI specification — `cncf-tags/container-device-interface`, `SPEC.md`](https://github.com/cncf-tags/container-device-interface/blob/main/SPEC.md) — fetched this session; **confirmed spec version v1.1.0** (the released-versions table also gives the field-to-version mapping used in §5.1). *Correction: an earlier pass of this lesson cited v0.6.0; that was the version at which `annotations` was added, not the current spec version.*
- [CDI configuration guide — `cncf-tags/container-device-interface`, `README.md`](https://github.com/cncf-tags/container-device-interface#readme) — fetched this session; the source of the §6.3 runtime support matrix, including "CRI-O CDI support is enabled by default", Docker default-on from 28.2.0 and supported from 25.0.0, and Podman requiring no configuration.
- [OCI Runtime Specification — `config.md`](https://github.com/opencontainers/runtime-spec/blob/main/config.md) — fetched this session; the hook-phase table in §2, including `prestart` being **deprecated** and the runtime-vs-container namespace distinction that the whole legacy/CDI comparison rests on.
- [NVIDIA Container Toolkit repository](https://github.com/NVIDIA/nvidia-container-toolkit) — cloned and read this session at **v1.20.0**. Specific files behind claims in this lesson: `api/config/v1/config.go` and `api/config/v1/toml_test.go` (the §6.1 default `config.toml`), `internal/info/auto.go` (the five modes and the `auto` resolution rule), `internal/modifier/stable.go` (the legacy prestart-hook insertion), `internal/modifier/cdi.go` (`jit-cdi` and the `runtime.nvidia.com/gpu` kind), `internal/config/image/capabilities.go` and `devices.go` (the capability and device-list semantics), `cmd/nvidia-ctk/cdi/generate/generate_test.go` (the §5.2 generated spec), `internal/info/proc/devices/devices.go` (device majors and the `nvidia-frontend`→`nvidia` rename at driver 550.40.x), `internal/nvcaps/nvcaps.go` and `internal/platform-support/dgpu/nvml.go` (the MIG cap device nodes).
- [NVIDIA Container Toolkit `CHANGELOG.md`](https://github.com/NVIDIA/nvidia-container-toolkit/blob/main/CHANGELOG.md) — read this session; the v1.4.0 "Add 'compute' capability to list of defaults" and v1.4.1 "Ignore `NVIDIA_VISIBLE_DEVICES` for containers with insufficient privileges" entries, plus the v1.20.0 CDI-hook changes. *Correction: an earlier pass of this lesson stated that an unset `NVIDIA_DRIVER_CAPABILITIES` defaults to `utility` only; the source says `utility,compute` and has since v1.4.0.*
- [containerd CRI plugin config guide — `docs/cri/config.md`](https://github.com/containerd/containerd/blob/main/docs/cri/config.md) — fetched this session on both `main` and `release/1.7`; the source of `enable_cdi = true` / `cdi_spec_dirs = ['/etc/cdi', '/var/run/cdi']` as containerd 2.x defaults, `enable_cdi = false` on 1.7, the config `version = 3` plugin rename to `io.containerd.cri.v1.runtime`, and containerd's own note that `enable_cdi` is expected to be deprecated once DRA or CDI-in-device-plugin-API reach GA. *Correction: an earlier pass said "containerd: supported, not default" without a version qualifier; the default depends on the containerd major version.*
- [NVIDIA `k8s-device-plugin` repository](https://github.com/NVIDIA/k8s-device-plugin) — cloned and read this session; `DEVICE_LIST_STRATEGY` (`envvar` | `volume-mounts` | `cdi-annotations` | `cdi-cri`, default `envvar`), `DEVICE_ID_STRATEGY` (`uuid` | `index`, default `uuid`), and the `cdi.k8s.io/nvidia-device-plugin_<claim>` annotation format in `internal/plugin/server.go`.
- [NVIDIA GPU Operator repository](https://github.com/NVIDIA/gpu-operator) — cloned and read this session at **v26.3.3**; `deployments/gpu-operator/values.yaml` confirms `cdi.enabled: true` and `toolkit.version: v1.20.0`, and `api/nvidia/v1/clusterpolicy_types.go` confirms the `CDIConfigSpec.Enabled` field defaults to `true`.

**Real-world engineering evidence**
- [NVIDIA/nvidia-container-toolkit#394](https://github.com/NVIDIA/nvidia-container-toolkit/issues/394), [NVIDIA/gpu-operator#898](https://github.com/NVIDIA/gpu-operator/issues/898), [NVIDIA/k8s-device-plugin#1431](https://github.com/nvidia/k8s-device-plugin/issues/1431) — three independent filings of `nvidia-container-cli: initialization error: nvml error: driver/library version mismatch: unknown`. *Titles and numbers found via search this session; bodies not independently fetched (GitHub API access is restricted in this environment). Use them as pointers to the symptom's prevalence; the mechanism in §8.1 is verified from the toolkit source, not from the issues.*
- [NVIDIA/gpu-operator#1220 — "gpu-operator breaks when upgrading EKS to K8s v1.30"](https://github.com/NVIDIA/gpu-operator/issues/1220) — the `failed to get sandbox runtime: no runtime for 'nvidia' is configured` runtime-wiring failure. *Recorded in an earlier pass of this course; not independently re-fetched this session.*

**Deeper dives**
- [`nvidia-cdi-hook` command reference](https://github.com/NVIDIA/nvidia-container-toolkit/tree/main/cmd/nvidia-cdi-hook) — the hook subcommands referenced in §5.2 (`create-symlinks`, `update-ldcache`, `enable-cuda-compat`, `disable-device-node-modification`, `update-application-profile`, `chmod`), each a small self-contained program worth reading if you want to know exactly what a generated spec will do to your container.
- [`cudacompat` hook source](https://github.com/NVIDIA/nvidia-container-toolkit/tree/main/cmd/nvidia-cdi-hook/cudacompat) — read this session; the `.note.cuda.fwd_compatibility` ELF-note parsing and the `UseCompat` decision rule behind §8.3, including the test fixtures that give the real `Driver` allowlists quoted there.
