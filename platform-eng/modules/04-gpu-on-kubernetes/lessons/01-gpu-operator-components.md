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
sources: 12
---

# 04.1 · GPU Operator components and the init-container dependency chain

> **Concept.** The GPU Operator is not one thing — it is a system of cooperating DaemonSets gated by init-container validations that pass a node from bare metal to "can schedule `nvidia.com/gpu`".
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Module 03 gave you hardware literacy: the driver/kernel-module relationship, the CUDA execution model, MIG as a silicon-level partition. Module 02 gave you the device-plugin gRPC API and the DRA object model — the *mechanics* of how a GPU becomes a schedulable Kubernetes resource. Both were taught as concepts you reason about in isolation. This module is where they stop being isolated concepts and become **one running system you operate** — the GPU Operator, plus everything downstream of it. Lesson 1 builds the map: every operand it deploys, the label chain that decides where each one lands, the barrier files that order them, and why a fresh install spends its first several minutes sitting in `Init:0/1`. Lesson 2 immediately breaks that map on purpose, because a map you haven't used to diagnose a failure isn't actually learned.

Everything in this lesson is checked against **GPU Operator v26.3.3** — the newest tag in `NVIDIA/gpu-operator` at the time of writing — reading the chart's `values.yaml`, the DaemonSet assets under `assets/`, and the validator source under `cmd/nvidia-validator/`. Where a name, path, default or log string appears below, it came out of that tree, not out of a blog post.

## Why this matters

This module's README calls itself "the core operational module, the one a hiring manager probes hardest" — and this lesson is the anchor of that probe. CoreWeave's *Sr Eng, Kubernetes Platforms* posting asks for engineers who build "controllers/operators that automate infrastructure testing" and maintain "visibility into system metrics/performance/health" across "tens of thousands of kubelets." NVIDIA's *Sr DevOps, Platform Eng* posting asks candidates to "troubleshoot… GPU-accelerated servers to minimize downtime." Neither of those is answerable from a device-plugin API diagram — they're answerable only if you can point at the exact pod that owns a symptom, on a live cluster, under time pressure.

The concrete cost of not knowing this: a GPU node that "isn't working" is, in the overwhelming majority of real incidents, **one stage of this chain that didn't pass while everything downstream sits blocked**. An engineer who doesn't know the chain exists debugs the *symptom* — a Pending workload pod — and burns an hour restarting things that were never broken. An engineer who knows the chain reads one `kubectl get pods -n gpu-operator`, walks to the first non-`Running` stage, and has localized the incident before the first engineer finishes typing `kubectl describe pod`.

Put a number on that hour. An 8×H100 node rents for roughly $20–30/hr on a mid-tier GPU cloud (rates move constantly; treat this as an order of magnitude, not a quote). One undiagnosed hour on one node is $20–30 of hardware you paid for and could not schedule. Multiply by the eight-node inference tier that all shares one bad driver image and you are at $160–240/hr with the invoice still running. The chain in this lesson is what turns that hour into two minutes.

## What's new here (calibration)

This module's README is explicit that 02/02b/03 already own the theory, and this module is the **operational integration layer** on top of it. Concretely, this lesson does **not** re-teach:

- The device-plugin gRPC API (`ListAndWatch`, `Allocate`) or the DRA object model — Module 02 owns the *mechanics* of how a GPU becomes `nvidia.com/gpu: 1` or a `ResourceClaim`. Lesson 04.3 revisits that API from the *attribution* side.
- Topology Manager or NUMA/PCIe topology reasoning — Module 02b's territory.
- The driver/kernel-module relationship or the CUDA execution model at the silicon level — Module 03's territory.

What this lesson adds instead:

- **The full operand inventory** as a table you can read like a parts list: pod name, container names, the host paths it mounts, and the specific thing that breaks when it is absent.
- **The label chain**, from the NFD PCI source's actual label-construction algorithm through `nvidia.com/gpu.present` to the eight `nvidia.com/gpu.deploy.*` switches that are the operands' `nodeSelector`s.
- **The barrier mechanism, precisely** — the real file names in `/run/nvidia/validations`, *which component writes each one* (this is where most write-ups, including the previous version of this lesson, get it wrong), and the two distinct waiting idioms in use.
- **What the driver and toolkit containers actually do**, step by step out of their entrypoint scripts, with the real timeouts.
- **Rollout arithmetic**: how long a driver change takes across N nodes at a given `maxUnavailable`, and what that costs in GPU-hours.

## Core concepts

### 1. The problem: five installs that must happen in one order

Start from what you'd do by hand on one bare-metal GPU box to make a CUDA container work. Five things, and they are not independent:

1. Build and load the NVIDIA kernel modules (`nvidia`, `nvidia_uvm`, `nvidia_modeset`) against the *running* kernel, and get the matching user-space driver (`libcuda.so`, `libnvidia-ml.so`, `nvidia-smi`) onto the filesystem.
2. Install the NVIDIA Container Toolkit, so that some runtime binary knows how to bind-mount that driver into a container.
3. Rewrite the container runtime's config so it *has* such a runtime, and reload the runtime so the config takes effect.
4. Run the device plugin so the kubelet learns there are GPUs and advertises `nvidia.com/gpu` to the scheduler.
5. Label the node with what kind of GPU it is, so schedulers and cost systems can tell an L4 from an H100.

Every arrow here is a hard ordering constraint. Step 2 is pointless before step 1, because the toolkit's job is to inject a driver that must already exist. Step 3 must follow step 2 or containerd points at a binary that isn't on disk. Step 4 must follow step 1, because the device plugin enumerates GPUs through NVML — which is part of the driver. And step 4 must *also* follow step 3, because the plugin's own validation runs a real GPU container.

Doing that once, by hand, on one node, is an afternoon. Four things break it at fleet scale, and they are the actual design pressure on the Operator. **Kernel drift:** modules are built against one exact kernel version, so a node that reboots into a new kernel from an unattended upgrade loses the modules that worked yesterday — something must rebuild them per node, per boot. **Node churn:** an autoscaler brings up a GPU node at 03:00 and nobody is there to run the afternoon's steps. **Heterogeneity:** half the fleet is L4s and half is H100s with MIG, so one operand must land on some nodes and not others. **Per-node ordering:** the five-step order has to hold on 200 nodes independently, with each node's stage 3 waiting on *its own* stage 1.

The GPU Operator's answer is: run each step as a privileged DaemonSet, decide placement with node labels, and enforce the ordering with **barrier files on a shared host path**, so each stage's init container blocks until the previous stage has written its marker on *that node*. That last mechanism is the load-bearing idea of the lesson, and §5 takes it apart.

### 2. The control plane: `ClusterPolicy` and the reconcile order

There is exactly one controller Deployment, `gpu-operator`, and one cluster-scoped custom resource it watches: **`ClusterPolicy`** (`clusterpolicies.nvidia.com`, API group `nvidia.com/v1`). The Helm values you pass are templated into a `ClusterPolicy` object; the controller reads it and materializes the DaemonSets.

**When you need to know "what did this cluster actually ask for," read the `ClusterPolicy`, not the Helm values file.** Drift between the two is real: anyone with RBAC can `kubectl edit clusterpolicy cluster-policy`, and a subsequent `helm upgrade --reuse-values` will not necessarily notice.

```bash
kubectl get clusterpolicy                       # NAME             STATUS   AGE
kubectl get clusterpolicy cluster-policy -o yaml | less
kubectl get clusterpolicy cluster-policy -o jsonpath='{.status.state}{"\n"}'
```

`.status` is deliberately coarse. It carries a `state` field whose CRD validation enum is exactly **`ignored` | `ready` | `notReady`** (there is also a `disabled` constant in the Go source that is not in the printed enum), plus a `namespace` and a list of `metav1.Condition`s. `ready` means every enabled operand the controller deployed reports its pods ready. `notReady` means at least one does not — it will not tell you *which*, which is why the rest of this lesson exists.

The controller's reconcile is a linear walk over a fixed list of "states," each of which is a directory of manifests baked into the operator image at `/opt/gpu-operator/`. From `controllers/state_manager.go` in v26.3.3, in source order:

```
pre-requisites            → RuntimeClasses: nvidia, nvidia-cdi, nvidia-legacy
state-operator-metrics
state-driver              → nvidia-driver-daemonset
state-container-toolkit   → nvidia-container-toolkit-daemonset
state-operator-validation → nvidia-operator-validator
state-device-plugin       → nvidia-device-plugin-daemonset
state-mps-control-daemon
state-dcgm                → nvidia-dcgm            (disabled by default)
state-dcgm-exporter       → nvidia-dcgm-exporter
gpu-feature-discovery     → gpu-feature-discovery
state-mig-manager         → nvidia-mig-manager
state-node-status-exporter                        (disabled by default)
… then the sandbox/vGPU/Kata/CC states, all inactive unless sandboxWorkloads.enabled
```

Two things to notice, because they trip people up.

First, **this list is the order in which objects are *applied*, not the order in which pods become ready.** Applying a DaemonSet is instantaneous; its pods then converge on their own schedule, gated by the barrier files. `state-operator-validation` being applied *before* `state-device-plugin` is why the validator's `plugin-validation` init container has a built-in retry loop (§9) — the plugin it is validating gets created a moment later.

Second, the operator does not shell out to `kubectl`. It reconciles typed objects and re-applies on drift. Delete `nvidia-driver-daemonset` by hand and it comes back within a reconcile interval. That is a feature when someone fat-fingers a delete and an annoyance when you're trying to keep an operand off a node — for which the supported lever is a label, not a deletion (§4).

The controller exposes its own Prometheus metrics under the namespace `gpu_operator`, and three of them are worth an alert:

| Metric | Type | What it tells you |
|---|---|---|
| `gpu_operator_gpu_nodes_total` | gauge | Number of nodes the operator considers GPU nodes. A drop here means NFD stopped labelling. |
| `gpu_operator_reconciliation_status` | gauge | Coded status: success / operands-not-ready / ClusterPolicy unavailable / operator error. |
| `gpu_operator_reconciliation_failed_total` | counter | Reconcile failures. Non-zero and rising = the controller itself is unhappy. |
| `gpu_operator_nodes_upgrades_in_progress` / `_done` / `_failed` / `_pending` / `_available` | gauge | The driver-upgrade state machine's census (lesson 04.5's territory; `_failed > 0` is a page). |

### 3. The operand inventory

This is the parts list. Container names matter because `kubectl logs` needs `-c <container>` on a multi-container pod, and every one of these pods has more than one container. Host mounts matter because they are how these pods talk to each other — the "shared bus" of the whole system is a handful of host directories, not a network.

