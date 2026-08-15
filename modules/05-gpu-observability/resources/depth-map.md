# Depth map — Module 05 · GPU observability

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **Near-1:1 match — and mostly confirmation.** The source's `gpu-observability/` track is 23
> chapters covering the same ground as this module, including the same util-lie thesis. Read it as
> a **second opinion** rather than new material: where it agrees, your understanding is solid;
> where it differs, one of you is wrong and finding out which is worth the time.
>
> The genuinely additive parts are the **three appendices** and the **lakehouse chapter**.

| Lesson | Go deeper in | Why |
|---|---|---|
| 01 The lie and the truth | [`gpu-observability/00-mental-models`](https://github.com/harut8/system-design/blob/main/gpu-observability/00-mental-models.md) · [`06-host-level-gpu-utilization`](https://github.com/harut8/system-design/blob/main/gpu-observability/06-host-level-gpu-utilization.md) | the same argument from the hardware side — SM occupancy vs kernel residency |
| 01 The lie and the truth | [`gpu-observability/appendix-b-field-ids`](https://github.com/harut8/system-design/blob/main/gpu-observability/appendix-b-field-ids.md) | **the DCGM field-ID cheat sheet** — the most immediately useful single page for the synthetic exporter and your dashboards |
| 02 DCGM architecture | [`gpu-observability/01-architecture-and-stack`](https://github.com/harut8/system-design/blob/main/gpu-observability/01-architecture-and-stack.md) | hostengine, NVML, the collection stack end to end |
| 03 dcgm-exporter at fleet scale | [`gpu-observability/02-dcgm-exporter-deep-dive`](https://github.com/harut8/system-design/blob/main/gpu-observability/02-dcgm-exporter-deep-dive.md) | config, field groups, the profiling sampler's cost, and the commented-out metrics |
| 03 dcgm-exporter at fleet scale | [`gpu-observability/08-prometheus-metrics-design-and-cardinality`](https://github.com/harut8/system-design/blob/main/gpu-observability/08-prometheus-metrics-design-and-cardinality.md) | the cardinality budget worked in GPU terms — pairs with `platform/03` L1/L3 |
| 04 Attribution | [`gpu-observability/03-k8s-gpu-cluster-observability`](https://github.com/harut8/system-design/blob/main/gpu-observability/03-k8s-gpu-cluster-observability.md) · [`13-multi-tenant-gpu-observability`](https://github.com/harut8/system-design/blob/main/gpu-observability/13-multi-tenant-gpu-observability.md) | the K8s identity join, then per-tenant isolation of the resulting series |
| 04 Attribution | [`gpu-observability/04-batch-vs-stateless-workloads`](https://github.com/harut8/system-design/blob/main/gpu-observability/04-batch-vs-stateless-workloads.md) | why a batch job and a serving deployment need different observability shapes — a distinction this module treats lightly |
| 05 Health & XID | [`gpu-observability/07-hardware-health-and-failure-detection`](https://github.com/harut8/system-design/blob/main/gpu-observability/07-hardware-health-and-failure-detection.md) · [`appendix-c-flowcharts`](https://github.com/harut8/system-design/blob/main/gpu-observability/appendix-c-flowcharts.md) | **the troubleshooting flowcharts are worth printing** — symptom → decision tree → action |
| 06 Inference SLOs | [`gpu-observability/14-llm-inference-observability`](https://github.com/harut8/system-design/blob/main/gpu-observability/14-llm-inference-observability.md) | TTFT/TPOT/ITL instrumentation specifics |
| 07 Profiling escalation | [`gpu-observability/11-profiling-integration`](https://github.com/harut8/system-design/blob/main/gpu-observability/11-profiling-integration.md) | the metrics → profiler → Nsight ladder, with the overhead numbers |
| 08 Capstone — allocated vs utilised | [`gpu-observability/05-gpu-allocation-and-utilization-efficiency`](https://github.com/harut8/system-design/blob/main/gpu-observability/05-gpu-allocation-and-utilization-efficiency.md) | the same deliverable's core calculation, independently derived |
| 08 Capstone — allocated vs utilised | [`gpu-observability/09-grafana-dashboards`](https://github.com/harut8/system-design/blob/main/gpu-observability/09-grafana-dashboards.md) · [`10-alerting-strategy`](https://github.com/harut8/system-design/blob/main/gpu-observability/10-alerting-strategy.md) | panel-by-panel dashboard design and an alert taxonomy to steal from |

## The genuinely new material

- [`gpu-observability/17-telemetry-lakehouse-and-sql-analytics`](https://github.com/harut8/system-design/blob/main/gpu-observability/17-telemetry-lakehouse-and-sql-analytics.md)
  — the question this module can't answer: *what did team X cost last quarter?* This gap was real
  enough that it became a new lesson —
  [`platform/03` L10](../../../platform/03-observability/lessons/10-telemetry-lakehouse.md).
- [`gpu-observability/16-incident-walkthrough`](https://github.com/harut8/system-design/blob/main/gpu-observability/16-incident-walkthrough.md)
  — a slowdown traced end to end. Read it as a worked debugging rep for Module 12's debugging drills.
- [`gpu-observability/15-distributed-training-observability`](https://github.com/harut8/system-design/blob/main/gpu-observability/15-distributed-training-observability.md)
  — straggler/goodput material; see Module 08's depth map.
- [`gpu-observability/tasks.md`](https://github.com/harut8/system-design/blob/main/gpu-observability/tasks.md)
  — practice tasks, all runnable against the
  [fake GPU fleet](../../04-gpu-on-kubernetes/practice/fake-gpu-fleet/README.md).

## Where this course goes further

Cost. The source stops at efficiency metrics; Module 11's attribution models, commitment
strategy, fragmentation economics and FOCUS output have no counterpart there. That remains your
differentiator — don't dilute it by re-importing an observability framing.
