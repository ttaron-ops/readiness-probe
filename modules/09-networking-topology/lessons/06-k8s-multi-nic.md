---
lesson: "09.6"
title: "Kubernetes multi-NIC for RDMA (Multus/SR-IOV/Network Operator)"
module: "09"
concept: "Kubernetes multi-NIC for RDMA"
status: not-started
est_time: "7h"
prev: "05-gpudirect-and-sharp.md"
next: "07-bandwidth-as-cost.md"
artifacts: []
sources: 10
---

# 09.6 · Kubernetes multi-NIC for RDMA (Multus/SR-IOV/Network Operator)

> **Concept.** RDMA reaches a pod through a *second* NIC that the default CNI never touches — Multus attaches it, a device plugin makes the VF schedulable, and Topology Manager must NUMA-align it with the GPU.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Where this fits

**05** established what a pod's container *wants*: a GPU and a rail-paired NIC, both visible to NCCL, `NCCL_IB_HCA` pinned correctly, GDR engaging on the paired path. That whole lesson quietly assumed the NIC was simply *there* in the pod's network namespace, ready to be named by an `mlx5_*` device string. On Kubernetes it is not. The default CNI hands every pod exactly one interface — `eth0`, on the cluster's pod network — built for ordinary service-to-service TCP/IP, with no notion of a second, RDMA-capable device tied to physical fabric hardware. This lesson is the plumbing gap between "05's ideal pod" and "a pod the default Kubernetes networking model can actually produce." It closes that gap with three purpose-built components — Multus, the SR-IOV stack, and the NVIDIA Network Operator — and shows exactly where the GPU-NIC NUMA alignment from 05 can silently fail once a scheduler, not a human, is choosing which GPU and which NIC a pod gets.

## Why this matters

This is the K8s-platform differentiator. Anyone can run `nccl-tests` on bare metal; the job you're targeting is *operating* the fabric on Kubernetes across 40+ clusters where pods, not hosts, are the unit. A pod scheduled onto a node with a perfectly railed ConnectX-7 will still stage every byte through the CPU if the RDMA device never got wired into the pod's network namespace, or if the scheduler handed it a GPU on NUMA0 and a NIC VF on NUMA1. Those failures are invisible to `kubectl get pod` — the pod is Running, throughput is just quietly halved. Being able to read a `NicClusterPolicy`, a `NetworkAttachmentDefinition`, and a two-resource pod spec and say *exactly* where the alignment can break is the differentiator, and it's the same skill that lets you attribute a "slow training run" to a scheduling bug instead of the model. It's also directly graded: the module checkpoint asks you to name what Multus, the SR-IOV device plugin, and the Network Operator each contribute, and why the default CNI can't do RDMA.

## What's new here (calibration)

- In **06 of the platform module** you learned Topology Manager aligns CPU + GPU on one NUMA node via device-plugin `TopologyInfo` hints. We do not re-teach Topology Manager's admission logic itself — we teach the *new resource* (an RDMA NIC VF) that has to feed into it alongside the GPU.
- In **02b/09.5** you learned GDR needs GPU and NIC on the same PCIe root complex / rail. We do not re-derive that rule — we teach the Kubernetes-native proxy for it (NUMA node) and the components that make a VF NUMA-visible to the scheduler in the first place.
- New here: the specific division of labor across **four** components that are easy to conflate — Multus (interface plumbing), the **SR-IOV Network *Operator*** (creates the VFs and generates NADs, via `SriovNetworkNodePolicy` — a distinct project from the SR-IOV device plugin), the SR-IOV **device plugin** (advertises already-existing VFs as a schedulable resource), and the Network Operator (bundles all of the above plus OFED and lifecycle-manages them).
- New here: the emerging alternative — Kubernetes Dynamic Resource Allocation (DRA), which some hyperscalers are now piloting specifically to replace this lesson's static SR-IOV-VF + Topology-Manager pairing with dynamic GPU-NIC co-scheduling. Flagged in Connections & what's next, not taught as the core mechanism — the SR-IOV/Multus/Network-Operator stack this lesson teaches is still what's deployed at most shops in 2026.

