---
lesson: "A03.2"
title: "Prometheus and PromQL"
module: "A-03"
concept: "promql-traps"
status: not-started
est_time: "3 hrs"
artifacts: ["corrected dashboard panel set"]
---

# A03.2 · Prometheus and PromQL

> **Concept.** The PromQL traps that ship confidently-wrong dashboards — and why each one is silently plausible until you know the semantics.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Why this matters

Every wrong dashboard is wrong *plausibly*. `rate()` on a gauge returns a number, `avg(p99)` returns a percentile-shaped line, and both render a smooth green panel that a senior engineer will trust in an incident. The staff-level skill is not writing PromQL — you already do — it is reading a panel and knowing, from the semantics of the function, that the number on it is fiction. In an incident the cost of a confidently-wrong latency panel is measured in minutes of chasing the wrong cause.

The interview stake is that these are the questions that separate "I know rate() and histogram_quantile()" from "I know *when they lie*." Quantile-of-quantiles, bucket-boundary interpolation error, staleness vs flatline, and rate-window sizing are the four or five semantic traps that a staff platform engineer must be able to spot on sight and correct on a whiteboard, with the corrected query written out.

On a GPU fleet these traps have teeth. `DCGM_FI_DEV_GPU_UTIL` is a **gauge**, and counter muscle memory makes `rate()`-ing it the single most common fleet-dashboard bug. SM-occupancy percentiles across thousands of GPUs are meaningless if you average per-node p99s. And node preemption — routine on a scheduled GPU fleet — must read as *staleness*, not a phantom flatline that pages the on-call at 3am for a node that was intentionally reclaimed.

## Core notes

**Skip (you already know):** the scrape/pull model and exposition format; `rate()` basics; the counter/gauge/histogram/summary types; that recording rules exist.

### Trap 1 — `rate()` on a gauge

`rate()` and `increase()` assume the series is a **monotonic counter** and apply **counter-reset correction**: any decrease is treated as a reset-to-zero and added back. On a gauge (which legitimately goes down), every downward movement is misread as a reset, so the output is garbage — not merely noisy, *wrong in sign and magnitude*.

For a gauge's rate of change use `deriv()` (least-squares per-second slope over the window, robust to noise) or `delta()` (last minus first). But usually the question you actually have about a gauge is "current value", "avg/min/max over window", or "how long above threshold" — not its derivative at all.

### Trap 2 — quantile-of-quantiles

`avg(p99)` — or `max`, or `quantile()` — **across instances is mathematically meaningless.** You cannot average percentiles; the p99 of the whole is not any function of the per-instance p99s. The correct pattern aggregates the **histogram bucket counters** *before* computing the quantile:

```promql
histogram_quantile(0.99,
  sum by (le) (rate(request_duration_seconds_bucket[5m]))
)
```

Sum the `rate()` of each `le` bucket across all instances (preserving `le`), *then* interpolate the quantile once over the merged distribution. The rule: **aggregate buckets, then quantile — never quantile, then aggregate.**

### Trap 3 — histogram_quantile is only as good as the buckets

`histogram_quantile` **linearly interpolates within the bucket** the quantile falls into. If your p99 lands in a `[1s, +Inf]` bucket, there is no upper bound to interpolate against and the result is a fabricated number (Prometheus effectively returns the lower bound / interpolates against an assumed edge — either way, fiction). Accuracy is bounded entirely by bucket-boundary placement: coarse or badly-placed buckets give confidently-precise-looking garbage. **Native histograms** (sparse, exponential, auto-scaling buckets) fix this by adapting resolution to the data; migrate latency SLIs to them where available.

### Trap 4 — rate() window sizing

The window must be **at least 4x the scrape interval** so it reliably contains >=2 samples even with a missed scrape; too short and you get gaps/NaN and jitter. Too long and the window *smooths away* the spikes you are trying to see — a 1h rate window will hide a 90-second latency excursion entirely. Match the window to what you need to detect, floored at 4x scrape.

### Trap 5 — counter resets and increase() extrapolation

`increase()` (and `rate()`) **extrapolate** to the window edges to correct for samples not landing exactly on the boundary, so `increase()` over a window can return **non-integer** counts (e.g. 7.3 events). Never alert on an *exact* value from these functions and never present them as an exact event count; they are rate estimates. Combined with reset correction across a restart, exact-count assertions are simply wrong.

### Staleness semantics

Prometheus marks a series **stale** when a scrape fails or a target disappears, and applies a **5-minute lookback**: after the staleness marker (or 5 min with no new sample) the series stops returning a value rather than carrying the last one forward. This is what prevents a **phantom flatline** — a scaled-down or preempted node holding its last value forever and either faking health or firing a stuck-metric alert. Understand it so your alerts distinguish "went away" (staleness / `absent()`) from "went to zero" (a real, present sample of 0).

### Recording rules as the scaling primitive

