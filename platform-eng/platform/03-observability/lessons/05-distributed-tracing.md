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
sources: 14
---

# A03.5 · Distributed tracing

> **Concept.** Tracing only pays off when you sample on outcome (tail) and wire metrics→trace navigation (exemplars) — otherwise it's a write-only data lake.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 04 built the Collector as the integration point and established *where* tail sampling can physically happen: on a gateway tier, behind trace-ID-keyed routing, because a whole-trace predicate needs the whole trace. This lesson is about *what to sample and whether tracing earns its keep at all* — the policy question, the cost curve behind it, and the two mechanisms that convert a trace store from a line item into the fastest path from "a graph looks wrong" to "here is the request that made it wrong."

The next lesson carries the same economics lens to logs, where the decision shifts from "which traces do I keep" to "which fields do I even index."

Everything below is checked against the **OpenTelemetry specification** (`open-telemetry/opentelemetry-specification`, `main`, August 2026 — in particular `specification/trace/tracestate-probability-sampling.md` and `tracestate-handling.md`), the **W3C Trace Context** specification repository (`w3c/trace-context`, `main`), **Grafana Tempo** (`main`, August 2026 — 3.x architecture), the **OpenTelemetry Collector Contrib** tail-sampling processor, and **Jaeger v2.20.0**. Where a mechanism is marked Development upstream, that is stated.

## Why this matters

Most tracing deployments fail silently. They get instrumented, they cost money, and then nobody uses them — because at the moment an SLO breaks, the trace you needed was the one head sampling threw away, or the hop that broke was the one nobody instrumented, or the latency panel had no link into the trace store and the operator went to logs instead.

At staff level your job is not "turn on Jaeger." It is to decide whether tracing earns its keep at all, and if so, to build the two mechanisms that make it pay: **outcome-based sampling**, so the traces you keep are the ones you will want, and **metric-to-trace linking**, so the traces you kept are reachable from the panel that made you look.

On a GPU fleet the payoff is specific. Knowing that goodput regressed is a metric question. Knowing *which* training step stalled, on which rank, at which collective, is a trace question — and the difference between a 20-minute diagnosis and a 4-hour one on a 512-GPU job is measured in thousands of dollars of idle silicon per incident.

The cost of getting it wrong is also specific and quantifiable. 100%-sampled tracing on a 20,000 req/s service is hundreds of gigabytes a day; uniform 1% sampling is affordable and useless. The entire discipline is in the middle.

## What's new here (calibration)

- **Skip (you already know):** what a span is; parent/child causality; that `traceparent` exists; that Jaeger and Tempo store traces.
- **New:** the W3C Trace Context wire format field by field — the 2-hex version, 32-hex trace ID, 16-hex parent ID, the **8 flag bits including the Level-2 `random` bit**, and the `tracestate` limits (32 list-members, ~512 characters) that make propagation a *budget*, not a free ride.
- **New:** OpenTelemetry's **consistent probability sampling** — the `th` rejection threshold and `rv` randomness value in `tracestate`, the `R >= T ⇒ keep` rule, and the **adjusted count** that makes span-derived metrics statistically valid even under sampling. This is the real answer to the span-metrics-bias problem, and it is far better than "compute metrics before sampling."
- **New:** tail sampling's cost as a **memory–time product** with the arithmetic worked, plus the two failure metrics that tell you which side of it you got wrong.
- **New:** how a trace backend physically stores and searches traces — Tempo's Kafka-as-WAL write path, Parquet columnar blocks, RF=1 durability, and why "search by attribute" is a columnar scan rather than an index lookup. That is what makes the storage cheap and the query cost shaped the way it is.
- **Corrected:** Prometheus's exemplar storage is a **single global circular buffer** shared by all series (`tsdb/exemplar.go`), not "one exemplar per label set per series." The eviction behaviour that follows is different and more surprising.
- **Corrected:** `decision_wait` must exceed the p99.9 *trace* duration, which is not the same as p99.9 request latency when traces have async tails.

## Core concepts

### 1. Why tracing usually fails to pay off

Four independent failure modes. Any one of them is sufficient to make the whole investment worthless, which is why "we have tracing" and "tracing is useful here" are only weakly correlated.

**(1) Instrumentation gaps.** A trace is a chain and it is exactly as long as its weakest hop. One un-instrumented service — a legacy proxy, a vendor SDK, a raw thread pool that drops context — severs it. You do not get a warning; you get two disconnected fragments, each of which looks like a complete short trace. The end-to-end latency you wanted is simply not representable.

**(2) Head sampling discards what you needed.** Deciding at the root, before the request runs, is structurally blind to outcome. The 0.1% of traces that errored or blew p99 are indistinguishable from the boring 99.9% at decision time.

**(3) No metrics→trace navigation.** Without exemplars, an operator looking at a bad latency panel has no bridge into the trace store. They must guess a time range and a service and go hunting. Most will go to logs instead, and the trace store becomes write-only: ingested, stored, billed, never queried.

**(4) Cost.** Storing 100% of spans is economically absurd at any real volume, so teams uniformly under-sample — which loops straight back to (2).

**The through-line:** (2), (3) and (4) are one problem with one shape. You need to *select* on outcome and *navigate* from the cheap signal. Everything else in this lesson is the machinery for those two verbs.

### 2. The propagation substrate: W3C Trace Context, field by field

Before sampling policy, the thing sampling decisions ride on. This is the one part of tracing that must live in the process (lesson 4 §2), so it is the one part you cannot fix in the Collector.

```
   ONE REQUEST, FOUR HOPS — WHAT TRAVELS ON THE WIRE AT EACH BOUNDARY
   ═══════════════════════════════════════════════════════════════════════════

   [browser] ──HTTP──▶ [api-gw] ──gRPC──▶ [router] ──HTTP──▶ [vllm] ──▶ [GPU]
                                              │
                                              └──enqueue──▶ [kafka] ──▶ [logger]

   ── HOP 1: browser → api-gw ───────────────────────────────────────────────
   traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-03
                └┬┘ └───────────────┬──────────────┘ └──────┬───────┘ └┬┘
                 │                  │                       │          │
       version (2 hex, "00")        │                       │          │
                        trace-id (32 hex = 16 B)            │          │
                        all-zero is INVALID                 │          │
                                             parent-id (16 hex = 8 B)  │
                                             the CALLER's span id      │
                                             all-zero is INVALID       │
                                                       trace-flags (2 hex, 8 bits)
                                                       bit 0 (0x01) = sampled
                                                       bit 1 (0x02) = random  ◀ Level 2
                                                       03 = sampled AND the
                                                            right-most 7 bytes
                                                            of trace-id are random
   tracestate: ot=th:fd70a4,vendorx=abc123
               └──────┬─────┘ └─────┬─────┘
                      │             └─ other vendors' entries, preserved verbatim
                      └─ the OTel entry: rejection threshold (§4)

   ── HOP 2: api-gw → router (gRPC) ─────────────────────────────────────────
   Same trace-id. NEW parent-id = api-gw's span id.
   traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-a1b2c3d4e5f60718-03
                                                    └──── changed ────┘

   ── HOP 3: router → vllm (HTTP) ───────────────────────────────────────────
   Same trace-id. parent-id = router's span id.

   ── HOP 4: router → kafka → logger  ◀── THE HOP THAT BREAKS ───────────────
   There is no HTTP request here. Context must be injected into the MESSAGE:
       Kafka record headers:  traceparent = 00-4bf92f…-…-03
                              tracestate  = ot=th:fd70a4
   If the producer does not inject and the consumer does not extract,
   the logger's spans start a NEW trace. You get two fragments and no
   causal link, with no error anywhere.

   ── AND THE HOP THAT IS INVISIBLE ─────────────────────────────────────────
   [vllm] → [GPU]. There is no propagation protocol into CUDA. The GPU work
   is represented by a span in the vllm process, annotated with device
   attributes — never by a span the GPU emits. Trace/kernel correlation is
   a separate mechanism (lesson 9), not a propagation problem.
```

**Constraints worth committing to memory**, from the W3C specification repository:

