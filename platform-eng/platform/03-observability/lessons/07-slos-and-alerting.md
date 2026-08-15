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
sources: 5
---

# A03.7 · SLOs and alerting

> **Concept.** Multi-window multi-burn-rate alerting is the noise/detection Pareto point — and for GPU fleets the SLI must be goodput, not utilization.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

Lessons 01–06 built the signal stack — metrics, traces, logs — and the correctness traps in each. This lesson is where those signals get *consumed*: alerting is the layer that turns a correct signal into a decision to wake a human (or not). Get the burn-rate math wrong here and every upstream correctness fix (cardinality discipline, tail-sampling, log demotion) is wasted, because the alert either drowns it in noise or misses it entirely. Once an alert fires correctly, the next question is *why* — that's lesson 08, which picks up exactly where this one stops: from "the SLI is burning budget" to "which stack is burning the cycles or the wait time."

## Why this matters

At staff scale the question is no longer "do we have SLOs" but "why does this alert set page at exactly the right moment and stay silent through a transient blip." Getting burn-rate math right is the difference between a fleet that pages 4000 nodes into fatigue and one that pages once, precisely, two days before budget exhaustion. And for GPU fleets the standard playbook actively misleads you: a utilization gauge at 100% can be burning wasted-GPU-hours budget while every dashboard looks green. This lesson is the design math and the GPU reframe, not the vocabulary.

## What's new here (calibration)

- **Skip:** SLI/SLO/error-budget definitions, symptom-over-cause as a principle, "page fatigue is bad," naive fixed-threshold alerts.
- **Push depth on:** why 14.4/6/1 are calibrated constants (not derived math) and how to justify your own ratio; the ratio-estimator variance problem underneath the low-traffic caveat; multi-service SLO composition; and the GPU reframe from "user pain tolerance" to "capital efficiency tolerance."

## Core concepts

### Burn rate, defined cold

Burn rate = the rate you are spending error budget relative to the rate that would exhaust it *exactly* at period end. Burn rate 1 = on-pace: you finish the 30-day window having consumed precisely 100% of budget. Burn rate 14.4 = you would exhaust a 30-day budget in ~2 days (30 / 14.4 ≈ 2.08). The core identity:

```
budget_fraction_consumed = burn_rate × (alert_window / SLO_window)
```

So 14.4× sustained over a 1h window against a 30-day SLO consumes 14.4 × (1h / 720h) ≈ 2% of budget in that hour.

### The canonical MWMBR tiers (30-day budget) — and why they're calibrated, not derived

| Tier | Long window | Short (confirm) window | Burn rate | Budget spent | Framing |
|------|-------------|------------------------|-----------|--------------|---------|
| Page (fast) | 1h | 5m | 14.4× | ~2% | "gone in ~2 days" |
| Page (slower) | 6h | 30m | 6× | ~5% | steady serious leak |
| Ticket | 3d | 6h | 1× | ~10% | slow leak, no wake-up |

A common staff-interview trap is presenting 14.4/6/1 as if they fall out of pure math. They don't: the intuition is real (a fast-page tier should catch an incident that would burn the *entire* monthly budget in about a day of continued burn, using a window short enough to page within the hour) but the exact multipliers are constants Google calibrated empirically against a noise-vs-detection-time Pareto curve over their own traffic patterns. Nothing about the derivation *requires* 14.4 over, say, 10 or 20 — it's the operating point Google published, not a law of nature.

The short/long window ratio (5m : 1h = 1/12, 30m : 6h = 1/12) is itself tunable. Google's own alternative scheme — a "2% burn in 1h + 5% burn in 6h" pairing recommended by practitioners as a simpler starting point — uses a different ratio and different thresholds entirely. The right move at staff level is to justify a ratio for *your* service (traffic volume, on-call response SLA, how fast a bad deploy typically manifests) rather than cargo-culting 1/12 because a blog post used it.

### The two-window AND is the entire trick

Each page tier fires only when *both* the long and short windows exceed the burn threshold. The long window supplies detection sensitivity and low reset-noise; the short window supplies inertia — it forces the condition to still be true *now*, so a 90-second blip that already recovered never pages, and a resolved incident clears fast instead of hanging on a 1h trailing average.

