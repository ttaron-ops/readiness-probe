---
lesson: "A02.7"
title: "GPU and RDMA networking (platform-integration angle)"
module: "A-02"
concept: "provisioning + operating RDMA on K8s"
status: not-started
est_time: "4 hrs"
prev: "06-service-mesh.md"
next: "08-network-observability-and-debugging.md"
artifacts: ["RDMA node acceptance-test runbook"]
sources: 8
---

# A02.7 · GPU and RDMA networking (platform-integration angle)

> **Concept.** The fabric can be physically perfect and a single misconfigured device plugin, NUMA-misaligned VF, or un-drained flapping NIC still silently halves training throughput — this lesson is the operations layer over the module-09 fabric.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Where this fits

**06** ended on a hard placement rule: mesh the inference frontend, never the RDMA data path, because a sidecar cannot ride RDMA and would be fatal to collective latency. This lesson picks up exactly there — the deliberately-bypassed path that must never touch a mesh, a Service VIP, or the primary CNI's conntrack table. Module 09 already gave you the physics of that path (RoCEv2 lossless mechanics, GPUDirect, rail-optimized Clos) and 09.6 already gave you the Kubernetes wiring that exposes it to a pod (Multus, the SR-IOV Network Operator, `NicClusterPolicy`, Topology Manager). What's left, and what nobody else in this curriculum owns, is the platform team's actual job once that wiring exists: deciding which allocation model to run it through, catching the NUMA misalignment that wiring alone doesn't prevent, and gating a node before it's allowed to poison a live collective. That's this lesson.

## Why this matters

Module 09 gets the photons right: RoCEv2, PFC/ECN/DCQCN, GPUDirect RDMA, rail-optimized Clos, oversubscription math. None of that reaches a training job until Kubernetes hands a pod a NIC that is (a) an actual RDMA device, (b) on the correct NUMA node relative to its GPU, and (c) on the correct rail relative to the other 7 nodes in the job. Every one of those is a platform decision you own, and every one fails *silently* — no error, no crash, just a 4-8× dataplane collapse on a cluster billing ~$40/hr/node. Staff-level here means you can reason about the DRA-vs-device-plugin migration path, own the topology-alignment failure mode as a scheduling-extensibility problem rather than a networking one, and produce an acceptance gate that catches a bad node *before* it joins the pool and poisons a collective. It's also where the ML org's cost model actually lives: a node that passes every health check but silently delivers half its busbw still bills at full GPU-hour rate.

## What's new here (calibration)

- **Module 09 boundary.** We do not re-derive RoCEv2/InfiniBand transport mechanics, PFC/ECN/DCQCN lossless flow control, GPUDirect's DMA path, or Clos/rail-optimized topology math — that's 09's territory and this lesson references it, not repeats it.
- **09.6 boundary.** We do not re-teach the NetworkAttachmentDefinition/SR-IOV-CNI wiring, the `NicClusterPolicy` CRD contents, or the Multus interface-attachment mechanics — 09.6 already walks that manifest chain line by line. This lesson assumes that wiring exists and asks the next question: *which allocation model should own it, and how do you know a node running it is actually healthy?*
- **New here:** the DRA structured-parameters model as an operational migration (not a CRD tour) — DeviceClass/ResourceClaim/ResourceClaimTemplate as the three objects you stage a fleet through; `numVFs` sizing as a hard PCIe/IOMMU ceiling with a production rule of thumb, not "just increase the number"; and the acceptance-test runbook as a recurring day-2 gate (re-run on reboot/firmware bump), not a one-time onboarding checkbox.
- **New framing:** topology misalignment and VF-pool exhaustion as scheduler-extensibility problems as much as networking ones — the default K8s scheduler has no native concept of "GPU N and NIC M share a PCIe/NUMA domain," which is why Topology Manager, DRA, and gang-scheduling frameworks (Volcano, Kueue) all exist to patch that gap from different angles.