| Pod (workload) | Kind | Containers | Key host mounts | What it does | What breaks if it's missing |
|---|---|---|---|---|---|
| `gpu-operator-…` | Deployment | `gpu-operator` | — | Reconciles `ClusterPolicy` → operands; manages `nvidia.com/gpu.*` node labels; runs the driver-upgrade controller. | Nothing changes state. Existing operands keep running; no new node ever gets set up, and label/CR edits are ignored. |
| `gpu-operator-node-feature-discovery-master` | Deployment | `master` | — | Turns `NodeFeature` objects into node labels; enforces the allowed label namespaces. | No `feature.node.kubernetes.io/*` labels appear → the operator never recognises any node as a GPU node → **nothing at all deploys**. |
| `gpu-operator-node-feature-discovery-worker` | DaemonSet | `worker` | `/sys`, `/proc`, `/etc/kubernetes/node-feature-discovery/features.d` | Walks PCI/CPU/kernel/OS sources and reports features. Emits `feature.node.kubernetes.io/pci-10de.present=true` on NVIDIA hardware. | Same as above, per node. |
| `gpu-operator-node-feature-discovery-gc` | Deployment | `gc` | — | Garbage-collects `NodeFeature`/`NodeResourceTopology` objects for nodes that are gone. | Stale objects accumulate. Harmless short-term. |
| `nvidia-driver-daemonset-…` | DaemonSet | `nvidia-driver-ctr` (+ optional `nvidia-peermem-ctr`, `nvidia-fs-ctr`, `nvidia-gdrcopy-ctr`); init: `k8s-driver-manager` | `/run/nvidia` (**Bidirectional**), `/`, `/sys`, `/var/log`, `/lib/firmware`, `/sys/module/firmware_class/parameters/path` | Builds/loads the kernel modules; publishes the whole user-space driver tree at `/run/nvidia/driver`. | No modules, no `/dev/nvidia*`, no NVML. **Everything downstream stalls.** |
| `nvidia-container-toolkit-daemonset-…` | DaemonSet | `nvidia-container-toolkit-ctr`; init: `driver-validation` | `/usr/local/nvidia`, `/run/nvidia/toolkit`, `/etc/containerd` (via config mount), `/var/run/cdi`, `/run/nvidia/driver`, `/` | Installs the toolkit binaries; writes the containerd/CRI-O runtime config; generates CDI specs. | Containerd has no `nvidia` runtime and no CDI specs → GPU containers get plain `runc` and see no devices. |
| `nvidia-operator-validator-…` | DaemonSet | `nvidia-operator-validator`; init: `driver-validation`, `toolkit-validation`, `cuda-validation`, `plugin-validation` | `/run/nvidia/validations` (**Bidirectional**), `/run/nvidia/driver`, `/dev/char`, `/` | Runs the four validations **and writes the barrier files** the other operands wait on. Also creates `/dev/char` symlinks. | `toolkit-ready`, `cuda-ready`, `plugin-ready` are never written → device plugin, GFD, DCGM-exporter and MIG-manager all sit in `Init` forever. |
| `nvidia-device-plugin-daemonset-…` | DaemonSet | `nvidia-device-plugin`, `config-manager`; init: `toolkit-validation`, `config-manager-init` | `/var/lib/kubelet/device-plugins`, `/run/nvidia/driver`, `/var/run/cdi`, `/run/nvidia/mps`, `/` | Enumerates GPUs via NVML, registers `nvidia.com/gpu` with the kubelet, serves `Allocate`. | `nvidia.com/gpu` capacity stays absent/0 → every GPU pod is `Pending` with `Insufficient nvidia.com/gpu`. |
| `gpu-feature-discovery-…` | DaemonSet | `gpu-feature-discovery`, `config-manager`; init: `toolkit-validation`, `config-manager-init` | `/sys`, `/etc/kubernetes/node-feature-discovery/features.d` | Writes GPU facts (product, memory, count, MIG geometry, driver version) as a feature file NFD turns into labels. | Scheduling still works; but `nvidia.com/gpu.product` and friends are gone, so node selectors and every cost query keyed on GPU model break. |
| `nvidia-dcgm-exporter-…` | DaemonSet | `nvidia-dcgm-exporter`; init: `toolkit-validation` | `/run/nvidia`, **`/var/lib/kubelet/pod-resources`** | Reads DCGM fields, joins them to pods via the pod-resources API, serves Prometheus on `:9400`. | No GPU telemetry. Your cost/efficiency layer goes blind — this is the operand lesson 04.3 and the capstone are built on. |
| `nvidia-dcgm-…` | DaemonSet | `nvidia-dcgm` | `/run/nvidia` | Standalone `nv-hostengine`. **`dcgm.enabled=false` by default** — the exporter embeds its own hostengine instead. | Nothing, unless you deliberately enabled it and something else expects port 5555. |
| `nvidia-mig-manager-…` | DaemonSet | `nvidia-mig-manager`; init: `toolkit-validation` | `/sys`, `/`, `/run/nvidia/validations`, `/run/nvidia/driver`, `/var/run/cdi`, MIG config + GPU-clients ConfigMaps | Applies MIG geometry with `nvidia-mig-parted`, draining GPU clients first. **Only scheduled on MIG-capable GPUs.** | On A100/H100/H200-class nodes, `nvidia.com/mig.config` changes are ignored. On L4/A10/L40S the pod is *supposed* to be absent. |
| `nvidia-cuda-validator-…` | Pod (`generateName`, `restartPolicy: OnFailure`) | `nvidia-cuda-validator`; init: `cuda-validation` | — | Created by the validator. The init container runs a real `vectorAdd`; the main container just echoes success and exits. | Without it, `cuda-ready` is never written and the validator's own init sequence never finishes. |
| `nvidia-device-plugin-validator-…` | Pod (`generateName`, `OnFailure`) | `nvidia-device-plugin-validator`; init: `plugin-validation` | — | Same shape, but its init container *requests* `nvidia.com/gpu: 1` — an end-to-end scheduling test. Created only when the validator runs with `WITH_WORKLOAD=true` (not the default). | Nothing by default; when enabled, its absence means the plugin path was never exercised through the scheduler. |
| `nvidia-node-status-exporter-…` | DaemonSet | `nvidia-node-status-exporter` | `/run/nvidia/validations` | Exports the barrier files as Prometheus gauges (`gpu_operator_node_driver_ready`, `…_toolkit_ready`, `…_plugin_ready`, `…_cuda_ready`). **Disabled by default.** | You lose the cheapest possible fleet-wide "which stage is stuck on which node" dashboard. Worth enabling. |
| `nvidia-mps-control-daemon-…` | DaemonSet | `mps-control-daemon-ctr` | `/run/nvidia/mps` | Runs the CUDA MPS control daemon. Deployed only when the device-plugin config selects MPS sharing. | MPS sharing silently doesn't work. Lesson 04.8's territory. |
| `nvidia-sandbox-device-plugin`, `nvidia-vgpu-manager`, `nvidia-vgpu-device-manager`, `nvidia-sandbox-validator`, `nvidia-vfio-manager`, `nvidia-kata-manager`, `nvidia-cc-manager` | DaemonSets | various | various | The KubeVirt/Kata/vGPU/Confidential-Computing branch. Gated behind `sandboxWorkloads.enabled=false` and per-node `nvidia.com/gpu.workload.config`. | Nothing, on a normal container-workload cluster. **Knowing these exist stops you from panicking when you see them in a `ClusterPolicy` dump.** |

The versions those images are pinned to in the v26.3.3 chart — useful because "which driver branch is this cluster on" is the first question in half of all GPU incidents:

| Component | Image | Version in v26.3.3 |
|---|---|---|
| Driver | `nvcr.io/nvidia/driver` | `580.126.20` |
| Container toolkit | `nvcr.io/nvidia/k8s/container-toolkit` | `v1.19.1` |
| Device plugin | `nvcr.io/nvidia/k8s-device-plugin` | `v0.19.3` |
| GFD | `nvcr.io/nvidia/k8s-device-plugin` (same image, different entrypoint) | `v0.19.3` |
| DCGM exporter | `nvcr.io/nvidia/k8s/dcgm-exporter` | `4.5.3-4.8.2-distroless` |
| Standalone DCGM | `nvcr.io/nvidia/cloud-native/dcgm` | `4.5.2-1-ubuntu22.04` |
| MIG manager | `nvcr.io/nvidia/cloud-native/k8s-mig-manager` | `v0.14.2` |
| Driver manager | `nvcr.io/nvidia/cloud-native/k8s-driver-manager` | `v0.11.0` |
| NFD (Helm subchart) | `registry.k8s.io/nfd/charts/node-feature-discovery` | chart `0.18.3` |

Read those as a snapshot with a date on it, not as a spec. The point of the table is the *shape*: nine independently-versioned pieces, one of which (the driver) has fleet-wide blast radius. Resolve the real numbers on your cluster with:

```bash
kubectl -n gpu-operator get ds -o custom-columns=\
'NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image'
```

### 4. The label chain: how the operator decides what runs where

Nothing above is scheduled by magic. Every operand DaemonSet has a one-key `nodeSelector`, and the operator's job is to put the right keys on the right nodes. There are three links in the chain and each one is a separate failure point.

```
  ┌───────────────────────────────────────────────────────────────────────────┐
  │ LINK 1 · NFD worker reads PCI config space                                │
  │                                                                           │
  │   for each PCI device:                                                    │
  │     class = attrs["class"]                    e.g. "0302" (3D controller) │
  │     if class starts with any of deviceClassWhitelist:                     │
  │        label = join(deviceLabelFields, "_") + ".present"                  │
  │                                                                           │
  │   NFD upstream default:  classWhitelist=["03","0b40","12"]                │
  │                          labelFields=["class","vendor"]                   │
  │       ⇒ feature.node.kubernetes.io/pci-0302_10de.present=true             │
  │                                                                           │
  │   GPU-Operator chart override (deployments/gpu-operator/values.yaml):     │
  │                          classWhitelist=["02","0200","0207","0300","0302"]│
  │                          labelFields=["vendor"]         ◀── vendor ONLY   │
  │       ⇒ feature.node.kubernetes.io/pci-10de.present=true                  │
  └───────────────────────────────────┬───────────────────────────────────────┘
                                      │  the operator accepts ANY of the three
                                      │  spellings (state_manager.go gpuNodeLabels):
                                      │    pci-10de.present
                                      │    pci-0302_10de.present
                                      │    pci-0300_10de.present
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────┐
  │ LINK 2 · operator stamps the common label + the workload config           │
  │            nvidia.com/gpu.present = "true"                                │
  │            nvidia.com/gpu.workload.config = "container"  (default)         │
  └───────────────────────────────────┬───────────────────────────────────────┘
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────┐
  │ LINK 3 · operator fans out the eight per-operand switches                 │
  │   (gpuStateLabels[ "container" ], all set to "true")                      │
  │                                                                           │
  │     nvidia.com/gpu.deploy.driver                 → nvidia-driver-daemonset│
  │     nvidia.com/gpu.deploy.container-toolkit      → toolkit DS             │
  │     nvidia.com/gpu.deploy.operator-validator     → validator DS           │
  │     nvidia.com/gpu.deploy.device-plugin          → device-plugin DS       │
  │     nvidia.com/gpu.deploy.gpu-feature-discovery  → GFD DS                 │
  │     nvidia.com/gpu.deploy.dcgm                   → nvidia-dcgm DS         │
  │     nvidia.com/gpu.deploy.dcgm-exporter          → dcgm-exporter DS       │
  │     nvidia.com/gpu.deploy.node-status-exporter   → node-status-exporter   │
  │                                                                           │
  │   + conditionally, if the node looks MIG-capable:                         │
  │     nvidia.com/gpu.deploy.mig-manager            → mig-manager DS         │
  │                                                                           │
  │   + master kill switch, honoured before everything else:                  │
  │     nvidia.com/gpu.deploy.operands = "false"  ⇒ ALL deploy labels removed │
  └───────────────────────────────────────────────────────────────────────────┘
```

Four consequences you should be able to state cold.

**(a) No NFD label, no anything.** This is the single most common root cause of "the operator ignored my node." The controller's `hasGPULabels()` returns false, no `nvidia.com/gpu.present` is stamped, no deploy labels are set, and every operand's `nodeSelector` fails to match. The node looks completely untouched, which reads as "the operator is broken" when it is in fact "the operator was never told this is a GPU node." Check it first, always:

```bash
kubectl get node <node> -o json | jq -r '.metadata.labels
  | to_entries[] | select(.key|test("pci-10de|pci-0302|pci-0300|nvidia.com"))
  | "\(.key)=\(.value)"'
```

**(b) The chart narrows `deviceLabelFields` to `vendor` on purpose.** Upstream NFD labels by class *and* vendor, which gives you a different label per PCI class. The Operator only cares "is there NVIDIA silicon here," so it collapses the label to the vendor ID. `10de` is NVIDIA's PCI vendor ID. The chart *widens* the class whitelist at the same time (adding `02`/`0200`/`0207` — network controllers — because the same NFD install is used to detect Mellanox NICs for GPUDirect RDMA, labelled `pci-15b3.present`). If you bring your own NFD with upstream defaults, you get `pci-0302_10de.present` instead, and the operator handles it — which is exactly why `gpuNodeLabels` in the source has three entries and not one.

**(c) The NFD master must allow the `nvidia.com` label namespace.** NFD refuses to publish labels outside its own namespace unless told otherwise. The chart sets `master.config.extraLabelNs: ["nvidia.com"]`. Without it, GFD's feature file is read and then silently dropped — you get `pci-10de.present` but no `nvidia.com/gpu.product`. That's a genuinely confusing half-broken state: scheduling works, everything model-specific doesn't.

**(d) `nvidia.com/gpu.deploy.<operand>=false` is the surgical off switch.** Set it and the operand's pod is evicted from that node within a reconcile; the operator honours an existing value rather than overwriting it. This is the supported way to keep one operand off one node — and, as lesson 04.2 uses it, a clean way to break one on purpose. `nvidia.com/gpu.deploy.operands=false` is the whole-node version.

**On MIG-manager placement**, since "why isn't the MIG manager running?" is a recurring question: the operator adds `gpu.deploy.mig-manager` only when `hasMIGCapableGPU()` is true. That function checks, in order: is this a vGPU node (`nvidia.com/vgpu.host-driver-version` present → not MIG-capable); does `nvidia.com/mig.capable=true` exist (GFD sets it, and this is the authoritative path); and only as a *fallback*, does `nvidia.com/gpu.product` contain the substring `h100`, `a100`, or `a30`. So on an L4, A10 or L40S the MIG manager is correctly absent. And note the fallback list is narrower than the set of MIG-capable silicon — on newer parts you are relying on GFD's `mig.capable` label, which is another reason link 3 of the label chain matters.

### 5. The barrier mechanism — where the ordering actually lives

Now the load-bearing part. Operands do not merely *depend* on each other logically; the ordering is enforced by files in one host directory:

```
/run/nvidia/validations/
```

mounted into the relevant pods, with `mountPropagation: Bidirectional` on the writers and plain or `HostToContainer` on the readers. The names come from `cmd/nvidia-validator/main.go`, where `defaultStatusPath = "/run/nvidia/validations"`:

| File | Written by | Meaning |
|---|---|---|
| `.driver-ctr-ready` | the driver container's **startupProbe script** (`nvidia-driver-startup-probe` ConfigMap) | `/sys/module/nvidia/refcnt` exists **and** `nvidia-smi` succeeded inside the driver container. Contents are three `KEY: value` lines recording `GDRCOPY_ENABLED`, `GDS_ENABLED`, `GPU_DIRECT_RDMA_ENABLED`. |
| `.driver-daemons-status` | the driver container entrypoint | Status of the optional driver-side daemons. |
| `driver-ready` | a **`driver-validation` init container** running `nvidia-validator -c driver` — one lives in the toolkit DS, one in the validator DS | The driver is usable: either a host driver passes `chroot /host nvidia-smi`, or the containerized driver's libraries and `nvidia-smi` are found under `/run/nvidia/driver` and run with `LD_PRELOAD` pointed at `libnvidia-ml.so.1`. **Contents are sourced as shell env by downstream entrypoints** — `IS_HOST_DRIVER`, `NVIDIA_DRIVER_ROOT`, `DRIVER_ROOT_CTR_PATH`, `NVIDIA_DEV_ROOT`, `DEV_ROOT_CTR_PATH`. |
| `toolkit-ready` | the **validator DS's `toolkit-validation` init container** (`nvidia-validator -c toolkit`) | `nvidia-smi` ran *inside a container that the toolkit was supposed to inject the driver into*. This is an end-to-end test of the runtime wiring, not a file check. |
| `cuda-ready` | the validator DS's `cuda-validation` init container | A real `vectorAdd` CUDA kernel completed in a separate pod. |
| `plugin-ready` | the validator DS's `plugin-validation` init container | The node's `status.capacity` contains `nvidia.com/gpu` or `nvidia.com/mig-*` with quantity ≥ 1. |
| `workload-type` | validator | Records the resolved `nvidia.com/gpu.workload.config` for the node. |
| `nvidia-fs-ready`, `gdrcopy-ready`, `nvidia-peermem-ready`, `mofed-ready`, `vfio-pci-ready`, `vgpu-manager-ready`, `vgpu-devices-ready`, `cc-manager-ready` | validator, per component | The optional branches. Present only when the corresponding feature is enabled. |

**Correct this if you have it wrong (the previous version of this lesson did): the toolkit DaemonSet does not write `toolkit-ready`.** It writes nothing. `toolkit-ready` is written by the *operator-validator's* second init container. The toolkit DS's contribution to the barrier set is `driver-ready`, written by its *first* init container. This matters operationally: if the device plugin is stuck waiting on `toolkit-ready`, the pod you go read is `nvidia-operator-validator`, not `nvidia-container-toolkit-daemonset`.

Two distinct waiting idioms are in use, and telling them apart tells you what a stuck pod is actually doing.

**Idiom A — a shell loop on a file.** Used by the device plugin, GFD, DCGM-exporter and MIG-manager init containers. Verbatim from `assets/state-device-plugin/0500_daemonset.yaml`:

```yaml
initContainers:
- name: toolkit-validation
  command: ['sh', '-c']
  args: ["until [ -f /run/nvidia/validations/toolkit-ready ]; do echo waiting for nvidia container stack to be setup; sleep 5; done"]
```

So a pod in `Init:0/2` on the device plugin is printing, every five seconds, forever:

```
waiting for nvidia container stack to be setup
waiting for nvidia container stack to be setup
```

The MIG manager's copy of the same init container says **`waiting for nvidia container toolkit to be setup`** instead — a one-word difference that is genuinely useful, because seeing which phrasing scrolls past tells you which operand's logs you're in.

The toolkit and device-plugin *main* containers do the same thing again on `driver-ready`, from their entrypoint ConfigMaps:

```sh
until [ -f /run/nvidia/validations/driver-ready ]
do
  echo "waiting for the driver validations to be ready..."
  sleep 5
done
set -o allexport
. /run/nvidia/validations/driver-ready     # imports NVIDIA_DRIVER_ROOT etc.
```

That second wait is not redundant. The init container proves the *barrier* is open; the main container needs the barrier file's *contents* as environment.

**Idiom B — a retrying validation.** Used by the `driver-validation` init containers, via `WITH_WAIT=true`. `runCommandWithWait()` is an unbounded `for {}` loop: run the command, and on failure log `error running command: …`, print `command failed, retrying after 5 seconds`, sleep, repeat. There is no give-up. A driver that never comes up produces an init container that runs forever, not one that fails — so `Init:0/1` with no restarts and a growing age is the expected appearance of "the driver is broken," not a crash loop.

The `toolkit-validation`, `cuda-validation` and `plugin-validation` init containers in the validator run with `WITH_WAIT=false`, so they *do* fail and the pod restarts with backoff. `plugin-validation` has its own internal retry instead: 30 attempts, 5 s apart — 150 s — logging `GPU resources are not yet discovered by the node, retry: N` before giving up with `GPU resources are not discovered by the node`.

Finally, a detail that explains a real behaviour: the validator's main container has a `preStop` hook

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "rm -f /run/nvidia/validations/*-ready"]
```

**Deleting the operator-validator pod tears down every barrier on that node.** Every downstream operand drops back into `Init` and waits for the validator to come back and re-open them. That is by design — it is how the system re-validates after a driver change — but it means "just delete the validator pod" is a much bigger hammer than it looks.

### 6. The startup sequence, on a timeline

Here is what actually happens on a fresh single-GPU node, with the barrier edges drawn. Times are representative of a first install with a driver image pull and a module build on a small cloud node; they will differ on your hardware, and the driver stage is the one with real variance.

```
 t=0s     helm install → ClusterPolicy created → controller reconciles the state list
          RuntimeClasses nvidia / nvidia-cdi / nvidia-legacy created (pre-requisites)

 t≈5s     NFD worker DaemonSet lands on the node
 t≈20s    NFD master publishes  feature.node.kubernetes.io/pci-10de.present=true
 t≈25s    operator sees the label →  nvidia.com/gpu.present=true
                                 →  nvidia.com/gpu.workload.config=container
                                 →  8 × nvidia.com/gpu.deploy.* = true
          ── every operand DaemonSet's nodeSelector now matches. Pods get scheduled. ──

 ═════════════════════════ ALL OF THESE START AT ONCE ═════════════════════════
 │
 │ nvidia-driver-daemonset            nvidia-container-toolkit-ds   nvidia-operator-validator
 │ ┌──────────────────────────┐       ┌────────────────────────┐    ┌──────────────────────┐
 │ │init k8s-driver-manager   │       │init driver-validation  │    │init driver-validation│
 │ │  uninstall_driver:       │       │  nvidia-validator -c   │    │  (identical)         │
 │ │  is a driver loaded?     │       │  driver  WITH_WAIT=true│    │  WITH_WAIT=true      │
 │ │  is it the right config? │       │                        │    │                      │
 │ │  → nothing to do, exit 0 │       │  loops:                │    │  loops               │
 │ └───────────┬──────────────┘       │  "command failed,      │    │                      │
 │             ▼                      │   retrying after 5s"   │    │                      │
 │ ┌──────────────────────────┐       └───────────┬────────────┘    └──────────┬───────────┘
 │ │nvidia-driver-ctr         │                   │                            │
 │ │ 1 pull image (~2 GB)     │                   │  ◀── BARRIER: driver-ready │
 │ │ 2 "Resolving Linux       │                   │      (whichever validator  │
 │ │    kernel version..."    │                   │       wins the race writes  │
 │ │ 3 apt install headers    │                   │       it; both then pass)   │
 │ │ 4 build modules          │                   │                            │
 │ │ 5 "Mounting NVIDIA       │                   │                            │
 │ │    driver rootfs..."     │                   │                            │
 │ │    mount --rbind /  →    │                   │                            │
 │ │      /run/nvidia/driver  │                   │                            │
 │ │ 6 "Installing userspace  │                   │                            │
 │ │    components..."        │                   │                            │
 │ │ 7 "Loading NVIDIA driver │                   │                            │
 │ │    kernel modules..."    │                   │                            │
 │ │   modprobe nvidia,       │                   │                            │
 │ │   nvidia-uvm,            │                   │                            │
 │ │   nvidia-modeset         │                   │                            │
 │ │ 8 "Done, now waiting     │                   │                            │
 │ │    for signal"           │                   │                            │
 │ │                          │                   │                            │
 │ │ startupProbe (from t+60s,│                   │                            │
 │ │ every 10s, ≤120 tries):  │                   │                            │
 │ │   refcnt? nvidia-smi?    │                   │                            │
 │ │   → write                │                   │                            │
 │ │     .driver-ctr-ready ───┼──▶ READY          │                            │
 │ └──────────────────────────┘                   │                            │
 │        t ≈ 3–8 min                             │                            │
 │                                    ┌───────────▼────────────┐               │
 │                                    │nvidia-container-       │               │
 │                                    │  toolkit-ctr           │               │
 │                                    │  waits on driver-ready │               │
 │                                    │  sources it as env     │               │
 │                                    │  sleep 5  (containerd  │               │
 │                                    │    SIGTERM workaround) │               │
 │                                    │  exec nvidia-toolkit:  │               │
 │                                    │   install → /usr/local/│               │
 │                                    │     nvidia/toolkit     │               │
 │                                    │   write containerd cfg │               │
 │                                    │   generate CDI specs   │               │
 │                                    │   signal containerd    │               │
 │                                    └───────────┬────────────┘               │
 │                                       RUNNING  │                            │
 │                                                │                ┌───────────▼───────────┐
 │                                                │                │init toolkit-validation│
 │                                                │◀───────────────│ nvidia-smi INSIDE this│
 │                                                │  needs the     │ container. Proves the │
 │                                                │  runtime wired │ runtime injected the  │
 │                                                │                │ driver.               │
 │                                                │                │ → writes toolkit-ready│
 │                                                │                └───────────┬───────────┘
 │  ══ BARRIER: toolkit-ready ════════════════════════════════════════════════╪══
 │                                                                            │
 │  device-plugin ──┐  GFD ──┐  dcgm-exporter ──┐  mig-manager ──┐            │
 │  (init unblocks) │        │                  │                │            │
 │        ▼         ▼        ▼                  ▼                ▼            │
 │  registers    labels   scrapes DCGM     applies MIG                        │
 │  nvidia.com/  the node  + pod-resources  geometry                          │
 │  gpu with                                                                  │
 │  kubelet                                                       ┌───────────▼───────────┐
 │        │                                                       │init cuda-validation   │
 │        │                                                       │ creates pod           │
 │        │                                                       │ nvidia-cuda-validator │
 │        │                                                       │ → vectorAdd → exits   │
 │        │                                                       │ → writes cuda-ready   │
 │        │                                                       └───────────┬───────────┘
 │        │                                                       ┌───────────▼───────────┐
 │        └──── node.status.capacity["nvidia.com/gpu"] = 1 ──────▶│init plugin-validation │
 │                                                                │ polls node capacity   │
 │                                                                │ 30 × 5s               │
 │                                                                │ → writes plugin-ready │
 │                                                                └───────────┬───────────┘
 │                                                                ┌───────────▼───────────┐
 │                                                                │nvidia-operator-       │
 │                                                                │  validator (main)     │
 │                                                                │ "all validations are  │
 │                                                                │  successful"          │
 │                                                                │ then sleep 86400 loop │
 │                                                                └───────────────────────┘
 │  t ≈ 4–10 min: ClusterPolicy .status.state → ready
 ═════════════════════════════════════════════════════════════════════════════
