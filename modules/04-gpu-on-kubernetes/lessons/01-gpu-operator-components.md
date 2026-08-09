---
lesson: "04.1"
title: "GPU Operator components and the init-container dependency chain"
module: "04"
concept: "GPU Operator components and the init-container dependency chain"
status: not-started
est_time: "8h"
artifacts: []
---

# 04.1 · GPU Operator components and the init-container dependency chain

> **Concept.** The GPU Operator is not one thing — it is a system of cooperating DaemonSets gated by init-container validations that pass a node from bare metal to "can schedule `nvidia.com/gpu`".
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Why this matters
This is the single most-probed operational skill in the module: when a GPU node "isn't working," the answer is almost always "one stage of the chain didn't pass, and everything downstream is stuck in Init." If you can name every operand, know what gates it, and read its logs, you can localize any Operator failure in minutes instead of thrashing across daemonsets. Every later lesson (crash-loop diagnosis, MIG rollout, cost attribution) assumes you can point at the exact pod that owns a given symptom.

## What's new here
Module 02 taught the **device-plugin gRPC API** (`ListAndWatch`, `Allocate`) and the **DRA object model**, and how kubelet advertises extended resources — the *mechanics* of how a GPU becomes `nvidia.com/gpu: 1`. Module 03 taught the **silicon**: the driver/kernel-module relationship, the CUDA execution model, and MIG partitioning. This lesson does **not** re-derive any of that. What it adds is the **operational integration layer**: the Operator is the thing that installs the driver, wires the container runtime, *starts* the device plugin you already understand, and — crucially — **orders** all of it with init-container gates so nothing runs before its prerequisite is proven healthy. You already know what a device plugin is; here you learn why it sits in `Init:0/1` for ten minutes and whose fault that is.

## Core notes

### The operands (what runs in `gpu-operator`)
A default install on a single GPU node produces roughly these pods:

