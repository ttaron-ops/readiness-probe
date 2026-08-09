# GPU Efficiency & Cost Benchmark Report — Module 03 deliverable

A short written report + notebook produced on **one rented H100 over a weekend
(~$25–35)**. It turns this module's concepts into a defensible **cost-per-unit-of-
useful-work** number and a SKU recommendation — portfolio-grade, and it maps 1:1 onto
the module's interview questions.

> One H100 by the hour: Lambda H100 SXM ~$3.78/hr, Together ~$3.49/hr, spot cheaper.
> The whole module's hands-on fits a single focused session ≈ 7–8 GPU-hours.

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
5. **Cost-per-useful-work analysis** (from L7) — using **live $/hr**, compute $/1M tokens
   (or $/useful-PFLOP) at FP16 vs FP8, and a one-page **SKU recommendation** (e.g. H100 vs
   H200 for a stated 70B serving workload) justified by the measured numbers.

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
- [ ] $/1M-token (FP16 vs FP8) from live $/hr, and a defended H100-vs-H200 recommendation
- [ ] every assumption stated; utilisation vs useful work distinguished throughout

## Guardrails

- Publishable-by-default (this is a strong blog post and the sibling of Module 05's
  "your GPU dashboard is lying" piece) — scrub any provider/account specifics.
- **Pricing is time-sensitive** — refresh $/hr from SemiAnalysis / provider pages at build time.
- No secrets or API keys in git (repo `.gitignore` guards these).
