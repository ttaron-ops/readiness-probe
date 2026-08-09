# Cost-per-million-tokens characterization — Module 07 deliverable

The unit-economics characterization of a real inference deployment — a flagship artifact
that **wires into the `gpu-cost-operator`'s token-cost scraping** and feeds Module 11.
Built on one rented GPU (H100 ~$2.89/hr or A100 ~$1.39/hr; ~$30–50 across the module).

> The core equation:
> `effective_cpm = hourly_rate / (peak_output_tokens_per_sec × 3600 × utilization) × 1e6`

## Four components (one report/notebook)

1. **CPM-vs-batch-size curve.** Measured output tokens/sec swept across request rates via
   `vllm bench serve`, converted to CPM, with the **SLO knee** annotated (where TTFT p99
   crosses your target before throughput saturates).
2. **FP8-vs-FP16 saving.** The same curve overlaid: throughput multiplier (~1.5–1.8×) and
   CPM reduction (~40–50%), **plus a measured accuracy delta** on a small eval subset — so
   the saving is honest, not asserted.
3. **Cold-start measurement.** Seconds from scale-zero → first token, **cached vs uncached**
   weights, broken into image-pull / weight-load / engine-warmup.
4. **A tuned-to-a-TTFT-target deployment.** The final vLLM config
   (`gpu-memory-utilization`, `max-num-seqs`, `max-model-len`, `tensor-parallel-size`, dtype)
   that hits a stated TTFT p99 (e.g. ≤ 500 ms) at the lowest CPM, justified from the curves.

## Suggested layout

```
cost-per-token/
├── report.md            # the writeup (components 1–4)
├── bench/               # raw vllm bench serve JSON + sweep configs
├── plot.py              # CPM-vs-batch + FP8-vs-FP16 plotting
├── eval/                # the FP8-vs-FP16 accuracy-delta subset run
├── configs/             # the tuned vLLM launch config
└── README.md            # how to reproduce + the CPM emitted into gpu-cost-operator
```

## Acceptance criteria (matches the [checkpoint](../../checkpoint.md))

- [ ] CPM-vs-batch curve with the SLO knee identified, from your own GPU run
- [ ] FP8-vs-FP16 CPM saving **with a measured accuracy delta** (not asserted)
- [ ] cold-start breakdown (cached vs uncached weights)
- [ ] a final config hitting a stated TTFT-p99 target at the lowest CPM, justified from the curves
- [ ] the CPM number is emitted into `gpu-cost-operator`'s scrape path (reusable, not a throwaway)

## Guardrails

- **Pin the vLLM version** in every command (V1 engine) — stale commands are the #1 way this rots.
- Rent on-demand, kill promptly; use an 8B model for mechanism slots, reserve 70B/TP for the capstone measurements.
- Publishable-by-default (sibling of Module 03's cost report and Module 05's dashboard) — scrub provider/account specifics.
- No API keys or credentials committed (repo `.gitignore` guards these).
