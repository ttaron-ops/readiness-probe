---
lesson: "03.5"
title: "Precision Formats & Tensor Cores"
module: "03"
concept: "Precision & tensor-core throughput"
status: not-started
est_time: "7h"
prev: "04-decode-bandwidth-batching.md"
next: "06-generational-and-software-stack.md"
artifacts: []
sources: 16
---

# 03.5 · Precision Formats & Tensor Cores

> **Concept.** Numeric precision sets your memory footprint and your roofline ceiling; tensor cores turn low precision into throughput — but only for dense matmul, and FP8 is a direct $/token lever.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Where this fits

Lesson 4 established the decode bandwidth ceiling and showed batching and prefill/decode disaggregation as the ways to push a memory-bound workload back toward the roofline's ridge point. This lesson hands you a different lever entirely: instead of restructuring how a workload runs, you can move the ridge point itself. Numeric precision is the one decision that touches both axes of the roofline at once — it shrinks the bytes every memory-bound kernel from lessons 3 and 4 has to move, *and* it raises the FLOPS ceiling every compute-bound kernel from lesson 2 can reach. Lesson 6 will show this lever is also gated by which hardware generation you're renting — FP8 requires Hopper or newer, FP4 requires Blackwell — so the precision conclusion you reach here determines what SKU argument you're even allowed to make there.

## Why this matters

Precision is where "reads a spec sheet" and "reasons about cost" diverge. The NVIDIA Solutions Architect role this module calibrates against explicitly names "inference optimization via FP16/INT8/FP8" and "arithmetic intensity vs peak compute and memory bandwidth" as core competencies — and the standard interview probe is blunt: *"FP8 vs FP16 — what's the cost lever, and what do you actually get?"* Answering "roughly 2x" is the answer that gets you screened out at a GPU-fleet operator, because the real answer is "2x on the tensor-core-bound matmuls, and something meaningfully less end-to-end, and here's why."

The concrete cost of not knowing this: quoting the *sparse* FP8 TFLOPS figure as your denominator silently doubles your apparent efficiency in a report someone else will act on. Assuming a precision flag "just works" the same way on an A100 as an H100 is the kind of mistake that turns into a failed deploy at 2 a.m., because A100 (Ampere) has **no FP8 tensor-core hardware at all** — FP8 is a Hopper-generation feature. And treating a vendor's own 1.5x throughput claim as if it contradicts a textbook 2x number — instead of recognizing they're measuring different things — is the kind of confusion that makes a report look sloppy in front of a hiring panel that has actually shipped an FP8 training run.

There is a second, quieter cost. Precision failures are *numerical*, and numerical failures do not announce themselves. An FP16 training run that silently flushes 30% of its gradients to zero does not crash; it converges to a worse model three days later. An FP8 inference deployment whose activation scale factors were calibrated on the wrong traffic distribution does not error; it degrades quality on the tail of the request distribution that nobody benchmarks. If you cannot reason about *what each format can and cannot represent*, you cannot tell those apart from ordinary model-quality noise, and you will spend a week blaming the data pipeline.

## What's new here (calibration)

The module README skips CUDA kernel authoring, PTX/SASS, deep microarchitecture, and occupancy-tuning-as-coding — you are not writing the matmul kernel or hand-tuning register allocation. This lesson holds that line: it does not teach you to write a quantization calibration routine or a custom FP8 GEMM. What it adds instead:

- **The encoding rule itself**, so that every format's max, min, and precision is something you *derive* in ten seconds rather than a table you memorize and misremember under pressure.
- **Translating a precision decision into capacity, placement, and cost** — bytes/param arithmetic that decides "one GPU or two," not the quantization math itself.
- **What a tensor core actually computes**, at the level of the matrix tile shape and the accumulator width — enough to explain why the multiplier is exactly 2x per format halving on some generations and not on others, and why "accumulate precision" is a real, load-bearing hardware property that has bitten frontier labs.
- **Observability into whether the precision change is actually landing** — DCGM's tensor-core-active signal as the ground truth that a "we switched to FP8" claim is real, not aspirational.
- **The gap between the theoretical multiplier and the realized one, quantified** — Databricks' verified ~1.5x end-to-end throughput against the ~2x tensor-core multiple, and why both numbers are correct at once.
- **PTQ vs. QAT as a named operational trade-off** — you don't own the accuracy call, but a Staff-level platform engineer can name that Character.AI chose native int8 *training* over post-hoc quantization specifically to eliminate train/serve mismatch, and explain why that's a real, deliberate architectural choice and not a detail.

## Core concepts

### 1. The problem a floating-point format exists to solve

A neural network has to represent numbers spanning an enormous range. Weights sit around 10⁻² to 10⁻¹. Activations after a GELU can be tens. Gradients late in training routinely reach 10⁻⁷ and below. Attention logits before softmax can hit hundreds. No fixed-point representation covers all of that with a single scale factor, which is why deep learning runs on floating point rather than integers.

A floating-point format spends its bits on a deliberate trade. Every format is three fields:

```
  [ S ][      E (exponent)      ][      M (mantissa/significand)      ]
    1 bit        e bits                        m bits
```

and the value it encodes follows one rule:

```
  if 0 < E < (2^e - 1):        value = (-1)^S × 2^(E - bias) × (1 + M/2^m)      ← NORMAL
  if E == 0 and M != 0:        value = (-1)^S × 2^(1 - bias) × (0 + M/2^m)      ← SUBNORMAL
  if E == 0 and M == 0:        value = ±0
  if E == (2^e - 1):           ±Inf (M == 0) or NaN (M != 0)     ← IEEE 754 convention

  bias = 2^(e-1) - 1
```

Three consequences fall straight out of that rule, and they are the whole lesson:

1. **The exponent field buys range.** Range is governed *only* by `e`. Every extra exponent bit doubles the number of binades (powers of two) the format can reach, so range grows *exponentially* in `e`.
2. **The mantissa field buys precision.** The gap between adjacent representable numbers near a value `x` is `x × 2^-m`. That quantity — `2^-m` — is the format's **relative precision** (machine epsilon, ε). It is *scale-invariant*: FP16 resolves 1.0 and 1000.0 to the same 0.1% relative accuracy, right up until it overflows.
3. **Range and precision are independent problems, and you buy them separately.** This is why two 16-bit formats exist. FP16 and BF16 are the same size and are *not* interchangeable, because they made opposite choices with the same 15 non-sign bits.

The "implicit leading 1" in the normal case is worth pausing on: since a normalized significand always starts with a 1, the format doesn't store it, buying one bit of precision for free. Subnormals are the escape hatch when the exponent has bottomed out — the leading bit becomes 0 and the format trades precision for a few more decades of reach toward zero. Whether subnormals are supported at all, and whether the hardware handles them at full rate, is a per-format, per-architecture property.

### 2. Every format you will meet, drawn field by field

```
  BIT LAYOUTS — sign | exponent | mantissa    (widths to scale, one cell = one bit)

  FP32 / IEEE 754 binary32                                          4 bytes
  ┌─┬────────────────┬──────────────────────────────────────────────────────┐
  │S│ E: 8 bits      │ M: 23 bits                                           │
  └─┴────────────────┴──────────────────────────────────────────────────────┘
     bias 127          ε = 2^-23 = 1.19e-7      max 3.40e38   min-nor 1.18e-38

  TF32 (Ampere+ tensor-core internal format; STORED in a 4-byte FP32 container)
  ┌─┬────────────────┬──────────────────────┐······································┐
  │S│ E: 8 bits      │ M: 10 bits           │ (13 mantissa bits discarded on read) │
  └─┴────────────────┴──────────────────────┘······································┘
     bias 127          ε = 2^-10 = 9.77e-4     max 3.40e38   min-nor 1.18e-38
     ── FP32's range, FP16's precision, FP32's memory cost. 19 bits of real content.

  BF16 / bfloat16                                                    2 bytes
  ┌─┬────────────────┬──────────────┐
  │S│ E: 8 bits      │ M: 7 bits    │
  └─┴────────────────┴──────────────┘
     bias 127          ε = 2^-7 = 7.81e-3      max 3.39e38   min-nor 1.18e-38
     ── literally FP32 with the bottom 16 mantissa bits sawn off. Truncating
        FP32→BF16 is a pointer cast; that is the entire point of the format.

  FP16 / IEEE 754 binary16                                           2 bytes
  ┌─┬──────────┬──────────────────────┐
  │S│ E: 5 bits│ M: 10 bits           │
  └─┴──────────┴──────────────────────┘
     bias 15           ε = 2^-10 = 9.77e-4     max 65504     min-nor 6.10e-5
     ── 8× more precise than BF16, 20 orders of magnitude less range.

  FP8 E4M3 (OCP / NVIDIA "e4m3fn")                                   1 byte
  ┌─┬────────┬──────┐
  │S│ E: 4   │ M: 3 │
  └─┴────────┴──────┘
     bias 7            ε = 2^-3 = 0.125        max 448       min-nor 1.56e-2
     ── NO infinities. Only S.1111.111 is NaN. Reclaiming those codes is what
        buys the top binade, taking max from 240 to 448.

  FP8 E5M2                                                           1 byte
  ┌─┬──────────┬────┐
  │S│ E: 5     │M: 2│
  └─┴──────────┴────┘
     bias 15           ε = 2^-2 = 0.25         max 57344     min-nor 6.10e-5
     ── IEEE-shaped: real ±Inf and NaN. It is FP16 with 8 mantissa bits removed,
        so its exponent range is IDENTICAL to FP16's.

  FP4 E2M1 (element format inside NVFP4 / MXFP4)                    0.5 bytes
  ┌─┬────┬─┐
  │S│E: 2│M│
  └─┴────┴─┘
     bias 1            ε = 2^-1 = 0.5          max 6.0       min-nor 1.0
     ── 16 codes TOTAL: ±{0, 0.5, 1, 1.5, 2, 3, 4, 6}. No Inf, no NaN.
        Unusable alone; only viable with a per-block scale factor (§9).
```

