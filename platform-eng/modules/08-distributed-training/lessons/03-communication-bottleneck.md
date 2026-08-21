---
lesson: "08.3"
title: "Communication as the bottleneck"
module: "08"
concept: "communication-bottleneck"
status: not-started
est_time: "7h"
prev: "02-nccl-collectives.md"
next: "04-checkpointing.md"
artifacts: []
sources: 12
---

# 08.3 · Communication as the bottleneck

> **Concept.** At scale a training step is bounded by interconnect bandwidth and collective latency, not FLOPs — and MFU is the number that tells you how much of the GPU you actually bought.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Where this fits

08.1 gave you the byte counts: which collective each parallelism strategy issues and how many bytes it carries per step. 08.2 gave you the cost of moving those bytes: the ring's `2(N−1)/N · S` per-rank volume, the `2(N−1)` step count, the bus-bandwidth definition, and — crucially — *measured* bandwidths rather than datasheet ones.

This lesson multiplies them together and divides by the compute. You get two things out of that: **a step-time model** you can evaluate on paper before a job runs, and **MFU**, the single number that tells finance what fraction of the hardware you rented was converted into gradient updates. It is the diagnostic layer on top of 08.2's mechanics: 08.2 tells you *why* a collective costs what it costs; this lesson tells you *whether that cost is eating your job* and which of three causes to go fix.

08.4 is next, and it takes the third of those three causes — failure overhead — and gives it the same treatment: a formula that turns two measured numbers into a policy decision.

## Why this matters

You have already sold the org 16,384 H100s. The finance question is not "how fast is an H100" — module 03 answered that with the roofline and the ~989 TFLOP/s BF16 dense peak. The question is **what fraction of that peak the cluster actually delivers on a real training step**, because you pay for the peak and bill against the delivered.

On frontier 3D-parallel dense-transformer runs that fraction — MFU — sits around **35–43%**. The other ~60% is money you rented and did not convert into gradient updates. Some of that gap is unrecoverable physics; a meaningful chunk is not.

Put a number on the difference. A 1,024-GPU H100 job at a nominal $3/GPU-hour costs **$3,072/hour**. A run that finishes in 30 days at 40% MFU needs `30 × 40/27 = 44.4` days at 27% MFU — 720 GPU-hours of wall-clock becoming 1,066, or **$2.21 M becoming $3.27 M**. A **$1.06 M swing on one project**, and the usual cause is a placement decision that nobody logged. Unlike a crash, low MFU never pages anybody: it survives code review, it survives the on-call rotation, and it dies in the invoice.

The skill this lesson buys you is the ability to say, in one sentence on an incident bridge: *"MFU dropped 38% → 22% at 14:03; the comms term roughly doubled; the DP gang was rescheduled off-rail after a node drain."* That is a diagnosis with an owner and a fix. "Training got slow" is not.

## What's new here (calibration)

Module 03 gave you two regimes on the roofline: **compute-bound** (on the flat FLOP ceiling) and **memory-bound** (on the HBM-bandwidth slope). Distributed training adds a third axis the roofline does not draw: **communication-bound**, where step time is set by an *off-chip* link — NVLink inside a node, InfiniBand or RoCE between nodes — and neither HBM nor the SMs are the limit.

Genuinely new here, beyond what 08.1 and 08.2 gave you:

- **The `6Ψ` FLOPs-per-token rule derived**, not quoted, so you can adjust it when the assumptions break (activation recomputation, MoE, long context).
- **MFU and HFU as two different denominators**, with the exact reason they differ and how much — plus the third denominator nobody mentions, the **measured** matmul ceiling, which is ~80% of the vendor peak on H100 and reframes what "40% MFU" means.
- **The comms-to-compute ratio derived symbolically**, yielding a **critical tokens-per-GPU threshold** below which a job is communication-bound. The striking result: for data parallelism that threshold is **independent of model size**, and depends only on the machine's FLOPs-to-bytes ratio.
- **How overlap actually works and how it breaks** — streams, buckets, prefetch — with a timeline showing exactly where the exposed bubble is and what closes it.
- **Why leaving the node costs ~2×, not ~18×**, worked through with measured numbers, and why that inverts completely for small messages.
- **The three causes of low MFU with distinguishing signatures**, so the metric becomes a diagnosis rather than a report card.

We assume 02b's topology vocabulary (NVLink domains, rail alignment, GPUDirect), 08.1's parallelism/collective mapping, and 08.2's transport, algorithm and bus-bandwidth vocabulary throughout. None is re-taught.

## Core concepts

### 1. The step-time model

A synchronous training step is two things happening on the same GPU: arithmetic, and bytes moving over a link. Write them as `T_comp` and `T_comm`. Two bounding cases:

```
   no overlap at all :   T_step = T_comp + T_comm
   perfect overlap   :   T_step = max(T_comp, T_comm)
   reality           :   T_step = max(T_comp, T_comm) + T_exposed
```

`T_exposed` is the part of the communication that could not be hidden behind compute — the tail of the last gradient bucket in DDP, the first all-gather of the step in FSDP, and any collective that sits on the critical path by construction (tensor parallelism's per-block all-reduce, which the next layer's matmul literally cannot start without).

**Everything in this lesson is an attempt to make one of those three terms concrete.** `T_comp` comes from §2. `T_comm` comes from 08.1's byte counts divided by 08.2's measured bus bandwidth (§4). `T_exposed` comes from §6.

The reason this decomposition is worth the trouble: the three causes of low MFU (§9) each attack a different term, and the fixes are not interchangeable. Adding bandwidth to a job whose problem is `T_exposed` buys nothing.

### 2. `6Ψ` FLOPs per token, derived

**The problem.** To compute utilisation you need a numerator: how much *useful* arithmetic a step performed. You could count every kernel's FLOPs with a profiler, but then the number depends on your implementation, which defeats the purpose — you want a hardware-independent, implementation-independent measure of the work the *model* required.

**The derivation.** Take a single weight matrix `W` of shape `(k, n)` inside the network, and a batch of `t` tokens flowing through it.

*Forward.* The operation is `Y = X · W`, with `X` of shape `(t, k)`. A matmul of `(t,k)·(k,n)` performs `t · k · n` multiply-accumulate operations, and each MAC is conventionally counted as **2 FLOPs** (one multiply, one add). So forward costs `2 · t · (k·n)` = **2 FLOPs per parameter per token**.

*Backward.* Backpropagation through the same layer needs two products, not one:

- the **gradient with respect to the input**, `dX = dY · Wᵀ` — needed to keep propagating backwards — which is again a `(t,n)·(n,k)` matmul: `2 · t · k · n` FLOPs;
- the **gradient with respect to the weights**, `dW = Xᵀ · dY`, a `(k,t)·(t,n)` matmul: another `2 · t · k · n` FLOPs.

So backward costs **4 FLOPs per parameter per token**, exactly twice forward. That 1:2 ratio is not an empirical fudge — it is the count of matmuls, and it is why the constant is 6 and not some other number.

```
   FLOPs per token  =  2Ψ (fwd)  +  4Ψ (bwd)  =  6Ψ
```

**When 6Ψ is wrong, and by how much.** State the assumptions so you can adjust:

| Situation | Effect on the constant | Why |
|---|---|---|
| Full activation checkpointing | `6Ψ → ~8Ψ` of *hardware* work | the forward pass is recomputed during backward: `+2Ψ` |
| Selective (attention-only) recompute | `6Ψ → ~6.25Ψ` hardware work | Megatron measures ~4% extra FLOPs for this |
| Attention score matrices | adds `~12 · L · s · h`-ish per token, growing with `s` | the `QKᵀ` and `AV` products are not weight matmuls, so `Ψ` does not see them; negligible at 2K context, material at 128K |
| Embedding / output projection | small over-count if you include the tied embedding in `Ψ` | it participates in one matmul, not the full stack |
| MoE | use **active** parameters, not total | a token touches `k` of `E` experts, so `Ψ_active ≪ Ψ_total` |

**The convention that matters most:** the `6Ψ` you put in an MFU numerator is *model* FLOPs — the arithmetic the maths required — and it deliberately **excludes recomputation**, because recompute is a memory-saving implementation choice, not work the model demanded. That single decision is the entire MFU/HFU distinction in §3.