### Why the alternatives fail

Fixed error-rate thresholds have no notion of budget — they page on every blip and miss slow burns entirely. Single-window burn-rate is jumpy: it flaps at the boundary and either resets too slowly (long window) or fires on noise (short window). MWMBR is the empirically-tuned point that maximizes detection time and minimizes false pages simultaneously.

### Symptom vs cause, operationalized

Page only on user-facing SLI breach via burn rate. Route saturation/cause signals — CPU, queue depth, run-queue length, PCIe/NVLink saturation — to dashboards and tickets. They explain a page; they do not fire one.

### The low-traffic caveat, quantified

Burn rate is a *ratio estimator* (errors / requests), and ratio estimators inherit ratio-estimator variance: the standard error of the estimated rate shrinks roughly as 1/√n in the request count n. At low request volume that variance is enormous. Concretely: an SLO of 99.9% (0.1% error budget) evaluated over a 5-minute window with 40 requests — one single error already reads as a 2.5% error rate, i.e. a 25× burn, comfortably above the 14.4× fast-page threshold, off *one* request. This is exactly why a bare "AND requests > N" floor is the correct fix and not an ad hoc patch — it's addressing the actual statistical mechanism (denominator too small for the ratio to be meaningful), not papering over a symptom. Raise N until one-or-two-error noise can no longer cross threshold on its own (roughly, N large enough that 1/N sits comfortably under the SLO's error budget), lengthen windows, aggregate related low-traffic services, or inject synthetic probe traffic to keep the denominator sane.

### Multi-service composition

Real requests touch multiple downstream services, and their SLOs compose multiplicatively, not additively. Five downstream calls each individually meeting a 99.9% SLO compose to roughly 0.999⁵ ≈ 99.5% — a full order of magnitude worse than any single service's stated number, and already below what many teams *think* their aggregate promise is. This is the same trap at the fleet layer: a training job's "goodput SLO" compounds the scheduler's, the network's, and the storage layer's individual SLOs, so a goodput target tighter than the loosest of those components is the only honest number to alert against. It also means a burn-rate alert should ideally evaluate the SLI you actually control — your own timeout/retry/circuit-breaker behavior wrapping a dependency — rather than raw dependency error rate, or you page your team for another team's incident.

### Error budgets require governance, or they're just fancier thresholds

Burn-rate alerting only pays off if someone owns the decision of what happens when budget is exhausted — feature freeze, mandatory rollback, a change-freeze policy. Without that governance step, MWMBR is mechanically identical to a well-tuned threshold alert: it still just tells someone to look at a screen. The math is necessary but not sufficient; the org process that consumes the budget signal is what makes it an *error budget* rather than a better dashboard.

### GPU tie — alert on goodput, not utilization

`GPU_UTIL` reports that *a* kernel is resident on the SM, not that it is doing useful work. A spinning kernel, a busy-wait on a lock, or a NCCL all-reduce stalled on the slowest rank all read 100% util (the util-lie). Define the SLI as **goodput**: `goodput_ratio = achieved_tokens_per_sec / expected_tokens_per_sec` (equivalently MFU = achieved FLOPs / peak FLOPs). The error budget is allowable **wasted GPU-hours per month**. Burn-rate-alert on *goodput regression*: a page fires when effective throughput drops far enough below the run's expected curve that it is burning the wasted-GPU-hours budget in ~2 days — not when a util gauge dips.

