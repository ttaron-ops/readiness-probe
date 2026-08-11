---
lesson: "A03.5"
title: "Distributed tracing"
module: "A-03"
concept: "tail sampling & exemplars"
status: not-started
est_time: "4 hrs"
prev: "04-opentelemetry.md"
next: "06-logging-pipelines.md"
artifacts: ["tail-sampling-policy", "exemplar-linked-latency-panel", "trace-context-propagation-gap-analysis"]
sources: 8
---

# A03.5 · Distributed tracing

> **Concept.** Tracing only pays off when you sample on outcome (tail) and wire metrics→trace navigation (exemplars) — otherwise it's a write-only data lake.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 04 built the Collector as the integration point — the two-tier gateway pattern that lets you buffer and transform telemetry before it hits a backend. This lesson takes that gateway and asks what it's actually *for* in tracing: making tail sampling possible at all, since tail sampling requires every span of a trace to transit a single collector instance. That mechanism — and its cost curve — is the spine of this lesson. The next lesson (06) carries the same sampling-economics lens over to logs, where the tradeoff shifts from "which traces do I keep" to "which fields do I even index."

## Why this matters

Most tracing deployments fail silently. They get instrumented, they cost money, and then nobody uses them — because at the moment an SLO breaks, the trace you needed was the one head sampling threw away, or the hop that broke was the one nobody instrumented. At staff level your job is not "turn on Jaeger"; it's to decide whether tracing earns its keep at all, and if so, to build the two mechanisms — outcome-based sampling and metric-to-trace linking — that convert a trace store from a cost center into the fastest path from "a graph looks wrong" to "here is the exact slow request." On a GPU fleet this is the difference between knowing goodput regressed and knowing *which* training step stalled on which kernel-launch boundary.

## What's new here (calibration)

- Not "what is a span" — you know spans, parent/child causality, and W3C `traceparent` propagation. This lesson is about the two mechanisms (tail sampling, exemplars) that decide whether tracing is worth running at all.
- The tail-sampling cost model is non-obvious: it doesn't scale with what you keep, it scales with what you must *buffer to decide on* — a distinction that trips up capacity planning.
- Context propagation is not "solved by adding the SDK" — actor-model and non-RPC concurrency boundaries (queues, mailboxes, thread pools) need hand-wired propagation, and Discord's Elixir tracing work is a concrete, recent case study in exactly this gap.
- Query-time cardinality support matters as much as ingest-time sampling correctness — an org can nail tail sampling and still fail to realize the value if the query UI can't slice on high-cardinality dimensions.

## Core concepts

**Why tracing usually fails to pay off.** Four independent failure modes, any one of which kills value:
1. **Instrumentation gaps.** The causal chain is only as long as its weakest hop. One un-instrumented service (a legacy proxy, a third-party SDK, a raw thread pool that drops context) severs the trace — you get two disconnected fragments and no end-to-end latency. Enforce propagation at the framework/mesh layer, not per-service, so coverage is structural rather than voluntary.
2. **Head sampling discards the traces you needed.** Deciding at the root (before the request runs) is blind to outcome. The 0.1% of traces that errored or blew p99 are exactly what you sample away, because they're indistinguishable from the boring 99.9% at decision time.
3. **No metrics→trace navigation.** Without exemplars, an operator sees a bad latency graph and has no bridge into the trace store. Traces become write-only: ingested, stored, billed, never queried.
4. **Cost.** Storing 100% of spans is economically absurd, so teams under-sample uniformly — and then can't find anything, which loops back to (2).

**Head vs tail sampling — the core tradeoff.** This is the decision.
- **Head sampling:** decide at the root span before the outcome is known. Cheap (no buffering, decision is local), but structurally blind to errors and latency.
- **Tail sampling:** buffer *all spans of a trace* until it completes, then decide *after seeing the outcome*. This lets you keep 100% of errors, 100% of slow traces, and a small baseline of normal ones — the only sampling strategy that keeps what you actually investigate.

**Tail sampling's cost, and the #1 gotcha.** The collector must hold every span of an in-flight trace in memory until the decision fires (bounded by a decision-wait timeout). Two consequences:
- Memory sizing = (traces in flight) × (spans/trace) × (span size), held for the decision-wait window. This is real RAM and it is the binding constraint — and it's a **memory-time product**, not a function of sampled-out volume: 100% of traffic must transit the buffer, even though only a fraction is ultimately kept. This means the cost curve gets *more* expensive as traffic or trace-depth grow, independent of your final keep-rate.
- **All spans of a trace must reach the same collector instance** — otherwise no single instance can see the whole trace to decide on it. This forces a two-tier topology: a gateway layer that load-balances by *trace-ID* (the `loadbalancing` exporter, keyed on trace-ID, hashing the trace ID against a discovered set of downstream collectors via DNS/k8s SD/static list) in front of the tail-sampling collectors. Miss this and tail sampling silently makes decisions on partial traces. It is the single most common tail-sampling misconfiguration.

