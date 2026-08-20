---
lesson: "08.2"
title: "NCCL collectives: topology, transport, and the silent hang"
module: "08"
concept: "NCCL collectives: topology, transport, and the silent hang"
status: not-started
est_time: "9h"
prev: "01-parallelism-strategies.md"
next: "03-communication-bottleneck.md"
artifacts: []
sources: 14
---

# 08.2 · NCCL collectives: topology, transport, and the silent hang

> **Concept.** NCCL is the layer that turns "which collective over which link" into real wire traffic — and its signature failure is a job that hangs at 100% GPU utilization with *no error*, which you debug by reading its topology/transport logs and, on 2.24+, querying the RAS subsystem.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)
>
> **Advanced Learning** — [The Ring, Derived](../../../learning/02-nccl-collectives.html): the ring all-reduce worked chunk by chunk, and why per-rank bytes converge to 2S and stop. Optional visual companion; this lesson stays the source of truth.

## Where this fits

This is the module's **anchor lesson** — lesson 2 of 8, and the one the module README names as its centerpiece ("06 places the gang; 08 keeps it alive; 05's XID is the signal that triggers a restart"). 08.1 told you *which* collective each parallelism strategy issues and how many bytes it carries. This lesson is the layer underneath: how NCCL turns `ncclAllReduce(...)` into an ordered sequence of point-to-point transfers over a specific set of physical links, how it decides which links and which ordering, and what happens to the whole job when one of the participants stops showing up.

Two concrete debts from 08.1 are paid here. First, 08.1 estimated collective times as "at 450 GB/s that takes 263 ms" and pointed forward to a bus-bandwidth model — that model is derived from first principles in §3–4 below, including why per-GPU bytes converge to a constant as world size grows. Second, 08.1 kept saying "a hang, by contrast, pins utilisation at 100% and freezes the step counter" — §11–13 is the mechanism behind that sentence and the triage procedure that follows from it. 08.3 (MFU), 08.4 (checkpoint sizing) and 08.5 (failure and elasticity) all assume you can already do what this lesson teaches.

## Why this matters

This is the single most valuable debugging skill in distributed training, and the one that most cleanly separates a senior platform engineer from everyone else on the incident bridge.