### 3. MFU, HFU, and the third denominator

**Model FLOPs Utilisation:**

```
                6 · Ψ · tokens_per_second
    MFU  =  ──────────────────────────────────
              n_gpu  ×  peak_FLOP/s_per_gpu
```

**Hardware FLOPs Utilisation (HFU)** uses the same denominator but puts *executed* FLOPs in the numerator, including activation-recomputation. So `HFU ≥ MFU` always, for the same run, with the gap equal to the recompute fraction.

Google's PaLM paper is where the split was formalised and where both numbers are published for the same run: **46.2% MFU and 57.8% HFU** on 6,144 TPU v4 chips. That 11.6-point gap is *definitional*, not a measurement difference — it is the recompute the run performed. Meta's Llama 3 405B run reports **38–43% BF16 MFU** on 16,384 H100s, at roughly 400 TFLOP/s per GPU at 8K sequence length. Those two numbers, from different hardware generations and different papers, are the frontier bar for dense-transformer training.

**Comparing a paper's HFU to another paper's MFU is the single most common mistake in casual utilisation discussion.** Check which one you are reading before concluding that one infra team outperformed another.

**Now the denominator nobody mentions.** `peak_FLOP/s_per_gpu` is a *vendor* number derived from clock × tensor cores × 2, and no real kernel reaches it. `stas00/ml-engineering` measures the ceiling directly by brute-force searching matmul shapes — call it MAMF, Maximum Achievable Matmul FLOPS — and reports, for BF16 without sparsity:

| Accelerator | Measured MAMF | Vendor peak | Efficiency |
|---|---:|---:|---:|
| NVIDIA H100 SXM | 794.5 TFLOP/s | 989 | **80.3%** |
| NVIDIA GH200 SXM | 828.6 TFLOP/s | 989 | 83.8% |
| NVIDIA B200 SXM | 1745.0 TFLOP/s | 2250 | 77.6% |
| NVIDIA B300 SXM | 1769.0 TFLOP/s | 2250 | 78.6% |
| NVIDIA GB200 SXM | 1822.0 TFLOP/s | 2500 | 72.9% |

Read that as: **a pure, perfectly shaped matmul with no communication, no data loading and no memory pressure gets ~80% of the peak.** So a 40% MFU run is running at **~50% of what the silicon can actually do on its best day**, not 40%. That reframing matters twice over. It tells you the realistic ceiling is nearer 55–60% MFU than 100%, so 38–43% is genuinely good. And it tells you which yardstick to use when optimising: measure MAMF on *your* nodes with *your* shapes, then compare your training's achieved TFLOP/s against that, not against the datasheet.

**Two more practical notes on measuring MFU.** Use a *median over steady-state steps*, discarding warmup — the first few steps include allocator growth, autotuning and lazy initialisation, and they will drag a mean down by several points. And measure over a **short window**: MFU averaged over a week silently absorbs restart and rework time (§9, cause 3), so an aggregate number cannot tell you which of the three causes is biting.

### 4. The communication term, computed

08.2 gave you the identity you need: for a collective of `S` bytes over `N` ranks, the elapsed time is the payload divided by the *algorithm* bandwidth, and bus bandwidth is algorithm bandwidth scaled by the collective's correction factor. Invert it:

```
    all-reduce      T = S · 2(N−1)/N  /  busbw
    all-gather      T = S ·  (N−1)/N  /  busbw
    reduce-scatter  T = S ·  (N−1)/N  /  busbw
```

`busbw` here must be a **measured** number for your fabric at a payload near yours, not a datasheet figure. 08.2's calibration: expect 80–88% of the unidirectional spec for a plain ring, more if NCCL selects NVLS on an all-reduce, and much less at small payloads where latency dominates.

**DDP, per step.** From 08.1: one logical gradient all-reduce, `S = 2Ψ` bytes for bf16 gradients (or `4Ψ` where the framework keeps an fp32 main-gradient buffer — check, it is a factor of two).

```
    T_comm(DDP)  =  2Ψ · 2(N−1)/N / busbw   ≈  4Ψ / busbw     (large N)
```

**FSDP / ZeRO-3, per step.** Two all-gathers of the parameters (once before forward, once before backward, because the unsharded copy is freed in between) plus one reduce-scatter of the gradients:

```
    per-rank bytes  =  2 × [2Ψ · (N−1)/N]   +   2Ψ · (N−1)/N
                    =  3 · 2Ψ · (N−1)/N     ≈   6Ψ           (large N)

    versus DDP's       2 · 2Ψ · (N−1)/N     ≈   4Ψ
```

**FSDP moves 1.5× the bytes DDP moves, per step.** That is the "ZeRO-3 costs 1.5× the communication volume" rule of thumb, derived rather than recalled — and it is the network side of 08.1's memory-for-network trade, now with a number on it. With `reshard_after_forward=False` the second all-gather disappears and the ratio drops back to 1.0× at the cost of holding the unsharded parameters through the step.

**Tensor parallelism, per step.** From 08.1: two all-reduces forward and two backward per transformer block, each carrying the activation tensor `s · b · h · 2` bytes. This term is different in kind from the other two — **it cannot be overlapped**, because the next operation in the block consumes the all-reduce's output. It goes straight into `T_exposed`.

### 5. The comms-to-compute ratio, and the batch size that flips it

Now put §2 and §4 together. Let `t` be the number of tokens processed **per GPU per step** and `P_eff` the achievable FLOP/s per GPU (use MAMF, not the datasheet).

```
                    6 · Ψ · t                                    2Ψ · f
    T_comp  =  ─────────────── ,        T_comm(DDP)  =  ─────────────────
                     P_eff                                     busbw

                            where  f = 2(N−1)/N   (≈ 2 for large N)
```

Take the ratio:

```
     T_comm      2Ψ·f / busbw          f · P_eff
    ───────  =  ───────────────   =   ────────────
     T_comp      6Ψ·t / P_eff          3 · t · busbw
                  ▲
       Ψ CANCELS. Both terms are linear in parameter count.
```

**That cancellation is the most useful thing in this lesson.** For data parallelism, whether you are communication-bound **does not depend on how big your model is**. It depends on two things only: how many tokens each GPU chews per step, and your machine's ratio of achievable FLOP/s to achievable bytes/s.

Set the ratio to 1 and solve for the crossover:

```
                 f · P_eff              2(N−1)/N · P_eff
    t*  =  ────────────────────  =  ──────────────────────
              3 · busbw                    3 · busbw

    Below t* tokens per GPU per step, the DDP all-reduce is longer than
    the compute it is supposed to hide behind → COMMUNICATION-BOUND.
```

**Evaluate it on real, measured hardware.** All bandwidths below are `busbw` measurements from `stas00/ml-engineering` on the platforms named; all `P_eff` values are that project's measured MAMF.

```
  ── 8× H200, single node, NVLink 4, ring (NCCL_NVLS_ENABLE=0) ──────────────
     P_eff  = 794.5e12 FLOP/s      busbw = 367.2e9 B/s      N = 8, f = 1.75
     t*  =  1.75 × 794.5e12 / (3 × 367.2e9)  =  1,262 tokens/GPU/step

  ── same node, all-reduce with NVLS enabled (SHARP in the NVSwitch) ────────
     busbw = 480.0e9 B/s
     t*  =  1.75 × 794.5e12 / (3 × 480.0e9)  =    966 tokens/GPU/step
     ▲ in-network reduction moves the crossover DOWN: you may run a
       smaller per-GPU batch before comms starts to dominate.

  ── 4× (8× B200) = 32 GPUs, NVLink 5 + 8×50 GB/s EFA v4, 4 GiB payload ─────
     P_eff  = 1745e12 FLOP/s       busbw = 377.3e9 B/s      N = 32, f = 1.9375
     t*  =  1.9375 × 1745e12 / (3 × 377.3e9)  =  2,987 tokens/GPU/step

  ── the same 32 GPUs, but with FSDP instead of DDP (1.5× the bytes) ────────
     t*  =  1.5 × 2,987  =  4,481 tokens/GPU/step
```