```

The mental model to carry out of this: **a pod in `Init:0/N` is never the culprit — it is a victim.** Walk *up* to the first stage that is `CrashLoopBackOff`, `Error`, or `Init` with a validation loop that keeps failing; that pod owns the incident. And the corollary: deleting a downstream pod never fixes anything, because the barrier is a property of the upstream stage's health, and the freshly recreated pod re-enters the identical wait.

### 7. Inside the driver container

The driver DaemonSet is the operand people most often mis-model, so be precise: **it does not install a driver "into the container" in the ordinary sense, and it does not install one into the host's package manager either.** It builds kernel modules against the host's running kernel, loads them into the host kernel, and then republishes its *own* root filesystem at a host path so that everything else can bind-mount the user-space driver out of it.

Walking `ubuntu24.04/nvidia-driver` from `NVIDIA/gpu-driver-container`, the sequence inside `nvidia-driver init` is:

1. **Resolve the kernel version.** `_resolve_kernel_version()` prints `Resolving Linux kernel version...` and runs `apt-cache show linux-headers-$(uname -r)`. If that returns nothing it prints **`Could not resolve Linux kernel version`** to stderr and returns 1. This is the single most common driver-pod failure and lesson 04.2's first break: the container's package repos have no headers for the kernel the host happens to be running. On success it prints `Proceeding with Linux kernel version <resolved>`.
2. **Install prerequisites.** `Installing Linux kernel headers...` (`apt-get install linux-headers-$KERNEL_VERSION`), then `Installing Linux kernel module files...` (downloads and unpacks `linux-image-*` and `linux-modules-*` to populate `/lib/modules/$KERNEL_VERSION`), then `depmod`, then `Generating Linux kernel version string...`.
3. **Build or link the modules.** Either compiled from the sources shipped in `/usr/src/nvidia-$DRIVER_VERSION/`, or — with `driver.usePrecompiled=true` — linked from a precompiled package. `KERNEL_MODULE_TYPE` (default `auto`) decides open (`kernel-open`) vs proprietary (`kernel`); `auto` asks `nvidia-installer --print-recommended-kernel-module-type` and falls back to "branch ≥ 560 → open."
4. **Publish the rootfs.** `Mounting NVIDIA driver rootfs...` then, literally:

   ```sh
   mount --make-runbindable /sys
   mount --make-private /sys
   mkdir -p /run/nvidia/driver
   mount --rbind / /run/nvidia/driver
   ```

   That recursive bind of the container's `/` onto `/run/nvidia/driver` is *the* mechanism the rest of the stack depends on. Because `/run/nvidia` is mounted with `mountPropagation: Bidirectional`, the mount becomes visible on the host and therefore inside every other pod that mounts `/run/nvidia/driver`. `libcuda.so`, `nvidia-smi`, `libnvidia-ml.so.1` all live under there — which is why `NVIDIA_DRIVER_ROOT=/run/nvidia/driver` shows up in the `driver-ready` file and gets sourced by the toolkit and plugin.
5. **Install user-space components.** `Installing userspace components (libraries and binaries)...` — unpacks the `.run` file and runs `nvidia-installer --silent --no-kernel-module …`.
6. **Load the modules.** `Loading ipmi and i2c_core kernel modules...`, then `Loading NVIDIA driver kernel modules...`, then `modprobe nvidia`, `modprobe nvidia-uvm`, `modprobe nvidia-modeset` (plus `nvidia-peermem` when RDMA is on). Module parameters, if you set `driver.kernelModuleConfig`, are written into `/etc/modprobe.d/nvidia.conf` first.
7. **Park.** `_wait_for_signal()` prints **`Done, now waiting for signal`** and blocks on `sleep infinity` with traps on HUP/INT/QUIT/PIPE/TERM. **`Done, now waiting for signal` is the healthy steady state of a driver pod.** A driver pod whose last log line is that string is fine; the container is a supervisor for a mount and a set of loaded modules, not a service.

Meanwhile the `startupProbe` runs `startup-probe.sh` every 10 s, starting 60 s in, with `failureThreshold: 120` and `timeoutSeconds: 60`. That script is short enough to reproduce in full because it is precisely the definition of "the driver is up":

```sh
#!/bin/sh
set -eu
VALIDATIONS_DIR="/run/nvidia/validations"
READY_FILE="${VALIDATIONS_DIR}/.driver-ctr-ready"
mkdir -p "${VALIDATIONS_DIR}"
if [ ! -f /sys/module/nvidia/refcnt ]; then
  echo "NVIDIA kernel module not loaded"; exit 1
fi
if ! nvidia-smi; then
  echo "nvidia-smi failed"; exit 1
fi
# ... writes GDRCOPY_ENABLED / GDS_ENABLED / GPU_DIRECT_RDMA_ENABLED to a temp file
mv "$TMP_FILE" "$READY_FILE"
```

**Do the arithmetic on that probe budget: 60 s initial delay + 120 failures × 10 s period = up to 1,260 s ≈ 21 minutes before Kubernetes gives up on the driver container and restarts it.** That is deliberately generous, because building kernel modules on a small node genuinely can take many minutes. The operational consequence is that a *broken* driver takes 21 minutes to admit it, and during those 21 minutes the pod shows `0/1 Running`, not `CrashLoopBackOff`. If you are waiting on a fresh driver pod and it has been under 20 minutes, you have not yet learned anything.

Three escape hatches, all of which you will meet on-prem:

- **`driver.enabled=false`** — the host already has a driver from a golden image. The driver DaemonSet is not deployed at all; `driver-validation` then takes its *host* path (`chroot /host nvidia-smi`, after checking `/host/usr/bin/nvidia-smi` exists and is non-empty) and writes a `driver-ready` with `IS_HOST_DRIVER=true` and `NVIDIA_DRIVER_ROOT=/`. **Everything else in the chain still runs, with all of its failure modes.**
- **`driver.usePrecompiled=true`** — trades flexibility for determinism: no header download, no compile, fast boots, but the image must match your kernel exactly.
- **`driver.nvidiaDriverCRD.enabled=true`** — switches from the single cluster-wide `ClusterPolicy.driver` block to per-`NVIDIADriver` custom resources, so a heterogeneous fleet can run different driver versions/OS images side by side. Still marked as a roadmap item for GA promotion in the repo README as of v26.3.3, so read your version's notes before betting on it.

And a fourth detail worth internalising because it changes incident timelines: **the driver DaemonSet's `updateStrategy` is `OnDelete`, always.** The chart comment is explicit — "note that driver Daemonset is always set with OnDelete to avoid unintended disruptions" — while every other operand defaults to `RollingUpdate` with `maxUnavailable: "1"`. Changing `driver.version` therefore does *not* by itself restart driver pods. Either the upgrade controller does it (`driver.upgradePolicy.autoUpgrade`, set to `true` in the chart) or you delete pods yourself. That's the mechanism behind lesson 04.5.

### 8. Inside the toolkit container

The toolkit container's job is to make the sentence "run this container with a GPU" mean something to containerd. It does four things.

**(a) Install the toolkit binaries.** Into `/usr/local/nvidia/toolkit` on the host (`toolkit.installDir`, mounted at `/usr/local/nvidia` in the pod). That's `nvidia-container-runtime`, `nvidia-container-runtime.cdi`, `nvidia-container-runtime.legacy`, `nvidia-container-runtime-hook`, `nvidia-ctk`, `nvidia-cdi-hook`, and `libnvidia-container`.

**(b) Write the runtime config.** Two files, because modern containerd supports drop-ins:

- top-level: `/etc/containerd/config.toml`
- drop-in: `/etc/containerd/conf.d/99-nvidia.toml`

(CRI-O's equivalents are `/etc/crio/crio.conf` and `/etc/crio/crio.conf.d/99-nvidia.conf`.) The block it adds, per runtime name, in containerd config-version-2 shape:

```toml
version = 2

[plugins."io.containerd.grpc.v1.cri".containerd]
  default_runtime_name = "nvidia"

  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
    runtime_type = "io.containerd.runc.v2"
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
      BinaryName = "/usr/local/nvidia/toolkit/nvidia-container-runtime"
      SystemdCgroup = true

  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia-cdi]
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia-cdi.options]
      BinaryName = "/usr/local/nvidia/toolkit/nvidia-container-runtime.cdi"

  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia-legacy]
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia-legacy.options]
      BinaryName = "/usr/local/nvidia/toolkit/nvidia-container-runtime.legacy"

[plugins."io.containerd.grpc.v1.cri"]
  enable_cdi = true
```

The plugin path is **version-dependent**, and this is a real trap when you're grepping a config you didn't write. From `pkg/config/engine/containerd/containerd.go`:

| containerd `version =` | CRI plugin key |
|---|---|
| 1 | `cri` |
| 2 | `io.containerd.grpc.v1.cri` |
| 3 (and later) | `io.containerd.cri.v1.runtime` |

So `grep 'io.containerd.grpc.v1.cri' /etc/containerd/config.toml` returning nothing does **not** mean the toolkit failed — it may mean you're on containerd with a v3 config and the key is `io.containerd.cri.v1.runtime`. Grep for `runtimes.nvidia` instead.

**(c) Create the RuntimeClasses.** The operator creates three `node.k8s.io/v1` RuntimeClass objects in the `pre-requisites` state, each with `handler` equal to its name: `nvidia` (whatever `operator.runtimeClass` says, default `nvidia`), `nvidia-cdi`, `nvidia-legacy`. A pod that sets `runtimeClassName: nvidia` is asking the kubelet to pass `nvidia` as the CRI runtime handler; containerd looks up that handler in the table above. If the toolkit never wrote the table, containerd answers with the error you'll meet in lesson 04.2:

```
failed to get sandbox runtime: no runtime for "nvidia" is configured
```

That exact string is from containerd's own `internal/cri/config/config.go` (`no runtime for %q is configured`), wrapped by `podsandbox/sandbox_run.go`. Because `CONTAINERD_SET_AS_DEFAULT=true` makes `nvidia` the *default* runtime, most GPU pods on an Operator-managed cluster never set `runtimeClassName` at all — which is why a missing RuntimeClass usually surfaces as a *sandbox* failure on every pod on the node, not as a GPU-specific error.

**(d) Reload containerd, and generate CDI specs.** The toolkit's default restart mode is `signal` — it sends `SIGHUP` to the containerd process rather than restarting the systemd unit. (There is a hard-coded `sleep 5` in the toolkit entrypoint before `exec nvidia-toolkit`, with a comment pointing at a containerd ≥ 1.6.9 bug where the toolkit container caught a `SIGTERM` shortly after restarting containerd. That workaround is still in v26.3.3.) It also writes CDI specs into `/var/run/cdi`, under two kinds:

- `nvidia.com/gpu` — the per-device kind, `nvidia.com/gpu=0`, `nvidia.com/gpu=GPU-<uuid>`, `nvidia.com/gpu=all`.
- `management.nvidia.com/gpu` — a "management" kind that injects the driver *without* any specific GPU, used by the operator's own tooling. You can see it in the cuda-validator pod spec's annotation: `nvidia.cdi.k8s.io/container.cuda-validation: "management.nvidia.com/gpu=all"`.

CDI became the default injection path in the **v25.10.0** release line (`cdi.enabled: true` in the chart), and remains the default in v26.3.x. Lesson 04.4 owns CDI in depth; for now the operational fact is: on a current install, `nvidia-ctk cdi list` is where you check device injection, and the legacy `nvidia-container-runtime-hook` prestart-hook path is *not* what's in use by default.

The two environment variables the whole injection mechanism keys off, since they come up in every interview on this topic:

| Variable | Set by | Controls |
|---|---|---|
| `NVIDIA_VISIBLE_DEVICES` | the device plugin's `Allocate` response (`DEVICE_LIST_STRATEGY=envvar`) | **Which** GPUs the container sees. Comma-separated UUIDs, or `all`, or `void`/`none`. `void` means "do not touch this container at all" — which is why the driver and toolkit DaemonSets set `NVIDIA_VISIBLE_DEVICES: void`/`"void"` on themselves, so the nvidia runtime doesn't try to inject a driver into the pod that is *building* the driver. |
| `NVIDIA_DRIVER_CAPABILITIES` | the image, or the pod spec | **What parts** of the driver get mounted. Default is `utility,compute`; the full supported set is `compute,compat32,graphics,utility,video,display,ngx`; `all` means everything. `utility` is what gives you `nvidia-smi`; `compute` is what gives you `libcuda.so`. A container with only `utility` runs `nvidia-smi` fine and cannot run CUDA — a genuinely confusing state worth recognising. |

### 9. Device plugin, GFD, and the config-manager sidecar

The device plugin's mechanics are Module 02's material and lesson 04.3 revisits the gRPC contract. What belongs *here* is the operand's shape, because that's what you debug.

The DaemonSet runs **two** containers and **two** init containers:

```
initContainers:  toolkit-validation      # shell loop on toolkit-ready
                 config-manager-init     # ONESHOT=true: materialise config.yaml once
containers:      nvidia-device-plugin    # the plugin itself
                 config-manager          # ONESHOT=false: watch the node label, SIGHUP the plugin