| Field | Constraint |
|---|---|
| `version` | 2 lowercase hex digits; `ff` invalid; `00` is the version this format defines |
| `trace-id` | 32 lowercase hex (16 bytes); all-zero invalid; SHOULD be globally unique |
| `parent-id` | 16 lowercase hex (8 bytes); all-zero invalid; changes at every hop |
| `trace-flags` | 2 hex = 8 bits. `FLAG_SAMPLED = 0x01`, `FLAG_RANDOM = 0x02` |
| `tracestate` | at most **32 list-members**; keys up to 256 chars; vendors SHOULD propagate at least **512 characters** total; on overflow the **right-most** member is dropped |

Three consequences that matter operationally:

**The `random` flag is not decoration.** W3C Trace Context Level 2 says: if the right-most 7 bytes of the trace ID are randomly generated, set bit 1. That flag is what lets a downstream sampler use the trace ID itself as its source of randomness (§4) instead of needing an explicit `rv` value. If your SDK generates trace IDs in a way that is not uniformly random in those bytes — some legacy formats encode a timestamp there — and still sets the flag, consistent sampling downstream is silently biased.

**`tracestate` is a budget with an eviction policy.** 32 members, ~512 characters, right-most dropped on overflow. In a mesh with several vendors' proxies each adding an entry, the OTel `ot=` entry can be evicted, which silently disables consistent sampling downstream. Audit what adds `tracestate` entries in your request path.

**Restarting a trace is a legitimate operation, and it destroys causality.** The spec explicitly permits regenerating all three fields at a "front gate into a secure network" as a DoS-mitigation measure. If an ingress proxy is configured that way, every trace begins at the ingress and the client-side spans are orphaned. Check your ingress configuration before concluding that browser instrumentation is broken.

**The propagation gap analysis is a real deliverable.** Enumerate every boundary in your request path and classify it:

| Boundary type | Propagates by default? | What to do |
|---|---|---|
| HTTP / gRPC between instrumented services | yes | verify the propagator is W3C, not B3-only |
| Service mesh sidecar (Envoy) | yes, if configured | Envoy must be told to propagate; it will not invent a trace |
| Message queue (Kafka, SQS, NATS) | **no** | inject/extract in message headers; the SDK's messaging instrumentation may do it |
| Thread pool / executor | **usually no** | the context is a thread-local or an async-local; submitting to a pool loses it unless the pool is instrumented |
| Actor mailbox (Erlang/Elixir, Akka) | **no** | there is no call stack; context must ride in the message envelope |
| Cron / batch job | **no** | there is no caller; start a root span and link it to whatever caused the schedule |
| Database driver | partial | most produce a client span but do not propagate into the server |
| GPU kernel launch | **n/a** | no propagation protocol exists; annotate a host-side span instead |

The rows marked **no** are where tracing quietly breaks, and they are exactly the boundaries a call-stack-based context model cannot cross. Discord's Elixir work is the canonical public example: in an actor system, messages rather than calls cross concurrency boundaries, so there is no stack for context to ride on, and they had to build a transport layer to carry it explicitly.

### 3. Head vs tail sampling — the core decision

**Head sampling** decides at the root, before the request runs, and propagates the decision in the `sampled` flag so every downstream service agrees. It is cheap: no buffering, decision is local, and unsampled spans are never created, so there is no serialisation or network cost either. It is structurally blind: at decision time the request has not errored yet and has not been slow yet.

**Tail sampling** buffers all spans of a trace until the trace is judged complete, then decides with the outcome in hand. It can keep 100% of errors and 100% of slow traces — the only strategy that keeps what you will actually investigate.

```
   THE SAME 1,000 REQUESTS, TWO STRATEGIES
   ═══════════════════════════════════════════════════════════════════════════
   Population: 1,000 requests. 3 errored. 10 exceeded 2 s. 987 normal.

   ── HEAD SAMPLING at 1 % ─────────────────────────────────────────────────
   decision made HERE, at t=0, before anything happened:
      t=0 ─┬─▶ req 1    keep?  hash(traceid) → drop
           ├─▶ req 2           drop
           …
           └─▶ req 743         KEEP     ← a perfectly normal request
   Expected kept: 10 traces, of which
      errors kept   = 3 × 0.01   = 0.03    ⇒ 97 % chance you keep ZERO errors
      slow kept     = 10 × 0.01  = 0.10    ⇒ 90 % chance you keep ZERO slow traces
   You paid for a trace store and it contains 10 boring requests.

   ── TAIL SAMPLING, decision_wait 20 s ────────────────────────────────────
   ALL 1,000 traces are ingested and buffered. Decision at trace completion:
      t=0 ──── spans stream in, held in memory ────▶ t=20s: decide
                                                      │
              ┌───────────────────────────────────────┤
              ▼                    ▼                  ▼
         3 errors            10 slow            987 normal
         status_code policy  latency policy     probabilistic 1 %
         KEEP 100 %          KEEP 100 %         KEEP ~10
   Kept: 23 traces — and they include every incident-relevant one.

   ── THE COST DIFFERENCE ──────────────────────────────────────────────────
   Head:  1 % of spans ever created, serialised, or sent.  Cheap end to end.
   Tail:  100 % of spans created, serialised, sent to the gateway, and held
          in gateway RAM for up to decision_wait. Only STORAGE is reduced.
          You pay full price for produce + transport; you save only on
          store + query.
```

**That last box is the point people miss.** Tail sampling does not reduce instrumentation overhead, serialisation cost, or network bandwidth from app to gateway. It reduces *backend storage and query cost only*. If your bottleneck is application CPU spent creating spans, tail sampling does not help and head sampling does.

**Which is why the real answer is usually both.** At extreme volume, a coarse head sample (say 25%) bounds the input to the tail-sampling tier, and tail sampling then selects on outcome within that 25%. You lose 75% of errors, which is fine when you have thousands of errors a minute and only need a representative sample of each error *type*. The composition rule: head-sample only when the population of interesting traces is large enough that a sample of it is still useful.

### 4. Consistent probability sampling — how sampled data stays statistically valid

This is the piece that turns sampling from a lossy compromise into a measurement technique, and it is the single most under-known mechanism in the tracing stack.

**The problem.** Different participants in a trace — the root SDK, an intermediate SDK, an agent Collector, a gateway Collector — may each want to sample at a different rate. Naively, that produces incoherent traces: a parent kept at 10% and a child kept at 1% gives you children without parents. And any metric derived from the surviving spans is biased by an unknown factor.

**The mechanism** (OpenTelemetry `specification/trace/tracestate-probability-sampling.md`). Two 56-bit quantities:

- **`R`, the randomness value.** Either an explicit value carried in the `ot` `tracestate` entry as `rv:`, or — per W3C Trace Context Level 2 — the least-significant 56 bits of the trace ID, valid as randomness precisely when the `random` flag (§2) is set.
- **`T`, the rejection threshold.** Derived from the sampling probability:

  ```
  T = (1 − sampling_probability) × 2^56
  ```

  and carried in the `ot` entry as `th:`, hex-encoded with trailing zeros stripped.

**The decision rule is one comparison:**

```
   if R >= T  →  KEEP the span
   else       →  DROP
```

Worked, with the spec's own example:

```
   SAMPLING PROBABILITY → THRESHOLD → tracestate ENCODING
   ═══════════════════════════════════════════════════════════════════════

   p = 1.00 (keep everything)
       T = (1 − 1.00) × 2^56 = 0                 → ot=th:0

   p = 0.01 (keep 1 %)
       T = (1 − 0.01) × 2^56
         = 0.99 × 72,057,594,037,927,936
         ≈ 71,337,018,784,743,424
         = 0xfd70a400000000                       → ot=th:fd70a4
                                                     (trailing zeros stripped)

   p = 0.10 (keep 10 %)
       T = 0.90 × 2^56 ≈ 0xe6666666666666        → ot=th:e66666

   Valid probability range: 2^-56 through 1. The 56 comes from the
   7 random bytes W3C Trace Context Level 2 guarantees in a trace ID.
   Zero is NOT a probability — "never sample" is not probability sampling.
```

