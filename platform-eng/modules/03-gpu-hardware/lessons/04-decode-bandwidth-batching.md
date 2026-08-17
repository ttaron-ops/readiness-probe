---
lesson: "03.4"
title: "Decode throughput: bandwidth ceilings, batching, and the prefill/decode split"
module: "03"
concept: "Decode bandwidth & batching"
status: not-started
est_time: "7h"
prev: "03-memory-hierarchy-hbm.md"
next: "05-precision-and-tensor-cores.md"
artifacts: []
sources: 15
---

# 03.4 · Decode throughput: bandwidth ceilings, batching, and the prefill/decode split

> **Concept.** Decode tok/s ≈ HBM bandwidth ÷ model bytes; batching is the only lever that raises it, KV cache is what stops you batching forever, and production systems now split prefill and decode onto separate GPU pools to buy for each bottleneck independently.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Where this fits

Lesson 03.3 built the two-number budget — HBM capacity (what fits) and HBM bandwidth (how fast) — and ended with a ceiling: a 70B FP16 model on an H100 can produce at most ~24 tokens/sec in a single stream, set entirely by how fast the GPU streams weights out of HBM. It also gave you the token budget: how many KV-cache tokens a given SKU can hold once weights and overhead are subtracted.

This lesson puts those two numbers into one equation and asks what a platform engineer actually gets paid to answer: how do you turn ~24 tok/s per stream into thousands of tokens/sec of *aggregate* throughput off one GPU, what stops that scaling, and where exactly does the curve bend? The answer — batching, bounded by the KV footprint from 03.3 — is the mechanism behind essentially every modern LLM serving architecture, up to and including the prefill/decode disaggregation systems now running at production scale.

## Why this matters

This is the lesson that turns "GPU literacy" into "I can predict the invoice." Derive a model's decode ceiling from one spec-sheet number and one model size, and you can estimate cost-per-token *before* renting anything, size a fleet, and immediately smell whether a serving setup is leaving money on the table.

It is also squarely the interview territory the module README names: *"why is decode memory-bandwidth bound and prefill compute-bound?"* and *"tokens/sec ceiling for a 70B on H100"* are stock probes, and the honest answers require exactly this arithmetic.

The counter-intuitive part — the part that trips up engineers who assume more FLOPS means faster — is that a single user's token stream uses a rounding error of a datacenter GPU's compute. An H100 does 1,979 TFLOP/s of dense FP8 tensor math. Generating one token from a 70B model needs about **140 GFLOP**. If compute were the limit you would get `1.979e15 ÷ 1.4e11 = 14,136 tokens/sec`. You get about 48. The GPU spends 99.7% of its time waiting on memory, reading 70 GB of weights to produce one token's worth of arithmetic. **Decode is a memory-bandwidth problem wearing a compute costume**, and production LLM-serving companies now build entire serving architectures around that fact.

## What's new here (calibration)

Per the module README you are not implementing a serving engine — you are learning to reason about its throughput and cost from the outside. This lesson does not re-teach FlashAttention kernel internals, how a continuous-batching scheduler is implemented, or warp-level softmax tricks. What it adds:

- **A derivation you can run from memory** — the bandwidth-only decode ceiling, and why FLOPS never enters it.
- **A two-term time model for a decode step** that predicts the whole throughput-versus-batch curve, including *where it bends and why*, instead of the usual hand-wave that "batching helps until it doesn't."
- **The itemised realisation gap** — what the missing 20–40% between the ideal ceiling and measured throughput actually is, line by line.
- **The prefill/decode split derived quantitatively**, with arithmetic intensity computed on both sides of the transition so you can see it collapse by three orders of magnitude at a specific instant.
- **The serving-architecture ladder** — continuous batching, chunked prefill, disaggregation — each named with the paper that introduced it, the mechanism, and the reported gain.
- **Production evidence and an honest failure-mode view**, so you know when disaggregation is worth the operational complexity and when it is not.

## Core concepts

### 1. Deriving the single-stream ceiling

Autoregressive decode generates one token per forward pass. Each pass must read essentially **every weight** from HBM exactly once, because every layer is used and no weight is reused within a single-token pass. The arithmetic performed per weight is tiny: one multiply and one add, i.e. 2 FLOP per parameter.

```
Per forward pass, batch = 1:
  FLOPs  = 2 · N_params                     (Kaplan convention, forward only)
  Bytes  = e · N_params                     (e = bytes/param: 2 for BF16, 1 for FP8)
  AI     = 2N / (e·N) = 2/e                 = 1 FLOP/byte at BF16, 2 at FP8
```

Compare to H100's ridge points (295 BF16, 591 FP8, lesson 03.2): decode sits **295× below** the relevant ridge in either precision. It is not marginally memory-bound; it is memory-bound by two and a half orders of magnitude. So the time per token is bounded below by the time to stream the weights:

```
t_token  ≥  model_bytes ÷ HBM_bandwidth
tok/s    ≤  HBM_bandwidth ÷ model_bytes
```

**FLOPS do not appear.** That is the whole result.

| Model / precision | model_bytes | A100 (2.039 TB/s) | H100 (3.35 TB/s) | H200 (4.8 TB/s) | B200 (7.7 TB/s) |
|---|---|---|---|---|---|
| 8B FP16 | 16 GB | 127 tok/s | 209 tok/s | 300 tok/s | 481 tok/s |
| 70B FP16 | 140 GB | 14.6 tok/s | **23.9 tok/s** | 34.3 tok/s | 55.0 tok/s |
| 70B FP8 | 70 GB | n/a (no FP8 HW) | **47.9 tok/s** | 68.6 tok/s | 110 tok/s |
| 405B FP8 | 405 GB | — | 8.3 tok/s (≥6 GPUs) | 11.9 tok/s | 19.0 tok/s |

Three readings:

- **Quantisation is a throughput lever, not just a capacity lever.** Halving bytes per parameter halves the bytes read per token and therefore doubles the ceiling: 23.9 → 47.9 tok/s on the same H100. Lesson 03.5 goes deep on why the *realised* gain is usually less than 2×.
- **Model size, not model quality, sets the ceiling.** A 70B model is inherently 8.75× slower per stream than an 8B model on the same card. No amount of kernel work changes that ratio.
- **The multi-GPU rows are honest only with a caveat.** Tensor-parallel sharding across G GPUs splits both the weight bytes and the aggregate bandwidth by G, so the per-token weight-read time is roughly preserved and the ceiling formula still describes the aggregate — *minus* an all-reduce per layer per token over NVLink, which is real and grows with G. Treat the TP rows as planning-grade, not measured.

### 2. Where the missing 20–40% goes

Measured single-stream throughput lands at roughly **60–80%** of that ceiling. That gap is not slop; it is an itemisable list, and being able to itemise it is what stops you chasing a phantom optimisation. For Llama-3-70B FP8 on an H100 at 4k context, batch 1:

| Term | Bytes or time per token | Share of the step |
|---|---|---|
| Weight read (the ceiling term) | 70.0 GB | — |
| KV-cache read (GQA, FP8 KV, 4k ctx) | 4,096 × 160 KiB = 0.67 GB | +0.96% of bytes |
| Activations / intermediate round-trips | ~0.1–0.3 GB | +0.3% |
| **Ideal step time at 3.35 TB/s** | **(70.0 + 0.67 + 0.2) / 3.35e12 = 21.2 ms** | |
| Achieved bandwidth is not 100% of spec (~85–90% best case: refresh, ECC, row activation, imperfect coalescing) | +12% time | |
| Kernel-launch and scheduler overhead across ~80 layers × several kernels each | +3–8% time | |
| Attention kernel is not perfectly bandwidth-saturating at batch 1 (little work to hide latency) | +5–10% time | |
| **Realistic step time** | **≈ 26–28 ms → 36–38 tok/s** | vs 47.9 ideal → **75–79%** |

**Every line except the first is fixed cost that batching amortises.** That is the reason the realisation factor *improves* as batch grows — it is not a constant 0.75 you multiply by, it is a set of fixed overheads divided by a larger batch.

### 3. Batching: the amortisation mechanism, precisely

The weight read is a **fixed cost paid once per forward pass**, and a forward pass can process many sequences at once by turning the per-token matrix-vector products into matrix-*matrix* products. Run a batch of B sequences and you read the weights once but emit B tokens.

But the KV read is *not* shared. Every sequence has its own cache, and every sequence's cache must be read in full to compute its attention. So the honest model of one decode step is:

```
t_step(B) = max(   [ W + B·S·k ] ÷ BW   ,   [ B·(2N + a·S) ] ÷ P   )
                  └──────────────┘             └───────────────┘
                   MEMORY TERM                  COMPUTE TERM
   W    = weight bytes                    (shared across the batch)
   B·S·k = KV bytes, k = bytes/token/seq  (NOT shared — scales with B)
   BW   = HBM bandwidth
   2N   = weight-GEMM FLOPs per token     (scales with B)
   a·S  = attention FLOPs per token       (scales with B and context)
   P    = peak FLOP/s at the precision in use

   aggregate throughput = B ÷ t_step(B)
```

That single expression predicts the entire throughput-versus-batch curve, including where it bends. Work it for **Llama-3-70B, FP8 weights and FP8 KV, on one H200** (141 GB, 4.8 TB/s, 1,979 TFLOP/s dense FP8; from lesson 03.3: `W` = 70 GB, `k` = 160 KiB/token/seq, free-for-KV = 67 GB):

```
Attention FLOPs per token  a·S = 4 · S · d_head · H_q · L
                               = 4 · S · 128 · 64 · 80 = 2.62e6 · S FLOP
Weight-GEMM FLOPs per token 2N = 1.40e11 FLOP

── Long context, S = 8,192 ────────────────────────────────────────────────
  per-sequence KV bytes      = 8,192 × 163,840 B      = 1.342 GB
  per-sequence KV read time  = 1.342e9 ÷ 4.8e12       = 280 µs
  per-sequence compute time  = (1.40e11 + 2.15e10) ÷ 1.979e15 = 81.6 µs
                                 ▲ adding one sequence costs 280 µs of memory
                                   but only 82 µs of compute → the marginal
                                   sequence is MEMORY-bound. Decode at long
                                   context NEVER crosses to compute-bound.

  B    KV bytes   memory time   compute time   t_step    aggregate tok/s
  ───  ─────────  ───────────   ────────────   ──────    ───────────────
    1     1.3 GB     14.9 ms        0.08 ms    14.9 ms         67
    8    10.7 GB     16.8 ms        0.65 ms    16.8 ms        476
   16    21.5 GB     19.1 ms        1.31 ms    19.1 ms        840
   32    42.9 GB     23.5 ms        2.61 ms    23.5 ms      1,360
   49    65.8 GB     28.3 ms        4.00 ms    28.3 ms      1,732   ← CAPACITY WALL
   50       —          —              —          —          OOM / preempt

── Short context, S = 512 ─────────────────────────────────────────────────
  per-sequence KV bytes      = 512 × 163,840        = 83.9 MB
  per-sequence KV read time  = 83.9e6 ÷ 4.8e12      = 17.5 µs
  per-sequence compute time  = (1.40e11 + 1.34e9) ÷ 1.979e15 = 71.4 µs
                                 ▲ now compute per sequence (71 µs) EXCEEDS
                                   memory per sequence (17.5 µs) → the curve
                                   WILL cross to compute-bound.

  crossover: 70e9/4.8e12 + B·17.5e-6 = B·71.4e-6
             14.58e-3 = B · 53.9e-6  →  B* ≈ 270 sequences
             (compare lesson 03.2's algebraic critical batch, ≈295 — the
              difference is attention FLOPs and KV reads, both ignored there)

  B    memory time  compute time   t_step   aggregate tok/s
  ───  ───────────  ────────────   ──────   ───────────────
   16     15.9 ms       1.14 ms    15.9 ms       1,006
   64     16.0 ms       4.57 ms    16.0 ms       4,000
  128     16.8 ms       9.14 ms    16.8 ms       7,619
  270     19.3 ms      19.3  ms    19.3 ms      13,990   ← RIDGE (compute = memory)
  512     23.5 ms      36.6  ms    36.6 ms      13,990   ← FLAT. compute-bound.
  799       —            —            —          —       ← capacity wall
                                                            (67 GB / 83.9 MB)

  plateau throughput = P ÷ FLOPs_per_token = 1.979e15 ÷ 1.413e11 = 14,006 tok/s
```

**Read what just happened.** Same model, same GPU, same precision, two context lengths, two completely different curve shapes:

- At **8k context** the KV read dominates the marginal cost, the curve keeps rising sublinearly, and it stops because you run out of *capacity* at B = 49. It never becomes compute-bound.
- At **512 context** the KV read is negligible, the curve rises nearly linearly to B ≈ 270, then goes **flat** because compute saturates. Capacity is nowhere near the limit (799 sequences would fit).

"Is decode memory-bound?" therefore has the answer *"almost always, but it depends on context length, and the thing that stops you batching is different in the two regimes."* That is a genuinely more useful statement than the slogan, and it is falsifiable from the two-term model.

