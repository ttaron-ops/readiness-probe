---
lesson: "A03.7"
title: "SLOs and alerting"
module: "A-03"
concept: "multi-window multi-burn-rate"
status: not-started
est_time: "3.5 hrs"
prev: "06-logging-pipelines.md"
next: "08-profiling-and-ebpf.md"
artifacts: ["mwmbr-alert-set.promql"]
sources: 14
---

# A03.7 · SLOs and alerting

> **Concept.** Multi-window multi-burn-rate alerting is the noise/detection Pareto point — and for GPU fleets the SLI must be goodput, not utilization.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

Lessons 01–06 built the signal stack — metrics, traces, logs — and the correctness traps in each. This lesson is where those signals get *consumed*: alerting is the layer that turns a correct signal into a decision to wake a human (or not). Get the burn-rate math wrong here and every upstream correctness fix (cardinality discipline, tail-sampling, log demotion) is wasted, because the alert either drowns it in noise or misses it entirely. Once an alert fires correctly, the next question is *why* — that's lesson 08, which picks up exactly where this one stops: from "the SLI is burning budget" to "which stack is burning the cycles or the wait time."

Everything below is checked against **Prometheus** (`main`, August 2026 — `docs/configuration/alerting_rules.md`, `docs/configuration/recording_rules.md`, `docs/configuration/configuration.md`), **Alertmanager** (`main` — `docs/configuration.md`, `notify/notify.go`, `dispatch/dispatch.go`, `inhibit/inhibit.go`, `cmd/alertmanager/main.go`) and **Sloth** (`main` — `internal/alert/window.go`, `internal/alert/windows/google-30d.yaml`, `internal/plugin/slo/core/alert_rules_v1/plugin.go`), all read from the upstream repositories because the rendered documentation sites are unreachable from this environment. Every burn-rate multiplier in this lesson is **derived from the formula**, not copied from a table, so you can re-derive them for any SLO period you are handed.

## Why this matters

At staff scale the question is no longer "do we have SLOs" but "why does this alert set page at exactly the right moment and stay silent through a transient blip." The difference is arithmetic, and it is arithmetic most engineers have never done: a threshold that is 2× too high misses a leak that eats a month's budget in four days; a threshold that is 2× too low pages your on-call every time a canary restarts.

There is a second, less obvious cost. **A page is a claim about the future** — "at this rate, you will be out of budget in about two days" — and a claim about the future can be checked. If you cannot state the burn rate an alert fires at, the budget fraction that burn consumes in the alert's window, and the detection delay that follows from both, then you cannot tell whether your alert set has a hole in it. Most alert sets have exactly one hole: a burn rate high enough to matter and low enough to sit under every threshold you configured, burning quietly for days.

And for GPU fleets the standard playbook actively misleads you: a utilization gauge at 100% can be burning wasted-GPU-hours budget while every dashboard looks green. This lesson is the design math and the GPU reframe, not the vocabulary.

## What's new here (calibration)

- **Skip:** SLI/SLO/error-budget definitions, symptom-over-cause as a principle, "page fatigue is bad," naive fixed-threshold alerts.
- **New: the multipliers as a derivation, not a table.** `burn_rate = budget_fraction × SLO_window / alert_window` — one line of algebra that regenerates 14.4 / 6 / 3 / 1 for a 30-day period, and gives you 13.44 / 5.6 / 2.8 / 0.93 for a 28-day one. If your SLO period is not 30 days, the famous numbers are wrong for you.
- **New: detection time and reset lag as closed-form functions** of window, threshold and actual burn rate — so "what does the fast window catch that the slow one doesn't" becomes two numbers rather than an intuition.
- **New: the traffic floor derived from the binomial**, including the exact request rate below which a single error crosses the fast-page threshold, and the tail probability of a false page at any given rate.
- **New: the Alertmanager pipeline as a mechanism** — the exact stage order (`gossip-settle → inhibit → time-active → time-mute → silence → cluster-wait → dedup → retry → set-notifies`), the real defaults (`group_wait: 30s`, `group_interval: 5m`, `repeat_interval: 4h`, `resolve_timeout: 5m`, `--cluster.peer-timeout=15s`), and what each one does to the page you just designed.
- **New: the recording-rule architecture** that makes five windows per SLO affordable, including the `sloth_window` label trick and the correct way to compute a 30-day ratio without a 30-day `rate()`.
- **New: the GPU goodput budget in GPU-hours**, integrated correctly with `sum_over_time(...) * step/3600` rather than the `avg_over_time(...) × window_hours` form that silently inflates any job that did not run for the whole window.

## Core concepts

### 1. What an SLO actually commits you to

An SLO is not a threshold. It is a **quantity of permitted failure over a stated period**, and everything in this lesson falls out of treating it as a quantity.

Write it out for a real service:

```
   THE BUDGET AS A QUANTITY
   ═══════════════════════════════════════════════════════════════════════════
   SLO         : 99.9 % of requests succeed, measured over a rolling 30 days
   Traffic     : 20,000,000 requests / 30 days  (≈ 7.72 req/s mean)

   error budget fraction   β = 1 − 0.999            = 0.001
   error budget (events)     = 20e6 × 0.001         = 20,000 bad requests
   sustainable bad rate      = 20,000 / 720 h       ≈ 27.8 bad requests/hour
                             = 20,000 / 2,592,000 s ≈ 0.0077 bad requests/s
```

Three things become expressible that were not before:

1. **"How bad is this incident" has a unit.** A 10-minute total outage at 7.72 req/s costs `7.72 × 600 ≈ 4,630` bad requests — 23% of the month's budget, in ten minutes.
2. **"Are we on track" is a comparison of two rates**, not a comparison of a gauge to a line.
3. **The alert can be about the *rate of spend*** rather than the level of failure. That is the entire idea of burn rate.

**The measurement window matters as much as the number.** A *rolling* 30-day window means the budget refills continuously as old failures scroll out; a *calendar-month* window means budget resets on the 1st and a bad 28th is survivable in a way a bad 2nd is not. Rolling is the default in every implementation you will meet (it is what `rate(...[30d])` computes) and is the assumption throughout this lesson. Say which one you mean in the SLO document, because the two produce different exhaustion dates from identical data.

### 2. Burn rate, defined by construction

**Burn rate is a dimensionless ratio: the rate you are consuming budget divided by the rate that would consume exactly all of it, exactly at period end.**

```
              observed bad-event ratio        r
   burn rate =──────────────────────────  =  ───
              error budget fraction           β
```

At burn rate 1 you finish the period having spent precisely 100% of budget. At burn rate 14.4 you would spend it in `30 / 14.4 ≈ 2.08` days. At burn rate 1000 (a total outage on a 99.9% SLO, `r = 1`) you spend it in `720 h / 1000 = 43.2 minutes`.

The identity everything else hangs off:

```
   budget_fraction_consumed  =  burn_rate × ( alert_window / SLO_window )
```

Check it dimensionally: burn rate is (fraction of budget per unit time) ÷ (fraction of budget per unit time over the full period) — dimensionless — and multiplying by a ratio of times gives a fraction of budget. Check it numerically: burn rate 14.4 sustained for 1 hour against a 30-day (720 h) SLO consumes `14.4 × 1/720 = 0.02` = **2% of the month's budget in that hour**.

Two consequences people get wrong constantly:

- **Burn rate is not a percentage.** "Burn rate 14.4" does not mean 14.4% of anything. It is a multiplier on the sustainable spend rate.
- **Burn rate is comparable across services.** A 99.9% service and a 99.99% service have error budgets that differ by 10×, and raw error rates that are not comparable at all. Burn rate 14.4 means the same operational thing in both: *two days to exhaustion*. This is why a single alert threshold can be templated across an entire estate — it is the normalisation that makes an SLO platform possible.

### 3. Deriving the multipliers — where 14.4 comes from

Rearranged, the identity is a recipe for picking a threshold. You choose **how much budget you are willing to let burn before you want to be told**, and **how long a window you are willing to wait to establish it**, and the multiplier falls out:

```
                       budget_fraction × SLO_window
   burn_rate_threshold = ────────────────────────────
                                alert_window
```

That is exactly the function Sloth implements in `internal/alert/window.go` (`getBurnRateFactor`): hours of budget consumption required = `errorBudgetPercent × totalWindow.Hours() / 100`, then divide by the consumption window's hours.

Sloth ships the canonical window catalogue as data, and reading it as data rather than as folklore is the whole point. `internal/alert/windows/google-30d.yaml`, in full:

| Tier | Budget % it lets burn | Long window | Short window | Derived multiplier |
|---|---:|---|---|---:|
| Page — quick | 2% | 1h | 5m | `0.02 × 720 / 1` = **14.4** |
| Page — slow | 5% | 6h | 30m | `0.05 × 720 / 6` = **6** |
| Ticket — quick | 10% | 1d | 2h | `0.10 × 720 / 24` = **3** |
| Ticket — slow | 10% | 3d | 6h | `0.10 × 720 / 72` = **1** |

Now the same catalogue for a **28-day** period (`google-28d.yaml`, `SLO_window = 672 h`), which is what you get if your SLO period is "4 weeks" rather than "a month":

| Tier | Budget % | Long window | Derived multiplier |
|---|---:|---|---:|
| Page — quick | 2% | 1h | `0.02 × 672 / 1` = **13.44** |
| Page — slow | 5% | 6h | `0.05 × 672 / 6` = **5.6** |
| Ticket — quick | 10% | 1d | `0.10 × 672 / 24` = **2.8** |
| Ticket — slow | 10% | 3d | `0.10 × 672 / 72` = **0.933** |