## Core concepts

### Why the default CNI isn't enough — what Multus adds

The default CNI (Calico, Cilium, …) is a *single-interface* model: one `eth0` per pod on the cluster pod network, for regular TCP/IP service traffic. RDMA needs a second interface bound to the physical InfiniBand/RoCE NIC (or a VF of it), with the RDMA verbs device visible in the pod. The default CNI has no concept of "also give me `net1` on the IB fabric."

**Multus** is a *meta-CNI* (a CNI-of-CNIs). It runs as the CNI, calls the default CNI first to create `eth0` (the pod keeps normal networking), then reads the pod's `k8s.v1.cni.cncf.io/networks` annotation and, for each entry, delegates to another CNI plugin to attach an **additional** interface (`net1`, `net2`, …). Each attachment is described by a **`NetworkAttachmentDefinition`** (NAD) CRD, which carries the CNI config (e.g. the `sriov` CNI, `ipoib`, or `macvlan`) plus IPAM. Multus's own contribution is *orchestration of multiple interfaces*; the actual RDMA-capable interface is created by the delegate CNI the NAD names.

### Creating and making the VF schedulable — two separate components

Attaching an interface isn't enough; the VF has to *exist*, and the scheduler must *account* for it so two pods don't claim the same Virtual Function. These are **two distinct jobs, done by two distinct upstream projects** that are easy to conflate:

- **SR-IOV** partitions one physical NIC (the Physical Function, PF) into many **Virtual Functions (VFs)**, each a lightweight PCIe function with its own RDMA context. VF *creation* is a PCIe/firmware-level reconfiguration of the PF — it is not elastic per-pod the way a Deployment's replica count is. Someone (or something) has to decide up front how many VFs a PF is carved into.
- The **SR-IOV Network *Operator*** (`k8snetworkplumbingwg/sriov-network-operator` — a *separate* project from the device plugin below) is what actually does that carving. A cluster admin writes a **`SriovNetworkNodePolicy`** custom resource naming a PF selector and a VF count; the operator reconfigures the PF's firmware to create that many VFs on matching nodes, and can generate the corresponding `NetworkAttachmentDefinition` automatically from a companion `SriovNetwork` resource.
- The **SR-IOV network device plugin** (`k8snetworkplumbingwg/sriov-network-device-plugin`) does a different job: it *discovers* VFs that already exist and advertises them to the kubelet as a countable **extended resource** — exactly like the GPU operator advertises `nvidia.com/gpu`. The resource name is configurable — e.g. `nvidia.com/sriov_rdma`, `rdma/rdma_vf` — and a pod requests it in `resources.limits`. The scheduler then treats VFs as a countable, exhaustible resource per node.

Put plainly: **the SR-IOV Network Operator creates the VFs (and can generate the NAD); the SR-IOV device plugin only discovers already-existing VFs and advertises them to the kubelet.** They are complementary, not the same component, and a cluster that's missing the first will never have VFs for the second to advertise.

Crucially, the device plugin also reports each VF's **NUMA node** via the device-plugin API `TopologyInfo`. That is what lets Topology Manager do its job (below).

### RDMA shared device plugin — the no-SR-IOV path