```
   AGGREGATE DECODE THROUGHPUT vs BATCH SIZE
   Llama-3-70B FP8 weights + FP8 KV, one H200 (141 GB, 4.8 TB/s, 1,979 TFLOP/s)
   ══════════════════════════════════════════════════════════════════════════════

 tok/s
 14000 ┤                                    ╭─────●═════════════════  S=512:
       │                                ╭───╯     ▲    COMPUTE-BOUND  flat at
 12000 ┤                            ╭───╯         │    PLATEAU        P/FLOPs
       │                        ╭───╯         B*≈270                  per token
 10000 ┤                    ╭───╯             compute time
       │                ╭───╯                 = memory time
  8000 ┤            ╭───●
       │        ╭───╯                    ← REGION 2: still memory-bound but
  6000 ┤    ╭───╯                          the weight read is fully amortised;
       │╭───╯                              growth ≈ linear in B
  4000 ●╯
       │   ← REGION 1: weight read (14.6 ms) dominates t_step. Each added
  2000 ┤     sequence is nearly free. Steepest part of the curve.
       │
  1732 ┤                          ●───╳  S=8192: HARD STOP at B=49
  1360 ┤                    ●────╯      (67 GB free ÷ 1.342 GB per seq)
   840 ┤             ●─────╯            curve is CONCAVE, never flattens —
   476 ┤      ●─────╯                   it is cut off by CAPACITY, not compute
    67 ●─────╯
       └──┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴──────▶ batch B
          1    8   16   32   49   64  128  200  270  400  512  799

   THE TWO WALLS, AND HOW TO TELL WHICH ONE YOU HIT
   ────────────────────────────────────────────────
   CAPACITY WALL (S=8192 case)          COMPUTE WALL (S=512 case)
   · throughput still RISING when it    · throughput goes FLAT while HBM
     stops                                still has free capacity
   · engine logs preemption / "KV       · DCGM PIPE_TENSOR_ACTIVE climbs
     cache full" / refuses admission      toward saturation
   · DRAM_ACTIVE high, TENSOR low       · DRAM_ACTIVE falls as B rises
   · FIX: quantise, shrink KV, GQA,     · FIX: lower precision (FP8→FP4),
     bigger-memory SKU                    higher-FLOP SKU. More memory
                                          buys NOTHING.
```

**Batching is close to free throughput in Region 1** because the compute was idle anyway (AI ≈ 1–2, hundreds of times below the ridge), so filling otherwise-wasted tensor-core cycles costs almost no extra wall clock. That is why every serious inference server batches, and why single-stream serving is the most expensive way to run a GPU: you are paying for the whole card to feed one user at bandwidth's mercy.

### 4. Why KV cache caps the batch

From lesson 03.3, the ceiling on B is a capacity statement:

```
B_max ≈ (HBM_capacity − weights − overhead) ÷ (KV_bytes_per_token × seq_len)

  where KV_bytes_per_token = 2 × L × H_kv × d_head × e_kv
```

Every term is a lever, and they are independent:

| Lever | Mechanism | Typical factor | Where it lives |
|---|---|---|---|
| Quantise **weights** | shrinks the subtrahend, freeing capacity for KV | 70 GB → 6 GB free becomes 70 GB → 67 GB free on H200 | serving engine (03.5) |
| Quantise **KV cache** | halves `e_kv`, independent of weight precision | 2× | serving engine, Hopper+ for FP8 KV |
| GQA / MQA model | shrinks `H_kv` | 4–8× | baked into the checkpoint |
| PagedAttention | eliminates fragmentation waste, not a compression ratio | recovers waste that capped batch in pre-paging systems | serving engine (03.3) |
| Prefix sharing | dedupes identical prefixes by reference-counting blocks | workload-dependent, large for shared system prompts | serving engine |
| Bigger-memory SKU | raises the minuend | H100 → H200: free-for-KV 6 GB → 67 GB for 70B FP8 | procurement |

Notice the asymmetry the worked example in 03.3 exposed: **because weights are a fixed subtraction, capacity headroom grows much faster than total capacity.** The H200's 76% more HBM became 11× more free-for-KV space for a 70B FP8 model, and therefore a 12× larger batch. This is the single most under-appreciated number in inference SKU selection.

### 5. When decode itself becomes compute-bound

Batching raises decode's *effective* arithmetic intensity, because one weight read now serves B tokens' worth of math. From lesson 03.2's algebra, the weight-streaming GEMM has `AI ≈ 2B/e`, so the crossover batch is:

```
B* ≈ (ridge point) × e / 2 = (P/BW) × e / 2

  H100 BF16:  (989e12/3.35e12) × 2 / 2 = 295
  H100 FP8:   (1979e12/3.35e12) × 1 / 2 = 295      ← invariant under precision
  H200 FP8:   (1979e12/4.8e12)  × 1 / 2 = 206
  B200 FP4:   (9000e12/7.7e12)  × 0.5 / 2 = 292
```

The refined two-term model of §3 adds the KV-read and attention terms and moves the crossover: at S = 512 on H200 it lands at B ≈ 270 rather than the algebraic 206, and at S = 8,192 it does not exist at all because the marginal sequence's KV read (280 µs) permanently exceeds its compute (82 µs).

**The rule to carry:** in practice you usually hit the *capacity* wall first, which is why decode stays memory-bound and HBM — capacity and bandwidth together — remains the binding constraint. But if you ever see decode throughput plateau **while HBM still has free capacity**, you have crossed into the compute-bound regime and the fix flips completely: from "raise batch / free capacity" to "lower precision / raise FLOPS." Distinguishing those two plateaus is exactly what the diagram in §3 is for.

### 6. Prefill vs decode: two regimes inside one request

A request has two phases with opposite hardware profiles. Compute both intensities and the difference is not a matter of degree.

| Phase | What it does | Tokens per pass | Arithmetic intensity | Bound by |
|---|---|---|---|---|
| **Prefill** | process the whole prompt at once, build the KV cache | all prompt tokens | **high — thousands of FLOP/byte** | **compute** |
| **Decode** | generate output tokens one at a time | 1 per sequence | **≈ 2/e — i.e. 1–2 FLOP/byte** | **memory bandwidth** |

Worked for a 2,048-token prompt to Llama-3-70B at FP8 on an H200:

```
── PREFILL ────────────────────────────────────────────────────────────────
  weight-GEMM FLOPs = 2 · 70e9 · 2048                       = 2.867e14 FLOP
  attention FLOPs   = 4 · S² · d_head · H_q · L
                    = 4 · 2048² · 128 · 64 · 80             = 1.100e13 FLOP
  total                                                     = 2.977e14 FLOP

  bytes = weights read once (70e9) + KV written (2048 × 163,840 = 3.4e8)
                                                            = 7.03e10 B
  AI = 2.977e14 ÷ 7.03e10 = 4,235 FLOP/byte
       vs H200 FP8 ridge of 412  →  10.3× ABOVE.  COMPUTE-BOUND.

  time (at ~60% of the 1,979 TFLOP/s roof) = 2.977e14 ÷ 1.19e15 = 250 ms
  memory time if it were memory-bound      = 7.03e10 ÷ 4.8e12   =  14.6 ms
       → compute takes 17× longer. The memory system is idle during prefill.

── DECODE, the very next millisecond ──────────────────────────────────────
  weight-GEMM FLOPs = 2 · 70e9                              = 1.400e11 FLOP
  attention FLOPs   = 4 · S · d_head · H_q · L
                    = 4 · 2048 · 128 · 64 · 80              = 5.369e9  FLOP
  total per token                                           = 1.454e11 FLOP

  bytes per token = 70e9 (weights) + 2048 × 163,840 (KV read) = 7.03e10 B
  AI = 1.454e11 ÷ 7.03e10 = 2.07 FLOP/byte
       vs the same ridge of 412  →  199× BELOW.  MEMORY-BOUND.

  ideal step time = 7.03e10 ÷ 4.8e12 = 14.6 ms  →  realised ≈ 21 ms (≈70%)

  ⇒ ARITHMETIC INTENSITY FELL BY A FACTOR OF ~2,050 IN ONE STEP,
    with no change of model, GPU, precision, or kernel.
```

