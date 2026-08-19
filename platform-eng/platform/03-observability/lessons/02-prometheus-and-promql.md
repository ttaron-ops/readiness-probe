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
sources: 11
---

# A03.2 · Prometheus and PromQL

> **Concept.** The PromQL traps that ship confidently-wrong dashboards — and why each one is silently plausible until you know the semantics.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 1 established cardinality as the master constraint on *which signal* answers a question and what it costs to keep queryable. This lesson picks up where that leaves off: once the signal is metrics and the series budget is sane, the next failure mode is not cost — it is correctness. PromQL has a handful of functions that return a plausible-looking number even when the query violates the semantics of the underlying signal, and they do it silently. Getting the signal model right in lesson 1 and then writing a query that lies in lesson 2 is how a well-architected metrics pipeline still ships a wrong dashboard.

The through-line is specific: **every trap in this lesson is a case where the function's contract is stricter than its type signature.** `rate()` accepts any range vector and returns a float. Nothing in the type system says "this must be a monotonic counter." The contract lives in the implementation, so you have to know the implementation.

Everything below is checked against the **Prometheus 3.14.0** tree (`main`, August 2026): `promql/functions.go` (`extrapolatedRate`, `instantValue`, `linearRegression`), `promql/quantile.go` (`BucketQuantile`, `HistogramQuantile`, `ensureMonotonicAndIgnoreSmallDeltas`), `promql/engine.go` (`defaultLookbackDelta`), `config/config.go` (defaults), and `docs/querying/basics.md`. Where behaviour changed in Prometheus 3.0, that is called out.

## Why this matters

Every wrong dashboard is wrong *plausibly*. `rate()` on a gauge returns a number. `avg(p99)` returns a percentile-shaped line. A `histogram_quantile` whose p99 lands in the `+Inf` bucket returns a clean, precise-looking value. All three render a smooth panel that a senior engineer will trust at 3am. The staff-level skill is not writing PromQL — you already do — it is reading a panel and knowing, from the semantics of the function, that the number on it is fiction.

The cost is measured in incident minutes. A confidently-wrong latency panel does not merely fail to help; it actively routes the investigation somewhere else. Ten minutes chasing a phantom latency regression on a fleet where the real problem was a stuck NCCL collective is ten minutes of a large training run producing nothing, which at 512 H100s is real money. And because the panel looked fine before the incident, nobody audits it after — the trap survives to the next incident.

On a GPU fleet these traps have teeth in specific, repeatable ways. `DCGM_FI_DEV_GPU_UTIL` is a **gauge**, and counter muscle memory makes `rate()`-ing it the single most common fleet-dashboard bug. SM-activity percentiles across thousands of GPUs are meaningless if you average per-node percentiles. And node preemption — routine on a scheduled GPU fleet — must read as *staleness*, not a phantom flatline that pages the on-call for a node that was intentionally reclaimed.

## What's new here (calibration)

- **Skip (you already know):** the scrape/pull model and exposition format; `rate()` basics; the counter/gauge/histogram/summary types; that recording rules exist.
- **New:** the PromQL **evaluation model** — instant vs range vector, lookback delta, step, and the left-open/right-closed range change in Prometheus 3.0 — because half the traps are really evaluation-model misunderstandings wearing a function's clothes.
- **New:** the exact `extrapolatedRate` algorithm from source: how counter resets are detected and corrected, when extrapolation runs to the boundary and when it stops at half an interval, the 1.1× threshold, and the zero-point clamp. This is what makes the 4× window rule a derivation rather than folklore.
- **New:** `BucketQuantile` step by step, including the exact behaviour when the quantile lands in the `+Inf` bucket (it is not "fiction" — it is a specific, predictable clamp), the interpolation-from-zero in the first bucket, and the silent forced-monotonicity fix-up.
- **New:** native histograms as they actually ship in Prometheus 3.x — the schema/growth-factor table, logarithmic interpolation for exponential buckets, and the three scrape-config switches that control the migration.
- **Corrected:** the previous version of this lesson described `rate()` as doing "least-squares-style smoothing." It does not. `deriv()` does least squares; `rate()` is `(last − first + reset corrections) × extrapolation_factor / window`. The distinction matters for how you reason about noise.

## Core concepts

### 0. The evaluation model, because half the traps live here

Before any function, hold the machine. PromQL evaluates in one of two modes and the difference explains a lot of "why does my panel look like that."

**Instant query.** One evaluation timestamp `T`. A selector `foo{job="x"}` returns, for every matching series, **the most recent sample at or before `T`, provided it is within the lookback delta** — 5 minutes by default (`defaultLookbackDelta = 5 * time.Minute` in `promql/engine.go`, overridable with `--query.lookback-delta` or a per-query `lookback_delta` parameter). If the newest sample is older than that, the series simply is not in the result. It does not return zero. It does not error. It is absent.

**Range query.** A start, an end, and a `step`. Prometheus runs an independent instant query at `start`, `start+step`, `start+2×step`, … Each point on your Grafana graph is a separate evaluation. **The step is a display-resolution parameter, not a data parameter** — Grafana computes it from panel width, which is why the same panel can look different on a laptop and a wall display, and why `increase(x[$__interval])` behaves differently at different zoom levels.

**Range vector selector.** `foo[5m]` returns, for each series, *all* samples in the window. Since Prometheus 3.0 the window is **left-open and right-closed**: `(T−5m, T]`. Before 3.0 it was closed on both ends, which meant a query whose left boundary happened to align exactly with a scrape returned 6 samples on a 1-minute scrape instead of 5. The 3.0 change makes the count deterministic. If you have old dashboards with off-by-one-sample sensitivities, this is why they moved.

```
   INSTANT vs RANGE — WHAT EACH SELECTOR ACTUALLY RETURNS AT T
   ═══════════════════════════════════════════════════════════════════════════

   samples for one series (15 s scrape):
     ─┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬──▶ t
      s1   s2   s3   s4   s5   s6   s7   s8   s9  s10  s11  s12  s13   T

   foo            (instant selector, evaluated at T)
      → s13 only, IF (T − s13.timestamp) ≤ 5m lookback.
        Older than that → series ABSENT from the result, not zero.

   foo[1m]        (range selector, left-open right-closed)
      →           ┌──────────────────┐
        window =  (T−1m ────────────  T]
        → s10, s11, s12, s13   ← 4 samples at 15 s scrape
        s9 lands exactly at T−1m → EXCLUDED (left-open, Prom ≥3.0)

   rate(foo[1m])  needs ≥2 samples in the window, or it returns nothing.
                  With a 1 m window and a 15 s scrape you have 4 — one
                  missed scrape still leaves 3. With a 30 s window you
                  have 2, and one missed scrape leaves 1 → EMPTY PANEL.
```

Three consequences you will use all lesson:

1. **"No data" and "zero" are different states**, and PromQL distinguishes them where most people's mental model does not. A panel that shows a gap is telling you something.
2. **A rate window that is too short does not error — it returns nothing**, which renders as a gap, which people read as an outage.
3. **`for:` durations on alerts interact with the step**, because the alerting engine evaluates on `evaluation_interval` (default 1 m), not on your dashboard's step.

### 1. Trap 1 — `rate()` on a gauge

**The claim:** `rate()` and `increase()` assume a monotonically non-decreasing counter and apply *counter-reset correction*. On a gauge — which legitimately goes down — every downward movement is misread as a process restart and added back, so the output is wrong in magnitude and often in sign of trend.

**The mechanism, from source.** In `extrapolatedRate` (`promql/functions.go`), the float path is:

```go
resultFloat = samples.Floats[last].F - samples.Floats[0].F   // last minus first
if !isCounter { break }                                       // delta() stops here
for i, currPoint := range samples.Floats[1:] {
    prevPoint := samples.Floats[i]
    if currPoint.F < prevPoint.F { resultFloat += prevPoint.F }   // ← reset correction
}
```

Read that loop literally: **any time sample `i+1` is less than sample `i`, the full value of sample `i` is added to the result.** The assumption is that the counter reset to zero at that point, so the "lost" progress is exactly the pre-reset value.

**Worked garbage.** `DCGM_FI_DEV_GPU_UTIL` on one GPU, five samples over a minute at 15 s scrape, a workload that idles briefly mid-window:

```
   t:      0s   15s   30s   45s   60s
   value:  95    97    12    88    91         (percent, a GAUGE)

   True answer to "how did utilisation change?":  95 → 91, roughly flat.

   What rate() computes:
     last − first            = 91 − 95            = −4
     decrease at 15s→30s (97 → 12): add prevPoint  = +97
     decrease at 45s→60s? 88 → 91 is an increase, no correction
                                                   ─────
     resultFloat                                    93
     × extrapolation factor (≈1.0 here) / 60 s     = 1.55 "per second"

   The panel reads 1.55. The GPU did not change by 1.55 %/s. It did not
   change meaningfully at all. The number is manufactured entirely by the
   reset-correction branch firing on a legitimate gauge decrease.
```

**Why it looks plausible.** The output is positive, small, and moves when the workload moves — it correlates with activity. That is worse than an obviously broken panel, because it survives eyeballing.

