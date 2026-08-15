---
lesson: "03.7"
title: "Capstone — cost per unit of useful work"
module: "03"
concept: "Cost-per-useful-work synthesis"
status: not-started
est_time: "10h"
prev: "06-generational-and-software-stack.md"
next: null
artifacts: []
sources: 15
---

# 03.7 · Capstone — cost per unit of useful work

> **Concept.** Combine achieved-vs-spec TFLOPS, the FP16→FP8 delta, the "utilisation lies" demo, and **tier-aware live $/hr** into a defensible cost-per-useful-work analysis and a SKU recommendation — and learn why the tier you price against can flip the recommendation.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Where this fits

Six lessons built six separate facts. L1 taught you that `GPU-Util` measures kernel *residency*, not useful work — the util-lie. L2 gave you the roofline: whether a workload is compute-bound or memory-bound, and how far it sits from its ceiling. L3–L4 gave you the memory-hierarchy arithmetic — what fits in HBM, and why decode's tokens/sec ceiling is a bandwidth number, not a compute number. L5 gave you the precision lever — FP8 halves bytes moved and roughly doubles throughput, with a compounding capacity side-effect. L6 gave you the generational SKU table (A100/H100/H200/B200) and the operational discipline that keeps a fleet from silently underperforming its spec.

Each of those facts is trivia in isolation. This capstone welds them into the one artifact a GPU-heavy platform team actually transacts on: **cost per unit of useful work** — `$/1M tokens` for serving, `$/useful-PFLOP` for compute-bound work — and a defended SKU recommendation. It also adds the one input none of the prior six lessons touched: **price**, and specifically the fact that price is not a single number per SKU. This lesson is where "I understand GPU hardware" becomes "I can tell you which box to buy, and defend the number in a room."

## Why this matters

Every one of this module's job signals — CoreWeave's "hardware validation across the fleet," NVIDIA's "arithmetic intensity vs peak compute using the roofline," the interview question "which SKU is cheaper per unit of useful work?" — collapses into this lesson. It is, verbatim, the last row of the module's own lesson table: **"the SKU recommendation, defended by numbers."**

The stake is concrete. A platform team renting GPU capacity is making a recurring six- or seven-figure decision, and "which GPU should we buy/rent" gets asked in nearly every infra review at a company running model training or inference at any scale. Most engineers answer it with a spec sheet or a sticker price — "H100 is $X/hr, H200 is $Y/hr, so..." — and stop there. That answer is wrong often enough to matter: a GPU that is 40% more expensive per hour can be cheaper per token, and a GPU that is cheaper per hour can be *more* expensive per token, depending on what the extra memory or bandwidth buys in throughput. The engineer who can only reason in $/hr loses that argument to the engineer who reasons in $/token — and in an interview, "why is the H200 sometimes the better inference buy despite costing more?" is precisely the checkpoint's own depth probe. This lesson is where you build the muscle to answer it with your own measured numbers instead of someone else's slide.

The cost of *not* knowing this shows up as real money: a team that buys on $/hr alone routinely overpays 10–30% for a given workload, and a report built on stale or tier-blind pricing gets a wrong SKU call through the door because nobody checked which reliability tier the number came from. Both failure modes are avoidable, and both are exactly what this lesson trains you out of.

## What's new here (calibration)

The module's calibration note still holds: you are not becoming a CUDA kernel author, and NVLink/PCIe topology and power/thermal throttling stay in 02b's territory, referenced not re-taught. For this capstone specifically:

- **Not new:** the four technical inputs (achieved-vs-spec TFLOPS, the FP16→FP8 delta, the util-lie, the memory/bandwidth table) — you built all four in L1–L6. This lesson does not re-derive them.
- **Genuinely new: the pricing model itself.** Every prior lesson treated `$/hr` as a single placeholder number. This lesson replaces that fiction with the real market structure — GPU-cloud pricing is **tiered by reliability and support**, not a single figure per SKU, and the tier you compare against changes the answer. That is new content, not a review.
- **Genuinely new: sensitivity analysis as a discipline.** Producing *one* cost-per-token number is the easy 80% of this lesson. Showing how that number moves when you change one input (here, pricing tier) — and stating which conclusions are robust to that change and which aren't — is the staff-level habit this capstone is built to install.
- **Genuinely new: turning a technical result into a document a non-engineer will act on.** The report is the deliverable; a number a finance stakeholder can't audit isn't done.