## Core concepts

**Skip (you already know, or 09.6 already taught):** RoCEv2 vs InfiniBand and PFC/ECN/DCQCN lossless mechanics; GPUDirect RDMA data path and NCCL ring/tree collectives; rail-optimized topology and oversubscription math; the Multus/`NetworkAttachmentDefinition`/SR-IOV-CNI wiring chain and the `NicClusterPolicy` CRD; the SR-IOV device plugin vs SR-IOV Network Operator split — all covered in module 09 and 09.6.

**Which allocation model, and why it's a platform decision.**

| Model | Isolation | QoS | Fits |
|---|---|---|---|
| RDMA shared device plugin | None — pods share one HCA, one traffic class | No per-pod dedicated bandwidth | Dev clusters, inference, cost-sensitive, single-tenant nodes |
| SR-IOV Network Device Plugin | Dedicated VF per pod, own netdev/PCI address | Per-VF QoS | Production multi-tenant training |
| DRA (SR-IOV driver / DraNet) | Dedicated VF, topology-aware binding | Per-VF QoS + NUMA-atomic GPU+NIC claim | Production training where mis-scheduling risk must be structurally prevented, not just caught |

Shared-HCA and SR-IOV are not interchangeable defaults — they're a tradeoff you make explicitly per node pool. Shared-HCA has no per-pod isolation or dedicated QoS: in a multi-tenant production training cluster that means one noisy pod can starve another's RDMA traffic with zero scheduling signal that it happened. SR-IOV's dedicated VFs are the production choice specifically because the isolation is structural, not policy.

**DRA is displacing the device-plugin model — as three specific objects, not a flag.** The device-plugin API is countable-and-opaque: it hands out "one of N `nvidia.com/gpu`" with no way to express "and a NIC that is NUMA-local to *that* GPU." Kubernetes **Dynamic Resource Allocation (DRA)** replaces counting with three declarative CRDs, and knowing them by name is the interview-and-production-both bar:

- **DeviceClass** — defines the resource *type* (e.g. "RDMA-capable NIC"), analogous to a StorageClass: a cluster-admin-owned template other objects reference.
- **ResourceClaim** — a concrete request for an instance matching constraints, e.g. "an RDMA NIC in the same NUMA domain as GPU claim X." This is where topology-affinity is expressed as a first-class constraint instead of hoped for via two independent `TopologyInfo` hints.
- **ResourceClaimTemplate** — lets a Pod spec request a ResourceClaim declaratively (like a PVC template in a StatefulSet), so the pod author writes intent, not a pre-created claim object.

The **DRA SR-IOV driver / DraNet** allocate VFs as structured resources through these objects and can bind a GPU + rail-aligned NIC in one claim — the atomic co-allocation that Topology Manager's `single-numa-node` policy only approximates by cross-checking two independent `TopologyInfo` hints at admission time.

**Operational gotcha you own: DRA and the device-plugin model cannot coexist for the same resource on the same node.** Enabling the DRA SR-IOV driver disables the classic SR-IOV Network Device Plugin's claim on those VFs — this is not a soft preference, it's a hard resource-ownership conflict. A live-fleet migration is therefore staged **node-pool by node-pool**, never node-by-node within a pool and never as a global flag flip: feature-gate enablement → DRA driver rollout to a canary pool → validate the acceptance gate below still passes on that pool → rewrite workload manifests from `resources.limits` to `resourceClaims` → widen. Treat it exactly like a CNI migration, not a config toggle.