**The fix depends on the question you actually have:**

| Question about a gauge | Function | Notes |
|---|---|---|
| what is it now? | `DCGM_FI_DEV_GPU_UTIL` | just read it |
| smoothed current value | `avg_over_time(x[5m])` | also `min_over_time`, `max_over_time`, `quantile_over_time` |
| how fast is it trending? | `deriv(x[30m])` | **least-squares** slope per second, robust to noise (`linearRegression` in `functions.go`) |
| how much did it change over the window? | `delta(x[1h])` | last − first, **with** extrapolation, **without** reset correction |
| will it cross a threshold? | `predict_linear(x[1h], 4*3600)` | extrapolates the `deriv` fit forward; the classic disk-full alert |
| how long was it above X? | `sum_over_time((x > bool 80)[1h:1m]) * 60` | subquery; each sample counted once |

Note the last row — **`delta()` extrapolates.** `funcDelta` calls `extrapolatedRate(..., isCounter=false, isRate=false)`, which skips the reset loop and skips the `/range` division but still applies the boundary extrapolation factor described in §2. So `delta()` on a gauge over an hour returns a slightly-more-than-observed change. That surprises people who expect exactly `last − first`.

### 2. The `rate()`/`increase()` algorithm in full — and where extrapolation comes from

You cannot size a rate window, reason about a spiky panel, or explain a fractional `increase()` without this. Here it is, in order, from `extrapolatedRate`:

```
  1. Collect samples in the window (T−range, T].  Need ≥2, else return nothing.
  2. resultFloat = last.value − first.value
  3. If counter (rate/increase): for each adjacent pair where value DROPS,
     add the pre-drop value back.  (One reset = one correction.)
  4. sampledInterval        = last.timestamp − first.timestamp
     durationToStart        = first.timestamp − rangeStart
     durationToEnd          = rangeEnd − last.timestamp
     avgInterval            = sampledInterval / (numSamples − 1)
     extrapolationThreshold = avgInterval × 1.1
  5. If durationToStart ≥ threshold → durationToStart = avgInterval / 2
     If durationToEnd   ≥ threshold → durationToEnd   = avgInterval / 2
     ("the series probably doesn't cover the whole window; guess halfway")
  6. Counter-only zero clamp: if the counter is rising, compute
        durationToZero = sampledInterval × (first.value / resultFloat)
     and if that is shorter than durationToStart, use it instead.
     (A counter cannot have been negative before the window started.)
  7. factor = (sampledInterval + durationToStart + durationToEnd) / sampledInterval
     If rate(): factor /= range_seconds
  8. return resultFloat × factor
```

Drawn over real samples, with a counter reset, this is the picture to keep:

```
   rate(http_requests_total[1m]) AT T — 15 s SCRAPE, ONE RESET IN WINDOW
   ═══════════════════════════════════════════════════════════════════════════

   value
    ▲
    │                                                          ● 240
    │                                              ● 180
    │                                  ● 120
    │  ● 990   ● 1050
    │                      ╳ ← process restarts; counter falls to 60
    │                      ● 60
    └──┬────────┬────────┬────────┬────────┬────────┬─────────┬──▶ time
    T−1m      s1       s2       s3       s4       s5         T
     ▲        :        :        :        :        :          ▲
     │        990     1050      60      120      180        240
     │                                                       │
   rangeStart (EXCLUSIVE, Prom ≥3.0)                    rangeEnd (INCLUSIVE)

   STEP 2   resultFloat = 240 − 990                        = −750
   STEP 3   s2→s3 drops 1050 → 60 ⇒ add prevPoint 1050     = +1050
            (no other drop)                                  ──────
                                                              300
            ← the TRUE increase: (1050−990) + (240−0) = 300  ✓ exact here,
              because the counter really did restart at 0 and the first
              post-reset sample was 60, i.e. 60 counted since restart.

   STEP 4   sampledInterval  = s5.t − s1.t = 60 s − 15 s     = 45 s? no:
            with 5 samples at 15 s spacing inside a 60 s window,
            first at T−45s and last at T:  sampledInterval   = 45 s
            durationToStart  = (T−45s) − (T−60s)             = 15 s
            durationToEnd    = T − T                         =  0 s
            avgInterval      = 45 / 4                        = 11.25 s
            threshold        = 11.25 × 1.1                   = 12.4 s

   STEP 5   durationToStart (15 s) ≥ threshold (12.4 s)
            ⇒ durationToStart = avgInterval / 2              = 5.6 s
            ── this is the "don't extrapolate all the way to the edge if
               the series looks like it started inside the window" rule.

   STEP 7   factor = (45 + 5.6 + 0) / 45                     = 1.125
            rate:  factor /= 60                              = 0.01875

   RESULT   300 × 0.01875                                    = 5.6 req/s
            increase() would give 300 × 1.125                = 337.5

   ── NOTE increase() returned 337.5, NOT 300. It is an ESTIMATE, always.
   ── NOTE the whole reset was recoverable only because a sample landed
      on BOTH sides of it. See §5 for what happens when one doesn't.
```

Three things fall out of that trace, and they are the answers to three separate common questions:

**Why `increase()` is fractional.** Step 7's factor is almost never exactly 1, because scrapes almost never land exactly on the window boundaries. You asked for a 60-second window; you got samples covering 45 seconds; Prometheus scaled up to estimate the missing 15. A genuine count of 300 reads as 337.5. **Never alert on an exact-equality threshold against `increase()`, and never present its output as an event count.** If you need exact counts, that is a log or an event stream, not a counter derivative.

**Why the "at least 4× the scrape interval" window rule holds.** Step 1 needs two samples. Step 4's `avgInterval` needs at least two. A window of exactly 2× the scrape interval gives you 2 samples with zero margin, so **one missed scrape empties the panel.** At 4× you have 4 samples and tolerate two consecutive misses. That is the derivation: the rule is not aesthetic, it is the failure-tolerance margin on step 1. The corollary is that the rule scales with *your* scrape interval — at a 60 s scrape (the Prometheus default `scrape_interval`), `rate(x[1m])` is broken by construction and `rate(x[4m])` is the floor.

**Why the zero clamp in step 6 exists.** Without it, a counter that started recently — say it is at 40 after 10 seconds of life — would extrapolate backwards to a negative value at the window start, producing an inflated rate. The clamp says: the counter was 0 at most `sampledInterval × first/result` ago, so do not extrapolate past that point. This is why a freshly-restarted target's first `rate()` values are conservative rather than enormous.

**Extended range selectors (experimental).** Prometheus 3.x ships `--enable-feature=promql-extended-range-selectors`, adding two modifiers that change exactly this behaviour: `anchored` uses the most recent sample within the lookback delta as the left boundary and applies **no extrapolation at all** (so `increase(x[5m] anchored)` returns integers); `smoothed` linearly interpolates values at both boundaries using the samples immediately outside them. `smoothed` needs a sample *after* the evaluation timestamp, so in rules it under-estimates unless the rule group carries a `query_offset` of at least one scrape interval. Both are experimental and unsupported for native histograms and subqueries — know they exist, do not build your SLO on them yet.

### 3. Trap 2 — the quantile of quantiles

**The claim:** `avg(p99)` across instances — or `max`, or `quantile()` over per-instance percentiles — is not an approximation of the fleet p99. It is not a number with a meaning.

**Why, with numbers.** Percentiles are not linear, so `p99(A ∪ B) ≠ f(p99(A), p99(B))` for any `f`. Construct the counterexample:

```
   Instance A: 10,000 requests, all fast.
       9,900 requests at 10 ms, 100 requests at 50 ms
       p99(A) = 50 ms

   Instance B: 100 requests, all slow (it is the sick one).
       99 requests at 3,000 ms, 1 request at 9,000 ms
       p99(B) = 9,000 ms

   avg(p99) = (50 + 9,000) / 2 = 4,525 ms          ← what the panel shows

   TRUE fleet p99: merge the populations.
       total = 10,100 requests
       rank of p99 = 0.99 × 10,100 = 9,999th request
       sorted:  9,900 at 10 ms  (ranks 1–9,900)
                  100 at 50 ms  (ranks 9,901–10,000)   ← 9,999th is HERE
                   99 at 3,000 ms
                    1 at 9,000 ms
       p99(fleet) = 50 ms                            ← the truth

   The panel reads 4,525 ms. The reality is 50 ms. 90× wrong,
   and wrong in the direction that pages you for nothing.
```

Now flip the traffic split and it is wrong in the *other* direction — a heavily-loaded sick instance averaged against many idle healthy ones will report a fleet p99 far below the truth, hiding a real regression. **`avg(p99)` is not conservatively wrong. It is arbitrarily wrong, and its sign depends on the traffic distribution**, which is exactly the thing that changes during an incident.

**The correct pattern**: aggregate the *bucket counters*, then take the quantile once over the merged distribution.

```promql
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)
```