**Why this makes the sampling *consistent*.** Every participant compares the *same* `R` against its *own* `T`. A span kept at probability `p1` is necessarily kept by any participant sampling at `p2 >= p1`, because a larger probability means a smaller threshold. So you can never end up with a child kept while its parent was dropped, as long as everyone uses the same `R`. That property is the definition the spec gives for a consistent sampling decision, and it is what makes coherent traces possible across independently-configured participants.

**The payoff: adjusted count.** The spec defines the **adjusted count** as the reciprocal of the sampling probability — the number of items this one item represents in the unsampled population. A span kept at 1% has an adjusted count of 100. Because `th` is propagated, a downstream consumer can *read the effective sampling probability off the span itself* and weight accordingly:

```
   ESTIMATING TRUE VOLUME FROM SAMPLED SPANS
   ═══════════════════════════════════════════════════════════════════════
   kept spans and their ot=th: values
      1,000 spans at th:0        →  p = 1.00   →  adjusted count 1     each
        200 spans at th:e66666   →  p = 0.10   →  adjusted count 10    each
         50 spans at th:fd70a4   →  p = 0.01   →  adjusted count 100   each

   estimated true span count
      = 1,000×1 + 200×10 + 50×100
      = 1,000 + 2,000 + 5,000
      = 8,000 spans

   A naive count of the survivors gives 1,250 — off by 6.4×, and the
   error changes whenever anyone retunes a sampler.
```

**This is the correct fix for the span-metrics bias problem**, and it is better than the usual advice. The usual advice — "derive RED metrics *before* the tail-sampling processor" — works only if all your metric derivation happens in one pipeline stage upstream of all sampling. Adjusted counts work everywhere, including in the backend, including across multiple independent sampling stages, and including for policies you added last week. The spec calls this out explicitly: the tracestate and threshold-management requirements exist "primarily for this purpose."

**The operational caveat:** consistent probability sampling is marked **Development** in the OTel spec, and support varies by SDK and by Collector component. The tail-sampling processor's `probabilistic` policy gains tracestate-based behaviour only behind the `processor.tailsamplingprocessor.usetracestate` feature gate. Check what your SDKs actually emit before building an accounting story on adjusted counts — but know that this is where the ecosystem is going, and that "we cannot get unbiased metrics from sampled traces" is no longer true in principle.

### 5. Tail sampling's cost: a memory–time product

The gateway must hold every span of every in-flight trace until it decides. That gives a cost model with an unusual shape.

```
   memory ≈ trace_rate × decision_wait × spans_per_trace × bytes_per_span
            └──────────┬──────────────┘
                   in-flight traces

   Note what is ABSENT from that expression: the keep rate.
   Cost is independent of how much you throw away. You buffer 100 %
   regardless. Sampling reduces STORAGE, not BUFFER.
```

Worked for the fleet from lesson 4:

```
   GATEWAY MEMORY SIZING — 20,000 traces/s, 5 gateway replicas
   ═══════════════════════════════════════════════════════════════════════
   per gateway                    20,000 / 5            = 4,000 traces/s
   decision_wait                                        = 20 s
   in-flight traces               4,000 × 20            = 80,000
   spans per trace (measured p50)                       = 12
   bytes per span (protobuf, in memory, incl. attrs)    ≈ 400 B

   span buffer  80,000 × 12 × 400 B                     ≈ 384 MB
   num_traces must exceed in-flight with margin         → 200,000
   worst case at 200,000 × 12 × 400 B                   ≈ 960 MB
   + decision caches (1 M sampled + 2 M non-sampled
     trace IDs × ~40 B)                                 ≈ 120 MB
   + receiver buffers, batch, Go heap slack (×1.6)      ≈ 1.7 GB
   ─────────────────────────────────────────────────────────────────
   container limit 8 GiB, memory_limiter limit_mib 6144, spike 1229
```

**Two failure metrics tell you which way you got it wrong**, and they are the ones to graph:

- `otelcol_processor_tail_sampling_sampling_trace_dropped_too_early` — traces evicted because `num_traces` was reached before their decision fired. Fix: raise `num_traces`, lower `decision_wait`, or add replicas.
- `otelcol_processor_tail_sampling_sampling_late_span_age` — how long after the decision stragglers arrive. If the p99 of this is large, raise `decision_wait` or (cheaper) enable the decision caches so late spans inherit the decision that was already made.

**`decision_wait` must exceed the p99.9 *trace* duration.** Not the p99.9 request latency — these differ whenever a request spawns asynchronous work. A user-facing request that returns in 200 ms but kicks off a background job producing spans for 45 seconds has a *trace* duration of 45 s. Set `decision_wait` from the request latency and every such trace is decided on a truncated view, where it looks fast and gets dropped by the baseline probabilistic policy. You have systematically discarded exactly the traces with interesting async behaviour. Measure trace duration directly — in Tempo, `{} | select(duration)` over a day gives you the distribution — rather than assuming it tracks request latency.

**The rebalancing cost.** The `loadbalancing` exporter hashes trace IDs across the resolved endpoint set, so any change to that set — a rollout, an HPA scale event, a node drain — remaps roughly `R/N` of routes (R routes, N backends). During the transition some traces are split across the old and new target, which reintroduces the fragment problem for the duration. Two mitigations: put `groupbytrace` in front so whole traces dispatch atomically, and treat gateway scaling as a deliberate, rate-limited operation rather than something an HPA does every five minutes.

### 6. How a trace backend actually stores this

The storage model explains both why traces are cheap to keep and why searching them costs what it does. Take Tempo, whose 3.x architecture is public and current.

```
   TEMPO 3.x — WRITE AND READ PATHS
   ═══════════════════════════════════════════════════════════════════════════

   WRITE (microservices mode)
   ──────────────────────────
   OTel Collector / Alloy
        │ OTLP
        ▼
   ┌─────────────┐   validates, shards by TRACE ID
   │ DISTRIBUTOR │──────────────┐
   └─────────────┘              ▼
                        ┌──────────────┐
                        │    KAFKA     │ ◀── the write-ahead log
                        └──────┬───────┘     Once Kafka ACKs, the write is
        ┌──────────────────────┼──────────┐  DURABLE. Tempo therefore runs
        ▼                      ▼          ▼  at REPLICATION FACTOR 1 on the
   ┌───────────┐      ┌──────────────┐  ┌──────────────┐  write path —
   │LIVE-STORE │      │BLOCK-BUILDER │  │  METRICS-    │  no 3× write
   │ recent    │      │ builds       │  │  GENERATOR   │  amplification.
   │ queries   │      │ Parquet      │  │  (optional)  │
   └───────────┘      └──────┬───────┘  └──────┬───────┘
                             ▼                 ▼
                    ┌─────────────────┐   Prometheus
                    │ OBJECT STORAGE  │   (RED metrics,
                    │ vParquet blocks │    service graphs)
                    └─────────────────┘

   READ
   ────
   query-frontend ──shards the query into jobs──▶ queriers
        │                                            │
        ├── recent jobs ──▶ live-stores              │
        └── historical jobs ──▶ object storage ──────┘
                                     │
        A search by attribute is a COLUMNAR SCAN of Parquet column
        chunks, not an index lookup. It reads only the columns the
        query names. Cost ∝ (blocks in range) × (columns touched),
        NOT ∝ (traces in range).

   BLOCK FORMAT
   ────────────
   vParquet4 (default) — columns for array attributes, events, links
   vParquet5 (opt-in)  — up to 20 dedicated string + 5 dedicated integer
                         columns per scope (span/resource/event), vs 10
                         string columns per scope in vParquet4
   (v2 and vParquet3 were REMOVED in Tempo 3.0.)
```

Three design consequences worth being able to explain:

**Object storage plus columnar format is why traces are cheap to *keep*.** There is no inverted index over span attributes to maintain, no in-memory series to hold. A trace is bytes in Parquet in S3. That is why 14-day trace retention costs less than you expect and why the expensive part of tracing is the *gateway*, not the store.