**`numVFs` is a hard PCIe ceiling, not a policy knob — size it like capacity, not like a feature flag.** Many Mellanox ConnectX NICs support up to 127 VFs per PF in principle. That number is not the production target. Each VF is a real PCIe function with per-function overhead, and every VF you carve subdivides the PF's physical bandwidth — a PF carved into 32 VFs gives each VF roughly 1/32 of the wire rate under contention, far below what a training job's collective needs. Two more constraints push the effective ceiling much lower than the hardware max: headroom for shared-HCA fallback if a VF-backed pod fails to schedule, and avoiding IOMMU-group aliasing, where VFs sharing an IOMMU group can't be isolated across pods regardless of what the device plugin advertises. Rule of thumb: size `numVFs` to the actual concurrent-pod-per-node ceiling you intend to run, plus a small fixed headroom — not to the hardware maximum, and never bump it reactively when pods start pending without checking IOMMU grouping first.

**Topology alignment is the whole game operationally, and it is a scheduler-extensibility gap, not a wiring bug.** A GPU scheduled with a NIC on the *wrong NUMA node* still works — traffic just traverses the inter-socket link instead of a NUMA-local PCIe path. NCCL's topology engine reads the PCIe/NUMA graph and allocates channels accordingly: a NUMA-aligned NIC gets **8-16 channels**; a SYS-distant NIC gets **~2**. That is a silent 4-8× reduction in effective collective bandwidth with **zero errors logged anywhere** — not in `kubectl describe pod`, not in dmesg, not in the CNI logs. The reason this keeps recurring across every generation of hardware is structural: the default K8s scheduler has no native concept of "GPU N and NIC M are in the same PCIe/NUMA domain." Everything that fixes it — Topology Manager's `single-numa-node` policy, DRA's atomic ResourceClaim binding, gang-scheduling frameworks like Volcano and Kueue that place an entire job's pods as one topology-aware unit — is a scheduling-extensibility patch over that gap, not a networking fix. `NCCL_TOPO_FILE` is the mechanism that makes the resulting topology explicit to NCCL instead of leaving it to autodetection: it's an XML description of the PCIe/NUMA graph, can be auto-generated with `nvidia-smi topo -m`, and — because it's just a file — can be diffed against a known-good baseline as part of the acceptance runbook below, catching a topology regression before a job ever runs on the node.

**Provisioning & day-2 ops — acceptance and health, not physics.**

- **Fabric validation:** `ib_write_bw` / `ib_read_bw` point-to-point between candidate nodes confirms line-rate RDMA before anything else — necessary, not sufficient (see pitfalls).
- **Collective acceptance gate:** `nccl-tests` `all_reduce_perf` — the number that matters is **bus bandwidth (busbw)**, not algbw; busbw normalizes for GPU count and is the real fabric-plus-topology health signal. A node that can't hit expected busbw fails acceptance even if `ib_write_bw` passed clean.
- **Lossless-fabric health:** PFC pause-frame counters (a storm means a slow receiver is back-pressuring the fabric), ECN mark rates, and the cardinal rule — *one bad link degrades the entire collective* because the slowest participant sets the pace of an all-reduce.
- **Fleet skew:** firmware/MOFED version drift across nodes causes subtle interop and performance faults; version-pin and audit as a recurring check, not a one-time image build.
- **VF-pool exhaustion:** `numVFs` is fixed at policy/reconfiguration time; running out means pods pend with no obvious cause pointing at the fleet-capacity root — this is the day-2 symptom of the sizing decision above going stale as node density grows.
- **Draining:** a flapping RDMA NIC must get the node **cordoned and drained out of the schedulable pool** before it lands in a job — a flapping link inside a running collective stalls every rank, and a currently-running job cannot be partially rescued once it does.

**The acceptance runbook is a fleet-lifecycle control, not an onboarding checkbox.** Firmware drifts, MOFED updates, and reboots all invalidate a prior acceptance pass. The correct operating model re-runs the fabric-validate → nccl-test-gate → topology-check sequence on every reboot and every firmware/driver update, not only when a node first joins the pool — the failure modes here (VF exhaustion, NUMA mis-pin, firmware skew, flapping-link drain) are exactly the ones that accumulate as a fleet ages, invisibly to the ML researcher and paged directly to the platform team that owns node health.