```

The `config-manager` pair implements **live reconfiguration by node label**. Both are configured with `NODE_LABEL=nvidia.com/device-plugin.config`; the sidecar watches that label, copies the matching key out of the device-plugin ConfigMap to `/config/config.yaml`, and sends `SIGNAL=1` (`SIGHUP`) to `PROCESS_TO_SIGNAL=nvidia-device-plugin`. The plugin's `main.go` handles it — `Received SIGHUP, restarting.` — and re-reads the config. **That is the mechanism behind per-node time-slicing and MIG-strategy configs:** you label a node, not restart a DaemonSet. Lessons 04.6 and 04.7 use it.

The env the operator sets on the plugin container, with what each actually does:

| Env | Value | Effect |
|---|---|---|
| `PASS_DEVICE_SPECS` | `true` | `Allocate` also returns explicit `DeviceSpec` entries (`/dev/nvidia0`, permissions). Upstream default is `false`; the Operator turns it on because it's required to interoperate with the kubelet's `CPUManager`. |
| `FAIL_ON_INIT_ERROR` | `true` | If NVML fails to initialise, exit non-zero instead of idling. This is what turns "no driver" into a visible plugin crash rather than a silent zero-GPU node. |
| `DEVICE_LIST_STRATEGY` | `envvar` | Advertise allocated devices via `NVIDIA_VISIBLE_DEVICES`. Alternatives: `volume-mounts`, `cdi-annotations`, `cdi-cri`. |
| `DEVICE_ID_STRATEGY` | `uuid` | Device IDs are GPU UUIDs, not indices. **This is the decision that makes per-pod attribution possible** — lesson 04.3 builds directly on it. |
| `NVIDIA_VISIBLE_DEVICES` / `NVIDIA_DRIVER_CAPABILITIES` | `all` / `all` | The plugin pod itself needs the full driver to enumerate devices. |
| `MPS_ROOT` | `/run/nvidia/mps` | Where to find the MPS control daemon's pipe directory. |

**GFD is the same container image** (`k8s-device-plugin`) with a different entrypoint, and the same `config-manager` pair — which is why the plugin and GFD always agree on MIG strategy. It writes a feature file into `/etc/kubernetes/node-feature-discovery/features.d/` every `GFD_SLEEP_INTERVAL` (default `60s`); NFD picks it up and publishes it as labels. The label set, from `internal/lm/` in `k8s-device-plugin` v0.19.3:

| Label | Example | Use |
|---|---|---|
| `nvidia.com/gpu.product` | `NVIDIA-L4` | Node selectors; the join key for `$/GPU-hour` in your cost operator. |
| `nvidia.com/gpu.count` | `1` | Capacity planning. |
| `nvidia.com/gpu.memory` | `23034` (MiB) | Fit checks. |
| `nvidia.com/gpu.family` | `ada-lovelace` | Coarse generation. |
| `nvidia.com/gpu.compute.major` / `.minor` | `8` / `9` | Compute capability. |
| `nvidia.com/gpu.multiprocessors` | `58` | SM count — the denominator in per-SM efficiency maths. |
| `nvidia.com/gpu.mode` | `compute` / `graphics` | Display vs compute mode. |
| `nvidia.com/gpu.machine` | `Standard-PC-…` | DMI product name. |
| `nvidia.com/cuda.driver-version.full` / `.major` / `.minor` / `.revision` | `580.126.20` / `580` / `126` / `20` | **Fleet driver inventory straight out of node labels** — worth knowing, because it makes "which nodes are on the bad driver" a `kubectl get nodes -L` away. |
| `nvidia.com/cuda.runtime-version.full` / `.major` / `.minor` | `13.0` | Max CUDA the driver supports. |
| `nvidia.com/mig.capable` | `true` | The authoritative MIG-manager placement signal (§4). |
| `nvidia.com/mig.strategy` | `single` / `mixed` / `none` | How MIG devices are advertised. |
| `nvidia.com/mig-1g.10gb.count`, `.memory`, `.engines.*`, `.slices.gi`, `.slices.ci`, `.product` | | Per-profile MIG facts (lesson 04.6). |
| `nvidia.com/gpu.sharing-strategy` | `time-slicing` / `mps` / `none` | What sharing mode is active (lessons 04.7/04.8). |
| `nvidia.com/gpu.replicas` | `10` | How many replicas per physical GPU under sharing. |
| `nvidia.com/gpu.shared` | `true` | Whether the advertised resource is a shared replica. |
| `nvidia.com/gpu.clique` | | IMEX clique ID for multi-node NVLink domains. |
| `nvidia.com/gfd.timestamp` | epoch seconds | When GFD last wrote. **Stale = GFD stopped.** |

Also useful: `nvidia.com/not-gpu=true` on a node GFD ran on and found nothing.

### 10. DCGM-exporter — the operand this course is actually about

`nvidia-dcgm-exporter` is the observability substrate the rest of the module builds on, and one line of its DaemonSet is the whole reason lesson 04.3 exists:

```yaml
volumeMounts:
  - name: "pod-gpu-resources"
    readOnly: true
    mountPath: "/var/lib/kubelet/pod-resources"
volumes:
  - name: "pod-gpu-resources"
    hostPath:
      path: "/var/lib/kubelet/pod-resources"
```

That is the kubelet **pod-resources API socket**, bind-mounted into the exporter. DCGM knows GPU UUIDs and nothing about Kubernetes; the kubelet knows which pod holds which device ID. The exporter joins them, and the result is the `pod`/`namespace`/`container` labels you see on GPU metrics. Lesson 04.3 makes you write that join yourself; the fact that NVIDIA's own reference exporter does it this way, with this mount, is the primary-source evidence that it's the right seam.

The rest of its config:

| Env | Value | Meaning |
|---|---|---|
| `DCGM_EXPORTER_LISTEN` | `:9400` | Prometheus scrape port. |
| `DCGM_EXPORTER_KUBERNETES` | `true` | Enable the pod-resources join. Without it you get metrics with no pod labels. |
| `DCGM_EXPORTER_COLLECTORS` | `/etc/dcgm-exporter/dcp-metrics-included.csv` | Which DCGM fields to publish. Override via `dcgmExporter.config` (a ConfigMap keyed `dcgm-metrics.csv`). |
| `NODE_NAME` | fieldRef | Scopes the pod informer to this node's pods only. |

The fields you will actually use for cost work: `DCGM_FI_DEV_GPU_UTIL` (SM-busy percent), `DCGM_FI_DEV_FB_USED` / `DCGM_FI_DEV_FB_FREE` (framebuffer MiB), `DCGM_FI_DEV_POWER_USAGE` (watts), `DCGM_FI_DEV_SM_CLOCK` / `DCGM_FI_DEV_MEM_CLOCK` (MHz), and the profiling family `DCGM_FI_PROF_*` (e.g. `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`, `DCGM_FI_PROF_DRAM_ACTIVE`) when you need real occupancy rather than the crude "was a kernel resident" signal `GPU_UTIL` gives you.

Two chart flags matter for the capstone and are off by default because of Prometheus cardinality: `dcgmExporter.enablePodLabels` and `dcgmExporter.enablePodUID`. Turning either on makes the operator create a cluster-scoped ClusterRole granting the exporter `get/list/watch` on pods, and `dcgmExporter.podLabelAllowlistRegex` is how you bound the label explosion. Read that as the vendor's own warning: attaching arbitrary pod labels to per-GPU time series is a cardinality footgun.

Finally: the exporter's init container waits on `toolkit-ready`, not on `driver-ready` — DCGM needs both the driver *and* an injected runtime to open the GPU, so its gate is the later one.

### 11. Installing, and the version discipline

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

**Do not hard-pin a version number from any lesson, including this one — resolve it live and pin *that*.** Public tags in `NVIDIA/gpu-operator` run 23.3.x → 23.6.x → 23.9.x → 24.3/24.6/24.9 → 25.3.x → 25.10.x → 26.3.x, with **v26.3.3 the newest at the time of writing**. Note what is *not* in that list: there has never been a public GPU Operator "25.8" — that number belonged to the DRA driver's own old CalVer scheme, and the module README flags it for exactly this reason.

Two release-line changes are worth knowing by name because they change what you'll see in the field:

- **v25.10.0 made CDI the default injection path** on containerd/CRI-O. On a 25.10-or-newer install, check injection with `nvidia-ctk cdi list`, not by looking for the legacy prestart hook.
- **v26.3.0 added the `NVIDIADriver` CRD** for running multiple driver versions/OS images across a heterogeneous fleet, where previously one `ClusterPolicy.driver` block applied cluster-wide. The repo README still lists promoting that CRD to GA as a roadmap item, so treat it as usable-but-moving.

Chart values you will reach for regardless of version:

| Value | Default (v26.3.3) | Why you'd change it |
|---|---|---|
| `driver.enabled` | `true` | `false` when nodes ship with a host driver. |
| `driver.version` | `580.126.20` | Pin to a branch your kernel supports. |
| `driver.usePrecompiled` | `false` | `true` for fast deterministic boots on a known kernel. |
| `driver.upgradePolicy.autoUpgrade` | `true` (chart) / `false` (CRD default) | Whether the operator drives driver-pod recreation itself. |
| `driver.upgradePolicy.maxParallelUpgrades` | `1` | Rollout width. `0` = unlimited. |
| `driver.upgradePolicy.maxUnavailable` | `25%` | Rollout safety ceiling. |
| `driver.upgradePolicy.drain.enable` | `false` | `true` when evicting GPU pods isn't enough and you need a real `kubectl drain`. |
| `toolkit.enabled` | `true` | `false` if the toolkit is pre-installed and pre-configured. |
| `mig.strategy` | `single` | `mixed` on nodes with heterogeneous MIG profiles. |
| `daemonsets.updateStrategy` / `.rollingUpdate.maxUnavailable` | `RollingUpdate` / `"1"` | Applies to every operand **except** the driver, which is always `OnDelete`. |
| `nodeStatusExporter.enabled` | `false` | `true` to get the barrier files as Prometheus gauges. |
| `dcgm.enabled` | `false` | `true` for a standalone hostengine instead of the exporter's embedded one. |

**Worked arithmetic: how long does a driver rollout take, and what does it cost?**

This is the calculation you'll be asked to do out loud, so carry the units. Setup: 200 GPU nodes, 8 GPUs each, `driver.version` bumped, `autoUpgrade: true`.

Per-node wall time, decomposed:

```
  t_image     pull the new driver image (~2 GB compressed)
              at 200 MB/s effective from a regional registry ......  ~10 s
              (cold cache; ~0 s if pre-pulled)
  t_evict     k8s-driver-manager: label GPU clients off the node,
              wait for plugin/GFD/DCGM pods to terminate,
              rmmod the old modules .............................. 30–90 s
              (longer if a GPU pod holds a module: EBUSY → retry)
  t_build     resolve kernel, install headers, build modules ....... 3–7 min
              (≈15 s with driver.usePrecompiled=true)
  t_load      modprobe + mount rootfs ............................. 10–20 s
  t_probe     startupProbe initialDelaySeconds ...................... 60 s
              (a floor, not an estimate: the probe will not run earlier)
  t_chain     driver-ready → toolkit → toolkit-ready → plugin
              re-registers → plugin-ready → validator green ....... 60–120 s
  ─────────────────────────────────────────────────────────────────────────
  T_node  ≈  10 + 60 + 300 + 15 + 60 + 90  ≈  535 s  ≈  9 min   (built)
  T_node  ≈  10 + 60 +  15 + 15 + 60 + 90  ≈  250 s  ≈  4.2 min (precompiled)
```

Now the fleet. Two independent limits apply and the *smaller* one binds:

```
  maxParallelUpgrades = 1          → 1 node at a time
  maxUnavailable      = 25% of 200 = 50 nodes may be unavailable

  effective width W = min(1, 50) = 1

  waves   = ceil(200 / 1) = 200
  T_fleet = 200 × 9 min = 1800 min = 30 hours          ← the default!
```

Thirty hours for a driver bump is not a hypothetical annoyance; it is why `maxParallelUpgrades: 1` is a default you should consciously accept or change. Redo it with a width you'd actually pick:

```
  maxParallelUpgrades = 20, maxUnavailable = 25% (=50)
  W = min(20, 50) = 20
  waves   = ceil(200/20) = 10
  T_fleet = 10 × 9 min = 90 min = 1.5 hours

  Same, with usePrecompiled=true (T_node ≈ 4.2 min):
  T_fleet = 10 × 4.2 = 42 min
```

The GPU-hours lost, which is the number a manager actually wants:

```
  GPU-hours lost = nodes × GPUs/node × (T_node / 60 min)
                 = 200 × 8 × (9/60)
                 = 200 × 8 × 0.15
                 = 240 GPU-hours

  Note this is independent of W: every node is down for T_node regardless of
  how many go down at once. Width buys you WALL-CLOCK, not GPU-hours.

  At an illustrative $2.50 / GPU-hour:  240 × $2.50 = $600 per fleet-wide
  driver bump.

  With usePrecompiled (T_node ≈ 4.2 min = 0.07 h):
  200 × 8 × 0.07 = 112 GPU-hours ≈ $280  →  precompiled images roughly HALVE
  the cost of every driver rollout you will ever do.
```

Two conclusions to take into an interview. First, **`maxParallelUpgrades` and `maxUnavailable` are different knobs**: the first is a *rate* limit the operator applies, the second is a *safety ceiling* computed from total managed nodes (`intstr.GetScaledValueFromIntOrPercent(…, totalNodes, roundUp=true)`), and the effective width is the minimum. Second, **width is a wall-clock lever and node downtime is a cost lever.** If you want the bill down, shorten `T_node` — pre-pull images, use precompiled drivers, keep the startup probe's initial delay in mind — not widen the wave.

### 12. Verifying a node end to end

A green operand list is necessary and not sufficient. The node is *ready* only when the full path from resource to executed kernel works. Four layers, cheapest first:

```bash
# Layer 1 — the resource is advertised AND allocatable
kubectl get node <node> -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{"\n"}'
# 1