Read it inside out. `rate(..._bucket[5m])` gives per-second observation rates for every `(instance, le)` pair. `sum by (le)` collapses instances while **preserving `le`**, producing one merged cumulative histogram. `histogram_quantile` then interpolates once. The rule to memorise: **aggregate buckets, then quantile — never quantile, then aggregate.**

**The variant that silently breaks it.** `sum without (le)` instead of `sum by (le)` removes `le`, leaving `histogram_quantile` with a single bucket per series and nothing to interpolate. You get `NaN` or nonsense. Similarly, if you need per-service fleet percentiles, you must keep both: `sum by (le, service)`.

**And the reason summaries cannot be fixed this way at all.** A Prometheus *summary* (`http_request_duration_seconds{quantile="0.99"}`) computes the quantile **client-side, per process**, and exports the result. There are no buckets to merge. Fleet-wide percentiles from summaries are impossible by construction — the information was destroyed in the client. That is the practical reason histograms beat summaries for anything running on more than one replica, and it is worth being able to say in a design review.

### 4. Trap 3 — `histogram_quantile` is only as good as its buckets

`histogram_quantile` does not know your distribution. It knows cumulative counts at a handful of boundaries and it interpolates. Here is `BucketQuantile` (`promql/quantile.go`) as an algorithm:

```
  1. Sort buckets by upper bound.
  2. If the highest bucket is not +Inf → return NaN. (No +Inf, no answer.)
  3. coalesceBuckets: merge buckets with identical upper bounds.
  4. ensureMonotonicAndIgnoreSmallDeltas: force cumulative counts to be
     non-decreasing by RAISING any bucket that dipped below its predecessor.
     Differences below 1e-12 are treated as float noise and silently equalised.
     Any real correction sets an annotation you will never see on a dashboard.
  5. observations = count of the +Inf bucket (i.e. total).
     If 0 → NaN.
  6. rank = q × observations
  7. b = the first bucket whose cumulative count ≥ rank
  8. THREE CASES:
     a) b is the +Inf bucket        → return the upper bound of the
                                       SECOND-TO-LAST bucket (the highest
                                       FINITE boundary). ← the clamp
     b) b is bucket 0 and its upper bound ≤ 0 → return that upper bound
     c) otherwise → LINEAR INTERPOLATION inside bucket b:
            bucketStart = upper bound of b−1   (0 if b is the first bucket)
            bucketEnd   = upper bound of b
            count       = b.count − (b−1).count
            rank       -= (b−1).count
            return bucketStart + (bucketEnd − bucketStart) × (rank / count)
```

Draw it, because the failure modes are geometric:

```
   histogram_quantile(0.99, …) OVER A CLASSIC HISTOGRAM
   ═══════════════════════════════════════════════════════════════════════════
   buckets (le):   0.005   0.01   0.025   0.05   0.1   0.25   0.5   1   2.5  +Inf
   cumulative:      1200   4100    8800   9300  9500   9700  9800  9900 9960 10000

   rank = 0.99 × 10000 = 9900

   cumulative
   10000 ┤                                              ╭───────────●  +Inf
    9960 ┤                                       ╭──────●  2.5
    9900 ┤ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─╭────●  1.0     ← rank lands EXACTLY
    9800 ┤                            ╭─────●  0.5             on the 1.0 bucket
    9700 ┤                     ╭──────●  0.25
    9500 ┤            ╭────────●  0.1
    9300 ┤       ╭────●  0.05
    8800 ┤   ╭───●  0.025
    4100 ┤ ╭─●  0.01
    1200 ┤●  0.005
         └────────────────────────────────────────────────────────▶  latency (s)

   CASE (c): b = the [0.5, 1.0] bucket.
      bucketStart = 0.5   bucketEnd = 1.0
      count in bucket = 9900 − 9800 = 100
      rank within bucket = 9900 − 9800 = 100
      result = 0.5 + (1.0 − 0.5) × (100/100) = 1.0 s

   ── THE UNIFORMITY ASSUMPTION ─────────────────────────────────────────────
   Those 100 observations could ALL be at 501 ms, or ALL at 999 ms.
   Prometheus assumes they are spread uniformly across [0.5, 1.0].
   Maximum error = the bucket width = 500 ms, on a p99 of 1 s. ±50 %.

   ── THE +Inf CLAMP (CASE a) ───────────────────────────────────────────────
   Now suppose traffic doubles at the tail and cumulative[2.5] drops to 9880:
       rank 9900 > 9880  ⇒  b = the +Inf bucket
       → returns the upper bound of the SECOND-TO-LAST bucket = 2.5 s
   The panel reads a rock-steady 2.5 s no matter how bad it gets. Requests
   taking 30 s and requests taking 3 s both render as 2.5 s. The signature
   is a latency line PINNED FLAT at exactly a bucket boundary. If you see
   that, your buckets are too coarse — the panel is at its ceiling.
```

**Three checkable diagnostics you can run today:**

1. *Is p99 clamped?* Compare `histogram_quantile(0.99, ...)` against your highest finite `le`. If it equals it exactly, and stays exactly equal, you are in case (a).
2. *How wide is the bucket p99 lives in?* That is your error bar. Quote it next to the number. A p99 of 1.0 s from a `[0.5, 1.0]` bucket is "somewhere in 500–1000 ms," and saying so in an incident is more useful than saying "1 second."
3. *Are the buckets monotonic?* Step 4 silently repairs non-monotonic cumulative counts, which occur when a scrape catches a target mid-update. It sets a `HistogramQuantileForcedMonotonicity` annotation. Annotations show in the Prometheus UI and the API response; **Grafana panels typically do not surface them**, so a systematically racy exporter can be quietly corrected forever.

**Bucket layout is a design decision, not a default.** The client libraries' default buckets (`.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10`) were chosen for sub-second web requests in 2015. They are wrong for an LLM inference service where time-to-first-token is 100 ms–2 s and end-to-end is 5–120 s, and wrong for a training step timer where the interesting range is 200 ms–5 s with a hard tail at 60 s on a straggler. **Place buckets so the SLO threshold is a boundary.** If your SLO is "p99 < 500 ms," having `le="0.5"` as an exact boundary lets you compute the SLI without any interpolation at all:

```promql
# Fraction of requests faster than 500 ms — NO interpolation, exact.
sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m]))
  /
sum(rate(http_request_duration_seconds_count[5m]))
```

**This is the query you should be using for SLO burn rate**, not `histogram_quantile`. Lesson 7 builds on it. Percentiles are for humans looking at graphs; bucket ratios are for alerts, because they are exact.

### 5. Native histograms — what they actually change

A classic histogram is `N+2` separate series (one per `le`, plus `_sum` and `_count`), with boundaries frozen at instrumentation time. A **native histogram** is *one* series whose sample is a whole sparse bucket structure, with boundaries defined by a schema rather than a config.

**The bucket layout.** Boundaries are powers of a base derived from the schema: bucket `i` spans `(base^i, base^(i+1)]` where `base = 2^(2^-schema)`. Prometheus's configuration documents the mapping from a desired growth factor to the resulting schema:

| Growth factor (bucket-to-bucket) | Schema (OTel "scale") | Relative bucket width |
|---:|---:|---|
| 65536 | −4 | absurdly coarse |
| 256 | −3 | |
| 16 | −2 | |
| 4 | −1 | |
| 2 | 0 | each bucket doubles |
| 1.4 | 1 | |
| 1.1 | 2 | |
| 1.09 | 3 | |
| 1.04 | 4 | |
| 1.02 | 5 | |
| 1.01 | 6 | |
| 1.005 | 7 | |
| 1.002 | 8 | ~0.2 % resolution |

Two knobs bound the cost: `native_histogram_bucket_limit` (default `0` = unlimited; if exceeded, Prometheus *reduces resolution* until it fits, and fails the scrape only if it cannot) and `native_histogram_min_bucket_factor` (default `0` = the finest supported factor, currently ~1.0027 / schema 8). Setting `min_bucket_factor: 1.1` caps you at schema 2 — 10 % relative error, which is plenty for latency SLOs and much cheaper than schema 8.

**The quantile is interpolated differently.** In `HistogramQuantile`, for exponential buckets the interpolation is **logarithmic**, not linear — the code comment says it explicitly: "For exponential buckets, we interpolate on a logarithmic scale." That is correct for multiplicatively-spaced boundaries and it is why native-histogram quantiles are better even at similar bucket counts. For NHCBs (native histograms with custom buckets — a classic layout stored in the native format) and for the zero bucket, interpolation stays linear, and the `+Inf` case returns `bucket.Lower` rather than the second-to-last upper bound.

**The migration, as it actually is in Prometheus 3.x.** Three scrape-config switches, all defaulting to false:

```yaml
global:
  # Recognise and ingest native histograms exposed by targets.
  scrape_native_histograms: false          # default
  # Convert scraped CLASSIC histograms into native histograms with custom buckets.
  convert_classic_histograms_to_nhcb: false # default
  # Also keep the classic series when one of the above would replace it.
  always_scrape_classic_histograms: false   # default
  # Bound the cost.
  native_histogram_bucket_limit: 160
  native_histogram_min_bucket_factor: 1.1   # → schema 2, ~10 % resolution
```