## Perspectives

**Platform-team-ownership.** Every failure mode in this lesson — VF exhaustion, NUMA mis-pin, firmware skew, a flapping link mid-collective — is invisible to the ML researcher who just sees a slow training run, and visible only to the platform engineer who owns node acceptance. The ML team's silent 4-8× tax is the platform team's paged incident; the acceptance gate exists precisely to move that discovery from "researcher notices the loss curve is taking forever" to "node never entered the pool."

**Scheduler-integration.** The default K8s scheduler has no native concept of GPU-NIC PCIe/NUMA co-location. That's a scheduling-extensibility problem as much as a networking one, and it's why the fix set spans Topology Manager (kubelet-local admission policy), DRA structured parameters (a new resource-allocation API), and gang-scheduling frameworks like Volcano and Kueue (job-level, all-pods-at-once placement) — three different layers of the stack, each patching the same underlying gap from a different angle.

**Fleet-lifecycle.** Firmware/MOFED drift, VF-pool exhaustion, and topology drift are day-2 problems that surface as a fleet ages, not day-0 onboarding problems. Teaching the acceptance runbook as a one-time gate misses the point — it has to be a recurring, triggered check (on reboot, on firmware/driver update) or it silently stops being true the moment the fleet's first firmware patch ships.

**Cost.** A silently-degraded node still bills at full GPU-hour rate while delivering a fraction of collective throughput. The quantifiable waste is `$/GPU-hour × (1 − busbw_ratio)` summed across every node in the affected job for its full runtime — a concrete, defensible number you can put in front of a budget owner to justify building the acceptance gate in the first place, and the number a staff engineer should be able to produce on request, not just gesture at.

## Real-world use cases

- **NVIDIA Developer Blog, "Streamlining Kubernetes Networking in Scale-out GPU Clusters with the new NVIDIA Network Operator 1.0."** Official first-party account of the Multus / MOFED-driver-container / RDMA-shared-device-plugin / SR-IOV-device-plugin composition — the K8s-operations layer this lesson sits above. https://developer.nvidia.com/blog/streamlining-kubernetes-networking-in-scale-out-gpu-clusters-with-the-new-nvidia-network-operator-1-0/
- **CoreWeave Docs, "Use GPUDirect RDMA with RoCE."** A real cloud provider's operational documentation for the K8s-facing RDMA device interface on a multi-rail RoCE fabric — what a customer actually sees when this stack works. https://docs.coreweave.com/products/networking/hpc-interconnect/use-gpudirect-rdma-roce
- **Meta Engineering, "RoCE networks for distributed AI training at scale."** Cited here only for the topology-aware job scheduling and NIC-tuning operational detail at 24,000-GPU scale — not for RoCE/PFC/ECN mechanics, which is module 09's territory. https://engineering.fb.com/2024/08/05/data-center-engineering/roce-network-distributed-ai-training-at-scale/

## Worked example

Stand up (or read the manifests for) a 2-node RDMA setup:
1. Install the NVIDIA Network Operator; confirm MOFED modules load (`lsmod | grep mlx5`) and the SR-IOV device plugin advertises the VF resource (`kubectl describe node` shows `nvidia.com/sriov_rdma_vf: 8`).
2. Create the `SriovNetworkNodePolicy` (numVFs, resourceName, PF selector) and the `SriovNetwork`/NetworkAttachmentDefinition wiring RDMA CNI + NVIDIA IPAM — the manifest chain 09.6 walks in detail.
3. Schedule a pod requesting both `nvidia.com/gpu: 1` and the SR-IOV VF resource, annotated onto the NAD via Multus.
4. Run `nccl-tests` `all_reduce_perf -b 8 -e 128M -f 2 -g <gpus>`; record **busbw** at the large-message asymptote — your baseline acceptance number.
5. **Deliberately mis-pin:** force the VF onto a NIC on the wrong NUMA node relative to the GPU, re-run, and observe NCCL (`NCCL_DEBUG=INFO`) report a collapsed channel count (~2 vs 8-16) and busbw fall 4-8× — *with no errors*. The failure mode made visible.
6. Generate `NCCL_TOPO_FILE` with `nvidia-smi topo -m`, diff it against the aligned-run baseline from step 4, and confirm the diff is exactly where the mis-pin lives — this is the automatable check the recurring acceptance runbook runs on every reboot.

