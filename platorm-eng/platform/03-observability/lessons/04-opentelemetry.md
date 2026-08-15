---
lesson: "A03.4"
title: "OpenTelemetry"
module: "A-03"
concept: "SDK vs Collector, two-tier pipelines, semantic conventions"
status: not-started
est_time: "4 hrs"
prev: "03-metrics-at-scale.md"
next: "05-distributed-tracing.md"
artifacts: ["fleet-observability design"]
sources: 8
---

# A03.4 · OpenTelemetry

> **Concept.** Keep the SDK thin and make the Collector the integration point — a two-tier agent→gateway pipeline where processor order and tail-sampling placement are load-bearing.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits
Lesson 03 sized and sharded the storage tier — Mimir vs Thanos, ingester replication, cardinality limits. That storage tier has to receive telemetry from somewhere, and *how* it gets there across hundreds of heterogeneous services is its own architecture problem: the collection-side integration point that feeds everything downstream, including the metrics pipeline you just designed. This lesson is that integration point. It also sets up lesson 05, which goes deep on the sampling *policy* question this lesson only introduces (head vs tail sampling).

## Why this matters
Seniors know OTel exists and can add an SDK. Staff decide *where the seams go* so the org can change backends, sampling, redaction, and enrichment for hundreds of services **without redeploying a single app**. Get the architecture wrong — fat SDK, tail-sampling on the wrong tier, `memory_limiter` in the wrong slot — and you get either app-team lock-in, broken sampled traces, or a Collector that OOMs the moment a backend hiccups. On a GPU training/inference fleet, OTel's emerging GenAI conventions are how you get token/model telemetry without touching training code.

## What's new here (calibration)
- OTel adoption reframed as a cost/lock-in decision (removed vendor-agent overhead, hedge against a future backend migration), not a technical-purity argument.
- Why OTLP's protocol choice (protobuf over gRPC/HTTP) is the specific technical decision that makes the Collector's receiver/exporter fan-out possible at all.
- The gateway tier as a new, org-wide failure domain — and why processor ordering is what contains it.
- The OTel stability-marking system as the correct way to reason about semantic-convention churn, rather than treating conventions as either frozen or unusable.

## Core concepts
**Skip (you already know):** traces/metrics/logs SDKs exist, OTLP is the wire protocol, auto-instrumentation exists.

**SDK vs Collector — the load-bearing distinction.** Keep the **SDK thin**: context propagation and span creation, nothing more. Push everything operational — backend choice, sampling policy, redaction, enrichment, tenant routing — into the **Collector**, which is the integration point you can change centrally. Every decision you bake into the SDK is a decision you must **redeploy hundreds of apps** to change. This is the whole game.

**What makes the Collector-as-seam argument actually work: OTLP's protocol design.** The reason the Collector can sit between every app and every backend and fan traffic in and out is that OTLP standardizes on protobuf over gRPC/HTTP as a single wire format, instead of each vendor speaking its own bespoke protocol. That single format is what lets one Collector binary ship generic `receivers` (accept OTLP, plus adapters for legacy formats like Prometheus scrape or Jaeger) and generic `exporters` (ship to any backend that speaks OTLP or has an exporter) without bespoke glue code per pair. The architecture only works because the protocol was designed to make it work.

**Two-tier topology (the staff pattern).**
thin SDK → **agent collector** (DaemonSet, one per node): resource detection, batching, cheap edge filtering, receives local OTLP. → **gateway collector** (Deployment, horizontally scaled): tail-sampling, routing, tenant fan-out, backend export.

**Pipeline anatomy:** `receivers → processors → exporters`, defined **per signal** (traces/metrics/logs pipelines are separate). The full staff-correct processor order for a gateway pipeline is:

`memory_limiter → k8sattributes/resourcedetection (if not already done at the agent) → filter/transform → tail_sampling → batch → exporter`

Two ends of that chain are equally load-bearing, for opposite reasons:
- **`memory_limiter` — order it FIRST.** It sheds/refuses load under back-pressure so the Collector doesn't OOM when a downstream backend slows. If it runs after `batch`, batches accumulate in RAM before the limiter can act — defeating it.
- **`batch` — order it LAST.** Batching before filtering wastes CPU packing spans that are about to be dropped anyway. Filter/sample first, batch what survives, then export — batch is a throughput optimization on the *surviving* data, not on everything that arrived.
- **`resourcedetection` / `k8sattributes`** — enrichment; attach node, pod, namespace, gpu_index, tenant. Runs on the **agent** tier where node-local metadata is available, so the gateway doesn't redo per-node lookups.
- **`tail_sampling`** — decide keep/drop after seeing the *whole* trace. **Must live on the gateway tier.**
- **`transform` / `filter`** — drop or redact high-cardinality junk and PII **at the edge** (agent) before it costs money downstream, and again at the gateway for anything session/policy-scoped.

