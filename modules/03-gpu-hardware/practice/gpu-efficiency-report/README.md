# GPU Efficiency & Cost Benchmark Report — Module 03 deliverable

A short written report + notebook produced on **one rented H100 over a weekend
(~$25–35)**. It turns this module's concepts into a defensible **cost-per-unit-of-
useful-work** number and a SKU recommendation — portfolio-grade, and it maps 1:1 onto
the module's interview questions.

> One H100 by the hour, dated snapshot (Aug 2026) via SemiAnalysis's ClusterMAX
> tiering: **Bronze $1.45–2.00/hr** (SF Compute, Vast.ai, Hyperstack, RunPod
> Community), **Silver $2.10–2.99/hr** (Lambda, AWS, GMI Cloud, Scaleway),
> cohort median ~$3.15/hr. Rent whatever tier is cheapest for this one-off
> session — but **record which tier you rented at**, since lesson 07 shows the
> cost-per-token conclusion can depend on it. The whole module's hands-on fits
> a single focused session ≈ 7–8 GPU-hours.

## The report must contain

1. **Achieved vs spec TFLOPS** (from L2) — a large matmul at BF16 and FP8, achieved/spec
   % (a crude MFU), with a **roofline plot** placing it under the compute roof.
2. **FP16 → FP8 delta** (from L5) — measured throughput multiple (~2×), memory-footprint
   halving, and the resulting change in max batch size / model fit.
3. **The "utilisation lies" demo** (from L1) — a workload where `nvidia-smi GPU-Util`
   ≈ 100% while DCGM tensor-active / SM-occupancy is near-zero, **side by side**, with a
   one-paragraph explanation.
4. **Decode ceiling** (from L3–L4) — measured single-stream tok/s for a real model vs the
   HBM-bandwidth-derived estimate, plus the batched-throughput curve until memory caps it.
5. **Cost-per-useful-work analysis** (from L7) — using **live, tier-labeled $/hr** (state the
   ClusterMAX-style tier — Bronze/Silver/Gold/Platinum — provider, contract term, and date for
   every price you cite), compute $/1M tokens (or $/useful-PFLOP) at FP16 vs FP8, and a
   one-page **SKU recommendation** (e.g. H100 vs H200 for a stated 70B serving workload)
   justified by the measured numbers — including a short note on whether the recommendation
   holds if you re-price both SKUs at a matched pricing tier (see L7's worked example).

## Suggested layout

```
gpu-efficiency-report/
├── report.md            # the writeup (sections 1–5 above)
├── benchmark.ipynb      # matmul / bandwidth / FP16-vs-FP8 / decode runs
├── roofline.png         # the roofline plot
├── util-lies.png        # GPU-Util vs tensor-active, side by side
├── raw/                 # captured nvidia-smi + DCGM logs, tok/s measurements
└── README.md            # how to reproduce + the SKU conclusion
```

## Acceptance criteria (matches the [checkpoint](../../checkpoint.md))

- [ ] achieved-vs-spec TFLOPS at BF16 & FP8, on a roofline plot
- [ ] measured FP16→FP8 throughput + memory + batch-size delta
- [ ] the util-lies screenshot (GPU-Util ≈100% vs tensor-active ≈0) with explanation
- [ ] measured decode tok/s vs the bandwidth-derived estimate + the batch curve
- [ ] $/1M-token (FP16 vs FP8) from live, **tier-labeled** $/hr, and a defended H100-vs-H200
      recommendation
- [ ] every assumption stated; utilisation vs useful work distinguished throughout;
      pricing tier/provider/date stated next to every dollar figure

## Guardrails

- Publishable-by-default (this is a strong blog post and the sibling of Module 05's
  "your GPU dashboard is lying" piece) — scrub any provider/account specifics.
- **Pricing is time-sensitive and tiered, not a single number per SKU** — refresh $/hr from
  SemiAnalysis's [ClusterMAX ratings](https://clustermax.semianalysis.com/) and
  [GPU rental index](https://gpu-index.semianalysis.com/) at build time, and cite the tier
  (Bronze/Silver/Gold/Platinum), provider, contract term, and date alongside every figure —
  a >2× spread exists across tiers for the same silicon, and comparing SKUs across mismatched
  tiers can flip your SKU recommendation (see lesson 07's worked example).
- No secrets or API keys in git (repo `.gitignore` guards these).