**It is also why search is a scan.** TraceQL queries that name a dedicated column are fast; queries over a rarely-used attribute must decode a generic key-value column across every block in range. That is the tuning knob: **dedicated attribute columns.** Promote the handful of attributes you actually search on — `tenant`, `gen_ai.request.model`, `k8s.namespace.name` — into dedicated columns, and leave the long tail generic.

**Kafka-as-WAL is what buys RF=1.** Compare with Mimir's classic path (lesson 3), which replicates each series to three ingesters and needs a quorum of two. Tempo pushes durability into Kafka and therefore does not replicate on the write path at all. Same architectural move, different component, and it is worth recognising the pattern: *when a durable log sits in front of your stateful tier, replication becomes the log's problem and your tier gets cheaper.*

**Jaeger, for contrast.** Jaeger v2 (v2.20.0, July 2026) is built **on the OpenTelemetry Collector framework** — the Jaeger collector is a Collector distribution with Jaeger-specific components, ingesting OTLP on the same 4317/4318 ports. Storage is pluggable (Cassandra, Elasticsearch/OpenSearch, or a plugin over gRPC). The practical difference from Tempo: Jaeger's Elasticsearch backend *does* index span attributes, which makes arbitrary attribute search fast and makes storage cost scale with indexed-field cardinality — the lesson-1 trade-off appearing again in a third system. Choose Jaeger+ES when attribute search latency matters more than storage cost; choose Tempo when the reverse holds and you are willing to declare your searchable attributes up front.

### 7. Exemplars: the navigation bridge, with its real limits

Failure mode (3) from §1 is fixed by exemplars, and the mechanism has more sharp edges than its reputation suggests.

**How it physically works.** When the SDK records a histogram observation, it stamps the currently-active `trace_id` and `span_id` onto that observation. In OpenMetrics exposition the exemplar rides on the bucket line after a `#`:

```
http_request_duration_seconds_bucket{le="2.5"} 84 # {trace_id="4bf92f3577b34da6a3ce929d0e0e4736",span_id="00f067aa0ba902b7"} 2.31 1755504312.451
                                                    └───────── exemplar labels ─────────┘  └value┘ └── timestamp ──┘
```

Read it: of the 84 observations in this bucket, here is one, it took 2.31 s, and here is its trace. The metric stays bounded — `trace_id` is a payload, not a label.

```
   THE NAVIGATION PATH, END TO END
   ═══════════════════════════════════════════════════════════════════════════

   ①  operator sees p99 latency panel spike at 14:32
              │
              │  panel query: histogram_quantile(0.99,
              │                 sum by (le) (rate(..._bucket[5m])))
              ▼
   ②  Grafana overlays EXEMPLAR DOTS on the graph
              │  fetched via Prometheus /api/v1/query_exemplars
              │  ?query=...&start=...&end=...
              ▼
   ③  operator clicks a dot at 14:32:11 sitting near the p99 line
              │  the dot carries trace_id=4bf92f35…
              ▼
   ④  Grafana's Prometheus data source has an exemplar config mapping
              │  the `trace_id` label → the Tempo data source
              ▼
   ⑤  Tempo opens the trace by ID (a direct lookup, not a search)
              ▼
   ⑥  waterfall shows: 2.1 s of the 2.31 s was one span,
      `nccl.allreduce`, on rank 37 — a straggler

   TOTAL: two clicks from "a graph moved" to "here is the rank".
   WITHOUT exemplars: guess a time range, guess a service, search,
   and hope the trace you need survived sampling.
```

**The limits, stated correctly.** Prometheus's exemplar storage (`tsdb/exemplar.go`, `NewCircularExemplarStorage`) is a **single fixed-size circular buffer shared by every series in the process**, sized by `storage.exemplars.max_exemplars` and enabled with `--enable-feature=exemplar-storage`. Prometheus's docs put an exemplar carrying just a `trace_id` at roughly **100 bytes**. Three consequences:

1. **High-throughput series evict low-throughput series' exemplars.** The ring is global and overwritten oldest-first. Your busiest service always has a dot to click; the quiet service that sees one slow request an hour usually does not, because the ring wrapped. This is *not* "one exemplar per series."
2. **Sizing is a single global number.** 1,000,000 exemplars ≈ 100 MB of RAM. That is the whole knob.
3. **Exemplars are written to the WAL**, so they survive a restart for as long as the WAL does — but they are not in blocks, so they do not survive beyond the head's lifetime.

**The synergy that is easy to miss:** tail sampling *improves* exemplar usefulness. Under uniform head sampling, the trace an exemplar points to has probably been discarded, so the click leads to a 404. Under outcome-based tail sampling, the slow request that produced the p99 exemplar is exactly the kind of trace the policy kept. The two mechanisms are complementary by construction: one selects the interesting traces, the other makes them reachable.

**Gauges cannot carry exemplars** in the classic exposition path — exemplars attach to counter and histogram observations. For a DCGM gauge the equivalent is a low-churn info metric joined at query time (lesson 1 §7, lesson 9).

### 8. Deriving metrics from spans — and the bias question, settled

Span-derived metrics are attractive: RED metrics and service graphs for free, from instrumentation you already have, guaranteed consistent with the traces they summarise.

Tempo's metrics-generator produces:

| Metric | Type | Notes |
|---|---|---|
| `traces_spanmetrics_calls_total` | Counter | request count, by configured dimensions |
| `traces_spanmetrics_latency` | Histogram | span duration |
| `traces_spanmetrics_size_total` | Counter | bytes of spans ingested |
| `traces_service_graph_request_total` | Counter | one series per **hop** (client→server edge) |
| `traces_service_graph_request_failed_total` | Counter | one series per hop |
| `traces_service_graph_request_server_seconds` | Histogram | `#buckets × #hops` series |
| `traces_service_graph_request_client_seconds` | Histogram | `#buckets × #hops` series |
| `traces_service_graph_unpaired_spans_total` | Counter | worst case one per service |
| `traces_service_graph_dropped_spans_total` | Counter | worst case one per service |

**The cardinality formula is published**, which is unusual and useful. With `#hb` histogram buckets and `#hops` distinct client→server edges:

```
   service-graph series ≈ [ (2 × #hb + 2) × #hops ] + [ 2 × #services ]

   and #hops is bounded by  (#services − 1)  ≤  #hops  ≤  #services!

   Worked: 120 services, ~400 observed hops, 12 histogram buckets
      = (2×12 + 2) × 400 + 2 × 120
      = 26 × 400 + 240
      = 10,640 series
   Cheap. But at 400 services with 3,000 hops:
      = 26 × 3,000 + 800 = 78,800 series — and hops grow superlinearly
      with service count. Budget it (lesson 1) before enabling.
```

**The bias question.** If the metrics generator runs *downstream* of tail sampling, it sees the sampled population — 100% of errors, 100% of slow traces, 1% of normal ones — and a naive `calls_total` from that population reports an error rate near 50% rather than 0.3%. Two fixes, in order of preference:

1. **Adjusted counts (§4).** If the spans carry `ot=th:`, the consumer weights each span by `1/p` and recovers an unbiased estimate. This works no matter how many sampling stages there were or where the generator sits. It is the mechanism the OTel spec built for exactly this purpose.
2. **Generate before sampling.** Put the metrics generator (or the `spanmetrics` connector) upstream of the tail-sampling processor. Simple and reliable, but it means the generator processes 100% of spans — which costs CPU proportional to full traffic, and only works if every sampling stage is downstream of it.

In Tempo's architecture the metrics-generator consumes from **Kafka**, i.e. from the full unsampled stream, which sidesteps the problem structurally — worth knowing, because it means "Tempo's RED metrics are biased by tail sampling" is not true for the metrics-generator, only for a `spanmetrics` connector placed after `tail_sampling` inside a Collector pipeline.

### 9. GPU-fleet tie

Tracing on a GPU fleet does two jobs that metrics cannot.

**Inference: latency attribution across the request path.** An inference request's latency decomposes into queue wait, prefill, decode, and any tool or retrieval hops. Only a trace shows which part moved. With OTel's GenAI conventions (lesson 4 §10) the span carries `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`; with the Collector's `k8sattributes` enrichment the same resource carries `k8s.node.name` and `tenant`. So one TraceQL query answers "which node's requests are slow for this model":

