---
lesson: "09.1"
title: "From intra-node to inter-node: extending the topology matrix past the NIC"
module: "09"
concept: "From intra-node to inter-node: extending the topology matrix past the NIC"
status: not-started
est_time: "7h"
prev: null
next: "02-inter-node-fabric.md"
artifacts: []
sources: 13
---

# 09.1 · From intra-node to inter-node: extending the topology matrix past the NIC

> **Concept.** `nvidia-smi topo -m` stops at the PCIe edge of one box; the fabric picks up at the NIC — learn to read the full `GPU → NIC → leaf → spine` path, name the *rail* each GPU rides, and turn 02b's "same root complex" rule into "same rail" across nodes.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Where this fits

02b ended with you fluent in one box. Lesson 02b.3 taught you to derive **63.0 GB/s per direction** for a PCIe Gen5 x16 link from 32 GT/s and 128b/130b encoding, and to expect about **54 GB/s** of it after TLP framing, DLLP acknowledgements and DMA overhead. Lesson 02b.4 gave you the second fabric on the same board: eight H100s, four third-generation NVSwitches, **18 NVLink-4 links per GPU at 25 GB/s per direction = 450 GB/s per direction per GPU**, an aggregate of 7.2 TB/s bidirectional and a bisection of 3.6 TB/s inside one chassis. You can read `PIX`/`PXB`/`PHB`/`NODE`/`SYS` out of `nvidia-smi topo -m` and say which GPU can DMA to which NIC without a CPU bounce.

All of that describes a single chassis, and it goes silent the moment a byte has to reach a GPU in a different server. This lesson walks you across that boundary. It puts a number on **the bandwidth cliff at the NIC** — the single fact the rest of module 09 exists to manage — names the *rail* (the inter-node generalisation of "same root complex"), traces the exact hop sequence from GPU HBM to a leaf switch with a latency budget attached, and derives the all-reduce time model for a job spanning *M* nodes so you can see, in arithmetic rather than in adjectives, which term dominates. Nothing here works without 02b; nothing in 09.2 works without this lesson.

## Why this matters

Every placement and procurement argument you make above eight GPUs lives or dies on one question the topology matrix cannot answer alone: *when a collective leaves the box, does it stay on its rail, or does it get funnelled the wrong way?* Get the GPU-to-NIC mapping wrong and GPUDirect RDMA silently falls back to a bounce through host memory — you keep the GPUs, lose the wire, and eat a step-time regression that no dashboard names, because `nvidia-smi` still reports high utilisation while the SMs stall on a starved collective.

The cliff is why this is not a rounding error. Inside the box a GPU has 450 GB/s per direction to its peers. Out the NIC it has 50 GB/s. **A byte that crosses the chassis boundary costs about nine times as much bandwidth as one that does not**, and every design decision in this module — rail cabling, oversubscription ratios, in-network reduction, job placement — is an attempt to keep bytes on the cheap side of that ratio or to make the expensive side hurt less.

In an interview for the platform track at a GPU neocloud, being able to draw the `GPU → NIC → leaf → spine` path, put a bandwidth number on each hop, and say exactly where NVLink stops — and where, on the newest hardware, it does *not* stop — is the line between "I run Kubernetes near GPUs" and "I can argue fabric with the network team."

## What's new here (calibration)

- **Already yours (02b, 08, 06):** the PCIe tree and its bandwidth arithmetic, NVLink/NVSwitch as a board-level fabric, rail alignment as an intra-node property, the GPUDirect same-root-complex rule, reading `topo -m`'s connectivity codes, ring/tree all-reduce as the traffic pattern, and topology-aware gang placement. We reference these; we do not re-teach them.
- **Genuinely new:** the *inter-node* rail — a vertical slice of GPU-N-to-leaf-N cabling spanning every node in a cluster, not a single board; the hop-by-hop bandwidth ladder from HBM to the wire and which term binds; the latency budget of one inter-node hop; NCCL's own path-type vocabulary, which is **longer than `nvidia-smi`'s five codes** and includes `PXN`, the mechanism that rescues a misaligned rail; the real NIC-per-GPU provisioning ratio vendors ship (it is not the clean 1:1 you would guess); the arithmetic that shows the intra-node and inter-node phases of a hierarchical all-reduce are, on H100 + ConnectX-7, roughly *balanced*; and the one genuine exception to "NVLink never leaves the chassis" — the GB200/GB300 NVL72 rack-scale NVLink domain.
- **Deliberately deferred:** the shape and cost of the network *above* the leaf (Clos tiers, bisection, oversubscription) is 09.2's job. How RDMA actually moves the bytes is 09.3. Why the fabric doesn't drop them is 09.4. Here you only need to know where the fabric begins, which NIC is the correct on-ramp, and what it costs to use it.

## Core concepts

### 1. The bandwidth cliff: the module's whole premise, in one diagram

Draw the path a single gradient byte takes from GPU0 on node A to GPU0 on node B, and label every hop with the bandwidth available to *one* GPU in *one* direction. The units discipline matters here and is the source of most bad interview answers: NVLink and PCIe are quoted in **GB/s** (bytes), NICs and switches in **Gb/s** (bits). Divide by 8 to compare. `400 Gb/s ÷ 8 = 50 GB/s`.

```
  ── NODE A (8× H100 SXM, HGX baseboard) ─────────────────────┐
                                                              │
   GPU0 HBM3                                                  │
     │                                                        │
     │  (a) NVLink 4: 18 links × 25 GB/s/dir                   │
     │      = 450 GB/s per direction, per GPU                  │
     ▼      non-blocking to the other 7 GPUs via 4× NVSwitch   │
   [ NVSwitch fabric ]  ── aggregate 7.2 TB/s bidirectional ───┤
     │                                                        │
     │  (b) PCIe Gen5 x16 stub off the baseboard               │
     │      63.0 GB/s/dir theoretical, ~54 GB/s usable         │
     ▼      (02b.3: 128b/130b, TLP framing, DLLP, DMA)         │
   [ PCIe switch on the CPU tray ]                             │
     │      GPU0 and mlx5_0 are SIBLINGS under this switch     │
     │      → the crossbar routes GPUDirect RDMA without ever  │
     │        reaching the root complex   (this is `PIX`)      │
     ▼                                                        │
   [ ConnectX-7 HCA  "mlx5_0" ]                                │
     │  (c) 400 Gb/s NDR = 50.0 GB/s per direction             │
     │      measured ceiling ~46-48 GB/s after protocol        │
  ───┼──────────────────────────────────────────────────────── ┘
     │  (d) OSFP cable — THE FABRIC BEGINS HERE
     ▼
   [ LEAF / rail switch 0 ]      ~100-130 ns per IB switch hop
     │  (e) uplinks to spine — 09.2's territory
     ▼
   [ SPINE ] ─► [ LEAF 0 (other pod) ] ─► NIC ─► PCIe ─► GPU0 (node B)


   THE LADDER, per GPU per direction:
       (a) NVLink 4            450 GB/s     ┐
       (b) PCIe Gen5 x16        63 GB/s     │  each hop is a ceiling;
           …usable              ~54 GB/s    │  the transfer runs at
       (c) ConnectX-7 NDR        50 GB/s    ┘  min(a, b, c) = 50 GB/s
                                              ────────────────────────
       ratio (a) : (c)  =  9 : 1     ← the cliff
```

Three things follow immediately, and each is load-bearing for the rest of the module.

**The NIC is the binding constraint, but only just.** `min(450, 54, 50) = 50 GB/s`. On this generation the PCIe link (54 GB/s usable) sits only 8% above the NIC's 50 GB/s, which means a PCIe link that has quietly trained down — the exact failure 02b.3 taught you to detect with `LnkSta` — moves the bottleneck from the wire to the bus without changing a single fabric setting. A Gen5 x8 link (31.5 GB/s) caps a 400G NIC at 63% of line rate, and nothing in the NIC's own counters will tell you why.