## Core concepts

### The shape of the method

Five steps, each consuming an output from a prior lesson:

```
L1: achieved TFLOPS/bandwidth  ─┐
L2: which roof you're under     ├──▶  MFU / bandwidth-util  ─┐
L5: FP8 vs FP16 delta          ─┘                             ├──▶  $/useful-PFLOP
L3–L4: fits + decode tok/s ceiling ──▶  achieved tok/s ───────┤     $/1M tokens
tier-aware $/hr (this lesson, NEW) ────────────────────────────┘
                                                                └──▶  SKU comparison,
                                                                      sensitivity-checked
```

### Step 1 — Achieved vs spec TFLOPS → a crude MFU (recap, L1–L2)

**MFU** (Model FLOPs Utilization) is the fraction of a GPU's theoretical peak compute your workload actually achieved:

```
MFU_compute = achieved_TFLOPS / spec_TFLOPS(dense, this precision)
```

Example, H100 SXM, FP8 dense GEMM, large square matmul:

```
achieved ≈ 1,400 TFLOPS   (measured)
spec     ≈ 1,979 TFLOPS   (FP8 dense — the 3,958 billboard number is WITH 2:4 sparsity)
MFU      ≈ 1400 / 1979 ≈ 0.71 → 71%
```

Two traps carried forward from L1, worth restating because they silently double or halve your final cost number if missed:

- **Sparsity inflation.** NVIDIA's headline 3,958 TFLOPS (FP8) and 1,979 TFLOPS (BF16) for Hopper are 2:4-sparsity numbers; dense peaks are half. Divide by the *dense* spec or your MFU — and every dollar figure downstream of it — is silently doubled.
- **MFU is a compute-bound metric.** For **decode**, single-digit compute-MFU is *correct*, not a red flag — decode is memory-bound (L3–L4), so report **bandwidth utilization** (achieved GB/s ÷ HBM spec GB/s) there instead. This is the same distinction the Google Cloud B200 benchmark below makes explicit in production.

### Step 2 — The precision delta (recap, L5)

