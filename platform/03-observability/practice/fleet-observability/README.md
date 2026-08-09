# Fleet observability design — Observability module deliverable

One portfolio artifact in three reinforcing parts, mapping 1:1 onto the three most common
GPU-infra observability interview loops (design, alerting/SRE, debugging depth). All finishable
on paper + a small Prometheus/Grafana — no real fleet required.

## A) The fleet observability design doc (4,000-GPU cluster)

A staff-level design doc covering:

- **Scrape sharding** — hashmod the DCGM scrape across a Prometheus pool; why a single Prometheus
  dies (head-block cardinality → RAM → OOM → slow WAL replay).
- **Long-term storage** — Thanos vs **Mimir** choice with the reasoning; per-tenant series limits
  (`max_global_series_per_user`); downsampling for capacity-planning history.
- **Per-tenant cardinality budget** — an actual spreadsheet: which labels are bounded
  (`node`, `gpu`, `tenant`, `model_class`) vs which become exemplars/traces (`gpu_uuid`,
  `pod_hash`, `job_id`), with `metric_relabel_configs` dropping the bombs. Show the series count
  before/after.
- **DCGM + NCCL signal plan** — the fleet metrics (SM_ACTIVE, goodput, XID/thermal/ECC events),
  NCCL collective observability (NIXT/inspector), and the recording rules for fleet rollups.
- **Two-tier OTel collector** — agent DaemonSet (`memory_limiter` first, `k8sattributes`
  enrichment) → gateway (tail-sampling, `loadbalancing` exporter keyed by trace-ID).

## B) The burn-rate alert set (deployable rule YAML)

- A full **multi-window multi-burn-rate** set for a **service SLO** (two page tiers + one ticket)
  against `error_budget_burn` recording rules.
- A **GPU goodput-regression SLO**: SLI `goodput_ratio = achieved_tokens_per_sec / expected`,
  error budget = allowable wasted GPU-hours/month, with the multi-window burn condition and a
  straggler alert (`per-rank step-time > 1.3 × median`).

## C) The "PromQL traps that lie" writeup

The five traps, each with the broken query, the dashboard symptom, the fix, and the one-line
reason — the kind of doc a staff engineer circulates to level up a team:

1. `rate()` on a gauge → use `deriv()`/`delta()`.
2. `avg(p99)` / quantile-of-quantiles → aggregate buckets then `histogram_quantile`.
3. `histogram_quantile` accuracy bounded by bucket edges → native histograms.
4. `rate()` window < 4× scrape interval → gaps/NaN.
5. `increase()` extrapolation on integer counters → don't alert on exact values.

## Suggested layout

```
fleet-observability/
├── design.md            # the 4,000-GPU design doc
├── cardinality-budget.*  # the per-tenant label/series spreadsheet
├── alerts/
│   ├── service-slo.yaml  # MWMBR rules for a service SLO
│   └── gpu-goodput.yaml  # goodput-regression SLO + straggler alert
├── promql-traps.md       # the five traps writeup
└── README.md             # how it fits + how to reproduce
```

## Acceptance criteria (matches the [checkpoint](../../checkpoint.md))

- [ ] a fleet design doc with scrape-sharding + Mimir/Thanos sizing + downsampling
- [ ] a per-tenant **cardinality budget** with `metric_relabel_configs` and before/after series counts
- [ ] a DCGM + NCCL signal plan with fleet-rollup recording rules
- [ ] a deployable **MWMBR** alert set for a service SLO (with the multiplier math shown)
- [ ] a **GPU goodput-regression** SLO with a wasted-GPU-hours budget + a straggler query
- [ ] the five-trap **"PromQL traps that lie"** writeup

## Guardrails

- No real fleet needed — a small Prometheus/Grafana + synthetic series demonstrates every query;
  the design doc and cardinality budget are pure analysis.
- **Publishable-by-default** — the PromQL-traps writeup and the goodput-SLO design are strong
  portfolio/blog pieces; scrub any employer-specific label names before posting.
