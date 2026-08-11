---
lesson: "09.5"
title: "GPUDirect over the fabric + SHARP"
module: "09"
concept: "GPUDirect and SHARP"
status: not-started
est_time: "7h"
prev: "04-ib-vs-roce-lossless.md"
next: "06-k8s-multi-nic.md"
artifacts: []
sources: 8
---

# 09.5 · GPUDirect over the fabric + SHARP

> **Concept.** GPUDirect RDMA extends the same-root-complex rule across the fabric so a NIC DMAs straight into a peer GPU's HBM, and SHARP moves the all-reduce sum into the switch ASIC.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Where this fits

**04** answered the fabric-technology question — IB vs RoCEv2, and *why* each is (or must be engineered to be) lossless. That lesson treated the fabric as a pipe and asked "does it drop packets." This lesson asks a different question about the same pipe: **once bytes are moving losslessly, whose job is it to move them, and does anything have to move at all?** GPUDirect RDMA answers the first half — it takes the CPU out of the data path entirely, across nodes, the same way NVLink P2P takes it out of the data path inside a node (02b). SHARP answers the second half — for a whole class of collectives, it deletes the *movement itself* by doing the reduction inside the switch. Both are levers you pull only once the lossless-fabric question from 04 is already settled; you cannot GDR or SHARP your way out of a fabric that drops packets. What this unlocks: the next lesson (06) is entirely about how Kubernetes gets a pod's GPU and a pod's NIC onto the same PCIe rail in the first place — this lesson is the reason that plumbing matters.

## Why this matters

At 40+ clusters your training jobs live and die on the tail of every all-reduce. A single H100 pair that silently falls back to a CPU-staged copy across nodes will drag an entire data-parallel step to the slowest rank, and you pay for the whole world size to sit idle — GPU-hours burned with nothing to show for them. GPUDirect RDMA and SHARP are the two levers that keep the data path off the CPU and, for SHARP, off the GPUs entirely during the reduction. Knowing when they engage — and how to prove they did from `nvidia-smi topo -m` plus the NCCL env — is the difference between a fabric you can defend in a procurement review and one that quietly wastes 30% of a 512-GPU reservation. This is also a live interview probe: "which collective shapes benefit most from SHARP" and "what breaks GPUDirect RDMA across root complexes" both sit on the module 09 checkpoint, and both require you to reason from mechanism, not memorize a marketing slide.

## What's new here (calibration)

- You already know 02b's rule — GPU↔GPU P2P needs a shared PCIe root complex — and 08's NCCL collective model (ring/tree of point-to-point sends). We do **not** re-teach either.
- New here: the *fabric-scale* restatement of the 02b rule (NIC↔GPU peer DMA, "same rail" as the multi-node analog of "same root complex"), the concrete NCCL knobs that enforce it (`NCCL_NET_GDR_LEVEL`, `NCCL_IB_HCA`), and how to read the proof of engagement out of an NCCL debug log — not just cite the concept.
- New here: SHARP as a *hardware reduction engine in switch silicon*, not a software trick — why that ties it irrevocably to InfiniBand, what generation runs on which switch, and which collective shapes it does and does not help (a MoE all-to-all gets nothing from it, and MoE-shaped traffic is an increasingly large share of 2025/2026 production training).
- New here: the production caveat that vendor docs don't advertise — CoreWeave's own NCCL guidance says *do not* flip on `NCCL_GDRCOPY_ENABLE` and assume it helps. That's the kind of fleet-tested judgment a staff engineer is expected to carry into a procurement or incident conversation.

## Core concepts

### GPUDirect RDMA — the same-root-complex rule, over the fabric

