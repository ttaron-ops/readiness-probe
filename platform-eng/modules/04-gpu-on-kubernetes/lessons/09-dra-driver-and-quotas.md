---
lesson: "04.9"
title: "The NVIDIA DRA driver and GPU quotas"
module: "04"
concept: "DRA driver install, ResourceClaim scheduling, GPU quotas"
status: not-started
est_time: "12h"
prev: "08-mps-choosing-sharing.md"
next: "10-capstone-per-pod-attribution.md"
artifacts: []
sources: 14
---

# 04.9 · The NVIDIA DRA driver and GPU quotas

> **Concept.** Install a real DRA driver, schedule a GPU through a ResourceClaim, and fence tenants with namespace quotas.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Lessons 06–08 spent their whole budget on one structural fact. The device plugin's contract with the kubelet is a **list of opaque strings and a count**, and everything downstream of that contract inherits its poverty. MIG (06) escapes by making the hardware itself mint one string per partition, so the string is a real identity all the way down to DCGM. Time-slicing (07) and MPS (08) do not escape: the plugin fabricates `GPU-<uuid>::0`, `::1`, `::2` and hands them out, which preserves a *scheduling* identity but leaves one device-level counter to be fanned out across N holders. Three lessons, one conclusion: **the device-plugin model can express "how many", and it cannot express "which one, with what properties, shared how, and to what extent."** It was never designed to.

This lesson is where that gets structurally addressed. Dynamic Resource Allocation is not a smarter device plugin — it is a different API family (`resource.k8s.io`) in which the allocation is a **first-class API object with a status**. You met the object model in module 02 and wrote a controller against it in the abstract. Here you install the real NVIDIA driver that publishes ResourceSlices from actual hardware, schedule a pod through a claim, and read the allocation back out of the API.

Then you fence tenants, because a claim-based scheduler with no quota just lets one namespace drain every GPU in the cluster — and DRA's quota story is genuinely different from the device plugin's in a way that has moved substantially since DRA went GA.

Be warned about one thing up front, because it is the most useful correction in this lesson: **DRA does not hand you a device UUID.** `status.allocation.devices.results[].device` is a driver-chosen name — for NVIDIA's driver that is `gpu-<minor>`, e.g. `gpu-0`. The UUID lives in the ResourceSlice as a device *attribute*. Getting a UUID out of DRA is a two-object join, and building your capstone on the assumption that the claim names the UUID directly will fail on contact with a real cluster.

## Why this matters

DRA is the industry's actual answer to "why can't I bill a GPU precisely", not a research proposal. Google Cloud's own writeup frames it exactly that way: the API exists so device management stops being "an integer plus a pile of node labels and hope" and becomes a scheduling primitive with structured attributes. Its core went GA in Kubernetes **1.34**, the `DynamicResourceAllocation` feature gate **locked to enabled-by-default in 1.35** (it cannot be turned off), and by **1.36** a large surface of the extension features had moved to beta or GA. This is no longer optional-curiosity territory; it is the direction the ecosystem has committed to.

The stakes for the capstone are concrete. A ResourceClaim's status is a **watchable Kubernetes object** that records what the scheduler decided, in the pod's namespace, with the pod in `status.reservedFor`. That is a fundamentally better substrate for an ownership map than polling a node-local gRPC socket: it is event-driven, it survives kubelet restarts, it is queryable cluster-wide from one place, and — with consumable capacity — it carries a per-**share** identity (`shareID`) for a device that several claims are using at once. That last one is the first mechanism in this entire module that gives a *shared* device a per-tenant identifier in the API. It is not a complete fix for the DCGM fan-out (the metric is still device-level), but it is the first structural crack in the problem.

For your résumé, "I ran DRA once" is a checkbox. "I can tell you what a claim's allocation actually names, why that is not a UUID, how to join it to one, which Kubernetes patch versions double-allocate devices, and why `ResourceQuota` on a DeviceClass counts the worst case rather than the allocation" is a staff-level answer that survives three follow-ups. And knowing the honest maturity position — GA core, beta-and-moving extensions, driver-side sharing still behind alpha gates — is itself interview-relevant, because both confident extremes ("rock solid" / "vaporware") read as someone who has not run it.

## What's new here (calibration)

Module 02 taught the DRA *object model* in the abstract — what a ResourceClaim, DeviceClass and ResourceSlice are — and you wrote a controller that reasons about them. That is not re-taught. What is new:

- **A real driver publishing real slices.** The DeviceClasses NVIDIA's driver creates, their exact CEL selectors, the device attributes it publishes (`uuid`, `productName`, `architecture`, `cudaComputeCapability`, `driverVersion`, PCIe root, NUMA node…), and the naming convention for devices in a pool.
- **The version landscape, verified against the Kubernetes changelogs**, not carried over from a draft. This includes a correction: the two DRA correctness bugs everyone cites shipped their fixes in **1.34.4**, not 1.34.2. That changes the safe-version rule.
- **What the allocation names, and the join to a UUID.** The single most load-bearing correction in this lesson, and the thing your capstone's DRA path depends on.
- **Per-claim sharing config, with its real gating.** `GpuConfig` with `sharing.strategy: TimeSlicing|MPS` is a per-workload declaration of the lesson 07/08 knobs — but `TimeSlicingSettings` and `MPSSupport` are **alpha, default-off** feature gates in the driver, which is why `unknown GPU sharing strategy: TimeSlicing` is a real, filed bug rather than a typo.
- **Consumable capacity and `shareID`** — DRA's native device-sharing model, enabled by default in Kubernetes 1.36, and the first per-share identity in the API.
- **Quota mechanics for a claim-based API, corrected.** Core `ResourceQuota` *does* limit devices per DeviceClass, via `<class>.deviceclass.resource.k8s.io/devices`, with worst-case accounting computed at claim admission. The previous version of this lesson said this did not exist and recommended a `ValidatingAdmissionPolicy` workaround; that guidance is out of date.
- **ComputeDomain / IMEX** — the multi-node-NVLink concept with no device-plugin equivalent at all.

## Core concepts

### 1 — What the integer cannot say

Before the API, the requirements. Here are five things a real GPU platform needs to express, and what the device-plugin model does with each:

| The request | Device plugin | DRA |
|---|---|---|
| "a GPU" | `nvidia.com/gpu: 1` ✅ | `deviceClassName: gpu.nvidia.com` ✅ |
| "a GPU with ≥ 40 GiB" | encode it in the *resource name* (`nvidia.com/a100-80gb`) or a node label + nodeSelector; the scheduler cannot reason about the number | CEL selector over a published device attribute/capacity ✅ |
| "two GPUs on the same NVLink island" | impossible — the plugin's `TopologyInfo` is NUMA-only; you approximate with node labels and hope | `constraints: [{matchAttribute: gpu.nvidia.com/parentUUID}]` (or an NVLink/partition attribute) ✅ |
| "this pod time-sliced, that pod exclusive, same node" | impossible — sharing config is node-wide in one ConfigMap | per-claim `opaque` config ✅ |
| "which device did I actually get?" | poll the kubelet's pod-resources socket out-of-band | `status.allocation.devices.results[]` in the API object ✅ |
| "devices spanning several nodes on one fabric" | no vocabulary at all | `ComputeDomain` + IMEX ✅ |

The row that matters most for the capstone is the fifth. With the device plugin you *reconstruct* ownership by polling a node-local socket; with DRA the scheduler *records* it in an object you can watch. And the row that matters most for the next three years is the sixth, which is the case where DRA is not a nicer API for the same job but the only API that can express the job at all.

### 2 — The object graph

Four object kinds, all in `resource.k8s.io/v1` since the 1.34 GA (with `v1beta1` and `v1beta2` still served for compatibility, and `v1alpha3` reduced to holding only `DeviceTaintRule`).

```
  THE DRA OBJECT GRAPH — WHO WRITES WHAT, AND WHO READS IT

  cluster-scoped                                namespace-scoped
  ──────────────                                ────────────────

  ┌───────────────────────────┐        ┌──────────────────────────────┐
  │ DeviceClass               │        │ ResourceClaimTemplate        │
  │  name: gpu.nvidia.com     │◀───────│  spec.spec.devices.requests  │
  │  spec.selectors[].cel     │  by    │   [0].exactly.               │
  │  spec.extendedResourceName│  name  │      deviceClassName ────────┘
  │                           │        │                              │
  │  WRITTEN BY: the driver's │        │  WRITTEN BY: you             │
  │   Helm chart (install)    │        │  READ BY: the ResourceClaim  │
  │  A CATALOGUE ENTRY: "what │        │   controller, which mints a  │
  │   counts as a GPU here"   │        │   fresh per-Pod claim        │
  └───────────────────────────┘        └───────────────┬──────────────┘
                                                        │ creates
  ┌───────────────────────────┐        ┌───────────────▼──────────────┐
  │ ResourceSlice             │        │ ResourceClaim                │
  │  spec.driver              │        │  metadata.name  pod1-gpu-x9z │
  │  spec.nodeName / allNodes │        │  spec.devices.requests[]     │
  │  spec.pool{name,          │        │  ──────────────────────────  │
  │       generation,         │        │  status.allocation           │
  │       resourceSliceCount} │        │    .devices.results[]        │
  │  spec.devices[]           │   the  │       .driver  gpu.nvidia.com│
  │    - name: gpu-0  ◀───────┼───same─┼──────▶.pool    <node-name>   │
  │      attributes:          │  triple│       .device  gpu-0         │
  │        uuid: GPU-8f2c…    │        │       .shareID <uuid|nil>    │
  │        productName: …     │        │       .consumedCapacity{}    │
  │        architecture: …    │        │    .nodeSelector             │
  │      capacity:            │        │  status.reservedFor[]        │
  │        memory: 80Gi       │        │    - resource: pods          │
  │                           │        │      name: pod1  uid: …      │
  │  WRITTEN BY: the driver's │        │  status.devices[] (device    │
  │   kubelet plugin, per node│        │    status reported by driver)│
  │  READ BY: the scheduler   │        │                              │
  └───────────────────────────┘        │  spec WRITTEN BY: controller │
                                       │  status WRITTEN BY: scheduler│
                                       │   (+ driver for .devices[])  │
                                       └──────────────────────────────┘

  THE IDENTITY TRIPLE:  <driver>/<pool>/<device>
      gpu.nvidia.com / gpu-node-01 / gpu-0
  That triple is what the allocation names. NOT a UUID.
  To get the UUID you must look up devices[].attributes.uuid in
  the ResourceSlice for that (driver, pool, device). See §6.
```

**DeviceClass** is a cluster-scoped catalogue entry: a named set of CEL selectors that decide which devices are in the class. NVIDIA's chart installs these two for GPUs (verbatim from `deployments/helm/dra-driver-nvidia-gpu/templates/deviceclass-gpu.yaml` and `deviceclass-mig.yaml`):

```yaml
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: gpu.nvidia.com
spec:
  selectors:
  - cel:
      expression: "device.driver == 'gpu.nvidia.com' && device.attributes['gpu.nvidia.com'].type == 'gpu'"
  extendedResourceName: nvidia.com/gpu     # ← only rendered on resource.k8s.io/v1
---
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: mig.nvidia.com
spec:
  selectors:
  - cel:
      expression: "device.driver == 'gpu.nvidia.com' && device.attributes['gpu.nvidia.com'].type == 'mig'"
```

Note `extendedResourceName: nvidia.com/gpu` on the GPU class. That is the **DRA extended-resource mapping** (`DRAExtendedResource`, beta in 1.36), and its effect is significant: a pod that writes the old-fashioned `resources.limits: {nvidia.com/gpu: 1}` — with no claim, no template, no knowledge that DRA exists — can be satisfied by a DRA device from this class. It is the migration bridge. It also means a cluster can be running DRA underneath workloads that were never rewritten, which is worth knowing before you conclude from pod specs that nobody is using DRA.

**ResourceSlice** is what the driver's per-node kubelet plugin publishes: the inventory. Here is what NVIDIA's driver actually puts in one, assembled from `cmd/gpu-kubelet-plugin/deviceinfo.go` (the attribute names and the `gpu-<minor>` naming are from the source; the values are representative):

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
metadata:
  name: gpu-node-01-gpu.nvidia.com-7bk4x