```
{ resource.k8s.node.name = "gpu-node-0417"
  && span.gen_ai.request.model = "llama-3.1-70b-instruct"
  && duration > 5s }
```

**Training: the step trace.** A training step is a natural trace — a root span per step, child spans for dataloader wait, forward, backward, optimiser, and each collective. The interesting latency is at the handoffs, and a straggler shows up as one rank's collective span being consistently longer. The tail-sampling policy that makes this work:

```yaml
- name: slow-training-step
  type: and
  and:
    and_sub_policy:
      - name: is-step
        type: string_attribute
        string_attribute: { key: workload.kind, values: [training-step] }
      - name: slower-than-normal
        type: latency
        latency: { threshold_ms: 1300 }    # 1.3 × the 1,000 ms median step
```

**What tracing structurally cannot see.** There is no context propagation into a CUDA kernel. A span around a kernel launch measures the *launch*, not the kernel; a span around a synchronisation point measures the *wait*. GPU-side timing comes from CUPTI, Nsight, or DCGM — a different mechanism on a different clock. The correlation between a host-side span and a device-side kernel is done by timestamp alignment, not by propagation, and it is lesson 9's subject. Do not promise a design review that traces will show you kernel time; promise that they will show you *where the host was blocked*, which is usually the more actionable fact anyway.

**And the sampling policy for a GPU fleet should be tenant-aware**, because the cost of a missed trace is not uniform: a 512-GPU training job stalling is worth far more diagnostic budget than a single inference request being slow. Allocate the sampling budget by the value of the workload, not by request count.

## Perspectives

**Cost accounting.** Tail sampling's cost is a memory–time product — `trace_rate × decision_wait × spans_per_trace × bytes_per_span` — and the keep rate does not appear in it. That is the counter-intuitive part worth internalising: buffering is paid on 100% of traffic regardless of what you throw away, so the cost curve steepens with traffic and trace depth, not with retention policy. It also means the two levers that actually reduce gateway cost are `decision_wait` (linear) and a head sample in front (linear), not a tighter tail policy.

**Protocol and plumbing.** The `loadbalancing` exporter is unglamorous and load-bearing. Its hash over a discovered endpoint set means every scale event on the gateway tier remaps a fraction of routes and briefly splits in-flight traces. A design review of a tail-sampling topology should ask explicitly: what happens to in-flight traces during a gateway autoscale, and is the gateway on an HPA at all? The usual right answer is that it should not be — scale it deliberately, on a schedule, with `groupbytrace` in front.

**Statistical.** Consistent probability sampling with propagated thresholds converts sampling from a lossy compression into a survey design. Once every span carries the probability at which it was kept, the surviving corpus supports unbiased estimation of the population — which means "we sample, therefore our span-derived metrics are approximate" stops being true. This is the strongest available argument for adopting the `tracestate`-based samplers as they stabilise, and it is the kind of detail that separates someone who has read a tracing tutorial from someone who has designed a tracing system.

**Organisational adoption.** Tracing pays off fastest when the *query* experience supports high-cardinality slicing. An org can build the pipeline perfectly — outcome-based policies, correct routing, well-sized `decision_wait` — and still get no value if the query UI only filters on a handful of fields. In Tempo terms that is a concrete decision: which attributes get **dedicated columns**. Get that list wrong and every interesting search is a full scan. The ingestion investment is necessary and not sufficient; the payoff shows up at query time and it is a schema decision.

**Concurrency model.** OTel's context propagation assumes context rides a call stack. Actor systems, message queues, thread pools and event loops break that assumption, and the SDK does not fix it for free. Before adopting any tracing standard wholesale, check whether your concurrency model matches its propagation assumption — and if it does not, budget for a transport layer, as Discord did for Elixir. This generalises beyond tracing: any mechanism that assumes a call stack (thread-locals, stack traces, structured concurrency, `AsyncLocalStorage`) has the same blind spot in the same places.

## Real-world use cases

- **Uber, "Evolving Distributed Tracing at Uber Engineering."** The origin story of Jaeger, including the move from head-based to post-trace (tail-based) sampling buffered in agents. **What it shows:** tail sampling was not invented as an optimisation; it was invented because head sampling at Uber's scale reliably discarded the traces engineers needed. It is the primary historical account of *why* the mechanism this lesson centres on exists, and it predates the OTel ecosystem by years — which tells you the constraint is fundamental rather than an artefact of any particular tool.

- **Grafana Labs, "How Grafana Labs enables horizontally scalable tail sampling in the OpenTelemetry Collector."** The `loadbalancing` exporter, the all-spans-one-instance requirement, and an aggregation design that reduces the N×M connection fan-out. **What it shows:** an organisation that runs this stack commercially hit the split-trace failure in production and needed a dedicated routing tier to fix it — not a tuning change. The generalisable lesson is that a stateful processor cannot be scaled horizontally behind a naive load balancer, which applies well beyond tracing.

- **Honeycomb, "Honeycomb Users Are Living in the Future, Part 1: Sampling."** Honeycomb's Refinery performs dynamic, outcome-aware tail sampling and reconstructs the sampled-out population when computing charts and SLOs. **What it shows:** the adjusted-count idea in §4 is not theoretical — a commercial product is built on it, and the reconstruction is accurate enough to base SLOs on. It is the strongest practical evidence that sampled tracing data can carry unbiased aggregate statistics.

- **Discord, "Distributed tracing in Elixir's actor model"** (as reported by InfoQ, March 2026 — flagged as a third-party report of engineering work, not a primary Discord post). Discord built a custom transport library to carry trace context across Elixir actor message passing, with dynamic sampling for million-user fan-out. **What it shows:** the concrete shape of the concurrency-model gap. There is no call stack between an actor sending a message and the actor receiving it, so context must be an explicit field in the message envelope. Any team on an actor, queue or event-driven architecture should read this before assuming "add the SDK" gives them coverage.

## Worked example

**Design the tracing layer for the fleet: an inference gateway plus a training fleet, 20,000 traces/s, on the topology from lesson 4.**

---

**Step 1 — measure the trace-duration distribution before choosing `decision_wait`.**

```
   TraceQL, run over 24 h of existing data:
     { } | select(duration)

   observed:
     p50   =    38 ms
     p95   =   410 ms
     p99   = 1,850 ms
     p99.9 = 9,200 ms     ◀── async tails: retrieval + tool calls
     max   = 71,000 ms    ◀── one batch job with a 71 s trace

   REQUEST latency p99.9 was 2,100 ms. TRACE duration p99.9 is 9,200 ms.
   Sizing decision_wait from request latency would truncate ~0.1 % of
   traces — and it is the 0.1 % with the interesting async behaviour.

   ⇒ decision_wait = 12s   (p99.9 of 9.2 s + 30 % margin)
   ⇒ accept that the 71 s batch traces will be decided incomplete;
     handle them with a separate pipeline keyed on workload.kind,
     or accept the loss and document it.
```

---

**Step 2 — size the gateway from §5.**

```
   5 gateway replicas, 20,000 traces/s fleet-wide
     per gateway              4,000 traces/s
     in-flight = 4,000 × 12                        = 48,000 traces
     num_traces (2.5× margin for bursts)           = 120,000
     spans/trace p50 = 12, p99 = 140  (use the mean, 18, for memory)
     bytes/span in memory                          ≈ 400 B
     buffer at num_traces  120,000 × 18 × 400 B    ≈ 864 MB
     decision caches (1 M + 2 M ids × ~40 B)       ≈ 120 MB
     receiver + batch + Go slack (×1.6 on total)   ≈ 1.6 GB
   ⇒ container 6 GiB, memory_limiter limit_mib 4608, spike_limit_mib 922
   ⇒ num_shards: 8   (parallel decision loops, trace-ID hashed)
```

---

**Step 3 — the policy set, with the budget allocated by workload value.**

