---
lesson: "09.5"
title: "GPUDirect and SHARP"
module: "09"
concept: "GPUDirect and SHARP"
status: not-started
est_time: "5h"
artifacts: []
---

# 09.5 · GPUDirect and SHARP

> **Concept.** GPUDirect RDMA extends the same-root-complex rule across the fabric so a NIC DMAs straight into a peer GPU's HBM, and SHARP moves the all-reduce sum into the switch ASIC.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Why this matters

At 40+ clusters your training jobs live and die on the tail of every all-reduce.
A single H100 pair that silently falls back to a CPU-staged copy across nodes will
drag an entire data-parallel step to the slowest rank, and you pay for the whole
world size to sit idle. GPUDirect RDMA and SHARP are the two levers that keep the
data path off the CPU and, for SHARP, off the GPUs entirely during the reduction.
Knowing when they engage — and how to prove they did from `nvidia-smi topo -m` plus
the NCCL env — is the difference between a fabric you can defend in a procurement
review and one that quietly wastes 30% of a 512-GPU reservation.

## What's new here

In **02b** you learned that GPUDirect P2P inside a node needs the two GPUs under the
same PCIe root complex — a CPU root port bounce kills the peer DMA. This lesson takes
that exact rule and pushes it *across the wire*: replace "peer GPU" with "peer NIC,"
and "same root complex" with "same rail." In **08** you learned NCCL runs the
all-reduce as a ring/tree of point-to-point sends. Here the transport underneath those
sends becomes GPUDirect RDMA, and SHARP replaces the reduction arithmetic itself with
in-switch compute. Three new mechanisms: **GPUDirect RDMA** (NIC↔GPU peer DMA over the
fabric), **GPUDirect Storage / GDS** (NVMe or network storage ↔ GPU HBM), and **SHARP**
(switch-ASIC in-network reduction).

## Core notes

### GPUDirect RDMA — the same-root-complex rule, over the fabric

A normal RDMA read/write moves bytes between two hosts' *system memory* without a CPU
copy on either side. GPUDirect RDMA (GDR) extends the RDMA target: the NIC's DMA engine
reads/writes a **BAR-mapped region of GPU HBM directly** over PCIe. The GPU exposes its
memory through a PCIe BAR; the NIC issues PCIe peer-to-peer transactions to that BAR.
End to end for an inter-node NCCL send:

```
sender:   GPU HBM → (PCIe P2P) → NIC → wire
receiver: wire → NIC → (PCIe P2P) → GPU HBM
```

There is **no bounce buffer in host RAM** and **no CPU memcpy** on the data path — the
CPU only sets up the queue pair and posts work requests once. Compare the non-GDR path,
which stages every message: `GPU HBM → cudaMemcpy → pinned host buffer → NIC → wire`,
and the mirror on receive. That staging adds two PCIe crossings, host memory bandwidth
pressure, and CPU involvement per message.

**The 02b rule reappears exactly.** PCIe P2P from NIC to GPU is only clean when the two
devices share a PCIe switch (or at worst a single host bridge). If the NIC hangs off a
different root complex than the GPU, the P2P transaction has to traverse the CPU's
inter-socket link (UPI/xGMI) or the root ports, which either fails outright or collapses
bandwidth. On an HGX H100/H200 board this is designed for you: GPUs are **railed** —
each GPU sits under a PCIe switch with a paired ConnectX-7 NIC, and all the "GPU0's NIC"
across every node in the cluster form **rail 0**. GPUDirect RDMA across nodes wants GPU
*i* talking out its rail-*i* NIC to another node's rail-*i* NIC. That is the fabric-level
statement of "same root complex": **same rail**.

### Reading the topology: `nvidia-smi topo -m`

The GPU↔NIC cells use the same legend you already know from 02b, best to worst path:

| Code | Meaning | GDR quality |
|------|---------|-------------|
| `PIX` | single PCIe bridge/switch between them | best |
| `PXB` | multiple PCIe switches, no host bridge crossing | good (typical HGX GPU↔paired NIC) |
| `NODE` | within a NUMA node, crossing the host bridge (PCIe→CPU→PCIe) | CPU on the path |
| `PHB` | through the PCIe host bridge (the CPU) | CPU on the path |
| `SYS` | crosses the SMP interconnect (UPI/QPI) between sockets | worst / may not do GDR |

You pin a pod's NIC with **`NCCL_IB_HCA`** and you gate *how far* NCCL is allowed to use
GDR with **`NCCL_NET_GDR_LEVEL`** (older alias: `NCCL_IB_GDR_LEVEL`). The level is the
**maximum NIC↔GPU distance** at which NCCL will still use GPUDirect RDMA:

- `PIX` — only if NIC and GPU share one PCIe switch
- `PXB` — allow across multiple PCIe switches (no host-bridge crossing)
- `PHB` — allow through the CPU/host bridge (same NUMA node)
- `SYS` — allow even across the SMP interconnect

If unset, NCCL auto-selects; its conservative default keeps GDR inside the same PCIe
complex, which on many HGX layouts *excludes* the GPU↔paired-NIC path you actually want,
so a fallback to host staging happens silently. In production you set it explicitly.
`NCCL_IB_HCA` names the device(s): `mlx5_3`, a list `mlx5_0,mlx5_1`, a port
`mlx5_3:1`, or `=mlx5_3` for an exact (non-prefix) match.

### GPUDirect Storage (GDS)

Same idea, storage side. With the cuFile API, a read from local NVMe or an NVMe-oF /
network filesystem DMAs **directly into GPU HBM** — no `read()` into a page-cache buffer
followed by `cudaMemcpy`. It removes the host bounce buffer from the checkpoint-load and
dataset-ingest paths, which matters when you reload a multi-hundred-GB optimizer state or
stream training shards fast enough to keep the GPUs fed. Requires the `nvidia-fs` kernel
module and a GDS-aware storage stack.

### SHARP — in-network reduction

SHARP (Scalable Hierarchical Aggregation and Reduction Protocol) moves the **arithmetic
of the collective into the InfiniBand switch ASIC**. Without SHARP, an all-reduce over N
ranks moves each rank's data around a ring so every GPU does its share of the summation —
GPU cycles and 2×(N−1)/N × data bytes on the wire. With SHARP, each endpoint sends its
buffer **once up a reduction tree** whose interior nodes are the switches; the switch's
Aggregation Node sums the contributions from its children and forwards one reduced result
up, then the final result is multicast back down. The GPUs never run the reduction and
each leaf sends its data essentially once.

- **Which generation:** SHARPv2 on Quantum-2 (NDR, H100/H200-class), SHARPv3 on
  Quantum-X800 (XDR). Data-reduction (floating-point sum) SHARP is the v2+ feature; the
  original v1 was barrier/small-payload aggregation.
- **How NCCL uses it:** through the **CollNet** transport, enabled with
  **`NCCL_COLLNET_ENABLE=1`** plus the NCCL-RDMA-SHARP plugin (ships in HPC-X), a running
  `sharpd` / aggregation manager, and SHARP-enabled fabric. NCCL then advertises
  `CollNet` as an algorithm alongside Ring/Tree.
- **Measured effect:** neocloud and vendor numbers land around a **~20–40% all-reduce
  wall-clock improvement** for the message sizes and world sizes where the collective is
  the bottleneck; Lambda's 1CC write-up reports gains in that band at scale.
- **Which shapes win:** SHARP helps most where the reduction, not raw link bandwidth, is
  the cost — **latency-bound small/medium all-reduce** (e.g. gradient all-reduce at large
  world size, frequent small syncs) and **all-reduce / reduce / broadcast / barrier**
  specifically. It does nothing for point-to-point-shaped traffic (all-to-all in
  expert-parallel MoE, pipeline sends), which never touches the reduction tree.

Tie back to **08**: SHARP doesn't change *that* you call `all_reduce`; it changes the
algorithm NCCL picks under it and where the sum executes. Your bucket sizes and overlap
strategy from 08 still set whether the collective is on the critical path at all.

## Worked example

Node: HGX H100, 8×GPU, 8×ConnectX-7, rail-optimized. Trimmed `nvidia-smi topo -m`:

```
        GPU0  GPU1  GPU2  GPU3  ...  NIC0  NIC1  NIC2  NIC3  CPUAffinity  NUMA
GPU0     X    NV18  NV18  NV18       PXB   SYS   SYS   SYS   0-47         0
GPU1    NV18   X    NV18  NV18       SYS   PXB   SYS   SYS   0-47         0
GPU2    NV18  NV18   X    NV18       SYS   SYS   PXB   SYS   48-95        1
GPU3    NV18  NV18  NV18   X         SYS   SYS   SYS   PXB   48-95        1
```

`NICk` here is `mlx5_k`. Read GPU-to-NIC only: GPU0↔NIC0 = `PXB` (paired, same PCIe
switch group), GPU0↔NIC1/2/3 = `SYS` (crosses the socket interconnect). So the peer NIC
for GPU0 is **NIC0 = `mlx5_0`**, and by the diagonal, GPU*k*'s NIC is `mlx5_k`.

**Decision for a pod pinned to GPU3:**

