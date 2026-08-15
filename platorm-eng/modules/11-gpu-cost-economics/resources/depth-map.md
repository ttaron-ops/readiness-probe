# Depth map — Module 11 · GPU cost & unit economics

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **This is the module where the source has the least to teach you — and that is the point.** It
> has no attribution-model taxonomy, no commitment-strategy material, no fragmentation economics,
> no FOCUS treatment, no neocloud-vs-hyperscaler TCO. Its cost material is *observability cost*
> (how much your telemetry stack itself costs), which is a genuinely useful adjacent discipline
> but not this module's subject.
>
> **Keep this module's framing intact.** It's your differentiator; don't dilute it toward an
> observability framing just because more material exists there.

| Lesson | Go deeper in | Why |
|---|---|---|
| 01 Attribution models | [`gpu-observability/13-multi-tenant-gpu-observability`](https://github.com/harut8/system-design/blob/main/gpu-observability/13-multi-tenant-gpu-observability.md) | the per-tenant telemetry join that every attribution model needs as input |
| 02 Allocated vs utilised | [`gpu-observability/05-gpu-allocation-and-utilization-efficiency`](https://github.com/harut8/system-design/blob/main/gpu-observability/05-gpu-allocation-and-utilization-efficiency.md) | the same spine derived independently — useful as a correctness check on your definitions |
| 03 Idle detection | [`gpu-observability/06-host-level-gpu-utilization`](https://github.com/harut8/system-design/blob/main/gpu-observability/06-host-level-gpu-utilization.md) | distinguishing idle-but-allocated from busy-but-inefficient at the host level, which is where false positives come from |
| 05 Unit economics | [`sre-observability/16-capacity-planning`](https://github.com/harut8/system-design/blob/main/sre-observability/16-capacity-planning.md) | headroom, growth modelling, and forecast error bars — transferable method |
| 08 Chargeback / showback | [`sre-observability/31-finops-for-observability`](https://github.com/harut8/system-design/blob/main/sre-observability/31-finops-for-observability.md) | showback mechanics and the social dynamics of billing internal teams — the same problem, a cheaper resource |
| 08 Chargeback / showback | [`sre-observability/19-multi-tenancy`](https://github.com/harut8/system-design/blob/main/sre-observability/19-multi-tenancy.md) | per-tenant limits and quota enforcement in a telemetry system — the enforcement half of showback |
| 09 Existing tooling limits | [`sre-observability/39-build-vs-buy-framework`](https://github.com/harut8/system-design/blob/main/sre-observability/39-build-vs-buy-framework.md) | a structured way to argue "OpenCost/Kubecost doesn't cover this, so we build" without it sounding like NIH |
| 10 FOCUS spec | [`gpu-observability/17-telemetry-lakehouse-and-sql-analytics`](https://github.com/harut8/system-design/blob/main/gpu-observability/17-telemetry-lakehouse-and-sql-analytics.md) | **the most useful chapter here** — the Kafka contract, `GpuSample` schema and rollup grains that make a FOCUS-shaped table producible over a quarter |

## The dependency you should read first

Your FOCUS output, your monthly per-team report, and every "last quarter" claim in this module all
need an analytics path that Prometheus cannot provide. That gap became a lesson:

**[`platform/03` L10 — the telemetry lakehouse](../../../platform/03-observability/lessons/10-telemetry-lakehouse.md)**,
including a worked `$/useful-GPU-hour by team` SQL query with the time-versioned rate-card join.
Read it before Lesson 08 or 10 — it's the substrate those lessons assume.

## Also worth a pass

- [`sre-observability/18-cardinality-and-cost`](https://github.com/harut8/system-design/blob/main/sre-observability/18-cardinality-and-cost.md)
  — "the hardest single problem." Your cost operator emits metrics too, and a cost tool that
  becomes a top-five telemetry spender is an interview story you don't want.
- [`databases/21-in-process-olap-duckdb-chdb`](https://github.com/harut8/system-design/blob/main/databases/21-in-process-olap-duckdb-chdb.md)
  — DuckDB in-process. The cheapest possible way to run the quarter-scale cost queries in this
  module without standing up any infrastructure.