**How to read those numbers.** On the 32-GPU B200 fabric running DDP, a per-GPU micro-batch of one 8,192-token sequence gives `t = 8,192 ≫ 2,987`: compute dominates by ~2.7×, and with decent overlap the collective is essentially free. Drop to 2,048-token sequences at micro-batch 1 and you are at `t = 2,048 < 2,987` — now the all-reduce is longer than the compute and no amount of overlap saves you, because there is not enough compute to hide behind. **The lever is per-GPU batch, and this is why "we added GPUs and throughput barely moved" is such a common report**: holding the *global* batch constant while raising `N` divides `t` by the same factor and walks you straight across `t*`.

Llama 3's own numbers show this effect at frontier scale: the paper records MFU dipping from 43% at 8K GPUs to 41% at 16K GPUs *purely* because the per-DP-group batch shrank to hold the global token count constant. Same model, same fabric, fewer tokens per GPU, lower MFU.

**Three caveats to carry with the formula.** It assumes zero overlap (so it is the conservative crossover — with good overlap the job stays healthy somewhat below `t*`). It assumes DDP-style single-collective communication, so for TP-heavy configurations you must add the per-block term separately, and that term is never hidden. And `busbw` is payload-dependent: if your per-step collective is small, look up the bandwidth at *that* payload, not at 4 GiB, or you will be optimistic by an order of magnitude (§7).

### 6. Overlap — how the collective hides, and how it stops hiding

Everything above assumed the two terms are separable. They are, because CUDA lets NCCL kernels run on a different stream from the compute kernels, and because backward propagates layer-by-layer so gradients become available progressively.

```
  DDP STEP TIMELINE — where the bubble is and what closes it
  ═══════════════════════════════════════════════════════════════════════════
  Model of 4 bucket-sized gradient groups. Backward produces gradients
  LAST-LAYER-FIRST, so bucket 3 is ready while layer 1 is still computing.

  (a) NO OVERLAP — one all-reduce after the whole backward
      compute  │███ bwd L4 ███│███ bwd L3 ███│███ bwd L2 ███│███ bwd L1 ███│
      NCCL     │                                                            │██████ AR(all) ██████│
      T_step   ├──────────────────── T_comp ───────────────────────────────┤├──── T_comm ────────┤
               ▲ THE ENTIRE COLLECTIVE IS EXPOSED.  T_step = T_comp + T_comm

  (b) BUCKETED OVERLAP — DDP's actual behaviour (bucket_cap_mb = 25 MiB)
      compute  │███ bwd L4 ███│███ bwd L3 ███│███ bwd L2 ███│███ bwd L1 ███│
      NCCL     │              │██ AR b3 ██│  │██ AR b2 ██│  │██ AR b1 ██│  │██ AR b0 ██│
                                                                            └────┬────┘
                                                          ONLY THE LAST BUCKET IS EXPOSED
      T_step   ├──────────────── T_comp ───────────────────────────────────┤├ T_exp ┤
               ▲ T_step = T_comp + T_comm/n_buckets  (roughly)

  (c) WHAT MAKES (b) DEGRADE BACK TOWARD (a)
      ─ buckets too LARGE  → the last one is huge → big exposed tail
      ─ buckets too SMALL  → each collective is latency-bound, aggregate
                             busbw collapses (see §7), so T_comm balloons
      ─ t < t* (§5)        → the compute bars are simply SHORTER than the
                             NCCL bars; there is nothing to hide behind
      ─ one straggler rank → every bucket's all-reduce waits for it; the
                             NCCL bars stretch and stop fitting under compute
      ─ CPU-bound launch   → the host cannot enqueue work fast enough to run
                             ahead of the GPU; gaps appear in BOTH rows

  ── TENSOR PARALLELISM IS THE EXCEPTION ───────────────────────────────────
      compute  │██ attn ██│           │██ mlp ██│           │██ attn ██│
      NCCL     │          │██ AR ██│  │         │██ AR ██│  │          │
                          └───┬────┘
      The next matmul CONSUMES this all-reduce's output. It CANNOT be
      overlapped with the work that depends on it. TP's collective is
      100% exposed by construction — which is the whole reason 08.1 says
      TP must stay inside the NVLink domain.

  ── FSDP: overlap runs FORWARD as well as backward ────────────────────────
      compute  │ fwd(m1) │ fwd(m2) │ fwd(m3) │ ... │ bwd(m3) │ bwd(m2) │ ...
      AG strm  │ AG(m2)  │ AG(m3)  │ AG(m4)  │     │ AG(m2)  │ AG(m1)  │
      RS strm  │         │         │         │     │ RS(m3)  │ RS(m2)  │
                 ▲ each group's all-gather is issued from the PREVIOUS
                   group's hook, so it lands before it is needed.
                   Break the prefetch (wrap only the root — 08.1 §4) and
                   every collective becomes exposed.
```

**The diagnosis, from the timeline.** If `T_step ≈ max(T_comp, T_comm)`, your overlap is working and the fix is either more compute per GPU or less traffic. If `T_step ≈ T_comp + T_comm`, overlap is *broken* and the fix is structural — wrap FSDP per layer rather than only at the root, retune bucket size, or find the straggler. Those are different work items, and the ratio tells you which one you have. You can measure it directly: run the same model on one GPU (pure `T_comp`) and on `N` GPUs, and compare.

**Scaling efficiency is the cheap proxy.** `eff = throughput(N) / (N × throughput(1))`. Ideal is 1.0. On one node over NVLink, DDP typically lands at 0.85–0.95. Well below that means either exposure or a `t < t*` problem, and `comms_fraction ≈ 1 − eff` is a serviceable first estimate of the exposed communication.

### 7. The small-message regime, and why bucketing exists

Everything so far used large-payload bandwidths. Collectives do not deliver those at small sizes, and the collapse is dramatic. Measured `busbw` for all-reduce on P6-B200 nodes (8× B200, NVLink 5 inside, 8× 50 GB/s EFA v4 out), one node versus four:

| payload | 1 node | 4 nodes | slowdown |
|---:|---:|---:|---:|
| 32 KiB | 1.20 GB/s | 0.01 GB/s | 120× |
| 256 KiB | 9.54 GB/s | 1.33 GB/s | 7.2× |
| 1 MiB | 36.06 GB/s | 5.37 GB/s | 6.7× |
| 16 MiB | 254.96 GB/s | 64.43 GB/s | 4.0× |
| 256 MiB | 646.11 GB/s | 229.09 GB/s | 2.8× |
| 1 GiB | 723.34 GB/s | 361.99 GB/s | 2.0× |
| 16 GiB | 845.67 GB/s | 381.80 GB/s | 2.2× |

Look down the "1 node" column first, ignoring the fabric entirely: **the same NVLink delivers 1.2 GB/s at 32 KiB and 845 GB/s at 16 GiB — a factor of 700.** That is 08.2's `2(N−1)·α` latency term dominating. Below roughly 1 MiB you are paying almost pure latency and getting almost no bandwidth.