The combination that de-risks a migration is `scrape_native_histograms: true` **plus** `always_scrape_classic_histograms: true`: you get both representations for the same metric, you can run the old and new queries side by side, compare, and only then drop the classic series. It doubles the cost during the overlap — which is a real, finite, plannable cost, unlike a flag-day cutover.

**What native histograms do *not* fix.** They do not make `avg(p99)` valid (that is Trap 2, unchanged). They do not make interpolation exact (it is still interpolation, just on a better grid). They do not eliminate the `+Inf` case. And the exact-boundary trick from §4 gets *harder*, not easier, because you no longer choose the boundaries — for an SLO threshold at exactly 500 ms you may prefer to keep one classic bucket alongside.

### 6. Trap 4 — window sizing, both directions

Derived in §2: **the floor is 4× the scrape interval.** The ceiling is set by what you need to detect.

`rate()` over a window is a low-pass filter with a time constant of roughly the window length. A 90-second latency excursion inside a 1-hour rate window is diluted by a factor of 40 and disappears into the noise floor. Concretely:

```
   A 90-SECOND ERROR SPIKE, SEEN THROUGH DIFFERENT rate() WINDOWS
   ═══════════════════════════════════════════════════════════════════════════
   Baseline: 1,000 req/s, 1 error/s (0.1 % error rate).
   Spike:    90 seconds at 500 errors/s.  Extra errors = 90 × 499 ≈ 44,910.

   window   errors counted in window   error rate reported at peak
   ──────   ────────────────────────   ───────────────────────────
   [1m]     ~30,000 in 60 s            ~500/s   →  50 %   ← unmissable
   [5m]     44,910 + 210 baseline      ~150/s   →  15 %   ← obvious
   [30m]    44,910 + 1,710             ~26/s    →  2.6 %  ← visible
   [1h]     44,910 + 3,510             ~13/s    →  1.3 %  ← plausible-looking
   [6h]     44,910 + 21,510            ~3/s     →  0.3 %  ← indistinguishable
                                                            from noise

   The spike did not get smaller. Your filter got wider.
```

This is precisely why lesson 7's burn-rate alerting uses **short and long windows together** rather than picking one: the short window catches the spike, the long window rejects the noise, and the AND of the two gets both.

**The `$__rate_interval` variable.** Grafana's `$__interval` is the display step, which at a wide zoom can be smaller than your scrape interval — producing `rate(x[10s])` on a 30 s scrape, which returns nothing. `$__rate_interval` exists to fix exactly this: it is defined as `max($__interval + scrape_interval, 4 × scrape_interval)`, which enforces the floor derived in §2. **Use `$__rate_interval` in every Grafana panel; use a literal duration in every recording and alerting rule**, because rules have no display step and you want the window pinned.

### 7. Trap 5 — invisible counter resets

§2 showed reset correction working. Here is when it silently fails.

Counter-reset detection is **sample-based**. Prometheus sees only the values at scrape times. If a counter resets and then climbs back *above* its pre-reset value before the next scrape, no sample ever records a decrease, and no correction is applied:

```
   INVISIBLE RESET — 15 s SCRAPE, PROCESS RESTARTS AT t=18s
   ═══════════════════════════════════════════════════════════════════════════

   true counter
    ▲
    │              ╱                       ← post-restart climb, VERY fast
    │  ╱          ╱                          (e.g. a batch job replaying)
    │ ╱  ╳ reset ╱
    │╱           ╱
    └──●─────────────●───────────────────▶  t
      s1 (t=15s)    s2 (t=30s)
      value 900     value 950

   Prometheus sees:  900 → 950.  Monotonic. No reset detected.
   increase() reports 50.  The truth is 900 (lost) + 950 = 1,850.
   Undercount: 97 %.

   The narrower the scrape interval relative to the counter's climb rate,
   the more likely you CATCH the reset. This is one of the few places where
   scraping faster genuinely buys correctness, not just resolution.
```

**Prometheus 3.x mitigates this with created/start timestamps.** When a target exposes a `_created` timestamp (OpenMetrics) — or when the `created-timestamp-zero-ingestion` / start-timestamp features synthesise one — `extrapolatedRate` uses `isStartTimestampReset(...)`: a change in the *start timestamp* between two samples proves a reset happened even when the values did not decrease. That closes the hole, but only for targets that actually expose the metadata. Check `/api/v1/status/tsdb` and your exporter before assuming you have it.

**Also invisible: a wrapped counter.** A 64-bit float counter does not realistically wrap, but a counter sourced from a 32-bit hardware register (some NIC and NVLink counters) does. `DCGM_FI_PROF_NVLINK_TX_BYTES` and friends are cumulative and can wrap on long-lived nodes; the wrap looks like a reset to Prometheus, which corrects it *as if it were a reset to zero* — adding back the pre-wrap value rather than the wrap modulus. The correction is wrong by exactly `2^32 − pre_wrap_value`. If you alert on absolute NVLink byte counts, bound the alert or use a rate over a short window where the error is diluted.

### 8. Trap 6 — `rate()` vs `irate()` are different questions

`irate()` (`instantValue` in `functions.go`) does **not** use the window's samples. It takes only the **last two** samples in the window and returns `(v2 − v1) / (t2 − t1)`, with reset handling that returns `v2` itself if `v2 < v1`. The range is only there to bound how far back it will look for the second sample.

```
   SAME DATA, TWO FUNCTIONS — 15 s SCRAPE, 5 m WINDOW
   ═══════════════════════════════════════════════════════════════════════════
   requests_total:  … 40200, 40260, 40320, 40380, 40381, 40800

   rate(x[5m])   uses first & last across the whole window, extrapolates,
                 divides by 300 s      → a smooth ~4 req/s trend line

   irate(x[5m])  uses ONLY the last two:  (40800 − 40381) / 15 s
                                        → 27.9 req/s

   One scrape of jitter — the app stalled 15 s then flushed — moves irate()
   by 7×. rate() barely notices. Neither is "more accurate": they answer
   "what has the recent trend been" and "what happened between the last two
   scrapes" respectively.
```

The operational rule: **`irate()` for a human staring at a zoomed-in dashboard; `rate()` for anything a machine acts on.** An alert built on `irate()` pages on single-scrape noise. And because `irate()` ignores everything but the last two samples, it also *misses* spikes that happened earlier in the window — it is not a strictly higher-resolution `rate()`, it is a different measurement.

`idelta()` is the gauge equivalent, with the same caveat.

### 9. Staleness, absence, and the phantom flatline

**The mechanism.** Prometheus writes an explicit **stale marker** — a special NaN payload, `StaleNaN = 0x7ff0000000000002` (`model/value/value.go`) — into a series when a scrape that previously produced it no longer does, or when the target disappears from service discovery. From `docs/querying/basics.md`: if a query evaluates at a timestamp *after* the stale marker, **no value is returned for that series.** Separately, even without a marker, the 5-minute lookback delta means a series with no sample in the last 5 minutes is absent from instant queries.

```
   PREEMPTION ON A GPU FLEET — WHAT THE SERIES DOES
   ═══════════════════════════════════════════════════════════════════════════
   t=0     node gpu-0417 healthy, DCGM_FI_PROF_PIPE_TENSOR_ACTIVE = 0.71
   ...
   t=300s  scheduler preempts the node. kubelet gone. Endpoint removed from SD.

   ┌─ WHAT PROMETHEUS DOES ────────────────────────────────────────────────┐
   │ t=300s   scrape fails / target removed                                │
   │ t≈305s   Prometheus appends StaleNaN to every series from that target │
   │ t>305s   instant queries return NOTHING for those series              │
   │          range queries show the line ENDING at t=300s                 │
   └───────────────────────────────────────────────────────────────────────┘

   ┌─ WHAT AN UNPREPARED ALERT DOES ───────────────────────────────────────┐
   │  alert: GPUIdle                                                       │
   │  expr:  DCGM_FI_PROF_PIPE_TENSOR_ACTIVE < 0.05                        │
   │  for:   30m                                                           │
   │                                                                       │
   │  t>305s  the series is ABSENT, so the expression matches NOTHING,     │
   │          so the alert RESOLVES. A node that vanished mid-training     │
   │          reads as "problem fixed."                     ← silent hole  │
   └───────────────────────────────────────────────────────────────────────┘

   ┌─ WHAT AN EXPORTER WITH ITS OWN TIMESTAMPS DOES (the nastier case) ────┐
   │  Some exporters attach their own timestamps to samples. Those series  │
   │  do NOT get a stale marker on target removal; they simply age out     │
   │  after the 5 m lookback — carrying their LAST VALUE for 5 minutes.    │
   │  That is the phantom flatline: 5 minutes of fake 0.71 tensor activity │
   │  from a node that is already gone.                                    │
   │  Controlled by the `track_timestamps_staleness` scrape setting.       │
   └───────────────────────────────────────────────────────────────────────┘
```

**Staleness is not an alert.** It is an ingestion behaviour. Nothing fires because a series went stale. If you need to know a target vanished, you must ask:

```promql
# 1. The target-level check — up{} exists for every SD-discovered target
#    and goes to 0 on scrape failure. But a REMOVED target has no up{} either.
up{job="dcgm-exporter"} == 0

# 2. The "this should exist and doesn't" check. absent() returns 1 (with the
#    labels you wrote into the matcher) exactly when the selector matches nothing.
absent(DCGM_FI_PROF_PIPE_TENSOR_ACTIVE{Hostname="gpu-node-0417"})

# 3. The fleet-scale version: compare observed node count to expected.
#    absent() per node needs one rule per node — unusable at 4,000 nodes.
count(count by (Hostname) (DCGM_FI_DEV_SM_ACTIVE))
  < 0.98 * count(count by (Hostname) (kube_node_info{}))

# 4. The time-window version, when a series may be legitimately intermittent.
absent_over_time(DCGM_FI_DEV_SM_ACTIVE{Hostname="gpu-node-0417"}[15m])
```

Note the practical limitation in (2): `absent()` only knows the labels you literally typed into the matcher, because there is no series to read them from. At fleet scale you use pattern (3) — a count comparison against an inventory metric — or a dedicated up-ness exporter. Lesson 7 wires this into a "coverage" SLI.

**The other half of the trap** is the alert that fires because a series went stale when it should not have. If a node is *supposed* to be reclaimed, `absent()` firing is noise. The distinction has to come from a second signal — a `kube_node_spec_unschedulable`, a maintenance-window label, or an inhibition rule in Alertmanager. This is why "preemption" is a first-class concept on a GPU fleet's alerting model rather than an edge case.

### 10. Recording rules — the scaling primitive, and what it cannot fix

**What they do.** A recording rule evaluates an expression on a schedule and writes the result as a new series. Rules live in groups; **rules within a group run sequentially at the same evaluation timestamp**, which is what lets a later rule depend on an earlier one's output within the same tick. Groups run independently and concurrently.

```yaml
groups:
  - name: gpu-fleet-rollups
    interval: 30s                 # default: global.evaluation_interval (1m)
    limit: 5000                   # max series this group may produce; 0 = unlimited
    query_offset: 30s             # evaluate at (now − 30s), so late samples land
    labels:
      rollup_source: dcgm         # added to every output series in the group
    rules:
      # Level 1 — per node, from raw per-GPU series.
      - record: node:dcgm_tensor_active:avg
        expr: avg by (Hostname) (DCGM_FI_PROF_PIPE_TENSOR_ACTIVE)

      # Level 2 — fleet, from the level-1 output. Same tick, so this sees it.
      - record: fleet:dcgm_tensor_active:avg
        expr: avg (node:dcgm_tensor_active:avg)

      # Bucket-preserving latency rollup — the ONLY correct fleet-percentile input.
      - record: service:request_duration_seconds_bucket:rate5m
        expr: sum by (service, le) (rate(request_duration_seconds_bucket[5m]))
```

**The naming convention** (`level:metric:operations`) is not decoration. `node:dcgm_tensor_active:avg` tells you the aggregation level, the source metric, and what was done to it. When someone finds this series in an autocomplete two years from now, the name is the documentation. Get it wrong and you get `gpu_util_avg_new_v2_fixed`.

**`query_offset` is the fix for a real problem.** Rules evaluate at `now`, but a sample scraped a moment ago may not have been committed yet, and remote-write / OTLP paths can deliver late. A rule with no offset can compute a rate over a window whose last sample is missing, producing a dip once per evaluation. Setting `query_offset` to about one scrape interval evaluates slightly in the past, where the data is settled. This is also **mandatory** if you use the experimental `smoothed` range modifier (§2).

**What recording rules cost.** The rule still has to *execute* the expression. A rule that pre-aggregates 32,000 raw series into 200 rollups reads 32,000 series every interval — you have collapsed the storage cost, not the query cost. Watch:

- `prometheus_rule_evaluation_duration_seconds` — per-group evaluation latency.
- `prometheus_rule_group_iterations_missed_total` — **the alarm bell.** A group whose evaluation takes longer than its interval starts skipping iterations. Your recording rules silently stop producing points; your alerts built on them silently stop firing.
- `prometheus_rule_group_last_duration_seconds` vs `prometheus_rule_group_interval_seconds` — the ratio is your headroom.

**What they cannot fix.** A recording rule caches whatever expression you gave it. If that expression is `avg(histogram_quantile(...))`, the rule makes the wrong answer cheaper and gives it an official-looking metric name that will be copied into six dashboards and two alerts. **A recording rule is a place a trap hides more durably, not a place it gets fixed.** `promtool check rules` validates syntax, not semantics. The review question is always "what is the expression?", never "is it fast now?"

### 11. Three more traps worth naming

**Vector-matching cardinality errors.** `a / b` with `on()`/`ignoring()` requires a one-to-one match by default. `group_left`/`group_right` allow many-to-one, and specify which side has the "many". The classic failure is silence: if the label sets don't match, the result is **empty**, not an error, so a panel goes blank and someone assumes the service is down. Debug by evaluating each side separately and diffing their label sets. The classic *wrong* result is a `group_left` on a right-hand side that isn't unique — Prometheus errors with "multiple matches for labels" at query time, which at least is loud.

**`rate()` inside vs outside an aggregation.** `sum(rate(x[5m]))` is correct. `rate(sum(x)[5m:])` is a subquery over an aggregate, which loses per-series reset detection: if one instance restarts, the summed series dips, and the outer `rate()` treats the dip as a single fleet-wide reset, adding back the *whole sum*. **Always rate first, aggregate second.** The mnemonic: reset correction is a per-series property and must happen while series are still separate.

**`offset` vs `@`.** `x offset 1w` means "one week before the evaluation timestamp" and slides with the query. `x @ 1755504312` pins to an absolute Unix timestamp and does not slide. Week-over-week comparisons want `offset`; a "compare against the value at the start of this incident" panel wants `@`. `@ end()` and `@ start()` pin to the range query's own bounds, which is how you build "growth since the left edge of this graph" panels.

### 12. GPU-fleet tie

Mapping every trap onto the fleet signals, since this is what lesson 9 will assume:

| Signal | Type | The trap it invites | The correct query |
|---|---|---|---|
| `DCGM_FI_DEV_GPU_UTIL` | gauge, 0–100 | `rate()` (Trap 1) | `avg_over_time(...[5m])`, and read it as *presence*, not intensity |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | gauge, 0.0–1.0 | comparing to GPU_UTIL directly (100× unit mismatch) | keep the units straight; `x * 100` if you must overlay |
| `DCGM_FI_DEV_FB_USED` | gauge, bytes | `rate()`; also `sum` across MIG instances double-counting the parent | `max_over_time`, and filter MIG parents |
| `DCGM_FI_PROF_NVLINK_TX_BYTES` | counter, possibly 32-bit-sourced | invisible wrap (Trap 5) | `rate()` over a short window; never absolute totals |
| `DCGM_FI_DEV_XID_ERRORS` | gauge carrying the last XID code | `rate()`, `sum()` — the *value* is a code, not a count | `changes(...[1h])` to count events; join to a code table for meaning |
| per-node step-time histogram | histogram | `avg(p99)` (Trap 2) | `histogram_quantile(0.99, sum by (le) (rate(...[5m])))` |
| any node metric | any | phantom flatline on preemption (§9) | count-vs-inventory coverage check |

Two GPU-specific notes that generalise:

**`DCGM_FI_DEV_GPU_UTIL` is a presence metric.** It reports the fraction of a short driver sample window during which at least one kernel was resident — not how much of the silicon was working. A batch-1 decode server pins it at 100 while the tensor pipes idle. That is derived from NVML's counter semantics in the `05-gpu-observability` module; here the consequence is narrower and just as important: **no PromQL fixes a metric that measures the wrong thing.** Getting `rate()` off it is necessary and insufficient.

**XID errors are the type-confusion case.** `DCGM_FI_DEV_XID_ERRORS` is a gauge whose *value is an enum* — the last XID error code observed. `sum()` over it produces the sum of error codes, a number with no meaning that will nonetheless trend upward as more distinct errors occur, looking exactly like a rising error rate. Use `changes()` to count transitions, or `count by (...)` over a filtered selector. Any metric whose value is an identifier rather than a measurement has this problem; DCGM is just where you will meet it.

## Perspectives

**Query-author (developer).** These traps are the *default* output of naive construction. Grafana's query builder will suggest `rate()` on anything numeric, gauge or counter alike, with no warning, because it does not know the type either — the exposition format carries a `# TYPE` line but Prometheus does not enforce it at query time. The discipline is a two-second check before typing `rate(`: what type is this series, and what question am I asking about it? That check is cheap and it catches Traps 1, 5, 6 and the XID case outright.

**Dashboard-consumer (incident responder).** Nobody re-derives PromQL during an incident; they trust the panel and act. So the staff responsibility is entirely *before* the incident: audit every panel, name the trap if any, fix it, and put the corrected expression in a recording rule with a documented name so it cannot drift back. A pre-incident PromQL audit is a deliverable with a date on it, not a good intention.