# Layer 2 — the descriptive labels landed (GFD + NFD did their jobs)
kubectl get node <node> -o json \
  | jq '.metadata.labels | with_entries(select(.key|test("nvidia.com/")))'

# Layer 3 — the driver sees the silicon
kubectl -n gpu-operator exec ds/nvidia-driver-daemonset -c nvidia-driver-ctr -- nvidia-smi -L
# GPU 0: NVIDIA L4 (UUID: GPU-3d1e2f4a-...)

# Layer 4 — the barriers are all open
kubectl -n gpu-operator exec ds/nvidia-operator-validator -- ls -1 /run/nvidia/validations/
# .driver-ctr-ready
# cuda-ready
# driver-ready
# plugin-ready
# toolkit-ready
# workload-type

# Layer 5 — a real container actually got a device
kubectl logs nvidia-cuda-validator-xxxxx -c cuda-validation
# [Vector addition of 50000 elements]
# Test PASSED
```

**A correction worth internalising, because the wrong version of it circulates widely:** `kubectl logs nvidia-cuda-validator-xxxxx` with no `-c` gives you the *main* container, whose entire job is `echo cuda workload validation is successful`. The `vectorAdd` output — the actual proof a CUDA kernel ran — is in the **`cuda-validation` init container**. Same for the plugin validator: main container says `device-plugin workload validation is successful`; the real work is in the `plugin-validation` init container, which requests `nvidia.com/gpu: 1` through the scheduler.

If layers 1–3 pass and layer 5 fails, the resource is advertised but the *runtime wiring* is wrong — the exact failure family 04.2 drills. Building the habit of checking layer 5 is what stops "the dashboard is green but jobs can't see a GPU" from becoming a two-hour incident.

### 13. Reading each component's logs

Every operand pod here is multi-container, so bare `kubectl logs` will either pick a default you didn't intend or refuse. Name the container:

```bash
kubectl -n gpu-operator logs ds/nvidia-driver-daemonset            -c nvidia-driver-ctr
kubectl -n gpu-operator logs ds/nvidia-container-toolkit-daemonset -c nvidia-container-toolkit-ctr
kubectl -n gpu-operator logs ds/nvidia-device-plugin-daemonset     -c nvidia-device-plugin
kubectl -n gpu-operator logs ds/gpu-feature-discovery              -c gpu-feature-discovery

# Init containers, for anything stuck in Init:
kubectl -n gpu-operator logs <toolkit-pod> -c driver-validation
kubectl -n gpu-operator logs <plugin-pod>  -c toolkit-validation
kubectl -n gpu-operator logs ds/nvidia-operator-validator -c cuda-validation
kubectl -n gpu-operator logs -l app=nvidia-cuda-validator -c cuda-validation

# What is blocking, and why:
kubectl -n gpu-operator describe pod <pod> | sed -n '/Init Containers/,/^Conditions/p'
```

`describe pod` on a pod in `Init:0/2` tells you *which* init container is pending; go to that container's logs, not the main container's — that returns `container "x" in pod "y" is waiting to start`.

## Perspectives

**Developer / platform-consumer.** From a workload author's chair none of this exists. They write `resources.limits: {nvidia.com/gpu: 1}` and either the pod schedules and runs, or it doesn't — which is the point of the abstraction, and also why they cannot self-serve the diagnosis when it breaks. Your job in this course is to be the person who *can* see it.

**Operator / SRE.** The chain is a state machine you read top-down, not a pile of independent pods. A pod stuck in `Init` is always a symptom, never a cause. Enable `nodeStatusExporter` and the barrier files become four Prometheus gauges per node (`gpu_operator_node_driver_ready`, `…_toolkit_ready`, `…_cuda_ready`, `…_plugin_ready`); "which barrier is closed on which node" is then one PromQL query, which converts this whole lesson from tribal knowledge into a dashboard.

**Kernel / systems.** Almost nothing in the driver stage is Kubernetes-specific. `mount --rbind / /run/nvidia/driver`, `mountPropagation: Bidirectional`, `modprobe`, `depmod`, `rmmod` returning `EBUSY`, `/sys/module/nvidia/refcnt` — this is Linux wearing a DaemonSet costume. If you can explain why the rebind mount must be `Bidirectional` for the toolkit to see it, you understand the mechanism; if you can only recite the YAML, you don't.

**Vendor / managed-platform.** Not every GPU-fleet operator wants you running this yourself. CoreWeave's documentation for its managed Kubernetes offering (CKS) recommends *against* tenants self-installing the GPU Operator; CoreWeave manages driver/toolkit/device-plugin lifecycle on your behalf, with a changelog entry recording GPU driver management features shipping to CKS in August 2025. On a platform where the vendor owns the hardware and the fleet-wide upgrade risk, two uncoordinated reconcilers fighting over the same nodes is a real hazard. Knowing the chain cold is still valuable there — you'll debug its *symptoms* either way — but recognise which side of the line you're on before you `helm install`.

**Economics.** Every minute a node sits in `Init` waiting on an upstream barrier is a fully-billed idle GPU-hour. A four-to-ten minute convergence on a fresh install is normal and cheap. But the 21-minute startup-probe budget means a *broken* driver looks normal for a third of an hour, and §11's arithmetic says the default `maxParallelUpgrades: 1` turns a 200-node driver bump into a 30-hour operation. Both are line items, not debugging inconveniences.

## Real-world use cases

- **[NVIDIA/gpu-operator](https://github.com/NVIDIA/gpu-operator) at tag `v26.3.3` — the primary source for everything above.** Read three directories and you have the whole system: `deployments/gpu-operator/values.yaml` (every default in this lesson), `assets/state-*/0500_daemonset.yaml` (every container name, mount, init container and wait message), and `cmd/nvidia-validator/main.go` (every barrier file name and validation semantic). *What it shows:* the component list and ordering are not folklore — they are 200 lines of YAML and one Go file, and reading them directly is faster than reading any write-up about them, including this one.
- **[NVIDIA/gpu-operator issue #430 — `/dev/char` symlinks and systemd cgroup management.](https://github.com/NVIDIA/gpu-operator/issues/430)** This one is *cited from inside the product*: the validator's `createDevCharSymlinks()` failure path prints a multi-line message pointing at this exact issue and explains that recent `runc` requires symlinks under `/dev/char` for any injected device node, that the NVIDIA driver doesn't create them, and that the impact is on runtimes with systemd cgroup management enabled. The same class of bug is tracked publicly as gpu-operator #485 and nvidia-container-toolkit #48, both titled *"NOTICE: Containers losing access to GPUs with error: `Failed to initialize NVML: Unknown Error`"*, and the failure is triggered by something as innocuous as `systemctl daemon-reload`. *What it shows:* (a) a running container can *lose* its GPU without anything in Kubernetes changing; (b) the operator's own escape hatch is `validator.driver.env` with `DISABLE_DEV_CHAR_SYMLINK_CREATION=true`; (c) the fix outside Kubernetes is `nvidia-ctk system create-dev-char-symlinks --create-all`.
- **[NVIDIA/gpu-operator issue #1220 — "gpu-operator breaks when upgrading EKS to K8s v1.30".](https://github.com/NVIDIA/gpu-operator/issues/1220)** A fleet running fine on Kubernetes 1.29 broke on the routine 1.30 upgrade because the Ubuntu 22.04 AMIs for 1.30 shipped kernel 6.5/6.8 instead of 5.15, and the driver container could not resolve headers for the running kernel — surfacing as `Could not resolve Linux kernel version`. *What it shows:* a vendor-driven control-plane bump you did not choose can move the kernel out from under a working driver image; §7's step 1 is not a theoretical failure. *Verification note:* the issue title and substance were confirmed via search this session; `github.com` HTML pages are not directly fetchable through this environment's egress proxy, so the exact comment thread was not re-read. The two log strings involved were independently verified against primary source — `Could not resolve Linux kernel version` in `NVIDIA/gpu-driver-container`'s `nvidia-driver` script, and `no runtime for %q is configured` in containerd's `internal/cri/config/config.go`.
- **[CoreWeave — About GPU Driver Management in CKS](https://docs.coreweave.com/docs/products/cks/nodes/gpu-driver-management/gpu-driver-management-cks).** A managed GPU cloud's documented policy of owning driver/toolkit lifecycle rather than letting tenants self-install the Operator. *Not independently fetched this session* (proxy-blocked domain); corroborated by an independent search hit and a CoreWeave changelog entry dated August 15, 2025. Treat as high-confidence, spot-check before quoting.
- **An honest gap.** No tier-1 engineering-blog postmortem specifically narrating a GPU-Operator dependency-chain incident (a Meta/Netflix/Discord-style timeline writeup) could be found or confirmed. The filed issues above are the strongest real-world evidence available, and are arguably *better* evidence for this purpose: primary, dated, version-pinned, and exactly on topic. If you find such a postmortem at study time it's a strong addition — but do not assume one exists behind this lesson's claims.

## Worked example

Fresh single-node cluster (k3s, containerd), one L4, GPU Operator installed from the chart with defaults.

**t+90 s — first snapshot.**

```
$ kubectl get pods -n gpu-operator
NAME                                              READY   STATUS     RESTARTS   AGE
gpu-operator-7c9d8b6f5-2mzln                      1/1     Running    0          90s
gpu-operator-node-feature-discovery-gc-...         1/1     Running    0          90s
gpu-operator-node-feature-discovery-master-...     1/1     Running    0          90s
gpu-operator-node-feature-discovery-worker-4hsx8   1/1     Running    0          90s
nvidia-driver-daemonset-abcde                     0/1     Running    0          75s
nvidia-container-toolkit-daemonset-fghij          0/1     Init:0/1   0          70s
nvidia-device-plugin-daemonset-klmno              0/2     Init:0/2   0          70s
gpu-feature-discovery-uvwxy                        0/2     Init:0/2   0          70s
nvidia-dcgm-exporter-zzzzz                         0/1     Init:0/1   0          70s
nvidia-operator-validator-pqrst                    0/1     Init:0/4   0          70s
```

Read that carefully. The driver pod is `0/1 Running` — the container started but its `startupProbe` hasn't passed yet, which is expected for at least 60 s and possibly minutes. Nothing here is broken; everything downstream is in `Init` because `driver-ready` doesn't exist yet. The `Init:0/2` on the plugin and GFD is the two-init-container shape from §9 (`toolkit-validation` + `config-manager-init`); `Init:0/4` on the validator is its four validations.

Confirm what's blocking, rather than guessing:

```
$ kubectl -n gpu-operator logs nvidia-container-toolkit-daemonset-fghij -c driver-validation | tail -4
time="..." level=info msg="Attempting to validate a pre-installed driver on the host"
time="..." level=info msg="No pre-installed driver detected on the host: no 'nvidia-smi' file present on the host: lstat /host/usr/bin/nvidia-smi: no such file or directory"
time="..." level=info msg="Validating containerized driver installation"
time="..." level=warning msg="failed to validate the driver, retrying after 5 seconds"
```

That is idiom B from §5 doing exactly what it should: no host driver (correct — this is a cloud node), so it's polling the containerized one. And the downstream shell loops:

```
$ kubectl -n gpu-operator logs nvidia-device-plugin-daemonset-klmno -c toolkit-validation | tail -2
waiting for nvidia container stack to be setup
waiting for nvidia container stack to be setup
```

Now the driver pod, which is where the actual work is:

```
$ kubectl -n gpu-operator logs nvidia-driver-daemonset-abcde -c nvidia-driver-ctr
DRIVER_ARCH is x86_64
Creating directory NVIDIA-Linux-x86_64-580.126.20
Verifying archive integrity... OK
Uncompressing NVIDIA Accelerated Graphics Driver for Linux-x86_64 580.126.20
Resolving Linux kernel version...
Proceeding with Linux kernel version 6.8.0-51-generic
Installing Linux kernel headers...
Installing Linux kernel module files...
Generating Linux kernel version string...
Compiling NVIDIA driver kernel modules...
...
Relinking NVIDIA driver kernel modules...
Building NVIDIA driver package NVIDIA-Linux-x86_64-580.126.20...
Installing userspace components (libraries and binaries)...
Mounting NVIDIA driver rootfs...
Parsing kernel module parameters...
Loading ipmi and i2c_core kernel modules...
Loading NVIDIA driver kernel modules...
Done, now waiting for signal
```

`Done, now waiting for signal` is the finish line for that container. Note the order: userspace install and rootfs mount happen *before* module load. (Representative transcript — the exact interleaving of `nvidia-installer` output varies by driver branch and base image.)

**t+6 min — second snapshot.**

```
$ kubectl get pods -n gpu-operator
nvidia-driver-daemonset-abcde                     1/1     Running     0          6m
nvidia-container-toolkit-daemonset-fghij          1/1     Running     0          5m55s
nvidia-device-plugin-daemonset-klmno              2/2     Running     0          5m55s
gpu-feature-discovery-uvwxy                        2/2     Running     0          5m55s
nvidia-dcgm-exporter-zzzzz                         1/1     Running     0          5m55s
nvidia-operator-validator-pqrst                    1/1     Running     0          5m55s
nvidia-cuda-validator-vvvvv                        0/1     Completed   0          3m10s
```

The chain drained top to bottom. Verify each barrier actually opened, and who opened it:

```
$ kubectl -n gpu-operator exec ds/nvidia-operator-validator -- ls -1 /run/nvidia/validations/
.driver-ctr-ready
cuda-ready
driver-ready
plugin-ready
toolkit-ready
workload-type