**This is the point of the derivation.** The famous numbers are not constants of nature; they are `14.4 = 2% × 720h / 1h`. Copy them onto a 7-day SLO period (`168 h`) and every threshold is 4.3× too high: the correct page-quick multiplier there is `0.02 × 168 / 1 = 3.36`. A team running weekly SLOs with a 14.4 threshold has an alert that fires only when they are burning budget nine times faster than "gone by Thursday."

**Where the 2% / 5% / 10% come from is a policy choice, not maths.** They encode three judgements: an event worth a 3 a.m. page should be one that would eat the whole month in about two days; a slower leak worth a daytime page should be one that would eat it in about five; and everything else is a ticket. The Sloth source comments them as the SRE-workbook defaults that "work correctly most of the times." Re-deriving them for your own service means answering one question honestly — *how much of the budget am I willing to have spent by the time someone looks?* — and turning the answer into a multiplier with the formula above. That is a defensible design conversation. "14.4 because a blog post said so" is not.

**A useful sanity anchor: the multiplier is the reciprocal of the exhaustion time.** Burn rate `b` exhausts the budget in `SLO_window / b`. So 14.4 → 2.08 days, 6 → 5 days, 3 → 10 days, 1 → 30 days (by definition). If someone proposes a threshold, convert it to an exhaustion time and ask whether that is page-worthy. It usually settles the argument in one sentence.

### 4. Detection time and reset lag — what each window catches, and what it misses

The multiplier tells you *whether* you fire. The window tells you *when*, and how long you keep firing after the incident ends. Both are closed-form, and deriving them is what turns "fast window vs slow window" from vibes into engineering.

**Setup.** A rolling window of length `W` computes the mean bad-event ratio over the last `W` of time. An incident starts at `t = 0` and holds a constant burn rate `b` (i.e. bad ratio `r = b·β`). The alert threshold is burn rate `B`.

**Detection time.** At time `t < W`, the window contains `t` seconds of incident and `W − t` seconds of clean traffic, so the windowed burn rate is `b · t/W` (assuming steady request rate). It crosses `B` when:

```
   b · t/W = B        ⇒       t_detect = W · B / b        (valid for b ≥ B)
```

Concretely, for the fast page tier (`W = 1h`, `B = 14.4`) on a 99.9% SLO:

| Actual burn rate `b` | What it is | `t_detect = 3600 s × 14.4 / b` | Budget spent by then |
|---:|---|---:|---:|
| 1000 | total outage (`r = 100%`) | **51.8 s** | 2.0% |
| 200 | 20% of requests failing | **4.3 min** | 2.0% |
| 50 | 5% failing | **17.3 min** | 2.0% |
| 20 | 2% failing | **43.2 min** | 2.0% |
| 14.4 | 1.44% failing | **60 min** (the full window) | 2.0% |
| 10 | 1% failing | **never** | — |

Read the last column first: **every row spends exactly 2% of the budget before the alert fires.** That is not a coincidence, it is the design — the tier is defined by the budget fraction, and detection time is whatever falls out. This is the single most useful reframing in the lesson: *a burn-rate tier is a promise about how much budget you will lose before someone is told, not a promise about how fast you are told.*

Read the last row second: **a sustained 10× burn never trips the fast tier.** It would exhaust the month in three days. That is precisely the hole the slower tiers exist to plug, and it is why a single-tier burn-rate alert is broken by construction.

**Coverage, stated as intervals.** With the 30-day catalogue:

```
   BURN-RATE COVERAGE — WHICH TIER OWNS WHICH FAILURE
   ═══════════════════════════════════════════════════════════════════════════
   burn rate  b:   1        3         6        14.4              1000
                   ├────────┼─────────┼─────────┼─────────────────┤
   exhausts in:   30d      10d        5d       2.08d            43 min

   ticket-slow    ██████████████████████████████████████████████  fires (1×)
   ticket-quick            ████████████████████████████████████   fires (3×)
   page-slow                         ██████████████████████████   fires (6×)
   page-quick                                  ████████████████   fires (14.4×)

   detection time at the tier's own threshold:
     page-quick   1h   ·  page-slow 6h  ·  ticket-quick 1d  ·  ticket-slow 3d
   detection time at b = 1000 (total outage), from t = W·B/b:
     page-quick   51.8 s  ·  page-slow 129.6 s
     ticket-quick 259.2 s ·  ticket-slow 259.2 s   ← identical, and not a typo
```

**Why those last two are identical is the cleanest result in the lesson.** Substitute the threshold's own definition, `B = budget_fraction × SLO_window / W`, back into the detection formula:

```
                 W · B        W   budget_fraction × SLO_window     budget_fraction × SLO_window
   t_detect  =  ───────  =  ───· ─────────────────────────────  = ─────────────────────────────
                   b          b                W                                b
```

**The window cancels.** Detection time depends only on the tier's *budget fraction*, the SLO period, and the actual burn rate — not on the window length at all. The 2% tier detects any burn `b` after `14.4h / b`; the 5% tier after `36h / b`; both 10% tiers after `72h / b`, which is why ticket-quick and ticket-slow detect simultaneously despite windows that differ by 3×.

So the window is not a speed knob. **The window is a noise knob**: for a fixed budget fraction, a longer window means a proportionally lower threshold, identical detection time, more samples in the denominator, and a longer reset lag. That is the entire trade, and it kills the common mental model of "the fast tier is for fast incidents." It is not. At a total outage every tier fires within minutes. What the fast tier uniquely provides is **firing once only 2% of the budget is gone, for any burn between 14.4× and infinity**; what the slow tiers uniquely provide is **firing at all when the burn is below 14.4×**, at the cost of letting 5% or 10% go first. Coverage and budget-cost, not speed, are the axes.

**Reset lag — why the short window exists.** Now let the incident end at time `d` and ask when the alert clears. The window still contains the incident until it scrolls out. With overlap `o(s) = d − s` at `s` seconds after recovery (for `d ≤ W`), the condition `b · o/W > B` holds until:

```
   s_reset = d − W · B / b          (floored at 0)
```

For a 10-minute total outage (`d = 600 s`, `b = 1000`, `β = 0.001`):

- Long window alone (`W = 1h`, `B = 14.4`): `s_reset = 600 − 51.8 = 548 s` ≈ **9.1 minutes of firing after the incident is over.**
- Short window (`W = 5m`, `B = 14.4`): `s_reset = 600 − 300 × 14.4/1000 = 600 − 4.3 = 596 s`… which looks worse until you notice `d > W`, so the correct expression for `d > W` is `s_reset = W − W·B/b = 300 − 4.3 = 296 s` ≈ **4.9 minutes.**

Because the tier is a logical AND, **the alert clears as soon as either window drops** — so the short window governs the reset, and it is 2× faster here. Push the incident duration up and the gap widens: a 45-minute outage clears 55.7 minutes after recovery on the 1h window alone, versus 4.9 minutes with the 5m confirm window.

**So state the two-window AND's job precisely, in two clauses:**

- The **long window** supplies *statistical confidence and budget accounting* — it is what makes the threshold mean "2% of the month," and its length is what keeps a handful of unlucky errors from crossing.
- The **short window** supplies *currency* — it forces the condition to still be true now, which (a) collapses reset lag from ~`W` to ~`W_short`, and (b) prevents a long-ago spike that is still inside the long window from re-firing a page after recovery.

The conventional short:long ratio of **1:12** (5m:1h, 30m:6h, 2h:1d, 6h:3d) is itself a design choice — short enough to be current, long enough not to be noise. If your traffic is bursty at the minute scale, 1:12 on a 5-minute window may be too jumpy and 1:6 (10m:1h) buys stability at the cost of ~5 minutes of extra reset lag. Justify the ratio you pick; do not inherit it silently.

### 5. The alert expression, exactly as it ships

A tier is two conditions ANDed. A severity is two tiers ORed. Here is the generated shape from Sloth's `alert_rules_v1` plugin, which is worth copying because it packs both page tiers into **one** alert rather than two:

```
(
    max(slo:sli_error:ratio_rate5m{sloth_id="api-availability"} > (14.4 * 0.001)) without (sloth_window)
    and
    max(slo:sli_error:ratio_rate1h{sloth_id="api-availability"} > (14.4 * 0.001)) without (sloth_window)
)
or
(
    max(slo:sli_error:ratio_rate30m{sloth_id="api-availability"} > (6 * 0.001)) without (sloth_window)
    and
    max(slo:sli_error:ratio_rate6h{sloth_id="api-availability"} > (6 * 0.001)) without (sloth_window)
)
```

Three details in there are load-bearing:

- **The comparison is against `multiplier × β`, not against the multiplier.** The recorded series is a *ratio of bad events*, not a burn rate; the multiplication converts the burn-rate threshold into ratio space. `14.4 × 0.001 = 0.0144` — fire when 1.44% of requests are failing. You can equally record burn rate directly (divide the ratio by β in the recording rule) and compare against `14.4`; pick one convention and never mix them, because a dashboard showing "0.0144" next to an alert saying "14.4" is a 3 a.m. mistake waiting to happen.
- **`max(...) without (sloth_window)`** strips the window label so the two sides of the `and` have matching label sets. Without it the vector match finds nothing and your alert silently never fires — the single most common failure when hand-rolling this. If you record per-window series with a window label (recommended, see §7), you must strip it before joining.
- **One alert, two tiers, one severity.** Two separate `page` alerts for the same SLO produce two pages for one incident unless Alertmanager groups them; folding them into one expression makes the grouping unnecessary and the alert's meaning ("this SLO is burning at a page-worthy rate, by either definition") precise.