**This is the entire justification for gradient bucketing.** A 7B model has hundreds of parameter tensors, many of them small (layer norms, biases). All-reducing each one separately means hundreds of collectives in the sub-100 KiB regime, at ~1% of achievable bandwidth. Coalescing them into 25 MiB buckets (PyTorch DDP's `bucket_cap_mb` default) moves the same bytes at ~30% of achievable bandwidth or better. **The bytes did not change; the bandwidth did, by two orders of magnitude.** The same reasoning explains why FSDP's per-layer communication groups must not be too fine-grained, and why `NCCL_MIN_CTAS` sometimes helps small collectives.

### 8. Why leaving the node costs ~2×, not ~18×

Look at the table again, at the bottom. On P6-B200 each accelerator has 900 GB/s of NVLink 5 and 50 GB/s of EFA — an **18× ratio between the links**. Yet a 16 GiB all-reduce is only **2.2× slower** on four nodes than on one. Almost everyone predicts 18×, and the prediction is wrong by nearly an order of magnitude. Understanding why is what stops you from over-provisioning fabric or from panicking about a multi-node job.

**The wrong model:** the whole payload has to cross one accelerator's NIC, so `T = P / 50 GB/s`. For 4 GiB that is 85.9 ms against 22.05 ms measured — off by 3.9×.

Two effects rescue it, and the first applies regardless of algorithm.

**Effect one: the node has `g` NICs, not one.** Each accelerator drives its own interface, so the node's inter-node bandwidth is `g × 50 = 400 GB/s`, not 50. NCCL opens multiple channels and different rings cross the node boundary at different accelerators, so all eight are in flight.

**Effect two: a hierarchical collective sends only the reduced shard over the slow links.**

```
  HIERARCHICAL ALL-REDUCE — where the bytes actually go
  ═══════════════════════════════════════════════════════════════════════════
  P = payload, g = GPUs per node (8), k = nodes (4), n = g·k = 32 ranks

  ┌── PHASE 1 ── intra-node reduce-scatter, over NVLink ────────────────────┐
  │   node 0 [ G0 G1 G2 ... G7 ]   each GPU ends holding 1/g of the         │
  │            ◀──── NVLink ────▶   node's summed vector                    │
  │   per-GPU bytes = (g−1)/g · P  =  7/8 · 4 GiB  =  3.50 GiB              │
  └─────────────────────────────────────────────────────────────────────────┘
                              │  only P/g = 0.5 GiB per GPU survives
                              ▼
  ┌── PHASE 2 ── inter-node all-reduce of that shard, over EFA ────────────┐
  │   node0.G0 ◀══ EFA ══▶ node1.G0 ◀══ EFA ══▶ node2.G0 ◀══▶ node3.G0     │
  │   (rail-aligned: GPU k on every node talks to GPU k elsewhere)          │
  │   per-GPU bytes = 2(k−1)/k · P/g  =  1.5 · 0.5 GiB  =  0.75 GiB        │
  └─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌── PHASE 3 ── intra-node all-gather, over NVLink ───────────────────────┐
  │   per-GPU bytes = (g−1)/g · P  =  3.50 GiB                             │
  └─────────────────────────────────────────────────────────────────────────┘

  ── THE ACCOUNTING ────────────────────────────────────────────────────────
     over NVLink : 2(g−1)/g · P      = 7.00 GiB per GPU
     over EFA    : 2(k−1)/k · P/g    = 0.75 GiB per GPU
     fraction leaving the node = 0.75 / 7.75  =  9.7%
                                       ▲
     THAT is why an 18× link ratio produces a ~2× collective ratio: only a
     tenth of the traffic touches the slow link, and all g NICs carry it.

  ── SANITY CHECK against the measurement ──────────────────────────────────
     0.75 GiB at 50 GB/s wire rate            = 16.1 ms  (fully overlapped)
     single-node time for the same payload    = 10.15 ms
     naive serial sum 16.1 + 10.15            = 26.25 ms (zero overlap)
     MEASURED (busbw 377.34 GB/s at 4 GiB)    = 22.05 ms  ← between them
     ⇒ the intra-node phases substantially overlap the inter-node exchange.
```

**And the conversion you actually want.** If `busbw` on a multi-node run is not a wire speed, what *is* comparable to the NIC's spec? Undo both scalings; the payload cancels:

```
    per-accelerator inter-node rate  =  busbw · (k−1)/(n−1)

    4-node 4 GiB row:  377.34 × 3/31  =  36.5 GB/s  =  73% of the 50 GB/s NIC
    single-node row :  740.64 / 900   =              82% of NVLink 5
```

**73% and 82% — the same ballpark.** Expressed per accelerator, the inter-node result stops looking anomalous. Use `busbw · (k−1)/(n−1)` when someone asks whether the fabric is healthy.

**The caveat that reverses the conclusion:** all of this is a *large-payload* effect. At 32 KiB the same comparison is 120× slower across nodes, because latency, not bandwidth, dominates and every node hop adds some. So "crossing the node is cheap" is true for your gradient all-reduce and false for anything small and chatty.

### 9. Rail alignment, and the three causes of low MFU

**Rail alignment as an MFU lever (tie to 02b).** Phase 2 in the diagram above is a rail-aligned exchange: GPU `k` on every node talks to GPU `k` elsewhere. If each node egresses GPU `k` on NIC `k`, and NIC `k` on every node lands on the same leaf switch, that entire phase stays on one switch. Mis-align it and the same bytes cross the spine, contending with every other job's traffic. `NCCL_CROSS_NIC=0` (08.2) is how you tell NCCL to preserve the alignment; 06's topology-aware placement is how you get the ranks in the right sockets in the first place.

**The tensor-parallel placement failure is the sharpest version of this.** TP's per-block all-reduce is 100% exposed (§6) and runs four times per transformer block. From 08.1, a Llama-3-70B-shaped block at `s = 8192, h = 8192` carries 134 MB per all-reduce. Split a TP=8 group 4+4 across two nodes and that collective moves from NVLink to the fabric. Using measured bandwidths at a ~134 MB payload from the table in §7 — roughly 568 GB/s single-node against 198 GB/s across nodes on the B200 platform — the exposed term grows by about **2.9×**, and because it is exposed it lands directly on step time rather than hiding.

> **Correction to the previous version of this lesson.** It stated that a 4+4 TP split "collapses MFU from ~38% to 12–15%" and described the link gap as "~18× slower." Both numbers were asserted without a source. The link *specs* are ~18× apart per accelerator (900 vs 50 GB/s on B200-class hardware), but measured collective bandwidth at realistic payloads is 2–4× apart, not 18×. The honest statement is: **a mis-placed TP group multiplies the exposed collective term by roughly 3× at typical payloads, which is enough to dominate a step whose compute was previously the larger term** — a severe, easily-several-fold regression whose exact size depends on payload, fabric and how much of the step the TP term represented. Measure it; do not quote a number you did not measure.

**The three causes, and how to tell them apart.** When MFU is below the ~35% bar, it is almost always one of three, and they need different fixes:

```
  LOW MFU — WHICH OF THE THREE?
  ═══════════════════════════════════════════════════════════════════════════

              is the step counter advancing normally?
                        │
        ┌───────────────┴──────────────┐
        NO                             YES
        │                              │
   ▶ not an MFU problem —              │
     a HANG. Go to 08.2.               │
                                       ▼
                       compare median step time on 1 GPU vs N GPUs
                                       │
        ┌──────────────────────────────┼───────────────────────────────┐
        │                              │                               │
  T_step(N) ≈ T_step(1)         T_step(N) ≫ T_step(1)          T_step(N) varies
  and both are slow                                             wildly step to step
        │                              │                               │
        ▼                              ▼                               ▼
  ╔═════════════════╗          ╔═══════════════════╗          ╔══════════════════╗
  ║ 2. DATA         ║          ║ 1. COMMUNICATION  ║          ║ 3. FAILURE       ║
  ║    STARVATION   ║          ║                   ║          ║    OVERHEAD      ║
  ╠═════════════════╣          ╠═══════════════════╣          ╠══════════════════╣
  ║ SM idle at STEP ║          ║ NCCL kernels are  ║          ║ long-window MFU  ║
  ║ BOUNDARIES,     ║          ║ a large share of  ║          ║ low, short-window║
  ║ periodic        ║          ║ the profile; SM   ║          ║ MFU healthy;     ║
  ║ host CPU or     ║          ║ occupancy dips    ║          ║ gaps in the step ║
  ║ storage pegged  ║          ║ during them       ║          ║ log; restarts in ║
  ║ loader workers  ║          ║                   ║          ║ the supervisor   ║
  ║ saturated       ║          ║                   ║          ║ log              ║
  ╠═════════════════╣          ╠═══════════════════╣          ╠══════════════════╣
  ║ CAUSE           ║          ║ CAUSE             ║          ║ CAUSE            ║
  ║ too few workers ║          ║ • t < t*  (§5)    ║          ║ MTBF × restart   ║
  ║ slow object     ║          ║ • overlap broken  ║          ║ cost + rework    ║
  ║ store, no       ║          ║   (§6)            ║          ║ since the last   ║
  ║ prefetch, JPEG  ║          ║ • bad placement   ║          ║ checkpoint       ║
  ║ decode on CPU   ║          ║   (§9)            ║          ║                  ║
  ║                 ║          ║ • messages too    ║          ║                  ║
  ║                 ║          ║   small (§7)      ║          ║                  ║
  ╠═════════════════╣          ╠═══════════════════╣          ╠══════════════════╣
  ║ → 08.7          ║          ║ → THIS LESSON     ║          ║ → 08.4 and 08.5  ║
  ╚═════════════════╝          ╚═══════════════════╝          ╚══════════════════╝
```

The single-GPU-versus-N-GPU comparison is the cheapest discriminator you have and it takes one extra run. If one GPU is *already* slow relative to its MAMF, the distributed layer is innocent and you have a data or kernel problem. If one GPU is fast and `N` GPUs are slow, the gap is communication, and §5–§8 tell you which flavour.

**MFU is the report card; these three are the line items.** A good incident note names the line item.

## Perspectives

**FinOps / economics view.** MFU is literally the ratio of what you are billed for to what you converted into gradient updates, which makes it the bridge from this lesson's mechanics to 08.8's dollar capstone. Every point of MFU recovered is a direct percentage cut in cost-per-successful-run. But it is a *report card*, not a diagnosis: it tells finance a delta exists, not which of the three causes to fix. Owning the translation — percentage → dollars → root cause → fix → re-measure — is the job. And note that the three causes have wildly different costs to fix: a placement change is free, a bigger per-GPU batch may change convergence and needs the ML team's agreement, and buying more fabric is a capital decision.

**Network / topology view.** The most concrete, testable claim in this lesson is that leaving the node costs ~2× at large payloads and ~100× at small ones. Both halves matter. The first says you should not panic about multi-node data parallelism and should not over-buy fabric for it. The second says you should be ruthless about message size — bucketing, coalescing, avoiding per-tensor collectives — because that is where the fabric genuinely punishes you. And the reason both are true simultaneously is that a hierarchical collective sends only `2(k−1)/k · P/g` bytes over the slow link while all `g` NICs carry them.

**ML-research view.** Researchers historically thought in loss curves and tokens/second, not MFU; the metric is a platform import. PaLM and later MosaicML/Databricks report it as a first-class number precisely because infrastructure teams pushed for a normalised way to compare "how well is this cluster being used" across models and hardware generations. There is a real division of labour here worth being able to state in an interview: the platform team measures and defends MFU; the ML team owns training quality. The seam is the per-GPU batch size, which is simultaneously an MFU lever (§5) and a convergence parameter — so it is the one number neither side can change unilaterally.

**Historical / definitional view.** HFU (counts recompute) predates the now-standard MFU (useful work only) framing, which trips people up comparing older papers to newer ones. PaLM reports both — 57.8% HFU and 46.2% MFU on the same run — and the 11.6-point gap is entirely definitional. Add the third denominator from §3 and there are really three numbers in play: model FLOPs over vendor peak (MFU), hardware FLOPs over vendor peak (HFU), and achieved FLOPs over *measured* matmul ceiling (the one that tells you how much headroom is actually left). Only the third one answers "should I keep optimising."

**Hardware view.** Every threshold in §5 is a statement about a machine's FLOPs-to-bytes ratio, and that ratio has been getting *worse* for communication across generations: H100 to B200 roughly doubles both peak FLOP/s and NVLink bandwidth, but the inter-node NIC has not kept pace in the same node configurations. Where the ratio worsens, `t*` rises, which means each new generation needs *more* tokens per GPU per step to stay compute-bound. That is a structural reason why frontier runs keep growing their global batch, and why "we upgraded the GPUs and MFU dropped" is a coherent, expected outcome rather than a bug.

## Real-world use cases

- **Meta — Llama 3 405B, 16,384 H100s.** Reports **38–43% BF16 MFU** at roughly 400 TFLOP/s per GPU at 8K sequence length, with 4D parallelism (TP=8, CP=2, PP=16, DP filling the rest). *What it shows:* the frontier bar for dense-transformer training, and — the detail that matters for §5 — the paper records MFU dipping from 43% at 8K GPUs to 41% at 16K GPUs purely because the per-DP-group batch shrank to hold the global token count constant. That is the `t*` crossover, observed at the largest published scale.

- **Google — PaLM 540B, 6,144 TPU v4 chips.** Reports **46.2% MFU and 57.8% HFU** for the same run, and is the paper that formalised the split. *What it shows:* the definitional gap between the two metrics, measured; and the fact that the higher number was achieved partly by a *model* change (a reformulated transformer block computing attention and feedforward in parallel), which is a reminder that MFU is not purely an infrastructure metric.

- **`stas00/ml-engineering` — the MAMF measurements.** Brute-force search over matmul shapes on real accelerators reports H100 SXM at **794.5 TFLOP/s against a 989 TFLOP/s spec (80.3%)** and B200 SXM at 1745 against 2250 (77.6%), for BF16 without sparsity. *What it shows:* the vendor peak in every MFU denominator is unreachable by 20%+ even for a pure matmul, so the realistic MFU ceiling is nearer 55–60% than 100% — and the right yardstick for "should I keep optimising" is your own measured MAMF at your own shapes, not the datasheet.

- **`stas00/ml-engineering` — the 1-node vs 4-node all-reduce sweep.** On P6-B200 nodes, `busbw` for all-reduce measured at every payload from 32 KiB to 16 GiB, single-node and 4-node, with the arithmetic worked through three competing models. *What it shows:* the two facts §7 and §8 are built on — the 700× spread in achievable bandwidth between small and large payloads on the *same* link, and the fact that the naive "payload over one NIC" model is wrong by 3.9× while the hierarchical model brackets the measurement.

- **SemiAnalysis — "100,000 H100 Clusters: Power, Network Topology, Ethernet vs InfiniBand, Reliability, Failures, Checkpointing."** Industry-analyst deep dive tying network topology to achievable utilisation at extreme scale. *What it shows:* the rail-alignment and topology arguments here recur as a known operational failure mode at hyperscale, from an independent vantage point. *Public preview only; the remainder is paywalled and was not read for this lesson.*

## Worked example

**The ticket.** "We moved our 7B DDP pre-training job from 8 GPUs on one node to 32 GPUs on four nodes. Throughput went up 2.6×, not 4×. Is the fabric broken, and what do we do?"

Platform: 4 × P6-B200 (8× B200 per node, NVLink 5 inside, 8 × 50 GB/s EFA v4 out). Model: 7B parameters, bf16 gradients. Per-GPU micro-batch: 2 sequences of 2,048 tokens, so `t = 4,096` tokens per GPU per step.

**Step 1 — is the fabric broken? Compute the expected collective time.**

```
   gradient bytes   S  =  2 × 7e9  =  14.0e9 B  =  14 GB
   ranks            N  =  32   →   f = 2(32−1)/32 = 1.9375

   From the measured sweep, busbw at a ~14 GB payload on 4 nodes
   is the large-payload plateau: 381.8e9 B/s.

   T_comm  =  S · f / busbw  =  14.0e9 × 1.9375 / 381.8e9  =  71.0 ms

   Same collective on ONE node (N = 8, f = 1.75, busbw 845.67e9):
   T_comm  =  14.0e9 × 1.75 / 845.67e9                     =  29.0 ms
```

So the collective got 2.4× more expensive. That is the expected hierarchical penalty from §8, not a fault. **The fabric is fine.** Confirm by converting to a per-accelerator rate: `381.8 × (4−1)/(32−1) = 36.9 GB/s`, which is 74% of the 50 GB/s NIC — squarely in the healthy 73–82% band.

**Step 2 — how much compute is there to hide it behind?**

```
   FLOPs per step per GPU  =  6 · Ψ · t  =  6 × 7e9 × 4096   =  1.72e14
   P_eff (measured MAMF, B200 BF16)                          =  1745e12 FLOP/s
   T_comp  =  1.72e14 / 1.745e15                             =  98.6 ms
```

**Step 3 — evaluate the ratio and the crossover.**

```
   T_comm / T_comp  =  71.0 / 98.6  =  0.72

   t*  =  f · P_eff / (3 · busbw)
       =  1.9375 × 1745e12 / (3 × 381.8e9)  =  2,951 tokens/GPU/step

   t = 4,096  >  t* = 2,951    →   compute-bound, but ONLY BY 1.39×.
```

You are on the right side of the line, but with very little margin. On one node the same arithmetic gives `t* = 1.75 × 1745e12/(3 × 845.67e9) = 1,204`, so `t/t* = 3.4` — comfortable. **Moving to four nodes did not break anything; it moved the crossover up by 2.45× while `t` stayed fixed.**

**Step 4 — reconcile with the observed 2.6× instead of 4×.** With imperfect overlap, say a third of the collective is exposed:

```
   1 node  : T_step ≈ 98.6 + 0.33 × 29.0  =  108.2 ms
   4 nodes : T_step ≈ 98.6 + 0.33 × 71.0  =  122.0 ms

   throughput ratio  =  4 × (108.2 / 122.0)  =  3.55×  expected
   observed                                  =  2.60×
   ⇒ implied measured 4-node step time = 4 × 108.2 / 2.60 = 166.5 ms
```

**The model predicts 3.55× and reality delivered 2.60×, so ~27% is unexplained** — which is the interesting part. Something beyond the collective's raw cost is being lost. Candidates, in the order you should check them: exposure worse than a third (measure it — compare `T_step` against `max(T_comp, T_comm)`); a straggler stretching every collective (08.2's RAS `MISMATCH` check); rails mis-aligned so phase 2 crosses the spine (`NCCL_DEBUG_SUBSYS=GRAPH` plus `NCCL_CROSS_NIC=0`); or the data loader now feeding 32 GPUs from the same object store (08.7).