FP8 roughly halves weight bytes moved per token and roughly doubles decode throughput on a bandwidth-bound kernel — and it compounds with a second effect: halved weights also free HBM for a bigger KV cache, so a larger batch fits, which raises throughput again. The two effects together commonly land FP8 at 2–2.5× cheaper per token, not merely 2×. Note the gap between this compounded 2–2.5× and Databricks/MosaicML's own reported **~1.5× realized end-to-end training throughput** from FP8 (against a theoretical ~2× tensor-core multiple) — the two numbers are not in tension. Databricks' 1.5× is *throughput-only, on a fixed hardware footprint* (data loading, optimizer steps, and communication don't get the FP8 speedup, diluting the average). This lesson's 2–2.5× additionally folds in a *hardware-footprint* change (fewer GPUs needed at FP8, because weights are smaller) — a compute-bound training multiple and a cost-bound serving multiple measuring genuinely different things. State which one you're quoting.

### Step 3 — Live $/hr, tier-aware (the genuinely new part of this lesson)

> **Dated snapshot — August 2026, verify before you quote it.** GPU-cloud pricing moves fast enough that a number six months old is close to useless in a report. Everything below is a snapshot from SemiAnalysis's ClusterMAX rating system and newsletter, current as of this research pass; re-pull it at build time from [`clustermax.semianalysis.com`](https://clustermax.semianalysis.com/) and [`gpu-index.semianalysis.com`](https://gpu-index.semianalysis.com/) before you ship a number.

Every prior version of this lesson (and most engineers reasoning about GPU cost) treated `$/hr` as one number per SKU. It isn't. SemiAnalysis's **ClusterMAX** rating system scores GPU-cloud providers on performance, networking, storage, security, and support, and buckets them into tiers — **Bronze, Silver, Gold, Platinum** — because raw $/hr comparison across neoclouds is actively misleading without knowing what you're getting for it. As of this research pass:

| Tier | 1-year-reserved H100 $/hr | Example providers | What the tier buys |
|---|---|---|---|
| **Bronze** | **$1.45 – $2.00** | SF Compute ($1.45), Vast.ai ($1.49), Hyperstack ($1.90), RunPod Community ($1.99) | Lowest price; weaker guarantees on networking, support responsiveness, hardware validation cadence |
| **Silver** | **$2.10 – $2.99** | Lambda, AWS, GMI Cloud, Scaleway | A +30–50% premium over the Bronze floor for materially better reliability, networking, and support SLAs |
| Gold / Platinum | above Silver, not itemized in this pass | — | Higher still; enterprise-grade support and validated fleet operations |

The **cohort median across the H100 rental market sits around $3.15/GPU-hour** in this pass — down from **more than $7/hr in early 2024**. That two-year collapse is itself a data point worth internalizing: a report's dollar figures decay on the scale of months, not years, in this market.

The load-bearing point for this lesson: **"H100 costs $X/hr" is not a well-formed sentence in 2026.** There is a >2× spread between the Bronze floor and the top of Silver alone, before Gold/Platinum, entirely explained by reliability and support tier — not by the silicon, which is identical. Any $/hr figure you cite needs a tier, a provider, a contract term, and a date attached, or it isn't a real number, it's a rounding error waiting to flip your conclusion (see the Worked example below).

Two live sources to combine: SemiAnalysis's [GPU rental index](https://gpu-index.semianalysis.com/) for a fast cross-SKU, cross-term snapshot, and the [ClusterMAX](https://clustermax.semianalysis.com/) ratings for the tier context that the index alone doesn't carry. [Artificial Analysis](https://artificialanalysis.ai/) is a useful independent cross-check for **serving** $/token specifically — its published methodology weights input:output tokens **3:1** and uses first-party API pricing where available (median-across-providers otherwise); a workload whose real input:output ratio differs from 3:1 will see a different true cost than the public benchmark implies, so treat it as a sanity check, not a substitute for your own measured tok/s.

### Step 4 — Two cost formulas

**$/useful-PFLOP (compute-bound work):**

```
$/useful-PFLOP = ($/hr) / (achieved_PFLOP/s × 3600)
```

Example, H100 FP8 at 1.4 PFLOP/s achieved, Bronze-tier $1.90/hr:

```
delivered = 1.4 × 3600 = 5,040 PFLOP/hr
$/useful-PFLOP = 1.90 / 5040 ≈ $3.77e-4 per PFLOP
```

Two independent levers move this number: raising achieved TFLOPS (better kernels, FP8, larger tiles) and lowering $/hr (a cheaper tier, a longer contract term). The formula makes them commensurate — a report should show both, not conflate them.

**$/1M tokens (serving — the number that ships):**

```
$/1M-tokens = ($/hr) / (tokens_per_sec × 3600 / 1e6)
```

`tokens_per_sec` is **aggregate decode throughput** across all concurrent requests on the GPU (or TP group), measured — not per-request. Everything that moves tok/s moves this cost: FP8 vs FP16 (Step 2), batch/concurrency (decode amortizes one weight-read across the whole batch — bigger batch, more tokens per weight-read, lower cost, until the KV-cache capacity wall or the latency SLO stops you — this is L4/L6's batch curve), and raw HBM capacity as a throughput lever in its own right (more capacity buys KV headroom, not just "fits a bigger model," and KV headroom buys batch, which buys tok/s — this is exactly why H200 can win despite zero extra compute).

### Step 5 — Compare SKUs on cost-per-useful-work, not $/hr — and check the comparison's sensitivity

The whole method exists to defeat the sticker-price instinct:

```
cheaper-per-token  ⇔  ($/hr ratio)  <  (tok/s ratio)
```

If the H200 costs 1.4× the H100 per hour but delivers 1.7× the tokens, it is 1.7/1.4 ≈ 1.2× cheaper per token. Per-hour intuition gets this backwards — but the ratio itself now has a second failure mode this lesson adds: **the $/hr ratio you plug in depends on which tier you priced each SKU at.** If you price the cheap SKU at its Bronze floor and the pricier SKU at a Silver quote, you haven't measured a hardware comparison — you've measured a tier comparison wearing a hardware costume. The Worked example below makes this concrete: the *same* two GPUs, at the *same* measured throughputs, rank in opposite orders depending on which tiers you compare.

## Perspectives

**Finance/FinOps view.** Finance wants one number — $/1M tokens, or $/useful-PFLOP — that survives a quarterly review. The ClusterMAX tiering adds a real complication finance doesn't intuitively expect: "cheaper $/hr" isn't even a clean, single number without specifying the reliability tier, so a finance-facing report has to carry the tier as a first-class field next to the price, not a footnote. A report that says "H100: $2.50/hr" without a tier is not more precise than one that says "H100: $1.45–2.99/hr depending on tier" — it's less precise, dressed up as more precise.

**Engineering view.** Engineering supplies the inputs finance can't produce alone: achieved TFLOPS, achieved bandwidth utilization, measured tok/s at a stated batch size and SLO. None of that is negotiable or vendor-dependent — it's what you actually measured on the box. The engineering discipline is refusing to let a spec-sheet number substitute for a measured one anywhere in the chain.

**Vendor/market view.** SemiAnalysis built ClusterMAX *because* the market itself is not homogeneous — the same silicon, rented from different providers, is not the same product once you account for networking quality, support responsiveness, and hardware-validation cadence (the same fleet-observability discipline CoreWeave's HPC Verification framework runs hourly against idle nodes). Tiering is a market-structure fact, not a pricing gimmick: a Bronze box and a Platinum box running the identical H100 SXM chip are genuinely different purchases, and treating them as interchangeable is the single most common analytical shortcut that produces a wrong recommendation.

**Failure-mode view.** The lesson's core thesis — "per-hour intuition gets it backwards" — is the most common mistake a junior platform engineer makes in a SKU review, and tier-blindness is its most common *specific* form in 2026: comparing a cherry-picked cheap quote for SKU A against a realistic quote for SKU B (or vice versa) and mistaking the resulting gap for a hardware conclusion. The fix is procedural, not clever: always state the tier, provider, contract term, and date next to every $/hr figure, and re-run the comparison at matched tiers before trusting the ranking.

## Real-world use cases

- **[SemiAnalysis, "The GPU Cloud ClusterMAX Rating System — How to Rent GPUs"](https://newsletter.semianalysis.com/p/the-gpu-cloud-clustermax-rating-system-how-to-rent-gpus).** The industry-standard framework for exactly the "which SKU, from which vendor, at what tier" decision this capstone asks you to make — what it shows: raw $/hr comparison across GPU clouds is misleading without a reliability/support tier attached, and quantifies the Bronze/Silver spread this lesson uses.
- **[SemiAnalysis, "How Much Do GPU Clusters Really Cost?"](https://newsletter.semianalysis.com/p/how-much-do-gpu-clusters-really-cost).** What it shows: the market-level pricing trend (cohort median falling from >$7/hr in early 2024 to ~$3.15/hr in this pass) that justifies this lesson's "always date your dollar figures" rule with a real, large number.
- **[Databricks/MosaicML, "LLM Inference Performance Engineering: Best Practices"](https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices).** What it shows: a production team running multiple serving backends (vLLM, TensorRT-LLM, FasterTransformer) grounding the same prefill-compute-bound/decode-memory-bound framing this capstone's cost formulas depend on — practitioner voice behind Step 4's math.
- **[Google Cloud (Medium), "What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0).** What it shows: a real B200 production benchmark (96×B200, GKE Autopilot, ~1M tok/s aggregate) reporting 4.4% FLOPS utilization and 1.5% tensor-active time at decode batch=1 — while `GPU-Util` would read near 100%. This is the util-lie (L1) and the roofline (L2) both landing in one real, dated, dollar-adjacent number: you could combine its published throughput with a current B200 rental rate to build a real $/token estimate, which is the exact move this lesson trains.

## Worked example — H100 vs H200, 70B serving, and why the tier you compare matters

**Workload (stated assumptions — copy these into the report):**

- Model: Llama-3-70B, decode-heavy serving (chat), ~1k-token prompts, ~512-token generations.
- Precision: FP8 weights (~70 GB) on both SKUs, single GPU each (the clean comparison — no TP tax on either side).
- Throughput: measured aggregate decode tok/s at the largest batch holding the p95 latency SLO — **H100: 2,000 tok/s** (KV room ≈10 GB after 70 GB weights → modest batch); **H200: 3,400 tok/s** (KV room ≈71 GB → much larger batch × 1.43× bandwidth). These are illustrative placeholders from an L6-style run — substitute your own measured numbers.
- Hardware facts (identical compute — this is the crux): both SKUs run 1,979 FP8 dense TFLOPS and 989.5 BF16 dense TFLOPS — *identical*. H100 has 80 GB HBM3 at 3.35 TB/s; H200 has 141 GB HBM3e at 4.8 TB/s (≈1.43×). H200 buys **zero** extra compute — it buys 1.43× bandwidth and 1.76× capacity, and decode is memory-bound, so both go straight to throughput.

**Pricing (dated snapshot, this research pass — re-verify at study time):** H100 ClusterMAX Bronze **$1.45/hr**, H100 ClusterMAX Silver **$2.99/hr** (both verified tier bounds, above). SemiAnalysis's verified tier breakdown in this pass covers H100 specifically; **H200 tier pricing was not independently confirmed in this research pass**, so the H200 figures below apply the ~1.4× premium commonly observed for H200 over H100 in neocloud listings as an *estimate*, flagged explicitly — re-verify against `gpu-index.semianalysis.com` before citing: H200 Bronze-equivalent (est.) **$2.03/hr**, H200 Silver-equivalent (est.) **$4.19/hr**.

```
$/1M-tokens = ($/hr) / (tok/s × 3600 / 1e6)
```

| Comparison | H100 $/hr | H100 tok/s | H100 $/1M tok | H200 $/hr | H200 tok/s | H200 $/1M tok | Cheaper SKU | Margin |
|---|---|---|---|---|---|---|---|---|
| **Same tier — Bronze vs Bronze** | $1.45 | 2,000 | $0.201 | $2.03 (est.) | 3,400 | $0.166 | **H200** | ~17% cheaper |
| **Same tier — Silver vs Silver** | $2.99 | 2,000 | $0.415 | $4.19 (est.) | 3,400 | $0.342 | **H200** | ~18% cheaper |
| **Cross-tier — H100 Bronze vs H200 Silver** | $1.45 | 2,000 | $0.201 | $4.19 (est.) | 3,400 | $0.342 | **H100** | ~70% cheaper |
| **Cross-tier — H100 Silver vs H200 Bronze** | $2.99 | 2,000 | $0.415 | $2.03 (est.) | 3,400 | $0.166 | **H200** | ~2.5× cheaper |

**The pedagogical point, stated plainly:** when the tiers are matched (Bronze vs Bronze, Silver vs Silver), the H200 wins by a stable ~17–18% — that ranking is a real hardware conclusion, driven by the bandwidth/capacity math in Step 4, and it holds regardless of which tier you happened to price at, because the tier premium applies roughly proportionally to both SKUs. But the moment you compare **across** tiers — H100's cheapest listing against H200's pricier listing, or vice versa — the ranking **flips entirely**, and by a much larger margin than the hardware difference itself. A report that quotes "H100 $1.45/hr" (a Bronze headline number, easy to find) against "H200 $3.50–4/hr" (a Silver-or-above quote, because that's what showed up in the search) has not measured a hardware comparison. It has measured a tier comparison, and drawn a hardware conclusion from it — and would recommend the *wrong* SKU on this workload.

**Recommendation (defended):** For this 70B decode-serving workload, **buy H200 and serve FP8, holding the pricing tier constant across both SKUs in the comparison.** At matched tiers, the H200's 141 GB unlocks the batch size the Hopper-class compute engine can already sustain but the 80 GB H100 starves, at a stable ~17–18% cost-per-token advantage. State the tier, provider, and date next to every dollar figure in the final report; if the report only has budget for one SKU's quote at a discount tier, get the comparable-tier quote for the other SKU before drawing a conclusion — don't compare a negotiated rate against a rack-rate.

## Practice — THE MODULE DELIVERABLE

This lesson's practice **is** the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md) — assemble it from your L1–L6 runs, now closing with this lesson's tier-aware pricing method instead of a single placeholder $/hr. It is one document with these sections, each backed by a committed benchmark artifact:

1. **Achieved-vs-spec TFLOPS + roofline (L1–L2).** Your measured GEMM vs the dense spec, the crude MFU, and the roofline point placing the workload on the compute or memory side.
2. **FP16→FP8 delta (L5).** Measured speedup and the byte-movement/capacity explanation — not just the ratio, the *why* — and explicitly state whether you're quoting a throughput-only multiple or a compounded (throughput + footprint) multiple, per Step 2 above.
3. **The utilisation-lies demo (L1).** A run where `nvidia-smi GPU-Util` reads high while useful work is low, and the metric that corrected it.
4. **Decode ceiling + batch curve (L3–L4).** Measured decode tok/s vs the bandwidth-derived ceiling, plus the throughput-vs-batch curve and where KV capacity saturates it.
5. **Cost-per-useful-work + one-page SKU recommendation (this lesson).** The `$/1M-tokens` comparison of at least two SKUs from measured throughput + **dated, tier-labeled $/hr**, ending in a defended buy call — and, per the Worked example above, a short note on whether your conclusion is robust to comparing at a different matched tier.

Acceptance = the module [checkpoint](../checkpoint.md). The report is done when every checkpoint box ticks and you can defend each number — including its pricing tier and date — from memory.

## Common pitfalls

1. **Tier-blind price comparison.** Comparing a Bronze-tier quote for one SKU against a Silver-or-above quote for another (or vice versa) and treating the resulting cost gap as a hardware conclusion. The Worked example above shows this can flip a recommendation entirely, by a margin much larger than the real hardware difference.
2. **Single-point pricing.** Quoting "the" H100 price as if one number existed. Real pricing spans Bronze ($1.45–2.00/hr) through Silver ($2.10–2.99/hr) and beyond, depending on contract term, provider tier, and date — always cite a range and a tier, not a point estimate.
3. **Stale pricing in a report, undated.** SemiAnalysis's own data shows the cohort median falling from >$7/hr (early 2024) to ~$3.15/hr in roughly two years. A dollar figure without a date attached is unverifiable and probably wrong within months — stamp every price with the date you pulled it.
4. **Treating public $/token benchmarks as directly reproducible.** Artificial Analysis and similar aggregators weight input:output tokens at a fixed ratio (3:1, per their published methodology); a workload with a different real ratio will see a different true cost. Use these as a sanity check on your own measured number, not a substitute for it.
5. **Conflating a throughput-only multiple with a compounded one.** Databricks' ~1.5× realized FP8 training throughput (fixed hardware footprint) and this lesson's ~2–2.5× FP8 serving cost improvement (throughput *and* footprint change together) are both real and both correctly cited — but citing one number while implying the other's scope is a common, easy-to-make error. State explicitly what's held fixed.

## Self-check

- For the stated 70B FP8 decode workload, is H100 or H200 cheaper per token? Does the answer depend on anything besides the hardware? **Answer:** At matched pricing tiers (Bronze-vs-Bronze or Silver-vs-Silver), H200 is ~17–18% cheaper per token: `$/1M = ($/hr)/(tok/s×3600/1e6)`; e.g. Bronze — H100 `1.45/(2000×3600/1e6)=$0.201`, H200 (est.) `2.03/(3400×3600/1e6)=$0.166`. The ranking holds at any *matched* tier because the H200's 1.43× bandwidth and 1.76× capacity deliver 1.7× the tokens for a roughly proportional price premium. But yes, it depends on something besides hardware: comparing *across* tiers (H100 Bronze vs H200 Silver) flips the ranking to H100 by a ~70% margin — the tier match, not just the SKU choice, determines whether your conclusion is a hardware fact or a pricing artifact.
- Why can't you meaningfully say "H100 costs $X/hr" as a single number in 2026? **Answer:** SemiAnalysis's ClusterMAX data shows a real, verified spread — Bronze-tier 1-year-reserved H100 pricing runs $1.45–2.00/hr, Silver-tier $2.10–2.99/hr, with a cohort median around $3.15/GPU-hour in this pass. The tier — a composite of performance, networking, storage, security, and support rating, not the silicon — is part of the price, not noise around one true number. Always state the tier, provider, and contract term alongside any $/hr figure you cite.
- Where did `nvidia-smi GPU-Util` mislead in your report, and which metric corrected it? **Answer:** In the decode/util-lies section, `GPU-Util` reads ~100% during decode because a kernel is resident on every sample interval, implying "fully used." But compute-MFU is single-digit there — decode is memory-bound (L3–L4), so the SMs are mostly stalled waiting on HBM, exactly matching the Google Cloud B200 benchmark's 4.4% FLOPS-utilization / 1.5% tensor-active result at real production scale. The corrective metrics are **achieved bandwidth utilization** (achieved GB/s ÷ HBM spec GB/s) for the ceiling, and **aggregate tok/s** for useful work delivered. `GPU-Util` measures residency, not FLOPs or tokens produced.
- Your FP8-vs-FP16 $/1M-token estimate, with assumptions, and which multiple (throughput-only or compounded) are you quoting? **Answer:** ~$0.20/1M (FP8, single H100 Bronze, $1.45/hr, 2,000 tok/s) vs. roughly 2–2.5× that at FP16 with forced TP=2 on this model size (two GPUs, halved KV room, added communication tax) — a **compounded** multiple, because it folds in both the doubled bytes-per-token *and* the hardware-footprint change from single-GPU to TP=2 placement. This is explicitly not the same number as Databricks' reported ~1.5× realized FP8 *training* throughput, which holds the hardware footprint fixed and measures throughput only — state which one you mean before quoting a multiple.
- What does the ClusterMAX rating actually score, beyond price? **Answer:** Performance, networking, storage, security, and support — the composite that produces the Bronze/Silver/Gold/Platinum tiers. It exists because the same GPU silicon, rented from different providers, is not a fungible product once you account for how reliably it stays up, how good the interconnect actually is under load, and how fast a support ticket gets answered — the same fleet-validation discipline CoreWeave runs hourly against idle nodes in its own HPC Verification framework.

## Connections & what's next

This lesson is the module's convergence point: L1's util-lie is the discipline that keeps your throughput numbers honest; L2's roofline tells you *which* ceiling (compute or memory) governs the workload you're pricing; L3–L4's memory-hierarchy math is where the tok/s ceiling and batch curve you plug into `$/1M-tokens` actually come from; L5's precision lever is the single biggest throughput multiplier available without new hardware; L6's generational SKU table and operational discipline is what you're choosing *between*. None of those lessons stand alone as interview answers — this one is where they become a single defensible number.

It also opens the next two modules directly. **Module 04 (GPU on Kubernetes)** takes the cost-attribution instinct built here and makes it *per-pod* and *live* — your `gpu-cost-operator` will need exactly this $/useful-work framing, but computed continuously against a running fleet instead of a weekend benchmark. **Module 11 (GPU cost and unit economics)** — the program's signature module — goes further still: allocated-vs-utilized cost, $/token and $/run as an organizational metric, and the FOCUS cost-schema standard, all resting on the cost-per-useful-work literacy this capstone builds. The tier-pricing discipline you just practiced — never quote a dollar figure without its date, provider, and tier — is the same discipline module 11 will demand at fleet scale, not benchmark scale.

There is no next lesson in this module — the [checkpoint](../checkpoint.md) is next. Ship the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md), answer the checkpoint's depth probes cold, and flip this module's status.

## References & further reading

**Primary sources**
- SemiAnalysis, [ClusterMAX GPU cloud ratings](https://clustermax.semianalysis.com/) — the live tiering system (Bronze/Silver/Gold/Platinum) this lesson's pricing table is built from; **time-sensitive, re-check the current tier bounds before quoting.**
- SemiAnalysis, [GPU rental price index](https://gpu-index.semianalysis.com/) — live $/hr across SKUs and contract terms; **time-sensitive, refresh every number before you cite it.**
- [Artificial Analysis](https://artificialanalysis.ai/) and its [pricing methodology](https://artificialanalysis.ai/methodology) — read for the 3:1 input:output weighting convention before treating a public $/token figure as directly comparable to your own.
- NVIDIA, H100 Tensor Core GPU Architecture whitepaper ([resources.nvidia.com/en-us-tensor-core](https://resources.nvidia.com/en-us-tensor-core)) — canonical dense/sparse FP8/BF16 TFLOPS source behind Step 1's MFU math.
- NVIDIA, [H200 datasheet / product page](https://www.nvidia.com/en-us/data-center/h200/) — canonical 141 GB / 4.8 TB/s source behind the Worked example's hardware table.

**Real-world engineering blogs**
- SemiAnalysis, ["The GPU Cloud ClusterMAX Rating System — How to Rent GPUs"](https://newsletter.semianalysis.com/p/the-gpu-cloud-clustermax-rating-system-how-to-rent-gpus) — what it shows: why raw $/hr comparison across neoclouds is misleading without a reliability tier attached.
- SemiAnalysis, ["How Much Do GPU Clusters Really Cost?"](https://newsletter.semianalysis.com/p/how-much-do-gpu-clusters-really-cost) — what it shows: the market-level price collapse (>$7/hr early 2024 → ~$3.15/hr this pass) that motivates dating every dollar figure.
- Databricks/MosaicML, ["LLM Inference Performance Engineering: Best Practices"](https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices) — what it shows: a production multi-backend serving team's prefill/decode cost framing, underlying this lesson's Step 4 formulas.
- Databricks/MosaicML, ["Turbocharged Training: Optimizing the Databricks Mosaic AI Stack With FP8"](https://www.databricks.com/blog/turbocharged-training-optimizing-databricks-mosaic-ai-stack-fp8) — what it shows: a real ~1.5× realized FP8 throughput number at >50% MFU, the throughput-only multiple this lesson contrasts against its own compounded 2–2.5× serving figure.
- CoreWeave, [HPC Verification docs](https://docs.coreweave.com/docs/platform/fleet-management/hpc-verification) — what it shows: the hourly, per-idle-node hardware-validation framework behind what a high-tier GPU-cloud rating is actually buying.
- Google Cloud (Medium), ["What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0) — what it shows: a real, dated B200 production benchmark (96×B200, ~1M tok/s aggregate at 4.4% FLOPS utilization) that ties L1's util-lie, L2's roofline, and this lesson's cost math into one number.

**Deeper dives**
- Horace He, ["Making Deep Learning Go Brrrr From First Principles"](https://horace.io/brrr_intro.html) — the module's conceptual anchor; the compute/memory/overhead trichotomy behind every roofline placement this lesson prices.
- Google DeepMind, ["How to Scale Your Model" — "All About Rooflines"](https://jax-ml.github.io/scaling-book/roofline/) — a modern (2025), free, frontier-lab-authored complement to the 2009 roofline paper, with a GPU-specific chapter.
- Imbue, ["From bare metal to a 70B model: infrastructure set-up and scripts"](https://imbue.com/research/70b-infrastructure/) — a real, warts-and-all account of training at scale; grounds the "reliability is part of the cost" argument this lesson's tiering section makes in a concrete engineering narrative.
- Character.AI, ["Optimizing AI Inference at Character.AI"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/) and ["Part Deux"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-part-deux-2/) — production evidence that the precision and memory levers from L3/L5 compound (>20× KV-cache reduction), each compounding lever a further multiplier on the cost-per-token number this capstone teaches you to compute.