**Next generation, the constraint moves.** ConnectX-8 is 800 Gb/s = **100 GB/s per direction**. A PCIe Gen5 x16 host link cannot feed it: 63 GB/s theoretical is 37% below what the NIC can put on the wire. That is precisely why ConnectX-8 is specified as a native **PCIe Gen6 x16** device (121 GB/s per direction, 02b.3's table) — the NIC generation and the PCIe generation had to move together. If you are ever handed a build sheet pairing an 800G NIC with a Gen5 host link, that is a real finding, not a nitpick: the NIC will run at roughly PCIe speed, and you will have paid for 800G optics to get about 500G of throughput.

**The cliff is the reason the rest of module 09 exists.** Nine to one is the gradient every design decision rolls downhill from. Rail-optimised cabling exists to make sure the 50 GB/s you do have is spent on a single switch hop instead of three. Oversubscription (09.2) exists because the traffic that crosses the cliff is a small, predictable fraction of total traffic. SHARP (09.5) exists to reduce how many bytes have to cross it at all.

### 2. The hop sequence, and what actually happens at each hop

"GPU to NIC to leaf" is a four-word summary of about a dozen mechanical steps. You need the mechanical version, because the failure modes live in the steps, not in the summary.

1. **The collective library decides.** NCCL has already built a topology graph of the node (it reads the same PCIe and NVLink topology `nvidia-smi topo -m` renders) and chosen, per channel, which GPU sends to which peer over which transport. For an inter-node leg it selects a **network device** — an `mlx5_*` HCA — and, if the GPU and that HCA are close enough in the PCIe tree, it enables GPUDirect RDMA for that pair.
2. **A work request is posted.** The sending process writes a work-queue entry describing "move N bytes from this registered region to that remote address" and rings a doorbell — a write to a memory-mapped NIC register. No syscall. (This is 09.3's subject; here it is one line of the trace.)
3. **The NIC DMAs the payload.** The HCA issues PCIe **memory read** requests for the source buffer. With GPUDirect RDMA the source addresses are **GPU HBM** (exposed through a BAR), so the reads are peer-to-peer transactions between two devices under the same PCIe switch and the crossbar routes them directly. Without GPUDirect — because the GPU and NIC are on different root complexes — the driver first stages the data into pinned host DRAM, adding a full GPU→DRAM copy at PCIe speed plus a DRAM→NIC read, doubling PCIe traffic and adding latency and jitter.
4. **The NIC segments to the path MTU.** The transport chops the message into packets at the negotiated MTU. InfiniBand path MTU is one of 256, 512, 1024, 2048 or 4096 bytes (`enum ibv_mtu` in `libibverbs/verbs.h`, rdma-core); RoCEv2 typically runs a 4 KB IB MTU inside jumbo Ethernet frames. Each packet gets transport headers and CRCs.
5. **Bits hit the cable.** For a 400G NDR port that is 4 lanes at 100 Gb/s effective per lane (09.4 has the full rate table). This is the first hop that is *not* inside the chassis, and it is the definitional start of "the fabric."
6. **The leaf switch forwards.** An InfiniBand switch hop is on the order of **100–130 ns** port-to-port. A modern Ethernet switch in cut-through mode is a few hundred nanoseconds; store-and-forward adds serialisation of the whole frame. Either way, a switch hop is *cheap in latency and expensive in queueing* — a busy switch's buffer occupancy, not its forwarding latency, is what shows up in your tail.
7. **The mirror image on node B**: leaf → NIC → PCIe switch → GPU HBM, with the receiving NIC issuing PCIe **memory writes** into the target GPU's BAR.

A useful sanity budget for one 400G inter-node hop, small message, GPUDirect enabled: roughly **1–2 µs** half-round-trip end to end, of which the switch hops are a few hundred nanoseconds and the rest is NIC processing plus two PCIe traversals. Compare an intra-node NVLink peer copy, which is sub-microsecond and has no packetisation at all. Treat these as order-of-magnitude anchors for the ConnectX-7/Quantum-2 generation, not as guarantees; the numbers that matter for a *collective* are bandwidth and tail stability, not the unloaded point-to-point latency.

### 3. What a rail is

An 8-GPU HGX node on current builds has **one compute NIC per GPU** — eight ConnectX-7 (400G NDR) or ConnectX-8 (800G XDR) HCAs, each paired to one GPU through the PCIe switch that GPU hangs off (02b.4: a Broadcom Gen5 switch on the CPU tray typically fans out to two GPU stubs, two ConnectX HCAs and some NVMe). A **rail** is the set of *same-indexed* GPU→NIC pairs across every node, cabled into the *same leaf switch*.

```
                 RAIL 0        RAIL 1        RAIL 2   ...   RAIL 7
              ┌──────────┐  ┌──────────┐  ┌──────────┐   ┌──────────┐
              │  LEAF 0  │  │  LEAF 1  │  │  LEAF 2  │   │  LEAF 7  │
              └────┬─────┘  └────┬─────┘  └────┬─────┘   └────┬─────┘
                   │             │             │              │
   ┌───────────────┼─────────────┼─────────────┼──────────────┼────┐
   │ NODE 0   mlx5_0          mlx5_1        mlx5_2         mlx5_7   │
   │            │               │             │              │     │
   │          GPU0            GPU1          GPU2           GPU7    │
   │            └───────────────┴──── NVSwitch ──┴──────────────┘   │
   │                     450 GB/s/dir any-to-any                   │
   └───────────────────────────────────────────────────────────────┘
   ┌───────────────┼─────────────┼─────────────┼──────────────┼────┐
   │ NODE 1   mlx5_0          mlx5_1        mlx5_2         mlx5_7   │
   │          GPU0            GPU1          GPU2           GPU7    │
   │            └───────────────┴──── NVSwitch ──┴──────────────┘   │
   └───────────────────────────────────────────────────────────────┘
   ┌───────────────┼─────────────┼─────────────┼──────────────┼────┐
   │ NODE k   …    same pattern, every node, every rail             │
   └───────────────────────────────────────────────────────────────┘

   A RAIL is a VERTICAL slice: one GPU position, one NIC per node,
   one leaf switch, spanning every node in the group.
   GPU-N on node i talks to GPU-N on node j across LEAF N — one hop.
   GPU-3 wanting GPU-5 goes SIDEWAYS over NVLink first (450 GB/s),
   then out rail 5 — never up to the spine to cross rails.
```

So a rail is a *vertical slice* of the cluster: one GPU position, one NIC per node, one leaf switch, spanning all nodes. This is **rail-optimised** cabling, and it is the inter-node generalisation of 02b's rail alignment. In 02b, "rail-aligned" meant a GPU and its NIC share a PCIe branch on the *board*. Here it means GPU-N's NIC on every node lands on the *same leaf*, so GPU-N-to-GPU-N traffic across nodes crosses exactly one switch and never touches the spine.

Two properties of that definition are worth stating explicitly because they get conflated constantly:

- **A rail is a fabric-wide equivalence class, not a piece of hardware.** "Rail 3" is the set {GPU3 on every node, its NIC on every node, leaf switch 3}. It has no existence inside a single server.
- **A rail is not a NUMA node.** They correlate strongly — on a two-socket 8-GPU box, rails 0–3 usually hang off socket 0 and rails 4–7 off socket 1 — but a NUMA node is a CPU/memory locality domain inside one server, and a rail spans the cluster. The moment a SKU's NIC-to-socket layout doesn't match its rail numbering, treating them as synonyms produces a wrong answer.

### 4. The rule promotion: "same root complex" → "same rail"

02b's law for zero-copy inside the box:

> GPUDirect RDMA needs the GPU and NIC under the **same PCIe root complex / switch** — otherwise the DMA crosses the CPU (or worse, the inter-socket link), which the matrix flags as `NODE`/`SYS`, and the copy falls back through host memory.

Its inter-node twin:

> For a collective to stay cheap across nodes, GPU-N should talk to GPU-N — **same rail** — so its bytes ride its own leaf and never climb to the spine. Cross-*rail* traffic (GPU3 on node A wanting GPU5 on node B) has two options: (a) hop laterally over NVLink to the local GPU5 first, *then* go out rail 5's NIC — cheap, because NVLink is board-abundant bandwidth; or (b) go out rail 3 and traverse the spine to reach rail 5 — expensive, and the reason you keep collectives rail-local.

Option (a) has a name and an implementation. NCCL calls it **PXN** — "PCI × NVLink." When a GPU's own rail NIC is unusable or the traffic needs to leave on a different rail, NCCL routes the data over NVLink to the GPU that *is* local to the target NIC, and sends from there. Two things make this a real mitigation rather than a consolation prize: the NVLink hop costs 450 GB/s-class bandwidth (i.e. nearly nothing relative to the 50 GB/s wire), and the intermediate GPU can **aggregate messages destined for the same remote node into one larger transfer**, which improves message rate on the NIC. In current NCCL the behaviour is controlled by `NCCL_P2P_PXN_LEVEL`, whose default is **2** — "use PXN as much as possible to maximise aggregation" — and can be switched off with `NCCL_PXN_DISABLE` (verified in `src/graph/search.cc` of NCCL v2.31.2: `NCCL_PARAM(P2pPxnLevel, "P2P_PXN_LEVEL", 2)` with the comment `0: don't use PXN for P2P, 1: use PXN if needed, 2: use PXN as much as possible to maximize aggregation`).

**PXN is a workaround for a topology defect, not a reason the defect doesn't matter.** It recovers most of the *bandwidth* by refusing to drag traffic across the inter-socket link, but it does not recover *rail locality* on the fabric side: the bytes still leave on whichever rail the intermediate GPU owns, and if that is not the rail the remote GPU is on, the fabric still has to carry the crossing.

### 5. Reading the matrix for the NIC columns — and NCCL's longer vocabulary

`nvidia-smi topo -m` prints GPU×GPU *and* GPU×NIC (`mlx5_*`) connectivity. The GPU-to-NIC cells are what you now care about. The codes, best to worst for GPUDirect:

| Code | Path GPU→NIC | GPUDirect RDMA? |
|---|---|---|
| `PIX` | single PCIe switch (bridge) | Best — clean peer-to-peer through one crossbar |
| `PXB` | multiple PCIe bridges (switch cascade) | Works; extra hops, slightly more latency |
| `PHB` | via the PCIe **Host Bridge** (CPU root port) | Marginal — the root complex is now in the path |
| `NODE` | across PCIe within one NUMA node, off the root complex | Degraded |
| `SYS` | across the inter-socket link (UPI/Infinity Fabric) | **Worst — crosses CPUs; falls back to host staging** |

The pairing you want is: each GPU's *chosen* NIC reads `PIX` (or at worst `PXB`). A GPU whose only reachable NIC reads `PHB`/`NODE`/`SYS` is mis-paired — its "GPUDirect" path is a CPU bounce wearing a costume. `SYS` in particular means the DMA would cross the socket-to-socket link: 02b.4 puts a healthy UPI link at roughly **38 GB/s** effective and **20–25 GB/s** observed under concurrent load, so a `SYS`-only GPU-NIC pair turns a 50 GB/s wire into a ~22 GB/s one — a 56% loss on that GPU's inter-node path, with no error anywhere.

**NCCL's own path vocabulary is longer than the five codes**, and knowing that is a real correction to the folk version of this topic. NCCL v2.31.2's `src/graph/topo.cc` enumerates its path types as:

```
LOC   NVL   NVB   C2C   PIX   PXB   P2C   PXN   PHB   SYS   NET   DIS
```

Reading the ones `nvidia-smi` does not print: `LOC` is "same device"; `NVL` is a direct NVLink connection and `NVB` is NVLink via an intermediate GPU (a bridge hop); `C2C` is a Grace–Blackwell chip-to-chip coherent link (the NVLink-C2C path that replaces the PCIe stub on GB200-class parts, 02b.4); `P2C` is a PCIe-to-C2C path; `PXN` is the PCI-×-NVLink route from §4; `NET` means the peers are only reachable over the network; `DIS` means disconnected. The practical consequence: **the five-code ranking you memorised is `nvidia-smi`'s rendering, not the model NCCL reasons with.** On Grace-Hopper and Grace-Blackwell systems in particular, the interesting cells are `C2C`, not `PHB`, and a ranking learned on x86 HGX boards does not transfer unmodified.

State the caveat plainly whenever you quote the table: these codes describe *that* SKU's PCIe topology and BIOS layout. A different CPU vendor, a different HGX generation, or an NVLink-connected rack like NVL72 can print a different pattern or make the ranking moot inside the NVLink domain. Read the matrix per box, not per assumption.

### 6. Per-generation numbers, and why cross-rail traffic prefers NVLink

The "hop sideways over NVLink before you hop out the NIC" preference only makes sense if you know the gap. It is generation-specific, and the figures below are the same ones 02b.4 derived, carried forward unchanged so the two modules agree:

| Platform | Link generation | Per-GPU bandwidth, **per direction** | Marketed (bidirectional) | Host/NIC path |
|---|---|---|---|---|
| H100 SXM, 8-GPU HGX | NVLink 4 — 18 links × 25 GB/s/dir | **450 GB/s** | 900 GB/s | PCIe Gen5 x16 (63 GB/s/dir) → ConnectX-7 400 Gb/s = **50 GB/s** |
| B200 / GB200 | NVLink 5 — 18 links × 50 GB/s/dir | **900 GB/s** | 1 800 GB/s | ConnectX-8 800 Gb/s = **100 GB/s**, native PCIe Gen6 x16 (121 GB/s/dir) |

**Always compare per-direction numbers.** NVIDIA markets the bidirectional figure; PCIe and NIC rates are quoted per direction. Comparing 900 GB/s against 50 GB/s gives 18:1 and overstates the real ratio by 2×. The honest H100 number is **450 : 50 = 9:1**, and the honest Blackwell number is **900 : 100 = 9:1** — the same ratio, which is not a coincidence: NVLink and the NIC roughly doubled together across the generation.

So moving a tensor from GPU3 to GPU5 *within* an H100 node over NVLink is about **9× the bandwidth** and a fraction of the latency of pushing it out rail 3's NIC, across a leaf, possibly up a spine, and back down rail 5. The fabric is precious; NVLink is abundant. Rail-aware collective software exploits exactly this: shuffle cross-rail data sideways over NVLink so every GPU lines up with its own rail, then do the inter-node leg rail-local. The NIC then only ever carries GPU-N ↔ GPU-N.

### 7. Worked math: what does the cliff cost a job spanning *M* nodes?

This is the calculation that turns the 9:1 ratio into a number you can defend in a placement argument. Take a hierarchical (rail-aware) all-reduce, which is what NCCL actually runs on a multi-node HGX cluster, and derive its time from first principles.

**Setup.** *M* nodes × 8 GPUs = *N* = 8*M* GPUs. Each GPU holds a gradient buffer of **S bytes** that must be summed across all *N* GPUs and the result returned to all of them. Bandwidths per GPU per direction: `B_nvl = 450 GB/s` (NVLink 4), `B_nic = 50 GB/s` (ConnectX-7 400G). Ignore latency terms — at gradient-bucket sizes (tens to hundreds of MB) the transfer is bandwidth-bound, and the latency term is a few microseconds against milliseconds of transfer.

**The algorithm, in three phases:**

```
  PHASE 1 — intra-node reduce-scatter, over NVLink
     8 GPUs in a node cooperatively reduce S bytes; each ends
     owning a summed shard of S/8.  Bytes each GPU must send: (7/8)·S
     time  t1 = (7/8)·S / B_nvl

  PHASE 2 — inter-node all-reduce of the shard, over the RAIL
     The M GPUs that share a rail (GPU-N of each node) ring-all-reduce
     their S/8 shards.  A ring all-reduce over M ranks moves
     2·(M-1)/M · (message size) per rank.
     Bytes each GPU must send over its NIC:  2·(M-1)/M · (S/8)
     time  t2 = 2·(M-1)/M · (S/8) / B_nic

  PHASE 3 — intra-node all-gather, over NVLink
     Mirror of phase 1.        time  t3 = (7/8)·S / B_nvl

  TOTAL   T = 2·(7/8)·S/B_nvl  +  2·(M-1)/M·(S/8)/B_nic
              └── NVLink term ──┘   └──── NIC term ────┘
```

**Plug in S = 1 GB (10⁹ bytes) and run it for three cluster sizes:**

```
  NVLink term (independent of M):
      2 × 0.875 × 1 GB / 450 GB/s = 1.75/450 = 3.89 ms

  NIC term:
      M = 8   (64 GPUs):  2×(7/8)  = 1.750 → 1.750×0.125/50 = 4.38 ms
      M = 64  (512 GPUs): 2×(63/64)= 1.969 → 1.969×0.125/50 = 4.92 ms
      M = 512 (4096 GPUs):2×(511/512)=1.996→ 1.996×0.125/50 = 4.99 ms
      M → ∞            :  → 2.000  → 2.000×0.125/50 = 5.00 ms

  TOTAL and effective all-reduce bandwidth (S / T):
      64 GPUs   : T = 3.89 + 4.38 = 8.27 ms  →  121 GB/s
      512 GPUs  : T = 3.89 + 4.92 = 8.81 ms  →  114 GB/s
      4096 GPUs : T = 3.89 + 4.99 = 8.88 ms  →  113 GB/s
      infinite  : T = 3.89 + 5.00 = 8.89 ms  →  112 GB/s
```

Read four conclusions out of that arithmetic, because each one is an argument you will need:

1. **Scaling out is nearly free — *if* the fabric holds up.** Going from 64 to 4,096 GPUs costs 7% of step time, not 64×. That is the entire reason data-parallel training scales: the ring factor `2(M−1)/M` is bounded above by 2, so the inter-node term converges rather than growing. Everything that breaks this promise in practice — oversubscription (09.2), congestion (09.4), stragglers — is a fabric property, not an algorithmic one.
2. **The two terms are nearly balanced on this hardware.** 3.89 ms of NVLink against 5.00 ms of NIC. That is not obvious: the NVLink phase moves **seven times more bytes** (7/8·S twice, versus 1/8·S twice) but over **nine times more bandwidth**, so the ratio lands near 1. The practical consequence is that on H100 + ConnectX-7 you cannot fix an all-reduce-bound job by improving only one side. On Blackwell + ConnectX-8 both terms roughly halve (900 GB/s and 100 GB/s), so the balance holds and the whole step gets about 2× faster.
3. **The hierarchy is worth about 4.5×.** Run the same all-reduce as a *flat* ring across all *N* GPUs with every hop over the NIC — i.e. what you get if the collective is not rail-aware, or if NVLink is unavailable: each GPU sends `2(N−1)/N · S ≈ 2 GB` at 50 GB/s = **40 ms**, for an effective 25 GB/s. Against the hierarchical 8.9 ms, that is **4.5× slower**. When you argue "keep this job's GPUs on nodes whose NVLink domains are intact and whose rails are aligned," this is the number behind the argument.
4. **Where the money goes.** The NIC term is `2·(S/8)/B_nic` in the limit. Doubling `B_nic` (400G → 800G) removes 2.5 ms of a 8.9 ms step: about **28%**. Doubling the NVLink domain from 8 to 72 GPUs (NVL72) changes the split entirely — the intra-domain phase absorbs 72 GPUs instead of 8, so the shard that must cross the wire is `S/72` instead of `S/8`, and the NIC term collapses by ~9×. That is the architectural argument for rack-scale NVLink stated as arithmetic rather than as marketing.

Re-run the same three formulas with your own `S`, `B_nvl`, `B_nic` and `M` and you have a defensible first-order model of any data-parallel step time on any generation.

### 8. The NVL72 exception — when NVLink *does* leave the chassis

"NVLink never leaves the chassis" is the right rule for an 8-GPU HGX box and is *wrong* if you apply it unmodified to NVIDIA's rack-scale parts. In **GB200 NVL72** (and GB300 NVL72), 72 Blackwell GPUs and 36 Grace CPUs sit in one liquid-cooled rack connected by a fifth-generation NVLink Switch fabric as **one single non-blocking NVLink domain**, quoted at **130 TB/s** of aggregate GPU-to-GPU bandwidth and **1 800 GB/s bidirectional (900 GB/s per direction) per GPU** — the same figures 02b.4 carries. Every GPU is one NVLink hop from every other GPU in the rack; there is no PCIe-switch/NIC hop *inside* that 72-GPU domain at all.

The boundary did not disappear — **it moved**. On an 8-GPU HGX box the NVLink domain's edge is the board. On NVL72 the edge is the *rack*. Nine NVLink switch trays interconnect all 18 NVLink ports on each of the 72 GPUs. Everything this lesson says about the handoff still applies; it applies **at the rack boundary** instead of the chassis boundary, and the NIC count and rail mapping are properties of the rack, not of one tray.

Two consequences worth being precise about, because they are the ones interviewers probe:

- **The collective arithmetic changes regime, not just degree.** Per §7, the shard crossing the wire shrinks from `S/8` to `S/72`. Jobs that previously had to cross the fabric for most of their traffic now stay on NVLink.
- **The rule still holds outside the domain.** An NVL72 rack still has NICs, still cables into leaf switches, and still pays the cliff for anything that leaves the rack. Rack-scale NVLink raises the floor; it does not remove the floor.

Treat this as the one load-bearing exception to memorise, not as a reason to discard the rule: 8-GPU HGX nodes remain the bulk of the installed base, and on them the fabric begins at the NIC exactly as drawn in §1.

### 9. Provisioning the rail: how many NICs per GPU, really

The clean mental model — one NIC per GPU, full stop — is close but is not what reference architectures actually ship. NVIDIA's HGX AI Factory reference build (the "**2-8-9-800**" configuration: 2 CPUs, 8 GPUs, **9 NICs**, 800 Gb/s per GPU-facing NIC) puts **nine** NICs on an 8-GPU node. Eight are the GPU-indexed rail NICs, one per GPU, carrying GPUDirect RDMA traffic for the compute fabric. The ninth is deliberately kept **off** the compute rail and dedicated to storage and cluster-management traffic.

The reason is contention, and it is worth being able to state mechanically: checkpoint writes are large, sustained, and bursty; dataset staging is continuous; control-plane traffic is small but latency-sensitive. All three would otherwise share a rail NIC with a barrier-synchronous collective, and because a collective is a barrier, *any* GPU whose NIC is momentarily busy writing a checkpoint stalls the entire step across the whole job. Isolating that traffic onto a separate NIC is not tidiness; it is removing a straggler source.

The same pattern appears in the concrete DGX H100 configuration 02b.4 documented: **eight single-port ConnectX-7 adapters at up to 400 Gb/s for the compute fabric** (packaged on two "Cedar" mezzanine modules feeding four 800G OSFP cages) **plus two dual-port QSFP112 adapters for storage and in-band management**. Different packaging, same architectural decision: compute NICs and storage/management NICs are separate populations.

Two practical rules fall out. First, **count the NIC populations separately** in any bandwidth or capex calculation — folding the management NIC into "8 × 400G of collective bandwidth" overstates what the job actually has. Second, when you read a `topo -m` and find a NIC that no GPU reaches at `PIX`, do not assume a cabling defect: check whether it is the storage NIC before you file a bug.

### 10. What the collective library prints when it makes these choices

The rail map is a static fact about the hardware; whether the software actually *used* it is a runtime fact, and it is visible in one place: NCCL's init log. Run any job with `NCCL_DEBUG=INFO` and read the network block once per new SKU. The format strings below are taken from NCCL v2.31.2's source (`src/transport/net_ib/init.cc`, `src/graph/topo.cc`, `src/transport/net.cc`), so the shapes are exact even though the values are illustrative:

```
  hostA:31842:31905 [0] NCCL INFO Using network IB
  hostA:31842:31905 [0] NCCL INFO NET/IB : Using [0]mlx5_0:1/IB [1]mlx5_1:1/IB [2]mlx5_2:1/IB
                                     [3]mlx5_3:1/IB [4]mlx5_4:1/IB [5]mlx5_5:1/IB
                                     [6]mlx5_6:1/IB [7]mlx5_7:1/IB [RO]; OOB eth0:10.0.4.19
  hostA:31842:31905 [0] NCCL INFO NET/IB : GPU Direct RDMA Enabled for HCA 0 'mlx5_0'
  hostA:31842:31905 [5] NCCL INFO NET/IB : GPU Direct RDMA Disabled for HCA 5 'mlx5_5'
  hostA:31842:31905 [0] NCCL INFO Channel 00/0 : 3[3] -> 8[0] [send] via NET/IB/0/GDRDMA
```

Line by line:

- **`Using network IB`** — the network plugin NCCL selected. If this says `Socket`, every inter-node byte is going over kernel TCP and none of this lesson's bandwidth numbers apply; that is the first thing to check when a multi-node job is inexplicably slow.
- **`NET/IB : Using [0]mlx5_0:1/IB …`** — the ordered list of HCAs and ports NCCL will use, one entry per rail. Eight entries on an 8-GPU node is what you want; fewer means NICs are missing, filtered by `NCCL_IB_HCA`, or down. The trailing **`[RO]`** marks that PCIe relaxed ordering is enabled (`NCCL_IB_PCI_RELAXED_ORDERING`, default `2` = auto), and `OOB eth0:…` names the out-of-band interface used for bootstrap — the management path, not a rail.
- **`GPU Direct RDMA Enabled|Disabled for HCA n`** — the line that proves or disproves the entire premise of §1 for one GPU-NIC pair. The rule NCCL applies is mechanical: GDR is enabled when the GPU→NIC path type is **`PXB` or closer** (in v2.31 the default threshold is `PATH_PXB`, or `PATH_P2C` on C2C-capable systems), and disabled when the path is further away. That is exactly why `PHB`, `NODE` and `SYS` pairings lose GPUDirect: they are past the threshold. The threshold is overridable with **`NCCL_NET_GDR_LEVEL`**, which is a debugging tool, not a fix — forcing GDR across a root complex does not make the root complex go away.
- **`Channel 00/0 : 3[3] -> 8[0] [send] via NET/IB/0/GDRDMA`** — a per-channel routing decision: rank 3 sends to rank 8 through network device 0, and the `GDRDMA` suffix confirms the zero-copy path. A missing suffix on a pair you expect to be rail-local is the signature of a topology problem.

When PXN is in play, NCCL evaluates the GDR distance **for the intermediate GPU rather than the originating one** — the source comment is explicit that in the PXN case it substitutes the proxy GPU's distance — which is the mechanical reason PXN can rescue bandwidth: it makes the relevant GPU-NIC pair a `PIX` pair again, even though the data started somewhere else.

### 11. The three failure modes at the handoff, and how each announces itself

Everything that goes wrong at this boundary shows up as "the job is slower than the paper said" with nothing in the logs. Learn the three by their fingerprints:

| Failure | Mechanism | Fingerprint |
|---|---|---|
| **GPU-NIC pair past the GDR threshold** (`PHB`/`NODE`/`SYS`) | NIC cannot peer-to-peer DMA the GPU's HBM, so the driver stages through pinned host DRAM; the inter-node leg is capped by the root complex or the inter-socket link | `GPU Direct RDMA Disabled for HCA n` in the init log; inter-node bandwidth lands near 20–25 GB/s rather than 46–48 GB/s |
| **Host PCIe link trained down** | Link negotiated a lower width or generation (02b.3's `LnkSta`); the NIC is fed slower than it can transmit | `topo -m` still prints `PIX`; `lspci -vv` shows e.g. `8GT/s x8` where `32GT/s x16` was expected; NIC counters show the port far below line rate with no drops |
| **Rail mis-cabled at the leaf** | GPU-N's NIC is patched into the wrong leaf switch, so nominally rail-local traffic must cross the spine | Nothing local is wrong at all — the node looks perfect — but inter-node collectives show extra hops and higher, more variable latency, and the fabric's spine counters carry traffic that should not exist |

The first two are visible from inside the node. The third is only visible from the fabric side, which is why 09.2's tier-by-tier reading of a published architecture is the other half of this skill.

## Perspectives

**Developer.** The rail is invisible right up until it isn't. NCCL auto-detects topology and picks the rail-aligned NIC for you — until a mis-cabled node or a `SYS`-only GPU-NIC pairing forces a fallback, and your all-reduce step time quietly changes with no error and no crash, just a number that doesn't match the benchmark you were promised. The one habit that catches it: read the `NCCL_DEBUG=INFO` init block once per new SKU and confirm which HCA each rank chose and whether GPUDirect RDMA was enabled for it.

**Operator.** Rail cabling is decided at data-centre build time — which NIC plugs into which leaf — and it is expensive and disruptive to re-cable a live rack. A rail-mapping mistake made once during bring-up becomes a standing tax paid on every job for the life of the cluster, which is why serious operators treat the `topo -m` plus cabling audit as a release gate rather than a one-time sanity check. The audit is cheap: the map is static per SKU, so you derive it once and then assert it.

**Hardware / kernel.** The path codes are static facts about a specific motherboard's PCIe topology and firmware layout. They don't change at runtime and don't need re-checking per job. What *does* change at runtime, and is worth monitoring, is whether each PCIe link is still trained at its rated width and speed — 02b.3's `LnkSta` check — because §1 showed that a degraded host link moves the bottleneck off the wire without touching a single network counter.

**Economics.** Every GPU-indexed NIC is a discrete capex line, and at 800G the optics on both ends of each rail cable are a meaningful fraction of a node's network bill. The "extra" ninth NIC in the 2-8-9-800 reference build is not vendor upsell; it is deliberate traffic isolation whose payoff is measured in avoided stragglers. Defending that line item with the straggler argument — rather than "the reference architecture says so" — is the difference between a procurement review that goes well and one that doesn't.

## Real-world use cases

- **CoreWeave — GB200 NVL72 general availability.** [CoreWeave: First cloud provider to announce GA of NVIDIA GB200 NVL72 instances](https://www.coreweave.com/news/coreweave-first-cloud-provider-to-announce-general-availability-of-nvidia-gb200-nvl72-instances). What it shows: a production deployment where **both** boundaries in this lesson exist at once — rack-scale NVLink inside the 72-GPU domain, and a rail-optimised Quantum-2 InfiniBand fabric at 400 Gb/s per GPU outside it, scaling to clusters in the hundreds of thousands of GPUs, with SHARP in-network reduction accelerating collectives. It is the cleanest public example of "the domain edge moved to the rack, and everything past it is still the fabric."
- **Microsoft Azure "Eagle" supercomputer.** [ServeTheHome write-up](https://www.servethehome.com/microsoft-azure-eagle-is-a-paradigm-shifting-cloud-supercomputer-nvidia-intel/), cross-checked against the [TOP500 system record](https://top500.org/system/180236/). What it shows: 14,400 H100 GPUs across 1,800 nodes on Quantum-2 / ConnectX-7 InfiniBand, ranked #3 on the November 2023 TOP500 list — an independently listed instance of exactly the GPU→NIC→leaf cabling drawn in §3, at cloud scale rather than in a lab.
- **ByteDance MegaScale (NSDI '24).** [USENIX: MegaScale — Scaling LLM Training to More Than 10,000 GPUs](https://www.usenix.org/conference/nsdi24/presentation/jiang-ziheng). What it shows: a production system, independent of Meta and NVIDIA, built on the same rail-aware GPU-to-NIC design, reporting **55.2% model-FLOPs utilisation** training a 175B-parameter model on 12,288 GPUs. MFU is the number that proves the design works: it is the fraction of theoretical FLOPs actually delivered, and a fabric that mishandles the handoff shows up there first.

## Worked example

Read a node's topology, produce its rail map, and put a bandwidth number on one job's placement.

**Step 1 — dump the matrix.** The transcript below is a *representative* 8-GPU HGX H100 node (two sockets, eight GPUs, eight compute HCAs), formatted the way `nvidia-smi topo -m` prints it. It is not a capture from a specific machine; treat the shape as canonical and always re-run the command on the box in front of you.

```
$ nvidia-smi topo -m
        GPU0  GPU1  GPU2  GPU3  GPU4  GPU5  GPU6  GPU7  mlx5_0 mlx5_1 mlx5_2 mlx5_3 mlx5_4 mlx5_5 mlx5_6 mlx5_7 CPU Affinity NUMA
GPU0     X    NV18  NV18  NV18  NV18  NV18  NV18  NV18   PIX    PXB    NODE   NODE   SYS    SYS    SYS    SYS    0-47         0
GPU1    NV18   X    NV18  NV18  NV18  NV18  NV18  NV18   PXB    PIX    NODE   NODE   SYS    SYS    SYS    SYS    0-47         0
GPU2    NV18  NV18   X    NV18  NV18  NV18  NV18  NV18   NODE   NODE   PIX    PXB    SYS    SYS    SYS    SYS    0-47         0
GPU3    NV18  NV18  NV18   X    NV18  NV18  NV18  NV18   NODE   NODE   PXB    PIX    SYS    SYS    SYS    SYS    0-47         0
GPU4    NV18  NV18  NV18  NV18   X    NV18  NV18  NV18   SYS    SYS    SYS    SYS    PIX    PXB    NODE   NODE   48-95        1
GPU5    NV18  NV18  NV18  NV18  NV18   X    NV18  NV18   SYS    SYS    SYS    SYS    PXB    PIX    NODE   NODE   48-95        1
GPU6    NV18  NV18  NV18  NV18  NV18  NV18   X    NV18   SYS    SYS    SYS    SYS    NODE   NODE   PIX    PXB    48-95        1
GPU7    NV18  NV18  NV18  NV18  NV18  NV18  NV18   X     SYS    SYS    SYS    SYS    NODE   NODE   PXB    PIX    48-95        1
```

**Step 2 — read it, cell class by cell class.**

- **The GPU×GPU block is uniformly `NV18`.** Eighteen bonded NVLink-4 links between every pair, through the four NVSwitches. No pair is privileged and none is on PCIe. That is the signature of a healthy NVSwitch baseboard; a mix of `NV18` and lower counts would mean a bridge-connected board, and any `SYS` in that block would mean the NVLink fabric is down and GPU-to-GPU traffic has fallen back to PCIe — a catastrophic and very visible defect.
- **Each GPU has exactly one `PIX` NIC.** GPU0↔mlx5_0, GPU1↔mlx5_1, … GPU7↔mlx5_7, running down the diagonal of the NIC block. That diagonal *is* the rail map. A `PIX` cell means the GPU and the HCA are siblings under one PCIe switch, so a GPUDirect RDMA transfer between them is routed by that switch's crossbar and never reaches the root complex.
- **The `PXB` neighbour is the same-switch-group partner.** GPU0 reads `PXB` to mlx5_1 because both GPUs and both NICs hang off the same switch group; the path traverses more than one bridge but still avoids the CPU. `PXB` is a usable fallback; it is not the intended pairing.
- **`NODE` spans switch groups within one socket.** GPU0→mlx5_2 leaves the local switch and comes back down another, through the root complex. GPUDirect still works on many platforms but the root complex is now in the path.
- **`SYS` crosses the sockets.** Every GPU0–3 cell against mlx5_4..7 is `SYS`, and vice versa. That is the UPI/inter-socket link: 02b.4 puts a healthy link near **38 GB/s** and **20–25 GB/s** observed under contention, versus 50 GB/s on the wire. If placement ever forces GPU0 onto mlx5_4, the transfer does not fail — it silently loses about **56%** of its inter-node bandwidth and picks up the jitter of a shared inter-socket link.

**Step 3 — write the rail map down.** This is the artefact; everything else in the module consumes it.

| GPU | Rail | Rail NIC (`PIX`) | Acceptable fallback | Never use | NUMA |
|---|---|---|---|---|---|
| GPU0 | rail 0 | `mlx5_0` | `mlx5_1` (`PXB`) | `mlx5_4..7` (`SYS`) | 0 |
| GPU1 | rail 1 | `mlx5_1` | `mlx5_0` (`PXB`) | `mlx5_4..7` (`SYS`) | 0 |
| GPU2 | rail 2 | `mlx5_2` | `mlx5_3` (`PXB`) | `mlx5_4..7` (`SYS`) | 0 |
| GPU3 | rail 3 | `mlx5_3` | `mlx5_2` (`PXB`) | `mlx5_4..7` (`SYS`) | 0 |
| GPU4 | rail 4 | `mlx5_4` | `mlx5_5` (`PXB`) | `mlx5_0..3` (`SYS`) | 1 |
| GPU5 | rail 5 | `mlx5_5` | `mlx5_4` (`PXB`) | `mlx5_0..3` (`SYS`) | 1 |
| GPU6 | rail 6 | `mlx5_6` | `mlx5_7` (`PXB`) | `mlx5_0..3` (`SYS`) | 1 |
| GPU7 | rail 7 | `mlx5_7` | `mlx5_6` (`PXB`) | `mlx5_0..3` (`SYS`) | 1 |

**Step 4 — confirm it from `/sys`, not from the pretty table.** The matrix is a rendering; the filesystem is the ground truth, and it is what you script an audit against.

```
$ ls -d /sys/class/infiniband/*
/sys/class/infiniband/mlx5_0  /sys/class/infiniband/mlx5_1  ...  /sys/class/infiniband/mlx5_7

$ readlink -f /sys/class/infiniband/mlx5_0/device
/sys/devices/pci0000:0c/0000:0c:00.0/0000:0d:00.0        ← HCA's PCI path

$ readlink -f /sys/class/drm/card0/device                  ← or nvidia-smi -q | grep "Bus Id"
/sys/devices/pci0000:0c/0000:0c:00.0/0000:0e:00.0        ← GPU0's PCI path
                    └──────────┬────────────┘
                     common ancestor = the PCIe switch: this pair is PIX

$ cat /sys/class/infiniband/mlx5_0/device/numa_node
0

$ ls /sys/class/infiniband/mlx5_0/device/net
ibp13s0f0                                                 ← the netdev name for this HCA
```

Two GPUs and two HCAs sharing a bus prefix under one switch is the mechanical definition of `PIX`. When you script a fleet audit, compare the PCI ancestry, not the pretty table: the letters can change with driver version, the topology cannot.

**Step 5 — turn the map into a bandwidth statement for one job.** Take a 512-GPU data-parallel job (64 nodes × 8 GPUs) with a 1 GB per-GPU gradient buffer, and use §7's model.

```
  Placement A — rails intact, GPUDirect on every pair (PIX)
      NVLink term  2 × (7/8) × 1 GB / 450 GB/s          = 3.89 ms
      NIC term     2 × (63/64) × 0.125 GB / 50 GB/s     = 4.92 ms
      step         8.81 ms          effective all-reduce  114 GB/s

  Placement B — one node mis-cabled: GPU5's only NIC is SYS,
                so its inter-node leg runs at ~22 GB/s
      That node's rail-5 rank takes  1.969 × 0.125 / 22  = 11.19 ms
      All 512 GPUs are in the same barrier, so the step is
      set by the SLOWEST rank:  3.89 + 11.19             = 15.08 ms
      effective all-reduce                                 66 GB/s
      → a 42% step-time regression from ONE misplaced NIC
        on ONE of 64 nodes.
```

That last line is the whole argument for auditing rail maps, expressed as a number rather than as a principle: a collective is a barrier, so a single mis-paired GPU-NIC pair on a single node taxes every GPU in the job. NCCL's PXN path (§4) is what stops this from being worse — instead of dragging GPU5's traffic across UPI, NCCL can hop it over NVLink to a GPU that owns a `PIX` NIC and send from there — but PXN restores bandwidth, not rail locality, and its aggregation benefit does not make the defect free.

**Contrast, briefly.** On a GB200 NVL72 tray the same exercise looks different: GPU-to-GPU cells inside the 72-GPU domain read as NVLink connectivity throughout, and CPU-to-GPU cells read `C2C` rather than `PHB`, so there is no `PIX`/`SYS` gradient to reason about *inside* the domain. The meaningful GPU→NIC mapping starts again at the rack's edge, where the rack's NICs connect out to the inter-rack InfiniBand or Ethernet fabric. Same rail concept, different boundary.

## Practice

Feeds the deliverable **Network architecture read**.

**Task.** Take an `nvidia-smi topo -m` matrix — from any multi-GPU box you can reach, or the representative 8-GPU HGX matrix above if you have no cluster — and produce the node's **rail map plus its bandwidth ladder**.

**Requirements / acceptance:**

1. A complete **GPU→rail→NIC table** for one node: every GPU, its rail number, its chosen NIC, the GPU→NIC path code, and the NUMA node. For each GPU also name the NIC you would *not* use and its bad code (for example "`mlx5_4` = `SYS`, cross-socket").
2. Every chosen NIC must be one that gives GPUDirect RDMA without crossing a root complex — `PIX`, or at worst `PXB`. Justify any `PXB` choice.
3. A **bandwidth ladder** for one GPU on this node, per direction: NVLink per-GPU bandwidth, PCIe host-link bandwidth (state the generation and width you actually observe, not the datasheet value — use `lspci -vv` and read `LnkSta` as in 02b.3), and NIC line rate converted from Gb/s to GB/s. State which term binds and by how much.
4. One paragraph stating where NVLink stops and the switched fabric begins on this node's `GPU → NIC → leaf → spine` path, and one sentence stating whether that answer would change on a GB200 NVL72 rack and why.
5. Count the NICs on the node and state whether every NIC is a GPU rail NIC or whether one or more are held off the compute rail for storage and management (the 2-8-9-800 pattern). Name which, and say how you determined it.
6. Using §7's three-phase model, compute the **effective all-reduce bandwidth** for a job of your choosing spanning *M* nodes on this hardware, and state which of the two terms dominates. Then recompute assuming one GPU in the job is stuck on a `SYS` NIC, and state the step-time penalty for the whole job.

Save the table and the arithmetic; they are the intra-node half of the network-architecture read, and 09.2 adds the inter-node half.

## Common pitfalls

- **Comparing a bidirectional NVLink number to a unidirectional NIC number.** NVIDIA markets NVLink bidirectionally (900 GB/s on H100, 1 800 GB/s on Blackwell); PCIe and NIC rates are per direction. Mixing them gives 18:1 instead of the true 9:1 and doubles every conclusion you draw from the ratio. Always normalise to per-direction before you divide.
- **Assuming the path-code ranking generalises across vendors and generations.** `PIX`/`PXB`/`PHB`/`NODE`/`SYS` describe *this* SKU's PCIe topology. NCCL's internal vocabulary is longer (`NVL`, `NVB`, `C2C`, `P2C`, `PXN`, `NET`, `DIS`), and on Grace-based systems the interesting cell is `C2C`, which `nvidia-smi`'s five-code ranking has no place for. Re-read the matrix per box; do not carry last year's SKU's answer forward.
- **Conflating "rail" with "NUMA node."** A rail is a fabric-wide, cross-node equivalence class — GPU-N's NIC on every node, landing on the same leaf switch. A NUMA node is a single-server CPU/memory locality domain. They correlate (rails 0–3 on socket 0, rails 4–7 on socket 1 in the worked example) but they are not the same thing, and the correlation breaks on SKUs whose NIC-to-socket layout doesn't match rail numbering.
- **Treating PXN as making a mis-cabled rail harmless.** PXN routes around a bad GPU-to-NIC pairing by hopping over NVLink to a GPU that owns a good NIC, and it aggregates messages while it's there. It recovers most of the bandwidth that a `SYS` path would have lost. It does **not** restore rail locality: the bytes still exit on the intermediate GPU's rail, so the fabric may still have to carry a cross-rail crossing that correct cabling would have avoided.
- **Forgetting the GB200 NVL72 exception — or over-applying it.** "NVLink never leaves the chassis" is the right default and the wrong absolute: on NVL72 the domain is the whole rack, 72 GPUs, one non-blocking NVLink fabric. But the exception does not delete the handoff, it relocates it. An NVL72 rack still has NICs and still pays the cliff for every byte that leaves the rack.
- **Counting the management NIC as collective bandwidth.** In the 2-8-9-800 reference build only eight of nine NICs carry GPU-indexed collective traffic; on a DGX H100 the compute fabric is eight ConnectX-7 ports and the storage/management fabric is separate QSFP112 adapters. Folding the off-rail NICs into a bandwidth or capex number for the compute fabric overstates what the job has and misstates the build's cost structure.
- **Reading `topo -m` and never checking `LnkSta`.** The matrix tells you the *shape* of the path; it says nothing about whether the link on that path trained to its rated width and speed. §1 showed that a Gen5 x8 link caps a 400G NIC at 63% of line rate. The matrix will still cheerfully print `PIX`.

## Self-check

**(a) Why does cross-rail GPU-to-GPU traffic prefer NVLink over going out the NIC? Put a number on it.**

**Answer:** Because NVLink is board-abundant and the NIC is scarce. Per direction, an H100 has 450 GB/s of NVLink (18 links × 25 GB/s) against 50 GB/s on a 400G ConnectX-7 — a 9:1 ratio, and the same 9:1 on Blackwell (900 GB/s vs 100 GB/s on ConnectX-8). Shuffling a tensor sideways over NVLink so each GPU lines up with its own rail therefore costs roughly one ninth of the bandwidth of pushing it across a leaf, up a spine and back down another rail, and a fraction of the latency. It also has a second-order benefit that matters more at the fabric level: it keeps the NIC carrying only rail-local GPU-N ↔ GPU-N traffic, which is exactly the property that lets the spine be oversubscribed safely (09.2). NCCL implements this as **PXN**, controlled by `NCCL_P2P_PXN_LEVEL` (default 2 in NCCL v2.31) and disabled by `NCCL_PXN_DISABLE`.

**(b) On a topo matrix, which GPU/NIC pairs show `PXB`/`SYS`, and what does each cost?**

**Answer:** `PXB` means the GPU→NIC path traverses multiple PCIe bridges — a switch cascade — but still avoids the CPU root complex; GPUDirect RDMA works with a small latency penalty, and it is the acceptable fallback when the `PIX` NIC is unavailable. `SYS` means the path crosses the inter-socket link (UPI or Infinity Fabric) to the other CPU. The NIC can no longer perform a clean peer-to-peer DMA of the GPU's HBM, so the transfer is staged through pinned host memory and the bandwidth ceiling collapses to the inter-socket link: roughly 38 GB/s in the best case and 20–25 GB/s under contention (02b.4), against 50 GB/s on the wire — about a 56% loss for that GPU's inter-node path, with no error raised anywhere. In a barrier-synchronous collective that loss is paid by every rank in the job, not just the affected one.

**(c) Where does NVLink stop and the switched fabric begin in the `GPU → NIC → leaf → spine` path?**

**Answer:** NVLink/NVSwitch is bounded by the NVLink domain — on 8-GPU HGX hardware, the baseboard inside one chassis. There is no NVLink cable between standard servers. Traffic to a GPU outside that domain exits GPU → PCIe → **NIC**, and the switched fabric (InfiniBand or RoCE Ethernet) begins at the **cable from NIC to leaf switch**. The NIC is the handoff point: everything to the left of it is 02b's material, everything to the right is module 09's. On GB200/GB300 NVL72 the domain is a whole rack, so the handoff point moves to the rack's edge — the mechanism is unchanged, the boundary is further out.

**(d) Does "NVLink never leaves the chassis" hold on GB200 NVL72? What exactly changes, and what doesn't?**

**Answer:** No — it is the one real exception. GB200 NVL72 binds 72 Blackwell GPUs and 36 Grace CPUs into a single non-blocking NVLink domain spanning the rack, via nine NVLink switch trays, at 130 TB/s aggregate GPU-to-GPU bandwidth and 900 GB/s per direction per GPU. What changes is the scope of "chassis": one board becomes one rack, and in §7's model the shard that has to cross the wire shrinks from `S/8` to `S/72`, which collapses the inter-node term by roughly 9×. What does *not* change: the rack still has NICs, still cables into leaf switches, and still pays the full cliff for every byte that leaves the rack. Rack-scale NVLink raises the floor rather than removing it.

**(e) Why does NVIDIA's HGX AI Factory reference architecture spec nine NICs for an 8-GPU node, not eight?**

**Answer:** Eight NICs are the GPU-indexed rail NICs, one per GPU, each carrying that GPU's GPUDirect RDMA collective traffic (the 2-8-9-800 build: 2 CPUs, 8 GPUs, 9 NICs, 800 Gb/s each). The ninth is deliberately kept off the compute rail and dedicated to storage and cluster-management traffic. The mechanism behind the decision is straggler avoidance rather than tidiness: checkpoint writes and dataset staging are large and bursty, a collective is a barrier, and a rail NIC momentarily busy with checkpoint I/O stalls every GPU in the job, not just its own. DGX H100 ships the same architectural split with different packaging — eight ConnectX-7 compute ports plus two dual-port QSFP112 adapters for storage and management. Count the two populations separately in any bandwidth or capex calculation.

**(f) A job spans 64 nodes (512 GPUs) with a 1 GB gradient buffer per GPU. Which dominates — the NVLink phase or the NIC phase — and what happens as you scale to 4,096 GPUs?**

**Answer:** Using the three-phase model: the NVLink term is `2 × (7/8) × 1 GB / 450 GB/s = 3.89 ms` and is independent of node count; the NIC term is `2 × (M−1)/M × (1/8) GB / 50 GB/s`, which is 4.92 ms at M = 64 and converges to 5.00 ms as M grows. So the NIC phase dominates, but only modestly — the phases are near-balanced on H100 + ConnectX-7, because the NVLink phase moves seven times more bytes over nine times more bandwidth. Scaling from 512 to 4,096 GPUs adds under 1% to step time, because the ring factor `2(M−1)/M` is bounded above by 2. That is why data-parallel training scales at all; every real-world deviation from it (oversubscription, congestion, stragglers) is a fabric property rather than an algorithmic one. For contrast, running the same all-reduce as a flat ring with every hop over the NIC costs ≈40 ms — about 4.5× worse — which is what the rail-aware hierarchy is buying you.

## Connections & what's next

This lesson is the hinge between 02b — everything inside one box — and the rest of module 09. It also reaches sideways: 08's ring and tree all-reduce is *why* rail locality matters, because it is the traffic pattern that makes GPU-N-to-GPU-N the hot path; 06's topology-aware gang placement is *how* a scheduler keeps a job's GPUs rail-aligned in the first place; and 02b.3's `LnkSta` check is what stops a degraded host link from silently moving the bottleneck off the wire.

The next lesson, **09.2 (the inter-node fabric)**, takes the rail you just defined and asks what happens once traffic climbs off it. It builds the Clos/fat-tree tiers above the leaf, defines full bisection bandwidth and oversubscription with the arithmetic that goes with them, and shows — using Meta's published 24,576-GPU Llama-3 cluster, plus real deviations from that design at Alibaba and in 100K-GPU builds — why a fabric can be heavily oversubscribed at the spine and lose almost nothing, *because* of the rail locality established here. Carry three things forward: the mapping `GPU index ↔ rail ↔ leaf switch ↔ dedicated NIC`, the 9:1 cliff, and the three-phase all-reduce model, which 09.2 extends with an oversubscription term.

## References & further reading

**Source-access note for this pass.** Several vendor and publisher domains (docs.nvidia.com, arxiv.org, IEEE, ACM, USENIX, Wikipedia) are blocked by this environment's egress proxy and could not be fetched while writing. Where a fact came from a source that *was* reachable — the Linux kernel tree, rdma-core, NCCL, perftest, and papers hosted on reachable mirrors — that is stated explicitly below and the fact was checked against the source text. Entries marked **not re-verified in this pass** are carried forward from the previous version of this lesson with their original attribution; treat their figures as citations to check rather than as verified-here numbers.

**Verified against source in this pass**

- **NCCL v2.31.2 source** (`github.com/NVIDIA/nccl`, commit on the v2.31.2 release line). Verified here: the path-type vocabulary `LOC NVL NVB C2C PIX PXB P2C PXN PHB SYS NET DIS` in `src/graph/topo.cc`; the PXN control parameter `NCCL_PARAM(P2pPxnLevel, "P2P_PXN_LEVEL", 2)` and its documented semantics (`0`: no PXN for P2P, `1`: use if needed, `2`: maximise aggregation) in `src/graph/search.cc`. Read for: the fact that NCCL's model of the topology is richer than `nvidia-smi`'s five-code rendering, and for PXN's real default.
- **rdma-core** (`github.com/linux-rdma/rdma-core`). Verified here: `enum ibv_mtu` in `libibverbs/verbs.h` giving the legal IB path MTUs (256, 512, 1024, 2048, 4096 bytes), used in §2's segmentation step. Read for: anything about what the verbs layer actually exposes, rather than what vendor documentation says it exposes.
- **Course lesson 02b.4, "The 8-GPU server: HGX/DGX H100 topology"** — the source of the NVLink generation table (18 links × 25 GB/s per direction on NVLink 4; 18 × 50 on NVLink 5), the 7.2 TB/s aggregate and 3.6 TB/s bisection figures, the four-NVSwitch baseboard, the UPI 38 GB/s figure, and the DGX H100 NIC population. All figures in this lesson are carried forward unchanged so the two modules agree.
- **Course lesson 02b.3, "PCIe"** — the source of 63.02 GB/s per direction for Gen5 x16, the ~54 GB/s usable figure and its four-loss breakdown, and the 121 GB/s Gen6 x16 figure used in the ConnectX-8 argument.

**Cited, not re-verified in this pass** (fetches blocked by the egress proxy)

- NVIDIA, [GB200 NVL72 product page](https://www.nvidia.com/en-us/data-center/gb200-nvl72/) — cited for: 72 GPUs and 36 Grace CPUs as one NVLink domain, 130 TB/s aggregate, nine NVLink switch trays.
- NVIDIA Technical Blog, [GB200 NVL72 Delivers Trillion-Parameter LLM Training and Real-Time Inference](https://developer.nvidia.com/blog/nvidia-gb200-nvl72-delivers-trillion-parameter-llm-training-and-real-time-inference/) — cited for: the fifth-generation NVLink Switch specification and the multi-rack NVLink fabric.
- NVIDIA, [HGX AI Factory Reference Architecture — network logical architecture](https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/network-logical-architecture.html) — cited for: the 2-8-9-800 reference build and the rationale for nine NICs on an 8-GPU node.
- NVIDIA, [ConnectX-8 SuperNIC documentation](https://docs.nvidia.com/networking/display/connectx8SuperNIC/Introduction) — cited for: 800 Gb/s, native PCIe Gen6 x16, single IB XDR port or dual 400G Ethernet ports.

**Real-world engineering**

- CoreWeave, [First cloud provider to announce GA of NVIDIA GB200 NVL72 instances](https://www.coreweave.com/news/coreweave-first-cloud-provider-to-announce-general-availability-of-nvidia-gb200-nvl72-instances) — what it shows: rack-scale NVLink and a rail-optimised InfiniBand fabric coexisting in one production deployment, which is the exact "the boundary moved, it didn't vanish" picture of §8. Not re-verified in this pass.
- ServeTheHome, [Microsoft Azure Eagle is a paradigm-shifting cloud supercomputer](https://www.servethehome.com/microsoft-azure-eagle-is-a-paradigm-shifting-cloud-supercomputer-nvidia-intel/), with the [TOP500 Eagle record](https://top500.org/system/180236/) — what it shows: 14,400 H100s on Quantum-2 / ConnectX-7 InfiniBand at #3 on the November 2023 list; independent confirmation that the GPU→NIC→leaf pattern is what large clouds actually cable. Not re-verified in this pass.
- ByteDance, via USENIX NSDI '24, [MegaScale: Scaling LLM training to more than 10,000 GPUs](https://www.usenix.org/conference/nsdi24/presentation/jiang-ziheng) — what it shows: 55.2% MFU on 12,288 GPUs with a rail-aware design, from a team unaffiliated with Meta or NVIDIA. Not re-verified in this pass.

**Deeper dives**

- Glenn K. Lockwood, ["NVLink" practitioner notes](https://www.glennklockwood.com/garden/nvlink) — an independent, non-vendor breakdown of NVLink generations and their real limits; useful as a counterweight to reading only datasheet numbers. Not re-verified in this pass.
- Jonathan Hui, ["NVIDIA Blackwell GB200 NVL72 and networking"](https://jonathan-hui.medium.com/nvidia-blackwell-gb200-nvl72-networking-e36aade6ced9) — an independent walkthrough of the NVL72 rack's physical NVLink/NVSwitch wiring, for building intuition past the datasheet. Not re-verified in this pass.
