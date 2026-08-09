---
lesson: "A03.7"
title: "SLOs and alerting"
module: "A-03"
concept: "multi-window multi-burn-rate"
status: not-started
est_time: "3 hrs"
artifacts: ["mwmbr-alert-set.promql"]
---

# A03.7 · SLOs and alerting

> **Concept.** Multi-window multi-burn-rate alerting is the noise/detection Pareto point — and for GPU fleets the SLI must be goodput, not utilization.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Why this matters

At staff scale the question is no longer "do we have SLOs" but "why does this alert set page at exactly the right moment and stay silent through a transient blip." Getting burn-rate math right is the difference between a fleet that pages 4000 nodes into fatigue and one that pages once, precisely, two days before budget exhaustion. And for GPU fleets the standard playbook actively misleads you: a utilization gauge at 100% can be burning wasted-GPU-hours budget while every dashboard looks green. This lesson is the design math and the GPU reframe, not the vocabulary.

## Core notes

**Skip (you already know):** SLI/SLO/error-budget definitions; symptom-over-cause as a principle; that page fatigue is bad; naive fixed-threshold alerts.

**Burn rate, defined cold.** Burn rate = the rate you are spending error budget relative to the rate that would exhaust it *exactly* at period end. Burn rate 1 = on-pace: you finish the 30-day window having consumed precisely 100% of budget. Burn rate 14.4 = you would exhaust a 30-day budget in ~2 days (30 / 14.4 ≈ 2.08). The core identity:

```
budget_fraction_consumed = burn_rate × (alert_window / SLO_window)
```

So 14.4× sustained over a 1h window against a 30-day SLO consumes 14.4 × (1h / 720h) ≈ 2% of budget in that hour.

**The canonical MWMBR tiers (30-day budget):**

| Tier | Long window | Short (confirm) window | Burn rate | Budget spent | Framing |
|------|-------------|------------------------|-----------|--------------|---------|
| Page (fast) | 1h | 5m | 14.4× | ~2% | "gone in ~2 days" |
| Page (slower) | 6h | 30m | 6× | ~5% | steady serious leak |
| Ticket | 3d | 6h | 1× | ~10% | slow leak, no wake-up |

**The two-window AND is the entire trick.** Each page tier fires only when *both* the long and short windows exceed the burn threshold. The long window supplies detection sensitivity and low reset-noise; the short window supplies inertia — it forces the condition to still be true *now*, so a 90-second blip that already recovered never pages, and a resolved incident clears fast instead of hanging on a 1h trailing average. Short window ≈ long window / 12 is the standard ratio.

**Why the alternatives fail.** Fixed error-rate thresholds have no notion of budget — they page on every blip and miss slow burns entirely. Single-window burn-rate is jumpy: it flaps at the boundary and either resets too slowly (long window) or fires on noise (short window). MWMBR is the empirically-tuned point that maximizes detection time and minimizes false pages simultaneously.

**Symptom vs cause, operationalized.** Page only on user-facing SLI breach via burn rate. Route saturation/cause signals — CPU, queue depth, run-queue length, PCIe/NVLink saturation — to dashboards and tickets. They explain a page; they do not fire one.

**Low-traffic caveat.** MWMBR degrades when request volume is low: a handful of errors becomes an enormous instantaneous rate, so the short window fires spuriously. Mitigations: lengthen windows, add a request-count floor (`AND requests > N`), aggregate related low-traffic services, or inject synthetic probe traffic to keep the denominator sane.

**GPU tie — alert on goodput, not utilization.** `GPU_UTIL` reports that *a* kernel is resident on the SM, not that it is doing useful work. A spinning kernel, a busy-wait on a lock, or a NCCL all-reduce stalled on the slowest rank all read 100% util (the util-lie). Define the SLI as **goodput**: `goodput_ratio = achieved_tokens_per_sec / expected_tokens_per_sec` (equivalently MFU = achieved FLOPs / peak FLOPs). The error budget is allowable **wasted GPU-hours per month**. Burn-rate-alert on *goodput regression*: a page fires when effective throughput drops far enough below the run's expected curve that it is burning the wasted-GPU-hours budget in ~2 days — not when a util gauge dips. This is the staff differentiator: same MWMBR machinery, a physically meaningful SLI underneath. (See the separate GPU-observability artifact for DCGM/util-lie/MFU/goodput mechanics.)

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
  labels: {severity: page}
- alert: SLOBurnSlow
  expr: |
    job:slo_error_budget_burn:ratio_rate6h > 6
      and job:slo_error_budget_burn:ratio_rate30m > 6
  labels: {severity: page}
- alert: SLOBurnTicket
  expr: |
    job:slo_error_budget_burn:ratio_rate3d > 1
      and job:slo_error_budget_burn:ratio_rate6h > 1
  labels: {severity: ticket}
```

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

## Practice

<feeds [fleet observability design](../practice/fleet-observability/README.md)>

Design the complete alerting layer for the fleet observability system: (1) write the three-tier MWMBR rule set for the API-serving SLO, choosing windows and justifying the short/long ratio; (2) define the GPU goodput SLI, pick a wasted-GPU-hours budget for a 512-GPU training pool, and build the burn-rate rules against it; (3) add the low-traffic guard for a sparsely-hit internal endpoint; (4) write the routing table stating which signals page, which ticket, and which are dashboard-only, and defend each placement with the symptom-vs-cause rule.

## Self-check

- Why does the fast-page tier require *both* a 1h and a 5m window to exceed 14.4×, rather than just the 1h window? **Answer:** The 1h window gives sensitive detection but high inertia — it would keep firing for ~an hour after an incident already resolved, and a recovered 2-minute blip would still show elevated. The 5m confirmation window forces the burn to still be true *right now*, killing false pages on transient blips and clearing the alert quickly on recovery. The AND is what makes MWMBR the noise/detection Pareto point.
- A GPU node reports `GPU_UTIL` = 100% for an hour, yet the run is behind schedule. What SLI would have paged, and why did utilization stay silent? **Answer:** A goodput SLI (`achieved_tokens_per_sec / expected_tokens_per_sec`, or MFU) would have caught it. Utilization only reports that a kernel is resident on the SM — a spinning kernel, a lock busy-wait, or a NCCL wait on the slowest rank all read 100% while producing near-zero useful FLOPs (the util-lie). Goodput measures useful work, so its burn rate rises even as util pins at 100%.
- Using `budget_fraction = burn_rate × (alert_window / SLO_window)`, how much of a 30-day budget does a sustained 6× burn consume over its 6h window, and does that justify a page? **Answer:** 6 × (6h / 720h) = 6 × 1/120 = 5%. Yes — 5% of a month's budget in six hours is a serious steady leak (full exhaustion in 30/6 = 5 days) that warrants waking someone, which is exactly why it is the "slower page" tier rather than a ticket.

## References

- https://sre.google/workbook/alerting-on-slos/
- https://grafana.com/blog/how-to-implement-multi-window-multi-burn-rate-alerts-with-grafana-cloud/
- https://sre.google/sre-book/service-level-objectives/