```yaml
env:
  - { name: NCCL_IB_HCA,        value: "mlx5_3" }   # GPU3's rail, PXB path
  - { name: NCCL_NET_GDR_LEVEL, value: "PXB" }      # allow GDR across PCIe switches, not through CPU
  - { name: NCCL_IB_PCI_RELAXED_ORDERING, value: "1" }
  - { name: NCCL_SOCKET_IFNAME, value: "eth0" }     # bootstrap only, keep off the IB rail
```

Why `PXB` and not `PIX`: the paired path shows `PXB`, so `PIX` would be too strict and
disable GDR on the very link we want. Why not `SYS`: that would *allow* GDR out
`mlx5_0..2`, which for GPU3 is a cross-socket path — worse bandwidth and defeats rail
alignment. If you must let a GPU use a non-paired NIC (degraded node), `SYS` is the knob,
and you accept the hit.

**Prove it engaged:** run with `NCCL_DEBUG=INFO` and grep the init log for
`[GDRDMA]` / `via NET/IB/0/GDRDMA` on the GPU3 rank. Absence of `GDRDMA` means it fell
back to host staging — the symptom you're hunting.

**If SHARP is available on this fabric**, add `NCCL_COLLNET_ENABLE=1` and confirm NCCL
prints `CollNet` as a selected algorithm for the all-reduce; expect the ~20–40% band on
large-world-size gradient reductions, nothing on all-to-all.

## Practice

Feeds the deliverable's **HCA-pinning decision**.

1. Take the `nvidia-smi topo -m` above (or your own node's real output) plus the
   NIC→`mlx5_*` map.
2. For **each** of GPU0–GPU3, state the NIC to pin and produce the pod env block:
   `NCCL_IB_HCA`, `NCCL_NET_GDR_LEVEL`, and whether SHARP (`NCCL_COLLNET_ENABLE`) applies
   to the workload's collective shape.
3. Write one paragraph justifying the `NCCL_NET_GDR_LEVEL` choice from the topo codes, and
   name the exact `NCCL_DEBUG=INFO` log token you'd grep to confirm GDR engaged.

**Acceptance:** a per-GPU HCA-pinning table (NIC + the three NCCL env values) plus the
one-line grep check, dropped into the deliverable. It is done when someone could paste
your env block into a pod spec and, on that topology, get GDR on the paired rail — no
silent host-staging fallback.

## Self-check

1. **What breaks GPUDirect RDMA if the GPU and NIC are on different PCIe root complexes,
   and how does that relate to 02b?**
   **Answer:** GDR is a PCIe peer-to-peer DMA between the NIC and the GPU's BAR-mapped
   HBM. On different root complexes the P2P transaction must route through the CPU root
   ports and the inter-socket link (UPI/xGMI), which either isn't supported for P2P or
   collapses bandwidth — so NCCL falls back to staging through pinned host memory. It's
   the same constraint as 02b's intra-node GPU↔GPU P2P (same root complex), just with the
   NIC as the peer and expressed at fabric scale as "same rail." `nvidia-smi topo -m`
   showing `SYS` on the GPU↔NIC cell is the tell.

2. **Which collective shapes benefit most from SHARP in-network reduction?**
   **Answer:** Reduction-shaped, latency-bound collectives: **all-reduce**, reduce,
   broadcast, and barrier — especially small/medium messages at large world size where
   the reduction (not link bandwidth) dominates, e.g. gradient all-reduce in
   data-parallel training. Point-to-point-shaped traffic — all-to-all in MoE
   expert-parallel, pipeline-parallel sends — never enters the reduction tree and gets no
   SHARP benefit.

3. **What does GPUDirect RDMA remove from the data path versus a CPU-staged transfer?**
   **Answer:** The host bounce buffer and the per-message CPU memcpy on both sender and
   receiver. Non-GDR: `GPU HBM → cudaMemcpy → pinned host RAM → NIC → wire` and the
   mirror. GDR: `GPU HBM → PCIe P2P → NIC → wire`. It eliminates two extra PCIe crossings,
   host-memory bandwidth pressure, and CPU involvement per message; the CPU only sets up
   the queue pair once.

## Resources

1. **NVIDIA GPUDirect RDMA — Developer docs.** The BAR-mapping and P2P-DMA mechanics,
   authoritative. https://docs.nvidia.com/cuda/gpudirect-rdma/
2. **NVIDIA SHARP on Lambda 1CC — measured all-reduce gains.** Neocloud write-up with the
   ~20–40% band and CollNet setup in context.
   https://lambda.ai/blog/nvidia-sharp-on-lambda-1cc
3. **NCCL Environment Variables — `NCCL_NET_GDR_LEVEL`, `NCCL_IB_HCA`, `NCCL_COLLNET_ENABLE`.**
   The exact value semantics you tune against.
   https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html
