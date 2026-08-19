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
sources: 13
---

# A03.4 · OpenTelemetry

> **Concept.** Keep the SDK thin and make the Collector the integration point — a two-tier agent→gateway pipeline where processor order and tail-sampling placement are load-bearing.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 03 sized and sharded the storage tier — Mimir vs Thanos, ingester replication, per-tenant cardinality limits. That storage tier has to receive telemetry from somewhere, and *how* it gets there across hundreds of heterogeneous services is its own architecture problem. This lesson is that collection-side integration point, and it is also where lesson 1's third cardinality-enforcement gate lives: the OTLP path has no scrape to relabel, so attribute limits must be enforced in the pipeline itself.

It also sets up lesson 05, which goes deep on the sampling *policy* question this lesson only introduces. Here you learn where sampling can physically happen and why; there you learn what to sample and what it costs.

Everything below is checked against the **OpenTelemetry Collector** (`open-telemetry/opentelemetry-collector` and `-contrib`, `main`, August 2026), the **OpenTelemetry specification** repo, and the **GenAI semantic conventions** repo (`open-telemetry/semantic-conventions-genai`, August 2026). Config keys, defaults and metric names come from the components' own `README.md`, `documentation.md` and `metadata.yaml` files in those trees. Where a component is marked alpha or experimental upstream, that is stated.

## Why this matters

Seniors know OTel exists and can add an SDK. Staff decide *where the seams go* so the org can change backends, sampling, redaction, enrichment and tenant routing for hundreds of services **without redeploying a single application**. That is the whole value proposition, and it is easy to lose: every operational decision baked into the SDK becomes a decision that requires a coordinated multi-team release to change.

The failure modes are specific and expensive. A `memory_limiter` in the wrong slot turns a downstream backend hiccup into an org-wide telemetry blackout. A `tail_sampling` processor on the wrong tier silently corrupts every sampling decision with no error anywhere — you get traces that look fine and are systematically incomplete. A gateway fleet behind a plain load balancer does the same thing more subtly. And an unbounded attribute in an OTLP metric pipeline is a cardinality bomb with no `metric_relabel_configs` to catch it, because there was no scrape.

On a GPU training and inference fleet, this is also the layer where token and model telemetry arrives. OTel's GenAI conventions give you `gen_ai.*` attributes on inference spans; the Collector joins them to `k8s.*` and node attributes without touching training or serving code. That join — model telemetry to infrastructure telemetry, on shared attribute names — is what makes lesson 9's goodput work possible.

## What's new here (calibration)

- **Skip (you already know):** traces/metrics/logs SDKs exist; OTLP is the wire protocol; auto-instrumentation exists; the Collector has receivers, processors and exporters.
- **New:** the OTLP data model as a *nesting* — Resource → Scope → Signal — and why that nesting is what makes the Collector's enrichment and routing cheap. Attributes at the resource level are stored once per batch, not once per span.
- **New:** `memory_limiter` mechanically — soft and hard limits, the `spike_limit_mib` derivation, the non-permanent error it returns, the backpressure contract with receivers, and the specific condition under which it **loses data permanently**.
- **New:** the real tail-sampling configuration surface — `decision_wait`, `num_traces`, `num_shards`, the two `sampling_strategy` modes, the decision caches — and the memory arithmetic that turns those into a gateway sizing number.
- **New:** the `loadbalancing` exporter's actual routing keys and their defaults, including the one people get wrong: the default for **logs is `service`, not `traceID`**.
- **New:** the Collector's own internal metric names as they exist today, with the correction that `otelcol_processor_dropped_spans` is not among them.
- **New:** OTel's **schema URL and schema translation** mechanism — the actual answer to semantic-convention churn, rather than "pin a version and hope."
- **New:** the current GenAI convention surface, including that it now lives in its own repository and that the token *metric* is `gen_ai.client.token.usage` with a `gen_ai.token.type` dimension, not a pair of counters.

## Core concepts

### 1. The OTLP data model — and why the nesting is the point

Before topology, the wire format, because the nesting explains most of the Collector's design.

OTLP is protobuf over gRPC (default port **4317**) or protobuf/JSON over HTTP (default port **4318**, with paths `/v1/traces`, `/v1/metrics`, `/v1/logs`). A single export request is not a flat list of spans. It is a three-level tree:

```
   THE OTLP TRACE PAYLOAD — WHY ENRICHMENT IS CHEAP
   ═══════════════════════════════════════════════════════════════════════════

   ExportTraceServiceRequest
   └── resource_spans[]                       ← ONE per emitting entity
       ├── resource
       │     attributes:                      ← WRITTEN ONCE for every span below
       │       service.name        = "vllm-router"
       │       service.version     = "2.4.1"
       │       k8s.pod.name        = "vllm-router-7d9f-x2k4"
       │       k8s.namespace.name  = "team-vision"
       │       k8s.node.name       = "gpu-node-0417"
       │       host.name           = "gpu-node-0417"
       ├── schema_url = "https://opentelemetry.io/schemas/1.44.0"   ◀── §8
       └── scope_spans[]                      ← ONE per instrumentation library
           ├── scope
           │     name    = "opentelemetry.instrumentation.fastapi"
           │     version = "0.48b0"
           └── spans[]                        ← the actual spans
               ├── trace_id (16 B), span_id (8 B), parent_span_id (8 B)
               ├── name, kind, start/end unix_nano
               ├── attributes:                ← PER SPAN
               │     http.request.method = "POST"
               │     http.route          = "/v1/chat/completions"
               │     gen_ai.request.model = "llama-3.1-70b-instruct"
               ├── events[]  (timestamped, with attributes)
               ├── links[]   (to other trace/span ids)
               └── status    (UNSET | OK | ERROR)

   ── CONSEQUENCE 1 ─────────────────────────────────────────────────────────
   A processor that adds k8s.* attributes writes them ONCE per resource,
   not once per span. Enriching 5,000 spans from one pod costs 6 string
   writes, not 30,000. This is why k8sattributes is affordable at all.

   ── CONSEQUENCE 2 ─────────────────────────────────────────────────────────
   Routing by "service" is a resource-level lookup — one hash per batch.
   Routing by traceID is a SPAN-level lookup — the batch must be split.
   That asymmetry is why the two routing keys have different costs (§6).

   ── CONSEQUENCE 3 ─────────────────────────────────────────────────────────
   A single OTLP request can carry many resources, so one Collector
   connection multiplexes many services. There is no per-service
   connection cost, which is what makes an agent-per-node viable.
```

**Why protobuf-over-gRPC is the enabling decision.** The Collector can sit between every producer and every backend only because there is *one* wire format to translate to and from. One generic `otlp` receiver, one generic `otlp` exporter, and a fixed set of adapters at the edges (`prometheus` receiver for scrape targets, `jaeger` receiver for legacy agents, `prometheusremotewrite` exporter for Mimir). Without a single format you are back to N×M bespoke glue. The architecture is downstream of the protocol choice.

### 2. SDK vs Collector — the load-bearing distinction

**Keep the SDK thin.** Context propagation, span creation, and metric instrument definitions. Nothing else.

**Push everything operational into the Collector.** Backend choice, sampling policy, redaction, enrichment, tenant routing, retry, batching, attribute limits.

The reason is a deployment-cost asymmetry, not aesthetics:

| Decision baked into… | Cost to change |
|---|---|
| SDK config in each app | a PR per service × N services, a release per service, coordinated across teams, weeks to months |
| Collector config | one ConfigMap change, a rolling restart of a DaemonSet/Deployment, minutes |

Concretely, things people wrongly leave in the SDK: the exporter endpoint (should be a fixed local `http://localhost:4318`, with the agent deciding where it actually goes), the sampling ratio (should be head-sample at 100% and let the gateway tail-sample), the resource attributes beyond `service.name`/`service.version` (the Collector knows the infrastructure better than the app does), and PII redaction (a policy that changes on a legal timescale, not a release timescale).

**The one thing that must live in the SDK** is context propagation, because it is the only part that requires being *inside* the process. W3C Trace Context (`traceparent`, `tracestate` headers) is what stitches spans across service boundaries, and no Collector can retrofit it. Lesson 5 goes deep on the propagation surface; the design consequence here is: audit propagation coverage as an SDK concern, and treat everything else as a Collector concern.

**"Auto-instrumentation keeps the SDK thin" is false.** Auto-instrumentation agents ship opinionated defaults — a sampler, an exporter endpoint, a set of enabled instrumentations — and those defaults *are* SDK-baked decisions. Thin-SDK is a design discipline enforced by reviewing what the agent's environment variables set, not a property you inherit by choosing auto-instrumentation.

### 3. The two-tier topology

