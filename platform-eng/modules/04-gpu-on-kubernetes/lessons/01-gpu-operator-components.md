---
lesson: "04.1"
title: "GPU Operator components and the init-container dependency chain"
module: "04"
concept: "GPU Operator components and the init-container dependency chain"
status: not-started
est_time: "10h"
prev: null
next: "02-crash-loop-diagnosis.md"
artifacts: []
sources: 5
---

# 04.1 · GPU Operator components and the init-container dependency chain

> **Concept.** The GPU Operator is not one thing — it is a system of cooperating DaemonSets gated by init-container validations that pass a node from bare metal to "can schedule `nvidia.com/gpu`".
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Module 03 gave you hardware literacy: the driver/kernel-module relationship, the CUDA execution model, MIG as a silicon-level partition. Module 02 gave you the device-plugin gRPC API and the DRA object model — the *mechanics* of how a GPU becomes a schedulable Kubernetes resource. Both of those were taught as concepts you reason about in isolation. This module is where they stop being isolated concepts and become **one running system you operate** — the GPU Operator, plus everything downstream of it. Lesson 1 builds the map: every operand it deploys, what each one gates, and why a fresh install spends its first several minutes sitting in `Init:0/1`. Lesson 2 immediately breaks that map on purpose, because a map you haven't used to diagnose a failure isn't actually learned.

## Why this matters

This module's README calls itself "the core operational module, the one a hiring manager probes hardest" — and this lesson is the anchor of that probe. CoreWeave's *Sr Eng, Kubernetes Platforms* posting asks for engineers who build "controllers/operators that automate infrastructure testing" and maintain "visibility into system metrics/performance/health" across "tens of thousands of kubelets." NVIDIA's *Sr DevOps, Platform Eng* posting asks candidates to "troubleshoot… GPU-accelerated servers to minimize downtime." Neither of those is answerable from a device-plugin API diagram — they're answerable only if you can point at the exact pod that owns a symptom, on a live cluster, under time pressure.

The concrete cost of not knowing this: a GPU node that "isn't working" is, in the overwhelming majority of real incidents, one stage of this chain that didn't pass while everything downstream sits blocked. An engineer who doesn't know the chain exists debugs the *symptom* — a Pending workload pod — and burns an hour restarting things that were never broken. An engineer who knows the chain reads one `kubectl get pods -n gpu-operator`, walks to the first non-`Running` stage, and has localized the incident before the first engineer finishes typing `kubectl describe pod`. On a GPU fleet, that hour is not free: it's GPU-hours billed at full rate while the hardware sits idle.

## What's new here (calibration)

This module's README is explicit that 02/02b/03 already own the theory, and this module is the **operational integration layer** on top of it. Concretely, this lesson does **not** re-teach:

- The device-plugin gRPC API (`ListAndWatch`, `Allocate`) or the DRA object model — Module 02 owns the *mechanics* of how a GPU becomes `nvidia.com/gpu: 1` or a `ResourceClaim`.
- Topology Manager or NUMA/PCIe topology reasoning — Module 02b's territory.
- The driver/kernel-module relationship or the CUDA execution model at the silicon level — Module 03's territory.

What this lesson adds instead: the Operator as a **running system** — the full operand inventory, the `ClusterPolicy` reconciliation model, the exact sentinel-file mechanism that orders every operand, and the operational judgment to tell "the operator ignored my node" apart from "a stage genuinely failed." You already know what a device plugin *is*; here you learn why it sits in `Init:0/1` for ten minutes and whose fault that is.

## Core concepts

### The operands (what runs in `gpu-operator`)

A default install on a single GPU node produces roughly these pods:

| Pod / workload | Kind | Role |
|---|---|---|
| `gpu-operator-…` | Deployment | The controller. Watches the `ClusterPolicy` CR, reconciles which operands land on which nodes. |
| `gpu-operator-node-feature-discovery-master` / `-worker` | Deployment / DaemonSet | **NFD.** Worker inspects hardware and emits `feature.node.kubernetes.io/pci-10de.present=true` (`0x10de` = NVIDIA's PCI vendor ID). This label is the trigger for everything else. |
| `nvidia-driver-daemonset-…` | DaemonSet | Builds/loads the kernel modules (`nvidia`, `nvidia_uvm`, `nvidia_modeset`) and exposes the userspace driver under `/run/nvidia/driver`. |
| `nvidia-container-toolkit-daemonset-…` | DaemonSet | Installs the toolkit to `/usr/local/nvidia/toolkit` and wires the container runtime so a GPU-aware runtime exists. |
| `nvidia-device-plugin-daemonset-…` | DaemonSet | The device plugin from Module 02 — registers `nvidia.com/gpu` with kubelet. |
| `gpu-feature-discovery-…` (GFD) | DaemonSet | Labels the node with GPU **product/memory/count** (`nvidia.com/gpu.product`, `nvidia.com/gpu.memory`, `nvidia.com/gpu.count`). |
| `nvidia-dcgm-exporter-…` | DaemonSet | Scrapes GPU telemetry via DCGM, exposes Prometheus metrics on `:9400`. Your cost/observability substrate. |
| `nvidia-dcgm-…` | DaemonSet | Optional standalone DCGM host engine (exporter can embed it instead). |
| `nvidia-mig-manager-…` | DaemonSet | Applies MIG geometry. **Only scheduled on MIG-capable GPUs (A100/H100/H200/GB200).** On L4/A10/L40S it will not appear — expected, not a bug. |
| `nvidia-operator-validator-…` | DaemonSet | The gate that declares the node truly ready. |
| `nvidia-cuda-validator-…` | Pod (Job-like) | Spawned by the validator; runs a real CUDA `vectorAdd` and exits. |

The Operator decides placement using per-node `nvidia.com/gpu.deploy.*` labels (e.g. `nvidia.com/gpu.deploy.device-plugin=true`) that it manages off the NFD label. Setting one to `false` is the supported way to keep an operand off a node — and, as you'll see in 04.2, a great way to break one on purpose.

Everything is reconciled from a single cluster-scoped CR, the **`ClusterPolicy`** (`kubectl get clusterpolicy -o yaml`). The Helm values you pass become fields on it; the operator Deployment watches it and materializes the DaemonSets above. When you need to know "what did this cluster actually ask for," read the `ClusterPolicy`, not the Helm values file — drift between them is a real failure source. Its `.status` also carries a coarse `state` (`ready` / `notReady`) that summarizes the whole chain.

### What the driver container actually does on the host

The driver DaemonSet is the operand most people mis-model. It does **not** install a driver into the container in the ordinary sense — it builds the kernel modules against the *host's* running kernel and exposes the resulting userspace driver tree on the host under **`/run/nvidia/driver`** (a bind-mount the toolkit and every GPU container later consume via mount propagation). That is why the driver is a privileged DaemonSet with host PID/mounts: it must `insmod` into the host kernel and publish `/dev/nvidia*` device nodes the host can see. The `driver.version` you pin selects the module source/precompiled branch; it must be compatible with the node's kernel — the entire premise of 04.2's first break. Two escape hatches you will use on-prem: `driver.enabled=false` (host already has the driver from a golden image — the operator skips the build and validates the host driver instead), and precompiled driver images (`driver.usePrecompiled=true`) that trade flexibility for fast, deterministic boots on a known kernel.

### GFD labels and DCGM metrics — the cost/observability substrate

Two operands exist mostly to feed the layer this course is really about. **GFD** stamps the node with machine-readable facts your scheduler *and* your cost operator will key off: `nvidia.com/gpu.product` (e.g. `NVIDIA-L4`), `nvidia.com/gpu.memory` (MiB), `nvidia.com/gpu.count`, and — under MIG — `nvidia.com/mig-<profile>.*`. **DCGM-Exporter** publishes Prometheus metrics on `:9400` that are the raw material of GPU cost/efficiency: `DCGM_FI_DEV_GPU_UTIL` (SM utilization %), `DCGM_FI_DEV_FB_USED` / `DCGM_FI_DEV_FB_FREE` (framebuffer memory), `DCGM_FI_DEV_POWER_USAGE` (watts), and `DCGM_FI_PROF_*` profiling counters. Critically for attribution, the exporter joins each metric to the consuming pod via `pod`/`namespace`/`container` labels it derives from the device-to-pod mapping — exactly the join your capstone operator will lean on to turn "GPU was 12% utilized" into "*this team's* pod wasted 88% of an L40S." Note DCGM needs both the driver and toolkit healthy to read the hardware, which is why it sits downstream of the toolkit gate.

### The dependency chain (the load-bearing idea)

Operands do not merely *depend* on each other logically; the Operator enforces the order with **init containers that block on sentinel files** written to a shared host path, `/run/nvidia/validations/`. Each stage, once healthy, drops a `*-ready` file; the next stage's init container spins until that file appears. Order:

```
NFD ──▶ Driver DS ──▶ Container-Toolkit DS ──▶ Device-Plugin DS ──▶ GFD
                                    └────────────▶ DCGM-Exporter / (MIG-Manager)
                                                          │
                                       operator-validator ┘ (driver→toolkit→cuda→plugin)
```

Stage by stage, and **what each gates**:

1. **NFD** → emits `pci-10de.present`. Gates *scheduling of every operand*: no label, no driver pod, no anything.
2. **Driver DaemonSet** → its init container `k8s-driver-manager` unloads stale modules / handles upgrades; the main container builds and `insmod`s the kernel modules. On success writes `driver-ready`. Gates the toolkit. **If this fails, the whole node is dead** — everything below is stuck in Init.
3. **Container-Toolkit DaemonSet** → init container `driver-validation` (from the `gpu-operator-validator` image) blocks on `driver-ready`; main container installs the toolkit and runs `nvidia-ctk runtime configure --runtime=containerd`, patching `/etc/containerd/config.toml` and restarting containerd. Writes `toolkit-ready`. Gates the device plugin, GFD, DCGM.
4. **Device-Plugin DaemonSet** → init container `toolkit-validation` blocks on `toolkit-ready`; main container registers `nvidia.com/gpu`. Writes `plugin-ready`. This is the stage that makes the resource *allocatable*.
5. **GFD** → also gated on toolkit; adds the descriptive `nvidia.com/gpu.*` labels used by schedulers and your cost operator.
6. **DCGM-Exporter** (and **MIG-Manager** where applicable) → gated on toolkit (DCGM needs the driver + toolkit to talk to the GPU).
7. **operator-validator** → the capstone. Its init containers run **driver-validation → toolkit-validation → cuda-validation → plugin-validation** in sequence (each writing the matching `*-ready`), then the main container marks the node good. `cuda-validation` launches the `nvidia-cuda-validator` pod that runs a real kernel end-to-end.

The mental model to carry: **a pod in `Init:0/1` is never the culprit — it is a victim.** Walk *up* the chain to the first stage that is `CrashLoopBackOff` or not `Running`; that pod owns the incident.

You can see the sentinel mechanism directly. The files live on the host and inside the validator's shared mount:
```bash
# On the node (or via a debug pod with hostPath /run/nvidia):
ls -l /run/nvidia/validations/
# driver-ready  toolkit-ready  cuda-ready  plugin-ready   <-- present == that gate released
```
An operand stuck in `Init` is literally a shell loop inside its init container polling for the matching file. This is why "delete the downstream pod" never fixes anything: the gate is a property of the *upstream* stage, and the freshly recreated pod re-enters the same wait.

### Upgrades and node reboots (why order matters twice)

The chain is enforced not just at first install but on every driver change and reboot. A `helm upgrade` that bumps `driver.version` triggers `k8s-driver-manager` to cordon/drain GPU pods, unload the old modules, and rebuild — during which the toolkit/plugin/validator briefly drop back to `Init` by design. On a node reboot the kernel modules are gone until the driver DaemonSet reloads them, so a few minutes of `nvidia.com/gpu: 0` after reboot is expected, not an incident. Knowing this keeps you from "fixing" a cluster that is simply mid-convergence. Lesson 5 goes deep on the upgrade state machine itself; this lesson only needs you to recognize the pattern when you see it.

### Install (Helm) — and a version note you should not skip

```bash
# 1. Add and refresh the NVIDIA Helm repo
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia && helm repo update

# 2. See what's actually current before you pin anything
helm search repo nvidia/gpu-operator -l | head

# 3. Install into its own namespace, waiting for operands to converge
helm install --wait gpu-operator \
  -n gpu-operator --create-namespace \
  nvidia/gpu-operator \
  --version=<the version you resolved above>
```

**Do not hard-pin a specific version number from any lesson, including this one — always resolve it live with `helm search repo nvidia/gpu-operator -l` and pin *that*.** The GPU Operator's public release line has moved fast: from the 25.3.x line, past 25.10.0/25.10.1, to the 26.3.x line as of mid-2026. Two changes are worth knowing by name because they change what you'll see in the field:

- **v25.10.0 made CDI (Container Device Interface) the default injection path** on containerd/CRI-O — the legacy prestart-hook mechanism is no longer what you're debugging by default. Lesson 4 covers CDI in depth; for now, just know that on a 25.10+ install, `nvidia-ctk cdi list` is where you check device injection, not the old hook.
- **v26.3.0 added an `NVIDIADriver` CRD** for running multiple driver versions/OS images across a heterogeneous fleet (previously a single `ClusterPolicy.driver` block applied cluster-wide), and **same-version driver-pod restarts now reuse already-loaded kernel modules instead of rebuilding them** — recovery from a driver-pod restart that doesn't actually change the driver version drops from minutes to seconds. This doesn't change anything in this lesson's chain, but it matters for 04.2 and 04.5: not every driver-pod restart you'll see in the wild is a full kernel-module rebuild anymore.

Other chart values you will reach for regardless of version: `driver.version`, `driver.enabled=false` (if the host already has a driver), `toolkit.enabled=false` (pre-installed toolkit), and `mig.strategy`.

If your nodes **already have the driver** (common on-prem with a golden image), set `--set driver.enabled=false`; the Operator then skips stage 2 and its `driver-ready` file is produced by the validator against the host driver.

**DRA** (structured GPU allocation, `resource.k8s.io`) is **GA in Kubernetes 1.34**; the Operator can run device-plugin mode or DRA mode. This lesson uses the classic device-plugin path — DRA gets its own lesson (04.9) once you've built the device-plugin-based attribution pipeline this chain enables.

### Reading each component's logs

```bash
kubectl get pods -n gpu-operator -o wide          # the whole system at a glance
kubectl -n gpu-operator logs ds/nvidia-driver-daemonset        # kmod build/insmod
kubectl -n gpu-operator logs ds/nvidia-container-toolkit-daemonset  # nvidia-ctk output
kubectl -n gpu-operator logs ds/nvidia-device-plugin-daemonset # ListAndWatch / registration
kubectl -n gpu-operator logs ds/nvidia-operator-validator -c cuda-validation
kubectl -n gpu-operator logs nvidia-cuda-validator-xxxxx       # "Test PASSED"
```
For a pod stuck in Init, name the init container explicitly: `kubectl logs <pod> -c toolkit-validation`. `kubectl describe pod <pod>` shows *which* init container is pending and its wait message.

### Verifying the node end-to-end (not just "pods are green")

A green operand list is necessary but not sufficient — the node is *ready* only when the full path from resource to executed kernel works. Confirm all four layers:
```bash
# 1. Resource advertised and allocatable
kubectl describe node <node> | grep -A3 Allocatable       # nvidia.com/gpu: 1

# 2. Descriptive labels present (GFD did its job)
kubectl get node <node> -o json | jq '.metadata.labels | with_entries(select(.key|test("nvidia.com/gpu")))'

# 3. Driver sees the silicon
kubectl -n gpu-operator exec ds/nvidia-driver-daemonset -- nvidia-smi -L

# 4. A pod actually got a device (after scheduling cuda-vectoradd)
kubectl exec cuda-vectoradd -- nvidia-smi   # only works if the nvidia runtime injected /dev/nvidia*
```
If 1–3 pass but 4 fails, the resource is advertised but the *runtime wiring* is wrong — the exact failure family 04.2 drills. Building the habit of checking layer 4 (a real device inside a real pod) is what stops "the dashboard is green but jobs can't see a GPU" from becoming a two-hour incident.

### Placement labels you will actually touch

The operator manages a family of `nvidia.com/gpu.deploy.*` labels — `driver`, `container-toolkit`, `device-plugin`, `gpu-feature-discovery`, `dcgm-exporter`, `operator-validator`, `mig-manager`. Each is the on/off switch for that operand on that node. In steady state you don't set them; the operator does, off the NFD `pci-10de.present` label. But they are your surgical tool: to keep the plugin off one node without uninstalling anything, set `nvidia.com/gpu.deploy.device-plugin=false`. To understand why an operand *isn't* on a node, read these labels first — an unexpected `false` (or a missing NFD label) explains most "the operator ignored my node" reports.

## Perspectives

**Developer/platform-consumer perspective.** From a workload author's chair, none of this chain is visible. They write `resources.limits: {nvidia.com/gpu: 1}` and either their pod schedules and runs, or it doesn't. They never see NFD, the toolkit, or the validator — which is exactly the point of the abstraction. Your job, in this course, is to be the person who *does* see it when their pod is stuck and they can't tell you why.

**Operator/SRE perspective.** The chain is a state machine you read top-down, not a pile of independent pods. A pod stuck in `Init` is always a *symptom*, never the *cause* — the discipline is walking up to the first stage that's actually unhealthy and stopping there. This is the single habit lesson 2 is built to drill until it's automatic.

**Vendor/managed-platform perspective.** Not every GPU-fleet operator wants you running this Operator yourself. CoreWeave's own documentation for its managed Kubernetes offering (CKS — CoreWeave Kubernetes Service) recommends *against* tenants self-installing the GPU Operator; CoreWeave manages driver/toolkit/device-plugin lifecycle on your behalf as part of the platform (their changelog records GPU driver management features shipping to CKS in August 2025). That's a real, deliberate design choice by a GPU-cloud vendor: on a platform where they own the hardware and the fleet-wide upgrade risk, they don't want tenants independently reconciling `ClusterPolicy` against nodes they don't fully control. Knowing this chain cold is still valuable there — you'll debug *symptoms* of it even when you don't own the install — but recognize when you're the operator of this system versus a consumer of someone else's.

**Economics perspective.** Every minute a node sits in `Init` waiting on an upstream gate is a fully-billed idle GPU-hour — the hardware is provisioned and billed whether or not a workload can actually use it yet. A ten-minute convergence on a fresh install is normal and cheap. A ten-minute convergence that's actually a wedged driver pod, undiagnosed for an hour because nobody knew to walk up the chain, is not a debugging inconvenience — it's a line item.

## Real-world use cases

- **[NVIDIA/gpu-operator](https://github.com/NVIDIA/gpu-operator) — the Operator's own repository.** Primary source for the component list, the `ClusterPolicy` model, and the exact operand behavior described above; read the README and `deployments/gpu-operator` chart values directly rather than trusting any single blog's snapshot of them, since the chart evolves release to release.
- **[CoreWeave — "About GPU Driver Management in CKS"](https://docs.coreweave.com/docs/products/cks/nodes/gpu-driver-management/gpu-driver-management-cks)** — a real GPU-fleet operator's documented policy of owning driver/Operator lifecycle rather than letting tenants self-install it. *Not independently fetched this session* (proxy-blocked domain) — corroborated by a second independent search hit and a CoreWeave changelog entry dated August 15, 2025; treat as high-confidence but spot-check the exact wording before quoting it.
- A note on a gap, deliberately left honest rather than papered over: this research could not find or confirm a tier-1 engineering-blog postmortem specifically narrating a GPU-Operator dependency-chain incident (the kind of "our driver DaemonSet crash-looped and here's the timeline" writeup you'd expect from Meta/Netflix/Discord-style postmortems). If you find one at study time, it's a strong addition — but the honest state right now is: the primary source (the repo itself) and the vendor-policy example above are what's verified.

## Worked example

Fresh single-node cluster, one L4. After `helm install`, first snapshot:
```
$ kubectl get pods -n gpu-operator
NAME                                          READY   STATUS     RESTARTS   AGE
gpu-operator-7c9d8b6f5-2mzln                  1/1     Running    0          90s
gpu-operator-node-feature-discovery-...       1/1     Running    0          90s
nvidia-driver-daemonset-abcde                 1/1     Running    0          80s
nvidia-container-toolkit-daemonset-fghij      0/1     Init:0/1   0          60s
nvidia-device-plugin-daemonset-klmno          0/1     Init:0/1   0          60s
nvidia-operator-validator-pqrst               0/1     Init:0/4   0          60s
```
Driver is `Running`; toolkit is `Init:0/1`. Its init container is `driver-validation`, waiting on `driver-ready`. Confirm the gate is about to release:
```
$ kubectl -n gpu-operator logs nvidia-driver-daemonset-abcde | tail -3
Loading NVIDIA driver kernel modules...
Successfully loaded nvidia, nvidia_uvm, nvidia_modeset
Done, now waiting for signal
```
Driver is healthy, so `driver-ready` should exist and toolkit should proceed. Ninety seconds later:
```
$ kubectl get pods -n gpu-operator
nvidia-container-toolkit-daemonset-fghij      1/1     Running    0          3m
nvidia-device-plugin-daemonset-klmno          1/1     Running    0          3m
gpu-feature-discovery-uvwxy                   1/1     Running    0          2m
nvidia-dcgm-exporter-zzzzz                    1/1     Running    0          2m
nvidia-operator-validator-pqrst               1/1     Running    0          3m
nvidia-cuda-validator-vvvvv                   0/1     Completed  0          2m
```
The chain drained top-to-bottom. Confirm the resource landed and the sample ran:
```
$ kubectl get node -o jsonpath='{.items[0].status.allocatable.nvidia\.com/gpu}'; echo
1
$ kubectl -n gpu-operator logs nvidia-cuda-validator-vvvvv
[Vector addition of 50000 elements]
Test PASSED
```
Node is genuinely ready — not "resource advertised," but "a real CUDA kernel executed on this GPU."

## Practice

On a fresh single-node cluster with a **rented L4 / A10 / L40S** (kubeadm or k3s, containerd runtime) — this is the cluster you'll keep across lessons 1–2 and reuse for the [Per-pod GPU attribution](../practice/per-pod-attribution/README.md) deliverable:

1. `helm search repo nvidia/gpu-operator -l` to resolve the current version, then install with the pinned Helm command above.
2. `kubectl get pods -n gpu-operator -w` and **watch the chain converge**. Record the moment each operand leaves `Init`.
3. For **every** operand, run `kubectl logs` and write one sentence: *what this pod does and what gates it.* This is your **component map** deliverable — a table with pod name, role, and the sentinel file it waits on / writes.
4. Deploy a GPU test pod and confirm it runs:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata: { name: cuda-vectoradd }
   spec:
     restartPolicy: OnFailure
     containers:
     - name: cuda-vectoradd
       image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubi8
       resources: { limits: { nvidia.com/gpu: 1 } }
   ```
   `kubectl logs cuda-vectoradd` must show `Test PASSED`.

**Acceptance:** (a) a `cuda-vectoradd` pod that reaches `Completed` with `Test PASSED`; (b) a written component map naming all operands, each pod's role, and its position/gate in the dependency chain (note that `nvidia-mig-manager` is *absent* on L4/A10/L40S and say why). Keep this cluster running — 04.2 breaks it on purpose.

## Common pitfalls

1. **Believing a self-managed GPU Operator install is always the right move.** On a managed GPU cloud (CoreWeave CKS and similar), the platform explicitly owns this layer — installing your own Operator against nodes the vendor already manages is a good way to fight the platform, not extend it. Know which situation you're in before you `helm install`.
2. **Treating `driver.enabled=false` as "the Operator does nothing."** It still runs the toolkit, device plugin, DCGM, and validator against the host's pre-existing driver; only the driver DaemonSet itself is skipped. The rest of the chain — and all of its failure modes — is still live.
3. **Assuming NFD absence is a bug rather than the single most common root cause of "the operator ignored my node."** No `pci-10de.present` label means no operand is scheduled there at all, by design — check NFD before suspecting the Operator's controller logic.
4. **Hard-pinning a version number from documentation (or this lesson) without re-checking.** The GPU Operator line moves release to release; `--version=v25.3.2` written into a runbook two years ago is stale today. Resolve current with `helm search repo nvidia/gpu-operator -l` every time you install.
5. **"Delete the downstream pod" as a fix reflex.** A pod stuck in `Init` restarts into the exact same wait, because the gate lives in the *upstream* stage's health, not in the pod you deleted. Fix the first broken stage; downstream pods recover on their own.

## Self-check

- Put the init-container dependency chain in order and say what each stage gates. **Answer:** NFD → Driver → Container-Toolkit → Device-Plugin → GFD, with DCGM-Exporter (and MIG-Manager on capable GPUs) and the operator-validator hanging off the toolkit. NFD's `pci-10de.present` label gates scheduling of every operand; the Driver DS writes `driver-ready` and gates the toolkit; the Toolkit DS writes `toolkit-ready` and gates the device plugin, GFD, and DCGM; the Device Plugin writes `plugin-ready` and makes `nvidia.com/gpu` allocatable; the operator-validator runs driver→toolkit→cuda→plugin validations and declares the node truly ready. Gating is enforced by init containers blocking on `*-ready` sentinel files in `/run/nvidia/validations/`.
- Which component configures the container runtime, and what breaks if it fails? **Answer:** The **container-toolkit DaemonSet**. It runs `nvidia-ctk runtime configure --runtime=containerd`, which patches `/etc/containerd/config.toml` to add an `nvidia` runtime and sets it as default, then restarts containerd. If it fails, `toolkit-ready` is never written, so the device plugin, GFD, DCGM, and validator all stay in `Init` — and even if a GPU pod were scheduled, containerd could not inject the GPU, so the container would see no device.
- What does the operator-validator actually validate before marking the node ready? **Answer:** Four things, in order, each in its own init container: **driver-validation** (kernel modules loaded, `nvidia-smi` works via the driver), **toolkit-validation** (the NVIDIA container runtime is installed and wired into containerd), **cuda-validation** (it launches the `nvidia-cuda-validator` pod that runs a real `vectorAdd` kernel end-to-end — proving CUDA actually executes, not just that a resource is advertised), and **plugin-validation** (`nvidia.com/gpu` is registered and allocatable > 0). Only when all four pass does the validator report the node good.
- Why shouldn't you hard-pin the GPU Operator to a specific version number like `v25.3.2` in a runbook? **Answer:** Because the public release line moves fast — from 25.3.x, past 25.10.0/25.10.1 (which made CDI the default injection path), to the 26.3.x line (which added the `NVIDIADriver` CRD and faster same-version driver-pod restarts) within about a year. A hard-pinned version in documentation goes stale and can mislead you about which behaviors (like CDI-by-default) are actually active on the cluster you're debugging. Resolve the current version live with `helm search repo nvidia/gpu-operator -l` at install/study time instead.
- On CoreWeave's CKS, why might you never run `helm install nvidia/gpu-operator` yourself even though you know how? **Answer:** CoreWeave documents managing GPU driver/toolkit lifecycle on tenants' behalf as part of the managed platform, rather than letting tenants self-install the Operator against infrastructure the vendor already owns and upgrades fleet-wide. Self-installing would create two uncoordinated reconcilers fighting over the same nodes — a real operational hazard, not a hypothetical one.

## Connections & what's next

This lesson is the map that 04.2 immediately stress-tests: every failure family in the next lesson is "one of these operands, broken, and here's how the logs prove which one." The `ClusterPolicy`/Helm version discipline here also sets up 04.5 (driver lifecycle and fleet upgrades), where the same chain re-runs under a live upgrade instead of a first install. GFD's labels and DCGM's metrics, introduced here as "the cost/observability substrate," are the exact join your capstone (04.10) performs at scale.

Next: **[04.2 · Crash-loop diagnosis](02-crash-loop-diagnosis.md)** takes this healthy chain, breaks it on purpose in three different ways, and builds the diagnostic discipline — and the failure-mode log — that turns "the GPU node is broken" from a page you dread into a two-minute localization.

## References & further reading

**Primary sources**
- [NVIDIA/gpu-operator](https://github.com/NVIDIA/gpu-operator) — the Operator's own repository; read for the exact component list, `ClusterPolicy` model, and chart values, verified this session against the live README.
- [NVIDIA GPU Operator docs — Getting Started](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html) — canonical Helm install path and per-operand `ClusterPolicy` values. *Not independently fetched this session* (proxy-blocked domain) — verify field names against your resolved version at study time.
- [NVIDIA k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) — confirmed at v0.17.1 this session; the plugin whose registration this chain gates, useful for cross-checking env vars and behavior against the README directly.

**Real-world engineering blogs**
- [CoreWeave — About GPU Driver Management in CKS](https://docs.coreweave.com/docs/products/cks/nodes/gpu-driver-management/gpu-driver-management-cks) — a managed GPU cloud's documented policy of not letting tenants self-install the Operator. *Not independently fetched this session*; corroborated by a second independent search result and a dated CoreWeave changelog entry.

**Deeper dives**
- [NVIDIA GPU Operator docs — Release Notes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/release-notes.html) — the place to confirm exactly which version line (25.10.x CDI-default, 26.x `NVIDIADriver` CRD, or newer) is current before you pin a Helm `--version`. *Not independently fetched this session* (proxy-blocked domain) — treat the version note above as directionally correct and re-verify the exact current tag here at study time.
