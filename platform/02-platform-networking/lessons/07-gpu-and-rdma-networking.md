---
lesson: "A02.7"
title: "GPU and RDMA networking (platform-integration angle)"
module: "A-02"
concept: "provisioning + operating RDMA on K8s"
status: not-started
est_time: "3 hrs"
artifacts: ["RDMA node acceptance-test runbook"]
---

# A02.7 · GPU and RDMA networking (platform-integration angle)

> **Concept.** The fabric can be physically perfect and a single misconfigured device plugin, NUMA-misaligned VF, or un-drained flapping NIC still silently halves training throughput — this lesson is the operations layer over the module-09 fabric.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Why this matters
Module 09 gets the photons right: RoCEv2, PFC/ECN/DCQCN, GPUDirect RDMA, rail-optimized Clos, oversubscription math. None of that reaches a training job until Kubernetes hands a pod a NIC that is (a) an actual RDMA device, (b) on the correct NUMA node relative to its GPU, and (c) on the correct rail relative to the other 7 nodes in the job. Every one of those is a platform decision you own, and every one fails *silently* — no error, no crash, just a 4-8× dataplane collapse on a cluster billing ~$40/hr/node. Staff-level here means you can reason about the Network Operator's composition, own the DRA-vs-device-plugin migration, and produce an acceptance gate that catches a bad node *before* it joins the pool and poisons a collective.

## Core notes
**Skip (you already know):** RoCEv2 vs InfiniBand and PFC/ECN/DCQCN lossless mechanics; GPUDirect RDMA data path and NCCL ring/tree collectives; rail-optimized topology + oversubscription math and that SR-IOV VFs + Multus are the K8s vehicle — all covered in module 09.

**The NVIDIA Network Operator stack — the pieces and how they compose.** The primary CNI (Calico/Cilium) owns `eth0` for pod-to-pod and API traffic. RDMA rides a *secondary* network. Name what each piece does:

- **MOFED/driver container** — loads the OFED kernel modules (`mlx5_core`, `ib_core`, `rdma_cm`) onto the host, or you use inbox drivers. This is where firmware/driver skew lives.
- **RDMA shared device plugin** — exposes the *whole* HCA as a countable resource `rdma/rdma_shared_device_a` shared across pods on the node. Netdev stays in the root namespace; pods get RDMA access but not a dedicated NIC. Cheap, no VFs, no isolation, one shared traffic class.
- **SR-IOV Network Device Plugin** — discovers SR-IOV **Virtual Functions** and advertises them as allocatable (`nvidia.com/sriov_rdma_vf` or similar). Each pod gets a *dedicated* VF with its own netdev, PCI address, and QoS. The real-fabric choice.
- **Multus** — the meta-CNI. Calls the primary CNI for `eth0`, then attaches additional interfaces per a **NetworkAttachmentDefinition** annotation. Without Multus there is no second interface to put RDMA on.
- **SR-IOV CNI** — the delegate Multus invokes to move a VF into the pod netns and configure it.
- **NVIDIA IPAM** (`nv-ipam`) — assigns IPs to RDMA interfaces from a pool without DHCP on the fabric.
- **RDMA CNI** — moves the RDMA device (the `/dev/infiniband` char devices and rdma cgroup) into the pod's isolated netns so the RDMA subsystem is namespaced, not leaked into root.

**Which combo:**
- *Shared-HCA RoCE* (dev clusters, inference, cost-sensitive): RDMA shared device plugin + Multus + macvlan/host-device CNI. No VFs, simplest, no isolation.
- *Dedicated-VF SR-IOV* (production training): SR-IOV Network Device Plugin + Multus + SR-IOV CNI + RDMA CNI + NVIDIA IPAM, driven by an `SriovNetworkNodePolicy` (numVFs, resourceName, device selectors) and an `SriovNetwork`/NetworkAttachmentDefinition.

**DRA is displacing the device-plugin model.** The device-plugin API is countable-and-opaque: it hands out "one of N `nvidia.com/gpu`" with no way to express "and a NIC that is NUMA-local to *that* GPU." **Dynamic Resource Allocation (DRA)** replaces counting with declarative **ResourceClaims** — a pod says "I need a GPU and a NIC in the same NUMA/PCIe domain" and a resource driver satisfies it with topology awareness. The **DRA SR-IOV driver / DraNet** allocate VFs as structured resources and can bind a GPU + rail-aligned NIC in one claim. **Operational gotcha you own:** you cannot run the DRA SR-IOV driver and the SR-IOV Network Device Plugin simultaneously for the same resources on a node — enabling DRA disables the device plugin. Migrating a live fleet is a staged decision (feature-gate enablement, driver rollout, rewriting manifests from `resources.limits` to `resourceClaims`), not a flag flip.

**Topology alignment is the whole game operationally.** A GPU scheduled with a NIC on the *wrong NUMA node* still works — traffic just traverses the inter-socket link (UPI/xGMI) instead of a NUMA-local PCIe path. NCCL's topology engine reads the PCIe/NUMA graph and allocates channels accordingly: a NUMA-aligned NIC gets **8-16 channels**; a SYS-distant NIC gets **~2**. That is a silent 4-8× reduction in effective collective bandwidth with **zero errors logged**. The platform job is to make the scheduler co-locate GPU + rail-aligned NIC:
- **Node labels** for `rack`, `block`, `nvlink-domain`, `rail` so gang scheduling pins a job to one rail group.
- **Topology-aware scheduling** — Topology Manager `single-numa-node` policy on the kubelet; DRA ResourceClaims binding GPU+NIC atomically in one claim.
- Surfacing the resulting topology to NCCL so it doesn't have to guess.