$ kubectl -n gpu-operator exec ds/nvidia-operator-validator -- cat /run/nvidia/validations/driver-ready
IS_HOST_DRIVER=false
NVIDIA_DRIVER_ROOT=/run/nvidia/driver
DRIVER_ROOT_CTR_PATH=/run/nvidia/driver
NVIDIA_DEV_ROOT=/run/nvidia/driver
DEV_ROOT_CTR_PATH=/run/nvidia/driver
```

`IS_HOST_DRIVER=false` and `NVIDIA_DRIVER_ROOT=/run/nvidia/driver` is the containerized-driver fingerprint. On a `driver.enabled=false` cluster those two lines read `true` and `/`, and that single file is the fastest way to answer "is this node using the operator's driver or the image's?"

The plugin's registration, which is what actually created the resource:

```
$ kubectl -n gpu-operator logs nvidia-device-plugin-daemonset-klmno -c nvidia-device-plugin
waiting for the driver validations to be ready...
IS_HOST_DRIVER=false
NVIDIA_DRIVER_ROOT=/run/nvidia/driver
...
Starting nvidia-device-plugin
I... Starting FS watcher for /var/lib/kubelet/device-plugins
I... Starting OS watcher.
I... Starting Plugins.
I... Loading configuration.
I... Updating config with default resource matching patterns.
I... Retrieving plugins.
I... Starting GRPC server for 'nvidia.com/gpu'
I... Starting to serve 'nvidia.com/gpu' on /var/lib/kubelet/device-plugins/nvidia-gpu.sock
I... Registered device plugin for 'nvidia.com/gpu' with Kubelet
```

Those last three lines are the healthy registration signature — memorise them, because their *absence* is the whole of lesson 04.2's failure family 3. Note the plugin waited on `driver-ready` and then sourced it (idiom A again, in the main container this time).

Layers 1, 2 and 5 of the verification ladder:

```
$ kubectl get node -o jsonpath='{.items[0].status.allocatable.nvidia\.com/gpu}'; echo
1

$ kubectl get node -o json | jq -r '.items[0].metadata.labels
  | to_entries[] | select(.key|test("^nvidia.com/(gpu|cuda|mig)"))
  | "\(.key)=\(.value)"' | sort
nvidia.com/cuda.driver-version.full=580.126.20
nvidia.com/cuda.runtime-version.full=13.0
nvidia.com/gpu.compute.major=8
nvidia.com/gpu.compute.minor=9
nvidia.com/gpu.count=1
nvidia.com/gpu.deploy.container-toolkit=true
nvidia.com/gpu.deploy.dcgm-exporter=true
nvidia.com/gpu.deploy.device-plugin=true
nvidia.com/gpu.deploy.driver=true
nvidia.com/gpu.deploy.gpu-feature-discovery=true
nvidia.com/gpu.deploy.operator-validator=true
nvidia.com/gpu.family=ada-lovelace
nvidia.com/gpu.memory=23034
nvidia.com/gpu.present=true
nvidia.com/gpu.product=NVIDIA-L4
nvidia.com/mig.capable=false
nvidia.com/mig.strategy=single

$ kubectl -n gpu-operator logs nvidia-cuda-validator-vvvvv -c cuda-validation
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```

**Notice what is not in that label list: `nvidia.com/gpu.deploy.mig-manager`.** `nvidia.com/mig.capable=false` on an L4, so §4's `hasMIGCapableGPU()` returned false and the operator never stamped the switch. `kubectl get pods -n gpu-operator` correspondingly has no `nvidia-mig-manager` pod. That absence is *correct*, and being able to say why in one sentence is one of the acceptance criteria below.

Finally, the whole-system summary and the coarse status:

```
$ kubectl get clusterpolicy cluster-policy -o jsonpath='{.status.state}{"\n"}'
ready
```

Node is genuinely ready — not "a resource is advertised," but "a real CUDA kernel executed on this GPU and every barrier on the node is open."

## Practice

On a fresh single-node cluster with a **rented L4 / A10 / L40S** (kubeadm or k3s, containerd runtime) — this is the cluster you'll keep across lessons 1–2 and reuse for the [Per-pod GPU attribution](../practice/per-pod-attribution/README.md) deliverable:

1. **Resolve, then install.** `helm search repo nvidia/gpu-operator -l` to find the current version; install with the pinned command from §11. Record the version you installed — every claim you write down below is scoped to it.
2. **Watch the chain converge.** `kubectl get pods -n gpu-operator -w`, and record a timestamp for each transition: NFD label appears → driver pod `Running` → driver pod `1/1` (probe passed) → toolkit `Running` → plugin `Running` → validator `Running` → `cuda-validator` `Completed`. You are building the real version of §6's timeline for *your* hardware.
3. **Build the component map (the deliverable).** For every operand pod, record: pod name · kind · container names · what it does in one sentence · **which barrier file it waits on** · **which barrier file it writes** (many write none). Get `toolkit-ready`'s author right. Include a row explaining why `nvidia-mig-manager` is absent, citing the label that decides it.
4. **Observe both waiting idioms.** While the driver is still building, capture (a) a shell-loop wait — `kubectl -n gpu-operator logs <plugin-pod> -c toolkit-validation` showing `waiting for nvidia container stack to be setup`, and (b) a retrying validation — `kubectl -n gpu-operator logs <toolkit-pod> -c driver-validation` showing `failed to validate the driver, retrying after 5 seconds`. Note in your map which operands use which.
5. **Dump the barrier directory and the `driver-ready` contents.** `kubectl -n gpu-operator exec ds/nvidia-operator-validator -- ls -l /run/nvidia/validations/` and `-- cat /run/nvidia/validations/driver-ready`. Explain in one sentence why the toolkit and device-plugin *main* containers need this file's contents and not just its existence.
6. **Trace the label chain on your node.** Show the NFD label, the operator-added `nvidia.com/gpu.present`, and the full `gpu.deploy.*` set. Then confirm which spelling of the PCI label you got (`pci-10de.present` vs `pci-0302_10de.present`) and say what that tells you about whose NFD config is in effect.
7. **Run the end-to-end test.**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: cuda-vectoradd
   spec:
     restartPolicy: OnFailure
     containers:
     - name: cuda-vectoradd
       image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubi8
       resources:
         limits:
           nvidia.com/gpu: 1
   ```

   `kubectl logs cuda-vectoradd` must show `Test PASSED`. Then read the *validator's* CUDA proof from the right container: `kubectl -n gpu-operator logs -l app=nvidia-cuda-validator -c cuda-validation`.
8. **Do the rollout arithmetic for your own fleet size.** Take §11's decomposition, substitute your measured `T_node` from step 2, and compute `T_fleet` and GPU-hours lost for 200 nodes at `maxParallelUpgrades` of 1, 10 and 20. Write the one-sentence recommendation you'd give a manager.

**Acceptance:**
(a) a `cuda-vectoradd` pod that reaches `Completed` with `Test PASSED`;
(b) a written **component map** naming every operand, its containers, its role, and its position in the barrier chain — with `toolkit-ready` correctly attributed to the operator-validator and `nvidia-mig-manager`'s absence explained by the `nvidia.com/mig.capable` label;
(c) captured log evidence of both waiting idioms;
(d) a rollout-time and GPU-hours calculation using your own measured per-node time.
Keep this cluster running — 04.2 breaks it on purpose.

## Common pitfalls

1. **Attributing `toolkit-ready` to the toolkit DaemonSet.** It's written by the operator-validator's `toolkit-validation` init container. When the device plugin is stuck on `toolkit-ready`, the pod to read is `nvidia-operator-validator`. Getting this wrong sends you to read the logs of a healthy pod while the broken one scrolls past unread. *Mechanism:* the toolkit DS only *consumes* barriers (`driver-ready`, via its init container and its entrypoint); the validator is the only writer of `toolkit-ready`, `cuda-ready` and `plugin-ready`.
2. **Reading a driver pod's `0/1 Running` as a failure.** The `startupProbe` has a 60 s initial delay and a 120 × 10 s budget: **up to 21 minutes** before Kubernetes restarts the container. Under twenty minutes, `0/1 Running` on a fresh driver pod carries no information. *Mechanism:* `initialDelaySeconds: 60`, `periodSeconds: 10`, `failureThreshold: 120` in the driver DaemonSet.
3. **"Delete the downstream pod" as a fix reflex.** A pod in `Init` restarts into the identical wait, because the barrier lives in the upstream stage's health. Worse, deleting the *validator* pod fires its `preStop` hook, `rm -f /run/nvidia/validations/*-ready`, which closes every barrier on the node and sends every other operand back to `Init`. *Mechanism:* the barrier is a file on a host path, not pod state.
4. **Assuming a missing NFD label is a bug rather than the most common root cause.** No `pci-10de.present` (or `pci-0302_10de.present`, or `pci-0300_10de.present`) means `hasGPULabels()` is false, no deploy labels are stamped, and no operand's `nodeSelector` matches — so the node looks untouched. *Mechanism:* three-link label chain, §4; check link 1 before suspecting the controller.
5. **Grepping for `io.containerd.grpc.v1.cri` and concluding the toolkit failed.** The CRI plugin key depends on the containerd config version: `cri` at v1, `io.containerd.grpc.v1.cri` at v2, `io.containerd.cri.v1.runtime` at v3+. Grep `runtimes.nvidia` instead, and remember the drop-in at `/etc/containerd/conf.d/99-nvidia.toml`. *Mechanism:* `criRuntimePluginName()` in `nvidia-container-toolkit`.
6. **Expecting `driver.version` changes to restart driver pods.** The driver DaemonSet is always `OnDelete`, deliberately. Either `driver.upgradePolicy.autoUpgrade` drives it or you delete pods yourself; the bump alone changes nothing. *Mechanism:* `updateStrategy.type: OnDelete` hard-coded in `assets/state-driver/0500_daemonset.yaml`.
7. **Reading `kubectl logs nvidia-cuda-validator-xxxxx` and thinking you saw CUDA run.** The main container only echoes `cuda workload validation is successful`; the `vectorAdd` output lives in the `cuda-validation` init container. *Mechanism:* the pod spec puts the real work in an init container and overrides the main container's command so the pod can complete.
8. **Treating `driver.enabled=false` as "the Operator does nothing," or hard-pinning a version out of documentation.** With the driver disabled, the toolkit, plugin, GFD, DCGM-exporter and validator all still run against the host driver with all of their failure modes; only the driver DaemonSet is skipped. And the release line has moved 23.3 → 26.3 in about two and a half years, with the CDI default flipping at 25.10.0 and the `NVIDIADriver` CRD arriving at 26.3.0 — a stale `--version=` in a runbook doesn't just miss features, it silently changes *which mechanism you're debugging*.

## Self-check

- **Put the dependency chain in order and say what each stage gates.** *Answer:* NFD (worker discovers PCI vendor `10de`, master publishes `feature.node.kubernetes.io/pci-10de.present=true`) → the operator stamps `nvidia.com/gpu.present=true` and the eight `nvidia.com/gpu.deploy.*` switches, which are every operand's `nodeSelector` → **driver DaemonSet** builds/loads the kernel modules, republishes its rootfs at `/run/nvidia/driver`, and its `startupProbe` writes `.driver-ctr-ready` → a **`driver-validation` init container** (one in the toolkit DS, one in the validator DS, running `nvidia-validator -c driver` with `WITH_WAIT=true`) proves the driver is usable and writes **`driver-ready`** → the **toolkit container** waits on and *sources* `driver-ready`, installs to `/usr/local/nvidia/toolkit`, writes the containerd runtime config plus `/etc/containerd/conf.d/99-nvidia.toml`, generates CDI specs, and signals containerd → the **validator's `toolkit-validation`** init container runs `nvidia-smi` inside its own container (an end-to-end test of the runtime wiring) and writes **`toolkit-ready`** → that unblocks the shell-loop init containers of the **device plugin, GFD, DCGM-exporter and MIG-manager** → the plugin registers `nvidia.com/gpu` with the kubelet, making it allocatable → the validator's **`cuda-validation`** creates the `nvidia-cuda-validator` pod (real `vectorAdd`) and writes **`cuda-ready`**, then **`plugin-validation`** polls the node's `status.capacity` for `nvidia.com/gpu` or `nvidia.com/mig-*` ≥ 1 (30 tries, 5 s apart) and writes **`plugin-ready`** → the validator's main container logs `all validations are successful` and the `ClusterPolicy` `.status.state` goes `ready`. All gating is via files in `/run/nvidia/validations` on the host, mounted `Bidirectional` into the writers.