**Step 5 — compute MFU and decide.**

```
   tokens/s       =  32 GPUs × 4096 tokens / 0.1665 s   =  787,300 tokens/s
   useful FLOP/s  =  6 × 7e9 × 787,300      =  3.31e16  =  33.1 PFLOP/s
   denominator    =  32 × 2250e12                       =  72.0 PFLOP/s  (vendor peak)
   MFU            =  33.1 / 72.0                        =  45.9%
   against MAMF   =  33.1 / (32 × 1745e12) = 33.1 / 55.8 =  59.3%
```

**45.9% MFU is above the frontier bar, and 59.3% of the measured matmul ceiling.** That reframes the whole ticket: this job is not broken — it is running above what Llama 3 reports — and the ~15 points between 59.3% and a realistic ~75% ceiling are the recoverable part. The 2.6× is largely the price of hierarchy plus a fixable overlap gap, and the highest-value change is not "buy more fabric."

**The recommendation you deliver.** The fabric is performing at 74% of NIC spec — healthy. Scaling is sub-linear because the collective is 2.4× more expensive across four nodes while per-GPU compute is unchanged, and you are only 1.39× above the communication-bound threshold. Three levers, cheapest first: **(1)** raise per-GPU micro-batch from 2 to 4 sequences — doubles `t` to 8,192, pushes `t/t*` to 2.8, and costs nothing but HBM (check against 08.1's activation budget) — but it changes the global batch, so the ML team must agree; **(2)** verify rail alignment and set `NCCL_CROSS_NIC=0`; **(3)** if per-GPU batch cannot grow, switch the DP dimension to HSDP so the expensive collective stays on NVLink and only a cheap gradient all-reduce crosses the fabric. Do not buy more network.

## Practice

Feeds the **Survive-a-failure lab** deliverable ([`../practice/survive-a-failure/README.md`](../practice/survive-a-failure/README.md)). Rent **2 GPUs** — same node if you can get NVLink; note it if you cannot, because the whole point is measuring the link you actually have. Reuse the DDP job from 08.1.

1. **Establish your denominator.** Before measuring the training job, measure the machine. Run a matmul benchmark at several large shapes (the MAMF finder from `stas00/ml-engineering`, or a short script timing `torch.matmul` on shapes like 2048×2048×13312 in bf16) and record the best TFLOP/s you can reach on **one** GPU. Compare it to the vendor peak. **You now have both denominators**, and you know your realistic ceiling.

2. **Establish your bandwidth.** Run `all_reduce_perf` from `nccl-tests`, or `all_reduce_bench.py`, at several payloads spanning 64 KiB to 1 GiB. Record `algbw` and `busbw` for each. Plot or tabulate `busbw` against payload — **you have just measured your own version of the §7 table**, and you will see the same collapse at small sizes. Note which payload first reaches 80% of the plateau; that is your bucketing target.

3. **Measure the two step times.** Run the identical job on 1 GPU, then on 2 GPUs (DDP), **same per-GPU batch size**. Log median step time over ≥100 steady-state steps, discarding warmup, plus tokens/s for each.

4. **Compute all four numbers**, showing the arithmetic:
   ```
   scaling efficiency  eff  = throughput_2gpu / (2 × throughput_1gpu)
   comms fraction           ≈ 1 − eff
   predicted T_comm         = 2Ψ × 2(N−1)/N / busbw     ← your §2 busbw at that payload
   MFU                      = 6 · Ψ · tokens_s / (n_gpu × peak)
   MFU vs MAMF              = 6 · Ψ · tokens_s / (n_gpu × your step-1 measurement)
   ```
   Does `1 − eff` roughly match `predicted T_comm / T_step`? If it is much larger, you have exposure or a straggler; if much smaller, your overlap is better than the naive model assumed.

5. **Cross the threshold deliberately.** Compute `t* = f · P_eff / (3 · busbw)` for your setup using your own numbers from steps 1 and 2. Then run the job at a per-GPU batch **above** `t*` and one **well below** it, and record MFU for each. **Watching MFU fall off a cliff as you cross your own computed threshold is the point of this whole lesson.**

6. **(Optional) Break the overlap on purpose.** Set `bucket_cap_mb` to something tiny (1 MiB) and re-run. Predict from §7 what will happen to aggregate bandwidth before you measure it.

**Acceptance:** a written note recording (i) your measured single-GPU matmul ceiling versus the vendor peak, (ii) a `busbw`-versus-payload table with at least four points, (iii) 1-GPU and 2-GPU median step times, (iv) scaling efficiency and comms-overhead estimate, (v) your computed `t*` and MFU measured on both sides of it, and (vi) one paragraph: is this job compute-bound or comms-bound at 2 GPUs, what would you predict at 8 and at 32, and which of the three causes in §9 would you check first. Commit it under `../practice/survive-a-failure/`.

## Common pitfalls

- **"PaLM's 57.8% beats Llama 3's 38–43%, so Google's infra was better."** *Mechanism:* 57.8% is *hardware* FLOPs utilisation, which counts activation-recomputation as useful work; Llama 3's number is *model* FLOPs utilisation, which excludes it. PaLM's own MFU is 46.2% — still higher, but now comparable. Different chips, different denominators, and one of them is inflated by exactly the amount of recompute the run performed.

- **"All-reduce cost grows unboundedly with world size."** *Mechanism:* split the terms (08.2 §3). The bandwidth term is `2(N−1)/N · S / B`, which asymptotes to `2S/B` and is flat in `N`; only the latency term `2(N−1)·α` grows, and tree algorithms cut that to `~2·log₂N`. Conflating the two leads to wrong capacity planning: adding replicas does not multiply your per-GPU bandwidth bill, but it does lengthen the synchronisation tail and it *does* shrink `t` if you hold the global batch fixed — which is usually the real cause of the slowdown being blamed on bandwidth.

- **"Low MFU means a network problem."** *Mechanism:* three causes exist and only one is the network. The discriminator is a single-GPU run: if one GPU is already far below its measured matmul ceiling, the distributed layer is innocent and you have a data-pipeline or kernel problem (08.7). Jumping to "add bandwidth" without that check wastes engineering time and sometimes capital.

- **"35–40% MFU is disappointing."** *Mechanism:* the denominator is a vendor peak that a pure matmul only reaches ~80% of (measured: 794.5 of 989 TFLOP/s on H100). So 40% MFU is ~50% of the achievable matmul rate, with the rest going to communication, memory-bound operations like layernorm and softmax, and the optimiser step. Below ~30% is worth investigating; below ~20% something is clearly broken; but <100% is fundamental, not a bug.

- **"MFU measured over any window tells the full story."** *Mechanism:* a long-window average silently absorbs failure-overhead time — restarts, rework since the last checkpoint, slow-node detection — so a restart-bound run looks comms-bound in aggregate. You need short-window MFU around incidents versus steady-state windows to attribute correctly. That decomposition is exactly what 08.8's capstone formula performs.

- **"Bigger models are more communication-bound."** *Mechanism:* for data parallelism, `Ψ` cancels out of the comms-to-compute ratio (§5) — both terms are linear in parameter count. What makes a job communication-bound is **few tokens per GPU per step** and a machine with a high FLOPs-to-bytes ratio. Large models *feel* comms-bound because they force small per-GPU batches through memory pressure, which is a different mechanism with a different fix (shard more, or recompute more, to buy back batch).

- **"Just use `busbw` from the datasheet."** *Mechanism:* `busbw` is payload-dependent by two orders of magnitude on the same link (1.20 GB/s at 32 KiB versus 845.67 GB/s at 16 GiB, measured on the same node), and NCCL's algorithm selection changes with rank count and platform. Plugging a plateau number into a model whose collectives are 200 KiB gives an answer that is wrong by ~100×. Always look up the bandwidth at *your* payload.

- **"Scaling efficiency of 0.65 means 35% of the time is communication."** *Mechanism:* `1 − eff` is a serviceable *first* estimate but it lumps together exposed communication, load imbalance, per-rank straggling and any per-step synchronisation cost. Confirm it against the predicted `T_comm` from the byte count and the measured `busbw`; a large discrepancy is itself the finding, and it usually points at a straggler or at broken overlap rather than at raw bandwidth.

## Self-check

**(a) Why does all-reduce cost grow with world size?**
**Answer:** Split it into two terms. The **bandwidth term** is `2(N−1)/N · S / B` per rank, and `2(N−1)/N = 2 − 2/N` is bounded above by 2 — so per-GPU bytes converge to `2S` and are essentially flat in `N` (1.75 S at N=8, 1.9999 S at N=16,384). Data volume is set by model size, not world size. What grows is the **latency/synchronisation term**: a ring all-reduce takes `2(N−1)` steps, linear in `N`, and every rank waits for the slowest link and the slowest peer, so more ranks means more hops and a fatter tail. Tree algorithms cut that to `~2·log₂N` at roughly half the bandwidth, which is why NCCL switches for small messages and large rank counts. There is a third, indirect effect that is usually the real culprit: adding GPUs while holding the *global* batch constant divides tokens-per-GPU by the same factor, which shrinks the compute term the collective was hiding behind and can push you below the communication-bound threshold `t*`.

**(b) What MFU would concern you, and what are the three usual causes of low MFU?**
**Answer:** The frontier bar for 3D-parallel dense-transformer training is **~35–43% MFU** (Llama 3 405B reports 38–43% on 16K H100s; PaLM reports 46.2% MFU on TPU v4). Below **~30%** I would open an investigation; below ~20% something is clearly broken. Context for those numbers: the denominator is a vendor peak that a pure matmul only reaches ~80% of on H100 (794.5 of 989 TFLOP/s measured), so 40% MFU is about half of the achievable rate — the realistic ceiling is nearer 55–60%. The three causes: **(1) communication** — GPUs idle inside NCCL kernels, from too few tokens per GPU (below `t*`), broken overlap, bad placement, or messages too small to reach plateau bandwidth; **(2) data starvation** — GPUs waiting on the next batch, with idle at step *boundaries* and host CPU or storage pegged (08.7); **(3) failure overhead** — restarts and rework since the last checkpoint, which a long-window MFU silently absorbs (08.4/08.5). The cheapest discriminator is a single-GPU run: if one GPU is already slow relative to its measured matmul ceiling, the distributed layer is innocent.

**(c) Derive the batch size at which a data-parallel job becomes communication-bound.**
**Answer:** Compute per GPU per step is `T_comp = 6Ψt / P_eff`, where `t` is tokens per GPU per step and `P_eff` the achievable FLOP/s. DDP's gradient all-reduce carries `S = 2Ψ` bytes (bf16 gradients) and takes `T_comm = 2Ψ · f / busbw` with `f = 2(N−1)/N`. The ratio is `T_comm/T_comp = f·P_eff / (3·t·busbw)` — **`Ψ` cancels**, because both terms are linear in parameter count. Setting the ratio to 1 gives the crossover **`t* = f · P_eff / (3 · busbw)`**. Worked on measured hardware: 8× H200 with a ring all-reduce (`P_eff = 794.5e12`, `busbw = 367.2e9`, `f = 1.75`) gives `t* ≈ 1,262` tokens per GPU per step; enabling NVLS raises `busbw` to 480e9 and lowers `t*` to ~966; 32 B200s across four nodes (`P_eff = 1745e12`, `busbw = 377.3e9`, `f = 1.9375`) gives `t* ≈ 2,987`. FSDP moves 1.5× the bytes (two all-gathers plus a reduce-scatter versus DDP's single all-reduce), so its threshold is 1.5× higher. The formula assumes zero overlap, so it is conservative; and TP's per-block all-reduce must be added separately because it is never hidden.

