---
lesson: "02b.4"
title: "The 8-GPU server: HGX/DGX H100 topology"
module: "02b"
concept: "The 8-GPU server: HGX/DGX H100 topology"
status: not-started
est_time: "7.5h"
prev: "03-pcie.md"
next: "05-topology-alignment-k8s.md"
artifacts: []
sources: 12
---
# 02b.4 · The 8-GPU server: HGX/DGX H100 topology

> **Concept.** The canonical 8x H100 node — drawn from memory — and how to map a rented instance's real GPUs onto it.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)
>
> **Advanced Learning** — [The 8-GPU Node](../../../learning/04-server-architecture-8gpu.html): the same eight GPUs drawn as two separate graphs, to check your from-memory version against. Optional visual companion; this lesson stays the source of truth.

## Where this fits

Lesson 03 gave you the single-link skill: read `LnkCap` vs `LnkSta` vs `LnkCap2` vs `LnkCtl2`, walk a PCIe tree, derive usable GB/s from a gen×width reading, and tell a configuration fact from a hardware fault. That answers *"is this one link healthy?"* It does not answer *"should this GPU's traffic even be on PCIe at all, and if it is, is it going to the right place?"*

This lesson supplies the reference topology that makes those questions answerable: the fixed, designed layout of a standard 8-GPU HGX/DGX node, including the **second fabric** — NVLink and NVSwitch — that does not exist on any machine you have run before this course. Everything you diagnose from here through lesson 08's capstone gets compared against the picture built here. Lesson 01 taught you to read a tree; lesson 02 taught you what its edges cost; lesson 03 taught you to verify one edge; this lesson tells you what the whole tree is supposed to look like.

## Why this matters

Every diagnosis in this module bottoms out in one question: *is this GPU where it should be in the machine, and is it talking to the right neighbour over the right link?* You cannot answer that from `nvidia-smi topo -m` output alone — you need the **reference topology in your head** to compare against. When a rented "8× H100" box shows GPU5's NIC reachable only via `SYS` instead of `PIX`, you only know that is wrong because you know GPU5 and its rail NIC are *supposed* to be co-located on socket 1 under a shared PCIe switch. Without the reference, the output is just letters.

The stakes are structural, not marginal. On this class of node there are two completely different networks connecting the same eight GPUs, differing in bandwidth by roughly **7×** and in topology entirely:

- **NVLink/NVSwitch** carries GPU↔GPU traffic at 450 GB/s per direction per GPU, in a non-blocking all-to-all fabric that never touches the CPU, the root complex, or the inter-socket link.
- **PCIe** carries GPU↔host, GPU↔NIC and GPU↔storage at 63 GB/s per direction, in a tree that is entirely NUMA-bound.

Confuse the two and you optimise the wrong thing. A team that "fixed" a slow training job by verifying GPU–GPU collectives were on NVLink fixed the half that was already fine, while the NIC on the wrong socket kept every inter-node all-reduce crossing UPI. That is the classic version of this failure, and it is invisible to every per-GPU metric, because the GPU is genuinely busy the whole time.

At neocloud scale you accept fleets you did not build. The reference-versus-real mapping *is* your acceptance test. Being able to sketch the HGX H100 node from memory — sockets, NVSwitches, retimers, PCIe switches, rail-aligned NICs — is table-stakes senior platform fluency in this space, and it is a live whiteboard question in the interviews this course targets.

## What's new here (calibration)

You know NUMA (lesson 01), memory-tier bandwidth (lesson 02), and PCIe link training (lesson 03). What is new is the **whole node as a designed system**, plus a fabric that has no analogue in your on-prem background:

- **NVLink and NVSwitch as a fabric parallel to PCIe.** GPU-to-GPU traffic does **not** ride PCIe on this class of node. It rides NVLink through dedicated switch silicon on the GPU baseboard. This breaks an assumption from on-prem work: GPU↔GPU is **not** NUMA-bound here, even though GPU↔CPU, GPU↔NIC and GPU↔storage still are.
- **The generation table with the arithmetic** — links per GPU, GB/s per link per direction, and therefore total, for NVLink 1 through 5, so you can derive rather than memorise.
- **Where the PCIe switches actually are.** There is no PCIe switch on the GPU baseboard; there are several on the CPU/motherboard tray. That distinction is the whole reason `PIX` between a GPU and its NIC is achievable at all, and it is the part most descriptions get wrong.
- **The GPU-to-socket split as a fixed design fact** (GPU0–3 → socket 0, GPU4–7 → socket 1), not something the OS decides.
- **Rail alignment** — the 1:1 GPU:NIC pairing that makes multi-node training fast, why the NIC's *physical* placement relative to its GPU is what matters, and why production nodes run two separate NIC fabrics.
- **Collective arithmetic** — the NCCL bus-bandwidth formula, so you can decide from first principles whether a given collective is bounded by NVLink, by PCIe, or by the network.

## Core concepts

### 1. Two networks, one set of GPUs

This is the diagram to commit to memory. It shows the **same eight GPUs twice**: once as leaves of the PCIe tree, once as nodes in the NVLink fabric. They are different graphs.

```
 ══════════════════ NETWORK 1: THE PCIe TREE (NUMA-bound, 63 GB/s/dir per link) ══════════════════

        Socket 0 — Xeon Platinum 8480C          UPI ×4          Socket 1 — Xeon Platinum 8480C
        8 ch DDR5-4800 = 307 GB/s          ◀══════════▶         8 ch DDR5-4800 = 307 GB/s
         │            │                    ~38 GB/s/dir/link      │            │
    root port    root port                                   root port     root port
    Gen5 x16     Gen5 x16                                    Gen5 x16      Gen5 x16
         │            │                                          │            │
   ┌─────┴────┐  ┌────┴─────┐                              ┌─────┴────┐  ┌────┴─────┐
   │  PCIe    │  │  PCIe    │   ← Broadcom Gen5 switches   │  PCIe    │  │  PCIe    │
   │ switch 0 │  │ switch 1 │     ON THE CPU/MOTHERBOARD   │ switch 2 │  │ switch 3 │
   └┬──┬──┬──┬┘  └┬──┬──┬──┬┘     TRAY, not the baseboard  └┬──┬──┬──┬┘  └┬──┬──┬──┬┘
    │  │  │  │    │  │  │  │                                │  │  │  │    │  │  │  │
   G0 G1 N0 N1   G2 G3 N2 N3                               G4 G5 N4 N5   G6 G7 N6 N7
    │  │           │  │                                     │  │          │  │
    │  │  +NVMe    │  │  +NVMe                              │  │  +NVMe   │  │  +NVMe
    │  │           │  │                                     │  │          │  │
    ╎  ╎           ╎  ╎        each GPU's link to its switch ╎  ╎          ╎  ╎
    ╎  ╎           ╎  ╎        is Gen5 x16 CARRIED OVER      ╎  ╎          ╎  ╎
    ╎  ╎           ╎  ╎        RETIMERS to the GPU baseboard ╎  ╎          ╎  ╎
    ▼  ▼           ▼  ▼                                      ▼  ▼          ▼  ▼

 ══════════════════ NETWORK 2: THE NVLINK FABRIC (not NUMA-bound, 450 GB/s/dir per GPU) ══════════

              ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
              │ GPU0  │ │ GPU1  │ │ GPU2  │ │ GPU3  │ │ GPU4  │ │ GPU5  │ │ GPU6  │ │ GPU7  │
              │H100SXM│ │       │ │       │ │       │ │       │ │       │ │       │ │       │
              │80GB   │ │       │ │       │ │       │ │       │ │       │ │       │ │       │
              │HBM3   │ │       │ │       │ │       │ │       │ │       │ │       │ │       │
              └─┬─┬─┬─┘ └─┬─┬─┬─┘ └─┬─┬─┬─┘ └─┬─┬─┬─┘ └─┬─┬─┬─┘ └─┬─┬─┬─┘ └─┬─┬─┬─┘ └─┬─┬─┬─┘
                │ │ │     │ │ │     │ │ │     │ │ │     │ │ │     │ │ │     │ │ │     │ │ │
                │ │ │  18 NVLink-4 ports per GPU, distributed across ALL FOUR switches
                │ │ │  (5/5/4/4 — 18 does not divide evenly by 4)
                ▼ ▼ ▼     ▼ ▼ ▼     ▼ ▼ ▼     ▼ ▼ ▼     ▼ ▼ ▼     ▼ ▼ ▼     ▼ ▼ ▼     ▼ ▼ ▼
        ┌──────────────────────────────────────────────────────────────────────────────────┐
        │   NVSwitch 0        NVSwitch 1        NVSwitch 2        NVSwitch 3               │
        │   (3rd gen)         (3rd gen)         (3rd gen)         (3rd gen)                │
        │   64 NVLink-4 ports each · 3.2 TB/s full-duplex each · SHARP in-network reduce   │
        │                    NON-BLOCKING ALL-TO-ALL WITHIN THE 8-GPU DOMAIN               │
        │        (surplus ports exit via OSFP to external NVLink Switch → 256-GPU domain)  │
        └──────────────────────────────────────────────────────────────────────────────────┘

           Aggregate GPU↔GPU: 8 × 900 GB/s bidirectional = 7.2 TB/s
           Bisection (any 4 vs the other 4): 4 × 900 GB/s = 3.6 TB/s

 ═════════════════════════════════════════════════════════════════════════════════════════════════
   THE POINT: GPU4 and GPU0 are on OPPOSITE SOCKETS in network 1 (topo -m says SYS between
   their NICs) and DIRECT NEIGHBOURS in network 2 (topo -m says NV18 between the GPUs).
   Both statements are true simultaneously. They describe different fabrics.
 ═════════════════════════════════════════════════════════════════════════════════════════════════
```

