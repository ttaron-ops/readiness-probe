---
lesson: "08.3"
title: "Communication as the bottleneck"
module: "08"
concept: "Communication as the bottleneck"
status: not-started
est_time: "5h"
artifacts: []
---

# 08.3 · Communication as the bottleneck

> **Concept.** At scale a training step is bounded by interconnect bandwidth and collective latency, not FLOPs — and MFU is the number that tells you how much of the GPU you actually bought.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Why this matters

You already sold the org 16,384 H100s at ~$2–4/GPU-hr. The finance question is not
"how fast is an H100" — module 03 answered that (roofline, ~990 TFLOP/s BF16 peak).
The question is **what fraction of that peak the cluster actually delivers on a real
training step**, because you pay for the peak and bill against the delivered. On
frontier 3D-parallel runs that fraction — MFU — sits around **35–40%**. The other
60% is money you rented and did not convert into gradient updates, and the single
largest recoverable chunk of it is **communication stall**: GPUs sitting idle inside
a collective waiting for bytes to cross a link. If you can name why MFU is 35% and
not 20%, and which of the three usual causes is biting this job, you are doing the
part of this role that an ML engineer usually cannot.

## What's new here (the platform view)

Module 03 gave you two regimes on the roofline: **compute-bound** (you're on the flat
FLOP ceiling) and **memory-bound** (you're on the HBM-bandwidth slope). Distributed
training adds a third axis the roofline doesn't draw: **communication-bound**, where
the step time is set by an *off-chip* link — NVLink inside a node, InfiniBand/RoCE
between nodes — not by HBM or the SMs at all.

The platform lens: a training step is `compute + communication`, and the two overlap
only partially. Data parallelism must **all-reduce the full gradient every step**.
Tensor parallelism must **all-reduce activations twice per transformer layer**. As
you add GPUs (world size `N`) and as the model grows (message size `M`), the collective
term grows on both axes while the per-GPU compute term stays fixed. Past some scale the
collective *is* the step. Your job is to keep collectives cheap by placing ranks on the
fastest available link — which is exactly the rail-alignment problem from 02b, now with
a throughput number attached.

## Core notes

### The 6N rule and the MFU definition

For a dense transformer, the forward+backward compute is very close to **6 × N_params
FLOPs per token** (2 for the forward matmuls, ~4 for the backward — the "6N" heuristic;
add activation checkpointing recompute and it's ~8N, but use 6N as the yardstick).

**MFU (Model FLOPs Utilisation)** is the achieved fraction of aggregate peak:

```
            model_FLOPs_per_token × tokens_per_second
MFU  =  ───────────────────────────────────────────────
              num_GPUs × peak_FLOP/s_per_GPU

     ≈   (6 · N_params · throughput_tokens_s) / (N_gpu · P_peak)
```

It is deliberately *model* FLOPs (the useful 6N), not *hardware* FLOPs — so recompute
from activation checkpointing does **not** get counted as useful work, which is why MFU
is always below "how busy are the SMs." HFU (Hardware FLOPs Utilisation) counts recompute
and is a few points higher; when a paper quotes one number it's usually MFU. Llama 3 405B
reports **38–43% BF16 MFU** on the 16K-H100 cluster — that is the frontier bar, not a
disappointment.

### Why the collective grows with world size

The canonical bandwidth-optimal collective is **ring all-reduce**. Each of `N` ranks
sends its shard around the ring in `2(N-1)` steps (reduce-scatter then all-gather). The
cost model:

```
T_allreduce(ring)  ≈  2(N-1)·α   +   2·(N-1)/N · (M / B)
                       └────┬───┘      └──────┬────────┘
                     latency term        bandwidth term
```

- `α` = per-hop latency, `M` = gradient bytes, `B` = per-link bandwidth.
- **Bandwidth term** → `2M/B` as `N` grows: the bytes each GPU pushes asymptote to `2M`,
  *independent of N*. So ring all-reduce does **not** blow up the data volume with world
  size — a common misconception. What it does do is send `2M` bytes **every step**, and
  `M` grows with the model.
- **Latency term** → `2Nα`, **linear in N**. At 16K ranks, with hierarchical topologies
  (intra-node NVLink hop cheap, inter-node fabric hop expensive) and one slow straggler
  anywhere on the ring, the synchronization tail dominates. Every rank waits for the
  slowest link on the ring — this is why *one* bad NIC or one misplaced gang tanks the
  whole step.

So the honest statement: all-reduce **data volume** grows with model size `M`; all-reduce
**latency/synchronization** grows with world size `N`; and you pay it every single step.
NCCL mitigates with tree/hierarchical algorithms (latency `~log N`) and NVLS/SHARP
in-network reduction, but the shape stands: more ranks + bigger model = collective creeps
toward owning the step. See lesson 08.2 for how NCCL picks the algorithm.

### Rail alignment is an MFU lever (tie to 02b)

Tensor-parallel ranks all-reduce **twice per layer** — the most bandwidth-hungry
collective in the stack — so TP is only ever placed **inside one NVLink domain** (the 8
GPUs on a node's NVSwitch, ~900 GB/s all-to-all). Data-parallel all-reduce, once per
step, tolerates the slower inter-node fabric (~400 Gb/s = 50 GB/s per direction on a
400G rail).

From 02b: if a TP=8 gang is scheduled **4+4 split across two nodes**, its per-layer
all-reduce now traverses InfiniBand instead of NVLink. A collective runs at the speed of
its **slowest link**, so an ~18× slower link on the hot path means the TP all-reduce goes
from "hidden under compute" to "the dominant term," and MFU collapses — a job that should
run at 38% MFU lands at 12–15%. On the roofline (03) you've been shoved off the compute
ceiling onto a communication wall that the roofline doesn't even plot. The fix is
placement, not tuning: keep TP ranks rail-aligned and NVLink-local, which is a scheduler/
topology constraint you own, not a hyperparameter the ML team owns.

### The three usual causes of low MFU

When MFU is below the ~35% bar, it is almost always one of three, and module 05
observability tells you which:

1. **Comms** — GPUs idle inside NCCL kernels. Signature: high `nccl` kernel time, SM
   occupancy dips during collectives, exposed (non-overlapped) all-reduce. Cause: bad
   placement (rail split), too-small per-GPU batch (compute doesn't cover the collective),
   or wrong parallelism split. This is the recoverable one.
2. **Data-loader / input starvation** — GPUs idle waiting for the *next batch*, not the
   network. Signature: periodic SM idle at step boundaries, host CPU or storage pegged,
   `DataLoader` workers saturated. Cause: too few loader workers, slow object store, no
   prefetch. Covered in 08.8.
3. **Failure overhead** — the run is up but bleeding time to restarts, rework since the
   last checkpoint, and slow-node detection. This is the module's spine and the subject of
   08.4/08.5: on a ~3-hr-MTBF cluster this term alone can eat >10% of wall-clock if
   checkpoint/restart is slow. MFU measured over a long window silently absorbs it.

MFU is the report card; these three are the line items. A good incident note says "MFU
dropped 38%→22% at 14:03, comms term doubled, cause = DP gang rescheduled off-rail after
a node drain," not "training got slow."

## Worked example

Ballpark an MFU for a data-parallel job and show comms is the swing factor.

- Model: 7B params. GPUs: 8× H100, `P_peak ≈ 990 TFLOP/s` BF16 each → aggregate
  `7.92 PFLOP/s`.
- Measured throughput: `12,000 tokens/s` aggregate.

```
useful FLOP/s = 6 · N_params · tokens_s = 6 · 7e9 · 12,000 = 5.04e14 = 504 TFLOP/s
MFU = 504 / 7,920 = 0.064  → 6.4%
```

6.4% is alarmingly low — this job is stalling. Now the diagnostic: gradient size for a
7B BF16 model is `~14 GB`. On a single 8-GPU node over NVLink, the DP all-reduce moves
`~2M = 28 GB` at ~450 GB/s effective → **~62 ms per step** of pure comms. If the compute
per step is only ~40 ms, the collective is **exposed** — it's longer than the compute it's
supposed to hide behind. Fix: raise per-GPU batch (more compute to overlap the fixed
collective), or shard optimizer state (ZeRO/FSDP) so each rank all-gathers less at a time.
Re-measure; a healthy same-node 7B DDP job should clear **35–40% MFU**. The point: the FLOP
number never moved — MFU moved entirely on the communication term.

## Practice

Feeds the **Survive-a-failure lab** deliverable. Rent **2 GPUs** (same node if you can get
NVLink; note it if you can't) and run the DDP job from lesson 08.1.

1. Run the identical job on **1 GPU**, then on **2 GPUs** (DDP), same per-GPU batch size.
   Log **step time** (median over ≥100 steady-state steps, discard warmup) and
   tokens/s for each.
2. Compute **scaling efficiency**:
   ```
   eff = throughput_2gpu / (2 × throughput_1gpu)
   ```
   Ideal linear scaling = 1.0. Real DDP on one node lands ~0.85–0.95; across a slow link,
   much less.
3. Estimate **communication overhead**: `comms_fraction ≈ 1 − eff`. Cross-check against
   the collective cost model — with `M` = your model's gradient bytes and `B` your measured
   link bandwidth (from 08.2 / `nccl-tests all_reduce_perf`), does `2M/B` roughly match the
   per-step slowdown you observed?

**Acceptance:** a written **scaling-efficiency number + a comms-overhead estimate** (with
the 1-GPU and 2-GPU step times behind it), committed under `../practice/survive-a-failure/`.
One paragraph: is this job compute-bound or comms-bound at 2 GPUs, and what would you expect
at 8?

## Self-check

**(a) Why does all-reduce cost grow with world size?**
**Answer:** Split the collective into two terms. The **bandwidth term** (`2(N-1)/N · M/B`)
actually *asymptotes* — each GPU pushes ~`2M` bytes regardless of `N`, so data volume is
set by model size, not world size. What grows with `N` is the **latency/synchronization
term**: ring all-reduce takes `2(N-1)` steps (`~2Nα`, linear in N; tree algorithms cut it
to `~log N`), and every rank must wait for the slowest link/straggler on the collective —
more ranks means more hops and a fatter tail. And you pay this cost *every step*. So at
scale the collective creeps toward owning the step even though the per-GPU compute is
constant.

**(b) What MFU would concern you, and what are the three usual causes of low MFU?**
**Answer:** The frontier bar for 3D-parallel H100 training is **~35–40% MFU** (Llama 3
reports 38–43%). Below **~30%** I'd open an investigation; below ~20% something is clearly
broken. The three usual causes: **(1) comms** — GPUs idle inside NCCL collectives, usually
bad placement (rail split), too-small per-GPU batch, or exposed all-reduce; **(2)
data-loader** — input-pipeline starvation, GPUs waiting on the next batch from CPU/storage;
**(3) failure overhead** — restarts, rework since last checkpoint, and slow-node detection,
which a long-window MFU silently absorbs. Module 05 observability tells you which of the
three is biting.

**(c) Why does keeping tensor-parallel ranks in one NVLink domain matter for MFU (tie to
02b)?**
**Answer:** TP all-reduces activations **twice per transformer layer** — the highest-
frequency, highest-bandwidth collective in the run. Intra-node NVLink is ~900 GB/s; inter-
node InfiniBand/RoCE is ~18× slower. A collective runs at the speed of its slowest link, so
a TP=8 gang split **4+4 across two nodes** (the 02b rail-alignment failure) drags every
per-layer all-reduce onto the fabric, turning a collective that *hid under compute* into
the dominant term. MFU collapses from ~38% to ~12–15% — off the roofline's compute ceiling
onto a communication wall. It's a placement/scheduling fix you own, not a hyperparameter.

## Resources

1. **Llama 3 paper, §3.3 "Infrastructure, Scaling, and Efficiency"** —
   https://arxiv.org/abs/2407.21783 — the real efficiency numbers: 16K H100s, 38–43% MFU,
   4D parallelism, and why placement/comms dominate at scale. The anchor.
2. **Megatron-LM README** — https://github.com/NVIDIA/Megatron-LM — context for 3D/4D
   parallelism (TP/PP/DP/CP) and where the MFU yardstick comes from; skim the parallelism
   sections, don't drown in the flags.