A NCCL collective is a **synchronisation barrier**: every rank must arrive or none proceed. So when one GPU on one node out of thousands dies, straggles, or loses its NIC, the collective does not error — it *waits*. Every other GPU spins at 100% utilisation inside a communication kernel, burning money, emitting nothing, until a watchdog timeout fires (10 minutes later, at PyTorch's default) with a message that names the *victims*, not the *cause*.

Put money on it. A 512-GPU H100 job at a nominal $3/GPU-hour burns **$1,536 per hour**, or $25.60 per minute, whether it is training or blocked in `ncclAllReduce`. Ten minutes to a watchdog timeout plus twenty minutes of log archaeology plus a restart is roughly **$1,300 for one incident**, and the Llama 3 fleet story says you get one of these every few hours. Teams that keep >90% effective training time are the ones that can localise a silent hang in minutes rather than hours. If you can read NCCL's topology and transport decisions and run the triage sequence cold, you own that number.

## What's new here (calibration)

**02b** taught you the *links*: NVLink/NVSwitch domains, rail alignment, GPUDirect RDMA, and the per-generation bandwidth table. **05** taught you GPU *health*: XID codes, DCGM, when a GPU is physically sick. **06** taught you gang scheduling and topology-aware placement. **08.1** (previous lesson) told you *which collective* each parallelism strategy issues and how large its messages are.

This lesson is the piece that connects them: **NCCL is the software that, at job startup, discovers the topology from 02b, builds a communication graph on it, decides which transport and which algorithm to use for each collective from 08.1, and then runs them.** Genuinely new here:

- **The ring all-reduce derived**, chunk by chunk and step by step, so the `2(N−1)` steps and the `2(N−1)/N · M` per-rank byte count are things you can re-derive rather than recall — and so the **bus-bandwidth** number that every NCCL benchmark reports means something to you.
- **The double binary tree**, why NCCL builds *two* trees rather than one, and the exact latency/bandwidth trade against ring — quantified with NCCL's own tuning constants, which are compiled-in numbers you can read out of the source.
- **Protocols (LL, LL128, Simple)** as a separate axis from algorithms, because `NCCL_PROTO` shows up in every tuning log line and almost nobody can explain what it selects.
- **The real log-line grammar** for each debug subsystem, so a `NCCL_DEBUG=INFO` capture is something you read rather than something you grep hopefully.
- **The RAS subsystem's actual output format** (NCCL ≥ 2.24) and PyTorch's flight recorder and desync report — the three tools that turn "3,000 GPUs hung" into "rank 4831 stopped at collective 6650".

A silent NCCL hang produces no XID (05 is quiet), the GPU is electrically fine, and gang scheduling (06) has done its job. **06 places the gang; 08 keeps it alive.** When it hangs, the diagnosis lives in NCCL's own logs and its RAS subsystem, not in DCGM. We skip ML-eng entirely: you are reading NCCL's *behaviour*, never writing custom collectives or kernel code.

> **Version note.** Every flag, default, log format and constant below was verified against the **NCCL v2.31.2** source and user guide cloned from `github.com/NVIDIA/nccl`, and against **PyTorch** at HEAD. The module's resource spine pins ~v2.30.x; where behaviour changed recently the introducing version is given inline. NVIDIA's hosted documentation site is unreachable from this environment's egress proxy — everything here comes from the in-repo `docs/userguide/source/*.rst` and from the C++ itself.

## Core concepts

### 1. What a collective actually is, and why that makes it a barrier

Start with the API surface, because the failure mode falls straight out of it. NCCL exposes a communicator (`ncclComm_t`) that binds a fixed set of ranks, and a handful of collective calls that every rank in that communicator must call, **in the same order, the same number of times, with the same sizes and datatypes**. `ncclAllReduce(sendbuff, recvbuff, count, datatype, op, comm, stream)` is asynchronous with respect to the host — it enqueues a kernel on a CUDA stream and returns. The synchronisation is not in the API; it is in the data dependency.

That ordering requirement is not advisory. Internally each communicator keeps an **operation counter** (`opCount`) that increments on every enqueued collective, and NCCL's own status reporting groups ranks by that counter. If rank 5 issues `AllReduce, AllGather` while every other rank issues `AllGather, AllReduce`, the two operations get matched pairwise by position and you get either a hang or silently wrong data. Frameworks avoid this by construction — DDP and FSDP issue collectives from the autograd engine in a deterministic order — which is why the ordering bugs you actually see come from *conditional* code: `if rank == 0: log_something_that_all_reduces()`, or an early `break` out of a training loop on one rank when its data shard runs out first.

**The barrier property is a consequence of the maths, not a design choice.** An all-reduce's output on any rank depends on the input of *every* rank:

```
  o_0 = o_1 = ... = o_{N-1} = i_0 + i_1 + ... + i_{N-1}
```

There is no partial answer. A rank cannot produce a correct result having heard from only some peers, so it must wait. This is the sentence the entire second half of this lesson is about.

Here is the collective set, with exactly what each one computes and who issues it (the "issued by" column is 08.1's table, now attached to real semantics):

| Collective | Input per rank | Output per rank | Issued by |
|---|---|---|---|
| **AllReduce** | full array `i_r` of `S` bytes | `Σ_r i_r`, all `S` bytes, on every rank | DDP gradient sync; Megatron TP per-block |
| **ReduceScatter** | full array `i_r` of `S` bytes | `(Σ_r i_r)` restricted to rank `r`'s `S/N` slice | FSDP gradient sharding; ZeRO-2/3 |
| **AllGather** | `S/N`-byte shard | the concatenation of all shards, `S` bytes | FSDP parameter materialisation; ZeRO-3 |
| **Broadcast** | root's array | root's array, on every rank | init, weight sync at start, checkpoint load |
| **Reduce** | full array | `Σ_r i_r`, on the root only | rarely, in eval/metric paths |
| **AllToAll** *(via grouped Send/Recv)* | `N` distinct chunks | the `r`-th chunk from every rank | MoE expert dispatch/combine |

The identity to hold onto: **an all-reduce is a reduce-scatter followed by an all-gather.** NCCL's ring implementation is literally that, back to back. It explains why those three collectives dominate every trace, why an all-reduce costs exactly twice a reduce-scatter, and why sequence parallelism (08.1 §6) can replace one all-reduce with an all-gather plus a reduce-scatter at *zero* wire cost — it is the same traffic, re-labelled.

### 2. Channels, and what NCCL actually builds at init

Before any bytes move, `ncclCommInitRank` does four things, in order, and each has a distinctive log signature (§9):

1. **Bootstrap.** Ranks find each other over plain TCP sockets using the out-of-band interface (`NCCL_SOCKET_IFNAME`) and a shared unique ID. Nothing GPU-specific has happened yet. Bootstrap failure is the classic *init-time* hang.
2. **Topology detection.** Each rank walks its local PCIe tree, NVLink connectivity and NIC inventory, and builds an XML representation of the node. You can dump it with `NCCL_TOPO_DUMP_FILE=/tmp/topo.xml` and feed a hand-written one back with `NCCL_TOPO_FILE` — which is how you work around a platform whose ACPI tables lie about NUMA locality.
3. **Graph search.** NCCL searches for a set of **rings** and **trees** over the discovered topology that maximise achievable bandwidth subject to the link speeds it found. This is the `GRAPH` subsystem's output.
4. **Transport setup.** For each pair of neighbours in each ring/tree, NCCL picks the fastest usable transport (§8) and allocates buffers.

The unit that comes out of graph search is a **channel**. A channel is one ring (or one tree) over all ranks, and it maps to **one CUDA thread block** on each GPU. NCCL runs several channels in parallel over disjoint-ish sets of links, because a single ring cannot saturate a GPU that has 18 NVLinks. So "the ring" in the next section is really "one of typically 8–32 rings running concurrently", and the aggregate bandwidth is per-channel bandwidth times channel count. Two knobs matter:

| Variable | What it does | Default | When you touch it |
|---|---|---|---|
| `NCCL_MIN_CTAS` / `NCCL_MAX_CTAS` | floor/ceiling on channels (= CUDA blocks) NCCL may use | auto from topology | raise the floor when small collectives underuse the fabric; lower the ceiling when NCCL is stealing SMs from compute |
| `NCCL_BUFFSIZE` | per-channel, per-peer staging buffer | **4194304 (4 MiB)** | shrink under HBM pressure; it is a real, non-trivial per-GPU allocation at high channel counts |
| `NCCL_NTHREADS` | CUDA threads per block | **512** on recent GPUs, 256 on older; legal values 64/128/256/512 | almost never |

`NCCL_MAX_NCHANNELS` / `NCCL_MIN_NCHANNELS` are the older spellings and were **deprecated in 2.17** in favour of the CTA names; if both are set, the more restrictive wins. That `NCCL_BUFFSIZE` default of 4 MiB is the "NCCL buffers" line item in 08.1's memory budget — at 32 channels with several peers it is gigabytes, not rounding error.

### 3. The ring all-reduce, derived

**The problem.** You have `N` GPUs each holding an `S`-byte array, and you need every GPU to end up with the elementwise sum. The naive schemes are both bad: sending everything to rank 0 to reduce and broadcasting back makes rank 0's single link carry `2(N−1)S/N ≈ 2S` bytes while every other link idles, and having every rank send to every rank moves `N(N−1)` messages. The ring solves it by making *every* link carry the same load, all the time.

**The construction.** Arrange the ranks in a logical ring `0 → 1 → 2 → … → N−1 → 0`. Split each rank's array into `N` chunks of `S/N` bytes each. Every step, every rank sends exactly one chunk to its successor and receives exactly one chunk from its predecessor — so every link in the ring is busy in every step, in both phases.

**Phase 1, reduce-scatter, `N−1` steps.** In step `s` (counting from 0), rank `r` sends the chunk with index `(r − s) mod N` and adds the incoming chunk into its own copy. After `N−1` steps rank `r` holds the *fully reduced* chunk `(r + 1) mod N`, and nobody holds any other chunk completely.

**Phase 2, all-gather, `N−1` steps.** Same ring, same direction, but the incoming chunk is *stored*, not accumulated. After `N−1` more steps every rank has every fully reduced chunk.

Drawn out for `N = 4`, with `aᵣ` meaning "rank r's contribution to chunk a":

```
  RING ALL-REDUCE, N = 4 — what each rank holds after each step
  ══════════════════════════════════════════════════════════════════════════
  Ring: 0 → 1 → 2 → 3 → 0.  Each array split into 4 chunks: a b c d.
  Chunk size = S/4 bytes.  Every step, every rank sends ONE chunk right.

  INITIAL                chunk a   chunk b   chunk c   chunk d
    rank 0                 a0        b0        c0        d0
    rank 1                 a1        b1        c1        d1
    rank 2                 a2        b2        c2        d2
    rank 3                 a3        b3        c3        d3

  ── PHASE 1: REDUCE-SCATTER  (N−1 = 3 steps, ACCUMULATE on arrival) ──

  step 1   rank r sends chunk r          0:a0▶1   1:b1▶2   2:c2▶3   3:d3▶0
    rank 0                 a0        b0        c0      d3+d0 ●
    rank 1               a0+a1 ●     b1        c1        d1
    rank 2                 a2      b1+b2 ●     c2        d2
    rank 3                 a3        b3      c2+c3 ●     d3

  step 2   rank r sends chunk (r−1)      1:a▶2    2:b▶3    3:c▶0    0:d▶1
    rank 0                 a0        b0     c2+c3+c0 ●  d3+d0
    rank 1               a0+a1       b1        c1     d3+d0+d1 ●
    rank 2             a0+a1+a2 ●   b1+b2      c2        d2
    rank 3                 a3     b1+b2+b3 ●  c2+c3      d3

  step 3   rank r sends chunk (r−2)      2:a▶3    3:b▶0    0:c▶1    1:d▶2
    rank 0                 a0      ┃  B  ┃  c2+c3+c0   d3+d0
    rank 1               a0+a1        b1    ┃  C  ┃  d3+d0+d1
    rank 2             a0+a1+a2      b1+b2     c2     ┃  D  ┃
    rank 3             ┃  A  ┃     b1+b2+b3   c2+c3      d3
                        ▲
      ┃X┃ = fully reduced.  After N−1 steps rank r owns exactly one
      complete chunk — chunk (r+1) mod N.  That IS a reduce-scatter.

  ── PHASE 2: ALL-GATHER  (N−1 = 3 more steps, OVERWRITE on arrival) ──
      The same ring, same direction, carrying the finished chunks:
      A travels 3→0→1→2,  B travels 0→1→2→3,  C: 1→2→3→0,  D: 2→3→0→1.

  FINAL    every rank holds  ┃A┃ ┃B┃ ┃C┃ ┃D┃  = the complete all-reduce result

  ── ACCOUNTING ────────────────────────────────────────────────────────────
     steps            = (N−1) + (N−1) = 2(N−1)          = 6 for N=4
     bytes per step   = S/N per rank                    = S/4
     bytes per rank   = 2(N−1)/N · S                    = 1.5 S for N=4
     bytes per link   = identical — no hot spot, ever
```

The last three lines are the whole point, so read them twice.

**Why the per-rank byte count converges to `2S` and stops.** `2(N−1)/N = 2 − 2/N`. At `N = 2` it is 1.0, at `N = 8` it is 1.75, at `N = 64` it is 1.969, at `N = 16,384` it is 1.99988. It is bounded above by 2 and it gets there fast. **Adding data-parallel replicas does not multiply your per-GPU bandwidth bill** — it is asymptotically flat. This is the single most misunderstood fact about scaling data parallelism, and it is why 08.1 could say "the reason to leave DDP is memory pressure, not bandwidth scaling."

**What *does* grow with `N` is the step count**, `2(N−1)`, and therefore the latency floor. Each step is a real hop with a real per-hop cost — kernel-side handshake, buffer copy, and on inter-node hops a proxy thread posting an RDMA operation. NCCL's own cost model, verbatim from `src/tuning/tuning_general.cc`, is:

```c
int ncclTuningGetNsteps(int coll, int nRanks) {
  if (coll == ncclFuncAllReduce)       return 2 * (nRanks - 1);
  else if (coll == ncclFuncReduceScatter || coll == ncclFuncAllGather)
                                        return nRanks - 1;
  else                                  return nRanks;
}
```

So the ring cost model is

```
                                    2(N−1)      S
   T_ring(S, N)  =  2(N−1)·α   +   ──────── · ─────
                                       N          B
                    └───┬────┘       └──────┬───────┘
                 latency term        bandwidth term
                 LINEAR in N         → 2S/B, FLAT in N

   α = per-hop latency (µs), B = per-rank link bandwidth (bytes/s)
```

Both terms matter, in different regimes: at small `S` the first dominates and ring is a bad choice; at large `S` the second dominates and ring is optimal. §5 is about the algorithm that flips the first term from linear to logarithmic.

### 4. Bus bandwidth — the number every NCCL benchmark reports

You will run `all_reduce_perf` and get two bandwidth columns, and you must know which one to compare against a spec sheet.

**Algorithm bandwidth** is the naive one: `algbw = S / t`. It answers "how long will an `S`-byte all-reduce take" — divide and you have your answer. It is useless for judging hardware, because it falls as `N` grows even on a perfectly healthy fabric: the same wire has to carry `2(N−1)/N · S` bytes rather than `S`.

**Bus bandwidth** corrects for exactly that. From the derivation above, the best possible time for a ring all-reduce with per-rank bandwidth `B` is `t = (S/B)·(2(N−1)/N)`. Invert it:

```
   busbw  =  algbw × 2(N−1)/N          (all-reduce)
   busbw  =  algbw × (N−1)/N           (all-gather, reduce-scatter)
   busbw  =  algbw                     (broadcast, reduce — root is the bottleneck)
```

Now `busbw` is comparable to the link's spec number regardless of rank count. The correction factors are exactly the step counts from §3 divided by `N`, and `nccl-tests` applies the right one per collective — which is why an all-gather's `busbw` is directly comparable to an all-reduce's even though its `algbw` is nearly double.

**Two traps.**

*Trap one: `busbw` is only a wire speed when the algorithm is a flat ring.* When NCCL selects a hierarchical algorithm (tree, or the intra-node-then-inter-node decomposition), the payload no longer maps onto any single link and `busbw` can exceed the physical link rate. On an 8×H200 node, `stas00/ml-engineering`'s measurements at a 16 GiB payload show `all_reduce` reporting **480.0 GB/s busbw against a 450 GB/s NVLink 4 unidirectional spec — 107%** — because NCCL selected `NVLS` and the NVSwitch did the reduction. Force the ring with `NCCL_NVLS_ENABLE=0` and the same collective reports **367.2 GB/s, 82% of spec**, which is the honest wire number. All-gather and reduce-scatter on the same node report 361.4 and 362.9 GB/s — 80–81% — because they have no NVLS path and always ran the ring.

*Trap two: ~80% of spec is the normal, healthy result*, not a problem to chase. Encoding overhead and protocol framing eat the rest. If you measure 80–88% of the unidirectional spec, your fabric is fine.

**Worked, so you can do it on your own cluster.** Suppose you measure a 4 GiB all-reduce on 8 GPUs taking 11.7 ms.

```
   S     = 4 GiB = 4 × 2^30 = 4.295e9 bytes     ← decimal bytes, see the trap below
   algbw = 4.295e9 / 0.0117 s   = 367.1e9 B/s   = 367.1 GB/s
   busbw = 367.1 × 2·(8−1)/8    = 367.1 × 1.75  = 642.4 GB/s
```

Wait — that exceeds any H100 link. The mistake is applying the all-reduce correction to something that is not one; if that 11.7 ms were an *all-gather*, the factor is `(N−1)/N = 0.875` and `busbw = 321 GB/s`. **Always confirm which collective and which algorithm ran** (`NCCL_DEBUG_SUBSYS=TUNING`, §9) before interpreting the number. And mind the bases: benchmarks print payloads in GiB (`2^30`) but bandwidths in GB/s (`10^9`); dividing GiB by GB/s as if they shared a base understates every time by about 7%.

### 5. Tree — trading bandwidth for latency, and why NCCL builds two of them

**The problem ring does not solve.** At 1,024 ranks a ring all-reduce takes `2 × 1023 = 2046` steps. Even at a generous 2 µs per inter-node hop that is ~4 ms of pure latency before a single useful byte of a small message has moved. For gradient buckets of a few hundred KB — and DDP's default bucket is 25 MiB, but FSDP's per-layer messages and any small tensor are far below that — the latency term is the entire cost.

**The tree.** Arrange the ranks in a binary tree. Reduce climbs from leaves to root (`log₂N` levels), then the result broadcasts back down (`log₂N` levels). Latency is `~2·log₂(N)·α` instead of `2(N−1)·α`: at `N = 1024` that is 20 hops instead of 2046, a **100× reduction in the latency term**.

**The cost.** In a plain binary tree, half the ranks are leaves and their upward link is idle during the broadcast phase; more importantly, internal nodes must handle traffic from two children on one link. The effective bandwidth is roughly half the ring's. NCCL's model encodes this literally — in `src/tuning/tree.cc` the tree's modelled bandwidth is `busBw * 0.5`.

**Why two trees.** NCCL does not build one binary tree; it builds a **double binary tree**. It constructs one tree over the ranks, then a second, complementary tree (mirrored when `N` is even, shifted by one rank when odd) with the property that **every rank that is an internal node in one tree is a leaf in the other**. Each tree carries half the data. Now every rank is doing useful work in both directions at once, and the aggregate recovers most of the bandwidth that a single tree gave away, while keeping the `log N` depth. The construction lives in `src/graph/trees.cc` (`ncclGetBtree`, `ncclGetDtree`) and is pure bit arithmetic on the rank index — no communication needed to agree on it.

```
  RING vs DOUBLE BINARY TREE — the same all-reduce, two shapes
  ═══════════════════════════════════════════════════════════════════════════

  RING (N = 8)                          TREE #1 of the double tree (N = 8)
  ────────────                          ──────────────────────────────────
   0 ─▶ 1 ─▶ 2 ─▶ 3                              0
   ▲              │                              └──── 4
   │              ▼                                  ┌─┴─┐
   7 ◀─ 6 ◀─ 5 ◀─ 4                                  2   6
                                                    ┌┴┐ ┌┴┐
   steps  = 2(N−1) = 14                             1 3 5 7
   depth  = 14 hops                        depth = ⌈log₂8⌉ = 3 up + 3 down
   bytes/rank = 2(N−1)/N·S = 1.75 S        every internal node here is a
   → BANDWIDTH-OPTIMAL, latency O(N)       LEAF in tree #2, so both trees
                                           run at once, each carrying S/2
                                           → LATENCY O(log N), bw ≈ ring/2

  WHERE EACH ONE WINS  (NCCL's own compiled-in cost constants,
                        src/tuning/cost_model.cc, values in µs)
  ───────────────────────────────────────────────────────────────────────────
                 base latency, Simple   per-hop latency, NET, Simple
      Ring              8.4 µs                    14.0 µs
      Tree              8.4 µs                    14.0 µs
                 base latency, LL       per-hop latency, NVLINK, Simple
      Ring              6.6 µs                     3.4 µs
      Tree              6.8 µs                     4.0 µs
      (baseLatencies[algo][proto] and hwLatencies[hw][algo][proto],
       proto order LL / LL128 / Simple)

      modelled latency, multi-node all-reduce:
        ring : baseLat + (nSteps − nInterSteps)·intraLat + nInterSteps·interLat
               where nSteps = 2(N−1),  nInterSteps = 2(nNodes−1)
        tree : baseLat + 2·[ (ranksPerNode − 1)·intraLat + log₂(nNodes)·interLat ]
                                                          ▲
                                       THIS is the whole difference: log₂(nNodes)
                                       inter-node hops instead of 2(nNodes−1)

      At 128 nodes:  ring → 254 inter-node hops.  tree → 7.  36× fewer.
      At 128 nodes, tree's inter-node latency term ≈ 2·7·14 µs   =  196 µs
                    ring's                          ≈ 254·14 µs  = 3556 µs
      A 1 MB message at 50 GB/s of wire time is 20 µs — so below a few MB
      per rank, the tree wins outright, and above ~100 MB the ring's 2×
      bandwidth advantage takes over.  NCCL crosses over somewhere between,
      per message size, from exactly this model.
```

**NCCL auto-tunes the crossover per message size and it is usually right.** It evaluates both models for the actual byte count and picks the lower predicted time; you can watch it do so with `NCCL_DEBUG_SUBSYS=TUNING`, which prints one line per collective naming the winner. One structural fact worth knowing: in current NCCL the **tree algorithm is enabled for `AllReduce` only** — `src/tuning/tree.cc` hard-disables it for every other function. So if you are debugging a slow all-gather, "try tree" is not an available lever.

### 6. NVLS, CollNet and SHARP — moving the reduction into the switch

Both ring and tree make GPUs do the arithmetic. **SHARP** (Scalable Hierarchical Aggregation and Reduction Protocol) puts arithmetic logic units in the switch itself, so a reduction happens *in the fabric*. Two independent implementations share the name:

- **NVLink SHARP (NVLS)** — ALUs inside NVSwitch (NVLink 4 and later). Selected as `NCCL_ALGO=NVLS` (single NVLink domain) or `NVLSTree` (NVLS within each node, tree across nodes). NCCL implements it with **multicast groups**: the switch reduces the contributions and fans the result back out, which is why the NVLS setup log line literally reads `NVLS Created Multicast group`.
- **In-network SHARP on InfiniBand switches** — `NCCL_ALGO=CollNet`, `CollnetChain`, `CollnetDirect`, gated behind `NCCL_COLLNET_ENABLE=1` (**default 0**) and `NCCL_COLLNET_NODE_THRESHOLD` (**default 2** nodes). Same name, different silicon, different path.

The theoretical win: an all-reduce needs `2(N−1)` sends per element in a ring but only `N+1` when the switch reduces — asymptotically a 2× improvement in effective bandwidth.

What it delivers, measured. On an 8×H200 node with NCCL 2.27.7 at an 8 GiB payload, running each GPU count twice — once as-is, once with `NCCL_NVLS_ENABLE=0` to force the ring (`stas00/ml-engineering`, `network/README.md`):

| GPUs | algo NCCL picks | busbw | busbw, NVLS off | gain |
|---:|---|---:|---:|---:|
| 4 | Ring | 363.3 GB/s | 363.3 GB/s | 1.00× |
| 5 | NVLS | 414.2 GB/s | 364.4 GB/s | 1.14× |
| 6 | NVLS | 435.6 GB/s | 365.1 GB/s | 1.19× |
| 7 | NVLS | 460.0 GB/s | 365.7 GB/s | 1.26× |
| 8 | NVLS | 473.1 GB/s | 365.6 GB/s | 1.29× |

Two platform lessons here, both operationally useful. **First, it is a ramp, not a cliff** — a partial-node collective gets a partial benefit. **Second, the threshold is not portable**: the same sweep on an 8×B200 AWS `p6-b200.48xlarge` node stays on `Ring` at 5 GPUs and only selects `NVLS` from 6 up, reaching 1.23× at 8 GPUs, because that instance loads AWS's own tuner plugin (`NCCL_TUNER_PLUGIN=ofi`, whose log line reads `base Tuner is chosen for platform: p6-b200.48xlarge`). **Never carry an algorithm-selection threshold from one instance type to another; read it out of the log.**

And note the constraint that catches people: **NVLS implements all-reduce and nothing else.** An FSDP job whose hot collectives are all-gather and reduce-scatter gets nothing from it.

### 7. Protocols — LL, LL128, Simple

`NCCL_PROTO` is a *different axis* from `NCCL_ALGO`, and every tuning log line names both. The protocol decides how a chunk is framed on the wire and how the receiver knows the data has arrived.

- **Simple** — the plain path. Data is written to the peer's buffer, then a memory fence and a separate flag write signal completion. Correct and full-bandwidth, but the fence costs latency, so small messages pay disproportionately.
- **LL (low latency)** — packs each 4-byte payload word with a 4-byte flag word, so the receiver can poll for the flag inline and no fence is needed. **Half the wire bandwidth is flags**, which NCCL's model reflects exactly: the LL bandwidth is `min(llMaxBw, busBw × 0.5)`. Used for tiny messages where latency is everything.
- **LL128** — the compromise: 128-byte lines carrying 120 bytes of payload and 8 bytes of flag, so the overhead is `120/128 = 93.75%` efficiency instead of 50%. NCCL's model uses the factor **0.92** for ring LL128. It depends on 128-byte atomicity guarantees that not every path provides, which is why the docs warn that forcing LL128 on an unsupported platform **can corrupt data**.

The compiled-in ceilings, from `src/tuning/cost_model.cc`, in GB/s:

| Generation | `llMaxBw` (intra-node) | `perChMaxRingLL128Bw` |
|---|---:|---:|
| Volta | 39.0 | 20.0 |
| Ampere | 87.7 | 20.0 |
| Hopper | 141.0 | 36.7 |
| Blackwell | 282.0 | 40.0 |

**Operational rule:** do not set `NCCL_PROTO`. The docs are unusually blunt — "users are discouraged from setting this variable, with the exception of disabling a specific protocol in case a bug in NCCL is suspected." The one legitimate production use is `NCCL_PROTO=^LL128` to rule out an LL128 correctness bug when you have unexplained NaNs on a specific fabric.

### 8. Transports — which physical link a collective actually rides

Once the rings and trees exist, NCCL must connect each pair of ring-neighbours. It ranks the available paths and picks the fastest usable one:

```
  TRANSPORT SELECTION — fastest usable path per rank pair
  ═══════════════════════════════════════════════════════════════════════

    same process / same node?
        │
        ├─ YES ─▶ can the pair do CUDA peer access?
        │            │
        │            ├─ over NVLink ──────────▶ [1] P2P/CUMEM   ~450 GB/s (H100)
        │            ├─ over PCIe   ──────────▶ [2] P2P/direct  ~63 GB/s (Gen5 x16)
        │            │      (gated by NCCL_P2P_LEVEL: LOC<NVL<PIX<PXB<PHB<SYS)
        │            └─ NO ───────────────────▶ [3] SHM/direct  host-memory bounce
        │                                            ← through /dev/shm, ~2 copies
        │
        └─ NO ──▶ is an RDMA verbs device usable for this pair?
                     ├─ YES + GPUDirect RDMA ─▶ [4] NET/IB/<dev>/GDRDMA  ~50 GB/s
                     │                              (NIC DMAs straight to HBM)
                     ├─ YES, no GDR ──────────▶ [4b] NET/IB/<dev>  (bounce via host)
                     └─ NO ───────────────────▶ [5] NET/Socket/<n>  TCP  ~1–5 GB/s
                                                     ▲
                          seeing THIS on an IB fleet is a top-severity bug:
                          an order of magnitude of bandwidth, silently

  The ladder is a PHYSICAL FACT about the machine plus your config.
  A transport lower on this list than you expected is ALWAYS a lead.
```

`NCCL_P2P_LEVEL` is the knob that decides how far apart two GPUs may be and still use direct peer access. Its values are path types, and knowing them lets you read `nvidia-smi topo -m` and predict NCCL's choice:

| Value | Meaning |
|---|---|
| `LOC` | never use P2P |
| `NVL` | only when the pair is connected by NVLink |
| `PIX` | also when on the same PCIe switch |
| `PXB` | also across multiple PCIe switches |
| `PHB` | also when on the same NUMA node (traffic traverses the CPU) |
| `SYS` | also across NUMA nodes / the socket interconnect (UPI) |

Unset, NCCL picks a value from the architecture. The legacy integer form (`LOC=0, PIX=1, PXB=2, PHB=3, SYS=4`) still parses but the docs discourage it because the numbering has changed across releases — and note **`NVL` has no integer equivalent**, so an old `NCCL_P2P_LEVEL=1` in a launch script cannot express "NVLink only."

Two more selectors you will use constantly:

- **`NCCL_SOCKET_IFNAME`** picks the interface for bootstrap *and* for the socket transport. The syntax is a prefix list, with `^` to exclude and `=` to force an exact match: `eth` matches `eth0`, `eth1`, …; `=eth0` matches only `eth0`; `^docker` excludes everything starting with `docker`. By default NCCL already skips `lo` and `docker*` unless nothing else exists, and **prefers interfaces starting with `ib`**. Setting it bypasses the automatic algorithm entirely — which is the point, and also the risk.
- **`NCCL_IB_HCA`** picks the RDMA devices, with entries of the form `<hca>[:<port>[:<rail>[:<plane>]]]`. `=mlx5_0:1,mlx5_1:1` uses port 1 of exactly those two cards. **Always use the `=` prefix**: without it, `mlx5_1` also matches `mlx5_10` through `mlx5_19` if they exist — a genuinely nasty bug on 16-NIC nodes. The optional `rail` and `plane` fields are how you tell NCCL your fabric's rail structure (02b); both default to `-1`, unassigned. There is a hard limit of 32 HCAs.

The companion is **`NCCL_CROSS_NIC`**, which decides whether one ring may enter a node on one NIC and leave on another. **Default is 2** — "prefer the same NIC, but allow crossing if it would be faster." Set it to `0` on a rail-optimised fabric where crossing rails means an expensive spine traversal, and `1` where all NICs on a node hit the same switch anyway.

### 9. Reading `NCCL_DEBUG=INFO` — the actual grammar

`NCCL_DEBUG` takes `VERSION` < `WARN` < `INFO` < `TRACE`. `WARN` is the production minimum; `INFO` is what you set for any investigation. `NCCL_DEBUG_SUBSYS` filters `INFO`/`TRACE` output; its **default is `INIT,BOOTSTRAP,ENV`**, and a leading `^` inverts the list. On multi-process jobs also set `NCCL_DEBUG_FILE=/tmp/nccl.%h.%p.log` (`%h` host, `%p` PID) or the ranks interleave into an unreadable mess.

Here is what each subsystem gives you, with the real line shapes (representative excerpts, shaped exactly as NCCL emits them):

```
  ── VERSION ────────────────────────────────────────────────────────────────
  NCCL version 2.30.3+cuda13.0
    ↳ First thing to capture on any ticket. Feature availability (RAS 2.24+,
      per-function NCCL_ALGO 2.24+, JSON RAS 2.28.7+) hangs off this line.

  ── INIT ───────────────────────────────────────────────────────────────────
  node-01:1895902:1895913 [0] NCCL INFO Initialized NET plugin IB
  node-01:1895902:1895913 [0] NCCL INFO Using network IB
  node-01:1895902:1895913 [0] NCCL INFO DMA-BUF is available on GPU device 0
  node-01:1895902:1895913 [0] NCCL INFO [Rank 0] ncclCommInitRankConfig comm 0x...
      rank 0 nranks 2 cudaDev 0 nvmlDev 0 busId ... commId 0x... - Init START
  node-01:1895902:1895913 [0] NCCL INFO ncclTopoGetCpuAffinity: Affinity for GPU 0 ...
    ↳ host:pid:tid [localdev].  "Using network IB" vs "Using network Socket"
      is the single highest-value line in the whole log.
      Init START on every rank and Init COMPLETE on every rank = healthy
      bootstrap. A subset missing COMPLETE = an INIT-TIME hang.

  ── GRAPH ──────────────────────────────────────────────────────────────────
  node-01:1895811:1895822 [0] NCCL INFO Ring 00 : 1 -> 0 -> 1
  node-01:1895811:1895822 [0] NCCL INFO Ring 01 : 1 -> 0 -> 1
  node-01:1895811:1895822 [0] NCCL INFO Tree 0 : -1 -> 0 -> 1/-1/-1
    ↳ "Ring NN : prev -> me -> next" — one line per channel. COUNT THEM:
      that count is your channel count.  "Tree d : parent -> me -> c0/c1/c2"
      with -1 meaning "none", so -1 -> 0 -> 1/-1/-1 says rank 0 is the tree
      root with one child.  This is where you confirm NCCL SEES the topology
      you think it has.

  ── P2P / SHM / NET  (the transport per hop) ───────────────────────────────
  node-01:631398:631398 [0] NCCL INFO Channel 00/0 : 0[0] -> 1[1] via P2P/CUMEM
  node-01:1896458:1896480 [0] NCCL INFO Channel 00 : 0[0] -> 1[1] via SHM/direct/direct
  node-01:1897709:1897731 [0] NCCL INFO Channel 00/0 : 0[0] -> 1[1] [send] via NET/Socket/0
    ↳ Channel <n>/<sub> : <srcRank>[<dev>] -> <dstRank>[<dev>] via <TRANSPORT>
      P2P/CUMEM  = direct GPU-to-GPU (NVLink or PCIe)      GOOD
      SHM/direct = host-memory bounce                       SUSPECT
      NET/IB/<dev>/GDRDMA = RDMA straight into HBM          GOOD, cross-node
      NET/Socket/<n> = TCP                                  BUG on an IB fleet

  ── TUNING  (which algorithm and protocol won, per size) ───────────────────
  node-01:631871:631871 [0] NCCL INFO AllReduce: 33554432 Bytes -> Algo RING
      proto SIMPLE channel{Lo..Hi}={0..31}
    ↳ <Collective>: <bytes> -> Algo <RING|TREE|NVLS|NVLSTREE|COLLNET*|PAT>
      proto <LL|LL128|SIMPLE> channel{Lo..Hi}={a..b}
      This is the ONE line that answers "ring or tree?" — and the channel
      range tells you how many CUDA blocks are doing it.

  ── COLL  (per-call detail; verbose, use on a short repro only) ────────────
  node-01:1896279:1896279 [0] NCCL INFO AllReduce: opCount 0 sendbuff 0x...
      recvbuff 0x... count 2 datatype 7 op 0 root 0 comm 0x... [nranks=2] stream 0x...
    ↳ opCount is the per-communicator operation counter from §1. Ranks whose
      opCount diverges are the ranks that fell out of lockstep.

  ── NVLS ───────────────────────────────────────────────────────────────────
  node-01:1538174:1538174 [0] NCCL INFO NVLS Creating Multicast group nranks 8 ...
    ↳ Careful: the multicast group is created even at GPU counts where NCCL
      then declines to USE the NVLS algorithm. "Group created" ≠ "SHARP is
      helping you." Confirm with the TUNING line.

  ── DESTROY ────────────────────────────────────────────────────────────────
  node-01:2056033:2056033 [1] NCCL INFO comm 0x... rank 1 nranks 4 cudaDev 1
      busId ... - Destroy COMPLETE
    ↳ A clean teardown. Its ABSENCE on a rank during a shutdown hang is
      the equivalent of a missing Init COMPLETE.
```

The three standard recipes:

```bash
# init-time hang (nothing ever starts stepping)
NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=INIT,BOOTSTRAP,NET \
  NCCL_DEBUG_FILE=/tmp/nccl.%h.%p.log  torchrun ... train.py

# "why is it slow" — topology + transport + algorithm choice
NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=INIT,GRAPH,NET,TUNING \
  NCCL_DEBUG_FILE=/tmp/nccl.%h.%p.log  torchrun ... train.py

# everything, on a small repro only — this is a firehose
NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=ALL NCCL_DEBUG_FILE=/tmp/nccl.%h.%p.log ...
```

### 10. The environment variables you actually debug with

Split them into three groups by intent, because mixing them up is how a diagnostic toggle ends up permanently in a production launch script.

**Group A — observation. Safe in production, changes nothing.**

| Variable | Effect | Default |
|---|---|---|
| `NCCL_DEBUG` | `VERSION` / `WARN` / `INFO` / `TRACE` | unset (silent) |
| `NCCL_DEBUG_SUBSYS` | filter, comma-separated, `^` to exclude | `INIT,BOOTSTRAP,ENV` |
| `NCCL_DEBUG_FILE` | per-process log file, `%h` host `%p` pid | stdout |
| `NCCL_DEBUG_TIMESTAMP_LEVELS` | which levels get timestamps | `WARN` only |
| `NCCL_DEBUG_TIMESTAMP_FORMAT` | strftime, plus `%3f` for ms | `[%F %T] ` |
| `NCCL_TOPO_DUMP_FILE` | write the detected topology XML | unset |
| `NCCL_RAS_ENABLE` | RAS subsystem on/off (2.24+) | **1 (on)** |
| `NCCL_RAS_ADDR` | RAS listener address | `localhost:28028` |

**Group B — configuration. These belong in your launch script, set deliberately, forever.**

| Variable | Effect | Default | When |
|---|---|---|---|
| `NCCL_SOCKET_IFNAME` | bootstrap + socket interface | auto (prefers `ib*`, skips `lo`/`docker*`) | containers with bridge interfaces, multi-homed hosts |
| `NCCL_IB_HCA` | which RDMA devices, `=` for exact, `:rail:plane` | all detected | rail alignment (02b); excluding a storage NIC |
| `NCCL_CROSS_NIC` | may a ring change NIC across nodes | **2** (prefer same, allow crossing) | `0` on rail-optimised fabrics |
| `NCCL_P2P_LEVEL` | max topological distance for P2P | auto | rarely — usually a symptom fix |
| `NCCL_BUFFSIZE` | per-channel per-peer buffer | **4 MiB** | HBM pressure at high channel counts |
| `NCCL_MIN_CTAS` / `NCCL_MAX_CTAS` | channel floor/ceiling | auto | cap NCCL's SM usage; raise for small-message throughput |
| `NCCL_IB_QPS_PER_CONNECTION` | queue pairs per connection | **1** | raise on ECMP/RoCE fabrics for path diversity |
| `NCCL_COLLNET_ENABLE` | allow IB SHARP | **0** | only with a SHARP-configured fabric |

**Group C — hypothesis tests. Set them for one run, then remove them.**

| Variable | Forces | What a change proves |
|---|---|---|
| `NCCL_ALGO=Ring` / `Tree` / `NVLS` … | that algorithm | if pinning `Tree` fixes a stall, one hop on the ring is bad |
| `NCCL_PROTO=^LL128` | disable LL128 | if NaNs vanish, suspect a 128-byte atomicity issue on that path |
| `NCCL_P2P_DISABLE=1` | no direct GPU-GPU; falls back to SHM/net | if a hang clears, suspect NVLink/P2P on that pair |
| `NCCL_IB_DISABLE=1` | no verbs; falls back to **TCP sockets** | if a broken run works (slowly), the IB path is the problem |
| `NCCL_NVLS_ENABLE=0` | no NVLink SHARP | isolates what SHARP contributes to a benchmark |
| `NCCL_SHM_DISABLE=1` | no shared-memory transport | rules out a `/dev/shm` sizing or permissions problem |

**Every variable in group C left set after the incident is a silent, permanent performance regression.** `NCCL_IB_DISABLE=1` forgotten in a Dockerfile means that image runs every future job over TCP. This is not a hypothetical failure mode; it is one of the most common causes of "our new cluster is mysteriously slow."

Since **NCCL 2.24**, `NCCL_ALGO` and `NCCL_PROTO` accept a per-function syntax, which is worth knowing because it appears in vendor-supplied launch scripts:

```
NCCL_ALGO="ring,collnetdirect;allreduce:tree,collnetdirect;broadcast:ring"
   ↑ ring+collnetdirect for everything not listed later, then tree+collnetdirect
     for allreduce specifically, and ring for broadcast.
NCCL_ALGO="allreduce:^tree"    ← everything except tree, for allreduce only
```

Also since 2.24: an unrecognised token is a hard error rather than a silent ignore, and **`ring` is no longer an implicit fallback** — if you specify a set with no valid algorithm for a function, the call fails instead of quietly reverting.

### 11. The silent hang — mechanism, and why utilisation reads 100%

Now the archetype. One rank stops participating: its host OOM-killer reaped the process, its GPU threw an XID, its NIC flapped, its data loader deadlocked on a stuck NFS mount, or the job's `nvidia-fabricmanager` service died. Every other rank has already enqueued the collective. The CUDA kernel implementing that collective is *resident on the SMs* and spinning on a flag in the peer's buffer that will never be written.

That last sentence explains the whole symptom set:

- **`nvidia-smi` shows 100% utilisation.** The utilisation counter measures "was at least one kernel resident in the sampling window", not "was useful work done." A spin-wait is a resident kernel.
- **Power draw is *low*** — typically well under an active training step, because spinning on a flag does not light up the tensor cores. This is the cheapest discriminator you have and it takes one command.
- **Memory stays allocated and flat.** No allocator churn, because no new tensors are being made.
- **The step counter in the application log stops**, and that is the ground truth.
- **No XID.** From module 05's perspective the fleet is healthy, because the GPUs *are* healthy — they are correctly executing a kernel that is correctly waiting.

Eventually PyTorch's watchdog fires. The real message, from `ProcessGroupNCCL::WorkNCCL::checkTimeout`:

```
[PG ID 0 PG GUID 0(default_pg) Rank 12] Watchdog caught collective operation timeout:
WorkNCCL(SeqNum=41207, OpType=ALLREDUCE, NumelIn=3145728, NumelOut=3145728,
         Timeout(ms)=600000) ran for 600041 milliseconds before timing out.
```

Read it field by field, because each field is a clue:

- **`Rank 12`** — the rank that *timed out waiting*. **Not** the rank that left. This is the trap.
- **`SeqNum=41207`** — the per-process-group collective sequence number. Comparing `SeqNum` across ranks is how you find who fell behind: the ranks with the *lower* number are the ones that never got there.
- **`OpType=ALLREDUCE`** — tells you which phase of the step you are in. `ALLGATHER` in an FSDP job means a parameter gather; `ALLREDUCE` in a DDP job means the gradient sync at the step boundary.
- **`NumelIn` / `NumelOut`** — element counts; multiply by dtype size to get the message size, which tells you *which* bucket or which layer.
- **`Timeout(ms)=600000`** — the process group's timeout. **The default is 10 minutes** (`kProcessGroupNCCLDefaultTimeout` in `ProcessGroupNCCL.hpp`), for all collectives, not a separate init/collective pair.

The PyTorch-side settings that shape how this surfaces — all verified in `ProcessGroupNCCL.cpp` at HEAD:

| Variable | Default | Meaning |
|---|---|---|
| `TORCH_NCCL_ASYNC_ERROR_HANDLING` | **3** (`SkipCleanUp`) | already on: the watchdog tears the process down on timeout instead of leaving it wedged. `0` = no handling, `1` = tear down, `2` = clean up only |
| `TORCH_NCCL_TRACE_BUFFER_SIZE` | **2000** | flight recorder ring-buffer entries per rank. **On by default in current PyTorch** — older builds defaulted to 0 |
| `TORCH_NCCL_DUMP_ON_TIMEOUT` | **true** | dump the flight recorder when a timeout or exception fires |
| `TORCH_NCCL_ENABLE_MONITORING` | **true** | a monitor thread kills the process if the *watchdog itself* stops heartbeating |
| `TORCH_NCCL_HEARTBEAT_TIMEOUT_SEC` | **480** (8 min) | how long the monitor waits for a watchdog heartbeat before concluding the watchdog is stuck |
| `TORCH_NCCL_WAIT_TIMEOUT_DUMP_MILSEC` | **15000** | grace period to collect the dump before tearing down |
| `TORCH_NCCL_COORD_CHECK_MILSEC` | **1000** | how often a rank checks the store for a peer's "I am dumping" signal |
| `TORCH_NCCL_DESYNC_DEBUG` | **false** | records collective start/end into the store so a **desync report** can be generated on timeout |
| `TORCH_NCCL_BLOCKING_WAIT` | **false** | makes `work.wait()` synchronous; **disables the watchdog thread entirely** — do not set this on a big fleet |

**Correction worth internalising:** the flight recorder and async error handling are *already on* in current PyTorch. The thing you must opt into is `TORCH_NCCL_DESYNC_DEBUG=1`, and it costs a little per-collective store traffic, which is why it is off.

```
  HANG TIMELINE — one dead rank, 512-GPU job, defaults everywhere
  ═══════════════════════════════════════════════════════════════════════════

  t=0s      rank 47's host OOM-killer reaps the python process.
            No XID. No NCCL error. Nothing is logged by the job.

  t=0s      the other 511 ranks are already inside ncclAllReduce.
            ├─ SM utilisation: 100%   (spin-wait kernel is resident)
            ├─ power draw:     ~35–45% of a training step   ◀── THE TELL
            ├─ HBM allocated:  flat
            └─ step counter:   FROZEN                        ◀── GROUND TRUTH

  t=0s      $25.60/min burning on a 512×H100 job at $3/GPU-hr.
            ─────────────────────────────────────────────────────────────
  t=0..600s  NOTHING HAPPENS. This is the window you are paid to shorten.
            ─────────────────────────────────────────────────────────────
            Tools that work RIGHT NOW, in this window:
              • ncclras                       → names the missing process
              • nvidia-smi power draw         → confirms hang vs slow
              • per-rank step counters        → confirms who stopped first

  t=600s    PyTorch watchdog fires on the 511 survivors.
            "Watchdog caught collective operation timeout: WorkNCCL(
             SeqNum=41207, OpType=ALLREDUCE, ..., Timeout(ms)=600000)"
            ▲ names the WAITERS. Rank 47 is not in this message anywhere.

  t=600s    TORCH_NCCL_DUMP_ON_TIMEOUT=true (default) dumps the flight
            recorder: last 2000 collectives per rank, with SeqNum + stack.
  t=615s    TORCH_NCCL_WAIT_TIMEOUT_DUMP_MILSEC grace expires;
            TORCH_NCCL_ASYNC_ERROR_HANDLING=3 tears the processes down.
  t=~620s   torchrun's agents observe the exits → re-rendezvous (08.5).

  TOTAL BURN if you do nothing: 512 GPUs × ~10.3 min ≈ 88 GPU-hours ≈ $264,
  plus the work lost back to the last checkpoint (08.4).
```

### 12. Telling a dead rank from a straggler — the three instruments

The watchdog message tells you *that* something is wrong and blames the wrong ranks. Three tools tell you *who*.

**(a) RAS — live, non-perturbing, NCCL ≥ 2.24.** Every NCCL process runs a lightweight RAS thread. The threads form their own TCP mesh over the bootstrap/out-of-band interface (not the RDMA fabric, so it does not compete with training traffic) and exchange keep-alives. Any of them will accept a client connection on `localhost:28028` and answer a `STATUS` query by gathering state from the whole job. It ships as the `ncclras` binary, and because the protocol is plain text you can also use netcat:

```bash
ncclras                                  # local, text output
ncclras -h <node> -p 28028 -v            # remote, verbose
ncclras -f json                          # machine-parsable (2.28.7+)
ncclras -m                               # monitoring mode, streams events (2.29+)
echo verbose status | nc localhost 28028 # no binary needed
```

A healthy 32-GPU job answers like this:

```
Job summary
===========

  Nodes  Processes         GPUs  Processes     GPUs
(total)   per node  per process    (total)  (total)
      4          8            1         32       32

Communicators... (0.00s)
=============

Group     Comms     Nodes     Ranks     Ranks     Ranks    Status  Errors
    #  in group  per comm  per node  per comm  in group
    0         8         4         1         4        32   RUNNING      OK
```

A job with a wedged process answers like this — and this is the output that ends the investigation:

```
Communicators... (2.05s)
=============

Group     Comms     Nodes     Ranks     Ranks     Ranks    Status  Errors
    #  in group  per comm  per node  per comm  in group
    0         1         4       7-8        32        32   RUNNING  INCOMPLETE

Errors
======

INCOMPLETE
  Missing communicator data from 1 job process
  Process 3487984 on node 172.16.64.213 managing GPU 5

#0-0 (cf264af53edbe986) INCOMPLETE
  Missing communicator data from 1 rank
  The missing rank: 21
```

Three things to notice. The **query itself took 2.05 s** rather than 0.00 s, because RAS waited on the unresponsive process — slowness of the query is itself a signal. The identifier **`#0-0 (cf264af53edbe986)`** is group-number-dash-communicator-number plus the communicator hash, and *that hash also appears in ordinary `NCCL_DEBUG` output*, so you can join the RAS view to the logs. And after **60 seconds** of failed reconnection attempts RAS promotes the diagnosis from `INCOMPLETE` to `DEAD`:

```
Errors
======

DEAD
  1 job process is considered dead (unreachable via the RAS network)
  Process 3487984 on node 172.16.64.213 managing GPU 5
```

**The straggler case looks different, and the difference is the point.** A rank that is merely slow — MoE routing imbalance (08.1 §8), a lagging data loader, a thermally throttled GPU — still answers RAS. What you see is a `MISMATCH` on collective counts:

```
Group     Comms     Nodes     Ranks     Ranks     Ranks    Status  Errors
    #  in group  per comm  per node  per comm  in group
    0         1         4         8        32        32   RUNNING  MISMATCH

Warnings
========

#0-0 (27a079b828ff1a75) MISMATCH
  Communicator ranks have different collective operation counts
  26 ranks have launched up to operation 6650
  6 ranks have launched up to operation 6649
  Rank 0 -- GPU 0 managed by process 483072 on node 172.16.64.210
  ...
```

**A `MISMATCH` on its own is not a problem.** NCCL collectives easily outpace RAS's own query, so a one-operation spread is normal on a healthy job. The discriminator is *repetition*: **run the query three times, ten seconds apart.**

| Observation across repeated `ncclras` queries | Diagnosis |
|---|---|
| counts advance, spread stays small | healthy — RAS is just sampling a moving target |
| counts advance, the *same* ranks are consistently behind by a stable amount | **straggler.** Hardware is fine; look at load imbalance, data loader, thermals |
| counts do not advance at all | **hang.** The communicator is stuck |
| `INCOMPLETE` / `DEAD` naming a process | **dead rank.** Go to that host |

RAS ≥ 2.31 also exposes a readiness probe, `ncclras -D` (or `echo diagnostics | nc <host> 28028`), which cross-checks GPU inventory, CUDA driver version, volatile ECC counters, NVLink link state, and — usefully — **whether the `NCCL_*` environment variables agree across ranks**. That last check catches an entire bug class: one node's launch wrapper setting `NCCL_IB_DISABLE=1` while the others do not.

**(b) The PyTorch flight recorder — post-mortem, on by default.** Each rank keeps a ring buffer of its last `TORCH_NCCL_TRACE_BUFFER_SIZE` (default 2000) collectives with sequence numbers, sizes, states and Python stacks, dumped automatically on timeout. Where RAS answers "who is missing *now*", the flight recorder answers "what was each rank doing at the moment of death" — including for ranks that have already exited. Correlating `SeqNum` across the dumps gives you the same "who fell behind first" answer, after the fact. If you see `Stack trace of the failed collective not found, potentially because FlightRecorder is disabled`, your build or config has `TORCH_NCCL_TRACE_BUFFER_SIZE=0`.

**(c) The desync report — opt-in, and the most direct answer of the three.** With `TORCH_NCCL_DESYNC_DEBUG=1`, each rank records its collective start/end into the c10d store, so on timeout PyTorch can compute and print who is out of step. The report is generated by `retrieveDesyncReport` in `TraceUtils.h` and reads:

```
 - To our best knowledge, the lagging/dead/mismatched ranks that caused the desync are:
   - [47] joined but didn't finish collective #41207 (count from 1)
     [0, 1, 2, ...] finished collective #41207, but didn't join collective #41208
 - Snapshot of ranks' latest states:
   ...
```

or, when a rank never appeared at all:

```
 - To our best knowledge, ranks [47] are the lagging ranks that caused this timeout.
   They never joined any collectives
```

Note the distinction the two messages draw, which is exactly the straggler-versus-dead distinction: *joined but didn't finish* versus *never joined*. Enabling `TORCH_NCCL_DESYNC_DEBUG` also force-enables `TORCH_NCCL_ENABLE_TIMING`, so it is not free — turn it on for a run you expect to debug.

### 13. The triage sequence

```
  SILENT-HANG TRIAGE — the order matters, each step is ~1 minute
  ═══════════════════════════════════════════════════════════════════════════

  ┌ 1 ─ IS IT A HANG OR JUST SLOW? ────────────────────────────────────────┐
  │  nvidia-smi --query-gpu=index,utilization.gpu,power.draw,memory.used \ │
  │             --format=csv  (on 2–3 suspect nodes)                       │
  │  and: has the application's step counter moved in the last 2 minutes?  │
  │                                                                        │
  │  100% util + LOW power + frozen step counter  ▶ HANG, go to 2          │
  │  100% util + HIGH power + advancing counter   ▶ NOT a hang → 08.3      │
  └────────────────────────────────────────────────────────────────────────┘
                              │
  ┌ 2 ─ WHO IS MISSING? (NCCL ≥ 2.24) ─────────────────────────────────────┐
  │  ncclras            (or: echo status | nc localhost 28028)             │
  │  RUN IT THREE TIMES, ~10 s apart.                                      │
  │                                                                        │
  │  INCOMPLETE/DEAD naming a process ▶ dead rank, go to 3                 │
  │  MISMATCH, counts NOT advancing   ▶ hang, go to 3 with the named ranks │
  │  MISMATCH, same ranks always behind, counts advancing ▶ STRAGGLER,     │
  │                                     go to 4                            │
  │  no RAS (< 2.24) ▶ fall back: grep every rank's log for the highest    │
  │                    step / SeqNum; the lowest is your suspect           │
  └────────────────────────────────────────────────────────────────────────┘
                              │
  ┌ 3 ─ WHY DID THAT RANK LEAVE?  (on the named host) ─────────────────────┐
  │  dmesg -T | tail -50            ▶ "Out of memory: Killed process ..."  │
  │  nvidia-smi -q | grep -i xid    ▶ module 05 territory: 79 fell off the │
  │  journalctl -u nvidia-fabricmanager      bus, 48 double-bit ECC, ...   │
  │  ibstat / rdma statistic        ▶ port not Active; rnr_nak_retry_err   │
  │  the rank's own stdout/stderr   ▶ python traceback, CUDA OOM, loader   │
  └────────────────────────────────────────────────────────────────────────┘
                              │
  ┌ 4 ─ IF NOBODY LEFT: is it TRANSPORT or IMBALANCE? ─────────────────────┐
  │  re-run with NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=INIT,GRAPH,NET,TUNING   │
  │  ▶ any "via SHM" where NVLink was expected?                            │
  │  ▶ any "via NET/Socket" where IB was expected?                         │
  │  ▶ "Using network Socket" instead of "Using network IB"?               │
  │  ▶ does the Ring/Tree line match the placement you asked 06 for?       │
  │  then, one hypothesis at a time:                                       │
  │     NCCL_P2P_DISABLE=1  → clears it? suspect NVLink/P2P on a pair      │
  │     NCCL_IB_DISABLE=1   → clears it? suspect the IB path               │
  │     NCCL_ALGO=Tree      → clears it? suspect one bad hop on the ring   │
  │  REMOVE ALL OF THESE AFTERWARDS.                                       │
  └────────────────────────────────────────────────────────────────────────┘
```

An **init-time** hang is a special case worth calling out, because it looks identical from the dashboard but has a disjoint cause set. If no rank ever prints `Init COMPLETE`, the problem is in bootstrap, not in the fabric: a wrong `NCCL_SOCKET_IFNAME` picking a Docker bridge that peers cannot reach; a firewall blocking NCCL's ephemeral TCP ports (restrict them with `net.ipv4.ip_local_port_range` and open that range); or, on NVSwitch systems, a dead `nvidia-fabricmanager` on a single node, which lets channel setup complete and then stalls forever.

## Perspectives

**Developer / ML-eng view.** Most ML engineers call `all_reduce` or wrap a model in `fully_shard` and expect it to work. Topology discovery, graph search and transport selection happen invisibly at init, and most have never read a `NCCL_DEBUG=INFO` line in their life. That is not a criticism — it is exactly the abstraction boundary that lets them work on models. It is also precisely the gap a platform engineer exists to fill. When their job hangs they see a frozen progress bar; you see a communicator, a sequence number, and a missing process.

**Operator / on-call view.** The whole value of this lesson is triage *speed under pressure*. Reading NCCL docs calmly at your desk and reading them while a 3,000-GPU job burns $80/minute are different skills, and the second is what you are hired for. RAS is the biggest recent improvement to that workflow: what used to be an hour of correlating log streams by hand is now one command that names a process. Knowing the pre-RAS method still matters — plenty of fleets run older NCCL, and the fallback (find the rank with the lowest step count or lowest `SeqNum`) is the same reasoning done manually.

**Hardware / topology view.** The transport ladder — NVLink P2P > PCIe P2P > SHM > NET/IB > NET/Socket — is a statement about what is physically wired to what, not a preference NCCL guesses at. So a "wrong" transport in the logs is *always* diagnostic of a real topology or configuration problem: a GPU pair you believed shared NVLink and doesn't, an HCA that didn't get selected, an interface prefix matching the wrong NIC. Treat every unexpected transport line as a lead, never as noise. The same applies to the algorithm lines: NCCL's choice is a function of the topology it detected, so a surprising algorithm means a surprising topology.

**Failure-mode view.** Collectives are barriers, so NCCL's failure signature is categorically different from an ordinary crash: a **silent stall**, not an exception. Everyone reports the symptom; nobody reports the cause. That asymmetry is the entire intellectual content of this lesson, and it is why triage is a *procedure* rather than a command — you are deliberately working around a system that by design tells you who is stuck but not who left.

**Economics view.** Every element of this lesson converts to money at a fixed rate. The 10-minute default timeout is 10 minutes of full-fleet burn on *every* hang, and it exists because lowering it risks killing healthy jobs that legitimately have a long collective. That is a real trade you may be asked to make: on a fleet with fast checkpointing (08.4) and fast restart (08.5), a shorter timeout is cheap insurance; on one with slow restarts, a spurious timeout costs more than the hang. Knowing that `TORCH_NCCL_ASYNC_ERROR_HANDLING` and the flight recorder are already on, and that `TORCH_NCCL_DESYNC_DEBUG` is the only thing you must add, is the difference between a five-minute diagnosis and a two-hour one.

## Real-world use cases

- **Crusoe Cloud — "NCCL Hangs and Multi-Node Training Stalls Caused by Failed nvidia-fabricmanager."** A GPU-cloud provider's own operational runbook for exactly the "job hangs, no error" scenario. When `nvidia-fabricmanager` dies on even one node of an NVSwitch system, NCCL initialisation stalls indefinitely with no clear error, typically after channel/tree setup has already been logged. Their triage: `journalctl -u nvidia-fabricmanager`, `/var/log/syslog`, and `nvidia-smi nvlink -s` to confirm NVLink topology health. *What it shows:* a root cause outside both the GPU and the network — a **host service** — producing an identical symptom, which is why step 3 of the triage sequence checks systemd units and not just XIDs.

- **Meta — OPT-175B chronicles** (`facebookresearch/metaseq`, `projects/OPT/chronicles`). A primary-source, unsanitised failure diary from a live 992-A100 run: NCCL and InfiniBand errors including data packets lost or corrupted in transit, "GPU is lost" events, and at least 35 manual restarts over roughly two months. *What it shows:* the triage sequence above is not an exercise — this is what a team's multi-week incident log looks like when they do it by hand, before RAS existed.

- **`stas00/ml-engineering` — the NVLS sweeps.** Measured `busbw` for all-reduce across GPU counts on 8×H200 and 8×B200 nodes, with and without `NCCL_NVLS_ENABLE=0`, showing NCCL selecting NVLS above 4 GPUs on one platform and above 5 on the other, and showing an all-reduce reporting 107% of the NVLink unidirectional spec while all-gather on the same node reports 80%. *What it shows:* algorithm selection is platform-specific and vendor-tuner-specific, `busbw` is not always a wire speed, and the only reliable way to know what your fleet does is to read the `TUNING` log line.

- **NCCL's own RAS documentation, the `MISMATCH` example.** The NCCL user guide walks through a status output where 26 ranks have launched up to operation 6650 and 6 ranks up to 6649, and explicitly says this "should not be a cause for concern, as long as the counts increase across repeated queries." *What it shows:* the single most important operational nuance of the tool — a snapshot of a distributed system in motion always looks skewed. The signal is in the *derivative*, which is why the triage sequence says run it three times.

- **`stas00/ml-engineering` — the 4-node vs 1-node all-reduce table.** On P6-B200 nodes with NVLink 5 inside and 8×50 GB/s EFA out, a 16 GiB all-reduce measures 845.67 GB/s `busbw` on one node and 381.80 GB/s across four — a 2.2× slowdown, against an 18× ratio between the link speeds. At 32 KiB the same comparison is 120× slower. *What it shows:* two things at once. Leaving the node is far cheaper than the link ratio suggests, because a hierarchical collective sends only the reduced shard over the slow links and every NIC in the node is in flight. And the effect **inverts completely for small messages**, where latency dominates — which is the strongest possible argument for bucketing gradients (08.1 §3) rather than reducing tensors individually.

## Worked example

**The ticket.** A 64-GPU FSDP run (8 nodes × 8 H100) wedges at step 4,120. Grafana shows all 64 GPUs at 100% utilisation. The loss curve and step counter froze 12 minutes ago. No XID on the health dashboard. The on-call reflex — "GPU util is 100%, it's working" — is wrong, and you know why from §11.

**Step 1 — hang or slow?**

```console
$ pdsh -w gpu-[090-097] 'nvidia-smi --query-gpu=index,utilization.gpu,power.draw \
                          --format=csv,noheader' | head
gpu-090: 0, 100 %, 148.22 W
gpu-090: 1, 100 %, 151.07 W
gpu-090: 2, 100 %, 149.63 W
...
```

148 W on a 700 W H100 SXM. During an actual training step these sit at 500–650 W. **100% utilisation at 21% of TDP is a spin-wait, not compute.** Combined with the frozen step counter: this is a hang. Elapsed: 40 seconds.

**Step 2 — who is missing?**

```console
$ ncclras
Job summary
===========

  Nodes  Processes         GPUs  Processes     GPUs
(total)   per node  per process    (total)  (total)
      8          8            1         64       64

Communicators... (2.11s)
=============

Group     Comms     Nodes     Ranks     Ranks     Ranks    Status  Errors
    #  in group  per comm  per node  per comm  in group
    0         1         8       7-8        64        64   RUNNING  INCOMPLETE

Errors
======

INCOMPLETE
  Missing communicator data from 1 job process
  Process 2214887 on node gpu-093 managing GPU 7

#0-0 (a3f10c9d2e7b4415) INCOMPLETE
  Missing communicator data from 1 rank
  The missing rank: 47
```

Note `Ranks per node: 7-8` — RAS is telling you one node has only seven live ranks. And the query took 2.11 s rather than 0.00 s, because it waited on the dead process. Repeat the query: it now returns fast, with `DEAD` instead of `INCOMPLETE`, because RAS has routed around the process and, after 60 s, declared it dead. Elapsed: 90 seconds. You have gone from "64 GPUs hung" to "global rank 47, GPU 7 on gpu-093, PID 2214887."

**Step 3 — why did rank 47 leave?**

```console
$ ssh gpu-093 'dmesg -T | tail -20'
[Tue Aug 18 04:11:52 2026] python invoked oom-killer: gfp_mask=0x140cca(...), order=0, oom_score_adj=0
[Tue Aug 18 04:11:52 2026] Out of memory: Killed process 2214887 (python)
                            total-vm:412887044kB, anon-rss:198773120kB, ...
$ ssh gpu-093 'nvidia-smi -q | grep -A2 Xid'
    Xid Errors                            : None
```

**Root cause: a host OOM, not a GPU fault.** Rank 47's `DataLoader` worker pool accumulated 189 GiB of resident memory — a prefetch queue with an unbounded depth, growing across the run — and the kernel reaped the process mid-all-gather. Module 05 was correctly silent: the GPU is fine. Module 06 did its job: the gang was placed correctly. The collective barrier turned one host's memory bug into a 64-GPU stall.

**Step 4 — quantify it, because that is what turns the incident into a decision.**

```
  detection with the tooling above     ≈  1.5 min   (what actually happened)
  detection without it (watchdog only) =   10 min   (the default timeout)
  64 GPUs × 8.5 min saved              =   9.1 GPU-hours
  at $3/GPU-hr                         =   $27 per incident
  at ~1 interruption per 3 hours       =   $216/day on this job alone
```

That is the argument for putting `ncclras` in your runbook, and it scales linearly with fleet size: the same reasoning at 16,384 GPUs is roughly **$7,000 per incident**.

**Step 5 — the follow-ups**, in priority order: cap the loader's prefetch and add a per-rank RSS alarm (the actual bug); confirm `TORCH_NCCL_ASYNC_ERROR_HANDLING` is at its default 3 so a hang self-terminates rather than wedging forever; consider lowering the process-group timeout from 600 s now that you know restart is cheap (08.4/08.5 give you the numbers to decide); and add `ncclras` polling to the job's supervisor so the next one is detected in seconds without a human.

**The trap worth naming explicitly:** had you waited for the watchdog, the message would have read `[... Rank 12] Watchdog caught collective operation timeout`, and a plausible-looking investigation of node `gpu-091` would have found a perfectly healthy GPU. **The timeout names a waiter. Always.**

## Practice

**Environment:** one node, **2 rented GPUs**, single node so NVLink or PCIe (and SHM) are the transports in play. Reuse the tiny DDP job from 08.1. If your NCCL is older than 2.24, steps 3(b) and 4 degrade to log correlation — note that in your write-up rather than skipping them.

1. **Capture the topology and the decisions.** Launch with:

   ```bash
   NCCL_DEBUG=INFO \
   NCCL_DEBUG_SUBSYS=INIT,GRAPH,NET,TUNING \
   NCCL_DEBUG_FILE=/tmp/nccl.%h.%p.log \
     torchrun --nproc_per_node=2 train.py
   ```

   From the logs, extract and annotate in your own words: (a) the `NCCL version` line; (b) the `Using network ...` line; (c) every `Channel NN/0 : 0[0] -> 1[1] via ...` line — name the transport and say why that transport and not the one above it in the ladder; (d) count the `Ring NN :` lines, which is your channel count; (e) at least one `TUNING` line, and read off the algorithm, the protocol and the byte count that produced them.

2. **Steer the decisions and diff.** Re-run three more times, changing exactly one thing each time, and diff the logs against the baseline:
   - `NCCL_ALGO=Tree` — the `TUNING` lines should now name `TREE`. Does step time change? On 2 ranks a tree and a ring are nearly the same shape, so predict the answer before you measure.
   - `NCCL_P2P_DISABLE=1` — the `Channel` lines should drop from `P2P/CUMEM` to `SHM/direct`. Record the step-time delta. **This is the memory-bounce penalty, measured.**
   - `NCCL_PROTO=Simple` — compare the `TUNING` lines and step time against the default at whatever message size your model produces.

3. **Manufacture a hang, twice, and diagnose it both ways.**
   (a) Insert `if rank == 1: time.sleep(900)` immediately before an all-reduce. Watch rank 0 pin to 100% utilisation with low power and a frozen step counter. Let the watchdog fire and **record the full message**, then answer: which rank does it blame, and which rank actually slept?
   (b) During the sleep, run `ncclras` three times, ten seconds apart, and capture the output. Does it report `MISMATCH` (rank 1 present but behind — the *straggler* signature, which is what a `sleep` really is) or `INCOMPLETE`? Now repeat with `kill -9` on rank 1 instead of a sleep and capture the output again. **The difference between those two captures is the whole lesson.**

4. **Turn on the desync report.** Re-run the sleep experiment with `TORCH_NCCL_DESYNC_DEBUG=1` and capture the "To our best knowledge, the lagging/dead/mismatched ranks..." block. Compare which of the three instruments (watchdog message, RAS, desync report) gave you the answer fastest and most directly.

5. **(Optional) Get a bandwidth baseline.** Run `all_reduce_perf` from `nccl-tests`, or `all_reduce_bench.py` from `stas00/ml-engineering`, at a large payload. Record `algbw` and `busbw`. Then verify the relationship yourself: `busbw / algbw` should equal `2(N−1)/N = 1.0` for `N=2`. Compare `busbw` against your link's unidirectional spec — 80–88% is healthy.

**Acceptance (feeds "Survive-a-failure"):** a captured `nccl.log` excerpt (15–30 lines) with the **transport identified** (NVLink/PCIe/SHM) and the **algorithm and protocol identified** (ring/tree, LL/LL128/Simple), annotated in your own words; one line showing how the log *changed* under `NCCL_ALGO=Tree` and under `NCCL_P2P_DISABLE=1`, with the step-time delta for the latter; and a short note on the injected hang naming (i) which rank the watchdog blamed, (ii) which rank actually stalled, and (iii) which instrument told you, with the timings. Commit it under [`../practice/survive-a-failure/`](../practice/survive-a-failure/README.md) — this is the "I can read a NCCL log" evidence the deliverable's failure-injection step builds on.

## Common pitfalls

- **"100% GPU utilisation means the GPU is working."** *Mechanism:* `utilization.gpu` samples whether any kernel was resident, and a NCCL collective waiting on a peer is a resident spin-wait kernel. The discriminator is **power draw** — a spinning GPU sits far below its TDP because the tensor cores are idle — plus a frozen step counter. Both are one command away, and skipping them is how teams lose the first ten minutes of every hang.

- **"The rank the watchdog names is the faulty one."** *Mechanism:* the timeout is raised by `WorkNCCL::checkTimeout` on the rank whose *own* work item aged past `Timeout(ms)`. The rank that left never enqueued anything and never times out. Every survivor produces a nearly identical message, so a log search finds dozens of "faulty" ranks and none of them are. Hunt for the rank that is *missing*, not the ones that are complaining.

- **"`NCCL_IB_DISABLE` and `NCCL_P2P_DISABLE` are fixes."** *Mechanism:* they force a slower fallback path, which is diagnostically useful because a behaviour change confirms a hypothesis. Left set, they are permanent regressions — `NCCL_IB_DISABLE=1` moves every collective onto TCP, an order of magnitude of bandwidth, with no error and no alert. Group C of the env-var table exists to make this boundary explicit.

- **"A NCCL hang means a hardware fault."** *Mechanism:* the barrier propagates *any* single-rank stop into a global stall, whatever caused it. The worked example above is a host OOM-killer. The Crusoe runbook is a dead systemd unit. Others: a stuck NFS mount in one rank's data loader, a `nan`-triggered early `return` on one rank, a conditional collective inside `if rank == 0`. Jumping straight to "bad GPU" means you check the one thing module 05 has already told you is fine.

- **"Ring is always right / tree is always faster at scale."** *Mechanism:* ring is bandwidth-optimal (`2(N−1)/N · S` per rank, flat in `N`) but its latency is `2(N−1)` hops; tree is `2·log₂N` hops at roughly half the bandwidth. The crossover is a function of message size, rank count, and the fabric's per-hop latency, and NCCL evaluates both models per call. Pinning `NCCL_ALGO` in production because "tree sounds faster" is a common self-inflicted slowdown — and since 2.24, pinning an algorithm that is invalid for a function makes the call *fail* rather than fall back to ring.

- **"`busbw` is the wire speed."** *Mechanism:* `busbw = algbw × 2(N−1)/N` is derived from a **flat ring** model. Under NVLS the NVSwitch performs the reduction, and under hierarchical algorithms the payload never traverses any single link — so `busbw` can exceed the link spec (measured: 107% on 8×H200). Always read the algorithm off the `TUNING` line before comparing a benchmark number to a datasheet.

- **"`NCCL_IB_HCA=mlx5_1` selects one card."** *Mechanism:* entries are treated as **prefixes** unless you write `=`. On a node with `mlx5_10`…`mlx5_19`, `mlx5_1` silently selects eleven devices. Always write `=mlx5_1`. The same prefix semantics apply to `NCCL_SOCKET_IFNAME`.

- **"A RAS `MISMATCH` means something is broken."** *Mechanism:* RAS gathers state from thousands of processes while they keep issuing collectives, so a one- or two-operation spread is the expected result of sampling a moving system. NCCL's own docs say so. The diagnosis is in the *change* across repeated queries: advancing counts are healthy, static counts are a hang, and consistently-behind-but-advancing ranks are a straggler.

- **"The flight recorder needs turning on."** *Mechanism:* in current PyTorch `TORCH_NCCL_TRACE_BUFFER_SIZE` defaults to **2000** and `TORCH_NCCL_DUMP_ON_TIMEOUT` to **true** — it is already recording, and dumps automatically. Assuming otherwise wastes a run. What genuinely is off by default is `TORCH_NCCL_DESYNC_DEBUG`, which is the one that names the lagging rank in prose.

## Self-check

- **A job hangs at 100% GPU util with no error — what are your first three steps?**
  **Answer:** (1) **Confirm it is a hang, not slow progress.** `nvidia-smi --query-gpu=utilization.gpu,power.draw --format=csv` on two or three suspect nodes: 100% utilisation with power far below TDP (e.g. ~150 W on a 700 W H100) means the GPUs are spinning inside a NCCL kernel, not computing; cross-check that the application's step counter has stopped. (2) **Find who is missing.** On NCCL ≥ 2.24 run `ncclras` (listener defaults to `localhost:28028`) **three times, ten seconds apart** — `INCOMPLETE`/`DEAD` names the unresponsive process and its host and GPU; `MISMATCH` with static collective counts means a hang; `MISMATCH` with advancing counts and the same ranks consistently behind means a straggler, not a failure. Pre-2.24, fall back to comparing the highest step or `SeqNum` across every rank's log; the lowest is your suspect. (3) **Go to that host and find out why it left**: `dmesg -T` for an OOM-kill, `nvidia-smi -q | grep -i xid` and DCGM for module 05 signals, `ibstat` / `rdma statistic` for the NIC, `journalctl -u nvidia-fabricmanager` on NVSwitch systems, and the rank's own stderr. The watchdog message blames the *waiters*; you are hunting the rank that *left*.

- **Derive the ring all-reduce's per-rank byte count, and explain what it implies for scaling data parallelism.**
  **Answer:** Arrange `N` ranks in a ring and split each rank's `S`-byte array into `N` chunks of `S/N`. **Reduce-scatter phase:** in each of `N−1` steps every rank sends one chunk to its successor and accumulates the chunk arriving from its predecessor; after `N−1` steps rank `r` holds the fully reduced chunk `(r+1) mod N` and nobody holds any other chunk completely. **All-gather phase:** `N−1` more steps around the same ring, storing instead of accumulating, until every rank has every finished chunk. Total `2(N−1)` steps × `S/N` bytes per step = **`2(N−1)/N · S` bytes per rank**, and every link carries the identical load so there is no hot spot. Since `2(N−1)/N = 2 − 2/N`, this converges to `2S` and is bounded by it: 1.75 S at `N=8`, 1.969 S at `N=64`, 1.99988 S at `N=16,384`. **So adding data-parallel replicas does not multiply your per-GPU bandwidth bill.** What grows with `N` is the *step count*, `2(N−1)`, hence the latency floor and the synchronisation tail — which is why tree (`2·log₂N` hops) exists and why the reason to leave DDP is memory pressure (08.1), not bandwidth scaling. The same derivation gives the bus-bandwidth correction factor `busbw = algbw × 2(N−1)/N`, and `(N−1)/N` for all-gather and reduce-scatter, which have one phase instead of two.

- **What does `NCCL_DEBUG_SUBSYS=GRAPH` tell you, and what would make you suspicious?**
  **Answer:** `GRAPH` prints NCCL's **topology detection and the communication graph built on top of it** — the discovered PCIe/NVLink/NIC connectivity and the rings and trees NCCL computed across it. The characteristic lines are `Ring NN : prev -> me -> next`, one per channel (count them for your channel count), and `Tree d : parent -> me -> child0/child1/child2` with `-1` meaning "none". You use it to confirm NCCL *sees* the physical topology from 02b correctly and chose sane paths. Suspicious: fewer rings than you expect (NCCL found less usable bandwidth than the hardware has); a ring ordering that crosses node boundaries more often than your placement should require; a tree whose depth does not match `log₂(nodes)`. Cross-check against `nvidia-smi topo -m` and `nvidia-smi topo -p2p n`, and dump the raw detection with `NCCL_TOPO_DUMP_FILE`. A wrong or degraded graph is the usual explanation for a run that is slow rather than broken.

- **Which env var confirms whether NCCL is using InfiniBand or TCP sockets?**
  **Answer:** Two answers, and the order matters. **To *see* what is in use without changing behaviour** — always do this first — read `NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=INIT,NET`: the line `Using network IB` versus `Using network Socket` is definitive, and each per-hop line reads `via NET/IB/<dev>/GDRDMA` versus `via NET/Socket/<n>`. **To *test* it,** `NCCL_IB_DISABLE=1` forces the verbs transport off and falls back to TCP; if a broken run suddenly works, slowly, your IB path was the problem. That second one is a hypothesis test, not a fix — left set it silently costs you an order of magnitude of bandwidth on every future run. Related: `NCCL_IB_HCA` selects which HCAs (use the `=` prefix for exact matches, or `mlx5_1` will also match `mlx5_10`–`mlx5_19`), and `NCCL_SOCKET_IFNAME` selects the bootstrap/socket interface.

- **Why is RAS a bigger deal than it sounds, and how do you avoid misreading it?**
  **Answer:** Before NCCL 2.24 a silent hang gave you nothing to query — you grepped and correlated every rank's log by hand to find who stopped incrementing first, which at thousands of ranks takes an hour. RAS runs a lightweight thread per process; the threads form a TCP mesh over the **out-of-band** interface (so it does not contend with RDMA training traffic) and exchange keep-alives. Any process's listener (`localhost:28028` by default) answers a `STATUS` query, gathering state from the whole job **without perturbing it**, via the `ncclras` client or plain netcat. It prints a job summary, a per-communicator-group table, and — the payoff — an `Errors` section that names the unresponsive process by PID, host and GPU (`INCOMPLETE`, promoted to `DEAD` after 60 s of failed reconnects). **The misreading to avoid is `MISMATCH`.** Collectives easily outrun RAS's own gather, so a snapshot showing "26 ranks at operation 6650, 6 ranks at 6649" is normal; NCCL's docs say so explicitly. Repeat the query: advancing counts are healthy, static counts are a hang, and the same ranks persistently behind by a stable amount is a **straggler** — a different fix (load balance, thermals, data loader) than a restart.

- **You see `via SHM` in the NCCL init log for a GPU pair you expected to use NVLink. Is this noise?**
  **Answer:** No — never. The transport ladder (P2P over NVLink > P2P over PCIe > SHM > NET/IB > NET/Socket) reflects a physical fact about the machine plus your configuration, so falling further down it than expected is always diagnostic. `SHM/direct` means NCCL could not establish CUDA peer access between that pair and is bouncing every byte through pinned host memory — two extra copies and PCIe-class bandwidth instead of NVLink. Causes, in order of likelihood: `NCCL_P2P_DISABLE=1` left over from a past investigation; `NCCL_P2P_LEVEL` set too restrictively (`LOC` or `NVL` on a pair that is only PCIe-connected); the GPUs genuinely not being NVLink-adjacent on this board; peer access blocked by the container or IOMMU configuration; on NVSwitch systems, a failed `nvidia-fabricmanager`. Confirm with `NCCL_DEBUG_SUBSYS=GRAPH` to see the topology NCCL discovered, then cross-check `nvidia-smi topo -m` and `nvidia-smi topo -p2p n`.

- **When does NCCL pick tree over ring, and what does that cost?**
  **Answer:** NCCL models both and picks the lower predicted time for the actual byte count. Ring cost is `2(N−1)·α + 2(N−1)/N · S/B`; tree cost is `~2·log₂(N)·α + S/(B/2)`. So **tree wins when the latency term dominates** — small messages, large rank counts, high per-hop latency — and **ring wins when the bandwidth term dominates**, i.e. large messages. Concretely, NCCL's compiled-in model for a multi-node all-reduce uses `2(nNodes−1)` inter-node hops for ring against `log₂(nNodes)` for tree: at 128 nodes that is 254 versus 7. The cost of tree is bandwidth: its modelled bandwidth is half the ring's, because internal nodes serialise their children's traffic. NCCL recovers much of that by building a **double binary tree** — two complementary trees where every rank that is internal in one is a leaf in the other, each carrying half the data, so all links are busy in both directions while depth stays `log N`. Two practical constraints: the tree algorithm is enabled for **AllReduce only** in current NCCL, and forcing `NCCL_ALGO` in production overrides a tuner that is usually right.

- **What is the difference between `NCCL_ALGO` and `NCCL_PROTO`, and why should you leave the latter alone?**
  **Answer:** `NCCL_ALGO` selects the **communication pattern** — `Ring`, `Tree`, `CollnetDirect`, `CollnetChain`, `NVLS`, `NVLSTree`, `PAT` — i.e. who sends what to whom, in what order. `NCCL_PROTO` selects the **wire framing** for each chunk. `Simple` writes data then a separate flag behind a memory fence: full bandwidth, higher latency. `LL` interleaves a 4-byte flag with every 4-byte payload word so the receiver can poll inline with no fence — half the wire is flags, and NCCL's model caps it at `min(llMaxBw, busBw × 0.5)`. `LL128` uses 128-byte lines carrying 120 bytes of payload, i.e. ~94% efficiency (NCCL models it at 0.92), but it depends on 128-byte write atomicity that not every path provides. That is why the docs say users are discouraged from setting `NCCL_PROTO` at all, and warn that forcing LL128 where it is unsupported **can corrupt data**. The one legitimate use is `NCCL_PROTO=^LL128` to rule out an LL128 bug when chasing unexplained NaNs on a specific fabric. Both appear together on every `TUNING` line: `AllReduce: 33554432 Bytes -> Algo RING proto SIMPLE channel{Lo..Hi}={0..31}`.

## Connections & what's next

You now own the module's anchor skill: deriving what a collective costs, reading NCCL's transport and algorithm decisions out of its own logs, and triaging a silent hang from "3,000 GPUs stuck" down to one named process on one named host.

**08.3** takes the same machinery and asks a throughput question instead of a failure question. The `2(N−1)/N · S / B` bandwidth term and the `2(N−1)·α` latency term derived here become the communication half of a step-time model; MFU is what you get when you divide the compute half by the whole. The measured `busbw` numbers in §4 and §6 are the `B` that model needs, and the "80–88% of spec is healthy" calibration is what stops you chasing a phantom.

**08.4** needs the same instinct applied to storage instead of the fabric: a checkpoint write is also a collective (the metadata exchange is a real barrier), and the same "one slow participant sets the pace" logic decides whether your checkpoint takes 15 seconds or two minutes. **08.5** picks up directly where §13 ends — you have localised the dead rank, and now something has to drain it, re-form the world, and resume; the watchdog and flight-recorder settings tabulated in §11 are exactly the ones that decide whether a hang becomes a restartable crash or an eternal spin.

Backward: **02b** supplied the link bandwidths every calculation here rests on and the rail structure that `NCCL_IB_HCA`'s `:rail:plane` fields express; **05** supplied the XID and DCGM signals that step 3 of the triage sequence checks; **06** placed the gang whose ring ordering you verified in the `GRAPH` output; **08.1** supplied the collectives and their message sizes.

## References & further reading

**Primary sources**

1. **NCCL user guide — environment variables** (`docs/userguide/source/env.rst` in the `NVIDIA/nccl` repository) — <https://github.com/NVIDIA/nccl>. **Read in full once, then keep it open on an incident.** Source of every default in §2, §8 and §10: `NCCL_BUFFSIZE` = 4 MiB, `NCCL_NTHREADS` = 512, `NCCL_DEBUG_SUBSYS` default `INIT,BOOTSTRAP,ENV`, `NCCL_CROSS_NIC` = 2, `NCCL_COLLNET_ENABLE` = 0, `NCCL_COLLNET_NODE_THRESHOLD` = 2, `NCCL_IB_QPS_PER_CONNECTION` = 1, the `NCCL_P2P_LEVEL` path types, the `NCCL_IB_HCA` `<hca>:<port>:<rail>:<plane>` grammar and its 32-device limit, the `NCCL_ALGO`/`NCCL_PROTO` value lists including the 2.24+ per-function syntax, and `NCCL_RAS_ENABLE`/`NCCL_RAS_ADDR`. *`docs.nvidia.com` is blocked by this environment's egress proxy; verified against the in-repo reStructuredText at NCCL v2.31.2.*

2. **NCCL user guide — troubleshooting: logging** (`docs/userguide/source/troubleshooting/logging.rst`, same repository). **Read in full.** Source of the per-subsystem log-line formats reproduced in §9 — `Ring NN : prev -> me -> next`, `Tree d : parent -> me -> c0/c1/c2`, `Channel NN/0 : 0[0] -> 1[1] via P2P/CUMEM`, `via SHM/direct/direct`, `via NET/Socket/0`, and the `TUNING` line `AllReduce: <bytes> -> Algo RING proto SIMPLE channel{Lo..Hi}={0..31}` — plus the subsystem table and the `NCCL_DEBUG_FILE` `%h`/`%p` recipe. Verified in-source.

3. **NCCL user guide — troubleshooting: RAS** (`docs/userguide/source/troubleshooting/ras.rst`, same repository). **Read in full; it is the highest-value page in this lesson.** Source of the RAS architecture (per-process threads over the out-of-band network), the `ncclras` client and its `-h/-p/-v/-t/-f/-m/-D` options, the netcat protocol (`STATUS`, `VERBOSE STATUS`, `TIMEOUT`, `DIAGNOSTICS`), and the exact `Job summary` / `Communicators` / `Errors` / `Warnings` output blocks quoted in §12 including `MISMATCH`, `INCOMPLETE`, the 60-second promotion to `DEAD`, JSON output (2.28.7+), monitoring mode (2.29+) and diagnostics (2.31+). **Correction:** the previous version of this lesson showed a RAS output of the form `NOT RESPONDING: global rank 47` — no such line exists. The real diagnosis is an `Errors` block naming the process by PID, host and GPU, plus `The missing rank: <n>`; this lesson now reproduces the documented format.

4. **NCCL user guide — troubleshooting: performance and tuning, and networking** (same repository). Source of the `nvbandwidth` / `nvloom` / `ib_write_bw` / `ib_write_lat` / `rdma statistic` / `mlxlink` diagnostic ladder referenced in §13, the `nvidia-smi topo -m` and `nvidia-smi topo -p2p n` cross-checks, the IMEX/`nvidia-imex-ctl` path for MNNVL systems, and the `net.ipv4.ip_local_port_range` firewall recipe for init-time hangs. Verified in-source.

5. **NCCL source — `src/tuning/tuning_general.cc`, `src/tuning/ring.cc`, `src/tuning/tree.cc`, `src/tuning/cost_model.cc`** (same repository). **Read `tuning_general.cc` and `cost_model.cc`; they are short and they are the model.** Source of `ncclTuningGetNsteps` returning `2(nRanks−1)` for all-reduce and `nRanks−1` for all-gather/reduce-scatter; the ring latency model with `nInterSteps = 2(nNodes−1)` and the tree's `2·[(ranksPerNode−1)·intraLat + log₂(nNodes)·interLat]`; the tree's `busBw × 0.5` bandwidth halving and its hard-disable for every function except all-reduce; the LL `× 0.5` and LL128 `× 0.92` protocol factors; and the compiled-in base/hardware latency and `llMaxBw` / `perChMaxRingLL128Bw` tables reproduced in §5 and §7. Verified in-source at v2.31.2.

6. **NCCL source — `src/graph/trees.cc`** (same repository). Source of the double-binary-tree construction described in §5: `ncclGetBtree` builds an alternating leaf/node binary tree by bit arithmetic on the rank index, and `ncclGetDtree` builds the complementary second tree (mirrored for even `N`, shifted by one for odd) so that every rank internal in one tree is a leaf in the other. Verified in-source.

7. **nccl-tests — `doc/PERFORMANCE.md`** — <https://github.com/NVIDIA/nccl-tests>. **Read in full; it is two pages and it is the definition of `busbw`.** Source of the `algbw = S/t` and `busbw = algbw × 2(n−1)/n` derivation for all-reduce, `(n−1)/n` for all-gather and reduce-scatter, and `busbw = algbw` for broadcast and reduce, including the argument that `2(n−1)` data transfers per element are required by *any* point-to-point algorithm, not just the ring. Verified by cloning at HEAD.

8. **PyTorch — `torch/csrc/distributed/c10d/ProcessGroupNCCL.cpp` and `.hpp`** — <https://github.com/pytorch/pytorch>. **Read the constructor and `HeartbeatMonitor`.** Source of the watchdog message text and its `WorkNCCL(SeqNum=…, OpType=…, NumelIn=…, NumelOut=…, Timeout(ms)=…)` format, the `[PG ID … PG GUID … Rank …]` log prefix, and every default in §11: `kProcessGroupNCCLDefaultTimeout` = **10 minutes**, `TORCH_NCCL_ASYNC_ERROR_HANDLING` = 3, `TORCH_NCCL_TRACE_BUFFER_SIZE` = 2000, `TORCH_NCCL_HEARTBEAT_TIMEOUT_SEC` = 480, `TORCH_NCCL_WAIT_TIMEOUT_DUMP_MILSEC` = 15000, `TORCH_NCCL_COORD_CHECK_MILSEC` = 1000, `TORCH_NCCL_DUMP_ON_TIMEOUT` = true, `TORCH_NCCL_ENABLE_MONITORING` = true. **Corrections:** the previous version of this lesson gave "default 10 min for init, 30 min for collectives" and quoted a 1,800,000 ms timeout — there is one default and it is 600,000 ms; and it advised turning on async error handling and the flight recorder, both of which are already on by default in current PyTorch. *`docs.pytorch.org` is blocked by this environment's egress proxy; verified against the C++ at HEAD.*

9. **PyTorch — `torch/csrc/distributed/c10d/TraceUtils.h`** (same repository). Source of the desync report in §12: `retrieveDesyncReport`, `analyzeMissingRanks` ("ranks [...] are the lagging ranks that caused this timeout. They never joined any collectives") and `analyzeLaggingRanks` ("[...] joined but didn't finish collective #N" / "finished collective #N, but didn't join collective #N+1"), gated on `TORCH_NCCL_DESYNC_DEBUG`. Verified in-source.

**Real-world engineering blogs and reports**

10. **`stas00/ml-engineering` — `network/README.md`** — <https://github.com/stas00/ml-engineering>. **Read the "Unidirectional vs Bidirectional", "SHARP" and "Inter-node speed depends on intra-node speed" sections in full.** Source of the measured NVLS sweeps on 8×H200 and 8×B200 in §6, the 480.0 / 367.2 / 361.4 / 362.9 GB/s collective comparison at 16 GiB in §4, the "80–88% of unidirectional spec is normal" calibration, and the 1-node vs 4-node P6-B200 all-reduce table (845.67 → 381.80 GB/s at 16 GiB, 2.2×; 120× at 32 KiB) cited in Real-world use cases. Practitioner measurements, clearly dated and reproducible, with the exact commands. Verified by cloning at HEAD.

11. **`stas00/ml-engineering` — `network/debug/README.md`**, with `network/benchmarks/all_reduce_bench.py` and `debug/torch-distributed-gpu-test.py` (same repository). **Skim the guide; use the two scripts.** A field guide for diagnosing NCCL and network issues, and the two hands-on companions for this lesson's Practice — one for a quick bandwidth number, one for confirming every GPU in a cluster can talk to every other. Verified by cloning at HEAD.

12. **Crusoe Cloud Support — "NCCL Hangs and Multi-Node Training Stalls Caused by Failed nvidia-fabricmanager"** — <https://support.crusoecloud.com/hc/en-us/articles/46061806112155-NCCL-Hangs-and-Multi-Node-Training-Stalls-Caused-by-Failed-nvidia-fabricmanager>. **Skim.** A GPU-cloud provider's live operational runbook for a "hang, no error" root cause that is neither a dead rank nor a bad NIC but a dead host service — read it as a second data point on the archetype. *Not fetched in this pass; the host is not reachable through this environment's egress proxy, so it is cited as a pointer rather than relied on for any specific number above beyond the triage steps it is credited with.*

13. **Meta — OPT-175B chronicles** — <https://github.com/facebookresearch/metaseq/tree/main/projects/OPT/chronicles>. **Skim the logbook.** Real, unsanitised NCCL and InfiniBand errors, "GPU is lost" events, and restart counts from a live 992-GPU run — what manual hang triage looked like before RAS existed.

**Deeper dives**

14. **"Mycroft: Tracing Dependencies in Collective Communication Towards Reliable LLM Training"** — <https://arxiv.org/abs/2509.03018>. An academic treatment of the exact "one dead rank hangs everyone" localisation problem, reported as deployed in production for six-plus months — where hang detection is heading past RAS. *`arxiv.org` is blocked by this environment's egress proxy; listed as optional depth and **not relied upon** for any claim in this lesson.*