**Provisioning & day-2 ops.** Acceptance and health, not physics:
- **Fabric validation:** `ib_write_bw` / `ib_read_bw` point-to-point between candidate nodes to confirm line-rate RDMA before anything else.
- **Collective acceptance gate:** `nccl-tests` `all_reduce_perf` — the number that matters is **bus bandwidth (busbw)**, not algbw; busbw normalizes for GPU count and is the real fabric health signal. A node that can't hit expected busbw fails acceptance.
- **Lossless-fabric health:** PFC pause-frame counters (a storm means a slow receiver is back-pressuring the fabric), ECN mark rates, and the cardinal rule — *one bad link degrades the entire collective* because the slowest participant sets the pace of an all-reduce.
- **Fleet skew:** firmware/MOFED version drift across nodes causes subtle interop and performance faults; version-pin and audit.
- **VF exhaustion:** `numVFs` is fixed at policy time; running out means pods pend with no obvious cause.
- **Draining:** a flapping RDMA NIC must get the node **cordoned and drained out of the schedulable pool** before it lands in a job — a flapping link inside a running collective stalls every rank.

**Scheduler/topology integration.** NetworkAttachmentDefinition must coexist cleanly with the primary CNI (Multus ordering, MTU consistency with the fabric — jumbo/9000). Gang + topology scheduling ensures an 8-node job lands on one rail group, not scattered across the Clos. Surface topology to NCCL explicitly: `NCCL_TOPO_FILE` (hand it the PCIe/NIC graph), `NCCL_IB_HCA` (which HCAs to use, exclude the wrong ones), `NCCL_SOCKET_IFNAME` (bootstrap/OOB interface — must be the primary net, not the RDMA net).

## Worked example
Stand up (or read the manifests for) a 2-node RDMA setup:
1. Install the NVIDIA Network Operator; confirm MOFED modules load (`lsmod | grep mlx5`) and the SR-IOV device plugin advertises the VF resource (`kubectl describe node` shows `nvidia.com/sriov_rdma_vf: 8`).
2. Create the `SriovNetworkNodePolicy` (numVFs, resourceName, PF selector) and the `SriovNetwork`/NetworkAttachmentDefinition wiring RDMA CNI + NVIDIA IPAM.
3. Schedule a pod requesting both `nvidia.com/gpu: 1` and the SR-IOV VF resource, annotated onto the NAD via Multus.
4. Run `nccl-tests` `all_reduce_perf -b 8 -e 128M -f 2 -g <gpus>`; record **busbw** at the large-message asymptote — your baseline acceptance number.
5. **Deliberately mis-pin:** force the VF onto a NIC on the wrong NUMA node relative to the GPU, re-run, and observe NCCL (`NCCL_DEBUG=INFO`) report a collapsed channel count (~2 vs 8-16) and busbw fall 4-8× — *with no errors*. The failure mode made visible.

**Deliverable:** an RDMA node acceptance-test runbook — **fabric validate** (`ib_write_bw` line rate) → **nccl-test gate** (busbw ≥ threshold) → **topology check** (NCCL channel count / NUMA-alignment assertion) — that a node must pass before joining the schedulable pool.

## Practice
Build the 2-node manifests and the acceptance runbook above; then break the NUMA pinning and capture the before/after busbw + channel count as the evidence the runbook is designed to catch. Carry the artifact into [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md), where the "all-reduce got slower" scenario reuses these signals.

## Self-check
- Why can a GPU pod with a working, line-rate RDMA NIC still run its all-reduce 4-8× slower than an identical pod, with no errors anywhere? **Answer:** The NIC is on the wrong NUMA node relative to the GPU. NCCL's topology engine sees a SYS-distant path and allocates only ~2 channels instead of 8-16; traffic crosses the inter-socket link. It's a silent bandwidth collapse — the fix is topology-aware scheduling (Topology Manager single-numa-node / DRA claim binding GPU+NIC atomically), not anything on the fabric.
- What breaks if you enable the DRA SR-IOV driver without decommissioning the SR-IOV Network Device Plugin, and why is this a staff-owned migration? **Answer:** They conflict — you cannot run both for the same VF resources on a node; enabling DRA disables the device plugin. It's staff-owned because it's a live-fleet migration: staged feature-gate enablement, driver rollout, and rewriting every workload manifest from countable `resources.limits` to declarative `resourceClaims`, not a single flag flip.
- On a node acceptance test, why is nccl-tests **busbw** the gate rather than `ib_write_bw` alone or algbw? **Answer:** `ib_write_bw` only proves a point-to-point link is healthy; it says nothing about the collective. busbw is the collective's normalized bus bandwidth — it factors out GPU count and exposes real end-to-end fabric health including topology/channel effects, so a node passing ib_write_bw can still fail the busbw gate (e.g. NUMA misalignment). algbw isn't normalized for participant count, so it isn't comparable across job sizes.

## References
- https://docs.nvidia.com/networking/display/kubernetes2640/overview.html
- https://mellanox.github.io/network-operator-docs/dra-sriov-driver/dra-sriov-driver.html
- https://developer.nvidia.com/blog/running-ai-workloads-on-rack-scale-supercomputers-from-hardware-to-topology-aware-scheduling/
- https://github.com/NVIDIA/nccl-tests