```yaml
processors:
  tail_sampling:
    decision_wait: 12s
    num_traces: 120000
    expected_new_traces_per_sec: 4000
    num_shards: 8
    decision_cache:
      sampled_cache_size: 1000000
      non_sampled_cache_size: 2000000
    maximum_trace_size_bytes: 10485760   # 10 MiB; drops pathological traces
    policies:
      # ── 1. Everything that failed. Non-negotiable.
      - name: errors
        type: status_code
        status_code: { status_codes: [ERROR] }

      # ── 2. Everything slow, measured against the SLO threshold,
      #      not against an arbitrary round number.
      - name: slo-violating-inference
        type: and
        and:
          and_sub_policy:
            - name: is-inference
              type: string_attribute
              string_attribute:
                key: gen_ai.operation.name
                values: [chat, generate_content]
            - name: over-slo
              type: latency
              latency: { threshold_ms: 5000 }      # the SLO threshold

      # ── 3. Training steps slower than 1.3× the median — the straggler
      #      signal that lesson 9 builds a goodput SLO on.
      - name: slow-training-step
        type: and
        and:
          and_sub_policy:
            - name: is-step
              type: string_attribute
              string_attribute: { key: workload.kind, values: [training-step] }
            - name: slower-than-normal
              type: latency
              latency: { threshold_ms: 1300 }

      # ── 4. A control distribution for the expensive tenants. Without a
      #      baseline you cannot tell "slow" from "always been like that".
      - name: gpu-tenant-baseline
        type: and
        and:
          and_sub_policy:
            - name: is-gpu-tenant
              type: string_attribute
              string_attribute:
                key: tenant
                values: ["team-vision", "team-nlp"]
            - name: sample-5pct
              type: probabilistic
              probabilistic: { sampling_percentage: 5 }

      # ── 5. Fleet-wide control sample.
      - name: baseline
        type: probabilistic
        probabilistic: { sampling_percentage: 0.5 }
```

**Compute the resulting volume before deploying it** — this is the number the design review will argue about:

```
   SAMPLED VOLUME — 20,000 traces/s, 18 spans/trace mean
   ═══════════════════════════════════════════════════════════════════════
   errors                 0.30 % → 60 traces/s   × 100 %  =    60 /s
   SLO-violating inference 0.80 % → 160 traces/s × 100 %  =   160 /s
   slow training steps    0.15 % → 30 traces/s   × 100 %  =    30 /s
   GPU tenants (35 % of the rest ≈ 6,900/s)      ×   5 %  =   345 /s
   fleet baseline (remaining ≈ 12,850/s)         × 0.5 %  =    64 /s
                                                            ──────────
   kept                                                     ≈ 659 /s
                                                            (3.3 % of traces)

   spans/s kept        659 × 18                            ≈ 11,862 /s
   bytes/s at ~350 B/span compressed in Parquet            ≈  4.2 MB/s
   per day                                                 ≈  359 GB/day
   14-day retention                                        ≈  5.0 TB
   object storage at ~$0.021/GB-month                      ≈  $105/month

   ── READ THE SHAPE ──────────────────────────────────────────────────────
   The three outcome policies together are only 250 traces/s — 38 % of
   what is kept, and they are the ones you will actually open. The
   GPU-tenant baseline at 345/s is more than half the bill.
   Storage is $105/month; the GATEWAY (5 × 6 GiB, always on) is the
   expensive part. Sampling policy is a budget allocation; spend it
   where a missing trace costs the most.
```

---

**Step 4 — the exemplar wiring.**

```yaml
# Prometheus: enable the exemplar store and size it.
# --enable-feature=exemplar-storage
storage:
  exemplars:
    max_exemplars: 1000000        # ≈100 MB RAM; ONE GLOBAL RING, not per-series
```

```yaml
# Grafana Prometheus data source: map the exemplar label to the trace store.
jsonData:
  exemplarTraceIdDestinations:
    - name: trace_id
      datasourceUid: tempo-prod
```

And the panel query that produces exemplar-bearing points:

```promql
histogram_quantile(0.99,
  sum by (le, gen_ai_request_model) (
    rate(gen_ai_server_request_duration_seconds_bucket[5m])
  )
)
```

**Verify the wiring rather than assuming it.** Three checks:

```promql
# 1. Are exemplars being stored at all?
prometheus_tsdb_exemplar_exemplars_in_storage

# 2. Is the ring wrapping? If in_storage sits pinned at max_exemplars
#    and series_with_exemplars is much lower than your series count,
#    busy series are evicting quiet ones.
prometheus_tsdb_exemplar_series_with_exemplars_in_storage

# 3. Direct check — does this series have an exemplar right now?
#    /api/v1/query_exemplars?query=gen_ai_server_request_duration_seconds_bucket
#      &start=<now-5m>&end=<now>
```

---

**Step 5 — the propagation gap analysis.**

Walk every boundary and record the verdict. This is the deliverable that finds the broken traces before an incident does:

| # | Boundary | Mechanism | Verdict | Action |
|---|---|---|---|---|
| 1 | browser → ingress | `traceparent` on fetch | ⚠ ingress restarts traces | reconfigure ingress not to regenerate; else document that traces start at ingress |
| 2 | ingress → api-gw | Envoy | ✅ | verify W3C propagator, not B3-only |
| 3 | api-gw → router | gRPC, OTel SDK | ✅ | — |
| 4 | router → vllm | HTTP, OTel SDK | ✅ | — |
| 5 | router → Kafka → audit-logger | Kafka headers | ❌ **broken** | enable the SDK's Kafka instrumentation, or inject/extract manually |
| 6 | vllm → executor thread pool | Python `ThreadPoolExecutor` | ❌ **broken** | wrap submissions with `contextvars` copy, or use the instrumented pool |
| 7 | scheduler → training pod | pod annotation | ❌ **no mechanism** | write `traceparent` into a pod annotation; the trainer reads it as a remote parent and adds a `Link` |
| 8 | trainer → NCCL collective | — | ❌ **n/a** | host-side span around the collective; device timing via a separate mechanism (lesson 9) |
| 9 | trainer → CUDA kernel | — | ❌ **n/a** | not a propagation problem; timestamp-correlate CUPTI data |

Rows 5, 6 and 7 are fixable and are where most of the value is. Rows 8 and 9 are structural, and saying so plainly in a design review is worth more than promising coverage you cannot deliver.

## Practice

Feeds the [fleet observability design](../practice/fleet-observability/README.md).

Design the tracing layer of the fleet observability system and deliver three artifacts.

**(a) The tail-sampling policy** — the full YAML, with:
1. `decision_wait` justified against a *measured* p99.9 **trace duration** (state how you measured it, and state explicitly how it differs from p99.9 request latency in your system).
2. `num_traces` derived from trace rate × gateway count × `decision_wait`, with the margin stated.
3. Explicit error / SLO-violation / straggler / baseline policies, with each threshold traceable to an SLO or a measured median rather than a round number.
4. The **sampled-volume calculation** — per-policy keep rate, resulting traces/s, spans/s, bytes/day, retention cost — and a sentence naming which policy dominates the bill.
5. The gateway memory budget, carried through to a `memory_limiter` configuration.

**(b) The exemplar-linked latency panel** — the PromQL, the Prometheus exemplar configuration with `max_exemplars` sized and justified, the Grafana data-source mapping, and the three verification queries that prove the link works. Include a paragraph on the **global circular buffer** and which of your services will lose exemplars to eviction.

**(c) The trace-context propagation gap analysis** — every boundary in one representative request path *and* one training-job path, each classified as propagating / broken / structurally impossible, with the fix or the honest "cannot" for each. Include at least one queue boundary, one thread-pool or async boundary, and the scheduler→pod boundary.

Additionally, write short answers to:

6. **The routing question.** Why `loadbalancing` keyed on `traceID` is mandatory, what breaks without it (be specific about what the corrupted output looks like), and what happens to in-flight traces during a gateway scale event. State whether your gateway is on an HPA and defend the answer.
7. **The bias question.** Are your span-derived RED metrics computed before or after sampling? If after, do your spans carry `ot=th:` thresholds, and can your backend apply adjusted counts? If neither, state the bias factor you are living with.
8. **The head-sampling question.** At what traffic level would you add a head sample in front of the tail-sampling tier, and what would you lose?

