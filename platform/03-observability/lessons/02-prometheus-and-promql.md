---
lesson: "A03.2"
title: "Prometheus and PromQL"
module: "A-03"
concept: "promql-traps"
status: not-started
est_time: "4 hrs"
prev: "01-signal-model.md"
next: "03-metrics-at-scale.md"
artifacts: ["corrected dashboard panel set"]
sources: 7
---

# A03.2 · Prometheus and PromQL

> **Concept.** The PromQL traps that ship confidently-wrong dashboards — and why each one is silently plausible until you know the semantics.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 1 established cardinality as the master constraint on *which signal* answers a question and *what it costs* to keep it queryable. This lesson picks up right where that leaves off: once the signal is metrics and the series budget is sane, the next failure mode isn't cost — it's correctness. PromQL has several functions that return a plausible-looking number even when the query violates the semantics of the underlying signal (a gauge treated as a counter, a percentile averaged across instances). Getting the signal model right in lesson 1 and then writing a query that silently lies in lesson 2 is how a well-architected metrics pipeline still ships a wrong dashboard.

## Why this matters

Every wrong dashboard is wrong *plausibly*. `rate()` on a gauge returns a number, `avg(p99)` returns a percentile-shaped line, and both render a smooth green panel that a senior engineer will trust in an incident. The staff-level skill is not writing PromQL — you already do — it is reading a panel and knowing, from the semantics of the function, that the number on it is fiction. In an incident the cost of a confidently-wrong latency panel is measured in minutes of chasing the wrong cause.

The interview stake is that these are the questions that separate "I know rate() and histogram_quantile()" from "I know *when they lie*." Quantile-of-quantiles, bucket-boundary interpolation error, staleness vs flatline, and rate-window sizing are the four or five semantic traps that a staff platform engineer must be able to spot on sight and correct on a whiteboard, with the corrected query written out.

On a GPU fleet these traps have teeth. `DCGM_FI_DEV_GPU_UTIL` is a **gauge**, and counter muscle memory makes `rate()`-ing it the single most common fleet-dashboard bug. SM-occupancy percentiles across thousands of GPUs are meaningless if you average per-node p99s. And node preemption — routine on a scheduled GPU fleet — must read as *staleness*, not a phantom flatline that pages the on-call at 3am for a node that was intentionally reclaimed.

## What's new here (calibration)

- **Skip (you already know):** the scrape/pull model and exposition format; `rate()` basics; the counter/gauge/histogram/summary types; that recording rules exist.
- **New:** the exact semantic traps, why each one produces a *plausible* wrong answer rather than an obvious error, and the corrected query for each.
- **New:** the interpolation math behind `histogram_quantile`'s bucket-boundary error, and what native histograms actually change about it (and don't, until you've done the rollout work).
- **New:** the query-author vs dashboard-consumer split — why a recording rule or a pre-incident audit is the actual staff-level fix, not just knowing the right query in the moment.

## Core concepts

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

`histogram_quantile` **linearly interpolates within the bucket** the quantile falls into, and it does so on an assumption: that observations are uniformly distributed within that bucket. It computes the target rank and interpolates linearly between the bucket's lower and upper `le` boundaries. When the true distribution is skewed within that bucket — the common case near a long tail — the interpolated value can be wrong by up to the full width of the bucket. A `[100ms, 1s]` bucket holding your p99 has a potential 900ms error band; the panel will show a single precise-looking number somewhere in that band with no indication of the uncertainty.

If your p99 lands in a `[1s, +Inf]` bucket, there is no upper bound to interpolate against and the result is outright fabricated (Prometheus effectively returns the lower bound or an assumed edge — either way, fiction). Accuracy is bounded entirely by bucket-boundary placement: coarse or badly-placed buckets give confidently-precise-looking garbage.