```
   TWO-TIER COLLECTOR TOPOLOGY — WHERE EACH DECISION PHYSICALLY HAPPENS
   ═══════════════════════════════════════════════════════════════════════════

  ┌─ NODE gpu-node-0417 ──────────────────────────────────────────────────┐
  │                                                                        │
  │  app pod        app pod        vllm pod       dcgm-exporter            │
  │  (thin SDK)     (thin SDK)     (thin SDK)     (/metrics)               │
  │      │              │              │                │                  │
  │      └──OTLP────────┴──────────────┘                │                  │
  │         localhost:4318                       scraped by ──┐            │
  │              │                                            │            │
  │              ▼                                            ▼            │
  │   ┌──────────────────────────── AGENT (DaemonSet) ──────────────────┐  │
  │   │  receivers:  otlp, prometheus, filelog, hostmetrics             │  │
  │   │  processors: memory_limiter ─▶ k8sattributes ─▶ resourcedetect  │  │
  │   │              ─▶ filter/transform ─▶ batch                       │  │
  │   │  exporters:  otlp/gateway  (or loadbalancing → gateway)         │  │
  │   │                                                                  │  │
  │   │  WHY HERE: node-local metadata (node name, GPU topology, pod     │  │
  │   │  UID from the local kubelet) is only available here. Cheap       │  │
  │   │  drops happen here so the gateway never pays for them.           │  │
  │   │  CANNOT do tail sampling: sees only this node's fragment.        │  │
  │   └─────────────────────────────┬────────────────────────────────────┘  │
  └─────────────────────────────────┼───────────────────────────────────────┘
                                    │ OTLP/gRPC
        ┌───────────────────────────┼───────────────────────────┐
        │  ~1,250 other node agents │                           │
        └───────────────────────────┼───────────────────────────┘
                                    ▼
        ┌────────────── ROUTING TIER (loadbalancing exporter) ─────────────┐
        │  routing_key: traceID    resolver: k8s (headless Service)        │
        │  hash(trace_id) → pick ONE gateway replica                        │
        │  ⇒ EVERY span of a trace lands on the SAME gateway               │
        └───────────┬──────────────────┬──────────────────┬────────────────┘
                    ▼                  ▼                  ▼
        ┌────── GATEWAY 1 ──────┐ ┌── GATEWAY 2 ──┐ ┌── GATEWAY 3 ──┐
        │ receivers:  otlp       │ │     …         │ │     …         │
        │ processors:            │ │               │ │               │
        │   memory_limiter       │ │               │ │               │
        │   ─▶ transform (schema │ │               │ │               │
        │       normalisation)   │ │               │ │               │
        │   ─▶ tail_sampling  ◀──┼─┼── ONLY HERE   │ │               │
        │   ─▶ batch             │ │               │ │               │
        │ exporters:             │ │               │ │               │
        │   otlp/tempo           │ │               │ │               │
        │   prometheusremotewrite│ │               │ │               │
        │     /mimir             │ │               │ │               │
        │   otlphttp/loki        │ │               │ │               │
        └────────────┬───────────┘ └───────┬───────┘ └───────┬───────┘
                     └─────────────────────┴─────────────────┘
                                    │
                     ┌──────────────┼───────────────┐
                     ▼              ▼               ▼
                  Tempo          Mimir            Loki
```

**Agent tier responsibilities** — everything that needs to be *local* or is cheap to do early:

- **Receive** OTLP from every pod on the node at a fixed localhost endpoint, so apps never learn a routable address.
- **Enrich** with node and pod metadata, which only exists locally.
- **Drop cheaply** — debug spans, health-check endpoints, `go_gc_*` metrics. Every byte dropped here is a byte the gateway, the network and the backend never pay for.
- **Batch** so the gateway sees few large requests instead of many small ones.

**Gateway tier responsibilities** — everything that needs a *global* or *cross-node* view:

- **Tail-sample**, which requires the whole trace (§5).
- **Normalise** semantic conventions across SDK versions (§8).
- **Route by tenant** and fan out to backends.
- **Own the backend credentials**, so no node agent holds a write token for Mimir.

**The new failure domain.** The gateway is a single collection point whose outage or backpressure affects every team's telemetry simultaneously. That is a real cost of this design and the reason `memory_limiter` placement (§4) is not a micro-optimisation. Mitigations: run the gateway as a Deployment with a PodDisruptionBudget and anti-affinity; give the agents a persistent sending queue so a gateway restart buffers rather than drops; and monitor the gateway with a Prometheus that does not depend on the gateway.

### 4. Processor order, and what each position is defending against

The canonical gateway order:

```
memory_limiter → k8sattributes/resourcedetection → filter/transform → tail_sampling → batch → exporter
```

Every position is load-bearing. Take them in turn.

**`memory_limiter` first — the mechanism, not the slogan.** The processor (`processor/memorylimiterprocessor`) polls process memory every `check_interval` and enforces two thresholds:

- **Hard limit** = `limit_mib` (or `limit_percentage` of the cgroup limit).
- **Soft limit** = `limit_mib − spike_limit_mib`. The default `spike_limit_mib` is **20% of `limit_mib`**, and the upstream guidance is to keep it at roughly that.

Above the soft limit the processor enters memory-limited mode and **returns a non-permanent error** to the component that called it. Above the hard limit it *additionally* forces a garbage collection. (Recent versions back that forced GC off exponentially when it proves ineffective — which it does when memory is held by live references such as a full exporter queue during a downstream outage.)

The word **non-permanent** is the whole contract. Receivers seeing a non-permanent error are expected to **retry**, and to apply backpressure to their own source — for OTLP/gRPC that means not acknowledging the RPC, which makes the sending SDK or agent hold the data. That is how backpressure propagates all the way to the edge instead of turning into loss.

And here is the sharp edge, stated in the component's own README: **data is permanently lost if the component preceding the memory limiter does not correctly retry.** Which means ordering matters twice — the limiter must be first so the thing in front of it is a *receiver* (which retries), not another processor (which may not).

```
   WHY memory_limiter GOES FIRST — TWO ORDERINGS, TWO OUTCOMES
   ═══════════════════════════════════════════════════════════════════════════

   ✅ CORRECT: receiver → memory_limiter → … → batch → exporter

      backend slows ──▶ exporter queue fills ──▶ heap rises
                                                   │
                     memory_limiter sees soft limit crossed
                                                   │
                        returns NON-PERMANENT error to the RECEIVER
                                                   │
                     receiver does not ACK the gRPC stream
                                                   │
                     sending agent/SDK holds the batch and retries
                                                   │
                            ⇒ BACKPRESSURE, NOT LOSS
                            ⇒ heap stops growing, collector survives

   ❌ WRONG: receiver → batch → memory_limiter → exporter

      backend slows ──▶ exporter queue fills ──▶ heap rises
                                                   │
                     batch has ALREADY accumulated 8,192 items × N
                     in RAM before the limiter is consulted
                                                   │
                     memory_limiter refuses ──▶ error goes to BATCH,
                     which is not a receiver and cannot retry upstream
                                                   │
                            ⇒ THE BATCH IS DROPPED
                            ⇒ heap keeps growing anyway → OOM
```

Also from the README, and easy to miss: **set `GOMEMLIMIT` to ~80% of the hard limit** in addition to configuring the processor. The Go runtime's own soft memory limit makes the GC work harder before the processor has to intervene, and the two together are far more effective than either alone. And be aware that incoming data consumes memory *before* the limiter can reject it — particularly for non-OTLP receivers — so leave headroom.

**`k8sattributes` / `resourcedetection` second — on the agent.** These enrich the resource. `k8sattributes` associates telemetry with a Pod by IP or by the connection's source address, then attaches `k8s.pod.name`, `k8s.namespace.name`, `k8s.deployment.name`, labels and annotations. **It does this by watching Kubernetes objects through an informer**, which is real API-server load at fleet scale. Two knobs matter:

```yaml
processors:
  k8sattributes:
    auth_type: serviceAccount
    passthrough: false
    # CRITICAL at scale: only watch pods on THIS node.
    filter:
      node_from_env_var: KUBE_NODE_NAME     # set via the downward API
    # The informer resync re-enqueues every cached object. At 100k pods this
    # is a periodic CPU and GC spike for no benefit, because watch events
    # already deliver changes. Upstream recommends 0s for large clusters.
    watch_sync_period: 0s
    extract:
      metadata:
        - k8s.namespace.name
        - k8s.pod.name
        - k8s.pod.uid
        - k8s.node.name
        - k8s.deployment.name
      labels:
        - tag_name: tenant
          key: team
          from: pod
    pod_association:
      - sources: [{ from: resource_attribute, name: k8s.pod.ip }]
      - sources: [{ from: connection }]
```

`filter.node_from_env_var` is the difference between 1,250 agents each watching every pod in the cluster and each watching ~30. Without it, `k8sattributes` on a large DaemonSet is a self-inflicted API-server denial of service. RBAC follows the attributes you extract: `get`/`list`/`watch` on `pods` and `namespaces` always; `replicasets` only if you extract deployment UIDs or deployment labels; `nodes` only if you use `k8s.node.uid`; `jobs` only for `k8s.cronjob.uid`. Grant the minimum, because each additional informer is another cache in every agent's heap.

**`filter` / `transform` third.** Drop and redact before anything expensive happens. On the agent this is where the cheap, universal drops go; on the gateway it is where policy-scoped and schema-normalisation work goes (§8).

**`tail_sampling` fourth — gateway only.** §5.