**(d) Why does keeping tensor-parallel ranks in one NVLink domain matter, and by how much?**
**Answer:** TP all-reduces activations twice forward and twice backward per transformer block, and — unlike DDP's gradient all-reduce — **it cannot be overlapped**, because the next matmul consumes its output. So it lands entirely in the exposed term, thousands of times per step. Each message is `s·b·h·2` bytes: 134 MB for a Llama-3-70B-shaped block at 8K sequence. Moving that collective from NVLink to the fabric drops the achievable bandwidth at that payload from roughly 568 GB/s to roughly 198 GB/s on measured B200-class hardware — about **2.9× more expensive**, applied to a term that previously fitted under compute and now sets the step time. The result is a several-fold step-time regression that *runs correctly*, so nothing alerts; the symptom is "training got slow after a reschedule." The rule from 08.1 is **TP degree ≤ NVLink domain size**, and 06's topology-aware placement plus `NCCL_CROSS_NIC=0` are how you enforce it. Note that the *link specs* are ~18× apart per accelerator but measured collective bandwidth at realistic payloads is 2–4× apart; quote the ratio you measured, not the datasheet ratio.

**(e) A paper reports 57.8% utilisation and another reports 40%. Can you compare them directly?**
**Answer:** Not without checking which metric each reports. HFU counts activation-recomputation as useful work; MFU does not, and is therefore always the lower, stricter number for the same run. PaLM's paper reports both for the same training run — 57.8% HFU and 46.2% MFU — an 11.6-point gap arising purely from definition. There is also a third framing worth raising: both are divided by the *vendor peak*, which no kernel reaches. Divided instead by the measured matmul ceiling (~80% of peak on H100), a 40% MFU run is at ~50% of achievable — which is the number that answers "is there headroom left." So before concluding one infrastructure stack outperformed another, confirm three things: same metric, same hardware generation, and whether the comparison is against spec or against achievable.