| Pod / workload | Kind | Role |
|---|---|---|
| `gpu-operator-…` | Deployment | The controller. Watches the `ClusterPolicy` CR, reconciles which operands land on which nodes. |
| `gpu-operator-node-feature-discovery-master` / `-worker` | Deployment / DaemonSet | **NFD.** Worker inspects hardware and emits `feature.node.kubernetes.io/pci-10de.present=true` (`0x10de` = NVIDIA's PCI vendor ID). This label is the trigger for everything else. |
| `nvidia-driver-daemonset-…` | DaemonSet | Builds/loads the kernel modules (`nvidia`, `nvidia_uvm`, `nvidia_modeset`) and exposes the userspace driver under `/run/nvidia/driver`. |
| `nvidia-container-toolkit-daemonset-…` | DaemonSet | Installs the toolkit to `/usr/local/nvidia/toolkit` and rewrites **containerd's** config so a GPU runtime exists. |
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
Two operands exist mostly to feed the layer this course is really about. **GFD** stamps the node with machine-readable facts your scheduler *and* your cost operator will key off: `nvidia.com/gpu.product` (e.g. `NVIDIA-L4`), `nvidia.com/gpu.memory` (MiB), `nvidia.com/gpu.count`, and — under MIG — `nvidia.com/mig-<profile>.*`. **DCGM-Exporter** publishes Prometheus metrics on `:9400` that are the raw material of GPU cost/efficiency: `DCGM_FI_DEV_GPU_UTIL` (SM utilization %), `DCGM_FI_DEV_FB_USED` / `DCGM_FI_DEV_FB_FREE` (framebuffer memory), `DCGM_FI_DEV_POWER_USAGE` (watts), and `DCGM_FI_PROF_*` profiling counters. Critically for attribution, the exporter joins each metric to the consuming pod via the `kubernetes.io/…` / `pod`, `namespace`, `container` labels it derives from the device-to-pod mapping — this is exactly the join your capstone operator will lean on to turn "GPU was 12% utilized" into "*this team's* pod wasted 88% of an L40S." Note DCGM needs both the driver and toolkit healthy to read the hardware, which is why it sits downstream of the toolkit gate.

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
The chain is enforced not just at first install but on every driver change and reboot. A `helm upgrade` that bumps `driver.version` triggers `k8s-driver-manager` to cordon/drain GPU pods, unload the old modules, and rebuild — during which the toolkit/plugin/validator briefly drop back to `Init` by design. On a node reboot the kernel modules are gone until the driver DaemonSet reloads them, so a few minutes of `nvidia.com/gpu: 0` after reboot is expected, not an incident. Knowing this keeps you from "fixing" a cluster that is simply mid-convergence.

### Install (Helm)
```bash
# 1. Add and refresh the NVIDIA Helm repo
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia && helm repo update

# 2. Install into its own namespace, waiting for operands to converge
helm install --wait gpu-operator \
  -n gpu-operator --create-namespace \
  nvidia/gpu-operator \
  --version=v25.3.2
```
Version notes (this area is version-sensitive — always pin `--version`):
- **GPU Operator 25.3.x** is the current line; it defaults to a recent driver branch and NVIDIA Container Toolkit ≥ 1.17. Chart values you will reach for: `driver.version`, `driver.enabled=false` (if the host already has a driver), `toolkit.enabled=false` (pre-installed toolkit), and `mig.strategy`.
- If your nodes **already have the driver** (common on-prem with a golden image), set `--set driver.enabled=false`; the Operator then skips stage 2 and its `driver-ready` file is produced by the validator against the host driver.
- **DRA** (structured GPU allocation) is **GA in Kubernetes 1.34**; the Operator can run device-plugin mode or DRA mode. This lesson uses the classic device-plugin path — DRA is covered separately.

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
On a fresh single-node cluster with a **rented L4 / A10 / L40S** (kubeadm or k3s, containerd runtime):

1. Install the Operator with the pinned Helm command above.
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

**Acceptance:** (a) a `cuda-vectoradd` pod that reaches `Completed` with `Test PASSED`; (b) a written component map naming all operands, each pod's role, and its position/gate in the dependency chain (note that `nvidia-mig-manager` is *absent* on L4/A10/L40S and say why).

## Self-check
**(a) Put the init-container dependency chain in order and say what each stage gates.**
**Answer:** NFD → Driver → Container-Toolkit → Device-Plugin → GFD, with DCGM-Exporter (and MIG-Manager on capable GPUs) and the operator-validator hanging off the toolkit. NFD's `pci-10de.present` label gates scheduling of every operand; the Driver DS writes `driver-ready` and gates the toolkit; the Toolkit DS writes `toolkit-ready` and gates the device plugin, GFD, and DCGM; the Device Plugin writes `plugin-ready` and makes `nvidia.com/gpu` allocatable; the operator-validator runs driver→toolkit→cuda→plugin validations and declares the node truly ready. Gating is enforced by init containers blocking on `*-ready` sentinel files in `/run/nvidia/validations/`.

**(b) Which component configures the container runtime, and what breaks if it fails?**
**Answer:** The **container-toolkit DaemonSet**. It runs `nvidia-ctk runtime configure --runtime=containerd`, which patches `/etc/containerd/config.toml` to add an `nvidia` runtime (binary `/usr/local/nvidia/toolkit/nvidia-container-runtime`) and sets it as default, then restarts containerd. If it fails, `toolkit-ready` is never written, so the device plugin, GFD, DCGM, and validator all stay in `Init` — and even if a GPU pod were scheduled, containerd could not inject the GPU (missing runtime / `nvidia-container-runtime-hook`), so the container sees no device.

**(c) What does the operator-validator actually validate before marking the node ready?**
**Answer:** Four things, in order, each in its own init container: **driver-validation** (kernel modules loaded, `nvidia-smi` works via the driver), **toolkit-validation** (the NVIDIA container runtime is installed and wired into containerd), **cuda-validation** (it launches the `nvidia-cuda-validator` pod that runs a real `vectorAdd` kernel end-to-end — proving CUDA actually executes, not just that a resource is advertised), and **plugin-validation** (`nvidia.com/gpu` is registered and allocatable > 0). Only when all four pass does the validator report the node good.

## Resources
1. **NVIDIA GPU Operator — Getting Started** — https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html — *deep.* Canonical Helm commands, prerequisites, and per-operand `ClusterPolicy` values; the source of truth for the install you just did. Version-pin against the matching `/25.3/` path.
2. **kubenatives — NVIDIA GPU Operator: 8 Components Explained** — https://www.kubenatives.com/p/nvidia-gpu-operator-kubernetes — *skim.* A practitioner's plain-language map of the operands; good for cementing the component/role table.
3. **DeepWiki — NVIDIA/gpu-operator overview** — https://deepwiki.com/NVIDIA/gpu-operator/1-overview — *skim.* Reverse-engineered architecture view of the operator and its reconciliation; useful when you need to reason about the controller rather than the operands.
