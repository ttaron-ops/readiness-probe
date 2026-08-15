# Depth map — platform/03 · Observability engineering

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **The deepest single match in the repo.** The `sre-observability/` track is 47 chapters plus
> three appendices — roughly five times this module's ten lessons. It is organised the same way
> (signal model → collection → storage → query → SLO → operations) so the mapping is nearly
> chapter-for-lesson, and then it keeps going into two dozen specialist topics this module doesn't
> reach.

| Lesson | Go deeper in | Why |
|---|---|---|
| 01 Signal model | [`sre-observability/00-mental-models`](https://github.com/harut8/system-design/blob/main/sre-observability/00-mental-models.md) · [`01-architecture-and-stack`](https://github.com/harut8/system-design/blob/main/sre-observability/01-architecture-and-stack.md) | the vocabulary and the whole-stack map, stated once before the specialisation |
| 01 Signal model | [`sre-observability/18-cardinality-and-cost`](https://github.com/harut8/system-design/blob/main/sre-observability/18-cardinality-and-cost.md) | "the hardest single problem" — the economics behind this module's master constraint |
| 02 Prometheus & PromQL | [`sre-observability/10-query-layer`](https://github.com/harut8/system-design/blob/main/sre-observability/10-query-layer.md) | PromQL/LogQL/TraceQL/SQL compared as query models — sharpens *why* the traps are traps |
| 02 Prometheus & PromQL | [`sre-observability/appendix-c-query-recipe-book`](https://github.com/harut8/system-design/blob/main/sre-observability/appendix-c-query-recipe-book.md) | a recipe book to steal from and to drill the broken-panel checkpoint against |
| 03 Metrics at scale | [`sre-observability/06-metrics-storage`](https://github.com/harut8/system-design/blob/main/sre-observability/06-metrics-storage.md) | TSDB internals: head block, WAL, compaction — the mechanism behind "RAM, not disk, is the wall" |
| 03 Metrics at scale | [`sre-observability/19-multi-tenancy`](https://github.com/harut8/system-design/blob/main/sre-observability/19-multi-tenancy.md) | per-tenant limits and isolation in Mimir-class systems |
| 04 OpenTelemetry | [`sre-observability/02-opentelemetry-deep-dive`](https://github.com/harut8/system-design/blob/main/sre-observability/02-opentelemetry-deep-dive.md) · [`03-instrumentation`](https://github.com/harut8/system-design/blob/main/sre-observability/03-instrumentation.md) | the SDK/API/Collector split and instrumentation practice |
| 04 OpenTelemetry | [`sre-observability/04-collection-and-edge`](https://github.com/harut8/system-design/blob/main/sre-observability/04-collection-and-edge.md) · [`05-transport-and-buffering`](https://github.com/harut8/system-design/blob/main/sre-observability/05-transport-and-buffering.md) | the two-tier collector topology and what happens when the backend is down |
| 05 Distributed tracing | [`sre-observability/08-traces-storage`](https://github.com/harut8/system-design/blob/main/sre-observability/08-traces-storage.md) | trace storage internals and why tail sampling must be gateway-side |
| 06 Logging pipelines | [`sre-observability/07-logs-storage`](https://github.com/harut8/system-design/blob/main/sre-observability/07-logs-storage.md) | index-vs-scan designs (Loki vs ELK) and the label-cardinality bomb |
| 07 SLOs & alerting | [`sre-observability/13-slo-engineering`](https://github.com/harut8/system-design/blob/main/sre-observability/13-slo-engineering.md) · [`12-alerting`](https://github.com/harut8/system-design/blob/main/sre-observability/12-alerting.md) | SLI selection, error budgets, and burn-rate alerting derived independently |
| 07 SLOs & alerting | [`sre-observability/14-on-call`](https://github.com/harut8/system-design/blob/main/sre-observability/14-on-call.md) · [`15-incident-response-and-postmortem`](https://github.com/harut8/system-design/blob/main/sre-observability/15-incident-response-and-postmortem.md) | the human layer the alerts land on — substance for Module 12's behavioural stories |
| 08 Profiling & eBPF | [`sre-observability/09-profiling`](https://github.com/harut8/system-design/blob/main/sre-observability/09-profiling.md) | continuous profiling, symbolization, and pprof/eBPF overhead |
| 09 GPU & ML observability | [`sre-observability/26-llm-and-ai-observability`](https://github.com/harut8/system-design/blob/main/sre-observability/26-llm-and-ai-observability.md) | the AI-workload signal set from the application side, complementing the DCGM view |
| 09 GPU & ML observability | [`gpu-observability/15-distributed-training-observability`](https://github.com/harut8/system-design/blob/main/gpu-observability/15-distributed-training-observability.md) | straggler and goodput detection — see also [Module 05's depth map](../../../modules/05-gpu-observability/resources/depth-map.md) |
| 10 Telemetry lakehouse | [`sre-observability/35-telemetry-lakehouse`](https://github.com/harut8/system-design/blob/main/sre-observability/35-telemetry-lakehouse.md) · [`gpu-observability/17`](https://github.com/harut8/system-design/blob/main/gpu-observability/17-telemetry-lakehouse-and-sql-analytics.md) | **the two chapters this lesson was written from** — the general architecture, then the GPU-specific Kafka contract and rollup grains |

## What this seeded

[Lesson 10 — the telemetry lakehouse](../lessons/10-telemetry-lakehouse.md) is new, and it exists
because these two chapters exposed a real gap: the course had no answer for *"what did team X's
$/useful-GPU-hour do last quarter?"*, which is a question Module 11 and the capstone both depend on.

## The specialist chapters worth knowing exist

Not on the critical path, but each is the answer to a specific interview question:

| Chapter | The question it answers |
|---|---|
| [`28-telemetry-pipeline-reliability`](https://github.com/harut8/system-design/blob/main/sre-observability/28-telemetry-pipeline-reliability.md) | "who observes the observer?" |
| [`36-dr-for-observability-stack`](https://github.com/harut8/system-design/blob/main/sre-observability/36-dr-for-observability-stack.md) | "your monitoring is in the region that just failed" |
| [`38-continuous-verification`](https://github.com/harut8/system-design/blob/main/sre-observability/38-continuous-verification.md) | "how do you know the alert would fire?" |
| [`34-schema-and-semantic-conventions-governance`](https://github.com/harut8/system-design/blob/main/sre-observability/34-schema-and-semantic-conventions-governance.md) | "how do you stop 200 teams inventing 200 label names?" |
| [`37-vendor-migration-patterns`](https://github.com/harut8/system-design/blob/main/sre-observability/37-vendor-migration-patterns.md) | "how would you get us off Datadog?" |
| [`41-brownfield-integration`](https://github.com/harut8/system-design/blob/main/sre-observability/41-brownfield-integration.md) | "what if we can't instrument it?" |
| [`appendix-b-reference-architectures`](https://github.com/harut8/system-design/blob/main/sre-observability/appendix-b-reference-architectures.md) | reference diagrams at several scales |