```
   ONE REQUEST, TWO REGIMES — the timeline that motivates every serving design
   ═════════════════════════════════════════════════════════════════════════════

   t=0                250 ms                                        ~11 s
   │◀── PREFILL ────▶│◀───────────────── DECODE (512 tokens) ──────────────▶│
   │  2048 prompt    │  one token per ~21 ms, forever
   │  tokens at once │

  ARITHMETIC INTENSITY (log scale)
   4235 ┤████████████████│
        │  compute-bound │
    412 ┤─ ─ ─ ─ ─ ─ ─ ─ ┼─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  ridge
        │                │                                          (H200 FP8)
     10 ┤                │
      2 ┤                └████████████████████████████████████████████████
        │                  memory-bound, flat at ≈2 FLOP/byte forever
      0 └────────────────────────────────────────────────────────────────▶ t

  TENSOR CORES
  100% ┤████████████████│
       │                └░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ~0.5%
    0% └────────────────────────────────────────────────────────────────▶ t

  HBM BANDWIDTH
  100% ┤                ┌████████████████████████████████████████████████  saturated
       │░░░░░░░░░░░░░░░░┘
    6% └────────────────────────────────────────────────────────────────▶ t
        └─ idle during prefill ─┘

  KV CACHE FOR THIS SEQUENCE (monotonically increasing, never shrinks until
                              the request completes)
 400 MiB┤                                                        ╭─────────
        │                                              ╭─────────╯
 370 MiB┤                                    ╭─────────╯
        │                          ╭─────────╯
        │                ╭─────────╯   +160 KiB per token, per sequence
 320 MiB┤       ┌────────╯
        │       │ ← 2048 prompt tokens × 160 KiB = 320 MiB (336 MB) in one shot
      0 └───────┘────────────────────────────────────────────────────────▶ t

  ┌─────────────────────────────────────────────────────────────────────────┐
  │ THE PROBLEM THIS PICTURE STATES                                         │
  │ The same GPU is compute-saturated with idle memory for 250 ms, then      │
  │ memory-saturated with idle tensor cores for 11 seconds. Whatever you     │
  │ buy the card for, it is wrong for half the request. And while one        │
  │ request is in prefill, every other request's decode is BLOCKED — a       │
  │ 250 ms head-of-line stall applied to everyone.                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

### 7. The serving-architecture ladder

Every technique below is a response to the picture above, at increasing levels of aggressiveness.

**(a) Continuous / iteration-level batching (Orca, OSDI 2022).** The naive approach batches at *request* granularity: form a batch, run it to completion, form the next. A batch finishes only when its longest sequence does, so short requests wait and freed slots stay empty. Orca's **iteration-level scheduling** invokes the engine for a *single iteration* at a time, returning control to the scheduler after every token, so finished sequences leave and new ones join mid-flight — keeping the batch, and therefore the amortised weight read, full at all times. Its companion trick, **selective batching**, applies token-wise batching to all non-attention operations while processing attention per sequence, which is necessary because sequences in a batch have different KV lengths and cannot be batched into one attention call naively. Orca reports **36.9× throughput at the same latency** versus FasterTransformer on GPT-3 175B. This is the mechanism vLLM and every modern engine now call "continuous batching."

**(b) Chunked prefill (Sarathi-Serve, OSDI 2024).** Continuous batching fixes the batch-composition problem but not the head-of-line stall: a 250 ms prefill still blocks every in-flight decode. Sarathi-Serve splits a prefill into near-equal chunks (typically **256 or 512 tokens**) and interleaves those chunks with decode iterations to form *hybrid* batches, so decode is never blocked behind a long prefill — the paper calls these **stall-free schedules**. It also improves hardware utilisation, because a decode-only batch leaves tensor cores idle and a prefill chunk fills them. Reported **up to 5.6× higher serving capacity** across models and hardware.

**(c) Prefill/decode disaggregation (DistServe OSDI 2024; Splitwise ISCA 2024; Mooncake FAST 2025).** The most aggressive move: stop running the two phases on the same GPU at all. Run prefill on one pool (compute-optimised, keep tensor cores hot) and decode on another (bandwidth- and capacity-optimised, run large batches), streaming the KV cache between pools over the network. This lets you *buy for each regime independently* — which is where the capacity/bandwidth levers from lesson 03.3 pay off at the architecture level rather than the single-GPU level.

Disaggregation is running in production. Moonshot AI's **Mooncake**, described at FAST 2025, is a KV-cache-centric disaggregated architecture serving Kimi across **thousands of nodes, processing more than 100 billion tokens per day**. In trace-based tests against non-disaggregated baselines under real SLOs it reports **59%–498% higher effective request capacity**, with gains up to 525% in long-context scenarios — the wide range reflecting how much the benefit depends on context length and SLO tightness. NVIDIA productises the same idea in **Dynamo**, reporting in its own benchmarks up to 7× higher throughput on DeepSeek-R1 on Blackwell and up to 30× more requests on a GB200 NVL72; treat those specific multiples as **vendor-reported**, measured against a baseline the vendor chose, not directly comparable to the academic numbers.

### 8. Disaggregation is not a free win

It adds real costs, and a platform engineer's value here is knowing them before recommending the architecture:

- **KV transfer.** Every request's KV cache must move from the prefill pool to the decode pool. For a 2,048-token prompt to Llama-3-70B with FP8 GQA KV that is `2,048 × 163,840 B = 336 MB` per request — at 400 Gb/s (50 GB/s) that is **6.7 ms** of pure transfer, on top of a 250 ms prefill: noise. For a 32k-token prompt it is `32,768 × 163,840 = 5.37 GB` and **107 ms**, which is no longer noise. The transfer cost scales with prompt length, exactly like the prefill it is trying to offload.
- **Two fleets to plan and operate** instead of one, with a ratio between them that must track your actual prompt-to-generation length distribution. Get the ratio wrong and one pool idles while the other queues.
- **A new failure surface**: the KV transfer path, and a request that is now split across two machines' lifetimes.
- **Scale threshold.** The win only materialises past a certain scale or SLO tightness; below it the complexity dominates.

This is precisely the finding of Hao AI Lab's retrospective, ["Disaggregated Inference: 18 Months Later"](https://haoailab.com/blogs/distserve-retro/) — written by the same lab that co-authored DistServe, reviewing how disaggregation actually played out operationally after the initial research result. Treat any single "disaggregation gives Nx" figure (including Mooncake's own 59–498%) as workload- and SLO-dependent, not a multiplier you get by flipping a switch.

### 9. From throughput to cost

```
$ per 1M output tokens = (GPU $/hr) ÷ (aggregate tok/s × 3600) × 1e6
```

Run it across the curve from §3 (H200 at an illustrative ~$3.70/hr; lesson 03.7 covers live pricing):

| Configuration | Aggregate tok/s | tokens/hr | $ per 1M output tokens |
|---|---|---|---|
| 70B FP8, 8k ctx, batch 1 | 67 | 0.24M | **$15.34** |
| 70B FP8, 8k ctx, batch 16 | 840 | 3.02M | $1.22 |
| 70B FP8, 8k ctx, batch 49 (capacity max) | 1,732 | 6.24M | **$0.59** |
| 70B FP8, 512 ctx, batch 270 (compute ridge) | 13,990 | 50.4M | **$0.073** |

**A 210× spread in cost per token, on one GPU, with one model, decided entirely by batch size and context length.** The batch-1 row is what you get from a naive deployment; the batch-49 row is what a properly configured serving engine gives you for free; the last row shows how much of your cost is really a property of your *traffic* (prompt and generation lengths) rather than your infrastructure.

## Perspectives

**Theory.** The whole lesson is the roofline applied twice to the same request. Prefill sits ten times above the ridge, decode two hundred times below it, and batching walks the decode point rightward along the x-axis toward the ridge at a rate of `2/e` FLOP/byte per unit of batch. The two-term time model in §3 is just that statement with the non-shared KV term restored, which is what makes it predictive rather than decorative.

**Practice.** Databricks/MosaicML's ["LLM Inference Performance Engineering: Best Practices"](https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices) is written by a team that operated multiple production inference backends (vLLM, TensorRT-LLM, FasterTransformer) and frames prefill-compute / decode-memory as the operating model for real infrastructure decisions. It is the practitioner's confirmation that the two-phase model is how production teams actually reason, not a textbook simplification.

**Failure mode / hard-won lesson.** Hao AI Lab's 18-months-later retrospective is the necessary counterweight to reading DistServe and Splitwise as "disaggregation always wins." Real deployments pay a KV-transfer cost and operational complexity, and the payoff is scale- and SLO-dependent. Pressure-test any disaggregation pitch against §8 before recommending it.

**Economics.** The §9 table is the argument in one figure: 210× between the worst and best configurations of the *same* GPU. Mooncake's 59–498% capacity gain at Kimi's production scale (>100B tokens/day) is the independent sanity check that these orders of magnitude are real. Note also which lever each row represents — batch 1 → 49 is *configuration*, and free; 8k → 512 context is *product design*, and not yours to choose.

## Real-world use cases

- **Yu et al., ["Orca: A Distributed Serving System for Transformer-Based Generative Models"](https://www.usenix.org/conference/osdi22/presentation/yu) (OSDI 2022).** Introduced iteration-level scheduling and selective batching — what the industry now calls continuous batching — reporting **36.9× throughput at the same latency** versus FasterTransformer on GPT-3 175B. What it shows: the single largest serving win in the last five years came from *scheduling*, not hardware, and it came from noticing that request-granularity batching leaves the amortised weight read half-empty.
- **Agrawal et al., ["Taming Throughput-Latency Tradeoff in LLM Inference with Sarathi-Serve"](https://www.microsoft.com/en-us/research/publication/taming-throughput-latency-tradeoff-in-llm-inference-with-sarathi-serve/) (OSDI 2024).** Chunked prefill (chunks of typically 256–512 tokens) interleaved with decode to form stall-free hybrid batches; **up to 5.6× higher serving capacity**. What it shows: the head-of-line stall in §6's timeline is a real, quantified production problem with a scheduling fix that does not require a second GPU pool.
- **Moonshot AI / Kimi, ["Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving"](https://arxiv.org/abs/2407.00079) (FAST 2025).** Production KV-cache-centric disaggregation serving Kimi at **>100 billion tokens/day across thousands of nodes**, with **59–498% higher effective request capacity** under real SLOs versus non-disaggregated baselines, up to 525% in long-context scenarios. What it shows: this lesson's thesis at the largest publicly documented scale — a live consumer product's serving stack, not a research prototype.
- **Hao AI Lab, ["Disaggregated Inference: 18 Months Later"](https://haoailab.com/blogs/distserve-retro/).** The honest retrospective from the lab that co-authored DistServe — what disaggregation promised versus what held up operationally, including the network and complexity costs §8 draws on. What it shows: the discipline of revisiting your own result after production contact. *(Title and URL verified via search during the 2026-08-15 QA pass.)*
- **Microsoft Research, ["Splitwise improves GPU usage by splitting LLM inference phases"](https://www.microsoft.com/en-us/research/blog/splitwise-improves-gpu-usage-by-splitting-llm-inference-phases/).** The engineering-blog companion to the ISCA 2024 Splitwise paper, making the phase-splitting argument for a production-infrastructure audience — including the observation that the two phases want *different hardware*, which is the procurement consequence of §6.

## Worked example

**Predict, measure, and cost Llama-3-70B decode on one H200 — end to end.**

**Step 1 — the single-stream ceiling, from the spec sheet alone.**

```
model_bytes (FP8) = 70e9 params × 1 B      = 70.0 GB
H200 HBM bandwidth                         =  4.80 TB/s
ceiling = 4.80e12 ÷ 70.0e9                 = 68.6 tok/s
expect measured ≈ 0.70–0.80 × ceiling      = 48–55 tok/s
```

**Step 2 — the capacity budget, from lesson 03.3.**

```
free for KV = 141 − 70 (weights) − 4 (overhead) = 67 GB
KV per token = 2 × 80 L × 8 H_kv × 128 d × 1 B  = 163,840 B = 160 KiB
total live token budget = 67e9 ÷ 163,840        = 409,000 tokens