Read that picture as a single story: **going from FP32 down to FP8-E4M3 costs you 20 mantissa bits (precision drops from 1 part in 8.4 million to 1 part in 8) and 4 exponent bits (range collapses from 83 decades to about 5).** The formats in between are different answers to "which of those two losses can my workload tolerate?"

### 3. The numeric-format table — the one to actually keep

Every value below is derived from the encoding rule in §1; you can regenerate the whole table on a whiteboard from `e`, `m`, and `bias`.

| Format | Bits S/E/M | Bytes | Bias | Max normal | Min normal | Min subnormal | ε (rel. precision) | Dynamic range | Tensor-core HW support |
|---|---|---|---|---|---|---|---|---|---|
| **FP64** | 1/11/52 | 8 | 1023 | 1.80e308 | 2.23e-308 | 4.94e-324 | 2.22e-16 | ~632 decades | FP64 tensor core: Ampere+ (A100 19.5 TF, H100 67 TF) |
| **FP32** | 1/8/23 | 4 | 127 | 3.40e38 | 1.18e-38 | 1.40e-45 | 1.19e-7 | ~83 decades | No — runs on the FP32 CUDA cores, not tensor cores |
| **TF32** | 1/8/10 | 4 (storage) | 127 | 3.40e38 | 1.18e-38 | n/a | 9.77e-4 | ~83 decades | Ampere, Hopper, Blackwell (matmul-internal only) |
| **BF16** | 1/8/7 | 2 | 127 | 3.39e38 | 1.18e-38 | 9.18e-41 | 7.81e-3 | ~78 decades | Ampere, Hopper, Blackwell (**not** Volta/Turing) |
| **FP16** | 1/5/10 | 2 | 15 | 6.55e4 | 6.10e-5 | 5.96e-8 | 9.77e-4 | ~12 decades | Volta onward — the original tensor-core format |
| **FP8 E4M3** | 1/4/3 | 1 | 7 | 448 | 1.56e-2 | 1.95e-3 | 0.125 | ~5.4 decades | Hopper, Ada, Blackwell (**not** Ampere) |
| **FP8 E5M2** | 1/5/2 | 1 | 15 | 5.73e4 | 6.10e-5 | 1.53e-5 | 0.25 | ~9.6 decades | Hopper, Ada, Blackwell (**not** Ampere) |
| **INT8** | signed integer | 1 | — | 127 | — | — | absolute, 1 LSB | 1 (needs a scale) | Turing onward |
| **INT4** | signed integer | 0.5 | — | 7 | — | — | absolute, 1 LSB | 1 (needs a scale) | Turing, Ampere (dropped from Hopper TC) |
| **FP4 E2M1** | 1/2/1 | 0.5 | 1 | 6.0 | 1.0 | 0.5 | 0.5 | ~1.1 decades | Blackwell only (5th-gen tensor core) |

*Dynamic range* here is `max normal ÷ min subnormal`, expressed in decimal orders of magnitude — the honest measure of "what can this format hold at all before it saturates or flushes to zero."

Two entries deserve a second look because they are the ones people get wrong in interviews.

**TF32 is a compute mode, not a storage format.** Your tensor stays FP32 in HBM, 4 bytes per element. When a matmul runs, the Ampere-or-newer tensor core reads the FP32 operand, *truncates the mantissa from 23 bits to 10 in hardware*, multiplies at that reduced width, and accumulates back into FP32. You get FP32's dynamic range and roughly an order of magnitude more matmul throughput than the FP32 CUDA-core path, at zero memory saving and zero code change. The correct one-line summary: **TF32 buys FLOPS, never bytes.** If a colleague says "we moved to TF32 to fit the model," they have made an error.

**FP8 E4M3 deliberately breaks IEEE 754.** Under the standard rule, exponent field `1111` would be reserved entirely for Inf/NaN, capping E4M3 at 240. NVIDIA, Arm and Intel's joint format spec (Micikevicius et al., *FP8 Formats for Deep Learning*, arXiv 2209.05433, 2022) instead spends that exponent field on ordinary numbers, keeping only the single pattern `S.1111.111` as NaN and dropping infinities entirely. That reclamation is exactly what takes the maximum from 240 to **448** — an extra binade, bought by giving up encodings the format could not afford. E5M2, by contrast, keeps the IEEE convention (real ±Inf, a full family of NaNs), which is why its max stops at 57344 rather than 65504: it loses the top binade to the reserved exponent, just as FP16 does not.

### 4. Range vs. precision: which one your workload actually needs

Here is the decision, stated as a rule you can apply without looking anything up: **look at the dynamic range of the tensor, not its magnitude.**

- **Weights** are the easy case. A trained layer's weights are roughly zero-centred with a tight distribution — typically within two or three orders of magnitude end to end. Nearly every format holds them; what varies is how much *precision* you keep, which is why weight-only quantization to INT8 or FP8 is close to free.
- **Activations** are harder. Post-GELU or post-softmax tensors are asymmetric and have long tails — outlier channels in transformer activations are a well-documented phenomenon, and a single 100× outlier in a tensor forces the scale factor to accommodate it, crushing every other value toward the bottom of the format.
- **Gradients** are the hard case, and they are why FP16 needs help. During backprop, gradient magnitudes fall as training progresses. FP16's minimum normal value is `2^-14 ≈ 6.1e-5`, and its subnormals bottom out at `2^-24 ≈ 6e-8`. Gradients below that are **flushed to exactly zero** — not rounded, annihilated. NVIDIA's own mixed-precision training guidance shows histograms where a substantial fraction of a real network's gradient values sit in the region FP16 cannot represent, while the top ~15 binades of FP16's range go completely unused.

That observation is the whole idea behind **loss scaling**, and it is elegant precisely because it costs nothing: multiply the loss by a constant `S` before calling backward. By the chain rule every gradient in the graph is scaled by exactly `S`, shifting the entire gradient histogram *up* into FP16's representable window. Divide the gradients by `S` again — in FP32 — before the optimizer step, and the math is unchanged.

### 5. The mixed-precision flow, end to end

This is the diagram to hold in your head, because every framework's AMP implementation is a variation on it. Follow one training step:

```
  MIXED-PRECISION TRAINING STEP — where each precision lives and why

  ┌────────────────────────────────────────────────────────────────────────┐
  │ FP32 MASTER WEIGHTS  (kept for the whole run; 4 bytes/param)           │
  │  reason: an optimizer update is w -= lr*g, and lr*g is often ~1e-8 of  │
  │  w. In BF16 (ε=7.8e-3) that update rounds to ZERO and training stalls. │
  └───────────────┬────────────────────────────────────────────────────────┘
                  │ cast (FP8: cast + scale)
                  ▼
  ┌────────────────────────────────────────────────────────────────────────┐
  │ LOW-PRECISION WEIGHT COPY  →  FORWARD PASS                             │
  │  matmuls/convs: tensor cores, BF16 or FP8 in, FP32 accumulate          │
  │  layernorm / softmax / loss: stay FP32 (reductions, exp(), 1/sqrt)     │
  └───────────────┬────────────────────────────────────────────────────────┘
                  ▼
              loss (FP32)
                  │
       ┌──────────┴───────────┐
       │  × S  (loss scale)   │   FP16 path only. BF16 usually skips this:
       │  init 65536 in torch │   BF16's exponent range == FP32's, so there
       │  AMP GradScaler      │   is no underflow cliff to climb away from.
       └──────────┬───────────┘
                  ▼
  ┌────────────────────────────────────────────────────────────────────────┐
  │ BACKWARD PASS — every grad is now S× larger, lifted off the FP16 floor │
  └───────────────┬────────────────────────────────────────────────────────┘
                  ▼
        ┌──────────────────────┐
        │ inspect grads for    │
        │ Inf / NaN            │
        └───┬──────────────┬───┘
     found  │              │  clean
            ▼              ▼
   ┌────────────────┐  ┌─────────────────────────────────────┐
   │ SKIP the step  │  │ unscale: g ← g / S  (in FP32)       │
   │ S ← S / 2      │  │ clip → optimizer → FP32 master upd. │
   │ (backoff=0.5)  │  │ after growth_interval clean steps   │
   └────────────────┘  │ (torch default 2000): S ← S × 2     │
                       └─────────────────────────────────────┘

  ── AND, ONLY FOR FP8, A SECOND SCALING LOOP RUNS PER TENSOR ──

   for each FP8 GEMM operand (X, W, dY):
        amax_t = max(|tensor|)              ← computed on the GPU, fused
        push amax_t into a ring buffer of length amax_history_len
                                              (Transformer Engine default 1024)
        scale = FP8_MAX / (2^margin × max(history))     ← margin default 0
        store tensor as   round_to_fp8(tensor × scale)
        the GEMM's output is de-scaled by 1/(scale_X · scale_W) afterwards

   This is "DELAYED SCALING": the scale used THIS step comes from PREVIOUS
   steps' amax, so the cast never has to wait on a reduction. The cost is a
   one-step lag — a sudden spike in activation magnitude overflows once
   before the history catches up. "Current scaling" (TE's
   Float8CurrentScaling) uses this step's own amax instead: no lag, but the
   cast now depends on a full-tensor reduction completing first.
```