**`batch` last.** Batching before filtering wastes CPU packing items that are about to be dropped, and — worse — inflates the heap the memory limiter is trying to protect. Defaults (`processor/batchprocessor`): `send_batch_size: 8192`, `timeout: 200ms`, `send_batch_max_size: 0` (no upper bound, so a batch *trigger* of 8,192 can still emit a much larger batch if a single incoming request was bigger). If your backend has a request-size limit — Mimir's `-distributor.max-recv-msg-size`, Tempo's gRPC max message size — set `send_batch_max_size` explicitly or you will meet it as a 4xx under load.

### 5. Why tail sampling can only live on the gateway — and what it costs

**The requirement.** A tail-sampling policy like "keep any trace containing an error, or whose total duration exceeds 2 s" is a predicate over the *whole trace*. You cannot evaluate it until you have all the spans.

**The obstacle.** In a real distributed system, the spans of one trace are emitted by processes on many different nodes. Each node's agent sees only its own fragment. A tail-sampling processor on an agent evaluates the policy against a fragment and answers a different question than the one you asked.

**The subtle version of the same bug**, which is the one that actually bites: a *gateway* fleet behind an ordinary Kubernetes Service. The Service load-balances TCP connections, so span batches for one trace scatter across replicas. Each replica evaluates the policy on its own partial view. There is no error. Nothing logs a warning. You simply get a sampled trace corpus that is systematically missing the spans that lived on other replicas — and the traces you *do* keep look complete enough that nobody notices.

```
   THE SPLIT-TRACE FAILURE, AND THE FIX
   ═══════════════════════════════════════════════════════════════════════════

   ❌ WITHOUT trace-ID-aware routing

     trace T = spans {A, B, C, D}.  B carries status=ERROR.

     agent(node1) ──[A,B]──┐
     agent(node2) ──[C]────┼──▶ plain Service (round-robins connections)
     agent(node3) ──[D]────┘
                              ├──▶ gateway-1 receives [A, B]
                              │      policy: "keep if any span has ERROR"
                              │      sees B ⇒ KEEP {A,B}
                              ├──▶ gateway-2 receives [C]
                              │      sees no error ⇒ DROP {C}
                              └──▶ gateway-3 receives [D]
                                     sees no error ⇒ DROP {D}

     RESULT: the backend stores a 2-span "trace" for a 4-span request.
             The waterfall is missing the two spans that would have
             explained the error. No component reported a problem.

   ✅ WITH loadbalancing exporter, routing_key: traceID

     agent(node1) ──[A,B]──┐
     agent(node2) ──[C]────┼──▶ loadbalancing exporter
     agent(node3) ──[D]────┘      hash(T) → gateway-2, ALWAYS
                                        │
                                        ▼
                                  gateway-2 receives [A,B,C,D]
                                  policy sees B ⇒ KEEP all four
```

**The routing layer, configured.** The `loadbalancing` exporter (contrib) creates one `otlp` sub-exporter per resolved endpoint and hashes a routing key to pick one:

```yaml
exporters:
  loadbalancing:
    protocol:
      otlp:
        timeout: 10s
        # Sub-exporters have their OWN queue/retry settings, and by default
        # queueing and retry are DISABLED — the loadbalancing exporter will
        # NOT reroute to a healthy endpoint on failure unless you enable them.
        sending_queue:
          enabled: true
          queue_size: 10000
        retry_on_failure:
          enabled: true
    resolver:
      k8s:
        service: otel-gateway.observability     # a HEADLESS Service
        ports: [4317]
    routing_key: traceID
```

Four properties worth knowing:

1. **Routing keys and their validity** (from the exporter's README):

   | `routing_key` | valid for | routes by |
   |---|---|---|
   | `service` | spans, logs, metrics | `service.name` resource attribute |
   | `traceID` | spans, logs | the trace ID |
   | `resource` | logs, metrics | hash of *all* resource attributes |
   | `metric` | metrics | metric name |
   | `streamID` | metrics | resource + scope + datapoint attributes |
   | `attributes` | spans, logs, metrics | named attribute values |

2. **The defaults are signal-specific**, and this is the one people misremember: with no `routing_key` set, the default is `traceID` for traces and **`service` for logs and metrics.** Logs do *not* default to trace-ID routing.

3. **Rebalancing moves about `R/N` routes** when the endpoint list changes (R routes, N backends), so a gateway rollout redistributes a fraction of traces and briefly splits some. If that matters, put the `groupbytrace` processor in front so whole traces are dispatched atomically.

4. **The RBAC requirement for the `k8s` resolver** is `get`/`list`/`watch` on `discovery.k8s.io/v1` `EndpointSlice` objects in the target namespace. Without it the resolver cache stays empty and the exporter logs `couldn't find the exporter for the endpoint ""` — a genuinely confusing symptom for a permissions problem.

**The routing key is use-case-specific, not universal.** Trace-ID routing exists to satisfy trace *completeness*. A log pipeline has no such constraint; routing logs by tenant or service gives stream affinity and even distribution, which is what a log backend wants. A metrics pipeline routing by `streamID` keeps a series' datapoints together so a downstream aggregation is correct. Copying trace-ID routing onto a log pipeline is a category error that also distributes logs badly, because most logs have no trace ID and are then randomly distributed.

**Sizing the gateway from the tail-sampling config.** The processor holds traces in memory until it decides. The knobs (from `processor/tailsamplingprocessor`):

| Option | Default | What it does |
|---|---:|---|
| `decision_wait` | **30s** | how long to accumulate spans before deciding |
| `num_traces` | **50000** | traces kept in memory |
| `expected_new_traces_per_sec` | 0 | pre-sizes data structures |
| `num_shards` | **1** (max 256) | parallel event loops; traces routed to shards by trace-ID hash |
| `sampling_strategy` | `trace-complete` | evaluate on the timer with accumulated data; `span-ingest` evaluates per batch at ingest and rejects stateful policies |
| `decision_cache.sampled_cache_size` | 0 (off) | LRU of "keep" decisions, for late spans |
| `decision_cache.non_sampled_cache_size` | 0 (off) | LRU of "drop" decisions |
| `maximum_trace_size_bytes` | unset | hard drop for pathological traces |

The memory arithmetic:

```
   in-flight traces ≈ trace_rate_per_gateway × decision_wait
   memory           ≈ in-flight traces × avg_spans_per_trace × bytes_per_span

   Example: 20,000 traces/s across 5 gateways = 4,000 traces/s each
            decision_wait 30 s
            in-flight = 4,000 × 30            = 120,000 traces
            ⇒ num_traces: 50000 (the DEFAULT) IS TOO SMALL BY 2.4×
              — traces are evicted before their decision, and you get
                "sampling_trace_dropped_too_early"

            at 12 spans/trace × ~400 B/span:
              120,000 × 12 × 400 B            ≈ 576 MB just for trace data
            plus decision caches, plus batch, plus receiver buffers
            ⇒ size the gateway at ≥ 2 GB and set memory_limiter accordingly
```

`decision_wait` is the parameter with the most leverage and the least attention. It must exceed the p99 *trace duration*, not the p99 request latency — a trace with an async tail (a background job spawned by the request) finishes long after the response. Set it too low and long traces are decided on partial data, which reintroduces the fragment problem you built the routing tier to avoid. Set it too high and gateway memory scales linearly with it.

**Decision caches are how you handle late spans.** A span that arrives after its trace has been evicted has no trace to join. With `sampled_cache_size` and `non_sampled_cache_size` set (both default to *off*), the processor remembers the decision for that trace ID and applies it to the straggler. Upstream guidance is to set them **much greater than `num_traces`**, so decisions outlive the span data. Watch `otelcol_processor_tail_sampling_sampling_late_span_age` to see whether you need them.

### 6. Attribute limits — lesson 1's third enforcement gate

There is no scrape in the OTLP path, so `metric_relabel_configs` does not exist. Cardinality enforcement has to happen in the pipeline. The tools:

```yaml
processors:
  # 1. The blunt instrument: delete or hash attributes that must never
  #    become labels. Runs on the AGENT so the bytes never travel.
  transform/cardinality:
    error_mode: ignore
    metric_statements:
      - context: datapoint
        statements:
          - delete_key(attributes, "request.id")
          - delete_key(attributes, "user.id")
          - delete_key(attributes, "session.id")
          # Collapse a high-cardinality path into a bounded route template.
          - replace_pattern(attributes["http.route"], "/[0-9a-f]{8,}", "/{id}")
          - replace_pattern(attributes["http.route"], "/[0-9]+", "/{n}")
    trace_statements:
      - context: span
        statements:
          # Redaction is a policy that changes on a legal timescale.
          # It belongs here, not in every app's release cycle.
          - replace_pattern(attributes["db.query.text"], "'[^']*'", "'?'")

  # 2. Drop whole metrics you never query. Cheapest possible saving.
  filter/noise:
    error_mode: ignore
    metrics:
      metric:
        - 'name == "go_gc_duration_seconds"'
        - 'IsMatch(name, "^promhttp_.*")'
    traces:
      span:
        - 'attributes["http.route"] == "/healthz"'
        - 'attributes["http.route"] == "/readyz"'
```

Two things to be precise about:

**`transform` uses OTTL** (OpenTelemetry Transformation Language), and the *context* you pick — `resource`, `scope`, `span`, `spanevent`, `metric`, `datapoint`, `log` — determines both what you can address and how expensive the statement is. A `datapoint`-context statement runs per datapoint; a `resource`-context statement runs once per resource. Prefer the outermost context that can express the rule (§1, consequence 1).

**`error_mode`** decides what happens when a statement fails — typically because the attribute is absent. `ignore` logs and continues; `propagate` fails the whole batch. On a cardinality-defence processor you almost always want `ignore`, because "the attribute wasn't there" is the normal case.

**The remaining gap.** Neither processor bounds cardinality *dynamically* — they enforce a list you wrote. A genuinely new unbounded attribute still gets through. The backstop is the same as lesson 3's: per-tenant limits in Mimir. Three layers, and the OTLP path skips the middle one, which is exactly why the Collector layer has to be thorough.

### 7. The Collector's own health is a first-class signal

A silently-degrading Collector — refusing spans under memory pressure, queueing without draining, failing exports — looks from the outside exactly like "the app isn't emitting telemetry." Instrument it.

The Collector emits its internal metrics with the prefix `otelcol_`, defined per component in `metadata.yaml` and documented in each component's `documentation.md`. The ones worth alerting on:

| Metric | What it tells you |
|---|---|
| `otelcol_receiver_accepted_spans` / `_refused_spans` / `_failed_spans` | intake health; `refused` climbing means backpressure is reaching the edge |
| `otelcol_processor_memory_limiter_refused_spans` (and `_metric_points`, `_log_records`) | the memory limiter is actively shedding — the single clearest "this Collector is under-provisioned" signal |
| `otelcol_processor_memory_limiter_accepted_spans` | denominator for the refusal ratio |
| `otelcol_exporter_sent_spans` / `_send_failed_spans` | export success |
| `otelcol_exporter_enqueue_failed_spans` | the sending queue was full — data lost at the exporter |
| `otelcol_exporter_queue_size` vs `otelcol_exporter_queue_capacity` | queue saturation; the ratio is the number to graph |
| `otelcol_exporter_in_flight_requests` | concurrency against the backend |
| `otelcol_processor_batch_batch_send_size` | are batches reaching `send_batch_size`, or timing out at 200 ms? |
| `otelcol_processor_batch_timeout_trigger_send` vs `_batch_size_trigger_send` | which trigger is firing — tells you if you are under-loaded or over-batched |
| `otelcol_process_runtime_heap_alloc_bytes`, `otelcol_process_memory_rss` | the memory the limiter is watching |

And for the tail-sampling processor specifically:

| Metric | What it tells you |
|---|---|
| `otelcol_processor_tail_sampling_sampling_traces_on_memory` | in-flight traces; compare against `num_traces` |
| `otelcol_processor_tail_sampling_sampling_trace_dropped_too_early` | **`num_traces` is too small or `decision_wait` too long** — traces evicted before a decision |
| `otelcol_processor_tail_sampling_sampling_late_span_age` | how late stragglers arrive; drives decision-cache sizing |
| `otelcol_processor_tail_sampling_count_traces_sampled` | effective sampling rate, per policy |
| `otelcol_processor_tail_sampling_sampling_decision_timer_latency` | decision-loop health |
| `otelcol_processor_tail_sampling_sampling_policy_evaluation_error` | a policy is erroring — often an OTTL condition against an absent attribute |
| `otelcol_processor_tail_sampling_traces_dropped_too_large` | `maximum_trace_size_bytes` is firing |

**A correction worth stating:** `otelcol_processor_dropped_spans` — widely cited in older material, including the previous version of this lesson — is **not** a metric the current Collector emits. Drops are attributed to the component that dropped them: `otelcol_processor_memory_limiter_refused_spans`, `otelcol_receiver_refused_spans`, `otelcol_exporter_enqueue_failed_spans`, `otelcol_exporter_send_failed_spans`. If your alerts reference the old name they have been silently matching nothing.

The alert set:

```yaml
groups:
  - name: otel-collector
    rules:
      - alert: CollectorSheddingLoad
        expr: |
          rate(otelcol_processor_memory_limiter_refused_spans[5m]) > 0
        for: 5m
        labels: { severity: page }
        annotations:
          summary: '{{ $labels.instance }} memory_limiter is refusing spans'

      - alert: CollectorExporterQueueSaturated
        expr: |
          otelcol_exporter_queue_size / otelcol_exporter_queue_capacity > 0.8
        for: 10m
        labels: { severity: page }

      - alert: CollectorDroppingAtExporter
        expr: |
          rate(otelcol_exporter_enqueue_failed_spans[5m]) > 0
            or rate(otelcol_exporter_send_failed_spans[5m]) > 0
        for: 5m
        labels: { severity: page }

      - alert: TailSamplingEvictingTracesEarly
        expr: |
          rate(otelcol_processor_tail_sampling_sampling_trace_dropped_too_early[5m]) > 0
        for: 10m
        labels: { severity: ticket }
        annotations:
          summary: 'raise num_traces or lower decision_wait on {{ $labels.instance }}'
```

**Scrape the Collector with something that is not the Collector.** If the gateway exports its own metrics through itself, a gateway outage takes its own alerting with it. Point a small Prometheus at the Collectors' `:8888/metrics` endpoints directly.

### 8. Semantic conventions: schema URLs, not hope

Stable attribute names — `service.name`, `k8s.pod.name`, `http.request.method` — are the entire reason to standardise: they make cross-service queries possible and make a backend swap a config change. But conventions churn. `http.method` became `http.request.method`. `gen_ai.system` became `gen_ai.provider.name`. GenAI conventions moved to a separate repository outright.

OTel's answer is more structured than "pin a version." Three mechanisms:

**(a) Stability levels.** Every attribute and signal carries a marker — Development (formerly Experimental), then Stable. A Stable convention carries backward-compatibility guarantees; a Development one explicitly does not. **Look up the actual marker of every convention you build a dashboard on.** As of the August 2026 tree, essentially the entire `gen_ai.*` surface is still marked Development.

**(b) Schema URLs on the wire.** Every `ResourceSpans`/`ResourceMetrics`/`ResourceLogs` message carries a `schema_url` field, and so does each `Scope*` message (where it applies more narrowly). So the payload states which convention version it was produced against. That is a fact you can route on, not something you have to guess.

**(c) Schema files and translation.** OpenTelemetry publishes machine-readable schema files (format v1.1.0) that describe how to transform telemetry from one version to another — `rename_attributes`, `rename_metrics` and friends. A schema translator can read the incoming `schema_url` and mechanically convert to a target version. That is what turns convention churn from a coordinated migration into a pipeline concern.

**In practice, today**, the workable pattern is Collector-side normalisation with an explicit target version:

```yaml
processors:
  transform/semconv-normalise:
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          # Old SDKs still emit http.method; normalise to the current name.
          - set(attributes["http.request.method"], attributes["http.method"])
              where attributes["http.method"] != nil
                and attributes["http.request.method"] == nil
          - delete_key(attributes, "http.method")

          - set(attributes["http.response.status_code"], attributes["http.status_code"])
              where attributes["http.status_code"] != nil
                and attributes["http.response.status_code"] == nil
          - delete_key(attributes, "http.status_code")

          # GenAI: gen_ai.system was superseded by gen_ai.provider.name.
          - set(attributes["gen_ai.provider.name"], attributes["gen_ai.system"])
              where attributes["gen_ai.system"] != nil
                and attributes["gen_ai.provider.name"] == nil
          - delete_key(attributes, "gen_ai.system")
      - context: resource
        statements:
          # Stamp the version the pipeline guarantees downstream.
          - set(attributes["telemetry.semconv.normalised_to"], "1.44.0")
```

**The staff position:** apps emit whatever their SDK version knows; the Collector normalises to one target version; dashboards and alerts are written against that one version only. Nobody re-releases a service because a convention moved. The cost is that the normalisation block grows over time and must be reviewed — which is a config-review problem, not a release-coordination problem, and that trade is the entire point of the Collector-as-seam argument.

### 9. Migration: dual-run, never big-bang

The Collector can subsume the metrics pipeline from lesson 3 as well as traces, which makes a phased migration genuinely possible:

```
   MIGRATION SEQUENCE — EACH STEP INDEPENDENTLY REVERSIBLE
   ═══════════════════════════════════════════════════════════════════════════

   PHASE 0  BASELINE
     Prometheus scrapes everything → Mimir.  Jaeger agents → Jaeger.
     Fluent Bit → Elasticsearch.  Nothing changes yet.

   PHASE 1  DEPLOY THE COLLECTOR ALONGSIDE, EXPORTING NOWHERE NEW
     Agent DaemonSet with:
       receivers:  [otlp, jaeger]        ← accepts BOTH new OTLP and
                                            legacy Jaeger thrift/gRPC
       exporters:  [otlp/jaeger-backend] ← same destination as before
     Apps keep their Jaeger clients. Zero behaviour change; you are only
     proving the Collector can carry the traffic.

   PHASE 2  DUAL-EXPORT TRACES
       exporters: [otlp/jaeger-backend, otlp/tempo]
     Both backends receive everything. Compare trace counts, span counts,
     and a sample of waterfalls. This is the parity gate.

   PHASE 3  MOVE METRICS INTO THE COLLECTOR
       receivers: [prometheus]                  ← the Collector SCRAPES
       exporters: [prometheusremotewrite/mimir] ← and remote-writes
     Run alongside the existing Prometheus shards writing to the same
     Mimir tenant; Mimir's HA tracker dedupes if you label them as a pair.
     Compare series counts before removing the old shards.

   PHASE 4  MOVE APPS TO OTLP, SERVICE BY SERVICE
     Each app switches its exporter to OTLP at localhost:4318. The agent's
     jaeger receiver stays until the last app has moved.

   PHASE 5  ENABLE TAIL SAMPLING
     Only now — with a full trace corpus flowing through the gateway and
     the routing tier verified — turn on tail_sampling. Start with
     always_sample plus a probabilistic policy so you can compare volumes,
     then tighten.

   PHASE 6  DECOMMISSION
     Remove the jaeger receiver, the legacy backend, the old Prometheus
     shards. Each removal is a separate change with a separate rollback.
```

The rule underneath: **no phase both changes the data path and changes the destination.** Every real production migration in the references ran a parity period; the ones that did not are not written up because they are not success stories.

### 10. GPU-fleet tie: GenAI conventions and the join

OTel's GenAI conventions are how you get token and model telemetry without touching inference code. As of the August 2026 tree they live in their own repository (`open-telemetry/semantic-conventions-genai`) and **the entire surface is marked Development.** Treat the names as moving targets and normalise at the gateway.

The pieces that matter for a GPU fleet:

**Metrics** (all Histograms, all Development):

| Metric | Unit | What it measures |
|---|---|---|
| `gen_ai.client.token.usage` | `{token}` | input and output tokens, split by the `gen_ai.token.type` attribute (`input` / `output`) |
| `gen_ai.client.operation.duration` | `s` | end-to-end client-observed operation duration |
| `gen_ai.server.time_to_first_token` | `s` | **TTFT** — the prefill-latency SLI |
| `gen_ai.server.time_per_output_token` | `s` | **TPOT** — the decode-throughput SLI |
| `gen_ai.server.request.duration` | `s` | server-side request duration |
| `gen_ai.client.operation.time_to_first_chunk` | `s` | streaming first-chunk latency |

**Note the shape of the token metric**, because it is a change people miss: tokens are a *single histogram* with a `gen_ai.token.type` dimension, not a pair of counters. The convention even specifies the bucket boundaries — `[1, 4, 16, 64, 256, 1024, 4096, 16384, 65536, 262144, 1048576, 4194304, 16777216, 67108864]` — which is a rare and welcome case of a convention doing the bucket-placement work from lesson 2 for you.

**Key attributes:** `gen_ai.operation.name` (`chat`, `embeddings`, `execute_tool`, `generate_content`, …), `gen_ai.provider.name` (the successor to `gen_ai.system`), `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.token.type`, plus span attributes `gen_ai.usage.input_tokens` and `gen_ai.usage.output_tokens` and the cache-accounting pair `gen_ai.usage.cache_read.input_tokens` / `gen_ai.usage.cache_creation.input_tokens`.

**The join is the point.** The Collector attaches `k8s.node.name`, `k8s.namespace.name` and your `tenant` label to the same resource that carries `gen_ai.*`. Downstream, one query can relate tokens delivered to the GPU that delivered them:

```promql
# Tokens/second by node and model — the numerator of a goodput SLI.
sum by (k8s_node_name, gen_ai_request_model) (
  rate(gen_ai_client_token_usage_sum{gen_ai_token_type="output"}[5m])
)
```

Two cardinality warnings, straight from lesson 1: `gen_ai.request.model` is bounded (a handful of models) and belongs as a label; `gen_ai.conversation.id`, `gen_ai.response.id` and anything resembling a prompt are unbounded and must be span attributes or dropped. And **never** put `gen_ai.input.messages` or `gen_ai.output.messages` on a metric — they are message content, they are potentially PII, and the conventions treat them as opt-in for exactly that reason.

The precise mapping from these signals to a GPU goodput SLO is lesson 9's job; what this lesson delivers is that the signals arrive, correctly named, joined to infrastructure attributes, without a single change to serving code.

## Perspectives

**Migration economics.** OTel adoption is a cost and lock-in decision, not a purity argument. Removing proprietary vendor agents recovers measurable per-host CPU and memory, and — more importantly — converts a future backend migration from an app-by-app re-instrumentation programme measured in quarters into a Collector config change measured in weeks. The Collector-as-seam argument is fundamentally a real-options argument: you are buying the option to change backends later, and the premium is the operational cost of running the Collector fleet.

**Protocol design.** Everything in this lesson is downstream of one decision: OTLP standardising on protobuf over gRPC/HTTP with a Resource → Scope → Signal nesting. The nesting is what makes enrichment cheap (attributes written once per resource, not once per span), what makes service-level routing a single hash per batch, and what lets one connection multiplex many services. A flat wire format would have produced a very different, much worse Collector.

**Operational risk.** The two-tier topology creates a failure domain that did not previously exist: a shared gateway whose backpressure or outage affects every team's telemetry at once. The mitigations are structural, not heroic — `memory_limiter` first so backpressure propagates instead of turning into loss, persistent sending queues on the agents so a gateway restart buffers, a PodDisruptionBudget so a node drain cannot take the fleet, and an external observer that does not depend on the thing it observes.

**Standards governance.** Convention churn is a recurring tax, not a one-time migration. The correct response is the layered one: know the stability marker of every convention you depend on, read the `schema_url` the producer stamped, normalise to one target version in the gateway, and write every dashboard against that single version. The alternative — apps agreeing to move in lockstep — has never worked in any organisation with more than about five services.

**Security and compliance.** The gateway is where redaction belongs, and that is a governance argument as much as an engineering one. Redaction rules change when legal asks, which is a different cadence from application releases; putting them in a Collector config makes "we now redact X" a same-day change instead of a quarter-long rollout. It also concentrates the credentials: node agents hold no backend write tokens, so compromising a node does not give an attacker write access to the observability plane.

## Real-world use cases

- **Cloudflare, "Adopting OpenTelemetry for our logging pipeline."** Cloudflare replaced syslog-ng with the OpenTelemetry Collector across a logging pipeline handling millions of events per second. **What it shows:** the Collector is production infrastructure for logs, not just a trace shim — which matters because it is the strongest available evidence that "one Collector fleet for all three signals" is a real option rather than a diagram. It also demonstrates the cost model: the migration was justified on operational consolidation, not on the Collector being faster than what it replaced.

- **Grafana Labs, "How Grafana Labs enables horizontally scalable tail sampling in the OpenTelemetry Collector."** A production write-up of exactly the split-trace failure in §5 and the routing layer that fixes it. **What it shows:** this is not a theoretical edge case — an organisation that runs this stack for a living hit it, and the fix required a dedicated routing tier rather than a tuning change. It is also the clearest available demonstration that a stateful processor cannot be scaled horizontally behind a naive load balancer, which is a general lesson well beyond OTel.

- **Shopify's OTel migration, as reported by Dotan Horovits, "Shopify's Journey to Planet-Scale Observability."** Flagged explicitly as a third-party analyst write-up of a conference talk, **not** Shopify's own engineering blog — treat the specifics as reported rather than primary. The reported substance: Shopify built an in-house OTel-based stack to escape rising vendor costs and cited proprietary-agent overhead in the 15–20% range. **What it shows:** the economics argument is what actually drives these migrations at scale, and vendor-agent overhead is a measurable line item rather than a talking point.

- **The `k8sattributes` informer-storm pattern.** Not a single postmortem but a repeated operational failure: a `k8sattributes` processor deployed as a DaemonSet without `filter.node_from_env_var`, so every agent watches every Pod in the cluster. At 1,250 nodes and 40,000 pods that is 1,250 full Pod informers against one API server. **What it shows:** enrichment is never free, and the cost of a Kubernetes-aware processor scales with cluster size times agent count — a product that is easy to miss when you test the config on a three-node dev cluster.

## Worked example

**A complete two-tier configuration for the 4,000-node GPU fleet**, annotated. This is the artifact the Practice section asks you to adapt.

---

**Agent — DaemonSet, one per node.**

```yaml
# otel-agent-config.yaml — mounted as a ConfigMap into the DaemonSet
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        # Bound a single request so one pathological client cannot
        # allocate an unbounded buffer before memory_limiter sees it.
        max_recv_msg_size_mib: 16
      http:
        endpoint: 0.0.0.0:4318

  # Subsume the DCGM scrape (lesson 3 phase 3).
  prometheus:
    config:
      scrape_configs:
        - job_name: dcgm-exporter
          scrape_interval: 15s
          kubernetes_sd_configs:
            - role: pod
              namespaces: { names: [gpu-operator] }
          relabel_configs:
            # Scrape only pods on THIS node — the DaemonSet equivalent of
            # hashmod sharding, with perfect locality.
            - source_labels: [__meta_kubernetes_pod_node_name]
              regex: ${env:KUBE_NODE_NAME}
              action: keep
            - source_labels: [__meta_kubernetes_pod_label_app]
              regex: nvidia-dcgm-exporter
              action: keep

processors:
  # ── 1. FIRST. Always. See §4.
  memory_limiter:
    check_interval: 1s          # upstream recommends 1s
    limit_percentage: 75        # of the container's cgroup limit
    spike_limit_percentage: 15  # soft limit = 60 % of the cgroup limit

  # ── 2. Enrichment: node-local metadata only exists here.
  k8sattributes:
    auth_type: serviceAccount
    passthrough: false
    filter:
      node_from_env_var: KUBE_NODE_NAME   # ← without this, 1,250 full informers
    watch_sync_period: 0s                 # ← no periodic resync at fleet scale
    extract:
      metadata:
        - k8s.namespace.name
        - k8s.pod.name
        - k8s.pod.uid
        - k8s.node.name
        - k8s.deployment.name
      labels:
        - tag_name: tenant
          key: team
          from: pod
    pod_association:
      - sources: [{ from: resource_attribute, name: k8s.pod.ip }]
      - sources: [{ from: connection }]

  resourcedetection:
    detectors: [env, system, k8snode]
    timeout: 5s
    override: false             # never clobber what the SDK set deliberately

  # ── 3. Drop cheaply, at the edge.
  filter/noise:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.route"] == "/healthz"'
        - 'attributes["http.route"] == "/readyz"'
        - 'attributes["http.route"] == "/metrics"'
    metrics:
      metric:
        - 'IsMatch(name, "^go_gc_.*")'
        - 'IsMatch(name, "^promhttp_.*")'

  # ── 4. Cardinality defence — lesson 1's third gate.
  transform/cardinality:
    error_mode: ignore
    metric_statements:
      - context: datapoint
        statements:
          - delete_key(attributes, "request.id")
          - delete_key(attributes, "user.id")
          - delete_key(attributes, "gen_ai.conversation.id")
          - replace_pattern(attributes["http.route"], "/[0-9a-f]{8,}", "/{id}")

  # ── 5. LAST. Batch only what survived.
  batch:
    send_batch_size: 8192
    send_batch_max_size: 16384   # bound it: the gateway has a recv limit
    timeout: 5s                  # 200 ms default is too chatty node→gateway

exporters:
  # Route to the gateway by trace ID so tail sampling is correct (§5).
  loadbalancing:
    protocol:
      otlp:
        timeout: 10s
        sending_queue:
          enabled: true          # NOT the default for sub-exporters
          queue_size: 10000
          # Survive a gateway restart without losing the buffer.
          storage: file_storage/queue
        retry_on_failure:
          enabled: true
          initial_interval: 1s
          max_interval: 30s
    resolver:
      k8s:
        service: otel-gateway.observability   # headless Service
        ports: [4317]
    routing_key: traceID

  # Metrics do not need trace-ID routing; send them straight through.
  otlp/gateway-metrics:
    endpoint: otel-gateway.observability.svc:4317
    tls: { insecure: true }
    sending_queue: { enabled: true, queue_size: 10000 }

extensions:
  file_storage/queue:
    directory: /var/lib/otelcol/queue
  health_check:
    endpoint: 0.0.0.0:13133

service:
  extensions: [file_storage/queue, health_check]
  telemetry:
    metrics:
      # Scraped directly by a small independent Prometheus (§7).
      readers:
        - pull: { exporter: { prometheus: { host: 0.0.0.0, port: 8888 } } }
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, k8sattributes, resourcedetection, filter/noise, batch]
      exporters: [loadbalancing]
    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, k8sattributes, resourcedetection,
                   filter/noise, transform/cardinality, batch]
      exporters: [otlp/gateway-metrics]
```

Read three details that are easy to skim past:

- `batch.timeout` is raised from the 200 ms default to 5 s on the agent. Node→gateway is an internal hop where latency does not matter and request count does: 1,250 agents at 200 ms is 6,250 requests/s of pure overhead; at 5 s it is 250/s.
- The trace pipeline has **no** `transform/cardinality`. Span attributes are not indexed the way metric labels are (lesson 1 §5), so high-cardinality span attributes are fine — that is what traces are *for*.
- `sending_queue.storage: file_storage/queue` gives the queue a disk backing, so a gateway outage buffers to disk rather than to a bounded heap. This is the single highest-value resilience setting on the agent tier and it is off by default.

---

**Gateway — Deployment, horizontally scaled.**

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        max_recv_msg_size_mib: 32     # must exceed the agent's send_batch_max_size

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 6144                   # container limit 8 GiB
    spike_limit_mib: 1229             # ≈20 % of limit_mib, per upstream guidance

  # Normalise convention drift before anything downstream sees it (§8).
  transform/semconv-normalise:
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          - set(attributes["http.request.method"], attributes["http.method"])
              where attributes["http.method"] != nil
                and attributes["http.request.method"] == nil
          - delete_key(attributes, "http.method")
          - set(attributes["gen_ai.provider.name"], attributes["gen_ai.system"])
              where attributes["gen_ai.system"] != nil
                and attributes["gen_ai.provider.name"] == nil
          - delete_key(attributes, "gen_ai.system")

  tail_sampling:
    decision_wait: 20s            # must exceed p99 TRACE duration, not p99 latency
    num_traces: 200000            # sized in §5: 4,000 traces/s × 20 s × 2.5 margin
    expected_new_traces_per_sec: 4000
    num_shards: 8                 # parallel decision loops; trace-ID hashed
    decision_cache:
      sampled_cache_size: 1000000       # ≫ num_traces, for late spans
      non_sampled_cache_size: 2000000
    maximum_trace_size_bytes: 10485760  # 10 MiB — drop pathological traces
    policies:
      - name: errors
        type: status_code
        status_code: { status_codes: [ERROR] }

      - name: slow-inference
        type: and
        and:
          and_sub_policy:
            - name: is-inference
              type: string_attribute
              string_attribute:
                key: gen_ai.operation.name
                values: [chat, generate_content]
            - name: slow
              type: latency
              latency: { threshold_ms: 5000 }

      - name: gpu-tenants-higher-rate
        type: and
        and:
          and_sub_policy:
            - name: is-gpu-tenant
              type: string_attribute
              string_attribute:
                key: tenant
                values: ["team-vision", "team-nlp"]
                enabled_regex_matching: false
            - name: sample-10pct
              type: probabilistic
              probabilistic: { sampling_percentage: 10 }

      - name: baseline
        type: probabilistic
        probabilistic: { sampling_percentage: 1 }

  batch:
    send_batch_size: 8192
    send_batch_max_size: 16384
    timeout: 1s

exporters:
  otlp/tempo:
    endpoint: tempo-distributor.tracing.svc:4317
    sending_queue: { enabled: true, queue_size: 20000, num_consumers: 20 }
    retry_on_failure: { enabled: true, max_elapsed_time: 300s }

  prometheusremotewrite/mimir:
    endpoint: http://mimir-distributor.mimir.svc:8080/api/v1/push
    headers: { X-Scope-OrgID: gpu-infra }
    resource_to_telemetry_conversion:
      enabled: false        # ← IMPORTANT: true would turn EVERY resource
                            #   attribute into a metric label. That is a
                            #   cardinality bomb (lesson 1) wearing a
                            #   convenience flag.
    sending_queue: { enabled: true, queue_size: 20000 }

service:
  telemetry:
    metrics:
      readers:
        - pull: { exporter: { prometheus: { host: 0.0.0.0, port: 8888 } } }
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, transform/semconv-normalise, tail_sampling, batch]
      exporters: [otlp/tempo]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite/mimir]