at 8,192-token sequences:  409,000 / 8,192 =  49 concurrent sequences
at   512-token sequences:  409,000 /   512 = 799 concurrent sequences
```

**Step 3 — build the curve with the two-term model** (the table in §3). The key predictions to check against measurement:

| Prediction | Value | How you falsify it |
|---|---|---|
| Single-stream tok/s | 48–55 | one request, `max_tokens=256`, temperature 0, measure output tokens ÷ (total time − TTFT) |
| Throughput at batch 16, 8k ctx | ~840 tok/s | 16 concurrent clients at 8k prompts |
| Batch at which 8k-context throughput stops | **49** | raise concurrency until the engine logs preemption or refuses admission |
| Shape at 8k ctx when it stops | still **rising** | if it went flat instead, you are compute-bound and mis-modelled |
| Batch at which 512-context throughput plateaus | **~270** | raise concurrency; watch for a flat top with HBM still free |
| Plateau value at 512 ctx | ~14,000 tok/s | `P ÷ FLOPs_per_token = 1.979e15 ÷ 1.413e11` |

**Step 4 — read the diagnosis off DCGM while you sweep.** This is where lesson 03.1's counters earn their keep:

```
batch 1,  8k ctx:  DRAM_ACTIVE ≈ 0.90   TENSOR_ACTIVE ≈ 0.005   → memory-bound
batch 16, 8k ctx:  DRAM_ACTIVE ≈ 0.92   TENSOR_ACTIVE ≈ 0.06    → memory-bound
batch 49, 8k ctx:  DRAM_ACTIVE ≈ 0.93   TENSOR_ACTIVE ≈ 0.17    → memory-bound,
                                                                   stopped by capacity