**(f) Why is leaving the node only ~2× slower when the links are ~18× apart, and when is that not true?**
**Answer:** Two effects. First, a node has one NIC per accelerator, so its inter-node bandwidth is `g × 50 = 400 GB/s` on an 8-GPU node, not one link's 50 — NCCL opens multiple channels and different rings cross the boundary at different accelerators. Second, a hierarchical collective reduces inside each node first, exchanges only the reduced shard between nodes, and broadcasts back inside: per accelerator that is `2(g−1)/g · P = 7 GiB` over NVLink against `2(k−1)/k · P/g = 0.75 GiB` over the fabric for a 4 GiB payload on 8 GPUs × 4 nodes — so **only ~9.7% of the traffic leaves the node**. Measured, a 16 GiB all-reduce goes from 845.67 GB/s `busbw` on one node to 381.80 GB/s on four, a 2.2× slowdown. **It stops being true at small payloads**, where latency rather than bandwidth dominates and each node hop adds a fixed cost: the same comparison at 32 KiB is 120× slower. That is the argument for coalescing gradients into large buckets rather than reducing tensors individually. And to check fabric health, convert `busbw` to a per-accelerator rate with `busbw · (k−1)/(n−1)` — 36.5 GB/s against a 50 GB/s NIC is 73%, right next to the single-node figure's 82% of NVLink.

