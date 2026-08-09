---
lesson: "03.7"
title: "Capstone — cost per unit of useful work"
module: "03"
concept: "Capstone — cost per useful work"
status: not-started
est_time: "6h"
artifacts: []
---
# 03.7 · Capstone — cost per unit of useful work
> **Concept.** Combine achieved-vs-spec TFLOPS, the FP16→FP8 delta, the "utilisation lies" demo, and live $/hr into a defensible cost-per-useful-work analysis and a SKU recommendation.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Why this matters

Every previous lesson produced a fact — a roofline, an FP8 speedup, a `GPU-Util`
number that lied. On its own each fact is trivia. This lesson welds them into the
one artifact a GPU-heavy platform team actually buys with: **cost per unit of
useful work**, expressed as `$/1M tokens` for serving and `$ per useful PFLOP`
for compute-bound work.

This is the artifact that maps 1:1 onto the interview and onto your FinOps
differentiator. "Is the H200 worth 40% more per hour?" is not a spec question and
not a $/hr question — it is a $/token question, and almost nobody in the room can
work it end to end. The number you produce is the rare object a finance team and
an eng team both accept: finance sees dollars per delivered token; eng sees the
throughput and the bandwidth ceiling behind it. Owning that translation is the
whole point of the module.

## The platform-engineer's lens

You are not benchmarking for a paper. You do not need the last 3% of MLPerf
tuning. You need a **defensible** figure and a **buy recommendation** you can
stand behind in a review with numbers a skeptic can re-derive.

Defensible means three things:

1. **Every assumption is stated** — model, precision, sequence lengths, batch/
   concurrency, prefill vs decode split, $/hr source and date, contract term.
2. **Utilisation is never confused with useful work.** `GPU-Util` at 100% proves
   a kernel was resident, not that FLOPs or tokens were produced. You report the
   metric that survives scrutiny (achieved TFLOPS, tok/s, bandwidth utilisation),
   and you show where the naive metric misled.
3. **The comparison is on cost-per-useful-work, not on sticker price.** A SKU that
   is cheaper per hour can be *dearer per token*. That inversion is the money
   insight of the whole report.

Pricing is the one input with a short shelf life. **Refresh $/hr at study time** —
the SemiAnalysis index below moves monthly (H100 1-yr rentals ran ~$1.70/hr in
Oct 2025 and ~$2.35/hr by Mar 2026). Treat any number in this file as a worked
placeholder, not a current quote.

## Core notes — the method

Four steps. Each reuses a run you already have from L1–L6.

### 1 — Achieved vs spec TFLOPS → a crude MFU

Spec TFLOPS is a billboard. What matters is what your kernel *achieved*. From your
L1 GEMM run:

```
MFU_compute = achieved_TFLOPS / spec_TFLOPS(dense, this precision)
```

Example, H100 SXM, FP8 dense GEMM, large square matmul:

```
achieved ≈ 1,400 TFLOPS   (measured)
spec     ≈ 1,979 TFLOPS   (FP8 dense; the 3,958 billboard number is WITH sparsity)
MFU      ≈ 1400 / 1979 ≈ 0.71   → 71%
```

Two traps to name in the report:

- **Sparsity inflation.** NVIDIA's headline 3,958 TFLOPS FP8 and 1,979 TFLOPS
  BF16 for Hopper are 2:4-sparsity numbers. Dense peaks are half: ~1,979 FP8,
  ~989.5 BF16. Divide by the *dense* spec or your MFU is silently doubled.
- **MFU is a compute-bound metric.** It is meaningful for training and prefill.
  For **decode** it is single-digit and *correctly so* — decode is memory-bound,
  so low compute-MFU is not waste, it is physics. Report bandwidth utilisation
  there instead (achieved GB/s ÷ HBM spec GB/s).

### 2 — $ per useful PFLOP (compute-bound work)

Convert achieved throughput and live price into cost per delivered FLOP:

```
$/useful-PFLOP = ($/hr) / (achieved_PFLOP/s × 3600)
```

The denominator is PFLOP *delivered* in one hour. Example, H100 FP8 at
1.4 PFLOP/s achieved, $2.50/hr:

```
delivered = 1.4 × 3600 = 5,040 PFLOP/hr
$/useful-PFLOP = 2.50 / 5040 ≈ $4.96e-4 per PFLOP
```

Note what this does: raising achieved TFLOPS (better kernels, FP8, larger tiles)
lowers cost *at the same $/hr*. Efficiency and price are the two levers, and this
formula makes them commensurate.

### 3 — $/1M tokens (serving — the number that ships)

For inference, tokens are the unit of useful work, not FLOPs:

```
$/1M-tokens = ($/hr) / (tokens_per_sec × 3600 / 1e6)
```