- **Which component configures the container runtime, and what exactly breaks if it fails?** *Answer:* the **container-toolkit DaemonSet** (`nvidia-container-toolkit-ctr`, running `nvidia-toolkit`). It installs the runtime binaries to `/usr/local/nvidia/toolkit`, then writes `/etc/containerd/config.toml` **and** the drop-in `/etc/containerd/conf.d/99-nvidia.toml` (CRI-O: `/etc/crio/crio.conf.d/99-nvidia.conf`), adding `runtimes.nvidia`, `runtimes.nvidia-cdi` and `runtimes.nvidia-legacy` blocks with `BinaryName` pointing into its install dir, setting `default_runtime_name = "nvidia"` and `enable_cdi = true`, and generating CDI specs into `/var/run/cdi`. It then reloads containerd — default mode `signal`, i.e. SIGHUP, not a unit restart. If it fails: the *validator's* `toolkit-validation` cannot run `nvidia-smi` in a container, so `toolkit-ready` is never written and the device plugin, GFD, DCGM-exporter and MIG-manager all sit in `Init`. And even if a GPU pod somehow got scheduled, containerd would answer `failed to get sandbox runtime: no runtime for "nvidia" is configured`, or fall through to plain `runc` and give the container no devices at all.

- **What does the operator-validator actually validate, and in what order?** *Answer:* four init containers in sequence, each running the same `nvidia-validator` binary with a different `COMPONENT`. **`driver-validation`** (`WITH_WAIT=true`): tries a host driver first — `/host/usr/bin/nvidia-smi` exists, non-empty, and `chroot /host nvidia-smi` succeeds — and otherwise validates the containerized driver by locating the driver libraries and `nvidia-smi` under `/run/nvidia/driver` and running it with `LD_PRELOAD` pointed at `libnvidia-ml.so.1`; on success it also creates `/dev/char` symlinks (gpu-operator #430) and writes `driver-ready` with the five `NVIDIA_*_ROOT` variables. **`toolkit-validation`** (`WITH_WAIT=false`, `NVIDIA_VISIBLE_DEVICES=all`): runs `nvidia-smi` inside *this* container, which only works if the toolkit rewired the runtime and injected the driver; writes `toolkit-ready`. **`cuda-validation`**: creates the `nvidia-cuda-validator` pod whose init container runs `vectorAdd`, waits for phase `Succeeded` (60 tries, 5 s), writes `cuda-ready`. **`plugin-validation`**: polls the node's `status.capacity` for `nvidia.com/gpu` or a `nvidia.com/mig-*` resource ≥ 1 (30 tries, 5 s, logging `GPU resources are not yet discovered by the node, retry: N`), writes `plugin-ready`. Only then does the main container print `all validations are successful` and idle. Its `preStop` hook removes every `*-ready` file.

- **Where do you look first when the operator "ignored" a node, and why there?** *Answer:* the node's labels, specifically link 1 of the chain. `hasGPULabels()` in the controller accepts exactly `feature.node.kubernetes.io/pci-10de.present=true`, `…pci-0302_10de.present=true`, or `…pci-0300_10de.present=true` — three spellings because the GPU Operator's own NFD subchart sets `deviceLabelFields: [vendor]` (giving `pci-10de.present`) while upstream NFD defaults to `["class","vendor"]` (giving `pci-0302_10de.present`). If none of the three is present, no `nvidia.com/gpu.present` is stamped, no `gpu.deploy.*` switches are set, and no operand's `nodeSelector` matches — the node looks completely untouched. Check next for `nvidia.com/gpu.deploy.operands=false` (the whole-node kill switch) and for an unexpected `nvidia.com/gpu.deploy.<operand>=false`, which the operator honours rather than overwriting. Also check NFD's master allows the `nvidia.com` namespace (`extraLabelNs: ["nvidia.com"]`), because without it GFD's labels are silently dropped while the PCI label still appears — half-working, and confusing.

- **A 200-node fleet, 8 GPUs per node, driver bump with defaults. How long, and what does it cost?** *Answer:* per-node time is roughly image pull (~10 s warm-cache) + evict/rmmod (30–90 s) + resolve-headers-and-build (3–7 min) + modprobe/mount (~15 s) + the `startupProbe`'s 60 s initial delay + barrier chain re-converge (60–120 s) ≈ **9 min**. Effective rollout width is `min(maxParallelUpgrades, maxUnavailable)`, and `maxUnavailable` is scaled from total managed nodes with round-up — so at the chart defaults `min(1, ceil(0.25×200)=50) = 1`, giving 200 waves × 9 min ≈ **30 hours**. Raise `maxParallelUpgrades` to 20 and it's 10 waves × 9 min ≈ **1.5 hours**. GPU-hours lost, however, is `200 nodes × 8 GPUs × 9/60 h = 240 GPU-hours` **regardless of width** — width buys wall-clock, not money. At an illustrative $2.50/GPU-hour that's ~$600 per fleet-wide bump; `driver.usePrecompiled=true` cuts `T_node` to ~4.2 min and roughly halves it to ~112 GPU-hours (~$280). Two knobs, two different jobs: widen the wave to shorten the outage window, shorten `T_node` to cut the bill.

- **Why is `nvidia-mig-manager` missing on your L4 node, and how would you prove it's not a bug?** *Answer:* because the operator only stamps `nvidia.com/gpu.deploy.mig-manager=true` when `hasMIGCapableGPU()` returns true, and the MIG-manager DaemonSet's `nodeSelector` is exactly that label. That function short-circuits to false on vGPU nodes (`nvidia.com/vgpu.host-driver-version` present), then trusts GFD's `nvidia.com/mig.capable` label if it exists, and only as a fallback matches `nvidia.com/gpu.product` against the substrings `h100`, `a100`, `a30`. An L4 gets `nvidia.com/mig.capable=false` from GFD, so no label, so no pod. Prove it with `kubectl get node <node> -L nvidia.com/mig.capable -L nvidia.com/gpu.deploy.mig-manager` — `false` and empty respectively. Note the fallback substring list is narrower than the real set of MIG-capable silicon, which is another reason the GFD label (link 3 of the chain) needs to be working.

## Connections & what's next

This lesson is the map that 04.2 immediately stress-tests: every failure family in the next lesson is "one of these operands, broken, and here's how the logs prove which one." The barrier-file mechanism and the two waiting idioms are the backbone of that diagnosis. The `ClusterPolicy`/version discipline and the rollout arithmetic in §11 set up 04.5, where the same chain re-runs under a live upgrade with the `nvidia.com/<driver>-driver-upgrade-state` state machine driving it. The `nvidia.com/device-plugin.config` label and its `config-manager` SIGHUP mechanism (§9) are how 04.6 and 04.7 reconfigure MIG and time-slicing per node without restarting anything. And DCGM-exporter's `/var/lib/kubelet/pod-resources` mount (§10) is the single line that lesson 04.3 unpacks into a Go client and the capstone (04.10) turns into per-namespace dollars.

Next: **[04.2 · Crash-loop diagnosis](02-crash-loop-diagnosis.md)** takes this healthy chain, breaks it seven different ways, and gives you an ordered diagnostic procedure — command, expected output, interpretation of the deviation — that turns "the GPU node is broken" from a page you dread into a two-minute localization.

## References & further reading

**Primary sources — read these, not summaries of them**

1. [NVIDIA/gpu-operator](https://github.com/NVIDIA/gpu-operator), tag **v26.3.3** — the source of every default, path, container name, label and log string in this lesson. The three files that matter most: `deployments/gpu-operator/values.yaml`, `assets/state-*/0500_daemonset.yaml`, and `cmd/nvidia-validator/main.go` (barrier file names at `defaultStatusPath = "/run/nvidia/validations"`, and the four `Component` implementations). `controllers/state_manager.go` has the reconcile order, the `gpuNodeLabels` / `gpuStateLabels` maps and `hasMIGCapableGPU()`.
2. [NVIDIA/gpu-driver-container](https://github.com/NVIDIA/gpu-driver-container) — the driver container's `nvidia-driver` entrypoint script, per base image (e.g. `ubuntu24.04/nvidia-driver`). Source of `Resolving Linux kernel version...`, `Could not resolve Linux kernel version`, `Mounting NVIDIA driver rootfs...`, `Loading NVIDIA driver kernel modules...`, `Done, now waiting for signal`, and the `mount --rbind / /run/nvidia/driver` mechanism.
3. [NVIDIA/nvidia-container-toolkit](https://github.com/NVIDIA/nvidia-container-toolkit), tag **v1.19.1** — `cmd/nvidia-ctk-installer/container/runtime/containerd/` for `DefaultConfig = /etc/containerd/config.toml`, `DefaultDropInConfig = /etc/containerd/conf.d/99-nvidia.toml`, `DefaultSocket`, `DefaultRestartMode = "signal"`; `pkg/config/engine/containerd/` for the TOML shape and the config-version → CRI-plugin-key mapping; `internal/config/image/capabilities.go` for `NVIDIA_DRIVER_CAPABILITIES` defaults (`utility,compute`) and the supported set.
4. [NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin), tag **v0.19.3** — `internal/plugin/server.go` for the registration log lines quoted in the Worked example; `internal/lm/` for the complete GFD label set; the README's flag table for `DEVICE_ID_STRATEGY` (default `uuid`) and `DEVICE_LIST_STRATEGY` (default `envvar`).
5. [NVIDIA/k8s-driver-manager](https://github.com/NVIDIA/k8s-driver-manager) — the `k8s-driver-manager` init container. `Checking if the currently loaded NVIDIA driver version and configuration matches the desired state...`, `Could not unload NVIDIA driver kernel modules, driver is in use`, `Unable to cleanup driver modules, attempting again with node drain...` all live here; useful background for 04.5.
6. [kubernetes-sigs/node-feature-discovery](https://github.com/kubernetes-sigs/node-feature-discovery) — `source/pci/pci.go` for the label-construction algorithm reproduced in §4, and the upstream defaults `deviceClassWhitelist: ["03","0b40","12"]`, `deviceLabelFields: ["class","vendor"]`.
7. [NVIDIA/dcgm-exporter](https://github.com/NVIDIA/dcgm-exporter) — `internal/pkg/transformation/kubernetes.go` for the pod-resources join, and `pkg/cmd/app.go` for the `--pod-resources-kubelet-socket` default `/var/lib/kubelet/pod-resources/kubelet.sock`. This is the reference implementation lesson 04.3 has you re-derive.

**Real-world incidents**

8. [gpu-operator #430](https://github.com/NVIDIA/gpu-operator/issues/430) and [#485](https://github.com/NVIDIA/gpu-operator/issues/485) / [nvidia-container-toolkit #48](https://github.com/NVIDIA/nvidia-container-toolkit/issues/48) — the `/dev/char` symlink / `Failed to initialize NVML: Unknown Error` class. #430 is cited from inside the validator's own error message, which is unusually strong evidence; #485/#48 are the "NOTICE" threads with the systemd-cgroup mechanism and the `nvidia-ctk system create-dev-char-symlinks --create-all` fix.
9. [gpu-operator #1220 — "gpu-operator breaks when upgrading EKS to K8s v1.30"](https://github.com/NVIDIA/gpu-operator/issues/1220) — kernel drift under a managed control-plane upgrade. Title and substance confirmed via search this session; `github.com` HTML is not fetchable through this environment's proxy, so the thread was not re-read. The two log strings involved were verified independently against the driver-container script and containerd's source.
10. [CoreWeave — About GPU Driver Management in CKS](https://docs.coreweave.com/docs/products/cks/nodes/gpu-driver-management/gpu-driver-management-cks) — a managed GPU cloud's policy of owning this layer. *Not independently fetched this session* (proxy-blocked); corroborated by search and a dated changelog entry.

**Deeper dives**

11. [containerd](https://github.com/containerd/containerd) — `internal/cri/config/config.go` (`no runtime for %q is configured`) and `internal/cri/server/podsandbox/sandbox_run.go` (`failed to get sandbox runtime: %w`). Worth reading once so you recognise the error as containerd's, not NVIDIA's.
12. [NVIDIA GPU Operator documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/) — release notes, platform support, and the `ClusterPolicy` field reference. *Not fetchable this session* (`docs.nvidia.com` is proxy-blocked in this environment), which is precisely why every claim above is sourced from the repositories instead. Use the docs at study time to confirm which release line your cluster is on and whether the `NVIDIADriver` CRD has reached GA.