**Acceptance criteria.** Done when (i) every numeric parameter traces to a measurement or a derivation, (ii) the sampled-volume number is computed and the dominant policy named, (iii) the gap analysis contains at least two boundaries you found to be broken, and (iv) a peer could re-run the whole design for a different traffic profile by changing the inputs.

## Common pitfalls

- **"Sampling is about cost control."** *Symptom:* uniform 1% sampling and a trace store nobody opens. *Mechanism:* it is equally about signal quality. Uniform undersampling destroys the ability to find the incident-relevant trace, because the one you need is statistically the one you discarded. Sampling is a *selection* problem that happens to also save money.

- **"Tail sampling reduces the cost of tracing."** *Symptom:* a design that budgets for 3% of the span volume everywhere. *Mechanism:* tail sampling reduces *storage and query* cost only. 100% of spans are still created, serialised, transported to the gateway, and held in gateway RAM for `decision_wait`. If your problem is application CPU or network egress, tail sampling does not help — head sampling does.

- **"`decision_wait` should be about p99 latency."** *Symptom:* `sampling_trace_dropped_too_early` climbing, and slow traces mysteriously absent. *Mechanism:* a trace ends when its *last* span ends. Requests with async tails have trace durations far exceeding request latency. Size against measured p99.9 trace duration, and verify with `otelcol_processor_tail_sampling_sampling_late_span_age`.

- **"Exemplars store one per series."** *Symptom:* a quiet service whose panels never have a clickable dot. *Mechanism:* `tsdb/exemplar.go` implements a **single global** circular buffer sized by `max_exemplars`, overwritten oldest-first across all series. High-throughput series evict low-throughput ones. Size the ring globally and expect eviction to be uneven.

- **"Exemplars need 100% sampling to be useful."** *Symptom:* teams postpone exemplars until they can "afford full tracing." *Mechanism:* an exemplar needs one representative trace ID per bucket per scrape, and outcome-based tail sampling makes the trace it points at *more* likely to be the interesting one. The two mechanisms compose; neither requires the other to be maximal.

- **"Context propagation is solved by adding the SDK."** *Symptom:* traces that end at a queue. *Mechanism:* the SDK's propagator handles the call-stack case. Message queues, thread pools, actor mailboxes and cron triggers have no call stack for context to ride on, so propagation must be explicit — a header on the message, a `contextvars` copy on submission, a `Link` from a new root span.

- **"Span-derived RED metrics are always safe."** *Symptom:* a service-graph dashboard showing a 40% error rate on a healthy service. *Mechanism:* a metrics generator downstream of tail sampling sees a population enriched for errors and slowness. Fix with adjusted counts (`ot=th:`) or by generating upstream of all sampling. Note that Tempo's metrics-generator consumes from Kafka — the unsampled stream — so it does not have this problem; a Collector `spanmetrics` connector placed after `tail_sampling` does.

- **"Service graphs are free."** *Symptom:* Mimir series count jumping after enabling them. *Mechanism:* the published cardinality formula is `[(2 × #buckets + 2) × #hops] + [2 × #services]`, and `#hops` grows superlinearly with service count — bounded below by `#services − 1` and above by `#services!`. At 400 services and 3,000 hops with 12 buckets that is ~79,000 series. Budget it against lesson 1's ledger before enabling.

- **"Every attribute is searchable."** *Symptom:* TraceQL queries that time out on a 7-day range. *Mechanism:* Tempo stores traces as Parquet in object storage with no inverted index. A search over an attribute without a dedicated column decodes a generic key-value column across every block in range. Declare the attributes you search on as dedicated columns; vParquet5 raised the budget to 20 string and 5 integer columns per scope.

## Self-check

**Why can't head sampling keep "all errors plus all slow traces"?**
Because head sampling decides at the root span, before the request executes, so the outcome is unknown at decision time — an errored trace and a normal one are indistinguishable when the coin is flipped. Concretely, with 3 errors in 1,000 requests and 1% head sampling, the expected number of errors kept is 0.03, so you keep zero errors about 97% of the time. Only tail sampling, which buffers the trace and decides on completion, can select on outcome. The trade is that tail sampling buffers 100% of spans and therefore saves only storage and query cost, not production or transport cost.

**What topology requirement does tail sampling impose, and what does the failure look like?**
Every span of a trace must reach the same processing instance, because the policy is a predicate over the whole trace. That forces a routing tier — the `loadbalancing` exporter with `routing_key: traceID` against a headless Service — in front of the sampling collectors. Without it, a plain Service balances connections, spans of one trace scatter across replicas, and each replica decides on its fragment. The output is not an error: it is stored "traces" that are silently missing the spans that lived on other replicas, including, typically, the erroring span that was the reason to keep the trace at all. Also budget for the fact that any endpoint-set change remaps roughly `R/N` of routes and briefly splits in-flight traces.

**Why must `decision_wait` exceed p99.9 trace duration rather than p99.9 request latency?**
A trace is complete when its *last* span ends, which for any request that spawns asynchronous work is well after the user-visible response. A request returning in 200 ms that kicks off a 45-second background job has a 45-second trace. Sizing `decision_wait` from request latency truncates exactly those traces: at decision time they look short and uninteresting, so a baseline probabilistic policy drops them. You have then systematically discarded the traces with the most interesting behaviour. Measure trace duration directly (`{} | select(duration)` in TraceQL) and verify with `otelcol_processor_tail_sampling_sampling_late_span_age`.