**Deliverable:** an RDMA node acceptance-test runbook — **fabric validate** (`ib_write_bw` line rate) → **nccl-test gate** (busbw ≥ threshold) → **topology check** (NCCL channel count / `NCCL_TOPO_FILE` diff against baseline) — that a node must pass before joining the schedulable pool, and must re-pass on every reboot and firmware/driver update.

## Practice

Build the 2-node manifests and the acceptance runbook above; then break the NUMA pinning and capture the before/after busbw + channel count as the evidence the runbook is designed to catch. Carry the artifact into [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md), where the "all-reduce got slower" scenario reuses these signals, and into 08, which teaches the decision-tree discipline for bisecting exactly this kind of silent regression when it shows up in production instead of in acceptance testing.

## Common pitfalls

- **"A passing `ib_write_bw` test means the node is training-ready."** Point-to-point bandwidth says nothing about topology alignment or collective behavior under contention. Only a collective test — `nccl-tests` busbw — exercises the real failure mode (NUMA mis-pin, channel collapse) that point-to-point tests structurally cannot see.
- **"DRA and the device-plugin model can run side by side during migration."** They cannot, for the same resource on the same node — enabling the DRA SR-IOV driver disables the classic device plugin's claim on those VFs. Migration is staged node-pool by node-pool, never as a partial mix within one pool.
- **"More VFs per PF is always better for flexibility."** Each VF carries per-function overhead and subdivides the PF's physical bandwidth; a PF over-provisioned with VFs delivers less bandwidth per VF than a training job needs, and can hit IOMMU-group aliasing that breaks per-pod isolation regardless of what the device plugin advertises.
- **"The RDMA shared device plugin and SR-IOV device plugin are interchangeable."** Shared-HCA has no per-pod isolation or dedicated QoS. In production multi-tenant training this risks noisy-neighbor RDMA contention that SR-IOV's dedicated VFs structurally prevent — it's a tradeoff decision per node pool, not a default.
- **"A NUMA mis-pin will show up as an error somewhere."** It produces zero errors anywhere in the stack — only a busbw regression and a channel-count drop visible via `NCCL_DEBUG=INFO`. This is exactly why the failure mode needs an active acceptance gate that runs a real collective, not passive error-log monitoring, which will never catch it.

## Self-check