**Why tail-sampling can't live on the agent.** Tail-sampling needs *all spans of a trace* co-located to evaluate a policy (e.g. "keep any trace with an error or p99 latency"). In a real cluster the spans of one distributed trace are emitted on **different nodes**, so each agent DaemonSet only ever sees a fragment. Only the **gateway** — where you route all spans of a trace to the same instance (trace-ID-aware load balancing, via the `loadbalancing` exporter) — can assemble the full trace and sample it correctly. Put it on the agent and you sample on partial traces, corrupting the result.

**Tail sampling doesn't scale horizontally for free — it needs a routing layer.** A naive fleet of gateway replicas behind a plain load balancer will split one trace's spans across multiple replicas, and each replica makes an independent, partially-informed keep/drop decision — silently wrong sampling with no error anywhere. Grafana Labs' own production writeup on this describes exactly that failure and the fix: an aggregation layer (an "aggregate processor" forwarding spans between collector replicas, or the `loadbalancing` exporter keyed by trace ID) that guarantees every span of a given trace lands on the same gateway instance before the sampling decision is made.

**The routing key is use-case-specific, not universal.** Trace-ID-keyed routing is specific to the tail-sampling constraint — completeness of one trace on one instance. A log pipeline has no trace-completeness constraint, so a `loadbalancing` exporter fronting a log pipeline should key on a different dimension entirely (tenant or stream), because the goal there is even distribution and stream-affinity, not trace assembly. Copying the trace-ID routing pattern onto a log pipeline is a category error.

**The Collector's own health is a signal-model decision.** The Collector emits its own metrics — `otelcol_processor_dropped_spans`, `otelcol_exporter_queue_size`, and similar — and if nobody scrapes and alerts on those, a silently-degrading Collector (dropping spans under load, queueing without draining) looks identical from the outside to "the app just isn't emitting spans." Treat the Collector as a telemetry-producing system in its own right, not just plumbing.

**`k8sattributes` has its own fleet-scale cost.** Attaching pod/namespace metadata means watching Pod objects via the Kubernetes API/informer cache. At thousands of nodes, that's real API-server load and requires RBAC scoped correctly per agent — it's a scaling concern on the agent tier, not a free enrichment step.

**Semantic conventions are the real payoff — and they churn.** Stable, agreed attribute names (`http.request.method`, `service.name`, `k8s.pod.name`) are what make **cross-service queries** and **vendor portability** possible — the whole reason to standardize. But conventions **churn** (e.g. `http.method` → `http.request.method`), and OTel's answer to that is an explicit stability-marking system: attributes and signals move through stages such as Experimental/Development toward Stable, and a Stable-marked convention carries backward-compatibility guarantees an Experimental one does not. Staff move: **pin a schema/convention version**, know which stage the conventions you depend on are actually at, and handle both old and new attribute names **in the Collector** (via `transform`), *not* by re-releasing every app. The apps emit whatever their SDK version knows; the Collector normalizes.

**Migration reality.** Don't big-bang. Run the Collector **alongside** existing Prometheus/Jaeger/Fluent, **dual-export**, and cut over **backend-by-backend**. The `prometheus` receiver plus `prometheusremotewrite` exporter let the Collector **subsume metrics** too, so it can scrape existing targets and forward into Mimir/Thanos — unifying the pipeline from lesson 03 under one Collector fleet.

**GPU-fleet tie (reference the DCGM/util-lie/MFU artifact).** OTel now ships **GenAI semantic conventions** (`gen_ai.request.model`, `gen_ai.usage.input_tokens` / `output_tokens`, latency) — the emerging standard for instrumenting inference and training, and currently marked development/experimental, meaning attribute names have already shifted across releases. Pin a version and normalize at the Collector, same as any other convention. The Collector enriches GPU/inference spans with `node`, `gpu_index`, and `tenant` via `k8sattributes` **without touching training code**, so model/token telemetry and infra telemetry join on the same attributes.

## Perspectives
**Migration-economics.** OTel adoption is a cost and lock-in decision, not a technical-purity argument. Removing proprietary vendor agents recovers measurable CPU overhead and removes the business risk of vendor lock-in. The "Collector is the integration point" argument is really a hedge: it turns a future backend migration into a config change measured in weeks, instead of an app-by-app re-instrumentation effort measured in months.