**Native histograms** (sparse, exponentially-scaled buckets, auto-chosen resolution per series) fix this at the source by storing a finer, adaptive bucket layout instead of a handful of hand-picked boundaries — a direct consequence of the fact that classic histograms only ever stored cumulative bucket counts, never the raw observations. The feature reached GA in Grafana Cloud in October 2025 ([Grafana Labs](https://grafana.com/blog/prometheus-native-histograms-in-grafana-cloud-more-precise-easier-to-use-and-better-compatibility/)) — evidence the interpolation error was significant enough industry-wide to drive a multi-year core Prometheus feature, not a theoretical nitpick. Migrate latency SLIs to native histograms where available, but see the rollout caveat below before treating it as a drop-in fix.

### Trap 4 — rate() window sizing

The window must be **at least 4x the scrape interval** so it reliably contains ≥2 samples even with one missed scrape. Violate this and `rate()` doesn't error — it returns `NaN` or a wildly extrapolated number, silently. Too long a window and it *smooths away* the spikes you are trying to see — a 1h rate window will hide a 90-second latency excursion entirely. Match the window to what you need to detect, floored at 4x scrape.

### Trap 5 — counter resets and increase() extrapolation

`increase()` (and `rate()`) **extrapolate** to the window edges to correct for samples not landing exactly on the boundary, so `increase()` over a window can return **non-integer** counts (e.g. 7.3 events) — a genuine count of 10 could just as easily read as 9.4 or 10.7. Never alert on an *exact* value from these functions and never present them as an exact event count; they are rate estimates. Counter-reset detection itself is heuristic, too: a counter that wraps or resets mid-scrape-interval and increases again *before* the next scrape is entirely invisible to Prometheus — there's no sample that captures the reset, so no correction gets applied.

### Trap 6 — `rate()` and `irate()` are not interchangeable

`irate()` computes an instant rate from only the last two data points in the window, which makes it spikier and noisier than `rate()`'s least-squares-style smoothing over the whole window. That makes `irate()` appropriate for a high-resolution dashboard zoom-in where you want to see the most recent wiggle, and inappropriate for alerting — a single noisy scrape produces a spike in `irate()` that `rate()` would have smoothed out, and an alert built on it pages on noise. Treat the choice as "which question am I asking" (recent instantaneous value vs. stable trend), not "which one is more precise."

### Staleness semantics

Prometheus marks a series **stale** when a scrape fails or a target disappears, and applies a **5-minute lookback**: after the staleness marker (or 5 min with no new sample) the series stops returning a value rather than carrying the last one forward. This is what prevents a **phantom flatline** — a scaled-down or preempted node holding its last value forever and either faking health or firing a stuck-metric alert.

Staleness and `absent()` are **not the same mechanism**, and conflating them is itself a trap: staleness is automatic — Prometheus's own 5-minute-lookback handling of a vanished series — while `absent()` is a query-time function you must explicitly wire into an alert expression to *act* on that absence. A series going stale, by itself, does not generate an alert; if nothing in your rule set calls `absent(...)` (or an equivalent "this series should exist and doesn't" check), a preempted node's disappearance is invisible to alerting even though the series is correctly, internally, marked stale.

### Recording rules as the scaling primitive — and what they don't fix

Recording rules **pre-aggregate high-fan-in queries** at evaluation time into new, low-cardinality series. They are how you make expensive dashboard/alert queries cheap and fast: collapse `sum by (tenant)(...)` over millions of raw series into a few hundred output series once per eval interval, then dashboards and alerts read the cheap rollup. This is also the mechanism that lets you keep raw series cardinality low (lesson 1) while still serving aggregate views.

What a recording rule does **not** do is fix a wrong query. A recording rule pre-computes and caches whatever expression you give it — if that expression is `avg(histogram_quantile(...))` or `rate()` on a gauge, the recording rule just makes the wrong answer cheaper to compute and, worse, gives it the look of an official, pre-vetted metric name. Treat a recording rule definition as a place the trap can hide *more* durably, not a place it gets fixed.

### GPU-fleet tie

`DCGM_FI_DEV_GPU_UTIL` is a **gauge** — `rate()`-ing it out of counter habit gives nonsense (Trap 1); use current-value/`deriv()`. Fleet-wide **SM-occupancy percentiles** (`DCGM_FI_DEV_SM_ACTIVE` histogrammed) must be computed bucket-wise across nodes, never `avg(p99)` per node (Trap 2). And **node preemption** must surface as **staleness / `absent()`**, not a flatline — otherwise a reclaimed node either fakes 100% util forever or pages the on-call. This class of trap isn't unique to DCGM: the same gauge-vs-counter confusion shows up broadly in Kubernetes metrics — CPU-throttling gauges `rate()`'d incorrectly, container-restart counters misread — which is exactly the shape of bug a DaemonSet-based DCGM exporter reproduces at fleet scale ([Robusta.dev](https://home.robusta.dev/blog/3-common-mistakes-with-promql-and-kubernetes-metrics)). (The GPU_UTIL-vs-SM_ACTIVE util-lie itself is covered in the separate GPU-observability artifact; here we only fix the *PromQL* over those signals.)

## Perspectives

**Query-author (developer).** These traps are the default output of naive query construction — Grafana's query builder will happily suggest `rate()` on anything that looks numeric, gauge or counter alike, and it won't warn you. The staff move is muscle memory, applied every single time before the query is written: "counter or gauge?" first, then pick the function. The trap isn't a knowledge gap once you've seen it; it's a discipline gap in the thirty seconds before you type `rate(`.

**Dashboard-consumer (incident-responder).** During an incident nobody re-derives the PromQL behind a panel — they trust it and act on it. That makes the staff responsibility upstream of the incident: auditing dashboards *before* they're needed, because an incident is the worst possible time to discover a panel has been lying. A pre-incident PromQL audit (walk every panel, name the trap if any, fix it) is a deliverable, not a nice-to-have.

**Storage-engine.** `histogram_quantile`'s interpolation-within-bucket behavior isn't an implementation quirk — it's a direct consequence of what classic histograms actually store: cumulative counts per bucket boundary, never the raw observations. There is nothing to interpolate *from* except an assumption of uniform distribution within the bucket. Native histograms fix this structurally, not incrementally — by storing a sparse, exponentially-scaled bucket layout that Prometheus auto-selects per series, so the resolution adapts to the data instead of being fixed at metric-definition time.

**Cost/perf.** An under-windowed `rate()` recomputed live on every dashboard load, fanned out across thousands of raw series, isn't just a correctness bug — it's a direct query-latency and Prometheus-CPU cost. The same trap that ships a wrong number often also ships a slow one, because the naive query pattern (compute the expensive thing live, on every panel load, over raw high-cardinality series) is exactly the pattern recording rules exist to eliminate. Fixing the semantics and fixing the performance are usually the same edit.

## Real-world use cases

- **PromLabs (Julius Volz, ex-Prometheus core team), ["Avoid These 6 Mistakes When Getting Started With Prometheus"](https://promlabs.com/blog/2022/12/11/avoid-these-6-mistakes-when-getting-started-with-prometheus/).** Written by an original Prometheus author. What it shows: the rate()-window, counter/gauge, and aggregation mistakes in this lesson aren't edge cases someone found once — they're common enough that a core author wrote a canonical mistakes list around exactly them.
- **Grafana Labs, ["Prometheus native histograms in Grafana Cloud"](https://grafana.com/blog/prometheus-native-histograms-in-grafana-cloud-more-precise-easier-to-use-and-better-compatibility/).** What it shows: the bucket-boundary interpolation error (Trap 3) was significant enough industry-wide to drive a multi-year Prometheus core feature through to GA (October 2025) — this is not a theoretical concern.
- **Robusta.dev, ["3 Common Mistakes with PromQL and Kubernetes Metrics"](https://home.robusta.dev/blog/3-common-mistakes-with-promql-and-kubernetes-metrics).** What it shows: the K8s-specific angle — CPU-throttling gauges `rate()`'d incorrectly, container-restart counters misread — maps directly onto GPU-fleet DaemonSet scrape patterns (DCGM exporters run the same way).

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

Also sanity-check that the top populated bucket boundary sits above the p99 (Trap 3) — if p99 lives in `[1s, +Inf]`, add finer buckets or migrate to native histograms before trusting the number. If the panel is a live-updating "current spike" view rather than an alerting input, consider whether `irate()` (Trap 6) is actually the better fit for the dashboard-zoom use case — but never swap it in for the version that feeds an alert.

## Practice

Feeds the [fleet observability design](../practice/fleet-observability/README.md).

Assemble a **corrected dashboard panel set** for the fleet GPU/service SLIs. Take a starter dashboard containing at least one instance of each trap — a `rate()` on a DCGM gauge, an `avg(p99)`, a too-long/too-short rate window, an `irate()` wired into an alert instead of a dashboard, and an alert that flatlines on preemption — and for each panel:

1. Write a one-line diagnosis naming the trap and *why the rendered number is wrong* (sign, meaning, or precision).
2. Write the corrected PromQL.
3. For the latency panel, state the bucket layout you need and whether native histograms are warranted — and if so, name the rollout dependencies (Prometheus version, client-library support, storage-format compatibility), not just "switch to native histograms."
4. For the preemption alert, replace the flatline condition with a staleness/`absent()`-based expression that distinguishes "node gone" from "util is genuinely 0".
5. Add the recording rule(s) that make the high-fan-in per-tenant panels cheap, and note the output cardinality — and confirm the recording rule's *expression* is itself already correct, not just cached.

## Common pitfalls

- **"rate() and irate() are interchangeable."** They answer different questions. `irate()` (last two points only) suits fast-changing dashboard zoom-ins; `rate()` suits alerting and trend stability. Wiring `irate()` into an alert produces false pages from single-scrape noise.
- **"A recording rule fixes a wrong query."** A recording rule pre-computes and caches whatever expression you give it. If the underlying PromQL has a trap baked in, the recording rule just makes it cheaper to compute and gives it the appearance of being official and pre-vetted — it doesn't validate the semantics.
- **"increase() gives an exact count, safe to alert on exact thresholds."** Extrapolation at window edges means the returned value is fractional/estimated, not exact — a genuine count of 10 could read as 9.4 or 10.7. Alert on ranges or rates, never on an exact-equality threshold against `increase()` output.
- **"Native histograms are a drop-in replacement."** They require Prometheus ≥2.40 with an experimental flag, client-library support for emitting them, and storage-format compatibility on the read side — a coordinated multi-component rollout, not a config flip you make once and forget.
- **"Staleness and absent() are the same mechanism."** Staleness is automatic — Prometheus's built-in 5-minute-lookback handling of a vanished series. `absent()` is a query-time function you must explicitly wire into an alert expression. A series correctly going stale generates no alert by itself; without an explicit `absent()` (or equivalent) check in your rule set, a preempted node's disappearance is silently invisible to alerting.

## Self-check

- Why does `rate(gauge[5m])` return a wrong number rather than just a noisy one? **Answer:** `rate()` assumes monotonicity and applies counter-reset correction — every legitimate decrease in the gauge is interpreted as a reset-to-zero and added back into the total, so the output is wrong in both sign and magnitude, not merely noisy. Use `deriv()`/`delta()` for slope, or `avg_over_time`/current value for the question you usually actually have.
- Give the correct way to compute a fleet-wide p99 latency and say why `avg(p99)` is invalid. **Answer:** `histogram_quantile(0.99, sum by (le)(rate(bucket[5m])))` — sum the bucket-counter rates across instances preserving `le`, then interpolate the quantile once. `avg(p99)` is invalid because percentiles are not linear and the p99 of the merged population is not any function of the per-instance p99s; you must aggregate buckets *before* taking the quantile.
- On a scheduled GPU fleet, why must node preemption read as staleness rather than a flatline, and what expression captures it? **Answer:** A preempted node's series must stop returning a value (Prometheus staleness marker + 5-minute lookback) rather than carrying its last sample forward; a flatline would either fake continued health/util or fire a stuck-value alert for a node that was intentionally reclaimed. Detect "gone" with `absent(...)` / staleness-aware alerting and reserve a real `== 0` sample for genuinely-idle-but-present GPUs — staleness itself does not generate an alert without an explicit check.
- A dashboard panel switches from `rate()` to `irate()` "for a more accurate real-time view" and then that same expression gets copied into an alert rule. What breaks? **Answer:** `irate()` computes an instant rate from only the last two samples, so it's much noisier than `rate()`'s smoothed trend over the whole window. Fine for a dashboard zoom where a human eyeballs the wiggle; wired into an alert, a single noisy scrape produces a spike that pages on noise rather than a real sustained change. The two functions answer different questions and shouldn't be interchanged across dashboard and alert use.
- A teammate says "I fixed the latency panel by wrapping it in a recording rule, it's fast now." What follow-up question exposes whether the fix is real? **Answer:** "What's the underlying expression?" A recording rule only pre-computes and caches whatever PromQL you give it — if that expression still contains a trap (quantile-of-quantiles, `rate()` on a gauge), the recording rule makes the wrong number cheaper to serve and gives it an official-looking metric name, without correcting the semantics. Speed is orthogonal to correctness here.

## Connections & what's next

This lesson assumes the label/cardinality decisions from [lesson 1](01-signal-model.md) are already made — the recording rules here exist to keep the aggregations those decisions require cheap, not to fix a mis-scoped label set. Lesson 3 picks up what happens when these same query patterns run at genuine fleet fan-in (rule-evaluation cost, federation, Thanos vs Mimir). Lesson 6 shows the equivalent correctness-trap domain for log queries. Lesson 9 applies these exact traps — gauge vs counter, quantile aggregation, staleness vs flatline — to the concrete DCGM and NCCL signal set at fleet scale.

Next: [03 · Metrics at scale](03-metrics-at-scale.md) — how a metrics system actually falls over once these correctly-written queries run against a fleet-sized series count.

## References & further reading

**Primary sources**
- [Prometheus histograms and summaries practices](https://prometheus.io/docs/practices/histograms/)
- [Prometheus query functions reference](https://prometheus.io/docs/prometheus/latest/querying/functions/)
- [Prometheus querying basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Prometheus native histograms specification](https://prometheus.io/docs/specs/native_histograms/) — the exact bucket-schema/exponential-boundary math behind Trap 3's fix.

**Real-world engineering blogs**
- PromLabs, [Avoid These 6 Mistakes When Getting Started With Prometheus](https://promlabs.com/blog/2022/12/11/avoid-these-6-mistakes-when-getting-started-with-prometheus/)
- Grafana Labs, [Prometheus native histograms in Grafana Cloud](https://grafana.com/blog/prometheus-native-histograms-in-grafana-cloud-more-precise-easier-to-use-and-better-compatibility/)

**Deeper dives**
- Robusta.dev, [3 Common Mistakes with PromQL and Kubernetes Metrics](https://home.robusta.dev/blog/3-common-mistakes-with-promql-and-kubernetes-metrics)
