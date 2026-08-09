---
id: "09"
title: "Networking and topology"
notion: "https://app.notion.com/p/3b33abaeb82381f69438c603703f933f"
phase: "Phase 4 · Months 12–16 (deferrable stretch)"
effort: "~40 hrs ≈ 3–4 weeks @ 10–12 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["02b"]
unlocks: []
started: null
completed: null
---

# 🔗 09 — Networking and topology

> **Goal.** Understand the interconnect well enough to make **placement and procurement
> arguments**. You target the platform/K8s track, but must argue credibly with the
> fabric track — and conceptual depth is enough (hard to practice without a multi-node cluster).

- **Notion page:** https://app.notion.com/p/3b33abaeb82381f69438c603703f933f
- **Phase:** Phase 4 · **deferrable stretch** · **Est. effort:** ~40 hrs ≈ 3–4 weeks
- **Deliverable:** [Network architecture read + placement/procurement argument](practice/network-architecture-read/) —
  finishable with published architectures + a 2-GPU NCCL bench.

## Why this module, and to what bar

- **CoreWeave** — *HPC Network Engineer, InfiniBand*: "operating some of the largest **InfiniBand fabrics**"; *Sr Network Engineer, DC*: "low-latency fabrics (**InfiniBand, RoCE**), **EVPN underlays**, data center routing at scale." (Runs **Quantum-X800 IB + SHARP** and **Spectrum-X RoCE**; publishes an NCCL config reference + `nccl-tests`.)
- **NVIDIA** — *Sr SWE, RDMA Transport*; *Sr Architect, GPU Fabric Networking* (sets the vocabulary ceiling).
- **Interview probes:** *IB vs RoCE tradeoffs* · *"read `nvidia-smi topo -m` and place a job"* · *"why does RoCE need lossless Ethernet"* (PFC/ECN/DCQCN) · *rail-optimized topology* · *what a 1:7 oversubscription ratio means* · *NCCL all-reduce as the acceptance test.*
- The market expects you to (1) read a published architecture and predict bottlenecks, (2) argue IB-vs-RoCE and oversubscription in $/perf terms, (3) wire GPUDirect RDMA through K8s and prove it with NCCL. **That is this module.**

## Calibrated to your background — what we skip

You did 02b (NVLink/PCIe topology, `nvidia-smi topo -m`), 08 (NCCL), 01b (kernel
datapath), 06 (topology-aware scheduling). So we **reference, not re-teach** those and
extend the topology matrix **outward past the NIC** to the switch fabric. RDMA is taught
as *the kernel-bypass that deletes the 01b datapath* — conceptual, not verbs-coding.

## Lessons

Builds outward: 02b matrix → fabric → protocol → K8s → cost. **IB-vs-RoCE (L4) is the highest-yield lesson.**

| # | Lesson | Hrs | Decision |
|---|--------|-----|----------|
| 01 | [Intra → inter handoff](lessons/01-intra-to-inter-handoff.md) | 5 | extend `topo -m` past the NIC; GPU→rail→NIC |
| 02 | [Inter-node fabric (Clos, rail-optimized, oversubscription)](lessons/02-inter-node-fabric.md) | 6 | compute the oversubscription ratio |
| 03 | [RDMA fundamentals](lessons/03-rdma-fundamentals.md) | 5 | why RDMA deletes the 01b datapath |
| 04 | [**IB vs RoCEv2 + lossless**](lessons/04-ib-vs-roce-lossless.md) (highest yield) | 6 | procurement: when each wins |
| 05 | [GPUDirect over the fabric + SHARP](lessons/05-gpudirect-and-sharp.md) | 5 | which NIC per GPU; is the IB premium worth it? |
| 06 | [Kubernetes multi-NIC (Multus/SR-IOV/Network Operator)](lessons/06-k8s-multi-nic.md) | 5 | how RDMA reaches a pod (the K8s differentiator) |
| 07 | [**Bandwidth as a cost input**](lessons/07-bandwidth-as-cost.md) | 5 | turn placement into a bandwidth + $ statement |

Total ≈ **40 hrs ≈ 3–4 weeks** (conceptual; drop the week-4 buffer to compress to 3). Spine = L2 + L4 + the capstone.

## Resource spine

- **Llama 3 paper §3.3.1 "Network"** — the anchor (real 24K-GPU RoCE Clos, 1:7 oversubscription).
- **Ghobadi rail-optimized paper** — *why* rail-optimized works.
- **NVIDIA HGX AI Factory reference architecture** — the "ideal" to contrast with the real one.
- **Glenn Lockwood's LLM-networking notes** — critical practitioner judgment (not vendor gloss).
- **NVIDIA Network Operator / Multus / SR-IOV** repos + **CoreWeave's NCCL config reference** for the K8s stack.

> ⚠️ Anchor generations: Quantum-2 = NDR 400G, **Quantum-X800 = XDR 800G**, Spectrum-X = RoCE;
> ConnectX-7 (400G) / ConnectX-8 (800G); BlueField-3 DPU. Older material predates these.

## Deliverable & checkpoint

- Build the **[Network architecture read](practice/network-architecture-read/)** — a 3–5 page doc:
  read the Llama-3 24K topology, predict where a named job bottlenecks under two placements,
  make the co-location argument with real bandwidth numbers, pick IB vs RoCE for a procurement
  scenario, and reason about oversubscription — grounded in one 2-GPU `nccl-tests` run.
- The [**checkpoint**](checkpoint.md) is the gate — read a `topo -m`, compute an
  oversubscription ratio, argue IB-vs-RoCE two ways, and turn topology into a bandwidth number.

## How to work this module

1. Deep-read Llama-3 §3.3.1 first — it's the capstone's primary source.
2. Hands-on is limited (no multi-node fabric) — lean on reading real architectures + one 2-GPU NCCL bench.
3. Answer the [checkpoint](checkpoint.md) from memory; flip `status` and update Notion when the doc is done.