`tokens_per_sec` is **aggregate decode throughput** across all concurrent
requests on the GPU (or on the TP group), measured — not per-request. Everything
that moves tok/s moves the cost:

- **FP8 vs FP16.** FP8 halves weight bytes moved per token, so on a
  bandwidth-bound decode it roughly doubles throughput, *and* it halves the KV
  cache and weight footprint so a larger batch fits — a second, compounding win.
  Net effect is often 2–2.5× cheaper per token, not merely 2×.
- **Batch / concurrency.** Decode reads the full weight set once per step and
  amortises it across every request in the batch. Bigger batch = more tokens per
  weight-read = lower $/token — until you hit the KV-cache capacity wall or the
  latency SLO. This is the batch curve from L4/L6: throughput rises, then
  saturates when HBM fills with KV.
- **Capacity as a throughput lever.** More HBM is not just "fits bigger models."
  It buys KV headroom → larger batch → more tok/s → lower $/token, even when raw
  compute is identical. This is exactly why the H200 can win.

### 4 — Compare SKUs on cost-per-useful-work, not $/hr

The whole method exists to defeat the sticker-price instinct. Lay two SKUs side
by side on `$/1M tokens` (serving) or `$/useful-PFLOP` (compute). The cheaper-per-
hour box loses whenever the pricier box delivers proportionally more useful work:

```
cheaper-per-token  ⇔  ($/hr ratio)  <  (tok/s ratio)
```

If the H200 costs 1.4× the H100 per hour but delivers 1.7× the tokens, it is
1.7/1.4 ≈ 1.2× cheaper per token. Per-hour intuition gets this backwards. Your
report's headline is this ratio, with the throughput measurement behind it.

## Worked example — H100 vs H200, 70B serving

**Workload (stated assumptions — copy these into the report):**

- Model: Llama-3-70B, decode-heavy serving (chat), ~1k-token prompts, ~512-token
  generations, prefill/decode roughly amortised into aggregate tok/s.
- Precisions compared: FP16 weights (140 GB) and FP8 weights (~70 GB).
- Placement: FP8 fits on **one** GPU (70 GB weights + KV in the remainder). FP16
  needs **two** GPUs (140 GB > 80/141 GB once KV is added), so FP16 runs TP=2.
- Throughput: **measured** aggregate decode tok/s at the largest batch that holds
  the p95 latency SLO. Numbers below are illustrative placeholders from a L6-style
  run — substitute your own.
- Price: neocloud on-demand, **refresh at study time**. Placeholders:
  H100 SXM = **$2.50/hr**, H200 SXM = **$3.50/hr** (H200 ≈ 1.4× H100/hr).

Hardware facts (identical compute — this is the crux):

| | H100 SXM | H200 SXM |
|---|---|---|
| FP8 dense TFLOPS | 1,979 | 1,979 (identical) |
| BF16 dense TFLOPS | 989.5 | 989.5 (identical) |
| HBM capacity | 80 GB HBM3 | 141 GB HBM3e |
| HBM bandwidth | 3.35 TB/s | 4.8 TB/s (≈1.43×) |

H200 buys **zero** extra compute. It buys 1.43× bandwidth and 1.76× capacity —
and decode is memory-bound, so both go straight to throughput.

**FP8, single GPU each (the clean comparison):**

```
Measured aggregate decode:
  H100 FP8: 2,000 tok/s   (KV room ≈ 10 GB after 70 GB weights → modest batch)
  H200 FP8: 3,400 tok/s   (KV room ≈ 71 GB → much larger batch × 1.43 bandwidth)
                          → 1.7× tokens for 1.4× the hourly price

$/1M-tokens = ($/hr) / (tok/s × 3600 / 1e6)
  H100: 2.50 / (2000 × 3600 / 1e6) = 2.50 / 7.2   = $0.347 / 1M tok
  H200: 3.50 / (3400 × 3600 / 1e6) = 3.50 / 12.24 = $0.286 / 1M tok
```

**H200 is ~18% cheaper per token despite costing 40% more per hour.** The lever is
not clock speed — the compute specs are byte-identical — it is 71 GB of KV
headroom letting batch grow, multiplied by 1.43× bandwidth feeding it.

**FP16 vs FP8 (the precision delta):**

```
70B FP16 needs TP=2 (140 GB weights):
  2× H100: $5.00/hr, measured 1,600 tok/s (TP comms + tiny KV room)
  $/1M = 5.00 / (1600 × 3600 / 1e6) = 5.00 / 5.76 = $0.868 / 1M tok

FP8 single H100: $0.347 / 1M tok
→ FP8 is ~2.5× cheaper per token than FP16 here — half the bytes per token
  AND single-GPU placement (no TP tax, more of the box spent on your model).
```