spec:
  driver: gpu.nvidia.com
  nodeName: gpu-node-01              # node-local pool
  pool:
    name: gpu-node-01                # NVIDIA uses the node name as the pool
    generation: 3                    # bumped on every republish; the scheduler
                                     # ignores slices from an incomplete pool
    resourceSliceCount: 1            # how many slices make up this pool
  devices:
  - name: gpu-0                      # ← CanonicalName() = "gpu-<minor>"
    attributes:
      type:                  { string: "gpu" }
      uuid:                  { string: "GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763" }
      productName:           { string: "NVIDIA H100 80GB HBM3" }
      brand:                 { string: "NVIDIA" }
      architecture:          { string: "Hopper" }
      cudaComputeCapability: { version: "9.0.0" }
      driverVersion:         { version: "580.82.7" }
      cudaDriverVersion:     { version: "13.0.0" }
      resource.kubernetes.io/pcieRoot: { string: "pci0000:0f" }
      # numaNode / addressingMode / gpuModuleID / partitionN also appear,
      # the last two only with the driver's FabricManagerPartitioning gate on
    capacity:
      memory:                { value: "80Gi" }
  - name: gpu-1
    attributes:
      uuid: { string: "GPU-a4e17b22-…" }
      # …
```

Two design details worth understanding, because they cause real confusion:

- **The device name is `gpu-<minor>`, not the UUID.** The driver's own comment says there is "quite a bit of history to using the minor number for device announcement". Minor numbers are short, DNS-label-safe, and stable across a boot; UUIDs are 40+ characters and would make the `<driver>/<pool>/<device>` triple unwieldy. The consequence is §6.
- **`pool.generation` and `resourceSliceCount` are how the scheduler knows the inventory is complete.** A driver that publishes N slices for a pool sets `resourceSliceCount: N` and a common `generation`; the scheduler will not allocate from a pool whose slices do not add up, which is what prevents allocation against a half-published inventory during a driver restart.

**ResourceClaim** and **ResourceClaimTemplate.** Use the template in almost every case. A bare ResourceClaim is a namespace-scoped, shared, long-lived object — it outlives pods and can be referenced by several — whereas a template mints a fresh claim per pod, garbage-collected with it. Getting this wrong is how you end up with a claim permanently `allocated,reserved` to a pod that no longer exists.

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  namespace: gpu-test1
  name: single-gpu
spec:
  spec:                              # note the double spec: template.spec.spec
    devices:
      requests:
      - name: gpu                    # request name; referenced from containers
        exactly:                     # the GA request shape. The alternative is
                                     # firstAvailable: [...] (prioritised list)
          deviceClassName: gpu.nvidia.com
          allocationMode: ExactCount # ExactCount (default) | All
          count: 1
          selectors:                 # optional: narrow within the class
          - cel:
              expression: "device.attributes['gpu.nvidia.com'].architecture == 'Hopper'"
          # adminAccess: true        # DRAAdminAccess (GA in 1.36) — see §4
---
apiVersion: v1
kind: Pod
metadata: { namespace: gpu-test1, name: pod1 }
spec:
  resourceClaims:                    # pod-level: name the claim(s)
  - name: gpu
    resourceClaimTemplateName: single-gpu
  containers:
  - name: ctr
    image: ubuntu:22.04
    command: ["bash", "-c"]
    args: ["nvidia-smi -L; trap 'exit 0' TERM; sleep 9999 & wait"]
    resources:
      claims:                        # container-level: opt in to a pod claim
      - name: gpu
      # - name: gpu
      #   request: gpu               # optional: opt into ONE request of a
      #                              # multi-request claim (see gpu-test5, §7)
  tolerations:
  - { key: nvidia.com/gpu, operator: Exists, effect: NoSchedule }
```

The two-level structure (`spec.resourceClaims` names them, `containers[].resources.claims` opts in) is deliberate: it lets several containers in one pod share one allocated device, or take different requests from one claim. NVIDIA's `demo/specs/quickstart/v1/gpu-test2.yaml` is exactly the two-containers-one-GPU case, and `gpu-test4.yaml` is the four-containers-four-MIG-devices-with-a-`matchAttribute`-constraint case.

Two constraints on the API you should know cold because they are hard ceilings: **at most 32 entries in `allocation.devices.results`** (`AllocationResultsMaxSize`), and **at most 256 entries in `status.reservedFor`** unless the workload API is in play. Both are in the upstream types.

### 3 — The scheduling sequence

Here is what actually happens, in order. This is the diagram to have in your head when a claim sits `pending` forever.

```
  A POD WITH A CLAIM, FROM SUBMIT TO RUNNING

  ┌ driver's kubelet plugin (per node), continuously ─────────────────┐
  │ 0. discover devices via NVML → publish/refresh ResourceSlice      │
  │    for pool <node>, generation++, resourceSliceCount              │
  └───────────────────────────────────────────────────────────────────┘

  1. you create Pod (+ ResourceClaimTemplate reference)
                          │
  2. resourceclaim controller (kube-controller-manager)
     └─▶ creates ResourceClaim  "pod1-gpu-<hash>"
         ownerRef: the Pod  (so it is GC'd with the Pod)
                          │
  3. kube-scheduler, DynamicResources plugin
     ├─ PreEnqueue/PreFilter: are all the pod's claims resolvable?
     ├─ Filter (per node): can this node's ResourceSlices satisfy
     │    every request, honouring selectors + constraints +
     │    device taints? Runs CEL. Times out at FilterTimeout
     │    (default 10s, configurable) to bound pathological cases.
     ├─ Reserve: pick concrete devices; first-fit, devices considered
     │    in lexicographical order
     └─ PreBind: WRITE status.allocation and status.reservedFor
                to the ResourceClaim via the API server
                          │
              ┌───────────┴─────────────┐
              │ ⚠ if the device has     │
              │   bindingConditions,    │
              │   binding WAITS for an  │
              │   external actor to set │
              │   them True (default    │
              │   timeout 600s;         │
              │   DynamicResourcesArgs. │
              │   bindingTimeout)       │
              └───────────┬─────────────┘
                          │
  4. Bind: pod.spec.nodeName = <node>
                          │
  5. kubelet on that node
     ├─ reads the claim's allocation
     ├─ calls the driver's NodePrepareResources (DRA kubelet gRPC,
     │    v1 since 1.34; v1beta1 deprecated)
     │    └─ driver: create/attach the device, apply the claim's
     │       opaque config (sharing! §7), emit a TRANSIENT CDI spec
     │       named  k8s.gpu.nvidia.com/claim=<claimUID>-gpu-0
     ├─ passes the CDI device name to the container runtime (04.4)
     └─ starts the container
                          │
  6. pod Running. nvidia-smi -L inside the container shows the GPU.
     pod-resources API now reports, per container, a
     DynamicResource{claim_name, claim_namespace,
                     claim_resources[]{cdi_devices[], driver_name,
                                       pool_name, device_name,
                                       share_id}}
                     ↑ this is the capstone's second ownership source

  ── WHERE IT HANGS, AND WHAT THAT MEANS ──
   Pod Pending, claim has no status.allocation
       → scheduler could not satisfy it. Check: are there
         ResourceSlices at all? does the CEL selector match any
         device's attributes? is a device taint blocking it?
   Claim allocated, pod Pending
       → bindingConditions waiting, or the node became unfit
   Pod stuck ContainerCreating
       → NodePrepareResources failing. Driver logs. Usually CDI
         not enabled in the runtime, or nvidiaDriverRoot wrong.
```

Two facts from that sequence that people get wrong: **preemption is not supported for DRA resources** in the current implementation, so a high-priority pod will not evict a low-priority one to free a device; and **pre-scheduled pods** (with `spec.nodeName` set directly, bypassing the scheduler) do not get an allocation from the scheduler — the kubelet will keep retrying and the pod will not start. Both are worth knowing before you design a batch system on top of DRA.

### 4 — The version landscape, verified

This is the section to re-check at study time. Everything below was verified against the Kubernetes `CHANGELOG/CHANGELOG-1.34.md`, `-1.35.md` and `-1.36.md` in the `kubernetes/kubernetes` repository, and against the upstream API types, in **August 2026**. At that point 1.36 is the current stable line and 1.37 is imminent.

**GA and gate locking.**