This is also a genuinely different stakeholder conversation than standard SRE burn-rate alerting. A user-facing error-budget burn rate is framed around user pain tolerance (SRE's classic audience). A wasted-GPU-hours burn rate is framed around **capital efficiency tolerance** — the audience for that conversation is finance and engineering leadership, not on-call. Same MWMBR machinery, a physically meaningful SLI underneath, and a different room it gets presented in. (See the separate GPU-observability artifact for DCGM/util-lie/MFU/goodput mechanics.)

## Perspectives

**Statistical/precision.** Burn-rate math is a ratio estimator, and ratio estimators have well-known variance problems at low sample counts. Treating the request-count floor as a magic number rather than a direct consequence of 1/√n variance is a tell that the underlying statistics weren't understood — staff engineers should be able to derive *why* the floor is needed, not just that it is.

**Org-process.** Burn-rate alerting is necessary but not sufficient. The math tells you *when* to page; it says nothing about what happens after the page. Without a governed policy for what "budget exhausted" triggers organizationally, MWMBR is a fancier threshold alert wearing a budget costume.

**Multi-service composition.** SLOs don't compose the way people intuitively expect. A team that owns one hop in a five-hop request path and hits 99.9% on their own service can still be the reason the end-to-end promise is 99.5%, and no amount of per-service dashboard-greenness reveals that without doing the multiplication. This is directly the shape of a GPU fleet's goodput SLO compounding scheduler/network/storage.

**GPU-economics.** Reframing the error budget from "user pain tolerance" to "capital efficiency tolerance" changes who the alert is *for*. A wasted-GPU-hours burn rate is a finance-adjacent signal as much as an SRE one — it's the same math applied to a completely different incentive structure, and staff engineers on GPU fleets need to be fluent presenting it either way.

## Real-world use cases

- **Google SRE Workbook — "Alerting on SLOs"** (https://sre.google/workbook/alerting-on-slos/): the canonical source of the exact 14.4×/6×/1× tiers and the multi-window rationale — treat this as the ground truth the tiers come from, not an interpretation of it.
- **Datadog — "Burn rate is a better error rate"** (https://www.datadoghq.com/blog/burn-rate-is-better-error-rate/): walks the error_rate/error_budget derivation with a concrete numeric example and argues burn rate's real value is as a normalized unit that's comparable *across* services with different SLOs and different traffic — a threshold of "14.4" means the same thing everywhere, a raw error rate doesn't.
- **Grafana Labs — "How to implement multi-window, multi-burn-rate alerts with Grafana Cloud"** (https://grafana.com/blog/how-to-implement-multi-window-multi-burn-rate-alerts-with-grafana-cloud/): a concrete operational walkthrough — recording rules, alert rules, SLO-object generation — of turning the workbook's tiers into deployable config.
- **Yuri Grinshteyn (Google Cloud engineer) — "How to alert on SLOs"** (https://medium.com/google-cloud/how-to-alert-on-slos-2a5ce8c4e7dd): recommends a "2% burn in 1h + 5% burn in 6h" pairing as a simpler practical starting point, distinct from the textbook 14.4×/6× tiers — good evidence that practitioners tune the canonical numbers rather than treating them as immutable.

## Worked example

**1. Standard availability MWMBR in PromQL.** Precompute burn as a recording rule so alerts stay cheap:

```yaml
# recording rules
- record: job:slo_error_budget_burn:ratio_rate1h
  expr: |
    sum(rate(http_requests_total{code=~"5.."}[1h]))
      / sum(rate(http_requests_total[1h]))
    / (1 - 0.999)      # normalize by the 0.1% error SLO -> burn rate units
- record: job:slo_error_budget_burn:ratio_rate5m
  expr: |
    sum(rate(http_requests_total{code=~"5.."}[5m]))
      / sum(rate(http_requests_total[5m]))
    / (1 - 0.999)
# (repeat for 6h/30m and 3d/6h)
```

```yaml
# alerts
- alert: SLOBurnFast
  expr: |
    job:slo_error_budget_burn:ratio_rate1h > 14.4
      and job:slo_error_budget_burn:ratio_rate5m > 14.4
      and sum(rate(http_requests_total[5m])) > 200   # low-traffic floor
  labels: {severity: page}
- alert: SLOBurnSlow
  expr: |
    job:slo_error_budget_burn:ratio_rate6h > 6
      and job:slo_error_budget_burn:ratio_rate30m > 6
      and sum(rate(http_requests_total[30m])) > 200
  labels: {severity: page}
- alert: SLOBurnTicket
  expr: |
    job:slo_error_budget_burn:ratio_rate3d > 1
      and job:slo_error_budget_burn:ratio_rate6h > 1
  labels: {severity: ticket}
```

The `requests > 200` floor exists precisely because of the ratio-estimator variance argument above: at 200 requests in a 5m window, a single error reads as 0.5% (5× burn) rather than 2.5% (25× burn) at 40 requests — enough headroom that isolated noise no longer crosses the 14.4× line by accident.

**2. The GPU analog.** SLI is goodput; budget is wasted GPU-hours/month.

```yaml
- record: run:goodput_ratio:5m
  expr: |
    sum(rate(achieved_tokens_total[5m]))
      / on(run) group_left expected_tokens_per_sec
# "error" = fraction of capacity wasted = 1 - goodput_ratio
- record: run:gpu_waste_burn:1h
  expr: (1 - avg_over_time(run:goodput_ratio:5m[1h])) / WASTE_BUDGET_FRACTION
```

Then the same 14.4× / 6× / 1× two-window AND structure fires on `run:gpu_waste_burn` — a page means "this run is bleeding wasted-GPU-hours fast enough to blow the monthly waste budget in ~2 days," which is actionable in a way `GPU_UTIL < 90%` never is.

**3. Multi-service composition, concretely.** A request fans out to 5 downstream services, each individually meeting 99.9%: 0.999⁵ ≈ 0.99501, i.e. ~99.5% end-to-end — roughly 5× the "allowed" monthly downtime of any single hop. If the product promise is 99.9% end-to-end, each of those 5 services actually needs to run closer to 99.98% individually (0.9998⁵ ≈ 0.999), a much tighter internal bar than the externally-advertised number.

## Practice

<feeds [fleet observability design](../practice/fleet-observability/README.md)>

Design the complete alerting layer for the fleet observability system: (1) write the three-tier MWMBR rule set for the API-serving SLO, choosing windows and justifying the short/long ratio for your own traffic profile rather than defaulting to 1/12; (2) define the GPU goodput SLI, pick a wasted-GPU-hours budget for a 512-GPU training pool, and build the burn-rate rules against it; (3) add a low-traffic guard for a sparsely-hit internal endpoint and justify the request-count floor from the ratio-estimator variance argument, not just "it felt right"; (4) write the routing table stating which signals page, which ticket, and which are dashboard-only, and defend each placement with the symptom-vs-cause rule; (5) state, in one paragraph, who owns the decision when the monthly error budget is exhausted — this is the governance step the math alone doesn't supply.

## Common pitfalls

- **"Burn rate 14.4 means 14.4% of budget consumed."** Wrong — burn rate is a multiplier on the sustainable spend rate, not a percentage. Percentage consumed requires the full identity: `burn_rate × (alert_window / SLO_window)`. 14.4× over a 1h window against a 30-day SLO is ~2% consumed, not 14.4%.
- **"The 14.4/6/1 tiers are universal constants."** They're Google's empirically calibrated operating point for a general-purpose noise/detection tradeoff, not a derived law. Unusual traffic patterns (bursty, low-volume, seasonal) legitimately justify re-tuned windows and thresholds — pretending otherwise in an interview is the tell of someone who memorized the table without understanding where it came from.
- **"A burn-rate alert firing means the SLO is already breached."** No — burn-rate alerts are leading indicators, deliberately designed to fire *before* the SLO period ends and *before* the budget is fully exhausted, so there's still time to respond. A 14.4× page means "at this rate you'll exhaust budget in ~2 days," not "budget is gone."
- **"GPU goodput SLOs are just infrastructure SLOs with a different name."** A goodput SLO measures useful work delivered, not infrastructure health. A job can be green on every conventional infra SLI — CPU fine, network fine, no restarts — while its goodput SLO burns because it's stuck at 40% MFU. It's a fundamentally different failure surface, not a relabeling of the same one.
- **Low-traffic services get paged on noise, not incidents.** Without a request-count floor, a handful of errors on a sparsely-hit endpoint reads as an enormous instantaneous rate and pages on statistical noise — the fix is the floor (or window lengthening / synthetic traffic), not disabling the alert.

## Self-check

- Why does the fast-page tier require *both* a 1h and a 5m window to exceed 14.4×, rather than just the 1h window? **Answer:** The 1h window gives sensitive detection but high inertia — it would keep firing for ~an hour after an incident already resolved, and a recovered 2-minute blip would still show elevated. The 5m confirmation window forces the burn to still be true *right now*, killing false pages on transient blips and clearing the alert quickly on recovery. The AND is what makes MWMBR the noise/detection Pareto point.
- A GPU node reports `GPU_UTIL` = 100% for an hour, yet the run is behind schedule. What SLI would have paged, and why did utilization stay silent? **Answer:** A goodput SLI (`achieved_tokens_per_sec / expected_tokens_per_sec`, or MFU) would have caught it. Utilization only reports that a kernel is resident on the SM — a spinning kernel, a lock busy-wait, or a NCCL wait on the slowest rank all read 100% while producing near-zero useful FLOPs (the util-lie). Goodput measures useful work, so its burn rate rises even as util pins at 100%.
- Using `budget_fraction = burn_rate × (alert_window / SLO_window)`, how much of a 30-day budget does a sustained 6× burn consume over its 6h window, and does that justify a page? **Answer:** 6 × (6h / 720h) = 6 × 1/120 = 5%. Yes — 5% of a month's budget in six hours is a serious steady leak (full exhaustion in 30/6 = 5 days) that warrants waking someone, which is exactly why it is the "slower page" tier rather than a ticket.
- A 5-minute window with only 40 total requests logs one 5xx. Why does this alone nearly trip the fast-page (14.4×) tier against a 99.9% SLO, and what's the correct fix? **Answer:** One error in 40 requests is a 2.5% error rate; against a 0.1% budget that's a 25× burn — above the 14.4× threshold off a single request, because the ratio estimator's variance is huge at n=40 (std error scales ~1/√n). The correct fix is a request-count floor (`AND requests > N`) sized so a single or double error can't cross threshold alone, not lengthening the window indefinitely or lowering the multiplier, which would blunt real detections too.
- A request path fans out to 5 downstream services each individually meeting a 99.9% SLO. What's the realistic end-to-end availability, and what does that imply for a product-level 99.9% promise? **Answer:** Roughly 0.999⁵ ≈ 99.5% end-to-end — about 5× the downtime any single hop is "allowed." To honestly promise 99.9% end-to-end, each of the 5 services needs a materially tighter internal SLO (≈99.98% each), not the same number advertised externally. This is the same compounding a GPU training job's goodput SLO faces against scheduler/network/storage.

## Connections & what's next

Burn-rate alerting is where the correctness work from [01 · The signal model](01-signal-model.md) through [06 · Logging pipelines](06-logging-pipelines.md) gets consumed — a wrong PromQL trap or a mislabeled log field shows up here as either a missed page or a false one. It also sets up the GPU goodput SLI that [09 · GPU and ML observability at fleet scale](09-gpu-and-ml-observability.md) scales to thousands of GPUs with per-tenant multitenancy and straggler detection.

Once a burn-rate alert fires correctly, the next move is finding out *why* — that's [08 · Continuous profiling and eBPF](08-profiling-and-ebpf.md), the escalation rung from "the SLI is burning budget" to "which stack is burning the cycles or the wait time."

## References & further reading

**Primary sources**
- Google SRE Workbook — Alerting on SLOs: https://sre.google/workbook/alerting-on-slos/
- Google SRE Book — Service Level Objectives: https://sre.google/sre-book/service-level-objectives/

**Real-world engineering blogs**
- Grafana Labs — How to implement multi-window, multi-burn-rate alerts with Grafana Cloud: https://grafana.com/blog/how-to-implement-multi-window-multi-burn-rate-alerts-with-grafana-cloud/
- Datadog — Burn rate is a better error rate: https://www.datadoghq.com/blog/burn-rate-is-better-error-rate/
- Yuri Grinshteyn — How to alert on SLOs: https://medium.com/google-cloud/how-to-alert-on-slos-2a5ce8c4e7dd