### 6. The SLI must be a ratio — and quantiles are not ratios

Everything above assumes you can compute a **bad-event ratio** over an arbitrary window. That constrains how you define the SLI, and it rules out the definition most teams reach for first.

**Two legitimate SLI shapes:**

| Shape | Definition | Numerator / denominator | When to use |
|---|---|---|---|
| **Event-based** | fraction of *events* that were bad | bad requests / all requests | anything request-driven; the default |
| **Window-based** | fraction of *time windows* that were bad | bad minutes / all minutes | pipelines, batch, anything where "requests" is not the unit of user pain |

**What is not an SLI: a quantile.** `histogram_quantile(0.99, ...)` produces a latency, not a ratio. You cannot average it across replicas (lesson 02's quantile-of-quantiles trap), you cannot re-window it, and you cannot subtract it from 1 to get a budget. Convert to a ratio by asking the compliance question instead:

```promql
# SLI: fraction of requests SLOWER than the 500 ms budget.
# Exact, not interpolated — a cumulative bucket count is a count, not an estimate.
1 - (
    sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m]))
  /
    sum(rate(http_request_duration_seconds_count[5m]))
)
```

This only works if **`0.5` is a real bucket boundary**. If your histogram's boundaries are `…, 0.25, 1.0, …`, then `le="0.5"` does not exist, `le="1.0"` answers a different question, and any interpolation you do instead re-imports the error you were avoiding. Fix the buckets, do not fix the query. (The OpenTelemetry GenAI conventions do exactly this for you: `gen_ai.server.time_to_first_token` specifies explicit boundaries `[0.001, 0.005, 0.01, 0.02, 0.04, 0.06, 0.08, 0.1, 0.25, 0.5, 0.75, 1.0, 2.5, 5.0, 7.5, 10.0]`, so 100 ms, 250 ms, 500 ms and 1 s are all exact SLO thresholds and 300 ms is not.)

**Composite SLIs need care with the denominator.** If availability and latency are both in the SLO, decide whether a failed request counts in the latency denominator. The usual correct answer is no — you cannot be slow if you never answered — which means the two SLIs have different denominators and therefore need separate budgets. Blending them into one number destroys the ability to say which one broke.

### 7. Recording rules: making five windows per SLO affordable

Each SLO needs the bad-ratio at 5m, 30m, 1h, 2h, 6h, 1d, 3d — seven windows if you run all four tiers — plus a 30-day series for the budget dashboard. Evaluating those inline in alert expressions means a `rate()` over 3 days of raw series on every evaluation cycle, per SLO. At a few hundred SLOs that is how you starve the rule evaluator (lesson 01's point that a heavy query load degrades alerting, arriving from the alerting side this time).

**The pattern: one recording rule per (SLO, window), carrying the window as a label.**

```yaml
groups:
  - name: slo-api-availability-sli
    interval: 30s                     # rules in a group run sequentially at this cadence
    rules:
      - record: slo:sli_error:ratio_rate5m
        expr: |
          sum(rate(http_requests_total{job="api", code=~"5.."}[5m]))
          /
          sum(rate(http_requests_total{job="api"}[5m]))
        labels: { sloth_id: "api-availability", sloth_service: "api", sloth_window: "5m" }

      - record: slo:sli_error:ratio_rate1h
        expr: |
          sum(rate(http_requests_total{job="api", code=~"5.."}[1h]))
          /
          sum(rate(http_requests_total{job="api"}[1h]))
        labels: { sloth_id: "api-availability", sloth_service: "api", sloth_window: "1h" }

      # …30m, 6h, 2h, 1d, 3d identically…

      # The 30-day series for the budget-remaining panel. DO NOT rate() over 30d.
      # Averaging the 5m ratio over 30d is a ratio-of-ratios; the sum/count form
      # below is the statistically correct way to average a ratio series.
      - record: slo:sli_error:ratio_rate30d
        expr: |
          sum_over_time(slo:sli_error:ratio_rate5m{sloth_id="api-availability"}[30d])
          / ignoring (sloth_window)
          count_over_time(slo:sli_error:ratio_rate5m{sloth_id="api-availability"}[30d])
        labels: { sloth_id: "api-availability", sloth_service: "api", sloth_window: "30d" }
```

Why the last one is written that way, since it looks like cheating: a 30-day `rate()` over raw counters reads 30 days of chunks on every evaluation, which is brutally expensive and, on a remote-read setup, often simply times out. `sum_over_time / count_over_time` over the pre-computed 5-minute ratio reads 30 days of *one recorded series* — `30d / 30s = 86,400` samples — and divides the sum of ratios by the number of ratios. That is the arithmetic mean of equally-spaced ratios, which is what you want, and it is *not* the same as `avg_over_time` of a ratio-of-different-denominators only in the sense that both share the equal-weighting assumption. The assumption is explicit: **each 5-minute bucket is weighted equally regardless of how many requests it contained.** For a service with strong diurnal traffic that slightly over-weights the quiet hours. If that matters, record the numerator and denominator counts separately and divide the sums — more series, exact answer. State which one you chose in the SLO document.

Two more mechanical details from `docs/configuration/recording_rules.md` worth knowing:

- **Rules in a group run sequentially with the same evaluation timestamp.** So an alert rule placed *after* the recording rules it depends on, *in the same group*, always sees fresh data from the same instant. Split them across groups and you introduce a one-interval skew that shows up as an alert that flaps at group boundaries.
- **A group that overruns its interval is skipped, not queued** — `rule_group_iterations_missed_total` increments and your recorded series gets a *gap*. A gap in `slo:sli_error:ratio_rate5m` makes the `and` in your alert expression evaluate to nothing, which looks exactly like "everything is fine." Alert on `rule_group_iterations_missed_total > 0`; it is the alerting system's own liveness signal.

### 8. The traffic floor, derived from the binomial

Burn rate is a ratio estimator, and ratio estimators are meaningless when the denominator is small. Everyone knows to add "AND enough traffic." Almost nobody can say what "enough" is. Derive it.

Let `n` be the number of requests in the *short* window (the short window is the binding constraint — it is smallest), `β` the budget fraction, `B` the burn threshold. The alert fires when the observed bad count `k` satisfies `k/n > B·β`, i.e. `k > B·β·n`.

**Step 1 — the single-error line.** If `B·β·n < 1`, then **one** bad request crosses the threshold:

```
   n_single = 1 / (B · β)
```

| SLO | β | B = 14.4 | Requests in 5m | Request rate |
|---|---:|---:|---:|---:|
| 99.9% | 0.001 | `1/(14.4×0.001)` = **69.4** | fewer than ~69 | < 0.23 req/s |
| 99.5% | 0.005 | **13.9** | fewer than ~14 | < 0.05 req/s |
| 99.99% | 0.0001 | **694** | fewer than ~694 | < 2.3 req/s |

Read the last row carefully: **a 99.99% SLO on a service doing less than 2.3 requests/second can be paged by a single error.** Tighter SLOs make the low-traffic problem *worse*, not better, which is the opposite of most people's intuition.

**Step 2 — the false-page rate.** Being above `n_single` is necessary, not sufficient. Bad events arrive as a Poisson process with mean `λ = n·β` per window (at exactly the SLO, i.e. the service is behaving). The alert needs `k ≥ ⌈B·β·n⌉`. So:

```
   P(false page in one window) = P( Poisson(λ = nβ) ≥ ⌈B·β·n⌉ )
```

Worked for a 99.9% SLO, `B = 14.4`, 5-minute short window:

| Requests in 5m (`n`) | `λ = nβ` | needs `k ≥` | P(one window) | Independent 5m windows/30d | Expected false pages/month |
|---:|---:|---:|---:|---:|---:|
| 100 | 0.10 | 2 | 4.7 × 10⁻³ | 8,640 | **≈ 41** |
| 200 | 0.20 | 3 | 1.1 × 10⁻³ | 8,640 | **≈ 9.8** |
| 500 | 0.50 | 8 | 1.0 × 10⁻⁷ | 8,640 | ≈ 0.001 |
| 1,000 | 1.00 | 15 | 1.0 × 10⁻¹² | 8,640 | negligible |
| 5,000 | 5.00 | 72 | ~10⁻⁴⁶ | 8,640 | negligible |

*(Poisson tail computed as `1 − Σ_{i<k} e^{−λ} λⁱ/i!`; the "independent windows" count treats consecutive non-overlapping 5-minute windows as independent, which slightly understates the true page count because a rolling window re-tests overlapping data — treat these as a lower bound.)*

**The cliff is between 200 and 500 requests per 5-minute window.** That is the actual justification for the "`AND rate(requests) > 200`"-style floor you see in every published rule set, and it is why the floor must move when the SLO does: at 99.99% the same table shifts right by roughly 10×.

**The rule of thumb that falls out:** size the floor so that the *expected bad count at the threshold* is at least about 5–10 events. `B·β·n ≥ 8` ⇒ `n ≥ 8/(B·β)`. For 99.9%/14.4 that is `n ≥ 555` in 5 minutes ≈ **1.85 req/s sustained**. Below that, do not tune the multiplier — change the design:

| Fix | Mechanism | Cost |
|---|---|---|
| Lengthen windows | more `n` per window | slower detection, proportionally |
| Aggregate related services into one SLO | bigger denominator | you lose per-service attribution |
| Switch to a **window-based** SLI (bad minutes / total minutes) | denominator is time, which is never sparse | coarser: a minute with 1 of 2 requests failing is 100% bad |
| Synthetic probes at a fixed rate | you own the denominator | probes measure the probe path, not user traffic |
| Do not alert; report | budget consumption reviewed weekly | no paging at all — often the right answer |

The last row is the staff answer that nobody gives: **a service with 0.05 req/s does not have a statistically meaningful availability SLO on a 5-minute window, and pretending otherwise manufactures pages.** Report its monthly budget consumption and alert on something else — a heartbeat, a queue age, a dependency's SLO.

### 9. The delivery path: what Alertmanager does to your page

A firing alert rule is not a page. Between `expr` evaluating true and a phone buzzing sit two systems and about eight stages, each with a default that will surprise you at least once.

```
   FROM EXPRESSION TO PAGER — THE FULL PATH, WITH REAL DEFAULTS
   ═══════════════════════════════════════════════════════════════════════════

   PROMETHEUS
     every  evaluation_interval (default 1m; set 30s for SLO groups)
        │
        ├─ evaluate expr ─▶ result vector non-empty?
        │                       │ yes
        │                       ▼
        │                  state = PENDING ──── held for `for:` ────▶ FIRING
        │                  (`for` default 0s — fires on first evaluation)
        │                  synthetic series: ALERTS{alertstate="pending"|"firing"}
        │                       │
        │                  `keep_firing_for` (default 0s) holds FIRING
        │                  for N more after the condition clears
        │                       ▼
        └─ push (min gap --rules.alert.resend-delay, default 1m), carrying
           EndsAt = now + 4 × max(group interval, resend-delay)
                                │
   ALERTMANAGER                 ▼
     ┌──────────────────────────────────────────────────────────────────┐
     │ 1. gossip-settle   wait for cluster state on startup             │
     │ 2. INHIBIT         muted by a matching source alert?  ───────────┼─▶ drop
     │ 3. time-active     outside active_time_intervals?     ───────────┼─▶ drop
     │ 4. time-mute       inside mute_time_intervals?        ───────────┼─▶ drop
     │ 5. SILENCE         matches an active silence?         ───────────┼─▶ drop
     │ 6. cluster-wait    sleep (peer position × 15s)   ← --cluster.peer-timeout
     │ 7. DEDUP           already notified per nflog?        ───────────┼─▶ drop
     │ 8. RETRY           send to the receiver, with backoff            │
     │ 9. set-notifies    write nflog entry, gossip it to peers         │
     └──────────────────────────────────────────────────────────────────┘
              ▲                                   ▲
              │                                   │
     DISPATCHER groups alerts by group_by     RESOLUTION
       first notification after group_wait      an alert with no new samples is
         (default 30s)                          resolved after resolve_timeout
       subsequent after group_interval          (default 5m) — or immediately if
         (default 5m)                           the rule sends an explicit
       unchanged group re-sent after            resolved notification
         repeat_interval (default 4h)
```

Five behaviours in there decide whether your carefully derived alert is useful:

**`group_wait: 30s` is an inhibition race window, not just batching.** The Alertmanager documentation is explicit: the wait exists so that alerts from other rule groups or other Prometheus servers *and one or more inhibiting alerts* can arrive before the first notification goes out. Set it to `0s` "to page faster" and you will page for the GPU-goodput alert 20 seconds before the `NodeDown` alert that should have inhibited it arrives.

**`repeat_interval` is rounded up to a multiple of `group_interval`.** It is only checked at each `group_interval` tick. `repeat_interval: 4h` with `group_interval: 5m` behaves as written; `repeat_interval: 7m` with `group_interval: 5m` behaves as 10m.

**`group_interval` doubles as the notification-pipeline context timeout.** A slow receiver (a webhook that takes 90 s) under a small `group_interval` (30 s) has its notification *cancelled* mid-flight. If pages are silently not arriving and the receiver looks healthy, this is the first thing to check.

**Alerts resolve by timeout, not by the rule going quiet.** Prometheus re-sends a firing alert no more often than `--rules.alert.resend-delay` (default `1m`) and stamps each send with `EndsAt = now + 4 × max(group interval, resend-delay)` (`rules/alerting.go`: `alert.ValidUntil = ts.Add(4 * delta)`, commented as "allow for two Eval or Alertmanager send failures"). Alertmanager's `resolve_timeout` (default `5m`) governs alerts that arrive without any end time at all. So an alert whose Prometheus died stays firing for minutes, and an alert that genuinely cleared resolves a few evaluation intervals later. Do not build "the alert cleared" automation on sub-minute precision.

**HA is position-based waiting plus a gossiped notification log.** With three Alertmanagers, replica 0 sends immediately, replica 1 waits 15 s, replica 2 waits 30 s (`--cluster.peer-timeout`, default `15s`); each writes an nflog entry that gossips to the others, and the dedup stage drops the send if a peer already notified. Consequence: **a network partition between Alertmanager replicas produces duplicate pages, not missing ones** — a deliberate trade you should be able to name.

### 10. Routing and inhibition, as real config

The symptom-vs-cause rule ("page on user-facing SLI burn, ticket the causes") is only a principle until it is a routing tree. Here is one for a GPU platform, annotated on the lines that matter:

```yaml
route:
  receiver: platform-default
  group_by: [alertname, cluster, slo_id]   # NOT by instance: one incident, one page
  group_wait: 30s                          # inhibition race window (§9)
  group_interval: 5m
  repeat_interval: 4h

  routes:
    # ── 1. SLO burn pages. The only thing allowed to wake a human at night. ──
    - receiver: pagerduty-oncall
      matchers: [ severity="page", alert_type="slo-burn" ]
      group_by: [slo_id, cluster]          # per SLO, not per tier: the OR-of-tiers
                                           # expression already collapsed those
      repeat_interval: 1h                  # a live page re-nags hourly, not 4-hourly

    # ── 2. GPU capital-efficiency burn. Real money, but never a 3 a.m. page. ──
    - receiver: slack-gpu-platform
      matchers: [ severity="page", alert_type="goodput-burn" ]
      active_time_intervals: [ business-hours ]
      group_by: [tenant, cluster]
      # Outside business hours this route does not match, so the alert falls
      # through to the next sibling — deliberately, see route 3.

    - receiver: ticket-gpu-platform
      matchers: [ alert_type="goodput-burn" ]
      group_by: [tenant]
      repeat_interval: 12h

    # ── 3. Causes. Explain a page; never raise one. ──────────────────────────
    - receiver: ticket-platform
      matchers: [ severity=~"warning|ticket" ]
      group_interval: 30m
      repeat_interval: 24h

    # ── 4. Meta: the alerting system's own health. Pages, because a broken
    #        rule evaluator makes every other route silently correct-looking. ──
    - receiver: pagerduty-oncall
      matchers: [ alert_type="meta" ]
      group_by: [alertname]

time_intervals:
  - name: business-hours
    time_intervals:
      - weekdays: ['monday:friday']
        times: [{ start_time: '09:00', end_time: '18:00' }]
        location: 'Europe/London'          # without this, times are UTC

inhibit_rules:
  # A dead node's GPUs are not "underutilised"; they are dead. Mute the
  # derived complaint when the root cause is already paging.
  - name: node-down-mutes-gpu-signals
    source_matchers: [ alertname="NodeUnreachable" ]
    target_matchers: [ alert_type=~"goodput-burn|gpu-health" ]
    equal: [ cluster, node ]               # same node only — this is the whole
                                           # safety property of an inhibit rule

  # A cluster-wide control-plane outage mutes every per-tenant SLO page.
  - name: control-plane-mutes-tenant-slos
    source_matchers: [ alertname="ControlPlaneDown", severity="page" ]
    target_matchers: [ alert_type="slo-burn" ]
    equal: [ cluster ]

  # A page mutes the ticket for the same SLO: you already know.
  - name: page-mutes-ticket
    source_matchers: [ severity="page" ]
    target_matchers: [ severity="ticket" ]
    equal: [ slo_id, cluster ]
```

**How inhibition actually evaluates**, because getting this wrong produces the worst possible outcome (a muted page): an alert is muted if there exists *some other* firing alert matching `source_matchers` whose values for **every** label in `equal` are identical to the target's. A missing label and an empty label are the same thing — so if `node` is absent from both source and target, `equal: [cluster, node]` still matches on the absent label and inhibits **cluster-wide**. That is the classic inhibition accident: an inhibit rule intended to be node-scoped silently becomes global because the source alert never carried `node`. Alertmanager also refuses to let an alert inhibit itself when it matches both sides, but the documentation's advice is the real fix: **write source and target matchers that cannot both match the same alert.**

### 11. Composition: your SLO is not your users' SLO

A user request touches several services. Their SLOs compose, and the composition is not the number on anybody's dashboard.

**Serial dependencies multiply.** Five hops, each independently at 99.9%:

```
   0.999^5 = 0.99501   →  99.5 % end to end
   monthly bad-request budget: 0.5 % vs the 0.1 % each hop advertises — 5×.
```

To honestly promise 99.9% end-to-end across 5 serial hops, each hop needs `0.999^(1/5) = 0.99980` — **99.98% each**, a budget 5× tighter than the number they publish. This is why "every service is green and the product is not" is a structural outcome, not a mystery.

**Two corrections that make this less naive:**

1. **Independence is an assumption, and usually a wrong one.** Shared failure domains — one availability zone, one control plane, one config push — correlate failures. Correlation makes the composite *better* than the product during correlated failures (one outage takes out everything at once, costing one outage's worth of budget rather than five independent ones) and *worse* in the tail (one bad config push blows every budget simultaneously). Treat `∏SLO` as a **lower bound on the composite in the independent case**, and name the shared domains explicitly rather than pretending the multiplication is exact.
2. **Retries and fallbacks decouple you from your dependencies.** If your service retries a dependency's 500 once with a 20 ms budget and succeeds 90% of the time, your *observed* dependency failure rate is `0.001 × 0.1 = 0.0001` — you have bought back an order of magnitude with code, not with someone else's SLO. **This is why you alert on the SLI you control** (the outcome your service returned to its caller) rather than on the dependency's error rate. Alerting on a dependency's raw errors pages your team for another team's incident, and worse, pages you for incidents your own retry logic already absorbed.

Apply the same lens to a training job: its goodput SLO compounds the scheduler's admission latency, the network's collective bandwidth, and the storage layer's checkpoint throughput. A goodput target tighter than the loosest of those components is a promise you cannot keep, and the honest artefact is a dependency table with each component's own SLO next to the composite it implies.

### 12. GPU tie: an error budget denominated in GPU-hours

Now change the currency. For a training or inference fleet the thing being spent is not user patience, it is **capital**, and the SLI that measures it is goodput, not utilization.

**Why `GPU_UTIL` cannot be an SLI.** `DCGM_FI_DEV_GPU_UTIL` is a *presence* metric: it reports the fraction of a short driver sample window in which at least one kernel was resident on the device. A single spinning kernel, a busy-wait on a lock, or a NCCL all-reduce blocked on the slowest rank all pin it at 100 while producing approximately zero useful FLOPs. Module 05 derives this from the NVML counter semantics; the consequence *here* is that an SLI built on it has an error budget that can never be spent, because the metric is structurally unable to represent the failure. (`DCGM_FI_PROF_SM_ACTIVE` — the ratio of cycles an SM had at least one warp resident — is the honest occupancy signal, and even it measures occupancy rather than productivity.)

**The SLI.** Define goodput as delivered work over expected work:

```
   goodput_ratio(t) = achieved_tokens_per_sec(t) / expected_tokens_per_sec
   bad_fraction(t)  = 1 − goodput_ratio(t)        ← this is the "error" ratio
```

`expected_tokens_per_sec` comes from the run's own measured healthy baseline (a recorded rule pinned at job-submit time), not from a hardware spec sheet. That distinction matters: **MFU** (achieved FLOPs ÷ peak FLOPs) measures how well the model and kernels use the silicon and is legitimately low for memory-bound work; **goodput** measures whether you are getting what this job, on this hardware, has already demonstrated it can produce. Alert on goodput; chart MFU.

**The budget, in GPU-hours.** For a 512-GPU pool with a policy that at most 3% of paid GPU-time may be wasted in a month:

```
   pool GPU-hours / 30 d      = 512 × 720                  = 368,640 GPU-h
   waste budget β = 3 %       = 368,640 × 0.03             =  11,059 GPU-h
   at a $2.50/GPU-hour rate   = 11,059 × 2.50              ≈ $27,650 / month
   sustainable waste rate     = 11,059 / 720               ≈  15.4 GPU-h wasted/hour
                              = 15.4 / 512                 ≈  3 % of the fleet, always
```

The burn-rate machinery transfers unchanged, because burn rate is dimensionless: burn 14.4 means "at this rate the month's 11,059 wasted-GPU-hour allowance is gone in 2.08 days," i.e. wasting `15.4 × 14.4 ≈ 222` GPU-hours per hour — which on a 512-GPU pool means **43% of the fleet is producing nothing.** That is a page. Burn 6 means 18% of the fleet idle-but-paid: a business-hours page. Burn 1 is the policy line itself.

**Integrating it correctly.** Wasted GPU-hours is an integral of a ratio over time, and the query that looks obvious is wrong:

```promql
# ✗ WRONG — inflates any job that was not running for the whole window.
#   avg_over_time averages only over samples that EXIST, so a job present for
#   6 h of a 24 h window gets its 6-hour mean multiplied by 24: a 4× overstatement.
sum by (tenant) (avg_over_time(gpu:goodput_ratio:5m[24h])) * 24

# ✓ CORRECT — a Riemann sum. Absent samples contribute zero, which is exactly
#   the semantics of "the job was not running".
#   The constant 30/3600 is the rule group's `interval:` in hours. If you change
#   `interval:`, you MUST change this constant.
sum by (tenant) (sum_over_time(gpu:wasted_fraction:ratio[24h])) * 30 / 3600
```

This is the same correction module 05 makes for allocated-vs-utilised GPU-hours, and it bites *harder* in an alerting context than in a report: an inflated waste number produces a page for a job that behaved perfectly and merely finished early.

**And the room it gets presented in is different.** A user-facing burn rate is argued in terms of user pain and reviewed by on-call. A wasted-GPU-hours burn rate is argued in terms of capital efficiency and reviewed by whoever signs the cluster's purchase order. Same algebra, same rules engine, different audience, different escalation path — which is exactly why route 2 in §10 sends it to Slack in business hours and a ticket otherwise, rather than to the pager.

## Perspectives

**Statistical.** Everything about low-traffic alerting is one fact: the ratio estimator's variance is `p(1−p)/n`, so the *count* of expected bad events at threshold is the quantity that decides whether an alert is meaningful. `B·β·n ≥ ~8` is not a rule of thumb pulled from nowhere; it is the point at which the Poisson tail stops producing pages. A staff engineer should be able to derive the request-rate floor for any SLO on a whiteboard in two lines, and should recognise that tighter SLOs make the low-traffic problem worse.

**Design-of-the-alert-set.** The tiers are a *coverage* structure, not a *speed* structure. Draw the burn-rate axis, mark the exhaustion times, and check that every rate you would care about is covered by some tier. Most broken alert sets are broken in the gap between the fast page and nothing — a 5–14× burn with no owner — and the fix is a slow tier, not a lower fast threshold.

**Operational.** Half of an alert's behaviour lives in Alertmanager, not in the rule. `group_wait` decides whether inhibition works; `group_interval` decides whether slow webhooks deliver at all; `repeat_interval` rounding decides how often the page re-nags; `resolve_timeout` decides how long a dead Prometheus keeps an alert alive. A design review that stops at the PromQL has reviewed maybe 60% of the system.

**Org-process.** Burn-rate alerting is necessary and not sufficient. The math tells you when to page; it says nothing about what happens after. Without a governed policy for budget exhaustion — feature freeze, mandatory reliability work, an explicit accepted-risk sign-off — MWMBR is a well-tuned threshold alert wearing a budget costume. The one artefact that makes it real is a written answer to "who decides, and what changes, when the budget hits zero on the 12th?"

**GPU-economics.** Denominating the budget in GPU-hours and then in dollars changes both the audience and the tuning. A 3% waste policy on a 512-GPU pool is $27.6k/month of deliberately accepted slack; whether the fast tier should be 14.4 (43% of the fleet idle) or 6 (18%) is a conversation with finance about how much money may burn before someone is woken, and it is a conversation they can actually have because the units are theirs.

## Real-world use cases

- **Sloth's window catalogue as an executable specification** (`slok/sloth`, `internal/alert/windows/google-30d.yaml`, `google-28d.yaml`, `internal/alert/window.go`). The project encodes the four tiers as *budget percentages plus windows* and computes the multiplier at runtime with `getBurnRateFactor`. **What it shows:** the industry's own tooling treats 14.4 as derived output, not input — and ships a 28-day catalogue whose multipliers (13.44 / 5.6 / 2.8 / 0.933) are visibly different, which is the cleanest possible proof that the famous numbers are period-specific.

- **The generated alert expression shape** (`slok/sloth`, `internal/plugin/slo/core/alert_rules_v1/plugin.go`). Two tiers ORed inside one alert, each tier an AND of short and long, with `max(...) without (sloth_window)` to make the vector match work. **What it shows:** the label-stripping detail that hand-rolled implementations forget; the failure mode is an alert that never fires and looks perfectly healthy in the rules UI.

- **Alertmanager's notification pipeline** (`prometheus/alertmanager`, `notify/notify.go` — `MultiStage{gossip-settle, inhibit, time-active, time-mute, silence, [cluster-wait, dedup, retry, set-notifies]}`). **What it shows:** silences are applied *after* inhibition and *after* time intervals, and dedup happens per-integration against a gossiped notification log — so "why did I get two pages" and "why did I get none" are questions with different answers at different stages, and you need the stage order to debug either.

- **The `group_wait` / inhibition race, documented in the config reference** (`prometheus/alertmanager`, `docs/configuration.md`): the wait exists partly so that inhibiting alerts have time to arrive. **What it shows:** an operational default (30 s) that is really a correctness mechanism. Teams that shorten it to reduce page latency reintroduce the noise that their inhibit rules were written to remove, and the symptom (occasional spurious pages during node failures) is very hard to trace back to the change.

- **OpenTelemetry GenAI semantic conventions' explicit histogram boundaries** (`open-telemetry/semantic-conventions-genai`, `docs/gen-ai/gen-ai-metrics.md`): `gen_ai.server.time_to_first_token` specifies `[0.001, 0.005, 0.01, 0.02, 0.04, 0.06, 0.08, 0.1, 0.25, 0.5, 0.75, 1.0, 2.5, 5.0, 7.5, 10.0]`. **What it shows:** a standards body choosing bucket boundaries so that common SLO thresholds land exactly on them. If your latency SLO threshold is not a boundary in your histogram, your SLI is an interpolation and your budget is fiction.

## Worked example

**Scenario.** The `api` service: 20,000,000 requests per 30 days, SLO 99.9% availability, rolling 30-day window. Alongside it, a 512-GPU training pool with a 3%-of-paid-time waste policy. Build the complete alert set, then trace one incident through it end to end.

### Step 1 — derive every threshold from the two policy inputs

```
   INPUTS (the only two things that are a choice)
     SLO_window      = 30 d = 720 h
     β (availability)= 1 − 0.999 = 0.001
     β (GPU waste)   = 0.03 of paid GPU-time

   DERIVED THRESHOLDS   burn = budget_fraction × 720 / alert_window_hours
     page-quick   2 % / 1h   → 14.4   → ratio threshold 14.4 × 0.001 = 0.0144
     page-slow    5 % / 6h   →  6     → ratio threshold  6   × 0.001 = 0.0060
     ticket-quick 10 % / 1d  →  3     → ratio threshold  3   × 0.001 = 0.0030
     ticket-slow  10 % / 3d  →  1     → ratio threshold  1   × 0.001 = 0.0010

   TRAFFIC FLOOR (§8)   n ≥ 8 / (B·β) = 8 / 0.0144 = 555 requests per 5 min
     mean traffic is 7.72 req/s = 2,316 per 5 min → 4.2× the floor. Fine.
     But the 03:00–05:00 trough runs at ~1.1 req/s = 330 per 5 min → BELOW it.
     ⇒ the floor must be an explicit clause, not an assumption.

   GPU BUDGET
     paid GPU-h/30d = 512 × 720 = 368,640
     waste budget   = 11,059 GPU-h  ($27.6k at $2.50/GPU-h, rate-card snapshot)
     sustainable    = 15.4 wasted GPU-h per hour = 3 % of the pool, continuously
     page-quick     = 14.4 × 15.4 ≈ 222 wasted GPU-h/h  ≈ 43 % of the pool idle
     page-slow      =  6   × 15.4 ≈  92 wasted GPU-h/h  ≈ 18 % of the pool idle
```

### Step 2 — the rule file, complete

```yaml
groups:
  # ─────────────────────────────────────────────────────────────────────────
  # SLI recording rules. Group interval 30s so the 5m window has 10 samples.
  # ─────────────────────────────────────────────────────────────────────────
  - name: slo-api-availability-sli
    interval: 30s
    labels: { sloth_id: api-availability, sloth_service: api }
    rules:
      - record: slo:sli_error:ratio_rate5m
        expr: |
          sum(rate(http_requests_total{job="api",code=~"5.."}[5m]))
          / sum(rate(http_requests_total{job="api"}[5m]))
        labels: { sloth_window: 5m }
      - record: slo:sli_error:ratio_rate30m
        expr: |
          sum(rate(http_requests_total{job="api",code=~"5.."}[30m]))
          / sum(rate(http_requests_total{job="api"}[30m]))
        labels: { sloth_window: 30m }
      - record: slo:sli_error:ratio_rate1h
        expr: |
          sum(rate(http_requests_total{job="api",code=~"5.."}[1h]))
          / sum(rate(http_requests_total{job="api"}[1h]))
        labels: { sloth_window: 1h }
      - record: slo:sli_error:ratio_rate6h
        expr: |
          sum(rate(http_requests_total{job="api",code=~"5.."}[6h]))
          / sum(rate(http_requests_total{job="api"}[6h]))
        labels: { sloth_window: 6h }
      - record: slo:sli_error:ratio_rate2h
        expr: |
          sum(rate(http_requests_total{job="api",code=~"5.."}[2h]))
          / sum(rate(http_requests_total{job="api"}[2h]))
        labels: { sloth_window: 2h }
      - record: slo:sli_error:ratio_rate1d
        expr: |
          sum(rate(http_requests_total{job="api",code=~"5.."}[1d]))
          / sum(rate(http_requests_total{job="api"}[1d]))
        labels: { sloth_window: 1d }
      - record: slo:sli_error:ratio_rate3d
        expr: |
          sum(rate(http_requests_total{job="api",code=~"5.."}[3d]))
          / sum(rate(http_requests_total{job="api"}[3d]))
        labels: { sloth_window: 3d }

      # Denominator, recorded once so the traffic floor is cheap to evaluate
      # and visible on a dashboard next to the alert it guards.
      - record: slo:requests:rate5m
        expr: sum(rate(http_requests_total{job="api"}[5m]))

      # 30d budget consumption for the dashboard (§7 — sum/count, not rate[30d]).
      - record: slo:sli_error:ratio_rate30d
        expr: |
          sum_over_time(slo:sli_error:ratio_rate5m{sloth_id="api-availability"}[30d])
          / ignoring (sloth_window)
          count_over_time(slo:sli_error:ratio_rate5m{sloth_id="api-availability"}[30d])
        labels: { sloth_window: 30d }
      - record: slo:budget_remaining:ratio
        expr: 1 - (slo:sli_error:ratio_rate30d{sloth_id="api-availability"} / 0.001)

  # ─────────────────────────────────────────────────────────────────────────
  # Alerting rules. Same group as nothing else: these must not be skipped.
  # ─────────────────────────────────────────────────────────────────────────
  - name: slo-api-availability-alerts
    interval: 30s
    rules:
      - alert: ApiAvailabilityBudgetBurnPage
        expr: |
          (
              max(slo:sli_error:ratio_rate5m{sloth_id="api-availability"} > (14.4 * 0.001)) without (sloth_window)
              and
              max(slo:sli_error:ratio_rate1h{sloth_id="api-availability"} > (14.4 * 0.001)) without (sloth_window)
              and on() slo:requests:rate5m > 1.85
          )
          or
          (
              max(slo:sli_error:ratio_rate30m{sloth_id="api-availability"} > (6 * 0.001)) without (sloth_window)
              and
              max(slo:sli_error:ratio_rate6h{sloth_id="api-availability"} > (6 * 0.001)) without (sloth_window)
              and on() slo:requests:rate5m > 1.85
          )
        for: 2m                       # ride out a single bad evaluation; costs 2 min
        labels:
          severity: page
          alert_type: slo-burn
          slo_id: api-availability
        annotations:
          summary: "api availability budget burning fast ({{ $value | humanize }} bad ratio)"
          description: >-
            Burn ≥ 6× on 6h/30m or ≥ 14.4× on 1h/5m. At 14.4× the 30-day budget
            (20,000 bad requests) is exhausted in ~2.1 days.
          runbook: "https://runbooks.internal/slo/api-availability"

      - alert: ApiAvailabilityBudgetBurnTicket
        expr: |
          (
              max(slo:sli_error:ratio_rate2h{sloth_id="api-availability"} > (3 * 0.001)) without (sloth_window)
              and
              max(slo:sli_error:ratio_rate1d{sloth_id="api-availability"} > (3 * 0.001)) without (sloth_window)
          )
          or
          (
              max(slo:sli_error:ratio_rate6h{sloth_id="api-availability"} > (1 * 0.001)) without (sloth_window)
              and
              max(slo:sli_error:ratio_rate3d{sloth_id="api-availability"} > (1 * 0.001)) without (sloth_window)
          )
        for: 15m
        labels: { severity: ticket, alert_type: slo-burn, slo_id: api-availability }

  # ─────────────────────────────────────────────────────────────────────────
  # GPU goodput. Same machinery, budget denominated in wasted GPU-hours.
  # ─────────────────────────────────────────────────────────────────────────
  - name: slo-gpu-goodput
    interval: 30s                    # ← the 30/3600 integration constant below
    rules:
      - record: gpu:goodput_ratio:5m
        expr: |
          sum by (tenant, job) (rate(training_tokens_total[5m]))
          / on (tenant, job) group_left job:expected_tokens_per_sec
      - record: gpu:wasted_fraction:ratio
        expr: clamp_min(1 - gpu:goodput_ratio:5m, 0)

      # Wasted GPU-hours per hour, as a burn rate against 15.4 GPU-h/h.
      # sum_over_time × interval/3600 = the Riemann sum (§12). NOT avg_over_time.
      - record: gpu:waste_burn:rate1h
        expr: |
          (
            sum(sum_over_time(gpu:wasted_fraction:ratio[1h]) * on() group_left() (gpus_allocated))
            * 30 / 3600
          ) / 15.4
      - record: gpu:waste_burn:rate5m
        expr: |
          (
            sum(sum_over_time(gpu:wasted_fraction:ratio[5m]) * on() group_left() (gpus_allocated))
            * 30 / 3600 * 12
          ) / 15.4

      - alert: GpuWasteBudgetBurnFast
        expr: gpu:waste_burn:rate1h > 14.4 and gpu:waste_burn:rate5m > 14.4
        for: 10m                     # checkpoint pauses are minutes; do not page on them
        labels: { severity: page, alert_type: goodput-burn }
        annotations:
          summary: "GPU waste burning 14.4× — ≈43% of the 512-GPU pool producing nothing"
          description: >-
            At this rate the month's 11,059 wasted-GPU-hour budget (≈ $27.6k)
            is gone in ~2.1 days. Check straggler ranks and the goodput panel
            before checking utilisation, which will look fine.
```

### Step 3 — trace one incident through the whole path

A bad deploy at **10:00:00** takes the `api` error rate from 0.02% to 6% (burn rate `0.06 / 0.001 = 60`).

```
   10:00:00  deploy lands. r = 6 %, b = 60.
   10:00:30  first evaluation after the change. 5m window holds 30 s of badness:
             observed ratio = 0.06 × 30/300 = 0.006 → below 0.0144. No fire.
   10:01:12  5m window crosses:  t = 300 × 14.4/60 = 72 s.
   10:14:24  1h window crosses:  t = 3600 × 14.4/60 = 864 s = 14.4 min.
             ⇒ both legs of the fast tier true. Alert enters PENDING.
   10:16:24  `for: 2m` satisfied → FIRING. ALERTS{alertstate="firing"} appears.
   10:16:30  next evaluation pushes the alert to Alertmanager.
   10:17:00  group_wait (30 s) elapses; inhibit/silence/time stages pass;
             cluster-wait = 0 s on replica 0; dedup finds no prior nflog entry.
   10:17:01  PagerDuty receives it. TOTAL: 17 min 1 s from deploy to page.
             Budget spent by then: 0.06 × 7.72 req/s × 1,021 s ≈ 473 bad requests
                                  = 2.4 % of the 20,000-request monthly budget.

   ── the slow tier, in parallel ──
   10:07:12  30m window crosses 0.006:  t = 1800 × 6/60 = 180 s.
   11:00:00  6h window crosses 0.006:   t = 21600 × 6/60 = 2,160 s = 36 min.
             ⇒ the slow tier would have paged at 11:00, 43 min later, having
               spent 5 % of the budget. The fast tier is what bought the 43 min.

   ── rollback at 10:25:00 (25 min of badness, d = 1,500 s) ──
   10:25:00  error rate returns to baseline.
   10:29:16  5m leg drops:  s = 300 − 300×14.4/60 = 228 s after recovery.
             ⇒ AND breaks; alert stops firing at the next evaluation (~10:29:30).
   10:34:30  Alertmanager sends the RESOLVED notification (resolve_timeout /
             explicit EndsAt; do not expect it to be instant).
             Had the alert used only the 1h window, it would have kept firing
             until s = 1500 − 864 = 636 s → 10:35:36, plus resolve lag.
   ── aftermath ──
             Budget spent in total: 0.06 × 7.72 × 1,500 ≈ 695 bad requests
                                    = 3.5 % of the month. Budget remaining 96.5 %.
             No feature freeze triggered. This is what a working alert set
             looks like: one page, one incident, three-and-a-half percent.
```

### Step 4 — the counterfactual that justifies each design choice

| Change one thing | What happens to this incident |
|---|---|
| Drop the 5m leg (long window only) | Page at 10:14:24 (2 min earlier), but keeps firing until 10:35:36 — 6 extra minutes of a firing page after a completed rollback, and any 90-second blip at `b > 14.4` also pages. |
| Drop the 1h leg (short window only) | Page at 10:01:12, 16 min earlier — and every burst of 5xx from a rolling restart pages too. Precision collapses; this is the single-window failure. |
| Only the fast tier | A 10× burn (exhausts the month in 3 days) is never detected. |
| `for: 15m` instead of `2m` | Page at 10:29:24 — after the rollback. The `for` clause is detection delay you pay unconditionally; keep it just long enough to survive one bad evaluation. |
| No traffic floor | At 03:30 (1.1 req/s ≈ 330 requests per 5m), 3 errors would produce ratio 0.009… below 0.0144, so not this time — but 5 errors (`λ = 0.33`, P ≈ 0.5%) would, roughly 4 times a month. |
| `group_wait: 0s` | The page arrives 30 s earlier and before the `ControlPlaneDown` inhibitor during correlated outages, producing a page storm exactly when it hurts most. |

## Practice

<feeds [fleet observability design](../practice/fleet-observability/README.md)>

Build the alerting layer of the fleet observability design. Deliverable is deployable YAML plus a one-page design note; the note is what gets marked.

1. **Derive, do not copy.** Pick your SLO period (justify 30d vs 28d vs calendar month) and produce the multiplier table from `burn = budget_fraction × SLO_window / alert_window` for all four tiers. Show the arithmetic. If you choose a non-standard budget-fraction set (say 1% / 4% / 10%), state why in one sentence.
2. **Write the coverage diagram.** Mark the burn-rate axis with your tiers' thresholds and the exhaustion time each implies. Identify explicitly which burn-rate band is covered by which tier and confirm there is no gap between the fastest ticket tier and the slowest page tier.
3. **Compute the detection and reset table** for your fastest tier: `t_detect = W·B/b` for `b ∈ {1000, 100, 30, B}` and `s_reset` for a 10-minute and a 45-minute incident, with and without the short window. This is the evidence that the AND earns its place.
4. **Derive the traffic floor** for your service's SLO from `n ≥ 8/(B·β)`, convert it to a request rate, and check it against your *trough* traffic, not your mean. If the trough is below the floor, choose and justify one of the five fixes in §8 — including "do not alert."
5. **Ship the rule file**: recording rules for every window with a window label, the page alert as one expression with both tiers ORed, a ticket alert, the traffic-floor clause, and the budget-remaining series. Validate with `promtool check rules`, and unit-test at least the fast tier with `promtool test rules` (an input series at `b = 60` should fire at the derived time, ±one evaluation interval).
6. **Ship the Alertmanager config**: routing tree with an SLO-burn page route, a GPU goodput route that reaches humans only in business hours, a cause/ticket route, and a meta route for `rule_group_iterations_missed_total`. Include at least two inhibit rules and, for each, state the `equal` labels and what happens if one of them is absent from the source alert.
7. **The GPU budget in money.** For your fleet size, compute paid GPU-hours/month, the waste budget at your chosen percentage, the dollar figure at a stated rate, and what fraction of the fleet must be idle to hit each tier's threshold. Write the goodput burn rules using `sum_over_time(...) * interval/3600` and put the integration constant in a comment next to the group's `interval:`.
8. **The governance paragraph.** One paragraph naming who decides what happens when the budget is exhausted, what specifically changes, and who can override. Without this the rest is a dashboard.

**Acceptance criteria:** every threshold in the artefact traceable to the derivation; no multiplier quoted without its period; a stated traffic floor with its arithmetic; `promtool check rules` clean; every inhibit rule's `equal` set justified; the GPU integral written with `sum_over_time`.

## Common pitfalls

- **"14.4 is a constant."** It is `2% × 720h / 1h`. On a 28-day period it is 13.44; on a 7-day period it is 3.36. Symptom: an alert set copied onto a weekly SLO that never fires, because every threshold is 4.3× too high. Mechanism: the multiplier is inversely proportional to the alert window and directly proportional to the SLO period; changing either without re-deriving breaks it silently, and "silently" is the problem — nothing errors, the alert simply becomes decorative.
- **"Burn rate 14.4 means 14.4% of budget consumed."** It is a multiplier on the sustainable spend rate. The consumed fraction needs the full identity: `14.4 × 1h/720h = 2%`. Symptom: a status page claiming an incident cost 14% of the month when it cost 2%.
- **Forgetting to strip the window label before the `and`.** The two sides of the join have different `sloth_window` values, the vector match returns empty, and the alert never fires. Symptom: a rule that shows healthy in the UI forever and never pages during a real outage. Mechanism: PromQL's `and` requires identical label sets on both sides; `max(...) without (sloth_window)` is the fix.
- **A `for:` clause long enough to defeat the point.** `for: 15m` on a tier whose whole purpose is a ~1h detection budget adds 25% to detection unconditionally, in exchange for surviving a single flapping evaluation. Use 2–5 minutes. If you need 15 minutes of confirmation, your short window is too short.
- **Setting `group_wait: 0s` to page faster.** You lose the window in which inhibiting alerts arrive (this is the documented purpose of the wait), so correlated failures produce page storms. Symptom: pages are fine in normal operation and catastrophic during node or AZ failures — the exact moment on-call is most loaded.
- **Inhibit rules whose `equal` labels are absent from the source alert.** A missing label and an empty label are treated as the same, so a node-scoped inhibition silently becomes cluster-wide. Symptom: a real page never arrives during an unrelated node failure. This one is dangerous precisely because its failure mode is silence.
- **`avg_over_time` for GPU-hours.** Averaging over only the samples that exist and multiplying by the window over-states any job that did not run for the whole window — 4× for a 6-hour job in a 24-hour window. Symptom: a page for a job that finished early and behaved perfectly. Fix: `sum_over_time(...) * interval/3600`.
- **A burn-rate alert with no traffic floor on a low-traffic endpoint.** Below `n = 1/(B·β)` requests per window (69 for 99.9%/14.4) a single error is a page. Symptom: an internal admin endpoint that pages twice a week at 4 a.m. Fix: the floor, a longer window, a window-based SLI, or no alert at all.
- **Treating a burn-rate page as "the SLO is breached."** It is a leading indicator: at 14.4× you have spent 2% and have about two days. Symptom: incident reviews that record "SLO breached" for every page, which destroys the credibility of the budget as a decision input.

## Self-check

- **Derive the page-quick multiplier for a 7-day SLO period with a 1-hour long window and a 2% budget allowance. Then say what a team using 14.4 instead would miss.** — `burn = 0.02 × 168h / 1h = 3.36`. A team using 14.4 fires only at 4.3× that rate; a sustained 5× burn — which exhausts the week's budget in 1.4 days — never trips the page tier at all, and the ticket tiers derived from the same wrong period are equally inflated, so nothing catches it.

- **Ticket-quick (1d window, 3×) and ticket-slow (3d window, 1×) detect a total outage at exactly the same moment, despite windows differing by 3×. Why — and what does that tell you the window is actually for?** — Detection time is `t = W·B/b`, and the threshold is itself `B = budget_fraction × SLO_window / W`, so `t = budget_fraction × SLO_window / b`: **the window cancels.** Both ticket tiers allow 10% of budget, so both detect after `0.10 × 720h / b = 72h/b` — 259 s at `b = 1000`. Detection time is set by the budget fraction alone. The window therefore is not a speed control; it is a noise control. Longer window ⇒ proportionally lower threshold ⇒ same detection time, more events in the denominator (better statistics), and a longer reset lag. Choose the window for signal quality and reset behaviour, and choose the budget fraction for how much you are willing to lose before being told.

- **Your fast tier is `1h/5m` at 14.4×. An incident runs 40 minutes at burn rate 30, then stops. When does it fire, and when does it clear — with, and without, the 5-minute leg?** — Fire: the 5m leg crosses at `300 × 14.4/30 = 144 s`; the 1h leg at `3600 × 14.4/30 = 1,728 s` = 28.8 min; the AND fires at 28.8 min (plus `for:`). Clear, with the short leg (`d = 2,400 s > W_short`): `s_reset = 300 − 300×14.4/30 = 156 s` ≈ 2.6 min after recovery. Clear, long leg only: `s_reset = 2400 − 1728 = 672 s` ≈ 11.2 min after recovery. The short leg is worth 8.6 minutes of not-firing-at-a-recovered-service, plus immunity to short bursts that the long window would still be carrying.

- **A 99.99%-SLO internal endpoint sees 1.5 requests/second. Is a 14.4× fast-burn alert on a 5-minute window meaningful? Show the numbers.** — No. `n = 1.5 × 300 = 450` requests per window; `β = 0.0001`; the single-error line is `1/(14.4 × 0.0001) = 694` requests, so at `n = 450` **one error** produces a ratio of `1/450 = 0.0022` = 22× burn and pages. The expected bad count at threshold is `B·β·n = 0.65` — far below the ~8 needed for the Poisson tail to be negligible. Fixes in order of preference: aggregate this endpoint into a larger SLO, switch to a window-based SLI, lengthen the windows by roughly the ratio `8/0.65 ≈ 12×`, or do not alert and report budget consumption weekly.

- **Why is `avg_over_time(goodput[24h]) * 24` wrong for wasted GPU-hours, and what does it do to your alerting specifically?** — `avg_over_time` averages only over samples that exist. A job present for 6 of 24 hours yields its 6-hour mean, and multiplying by 24 inflates the result 4×. In an alerting context that inflation is applied to the *numerator of a burn rate*, so a well-behaved job that finished early produces a burn rate 4× its true value and pages. The correct form is the Riemann sum `sum_over_time(x[24h]) * Δ/3600` where `Δ` is the rule group's evaluation interval; absent samples then contribute zero, which is exactly the semantics of "not running."

- **In Alertmanager, at which stage is a silence applied, and why does the order matter?** — The pipeline is `gossip-settle → inhibit → time-active → time-mute → silence → [cluster-wait → dedup → retry → set-notifies]`. Silences are applied *after* inhibition and time intervals, so an alert already muted by an inhibit rule never reaches the silence stage and never appears as "silenced" in the UI — it appears as inhibited. Debugging "why no page" therefore requires checking the stages in order: inhibition first (most common and most silent), then time intervals, then silences, then the notification log for a dedup drop.

- **Five services each meet 99.9%. What is the honest end-to-end number, and name one reason the multiplication is optimistic and one reason it is pessimistic.** — `0.999^5 ≈ 99.5%`, i.e. 5× the per-hop budget. Pessimistic: retries and fallbacks in your own service absorb a large fraction of a dependency's failures, so the *observed* dependency failure rate can be an order of magnitude below its published SLO. Optimistic: the hops share failure domains (one AZ, one control plane, one config push), so failures correlate; a single correlated event blows all five budgets at once, which the independent-product model does not represent at all.

## Connections & what's next

Burn-rate alerting is where the correctness work from [01 · The signal model](01-signal-model.md) through [06 · Logging pipelines](06-logging-pipelines.md) gets consumed — a `rate()`-on-a-gauge trap or a mislabeled log field shows up here as either a missed page or a false one, and the recording-rule architecture in §7 is lesson 02's scaling primitive applied to alerting. The traffic floor in §8 is the same "know your denominator" discipline that lesson 02's histogram traps teach, arriving from the statistics side.

Once an alert fires correctly, the next move is finding out *why* — that's [08 · Continuous profiling and eBPF](08-profiling-and-ebpf.md), the escalation rung from "the SLI is burning budget" to "which stack is burning the cycles or the wait time." The goodput SLI defined in §12 is scaled to thousands of GPUs, with straggler detection and per-tenant budgets, in [09 · GPU and ML observability at fleet scale](09-gpu-and-ml-observability.md); and the question "what did this budget cost us in dollars, by team, last quarter" is deliberately out of reach of everything here — that is [10 · The telemetry lakehouse](10-telemetry-lakehouse.md).

Checkpoint item 2 is this lesson stated as a pass criterion: derive the tiers and the multiplier math cold, without the table.

## References & further reading

**Primary sources (read from upstream repositories; the rendered docs sites are unreachable from this environment)**
- Prometheus — alerting rules reference: `prometheus/prometheus`, `docs/configuration/alerting_rules.md` (`for`, `keep_firing_for`, the `ALERTS` synthetic series, templating). *Verified defaults: `for: 0s`, `keep_firing_for: 0s`.*
- Prometheus — recording rules reference: `prometheus/prometheus`, `docs/configuration/recording_rules.md` (group `interval`, `limit`, `query_offset`, sequential evaluation with a shared timestamp, skipped-iteration behaviour and `rule_group_iterations_missed_total`).
- Prometheus — configuration reference: `prometheus/prometheus`, `docs/configuration/configuration.md`. *Verified: `global.evaluation_interval` default `1m`, `global.scrape_interval` default `1m`, `rule_query_offset` default `0s`.*
- Prometheus — rule-manager source: `prometheus/prometheus`, `rules/alerting.go` and `cmd/prometheus/main.go`. *Verified: `alert.ValidUntil = ts.Add(4 * max(group interval, resendDelay))` sent as `EndsAt`, with the source comment "allow for two Eval or Alertmanager send failures"; `--rules.alert.resend-delay` default `1m`; `--rules.alert.for-grace-period` default `10m`.*
- Alertmanager — configuration reference: `prometheus/alertmanager`, `docs/configuration.md`. *Verified: `group_wait: 30s`, `group_interval: 5m`, `repeat_interval: 4h`, `resolve_timeout: 5m`, `repeat_interval` rounding, `group_interval` as notification context timeout, inhibit-rule `equal` semantics including the missing-label-equals-empty rule.*
- Alertmanager — notification pipeline source: `prometheus/alertmanager`, `notify/notify.go`. *The stage order in §9's diagram is read directly from `PipelineBuilder.New` and `createReceiverStage`.*
- Alertmanager — cluster flags: `prometheus/alertmanager`, `cmd/alertmanager/main.go`. *Verified: `--cluster.peer-timeout` default `15s`, which is the per-position notification delay in HA.*
- Sloth — burn-rate derivation and window catalogue: `slok/sloth`, `internal/alert/window.go` (`getBurnRateFactor`), `internal/alert/windows/google-30d.yaml`, `internal/alert/windows/google-28d.yaml`. *The 2/5/10/10 budget percentages and 5m:1h, 30m:6h, 2h:1d, 6h:3d window pairs are quoted from these files; every multiplier in this lesson is computed from them rather than copied.*
- Sloth — generated alert expression: `slok/sloth`, `internal/plugin/slo/core/alert_rules_v1/plugin.go` (`mwmbAlertTpl`), and SLI rule generation including the `sum_over_time / count_over_time` long-window optimisation: `internal/plugin/slo/core/sli_rules_v1/plugin.go`.
- OpenTelemetry — GenAI semantic conventions: `open-telemetry/semantic-conventions-genai`, `docs/gen-ai/gen-ai-metrics.md`. *Source of the explicit histogram bucket boundaries quoted in §6.*

**Not relied upon (unreachable from this environment)**
- Google SRE Workbook — "Alerting on SLOs": https://sre.google/workbook/alerting-on-slos/ — the original source of the 2%/5%/10% budget-fraction policy and the recommended window pairs. **Blocked by the egress proxy here; not fetched.** The parameters used in this lesson were taken from Sloth's embedded catalogue, which cites this page as its source, and every multiplier was re-derived rather than quoted.
- Google SRE Book — "Service Level Objectives": https://sre.google/sre-book/service-level-objectives/ — **blocked; not fetched.**

**Real-world engineering writing (cited from prior reading, not fetched here)**
- Grafana Labs — implementing multi-window multi-burn-rate alerts with Grafana Cloud: https://grafana.com/blog/how-to-implement-multi-window-multi-burn-rate-alerts-with-grafana-cloud/ — an operational walkthrough of turning the tiers into deployable rules and SLO objects.
- Datadog — "Burn rate is a better error rate": https://www.datadoghq.com/blog/burn-rate-is-better-error-rate/ — argues burn rate's real value as a *normalised* unit comparable across services with different SLOs, which is the §2 point restated commercially.