**Recommendation (defended):** For this 70B decode-serving workload, **buy H200
and serve FP8.** The 40%-higher hourly rate is a red herring; on
cost-per-useful-work the H200 wins by ~18% because its 141 GB unlocks the batch
size the Hopper compute engine can already sustain but the 80 GB H100 starves.
Reach for **2× H100 FP16 only** when accuracy validation rejects FP8 for this
model — and price that decision explicitly at ~2.5× the per-token cost so the
quality/cost trade is a visible line item, not an accident. Caveats to state:
numbers assume decode-bound traffic and an SLO that permits large batch; a
latency-tight, low-concurrency service narrows the H200 gap because it cannot use
the KV headroom, and at that point the cheaper-per-hour H100 can retake the lead.

## Practice — THE MODULE DELIVERABLE

Assemble the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)
from your L1–L6 runs. It is one document with these sections, each backed by a
committed benchmark artifact:

1. **Achieved-vs-spec TFLOPS + roofline (L1–L3).** Your measured GEMM vs the dense
   spec, the crude MFU, and the roofline point placing the workload on the
   compute or memory side.
2. **FP16→FP8 delta (L4–L5).** Measured speedup and the byte-movement / capacity
   explanation — not just the ratio, the *why*.
3. **The utilisation-lies demo (L1).** A run where `nvidia-smi GPU-Util` reads high
   while useful work is low, and the metric that corrected it.
4. **Decode ceiling + batch curve (L4/L6).** Measured decode tok/s vs the
   bandwidth-derived ceiling, plus the throughput-vs-batch curve and where KV
   capacity saturates it.
5. **Cost-per-useful-work + one-page SKU recommendation (this lesson).** The
   `$/1M-tokens` comparison of at least two SKUs from measured throughput + dated
   live $/hr, ending in a defended buy call.

Acceptance = the module [checkpoint](../checkpoint.md). The report is done when
every checkpoint box ticks and you can defend each number from memory.

## Self-check

**Q1. For the stated 70B FP8 decode workload, is H100 or H200 cheaper per token? Show the working.**
**Answer:** H200. `$/1M = ($/hr)/(tok/s×3600/1e6)`. H100: `2.50/(2000×3600/1e6)=2.50/7.2=$0.347`. H200: `3.50/(3400×3600/1e6)=3.50/12.24=$0.286`. The H200 is 1.4× the hourly price but delivers 1.7× the tokens (1.43× bandwidth × larger batch from 71 GB KV headroom), so it is 1.7/1.4≈1.2× cheaper per token — ~18% lower. The per-hour instinct gets it backwards. (Refresh both $/hr before quoting.)

**Q2. Where did `nvidia-smi GPU-Util` mislead in your report, and which metric corrected it?**
**Answer:** In the decode/util-lies section: `GPU-Util` read ~100% during decode because a kernel was resident every sample interval, implying the GPU was "fully used." But compute-MFU was single-digit — decode is memory-bound, so the SMs were mostly stalled waiting on HBM. The corrective metrics were **achieved bandwidth utilisation** (achieved GB/s ÷ 3.35 TB/s, near the roofline) for the ceiling and **aggregate tok/s** for useful work. `GPU-Util` measures residency, not FLOPs or tokens delivered.

**Q3. Your $/1M-token estimate at FP8 vs FP16, with assumptions?**
**Answer:** ~$0.35/1M (FP8, single H100, $2.50/hr, 2,000 tok/s) vs ~$0.87/1M (FP16, 2× H100 TP=2, $5.00/hr, 1,600 tok/s) → FP8 ≈ 2.5× cheaper. Assumptions: Llama-3-70B decode-heavy, ~1k-in/512-out, largest batch under the p95 SLO, neocloud on-demand pricing dated at study time, FP8 accuracy validated as acceptable. The FP16 penalty is two-fold — 2× bytes per token and forced TP=2 (comms tax + shrunken KV room). If FP8 fails accuracy validation, the ~2.5× premium is the stated cost of that quality requirement.

## Resources

- **SemiAnalysis GPU rental index** — https://gpu-index.semianalysis.com/ · the
  canonical $/hr source across SKUs and contract terms. **Time-sensitive — skim
  and refresh every number before you quote it** (moved ~40% in the 6 months to
  Mar 2026).
- **NVIDIA H100 datasheet / product page** — https://resources.nvidia.com/en-us-tensor-core
  and https://www.nvidia.com/en-us/data-center/h200/ · spec TFLOPS (use the
  **dense** rows, not the sparsity billboards) and HBM capacity/bandwidth.
- **Your own L1–L6 benchmark data** under [`../practice/`](../practice/) —
  achieved TFLOPS, FP8 delta, decode tok/s, batch curve. These are the measured
  inputs; the method here is just the arithmetic on top.
- **$/1M-token methodology** — e.g. the Artificial Analysis inference cost pages
  (https://artificialanalysis.ai/) for a cross-check on public per-token prices
  and the tok/s-to-$ derivation.