Recording rules **pre-aggregate high-fan-in queries** at evaluation time into new, low-cardinality series. They are how you make expensive dashboard/alert queries cheap and fast: collapse `sum by (tenant)(...)` over millions of raw series into a few hundred output series once per eval interval, then dashboards and alerts read the cheap rollup. This is also the mechanism that lets you keep raw series cardinality low (lesson 1) while still serving aggregate views.

### GPU-fleet tie

`DCGM_FI_DEV_GPU_UTIL` is a **gauge** — `rate()`-ing it out of counter habit gives nonsense (Trap 1); use current-value/`deriv()`. Fleet-wide **SM-occupancy percentiles** (`DCGM_FI_DEV_SM_ACTIVE` histogrammed) must be computed bucket-wise across nodes, never `avg(p99)` per node (Trap 2). And **node preemption** must surface as **staleness / `absent()`**, not a flatline — otherwise a reclaimed node either fakes 100% util forever or pages the on-call. (The GPU_UTIL-vs-SM_ACTIVE util-lie itself is covered in the separate GPU-observability artifact; here we only fix the *PromQL* over those signals.)

## Worked example

**A broken dashboard, diagnosed and rewritten.**

Panel A, "GPU utilization rate":

```promql
rate(DCGM_FI_DEV_GPU_UTIL[5m])
```

*Diagnosis:* `DCGM_FI_DEV_GPU_UTIL` is a gauge; `rate()` applies counter-reset correction, so every dip in utilization is misread as a reset and added back — the panel is garbage. *Fix* (the intended question is almost certainly "current utilization", possibly smoothed):

```promql
avg_over_time(DCGM_FI_DEV_GPU_UTIL[5m])
```

or, if genuinely asking for trend/slope, `deriv(DCGM_FI_DEV_GPU_UTIL[5m])`.

Panel B, "p99 request latency across fleet":

```promql
avg(histogram_quantile(0.99, rate(request_duration_seconds_bucket[5m])))
```

*Diagnosis:* two bugs stacked. (1) `histogram_quantile` is computed per-series first, then `avg()`-ed — a quantile-of-quantiles that is mathematically meaningless (Trap 2). (2) There is no `by (le)` grouping feeding the quantile, so `le` is not preserved for a clean read either. *Fix* — aggregate buckets first, quantile once:

```promql
histogram_quantile(0.99,
  sum by (le) (rate(request_duration_seconds_bucket[5m]))
)
```

Also sanity-check that the top populated bucket boundary sits above the p99 (Trap 3) — if p99 lives in `[1s, +Inf]`, add finer buckets or migrate to native histograms before trusting the number.

## Practice

Feeds the [fleet observability design](../practice/fleet-observability/README.md).

Assemble a **corrected dashboard panel set** for the fleet GPU/service SLIs. Take a starter dashboard containing at least one instance of each trap — a `rate()` on a DCGM gauge, an `avg(p99)`, a too-long/too-short rate window, and an alert that flatlines on preemption — and for each panel:

1. Write a one-line diagnosis naming the trap and *why the rendered number is wrong* (sign, meaning, or precision).
2. Write the corrected PromQL.
3. For the latency panel, state the bucket layout you need and whether native histograms are warranted.
4. For the preemption alert, replace the flatline condition with a staleness/`absent()`-based expression that distinguishes "node gone" from "util is genuinely 0".
5. Add the recording rule(s) that make the high-fan-in per-tenant panels cheap, and note the output cardinality.

## Self-check

- Why does `rate(gauge[5m])` return a wrong number rather than just a noisy one? **Answer:** `rate()` assumes monotonicity and applies counter-reset correction — every legitimate decrease in the gauge is interpreted as a reset-to-zero and added back into the total, so the output is wrong in both sign and magnitude, not merely noisy. Use `deriv()`/`delta()` for slope, or `avg_over_time`/current value for the question you usually actually have.
- Give the correct way to compute a fleet-wide p99 latency and say why `avg(p99)` is invalid. **Answer:** `histogram_quantile(0.99, sum by (le)(rate(bucket[5m])))` — sum the bucket-counter rates across instances preserving `le`, then interpolate the quantile once. `avg(p99)` is invalid because percentiles are not linear and the p99 of the merged population is not any function of the per-instance p99s; you must aggregate buckets *before* taking the quantile.
- On a scheduled GPU fleet, why must node preemption read as staleness rather than a flatline, and what expression captures it? **Answer:** A preempted node's series must stop returning a value (Prometheus staleness marker + 5-minute lookback) rather than carrying its last sample forward; a flatline would either fake continued health/util or fire a stuck-value alert for a node that was intentionally reclaimed. Detect "gone" with `absent(...)` / staleness and reserve a real `== 0` sample for genuinely-idle-but-present GPUs.

## References

- https://prometheus.io/docs/practices/histograms/
- https://prometheus.io/docs/prometheus/latest/querying/functions/
- https://prometheus.io/docs/prometheus/latest/querying/basics/