**Protocol-design.** OTLP's choice of protobuf-over-gRPC/HTTP, instead of each vendor's bespoke wire format, is the specific enabling decision under "the Collector is the seam." Without a single standardized wire protocol, a general-purpose Collector with generic receivers and exporters couldn't exist — you'd be back to per-vendor glue.

**Operational-risk.** The two-tier topology introduces a new failure domain that didn't exist before: the gateway tier is now a single collection point whose outage or back-pressure affects every downstream signal simultaneously, across every team. Ordering `memory_limiter` first isn't a micro-optimization — it's what prevents a downstream backend hiccup from cascading into an org-wide telemetry blackout.

**Standards-governance.** Semantic-convention churn (`http.method` → `http.request.method`, and the still-shifting GenAI conventions) is a recurring tax, not a one-time migration cost. The correct staff response is to know and use OTel's stability-marking system — Experimental/Development through Stable — and to pin a schema version plus normalize via Collector-side `transform`, rather than treating conventions as either permanently frozen or too unstable to adopt.

## Real-world use cases
- **Cloudflare, "Adopting OpenTelemetry for our logging pipeline"** — https://blog.cloudflare.com/adopting-opentelemetry-for-our-logging-pipeline/ — Cloudflare replaced syslog-ng with the OTel Collector across their logging pipeline at millions of events/sec, showing the Collector doing real production duty beyond traces and metrics.
- **Grafana Labs, "How Grafana Labs enables horizontally scalable tail sampling in the OpenTelemetry Collector"** — https://grafana.com/blog/how-grafana-labs-enables-horizontally-scalable-tail-sampling-in-the-opentelemetry-collector/ — demonstrates directly that tail sampling can't scale horizontally without trace-ID-aware routing, and describes the fix (an aggregate processor forwarding spans between collector replicas) that this lesson's gateway design relies on.
- **Shopify's OTel migration, as reported by Dotan Horovits, "Shopify's Journey to Planet-Scale Observability"** — https://horovits.medium.com/shopifys-journey-to-planet-scale-observability-9c0b299a04dd — flagged explicitly as a third-party (journalist/analyst) write-up of a Shopify conference talk, not Shopify's own blog; describes Shopify building an in-house OTel-based stack to escape rising vendor costs and reported 15–20% proprietary-agent overhead.

## Worked example
**Two-tier config sketch.**

*Agent (DaemonSet), traces pipeline:*
`receivers: [otlp]` → `processors: [memory_limiter, k8sattributes, resourcedetection, filter, batch]` → `exporters: [otlp/gateway]`
- `memory_limiter` first (survive back-pressure), `k8sattributes`/`resourcedetection` add node/gpu_index/tenant, `filter` drops high-cardinality debug spans at the edge, `batch` last before shipping to the gateway over OTLP.

*Gateway (Deployment), traces pipeline:*
`receivers: [otlp]` (fronted by a trace-ID-aware `loadbalancing` exporter upstream, so all spans of a trace land on one instance) → `processors: [memory_limiter, tail_sampling, batch]` → `exporters: [otlp/backend]`
- `tail_sampling` policy: keep 100% of traces with an error status or latency > threshold, plus a low probabilistic sample of the rest.
- `batch` still runs *after* `tail_sampling` here too — the whole point of tail sampling is to shrink what gets batched and exported.

*Gateway (Deployment), logs pipeline (contrast):*
`receivers: [otlp]` fronted by a `loadbalancing` exporter keyed on **tenant/stream**, not trace ID — logs have no trace-completeness constraint, so the routing goal is even distribution and stream affinity, not co-locating a trace.

**The question the learner must answer:** why can't that `tail_sampling` processor live on the agent tier? Because the agent DaemonSet only sees the spans emitted on its own node, and the spans of a single distributed trace are scattered across many nodes. Only the gateway, behind trace-ID-aware routing, sees the complete trace and can apply a whole-trace keep/drop decision.

**Second question:** why would you route the logs pipeline's load balancer by tenant instead of trace ID, when the traces pipeline routes by trace ID? Because the routing key exists to satisfy a specific constraint — trace-ID routing satisfies "all spans of one trace, one instance," which logs don't need; tenant/stream routing satisfies even load distribution and stream affinity instead.

## Practice
<feeds [fleet observability design](../practice/fleet-observability/README.md)>

Write the tracing/telemetry section of the fleet-observability design: the two-tier Collector topology, both pipeline configs with processors in the correct order (justify `memory_limiter` first *and* `batch` last), the trace-ID-aware routing that makes gateway tail-sampling correct (and contrast it with a tenant-keyed routing choice for the logs pipeline), a convention-version normalization step in the Collector, the Collector self-observability metrics you'll alert on, and the GenAI-convention enrichment plan for the GPU fleet. State the migration sequence from the existing Prometheus/Jaeger stack.

