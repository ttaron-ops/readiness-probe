---
lesson: "09.6"
title: "Kubernetes multi-NIC for RDMA"
module: "09"
concept: "Kubernetes multi-NIC for RDMA"
status: not-started
est_time: "5h"
artifacts: []
---

# 09.6 · Kubernetes multi-NIC for RDMA

> **Concept.** RDMA reaches a pod through a *second* NIC that the default CNI never touches — Multus attaches it, a device plugin makes the VF schedulable, and Topology Manager must NUMA-align it with the GPU.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Why this matters

This is the K8s-platform differentiator. Anyone can run `nccl-tests` on bare metal; the
job you're targeting is *operating* the fabric on Kubernetes across 40+ clusters where
pods, not hosts, are the unit. A pod scheduled onto a node with a perfectly railed
ConnectX-7 will still stage every byte through the CPU if the RDMA device never got wired
into the pod's network namespace, or if the scheduler handed it a GPU on NUMA0 and a NIC
VF on NUMA1. Those failures are invisible to `kubectl get pod` — the pod is Running,
throughput is just quietly halved. Being able to read a `NicClusterPolicy`, a
`NetworkAttachmentDefinition`, and a two-resource pod spec and say *exactly* where the
alignment can break is the differentiator, and it's the same skill that lets you attribute
a "slow training run" to a scheduling bug instead of the model.

## What's new here

In **06** you learned Topology Manager aligns CPU + GPU on one NUMA node via the device
plugin `TopologyInfo` hints, and in **02b/09.5** you learned GDR needs GPU and NIC on the
same PCIe root complex / rail. Everything so far assumed the NIC was simply *there*. On
Kubernetes it isn't: the default CNI gives a pod exactly one interface (`eth0`) on the pod
network, and that interface is not your RDMA device. This lesson is the plumbing that puts
a *second, RDMA-capable* interface into the pod and makes its underlying VF a schedulable,
NUMA-tagged resource so Topology Manager can co-align it with the GPU. New pieces:
**Multus** (meta-CNI for a secondary interface), the **SR-IOV network device plugin** (VFs
as resources), the **RDMA shared device plugin** (share one RDMA device), and the
**NVIDIA Network Operator** (bundles all of it plus the OFED driver).

## Core notes

### Why the default CNI isn't enough — what Multus adds

The default CNI (Calico, Cilium, …) is a *single-interface* model: one `eth0` per pod on
the cluster pod network, for regular TCP/IP service traffic. RDMA needs a second interface
bound to the physical InfiniBand/RoCE NIC (or a VF of it), with the RDMA verbs device
visible in the pod. The default CNI has no concept of "also give me `net1` on the IB
fabric."

**Multus** is a *meta-CNI* (a CNI-of-CNIs). It runs as the CNI, calls the default CNI
first to create `eth0` (the pod keeps normal networking), then reads the pod's
`k8s.v1.cni.cncf.io/networks` annotation and, for each entry, delegates to another CNI
plugin to attach an **additional** interface (`net1`, `net2`, …). Each attachment is
described by a **`NetworkAttachmentDefinition`** (NAD) CRD, which carries the CNI config
(e.g. the `sriov` CNI, `ipoib`, or `macvlan`) plus IPAM. Multus's own contribution is
*orchestration of multiple interfaces*; the actual RDMA-capable interface is created by the
delegate CNI the NAD names.

### Making the VF schedulable — SR-IOV device plugin

Attaching an interface isn't enough; the scheduler must *account* for the NIC hardware so
two pods don't claim the same Virtual Function. **SR-IOV** partitions one physical NIC
(the Physical Function, PF) into many **Virtual Functions (VFs)**, each a lightweight PCIe
function with its own RDMA context. The **SR-IOV network device plugin** discovers those
VFs and advertises them to the kubelet as an **extended resource**, exactly like the GPU
operator advertises `nvidia.com/gpu`. The resource name is configurable — e.g.
`nvidia.com/sriov_rdma`, `rdma/rdma_vf`, or on the NVIDIA operator path something like
`nvidia.com/hostdev` — and a pod requests it in `resources.limits`. The scheduler then
treats VFs as a countable, exhaustible resource per node.

Crucially, the device plugin also reports each VF's **NUMA node** via the device-plugin
API `TopologyInfo`. That is what lets Topology Manager do its job (below).

### RDMA shared device plugin — the no-SR-IOV path