```

**The policy set is an OR with an implicit union.** The tail-sampling processor evaluates every policy; a "sample" decision from any policy (absent a "drop") keeps the trace. So the effective corpus is: 100% of errors, 100% of slow inference calls, 10% of the two GPU tenants' remaining traffic, and 1% of everything else. Compute the volume before you deploy it:

```
   VOLUME AFTER SAMPLING — 20,000 traces/s fleet-wide, 12 spans/trace avg
   ═══════════════════════════════════════════════════════════════════════
   errors            0.3 % of traces  → 60 traces/s      (100 % kept)
   slow inference    0.8 % of traces  → 160 traces/s     (100 % kept)
   GPU tenants       35 % of traces   → 7,000 × 10 %  = 700 traces/s
   baseline          rest ≈ 12,780    → × 1 %        = 128 traces/s
                                        ───────────────────────────
   kept                                ≈ 1,048 traces/s  (5.2 % of traffic)
   spans/s kept      1,048 × 12                       ≈ 12,600 spans/s
   at ~400 B/span compressed                          ≈ 5.0 MB/s
   per day                                            ≈ 435 GB/day
   at 14-day retention                                ≈ 6.1 TB
```

That is the number to put in the design review, and it is the number that will be argued about. Note that the *error* and *slow* policies together are only 460 traces/s — the expensive part is the 10% tenant sample. Sampling policy is a budget allocation, and lesson 5 is about spending it well.

---

**The two questions to be able to answer cold.**

*Why can't `tail_sampling` live on the agent?* Because the agent DaemonSet sees only the spans emitted by processes on its own node, and the spans of one distributed trace are scattered across many nodes. A whole-trace predicate ("any span has ERROR") evaluated against a fragment answers a different question, silently. Only the gateway — behind trace-ID-keyed routing that guarantees every span of a trace lands on one replica — has the complete trace.

*Why route the logs pipeline by tenant instead of trace ID?* Because the routing key exists to satisfy a *specific* downstream constraint. Trace-ID routing satisfies trace completeness, which is a tail-sampling requirement. Logs have no completeness constraint; they have a stream-affinity and even-distribution requirement, which `service` or an attribute-based key satisfies. Routing logs by trace ID also distributes badly, because most log records carry no trace ID and are then scattered at random — which is why the exporter's default routing key for logs is `service`, not `traceID`.

## Practice

Feeds the [fleet observability design](../practice/fleet-observability/README.md).

Write the **telemetry-collection section of the fleet observability design doc**. Deliver:

1. **The two-tier topology diagram** in ASCII, showing which decision happens on which tier and *why it cannot happen on the other one*. Include the routing tier explicitly.
2. **Both complete pipeline configs** — agent and gateway, all three signals — with processors in the correct order. Annotate `memory_limiter` first and `batch` last with the mechanism, not the slogan: what error the limiter returns, who is expected to retry it, and what is lost if the preceding component does not.
3. **The routing decision, per signal.** Give the `routing_key` for traces, logs and metrics, with the constraint each one satisfies. State what the exporter's default would have been and whether you kept it.
4. **Tail-sampling sizing.** Derive `num_traces` from your trace rate, gateway count and `decision_wait`; justify `decision_wait` against the p99 *trace* duration; size gateway memory from the arithmetic in §5; and set the decision caches with a reason.
5. **The sampling volume calculation** — policies, per-policy keep rate, resulting traces/s, spans/s, bytes/day, and retention cost. Then state which policy dominates the bill.
6. **The cardinality gate for the OTLP path.** The `transform` and `filter` processors, the `resource_to_telemetry_conversion` setting with its justification, and a sentence on what this layer *cannot* catch and which downstream limit backstops it.
7. **The convention-normalisation block**, with your target semconv version, at least three real renames, and your policy for what happens when a new one appears.
8. **Collector self-observability** — the metric names you alert on (current ones, not `otelcol_processor_dropped_spans`), the thresholds, and the answer to "who scrapes the Collector."
9. **The `k8sattributes` scaling plan** — `filter.node_from_env_var`, `watch_sync_period`, the exact RBAC you grant and why each rule is needed, and an estimate of the API-server watch load at your node count.
10. **The migration sequence** from the existing Prometheus/Jaeger/Fluent stack, phase by phase, with the parity check and rollback for each phase.
11. **The GenAI enrichment plan** — which `gen_ai.*` signals you collect, which attributes become metric labels versus span attributes (with the cardinality justification), and how they join to `k8s.*` attributes.

**Acceptance criteria.** Done when (i) both configs would start without error against the current Collector, (ii) every processor's position is justified by what it defends against rather than by convention, (iii) the sampling volume number is derived and the dominant policy is named, and (iv) a reviewer can find, for each of the three signals, the answer to "where does this get dropped, and who would know."

## Common pitfalls

- **"The Collector is a passive proxy, so any topology works."** *Symptom:* tail-sampled traces that are systematically missing spans, with no errors anywhere. *Mechanism:* `tail_sampling` is a *stateful* stage. Scaling a stateful stage horizontally behind a connection-balancing Service splits each trace across replicas, and each replica decides on its fragment. The fix is a routing tier keyed on trace ID, not more replicas.

- **"`memory_limiter` anywhere in the chain protects the Collector."** *Symptom:* OOM despite a configured limiter. *Mechanism:* the limiter refuses by returning a non-permanent error to its *caller*. If the caller is `batch`, the batch is already in RAM and cannot retry upstream — so the data is dropped and the heap keeps growing. It must be first, so the caller is a receiver that can propagate backpressure to the sender.

- **"Auto-instrumentation means the SDK stays thin."** *Symptom:* changing the sampling ratio requires a redeploy of forty services. *Mechanism:* auto-instrumentation agents ship a sampler and an exporter endpoint as defaults, which are SDK-baked decisions by another name. Thin-SDK is enforced by reviewing the agent's environment variables, not inherited from the choice of agent.

- **"GenAI semantic conventions are stable."** *Symptom:* dashboards that break on an SDK upgrade. *Mechanism:* essentially the whole `gen_ai.*` surface is marked Development, the conventions moved to a separate repository, and names have already changed (`gen_ai.system` → `gen_ai.provider.name`). Normalise at the gateway to a pinned target version.

- **"Migrating to OTel means removing the old stack."** *Symptom:* a cutover weekend that produces a telemetry gap during the incident it caused. *Mechanism:* every production migration in the references ran a dual-export parity period. Structure each phase so it changes either the data path or the destination, never both.

- **"`k8sattributes` is free enrichment."** *Symptom:* API-server latency climbing after an agent rollout. *Mechanism:* the processor runs Kubernetes informers. Without `filter.node_from_env_var` every agent watches every Pod, so watch load is nodes × pods. With the default 5-minute `watch_sync_period`, every agent also re-enqueues its whole cache periodically — a synchronised CPU and GC spike across the fleet.

- **"Turn on `resource_to_telemetry_conversion` so we get pod names on metrics."** *Symptom:* Mimir series count multiplies overnight. *Mechanism:* the flag promotes *every* resource attribute to a metric label, including `k8s.pod.name` and `k8s.pod.uid`. It is lesson 1's cardinality bomb behind a convenience toggle. If you need pod-level metric attribution, do it with a bounded info metric and a query-time join.

- **"Alert on `otelcol_processor_dropped_spans`."** *Symptom:* an alert that has never fired, on a Collector that has been dropping data for months. *Mechanism:* that metric does not exist in the current Collector. Drops are attributed per component: `otelcol_processor_memory_limiter_refused_spans`, `otelcol_receiver_refused_spans`, `otelcol_exporter_enqueue_failed_spans`, `otelcol_exporter_send_failed_spans`.

- **"`decision_wait` should match p99 latency."** *Symptom:* long traces decided on partial data, and `sampling_trace_dropped_too_early` climbing. *Mechanism:* a trace ends when its *last* span ends, which for anything with an async tail is well after the user-visible response. Set `decision_wait` against p99 trace duration and verify with `otelcol_processor_tail_sampling_sampling_late_span_age`.

## Self-check

**Why keep the SDK thin and push sampling, redaction and routing into the Collector?**
Because of a deployment-cost asymmetry. Anything baked into the SDK requires a PR, a release and a rollout per service to change — N services, coordinated across teams, on a quarterly timescale. The same decision in a Collector config is a ConfigMap change and a rolling restart, on a same-day timescale. Redaction in particular changes when legal asks, which is a cadence no application release train can match. The one thing that *must* stay in the SDK is context propagation, because it requires being inside the process and no downstream component can retrofit a missing `traceparent` header.

**Where must `tail_sampling` run, and why can't it run on the agent DaemonSet?**
On the gateway tier, behind trace-ID-keyed routing. A tail-sampling policy is a predicate over the *whole* trace ("keep if any span has ERROR"), and an agent sees only the spans emitted on its own node while a distributed trace's spans are scattered across many nodes. Evaluating the predicate against a fragment silently answers a different question. The same bug appears in a subtler form on a gateway fleet behind a plain Service, because connection-level load balancing splits a trace's batches across replicas — hence the `loadbalancing` exporter with `routing_key: traceID` and a headless-Service resolver.

**Why is `memory_limiter` ordered first, mechanically?**
It polls memory every `check_interval` against a hard limit (`limit_mib`) and a soft limit (`limit_mib − spike_limit_mib`, with `spike_limit_mib` defaulting to 20%). Above the soft limit it returns a **non-permanent** error to whatever called it; above the hard limit it also forces a GC. A receiver seeing a non-permanent error retries and withholds its acknowledgement, propagating backpressure to the sender — which is what stops the heap growing. If the limiter sits after `batch`, its caller is `batch`, which is not a receiver, cannot retry upstream, and drops the already-accumulated batch — so you lose data *and* keep OOMing. Also set `GOMEMLIMIT` to ~80% of the hard limit so the Go runtime helps before the processor has to.

**Why does `batch` run last rather than early for throughput?**
Because everything before it removes data. Batching before filtering and sampling packs items that are about to be discarded, wasting CPU and — more dangerously — inflating the heap that `memory_limiter` is defending. Batch compresses only what survived. Note the defaults while you are there: `send_batch_size: 8192` is a *trigger*, not a cap; `send_batch_max_size: 0` means no upper bound, so a single large incoming request can produce a batch that exceeds your backend's max receive size. Set it explicitly when the destination has a limit.

**A gateway fleet is horizontally scaled behind a plain Service and tail-sampling results look wrong with no errors. Cause and fix?**
Spans of individual traces are landing on different gateway replicas, so each replica evaluates the policy on a partial view and makes an independent decision. The kept "traces" are missing whatever lived on the other replicas, and nothing logs a warning because every component did its job correctly on the data it saw. The fix is a routing tier that guarantees trace affinity: the `loadbalancing` exporter with `routing_key: traceID` and a `k8s` resolver against a headless Service — plus `groupbytrace` in front of it if rollouts causing `R/N` route reshuffles are a concern.

**Your Collector is losing data. Which metrics tell you where, and which commonly-cited metric name is wrong?**
Work the pipeline in order: `otelcol_receiver_refused_spans` (backpressure reaching the edge) and `otelcol_receiver_failed_spans` (intake errors); `otelcol_processor_memory_limiter_refused_spans` (the limiter shedding — the clearest under-provisioning signal); `otelcol_exporter_enqueue_failed_spans` (sending queue full, data lost at the exporter) and `otelcol_exporter_send_failed_spans` (backend rejecting); with `otelcol_exporter_queue_size / otelcol_exporter_queue_capacity` as the saturation ratio to graph. `otelcol_processor_dropped_spans` is **not** a current metric — alerts referencing it match nothing.

**How do you absorb semantic-convention churn without redeploying every app?**
Three layers. Know the **stability marker** of every convention you depend on (Development conventions carry no compatibility guarantee; the whole `gen_ai.*` surface is currently Development). Read the **`schema_url`** that producers stamp on every `Resource*` message, which states the version the payload was produced against. Then **normalise in the gateway** with a `transform` processor to one target version — `http.method` → `http.request.method`, `gen_ai.system` → `gen_ai.provider.name` — and write every dashboard and alert against that single version. OTel also publishes machine-readable schema files describing these renames, which a schema translator can apply mechanically. Apps emit whatever their SDK knows; the pipeline reconciles.

**Size a tail-sampling gateway for 20,000 traces/s fleet-wide across 5 replicas.**
Per gateway, 4,000 traces/s. With `decision_wait: 20s`, in-flight traces ≈ 4,000 × 20 = 80,000, so `num_traces` must exceed that with margin — the default of 50,000 would evict traces before their decision and surface as `otelcol_processor_tail_sampling_sampling_trace_dropped_too_early`. Set it to ~200,000. At 12 spans/trace × ~400 B/span, trace data alone is 80,000 × 12 × 400 B ≈ 384 MB, plus decision caches, batch buffers and receiver buffers — so an 8 GiB container with `limit_mib: 6144` and `spike_limit_mib: ~1229`. Set `num_shards: 8` so ingestion and decision evaluation do not contend on one event loop (traces are hashed to shards by trace ID, so affinity holds), and set both decision caches well above `num_traces` so late spans inherit the right decision.

## Connections & what's next

This lesson adds the third cardinality-enforcement gate promised in [01 · The signal model](01-signal-model.md) — the OTLP path has no scrape to relabel, so `transform`/`filter` and the `resource_to_telemetry_conversion` setting are where the budget is defended. It feeds the storage tier designed in [03 · Metrics at scale](03-metrics-at-scale.md), which the Collector can subsume entirely via the `prometheus` receiver and `prometheusremotewrite` exporter, and whose per-tenant limits are the backstop for anything this layer misses. The `X-Scope-OrgID` routing here is the same multi-tenancy boundary sized there.

The sampling *policy* question — what to keep, what it costs, and how to make tracing pay for itself — is the whole subject of the next lesson; here you learned only where sampling can physically happen. Lesson 6 carries the pipeline pattern to logs, where the same Collector fleet competes with Vector and Fluent Bit and the routing key changes for the reasons in §5. Lesson 9 consumes the `gen_ai.*` and `k8s.*` join built in §10 as the raw material for a GPU goodput SLO.

Next: [05 — Distributed tracing](05-distributed-tracing.md).

## References & further reading

**Primary sources — read directly from the repositories**
- OpenTelemetry Collector (`open-telemetry/opentelemetry-collector`, `main`, August 2026), `processor/memorylimiterprocessor/README.md` — soft/hard limits, the 20% `spike_limit_mib` guidance, the non-permanent-error contract, the "data is permanently lost if the preceding component does not retry" warning, the forced-GC back-off, and the `GOMEMLIMIT` recommendation.
- OpenTelemetry Collector, `processor/batchprocessor/README.md` — `send_batch_size: 8192`, `timeout: 200ms`, `send_batch_max_size: 0`, and the metadata-batching options.
- OpenTelemetry Collector, `processor/*/documentation.md`, `receiver/receiverhelper/documentation.md`, `exporter/exporterhelper/documentation.md`, `service/documentation.md` — the complete current list of `otelcol_*` internal metric names used in §7. **This corrects the previous version of this lesson**, which cited `otelcol_processor_dropped_spans`; that metric is not emitted by the current Collector.
- OpenTelemetry Collector, `docs/observability.md` — the `otelcol_` prefix convention and the `metadata.yaml`/mdatagen process that defines internal metrics.
- OpenTelemetry Collector Contrib (`main`, August 2026), `processor/tailsamplingprocessor/README.md` — the full policy list, `decision_wait: 30s`, `num_traces: 50000`, `num_shards` (default 1, max 256), the `trace-complete` vs `span-ingest` strategies, decision caches, `maximum_trace_size_bytes`, and the decision-flow rules.
- OpenTelemetry Collector Contrib, `processor/tailsamplingprocessor/documentation.md` — the `otelcol_processor_tail_sampling_*` metric names in §7.
- OpenTelemetry Collector Contrib, `exporter/loadbalancingexporter/README.md` — the routing-key table and validity matrix, the signal-specific defaults (`traceID` for traces, **`service` for logs and metrics**), the `R/N` rebalancing property, the `static`/`dns`/`k8s`/`aws_cloud_map` resolvers, the EndpointSlice RBAC requirement, and the fact that sub-exporter queueing and retry are **disabled by default**.
- OpenTelemetry Collector Contrib, `processor/k8sattributesprocessor/README.md` — `filter.node_from_env_var`, `watch_sync_period` (default 5 m, `0s` recommended for large clusters), the informer-resync cost at 100k pods, and the per-attribute RBAC requirements.
- OpenTelemetry specification (`open-telemetry/opentelemetry-specification`, `main`), `specification/protocol/exporter.md` — the default OTLP endpoints (4317 gRPC, 4318 HTTP) and the `/v1/traces`, `/v1/metrics`, `/v1/logs` paths.
- OpenTelemetry specification, `specification/schemas/README.md` and `file_format_v1.1.0.md` — schema URLs carried in `Resource*`/`Scope*` messages and the schema-translation mechanism described in §8.
- OpenTelemetry GenAI semantic conventions (`open-telemetry/semantic-conventions-genai`, `main`, August 2026), `docs/gen-ai/gen-ai-metrics.md` and `docs/registry/attributes/gen-ai.md` — the metric list with units and Development stability markers, the `gen_ai.client.token.usage` histogram with its specified bucket boundaries and `gen_ai.token.type` dimension, `gen_ai.provider.name` as the successor to `gen_ai.system`, and the full attribute registry. **Note:** these conventions moved out of `open-telemetry/semantic-conventions` into their own repository; the old paths now contain only redirect stubs.
- [OpenTelemetry Collector documentation](https://opentelemetry.io/docs/collector/) and [Collector configuration](https://opentelemetry.io/docs/collector/configuration/) — the canonical prose versions of the component model.
- [OpenTelemetry semantic conventions](https://opentelemetry.io/docs/concepts/semantic-conventions/) and [document status / stability levels](https://opentelemetry.io/docs/specs/otel/document-status/).

**Real-world engineering write-ups**
- Cloudflare, [Adopting OpenTelemetry for our logging pipeline](https://blog.cloudflare.com/adopting-opentelemetry-for-our-logging-pipeline/)
- Grafana Labs, [How Grafana Labs enables horizontally scalable tail sampling in the OpenTelemetry Collector](https://grafana.com/blog/how-grafana-labs-enables-horizontally-scalable-tail-sampling-in-the-opentelemetry-collector/)
- Dotan Horovits, [Shopify's Journey to Planet-Scale Observability](https://horovits.medium.com/shopifys-journey-to-planet-scale-observability-9c0b299a04dd) — **third-party report of a conference talk, not a primary Shopify source**; the 15–20% agent-overhead figure is as reported there and should be treated as indicative rather than verified.

**Sources consulted but not relied upon.** Several vendor documentation domains are unreachable from this environment's egress proxy. Every configuration key, default value and metric name above was verified against the cloned upstream repositories as itemised; where a page could not be fetched and no repository equivalent existed, the claim was omitted rather than approximated. Component stability markers (alpha/beta/development) change between releases — check the component's own README status table before quoting one in a design review.