**Explain consistent probability sampling, with the threshold arithmetic.**
Two 56-bit values. `R`, the randomness, is either an explicit `rv:` in the OTel `tracestate` entry or the least-significant 56 bits of the trace ID (valid when W3C Trace Context Level 2's `random` flag, bit `0x02`, is set). `T`, the rejection threshold, is `(1 − p) × 2^56`, carried as `th:` with trailing zeros stripped — so `p = 0.01` gives `T ≈ 0xfd70a400000000`, encoded `ot=th:fd70a4`. The rule is `R >= T ⇒ keep`. Because every participant compares the same `R` against its own `T`, a span kept at probability `p1` is necessarily kept by anyone sampling at `p2 >= p1` — so you can never get a child without its parent, even when participants are configured independently. Valid probabilities run from `2^-56` to 1; zero is not a probability.

**What is an adjusted count and what problem does it solve?**
The reciprocal of the sampling probability — the number of items in the unsampled population that this surviving item represents. A span kept at 1% has an adjusted count of 100. Because the effective threshold travels with the span in `tracestate`, any downstream consumer can read the probability off the span and weight by `1/p`, recovering unbiased estimates of true volume, error rate and latency distribution from a sampled corpus. This is the correct fix for span-metrics bias: it works regardless of how many sampling stages ran or where the metric generator sits, unlike the "generate metrics before sampling" workaround, which only holds if every sampling stage is downstream of every generator.

**How does an exemplar physically connect a metric to a trace, and what is its capacity behaviour?**
On a histogram observation the SDK stamps the currently-active `trace_id` and `span_id`; in OpenMetrics exposition this rides on the bucket line after a `#`, with the observation's value and timestamp. Prometheus stores exemplars only with `--enable-feature=exemplar-storage`, in a **single global circular buffer** sized by `storage.exemplars.max_exemplars` (`tsdb/exemplar.go`), at roughly 100 bytes each — so a million exemplars is about 100 MB. Because the ring is global and overwritten oldest-first, high-throughput series evict low-throughput ones, which is why busy services always have a clickable dot and quiet ones often do not. Grafana fetches them via `/api/v1/query_exemplars` and maps the `trace_id` label to a trace data source.

**Why does Discord's actor-model tracing case generalise beyond Elixir?**
Because OTel's propagation model assumes context rides a call stack from caller to callee. In an actor system a message is placed in a mailbox and processed later by a different scheduler entity — there is no stack connecting the two — so context must be carried as an explicit field in the message envelope, which requires a transport layer nobody's SDK provides. The same gap exists for Kafka and SQS, for thread pools and executors, for event loops, and for cron-triggered work. The general test before adopting any tracing standard is whether your concurrency model matches its propagation assumption, and the general remedy is an explicit envelope field plus a `Link` when a true parent-child relationship does not exist.

**Why is Tempo able to run at replication factor 1, and what does that tell you about the pattern?**
Because in microservices mode the distributor writes to a Kafka-compatible queue that acts as the write-ahead log; once Kafka acknowledges, the data is durable, so Tempo does not replicate across instances on the write path. Live-stores and block-builders consume asynchronously — the former serving recent queries, the latter building Parquet blocks for object storage. The generalisable pattern is that a durable log in front of a stateful tier moves replication into the log and makes the tier cheaper: Mimir's newer ingest-storage architecture does exactly the same thing, replacing RF=3 hash-ring replication with Kafka partitions and a read quorum of one.

## Connections & what's next

This lesson builds directly on [04 · OpenTelemetry](04-opentelemetry.md) — the two-tier gateway exists *because* tail sampling needs it, and the `loadbalancing` routing key introduced there is what makes the policies here correct. It also closes a loop with [01 · The signal model](01-signal-model.md): exemplars are the concrete mechanism that makes "demote the question to the cheapest signal" safe, and this lesson gives their real storage limits. The service-graph cardinality formula in §8 is a direct charge against the lesson-1 series budget and the lesson-3 per-tenant limits.

Lesson 6 carries the same economics lens to logs, where the decision becomes "which fields do I index" and the cost driver shifts from buffering to cardinality. Lesson 7 consumes the exact-threshold latency SLI that these traces explain, and the straggler policy in §9 is the trace-side half of the goodput-regression alert built in lesson 9 — where exemplar-linked traces turn a fleet-wide goodput regression into a single stalled rank.

Next: [06 · Logging pipelines](06-logging-pipelines.md).

## References & further reading

**Primary sources — read directly from the repositories**
- OpenTelemetry specification (`open-telemetry/opentelemetry-specification`, `main`, August 2026), `specification/trace/tracestate-probability-sampling.md` — the `R`/`T` model, `T = (1 − p) × 2^56`, the `R >= T ⇒ keep` rule, the `0xfd70a400000000` / `ot=th:fd70a4` worked encoding for p = 1%, the `2^-56`-to-1 valid probability range, the parent/child vs downstream sampling stages, and the **adjusted count** definition and its stated purpose for span-to-metrics. Marked **Development** upstream.
- OpenTelemetry specification, `specification/trace/tracestate-handling.md` — the `ot=` entry format, the semicolon-separated sub-key list, the 256-character limit, and the `th`/`rv` sub-keys.
- OpenTelemetry specification, `specification/context/api-propagators.md` — the requirement to parse and propagate W3C Trace Context Level 2 `traceparent` and `tracestate`.
- W3C Trace Context specification repository (`w3c/trace-context`, `main`), `spec/20-http_request_header_format.md` — the `version-format` ABNF (32-hex trace-id, 16-hex parent-id, 2-hex trace-flags), the all-zero-invalid rules, `FLAG_SAMPLED = 0x01` and `FLAG_RANDOM = 0x02` with the `01`/`02`/`03` examples, the right-most-7-bytes randomness requirement, the 32-list-member and ~512-character `tracestate` limits with right-most eviction, and the "restart trace" mutation at a secure-network front gate. **Read from the repository because `www.w3.org` is blocked by this environment's egress proxy.**
- OpenTelemetry Collector Contrib (`main`, August 2026), `processor/tailsamplingprocessor/README.md` — the policy catalogue (`latency`, `status_code`, `string_attribute`, `probabilistic`, `rate_limiting`, `span_count`, `ottl_condition`, `and`, `not`, `drop`, `composite`), `decision_wait: 30s`, `num_traces: 50000`, `num_shards`, the `trace-complete`/`span-ingest` strategies, decision caches, `maximum_trace_size_bytes`, and the `processor.tailsamplingprocessor.usetracestate` feature gate for tracestate-aware probabilistic sampling.
- OpenTelemetry Collector Contrib, `processor/tailsamplingprocessor/documentation.md` — the `otelcol_processor_tail_sampling_*` metric names used in §5.
- OpenTelemetry Collector Contrib, `exporter/loadbalancingexporter/README.md` — the routing keys, the `R/N` rebalancing property, and the `groupbytrace` recommendation.
- Grafana Tempo (`main`, August 2026), `docs/sources/tempo/introduction/architecture.md` — the Kafka-as-WAL write path, RF=1 durability, live-store / block-builder / metrics-generator consumers, the query-frontend sharding read path, and the columnar-Parquet design goal.
- Grafana Tempo, `docs/sources/tempo/configuration/parquet.md` — vParquet4 as the default, vParquet5's expanded dedicated columns (20 string + 5 integer per scope vs 10 string), and the removal of `v2`/`vParquet3` in Tempo 3.0.
- Grafana Tempo, `docs/sources/tempo/metrics-from-traces/span-metrics/span-metrics-metrics-generator.md` — `traces_spanmetrics_latency`, `traces_spanmetrics_calls_total`, `traces_spanmetrics_size_total`.
- Grafana Tempo, `docs/sources/tempo/metrics-from-traces/metrics-generator/estimate-cardinality.md` — the service-graph metric list and the published cardinality formula `[(2 × #hb + 2) × #hops] + [2 × #services]`, with `#services − 1 ≤ #hops ≤ #services!`.
- Prometheus 3.14.0, `tsdb/exemplar.go` — `NewCircularExemplarStorage` confirming a **single global** ring rather than per-series buffers. **This corrects the previous version of this lesson**, which stated "up to one exemplar per unique label-set per series."
- [Prometheus feature flags](https://prometheus.io/docs/prometheus/latest/feature_flags/) — `exemplar-storage`, the ~100 bytes/exemplar figure, and WAL persistence of exemplars.
- Jaeger (`jaegertracing/jaeger`, v2.20.0, July 2026) — v2 built on the OpenTelemetry Collector framework, ingesting OTLP on 4317/4318, with pluggable storage.
- [OpenTelemetry metrics data model — exemplars](https://opentelemetry.io/docs/specs/otel/metrics/data-model/#exemplars) and [OpenMetrics 1.0 exemplars](https://github.com/OpenMetrics/OpenMetrics/blob/v1.0.0/specification/OpenMetrics.md#exemplars) — the exposition format in §7.
- [Grafana Tempo TraceQL documentation](https://grafana.com/docs/tempo/latest/traceql/) — query syntax used in the worked example.

**Real-world engineering write-ups**
- Uber, [Evolving Distributed Tracing at Uber Engineering](https://www.uber.com/en-IN/blog/distributed-tracing/)
- Grafana Labs, [How Grafana Labs enables horizontally scalable tail sampling in the OpenTelemetry Collector](https://grafana.com/blog/how-grafana-labs-enables-horizontally-scalable-tail-sampling-in-the-opentelemetry-collector/)
- Honeycomb, [Honeycomb Users Are Living in the Future, Part 1: Sampling](https://www.honeycomb.io/blog/honeycomb-users-living-in-future-pt-1-sampling)
- InfoQ, [Discord Engineers Add Distributed Tracing to Elixir's Actor Model without Performance Penalty](https://www.infoq.com/news/2026/03/discord-elixir-actor-tracing/) — **third-party report of a conference talk, not a primary Discord engineering post**; treat the specifics as reported.
- OpenTelemetry blog, [Tail Sampling in OpenTelemetry](https://opentelemetry.io/blog/2022/tail-sampling/)

**Sources consulted but not relied upon.** `www.w3.org` is blocked by this environment's egress proxy, so the W3C Trace Context facts above were verified against the `w3c/trace-context` Git repository instead — noted inline. Several vendor documentation domains were likewise unreachable; where no repository equivalent existed the claim was omitted. Consistent probability sampling is marked **Development** in the OpenTelemetry specification and SDK/Collector support is uneven — verify what your SDKs emit before building accounting on adjusted counts.