**(g) MFU on a job has averaged 30% over the last week, but a teammate says "the comms picture actually looks fine." How do both statements hold?**
**Answer:** Long-window MFU blends all three causes into one number and does not tell you the mix. If the run has been eating restarts and rework from a rough failure week (08.4/08.5's territory), the weekly average absorbs that dead time even though the comms term, measured in a clean steady-state window, is healthy. The fix is decomposition: measure MFU — or better, median step time — in short windows around known incidents versus steady-state windows, and account for wall-clock spent not stepping at all. If steady-state MFU is 38% and the weekly average is 30%, roughly a fifth of the wall-clock was not making progress, and the work item is 08.4's checkpoint interval and 08.5's restart speed, not the network. That decomposition is exactly what 08.8's capstone formula performs.

## Connections & what's next

08.2 gave you the mechanics — transports, algorithms, the ring's cost model, hang triage. This lesson turned those into a **step-time model** (`max(T_comp, T_comm) + T_exposed`), a **metric** (MFU, with its three competing denominators), and a **threshold** (`t*`) that predicts when a job flips from compute-bound to communication-bound before you run it.

**08.4** picks up the third cause of low MFU — failure overhead — and gives it exactly the same treatment: a formula (Young/Daly) that turns two measured numbers, checkpoint cost and MTBF, into a policy decision, the way this lesson turned a placement decision into a measurable MFU delta. The connection is tighter than it looks: checkpoint cost `C` is itself a bandwidth-and-overlap problem, and the async staging that shrinks it is the same "hide it behind compute" trick as §6.

**08.5** owns the restart side of that term, and **08.8's capstone** puts all three line items — comms, data, failure — into one cost-per-successful-run number. The `busbw` measurements and the MAMF ceiling you take from this lesson's Practice are direct inputs to it.

Backward: **02b** supplied the link topology and rail alignment that §8's phase-2 exchange depends on; **03** supplied the roofline this lesson adds a third axis to, and the peak-FLOP numbers in every denominator; **08.1** supplied the per-step byte counts for every strategy; **08.2** supplied the bus-bandwidth definition, the measured bandwidths, and the `TUNING` log line that tells you which algorithm produced them.

## References & further reading

**Primary sources**

1. **The Llama 3 Herd of Models, §3.3 "Infrastructure, Scaling, and Efficiency"** — <https://arxiv.org/abs/2407.21783>. The module's anchor: 16,384 H100s, 4D parallelism at TP=8 / CP=2 / PP=16, ~400 TFLOP/s per GPU at 8K sequence length, **38–43% BF16 MFU**, and the documented dip from 43% at 8K GPUs to 41% at 16K GPUs as the per-DP-group batch shrank. *`arxiv.org` is blocked by this environment's egress proxy; the parallelism degrees, throughput and MFU figures were confirmed via search snippets of the paper's own text, not by reading the PDF in this pass.*

2. **PaLM: Scaling Language Modeling with Pathways** — <https://arxiv.org/abs/2204.02311>. Origin of the MFU/HFU split, and the source of the **46.2% MFU / 57.8% HFU** pair on 6,144 TPU v4 chips for the same run, along with the note that the high HFU came partly from a reformulated transformer block computing attention and feedforward in parallel. *`arxiv.org` blocked here; both figures and the 6,144-chip count confirmed via search snippets of the paper, not by reading the PDF.*

3. **nccl-tests — `doc/PERFORMANCE.md`** — <https://github.com/NVIDIA/nccl-tests>. **Read in full; it is two pages.** The definition of `algbw` and `busbw` and the derivation of the per-collective correction factors — `2(n−1)/n` for all-reduce, `(n−1)/n` for all-gather and reduce-scatter, 1 for broadcast and reduce — which every `T = S·f/busbw` calculation in §4 inverts. Verified by cloning at HEAD.

4. **NCCL source — `src/tuning/tuning_general.cc` and `src/tuning/cost_model.cc`** — <https://github.com/NVIDIA/nccl>. Source of the step counts (`2(nRanks−1)` for all-reduce, `nRanks−1` for all-gather/reduce-scatter) and the compiled-in latency constants behind the latency-versus-bandwidth split invoked in §7 and in Self-check (a). Verified in-source at v2.31.2; the full treatment is in 08.2.

5. **PyTorch — `torch/nn/parallel/distributed.py`** — <https://github.com/pytorch/pytorch>. Source of the DDP `bucket_cap_mb` default of **25 MiB** that §6 and §7 build the bucketing argument on. *`docs.pytorch.org` is blocked by this environment's egress proxy; verified against the in-repo source at HEAD.*

**Real-world engineering blogs**

6. **`stas00/ml-engineering` — `compute/accelerator/README.md`** — <https://github.com/stas00/ml-engineering>. **Read the "Maximum Achievable FLOPS" section in full.** Source of the MAMF table in §3 — H100 SXM 794.5 of 989 TFLOP/s (80.3%), GH200 828.6 (83.8%), B200 1745 of 2250 (77.6%), B300 1769 (78.6%), GB200 1822 of 2500 (72.9%), all BF16 without sparsity — and of the derivation `1830 MHz × 512 × 2 × 528 = 989.4 TFLOP/s` that reproduces NVIDIA's own H100 spec. Also the caveat that these are brute-force searches over a non-exhaustive shape space and should be re-run on your own hardware. Verified by cloning at HEAD.

7. **`stas00/ml-engineering` — `network/README.md`** — same repository. **Read "Inter-node speed depends on intra-node speed" and "SHARP" in full.** Source of the payload-versus-`busbw` table in §7 (1-node and 4-node P6-B200, 32 KiB through 16 GiB), the three competing models for the 4-node number and the arithmetic that rules out the naive one, the hierarchical byte accounting reproduced in §8 (7 GiB over NVLink versus 0.75 GiB over EFA per accelerator), the `busbw · (k−1)/(n−1)` per-accelerator conversion, and the single-node NVLS-versus-ring measurements (480.0 versus 367.2 GB/s at 16 GiB) used as `busbw` inputs in §5. Verified by cloning at HEAD.

8. **`stas00/ml-engineering` — `network/benchmarks/all_reduce_bench.py`** and **`compute/accelerator/benchmarks/` (MAMF finder)** — same repository. The two tools the Practice section uses to reproduce steps 1 and 2 on your own hardware. Verified by cloning at HEAD.

9. **Databricks / MosaicML — "Mosaic LLMs (Part 2): GPT-3 quality for <$500k"** — <https://www.databricks.com/blog/gpt-3-quality-for-500k>. Reports 40–60% MFU achievable on MPT-class training, a second independent data point that MFU discipline is not unique to frontier labs. Cost figure is a dated (2023) snapshot. *Not fetched in this pass; cited as a pointer, and the MFU range above is reported as its claim rather than relied upon for any calculation here.*

10. **Lambda — MFU whitepaper (B200 / GB300 NVL72)** — <https://lambda.ai/hubfs/4.%20Resources/White%20Papers/Lambda%20MFU.pdf>. A vendor whitepaper reporting a Llama-3.1-70B run moving from 23.8% to 50.2% MFU through interconnect-aware parallelism placement alone. *`lambda.ai` was not reachable in this pass and the document was **not read**; the previous version of this lesson quoted these figures as established. They are listed here as an unverified vendor claim and are **not relied upon** — §9's placement argument is instead built on the measured `busbw`-versus-payload data in source 7.*

11. **SemiAnalysis — "100,000 H100 Clusters: Power, Network Topology, Ethernet vs InfiniBand, Reliability, Failures, Checkpointing"** — <https://newsletter.semianalysis.com/p/100000-h100-clusters-power-network>. Topology-to-utilisation analysis at extreme scale. *Public preview only, remainder paywalled; **not read** in this pass and not relied upon for any number above.*

**Deeper dives**

12. **Adam Casson — "Transformer FLOPs"** — <https://www.adamcasson.com/posts/transformer-flops>. A term-by-term derivation of the `6Ψ`-per-token heuristic including the attention and embedding corrections that §2's table summarises. *Not fetched in this pass; the derivation in §2 is worked from the matmul shapes directly rather than taken from this source, which is listed as optional corroboration and is **not relied upon**.*