**Storage-engine.** The interpolation error in Trap 3 is not an implementation quirk; it is the direct consequence of what a classic histogram stores. Cumulative counts at fixed boundaries and nothing else. There is no information to interpolate *from* except an assumption. Native histograms fix it structurally — a schema-defined exponential grid, logarithmic interpolation, one series instead of N+2 — which is why the feature took years and touched the wire format, the chunk encoding, the query engine and the remote-write protocol. The scale of the change is the measure of how fundamental the limitation was.

**Cost/performance.** A trap and a performance problem are usually the same edit. `avg(histogram_quantile(...))` is both wrong and expensive: it computes a quantile per series before aggregating, so it does the interpolation N times instead of once. `rate()` over raw high-cardinality series on every dashboard load is both a correctness risk (window sizing drifts with `$__interval`) and a sustained query load. Fixing the semantics and moving it into a recording rule is one change that pays twice.

**Reliability.** The scariest item in this lesson is not a wrong number, it is §10's `prometheus_rule_group_iterations_missed_total`. When rule evaluation falls behind its interval, recording rules stop producing points and alerts stop evaluating — silently, with no alert, because the thing that would alert you is the thing that stopped. Monitor your monitoring's rule-group headroom as a first-class SLI; it is the single highest-leverage meta-alert in a Prometheus deployment.

## Real-world use cases

- **PromLabs (Julius Volz, an original Prometheus author), "Avoid These 6 Mistakes When Getting Started With Prometheus."** The list includes rate-window sizing, counter/gauge confusion, and aggregation-order errors — the same set this lesson dissects. **What it shows:** these are not exotic edge cases discovered once. They are common enough that a core author wrote a canonical list, which tells you the failure is in the interface design (functions that accept anything and return something), not in the users.

- **Grafana Labs, "Prometheus native histograms in Grafana Cloud."** Native histograms reaching general availability in a major managed Prometheus offering, after years as an experimental core feature. **What it shows:** the bucket-boundary interpolation error was significant enough industry-wide to justify changing the exposition format, the chunk encoding, the query engine and the remote-write protocol. Read the size of the fix as a measurement of the size of the problem.

- **Robusta.dev, "3 Common Mistakes with PromQL and Kubernetes Metrics."** CPU-throttling gauges `rate()`'d as if they were counters, container-restart counters misread, and `container_cpu_usage_seconds_total` aggregated before rating. **What it shows:** the exact same shapes appear in the Kubernetes metric set that everyone already runs, which is the best possible practice ground — and a DaemonSet-deployed DCGM exporter reproduces the same patterns at GPU-fleet scale.

- **The `rate(sum(...))` reset-storm.** A widely-reproduced pattern rather than one named incident: a fleet dashboard sums a counter across replicas and then rates the sum. On any rolling deploy, replicas restart at staggered times; each restart makes the summed series dip; the outer `rate()` interprets each dip as a fleet-wide reset and adds back the entire aggregate. **What it shows:** the panel spikes *during every deploy*, teams learn to ignore it, and then it is ignored during the one deploy that genuinely broke something. Traps do not only mislead — they train people to distrust the signal.

## Worked example

**A broken GPU-fleet dashboard, diagnosed panel by panel and rewritten.** This is the checkpoint exercise; work it end to end.

---

**Panel A — "GPU utilisation rate"**

```promql
rate(DCGM_FI_DEV_GPU_UTIL[5m])
```

*Diagnosis:* Trap 1. `DCGM_FI_DEV_GPU_UTIL` is a gauge in units of percent. `rate()` applies the counter-reset branch, so every legitimate dip in utilisation is added back as if the counter had restarted. With the sample sequence from §1 the panel reads 1.55 for a GPU whose utilisation was flat. The number correlates with activity, which is exactly why nobody catches it.

*Fix:* decide the question first.

```promql
# "How busy is each GPU, smoothed?"  (the question 95 % of people mean)
avg_over_time(DCGM_FI_DEV_GPU_UTIL[5m])

# "Is utilisation trending up or down?"  (least-squares slope, %/s)
deriv(DCGM_FI_DEV_GPU_UTIL[30m])

# "How much of the last hour was this GPU above 80 %?"  (seconds)
sum_over_time((DCGM_FI_DEV_GPU_UTIL > bool 80)[1h:1m]) * 60
```

*And the deeper fix:* rename the panel. It should not say "utilisation" at all, because the field measures kernel *presence*. Call it "GPU busy (presence)" and put `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` next to it.

---

**Panel B — "p99 inference latency, fleet"**

```promql
avg(histogram_quantile(0.99, rate(inference_duration_seconds_bucket[5m])))
```

*Diagnosis:* two stacked bugs. (1) `histogram_quantile` is applied per-series and then averaged — the quantile-of-quantiles of Trap 2, arbitrarily wrong in a direction that depends on the traffic split. (2) There is no `by (le)` anywhere, so the inner `rate()` produces one series per `(pod, le)` and `histogram_quantile` sees a fragmented bucket set per pod, which then gets averaged. During a canary rollout where one pod is slow and lightly loaded, this panel reads high and pages; during a rollout where one pod is slow and heavily loaded, it reads low and hides the regression.

*Fix:*

```promql
histogram_quantile(0.99,
  sum by (le) (rate(inference_duration_seconds_bucket[5m]))
)
```

*And verify the buckets.* Check whether p99 is clamped:

```promql
# If these two are equal and STAY equal, your p99 is pinned at the ceiling.
histogram_quantile(0.99, sum by (le) (rate(inference_duration_seconds_bucket[5m])))
# vs the highest finite le in the metric:
max(inference_duration_seconds_bucket_le)   # or read it off /api/v1/series
```

For an inference service, the classic default buckets top out at 10 s, which is *inside* the normal range for a long-context request. Re-instrument with boundaries that bracket both the SLO threshold and the tail — e.g. `0.1, 0.25, 0.5, 1, 2, 5, 10, 20, 30, 60, 120` — and put the SLO threshold on an exact boundary.

---

**Panel C — "requests in the last hour"**

```promql
increase(http_requests_total[1h]) > 1000000
```

*Diagnosis:* Trap 5. `increase()` extrapolates to the window edges, so the value is an estimate; on a 1-hour window with a 15 s scrape the factor is close to 1, but it is not 1, and the comparison is exact. Worse, the panel is presented as a count. Someone will reconcile it against a billing system and find a discrepancy.

*Fix:* if it feeds an alert, express it as a rate with margin. If it must be a count, get it from a source that counts.

```promql
# As an alert input — rate with a threshold that has margin:
sum(rate(http_requests_total[1h])) > 278       # 1e6/3600 ≈ 277.8 req/s

# Presented to a human — label the estimate as one:
sum(increase(http_requests_total[1h]))         # panel title: "≈ requests (est.)"
```

---

**Panel D — "GPU idle alert"**

```yaml
- alert: GPUIdle
  expr: DCGM_FI_PROF_PIPE_TENSOR_ACTIVE < 0.05
  for: 30m
```

*Diagnosis:* §9. When a node is preempted the series goes stale, the expression matches nothing, and the alert **resolves**. A GPU that vanished mid-training reports as healthy. Conversely, if the exporter attaches its own timestamps, the series carries its last value for the 5-minute lookback window, producing a phantom flatline.

*Fix:* split "idle" from "gone", and derive "gone" from an inventory comparison rather than per-node `absent()`.

```yaml
- alert: GPUAllocatedButIdle
  expr: |
    DCGM_FI_PROF_PIPE_TENSOR_ACTIVE < 0.05
      and on (Hostname, gpu)
    dcgm_gpu_allocated == 1
  for: 30m
  labels: { severity: ticket }
  annotations:
    summary: 'GPU {{ $labels.Hostname }}/{{ $labels.gpu }} allocated but idle 30m'

- alert: GPUFleetCoverageLoss
  expr: |
    count(count by (Hostname) (DCGM_FI_DEV_SM_ACTIVE))
      < 0.98 * count(kube_node_info{node_role="gpu"})
  for: 10m
  labels: { severity: page }
  annotations:
    summary: '{{ $value }} GPU nodes reporting; expected ≥98 % of inventory'
```

The `and on (Hostname, gpu) dcgm_gpu_allocated == 1` clause is doing real work: it converts "idle" into "idle *and someone is paying for it*", which is the only version worth waking a human for.

---

**Panel E — the per-tenant rollup that is slow**

```promql
sum by (namespace) (
  DCGM_FI_PROF_PIPE_TENSOR_ACTIVE
    * on (Hostname, gpu) group_left (namespace) dcgm_gpu_info
)
```

*Diagnosis:* semantically correct (this is the info-metric join from lesson 1), but it fans out over 32,000 series with a hash join on every dashboard refresh. Six panels at 10 s refresh is a sustained load that will eat into rule-evaluation headroom.

*Fix:* move it to a recording rule and read the rollup.

