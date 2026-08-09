---
lesson: "A03.4"
title: "OpenTelemetry"
module: "A-03"
concept: "SDK vs Collector, two-tier pipelines, semantic conventions"
status: not-started
est_time: "3 hrs"
artifacts: ["fleet-observability design"]
---

# A03.4 · OpenTelemetry

> **Concept.** Keep the SDK thin and make the Collector the integration point — a two-tier agent→gateway pipeline where processor order and tail-sampling placement are load-bearing.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Why this matters
Seniors know OTel exists and can add an SDK. Staff decide *where the seams go* so the org can change backends, sampling, redaction, and enrichment for hundreds of services **without redeploying a single app**. Get the architecture wrong — fat SDK, tail-sampling on the wrong tier, `memory_limiter` in the wrong slot — and you get either app-team lock-in, broken sampled traces, or a Collector that OOMs the moment a backend hiccups. On a GPU training/inference fleet, OTel's emerging GenAI conventions are how you get token/model telemetry without touching training code.

## Core notes
**Skip (you already know):** traces/metrics/logs SDKs exist, OTLP is the wire protocol, auto-instrumentation exists.

**SDK vs Collector — the load-bearing distinction.** Keep the **SDK thin**: context propagation and span creation, nothing more. Push everything operational — backend choice, sampling policy, redaction, enrichment, tenant routing — into the **Collector**, which is the integration point you can change centrally. Every decision you bake into the SDK is a decision you must **redeploy hundreds of apps** to change. This is the whole game.

**Two-tier topology (the staff pattern).**
thin SDK → **agent collector** (DaemonSet, one per node): resource detection, batching, cheap edge filtering, receives local OTLP. → **gateway collector** (Deployment, horizontally scaled): tail-sampling, routing, tenant fan-out, backend export.

**Pipeline anatomy:** `receivers → processors → exporters`, defined **per signal** (traces/metrics/logs pipelines are separate). Processors that matter and *why*:
- **`memory_limiter` — order it FIRST.** It sheds/refuses load under back-pressure so the Collector doesn't OOM when a downstream backend slows. If it runs after `batch`, batches accumulate in RAM before the limiter can act — defeating it.
- **`batch`** — throughput; compresses many spans/points into fewer export calls. Runs after the limiter.
- **`resourcedetection` / `k8sattributes`** — enrichment; attach node, pod, namespace, gpu_index, tenant. Runs on the **agent** tier where node-local metadata is available.
- **`tail_sampling`** — decide keep/drop after seeing the *whole* trace. **Must live on the gateway tier.**
- **`transform` / `filter`** — drop or redact high-cardinality junk and PII **at the edge** (agent) before it costs money downstream.

**Why tail-sampling can't live on the agent.** Tail-sampling needs *all spans of a trace* co-located to evaluate a policy (e.g. "keep any trace with an error or p99 latency"). In a real cluster the spans of one distributed trace are emitted on **different nodes**, so each agent DaemonSet only ever sees a fragment. Only the **gateway** — where you route all spans of a trace to the same instance (trace-ID-aware load balancing) — can assemble the full trace and sample it correctly. Put it on the agent and you sample on partial traces, corrupting the result.

**Semantic conventions are the real payoff.** Stable, agreed attribute names (`http.request.method`, `service.name`, `k8s.pod.name`) are what make **cross-service queries** and **vendor portability** possible — the whole reason to standardize. But conventions **churn** (e.g. `http.method` → `http.request.method`). Staff move: **pin convention versions** and handle both old and new names **in the Collector** (via `transform`), *not* by re-releasing every app. The apps emit whatever their SDK version knows; the Collector normalizes.

**Migration reality.** Don't big-bang. Run the Collector **alongside** existing Prometheus/Jaeger/Fluent, **dual-export**, and cut over **backend-by-backend**. The `prometheus` receiver plus `prometheusremotewrite` exporter let the Collector **subsume metrics** too, so it can scrape existing targets and forward into Mimir/Thanos — unifying the pipeline from A03.3 under one Collector fleet.

**GPU-fleet tie (reference the DCGM/util-lie/MFU artifact).** OTel now ships **GenAI semantic conventions** (`gen_ai.request.model`, `gen_ai.usage.input_tokens` / `output_tokens`, latency) — the emerging standard for instrumenting inference and training. The Collector enriches GPU/inference spans with `node`, `gpu_index`, and `tenant` via `k8sattributes` **without touching training code**, so model/token telemetry and infra telemetry join on the same attributes.

## Worked example
**Two-tier config sketch.**

*Agent (DaemonSet), traces pipeline:*
`receivers: [otlp]` → `processors: [memory_limiter, k8sattributes, resourcedetection, filter, batch]` → `exporters: [otlp/gateway]`
- `memory_limiter` first (survive back-pressure), `k8sattributes`/`resourcedetection` add node/gpu_index/tenant, `filter` drops high-cardinality debug spans at the edge, `batch` last before shipping to the gateway over OTLP.

*Gateway (Deployment), traces pipeline:*
`receivers: [otlp]` (fronted by a trace-ID-aware load balancer so all spans of a trace land on one instance) → `processors: [memory_limiter, tail_sampling, batch]` → `exporters: [otlp/backend]`
- `tail_sampling` policy: keep 100% of traces with an error status or latency > threshold, plus a low probabilistic sample of the rest.

**The question the learner must answer:** why can't that `tail_sampling` processor live on the agent tier? Because the agent DaemonSet only sees the spans emitted on its own node, and the spans of a single distributed trace are scattered across many nodes. Only the gateway, behind trace-ID-aware routing, sees the complete trace and can apply a whole-trace keep/drop decision.

## Practice
<feeds [fleet observability design](../practice/fleet-observability/README.md)>

Write the tracing/telemetry section of the fleet-observability design: the two-tier Collector topology, both pipeline configs with processors in the correct order (justify `memory_limiter` first), the trace-ID-aware routing that makes gateway tail-sampling correct, a convention-version normalization step in the Collector, and the GenAI-convention enrichment plan for the GPU fleet. State the migration sequence from the existing Prometheus/Jaeger stack.

## Self-check
- Why keep the SDK thin and push sampling/redaction/routing into the Collector? **Answer:** Anything baked into the SDK requires redeploying every app to change; the Collector is a central integration point where you swap backends, sampling, redaction, and enrichment for hundreds of services with zero app redeploys.
- Where must `tail_sampling` run and why can't it run on the agent DaemonSet? **Answer:** On the gateway tier behind trace-ID-aware routing, because a whole-trace keep/drop decision needs all spans of the trace co-located; agents only see the fragment emitted on their own node, since one distributed trace's spans are scattered across many nodes.
- Why is `memory_limiter` ordered first in the processor chain, and how do you absorb semantic-convention churn without redeploying apps? **Answer:** `memory_limiter` first so it sheds/refuses load before batches accumulate in RAM, preventing the Collector from OOMing under downstream back-pressure; handle convention churn (e.g. `http.method` → `http.request.method`) by pinning versions and normalizing old/new attribute names in the Collector via `transform`, not by re-releasing every service.

## References
- OpenTelemetry Collector: https://opentelemetry.io/docs/collector/
- Semantic conventions: https://opentelemetry.io/docs/concepts/semantic-conventions/
- Collector configuration (processors/pipelines): https://opentelemetry.io/docs/collector/configuration/
- GenAI semantic conventions: https://opentelemetry.io/docs/specs/semconv/gen-ai/