A normal RDMA read/write moves bytes between two hosts' *system memory* without a CPU copy on either side. GPUDirect RDMA (GDR) extends the RDMA target: the NIC's DMA engine reads/writes a **BAR-mapped region of GPU HBM directly** over PCIe. ("BAR" = Base Address Register — the mechanism a PCIe device uses to expose a chunk of its own memory into the host's address space, so another device can address it directly.) The GPU exposes its memory through a PCIe BAR; the NIC issues PCIe peer-to-peer transactions to that BAR. End to end for an inter-node NCCL send:

```
sender:   GPU HBM → (PCIe P2P) → NIC → wire
receiver: wire → NIC → (PCIe P2P) → GPU HBM
```

There is **no bounce buffer in host RAM** and **no CPU memcpy** on the data path — the CPU only sets up the queue pair and posts work requests once. Compare the non-GDR path, which stages every message: `GPU HBM → cudaMemcpy → pinned host buffer → NIC → wire`, and the mirror on receive. That staging adds two PCIe crossings, host memory bandwidth pressure, and CPU involvement per message.

**The 02b rule reappears exactly.** PCIe P2P from NIC to GPU is only clean when the two devices share a PCIe switch (or at worst a single host bridge). If the NIC hangs off a different root complex than the GPU, the P2P transaction has to traverse the CPU's inter-socket link (UPI/xGMI) or the root ports, which either fails outright or collapses bandwidth. On an HGX H100/H200 board this is designed for you: GPUs are **railed** — each GPU sits under a PCIe switch with a paired ConnectX-7 NIC, and all the "GPU0's NIC" across every node in the cluster form **rail 0**. GPUDirect RDMA across nodes wants GPU *i* talking out its rail-*i* NIC to another node's rail-*i* NIC. That is the fabric-level statement of "same root complex": **same rail**.

### Reading the topology: `nvidia-smi topo -m`

The GPU↔NIC cells use the same legend you already know from 02b, best to worst path:

| Code | Meaning | GDR quality |
|------|---------|-------------|
| `PIX` | single PCIe bridge/switch between them | best |
| `PXB` | multiple PCIe switches, no host bridge crossing | good (typical HGX GPU↔paired NIC) |
| `NODE` | within a NUMA node, crossing the host bridge (PCIe→CPU→PCIe) | CPU on the path |
| `PHB` | through the PCIe host bridge (the CPU) | CPU on the path |
| `SYS` | crosses the SMP interconnect (UPI/QPI) between sockets | worst / may not do GDR |

You pin a pod's NIC with **`NCCL_IB_HCA`** and you gate *how far* NCCL is allowed to use GDR with **`NCCL_NET_GDR_LEVEL`** (older alias: `NCCL_IB_GDR_LEVEL`). The level is the **maximum NIC↔GPU distance** at which NCCL will still use GPUDirect RDMA:

- `PIX` — only if NIC and GPU share one PCIe switch
- `PXB` — allow across multiple PCIe switches (no host-bridge crossing)
- `PHB` — allow through the CPU/host bridge (same NUMA node)
- `SYS` — allow even across the SMP interconnect

If unset, NCCL auto-selects; its conservative default keeps GDR inside the same PCIe complex, which on many HGX layouts *excludes* the GPU↔paired-NIC path you actually want, so a fallback to host staging happens silently. In production you set it explicitly. `NCCL_IB_HCA` names the device(s): `mlx5_3`, a list `mlx5_0,mlx5_1`, a port `mlx5_3:1`, or `=mlx5_3` for an exact (non-prefix) match.

### GPUDirect Storage (GDS)

Same idea, storage side. With the cuFile API, a read from local NVMe or an NVMe-oF / network filesystem DMAs **directly into GPU HBM** — no `read()` into a page-cache buffer followed by `cudaMemcpy`. It removes the host bounce buffer from the checkpoint-load and dataset-ingest paths, which matters when you reload a multi-hundred-GB optimizer state or stream training shards fast enough to keep the GPUs fed. Requires the `nvidia-fs` kernel module and a GDS-aware storage stack.

### SHARP — in-network reduction

SHARP (Scalable Hierarchical Aggregation and Reduction Protocol) moves the **arithmetic of the collective into the InfiniBand switch ASIC**. Without SHARP, an all-reduce over N ranks moves each rank's data around a ring so every GPU does its share of the summation — GPU cycles and 2×(N−1)/N × data bytes on the wire. With SHARP, each endpoint sends its buffer **once up a reduction tree** whose interior nodes are the switches; the switch's Aggregation Node sums the contributions from its children and forwards one reduced result up, then the final result is multicast back down. The GPUs never run the reduction and each leaf sends its data essentially once.

- **Which generation:** **SHARPv3 on Quantum-2 (NDR, 400G — H100/H200-class fleets)**, **SHARPv4 on Quantum-X800 (XDR, 800G)**. (If you've seen older material claim "SHARPv2 on Quantum-2 / SHARPv3 on Quantum-X800," that mapping is off by one generation — the pairing above is the current, correct one.) Data-reduction (floating-point sum) SHARP is the v2+ feature; the original v1 was barrier/small-payload aggregation. As a concrete throughput anchor: Quantum-X800's in-network compute engine delivers **14.4 TFLOPS of in-network compute — roughly 9× the prior (NDR) generation — across 144×800G ports, at sub-100ns port-to-port latency.** That is a hardware reduction ASIC doing real floating-point work at line rate, not a marketing abstraction.
- **How NCCL uses it:** through the **CollNet** transport, enabled with **`NCCL_COLLNET_ENABLE=1`** plus the NCCL-RDMA-SHARP plugin (ships in HPC-X), a running **`sharpd`** (the SHARP aggregation-manager daemon) and a SHARP-enabled fabric. NCCL then advertises `CollNet` as an algorithm alongside Ring/Tree — but it's a candidate NCCL *may* pick, not a guarantee; if `sharpd` isn't running or the fabric isn't SHARP-provisioned, NCCL silently falls back to Ring/Tree even with the env var set.
- **Measured effect — two different benchmarks, don't conflate them.** Lambda's own production measurements on their 1CC (one-click cluster) infrastructure show **~45–63% all-reduce bandwidth improvement**, measured across cluster sizes from 16 to 1,500 GPUs. Separately, NVIDIA-published benchmarks on other workloads report **17% faster BERT training wall-clock and up to 8× reduction in communication latency** on SHARP-enabled systems — a different model, different metric, different cluster. Treat these as two independent data points that both point the same direction, not one number restated twice.
- **Which shapes win:** SHARP helps most where the reduction, not raw link bandwidth, is the cost — **latency-bound small/medium all-reduce** (e.g. gradient all-reduce at large world size, frequent small syncs) and **all-reduce / reduce / broadcast / barrier** specifically. It does **nothing** for point-to-point-shaped traffic — all-to-all in expert-parallel MoE, pipeline-parallel sends — which never touches the reduction tree. As MoE-style architectures become a larger share of 2025/2026 production training, this is a growing carve-out, not a footnote: a cluster tuned around SHARP gains for a dense model can see none of that benefit once the workload mix shifts to MoE.

Tie back to **08**: SHARP doesn't change *that* you call `all_reduce`; it changes the algorithm NCCL picks under it and where the sum executes. Your bucket sizes and overlap strategy from 08 still set whether the collective is on the critical path at all.

## Perspectives

**Developer.** From inside a training script, SHARP is invisible — it's just another NCCL transport (`CollNet`) that NCCL may or may not select for a given collective, and GPUDirect RDMA is just "NCCL got faster." Your job doesn't change: you still size gradient buckets and structure overlap (08) so the collective is off the critical path. What *does* change is your debugging instinct — when an all-reduce is slower than expected, "did GDR/SHARP actually engage" becomes a real first question, answerable from the NCCL init log, not a guess.

**Operator.** GDR-level tuning (`NCCL_NET_GDR_LEVEL`, `NCCL_IB_HCA`) is a **per-fleet, per-SKU** setting, not a global constant. A value that's correct for an H100 HGX board (`PXB`, paired ConnectX-7) can be silently wrong on the next node generation with a different PCIe topology or NIC count — and nothing pages you when it's wrong, throughput just quietly drops. Every new node SKU that lands in the fleet needs its `topo -m` re-read and its GDR level re-validated; treating last generation's tuning as portable is a recurring, invisible source of underperformance across 40+ clusters.

**Hardware.** SHARP is not a clever software trick layered on top of ordinary switches — it's a genuine in-network compute engine burned into the InfiniBand switch ASIC, with real floating-point throughput (14.4 TFLOPS on Quantum-X800) and real silicon area. That's precisely *why* it's IB-only: RoCE/Ethernet switches, including NVIDIA's own Spectrum-X, don't carry that reduction engine — they optimize congestion and routing, not arithmetic. If someone tells you "we'll get SHARP-like gains on RoCE with better tuning," the hardware answer is no: there is no ASIC there to do the sum.

**Economics.** SHARP is a byte-reduction lever, and byte reduction is a direct input to the IB-vs-RoCE TCO calculus from 04 and 07. On a reduction-heavy dense-model trainer, SHARP's ~halved wire bytes can be the specific line item that makes the IB premium pay for itself — either by hitting the same step time on a cheaper (more oversubscribed) fabric, or by pushing effective bandwidth past what RoCE could match. But that argument evaporates for an all-to-all-heavy MoE workload: no SHARP benefit means the IB premium has to be justified on other axes (determinism, tuning risk) alone, which is a much weaker case at scale. The verdict on "is the IB tax worth it" is not fixed — it moves with the collective mix of the workload you're procuring for.

## Real-world use cases

- **Lambda Labs — "Introducing NVIDIA SHARP on Lambda 1CC."** Named neocloud's in-production, measured SHARP gains: ~45–63% all-reduce bandwidth improvement across cluster sizes from 16 to 1,500 GPUs on their 1-Click Cluster offering. Shows SHARP's benefit holding across a wide range of scale, not just at the extreme end. https://lambda.ai/blog/nvidia-sharp-on-lambda-1cc
- **CoreWeave — GPUDirect RDMA how-to (InfiniBand).** Operational setup guidance from a production GPU cloud: the OFED driver and NCCL requirements, and the `rdma/ib: 1` pod resource request used as a boolean scheduling gate onto RDMA-capable nodes. Shows what GDR setup actually looks like on a real multi-tenant Kubernetes fleet, not a single-tenant lab box. https://docs.coreweave.com/docs/products/networking/hpc-interconnect/use-gpudirect-rdma
- **CoreWeave — NCCL configuration reference.** Real fleet-tested guidance on `NCCL_IB_HCA`, GDR levels, and CollNet settings — including the explicit caution (see Pitfalls) against assuming `NCCL_GDRCOPY_ENABLE` helps without benchmarking. Shows the gap between "the env var exists" and "the env var is safe to flip on in production." https://docs.coreweave.com/products/networking/hpc-interconnect/nccl-configuration-reference

## Worked example

Node: HGX H100, 8×GPU, 8×ConnectX-7, rail-optimized. Trimmed `nvidia-smi topo -m`:

```
        GPU0  GPU1  GPU2  GPU3  ...  NIC0  NIC1  NIC2  NIC3  CPUAffinity  NUMA
GPU0     X    NV18  NV18  NV18       PXB   SYS   SYS   SYS   0-47         0
GPU1    NV18   X    NV18  NV18       SYS   PXB   SYS   SYS   0-47         0
GPU2    NV18  NV18   X    NV18       SYS   SYS   PXB   SYS   48-95        1
GPU3    NV18  NV18  NV18   X         SYS   SYS   SYS   PXB   48-95        1
```

`NICk` here is `mlx5_k`. Read GPU-to-NIC only: GPU0↔NIC0 = `PXB` (paired, same PCIe switch group), GPU0↔NIC1/2/3 = `SYS` (crosses the socket interconnect). So the peer NIC for GPU0 is **NIC0 = `mlx5_0`**, and by the diagonal, GPU*k*'s NIC is `mlx5_k`.

**Decision for a pod pinned to GPU3:**

```yaml
env:
  - { name: NCCL_IB_HCA,        value: "mlx5_3" }   # GPU3's rail, PXB path
  - { name: NCCL_NET_GDR_LEVEL, value: "PXB" }      # allow GDR across PCIe switches, not through CPU
  - { name: NCCL_IB_PCI_RELAXED_ORDERING, value: "1" }
  - { name: NCCL_SOCKET_IFNAME, value: "eth0" }     # bootstrap only, keep off the IB rail
```

Why `PXB` and not `PIX`: the paired path shows `PXB`, so `PIX` would be too strict and disable GDR on the very link we want. Why not `SYS`: that would *allow* GDR out `mlx5_0..2`, which for GPU3 is a cross-socket path — worse bandwidth and defeats rail alignment. If you must let a GPU use a non-paired NIC (degraded node), `SYS` is the knob, and you accept the hit.

**Prove it engaged:** run with `NCCL_DEBUG=INFO` and grep the init log for `[GDRDMA]` / `via NET/IB/0/GDRDMA` on the GPU3 rank. Absence of `GDRDMA` means it fell back to host staging — the symptom you're hunting.

**If SHARP is available on this fabric**, add `NCCL_COLLNET_ENABLE=1`, confirm `sharpd` is running on the fabric, and confirm NCCL prints `CollNet` as a selected algorithm for the all-reduce; expect the gains from the two benchmark bands above on large-world-size gradient reductions, nothing on all-to-all.

## Practice

Feeds the deliverable's **HCA-pinning decision**.

1. Take the `nvidia-smi topo -m` above (or your own node's real output) plus the NIC→`mlx5_*` map.
2. For **each** of GPU0–GPU3, state the NIC to pin and produce the pod env block: `NCCL_IB_HCA`, `NCCL_NET_GDR_LEVEL`, and whether SHARP (`NCCL_COLLNET_ENABLE`) applies to the workload's collective shape.
3. Write one paragraph justifying the `NCCL_NET_GDR_LEVEL` choice from the topo codes, and name the exact `NCCL_DEBUG=INFO` log token you'd grep to confirm GDR engaged.
4. Write one sentence stating whether you would enable `NCCL_GDRCOPY_ENABLE` on this fleet, and what you would do *before* flipping it on in production (tie to Pitfalls, below).

**Acceptance:** a per-GPU HCA-pinning table (NIC + the three NCCL env values) plus the one-line grep check, dropped into the deliverable. It is done when someone could paste your env block into a pod spec and, on that topology, get GDR on the paired rail — no silent host-staging fallback.

## Common pitfalls

- **Citing the SHARP-generation mapping inconsistently.** The correct pairing is **SHARPv3 on Quantum-2 (NDR)** and **SHARPv4 on Quantum-X800 (XDR)**. Older or careless material sometimes shifts this by one generation — always state the switch generation *and* the SHARP version together, and double-check against a current NVIDIA source before you repeat it in an interview or a procurement doc.
- **Enabling `NCCL_GDRCOPY_ENABLE` and assuming it always helps.** GDRCopy is a *different* mechanism from GPUDirect RDMA — it lets the CPU read/write GPU memory directly for small control-path operations, not the NIC↔GPU data path. CoreWeave's own production NCCL guidance explicitly advises against turning it on without benchmarking first: in their testing it has shown no measurable gain for standard all-reduce, and can regress performance. Treat it as an experiment, not a default.
- **Assuming SHARP applies to every collective.** SHARP accelerates reduction-shaped traffic (all-reduce, reduce, broadcast, barrier). It does nothing for all-to-all — the dominant traffic shape in expert-parallel MoE — because that traffic never enters the switch's reduction tree. A cluster procured on SHARP's promised gains for a dense model will not see those gains once the workload mix includes MoE.
- **Forgetting SHARP has runtime dependencies beyond a single env var.** `NCCL_COLLNET_ENABLE=1` alone is not sufficient — you also need a running `sharpd` aggregation-manager daemon and a SHARP-enabled fabric (switches provisioned for it, the NCCL-RDMA-SHARP plugin present). Without those, NCCL silently falls back to Ring/Tree; nothing errors, it's just slower than you budgeted for.
- **Trusting NCCL's default GDR level on a new node generation.** NCCL's auto-selected default is conservative and, on many HGX layouts, *excludes* the exact GPU↔paired-NIC path you want. Always set `NCCL_NET_GDR_LEVEL` explicitly from a real `topo -m` read rather than trusting the default to find the paired rail.

## Self-check

- **What breaks GPUDirect RDMA if the GPU and NIC are on different PCIe root complexes, and how does that relate to 02b?**
  **Answer:** GDR is a PCIe peer-to-peer DMA between the NIC and the GPU's BAR-mapped HBM. On different root complexes the P2P transaction must route through the CPU root ports and the inter-socket link (UPI/xGMI), which either isn't supported for P2P or collapses bandwidth — so NCCL falls back to staging through pinned host memory. It's the same constraint as 02b's intra-node GPU↔GPU P2P (same root complex), just with the NIC as the peer and expressed at fabric scale as "same rail." `nvidia-smi topo -m` showing `SYS` on the GPU↔NIC cell is the tell.

- **Why is SHARP unavailable on RoCE/Ethernet fabrics?**
  **Answer:** SHARP's reduction executes inside a dedicated in-network compute engine built into the InfiniBand switch ASIC — real silicon doing floating-point sums at line rate (14.4 TFLOPS on Quantum-X800). RoCE/commodity Ethernet switches, including NVIDIA's own Spectrum-X, don't carry that compute engine; Spectrum-X optimizes routing and congestion control, not arithmetic. It's an architectural property tied to IB switch silicon, not a software or protocol gap you can tune your way around.

- **Which collective shapes benefit most from SHARP in-network reduction?**
  **Answer:** Reduction-shaped, latency-bound collectives: all-reduce, reduce, broadcast, and barrier — especially small/medium messages at large world size where the reduction (not link bandwidth) dominates, e.g. gradient all-reduce in data-parallel training. Point-to-point-shaped traffic — all-to-all in MoE expert-parallel, pipeline-parallel sends — never enters the reduction tree and gets no SHARP benefit.

- **What did CoreWeave's own NCCL reference caution against, and why does that matter for procurement/ops decisions?**
  **Answer:** Enabling `NCCL_GDRCOPY_ENABLE` without first benchmarking on the actual fleet — their own testing found no measurable gain for standard all-reduce and noted it can regress performance in some cases. It matters because it's a real example of a plausible-sounding NCCL tuning knob that a vendor doc doesn't warn you off, but a production operator's fleet-tested guidance does — exactly the kind of judgment call that separates "read the docs" from "operate the fleet."

## Connections & what's next

GDR is the fabric-scale continuation of 02b's PCIe P2P rule, and it's the concrete mechanism underneath every inter-node NCCL send from 08. SHARP is a direct extension of 04's IB-vs-RoCE argument: it's one of the concrete reasons the "IB tax" from that lesson can pay for itself on the right workload, and 07 will turn that into an explicit dollar figure — SHARP as a byte-reduction lever that changes the bandwidth-per-dollar math on the spine tier. This lesson assumed a pod could simply *have* a rail-aligned NIC to pin `NCCL_IB_HCA` against; **06** is the reality check — it's the Kubernetes plumbing (Multus, SR-IOV, the Network Operator, Topology Manager) that actually gets a second, RDMA-capable, NUMA-aligned interface into a pod in the first place. Read 06 as "everything in this lesson, but now the resource is a schedulable Kubernetes object that has to survive admission control."

## References & further reading

**Primary sources**
- NVIDIA GPUDirect RDMA — Developer docs. The BAR-mapping and P2P-DMA mechanics, authoritative. https://docs.nvidia.com/cuda/gpudirect-rdma/
- NVIDIA GPUDirect Storage — Overview Guide. The cuFile API and direct NVMe/network-storage↔HBM DMA path referenced in Core concepts. https://docs.nvidia.com/gpudirect-storage/overview-guide/index.html
- NCCL Environment Variables — `NCCL_NET_GDR_LEVEL`, `NCCL_IB_HCA`, `NCCL_COLLNET_ENABLE`. The exact value semantics you tune against. https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html
- NVIDIA Quantum-X800 InfiniBand Platform — product page. Source for the 14.4 TFLOPS in-network compute, 144×800G ports, and sub-100ns port-to-port figures. https://www.nvidia.com/en-us/networking/products/infiniband/quantum-x800/

**Real-world engineering blogs**
- Lambda Labs — "Introducing NVIDIA SHARP on Lambda 1CC." Measured 45–63% all-reduce bandwidth gains across 16–1,500 GPU scale, plus the NVIDIA BERT/latency benchmark cited alongside it. https://lambda.ai/blog/nvidia-sharp-on-lambda-1cc
- CoreWeave — Use GPUDirect RDMA with InfiniBand. Production GDR setup guidance, including the `rdma/ib` pod resource pattern. https://docs.coreweave.com/docs/products/networking/hpc-interconnect/use-gpudirect-rdma
- CoreWeave — NCCL configuration reference. Real fleet guidance, including the `NCCL_GDRCOPY_ENABLE` benchmark-first caution. https://docs.coreweave.com/products/networking/hpc-interconnect/nccl-configuration-reference

**Deeper dives**
- NVIDIA Technical Blog — "Advancing Performance with NVIDIA SHARP In-Network Computing." The mechanism behind the reduction-tree model and the multi-tenant SHARP capability introduced in v3. https://developer.nvidia.com/blog/advancing-performance-with-nvidia-sharp-in-network-computing/