- Why can a GPU pod with a working, line-rate RDMA NIC still run its all-reduce 4-8× slower than an identical pod, with no errors anywhere? **Answer:** The NIC is on the wrong NUMA node relative to the GPU. NCCL's topology engine sees a SYS-distant path and allocates only ~2 channels instead of 8-16; traffic crosses the inter-socket link. It's a silent bandwidth collapse — the fix is topology-aware scheduling (Topology Manager `single-numa-node` / DRA claim binding GPU+NIC atomically), not anything on the fabric.
- What breaks if you enable the DRA SR-IOV driver without decommissioning the SR-IOV Network Device Plugin, and why is this a staff-owned migration? **Answer:** They conflict — you cannot run both for the same VF resources on a node; enabling DRA disables the device plugin's claim on those VFs. It's staff-owned because it's a live-fleet migration: staged feature-gate enablement, DRA driver rollout to a canary pool, and rewriting every workload manifest from countable `resources.limits` to declarative `resourceClaims`, not a single flag flip — and it has to be staged node-pool by node-pool, never mixed within one pool.
- On a node acceptance test, why is `nccl-tests` busbw the gate rather than `ib_write_bw` alone or algbw? **Answer:** `ib_write_bw` only proves a point-to-point link is healthy; it says nothing about the collective. busbw is the collective's normalized bus bandwidth — it factors out GPU count and exposes real end-to-end fabric-plus-topology health, so a node passing `ib_write_bw` can still fail the busbw gate (e.g. NUMA misalignment). algbw isn't normalized for participant count, so it isn't comparable across job sizes.
- What are the three DRA CRDs involved in allocating a NUMA-aligned GPU+NIC pair, and what does each one own? **Answer:** DeviceClass defines the resource type (e.g. "RDMA-capable NIC"); ResourceClaim is the concrete request for an instance matching constraints, including NUMA-affinity to a named GPU claim; ResourceClaimTemplate lets a Pod spec request a ResourceClaim declaratively instead of requiring a pre-created claim object. Together they let a pod express "GPU and NIC in the same NUMA domain" as one atomic claim, something the countable device-plugin API cannot express.
- Why isn't "just increase `numVFs`" a free fix for VF-pool exhaustion? **Answer:** Each VF is a real PCIe function with per-function overhead, and every VF carved off a PF subdivides its physical bandwidth — over-provisioning drops effective bandwidth per VF below what a training job needs, and can trigger IOMMU-group aliasing that breaks per-pod isolation. The production ceiling is set by concurrent-pod-per-node capacity plus small headroom, far below the hardware max (up to 127 VFs/PF on many Mellanox ConnectX NICs), not by "more is more flexible."

## Connections & what's next

Back-references 06's placement rule (mesh the frontend, never the RDMA path) by defining exactly what that excluded path is and how you operate it. Leans on module 09 for fabric physics (RoCEv2, PFC/ECN/DCQCN, GPUDirect, Clos topology) and on 09.6 for the Kubernetes wiring (Multus, `NicClusterPolicy`, SR-IOV Network Operator, Topology Manager) — go there for the manifest-level detail this lesson deliberately doesn't repeat. Feeds forward into **08**, which teaches the general bisection discipline for diagnosing a production regression, including the fabric branch of its decision tree that traces straight back to the busbw/channel-count signals built here.

## References & further reading

**Primary sources**
- NVIDIA Networking Kubernetes deployment overview. https://docs.nvidia.com/networking/display/kubernetes2640/overview.html
- Kubernetes DRA (Dynamic Resource Allocation) official docs — DeviceClass/ResourceClaim/ResourceClaimTemplate spec. https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/
- DRA SR-IOV driver docs. https://mellanox.github.io/network-operator-docs/dra-sriov-driver/dra-sriov-driver.html
- `NVIDIA/nccl-tests` — the `all_reduce_perf` busbw benchmark used as the acceptance gate. https://github.com/NVIDIA/nccl-tests

**Real-world engineering blogs**
- NVIDIA Developer Blog, "Streamlining Kubernetes Networking in Scale-out GPU Clusters with the new NVIDIA Network Operator 1.0." https://developer.nvidia.com/blog/streamlining-kubernetes-networking-in-scale-out-gpu-clusters-with-the-new-nvidia-network-operator-1-0/
- CoreWeave Docs, "Use GPUDirect RDMA with RoCE." https://docs.coreweave.com/products/networking/hpc-interconnect/use-gpudirect-rdma-roce
- Meta Engineering, "RoCE networks for distributed AI training at scale" (topology-aware scheduling / NIC tuning at 24,000-GPU scale only). https://engineering.fb.com/2024/08/05/data-center-engineering/roce-network-distributed-ai-training-at-scale/

**Deeper dives**
- NVIDIA Developer Blog, "Running AI Workloads on Rack-Scale Supercomputers: From Hardware to Topology-Aware Scheduling." https://developer.nvidia.com/blog/running-ai-workloads-on-rack-scale-supercomputers-from-hardware-to-topology-aware-scheduling/