Two details in that diagram carry real operational weight.

**Why FP32 master weights exist at all.** It is not superstition. Consider a weight `w = 0.1` and an update `lr·g = 1e-7`. In BF16, the spacing of representable numbers near 0.1 is `0.1 × 2^-7 ≈ 7.8e-4`. The update is four orders of magnitude smaller than the gap to the next representable value, so `w + lr·g` rounds back to `w` and the parameter never moves. Keeping a 4-byte master copy means the update accumulates correctly; the low-precision copy is regenerated from it each step. This is why "BF16 training" still costs ~6 bytes/param for weights alone (4 master + 2 working), before optimizer state.

**Why BF16 mostly killed loss scaling.** BF16 has the same 8-bit exponent as FP32, hence the same ~78-decade reach. Gradients that would underflow FP16 are perfectly representable in BF16 — just coarsely. That single property is why BF16 became the default training format on Ampere and later despite being *less* precise than FP16: **stability beats precision when the failure mode is a silent flush-to-zero.** The rule of thumb: *BF16 for range and stability, FP16 for precision inside a range you already control (i.e. inference).*

### 6. What a tensor core actually is

A CUDA core does one fused multiply-add per clock: `d = a*b + c`, scalar. A tensor core does an entire small **matrix** multiply-accumulate per instruction:

```
  D  =  A × B  +  C
```

where A, B, C, D are tiles, and — this is the load-bearing part — **C and D are wider than A and B.** The inputs come in at FP16/BF16/FP8/FP4; the products are accumulated in FP32 (or FP16, on some paths). That asymmetry is the entire reason low precision is usable: you get the bandwidth and area savings of a narrow input format while the running sum, which is where catastrophic cancellation would otherwise destroy you, is kept wide.

```
  ONE WARP-LEVEL MMA — PTX `mma.sync.aligned.m16n8k16.f32.f16.f16.f32`
  (Ampere+ FP16 shape; all 32 threads of the warp cooperate on one instruction)

            B fragment                     each thread of the warp holds a few
            k=16 ┌──────┐ n=8              elements of A, B and C in ITS OWN
                 │      │                  registers; `ldmatrix` is the
                 │  B   │                  instruction that shuffles a shared-
                 │      │                  memory tile into that layout.
                 └──────┘
   A fragment    ┌──────┐                  Operand precision:  FP16 / BF16
   ┌────────────┐│      │  = D (+C)        Accumulate precision: FP32
 m=│     A      ││  D   │  m=16 × n=8      Tile:  16×16 × 16×8 → 16×8
 16└────────────┘└──────┘                  = 16·8·16 = 2048 MACs = 4096 FLOP
      k=16                                   per instruction, per warp

  THE ACCUMULATE PATH — why the wide accumulator is not optional
  ──────────────────────────────────────────────────────────────
   a0(fp8) ─┐
   b0(fp8) ─┴─▶ [ exact product, ~8-bit significand ] ─┐
   a1(fp8) ─┐                                          ├─▶ [ ADDER TREE ]
   b1(fp8) ─┴─▶ [ exact product                    ] ─┘        │
      ... k terms ...                                          ▼
                                          ┌────────────────────────────────┐
                                          │ ACCUMULATOR (FP32 register)    │
                                          │ ε = 1.19e-7 — the running sum  │
                                          │ keeps ~24 bits even though the │
                                          │ inputs carried 4               │
                                          └────────────────────────────────┘

   Why it matters numerically: summing k=4096 FP8 products IN FP8 would
   lose every term smaller than 1/8 of the running total (ε=0.125). The sum
   would saturate almost immediately. Accumulating in FP32 means the error
   grows like sqrt(k)·ε_fp32, not like k·ε_fp8.
```

Three generations of *how the instruction is issued* matter, because they explain why peak rates keep doubling without the clock changing:

- **Volta/Turing/Ampere — `mma.sync`, warp-synchronous.** The whole warp issues the MMA; operands live in the warp's registers. Shapes like `m8n8k4` (Volta), `m16n8k8` and `m16n8k16` (Ampere FP16), `m16n8k32` (INT8), `m16n8k64` (INT4).
- **Hopper — `wgmma.mma_async`, warpgroup-asynchronous.** A *warpgroup* (4 warps, 128 threads) issues one much larger MMA, the A operand can come straight from shared memory rather than registers, and the instruction is asynchronous — you `commit` and `fence` rather than blocking. Paired with the Tensor Memory Accelerator (TMA) doing the global→shared copies, this is what keeps the much wider Hopper tensor core fed. Shapes are `m64nNk16` families.
- **Blackwell — `tcgen05.mma`, single-thread issue.** Operands move out of registers entirely into a dedicated **Tensor Memory**, loads into it (`tcgen05.ld/.st/.cp`) are explicitly asynchronous, and a *single thread* initiates the MMA. Two CTAs in a thread-block cluster can form a CTA pair and share input operands, halving the shared-memory bandwidth needed per unit of math.

You will not write any of these. You need them for one reason: **when someone says "we're using tensor cores," the checkable claim is that an HMMA/QMMA-class instruction is issuing**, and the DCGM field `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` (field 1004, "ratio of cycles the tensor (HMMA) pipe is active") is literally counting those issue cycles. That is the bridge from this section to your dashboard.

### 7. The per-architecture tensor-core generation table

Dense figures throughout, for one **named** SKU per generation — because "Hopper does 4 petaflops" is meaningless without saying which Hopper part and whether sparsity is included.

| Gen | Arch (year) | Named SKU | New input types | Dense peak (that SKU) | Sparse peak | Memory / BW |
|---|---|---|---|---|---|---|
| **1st** | Volta (2017) | V100 SXM2 32GB | FP16 in / FP32 acc | **125 TFLOPS FP16** | — (no sparsity HW) | 32 GB HBM2, 900 GB/s |
| **2nd** | Turing (2018) | T4 (70 W, PCIe) | +INT8, INT4, INT1 | **65 TFLOPS FP16**, 130 TOPS INT8, 260 TOPS INT4 | — | 16 GB GDDR6, 320 GB/s |
| **3rd** | Ampere (2020) | A100 SXM 80GB | +TF32, +BF16, +FP64-TC, +2:4 sparsity | **312 TFLOPS BF16/FP16**, 156 TF TF32, 624 TOPS INT8, 19.5 TF FP64-TC | 624 / 312 / 1248 | 80 GB HBM2e, 2,039 GB/s |
| **4th** | Hopper (2022) | H100 SXM5 80GB | +FP8 (E4M3/E5M2), Transformer Engine, `wgmma` | **989.5 TFLOPS BF16**, **1,979 TFLOPS FP8**, 494.7 TF TF32, 1,979 TOPS INT8, 67 TF FP64-TC | 1,979 / 3,958 / 989.4 / 3,958 | 80 GB HBM3, 3,350 GB/s |
| **4th** | Hopper (2023) | H200 SXM 141GB | *(identical GH100 die)* | **989.5 TFLOPS BF16**, **1,979 TFLOPS FP8** | 1,979 / 3,958 | 141 GB HBM3e, 4,800 GB/s |
| **5th** | Blackwell (2024) | B200 SXM | +FP4 (NVFP4/MXFP4), +FP6, +MXFP8; 2nd-gen TE; `tcgen05` | **~2,250 TFLOPS BF16**, **~4,500 TFLOPS FP8**, **~9,000 TFLOPS FP4** | ~4,500 / ~9,000 / ~18,000 | 180 GB HBM3e @ ~7.7 TB/s (bare B200); board-level HGX B200 quoted at 186 GB @ 8 TB/s — **confirm the exact SKU variant** |

**Sanity-check any peak figure with this identity**, which is how the numbers are constructed in the first place:

```
  peak FLOPS = (SMs) × (FLOP per SM per clock, for that format) × (boost clock)

  V100 SXM2 : 80 SM × 1024 FLOP/SM/clk × 1.530 GHz = 125.3 TFLOPS  (FP16)  ✓
  A100 SXM  : 108 SM × 2048 FLOP/SM/clk × 1.410 GHz = 311.9 TFLOPS (BF16)  ✓
  A100 SXM  : 108 SM × 1024 FLOP/SM/clk × 1.410 GHz = 155.9 TFLOPS (TF32)  ✓
  H100 SXM5 : 132 SM × 4096 FLOP/SM/clk × 1.830 GHz = 989.6 TFLOPS (BF16)  ✓
  H100 SXM5 : 132 SM × 8192 FLOP/SM/clk × 1.830 GHz = 1979 TFLOPS  (FP8)   ✓
```

Read the pattern: **the per-SM tensor throughput doubled at each generation (1024 → 2048 → 4096 FLOP/SM/clock at 16-bit), and doubles again for each halving of the input format on hardware that supports it.** That is where "FP8 is 2× FP16" comes from — it is not a software effect, it is 8192 vs 4096 FLOP/SM/clock in the same silicon at the same clock. It also tells you exactly why an A100 gets *no* FP8 speedup: the A100's tensor core has no 8192-FLOP/clock FP8 path at all, so "FP8 on A100" means casting in software and running the math at BF16 rate or worse.

**2:4 structured sparsity, and why the billboard number is double.** Ampere added hardware that skips zeros in a *specific* pattern: in every contiguous group of 4 weights, at most 2 may be non-zero. The compressed weight matrix stores the 2 values plus a 2-bit index per group; the tensor core's operand-selection hardware uses the index to pull the matching activations, so it does half the MACs for the same output. That is a genuine 2×, but only for a model that has been pruned into the 2:4 pattern *and* retrained to recover accuracy. NVIDIA's marketing figures — 3,958 TFLOPS FP8 on H100, 624 TFLOPS BF16 on A100 — are the sparse numbers. **Unless you pruned the model yourself, your denominator is the dense figure.** Getting this wrong halves your reported MFU, in the flattering direction.

### 8. The accumulator is a hardware property, and it bites

Two real, checkable cases where the accumulate path — not the input format — determined the throughput and the numerics:

**Case 1: FP32 accumulate at half rate on consumer Ada.** NVIDIA's Ada whitepaper rates the RTX 4090 at 660 TFLOPS FP8 tensor (1,321 with sparsity). Engineers benchmarking `cublasLt` FP8 GEMMs on a 4090 reported reaching only ~330–340 TFLOPS and filed it on the NVIDIA developer forums; the resolution is that on GeForce Ada silicon, the **FP32-accumulate** path runs at half the FP16-accumulate rate. The same product-segmentation pattern applied to GA102 (RTX 3090) for FP16-with-FP32-accumulate. This is not a bug and not a driver problem; it is a deliberate difference between the consumer die and the datacenter die. **Operational consequence: a benchmark number measured on a workstation GPU does not transfer to the datacenter part of the same generation, and vice versa.** Always state which die you measured on.

**Case 2: Hopper's FP8 accumulator is narrower than FP32.** DeepSeek's V3 technical report documents that on H800 (Hopper), FP8 GEMM accumulation inside the tensor core retains only about **14 bits** of mantissa: partial products are right-shifted to align to the maximum exponent, the top 14 bits are added, and the rest are truncated. For a k of a few thousand, that error is large enough to matter for training stability. Their fix is instructive because it is pure platform engineering: every **128** elements along the k dimension, promote the partial sum out of the tensor core into an FP32 register in the CUDA cores and accumulate there, then continue. They pair it with **fine-grained scaling** — 1×128 tiles for activations, 128×128 blocks for weights — instead of one scale per tensor. The lesson for you: *"we run FP8" is under-specified.* The scaling granularity and the accumulation strategy are part of the configuration, they differ between frameworks, and they are where an FP8 run's accuracy actually comes from.

### 9. FP8 on Hopper — the Transformer Engine — and FP4 on Blackwell

FP8's problem is stated by its own table row: E4M3 covers about 5.4 decades. A transformer's tensors do not naturally live inside 5.4 decades. So FP8 is never used raw; it is always **format + per-tensor scale factor**, and something has to choose that scale factor every step. That something is NVIDIA's **Transformer Engine** (TE) library.

TE's default recipe is `DelayedScaling`, with these actual defaults:

| TE `DelayedScaling` parameter | Default | What it does |
|---|---|---|
| `fp8_format` | `HYBRID` | E4M3 for forward-pass tensors (weights, activations); **E5M2 for gradients** in the backward pass |
| `amax_history_len` | **1024** | Ring buffer of per-tensor `max(abs(x))` values from prior steps |
| `amax_compute_algo` | `max` | Take the largest amax in the window (alternative: `most_recent`) |
| `margin` | 0 | Extra safety headroom: `scale = FP8_MAX / (2^margin × amax)` |