Not every deployment wants SR-IOV (VF count is fixed at PF config time, and some clouds don't expose it). The **RDMA shared device plugin** (`k8s-rdma-shared-dev-plugin`) advertises a *shared* RDMA device — e.g. `rdma/rdma_shared_device_a` — that **multiple pods use concurrently** off one PF, typically paired with a `macvlan` or `ipoib` secondary interface. You trade the hard isolation of a dedicated VF for simpler provisioning; for many inference and single-tenant training nodes that's the right call. SR-IOV gives isolation and per-VF scheduling; shared gives density and no VF plumbing.

### The bundle — NVIDIA Network Operator

Wiring OFED drivers, Multus, the SR-IOV network operator, the SR-IOV device plugin, the RDMA shared device plugin, IPoIB CNI, and IPAM by hand across 40+ clusters is how you get drift. The **NVIDIA Network Operator** deploys and lifecycle-manages the whole stack from one cluster-scoped CRD, the **`NicClusterPolicy`**. It bundles:

- **OFED / DOCA-OFED driver** container (`ofedDriver`) — the `mlx5` kernel modules and user-space verbs, matched to the running kernel.
- **Multus** (`secondaryNetwork.multus`) — the meta-CNI.
- **SR-IOV device plugin** (`sriovDevicePlugin`) — VFs as resources. (SR-IOV VF *creation* and NAD generation is driven by the companion SR-IOV Network Operator / `SriovNetworkNodePolicy`, above.)
- **RDMA shared device plugin** (`rdmaSharedDevicePlugin`) — shared RDMA resources.
- **IPoIB CNI** and **IPAM** (`nvIpam` / whereabouts) — addressing for the secondary net.
- **NV Peer Memory / nvidia-peermem** hookup so GDR (05) actually works.

It is the *networking* counterpart to the GPU Operator, and the two are designed to run side by side: GPU Operator owns `nvidia.com/gpu`, Network Operator owns the RDMA resources. NVIDIA's own reference deployment guide walks through exactly this pairing on the EGX stack.

### Where it interlocks with Topology Manager

Now the payoff. A GDR-optimal pod requests **both** `nvidia.com/gpu: 1` **and** an RDMA resource (`nvidia.com/sriov_rdma: 1` or `rdma/…: 1`). Both device plugins report NUMA affinity via `TopologyInfo`. With the kubelet `topologyManagerPolicy: single-numa-node`, Topology Manager will only admit the pod if it can allocate the GPU **and** the NIC VF **and** the CPUs on **one** NUMA node — the coarse Kubernetes proxy for 05's "same rail / same root complex." If the only free GPU is on NUMA1 and the only free VF is on NUMA0, admission fails with a **`TopologyAffinityError`** rather than silently placing a cross-NUMA pod. That's the behavior you *want*: a hard failure you can see beats a Running pod at half bandwidth. Under the looser `best-effort` policy the pod is admitted anyway and the misalignment becomes an invisible performance tax — the exact cost/observability trap, and precisely why choosing `best-effort` "to avoid scheduling failures" is a trap, not a fix (see Pitfalls).

**Where alignment breaks in practice:** (1) VF pools not pinned per NUMA, so the plugin advertises VFs the scheduler can't NUMA-match to the GPU; (2) `best-effort` policy masking misalignment; (3) the NAD pointing at a PF on the wrong socket for the GPUs the node schedules; (4) requesting the RDMA resource but pinning `NCCL_IB_HCA` (05) to a different rail's device, so K8s aligns the VF but NCCL still uses the wrong NIC.

## Perspectives

**Developer.** From the pod-author's seat, this entire mechanism is invisible on the happy path. You write two resource requests and one annotation — `nvidia.com/gpu`, `nvidia.com/sriov_rdma`, and `k8s.v1.cni.cncf.io/networks: rdma-net` — and the pod either comes up with a fast `net1` or it doesn't. The Multus/SR-IOV/Topology-Manager machinery underneath stays invisible right up until admission fails (a `TopologyAffinityError` you now know how to read) or, worse, it silently succeeds with half the bandwidth you expected. Your job is to know what those two failure signatures mean, not to operate the stack that produces them.

**Operator.** Running this exact stack — OFED, Multus, the SR-IOV operator and device plugin, IPAM, NUMA policy — by hand, correctly, across 40+ clusters is precisely why the Network Operator exists. Hand-wiring per-cluster is where version drift compounds: one cluster on an older OFED build, another with a stale `SriovNetworkNodePolicy` VF count, a third with `best-effort` left on from a debugging session and never reverted. The Network Operator turns that into one CRD (`NicClusterPolicy`) reconciled per cluster, which is the only way this scales past a handful of clusters without an on-call nightmare.

**Hardware / kernel.** SR-IOV VF creation happens at the PCIe/firmware layer, driven by `SriovNetworkNodePolicy` reconfiguring the PF. This is a hard constraint worth internalizing: **VF count is not dynamically elastic per-pod** the way a Deployment's replica count is. If a node's PF was provisioned for 8 VFs and you need a 9th, that's a firmware reconfiguration (often a node reboot or NIC re-init), not a live scale-up. Capacity planning for RDMA-heavy fleets has to account for VF count as a fixed, provisioned quantity, not an elastic one.

**Economics / failure-mode.** DRANET (below) signals where the industry is heading: away from static SR-IOV-VF-plus-Topology-Manager pairing and toward Kubernetes Dynamic Resource Allocation (DRA) for GPU+NIC co-scheduling. The economic argument is the same one that motivates the Network Operator in the first place — static per-node VF provisioning and admission-time NUMA matching is real operational surface area, and every generation of hardware with a different NUMA layout (like the 4-GPU/4-NIC Azure ND GB300-v6 node) re-exposes it. A dynamic, driver-based co-allocation model is a bet that this surface area shrinks; it isn't yet the dominant production pattern, but it's the direction the largest operators are placing bets on.

## Real-world use cases

- **Microsoft Azure AKS Engineering Blog — "Simplifying InfiniBand on AKS" (April 2025).** Configuring the NVIDIA Network Operator alongside the GPU Operator on AKS for production HPC/AI workloads — exactly this lesson's stack, from a named hyperscaler's managed Kubernetes offering. Shows the Network-Operator-plus-GPU-Operator pairing running at hyperscaler scale, not just in a vendor reference doc. https://blog.aks.azure.com/2025/04/11/infiniband-on-aks
- **Microsoft Community Hub — "Running tightly coupled HPC/AI workloads with InfiniBand using NVIDIA Network Operator on AKS" (Oct 2024).** A deeper walkthrough of `NicClusterPolicy` configuration, node labelling, and namespace setup for tightly-coupled multi-node training on AKS. Shows the operational detail level (labels, node pools, RDMA validation) a real deployment needs beyond the CRD reference alone. https://techcommunity.microsoft.com/blog/azurehighperformancecomputingblog/running-tightly-coupled-hpcai-workloads-with-infiniband-using-nvidia-network-ope/4117209
- **NVIDIA Developer Blog — "Deploying GPUDirect RDMA on the EGX Stack with the NVIDIA Network Operator."** NVIDIA's own reference deployment guide for the Multus + SR-IOV + Network-Operator stack, showing the Network Operator installed alongside the GPU Operator to enable GDR end to end. Shows the vendor-canonical version of the pairing this lesson teaches. https://developer.nvidia.com/blog/deploying-gpudirect-rdma-on-egx-stack-with-the-network-operator/
- **Azure/aks-rdma-infiniband (GitHub).** Runnable guidance, samples, and validation tools for RDMA/InfiniBand/GPUDirect RDMA on AKS clusters. Useful as a hands-on companion to the Practice section below — real manifests and validation scripts, not just prose. https://github.com/Azure/aks-rdma-infiniband

## Worked example

Four manifests, annotated. **`NicClusterPolicy`** (cluster-scoped, one per cluster):

```yaml
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata: { name: nic-cluster-policy }
spec:
  ofedDriver:                          # (1) mlx5 kernel modules + verbs, per-kernel
    image: doca-driver
    repository: nvcr.io/nvidia/mellanox
    version: "24.10-0.7.0.0"
  sriovDevicePlugin:                   # (2) advertise already-existing VFs as a schedulable resource
    image: sriov-network-device-plugin
    config: |
      { "resourceList": [ {
          "resourceName": "sriov_rdma",            # -> nvidia.com/sriov_rdma
          "resourcePrefix": "nvidia.com",
          "selectors": { "vendors": ["15b3"],      # Mellanox/NVIDIA PCI vendor
                         "isRdma": true } } ] }
  secondaryNetwork:
    multus: { image: multus-cni }      # (3) meta-CNI: enables net1 alongside eth0
    ipamPlugin: { image: whereabouts } # (4) IP address management for the RDMA net
```

**`SriovNetworkNodePolicy`** (from the SEPARATE SR-IOV Network Operator — this is what actually creates the VFs (2) above merely advertises):

```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata: { name: rail-vfs, namespace: sriov-network-operator }
spec:
  nodeSelector: { feature.node.kubernetes.io/network-sriov.capable: "true" }
  nicSelector: { vendor: "15b3", deviceID: "1021" }   # (5) which PF to carve
  numVfs: 8                                            # (6) VF count is fixed at reconfig time, not elastic
  resourceName: sriov_rdma                             # (7) must match (2)'s resourceName
```

**`NetworkAttachmentDefinition`** (namespaced; what Multus attaches as `net1` — can be hand-written or generated by an `SriovNetwork` CR from the SR-IOV Network Operator):

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: rdma-net
  annotations:
    k8s.v1.cni.cncf.io/resourceName: nvidia.com/sriov_rdma   # (8) tie NAD to the VF resource
spec:
  config: |
    { "cniVersion": "0.3.1",
      "type": "sriov",                                        # (9) delegate CNI = sriov
      "ipam": { "type": "whereabouts",
                "range": "192.168.100.0/24" } }               # (10) address on the fabric subnet
```

**Pod spec** (requests GPU + VF, attaches the secondary net):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nccl-worker
  annotations:
    k8s.v1.cni.cncf.io/networks: rdma-net                     # (11) Multus attaches net1 from this NAD
spec:
  containers:
    - name: trainer
      image: nvcr.io/nvidia/pytorch:24.10-py3
      resources:
        limits:
          nvidia.com/gpu: 1                                   # (12) GPU device plugin
          nvidia.com/sriov_rdma: 1                            # (13) SR-IOV VF — the same NAD resource
      securityContext:
        capabilities: { add: ["IPC_LOCK"] }                   # (14) required to pin RDMA memory regions
```

Read it as a chain: (6) the SR-IOV Network Operator carves 8 VFs off the PF → (2) the SR-IOV device plugin discovers those VFs and advertises `nvidia.com/sriov_rdma` → (11) tells Multus which NAD → (8) that NAD is backed by resource (13) → both (12) GPU and (13) VF report NUMA via `TopologyInfo` → with `single-numa-node` the kubelet co-allocates them on one NUMA node or rejects with `TopologyAffinityError`.

**Where NUMA/GPU/NIC alignment can go wrong here:** if the node's free GPU (12) is NUMA1 but the only free `sriov_rdma` VF (13) is NUMA0, `single-numa-node` → admission failure (good, visible); `best-effort` → Running pod, cross-NUMA GDR path, ~half bandwidth (bad, silent). And even when K8s aligns them, the container must still set `NCCL_IB_HCA` (05) to the `mlx5` device that *is* GPU (12)'s paired rail, or NCCL uses the wrong NIC despite correct scheduling. Missing `IPC_LOCK` (14) → `ibv_reg_mr` fails and RDMA silently degrades.

## Practice

Feeds the deliverable's **annotated multi-NIC manifest set**.

1. Take the four manifests above (or a real set from your cluster, the Network Operator repo examples, or the `Azure/aks-rdma-infiniband` repo).
2. Add a line-by-line annotation: for **every** non-trivial line, one clause on what it does and which component consumes it (SR-IOV Network Operator, Multus, SR-IOV device plugin, IPAM, kubelet/Topology Manager, NCCL).
3. Mark **three** distinct places NUMA/GPU/NIC alignment can break, and for each state the symptom (`TopologyAffinityError` vs silent half-bandwidth vs `ibv_reg_mr` failure) and the fix.
4. Name the exact CRD (and which repo it comes from) that creates a VF, and the exact CRD that merely advertises one — this is the distinction most people blur, and the acceptance check specifically probes it.

**Acceptance:** an annotated `NicClusterPolicy` + `SriovNetworkNodePolicy` + NAD + pod-spec set with the resource chain traced end to end and the three failure modes called out with symptom and fix. Done when a reader who has never seen these CRDs can point to the exact line that creates the VF, the exact line that puts the RDMA interface in the pod, the exact line that makes the VF schedulable, and the exact policy that forces NUMA alignment.

## Common pitfalls

- **Treating the Network Operator as set-and-forget.** OFED driver versions must track the kernel version shipped in each node image; a node-image bump without a matching OFED bump is a real, recurring outage cause. The operator reduces drift across clusters, but someone still has to keep `ofedDriver.version` current as node images move.
- **Assuming VFs and NADs just "exist" without naming what creates them.** The SR-IOV device plugin only *discovers and advertises* VFs that already exist — it does not create them. VF creation and (optionally) NAD generation is the job of the **SR-IOV Network Operator**, driven by a `SriovNetworkNodePolicy` CR, a separate upstream project from the device plugin. Conflating the two means you can't diagnose "why are there zero VFs to schedule" — the answer usually lives in the operator/policy, not the plugin.
- **Using `best-effort` Topology Manager policy in production "to avoid scheduling failures."** This converts a loud, visible admission failure (`TopologyAffinityError`) into a silent, half-bandwidth Running pod — strictly worse for on-call, because nothing alerts on it and the first signal is usually a confused researcher asking why their job is slow. `single-numa-node` failing fast is the correct trade for GDR-sensitive workloads.
- **Forgetting VF count is fixed at PF-reconfiguration time, not elastic per pod.** A node provisioned for 8 VFs cannot silently grow to 9 to satisfy a scheduling burst; reconfiguring `numVfs` is a firmware-level PF change, often disruptive. Capacity planning has to treat VF count as a provisioned quantity, like GPU count, not something the scheduler can conjure.
- **Not distinguishing DRANET/DRA (2026, emerging) from SR-IOV + Topology Manager (this lesson's core teaching, still dominant in 2026 production).** DRANET is a real, named direction from a hyperscaler, but it is a pilot on newer hardware generations (Azure ND GB300-v6, AKS 1.34), not yet the default deployment pattern most shops run. Don't present it as having replaced the SR-IOV/Multus/Network-Operator stack — it's the emerging alternative, not the current baseline.

## Self-check

- **Why isn't the default CNI enough for RDMA — what does Multus add?**
  **Answer:** The default CNI gives a pod a single `eth0` on the cluster pod network for ordinary TCP/IP; it has no way to attach a second interface bound to the RDMA NIC/VF. Multus is a meta-CNI: it creates `eth0` via the default CNI, then reads the pod's `k8s.v1.cni.cncf.io/networks` annotation and delegates to another CNI (sriov/ipoib/macvlan, named by a `NetworkAttachmentDefinition`) to attach an additional RDMA-capable interface (`net1`). It orchestrates multiple interfaces; the delegate CNI builds the actual RDMA one.

- **What repo actually creates SR-IOV VFs and generates the NAD, versus the SR-IOV device plugin?**
  **Answer:** The **SR-IOV Network Operator** (`k8snetworkplumbingwg/sriov-network-operator`), driven by a `SriovNetworkNodePolicy` custom resource, reconfigures the PF's firmware to create the requested number of VFs on matching nodes, and can generate the corresponding `NetworkAttachmentDefinition` from a companion `SriovNetwork` resource. The **SR-IOV network device plugin** (`k8snetworkplumbingwg/sriov-network-device-plugin`) is a separate project: it only discovers VFs that already exist and advertises them to the kubelet as a schedulable extended resource. They are complementary — one creates, the other advertises — and both must be present for the whole pipeline to work.

- **What resource does the SR-IOV device plugin advertise, and why must the scheduler NUMA-align it with the GPU?**
  **Answer:** It advertises NIC Virtual Functions as a countable extended resource (e.g. `nvidia.com/sriov_rdma`), just like `nvidia.com/gpu`, and reports each VF's NUMA node via the device-plugin `TopologyInfo`. Alignment matters because GPUDirect RDMA needs the GPU and NIC on the same PCIe root complex / rail (02b, 05); NUMA node is the scheduler's proxy for that. Topology Manager's `single-numa-node` policy co-allocates GPU + VF + CPU on one NUMA node or rejects with `TopologyAffinityError` — preventing a silently cross-NUMA, half-bandwidth GDR path.

- **What newer Kubernetes mechanism is a hyperscaler piloting (2026) to replace static SR-IOV-VF + Topology-Manager NUMA alignment, and what problem does it target?**
  **Answer:** DRANET, built on Kubernetes Dynamic Resource Allocation (DRA) — Azure's AKS engineering team describes it in an April 2026 post targeting ND GB300-v6 nodes running AKS 1.34, where 4 GPUs and 4 ConnectX-8 NICs split across two NUMA domains. DRANET discovers RDMA-capable devices, advertises them as DRA `ResourceSlice`s, and co-schedules them with the NVIDIA GPU DRA driver, injecting the aligned devices into the pod via an NRI plugin — a more dynamic take on the same GPU-NIC NUMA co-alignment problem this lesson's SR-IOV + Topology-Manager stack solves statically.

## Connections & what's next

This lesson is where 05's NCCL-level tuning (`NCCL_IB_HCA`, `NCCL_NET_GDR_LEVEL`) meets the Kubernetes scheduler: everything in 05 assumed a pod already had a rail-aligned NIC to point NCCL at, and this lesson is the mechanism — Multus, SR-IOV (operator and device plugin), the Network Operator, Topology Manager — that actually produces that pod. It also reaches back to the platform module's Topology Manager lesson, extending CPU+GPU NUMA alignment to a third resource (the NIC VF). Next, **07** turns this module's fabric and placement knowledge into the thing a procurement review actually wants: a bandwidth-and-dollar statement. Where this lesson taught you *how* a pod gets a rail-aligned NIC, 07 teaches you *why that placement is worth defending in dollars* — oversubscription as a capex lever, SHARP (05) as a byte-reduction lever, and the co-location argument priced out end to end.

## References & further reading

**Primary sources**
- NVIDIA Network Operator (`Mellanox/network-operator`). The `NicClusterPolicy` CRD and what the operator bundles. https://github.com/Mellanox/network-operator
- Multus CNI. The meta-CNI, NAD spec, and multi-interface orchestration. https://github.com/k8snetworkplumbingwg/multus-cni
- SR-IOV network device plugin. VF discovery and advertisement as a schedulable resource. https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin
- SR-IOV Network Operator. The `SriovNetworkNodePolicy` CRD that actually creates VFs and can generate NADs — distinct from the device plugin above. https://github.com/k8snetworkplumbingwg/sriov-network-operator
- NVIDIA Developer Blog — "Deploying GPUDirect RDMA on the EGX Stack with the NVIDIA Network Operator." NVIDIA's own reference deployment for the full Multus + SR-IOV + Network-Operator stack. https://developer.nvidia.com/blog/deploying-gpudirect-rdma-on-egx-stack-with-the-network-operator/

**Real-world engineering blogs**
- Microsoft Azure AKS Engineering Blog — "Simplifying InfiniBand on AKS" (April 2025). Network Operator + GPU Operator configured together for production HPC/AI on a hyperscaler's managed Kubernetes. https://blog.aks.azure.com/2025/04/11/infiniband-on-aks
- Microsoft Community Hub — "Running tightly coupled HPC/AI workloads with InfiniBand using NVIDIA Network Operator on AKS" (Oct 2024). Deeper operational walkthrough of `NicClusterPolicy`, node labelling, and namespace setup. https://techcommunity.microsoft.com/blog/azurehighperformancecomputingblog/running-tightly-coupled-hpcai-workloads-with-infiniband-using-nvidia-network-ope/4117209
- Azure/aks-rdma-infiniband (GitHub). Runnable guidance, samples, and validation tools — a hands-on companion for the Practice section. https://github.com/Azure/aks-rdma-infiniband
- CoreWeave — NCCL configuration reference. Production `NCCL_IB_HCA` / GDR settings that must match the VF Kubernetes scheduled — the join point between this lesson and 05. https://docs.coreweave.com/products/networking/hpc-interconnect/nccl-configuration-reference

**Deeper dives**
- Microsoft Azure AKS Engineering Blog — "Optimizing RDMA performance for AI workloads on AKS with DRANET" (April 2026). The emerging DRA-based direction for GPU-NIC NUMA co-scheduling, on Azure ND GB300-v6 / AKS 1.34 — read for where this stack is heading, not as this lesson's core mechanism. https://blog.aks.azure.com/2026/04/01/dranet-rdma-optimization-for-ai-on-aks