batch 270, 512 ctx: DRAM_ACTIVE ≈ 0.55  TENSOR_ACTIVE ≈ 0.55    → at the ridge
batch 512, 512 ctx: DRAM_ACTIVE ≈ 0.35  TENSOR_ACTIVE ≈ 0.80    → COMPUTE-bound
```

**`DRAM_ACTIVE` falling while `TENSOR_ACTIVE` rises, as batch grows, is the roofline crossover happening in front of you.** If you see that transition, no further capacity buys throughput.

**Step 5 — sanity-check the batching gain against production evidence.** The 8k-context curve gives `1,732 ÷ 67 = 25.9×` from batching alone versus single-stream. Mooncake's independently measured 59–498% is the *incremental* gain of disaggregation on top of an already-batched baseline, so the two numbers are not measuring the same thing — but both say the same underlying thing: the gap between naive serving and a properly architected system is close to or beyond an order of magnitude, not a tuning percentage.

**Step 6 — cost, and the recommendation.**

```
H200 at ~$3.70/hr, 8k-context traffic, batch 49:
  1,732 tok/s × 3,600 s = 6.235M tokens/hr
  $3.70 ÷ 6.235M × 1e6  = $0.59 per 1M output tokens

Same card, batch 1 (a naive deployment):
  67 tok/s × 3,600      = 241,200 tokens/hr
  $3.70 ÷ 0.2412M × 1e6 = $15.34 per 1M output tokens

Delta: 26× — achieved by setting a concurrency limit correctly.
```

Now the comparison that decides a purchase. Repeat step 2 for an **H100 80 GB**: free-for-KV = 80 − 70 − 4 = 6 GB → 36,700 tokens → **4** concurrent sequences at 8k context. Two-term model at B = 4: memory time = `(70e9 + 4 × 1.342e9) ÷ 3.35e12` = 22.5 ms → 178 tok/s aggregate.

```
H100 @ ~$3.00/hr:   178 tok/s → 0.641M tok/hr → $4.68 per 1M tokens
H200 @ ~$3.70/hr: 1,732 tok/s → 6.235M tok/hr → $0.59 per 1M tokens
                                                 ─────────────────────
                                    H200 is 7.9× cheaper per token
                                    for 1.23× the hourly rate.