The E4M3-forward / E5M2-backward split is the format table doing its job: forward tensors are well-conditioned and want the extra mantissa bit (E4M3, ε=0.125); gradients span a much wider range and want the extra exponent bit (E5M2, ~9.6 decades — exactly FP16's exponent reach). This is the same range-vs-precision decision from §4, made once per tensor role.

Two newer recipes exist and you should be able to name them, because the choice shows up in framework configs:

- **`Float8CurrentScaling`** — compute the scale from *this* step's amax. No one-step lag, so no "first spike overflows"; the cost is that the cast now serializes behind a full-tensor reduction.
- **`MXFP8BlockScaling`** — Blackwell-only. Instead of one scale per tensor, one shared scale per **32-element block**, in the OCP microscaling `E8M0` format (8 exponent bits, no mantissa — a pure power of two).

**FP4 is where scaling stops being optional and becomes the format.** E2M1 has sixteen codes and a 12:1 dynamic range; nothing survives that without help. Two competing block-scaled containers exist:

| | **NVFP4** (NVIDIA, Blackwell) | **MXFP4** (OCP microscaling standard) |
|---|---|---|
| Element format | E2M1 (4 bits) | E2M1 (4 bits) |
| Block size | **16 elements** | 32 elements |
| Scale format | **FP8 E4M3** (a real float — 256 distinct scale values) | E8M0 (power-of-two only) |
| Second-level scale | Optional FP32 per-tensor scale | None |
| Effective bits/element | 4 + 8/16 = **4.5** | 4 + 8/32 = **4.25** |
| Portability | NVIDIA Blackwell | Cross-vendor (also AMD MI355X) |

NVFP4's finer blocks and floating-point scale generally deliver better accuracy at equal element width; MXFP4 costs half the scale overhead and runs on more than one vendor's silicon. That is the whole trade, and it is the sort of thing you state in a report rather than "we used FP4."

### 10. Memory footprint: the arithmetic you'll do constantly

Weights alone: `params × bytes/param`.

| Model | FP32 (4 B) | FP16/BF16 (2 B) | FP8 (1 B) | FP4 (0.5 B) |
|---|---|---|---|---|
| 7B | 28 GB | 14 GB | 7 GB | 3.5 GB |
| 70B | 280 GB | 140 GB | **70 GB** | 35 GB |
| 405B | 1,620 GB | 810 GB | 405 GB | 203 GB |

A 70B model at FP16 is 140 GB — it does **not** fit on one 80 GB H100 and barely fits an H200 (141 GB) with almost no room for KV cache. At FP8 it is 70 GB and fits a single H100 with headroom for cache and activations. That single fact — one GPU vs. two — is the most common precision-driven cost decision you will make.

Real serving footprint is three terms, and skipping any of them produces a "does it fit" answer that fails on the box:

```
  HBM budget = weights + KV cache + (activations & workspace) + CUDA/NCCL overhead

  KV cache per token (per lesson 3's formula):
      2 (K and V) × layers × kv_heads × head_dim × bytes_per_elem

  Llama-3-70B: 80 layers, 8 KV heads (GQA), head_dim 128
      FP16:  2 × 80 × 8 × 128 × 2 B = 327,680 B ≈ 0.313 MiB/token
      FP8 :  2 × 80 × 8 × 128 × 1 B = 163,840 B ≈ 0.156 MiB/token

  On an 80 GB H100 with FP8 weights (70 GB) and ~4 GB of runtime overhead:
      free for KV ≈ 6 GB → 6144 MiB / 0.156 MiB = ~39,000 tokens of cache
                          ≈ 19 concurrent requests at 2k context.

  On a 141 GB H200, same FP8 weights:
      free for KV ≈ 67 GB → 68,608 MiB / 0.156 = ~440,000 tokens
                          ≈ 214 concurrent requests at 2k context.  ← 11× the batch
```

That 11× jump in achievable batch, from capacity alone, is the number lesson 6's H100-vs-H200 argument and lesson 7's cost model both rest on. Note that KV-cache precision is an **independent** lever from weight precision: many serving stacks let you run BF16 weights with an FP8 KV cache, or the reverse. GQA/MQA (lesson 3) shrinks the same number by cutting `kv_heads`; the two levers multiply.

### 11. The roofline connection — precision moves both axes

Recall lesson 2: a kernel is compute-bound or memory-bound depending on whether its **arithmetic intensity** (FLOP per byte of HBM traffic) is above or below the hardware's **ridge point**, `peak FLOPS ÷ peak bandwidth`. Precision moves the ridge point, and it moves it *the wrong way*:

```
  H100 SXM5 ridge point, by precision (dense peaks, 3.35 TB/s HBM3):

    BF16 : 989.5e12 FLOP/s ÷ 3.35e12 B/s  =  295 FLOP/byte
    FP8  : 1979e12  FLOP/s ÷ 3.35e12 B/s  =  591 FLOP/byte

  H200 SXM, same compute die, 4.8 TB/s HBM3e:

    BF16 : 989.5e12 ÷ 4.8e12 = 206 FLOP/byte
    FP8  : 1979e12  ÷ 4.8e12 = 412 FLOP/byte
```

**Halving the precision doubles the ridge point**, because compute doubled and bandwidth did not. A kernel needs *twice* the arithmetic intensity to stay compute-bound at FP8 as at BF16. So the effect of a precision change splits cleanly:

- **Compute-bound dense matmul** (large GEMMs, prefill, training steps): the FLOPS ceiling doubles and the kernel — if its intensity is high enough to stay right of the new, higher ridge — runs close to 2× faster. This is where tensor cores earn their keep.
- **Memory-bound ops** (elementwise, layernorm, softmax, single-token decode, small batch): the ceiling is bytes/sec, not FLOPS. You still win, because you moved half as many bytes — but the win is the byte reduction, not the tensor-core multiple, and it caps out at 2× only if the kernel was purely weight-streaming. Tensor-active for a pure elementwise kernel is **0.00**, no matter what dtype you passed in.

Worked, for decode on an H100 serving 70B:

```
  Decode is one weight sweep per token-batch. Time per step ≈ bytes ÷ bandwidth.

    FP16 weights: 140 GB ÷ 3.35 TB/s = 41.8 ms/step  → 23.9 steps/s
    FP8  weights:  70 GB ÷ 3.35 TB/s = 20.9 ms/step  → 47.8 steps/s

  ...but 140 GB does not fit one H100. Both numbers come from bandwidth alone
  and ignore KV traffic, which grows with batch and context. Treat them as the
  ceiling, not the forecast — lesson 4's number, re-derived at two precisions.
```

### 12. The realized multiple vs. the theoretical multiple — a verified gap

The tensor-core-level multiple (~2× FP8 vs FP16) is a hardware fact. What you'll actually observe end to end is smaller, and you should be able to quote a real number for *why*, not just assert "less than 2x." Databricks/MosaicML's published FP8 training results report **>50% MFU** (among the highest published at the time) and **~1.5× realized training throughput** from FP8 plus other stack optimizations, on real large-scale training runs.

That gap is not a discrepancy to explain away — it is Amdahl's law with the roofline supplying the fractions. Write it out:

```
  Let f = fraction of step time in FP8-accelerated tensor-core matmul.
  Everything else (dataloading, optimizer, all-reduce, layernorm, softmax,
  attention softmax/exp, host sync) is unchanged by the precision switch.

      speedup = 1 / [ (1 - f) + f/2 ]

      f = 0.90  →  1 / (0.10 + 0.45) = 1.82×
      f = 0.75  →  1 / (0.25 + 0.375) = 1.60×
      f = 0.67  →  1 / (0.33 + 0.335) = 1.50×   ← the Databricks-shaped answer
      f = 0.50  →  1 / (0.50 + 0.25) = 1.33×

  So a reported 1.5× end-to-end implies roughly two-thirds of step time was in
  the accelerated matmuls. That is a plausible, healthy training pipeline —
  not a sign anything is broken.
```

Distinguish this from a *second*, differently-shaped calculation later in this lesson: a footprint change (2 GPUs → 1 GPU) **compounds** with a throughput change, producing a bigger overall $/token improvement than the throughput multiple alone. Databricks' 1.5× is throughput-only on a fixed hardware footprint; this lesson's own worked example below compounds a footprint halving with a throughput gain. Both numbers are correct — they're measuring different things, and being explicit about which one you're citing is the discipline that separates a defensible cost argument from a sound-bite.

### 13. BF16 vs FP16 — same size, different failure mode

Both are 2 bytes. **FP16** = 5 exponent / 10 mantissa: ε = 9.77e-4, but the world ends at 65504 and the floor is 6e-8 — about 12 decades total. **BF16** = 8 exponent / 7 mantissa: ε = 7.81e-3 (8× coarser), with FP32's full ~78 decades.

The practical distinction is what happens when you exceed the format:

- FP16 overflow → `Inf` → the loss becomes `NaN` → the run is dead and you know within minutes.
- FP16 underflow → silent `0` → the run *appears* healthy and converges worse. **This is the dangerous one**, and it is why dynamic loss scaling exists.
- BF16 → neither, in practice. You just carry more rounding noise, which stochastic gradient descent tolerates remarkably well.

Rule of thumb: *BF16 for range/stability (training), FP16 for precision within a range you already control (inference).* One more consequence worth knowing: **FP32 → BF16 conversion is a truncation of the low 16 bits**, since the exponent field and bias are identical. FP32 → FP16 requires a real exponent re-bias and range check. That is why BF16 conversion is nearly free in hardware and why BF16 showed up on TPUs first.

### 14. The quantization trade — PTQ vs QAT, and the accuracy number to quote

Going lower precision trades accuracy for throughput and footprint. Two operational paths exist, and platform engineers should be able to name both even though the ML team makes the call:

- **PTQ (post-training quantization).** Train in higher precision, then quantize the finished checkpoint, using a calibration dataset to set scale factors (for "static" quantization) or computing them at runtime per tensor ("dynamic"). Cheap, fast, the default assumption most engineers reach for.
- **QAT (quantization-aware training) / native low-precision training.** Train *in* the target low precision from the start, so the model learns to be robust to it. Character.AI's engineering blog is explicit that they train natively in int8 rather than quantizing post-hoc, stating this "eliminates the risk of training/serving mismatch" — a mismatch PTQ can introduce if the calibration distribution doesn't match production traffic.

**The number to have ready when someone asks "what does FP8 cost us in quality?"** Neural Magic (now Red Hat AI) published fully FP8-quantized Llama 3.1 checkpoints with measured recovery against the unquantized baseline: the 405B FP8 model scores **86.55 average on the OpenLLM v1 benchmark suite vs. 86.63 for the BF16 original — 99.91% recovery** — while cutting weight memory roughly in half (their write-up cites ~400 GB vs ~500 GB for a variant that quantizes every linear module rather than skipping some). For well-behaved dense transformers, **FP8 weight+activation quantization is a sub-0.5% accuracy event, and you should say so with the citation rather than hedging.** INT4 and FP4 are a different conversation — that is where calibration quality and scaling granularity start to show up in benchmark scores.

Two levers matter operationally regardless of path: **which tensors** you quantize (weights are easy; activations are harder because of outlier channels; the KV cache is an independent lever again) and **granularity** of scaling (per-tensor is cheap; per-channel, per-block, or DeepSeek's 1×128 tiles preserve more accuracy — and it is exactly why FP4 needed 16- or 32-element blocks to be viable at all). Your report should state the throughput gain *and* the measured or vendor-cited accuracy delta, so the trade is explicit rather than assumed.

### 15. The hardware gate: FP8 is not available everywhere

Worth stating as its own fact rather than leaving it implicit in the generation table: **A100 (Ampere) has no FP8 tensor-core pipeline.** FP8 tensor cores arrived with Hopper's 4th-generation tensor core; the CUDA-level types (`__nv_fp8_e4m3`, `__nv_fp8_e5m2`, header `cuda_fp8.h`) landed in **CUDA 11.8**, and `cublasLtMatmul` gained FP8 GEMM support around the same window, extended in CUDA 12.0 (FP8 with non-zero beta) and in CUDA 12.2 (FP8 on Ada). On an A100, "FP8" is only achievable via software casting — there is no hardware path and no throughput win to claim. If a report or a colleague says "we're running this model in FP8 on our A100 fleet," that claim needs a follow-up question, because the accelerated path they're describing does not exist on that silicon. Precision is not purely a software configuration flag; it is gated by which generation you're renting — exactly the thread lesson 6 picks up.

## Perspectives

**Developer/ML view.** The choice between PTQ and QAT is made at training time, by people who own the training recipe — but it fixes what a platform engineer can later promise. Character.AI's decision to train natively in int8 (rather than the more common PTQ-after-training path) is a real, deliberate engineering choice with a stated rationale (eliminating train/serve mismatch), not an implementation detail. Knowing this distinction — and which one your org uses — changes what accuracy guarantee you can respond with when someone asks "if we go lower precision, what breaks?"

**Operator/observability view.** `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` (field ID 1004, the same field lesson 1 introduced for the utilization-lie discussion) is your direct evidence that a precision change actually landed on the tensor cores rather than staying a config flag nobody verified. It literally counts cycles in which the HMMA pipe issued. A model reported as "running in FP8" at 5% tensor-active is memory-bound or launch-bound regardless of what the config says — the precision change bought you the memory win but not the compute win, and your dashboard should be able to say so.

**Hardware view.** Peak throughput is `SMs × FLOP/SM/clock × clock`, and the only term precision touches is the middle one — 4096 FLOP/SM/clock at BF16 on Hopper, 8192 at FP8. That is why the multiple is exactly 2 in the silicon and never exactly 2 in your logs: everything else in the pipeline is unchanged. Support is gated generationally — FP8 needs Hopper's 4th-gen tensor cores (A100 has none), FP4 needs Blackwell's 5th-gen with block scaling. And the accumulator is its own hardware property: FP32-accumulate at half rate on GeForce Ada, ~14-bit accumulation for FP8 GEMM on Hopper.

**Economics view.** Databricks' verified ~1.5× *realized* end-to-end throughput against the ~2× *theoretical* tensor-core multiple is the single cleanest illustration in this module of "you get the memory win always, a partial throughput win depending on workload mix." The Amdahl arithmetic in §12 lets you invert it: a reported multiple tells you what fraction of the pipeline was actually on the tensor cores. Quantifying that gap — rather than rounding it up to "2×" in a report — is the difference between a number that survives review and one that doesn't.

## Real-world use cases

- **[Databricks/MosaicML, "Turbocharged Training: Optimizing the Databricks Mosaic AI Stack With FP8"](https://www.databricks.com/blog/turbocharged-training-optimizing-databricks-mosaic-ai-stack-fp8).** Real large-scale training results: **>50% MFU** (claimed the highest among published numbers at the time) and **~1.5× realized throughput** from FP8 plus stack optimizations — the concrete "theoretical 2× vs. realized 1.5×" number this lesson's Amdahl derivation in §12 reproduces from first principles.
- **DeepSeek-V3 technical report (arXiv 2412.19437), FP8 mixed-precision section.** The most detailed public account of what running FP8 training *actually* requires: fine-grained 1×128 activation-tile and 128×128 weight-block scaling instead of per-tensor; promotion of partial sums from the tensor core to FP32 in the CUDA cores every 128 k-elements, because they measured Hopper's FP8 GEMM accumulator as retaining only ~14 mantissa bits; and an explicit list of operators kept at higher precision (embeddings, output head, MoE gating, normalization, attention). What it shows: "we use FP8" is a configuration space, not a switch — and a hardware limitation you cannot read off a spec sheet can dictate the whole kernel design.
- **Neural Magic / Red Hat AI FP8 Llama 3.1 model cards.** Published, reproducible accuracy-recovery numbers for FP8 PTQ: **99.91% average OpenLLM recovery on the fully-quantized 405B** (86.55 vs 86.63), with weight memory roughly halved. What it shows: the accuracy side of the FP8 trade has a real, citable number for dense transformers — you do not have to hedge, and you should not have to re-measure it yourself to justify a rollout.
- **Character.AI, ["Optimizing AI Inference at Character.AI"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/) and ["Part Deux"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-part-deux-2/).** Native int8 QAT training (not PTQ) combined with MQA and KV-sharing, at real production LLM-chat scale (billions of messages/day) — the clearest real example of a team choosing train-time quantization over post-hoc quantization on purpose, and stating why.
- **[Google Cloud, "What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0) (Medium, official Google Cloud publication).** A benchmark on 96×B200 GPUs serving ~1M tok/s reports **tensor cores active only 1.5% of the time** at decode batch=1 even though GPU-Util reads near 100% — a real, current (B200-generation) demonstration of exactly the "precision doesn't help if the kernel never reaches the tensor cores" point §11 makes, reused here through the tensor-core-observability lens rather than the raw-utilization lens lesson 1 used it for.
- **NVIDIA developer-forum reports of RTX 4090 FP8 `cublasLt` throughput landing at ~330–340 TFLOPS against a 660 TFLOPS rated figure.** What it shows: the accumulate width is a product-segmentation lever, not a constant — a measured number from consumer silicon does not transfer to the datacenter part of the same architecture, and "we benchmarked it on a 4090" is a caveat, not a data point.

## Worked example

**Question: a 70B-parameter chat model, target one GPU per replica — what precision, and what does it buy?**

**Step 1 — does it fit?**

```
  weights(FP16) = 70e9 params × 2 B/param = 140 GB   → NOT on an 80 GB H100.
                                                       Needs 2× H100 with TP=2,
                                                       or one 141 GB H200 with
                                                       ~0 GB left for KV cache.
  weights(FP8)  = 70e9 params × 1 B/param =  70 GB   → fits one 80 GB H100 with
                                                       ~6 GB for KV after runtime
                                                       overhead (~4 GB).
```

**Step 2 — what does the byte reduction buy on the memory side?** Decode re-reads the entire weight set per step (lesson 4). Bytes per decode step, and the bandwidth-limited step time on an H100 at 3.35 TB/s:

```
  FP16: 140 GB / 3.35 TB/s = 41.8 ms → 23.9 steps/s   (theoretical, ignores KV)
  FP8 :  70 GB / 3.35 TB/s = 20.9 ms → 47.8 steps/s

  Bandwidth saved per step: 70 GB. Over an hour of continuous decode at the
  FP8 rate:  70 GB × 47.8 steps/s × 3600 s = 12.0 EB/hr of HBM traffic avoided
  relative to running the same step count at FP16. The HBM is the scarce
  resource in decode; this is the resource you are actually buying back.
```

**Step 3 — what does the FLOPS ceiling buy on the compute side?** Prefill is compute-bound. For a 1,000-token prompt, forward-pass FLOPs ≈ `2 × params × tokens`:

```
  FLOPs = 2 × 70e9 × 1000 = 1.4e14 = 140 TFLOP per prompt

  At H100 BF16 dense peak (989.5 TFLOPS), at a realistic 60% of peak:
      140 TFLOP / (0.60 × 989.5 TFLOP/s) = 236 ms
  At H100 FP8 dense peak (1,979 TFLOPS), at the same 60%:
      140 TFLOP / (0.60 × 1979 TFLOP/s)  = 118 ms

  Prefill halves. Decode halves. Everything between them does not.
```

**Step 4 — the end-to-end multiple, honestly.** Apply §12's Amdahl form to a *serving* mix. Suppose measurement shows 70% of wall-clock in FP8-accelerated GEMMs and weight streaming, 30% in attention softmax, sampling, scheduler and Python overhead:

```
  speedup = 1 / [0.30 + 0.70/2] = 1 / 0.65 = 1.54×

  So: quote ~2× for the raw GEMM, ~1.5–1.6× end to end, and say which is which.
```

**Step 5 — the cost arithmetic, units carried.** Illustrative on-demand rate of **$3.00/H100-hr** (a 2026 placeholder — lesson 7 replaces it with tier-labelled live pricing; the *method* is what matters here):

```
  FP16 path — 2× H100 per replica (TP=2), measured 1,000 tok/s aggregate:
     cost rate       = 2 × $3.00/hr = $6.00 per replica-hour
     tokens per hour = 1,000 tok/s × 3,600 s/hr = 3.6e6 tok/hr
     $/1M tokens     = $6.00 / 3.6 = $1.67 per million tokens

  FP8 path — 1× H100 per replica, 1,000 × 1.6 = 1,600 tok/s aggregate:
     cost rate       = 1 × $3.00/hr = $3.00 per replica-hour
     tokens per hour = 1,600 × 3,600 = 5.76e6 tok/hr
     $/1M tokens     = $3.00 / 5.76 = $0.52 per million tokens

  improvement = 1.67 / 0.52 = 3.2×
              = (2× from footprint) × (1.6× from throughput)   ← it COMPOUNDS
```

**Step 6 — price the accuracy.** Using the published FP8 PTQ recovery figure as the prior: ~0.1% average benchmark degradation for a dense transformer of this class. Expressed as a trade: **you are paying roughly 0.1 percentage points of benchmark average to cut $/1M-tokens by 3.2× and free 70 GB of HBM.** Write it that way in the report — as a priced trade with both sides quantified — and then verify it on your own eval set before rollout, because your traffic is not the benchmark's traffic.

**Why this "3.2×" doesn't contradict Databricks' verified "1.5×."** This example's figure is a *compounded* result: halving the hardware footprint (2 GPUs → 1, a 2× on its own) **times** an effective ~1.6× throughput gain on the surviving GPU. Databricks' ~1.5× is throughput-only, measured on an *already-fixed* hardware footprint. The two numbers answer different questions — "what does FP8 buy me if I also right-size the fleet" vs. "what does FP8 buy me on the fleet I already have." Conflating them is the fastest way to produce a defensible-looking number that falls apart under a follow-up question.

The decision you write up: *FP8 halves the hardware and cuts $/M-token ~3× on this footprint-plus-throughput view (or ~1.5–1.6× on a throughput-only view at fixed hardware); the accuracy delta is expected below 0.5% based on published FP8 PTQ recovery, and we measure it against the FP16 baseline on our own eval set before rollout.*

## Practice — feeds the deliverable

On a rented single GPU (H100 or H200 ideal, since FP8 needs Hopper+):

1. **Matmul throughput sweep.** Run a large square GEMM (8192³ is a good size — 2×8192³ = 1.1 TFLOP per call, big enough to amortize launch overhead) at BF16 and at FP8, and record achieved TFLOPS for each. A minimal PyTorch harness:

   ```python
   import torch, time

   def bench(fn, warmup=10, iters=50):
       for _ in range(warmup): fn()
       torch.cuda.synchronize()
       t0 = time.perf_counter()
       for _ in range(iters): fn()
       torch.cuda.synchronize()
       return (time.perf_counter() - t0) / iters      # seconds per call

   N = 8192
   flop = 2 * N**3                                    # 2 FLOP per MAC

   a16 = torch.randn(N, N, device="cuda", dtype=torch.bfloat16)
   b16 = torch.randn(N, N, device="cuda", dtype=torch.bfloat16)
   t = bench(lambda: torch.matmul(a16, b16))
   print(f"BF16: {flop / t / 1e12:8.1f} TFLOP/s   ({t*1e3:.2f} ms)")

   # FP8 requires per-tensor scales and a column-major B operand.
   a8 = a16.to(torch.float8_e4m3fn)
   b8 = b16.t().contiguous().t().to(torch.float8_e4m3fn)
   s  = torch.tensor(1.0, device="cuda")
   t = bench(lambda: torch._scaled_mm(a8, b8, scale_a=s, scale_b=s,
                                      out_dtype=torch.bfloat16))
   print(f"FP8 : {flop / t / 1e12:8.1f} TFLOP/s   ({t*1e3:.2f} ms)")
   ```

   Record the **throughput multiple** and each side's fraction of the *dense* spec peak (989.5 TFLOPS BF16, 1,979 TFLOPS FP8 on H100 SXM5). Expect roughly 1.7–2× on the raw matmul; published `cublasLt`-backed FP8 GEMM figures on H100 land in the ~1,200–1,300 TFLOPS region at this size, i.e. ~60–65% of dense peak. If yours is far below that, your matrices are too small or you are timing the launch, not the kernel.

2. **Memory halving.** Load a weight tensor (or a small model) at BF16 and again at FP8 and capture the footprint both ways:

   ```
   python -c "import torch; ..."  # torch.cuda.memory_allocated() before/after
   nvidia-smi --query-gpu=memory.used --format=csv,noheader -l 1
   ```

   Confirm the FP8 footprint is ~½ and note the gap between `torch.cuda.memory_allocated()` (tensor bytes) and `nvidia-smi memory.used` (allocator reservation + CUDA context, typically 0.5–1 GB more).

3. **Tensor-core activity.** While each run executes, sample DCGM in a second terminal:

   ```
   dcgmi dmon -e 1001,1002,1004,1005 -d 100
   #            │    │    │    │
   #            │    │    │    └── DRAM_ACTIVE       (memory-bound signal)
   #            │    │    └─────── PIPE_TENSOR_ACTIVE (field 1004 — the proof)
   #            │    └──────────── SM_ACTIVE
   #            └───────────────── GR_ENGINE_ACTIVE
   ```

   Confirm the dense-matmul run drives `TENSO` high, then run a pure elementwise kernel (`torch.add` on two large tensors in a loop) and confirm tensor-active sits at ~0.00 while `DRAM_ACTIVE` goes high — proving the "tensor cores idle on memory-bound ops" claim with your own telemetry, not this lesson's assertion.

4. **Accuracy note.** If you ran a small inference both ways, record any output/quality difference (or note "not measured — flagged as open"). State in your write-up whether your model/framework used PTQ-style post-hoc casting or trained natively in low precision, and what scaling granularity the stack used (per-tensor? per-block?) — this determines how much you should trust the accuracy delta you observe.

**Acceptance:** the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md) contains a measured **FP16-vs-FP8 throughput multiple and memory delta** from your rented GPU, each side expressed as a percentage of the *dense* spec peak, a DCGM tensor-active reading showing dense-matmul vs. elementwise, and a one-line accuracy caveat stating whether the multiple you're reporting is raw-kernel or end-to-end.

## Common pitfalls

1. **Assuming FP8 works identically on A100 and H100.** It doesn't — A100 has no FP8 tensor-core pipeline; the 8192-FLOP/SM/clock FP8 path first exists on Hopper. "FP8 on A100" means software casting with no throughput win. Symptom: a config change lands cleanly, memory drops, and throughput does not move at all.
2. **Quoting the sparse TFLOPS figure as your MFU or roofline denominator.** H100's 3,958 FP8 and 1,979 BF16 headline numbers are 2:4-sparsity figures; dense peaks are exactly half. Using the sparse figure silently doubles your apparent efficiency — in the flattering direction, which is why nobody catches it. Unless you pruned the model into the 2:4 pattern, your denominator is dense.
3. **Conflating the tensor-core throughput multiple (~2×, hardware-level) with realized end-to-end throughput (often ~1.5–1.6×, workload-level).** They differ by exactly the fraction of the pipeline that isn't matmul — §12's Amdahl form lets you compute one from the other instead of hand-waving.
4. **Treating TF32 as a memory saving.** It is 4 bytes on the wire and 4 bytes in HBM; only the tensor core's internal mantissa is 10 bits. TF32 buys FLOPS, never bytes. A capacity plan that assumes otherwise is wrong by 2×.
5. **Assuming PTQ is the only path to low precision.** Character.AI's native int8 training shows train-time quantization (QAT) is a real, chosen-on-purpose alternative with different — and in their framing, better — accuracy trade-offs than post-hoc calibration.
6. **Trusting a "running in FP8" claim without checking tensor-active telemetry.** A config flag is not proof of an accelerated kernel path; `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` near zero on a supposedly-FP8 workload means the precision change bought memory savings but not the compute win.
7. **Benchmarking on consumer silicon and quoting it for the datacenter part.** GeForce Ada runs FP32-accumulate at half rate; a 4090 FP8 GEMM measured at ~330 TFLOPS against a 660 TFLOPS rating is the documented symptom. State the die, not just the architecture.
8. **Ignoring FP16's underflow, because overflow is the one you've heard of.** Overflow gives you `NaN` and a dead run in minutes. Underflow gives you silent zeros and a worse model in three days. Loss scaling exists for the second one; if you disable it "because the run is stable," you have removed the guard against the failure that does not announce itself.

## Self-check

- **Derive FP8-E4M3's maximum representable value from its field widths, and explain why it is 448 and not 240.** *Answer:* E4M3 has bias `2^(4-1) - 1 = 7`. Under the IEEE convention the top exponent field (`1111`) is reserved for Inf/NaN, so the largest usable unbiased exponent would be `14 - 7 = 7`, giving `1.875 × 2^7 = 240`. NVIDIA/Arm/Intel's FP8 spec instead spends that exponent field on ordinary numbers, reserving only the single pattern `S.1111.111` as NaN and dropping infinities entirely. The largest value is then `1.75 × 2^(15-7) = 1.75 × 256 = 448`. The extra binade is bought by giving up encodings the format couldn't afford. E5M2 keeps the IEEE convention, which is why it stops at `1.75 × 2^15 = 57344` rather than 65504.
- **How much GPU memory does a 70B model's weights need at FP16 vs. FP8, and what else has to fit?** *Answer:* FP16 = `70e9 × 2 B = 140 GB`; FP8 = `70e9 × 1 B = 70 GB`. FP16 needs two 80 GB H100s (or one H200 with essentially zero headroom); FP8 fits one 80 GB H100. But "fits" means `weights + KV cache + activations/workspace + ~4 GB runtime overhead`. For Llama-3-70B (80 layers, 8 GQA KV heads, head_dim 128) the KV cache is `2×80×8×128×bytes` = 0.313 MiB/token at FP16, 0.156 MiB/token at FP8 — so an 80 GB H100 with FP8 weights has room for roughly 39k tokens of cache (~19 concurrent 2k-context requests), while a 141 GB H200 has room for ~440k (~214 requests). That 11× batch difference, not the compute, is the H200 argument.
- **Why doesn't FP8 speed up a memory-bound elementwise kernel proportionally to its ~2× tensor-core throughput?** *Answer:* An elementwise kernel has no matmul, so no HMMA instruction issues and the tensor cores contribute nothing — `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` reads ~0.00. Its ceiling is bytes/sec off HBM, not FLOPS. Halving the element width halves the bytes, so you get up to a 2× *bandwidth-side* win, but that is a different mechanism from the tensor-core multiple and it does not stack with it. Structurally: halving precision doubles the ridge point (`peak FLOPS / peak BW`: 295 → 591 FLOP/byte on H100), so a kernel below the ridge stays below it and only benefits from the smaller footprint.
- **Databricks reports ~1.5× realized training throughput from FP8, while raw tensor-core FP8 throughput is ~2× FP16. Why the gap, and does it contradict this lesson's ~3× worked-example figure?** *Answer:* Amdahl's law: `speedup = 1 / [(1-f) + f/2]` where `f` is the fraction of step time in FP8-accelerated matmul. `f = 0.67` gives exactly 1.5×. The other third — dataloading, optimizer, all-reduce, layernorm, softmax — is untouched by the precision change. It does not contradict the ~3× figure, because that one additionally compounds a *footprint* halving (2 GPUs → 1) with the throughput gain; Databricks' 1.5× is throughput-only on a fixed footprint. Same physics, different question.
- **What does moving from FP16 to FP8 do to your $/token, your max batch size, and your accuracy?** *Answer:* Memory per parameter halves, which frees HBM for a bigger KV cache (more concurrent requests per GPU) and often lets you drop from two GPUs to one. Combined with the ~1.5–1.6× realized throughput gain, end-to-end $/token typically falls 1.5–3× depending on whether a hardware-footprint reduction is compounded in. On accuracy, the citable prior for dense transformers is Neural Magic/Red Hat's FP8 Llama 3.1 405B result: 86.55 vs 86.63 OpenLLM average, **99.91% recovery** — i.e. well under half a percent. Always re-measure on your own eval set, since the published number is against a benchmark suite, not your traffic.
- **Why does BF16 rarely need loss scaling when FP16 almost always does?** *Answer:* Loss scaling exists to lift gradients above FP16's underflow floor. FP16 has a 5-bit exponent: minimum normal `2^-14 ≈ 6.1e-5`, subnormals down to `2^-24 ≈ 6e-8`, about 12 decades total, and late-training gradients routinely fall below that and are flushed to zero. BF16 has an 8-bit exponent, identical to FP32 — ~78 decades — so those gradients are representable (coarsely, at ε = 7.8e-3, but representable). BF16 trades precision for range, and the failure that matters is silent underflow, not rounding noise.
- **Why do FP32 master weights exist in a BF16 training run, and what does that do to your memory budget?** *Answer:* An optimizer update `lr·g` can be ~1e-8 relative to the weight. Near `w = 0.1`, BF16's spacing is `0.1 × 2^-7 ≈ 7.8e-4` — four orders of magnitude larger than the update, so `w + lr·g` rounds back to `w` and the parameter never moves. Keeping a 4-byte FP32 master copy lets updates accumulate; the BF16 working copy is regenerated from it each step. Budget: ~6 bytes/param for weights alone (4 master + 2 working) before optimizer state — which is why "BF16 training" costs far more than `params × 2`.
- **A colleague benchmarks FP8 GEMM on an RTX 4090 and reports 335 TFLOPS against a 660 TFLOPS rating, and concludes the driver is broken. What do you tell them?** *Answer:* It is not broken — on GeForce Ada silicon the FP32-accumulate tensor path runs at half the FP16-accumulate rate, a product-segmentation difference between the consumer and datacenter dies (the same pattern applied to GA102 for FP16-with-FP32-accumulate). Either switch to FP16 accumulation if the numerics allow it, or accept the half rate. The broader rule: **the accumulator width is part of the hardware spec, not a software detail**, and a number measured on consumer silicon does not transfer to the datacenter part of the same architecture.

## Connections & what's next

This lesson is the second half of the roofline-manipulation pair that started in lesson 2: lesson 2 taught you where a workload sits on the roofline, this one taught you how to move the roofline itself — and §11 quantified the move (H100's ridge point goes from 295 to 591 FLOP/byte when you halve the format). It also closes the loop lesson 3 opened: GQA/MQA shrink the KV-cache constant factor by cutting `kv_heads`; precision (including KV-cache quantization specifically, as Character.AI's stack shows) is an independent, stackable lever on the same product, not a competing one. The `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` telemetry here is the same field 1004 lesson 1 used to debunk the utilization lie — this lesson is where you learn to read it as a precision-verification signal specifically, because it counts HMMA issue cycles and nothing else.

Next: **[03.6 · Generational SKUs & the software stack](06-generational-and-software-stack.md)** takes the hardware-gating fact from this lesson (FP8 needs Hopper+, FP4 needs Blackwell+, and the per-SM FLOP/clock table that explains why) and turns it into a purchasing argument — which generation actually removes your workload's bottleneck, and what breaks in the driver/CUDA/NCCL stack if you get the pairing wrong.

## References & further reading

**Primary sources**
- Micikevicius et al. (NVIDIA, Arm, Intel), ["FP8 Formats for Deep Learning"](https://arxiv.org/abs/2209.05433), arXiv 2209.05433 (2022) — the definitive E4M3/E5M2 specification: the bias values, the max/min tables, and the explicit rationale for E4M3 abandoning infinities to reclaim a binade (240 → 448). Read §3 for the encoding tables reproduced in this lesson.
- [NVIDIA H100 Tensor Core GPU Architecture whitepaper](https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper) — authoritative dense/sparse FP8, BF16, TF32 and FP64-tensor TFLOPS figures for H100 SXM5, the 132-SM count, and the 4th-generation tensor core and Transformer Engine description.
- [NVIDIA A100 Tensor Core GPU Architecture whitepaper](https://www.nvidia.com/en-us/data-center/a100/) — the TF32 bit layout (8 exponent / 10 mantissa in a 4-byte container), the 3rd-generation tensor core, and the 2:4 structured-sparsity mechanism with its compressed-index encoding.
- [NVIDIA CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/) and [CUDA Math API — FP8 types](https://docs.nvidia.com/cuda/cuda-math-api/) — `__nv_fp8_e4m3` / `__nv_fp8_e5m2` and `cuda_fp8.h`, introduced in **CUDA 11.8**; the conversion and saturation semantics.
- [NVIDIA PTX ISA — warp-level matrix instructions](https://docs.nvidia.com/cuda/parallel-thread-execution/) — the `mma.sync.aligned.mMnNkK` shape families, Hopper's asynchronous `wgmma.mma_async`, and Blackwell's `tcgen05.mma` with its Tensor Memory operands. Read for the exact tile shapes and accumulator types per architecture.
- [NVIDIA Transformer Engine documentation](https://docs.nvidia.com/deeplearning/transformer-engine/user-guide/) — the `DelayedScaling` recipe with its real defaults (`amax_history_len=1024`, `amax_compute_algo='max'`, `margin=0`, `HYBRID` format), plus `Float8CurrentScaling` and `MXFP8BlockScaling`.
- [NVIDIA DCGM documentation — feature overview](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html) — canonical `DCGM_FI_PROF_*` field definitions, including field 1004 `PIPE_TENSOR_ACTIVE` ("ratio of cycles the tensor (HMMA) pipe is active") and 1005 `DRAM_ACTIVE`.
- DeepSeek-AI, "DeepSeek-V3 Technical Report" ([arXiv 2412.19437](https://arxiv.org/abs/2412.19437)) — the FP8 mixed-precision framework: 1×128 / 128×128 fine-grained scaling, the ~14-bit Hopper FP8 accumulator measurement, and the every-128-elements promotion to FP32 in the CUDA cores.

**Real-world engineering blogs**
- Databricks/MosaicML, ["Turbocharged Training: Optimizing the Databricks Mosaic AI Stack With FP8"](https://www.databricks.com/blog/turbocharged-training-optimizing-databricks-mosaic-ai-stack-fp8) — the theoretical-vs-realized FP8 gap at real training scale (>50% MFU, ~1.5× throughput).
- Neural Magic / Red Hat AI, FP8 Llama 3.1 model cards (e.g. [`neuralmagic/Meta-Llama-3.1-405B-FP8`](https://huggingface.co/neuralmagic/Meta-Llama-3.1-405B-FP8)) — published OpenLLM recovery figures for FP8 PTQ (86.55 vs 86.63 average, 99.91% recovery), the accuracy half of this lesson's trade.
- Character.AI, ["Optimizing AI Inference at Character.AI"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/) and ["Part Deux"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-part-deux-2/) — native int8 QAT training chosen over PTQ, at production chat scale.
- Google Cloud, ["What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0) — a real B200 benchmark showing tensor cores active only 1.5% of the time despite near-100% GPU-Util; the clearest production evidence that a precision win and a utilization win are not the same claim.
- SemiAnalysis, ["NVIDIA Tensor Core Evolution: From Volta To Blackwell"](https://newsletter.semianalysis.com/p/nvidia-tensor-core-evolution-from-volta-to-blackwell) — the per-generation issue-model story (warp-synchronous `mma.sync` → warpgroup-async `wgmma` → single-thread `tcgen05.mma` with Tensor Memory and CTA pairs) that explains how throughput kept doubling without a clock increase.

**Deeper dives**
- Horace He, ["Making Deep Learning Go Brrrr From First Principles"](https://horace.io/brrr_intro.html) — the compute/memory/overhead-bound trichotomy that explains, in general form, why elementwise kernels never benefit from a precision-driven tensor-core speedup.
- [Open Compute Project, Microscaling (MX) Formats specification](https://www.opencompute.org/documents/ocp-microscaling-formats-mx-v1-0-spec-final-pdf) — the block-scaled MXFP8/MXFP6/MXFP4 definitions (32-element blocks, E8M0 shared scale) that Blackwell implements and against which NVFP4's 16-element/E4M3-scale variant is the NVIDIA-specific alternative.
- NVIDIA, ["Train With Mixed Precision"](https://docs.nvidia.com/deeplearning/performance/mixed-precision-training/index.html) — the gradient-histogram evidence behind loss scaling, and the full list of which operations must stay in FP32.