## Common pitfalls
- **"The Collector is a passive proxy, so any topology works."** It isn't. It's a stateful pipeline processor for tail-sampling and batching — naively scaling a stateful stage horizontally without a trace-ID-aware routing layer silently corrupts tail-sampling decisions, exactly as described in Grafana Labs' own production writeup on the problem.
- **"Auto-instrumentation means the SDK stays thin automatically."** It doesn't guarantee it. Auto-instrumentation agents can bake in sampling and export decisions by default without you noticing. "Thin SDK" is a design discipline you enforce via Collector-side config review, not a property you get for free by choosing auto-instrumentation over manual.
- **"GenAI semantic conventions are stable, adopt as-is."** They're currently marked development/experimental, and attribute names have already changed across releases. Pin a version and normalize at the Collector — treat GenAI conventions the same as any other convention that hasn't reached Stable.
- **"Migrating to OTel means ripping out the old stack immediately."** Every real production example here (Cloudflare, Shopify) ran a dual-run or parity period before cutover. Big-bang migration off Prometheus/Jaeger/Fluent straight to OTel is the anti-pattern, not the goal.

## Self-check
- Why keep the SDK thin and push sampling/redaction/routing into the Collector? **Answer:** Anything baked into the SDK requires redeploying every app to change; the Collector is a central integration point where you swap backends, sampling, redaction, and enrichment for hundreds of services with zero app redeploys.
- Where must `tail_sampling` run and why can't it run on the agent DaemonSet? **Answer:** On the gateway tier behind trace-ID-aware routing, because a whole-trace keep/drop decision needs all spans of the trace co-located; agents only see the fragment emitted on their own node, since one distributed trace's spans are scattered across many nodes.
- Why is `memory_limiter` ordered first in the processor chain, and how do you absorb semantic-convention churn without redeploying apps? **Answer:** `memory_limiter` first so it sheds/refuses load before batches accumulate in RAM, preventing the Collector from OOMing under downstream back-pressure; handle convention churn (e.g. `http.method` → `http.request.method`) by pinning versions and normalizing old/new attribute names in the Collector via `transform`, not by re-releasing every service.
- Why does `batch` run last in the processor chain rather than early for throughput? **Answer:** Because filtering and tail-sampling happen before it — batching before filtering wastes CPU packing spans that are about to be dropped anyway; `batch` should compress only the data that survived sampling, not everything that arrived.
- A gateway fleet is horizontally scaled behind a plain load balancer and tail-sampling decisions start looking wrong for no visible error. What's the likely cause and the fix? **Answer:** Spans of individual traces are being split across different gateway replicas, so each replica makes an independent, partially-informed sampling decision — the fix is trace-ID-aware routing (e.g. a `loadbalancing` exporter or aggregate processor) that guarantees every span of a trace lands on the same replica before sampling.

## Connections & what's next
Builds on the storage-tier sizing in [03 — Metrics at scale](03-metrics-at-scale.md), whose remote-write pipeline this lesson's Collector can subsume via the `prometheus`/`prometheusremotewrite` receiver-exporter pair. The sampling *policy* question this lesson only introduces (head vs tail, and how to make tracing pay off with exemplars) is the full subject of the next lesson. Next: [05 — Distributed tracing](05-distributed-tracing.md).

## References & further reading
**Primary sources**
- OpenTelemetry Collector: https://opentelemetry.io/docs/collector/
- Semantic conventions: https://opentelemetry.io/docs/concepts/semantic-conventions/
- Collector configuration (processors/pipelines): https://opentelemetry.io/docs/collector/configuration/
- GenAI semantic conventions: https://opentelemetry.io/docs/specs/semconv/gen-ai/
- OTel semantic convention stability levels (document status): https://opentelemetry.io/docs/specs/otel/document-status/

**Real-world engineering blogs**
- Cloudflare, "Adopting OpenTelemetry for our logging pipeline": https://blog.cloudflare.com/adopting-opentelemetry-for-our-logging-pipeline/
- Grafana Labs, "How Grafana Labs enables horizontally scalable tail sampling in the OpenTelemetry Collector": https://grafana.com/blog/how-grafana-labs-enables-horizontally-scalable-tail-sampling-in-the-opentelemetry-collector/
- Dotan Horovits, "Shopify's Journey to Planet-Scale Observability" (third-party report on a Shopify talk, not Shopify's own blog): https://horovits.medium.com/shopifys-journey-to-planet-scale-observability-9c0b299a04dd