*(Reference shape for `HGX H100 SXM5 8-GPU`, the same baseboard inside a DGX H100 and inside OEM/neocloud "8× H100 SXM" servers. NVSwitch count, port count and bisection from NVIDIA's HGX H100 platform documentation; switch and NIC placement on the CPU tray from OEM platform block diagrams. Exact NIC-to-switch grouping varies by OEM — verify per SKU.)*

### 2. NVLink: the generation table, derived

NVLink is a point-to-point, cache-coherent-capable, GPU-native interconnect. A **link** (NVIDIA also says "brick") is a bundle of differential lanes in each direction. What changes between generations is the number of links a GPU has and the per-link rate.

| NVLink gen | Architecture / GPU | Links per GPU | **GB/s per link, per direction** | **GB/s per GPU, per direction** | **Total per GPU (NVIDIA's marketed bidirectional figure)** |
|---|---|---|---|---|---|
| 1.0 | Pascal — P100 | 4 | 20 | 80 | **160 GB/s** |
| 2.0 | Volta — V100 | 6 | 25 | 150 | **300 GB/s** |
| 3.0 | Ampere — A100 | 12 | 25 | 300 | **600 GB/s** |
| 4.0 | Hopper — H100 / H200 | 18 | 25 | 450 | **900 GB/s** |
| 5.0 | Blackwell — B200 / GB200 | 18 | 50 | 900 | **1 800 GB/s** |

**Read the arithmetic, because this is exactly the whiteboard follow-up.** NVIDIA markets the *bidirectional* number. `18 links × 50 GB/s bidirectional = 900 GB/s` is the same statement as `18 links × 25 GB/s per direction = 450 GB/s each way`. When you compare against PCIe — which is universally quoted per direction — **you must use 450, not 900**, or you will overstate the ratio by 2×.

```
  H100 NVLink-4 vs PCIe Gen5 x16, both per direction:
      450 GB/s ÷ 63.0 GB/s = 7.1×

  If you carelessly compare 900 GB/s against 63 GB/s you get 14×,
  which is the most common arithmetic error in this whole subject area.
```

Two generational patterns worth naming: **Volta→Ampere doubled the link count** (6→12) at a constant 25 GB/s per direction; **Hopper→Blackwell doubled the per-link rate** (25→50 GB/s per direction) at a constant 18 links. Same headline result, different engineering.

### 3. NVSwitch: how eight GPUs become non-blocking

A GPU with 18 links could connect directly to seven peers — but the wiring is awkward (18 does not divide by 7), the bandwidth to any single peer is a fraction of the total, and the topology does not scale. **NVSwitch** solves this the way an Ethernet switch does: every GPU connects only to switches, and the switches provide any-to-any routing.

| NVSwitch gen | Used in | Ports per switch | Switches per 8-GPU baseboard | Links per GPU per switch | Notes |
|---|---|---|---|---|---|
| 1st | HGX-2 / DGX-2 (V100) | 18 NVLink-2 | 6 per baseboard (12 in a 16-GPU DGX-2) | 1 | first NVSwitch; 16-GPU domain |
| 2nd | HGX A100 | 36 NVLink-3 | **6** | 2 | 8-GPU domain; 4.8 TB/s aggregate, 2.4 TB/s bisection |
| 3rd | HGX H100 / H200 | 64 NVLink-4 | **4** | 5/5/4/4 | 3.2 TB/s full-duplex per switch; **SHARP** in-network reduction; extensible to a 256-GPU NVLink Network domain via external switches |
| 4th | GB200 NVL72 | NVLink-5 | 9 switch trays per rack | — | 72-GPU single NVLink domain, 130 TB/s aggregate |

**Why the switch count went *down* from six (A100) to four (H100) while bandwidth went up.** NVSwitch 3 has 64 ports where NVSwitch 2 had 36. Eight GPUs × 18 links = 144 GPU-side link terminations. Four switches × 64 ports = 256 ports, comfortably enough — and the surplus is what gets routed out to OSFP cages for the external NVLink Network. On A100, eight GPUs × 12 links = 96 terminations against 36 ports per switch, needing at least three switches for connectivity and six for the non-blocking property with the chosen link distribution. **Fewer, wider switches is the same consolidation story as any network fabric.**

**The two aggregate numbers, and why they differ.** You will see both quoted and they are not the same thing:

```
  AGGREGATE GPU-TO-GPU BANDWIDTH  = 8 GPUs × 900 GB/s bidirectional = 7.2 TB/s
     "If every GPU is transmitting and receiving at full rate simultaneously,
      the fabric carries this much." NVIDIA's DGX H100 datasheet figure.

  BISECTION BANDWIDTH             = 4 GPUs × 900 GB/s bidirectional = 3.6 TB/s
     "Cut the 8-GPU domain in half; this much can cross the cut."
      NVIDIA's HGX H100 platform figure.

  Bisection is exactly half of aggregate here BECAUSE the fabric is
  non-blocking: no cut is worse than any other. On a blocking fabric,
  bisection would be lower than half and the ratio would be the
  oversubscription factor.
```

**SHARP is the third-generation feature that changes collective arithmetic.** NVSwitch 3 includes hardware for multicast and for in-network reduction — the Scalable Hierarchical Aggregation and Reduction Protocol, previously an InfiniBand switch feature. Instead of every GPU sending its gradient chunk around a ring, GPUs send once to the switch, the switch performs the reduction in silicon, and multicasts the result back. NVIDIA reports this delivers roughly **2× the all-reduce throughput** within an 8-GPU H100/H200 server compared with the previous A100 generation. Operationally this matters because it means measured `nccl-tests` bus bandwidth on an H100 node can *exceed* what a naive ring-algorithm calculation predicts — if you see that, the fabric is working, not lying.

### 4. Where the PCIe switches actually are

This is the detail most descriptions get wrong, and it is the one that decides whether GPUDirect RDMA has a fast path.

**There is no PCIe switch on the GPU baseboard.** The HGX baseboard carries eight GPUs, four NVSwitches, and the power/cooling infrastructure. Each GPU exposes **one independent PCIe Gen5 x16 stub** that leaves the baseboard through the board-to-board connector. There is no silicon merging those eight stubs, and there does not need to be, because GPU↔GPU traffic is not on PCIe.

**The PCIe switches are on the CPU/motherboard tray.** In an 8-GPU server, Broadcom PCIe Gen5 switches sit between the CPU root ports and the devices, and each switch typically fans out to a small group: **two GPU stubs, two ConnectX-7 NICs, and NVMe drives.** That grouping is the entire mechanism behind rail alignment — a GPU and its paired NIC are siblings under one switch, so the switch's crossbar can route a GPUDirect RDMA transfer between them without the transaction ever reaching the root complex.

**Retimers carry the baseboard-to-tray segment.** The trace from a CPU root port, through the motherboard, across the board-to-board connector, and onto the GPU baseboard exceeds what a passive Gen5 channel can do (lesson 03 §7: ~31.25 ps unit interval, ~8–12 inches of typical PCB before the eye closes). PCIe Gen5 retimers re-time the signal mid-channel. This is why `LnkCap2` on a GPU in one of these systems reports `Retimer+ 2Retimers+`, and why a failing retimer is a leading cause of a single GPU sitting at Gen4 while its seven siblings are at Gen5.

**Why no switch on the baseboard is a deliberate choice, not an omission.** A switch merging the eight GPU stubs would add shared-bandwidth contention on a path that carries none of the GPU↔GPU traffic — the traffic that would have justified it moved to NVLink. It would also add latency and a failure domain. The designers moved GPU↔GPU off PCIe entirely rather than trying to make PCIe carry it.

**Where DGX and OEM HGX platforms differ.** On a DGX H100 the eight compute ConnectX-7 ICs are not discrete cards: they sit on two **"Cedar" mezzanine modules**, four ICs each, and each module presents two 800G OSFP ports externally — so four OSFP cages carry eight 400 Gb/s NDR ports. OEM HGX platforms (Supermicro, Dell, and others) more commonly use discrete ConnectX-7 adapters in slots behind the same Broadcom switches. **The logical topology is the same; the physical packaging is not**, which is exactly why you verify a new SKU against the reference rather than assuming it.

### 5. What rides where — the traffic table

```
| Traffic                        | Path                                   | NUMA-bound? | Per-direction ceiling |
```

| Traffic | Path | NUMA-bound? | Ceiling per direction |
|---|---|---|---|
| GPU ↔ GPU, same node | NVLink → NVSwitch → NVLink | **No** — never touches CPU or PCIe | 450 GB/s per GPU (H100) |
| GPU ↔ host DRAM | PCIe Gen5 x16 → switch → root port → memory controller | **Yes** — bound to the GPU's socket | 63 GB/s theoretical, ~54 usable |
| GPU ↔ NIC (GPUDirect RDMA) | PCIe, GPU → switch crossbar → NIC | **Yes** — must stay switch-local | min(63 GB/s PCIe, 50 GB/s NIC) |
| GPU ↔ NVMe (GPUDirect Storage) | PCIe, GPU → switch crossbar → drive | **Yes** — bound by PCIe placement | ~14 GB/s per Gen5 x4 drive |
| GPU ↔ remote node's GPU | NVLink → local NIC → wire → remote NIC → NVLink | **Yes** at the NIC hop | 50 GB/s per rail NIC |
| CPU ↔ CPU (coherence, remote DRAM) | UPI / Infinity Fabric | — | ~38 GB/s per link |

**The single most useful line to internalise: GPU–GPU all-reduce inside one node is not affected by NUMA.** It is on the NVSwitch fabric. But the moment data leaves a GPU for host RAM, a NIC, or a disk, PCIe and NUMA locality govern it completely.

**One nuance on "non-blocking."** NVSwitch's guarantee is **per-domain**, not unconditional. Within one HGX baseboard the domain is 8 GPUs and the fabric is genuinely non-blocking for all-to-all patterns. Extended through external NVLink switches the domain grows — 256 GPUs for H100-generation NVLink Network, 72 GPUs in a GB200 NVL72 rack — and the guarantee still holds within that boundary. "Non-blocking" is a statement about the fabric's topology (no cut is worse than the bisection), not a promise that every conceivable traffic pattern will run at peak. Knowing the domain size for the hardware in front of you is part of the fluency.

### 6. Rail alignment

In a multi-node GPU cluster the network is organised into **rails**: NIC *k* on every node connects to leaf switch *k*. "Rail-aligned" means GPU *k*'s traffic egresses on NIC *k* and lands on rail *k*'s leaf switch — so a collective between the GPU5s of many nodes stays on one leaf switch and never traverses the spine.

Two placement requirements must both hold; one without the other still loses the throughput.

```
   REQUIREMENT 1 — WITHIN THE NODE (this module's concern)

     GPU5 ──▶ PCIe switch 3 crossbar ──▶ NIC5 ──▶ wire
       topo -m: GPU5 ↔ NIC5 = PIX (or PXB)
       GPUDirect RDMA: HBM → NIC directly. Host DRAM never touched.
       Cost: one switch hop.  Ceiling: min(63, 50) = 50 GB/s.

   BROKEN VERSION — NIC5 cabled to a socket-0 slot

     GPU5 ──▶ PCIe switch 3 ──▶ root port ──▶ socket 1 ──UPI──▶ socket 0
                                       ──▶ root port ──▶ switch 0 ──▶ NIC5
       topo -m: GPU5 ↔ NIC5 = SYS
       GPUDirect RDMA is NOT a supported fast path across sockets.
       The stack falls back to staging through host DRAM:
         HBM → PCIe → DRAM(node2) → UPI → DRAM(node0) → PCIe → NIC5 → wire
       Ceiling collapses to the UPI share: ~38 GB/s best case,
       ~19 GB/s if a second misplaced flow contends.

   REQUIREMENT 2 — ACROSS THE FABRIC (lesson scope: named, not built here)

     NIC k on EVERY node cables to leaf switch k, consistently, fleet-wide.
     A perfectly aligned single node still loses if the cabling is
     inconsistent across nodes: a collective between GPU5s then has to
     traverse the spine on some node pairs and not others, and the
     slowest pair sets the collective's time.
```

**NCCL is topology-aware, and "aware" is not "correct."** At initialisation NCCL walks sysfs, builds its own topology graph, and chooses rings, trees and channel counts from it. That means it trusts what the drivers and system tools report — and as lesson 03's real-world example showed (NCCL issue #246), NCCL's graph and `nvidia-smi topo -m` can classify the same physical hierarchy differently. When debugging a slow all-reduce, dump both and compare:

```
$ NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=INIT,GRAPH,NET ./all_reduce_perf -b 8 -e 8G -f 2 -g 8 2>&1 | grep -E 'NET/IB|via|Channel|Ring'
node:1234:1234 [0] NCCL INFO NET/IB : Using [0]mlx5_0:1/IB [1]mlx5_1:1/IB ... 
node:1234:1234 [0] NCCL INFO Channel 00/24 :    0   1   2   3   4   5   6   7
node:1234:1234 [0] NCCL INFO Channel 00 : 0[1b000] -> 1[43000] via P2P/CUMEM
node:1234:1234 [0] NCCL INFO NET/IB: GPU Direct RDMA Enabled for HCA 0 'mlx5_0'
```

Two things to read there: **`via P2P/CUMEM`** on GPU-to-GPU channels means NCCL chose the NVLink path (`via SHM` or `via NET` would mean it did not), and **`GPU Direct RDMA Enabled for HCA`** means the GPU–NIC pairing passed NCCL's own locality test. If that second line is absent or reads `Disabled`, NCCL has decided the GPU and the NIC are too far apart — which is the functional confirmation of a `SYS` reading.

**PXN is the mitigation when a rail is misaligned.** NCCL's **PXN** ("PCI × NVLink") path lets a GPU reach a NIC that is not its own local NIC by first hopping over NVLink to a GPU that *is* local to that NIC, then going out over PCIe. So GPU5 with a misplaced NIC5 can route GPU5 → NVLink → GPU4 → NIC4 → wire, paying an NVLink hop (cheap, 450 GB/s) instead of a UPI crossing (expensive, ~38 GB/s shared). It also lets NCCL aggregate up to eight messages into one on the intermediate GPU, improving message rate. `NCCL_PXN_DISABLE` and `NCCL_P2P_PXN_LEVEL` control it. **PXN is a workaround for a topology defect, not a reason the defect does not matter** — it recovers most of the bandwidth but not the rail-locality property on the fabric side.

### 7. Collective arithmetic: is this bounded by NVLink, PCIe, or the network?

This is the mechanical skill the module builds toward. Given a collective, decide from first principles which link bounds it.

**The NCCL bus-bandwidth formula.** `nccl-tests` reports two numbers: **algorithm bandwidth** (`algbw = message_size ÷ time`) and **bus bandwidth** (`busbw`). Bus bandwidth normalises out the collective's inherent data-movement factor so you can compare against hardware peak regardless of rank count. For a ring all-reduce over *n* ranks:

```
  busbw = algbw × 2(n−1)/n

  Why 2(n−1)/n: a ring all-reduce is a reduce-scatter followed by an
  all-gather. Each phase moves (n−1)/n of the data around the ring, so
  each rank sends and receives 2(n−1)/n × message_size in total.

  For n = 8:   2 × 7 / 8 = 1.75
```

**Worked: an 8-GPU intra-node all-reduce of a 14 GB gradient buffer** (a 7B-parameter model at BF16: `7e9 × 2 B = 14 GB`).

```
  BOUND BY NVLINK (what the reference topology delivers)
    per-GPU NVLink bandwidth, per direction        = 450 GB/s
    busbw ceiling                                   = 450 GB/s
    algbw ceiling  = busbw ÷ 1.75                   = 257 GB/s
    time           = 14 GB ÷ 257 GB/s               = 54.5 ms

  BOUND BY PCIe (what you get if the GPUs are NOT NVLink-connected,
                 e.g. PCIe-form-factor H100 cards without NVLink bridges)
    per-GPU PCIe Gen5 x16, per direction            = 63 GB/s theoretical
                                                      ~54 GB/s usable (lesson 03)
    algbw ceiling  = 54 ÷ 1.75                      = 30.9 GB/s
    time           = 14 GB ÷ 30.9 GB/s              = 453 ms

    → 8.3× slower. And worse in practice: on a PCIe topology the ring
      has to traverse the root complex and possibly UPI, so the effective
      per-GPU figure is lower than 54 GB/s.

  BOUND BY THE NETWORK (the same all-reduce across 16 nodes, 128 GPUs)
    per-GPU rail NIC                                = 400 Gb/s = 50 GB/s
    n = 128:  2(n−1)/n = 2 × 127/128                = 1.984
    algbw ceiling  = 50 ÷ 1.984                     = 25.2 GB/s
    time           = 14 GB ÷ 25.2 GB/s              = 555 ms

    → the NETWORK is the bound at scale, by roughly 10× over NVLink.
      This is why hierarchical algorithms exist: reduce within the node
      on NVLink first (54.5 ms), then all-reduce the reduced result
      across nodes (14 GB ÷ 25.2 = 555 ms, but now only 1/8 of the
      per-GPU traffic crosses the wire, so ~69 ms), then broadcast back
      on NVLink. NCCL does this automatically as its "tree" algorithm.
```

**The decision rule that falls out:** compare the per-GPU ceiling of each candidate link, divide each by the collective's `2(n−1)/n` factor for the relevant *n*, and the smallest algbw wins. If NVLink and the network are within 2× of each other you are near a regime change and hierarchical algorithms will dominate; if they differ by 10× (as above) the network is unambiguously the constraint and node-internal optimisation is not where your effort belongs.

**And the sanity check against reality.** Measured `nccl-tests` bus bandwidth on a healthy 8× H100 NVSwitch node typically lands in the **370–480 GB/s** range for large messages, against the 450 GB/s per-GPU theoretical — and can appear to *exceed* it on all-reduce specifically, because NVLink SHARP performs the reduction in the switch rather than moving all the data around a ring, breaking the assumption the `2(n−1)/n` formula was derived under. **A busbw above the link ceiling is evidence SHARP is engaged, not evidence of a measurement error.** A busbw at half the ceiling on a supposedly NVSwitch node is the finding worth chasing.

### 8. Where it's going: Grace-Blackwell and rack-scale NVLink

Enough to speak to the direction credibly.

The **GB200** pairs a Grace CPU with two Blackwell GPUs in a "Superchip," connected by **NVLink-C2C** at **900 GB/s bidirectional** — a *coherent* CPU↔GPU link that replaces the PCIe stub for the CPU–GPU path. That is the architectural change that matters most for this module: on a GB200, the "GPU ↔ host DRAM" row of §5's table is no longer a PCIe row, and it is no longer 63 GB/s.

The **GB200 NVL72** rack connects **36 Grace CPUs and 72 Blackwell GPUs** into a **single NVLink domain** via **nine NVLink switch trays**, delivering **130 TB/s** of aggregate low-latency GPU communication and **1 800 GB/s** bidirectional per GPU. "All-to-all NVLink" now spans a whole rack rather than eight GPUs in a chassis.

**Do not model this as "a bigger HGX box."** Three things change qualitatively:

1. The NVLink domain crosses chassis boundaries, so the fabric has cabling, optics and failure domains that a baseboard does not.
2. The CPU joins the coherent domain over NVLink-C2C, which changes the host-feed path this whole module has been analysing.
3. Collective arithmetic changes regime: with 72 GPUs in one NVLink domain, jobs that previously had to cross the InfiniBand fabric now stay on NVLink, moving the bound from ~25 GB/s per GPU to ~900 GB/s per GPU.

The *operational lens* is unchanged — trained links, socket and rail placement, GPU–GPU on the fast fabric and everything else on PCIe — but the numbers and the domain boundary both move.

## Perspectives

**Developer.** NCCL auto-detects the NVLink/PCIe/IB paths at initialisation, which hides almost all of this from application code. But "auto-detects" is not "always correct": it builds its graph from sysfs and can disagree with `nvidia-smi topo -m` (issue #246), and its decisions are visible only through `NCCL_DEBUG=INFO`. A developer debugging a mysteriously slow all-reduce should know to dump NCCL's topology output and check for `via P2P` on GPU-to-GPU channels and `GPU Direct RDMA Enabled` on the NIC lines — the same cross-tool reconciliation habit lessons 01, 03 and 08 all build.

**Operator.** The reference topology is the acceptance-test baseline for any rented or leased 8-GPU box. The operator's job is proving a specific SKU matches the reference — socket split, NVLink domain shape, per-GPU rail alignment, PCIe switch grouping — not re-deriving the reference on every new node. Capture `nvidia-smi topo -m`, `lspci -tv` and `lstopo --output-format xml` at bring-up and diff against a known-good node of the same SKU; a structural diff is a five-second read and catches everything in this lesson.

**Hardware.** Two design decisions on this board are worth being able to explain. **Retimers instead of a baseboard PCIe switch** is a direct consequence of lesson 03's signal-integrity arithmetic: the baseboard-to-tray channel needs re-timing at 32 GT/s, and a switch would solve a bandwidth problem the board does not have while adding contention on a path that carries no GPU–GPU traffic. **Four NVSwitches instead of six** is the fabric-consolidation story: wider switch silicon (64 ports vs 36) does the same job with fewer parts and leaves surplus ports for rack-scale extension.

**Economics / scale.** Rail alignment is what makes *multi-node* training economical. A single node's perfect internal topology is necessary but not sufficient — the rail design across the fleet is what lets a job scale past one chassis without the per-node correctness going to waste. And the collective arithmetic in §7 is the honest framing for where optimisation effort belongs: when the network bound is 10× tighter than the NVLink bound, a week spent on intra-node placement buys nothing, and a week spent on rail cabling buys everything. Being able to say *which* is which, with the arithmetic, is the difference between an engineer with an opinion and one with an argument.

## Real-world use cases

- **NVIDIA — HGX H100 platform documentation** ([developer.nvidia.com](https://developer.nvidia.com/blog/introducing-nvidia-hgx-h100-an-accelerated-server-platform-for-ai-and-high-performance-computing)). The substance: the HGX H100 8-GPU baseboard hosts eight H100 GPUs and **four third-generation NVSwitch** chips, with each GPU connected to *all four* switches over its fourth-generation NVLink ports. Each NVSwitch has **64 NVLink-4 ports** and supports **SHARP** in-network reduction with multicast, previously an InfiniBand-only feature; NVIDIA reports **2× all-reduce throughput** within an 8-GPU H100/H200 server versus the A100 generation. Bisection bandwidth of the 8-GPU domain is **3.6 TB/s**. What it shows: the authoritative numbers behind §§1–3, and the reason a measured busbw above the naive ring ceiling is a healthy signal rather than an error.

- **NVIDIA — DGX H100 system specifications** ([nvidia.com](https://www.nvidia.com/en-us/data-center/dgx-h100/)). The substance: 8× H100 SXM (640 GB total HBM3), **4× NVSwitch delivering 7.2 TB/s of bidirectional GPU-to-GPU bandwidth**, **dual Intel Xeon Platinum 8480C (112 cores total)**, **2 TB** of system memory, **4× OSFP cages serving 8× single-port ConnectX-7** at up to 400 Gb/s for the compute fabric plus **2× dual-port QSFP112** adapters for storage and in-band management, **2× 1.92 TB NVMe M.2** for the OS and **8× 3.84 TB NVMe U.2** for data cache, and **10.2 kW** maximum power. What it shows: the concrete instantiation of the reference, and the dual-fabric pattern (compute NICs separate from storage/management NICs) that §5's traffic table implies.

- **CoreWeave — H100 large-scale benchmark results** ([coreweave.com](https://www.coreweave.com/blog/nvidia-h100-gpu-benchmark-results-what-we-learned-from-large-scale-gpu-testing)). The substance: CoreWeave reports **49.2% Model FLOPs Utilisation on a 128-H100 run, 18 points above a comparison baseline**, and describes a dual-fabric architecture — NVIDIA Quantum InfiniBand for collectives, a separate DPU-offloaded Ethernet fabric for storage — explicitly so that checkpoint and data-loading I/O never contends on the wire with all-reduce traffic. They also describe topology-aware scheduling with health probes evicting failing nodes before they affect a job. What it shows: MFU is the aggregate metric a systemic topology defect actually moves, because unlike per-GPU utilisation it cannot be fooled by a stalled GPU that still shows busy SMs. A rail-misalignment bug affecting one GPU in eight shows up as multiple points of MFU across a whole run.

- **NADDOD — 8-card A100/A800 and H100 host configuration deep dive** ([naddod.com](https://www.naddod.com/blog/high-performance-gpu-server-hardware-topology-and-cluster-networking-2)). The substance: an independent practitioner walkthrough of the 8-GPU baseboard, the NVSwitch fabric, the eight PCIe Gen5 x16 host stubs, and the PCIe switch chips on the server head motherboard from which both the ConnectX HCA and the NVMe U.2 signals originate. What it shows: independent confirmation of §4's claim about *where* the PCIe switches live — the single most commonly mis-stated fact about these systems, and the one that determines whether `PIX` between a GPU and its NIC is achievable.

## Worked example

**Goal: map a rented "8× H100 SXM" instance onto the reference, verify rail alignment, and price any divergence.**

**Step 1 — pull the matrix.**

```
$ nvidia-smi topo -m
        GPU0  GPU1  GPU2  GPU3  GPU4  GPU5  GPU6  GPU7  NIC0  NIC1  NIC2  NIC3  NIC4  NIC5  NIC6  NIC7  CPU Affinity     NUMA
GPU0     X    NV18  NV18  NV18  NV18  NV18  NV18  NV18  PIX   NODE  NODE  NODE  SYS   SYS   SYS   SYS   0-27,112-139      0
GPU1    NV18   X    NV18  NV18  NV18  NV18  NV18  NV18  NODE  PIX   NODE  NODE  SYS   SYS   SYS   SYS   0-27,112-139      0
GPU2    NV18  NV18   X    NV18  NV18  NV18  NV18  NV18  NODE  NODE  PIX   NODE  SYS   SYS   SYS   SYS   28-55,140-167     1
GPU3    NV18  NV18  NV18   X    NV18  NV18  NV18  NV18  NODE  NODE  NODE  PIX   SYS   SYS   SYS   SYS   28-55,140-167     1
GPU4    NV18  NV18  NV18  NV18   X    NV18  NV18  NV18  SYS   SYS   SYS   SYS   PIX   NODE  NODE  NODE  56-83,168-195     2
GPU5    NV18  NV18  NV18  NV18  NV18   X    NV18  NV18  SYS   SYS   SYS   SYS   NODE  SYS   NODE  NODE  56-83,168-195     2
GPU6    NV18  NV18  NV18  NV18  NV18  NV18   X    NV18  SYS   SYS   SYS   SYS   NODE  SYS   PIX   NODE  84-111,196-223    3
GPU7    NV18  NV18  NV18  NV18  NV18  NV18  NV18   X    SYS   SYS   SYS   SYS   NODE  SYS   NODE  PIX   84-111,196-223    3
NIC5    SYS   SYS   SYS   SYS   SYS   SYS   SYS   SYS   PIX   NODE  NODE  NODE   SYS   X    SYS   SYS

Legend:
  X   = Self
  SYS = Connection traversing PCIe as well as the SMP interconnect between NUMA nodes (e.g., QPI/UPI)
  NODE= Connection traversing PCIe as well as the interconnect between PCIe Host Bridges within a NUMA node
  PHB = Connection traversing PCIe as well as a PCIe Host Bridge (typically the CPU)
  PXB = Connection traversing multiple PCIe bridges (without traversing the PCIe Host Bridge)
  PIX = Connection traversing at most a single PCIe bridge
  NV# = Connection traversing a bonded set of # NVLinks
```

*(Representative transcript; capture your own.)*

**Step 2 — check the three reference invariants, in order.**

**(a) GPU×GPU block is uniformly `NV18`.** ✓ Every pair, 18 bonded NVLink-4 links, through the four NVSwitches. That is the signature of an NVSwitch baseboard: no pair is privileged, none is on PCIe. A mix of `NV18` and lower counts would mean a direct-attach (bridge-connected) board; any `SYS` in that block would mean the NVLink fabric is down and traffic has fallen back to PCIe — a catastrophic, and very visible, defect.

**(b) Socket split is 4/4.** ✓ NUMA affinities read 0,0,1,1,2,2,3,3 over four SNC nodes on two sockets: GPU0–3 on socket 0, GPU4–7 on socket 1. Matches the reference. If a scheduler pins a GPU5 job to cores 0–27, every host copy crosses UPI — the placement bug lesson 02 quantified at −41%.

**(c) Each GPU has exactly one `PIX` NIC.** ✗ **GPU5 does not.**

```
  GPU0 → NIC0  PIX   ✓
  GPU1 → NIC1  PIX   ✓
  GPU2 → NIC2  PIX   ✓
  GPU3 → NIC3  PIX   ✓
  GPU4 → NIC4  PIX   ✓
  GPU5 → NIC5  SYS   ✗   ← should be PIX
  GPU6 → NIC6  PIX   ✓
  GPU7 → NIC7  PIX   ✓
```

And the NIC5 row confirms it from the other side: NIC5's only `PIX` partner is **GPU0**, on socket 0. **NIC5 is physically in a socket-0 slot.**

**Step 3 — confirm from the hierarchy, not the label** (lesson 03's tiebreaker discipline).

```
$ readlink -f /sys/class/infiniband/mlx5_5/device
/sys/devices/pci0000:00/0000:00:01.1/0000:01:00.0/0000:02:08.0/0000:05:00.0

$ readlink -f /sys/bus/pci/devices/0000:c3:00.0            # GPU5
/sys/devices/pci0000:c0/0000:c0:01.1/0000:c1:00.0/0000:c2:04.0/0000:c3:00.0
```

Shared path prefix: **none beyond the machine root.** NIC5 descends from host bridge `pci0000:00` (socket 0, buses below 0x80); GPU5 from `pci0000:c0` (socket 1). Their lowest common ancestor is the machine. The `SYS` label is correct and the cause is physical placement, not a firmware mislabel or a tool disagreement.

**Step 4 — verify the links themselves are healthy** (lesson 03), so you are not reporting two faults as one.

```
$ nvidia-smi --query-gpu=index,pcie.link.gen.max,pcie.link.gen.current,pcie.link.width.max,pcie.link.width.current --format=csv,noheader
0, 5, 5, 16, 16
1, 5, 5, 16, 16
2, 5, 5, 16, 16
3, 5, 5, 16, 16
4, 5, 5, 16, 16
5, 5, 5, 16, 16
6, 5, 5, 16, 16
7, 5, 5, 16, 16
```

All eight at Gen5 x16. The links are fine; the *placement* is not. That distinction matters because they go to different owners: a degraded link is a hardware repair ticket, a misplaced NIC is a build-spec or cabling ticket.

**Step 5 — confirm NCCL's own view agrees, and see what it does about it.**

```
$ NCCL_DEBUG=INFO ./all_reduce_perf -b 1G -e 1G -g 8 2>&1 | grep -E 'GPU Direct RDMA|PXN|NET/IB'
node:9182:9182 [0] NCCL INFO NET/IB : Using [0]mlx5_0:1/IB [1]mlx5_1:1/IB [2]mlx5_2:1/IB ...
node:9182:9182 [0] NCCL INFO NET/IB: GPU Direct RDMA Enabled for HCA 0 'mlx5_0'
node:9182:9182 [5] NCCL INFO NET/IB: GPU Direct RDMA Disabled for HCA 5 'mlx5_5'   ← GPU5's NIC
node:9182:9182 [5] NCCL INFO PXN: using GPU 4 as intermediate for HCA 5
```

NCCL independently reached the same conclusion, disabled GPUDirect RDMA for that pairing, and engaged **PXN** — routing GPU5's network traffic over NVLink to GPU4 and out through GPU4's local path. That is the mitigation from §6, applied automatically. It recovers most of the bandwidth; it does not restore rail locality on the fabric side.

**Step 6 — price the divergence.**

```
  DESIGN INTENT for GPU5's inter-node path:
     GPUDirect RDMA, GPU5 → PCIe switch 3 → NIC5 → rail-5 leaf switch
     ceiling = min(PCIe Gen5 x16 63 GB/s, ConnectX-7 400 Gb/s = 50 GB/s)
             = 50 GB/s

  WITHOUT PXN (cross-socket, host-staged):
     bounded by one UPI 2.0 link  ≈ 38 GB/s effective, shared
     observed under concurrent load ≈ 20-25 GB/s
     → 50 → ~22 GB/s = 56% loss on GPU5's inter-node path

  WITH PXN (NVLink hop to GPU4, then GPU4's local NIC):
     NVLink hop is effectively free (450 GB/s ≫ 50 GB/s)
     but GPU4's NIC now carries BOTH GPU4's and GPU5's traffic
     → both share 50 GB/s = 25 GB/s each
     → 50 → 25 GB/s = 50% loss, now spread across TWO GPUs

  BLAST RADIUS: in a synchronous data-parallel job the slowest rank sets the
  step time. A 50% loss on 2 of 8 ranks' network path is a whole-job defect,
  not a 25% one.

  FLEET FRAMING: expressed the way CoreWeave expresses it, this is the class
  of defect that moves MFU by multiple points across an entire run —
  comparable in scale to the 18-point spread they published between a
  well-tuned and a less-tuned 128-H100 configuration. State it as
  "X points of MFU × Y GPU-hours × $Z/GPU-hr over a W-week run", not as
  an abstract percentage.
```

**Outcome:** a reference-versus-real table for all eight GPUs and their NICs, one flagged divergence confirmed three independent ways (`topo -m` label, sysfs path prefix, NCCL's own decision), the link health separately verified so the two fault classes are not conflated, and the cost stated in the metric a fleet operator already tracks.

## Practice

Feeds the **Topology Teardown** deliverable.

1. **Draw the reference from memory first, before touching a machine.** Two sockets with the 4/4 GPU split; four NVSwitches with all-to-all NVLink at 18 links × 25 GB/s per direction = 450 GB/s per GPU; eight independent Gen5 x16 host stubs carried over retimers with **no PCIe switch on the GPU baseboard**; four PCIe Gen5 switches on the CPU tray, each fanning out to two GPUs, two ConnectX-7 NICs and NVMe; eight rail-aligned 400 Gb/s NICs plus a separate storage/management fabric. Save the sketch. Only then compare it against the references below and correct the gaps — the gaps are the lesson.

2. **Capture the real node.**
   ```
   nvidia-smi topo -m                                       > topo.txt
   nvidia-smi topo -p2p rw                                  > p2p.txt
   lspci -tv                                                > tree.txt
   lstopo --output-format console --no-caches --whole-io    > lstopo.txt
   nvidia-smi --query-gpu=index,pci.bus_id,pcie.link.gen.max,pcie.link.gen.current,pcie.link.width.max,pcie.link.width.current --format=csv > links.csv
   nvidia-smi nvlink -s                                     > nvlink.txt
   ```

3. **Build the reference-vs-real mapping table.** Columns: `GPU | BDF | expected socket/NUMA | observed CPU affinity + NUMA | expected rail NIC | observed PIX/PXB NIC | topo label | aligned? (Y/N)`. Verify all three invariants from the worked example: the GPU×GPU block is uniformly `NV18`; the socket split matches; each GPU has exactly one `PIX`/`PXB` NIC.

4. **Confirm one pairing from the hierarchy, not the label.** Pick one GPU–NIC pair, `readlink -f` both sysfs device paths, and show that the shared path prefix matches the code `topo -m` gave them. Do the same for one pair that reads `SYS`, to show the contrast.

5. **Verify the NVLink fabric itself, not just the labels.** `nvidia-smi nvlink -s` reports per-link state and speed for every NVLink on every GPU. Count them: an H100 should report 18 active links per GPU. A GPU with 16 active links is running a degraded fabric — an `NV18` label in `topo -m` is a topology classification, not a health check.

6. **Do the collective arithmetic for your node.** Using §7's formula, compute the algbw ceiling for an 8-GPU all-reduce over NVLink and over PCIe. Then run `nccl-tests`' `all_reduce_perf -b 8 -e 8G -f 2 -g 8` and compare the measured busbw against your NVLink ceiling. State whether the result is consistent with SHARP being engaged.

7. **Flag every divergence with its throughput consequence**, and where possible translate it into MFU-style fleet framing.

**Acceptance:** a committed reference-vs-real mapping in the deliverable — your from-memory HGX H100 diagram, plus a table binding all 8 real GPUs and their NICs to the reference, with socket split, GPU–GPU-on-NVLink, per-GPU rail alignment, per-GPU PCIe link health, and per-GPU NVLink count each explicitly confirmed or flagged. Plus your computed collective ceilings and a measured `nccl-tests` busbw to compare against. A fully-aligned node is a valid result; the mapping must *prove* it with the labels, the affinities and the numbers, not assert it.

## Common pitfalls

1. **Comparing NVIDIA's bidirectional NVLink figure against PCIe's per-direction figure.** *Symptom:* claiming NVLink is 14× PCIe. *Mechanism:* NVIDIA markets 900 GB/s for H100, which is `18 links × 50 GB/s bidirectional`. PCIe is universally quoted per direction. Compare like with like: `450 ÷ 63 = 7.1×`. This is the single most common arithmetic error in the subject.

2. **Believing there are no PCIe switches anywhere.** *Symptom:* being unable to explain how a GPU and its NIC can read `PIX`. *Mechanism:* the *GPU baseboard* has no PCIe switch — that is the correct and load-bearing claim, and it is why each GPU has an independent Gen5 x16 stub over retimers. The switches are on the **CPU/motherboard tray**, where they fan out root-port lanes to GPU stubs, ConnectX NICs and NVMe. Without them, every GPU–NIC pair would read `PHB` or worse and GPUDirect RDMA would have no fast path at all.

3. **Believing NVSwitch means "no bottleneck, ever."** *Symptom:* dismissing a slow collective because "it's on NVLink." *Mechanism:* non-blocking is a property of the fabric's topology within a domain — no cut is worse than the bisection. It is not a promise about every traffic pattern, it says nothing about the domain *boundary*, and it does not cover the PCIe and network hops that a multi-node collective still traverses.

4. **Conflating within-node rail alignment with the fabric's rail-optimised design.** *Symptom:* fixing NIC placement on one node and seeing no improvement. *Mechanism:* they are two separate requirements. Within-node alignment is a PCIe-placement fact about one server; the fabric's rail design is a switch-and-cabling fact about the whole fleet. Getting one right without the other still loses the throughput, because a collective's time is set by its worst node pair.

5. **Assuming every vendor's "8× H100" box is topologically identical.** *Symptom:* a runbook that works on one SKU and silently misplaces work on another. *Mechanism:* the HGX baseboard is common, but host CPU choice (Intel vs AMD, and therefore SNC vs NPS and different lane budgets), NIC packaging (Cedar mezzanine vs discrete cards), NIC count, PCIe switch grouping and storage design all vary. Always verify a new SKU against the reference; never assume the 4/4 split or the NIC pairing from memory alone.

6. **Reading `NV18` as a health check.** *Symptom:* "topo -m says NV18, the fabric is fine," while a GPU runs with two dead links. *Mechanism:* `NV#` in `topo -m` classifies the *connection type* and reports the bonded link count as the driver understands it; `nvidia-smi nvlink -s` reports per-link *state*. Check both.

7. **Treating GB200 NVL72 as "a bigger HGX box."** *Symptom:* applying 8-GPU intuitions to rack-scale planning. *Mechanism:* the NVLink domain now crosses chassis (72 GPUs, nine switch trays, cabling and optics), the CPU joins the coherent domain over NVLink-C2C at 900 GB/s replacing the PCIe host stub, and collectives that used to be network-bound become NVLink-bound. The operational lens carries over; the numbers and the domain boundary do not.

## Self-check

- **Which GPUs share socket 0 on a standard HGX H100 node, and how do you verify it?**
  **Answer:** **GPU0, GPU1, GPU2, GPU3.** The eight GPUs split 4/4 across the two CPU sockets — GPU0–3 to socket 0, GPU4–7 to socket 1. Verify with the `NUMA Affinity` and `CPU Affinity` columns of `nvidia-smi topo -m`, cross-checked against `/sys/bus/pci/devices/<bdf>/numa_node` and, as a fast pre-check, the bus-number range in the BDF (buses below 0x80 on socket 0, above on socket 1). The split governs GPU↔host and GPU↔NIC locality only; GPU↔GPU is unaffected because it rides NVLink. Note that on a node with SNC2 or NPS2 you will see four NUMA nodes rather than two, with two GPUs per node — group nodes by the `11` entries in the `numactl --hardware` distance matrix to recover the socket boundary.

- **Where should GPU5's training NIC attach, and what exactly breaks if it does not?**
  **Answer:** Under the **same PCIe switch as GPU5 on the CPU tray, on socket 1** — so `nvidia-smi topo -m` reads `PIX` (or `PXB`) between them. That lets GPUDirect RDMA route the transfer through the switch crossbar directly between the NIC and GPU5's HBM, never entering the root complex, at a ceiling of `min(63 GB/s PCIe, 50 GB/s ConnectX-7) = 50 GB/s`. If the NIC is in a socket-0 slot, `topo -m` reads `SYS`, cross-socket GPUDirect RDMA is not a supported fast path, and the stack stages through host DRAM: `HBM → PCIe → DRAM(node2) → UPI → DRAM(node0) → PCIe → NIC`. The ceiling collapses to one UPI link's ~38 GB/s effective, shared — typically ~20–25 GB/s under load. NCCL will detect this itself (`GPU Direct RDMA Disabled for HCA`) and may engage **PXN**, hopping over NVLink to a GPU that is local to a working NIC — which recovers most of the bandwidth by making two GPUs share one NIC, and does not restore rail locality on the fabric side.

- **Why is there no PCIe switch on the HGX baseboard, and what carries GPU–GPU traffic instead?**
  **Answer:** Because GPU–GPU traffic does not use PCIe. It rides **NVLink through four third-generation NVSwitches** as a non-blocking all-to-all fabric at **18 NVLink-4 links × 25 GB/s per direction = 450 GB/s per GPU each way** (NVIDIA's marketed 900 GB/s bidirectional), which is 7.1× a PCIe Gen5 x16 link's 63 GB/s. A PCIe switch merging the eight GPU stubs on the baseboard would add shared-bandwidth contention, latency and a failure domain on a path that carries none of that traffic. Instead each GPU gets an **independent PCIe Gen5 x16 stub to the CPU tray, carried over PCIe Gen5 retimers** because the baseboard-to-tray channel exceeds what a passive 32 GT/s channel can drive. That stub is used only for GPU↔host, GPU↔NIC and GPU↔storage — the paths that genuinely need PCIe. Note the precision: there is no switch *on the baseboard*; there are four PCIe Gen5 switches *on the CPU tray*, and those are what make `PIX` between a GPU and its NIC possible.

- **Given an 8-GPU all-reduce of a 14 GB gradient buffer, is it bounded by NVLink, PCIe, or the network — and show the arithmetic.**
  **Answer:** Use `busbw = algbw × 2(n−1)/n`; for n = 8 the factor is **1.75**. **NVLink:** ceiling 450 GB/s per direction per GPU → `algbw = 450 ÷ 1.75 = 257 GB/s` → `14 GB ÷ 257 = 54.5 ms`. **PCIe Gen5 x16** (a non-NVLink 8-GPU box): usable ~54 GB/s → `algbw = 54 ÷ 1.75 = 30.9 GB/s` → `453 ms`, **8.3× slower**. **Network**, for the same collective across 128 GPUs on 16 nodes: rail NIC 400 Gb/s = 50 GB/s, factor `2 × 127/128 = 1.984` → `algbw = 25.2 GB/s` → `555 ms` — so at scale the **network** is the bound by roughly 10× over NVLink, which is why NCCL's hierarchical tree algorithm reduces within the node on NVLink first and only sends the reduced result over the wire. Sanity check the measurement: a healthy 8× H100 node measures 370–480 GB/s busbw, and can exceed the 450 GB/s link ceiling on all-reduce specifically because NVLink **SHARP** performs the reduction in the switch rather than moving all the data around a ring.

- **What production metric shows a systemic rail-misalignment bug across a fleet, and roughly what does it do?**
  **Answer:** **MFU (Model FLOPs Utilisation)** — delivered FLOPs against theoretical peak, aggregated over the run. Per-GPU utilisation cannot show it: a GPU stalled on a starved network path still issues instructions and still reports busy SMs. MFU can, because it compares work *completed* against the hardware's capability. A systemic rail-misalignment bug shows up as a multi-point MFU drop across the run, comparable in scale to the 18-point spread CoreWeave published between a well-tuned 128-H100 configuration at 49.2% MFU and a lower baseline. The framing that lands in a review is "X points of MFU × Y GPU-hours × $Z/GPU-hr over a W-week run."

- **Name the architectural reason production 8-GPU nodes carry two separate NIC fabrics.**
  **Answer:** To keep collective/RDMA traffic and storage/management traffic off the same wires. A DGX H100 has 4× OSFP cages serving 8× single-port ConnectX-7 at 400 Gb/s for the **compute fabric** (InfiniBand or RoCE, carrying all-reduce), plus 2× dual-port QSFP112 adapters for **storage and in-band management**. CoreWeave describes the same split explicitly — Quantum InfiniBand for collectives, a separate DPU-offloaded Ethernet fabric for storage. The reason is contention: a checkpoint write or a dataloader burst on the same fabric as an all-reduce delays the collective, and because the collective is synchronous, that delay propagates to every rank in the job. Two fabrics make the two traffic classes independent.

- **`nvidia-smi topo -m` reports `NV18` between every GPU pair. What does that tell you, and what does it not?**
  **Answer:** It tells you every pair's connection is classified as a bonded set of 18 NVLinks — the signature of an NVSwitch-based baseboard where no pair is privileged and none is on PCIe. It does **not** tell you (a) that all 18 links per GPU are actually *up* — `topo -m` reports the classification, while `nvidia-smi nvlink -s` reports per-link state, and a GPU running 16 of 18 links is a degraded fabric; (b) anything about the host-feed path — the NIC columns and the NUMA affinity column are separate questions on a separate fabric; or (c) anything about the PCIe links' trained speed and width, which you get from `nvidia-smi --query-gpu=pcie.link.gen.current,...` or `lspci -vvv`. A perfect `NV18` block coexists happily with a NIC on the wrong socket and a GPU stuck at Gen4.

## Connections & what's next

This lesson is the reference picture the rest of the module measures against. **Lesson 01** gave you the tree and the `topo -m` codes; **lesson 02** gave you the memory-tier bandwidths this lesson's ceilings sit on top of; **lesson 03** gave you the single-link vocabulary and the ability to derive usable GB/s, which this lesson applied at node scale.

**Lesson 05** takes the socket/NUMA split established here and asks how to make Kubernetes *guarantee* — not merely hope for — GPU–CPU–memory alignment on a scheduler that is otherwise topology-blind, including why alignment silently fails when a device plugin omits `TopologyInfo`. **Lesson 06** reuses the same PCIe-locality reasoning for NVMe placement and GPUDirect Storage, where the drives sit on the same CPU-tray switches as the GPUs and the link widths are x4 rather than x16. **Lesson 07** covers the power and thermal envelope of a 10.2 kW node and what throttling does to the ceilings computed here. **Lesson 08**'s capstone is where you reconcile all four tools — `lstopo`, `lspci -tv`, `nvidia-smi topo -m`, `numactl --hardware` — against a real node *and* against this lesson's reference diagram simultaneously.

Next: **lesson 05** moves from "can I read and draw the topology" to "can I make Kubernetes respect it" — Topology Manager, CPU Manager and Memory Manager, and the difference between what each policy *guarantees* and what it merely *attempts*.

## References & further reading

**Primary sources**

- **NVIDIA — "Introducing NVIDIA HGX H100"** — [developer.nvidia.com](https://developer.nvidia.com/blog/introducing-nvidia-hgx-h100-an-accelerated-server-platform-for-ai-and-high-performance-computing) — authoritative for the four third-generation NVSwitches, each GPU connecting to all four, the 64-port NVSwitch, SHARP in-network reduction with 2× all-reduce throughput versus A100, the 3.6 TB/s bisection figure, and PCIe Gen5 host connectivity.
- **NVIDIA — DGX H100 system page and datasheet** — [nvidia.com](https://www.nvidia.com/en-us/data-center/dgx-h100/) — the concrete instantiation: 8× H100 SXM (640 GB HBM3), 4× NVSwitch at 7.2 TB/s bidirectional GPU-to-GPU, dual Xeon Platinum 8480C (112 cores), 2 TB system memory, 4× OSFP serving 8× ConnectX-7 at 400 Gb/s plus 2× dual-port QSFP112, 2× 1.92 TB NVMe M.2 + 8× 3.84 TB NVMe U.2, 10.2 kW.
- **NVIDIA — H100 Tensor Core GPU Architecture whitepaper** — the NVLink-4 details (18 links, 25 GB/s per direction each), HBM3 configuration, and the NVLink Network extension to a 256-GPU domain.
- **NVIDIA — GB200 NVL72** — [nvidia.com](https://www.nvidia.com/en-us/data-center/gb200-nvl72/) — 36 Grace CPUs and 72 Blackwell GPUs in one NVLink domain, nine NVLink switch trays, 130 TB/s aggregate, 1 800 GB/s bidirectional per GPU, NVLink-C2C at 900 GB/s between Grace and Blackwell.
- **NVIDIA — NCCL documentation and `nccl-tests` PERFORMANCE.md** — [github.com/NVIDIA/nccl-tests](https://github.com/NVIDIA/nccl-tests/blob/master/doc/PERFORMANCE.md) — the `busbw = algbw × 2(n−1)/n` derivation used in §7, and why bus bandwidth is the figure to compare against hardware peak. NCCL's env-var docs cover `NCCL_DEBUG`, `NCCL_PXN_DISABLE` and `NCCL_P2P_PXN_LEVEL`.
- **NVIDIA — DGX SuperPOD reference architecture (DGX H100)** — [docs.nvidia.com](https://docs.nvidia.com/dgx-superpod-reference-architecture-dgx-h100.pdf) — the rail-aligned cabling scheme at cluster scale: which NIC goes to which leaf switch, and why consistency across nodes is a hard requirement.

**Real-world engineering**

- **CoreWeave — "NVIDIA H100 GPU Benchmark Results"** — [coreweave.com](https://www.coreweave.com/blog/nvidia-h100-gpu-benchmark-results-what-we-learned-from-large-scale-gpu-testing) — 49.2% MFU on 128 H100s (18 points above a comparison baseline), the dual-fabric compute/storage architecture, and topology-aware scheduling with health-probe eviction.
- **NADDOD — "High-Performance GPU Server Hardware Topology and Cluster Networking-2"** — [naddod.com](https://www.naddod.com/blog/high-performance-gpu-server-hardware-topology-and-cluster-networking-2) — an independent practitioner walkthrough confirming that the PCIe switch chips are integrated on the server head motherboard and that both the ConnectX HCA and NVMe U.2 signals originate there.
- **NVIDIA/nccl issue #246** — [github.com/NVIDIA/nccl/issues/246](https://github.com/NVIDIA/nccl/issues/246) — NCCL's topology graph and `nvidia-smi topo -m` disagreeing on GPU–NIC classification for the same node; the case for reconciling rather than trusting one tool.
- **Frank Denneman — "Understanding Multi-GPU Topologies Within a Single Host"** and **"Topology-Aware Multi-GPU VM Placement"** — [frankdenneman.ai](https://frankdenneman.ai/2026-03-27-Understanding-Multi-GPU-Topologies-Within-a-Single-Host/) — the same 8-GPU reality from a placement-first and then a virtualisation angle: what happens to this topology once it is carved up for VMs rather than run bare-metal.

**Deeper dives**

- **NVIDIA — "NVLink-Network Switch" (Hot Chips 34, 2022)** — the NVSwitch design talk: port counts, switch throughput per generation, and the SHARP reduction engine's implementation. The source behind §3's generation table.
- **NVIDIA — HGX A100 platform documentation** — [developer.nvidia.com](https://developer.nvidia.com/blog/introducing-hgx-a100-most-powerful-accelerated-server-platform-for-ai-hpc) — the six-NVSwitch, 12-NVLink-per-GPU, 600 GB/s previous generation, useful for seeing what changed and why the switch count fell.