- DRA core (structured parameters) **graduated to GA in 1.34** (PR #132706). The API group is `resource.k8s.io`, and `v1` is the served version for `DeviceClass`, `ResourceClaim`, `ResourceClaimTemplate` and `ResourceSlice`. `v1beta1` and `v1beta2` remain served; `v1alpha3` was reduced to holding only `DeviceTaintRule`.
- In **1.35** the `DynamicResourceAllocation` feature gate was **locked to enabled-by-default and cannot be disabled** (PR #134452). There is no "turn DRA off" switch from 1.35 onward.
- The kubelet-facing DRA gRPC API **graduated to v1 in 1.34**, with `v1beta1` deprecated (PR #132700). A driver built against the 1.32+ helper supports both.
- `KubeletPodResourcesDynamicResources` and `KubeletPodResourcesGet` were **promoted to beta in 1.34 and are enabled by default when DRA is GA** (PR #132940). This is why the pod-resources API can report DRA claims at all — the capstone depends on it.

**The correctness bugs, and the right version rule. This is a correction.** Two real DRA bugs are widely cited as "the 1.34 caveat":

- A goroutine race in which, under rapid pod scheduling, **the same device could be allocated twice to different ResourceClaims**. Depending on whether the driver checks for this in `NodePrepareResources` (it should, but not all do), the second pod either failed to start or — worse — ran in parallel on a device someone else owned. Fixed by **PR #136566**.
- The **kubelet mishandling a pod with multiple ResourceClaims when one of them was already prepared.** Fixed by **PR #136480**.

Both of those PRs appear in the **v1.34.4** changelog section, under "Changelog since v1.34.3". They are **not** in 1.34.2 — 1.34.2's DRA fix is a different one (PR #133934, a kubelet deadlock making a DRA driver connection unusable after 30 idle minutes). The previous version of this lesson said the fixes landed in 1.34.2; that is wrong, and the practical rule changes accordingly:

```
  SAFE KUBERNETES VERSIONS FOR DRA WORKLOADS  (as of Aug 2026)

   1.33.x            ✅ DRA beta, no double-allocation race
   1.34.0 – 1.34.3   ❌ double-allocation race (#136566) and the
                        multi-claim kubelet bug (#136480) are BOTH
                        still present. Silent correctness bugs, not
                        crashes — the worst kind to find in prod.
   ≥ 1.34.4          ✅ both fixed. 1.34.10 is the latest 1.34 patch
                        at the time of writing.
   1.35.x            ✅ gate locked on; DRAExtendedResource quota
                        accounting added (#134210)
   1.36.x            ✅ current stable. Most extensions beta/GA (below)

   RULE: 1.33.x, or ≥ 1.34.4. Never 1.34.0–1.34.3.
   Run `kubectl version` before anything else on a new cluster, and
   re-derive this table from CHANGELOG-1.3x.md — do not trust a
   lesson's version numbers, including this one's.
```

**The extension feature gates**, with status as of the 1.36 line. These matter because "DRA is GA" is true of the core and not of most of what you will actually want:

| Gate | Status | Notes |
|---|---|---|
| `DynamicResourceAllocation` | **GA**, locked on since 1.35 | the core |
| `DRAAdminAccess` | **GA** in 1.36 (PR #137373) | privileged claims that can reach in-use devices, for monitoring/health tooling; requires the namespace label `resource.kubernetes.io/admin-access: "true"` |
| `DRAPrioritizedList` | **GA** in 1.36 (PR #136924) | `firstAvailable: [...]` subrequests; scheduler takes the first allocatable |
| `DRADeviceTaints` | **beta** in 1.36 (PR #137170) | taints on devices in ResourceSlices, on by default |
| `DRADeviceTaintRules` | separate gate; `DeviceTaintRule` needs `resource.k8s.io/v1beta2` | split out in 1.35 (PR #135068) so slice-based taints can stay on while rules stay off |
| `DRAConsumableCapacity` | **enabled by default** in 1.36 (PR #136611) | multi-allocatable devices, `shareID`, `consumedCapacity` — see §8 |
| `DRAPartitionableDevices` | **beta** in 1.36 (PR #137350) | `sharedCounters` / `consumesCounters`; note the 1.35 change (#134189) was backwards-incompatible and required removing affected ResourceSlices before up/downgrade |
| `DRAExtendedResource` | **beta** in 1.36 (PR #135048) | `DeviceClass.spec.extendedResourceName`; adds quota keys (§10) |
| `DRADeviceBindingConditions` | **beta** in 1.36 (PR #137795) | delayed binding on external readiness signals |
| `DRAResourceClaimDeviceStatus` | beta | driver-reported `status.devices[]` on a claim |
| `DRAResourceClaimGranularStatusAuthorization` | **beta** in 1.36 | **ACTION REQUIRED** — see below |
| `DRAResourcePoolStatus`, `DRAListTypeAttributes`, `DRANodeAllocatableResources`, `DRAWorkloadResourceClaims` | alpha | not for production |

**The 1.36 RBAC action-required is the one that will break your driver upgrade.** With `DRAResourceClaimGranularStatusAuthorization` (beta in 1.36), DRA drivers and controllers need *granular* permissions to update claim statuses: schedulers and controllers must be granted `update`/`patch` on **`resourceclaims/binding`**, and DRA drivers must be granted **`associated-node:update`** or **`arbitrary-node:update`** (or the patch equivalents) on **`resourceclaims/driver`**, restricted by their specific `resourceNames` (PR #134947). If a driver that worked on 1.35 starts failing to write claim status on 1.36, this is why. Check the driver chart's RBAC before you upgrade the control plane, not after.

### 5 — Installing the real NVIDIA driver

**The repo and chart history, because the renaming trips everyone.** The driver began as `NVIDIA/k8s-dra-driver-gpu` with its own CalVer tag scheme: `v25.3.0/.1/.2`, `v25.8.0/.1`, `v25.12.0`. **Starting at v0.4.0 it moved to `kubernetes-sigs/dra-driver-nvidia-gpu`, adopted semantic versioning, and the Helm chart was renamed from `nvidia-dra-driver-gpu` to `dra-driver-nvidia-gpu`**, published to new registries.

Two consequences you will hit:

- **"GPU Operator 25.8 folds the DRA driver in-tree" is false, and it is a version-number collision.** The public GPU Operator line went `v25.3.x → v25.10.x → v26.3.x` (v26.3.3 is the latest tag at the time of writing). There is no public Operator 25.8. "25.8" is a *DRA-driver* CalVer tag. And as a check on the substance rather than the numbering: **the GPU Operator's `ClusterPolicy` CRD at v26.3.3 contains no DRA field at all** — verified directly against `deployments/gpu-operator/crds/nvidia.com_clusterpolicies.yaml`. The driver installs as a companion chart. NVIDIA's own documentation describes Operator-managed installation as the *preferred future* method; treat any claim that a given Operator version makes the standalone chart unnecessary as something to verify in that version's release notes.
- **Search results and docs disagree on the chart name and version scheme**, because half of them predate the move. Older material says `nvidia/nvidia-dra-driver-gpu --version 25.12.0`; current material says `dra-driver-nvidia-gpu` at a semver tag.

**Current state, verified.** The latest release tag in `kubernetes-sigs/dra-driver-nvidia-gpu` is **v0.4.1**; the `main` branch's chart declares `version: 0.5.0-dev` / `appVersion: 0.5.0-dev`. The chart's `kubeVersion` constraint is **`>=1.32.0-0`**, and the repo's own feature-gate registry pins its component-base emulation version at Kubernetes **1.37**, which tells you which kube line `main` is being developed against. Chart values of interest, from `deployments/helm/dra-driver-nvidia-gpu/values.yaml`:

| Value | Default | Meaning |
|---|---|---|
| `resources.gpus.enabled` | `true` | publish GPU (and MIG) devices |
| `resources.computeDomains.enabled` | `true` | publish ComputeDomain support (§9); IMEX mode `driverManaged`, isolation `domain` |
| `nvidiaDriverRoot` | `/` | `/` for a host-installed driver, `/run/nvidia/driver` when the GPU Operator's driver container owns it |
| `nvidiaCDIHookPath` | `""` | optional path to `nvidia-cdi-hook` |
| `image.repository` | `registry.k8s.io/dra-driver-nvidia/dra-driver-nvidia-gpu` | note: `registry.k8s.io`, not NGC |
| `image.tag` | `""` | defaults to chart `appVersion` with a `v` prefix |
| `resourceApiVersion` | auto-detect | picks the highest served of `v1 > v1beta2 > v1beta1`; override it when a cluster misreports `.Capabilities.APIVersions` |
| `webhook.enabled` | `false` | TLS via `cert-manager` (default) or a `secret` |
| `consumableShares` | `""` | `"disabled"`, `"memory"`, `"unlimited"`, or a positive integer (§8) |
| `featureGates` | `{}` | driver-side gates, including `ConsumableShares`, `DRAListTypeAttributes`, `NVMLDeviceHealthCheck`, `HostManagedIMEXDaemon` |
| `logVerbosity` | `"4"` | 0–7 |

Install:

```bash
# Prereqs, all of which you must actually check:
#  1. Kubernetes 1.33.x or >= 1.34.4  (see §4)
#  2. chart kubeVersion >= 1.32.0-0
#  3. CDI enabled in the container runtime (containerd: enable_cdi = true) — 04.4
#  4. An NVIDIA datacenter driver new enough for the DRA-driver release you pin.
#     This minimum HAS MOVED (>=565 at one point, >=580 later). Read the release
#     notes for the exact tag you install; do not memorise a number.

# The chart is published both to NGC and to an OCI registry under registry.k8s.io.
# OCI form (matches the image repository above):
helm install dra-driver-nvidia-gpu \
  oci://registry.k8s.io/dra-driver-nvidia/charts/dra-driver-nvidia-gpu \
  --version v0.4.1 \
  --create-namespace --namespace dra-driver-nvidia-gpu \
  --set resources.gpus.enabled=true \
  --set nvidiaDriverRoot=/run/nvidia/driver

# NGC form (also valid; verify the current chart name before pinning):
# helm repo add nvidia https://helm.ngc.nvidia.com/nvidia && helm repo update
# helm search repo nvidia/dra-driver-nvidia-gpu --versions | head

# Verify — in this order, because each step depends on the last:
kubectl -n dra-driver-nvidia-gpu get pods            # controller + node plugin Running
kubectl get deviceclasses                            # gpu.nvidia.com, mig.nvidia.com,
                                                     # compute-domain.nvidia.com
kubectl get resourceslices                           # MUST be non-empty
kubectl get resourceslices -o yaml | \
  yq '.items[].spec.devices[] | {name: .name, uuid: .attributes.uuid.string}'
```

That last command is the one to run first when anything is wrong: **if `kubectl get resourceslices` is empty, nothing else can possibly work**, and the cause is almost always the node plugin failing to reach NVML (wrong `nvidiaDriverRoot`) or CDI not being enabled in the runtime.

**Driver-side feature gates you will care about**, from `pkg/featuregates/featuregates.go` — all registered with `Default: false, PreRelease: Alpha` unless noted:

| Driver gate | Default | What it unlocks |
|---|---|---|
| `TimeSlicingSettings` | **false (alpha)** | `sharing.strategy: TimeSlicing` with `timeSlicingConfig.interval` |
| `MPSSupport` | **false (alpha)** | `sharing.strategy: MPS` with `mpsConfig.*` |
| `ConsumableShares` | false | publish consumable capacity + `AllowMultipleAllocations` (§8). Note: **MPS sharing is not supported when consumable shares is enabled** |
| `DynamicMIG` | false | dynamic MIG device management |
| `PassthroughSupport` | false | configure GPUs with `vfio-pci` |
| `FabricManagerPartitioning` | false | NVSwitch partition management; publishes `gpuModuleID` and `partitionN` attributes; requires Fabric Manager with `FABRIC_MODE=1` |
| `NVMLDeviceHealthCheck` | false | GPU health checking via NVML |
| `IMEXDaemonsWithDNSNames` | **true (beta)** | DNS names instead of raw IPs for IMEX daemons |
| `DRAListTypeAttributes` | false | list-valued device attributes; needs the matching *Kubernetes* gate too |

**Read the first two rows again.** Per-claim time-slicing and MPS — the headline "DRA fixes sharing configuration" feature — are behind **alpha, default-off** gates in the driver as of `main`. That is why `unknown GPU sharing strategy: TimeSlicing` is a filed issue (`NVIDIA/k8s-dra-driver-gpu` #762) rather than a typo: the config is accepted by the API server and rejected by the driver because the gate is off. Enable it explicitly with `--set featureGates.TimeSlicingSettings=true` (and/or `MPSSupport=true`) and expect pre-beta rough edges.

### 6 — What the allocation actually names, and how to get a UUID

This is the correction that matters most for the capstone.

```bash
$ kubectl -n gpu-test1 get resourceclaims
NAME               STATE                AGE
pod1-gpu-x9zq2     allocated,reserved   12s

$ kubectl -n gpu-test1 get resourceclaim pod1-gpu-x9zq2 -o yaml | yq '.status'
allocation:
  devices:
    results:
    - request: gpu
      driver: gpu.nvidia.com
      pool: gpu-node-01
      device: gpu-0              # ← NOT a UUID. A driver-chosen DNS label.
      adminAccess: false
  nodeSelector:
    nodeSelectorTerms:
    - matchFields:
      - key: metadata.name
        operator: In
        values: ["gpu-node-01"]
  allocationTimestamp: "2026-08-17T09:14:02Z"
reservedFor:
- resource: pods
  name: pod1
  uid: 4a1f8e02-…
```

`device: gpu-0`. The upstream type documents the semantics explicitly: pool "together with the driver name and the device name field identify which device was allocated (`<driver name>/<pool name>/<device name>`)", and device "references one device instance via its name in the driver's resource pool. It must be a DNS label." A UUID is not a DNS label — hence `gpu-<minor>`.

So the attribution path under DRA is a **join, not a lookup**:

```
  GETTING FROM A CLAIM TO A DCGM SERIES — THE DRA JOIN PATH

  ResourceClaim (namespaced)              ResourceSlice (cluster-scoped)
  ┌────────────────────────────┐          ┌──────────────────────────────┐
  │ metadata.namespace  team-a │          │ spec.driver  gpu.nvidia.com  │
  │ metadata.name  pod1-gpu-x9 │          │ spec.pool.name  gpu-node-01  │
  │ status.reservedFor[]        │          │ spec.devices[]               │
  │   pods/pod1  uid=4a1f…      │          │  - name: gpu-0               │
  │ status.allocation           │          │    attributes:               │
  │  .devices.results[0]        │          │      uuid: GPU-8f2c1d90-…    │
  │    driver  gpu.nvidia.com ──┼──┐   ┌───┼──▶  productName: H100 80GB   │
  │    pool    gpu-node-01 ─────┼──┤   │   │      memory: 80Gi            │
  │    device  gpu-0 ───────────┼──┤   │   └──────────────────────────────┘
  │    shareID <uuid|nil> ──┐   │  │   │
  └─────────────────────────┼───┘  └───┴── JOIN KEY:
                            │              (driver, pool, device)
                            │                        │
                            │                        ▼
                            │              uuid = "GPU-8f2c1d90-…"
                            │                        │
                            │        ┌───────────────┴────────────────┐
                            │        ▼                                ▼
                            │  DCGM_FI_PROF_GR_ENGINE_ACTIVE   nvidia-smi -L
                            │    {UUID="GPU-8f2c1d90-…"}       inside the pod
                            │                │                 (cross-check)
                            │                ▼
                            │        ONE value per PHYSICAL device
                            └────────────────┬───────────────────────
                                             │
             If shareID is set (consumable capacity, §8), several
             claims map to the SAME uuid with DIFFERENT shareIDs.
             The API now distinguishes the shares.
             DCGM STILL DOES NOT.  ← the fan-out survives DRA.

  ── THE SHORTCUT: pod-resources reports the triple directly ──
   ContainerResources.dynamic_resources[] →
     DynamicResource{claim_name, claim_namespace, claim_resources[]}
     ClaimResource{cdi_devices[], driver_name, pool_name,
                   device_name, share_id}
   …so a single List() on the kubelet socket gives you
   pod → (driver, pool, device, share_id) for DRA-allocated devices,
   WITHOUT watching claims. You still need the ResourceSlice join to
   reach the UUID. (Requires KubeletPodResourcesDynamicResources,
   beta and on by default when DRA is GA.)
```

Three practical routes to the UUID, in the order you should try them:

1. **ResourceSlice join** (shown above). Watch `ResourceSlice` with an informer, build `(driver, pool, device) → uuid` from `devices[].attributes.uuid.string`, and join the claim's allocation to it. Purely API-driven, works from anywhere in the cluster, no node access. This is the route to build.
2. **Claim `status.devices[]`** — with `DRAResourceClaimDeviceStatus` (beta), the driver can report per-device status on the claim, including driver-specific data. Convenient when available, but it is optional for a driver to populate, so do not depend on it as your only path.
3. **CDI device name from pod-resources.** `ClaimResource.cdi_devices[]` contains the transient CDI qualified name the driver generated. NVIDIA's driver builds it as `k8s.gpu.nvidia.com/claim=<claimUID>-<deviceCanonicalName>` — e.g. `k8s.gpu.nvidia.com/claim=4a1f8e02-…-gpu-0` (from `cmd/gpu-kubelet-plugin/cdi.go`: vendor `k8s.` + driver name, class `claim`, name `<claimUID>-<canonicalName>`). Useful for correlating with runtime state; still not a UUID.

And the cross-check you should always run once per cluster: `kubectl logs pod1 | grep UUID` against `nvidia-smi -L` inside the pod, compared with the ResourceSlice attribute and (if you also run the device plugin) the pod-resources device ID. **Four sources, one UUID** — if they disagree you have found something worth understanding before you build on it.

Note what DRA does and does not fix, precisely:

- ✅ **Ownership is declarative, event-driven, and cluster-visible.** No node-local socket poll required for the *map*.
- ✅ **A shared device can have per-share identity** in the API (`shareID`), which the device plugin's `::N` annotation only ever had inside the kubelet.
- ❌ **The measurement is unchanged.** DCGM still programs counters per physical device. A time-sliced or MPS-shared GPU allocated through three claims still reports one `GR_ENGINE_ACTIVE`, and `dcgm-exporter` (with `--kubernetes-enable-dra` / `KUBERNETES_ENABLE_DRA=true`) will still fan it out across the holders. **Lesson 07's arithmetic applies to DRA clusters without modification.**

### 7 — Per-claim sharing config

Sharing moves from a node-wide ConfigMap to the claim, expressed as an `opaque` config blob the driver interprets. This is NVIDIA's own `demo/specs/quickstart/v1/gpu-test5.yaml`, reproduced because it is the canonical example of both strategies in one claim:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata: { namespace: gpu-test5, name: multiple-gpus }
spec:
  spec:
    devices:
      requests:
      - name: ts-gpu
        exactly: { deviceClassName: gpu.nvidia.com }
      - name: mps-gpu
        exactly: { deviceClassName: gpu.nvidia.com }
      config:
      - requests: ["ts-gpu"]              # config scoped to ONE request
        opaque:
          driver: gpu.nvidia.com          # who interprets `parameters`
          parameters:
            apiVersion: resource.nvidia.com/v1beta1
            kind: GpuConfig
            sharing:
              strategy: TimeSlicing       # requires driver gate
                                          # TimeSlicingSettings=true
              timeSlicingConfig:
                interval: Long            # Default|Short|Medium|Long
                                          # → 0|1|2|3, the same values as
                                          # nvidia-smi compute-policy
                                          # --set-timeslice (lesson 07 §1)
      - requests: ["mps-gpu"]
        opaque:
          driver: gpu.nvidia.com
          parameters:
            apiVersion: resource.nvidia.com/v1beta1
            kind: GpuConfig
            sharing:
              strategy: MPS               # requires driver gate
                                          # MPSSupport=true
              mpsConfig:
                defaultActiveThreadPercentage: 50
                defaultPinnedDeviceMemoryLimit: 10Gi
                # defaultPerDevicePinnedMemoryLimit:
                #   "0": 10Gi             # per index or UUID; overrides
                #   "GPU-8f2c…": 20Gi     # the default above
                # multiUser: false        # allow one MPS server to serve
                                          # clients with different uids
```

Every field there is real, taken from `api/nvidia.com/resource/v1beta1/sharing.go` in the driver: `GpuSharing{Strategy, TimeSlicingConfig, MpsConfig}`, `TimeSlicingConfig{Interval}`, and `MpsConfig{DefaultActiveThreadPercentage, DefaultPinnedDeviceMemoryLimit, DefaultPerDevicePinnedMemoryLimit, MultiUser}`. There is also a distinct `MigDeviceSharing{Strategy, MpsConfig}` type for MIG devices — which is the primary-source proof that **MPS-on-MIG is modelled by the DRA driver even though the device plugin refuses it** (lesson 08 §5).

This is a real improvement, and it is worth naming exactly what improved: **the sharing knobs from lessons 07 and 08 become per-workload declarations instead of per-node policy.** Lesson 08 §6 argued that billing tenants at their MPS ceiling is defensible *if the tenant chose the ceiling*; DRA is the mechanism that lets them. A claim that asks for `defaultActiveThreadPercentage: 50` is a workload stating its own cost basis in an API object, and your cost operator can read it straight off the claim.

Two caveats, both load-bearing:

1. **The gates are alpha and off.** `TimeSlicingSettings` and `MPSSupport` are both `Default: false, PreRelease: Alpha`. Without them the driver rejects the config with `unknown GPU sharing strategy: TimeSlicing` (issue #762). Enable deliberately; do not build a product on it yet.
2. **Per-claim config does not repeal the fan-out.** A time-sliced claim still means several pods on one physical UUID at the DCGM level. What DRA fixes is *who asked for what and got what*. What fraction of the shared device's work belongs to which pod is still the lesson 07 §8 problem, and it is still the capstone's job.

### 8 — Consumable capacity and `shareID`: the first per-share identity

`DRAConsumableCapacity` is **enabled by default in Kubernetes 1.36** (PR #136611). It is the mechanism by which DRA models device sharing natively, rather than by having a driver fabricate N fake devices.

The device side: a device declares `allowMultipleAllocations: true` and publishes a divisible `capacity` with an optional `requestPolicy`:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
metadata: { name: gpu-node-01-shares }
spec:
  driver: gpu.nvidia.com
  nodeName: gpu-node-01
  pool: { name: gpu-node-01, generation: 4, resourceSliceCount: 1 }
  devices:
  - name: gpu-0
    allowMultipleAllocations: true       # ← several claims may allocate this
    attributes:
      uuid: { string: "GPU-8f2c1d90-…" }
    capacity:
      memory:
        value: "80Gi"
        requestPolicy:
          default: "10Gi"
          validRange: { min: "1Gi", step: "1Gi" }
```

The claim side asks for a slice of that capacity:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata: { namespace: team-a, name: gpu-20gi }
spec:
  spec:
    devices:
      requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.nvidia.com
          capacity:
            requests:
              memory: 20Gi              # ← 20 GiB of the device's 80 GiB
```

And the allocation result now carries two new fields (both from the upstream `DeviceRequestAllocationResult`):

```yaml
status:
  allocation:
    devices:
      results:
      - request: gpu
        driver: gpu.nvidia.com
        pool: gpu-node-01
        device: gpu-0
        shareID: "a671734a-e8e5-11e4-8fde-42010af09327"   # ← per-SHARE identity
        consumedCapacity:
          memory: 20Gi                                    # ← rounded up per
                                                          #   requestPolicy
```

`shareID` "uniquely identifies an individual allocation share of the device, used when the device supports multiple simultaneous allocations. It serves as an additional map key to differentiate concurrent shares of the same device." `consumedCapacity` "tracks the amount of capacity consumed per device as part of the claim request" and may exceed the request because it is rounded up to the nearest valid value under the device's `requestPolicy`. The scheduler enforces that total consumed capacity across all shares does not exceed the device's declared capacity.

Here is why this belongs in this module's through-line:

```
  THE ATTRIBUTION LADDER, WITH CONSUMABLE CAPACITY ADDED

  device plugin, time-slicing/MPS
    identity above the kubelet : GPU-uuid::N   (kubelet-internal only)
    identity in the API        : NONE
    identity in DCGM          : one per DEVICE
    → allocation is uniform; the split is unmeasurable → estimate

  DRA, driver-fabricated shares (same as above, expressed better)
    identity in the API        : one claim per share, but every share's
                                 allocation names the SAME device
    identity in DCGM          : one per DEVICE
    → the map improves; the metric does not

  DRA + consumable capacity
    identity in the API        : (driver, pool, device, shareID)
                                 + consumedCapacity PER SHARE
    identity in DCGM          : one per DEVICE   ← STILL
    → you now know each tenant's *entitlement* exactly, from the API.
      That is not a measurement of use, but it IS an authoritative
      denominator: entitlement-weighted splitting is defensible in a
      way replica-count splitting never was, because the entitlements
      are heterogeneous and chosen by the tenant.

  MIG
    identity in the API        : distinct device per instance
    identity in DCGM          : PER INSTANCE
    → exact. The only mode where the metric key matches the billing key.
```

That third rung is genuinely new ground: **an authoritative, heterogeneous, per-tenant entitlement in the API.** Under time-slicing every tenant's entitlement is `1/N` by construction, so entitlement-weighted billing degenerates into fair share. Under consumable capacity a tenant that asked for 20 GiB of an 80 GiB card has a 25% entitlement and a tenant that asked for 5 GiB has a 6.25% entitlement, both recorded in `consumedCapacity`, and weighting by that is a defensible split with an argument behind it. Your capstone should read it.

Two constraints. The NVIDIA driver gates this behind its own `ConsumableShares` feature gate plus the `consumableShares` Helm value (`"disabled"`, `"memory"`, `"unlimited"`, or a positive integer), and the gate's own documentation notes that **MPS sharing is not supported when consumable shares is enabled**. So today you pick consumable capacity *or* MPS, not both. Treat this whole section as the newest and fastest-moving material in the lesson and re-verify it.

### 9 — ComputeDomain and IMEX

The case where DRA is not a nicer API for the same job but the only API that can express the job.

On multi-node NVLink systems (GB200 NVL72-class and successors), GPUs across *several physical nodes* participate in one coherent memory fabric. A workload's requirement is not "8 GPUs" but "8 GPUs that can address each other's memory over NVLink". The device-plugin model has **no vocabulary for a resource that spans nodes** — its entire contract is a per-node integer.

The DRA driver models it with a `compute-domain.nvidia.com` DeviceClass and a `ComputeDomain` object. The driver's own description: ComputeDomains are "an abstraction for robust and secure Multi-Node NVLink", they "guarantee MNNVL-reachability between pods within the domain and isolation from external pods", and they are "ephemeral and follow workload lifetimes". Declaring one causes the driver to co-schedule the participating pods and wire up **IMEX** (Internode Memory Exchange) channels so those GPUs can address each other's memory across the fabric. The chart controls this with `resources.computeDomains.enabled` (default `true`), an IMEX mode (`driverManaged` by default, or `hostManaged` behind the driver's `HostManagedIMEXDaemon` gate when the cluster admin owns the host `nvidia-imex` daemon lifecycle), and an isolation setting (`domain` by default).

There is a related mechanism worth knowing about even on single nodes: the device plugin also grew IMEX support, injecting IMEX channels into workloads globally via `imex.channelIDs` and `imex.required`. At present the only valid `channelIDs` values are `[]` and `[0]`, and the channel device nodes must be visible to the plugin container for discovery to work. That is the device-plugin-shaped approximation; ComputeDomain is the DRA-shaped answer.

You will not exercise this on the single GPU you are renting for this module. You should be able to describe *why* it exists and *what it solves* cold, because it is the strongest concrete answer to "when does DRA's extra complexity actually pay for itself?" — and the answer is not "it is more elegant", it is "there is no other way to say it".

### 10 — Quotas: fencing tenants on both APIs

Scheduling is not isolation. Without a quota, one namespace drains every GPU. You need both mechanisms because the two APIs count different things.

**A. `ResourceQuota` for device-plugin GPUs.** The extended resource is an integer, so quota is an integer:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { namespace: team-a, name: gpu-quota }
spec:
  hard:
    requests.nvidia.com/gpu: "4"
    limits.nvidia.com/gpu: "4"
```

Under the MIG **mixed** strategy the resource names are per-profile, so you can grant slices but not whole cards — which is the cleanest expression of "this tenant may not have a whole GPU" available anywhere in Kubernetes:

```yaml
  hard:
    requests.nvidia.com/mig-1g.10gb: "8"
    requests.nvidia.com/mig-3g.40gb: "2"
    requests.nvidia.com/gpu:         "0"     # explicitly: no whole GPUs
```

And with `renameByDefault: true` on a time-sliced or MPS pool, `nvidia.com/gpu.shared` is a *separate quota key* from `nvidia.com/gpu` — so you can allow a tenant unlimited shared access and zero exclusive access. That is a real reason to prefer `renameByDefault: true` on shared pools, and it is rarely mentioned.

**B. `ResourceQuota` for DRA claims — this is the corrected section.** Earlier guidance (including the previous version of this lesson) said core `ResourceQuota` at GA counts only *objects* for DRA and that per-device limits required a `ValidatingAdmissionPolicy` workaround. **That is out of date.** Kubernetes' claim quota evaluator computes two kinds of usage:

```go
// pkg/quota/v1/evaluator/core/resource_claims.go

var ClaimObjectCountName = generic.ObjectCountQuotaResourceNameFor(
    resourceapi.SchemeGroupVersion.WithResource("resourceclaims").GroupResource())
// → "count/resourceclaims.resource.k8s.io"

// V1ResourceByDeviceClass returns a quota resource name by device class.
// gpuclass -> gpuclass.deviceclass.resource.k8s.io/devices
func V1ResourceByDeviceClass(className string) corev1.ResourceName {
    return corev1.ResourceName(className + corev1.ResourceClaimsPerClass)
}
```

and the `Usage` function charges, per ResourceClaim:

```go
// charge for claim
result[ClaimObjectCountName] = *(resource.NewQuantity(1, resource.DecimalSI))
for _, request := range claim.Spec.Devices.Requests {
    switch {
    case len(request.FirstAvailable) > 0:
        // If there are subrequests, we want to use the worst case per device class
        // to quota. So for each device class, we need to find the max number of
        // devices that might be allocated.
        …
    case request.Exactly != nil:
        deviceClassClaim := V1ResourceByDeviceClass(request.Exactly.DeviceClassName)
        var numDevices int64
        switch request.Exactly.AllocationMode {
        case resourceapi.DeviceAllocationModeExactCount:
            numDevices = request.Exactly.Count
        case resourceapi.DeviceAllocationModeAll:
            // Worst case...
            numDevices = resourceapi.AllocationResultsMaxSize   // = 32
        …
```

So the working quota for a DRA tenant is:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { namespace: team-a, name: dra-gpu-quota }
spec:
  hard:
    # Cap DEVICES requested through the gpu.nvidia.com class:
    gpu.nvidia.com.deviceclass.resource.k8s.io/devices: "4"
    # …and MIG devices separately, because it is a different class:
    mig.nvidia.com.deviceclass.resource.k8s.io/devices: "8"
    # Belt and braces: cap the number of claim OBJECTS too, so a tenant
    # cannot exhaust etcd or the controller with tiny claims.
    count/resourceclaims.resource.k8s.io: "16"
```

Four semantics you must understand or you will misconfigure this:

1. **It is charged at claim admission, from the `spec`, not from the allocation.** A claim that has not been allocated yet still consumes quota. That is deliberate — quota must be decidable at admission — and it means quota is a cap on *requests in flight*, not on devices in use.
2. **`allocationMode: All` costs 32.** `AllocationResultsMaxSize` is 32, and `All` is charged at that worst case regardless of how many devices the node actually has. A single `All` request will blow through any sane per-class quota. Treat `All` as a privileged pattern.
3. **`firstAvailable` (prioritised list) is charged at the worst case per class** — the maximum count across the alternatives, per class, summed. So a prioritised list of "1 H100 or 2 A100s" charges 2 against the A100's class.
4. **Unknown allocation modes count as zero.** The evaluator has an explicit `default:` branch noting that "unknown modes don't count towards the quota and users shouldn't expect that when downgrading". This is a real (if narrow) downgrade-window gap: a claim created with a newer mode against an older API server may be uncharged.

**C. Extended-resource-backed quota (1.35+).** With `DRAExtendedResource` enabled, `ResourceQuota` counts device class requests inside a ResourceClaim as **two additional** quotas (PR #134210):

- `requests.deviceclass.resource.k8s.io/<deviceclass>` — charged at the worst-case device count;
- and for classes that map to an extended resource, `requests.<extended resource name>` — so a DRA-backed GPU also consumes `requests.nvidia.com/gpu`.

That second one is the important one operationally: **a single `requests.nvidia.com/gpu` quota can fence both device-plugin pods and DRA claims** on a cluster running both, which is exactly what you want during a migration. Kubernetes 1.36 also fixed "a loophole that allowed users to work around DRA extended resource quota set by system administrators" (PR #135434) — a reminder that this accounting is young and worth testing rather than assuming.

**D. `LimitRange` for defaults and per-container caps.** So that a forgetful pod is not silently CPU-only, and no single container grabs more than intended:

```yaml
apiVersion: v1
kind: LimitRange
metadata: { namespace: team-a, name: gpu-defaults }
spec:
  limits:
  - type: Container
    max:     { nvidia.com/gpu: "1" }   # no container may request >1
    default: { nvidia.com/gpu: "0" }   # default to none unless asked
```

Note that `LimitRange` operates on container resources, so it fences the device-plugin and extended-resource paths, **not** raw claim requests. For per-claim device caps beyond what quota gives you (e.g. "no claim in this namespace may request more than 2 devices regardless of class"), a `ValidatingAdmissionPolicy` over `resource.k8s.io/v1 ResourceClaim` is still the right tool — not as a workaround for missing quota, but as the complement to it.

Putting the accounting together:

```
  QUOTA ACCOUNTING — WHAT IS CHARGED, WHEN, FROM WHAT

  DEVICE-PLUGIN PATH
    Pod created with resources.limits[nvidia.com/gpu]=2
      │
      ▼  admission (quota plugin), reads the POD spec
    requests.nvidia.com/gpu  += 2     ────▶ Forbidden if over hard cap
    (per-MIG-profile names and nvidia.com/gpu.shared are SEPARATE keys)

  DRA PATH
    ResourceClaim created (by you, or by the template controller)
      │
      ▼  admission, reads the CLAIM SPEC — not the allocation
    count/resourceclaims.resource.k8s.io                  += 1
    <class>.deviceclass.resource.k8s.io/devices            += worst case
       ExactCount   → count
       All          → 32  (AllocationResultsMaxSize)
       firstAvailable → max per class across alternatives
      │
      ▼  additionally, if DRAExtendedResource is enabled
    requests.deviceclass.resource.k8s.io/<class>           += worst case
    requests.<extendedResourceName>                        += worst case
       (e.g. requests.nvidia.com/gpu — the SHARED key that fences
        device-plugin pods and DRA claims with ONE number)

  ENFORCEMENT POINT: admission, in BOTH cases. The rejected object
  never reaches the scheduler, so an over-quota pod does not appear
  as Pending — the API server refuses it outright.

  ⚠ Quota caps REQUESTS IN FLIGHT, not devices in use. A namespace
    full of unallocatable claims is at quota while using zero GPUs.
    Alert on (quota used) − (claims with status.allocation).
```

That last warning is a real operational failure mode: a tenant whose claims cannot be satisfied (bad CEL selector, no matching hardware, device taints) will sit at 100% of quota consuming nothing, and every subsequent legitimate claim in that namespace is rejected. The metric to build is the gap between charged quota and allocated claims.

## Perspectives

**API-design / scheduling-theory view.** The device plugin models "how many". DRA models "which, with what attributes, constrained how, shared to what extent". That is not an increment; it is a change in what the scheduler is allowed to reason about. Mechanically the scheduler barely changed — the `DynamicResources` plugin hangs off the same Filter/Score/Reserve/PreBind cycle from module 02 — but instead of decrementing an integer in `NodeResourcesFit` it evaluates CEL over structured attributes and writes a concrete allocation into an object's status. Note the cost of that expressiveness, visible in the API: a 10-second default `FilterTimeout`, a 32-device cap on allocation results, an explicit early-abort when a prioritised list would explode the search, and no preemption support. Structured selection over heterogeneous inventory is a combinatorial problem, and the design is full of guardrails against that.

**Multi-tenancy / quota view.** DRA's governance layer is now genuinely usable and was not when DRA went GA — per-DeviceClass device quota, extended-resource-backed quota that spans both APIs, device taints, admin access as a first-class privileged mode. But the accounting has a distinctive shape you must internalise: **it charges the worst case from the spec at admission time**, which makes it a request-rate limiter rather than a utilisation cap. Combined with `allocationMode: All` costing 32, that means a well-meaning tenant can lock themselves out of their own quota with one claim. The device-plugin model, whatever else was wrong with it, never had that failure mode.

**Fleet / hardware-topology view.** ComputeDomain and IMEX are where the argument for DRA stops being aesthetic. A per-node integer resource has no way to say "these devices, on different machines, are one fabric". On GB200 NVL72-class hardware that property *is* the product. If you are asked to justify DRA's complexity in one sentence, this is the sentence — and notice it is a hardware-driven argument, not a software-taste argument, which is why it is the one that lands.

**Attribution view — what DRA does and does not buy the capstone.** DRA gives you a better *map*: declarative, event-driven, cluster-visible, with per-share identity when consumable capacity is on, and — via `consumedCapacity` — an authoritative heterogeneous *entitlement* per tenant that no previous mechanism provided. It gives you nothing on the *metric*: DCGM still counts per physical device, so a shared GPU still fans one value out across holders and `sum` still over-reports by the number of holders. **The honest summary is that DRA halves the problem.** Anyone who tells you DRA fixes GPU cost attribution has not looked at where the numbers come from.

**Migration-reality view.** DRA core is GA and its gate is locked on; the extension features you actually want are mostly beta in 1.36; the NVIDIA driver's per-claim sharing is behind alpha, default-off gates; the chart, repo and registry all renamed within the last year; the minimum host driver version has moved at least once; and 1.36 introduced an ACTION-REQUIRED RBAC change for drivers writing claim status. None of that means DRA is not real — it means the correct posture in mid-2026 is a deliberate pilot on a defined node pool, with every version number in this lesson re-derived from primary sources at the time you run it. Say that in an interview and you sound like someone who has operated it. Say "it's GA, it's fine" and you do not.

## Real-world use cases

- **`kubernetes/kubernetes` `CHANGELOG-1.34.md`, `-1.35.md`, `-1.36.md`.** **Fetched and read this session.** The primary record for everything in §4: DRA core GA (PR #132706), the gate locking in 1.35 (#134452), the kubelet gRPC v1 graduation (#132700), the pod-resources DRA gates going beta (#132940), the double-allocation race (#136566) and multi-claim kubelet bug (#136480) **both landing in v1.34.4**, the 1.35 extended-resource quota accounting (#134210), and the 1.36 promotions plus the `resourceclaims/binding` and `resourceclaims/driver` RBAC action-required (#134947). **Correction:** the previous version of this lesson placed the two correctness fixes in 1.34.2 and attributed the multi-claim bug to issue #135901; the changelog shows the fixes in 1.34.4, and 1.34.2's DRA fix is a different one (#133934, an idle-connection deadlock). Read the changelog, not a summary of it.
- **`kubernetes/kubernetes` `pkg/quota/v1/evaluator/core/resource_claims.go`.** **Fetched and read this session.** The quota semantics in §10 are read straight out of this file: `ClaimObjectCountName`, `V1ResourceByDeviceClass` with its `gpuclass -> gpuclass.deviceclass.resource.k8s.io/devices` comment, the worst-case charging for `All` (32) and `firstAvailable`, and the explicit "unknown modes don't count towards the quota" downgrade note. This is the source that corrects the "DRA quota only counts objects" claim.
- **`kubernetes/api` `resource/v1/types.go`.** **Fetched and read this session.** `AllocationResultsMaxSize` = 32; the `reservedFor` 256-entry limit; the exact `DeviceRequestAllocationResult` field set including the documented `<driver name>/<pool name>/<device name>` identity triple, `shareID` ("uniquely identifies an individual allocation share of the device") and `consumedCapacity` ("rounded up to the nearest valid value based on the device's requestPolicy"). **This is the source for §6's correction that the allocation names a device, not a UUID.**
- **`kubernetes-sigs/dra-driver-nvidia-gpu`.** **Fetched and read this session.** `README.md` (ComputeDomains as "an abstraction for robust and secure Multi-Node NVLink", MNNVL-reachability guarantees, ephemeral lifetime); `deployments/helm/dra-driver-nvidia-gpu/Chart.yaml` (`0.5.0-dev` on main, `kubeVersion: ">=1.32.0-0"`); `values.yaml` (all values in §5); `templates/deviceclass-gpu.yaml` and `deviceclass-mig.yaml` (the exact CEL selectors and `extendedResourceName: nvidia.com/gpu`); `cmd/gpu-kubelet-plugin/deviceinfo.go` (the `gpu-<minor>` `CanonicalName()` and the full published attribute set); `cmd/gpu-kubelet-plugin/cdi.go` (the `k8s.gpu.nvidia.com/claim=<claimUID>-<device>` naming); `api/nvidia.com/resource/v1beta1/sharing.go` and `gpuconfig.go` (`GpuConfig`, `GpuSharing`, `MigDeviceSharing`, `MpsConfig`, `TimeSliceInterval` and its 0–3 mapping); `pkg/featuregates/featuregates.go` (the full driver gate table, with `TimeSlicingSettings` and `MPSSupport` registered **Alpha, `Default: false`**, and the `ConsumableShares` note that MPS is unsupported alongside it); and `demo/specs/quickstart/v1/gpu-test{1,2,4,5}.yaml` (the claim examples reproduced in §2 and §7). Latest release tag confirmed as **v0.4.1**.
- **`NVIDIA/gpu-operator` `deployments/gpu-operator/crds/nvidia.com_clusterpolicies.yaml`.** **Fetched and read this session at v26.3.3.** Contains `spec.devicePlugin.config.{name,default}` and **no DRA field whatsoever** — the primary-source refutation of "the GPU Operator folds the DRA driver in-tree". Latest Operator tag confirmed as v26.3.3.
- **`NVIDIA/k8s-dra-driver-gpu` #762 — "GPU plugin: `unknown GPU sharing strategy: TimeSlicing`."** The bug you file when you write the §7 config without enabling the driver's alpha `TimeSlicingSettings` gate. Worth knowing because the error message does not mention feature gates. *(Located and corroborated via search this session; GitHub HTML is not fetchable through this environment's egress proxy.)*
- **Google Cloud, "Kubernetes device management with DRA (Dynamic Resource Allocation)."** A production platform's justification of the claim/attribute model as the replacement for "an integer plus node labels", and DRA's role in GKE's AI-workload story. Read for the *why*, not the API mechanics. *(Not independently fetched this session — domain blocked by this environment's proxy; corroborated via search.)*
- **NVIDIA Developer Blog, "Enabling Multi-Node NVLink on Kubernetes for NVIDIA GB200 NVL72 and Beyond."** The primary public description of ComputeDomain/IMEX at production multi-node-NVLink scale — the concrete "why DRA's complexity earns its keep" case. *(Not independently fetched — proxy-blocked; corroborated via search and against the driver README's own ComputeDomain description.)*

## Worked example

Cluster on **v1.34.6** — deliberately past the 1.34.4 fix line, and deliberately not 1.34.0–1.34.3. Goal: one claim-scheduled pod, the UUID recovered three ways, and two enforced quotas (one of which is deliberately misconfigured so you see the failure).

### Step 1 — verify the version before anything else

```bash
$ kubectl version | grep Server
Server Version: v1.34.6
# ✅ >= 1.34.4, so #136566 (device double-allocation) and #136480
#    (kubelet multi-claim) are both fixed. 1.34.0-.3 would be unsafe.

$ kubectl get --raw /apis/resource.k8s.io | jq -r '.versions[].groupVersion'
resource.k8s.io/v1
resource.k8s.io/v1beta2
resource.k8s.io/v1beta1
# v1 is served → use it. The chart's resourceApiVersion auto-detect
# picks the highest of v1 > v1beta2 > v1beta1.
```

### Step 2 — install and confirm the inventory exists

```bash
$ helm install dra-driver-nvidia-gpu \
    oci://registry.k8s.io/dra-driver-nvidia/charts/dra-driver-nvidia-gpu \
    --version v0.4.1 \
    --create-namespace -n dra-driver-nvidia-gpu \
    --set resources.gpus.enabled=true \
    --set nvidiaDriverRoot=/run/nvidia/driver

$ kubectl -n dra-driver-nvidia-gpu get pods
NAME                                             READY   STATUS    AGE
dra-driver-nvidia-gpu-controller-6d9f7c8b4-2xk9p   1/1     Running   40s
dra-driver-nvidia-gpu-kubelet-plugin-hn4rt         2/2     Running   40s

$ kubectl get deviceclasses
NAME                        AGE
compute-domain.nvidia.com   41s
gpu.nvidia.com              41s
mig.nvidia.com              41s

$ kubectl get resourceslices
NAME                                 NODE          DRIVER           POOL          AGE
gpu-node-01-gpu.nvidia.com-7bk4x     gpu-node-01   gpu.nvidia.com   gpu-node-01   38s

$ kubectl get resourceslices -o yaml | \
    yq '.items[].spec.devices[] | {n: .name, uuid: .attributes.uuid.string}'
{"n": "gpu-0", "uuid": "GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763"}
```

*(Representative transcript — pod names and the UUID will differ. The commands and the object shapes are real.)*

**The `resourceslices` output is the gate.** Empty means the node plugin cannot see the hardware; nothing downstream can work. Check `nvidiaDriverRoot` and CDI first.

### Step 3 — schedule through a claim, then recover the UUID three ways

```bash
$ kubectl apply -f gpu-test1.yaml      # the §2 template + pod1
$ kubectl -n gpu-test1 wait --for=condition=Ready pod/pod1 --timeout=120s
pod/pod1 condition met

# (a) from the claim — note what it does and does not give you
$ kubectl -n gpu-test1 get resourceclaim -o \
    jsonpath='{.items[0].status.allocation.devices.results[0]}' | jq
{
  "request": "gpu",
  "driver": "gpu.nvidia.com",
  "pool": "gpu-node-01",
  "device": "gpu-0"            # ← the triple. NOT a UUID.
}

# (b) the ResourceSlice join — this is the step people miss
$ kubectl get resourceslices -o json | jq -r '
    .items[]
    | select(.spec.driver=="gpu.nvidia.com" and .spec.pool.name=="gpu-node-01")
    | .spec.devices[] | select(.name=="gpu-0")
    | .attributes.uuid.string'
GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763

# (c) ground truth from inside the container
$ kubectl -n gpu-test1 logs pod1 | grep UUID
GPU 0: NVIDIA H100 80GB HBM3 (UUID: GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763)

# (d) and the pod-resources shortcut, from the node
$ grpcurl -plaintext -unix /var/lib/kubelet/pod-resources/kubelet.sock \
    v1.PodResourcesLister/List | jq -c '.podResources[]
      | select(.namespace=="gpu-test1")
      | {pod:.name, dyn:[.containers[].dynamicResources[]?
          | {claim:.claimName, res:[.claimResources[]
              | {drv:.driverName, pool:.poolName, dev:.deviceName,
                 share:.shareId, cdi:[.cdiDevices[].name]}]}]}'
{"pod":"pod1","dyn":[{"claim":"pod1-gpu-x9zq2","res":[{"drv":"gpu.nvidia.com",
 "pool":"gpu-node-01","dev":"gpu-0","share":null,
 "cdi":["k8s.gpu.nvidia.com/claim=4a1f8e02-…-gpu-0"]}]}]}
```

Four sources, one UUID — with the load-bearing observation that **only (b) and (c) actually produce it.** (a) and (d) produce the `(driver, pool, device)` triple, and (d) additionally produces the CDI name and `share_id`. Write that down; it is the design constraint on your capstone's DRA path.

### Step 4 — quota that works, and quota that bites

```bash
$ kubectl create ns team-a
$ kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ResourceQuota
metadata: { namespace: team-a, name: dra-gpu-quota }
spec:
  hard:
    gpu.nvidia.com.deviceclass.resource.k8s.io/devices: "2"
    count/resourceclaims.resource.k8s.io: "8"
EOF

# Two single-device claims: both admitted.
$ kubectl -n team-a apply -f two-single-gpu-pods.yaml
pod/a created
pod/b created

$ kubectl -n team-a describe quota dra-gpu-quota
Name:       dra-gpu-quota
Namespace:  team-a
Resource                                            Used  Hard
--------                                            ----  ----
count/resourceclaims.resource.k8s.io                2     8
gpu.nvidia.com.deviceclass.resource.k8s.io/devices  2     2

# A third: rejected at ADMISSION, before scheduling.
$ kubectl -n team-a apply -f third-gpu-pod.yaml
Error from server (Forbidden): error when creating "third-gpu-pod.yaml":
  resourceclaims "c-gpu" is forbidden: exceeded quota: dra-gpu-quota,
  requested: gpu.nvidia.com.deviceclass.resource.k8s.io/devices=1,
  used: gpu.nvidia.com.deviceclass.resource.k8s.io/devices=2,
  limited: gpu.nvidia.com.deviceclass.resource.k8s.io/devices=2
```

*(Representative error text — the exact wording is produced by the quota admission plugin and may differ in punctuation. The mechanism and the resource-name form are real.)*

Now the failure mode worth engineering for. Replace the claim with `allocationMode: All`:

```bash
$ kubectl -n team-a apply -f - <<'EOF'
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata: { namespace: team-a, name: greedy }
spec:
  devices:
    requests:
    - name: all-gpus
      exactly:
        deviceClassName: gpu.nvidia.com
        allocationMode: All
EOF
Error from server (Forbidden): ... requested:
  gpu.nvidia.com.deviceclass.resource.k8s.io/devices=32, ...
#                                                    ^^
# AllocationResultsMaxSize = 32. `All` is charged at the WORST CASE,
# regardless of how many GPUs the cluster actually has. One `All`
# request blows any sane per-class quota.
```

And the silent version of the same class of problem — quota consumed by claims that will never be satisfied:

```bash
# A claim whose CEL selector matches nothing on this fleet:
#   expression: device.attributes['gpu.nvidia.com'].architecture == 'Blackwell'
$ kubectl -n team-a get resourceclaims
NAME              STATE     AGE
hopeful-gpu       pending   4m       # ← never allocated
$ kubectl -n team-a describe quota dra-gpu-quota | grep devices
gpu.nvidia.com.deviceclass.resource.k8s.io/devices  2     2
# At quota. Using ZERO GPUs. Every legitimate claim in team-a is now
# rejected. This is why you alert on:
#   charged quota  −  count of claims with status.allocation
```

### Step 5 — the reconciliation identity for a DRA cluster

The lesson 07 identities still hold, and DRA adds a third that is cheap and catches a whole class of drift:

```
  IDENTITY 1 — allocation conservation (exact, per physical GPU)
    Σ over claims allocating this device of their share
      + unallocated share  =  1.00
    Without consumable capacity, "share" is 1/N over the N claims on
    that device. With consumable capacity, share = consumedCapacity
    ÷ device capacity, and the SCHEDULER enforces Σ ≤ capacity —
    so this identity is guaranteed by the API, not just asserted.

  IDENTITY 2 — utilisation conservation (within scrape jitter)
    sum by (UUID) (attributed util) ≈ max by (UUID) (device util)
    Unchanged from lesson 07. DRA does not touch this.

  IDENTITY 3 — inventory conservation (the DRA-specific one)
    For every node:
      count of devices in its ResourceSlices
        == count of physical GPUs visible to DCGM on that node
    A mismatch means: the node plugin is publishing a stale pool
    (check pool.generation and resourceSliceCount), a device is
    tainted out, or the plugin restarted mid-publish. Alert on it —
    a silently shrunk inventory looks like a capacity shortage, and
    a silently stale one looks like a scheduling bug.
```

Before running any of this against a real cluster: re-run `kubectl version`, re-read the DRA driver's release notes for the actual minimum NVIDIA driver version for the tag you are pinning (this has moved from ≥565 to ≥580 at different points — do not memorise it), re-check the chart name and latest tag (v0.4.1 at the time of writing, main is 0.5.0-dev), and if you are on 1.36+, check the driver chart's RBAC for `resourceclaims/binding` and `resourceclaims/driver`.

## Practice

**Feeds the deliverable** ([../practice/per-pod-attribution/README.md](../practice/per-pod-attribution/README.md)). On a cluster running **K8s 1.33.x or ≥ 1.34.4** — verify with `kubectl version` and record it.

1. **Record the version landscape you are actually on.** `kubectl version`, `kubectl get --raw /apis/resource.k8s.io` for served versions, and `kubectl get --raw /metrics | grep kubernetes_feature_enabled | grep -i dra` (or your distribution's equivalent) for which DRA gates are enabled. Write down which of the §4 table's rows apply to *your* cluster. Do not copy this lesson's table; derive yours.

2. **Install the driver and prove the inventory.** Install the standalone chart (or via the GPU Operator *if and only if* you have confirmed that specific Operator version supports it in its own release notes — the v26.3.3 ClusterPolicy CRD has no DRA field). Confirm `kubectl get deviceclasses` shows `gpu.nvidia.com` and `kubectl get resourceslices` is non-empty. Then dump one ResourceSlice in full and annotate it: which fields are the pool identity, which are attributes you could select on, which are capacity.

3. **Schedule through a `ResourceClaimTemplate` and recover the UUID four ways.** Capture (a) `status.allocation.devices.results[0]` showing the `(driver, pool, device)` triple; (b) the ResourceSlice join producing the UUID; (c) `nvidia-smi -L` from inside the pod; (d) the pod-resources `dynamic_resources` output showing the triple, the CDI device name, and `share_id`. **Write one sentence stating explicitly that (a) does not contain a UUID and (b) is required.** This is the finding your capstone's DRA path is built on.

4. **Select by attribute, and make a selector fail on purpose.** Write a claim with a CEL selector on a real attribute (`architecture`, `productName`, `cudaComputeCapability`, or a `memory` capacity request) and show it scheduling. Then write one that cannot match, and capture the pod Pending with the claim having no `status.allocation`, plus whatever the scheduler events say. Knowing what an unsatisfiable claim looks like is worth more than knowing what a satisfiable one looks like.

5. **Per-claim sharing, including its gate.** Apply a `GpuConfig` opaque config with `sharing.strategy: TimeSlicing` **without** enabling the driver's `TimeSlicingSettings` gate, and capture the `unknown GPU sharing strategy` failure. Then enable the gate (`--set featureGates.TimeSlicingSettings=true`), re-apply, and show it working. Record the driver version. This exercise is the difference between "DRA supports per-claim sharing" and knowing what that sentence actually costs.

6. **Quota, three ways, including the two failures.** (a) Apply `<class>.deviceclass.resource.k8s.io/devices` and capture the verbatim `Forbidden … exceeded quota` error on the over-limit claim. (b) Submit an `allocationMode: All` claim and capture it being charged 32. (c) Create a claim with an unsatisfiable selector and show it consuming quota while allocating nothing — then write the PromQL or `kubectl` one-liner you would alert on. Optionally (d): if `DRAExtendedResource` is enabled, show a single `requests.nvidia.com/gpu` quota fencing both a device-plugin pod and a DRA claim.

7. **Describe ComputeDomain/IMEX in writing.** You will not run it. Write 150 words on what problem it solves that a per-node integer resource cannot express, what the driver does when you declare one, and why this is the strongest justification for DRA's complexity. This is an interview answer, so write it as one.

**Acceptance:** a committed note under `practice/per-pod-attribution/` containing (a) the Kubernetes version, served `resource.k8s.io` versions, and enabled DRA gates on your cluster, with why that version is safe; (b) a full annotated ResourceSlice; (c) the claim YAML plus the four-way UUID recovery with an explicit statement that the allocation names a device and not a UUID; (d) the failing-selector transcript; (e) the sharing-config gate failure and its fix, with the driver version; (f) the verbatim quota-rejection error, the `All`-costs-32 rejection, and your alert expression for quota-consumed-but-unallocated. That `(driver, pool, device) → uuid → pod` chain is the seed the capstone joins to DCGM.

## Common pitfalls

1. **Believing the allocation names a device UUID.** It names `<driver>/<pool>/<device>`, and for NVIDIA's driver `device` is `gpu-<minor>`. Building a UUID-keyed join on `status.allocation.devices.results[].device` will produce empty joins against DCGM on the first real cluster. The UUID is a ResourceSlice device *attribute*; the join is mandatory.
2. **Running Kubernetes 1.34.0–1.34.3 for DRA workloads.** Both the device double-allocation race (#136566) and the kubelet multi-claim bug (#136480) are present until **1.34.4** — not 1.34.2, which is the number that circulates. These are silent correctness bugs, which is the worst kind to meet in production. Use 1.33.x or ≥1.34.4.
3. **Assuming a GPU Operator version folds the DRA driver in-tree.** There is no public Operator 25.8 — that is a DRA-driver CalVer tag — and the Operator's ClusterPolicy CRD at v26.3.3 has no DRA field at all. It is a companion chart. Check the exact Operator version's release notes rather than a blog's summary.
4. **Writing a `GpuConfig` sharing block without enabling the driver's alpha gates.** `TimeSlicingSettings` and `MPSSupport` are `Default: false, PreRelease: Alpha`. The API server accepts the config and the driver rejects it with `unknown GPU sharing strategy: …`, which does not mention feature gates anywhere.
5. **Assuming `ResourceQuota` cannot cap DRA devices.** It can, via `<class>.deviceclass.resource.k8s.io/devices`, and with `DRAExtendedResource` also via `requests.<extendedResourceName>` — which is how you fence device-plugin pods and DRA claims with one number during a migration. The "only object counts, use a ValidatingAdmissionPolicy" guidance is out of date; a VAP is now a complement, not a workaround.
6. **Forgetting that DRA quota charges the worst case from the spec at admission.** `allocationMode: All` costs 32 (`AllocationResultsMaxSize`) regardless of real hardware; `firstAvailable` costs the max per class across alternatives; and an unallocatable claim consumes quota forever while using nothing. Alert on charged-quota minus allocated-claims.
7. **Using a bare `ResourceClaim` where you wanted a template.** A bare claim is shared and outlives pods; a `ResourceClaimTemplate` mints one per pod with an owner reference so it is garbage-collected. The symptom of getting this wrong is a claim stuck `allocated,reserved` holding a device for a pod that no longer exists.
8. **Upgrading a control plane to 1.36 without checking the driver's RBAC.** `DRAResourceClaimGranularStatusAuthorization` (beta in 1.36) requires `update`/`patch` on `resourceclaims/binding` for schedulers and controllers and `associated-node:update`/`arbitrary-node:update` on `resourceclaims/driver` for drivers, scoped by `resourceNames`. A driver that worked on 1.35 can stop writing claim status.
9. **Expecting preemption, or setting `spec.nodeName` directly.** DRA resources do not support preemption in the current implementation, so priority will not free a device. And a pre-scheduled pod bypasses the scheduler, so nothing writes an allocation and the kubelet retries indefinitely.
10. **Treating DRA as "done" because the core is GA.** GA means the core API is stable. The extensions you want are beta and moving, the driver renamed its repo/chart/registry inside a year, the host-driver minimum has moved, `DRAPartitionableDevices` had a backwards-incompatible change requiring ResourceSlice removal across a 1.34↔1.35 hop, and 1.36 shipped a fix for a quota-bypass loophole. Re-verify every version number here, including from this lesson.
11. **Thinking DRA fixes the attribution hole.** It fixes the map, not the metric. DCGM still counts per physical device; a shared GPU allocated through three claims still reports one number, and `dcgm-exporter --kubernetes-enable-dra` still fans it out. Lesson 07's arithmetic is unchanged.

## Self-check

- **ResourceClaim vs a device-plugin `nvidia.com/gpu` request — what is structurally better for cost attribution, and what precisely does the claim give you?** *Answer:* The device plugin gives the pod an opaque string chosen from a fabricated list, discoverable only by polling the kubelet's pod-resources socket on that node. A DRA `ResourceClaim` records the decision in an API object in the pod's namespace: `status.allocation.devices.results[]` names the allocation as the triple `<driver>/<pool>/<device>`, `status.reservedFor[]` names the consuming pod with its UID, and the whole thing is watchable cluster-wide and garbage-collected with the pod when a template minted it. You also get to *request* by attribute (CEL over `architecture`, `productName`, `memory` capacity, MIG `profile`, `parentUUID` constraints) and to attach per-claim sharing config, so a shared device carries per-workload intent instead of node-wide ConfigMap policy. **What it does not give you is the device UUID** — `device` is a DNS label, `gpu-<minor>` for NVIDIA's driver — so reaching DCGM requires joining `(driver, pool, device)` to `ResourceSlice.spec.devices[].attributes.uuid`.

- **The Kubernetes 1.34 DRA version caveat — precisely, and what do you run?** *Answer:* DRA core went GA in 1.34 (PR #132706) and the `DynamicResourceAllocation` gate was locked to enabled-by-default in 1.35 (PR #134452), so it cannot be disabled from 1.35 on. But two silent correctness bugs persisted through 1.34.3: a goroutine race that could **allocate the same device twice to different ResourceClaims** — with the second pod either failing to start or, worse, running in parallel on someone else's device, depending on whether the driver checks in `NodePrepareResources` — fixed by **PR #136566**; and the **kubelet mishandling a pod with multiple ResourceClaims when one was already prepared**, fixed by **PR #136480**. Both PRs appear in the **v1.34.4** changelog, not 1.34.2 (whose DRA fix is a different one, #133934, an idle-connection deadlock). So the rule is **1.33.x or ≥1.34.4, never 1.34.0–1.34.3** — and derive it from `CHANGELOG-1.34.md` yourself rather than trusting any summary, including this one.

- **What is wrong with "GPU Operator 25.8 folds the DRA driver in-tree", and what is actually true?** *Answer:* Two things. First, the version number is a collision: the public GPU Operator line went 25.3.x → 25.10.x → 26.3.x (v26.3.3 latest at the time of writing) and never had a 25.8; "25.8" is a *DRA-driver* CalVer tag from before that project adopted semver at v0.4.0 and moved from `NVIDIA/k8s-dra-driver-gpu` to `kubernetes-sigs/dra-driver-nvidia-gpu`, renaming the chart from `nvidia-dra-driver-gpu` to `dra-driver-nvidia-gpu`. Second, and more usefully, the substance is checkable independently of the numbering: the Operator's `ClusterPolicy` CRD at v26.3.3 contains no DRA field whatsoever. The driver installs as a companion Helm chart (latest release v0.4.1; `main` is 0.5.0-dev; `kubeVersion >= 1.32.0-0`). NVIDIA describes Operator-managed installation as the preferred *future* method — verify against the specific Operator version's release notes.

- **How does `ResourceQuota` interact with MIG-profile resources and with DRA claims?** *Answer:* For device-plugin GPUs the quota keys are extended-resource names — `requests.nvidia.com/gpu`, the per-profile names under the MIG *mixed* strategy (`requests.nvidia.com/mig-1g.10gb`), and, with `renameByDefault: true`, `nvidia.com/gpu.shared` as a key separate from `nvidia.com/gpu` — so you can grant a namespace slices or shared access but zero whole cards. For DRA, core quota has two keys: `count/resourceclaims.resource.k8s.io` for claim objects, and **`<device-class>.deviceclass.resource.k8s.io/devices`** for devices requested through that class. The device count is charged **from the claim spec at admission, at the worst case**: `ExactCount` charges `count`, `allocationMode: All` charges `AllocationResultsMaxSize` = **32** regardless of real hardware, and `firstAvailable` charges the maximum per class across the alternatives. With `DRAExtendedResource` (beta in 1.36) two more keys are charged, `requests.deviceclass.resource.k8s.io/<class>` and `requests.<extendedResourceName>` — the latter meaning one `requests.nvidia.com/gpu` number can fence both APIs during a migration. Enforcement is at admission in every case, so an over-quota object never becomes Pending; the API server refuses it. The trap: quota caps *requests in flight*, so unallocatable claims hold quota while using zero GPUs.

- **What does ComputeDomain/IMEX solve that no device-plugin mechanism can?** *Answer:* A resource that spans multiple physical nodes. On multi-node NVLink systems (GB200 NVL72-class), GPUs on several machines participate in one coherent memory fabric, and the workload's real requirement is "these GPUs must be MNNVL-reachable from each other", not "eight GPUs". The device plugin's contract is a per-node integer — it has no vocabulary for cross-node adjacency, and its `TopologyInfo` is NUMA-only. The DRA driver models it as a `compute-domain.nvidia.com` DeviceClass plus a `ComputeDomain` object, described by the driver as an abstraction for robust and secure Multi-Node NVLink that guarantees MNNVL-reachability within the domain and isolation from external pods, ephemeral and following the workload lifetime; declaring one co-schedules the participating pods and wires up IMEX (Internode Memory Exchange) channels. It is the strongest justification for DRA's complexity precisely because it is a hardware-driven argument rather than an aesthetic one.

- **What is `shareID`, and why does it matter to this module's through-line?** *Answer:* With `DRAConsumableCapacity` (enabled by default in Kubernetes 1.36), a device can declare `allowMultipleAllocations: true` and publish divisible `capacity` with a `requestPolicy`; a claim requests a slice via `devices.requests[].exactly.capacity.requests`; and the allocation result gains `shareID` — "uniquely identifies an individual allocation share of the device… an additional map key to differentiate concurrent shares of the same device" — plus `consumedCapacity`, the actually-charged amount rounded up per the request policy. It matters because it is the **first mechanism in this module that gives a shared device a per-tenant identity in the API**, and because `consumedCapacity` is an authoritative, *heterogeneous* entitlement: under time-slicing every tenant's entitlement is `1/N` by construction, so entitlement-weighted billing degenerates to fair share, whereas here a tenant that asked for 20 GiB of an 80 GiB card has a recorded 25% entitlement and one that asked for 5 GiB has 6.25%. The scheduler enforces that shares do not exceed capacity, so the allocation-conservation identity is guaranteed by the API rather than asserted. Caveats: NVIDIA gates it behind its own `ConsumableShares` gate and the `consumableShares` Helm value, MPS sharing is unsupported when it is enabled, and DCGM still measures per physical device — so this improves the denominator, not the measurement.

- **Does DRA fix GPU cost attribution?** *Answer:* It fixes half of it. It fixes the **map**: ownership becomes a declarative, event-driven, cluster-visible object with the consuming pod in `reservedFor`, per-share identity via `shareID`, and per-tenant entitlement via `consumedCapacity`. It does nothing to the **metric**: DCGM programs performance counters per physical device, so a time-sliced, MPS-shared or consumable-capacity-shared GPU still reports exactly one `DCGM_FI_PROF_GR_ENGINE_ACTIVE`, and `dcgm-exporter` with `--kubernetes-enable-dra` / `KUBERNETES_ENABLE_DRA=true` still fans that single value out across every holder — so `sum` still over-reports by the number of holders, exactly as in lesson 07. The only mode where the measurement key equals the billing key remains MIG. Anyone who says DRA solves attribution has not looked at where the numbers come from.

## Connections & what's next

This closes the loop the module opened at 06/07/08. MIG's clean UUIDs, time-slicing's fan-out and MPS's throughput-versus-blast-radius trade were all consequences of the device plugin's integer contract; DRA replaces that contract with an object model in which the allocation is a first-class, watchable, quotable API object. Module 02's DRA object-model lesson is the theory this operationalises, and module 02's scheduler-internals lesson is the Filter/Score/Reserve/PreBind cycle the `DynamicResources` plugin runs inside — now evaluating CEL over structured attributes instead of decrementing an integer.

Three things from this lesson land directly in the capstone. The **`(driver, pool, device) → ResourceSlice attribute → uuid` join** is the DRA-sourced ownership path, and it is a join, not a lookup. The **`DynamicResource`/`ClaimResource` messages on the pod-resources API** (`claim_name`, `claim_namespace`, `driver_name`, `pool_name`, `device_name`, `share_id`, `cdi_devices`) are a second, node-local source for the same fact, available because `KubeletPodResourcesDynamicResources` went beta and on-by-default with the DRA GA. And **`consumedCapacity`** is the authoritative entitlement your cost model should weight by when it is present, in preference to a replica-count fair share.

What does *not* carry forward is any hope that DRA repealed lesson 07. It did not. The capstone still has to detect the sharing regime, still has to dedupe the DCGM fan-out, and still has to label an estimate as an estimate.

Next: **[04.10 · Capstone — per-pod GPU attribution](10-capstone-per-pod-attribution.md)** assembles L1 through here into the module's deliverable — the pod-resources join from 04.3, MIG's exact 1:1 case from 04.6, the fan-out and its arithmetic from 04.7, MPS's bounded-ceiling estimate from 04.8, and this lesson's claim-and-slice path as the second, structurally cleaner source of the same ownership map.

## References & further reading

**Primary sources**

- [Kubernetes documentation — Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/) — **the source markdown was fetched and read this session from `kubernetes/website`.** The authoritative feature-gate list, the served API versions (`v1`, `v1beta1`, `v1beta2`, and `v1alpha3` holding only `DeviceTaintRule`), the canonical YAML for DeviceClass / ResourceClaim / ResourceClaimTemplate / ResourceSlice including the prioritised-list, partitionable-devices, consumable-capacity and binding-conditions variants, the `resource.kubernetes.io/admin-access` namespace-label requirement, the 600 s default binding timeout and `DynamicResourcesArgs.bindingTimeout`, the 256-entry `reservedFor` limit, and the statements that preemption is unsupported and pre-scheduled pods bypass allocation.
- [kubernetes/kubernetes — `CHANGELOG-1.34.md`, `CHANGELOG-1.35.md`, `CHANGELOG-1.36.md`](https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG) — **fetched and read this session.** Every version claim in §4, with PR numbers. **This is the source that corrects the "fixes landed in 1.34.2" claim** — #136566 and #136480 are both in the v1.34.4 section.
- [kubernetes/kubernetes — `pkg/quota/v1/evaluator/core/resource_claims.go`](https://github.com/kubernetes/kubernetes/blob/master/pkg/quota/v1/evaluator/core/resource_claims.go) — **fetched and read this session.** The quota resource names and the worst-case `Usage()` accounting quoted in §10. **Corrects the "DRA quota counts only objects" claim.**
- [kubernetes/api — `resource/v1/types.go`](https://github.com/kubernetes/api/blob/master/resource/v1/types.go) — **fetched and read this session.** `AllocationResultsMaxSize` = 32, the `<driver>/<pool>/<device>` identity-triple documentation, the "must be a DNS label" constraint on `device`, and the full `shareID` / `consumedCapacity` field documentation.
- [kubelet pod-resources API proto (`v1.PodResourcesLister`)](https://github.com/kubernetes/kubelet/tree/master/pkg/apis/podresources) — **fetched and read this session.** The `DynamicResource{claim_name, claim_namespace, claim_resources[]}` and `ClaimResource{cdi_devices[], driver_name, pool_name, device_name, share_id}` messages that make the pod-resources shortcut in §6 possible.
- [kubernetes-sigs/dra-driver-nvidia-gpu](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu) — **fetched and read this session; latest release tag confirmed v0.4.1, `main` chart at 0.5.0-dev.** `README.md`, `Chart.yaml`, `values.yaml`, the two DeviceClass templates with their exact CEL selectors and `extendedResourceName`, `cmd/gpu-kubelet-plugin/deviceinfo.go` (the `gpu-<minor>` naming and the published attribute set), `cmd/gpu-kubelet-plugin/cdi.go` (CDI qualified-name construction), `api/nvidia.com/resource/v1beta1/{sharing.go,gpuconfig.go}` (the full sharing API including `MigDeviceSharing`), `pkg/featuregates/featuregates.go` (the driver gate table with `TimeSlicingSettings` and `MPSSupport` **Alpha, default false**), and `demo/specs/quickstart/v1/gpu-test{1,2,4,5}.yaml`.
- [NVIDIA/gpu-operator — `nvidia.com_clusterpolicies.yaml`](https://github.com/NVIDIA/gpu-operator) — **fetched and read this session at v26.3.3.** Contains `spec.devicePlugin.config.{name,default}` and no DRA field; the primary-source refutation of the in-tree-folding claim.
- [NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) — **fetched and read this session at v0.19.2.** The `imex.channelIDs` / `imex.required` table and the `[]`/`[0]` valid-values constraint cited in §9, plus the node-label catalogue used in the quota discussion.

**Bug reports and vendor accounts**

- [NVIDIA/k8s-dra-driver-gpu #762 — "GPU plugin: `unknown GPU sharing strategy: TimeSlicing`"](https://github.com/NVIDIA/k8s-dra-driver-gpu/issues/762) — the symptom of the driver's alpha `TimeSlicingSettings` gate being off. *(Corroborated via search this session; GitHub HTML is blocked by this environment's egress proxy.)*
- Google Cloud, ["Kubernetes device management with DRA (Dynamic Resource Allocation)"](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-device-management-with-dra-dynamic-resource-allocation) — the production-platform case for the claim/attribute model. *(Not independently fetched — proxy-blocked; corroborated via search.)*
- NVIDIA Developer Blog, ["Enabling Multi-Node NVLink on Kubernetes for NVIDIA GB200 NVL72 and Beyond"](https://developer.nvidia.com/blog/enabling-multi-node-nvlink-on-kubernetes-for-gb200-and-beyond/) — ComputeDomain/IMEX at production scale. *(Not independently fetched — proxy-blocked; corroborated via search and cross-checked against the driver README's own ComputeDomain description.)*
- AKS Engineering Blog, ["Delve into Dynamic Resource Allocation, devices, and drivers on Kubernetes"](https://blog.aks.azure.com/2025/11/17/dra-devices-and-drivers-on-kubernetes) — a second managed-Kubernetes vendor's rollout and version guidance, useful for triangulating. *(Not independently fetched — proxy-blocked.)*

**Deeper dives**

- [kubernetes/enhancements — KEP-4381, DRA: structured parameters](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/4381-dra-structured-parameters/README.md) — the design rationale behind the ResourceSlice/attribute model and the quota naming scheme. Read this if you want to understand *why* the allocation names a pool-scoped device rather than a vendor identifier.
- [Lesson 07 — Time-slicing and the attribution trap](07-time-slicing-attribution.md) — the DCGM fan-out and the `sum`-over-shared-series arithmetic that DRA does **not** repeal.
- [Lesson 08 — MPS and choosing a sharing mode](08-mps-choosing-sharing.md) — the ceilings that §7's `mpsConfig` sets per claim, and the bounded-estimate argument that per-claim configuration finally makes practical.