```yaml
groups:
  - name: gpu-tenant-rollups
    interval: 30s
    query_offset: 30s
    rules:
      - record: namespace:dcgm_tensor_active:sum
        expr: |
          sum by (namespace) (
            DCGM_FI_PROF_PIPE_TENSOR_ACTIVE
              * on (Hostname, gpu) group_left (namespace) dcgm_gpu_info
          )
      - record: namespace:dcgm_gpu_count:sum
        expr: sum by (namespace) (dcgm_gpu_info)
      - record: namespace:dcgm_tensor_active:avg
        expr: |
          namespace:dcgm_tensor_active:sum / namespace:dcgm_gpu_count:sum
```

Output: ~200 series per rule instead of 32,000 read per panel load. Note the third rule depends on the first two and is in the same group, so it sees them at the same evaluation timestamp — that ordering guarantee is why they must share a group.

*Then verify the rule is not itself the problem:*

```promql
# Headroom. Anything above ~0.7 sustained needs a longer interval or a split group.
prometheus_rule_group_last_duration_seconds
  / prometheus_rule_group_interval_seconds

# The alarm bell.
increase(prometheus_rule_group_iterations_missed_total[1h]) > 0
```

---

**Panel F — the one that was already right**

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))
```

*Diagnosis:* correct. `rate()` is applied per-series *before* aggregation, so per-instance counter resets are handled individually; the two `sum`s then divide cleanly because both sides have no remaining labels. This is the shape lesson 7's burn-rate alerts are built on.

*The only thing to check* is the window against your scrape interval (§6) and whether `status` is the right label name in your instrumentation. Add it to the audit as "verified", with a date — an audit that only lists broken panels does not tell the next person what was checked.

## Practice

Feeds the [fleet observability design](../practice/fleet-observability/README.md).

Assemble a **corrected dashboard panel set** and a written audit. Take a starter dashboard containing at least one instance of each trap — a `rate()` on a DCGM gauge, an `avg(p99)`, a rate window below 4× scrape, an `irate()` wired into an alert, a `histogram_quantile` clamped at `+Inf`, an alert that resolves on preemption, and a `rate(sum(...))` — and for each panel produce:

1. **A one-line diagnosis** naming the trap and stating *how the rendered number is wrong* — sign, magnitude, meaning, or precision. "It's wrong" is not a diagnosis; "reset correction fires on every gauge decrease, so the output is a function of volatility rather than of level" is.
2. **The corrected PromQL**, plus the question it now answers. If the corrected query answers a *different* question than the panel title claims, rename the panel.
3. **For the latency panel:** state the current highest finite bucket, demonstrate whether p99 is clamped, propose a bucket layout with the SLO threshold on an exact boundary, and write the exact-ratio SLI query that needs no interpolation. If you propose native histograms, name the three scrape-config switches and the rollout plan (dual-emit period, bucket limit, min bucket factor), not just "switch to native histograms."
4. **For the preemption alert:** replace it with the two-alert split — an "allocated but idle" alert gated on an allocation signal, and a fleet-coverage alert derived from an inventory comparison. State why per-node `absent()` does not scale to 4,000 nodes.
5. **The recording rules** that make the high-fan-in per-tenant panels cheap. Give the output cardinality of each, confirm the expression itself is already correct, put dependent rules in one group and say why, and set `query_offset` with a justification.
6. **The meta-check:** the two queries that tell you whether your rule groups have headroom, with the thresholds you would alert at.
7. **A "verified correct" list.** Every panel you checked and found sound, with the reason. This is what makes the audit re-runnable.

**Acceptance criteria.** The audit is done when (i) every diagnosis names the *mechanism*, not just the symptom; (ii) every corrected query is one you have actually evaluated against real or synthetic data, with the before/after values recorded; (iii) each recording rule's output cardinality is stated and lands inside the budget from lesson 1; and (iv) a peer could re-run the audit on a different dashboard using only your write-up.

Carry the corrected latency SLI query forward — lesson 7 turns the exact-boundary bucket ratio into a burn-rate alert, and lesson 9 applies the whole trap list to the DCGM and NCCL signal set.

## Common pitfalls

- **"`rate()` smooths, so it must be doing regression."** *Symptom:* people reason about `rate()` as noise-robust and are surprised when a single bad sample at a window edge moves it. *Mechanism:* `rate()` is `(last − first + reset corrections) × factor / range`. It uses exactly two sample *values* plus the reset scan; the smoothing is entirely from dividing by a long window, not from fitting. `deriv()` is the least-squares function. If you want regression, ask for it.

- **"`rate()` and `irate()` are interchangeable, `irate` is just higher resolution."** *Symptom:* an alert that fires on single-scrape jitter. *Mechanism:* `irate()` reads only the last two samples in the window; everything earlier is discarded. It is not higher-resolution `rate()`, it is a different statistic — and it can *miss* a spike that occurred earlier in the same window.

- **"A recording rule fixes a wrong query."** *Symptom:* "I made the latency panel fast." *Mechanism:* the rule caches whatever expression you gave it and blesses it with an official metric name that spreads. `promtool check rules` validates syntax only. The review question is "what is the expression?"

- **"`increase()` gives an exact count."** *Symptom:* an alert on `increase(x[1h]) == 1000`, or a count reconciled against billing. *Mechanism:* boundary extrapolation (§2 step 7) scales the observed change by `(sampledInterval + durationToStart + durationToEnd) / sampledInterval`, which is almost never exactly 1. Alert on rates with margin; get exact counts from an event stream.

- **"Native histograms are a drop-in replacement."** *Symptom:* a flag flip that changes every latency number on every dashboard overnight. *Mechanism:* they require target-side emission, `scrape_native_histograms: true`, a bucket-limit and min-factor policy, and a query rewrite (`histogram_quantile` over the native series takes no `sum by (le)`). Run `always_scrape_classic_histograms: true` through an overlap period and compare, then cut over.

- **"Staleness and `absent()` are the same mechanism."** *Symptom:* an alert that resolves when the node disappears. *Mechanism:* staleness is an ingestion behaviour — Prometheus writes a `StaleNaN` marker and the series stops being returned. Nothing fires. `absent()` is a query-time function you must wire in explicitly, and it only knows the labels you typed into it, so it does not scale per-node. Use an inventory count comparison at fleet scale.

- **"`sum` then `rate` is fine, it's the same arithmetic."** *Symptom:* the fleet request-rate panel spikes on every rolling deploy. *Mechanism:* reset correction is per-series and must run while series are separate. Summing first merges the resets into dips in one aggregate series; the outer `rate()` then adds back the *entire aggregate value* at each dip. Rate first, aggregate second — always.

- **"An empty panel means the service is down."** *Symptom:* a false page during a scrape hiccup. *Mechanism:* a rate window shorter than ~4× the scrape interval returns nothing when a single scrape is missed, and an instant selector returns nothing when the newest sample is older than the 5-minute lookback. Both render identically to "no traffic." Distinguish with `up{}`, `absent_over_time()`, or a coverage count.

- **"The value of this metric is a number, so I can `sum` it."** *Symptom:* an XID-error panel that trends smoothly upward. *Mechanism:* `DCGM_FI_DEV_XID_ERRORS` carries an error *code* as its value. Summing codes yields a number that rises as more distinct error types appear and looks exactly like a rising error count. Any metric whose value is an identifier needs `changes()` or `count`, never arithmetic.

## Self-check

**Why does `rate(gauge[5m])` return a wrong number rather than just a noisy one? Walk the code path.**
`extrapolatedRate` computes `last − first`, then — because `isCounter` is true for `rate` — scans adjacent pairs and, for every pair where the value *decreased*, adds the pre-decrease value back to the result. On a monotonic counter that correction reconstructs progress lost to a restart. On a gauge, every legitimate downward movement triggers it, so the result becomes roughly the sum of all the gauge's peaks — a function of volatility, not of level or slope, and always positive. It is then scaled by the extrapolation factor and divided by the window. The output is wrong in magnitude and meaningless in sign of trend. For a gauge use `avg_over_time` (level), `deriv` (least-squares slope), `delta` (change over window, still extrapolated), or `predict_linear` (threshold crossing).

**Give the correct fleet-wide p99 query and explain, with a concrete counterexample, why `avg(p99)` is invalid.**
`histogram_quantile(0.99, sum by (le) (rate(request_duration_seconds_bucket[5m])))` — rate each bucket counter per series, sum across instances while preserving `le`, then interpolate once over the merged cumulative distribution. `avg(p99)` is invalid because percentiles are not linear: with instance A at 10,000 requests (p99 = 50 ms) and instance B at 100 requests (p99 = 9,000 ms), `avg(p99)` = 4,525 ms while the true merged p99 is 50 ms, because the 9,999th-ranked request of 10,100 still falls in A's 50 ms bucket. Flip the traffic split and the error flips sign. Note also that summaries cannot be fixed this way at all — the quantile was computed client-side and the distribution is gone.

**Your p99 latency panel shows exactly 2.5 s, flat, for six hours. What is happening and how do you confirm it?**
The quantile's rank is falling in the `+Inf` bucket, so `BucketQuantile` takes case (a) and returns the upper bound of the second-to-last bucket — the highest *finite* boundary, here 2.5 s. The panel is pinned at its ceiling and cannot distinguish a 3-second p99 from a 30-second one. Confirm by comparing the panel value to the highest finite `le` in the metric (they will be exactly equal) and by checking `sum(rate(..._bucket{le="2.5"}[5m])) / sum(rate(..._count[5m]))` — if that ratio is below 0.99, more than 1 % of requests are past the top finite bucket. Fix by adding boundaries above the tail, or by migrating to native histograms.

**Derive the "rate window ≥ 4× scrape interval" rule.**
`extrapolatedRate` returns nothing unless the window contains at least two samples, and needs at least two to compute `avgInterval` for the extrapolation threshold. A window of exactly 2× the scrape interval contains 2 samples with zero margin, so a single missed scrape empties the result — which renders as a gap, which reads as an outage. At 4× you hold 4 samples and tolerate two consecutive misses. The rule therefore scales with *your* scrape interval: at the Prometheus default 60 s `scrape_interval`, `rate(x[1m])` is broken by construction and 4 m is the floor. In Grafana use `$__rate_interval`, which is defined as `max($__interval + scrape_interval, 4 × scrape_interval)` precisely to enforce this; in rules pin a literal duration.

**A counter resets and climbs back above its old value between two scrapes. What does `increase()` report and why?**
It reports only `last − first`, with no correction, because reset detection is sample-based: Prometheus compares consecutive *sampled* values and only corrects when it observes a decrease. If no sample lands between the reset and the recovery past the old value, the sampled sequence is monotonic and looks normal. In the §7 example the true increase was 1,850 and `increase()` reported 50 — a 97 % undercount. Prometheus 3.x closes this when the target exposes a created/start timestamp: `isStartTimestampReset` detects the reset from the metadata even when values did not decrease. Check whether your exporter actually emits it before relying on it.

**Why does the fleet request-rate panel spike on every rolling deploy, and what is the one-word fix?**
The expression is `rate(sum(...))` rather than `sum(rate(...))`. Counter-reset correction is a per-series property. Summing first merges every replica's restart into a dip in one aggregate series; the outer `rate()` sees the dip, treats it as a single reset, and adds back the *entire aggregate value* — producing a spike proportional to total fleet traffic on every staggered restart. Fix: rate first, aggregate second. The deeper harm is that teams learn to ignore the deploy spike, so the signal is dead when a deploy genuinely breaks something.

**What is `prometheus_rule_group_iterations_missed_total` and why is it the most important meta-alert you can have?**
Rule groups evaluate sequentially at a fixed interval. If a group's evaluation takes longer than its interval, Prometheus skips iterations rather than queueing them. When that happens, recording rules stop producing points and alerting rules stop evaluating — with no error and no alert, because the mechanism that would alert you is the one that stopped. Monitor `prometheus_rule_group_last_duration_seconds / prometheus_rule_group_interval_seconds` (alert above ~0.7 sustained) and page on any increase in `iterations_missed_total`. The usual causes are a recording rule fanning out over too many raw series and a heavy dashboard starving the shared query engine.

**A GPU node is preempted mid-training. Trace what happens to its series, its alerts, and what you should have written instead.**
Service discovery drops the endpoint; Prometheus appends a `StaleNaN` stale marker to every series from that target; from that point instant queries return nothing for those series and range queries end the line at the last real sample. An alert expressed as `metric < threshold` therefore matches nothing and **resolves** — the vanished node reads as healthy. (If the exporter attaches its own timestamps, you get the opposite: the last value persists for the 5-minute lookback, a phantom flatline, controlled by `track_timestamps_staleness`.) The correct construction is two alerts: "allocated but idle", gated with `and on (Hostname, gpu) dcgm_gpu_allocated == 1` so it only fires when someone is paying; and a fleet-coverage alert comparing `count(count by (Hostname) (...))` against an inventory metric, because per-node `absent()` would need one rule per node and `absent()` can only report the labels you typed into it.

## Connections & what's next

This lesson assumes the label and cardinality decisions from [01 · The signal model](01-signal-model.md) are already made — the recording rules here exist to keep the aggregations those decisions require cheap, and the `group_left` info-metric join in Panel E is the query-side half of the label demotion made there. Lesson 3 picks up what happens when these same correctly-written queries run at genuine fleet fan-in: rule-evaluation cost, query sharding, and why federation is the wrong answer. Lesson 6 covers the equivalent correctness traps in LogQL, where the aggregation-order mistakes rhyme but the cost model differs. Lesson 7 takes the exact-boundary bucket-ratio SLI from §4 and builds multi-window burn-rate alerting on it — the reason that query, and not `histogram_quantile`, is the one to get right. Lesson 9 applies this whole trap list to the concrete DCGM and NCCL signal set at fleet scale.

Next: [03 · Metrics at scale](03-metrics-at-scale.md) — how a metrics system actually falls over once these correctly-written queries run against a fleet-sized series count.

## References & further reading

**Primary sources — read directly from the tree**
- Prometheus 3.14.0 source, `promql/functions.go` — `extrapolatedRate` (the reset-correction loop, the 1.1× extrapolation threshold, the half-interval fallback, the counter zero clamp), `instantValue` (the `irate`/`idelta` last-two-samples algorithm), and `linearRegression` (what `deriv` and `predict_linear` actually do). **This corrects the previous version of this lesson**, which described `rate()` as performing "least-squares-style smoothing"; it does not, and the distinction changes how you reason about noise sensitivity.
- Prometheus 3.14.0 source, `promql/quantile.go` — `BucketQuantile` (the three cases, including the `+Inf` clamp returning the second-to-last bucket's upper bound), `HistogramQuantile` (logarithmic interpolation for exponential native-histogram buckets), and `ensureMonotonicAndIgnoreSmallDeltas` (the silent monotonicity repair and its `1e-12` tolerance). **Also a correction:** the previous version said the `+Inf` case returns "the lower bound or an assumed edge — either way, fiction." The behaviour is precise and predictable, which makes it diagnosable.
- Prometheus 3.14.0 source, `promql/engine.go` — `defaultLookbackDelta = 5 * time.Minute`.
- Prometheus 3.14.0 source, `config/config.go` — `DefaultGlobalConfig`: `scrape_interval: 1m`, `scrape_timeout: 10s`, `evaluation_interval: 1m`, `scrape_native_histograms: false`, `convert_classic_histograms_to_nhcb: false`, `always_scrape_classic_histograms: false`.
- Prometheus 3.14.0 source, `model/value/value.go` — `StaleNaN = 0x7ff0000000000002`, the stale-marker payload.
- [Prometheus querying basics](https://prometheus.io/docs/prometheus/latest/querying/basics/) — the staleness and lookback semantics quoted in §9, and the left-open/right-closed range-selector definition. Mirrored in the repo at `docs/querying/basics.md`.
- [Prometheus 3.0 migration guide](https://prometheus.io/docs/prometheus/latest/migration/) — the range-selector boundary change and its effect on sample counts. Repo: `docs/migration.md`.
- [Prometheus query functions reference](https://prometheus.io/docs/prometheus/latest/querying/functions/) — the per-function contracts.
- [Prometheus configuration reference](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) — the native-histogram scrape switches, the schema/growth-factor table reproduced in §5, `native_histogram_bucket_limit`, `native_histogram_min_bucket_factor`, and `track_timestamps_staleness`. Repo: `docs/configuration/configuration.md`.
- [Prometheus recording rules configuration](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/) — rule-group semantics, sequential in-group evaluation at a shared timestamp, `interval`, `limit`, `query_offset`. Repo: `docs/configuration/recording_rules.md`.
- [Prometheus feature flags](https://prometheus.io/docs/prometheus/latest/feature_flags/) — `promql-extended-range-selectors` (`anchored`, `smoothed`) and the rule-offset requirement. Repo: `docs/feature_flags.md`.
- [Prometheus histograms and summaries practices](https://prometheus.io/docs/practices/histograms/) — the bucket-placement guidance and the client-side-quantile limitation of summaries.

**Real-world engineering write-ups**
- PromLabs, [Avoid These 6 Mistakes When Getting Started With Prometheus](https://promlabs.com/blog/2022/12/11/avoid-these-6-mistakes-when-getting-started-with-prometheus/)
- Grafana Labs, [Prometheus native histograms in Grafana Cloud](https://grafana.com/blog/prometheus-native-histograms-in-grafana-cloud-more-precise-easier-to-use-and-better-compatibility/)
- Robusta.dev, [3 Common Mistakes with PromQL and Kubernetes Metrics](https://home.robusta.dev/blog/3-common-mistakes-with-promql-and-kubernetes-metrics)

**Sources consulted but not relied upon.** Where a vendor documentation page was unreachable from this environment's egress proxy, the fact was verified against the upstream Git repository instead (cloned and read locally, as noted above) or omitted. The Grafana `$__rate_interval` definition in §6 is stated as the widely-documented formula `max($__interval + scrape_interval, 4 × scrape_interval)`; confirm against your Grafana version's docs before quoting it in a design review.
