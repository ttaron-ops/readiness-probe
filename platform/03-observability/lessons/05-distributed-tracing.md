---
lesson: "A03.5"
title: "Distributed tracing"
module: "A-03"
concept: "tail sampling & exemplars"
status: not-started
est_time: "3 hrs"
artifacts: ["tail-sampling-policy", "exemplar-linked-latency-panel"]
---

# A03.5 · Distributed tracing

> **Concept.** Tracing only pays off when you sample on outcome (tail) and wire metrics→trace navigation (exemplars) — otherwise it's a write-only data lake.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Why this matters

Most tracing deployments fail silently. They get instrumented, they cost money, and then nobody uses them — because at the moment an SLO breaks, the trace you needed was the one head sampling threw away, or the hop that broke was the one nobody instrumented. At staff level your job is not "turn on Jaeger"; it's to decide whether tracing earns its keep at all, and if so, to build the two mechanisms — outcome-based sampling and metric-to-trace linking — that convert a trace store from a cost center into the fastest path from "a graph looks wrong" to "here is the exact slow request." On a GPU fleet this is the difference between knowing goodput regressed and knowing *which* training step stalled on which kernel-launch boundary.

## Core notes

**Skip (you already know):** spans and parent/child causality; W3C `traceparent` context propagation; that Jaeger/Tempo exist as backends; that head sampling drops a fixed percentage at the root.

**Why tracing usually fails to pay off.** Four independent failure modes, any one of which kills value:
1. **Instrumentation gaps.** The causal chain is only as long as its weakest hop. One un-instrumented service (a legacy proxy, a third-party SDK, a raw thread pool that drops context) severs the trace — you get two disconnected fragments and no end-to-end latency. Enforce propagation at the framework/mesh layer, not per-service, so coverage is structural rather than voluntary.
2. **Head sampling discards the traces you needed.** Deciding at the root (before the request runs) is blind to outcome. The 0.1% of traces that errored or blew p99 are exactly what you sample away, because they're indistinguishable from the boring 99.9% at decision time.
3. **No metrics→trace navigation.** Without exemplars, an operator sees a bad latency graph and has no bridge into the trace store. Traces become write-only: ingested, stored, billed, never queried.
4. **Cost.** Storing 100% of spans is economically absurd, so teams under-sample uniformly — and then can't find anything, which loops back to (2).

**Head vs tail sampling — the core tradeoff.** This is the decision.
- **Head sampling:** decide at the root span before the outcome is known. Cheap (no buffering, decision is local), but structurally blind to errors and latency.
- **Tail sampling:** buffer *all spans of a trace* until it completes, then decide *after seeing the outcome*. This lets you keep 100% of errors, 100% of slow traces, and a small baseline of normal ones — the only sampling strategy that keeps what you actually investigate.

**Tail sampling's cost, and the #1 gotcha.** The collector must hold every span of an in-flight trace in memory until the decision fires (bounded by a decision-wait timeout). Two consequences:
- Memory sizing = (traces in flight) × (spans/trace) × (span size), held for the decision-wait window. This is real RAM and it is the binding constraint.
- **All spans of a trace must reach the same collector instance** — otherwise no single instance can see the whole trace to decide on it. This forces a two-tier topology: a gateway layer that load-balances by *trace-ID* (the `loadbalancing` exporter, keyed on trace-ID) in front of the tail-sampling collectors. Miss this and tail sampling silently makes decisions on partial traces. It is the single most common tail-sampling misconfiguration.

**Make tracing pay off.** Three moves:
- **Tail-sample on outcome:** keep 100% errors + 100% p99-latency traces + ~1% probabilistic baseline. The baseline gives you a control distribution; the outcome buckets give you every incident.
- **Exemplars** link a metric bucket to a representative trace. Mechanics: the SDK stamps the currently-active `trace_id`/`span_id` onto a histogram observation; Prometheus stores it via the OpenMetrics exemplars extension; Grafana renders a clickable jump from the histogram bar to that exact trace. An SLO breach on a p99 panel becomes one click to the slow trace — this is the navigation bridge that closes failure mode (3).
- **Derive RED metrics from spans** (span-metrics connector), so rate/errors/duration come free from the same instrumentation and stay consistent with the traces they summarize.

**GPU tie.** Trace a distributed training step or an inference request across the scheduler→pod→GPU-kernel-launch boundaries — the interesting latency lives at those handoffs. Exemplars connect a goodput or latency-regression metric to the specific slow step/trace, so a regression on a fleet-wide panel resolves to one stalled rank. Per-request inference traces carry `gen_ai.*` attributes for token-level latency attribution (time-to-first-token, inter-token latency). See the separately built GPU-observability artifact (DCGM / util-lie / MFU) for the metric side that these exemplars point back into.

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

**Justify the sizing.** Memory ≈ `num_traces` × spans/trace × avg span size, resident for up to `decision_wait`. If a trace averages 40 spans at ~1 KB, 100k in-flight traces ≈ 4 GB just for span buffers — that is your collector RAM floor, and `decision_wait` must exceed your p99 end-to-end latency or slow traces get decided before they finish (and are misclassified as fast). **Exemplar link:** the latency histogram feeding the SLO panel is emitted with exemplars on; the p99 bucket carries a `trace_id`, so a breach on the panel is one click from the exact trace this policy chose to keep.

## Practice

Design the tracing layer of the fleet observability system. Specify: (a) a tail-sampling policy with explicit error/latency/baseline buckets and the numeric `decision_wait`/`num_traces` you'd run; (b) the gateway load-balancing topology (why `loadbalancing` by trace-ID is mandatory, what breaks without it); (c) the memory budget derivation for the collector tier; (d) one exemplar-linked panel that navigates from a goodput/latency metric to a training-step trace. State the failure mode each choice defends against. <feeds [fleet observability design](../practice/fleet-observability/README.md)>

## Self-check

- Why can't head sampling keep "all errors + all slow traces"? **Answer:** Head sampling decides at the root span before the request runs, so the outcome (error status, final latency) is unknown at decision time — errored and slow traces are indistinguishable from normal ones when the decision is made. Only tail sampling, which buffers the whole trace and decides after completion, can select on outcome.
- What topology requirement does tail sampling impose, and why? **Answer:** All spans of a single trace must reach the same collector instance, because the tail decision needs to see the complete trace. This forces a gateway tier that load-balances by trace-ID (the `loadbalancing` exporter keyed on trace-ID) in front of the sampling collectors; without it, instances make decisions on partial traces. It's the most common tail-sampling misconfiguration.
- How does an exemplar physically connect a metric bucket to a trace? **Answer:** When the SDK records a histogram observation, it stamps the currently-active `trace_id`/`span_id` onto that observation. Prometheus stores it via the OpenMetrics exemplars extension, and Grafana renders a clickable jump from the histogram bar to that exact trace — turning a bad latency panel into one click to the representative slow trace.

## References

- https://opentelemetry.io/blog/2022/tail-sampling/
- https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/tailsamplingprocessor/README.md
- https://grafana.com/docs/tempo/latest/metrics-generator/
- https://opentelemetry.io/docs/specs/otel/metrics/data-model/#exemplars