```

**And note where that 7.9× came from.** Decompose the 9.74× throughput ratio by changing one thing at a time: hold the batch at 4 and move only to the H200's bandwidth → `(70e9 + 4 × 1.342e9) ÷ 4.8e12 = 15.70 ms` → 255 tok/s, a **1.43×** gain, exactly the bandwidth ratio. Then raise the batch from 4 to 49 on the H200 → 1,732 tok/s, a further **6.80×**. `1.43 × 6.80 = 9.74` ✓. So **bandwidth contributed 1.4× and capacity-enabled batching contributed 6.8×.** If you argued this purchase on bandwidth alone you would have claimed 43% and lost the argument on price. *(Rates are illustrative 2026-era snapshots; the ratio is set by hardware, the absolute by the market.)*

## Practice

Rent a GPU (H100 80 GB ideal; adjust spec numbers to your SKU) and serve with **vLLM**. Budget ~1.5 GPU-hours. Results feed the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md).

1. **Predict before you measure.** Write down, from the spec sheet and the model config alone: the single-stream ceiling (`BW ÷ model_bytes`), the KV bytes per token, the token budget, and `B_max` at two context lengths. Do this *first* — the exercise is worthless if you fit the model to the data afterwards.
2. **Serve a model** you can fit (an 8B–13B at FP16, or a quantised larger model). `vllm serve <model>` exposes an OpenAI-compatible endpoint. Record the startup log line reporting **GPU KV cache size** and compare it with your hand-computed token budget.
3. **Measure single-stream decode tok/s.** One request, `max_tokens=256`, `temperature=0`. Record output tokens ÷ (end-to-end time − time-to-first-token), so prefill is excluded. Compute measured ÷ ideal and account for the gap using the itemised list in §2.
4. **Sweep concurrency:** 1, 2, 4, 8, 16, 32, 64, 128, … using `benchmarks/benchmark_serving.py` or an async client. Record aggregate output tok/s at each level, at a **fixed** prompt and generation length.
5. **Find the knee, and identify which wall it is.** Keep raising concurrency until aggregate throughput stops rising. Then classify: was throughput still climbing when the engine began preempting or refusing (**capacity wall**), or did it go flat while `nvidia-smi` still showed free HBM (**compute wall**)? Capture the vLLM preemption log line or KV-cache-usage gauge as evidence.
6. **Repeat the sweep at a second context length** — one short (≈512) and one long (≈8k). You should get visibly different curve *shapes*, per §3. This is the step that proves you understand the mechanism rather than the slogan.
7. **Cross-check with DCGM** at three points on each curve: `DCGM_FI_PROF_DRAM_ACTIVE` and `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`. Confirm DRAM_ACTIVE stays high through a capacity-limited sweep, and falls as TENSOR_ACTIVE rises through a compute-limited one.
8. **(Stretch, no implementation needed.)** For your measured workload, compute the KV bytes a disaggregated deployment would have to move per request (`prompt_len × KV_bytes_per_token`), convert to milliseconds at 50 GB/s, and state whether that transfer would be noise or a material fraction of your measured TTFT. That number is the entire go/no-go on disaggregation for your workload.

**Acceptance:** a **decode-throughput-versus-batch-size curve** (table or plot: concurrency on x, aggregate tok/s on y) for one model on one SKU **at two context lengths**, annotated with:

- (a) the predicted single-stream ceiling and the measured value, with the ratio and an explanation of the gap;
- (b) the hand-computed `B_max` and the observed knee, with the discrepancy explained;
- (c) which wall each curve hit — capacity or compute — with the DCGM or log-line evidence that proves it;
- (d) a cost-per-1M-output-tokens figure at the best sustained throughput and at batch 1, so the ratio is visible.

## Common pitfalls

1. **Assuming aggregate throughput is `B × single-stream`.** It is not — the KV read is not amortised. At 8k context on an H200, batch 49 gives 25.9× the single-stream throughput, not 49×, because KV bytes have grown to 65.8 GB against 70 GB of weights. Use the two-term model. *Symptom:* your capacity plan over-predicts by 2× at high batch.
2. **Believing batching benefits are unbounded.** The curve rises steeply then either hits a capacity wall or flattens at a compute plateau. Which one you hit depends on context length, and the fixes are opposite.
3. **Assuming the wall is always capacity.** At short context on a big-memory card, decode genuinely crosses to compute-bound (B ≈ 270 in §3's example, with 67% of the KV budget still unused). Buying more memory there does nothing. Check whether HBM was actually full before prescribing capacity.
4. **Assuming the wall is never compute, because "decode is memory-bound."** Same error, opposite direction. "Decode is memory-bound" is a statement about batch 1; it stops being true somewhere, and the two-term model tells you where.
5. **Applying prefill-side reasoning to decode, or vice versa.** Prefill is already compute-saturated, so batching more prompts into it does not raise throughput — you are FLOP-limited, not amortising a memory read. Decode is the opposite. A fix aimed at one regime can do nothing or hurt in the other.
6. **Assuming disaggregation is a free win.** It adds a per-request KV transfer (320 MB for a 2k prompt at FP8 GQA; 5.1 GB for a 32k prompt), two fleets to operate, and a new failure surface, and it pays off only past a scale/SLO threshold. Hao AI Lab's retrospective is the source to cite before recommending it for a modest deployment.
7. **Treating vendor-reported multiples as comparable to measured research numbers.** NVIDIA's Dynamo 7×/30× figures are the vendor's own benchmarks against a baseline they chose; Orca's 36.9×, Sarathi-Serve's 5.6× and Mooncake's 59–498% come from academic or production measurement against stated baselines. Always ask "against what baseline" before comparing two "Nx" claims.
8. **Benchmarking with variable prompt and generation lengths and then reporting one throughput number.** Throughput is a function of *(batch, context)*, and §3 shows the curve shape changes qualitatively with context. A single number from mixed traffic is not reproducible and cannot be extrapolated.
9. **Measuring throughput including prefill and calling it decode throughput.** For a 2k-prompt/256-token request the prefill is ~250 ms of a ~5.6 s request — 4.5% of the time but 100% of the compute-bound portion. Exclude TTFT when measuring decode.

## Self-check

- **Estimate the single-stream tok/s ceiling for a 70B model at FP16 on an H100. Show the arithmetic.** Decode reads every weight per token, so `tok/s ≤ HBM_bandwidth ÷ model_bytes`. Weights = 70e9 × 2 B = 140 GB; H100 bandwidth = 3.35 TB/s; `3.35e12 ÷ 1.40e11 = 23.9 tok/s`. FLOPS never enter, because the arithmetic intensity is `2N/(e·N) = 1 FLOP/byte`, roughly 295× below H100's BF16 ridge. Expect measured ~16–20 tok/s (70–80%), the gap coming from KV reads, sub-spec achieved bandwidth, kernel-launch overhead and an attention kernel that cannot saturate bandwidth at batch 1.
- **Why does batching help decode but not a single long prefill?** Decode's per-token cost is dominated by reading the weights, a fixed cost per forward pass that can be shared across many sequences: batch B reads the weights once and emits B tokens. Because decode's arithmetic intensity is ~1–2 FLOP/byte, the tensor cores were idle anyway, so the extra math is nearly free — that is Region 1 of the curve. A prefill is already compute-bound: one weight read already feeds math over every prompt token at once, giving thousands of FLOP/byte and saturated tensor cores. There is no idle compute to fill, so adding work does not raise throughput; you are FLOP-limited.
- **Write the two-term model for a decode step and say what each term does to the curve.** `t_step(B) = max( (W + B·S·k)/BW , B·(2N + a·S)/P )`. The memory term has a *shared* part (W, the weight read) and a *per-sequence* part (B·S·k, the KV read); the shared part is what batching amortises and it produces the steep initial rise, while the per-sequence part makes the curve concave. The compute term is entirely per-sequence and linear in B, so it eventually overtakes and produces a flat plateau at `P ÷ FLOPs_per_token`. Whether you reach that plateau or hit `B_max = (capacity − weights − overhead)/(S·k)` first depends on context length.
- **At what batch does decode become compute-bound, and why is the answer precision-invariant in the simple model?** `B* ≈ (P/BW) × e/2`. On H100: BF16 gives `295 × 2/2 = 295`; FP8 gives `591 × 1/2 = 295`. Quantisation halves the bytes per parameter (doubling arithmetic intensity for a given batch) *and* doubles the compute roof (doubling the ridge), so the crossover batch is unchanged. What FP8 does buy is 2× throughput everywhere below the crossover — which is where inference lives. The refined two-term model moves the crossover somewhat (to ≈270 at 512 context on H200) because it accounts for KV reads and attention FLOPs, and abolishes it entirely at long context, where the marginal sequence's KV read (280 µs at 8k) exceeds its compute (82 µs).
- **A serving deployment's throughput stops rising at batch 40. How do you tell whether to buy memory or FLOPs?** Look at whether it *stopped* or *flattened*, and at what HBM was doing. If throughput was still climbing when the engine began preempting or refusing admission, and `nvidia-smi` shows HBM full and `DRAM_ACTIVE` high with `TENSOR_ACTIVE` low, it is the **capacity wall** — buy memory, quantise, or shrink KV. If throughput went flat while HBM still had free capacity, and `TENSOR_ACTIVE` is climbing toward saturation while `DRAM_ACTIVE` falls, it is the **compute wall** — buy FLOPs or drop precision; more memory buys nothing.
- **Quantify the prefill-to-decode transition for a 2k-token prompt to a 70B model at FP8.** Prefill: `2·70e9·2048 = 2.87e14` weight FLOPs plus `4·2048²·128·64·80 = 1.10e13` attention FLOPs = 2.98e14 FLOP, against 7.03e10 bytes (weights read once plus KV written) → AI ≈ 4,235 FLOP/byte, ten times *above* the H200 FP8 ridge of 412 → compute-bound, ~250 ms. Decode, immediately after: ~1.45e11 FLOP against the same ~7.03e10 bytes → AI ≈ 2 FLOP/byte, 199× *below* the ridge → memory-bound, ~21 ms per token. Arithmetic intensity falls by roughly 2,000× in a single step with no change of model, hardware, precision or kernel — which is the whole reason the two phases want different machines.
- **What did Orca change, and why was the gain so large?** Orca (OSDI 2022) replaced request-granularity batching with **iteration-level scheduling**: the engine runs one iteration and returns to the scheduler, so completed sequences leave the batch and new ones join mid-flight instead of the batch waiting for its longest member. **Selective batching** makes this possible by batching non-attention ops token-wise while handling attention per sequence, since batched sequences have different KV lengths. The gain (36.9× at equal latency versus FasterTransformer on GPT-3 175B) was large because request-granularity batching leaves the amortised weight read serving a mostly-empty batch for most of its life — and the weight read is the entire cost of decode.
- **Mooncake reports 59–498% higher capacity from disaggregation. Mechanism, and why is the range so wide?** The mechanism is running prefill and decode on separate pools so neither blocks or wastes the other's resources, with a KV-cache-centric design that uses otherwise-idle CPU, DRAM, SSD and network capacity for a distributed cache. The range is wide because the benefit scales with context length and SLO tightness — long-context, tightly-SLO'd traffic (where prefill stalls hurt most and KV reuse pays most) benefits far more than short, loosely-SLO'd traffic. That workload dependence is exactly why Hao AI Lab's retrospective stresses that disaggregation's payoff is not a fixed multiplier.

## Connections & what's next

This lesson is the direct continuation of 03.3: the bandwidth ceiling derived there is the single-stream number here, and the KV-cache token budget derived there is the batch cap here — capacity and bandwidth are literally the two inputs to every calculation above. It applies 03.2's roofline twice within one request and shows the crossover happening as a function of batch. It previews 03.5 (precision), the next multiplier on both the decode ceiling (FP8 roughly doubles it) and the batch cap (quantised KV frees capacity for more sequences) — the same two-lever structure one level deeper. And it sets up module 07 (inference serving), where continuous batching, chunked prefill and disaggregation become full production system design, and module 11 (GPU cost economics), where the cost-per-token arithmetic here becomes a fleet-wide capacity-planning model.

Next: **[03.5 · Precision & tensor cores](05-precision-and-tensor-cores.md)** takes the FP8-halves-the-bytes, FP8-doubles-the-roof observation used repeatedly here and derives it properly — including exactly why the *realised* throughput gain (Databricks reports ~1.5×) is smaller than the raw tensor-core multiple (~2×), the same theory-versus-amortised-reality gap this lesson taught you to expect and to itemise.

## References & further reading

**Primary sources**

1. Yu et al., ["Orca: A Distributed Serving System for Transformer-Based Generative Models"](https://www.usenix.org/conference/osdi22/presentation/yu) (OSDI 2022) — iteration-level scheduling and selective batching, the origin of continuous batching; **36.9× throughput at equal latency** versus FasterTransformer on GPT-3 175B. Read §3–4 for why request-granularity batching wastes the amortised weight read.
2. Agrawal et al., ["Taming Throughput-Latency Tradeoff in LLM Inference with Sarathi-Serve"](https://www.microsoft.com/en-us/research/publication/taming-throughput-latency-tradeoff-in-llm-inference-with-sarathi-serve/) (OSDI 2024) — chunked prefill (typically 256–512-token chunks) and stall-free hybrid batches; **up to 5.6× higher serving capacity**. The fix for the head-of-line stall in this lesson's timeline diagram.
3. Kwon et al., ["Efficient Memory Management for Large Language Model Serving with PagedAttention"](https://arxiv.org/abs/2309.06180) (SOSP 2023) — the KV-cache paging mechanism that makes the `B_max` formula achievable in practice rather than in theory; **2–4× throughput** over FasterTransformer and Orca by eliminating fragmentation waste.
4. Zhong et al., ["DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving"](https://arxiv.org/abs/2401.09670) (OSDI 2024) — the research paper establishing prefill/decode disaggregation as a serving architecture, and the origin of the retrospective cited below.
5. Patel et al., ["Splitwise: Efficient Generative LLM Inference Using Phase Splitting"](https://arxiv.org/abs/2311.18677) (Microsoft, ISCA 2024) — an independent disaggregation architecture and evaluation, notable for arguing the two phases want *different hardware*.
6. Moonshot AI, ["Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving"](https://arxiv.org/abs/2407.00079) (FAST 2025) — production-scale disaggregation behind this lesson's headline numbers: >100B tokens/day across thousands of nodes, 59–498% higher effective request capacity under real SLOs (up to 525% long-context).
7. Williams, Waterman & Patterson, ["Roofline: An Insightful Visual Performance Model for Multicore Architectures"](https://dl.acm.org/doi/10.1145/1498765.1498785) (CACM 52(4), 2009; free mirror at [escholarship.org](https://escholarship.org/content/qt78h8v7mr/qt78h8v7mr.pdf)) — the formal ridge-point and operational-intensity definitions this lesson's crossover analysis relies on.
8. Kaplan et al., ["Scaling Laws for Neural Language Models"](https://arxiv.org/abs/2001.08361) — the `2N` forward-pass FLOP-per-token convention used in every intensity calculation above.
9. NVIDIA, [H100](https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper) and [H200](https://www.nvidia.com/en-us/data-center/h200/) datasheets/whitepapers — source of truth for the 989/1,979 dense TFLOP/s, 3.35 TB/s and 4.8 TB/s figures used in every calculation.

**Real-world engineering blogs**

10. Microsoft Research, ["Splitwise improves GPU usage by splitting LLM inference phases"](https://www.microsoft.com/en-us/research/blog/splitwise-improves-gpu-usage-by-splitting-llm-inference-phases/) — what it shows: the Splitwise research made concrete for a production-infrastructure audience, including the procurement consequence.
11. Databricks/MosaicML, ["LLM Inference Performance Engineering: Best Practices"](https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices) — what it shows: the prefill-compute/decode-memory model confirmed from real multi-backend serving experience (vLLM, TensorRT-LLM, FasterTransformer).
12. Hao AI Lab, ["Disaggregated Inference: 18 Months Later"](https://haoailab.com/blogs/distserve-retro/) — what it shows: the honest retrospective on disaggregation's real operational costs and when it is not worth it, from the same lab that co-authored DistServe. *(Title and URL verified via search during the 2026-08-15 QA pass.)*
13. ["Prefill is compute-bound, decode is memory-bound — why your GPU shouldn't do both"](https://towardsdatascience.com/prefill-is-compute-bound-decode-is-memory-bound-why-your-gpu-shouldnt-do-both/) — Towards Data Science — a full walkthrough of the two-regime model that motivates disaggregation; read alongside Horace He.

**Deeper dives**

14. NVIDIA Dynamo [documentation](https://docs.nvidia.com/dynamo/) — the vendor's productised disaggregated-serving framework. Treat its 7× (DeepSeek-R1 on Blackwell) and 30× (GB200 NVL72) throughput claims as vendor-reported benchmarks against a baseline the vendor chose, not independently verified against the same baselines as the academic numbers above.
15. [vLLM documentation](https://docs.vllm.ai/) — continuous/inflight batching, PagedAttention, chunked prefill, `gpu_memory_utilization`, `max_num_seqs`, preemption metrics, and `benchmarks/benchmark_serving.py` for the practice section. The fastest way to check every prediction in this lesson against a real engine.