**The `decision_wait` correctness constraint.** `decision_wait` must exceed your **p99.9** end-to-end latency, not p99. Any trace whose true duration exceeds `decision_wait` gets a keep/drop decision made on a truncated view of itself — a slow-tail trace can be misclassified as "fast" (because it hadn't finished yet) and dropped by a baseline-probabilistic policy purely due to timing, not because it was uninteresting. Undersizing this window quietly discards exactly the outliers tail sampling exists to catch.

**Load-balancing exporter mechanics and its own scaling cost.** The `loadbalancing` exporter must maintain a live connection pool to every downstream tail-sampling collector and route per-trace by consistent hash of trace ID. This N×M connection fan-out becomes its own throughput ceiling at fleet scale, and — because it's a hash over a discovered replica set — **every scale event on the collector tier triggers rebalancing**: as replicas are added or removed, some fraction of trace-ID hashes remap to a different downstream instance, briefly misrouting in-flight spans mid-scale-event. Grafana Labs' aggregate-processor design addresses this by reducing the connection fan-out, rather than eliminating the rebalancing cost outright.

**Make tracing pay off.** Three moves:
- **Tail-sample on outcome:** keep 100% errors + 100% p99-latency traces + ~1% probabilistic baseline. The baseline gives you a control distribution; the outcome buckets give you every incident.
- **Exemplars** link a metric bucket to a representative trace. Mechanics: the SDK stamps the currently-active `trace_id`/`span_id` onto a histogram observation; Prometheus stores it via the OpenMetrics exemplars extension; Grafana renders a clickable jump from the histogram bar to that exact trace. An SLO breach on a p99 panel becomes one click to the slow trace — this is the navigation bridge that closes failure mode (3). Note the cardinality cap: Prometheus stores by default up to one exemplar per unique label-set per series in a fixed-size circular buffer (default on the order of 100,000 total across a process) — at fleet scale with many bucket/tenant combinations this buffer can silently undersize and start evicting exemplars you assumed were retained.
- **Derive RED metrics from spans** (span-metrics connector), so rate/errors/duration come free from the same instrumentation and stay consistent with the traces they summarize — with a caveat: see pitfalls below.

**GPU tie.** Trace a distributed training step or an inference request across the scheduler→pod→GPU-kernel-launch boundaries — the interesting latency lives at those handoffs. Exemplars connect a goodput or latency-regression metric to the specific slow step/trace, so a regression on a fleet-wide panel resolves to one stalled rank. Per-request inference traces carry `gen_ai.*` attributes for token-level latency attribution (time-to-first-token, inter-token latency). See the separately built GPU-observability artifact (DCGM / util-lie / MFU) for the metric side that these exemplars point back into.

## Perspectives

**Cost-accounting.** Tail sampling trades ingest-time selectivity for a memory-time-product cost: `num_traces × spans/trace × span size`, held for the `decision_wait` window. This is worth internalizing as a curve, not a constant — it gets more expensive as traffic or trace depth grow, *not* linearly with how much you sample out, because every trace must transit the buffer regardless of its eventual fate. Sizing this correctly is capacity planning for a cost that scales with your busiest moments, not your average keep-rate.

**Protocol/plumbing.** The `loadbalancing` exporter is the unglamorous but load-bearing piece: it hashes trace IDs to pick a downstream collector from a discovered set (DNS, k8s service discovery, or a static list). That hash-based routing has a scaling cost of its own — whenever the collector-replica set scales up or down, a fraction of trace-ID hashes remap, briefly misrouting spans mid-scale-event. Design reviews of tail-sampling topologies should ask explicitly what happens to in-flight traces during a collector autoscale event.

**Org-adoption.** Tracing pays off fastest when the *query* experience, not just the ingestion pipeline, supports high-cardinality, high-dimension slicing. An org can ship tail sampling correctly — outcome-based keep policies, a solid gateway topology, sane `decision_wait` sizing — and still fail to realize the value if the trace query UI only lets you filter on a handful of low-cardinality fields. The ingestion investment is necessary but not sufficient; the payoff shows up at query time.

**Concurrency-model (the standard tooling doesn't always fit).** Discord's Elixir/actor-model tracing work is a useful "when does the standard model break" case. OTel's context-propagation assumption is that context rides along a call stack — a synchronous chain of function calls. In an actor system, messages (not calls) cross concurrency boundaries: a process sends a message to another process's mailbox, and there's no call stack connecting sender to receiver for context to ride on. Discord had to build a custom transport layer to propagate trace context across actor message passing, with dynamic sampling to handle million-user fanout. The general lesson: before adopting a tracing standard wholesale, check whether your concurrency model matches the standard's propagation assumption — event-driven, actor-based, and message-queue-mediated systems often don't.

## Real-world use cases

- **Uber — "Evolving Distributed Tracing at Uber Engineering."** https://www.uber.com/en-IN/blog/distributed-tracing/ — The origin story of Jaeger and its move toward post-trace (tail-based) sampling buffered in agents. This is the primary historical account of *why* tail sampling was invented — worth reading as the origin of the mechanism this lesson centers on.
- **Grafana Labs — "How Grafana Labs enables horizontally scalable tail sampling in the OpenTelemetry Collector."** https://grafana.com/blog/how-grafana-labs-enables-horizontally-scalable-tail-sampling-in-the-opentelemetry-collector/ — The most directly on-topic source for this lesson's core mechanism: the `loadbalancing` exporter, the all-spans-same-instance requirement, and how to make it scale.
- **Honeycomb — "Honeycomb Users Are Living in the Future, Part 1: Sampling."** https://www.honeycomb.io/blog/honeycomb-users-living-in-future-pt-1-sampling — Honeycomb's Refinery does dynamic, outcome-aware tail sampling and reconstructs sampled-out data into charts and SLOs. A concrete picture of what tracing looks like when the sampling strategy is good enough that it actually pays off.
- **InfoQ — "Discord Engineers Add Distributed Tracing to Elixir's Actor Model without Performance Penalty."** https://www.infoq.com/news/2026/03/discord-elixir-actor-tracing/ — Discord built a custom Transport library to propagate trace context across Elixir's actor-model message passing, with dynamic sampling for million-user fanout. Flagged as a 2026 report — recent, and the best available concrete example of the concurrency-model perspective above.

## Worked example

Design a tail-sampling policy for an inference gateway. The policy composite:

```yaml
tail_sampling:
  decision_wait: 10s          # buffer window before deciding
  num_traces: 100000          # in-flight traces held in memory
  policies:
    - name: errors
      type: status_code
      status_code: { status_codes: [ERROR] }     # 100% of errors
    - name: slow
      type: latency
      latency: { threshold_ms: 800 }              # ~p99, 100% kept
    - name: baseline
      type: probabilistic
      probabilistic: { sampling_percentage: 1 }   # 1% control sample
```

Fronted by a gateway tier running the `loadbalancing` exporter keyed on `traceID`, so every span of a given trace lands on the same tail-sampling collector.

**Justify the sizing.** Memory ≈ `num_traces` × spans/trace × avg span size, resident for up to `decision_wait`. If a trace averages 40 spans at ~1 KB, 100k in-flight traces ≈ 4 GB just for span buffers — that is your collector RAM floor, and `decision_wait` must exceed your **p99.9** (not p99) end-to-end latency or slow traces get decided before they finish (and are misclassified as fast). **Exemplar link:** the latency histogram feeding the SLO panel is emitted with exemplars on; the p99 bucket carries a `trace_id`, so a breach on the panel is one click from the exact trace this policy chose to keep.

**Span-metrics caveat, made concrete.** If the span-metrics connector sits *after* this tail-sampling policy, the RED metrics it derives are computed only from the ~1% baseline plus 100% of errors/slow traces — a rate metric built from that surviving population is skewed toward errors and slow requests, and cannot be trusted as an unbiased estimate of true request volume or error rate. To get correct RED metrics alongside correct tail-sampled traces, derive the span-metrics connector's input *before* the tail-sampling processor in the pipeline, not after.

## Practice

Design the tracing layer of the fleet observability system. Specify: (a) a tail-sampling policy with explicit error/latency/baseline buckets and the numeric `decision_wait`/`num_traces` you'd run, justified against p99.9 latency; (b) the gateway load-balancing topology (why `loadbalancing` by trace-ID is mandatory, what breaks without it, and what happens to in-flight traces during a collector-tier scale event); (c) the memory budget derivation for the collector tier; (d) one exemplar-linked panel that navigates from a goodput/latency metric to a training-step trace, with a note on the exemplar buffer's cardinality cap; (e) whether your RED metrics are derived pre- or post-tail-sampling, and why that ordering matters. State the failure mode each choice defends against. <feeds [fleet observability design](../practice/fleet-observability/README.md)>

## Common pitfalls

- **"Sampling is exclusively about cost control."** It's equally about signal quality. Uniform undersampling doesn't just save money — it destroys the ability to find the incident-relevant trace later, because the one trace you need is statistically likely to be the one you threw away.
- **"Exemplars require 100% trace sampling to be useful."** They don't — exemplars just need a representative `trace_id` per bucket per scrape. Tail sampling's "keep 100% errors/slow" policy actually *improves* exemplar usefulness, because the trace an exemplar points to is more likely to be the interesting one, not a random survivor of uniform sampling.
- **"Tail sampling replaces the need for head sampling."** It doesn't — tail sampling still requires ingesting 100% of spans up to the gateway, since the decision can't be made until the trace is buffered. Extremely high-volume systems often combine a coarse head-sample *before* the gateway with tail sampling on the remainder, to bound the buffer's input volume.
- **"Context propagation is solved once you add the OTel SDK."** As Discord's actor-model case shows, propagation across non-HTTP/RPC boundaries — actor mailboxes, message queues, thread pools that don't propagate context — requires explicit, hand-wired propagation. The SDK solves the call-stack case; it does not solve the message-passing case for free.
- **"Deriving RED metrics from spans is always safe."** If the span-metrics connector runs downstream of a tail-sampling processor, the derived rate/error/duration metrics inherit the sampling policy's bias — they are not an unbiased estimate of true traffic unless computed pre-sampling.

## Self-check

- Why can't head sampling keep "all errors + all slow traces"? **Answer:** Head sampling decides at the root span before the request runs, so the outcome (error status, final latency) is unknown at decision time — errored and slow traces are indistinguishable from normal ones when the decision is made. Only tail sampling, which buffers the whole trace and decides after completion, can select on outcome.
- What topology requirement does tail sampling impose, and why? **Answer:** All spans of a single trace must reach the same collector instance, because the tail decision needs to see the complete trace. This forces a gateway tier that load-balances by trace-ID (the `loadbalancing` exporter keyed on trace-ID) in front of the sampling collectors; without it, instances make decisions on partial traces. It's the most common tail-sampling misconfiguration.
- Why must `decision_wait` exceed p99.9 latency rather than p99? **Answer:** Any trace whose true duration exceeds `decision_wait` gets a keep/drop decision made on a truncated view — it hasn't finished yet, so a genuinely slow trace can look "fast" (or simply incomplete) at decision time and get dropped by a baseline policy. Sizing to p99 would still truncate roughly 1 in 100 traces at their most interesting (slowest) moment; p99.9 sizing captures the outliers tail sampling exists to keep.
- How does an exemplar physically connect a metric bucket to a trace, and what's its main capacity gotcha? **Answer:** When the SDK records a histogram observation, it stamps the currently-active `trace_id`/`span_id` onto that observation. Prometheus stores it via the OpenMetrics exemplars extension, in a fixed-size circular buffer capped by default around one exemplar per label-set (roughly 100,000 total) — at fleet scale with many bucket/tenant combinations, this buffer can silently undersize and evict exemplars you assumed were retained.
- Why does Discord's actor-model tracing case matter beyond Elixir specifically? **Answer:** It demonstrates that OTel's default context-propagation model assumes a call stack connecting caller and callee, which doesn't exist across actor-model message passing (or any queue/mailbox boundary). Any event-driven or actor-based system needs explicit, hand-wired context propagation across those boundaries — the SDK does not solve it automatically, which is a general caution against assuming "add the SDK" is sufficient coverage.

## Connections & what's next

This lesson builds directly on lesson 04's Collector/two-tier gateway pattern — tail sampling *is* the reason that gateway topology exists. It also sets up lesson 09's GPU/ML observability synthesis, where exemplar-linked traces are the mechanism that turns a fleet-wide goodput regression into a single stalled rank. Next: [06 · Logging pipelines](06-logging-pipelines.md), which carries the sampling-economics lens from traces over to logs, where the tradeoff shifts from "which traces do I keep" to "which fields do I even index."

## References & further reading

**Primary sources**
- OpenTelemetry, "Tail Sampling in OpenTelemetry" — https://opentelemetry.io/blog/2022/tail-sampling/
- OpenTelemetry Collector Contrib, `tailsamplingprocessor` README — https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/tailsamplingprocessor/README.md
- Grafana Tempo, metrics-generator docs — https://grafana.com/docs/tempo/latest/metrics-generator/
- OpenTelemetry, metrics data model — exemplars spec — https://opentelemetry.io/docs/specs/otel/metrics/data-model/#exemplars

**Real-world engineering blogs**
- Uber Engineering, "Evolving Distributed Tracing at Uber Engineering" — https://www.uber.com/en-IN/blog/distributed-tracing/
- Grafana Labs, "How Grafana Labs enables horizontally scalable tail sampling in the OpenTelemetry Collector" — https://grafana.com/blog/how-grafana-labs-enables-horizontally-scalable-tail-sampling-in-the-opentelemetry-collector/
- Honeycomb, "Honeycomb Users Are Living in the Future, Part 1: Sampling" — https://www.honeycomb.io/blog/honeycomb-users-living-in-future-pt-1-sampling

**Deeper dives**
- InfoQ, "Discord Engineers Add Distributed Tracing to Elixir's Actor Model without Performance Penalty" (2026) — https://www.infoq.com/news/2026/03/discord-elixir-actor-tracing/