Not every deployment wants SR-IOV (VF count is fixed at PF config time, and some clouds
don't expose it). The **RDMA shared device plugin** (`k8s-rdma-shared-dev-plugin`)
advertises a *shared* RDMA device — e.g. `rdma/rdma_shared_device_a` — that **multiple
pods use concurrently** off one PF, typically paired with a `macvlan` or `ipoib` secondary
interface. You trade the hard isolation of a dedicated VF for simpler provisioning; for
many inference and single-tenant training nodes that's the right call. SR-IOV gives
isolation and per-VF scheduling; shared gives density and no VF plumbing.

### The bundle — NVIDIA Network Operator

Wiring OFED drivers, Multus, the device plugin, the RDMA plugin, IPoIB CNI, and IPAM by
hand across 40+ clusters is how you get drift. The **NVIDIA Network Operator** deploys and
lifecycle-manages the whole stack from one cluster-scoped CRD, the **`NicClusterPolicy`**.
It bundles:

- **OFED / DOCA-OFED driver** container (`ofedDriver`) — the `mlx5` kernel modules and
  user-space verbs, matched to the running kernel.
- **Multus** (`secondaryNetwork.multus`) — the meta-CNI.
- **SR-IOV device plugin** (`sriovDevicePlugin`) — VFs as resources. (SR-IOV VF *creation*
  and NAD generation is driven by the companion SR-IOV Network Operator /
  `SriovNetworkNodePolicy`.)
- **RDMA shared device plugin** (`rdmaSharedDevicePlugin`) — shared RDMA resources.
- **IPoIB CNI** and **IPAM** (`nvIpam` / whereabouts) — addressing for the secondary net.
- **NV Peer Memory / nvidia-peermem** hookup so GDR (09.5) actually works.

It is the *networking* counterpart to the GPU Operator, and the two are designed to run
side by side: GPU Operator owns `nvidia.com/gpu`, Network Operator owns the RDMA resources.

### Where it interlocks with Topology Manager (06)

Now the payoff. A GDR-optimal pod requests **both** `nvidia.com/gpu: 1` **and** an RDMA
resource (`nvidia.com/sriov_rdma: 1` or `rdma/…: 1`). Both device plugins report NUMA
affinity via `TopologyInfo`. With the kubelet `topologyManagerPolicy: single-numa-node`
(from 06), Topology Manager will only admit the pod if it can allocate the GPU **and** the
NIC VF **and** the CPUs on **one** NUMA node — the coarse Kubernetes proxy for 09.5's
"same rail / same root complex." If the only free GPU is on NUMA1 and the only free VF is
on NUMA0, admission fails with a **`TopologyAffinityError`** rather than silently placing a
cross-NUMA pod. That's the behavior you *want*: a hard failure you can see beats a Running
pod at half bandwidth. Under the looser `best-effort` policy the pod is admitted anyway and
the misalignment becomes an invisible performance tax — the exact cost/observability trap.

**Where alignment breaks in practice:** (1) VF pools not pinned per NUMA, so the plugin
advertises VFs the scheduler can't NUMA-match to the GPU; (2) `best-effort` policy masking
misalignment; (3) the NAD pointing at a PF on the wrong socket for the GPUs the node
schedules; (4) requesting the RDMA resource but pinning `NCCL_IB_HCA` (09.5) to a
different rail's device, so K8s aligns the VF but NCCL still uses the wrong NIC.

## Worked example

Three manifests, annotated. **`NicClusterPolicy`** (cluster-scoped, one per cluster):

```yaml
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata: { name: nic-cluster-policy }
spec:
  ofedDriver:                          # (1) mlx5 kernel modules + verbs, per-kernel
    image: doca-driver
    repository: nvcr.io/nvidia/mellanox
    version: "24.10-0.7.0.0"
  sriovDevicePlugin:                   # (2) advertise VFs as a schedulable resource
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

**`NetworkAttachmentDefinition`** (namespaced; what Multus attaches as `net1`):

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: rdma-net
  annotations:
    k8s.v1.cni.cncf.io/resourceName: nvidia.com/sriov_rdma   # (5) tie NAD to the VF resource
spec:
  config: |
    { "cniVersion": "0.3.1",
      "type": "sriov",                                        # (6) delegate CNI = sriov
      "ipam": { "type": "whereabouts",
                "range": "192.168.100.0/24" } }               # (7) address on the fabric subnet
```

**Pod spec** (requests GPU + VF, attaches the secondary net):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nccl-worker
  annotations:
    k8s.v1.cni.cncf.io/networks: rdma-net                     # (8) Multus attaches net1 from this NAD
spec:
  containers:
    - name: trainer
      image: nvcr.io/nvidia/pytorch:24.10-py3
      resources:
        limits:
          nvidia.com/gpu: 1                                   # (9) GPU device plugin
          nvidia.com/sriov_rdma: 1                            # (10) SR-IOV VF — the same NAD resource
      securityContext:
        capabilities: { add: ["IPC_LOCK"] }                   # (11) required to pin RDMA memory regions
```

Read it as a chain: (8) tells Multus which NAD → (5) that NAD is backed by resource (10) →
(10) is advertised by the plugin in (2) → both (9) GPU and (10) VF report NUMA via
`TopologyInfo` → with `single-numa-node` the kubelet co-allocates them on one NUMA node or
rejects with `TopologyAffinityError`.

**Where NUMA/GPU/NIC alignment can go wrong here:** if the node's free GPU (9) is NUMA1 but
the only free `sriov_rdma` VF (10) is NUMA0, `single-numa-node` → admission failure (good,
visible); `best-effort` → Running pod, cross-NUMA GDR path, ~half bandwidth (bad, silent).
And even when K8s aligns them, the container must still set `NCCL_IB_HCA` (09.5) to the
`mlx5` device that *is* GPU (9)'s paired rail, or NCCL uses the wrong NIC despite correct
scheduling. Missing `IPC_LOCK` (11) → `ibv_reg_mr` fails and RDMA silently degrades.

## Practice

Feeds the deliverable's **annotated multi-NIC manifest set**.

1. Take the three manifests above (or a real set from your cluster / the Network Operator
   repo examples).
2. Add a line-by-line annotation: for **every** non-trivial line, one clause on what it
   does and which component consumes it (Multus, SR-IOV plugin, device plugin, IPAM,
   kubelet/Topology Manager, NCCL).
3. Mark **three** distinct places NUMA/GPU/NIC alignment can break, and for each state the
   symptom (`TopologyAffinityError` vs silent half-bandwidth vs `ibv_reg_mr` failure) and
   the fix.

**Acceptance:** an annotated `NicClusterPolicy` + NAD + pod-spec set with the resource
chain traced end to end and the three failure modes called out with symptom and fix.
Done when a reader who has never seen these CRDs can point to the exact line that puts the
RDMA interface in the pod, the exact line that makes the VF schedulable, and the exact
policy that forces NUMA alignment.

## Self-check

1. **Why isn't the default CNI enough for RDMA — what does Multus add?**
   **Answer:** The default CNI gives a pod a single `eth0` on the cluster pod network for
   ordinary TCP/IP; it has no way to attach a second interface bound to the RDMA NIC/VF.
   Multus is a meta-CNI: it creates `eth0` via the default CNI, then reads the pod's
   `k8s.v1.cni.cncf.io/networks` annotation and delegates to another CNI (sriov/ipoib/
   macvlan, named by a `NetworkAttachmentDefinition`) to attach an additional RDMA-capable
   interface (`net1`). It orchestrates multiple interfaces; the delegate CNI builds the
   actual RDMA one.

2. **What resource does the SR-IOV device plugin advertise, and why must the scheduler
   NUMA-align it with the GPU (tie to 06/02b)?**
   **Answer:** It advertises NIC **Virtual Functions** as a countable extended resource
   (e.g. `nvidia.com/sriov_rdma`), just like `nvidia.com/gpu`, and reports each VF's NUMA
   node via the device-plugin `TopologyInfo`. Alignment matters because GPUDirect RDMA
   needs the GPU and NIC on the same PCIe root complex / rail (02b, 09.5); NUMA node is the
   scheduler's proxy for that. Topology Manager's `single-numa-node` policy (06) co-allocates
   GPU + VF + CPU on one NUMA node or rejects with `TopologyAffinityError` — preventing a
   silently cross-NUMA, half-bandwidth GDR path.

3. **What does the NVIDIA Network Operator bundle?**
   **Answer:** From one `NicClusterPolicy` CRD it deploys and lifecycle-manages the OFED/
   DOCA-OFED driver (`mlx5`), Multus, the SR-IOV network device plugin, the RDMA shared
   device plugin, the IPoIB CNI, IPAM (nv-ipam/whereabouts), and the nvidia-peermem hookup
   that GDR needs — the networking counterpart to the GPU Operator, run side by side with
   it.

## Resources

1. **NVIDIA Network Operator (`Mellanox/network-operator`).** The `NicClusterPolicy` CRD
   and what the operator bundles. https://github.com/Mellanox/network-operator
2. **Multus CNI + SR-IOV network device plugin.** The meta-CNI and the VF-advertising
   plugin, with NAD and config examples. https://github.com/k8snetworkplumbingwg/multus-cni
   · https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin
3. **CoreWeave NCCL configuration reference.** Production `NCCL_IB_HCA` / GDR / CollNet
   settings that must match the VF you scheduled.
   https://docs.coreweave.com/products/networking/hpc-interconnect/nccl-configuration-reference
