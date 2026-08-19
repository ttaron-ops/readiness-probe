---
lesson: "A03.6"
title: "Logging pipelines"
module: "A-03"
concept: "cost/value & label cardinality"
status: not-started
est_time: "4 hrs"
prev: "05-distributed-tracing.md"
next: "07-slos-and-alerting.md"
artifacts: ["loki-label-schema", "logql-straggler-query", "loki-vs-elk-decision-memo"]
sources: 13
---

# A03.6 · Logging pipelines

> **Concept.** Logs are the most-expensive-per-value signal — a last resort; the Loki-vs-ELK fork and label cardinality decide whether the pipeline survives at fleet scale.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 05 built the sampling-economics lens: decide what to keep by outcome, and pay a cost proportional to what you must buffer to decide. This lesson carries that lens to logs, where the equivalent decision is not "which trace do I keep" but **"which fields do I even index"** — and where the cost curve is driven by cardinality rather than buffering.

Logs sit at the expensive end of the signal-cost gradient this module has walked since lesson 01 (metrics → traces → logs). This is the lesson that makes that cost concrete enough to defend in a design review, and it closes the loop on lesson 01's claim that cardinality is a property of *indexes*, not of Prometheus — because you are about to watch the same bomb detonate in two completely different buildings.

Everything below is checked against **Grafana Loki** (`main`, August 2026 — `docs/sources/shared/configuration.md`, `docs/sources/get-started/labels/`, `docs/sources/operations/bloom-filters.md`), **Vector 0.58.0** (`website/content/en/docs/architecture/`), and **Fluent Bit 5.1.1** (`CHUNKS.md`, `include/fluent-bit/flb_storage.h`, `include/fluent-bit/flb_input_chunk.h`). Defaults are quoted with their config key so you can check them against your own version.

## Why this matters

Logs are where observability budgets go to die. They are the highest cost-per-unit-value signal you run, and at fleet scale a single bad indexing decision — one high-cardinality field promoted to an index dimension — takes down the ingest path for everyone, not just for the team that made the mistake.

The staff move has two halves. First, **demote logs to a last resort**: anything a metric or an exemplar-linked trace can answer should never be a log. Second, make the two architectural decisions that determine whether the pipeline is affordable at all: the Loki-vs-ELK fork, and where you draw the indexing line.

On a GPU fleet the numbers are brutal. Per-rank, per-step logging across 4,000 GPUs is a firehose, and NCCL debug logging on a large synchronous job produces *N* near-identical lines for every collective — one per rank, within milliseconds of each other. A naive pipeline pays 4,000× to store what is informationally one event, and it will bankrupt a log budget in days rather than months.

There is also a correctness stake that is easy to miss. When the log pipeline saturates, you do not merely lose logs — you lose them *selectively*, because backpressure propagates unevenly and the noisiest sources win. The logs you lose in an incident are disproportionately the ones from the component that is failing, because that is the component generating the burst.

## What's new here (calibration)

- **Skip (you already know):** what structured logging is; the ship → parse → index → store pipeline; that Loki is cheaper than Elasticsearch.
- **New:** Loki's storage model at the level of *chunks* — `chunk_target_size`, `chunk_idle_period`, `max_chunk_age`, `chunk_block_size` — because that is what turns a cardinality mistake into an ingester memory problem, and it explains why a *low-volume* stream can be as expensive as a high-volume one.
- **New: structured metadata**, the third tier that did not exist in earlier Loki. It sits between labels and the log line: high-cardinality, not indexed, but attached to the line without being embedded in it — and it is what bloom filters accelerate. This changes the label-schema advice materially.
- **New:** the actual Loki limits with their defaults — `max_global_streams_per_user: 5000`, `max_label_names_per_series: 15`, `per_stream_rate_limit: 3MB`, `ingestion_rate_mb: 4` — so "size the stream limit" becomes arithmetic instead of a gesture.
- **New:** the LogQL execution model as a cost model — label matcher → chunk selection → line filter on raw bytes → parser → label filter — and where each stage's cost lives.
- **New:** the collection tier as an engineering choice. Vector's disk buffers (write-ahead-log semantics, ~256 MiB minimum, 128 MiB files, 500 ms fsync, **forced process exit on I/O error**) versus Fluent Bit's chunk model (2 MB filesystem chunks, `storage.max_chunks_up: 128`, `storage.backlog.mem_limit: 100M`). These differences decide what happens to your logs during a backend outage.
- **New:** the failure asymmetry between Loki and Elasticsearch stated precisely — fail-fast rejection versus cluster destabilisation — and why that is a genuine operational argument, not a preference.

## Core concepts

### 1. Logs are a last resort — priced

Lesson 01 gave the ordering; here is the arithmetic that makes it a policy rather than an opinion. One service, 10,000 requests/second, one access-log line per request:

```
   THE SAME FACT, PRICED IN THREE SIGNALS
   ═══════════════════════════════════════════════════════════════════════════
   Question: "what fraction of requests are failing?"

   METRIC   one counter with a bounded `status` label
              4 series × 1 sample / 15 s
              = 23,040 samples/day × ~1.5 B          ≈ 35 KB/day
              query cost: read 4 series               ≈ microseconds

   TRACE    1 span/request, tail-sampled at 3.3 % (lesson 5)
              10,000/s × 86,400 × 0.033 × ~350 B      ≈ 10 GB/day
              query cost: TraceQL scan over Parquet    ≈ seconds

   LOG      1 line/request, ~300 B, structured JSON ~450 B
              10,000/s × 86,400 × 450 B               ≈ 389 GB/day raw
              at 8× compression                       ≈  49 GB/day stored
              query cost: scan all selected chunks     ≈ tens of seconds

   ── RATIO: log storage is ~1.4 MILLION times the metric's, for an
      answer the metric gives exactly and instantly.
```

**The policy that falls out:** every log line must justify itself against the question "could a metric or a trace answer this?" Logs earn their place only for **high-cardinality, per-event forensic detail you genuinely need after the fact** — the stack trace, the request body that broke the parser, the exact sequence of retries. Everything else is a metric with a bounded label, or a span attribute on a sampled trace.

This is not asceticism. It is what makes the logs you *do* keep affordable enough to keep for long enough to be useful. A team that logs everything at 7-day retention has strictly less forensic capability than a team that logs selectively at 90-day retention, and pays more for it.

### 2. Loki's storage model — what a "stream" physically is

Loki is Prometheus's cost model applied to logs. Recognising the isomorphism means the discipline from lesson 01 transfers almost line for line, but the *mechanics* differ enough that the failure modes are distinct.

```
   LOKI INGEST — FROM A LOG LINE TO OBJECT STORAGE
   ═══════════════════════════════════════════════════════════════════════════

   promtail / Alloy / OTel Collector
        │  push: {stream labels} + [ (ts, line, structured_metadata) … ]
        ▼
   ┌──────────────┐  validates: max_label_names_per_series (15),
   │ DISTRIBUTOR  │  max_label_name_length (1024), max_label_value_length (2048),
   │              │  max_line_size (256 KB), per-tenant ingestion_rate_mb (4)
   │              │  hashes the LABEL SET → picks ingesters from the ring
   └──────┬───────┘
          │  replication factor (typically 3)
          ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │ INGESTER                                                              │
   │                                                                       │
   │  ONE STREAM = ONE UNIQUE LABEL SET                                    │
   │  ┌──────────────────────────────────────────────────────────────┐    │
   │  │ stream {job="pretrain", node="gpu-0417", rank="3"}            │    │
   │  │                                                               │    │
   │  │  ┌──────── open CHUNK (in RAM, uncompressed head block) ────┐ │    │
   │  │  │ block 0 [compressed]  block 1 [compressed]  HEAD BLOCK   │ │    │
   │  │  │                                             (raw lines)  │ │    │
   │  │  │ chunk_block_size: 262144 B  ← head block cut+compressed  │ │    │
   │  │  │                               when it exceeds this        │ │    │
   │  │  └───────────────────────────────────────────────────────────┘ │    │
   │  │                                                               │    │
   │  │  FLUSH TRIGGERS — whichever fires first:                      │    │
   │  │    chunk_target_size  1572864 B  (1.5 MiB COMPRESSED)         │    │
   │  │    chunk_idle_period  30m        (no writes for 30 minutes)   │    │
   │  │    max_chunk_age      2h         (open too long)              │    │
   │  │  chunk_encoding: gzip (default; lz4/snappy/zstd available)    │    │
   │  └──────────────────────────────────────────────────────────────┘    │
   │                                                                       │
   │  ← EVERY ACTIVE STREAM HOLDS AN OPEN CHUNK IN RAM, REGARDLESS OF      │
   │    HOW LITTLE IT WRITES. This is the whole cardinality problem.       │
   └──────────────────────────────┬───────────────────────────────────────┘
                                  │  flush
                                  ▼
                    ┌──────────────────────────────┐
                    │  OBJECT STORAGE               │
                    │   · chunks/  (compressed logs)│
                    │   · index/   (TSDB-format     │
                    │      label→chunk index only)  │
                    └──────────────────────────────┘

   THE INDEX CONTAINS: label sets, and which chunks hold them, and when.
   THE INDEX DOES NOT CONTAIN: anything about the CONTENT of the lines.
```

**The consequence people miss.** An active stream holds an open chunk in ingester memory until one of the three flush triggers fires. A stream that writes one line an hour holds that chunk for up to `chunk_idle_period` (30 minutes) or `max_chunk_age` (2 hours) — the same as a stream writing thousands of lines a second. **Ingester memory scales with stream count, not with log volume.** A schema producing 200,000 sparse streams is far more expensive than one producing 4,000 busy ones carrying the same bytes.

It also destroys compression. A chunk targets 1.5 MiB *compressed*; a sparse stream flushes on idle or age with a chunk containing a handful of lines. You get thousands of tiny objects in the bucket, each with its own index entry, each requiring a separate GET at query time. Loki's own cardinality documentation names this directly: high cardinality makes Loki "build a huge index, and flush thousands of tiny chunks to the object store."

### 3. The three tiers: label, structured metadata, log line

This is the part that has genuinely changed, and any label-schema advice that predates it is incomplete.

| Tier | Indexed? | Cardinality tolerance | Query mechanism | Cost |
|---|---|---|---|---|
| **Stream label** | yes — defines the stream | **bounded**, ≤15 labels (`max_label_names_per_series`) | `{job="x", node="y"}` selector | one stream + one open chunk per unique combination |
| **Structured metadata** | no index, but stored per-line separately from the body | **high, deliberately** | `| traceID="abc"` filter; **bloom-accelerated** | counts against ingestion rate; `max_structured_metadata_size` 64 KB/line, `max_structured_metadata_entries_count` 128/line |
| **Log line body** | no | unbounded | `|= "text"` then `| json` then field filter | storage only; parse cost at query time |

**Structured metadata** (chunk format V4, schema version ≥ 13, enabled with `limits_config.allow_structured_metadata: true`) exists precisely for the fields that are too high-cardinality to be labels but that you filter on constantly and do not want to re-parse out of the body every query. Loki's documentation names the intended cases: OpenTelemetry-format ingestion (it was designed for OTel's attribute model), Kubernetes pod names, process and thread IDs, container IDs, customer and trace IDs.

Two hard limits are worth memorising because they produce a confusing 400: a log line's structured metadata may not exceed **64 KB total** or **128 entries**. Exceeding either gets the line rejected by the distributor with HTTP 400, counted in `loki_discarded_samples_total` with reason `structured_metadata_too_large` or `structured_metadata_too_many`. The classic trigger is an OTel log record carrying `exception.stacktrace` as an attribute — a big Java stack trace blows the 64 KB budget and the line vanishes with a status code most people never look at.

**Bloom filters** (experimental; intended for tenants ingesting **>75 TB/month**) build per-chunk bloom filters over structured-metadata key-value pairs, letting the query path skip chunks that statistically cannot contain the value. That is what makes `{cluster="prod"} | traceID="3c0e3dcd33e7"` viable across a day of logs without scanning every chunk. Two operational constraints from the docs: blooms are **not supported in single-binary mode** (they need the Bloom Planner/Builder and Bloom Gateway components), and they only accelerate **structured metadata**, not the log body. If you want trace-ID lookup to be fast, `traceID` must be structured metadata — putting it in the JSON body means no bloom, and putting it in a label means stream explosion.

**So the decision procedure for any field is now three-way, not two-way:**

```
   WHERE DOES THIS FIELD GO?
   ═══════════════════════════════════════════════════════════════════════
                      ┌─────────────────────────────┐
                      │ Do I SELECT on it, i.e. is  │
                      │ it how I scope a query?     │
                      └───────────┬─────────────────┘
                       yes        │        no
              ┌───────────────────┘        └──────────────┐
              ▼                                           ▼
   ┌──────────────────────────┐            ┌──────────────────────────────┐
   │ Is it bounded (<~100     │            │ Do I FILTER on it often, and │
   │ values) AND long-lived?  │            │ is it high-cardinality?      │
   └──────┬──────────┬────────┘            └────────┬──────────┬──────────┘
     yes  │          │  no                     yes  │          │  no
          ▼          ▼                              ▼          ▼
    ┌──────────┐  ┌────────────────────┐   ┌────────────────┐ ┌──────────┐
    │  LABEL   │  │ structured metadata │   │   STRUCTURED   │ │ LOG LINE │
    │          │  │ (you were about to  │   │    METADATA    │ │  BODY    │
    │ job,     │  │  make a mistake)    │   │  (bloom-       │ │ step,    │
    │ node,    │  │                     │   │   accelerated) │ │ loss,    │
    │ tenant,  │  │                     │   │  trace_id,     │ │ msg,     │
    │ level    │  │                     │   │  pod, pid,     │ │ payload  │
    └──────────┘  └────────────────────┘   │  container_id  │ └──────────┘
                                            └────────────────┘
```

### 4. Stream-count arithmetic and the limits that catch it

```
   STREAM COUNT FOR A GPU TRAINING FLEET
   ═══════════════════════════════════════════════════════════════════════════

   SCHEMA A — bounded labels only
     labels: {job, node, rank, tenant, level}
       job     concurrent training jobs           =    20
       node    GPU nodes                          =   500  (per job, worst case)
       rank    local ranks per node               =     8
       tenant  namespaces                         =     1  (implied by job)
       level   info|warn|error                    =     3
     streams = 20 × 500 × 8 × 1 × 3               = 240,000

     ⚠ ALREADY 48× the default max_global_streams_per_user (5000).
     ⚠ AND: at ~1 KB of open-chunk overhead per stream, 240 MB of ingester
       RAM before a single line is stored. Multiply by replication factor 3.

   SCHEMA A′ — drop `level` from labels, keep it in the line
     streams = 20 × 500 × 8                       =  80,000     (3× better)

   SCHEMA A″ — drop `rank` too; put it in structured metadata
     streams = 20 × 500                           =  10,000     (24× better)
     Cost accepted: filtering by rank is now `| rank="3"` rather than a
     label selector — bloom-accelerated if blooms are on, a scan if not.

   SCHEMA B — the bomb: trace_id promoted to a label
     streams = 10,000 × (distinct trace_ids/day ≈ 10^6)
             = 10,000,000,000 stream-days
     Ingest rejects at the stream limit almost immediately. LOUDLY.
```

**The limits that fire, with their real defaults** (`docs/sources/shared/configuration.md`):

| Limit | Default | Effect |
|---|---:|---|
| `max_global_streams_per_user` | **5000** | across the cluster; each ingester gets a dynamic local share based on RF and healthy-ingester count |
| `max_streams_per_user` | 0 (disabled) | per-ingester hard cap, if you prefer local enforcement |
| `max_label_names_per_series` | **15** | a hard stop on label-schema sprawl |
| `max_label_name_length` | 1024 | |
| `max_label_value_length` | 2048 | |
| `per_stream_rate_limit` | **3MB** /s | per *stream*; a single hot stream is throttled independently |
| `per_stream_rate_limit_burst` | **15MB** | |
| `ingestion_rate_mb` | **4** MB/s | per tenant |
| `ingestion_burst_size_mb` | **6** MB | per tenant |
| `max_line_size` | **256KB** | with `max_line_size_truncate: false` — oversize lines are **rejected**, not truncated, unless you opt in |
| `max_chunks_per_query` | 2000000 | read-path protection |
| `max_query_series` | 500 | for metric queries over logs |

Note `ingestion_rate_mb: 4` — 4 MB/s per tenant is nothing at fleet scale. This is the same shape of trap as Mimir's `ingestion_rate: 10000` in lesson 03: a conservative default that must be raised deliberately, and if you do not, the failure is at the distributor.

**`per_stream_rate_limit` is subtler and worth pausing on.** It is per stream, so it does not protect the tenant — it protects against one pathological stream monopolising an ingester. A job that logs a tight loop at 50 MB/s on one stream gets throttled to 3 MB/s and the rest is dropped, while every other stream is unaffected. That is the desired behaviour, but it means "we are losing logs" can be true for exactly one component while the tenant-level ingestion rate looks fine. Alert on `loki_discarded_samples_total` by `reason`, not on the tenant aggregate.

### 5. The same bomb, two buildings — and the failure asymmetry

Elasticsearch dies of the same disease with a completely different clinical picture.

```
   CARDINALITY FAILURE — LOKI vs ELASTICSEARCH
   ═══════════════════════════════════════════════════════════════════════════

   ┌─────────────────── LOKI ────────────────┐ ┌──────── ELASTICSEARCH ──────┐
   │ unbounded value in a LABEL              │ │ unbounded KEY in a JSON doc │
   │            │                            │ │            │                │
   │            ▼                            │ │            ▼                │
   │ stream count grows                      │ │ DYNAMIC MAPPING creates a   │
   │            │                            │ │ new indexed FIELD per key   │
   │            ▼                            │ │            │                │
   │ max_global_streams_per_user reached     │ │            ▼                │
   │            │                            │ │ cluster state (the mapping) │
   │            ▼                            │ │ grows; every node holds it; │
   │ ✅ WRITES REJECTED, HTTP 429            │ │ mapping updates are a       │
   │    "maximum active stream limit         │ │ MASTER-serialised operation │
   │     exceeded"                            │ │            │                │
   │    loki_discarded_samples_total++       │ │            ▼                │
   │                                          │ │ ❌ master node saturates,   │
   │ ⇒ LOUD. You know within a minute.        │ │    cluster state publishing │
   │ ⇒ Roll back the label, streams age out.  │ │    stalls, indexing blocks, │
   │ ⇒ Blast radius: ONE TENANT.              │ │    heap pressure, cascading │
   │                                          │ │    node failure             │
   │                                          │ │ ⇒ QUIET, then catastrophic. │
   │                                          │ │ ⇒ Recovery = reindex.       │
   │                                          │ │ ⇒ Blast radius: THE CLUSTER.│
   └──────────────────────────────────────────┘ └─────────────────────────────┘
```

**This asymmetry is a real operational argument**, not a preference. Loki's failure is a rejected write with a named reason and a bounded blast radius; Elasticsearch's is a slow cluster-state degradation whose root cause is several hops from the symptom. If you run Elasticsearch, the mitigations are:

```json
PUT /logs-app-000001
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "@timestamp":  { "type": "date" },
      "service":     { "type": "keyword" },
      "level":       { "type": "keyword" },
      "message":     { "type": "text" },
      "trace_id":    { "type": "keyword" },
      "attributes":  { "type": "flattened" }
    }
  },
  "settings": {
    "index.mapping.total_fields.limit": 1000,
    "index.mapping.depth.limit": 5,
    "index.mapping.nested_fields.limit": 50
  }
}
```

Three notes on that, because each is a trap:

- **`"dynamic": "strict"` rejects documents** containing unmapped fields, rather than silently dropping the field (`"dynamic": false`) or indexing it (`"dynamic": true`, the default). Strict is what you want in production, and it means you have converted a silent explosion into a loud rejection — the same trade Loki makes by default.
- **`total_fields.limit` defaults to 1000** and is the backstop when you cannot enumerate the schema. Raising it is almost always the wrong instinct.
- **The `flattened` field type** is Elasticsearch's structured-metadata equivalent: an entire JSON object indexed as a single field, with keys and values searchable but no per-key mapping created. It is the correct home for genuinely unpredictable attribute bags, and knowing it exists is the difference between "we had to turn dynamic mapping off and lost searchability" and "we scoped the unpredictable part."

**So when do you actually choose Elasticsearch?** When ad-hoc full-text search over fields nobody predicted is the *primary* workload. Security forensics is the canonical case: you need to find an IOC hash that appears in an unexpected header, six weeks after the fact, and no label schema could have anticipated indexing it. A label-indexed system structurally cannot serve that well, because the premise is that you did not know to index it. That is a confident, defensible choice — not a consolation prize for teams that could not manage Loki's cardinality discipline.

Choose Loki when logs are mostly label-scoped tail-and-filter, when you already run Prometheus and want one label model, and when cost is the binding constraint. That is the common platform case.

### 6. LogQL as a cost model

A LogQL query is a pipeline of stages with wildly different costs. Knowing the order is knowing how to make a query fast.

```
   LOGQL EXECUTION — WHERE THE TIME GOES
   ═══════════════════════════════════════════════════════════════════════════

   {job="pretrain", node=~"gpu-04.*"} |= "nccl" | json | duration_ms > 1300
   └──────────────┬────────────────┘ └───┬───┘ └──┬─┘ └────────┬──────────┘
                  │                      │        │            │
   ① STREAM SELECTOR                     │        │            │
      Index lookup. Resolves label        │        │            │
      matchers → the set of CHUNKS to     │        │            │
      fetch. THE ONLY STAGE THAT USES     │        │            │
      AN INDEX. Cost ∝ number of streams  │        │            │
      matched. A regex here (node=~) is   │        │            │
      evaluated against the label index,  │        │            │
      not the data — cheap.               │        │            │
                                          │        │            │
   ② LINE FILTER  |= "nccl"  ─────────────┘        │            │
      Runs on RAW COMPRESSED-THEN-DECOMPRESSED     │            │
      BYTES. No parsing. Substring match.          │            │
      ~GB/s per core. Cost ∝ bytes in the          │            │
      selected chunks. PUT THIS FIRST.             │            │
                                                   │            │
   ③ PARSER  | json  ────────────────────────────-─┘            │
      Full JSON decode of every SURVIVING line.                 │
      1–2 orders of magnitude more expensive per                │
      byte than ②. Cost ∝ lines that passed ②.                  │
                                                                │
   ④ LABEL FILTER  | duration_ms > 1300  ──────────────────────-┘
      Typed comparison on extracted fields. Cheap
      per line, but only reachable after ③.

   ── THE OPTIMISATION IS ORDERING ──────────────────────────────────────────
   {job="pretrain"} | json | msg=~"nccl.*"           ← parses EVERY line
   {job="pretrain"} |= "nccl" | json | msg=~"nccl.*" ← parses ~0.1 % of them

   Same result. Often 100–1000× difference in query time and in the
   `max_chunks_per_query` / bytes-scanned budget you consume.
```

**Bloom filters change stage ① for structured metadata only.** With blooms enabled, `{cluster="prod"} | traceID="3c0e"` can skip chunks that provably do not contain that key-value pair, converting a full scan of the selected streams into a scan of a handful of chunks. Without blooms, the same query fetches every chunk for `{cluster="prod"}` over the range and iterates. This is why "which fields are structured metadata" is a *query-performance* decision as much as a storage one.

**Metric queries over logs** (`sum(rate({job="x"} |= "error" [5m]))`) run the same pipeline and then aggregate, so all the same ordering rules apply — plus `max_query_series: 500`, which caps the output cardinality. A log-derived metric is a fine thing; a log-derived metric that is *recomputed on every dashboard refresh* over a week of chunks is not. If you find yourself running the same log-derived metric constantly, that is the signal to move it into a Loki recording rule (the ruler writes results to Prometheus/Mimir), which is the log-world equivalent of lesson 02's recording rules.

### 7. The collection tier: Vector vs Fluent Bit vs the OTel Collector

The agent that reads your logs is where backpressure is either absorbed or converted into loss. The three realistic choices behave differently and the differences are concrete.

| | **Vector 0.58** | **Fluent Bit 5.1** | **OTel Collector** |
|---|---|---|---|
| Language / footprint | Rust; larger binary, higher throughput per core | C; very small footprint, designed for constrained edges | Go; moderate |
| Transform language | **VRL** — a purpose-built, compiled, type-checked expression language | Lua scripts or built-in filters | **OTTL** |
| Buffering | in-memory (default 500 events at sinks) or **disk** | in-memory or **filesystem chunks** | in-memory queue, optional `file_storage` extension |
| Disk-buffer minimum | **~256 MiB**, files up to **128 MiB**, fsync every **500 ms** | chunk max **2 MB** (`FLB_INPUT_CHUNK_FS_MAX_SIZE = 2048000`) | queue-size in items |
| Backpressure knob | buffer `when_full: block` \| `drop_newest` | `Mem_Buf_Limit`, `storage.max_chunks_up` (**128**), `storage.backlog.mem_limit` (**100M**) | `memory_limiter` (lesson 4) |
| Delivery guarantee | end-to-end acknowledgements | at-least-once with filesystem storage | depends on exporter queue |
| Adaptive concurrency | **ARC** — TCP-congestion-control-style feedback loop against downstream latency and errors | fixed retry/backoff | exporter-level retry |
| Behaviour on disk I/O error | **forcefully stops the process** | continues; chunk marked | depends |

**Vector's disk buffers are a write-ahead log**, and the details matter. Every event is written to the data files before being read back out; the implementation does not fsync per write but on a **500 ms interval**, trading a bounded loss window for throughput. Buffers have a rigidly-enforced maximum size and a **~256 MiB minimum**; on-disk they are append-only files up to **128 MiB** each, deleted once fully processed, with per-event checksums and automatic partial recovery on corruption.

The operational sharp edge is stated bluntly in Vector's own documentation: **on an I/O error during flush, Vector forcefully stops itself.** The reasoning is that after an I/O error it cannot know what reached disk, so it cannot honour its durability guarantee, and the only safe recovery is a full buffer reload at startup. The consequence for you is that **free disk space on the buffer volume is a first-class alert.** Vector will refuse to start if the configured buffers could exceed the volume, but it cannot detect another process eating the space at runtime.

**Fluent Bit's model is chunk-based.** Records are MessagePack-encoded and grouped by Tag into chunks; a filesystem chunk maxes at **2 MB**. `storage.max_chunks_up` (default **128**) bounds how many chunks are held in memory simultaneously — so with the default you have roughly 256 MB of in-memory chunk residency before the rest stays on disk. `storage.backlog.mem_limit` (default **100M**) bounds how much of the on-disk backlog is loaded back into memory at once during recovery, which is what stops a restart after a long outage from OOMing immediately. `Mem_Buf_Limit` on an input caps that input's in-memory chunk size and, when reached, **pauses the input** — for a tail input that means it stops reading the file, which is graceful if the file persists and lossy if the container is deleted first.

**The choice, honestly:** Fluent Bit when footprint on thousands of nodes is the binding constraint and the transformations are simple. Vector when you need real transformation logic (VRL is genuinely better than Lua for this), strong durability, and ARC's automatic backpressure adaptation. The OTel Collector when you have already deployed it for traces and metrics and the operational win of one agent outweighs Vector's throughput and VRL's ergonomics — which, per lesson 04, is often the right call.

### 8. Cost-control levers, in the order you should apply them

Ordered by return on effort. Each is a lever with a number attached.

**(1) Do not emit it.** The cheapest log is the one the application never writes. Audit `DEBUG` in production; audit per-iteration logging in loops; audit success-path logging where a counter would do. This is unglamorous and routinely halves volume.

**(2) Drop at the agent, before the network.** A line dropped at the DaemonSet costs zero network, zero ingest, zero storage. In Vector:

```toml
[transforms.drop_noise]
type = "filter"
inputs = ["kubernetes_logs"]
condition = '''
  # VRL: keep everything that is not health-check noise.
  !(
    .kubernetes.container_name == "istio-proxy" &&
    contains(string!(.message), "/healthz")
  )
'''

[transforms.sample_info]
type = "sample"
inputs = ["drop_noise"]
rate = 20                      # keep 1 in 20 …
exclude = '.level == "error" || .level == "warn"'   # … but never sample these
```

**(3) Sample by level, never uniformly.** 100% of `ERROR` and `WARN`; 1-in-N of `INFO`; drop `DEBUG` outright unless a flag is set. This is exactly lesson 05's outcome-based sampling in a different signal, and it has the same justification: uniform sampling throws away the lines you will want.

**(4) Redact and trim fields at the agent.** Remove `authorization` headers, cookies, and full request bodies before they leave the node. This is a compliance lever as much as a cost lever, and it belongs at the edge for the same reason it belongs in the Collector for traces (lesson 04): it changes on a legal timescale, not a release timescale.

**(5) Route by value to different retention tiers.** Audit logs to cold object storage at 7-year retention; application `INFO` to Loki at 14 days; `DEBUG` to Loki at 24 hours. Loki supports per-stream retention:

```yaml
limits_config:
  retention_period: 336h            # 14 days default
  retention_stream:
    - selector: '{level="debug"}'
      priority: 1
      period: 24h
    - selector: '{log_type="audit"}'
      priority: 2
      period: 8760h                 # 1 year
```

**(6) Promote the signal to a metric and delete the log.** The highest-leverage lever and the one people reach for last. If you are logging a number in order to graph it later, emit a metric. §9's NCCL case is the canonical instance.

**(7) Only then, tune compression and chunk sizing.** `chunk_encoding: zstd` or `lz4` instead of the default `gzip` trades CPU for ratio; raising `chunk_target_size` produces fewer, larger objects. These are single-digit-percent levers. Do not start here.

### 9. GPU-fleet tie: the NCCL duplication problem

Training and inference logs on a GPU fleet are firehoses, and one of them has a structure that makes ordinary sampling the wrong tool.

**The label schema.** `job`, `node`, `tenant` as labels; `rank` as structured metadata if you have blooms and as a label only if rank count is small and you genuinely select on it; `step`, `loss`, `lr`, `throughput` in the log line; `trace_id` and `pod` as structured metadata.

**The duplication problem, sized.** `NCCL_DEBUG=INFO` on a 500-node, 8-rank-per-node synchronous job:

```
   NCCL DEBUG LOGGING — WHY SAMPLING IS THE WRONG TOOL
   ═══════════════════════════════════════════════════════════════════════════
   ranks                          500 × 8                = 4,000
   collectives per training step  (allreduce, bcast, …)  ≈ 4
   step time                                              ≈ 1.2 s
   lines per collective per rank                          ≈ 1

   lines/s = 4,000 ranks × 4 collectives / 1.2 s          ≈ 13,333 /s
   at ~350 B/line                                          ≈ 4.7 MB/s
                                                            ≈ 403 GB/day raw
   at 8× compression                                        ≈  50 GB/day stored

   ── AND HERE IS THE PROBLEM ───────────────────────────────────────────────
   Those 4,000 lines per collective are the SAME EVENT. They differ in the
   rank ID and in a timestamp that varies by milliseconds. Informationally
   this is ONE event described 4,000 times.

   Sampling at 1-in-100 gives 40 of the 4,000 — still 40 descriptions of one
   event, and now you have no guarantee that the STRAGGLER's line survived,
   which was the only line that mattered.

   ── THE DURABLE FIX ───────────────────────────────────────────────────────
   Extract the signal as a METRIC at the source:
       nccl_collective_duration_seconds{job,node,rank,op}  (histogram)
   Then the straggler query is a metric query, exact and cheap:

       (
         nccl_collective_duration_seconds{job="pretrain"}
           > 1.3 * quantile(0.5, nccl_collective_duration_seconds{job="pretrain"})
       )

   and NCCL_DEBUG can be left OFF in steady state, turned on for one job
   for ten minutes when you need the detail — 50 GB/day → ~0.
```

**The general principle** the NCCL case teaches: **when N processes log the same event because they participate in the same synchronous operation, the redundancy is structural and sampling cannot remove it.** Sampling reduces volume proportionally while leaving the information content unchanged and destroying the guarantee that the *interesting* instance survived. The fix is to change what is emitted, not how much of it is kept. Look for this shape wherever you have SPMD workloads, leader-follower replication, or fan-out to identical workers.

**The LogQL straggler query, for when you do need the logs.** Steady state should be the metric above; when you turn NCCL debug on for a diagnosis, this is the query:

```logql
{job="llama-70b-pretrain", tenant="research"}
  |= "NCCL"
  |= "AllReduce"
  | json
  | duration_ms > 1300
  | line_format "{{.rank}} {{.duration_ms}}ms {{.op}}"
```

Read the ordering against §6: the stream selector scopes to one job's streams; two line filters run on raw bytes and eliminate the vast majority; only survivors are JSON-parsed; the typed comparison runs on the handful that remain; `line_format` reshapes the output for reading. Written the other way round — `| json` first — the same query parses every line in the range.

## Perspectives

**Data model.** Loki is Prometheus's cost model applied to logs, and structured metadata is the piece that makes the analogy complete: it is the log-world equivalent of the exemplar or the info-metric join — a place to put high-cardinality identity that you can filter on without indexing. Once you see that, lesson 01's discipline transfers wholesale, and the three-way decision in §3 is just lesson 01's "bounded label / recording rule / exemplar" trichotomy in different clothes.

**Compliance and security.** Elasticsearch's inverted index is not "faster search" as a generic property; it is specifically the ability to query fields nobody predicted needing. That is exactly the security-forensics requirement — an IOC hash in an unexpected header, six weeks later — and no label schema can anticipate it. This is a case where the expensive architecture is the correct one, and being able to say so confidently, with the `flattened` field type as the scoping mechanism, is more useful than a reflexive "Loki is cheaper."

**Operational.** The Loki-versus-Elasticsearch failure asymmetry is a first-class selection criterion. A system whose cardinality failure is a named HTTP 429 with a metric attached, bounded to one tenant, is operationally different from one whose cardinality failure is master-node saturation and a cluster-wide reindex. Prefer systems that fail loudly and locally, and when you must run one that does not, buy the loudness explicitly (`"dynamic": "strict"`, field limits, mapping review in CI).

**Cost engineering.** The lever ordering in §8 is the whole discipline: emit less, drop earlier, sample by outcome, redact, route by value, promote to metrics, and only then tune compression. Teams reliably invert this — they start with compression and chunk sizing, gain 8%, and never touch the `DEBUG` logging that is 60% of the volume. The reason is that compression is a config change and log hygiene is a conversation with twelve teams. That conversation is the staff work.

**Reliability.** When the log pipeline saturates you do not lose logs uniformly — you lose them where the backpressure lands, and the noisiest source wins the race to fill the buffer. During an incident the failing component is usually the noisiest, so the logs most likely to be dropped are the ones you need. That is the argument for per-stream rate limits (which localise the loss), for durable agent buffers (which absorb the burst), and for alerting on `loki_discarded_samples_total` *by reason* rather than on an aggregate.

## Real-world use cases

- **Cloudflare, "An overview of Cloudflare's logging pipeline."** One of Cloudflare's largest data pipelines, ingesting millions of log events per second from every edge server. **What it shows:** a genuine fleet-scale logging architecture with the tiering, routing and sampling decisions made explicitly rather than emergently. It is the best available public grounding for §8's lever ordering, and it demonstrates that at sufficient scale the pipeline itself becomes a product with its own SLOs.

- **Cloudflare, "Adopting OpenTelemetry for our logging pipeline."** Cloudflare replaced syslog-ng with the OpenTelemetry Collector for log collection while keeping storage and query as separate concerns. **What it shows:** lesson 04's claim that OTel unifies *collection*, not storage, holds for logs specifically — the Collector is a credible production log agent, and the "one agent for three signals" consolidation is achievable. It also shows the migration was justified on operational consolidation rather than raw performance.

- **A practitioner's account of Loki's stream-cardinality wall** (University of Toronto CS sysadmin blog, "Grafana Loki and what can go wrong with label cardinality"). A first-hand operator report of hitting the limit in a real deployment. **What it shows:** what the failure looks like from the operator's chair rather than from the documentation — which labels seemed reasonable, how quickly the limit arrived, and what the rollback involved. Worth reading precisely because it is a small deployment: the wall is not a hyperscale-only phenomenon.

- **The `exception.stacktrace` rejection pattern.** Not a single postmortem but a recurring OTel-to-Loki integration failure: OpenTelemetry log records map attributes to Loki structured metadata, a Java exception's `exception.stacktrace` attribute exceeds `max_structured_metadata_size` (64 KB), and the distributor rejects the line with HTTP 400. **What it shows:** the lines you lose to a limit are correlated with the lines you most want — the ones carrying big stack traces are the ones describing failures. Always check `loki_discarded_samples_total` by `reason` before concluding that an application "stopped logging errors."

## Worked example

**Design the logging layer for the 4,000-GPU fleet.** 500 GPU nodes × 8 GPUs, 20 concurrent training jobs, 3 tenants, plus an inference fleet.

---

**Step 1 — measure the current volume and classify it.**

```
   CURRENT STATE (measured over 24 h with a sampling agent)
   ═══════════════════════════════════════════════════════════════════════
   source                      lines/s    bytes/s   share   verdict
   ─────────────────────────  ────────   ────────   ─────   ─────────────
   NCCL_DEBUG=INFO             13,333     4.7 MB/s   58 %   → METRIC (§9)
   trainer step logs (INFO)     8,300     3.1 MB/s   38 %   → sample 1:20
   kubelet / containerd           420     0.12 MB/s   1.5%  → keep, drop DEBUG
   istio-proxy access logs      1,900     0.14 MB/s   1.7%  → drop /healthz
   application ERROR/WARN          38     0.02 MB/s   0.2%  → KEEP 100 %
   inference access logs        2,100     0.08 MB/s   1.0%  → sample 1:10
   ─────────────────────────  ────────   ────────   ─────
   TOTAL                       26,091     8.2 MB/s          ≈ 708 GB/day raw
```

**Read that table before doing anything else.** 58% of the volume is one debug flag, and 0.2% of the volume is the errors. The single highest-value action is turning `NCCL_DEBUG` off and emitting a metric — before any schema design, any compression tuning, any retention change.

---

**Step 2 — the label schema, with the stream arithmetic.**

```yaml
# The three tiers, stated explicitly.
#
# LABELS (bounded, selected on, ≤15 of them)
#   cluster       1
#   tenant        3
#   job           20 concurrent
#   node          500
#   log_type      { training | inference | system | audit }   4
#
#   streams = 1 × 3 × 20 × 500 × 4  ... = 120,000  ← TOO MANY
#
#   But job and tenant are functionally dependent (a job belongs to one
#   tenant) and a node runs one job at a time, so the REALISED product is
#   far smaller than the naive one:
#
#   realised streams ≈ nodes × log_type = 500 × 4     = 2,000
#   plus inference fleet (80 nodes × 2 log_types)     =   160
#   plus system/audit on non-GPU nodes (40 × 2)       =    80
#                                                       ───────
#   TOTAL                                              ≈ 2,240 streams
#
#   ✔ under max_global_streams_per_user (5000 default) with 2× headroom
#   ✔ ingester RAM ≈ 2,240 × ~1 KB open chunk × RF 3   ≈ 7 MB. Trivial.
#
# STRUCTURED METADATA (high-cardinality, filtered on, bloom-accelerated)
#   rank, pod, container_id, trace_id, gpu_uuid
#
# LOG LINE BODY (everything else)
#   step, loss, lr, throughput, msg, exception (see the 64 KB limit!)
```

**Why `rank` is structured metadata and not a label.** Adding `rank` (8 per node) as a label multiplies streams by 8 → 17,920, over the default limit and 8× the ingester memory. As structured metadata it costs nothing in stream count, is filterable with `| rank="3"`, and — with blooms on — is bloom-accelerated. The cost accepted: without blooms, `| rank="3"` is a scan over the node's chunks rather than an index lookup. For a query already scoped to one node that is a small scan, so the trade is clearly right.

---

**Step 3 — the tenant limits, raised from defaults deliberately.**

```yaml
# runtime overrides — gpu-infra tenant
overrides:
  gpu-infra:
    # Streams: 2,240 realised + headroom for a job-count spike.
    max_global_streams_per_user: 10000

    # Ingestion: post-mitigation volume is ~1.4 MB/s (see step 4);
    # 4× headroom for a burst, well above the 4 MB/s default.
    ingestion_rate_mb: 24
    ingestion_burst_size_mb: 48

    # One pathological stream must not monopolise an ingester.
    per_stream_rate_limit: 8MB
    per_stream_rate_limit_burst: 32MB

    # Stack traces need room, but not unbounded room.
    max_line_size: 512KB
    max_line_size_truncate: true      # truncate rather than reject — a
                                      # truncated stack trace beats no line

    # Structured metadata for OTel ingestion.
    allow_structured_metadata: true

    # Retention, by value.
    retention_period: 336h            # 14 d default
    retention_stream:
      - selector: '{log_type="audit"}'
        priority: 1
        period: 8760h                 # 1 year
      - selector: '{log_type="system"}'
        priority: 2
        period: 168h                  # 7 days
```

Note `max_line_size_truncate: true`. The default is `false`, meaning oversize lines are **rejected outright**. For a fleet where a Java or Python stack trace can legitimately exceed the limit, a truncated line is strictly more useful than a 400 — and it changes the failure from "the service stopped logging errors" to "the error line ends with a truncation marker."

---

**Step 4 — the agent configuration and the volume it produces.**

```toml
# vector.toml — DaemonSet on every node
[sources.k8s]
type = "kubernetes_logs"

# ── 1. Drop the noise, at the edge, before the network. ──────────────────
[transforms.drop_noise]
type   = "filter"
inputs = ["k8s"]
condition = '''
  msg = string!(.message)
  container = string!(.kubernetes.container_name)
  !(
    (container == "istio-proxy" && (contains(msg, "/healthz") ||
                                    contains(msg, "/readyz"))) ||
    (contains(msg, "level=debug") && !exists(.kubernetes.pod_labels."debug-mode"))
  )
'''

# ── 2. Promote fields into structured metadata, strip them from the body. ─
[transforms.shape]
type   = "remap"
inputs = ["drop_noise"]
source = '''
  # Parse once, at the edge, so Loki never re-parses at query time.
  parsed, err = parse_json(.message)
  if err == null {
    .level     = parsed.level
    .rank      = parsed.rank
    .trace_id  = parsed.trace_id
    .message   = encode_json(del_all_but(parsed, ["msg","step","loss","lr"]))
  }
  # Redaction: policy that changes on a legal timescale, applied at the edge.
  .message = replace(string!(.message), r'"authorization":"[^"]*"',
                                        '"authorization":"REDACTED"')
'''

# ── 3. Sample by OUTCOME, never uniformly. ───────────────────────────────
[transforms.sample]
type    = "sample"
inputs  = ["shape"]
rate    = 20
exclude = '.level == "error" || .level == "warn" || .log_type == "audit"'

[sinks.loki]
type     = "loki"
inputs   = ["sample"]
endpoint = "http://loki-distributor.logging.svc:3100"
tenant_id = "gpu-infra"

  # LABELS — bounded only.
  [sinks.loki.labels]
  cluster  = "prod-gpu"
  tenant   = "{{ kubernetes.pod_namespace }}"
  node     = "{{ kubernetes.pod_node_name }}"
  log_type = "{{ log_type }}"

  # STRUCTURED METADATA — high cardinality, filterable, bloom-accelerated.
  [sinks.loki.structured_metadata]
  rank         = "{{ rank }}"
  pod          = "{{ kubernetes.pod_name }}"
  container_id = "{{ kubernetes.container_id }}"
  trace_id     = "{{ trace_id }}"

  # DURABILITY — survive a Loki outage instead of dropping.
  [sinks.loki.buffer]
  type      = "disk"
  max_size  = 1073741824        # 1 GiB; minimum is ~256 MiB
  when_full = "block"           # apply backpressure, do not silently drop
```

The resulting volume:

```
   POST-MITIGATION VOLUME
   ═══════════════════════════════════════════════════════════════════════
   NCCL debug        13,333 /s → 0        (metric instead)         −4.70 MB/s
   trainer INFO       8,300 /s → 415 /s   (1:20 sample)            −2.95 MB/s
   istio /healthz     1,900 /s → 0        (dropped at agent)       −0.14 MB/s
   inference access   2,100 /s → 210 /s   (1:10 sample)            −0.07 MB/s
   errors/warns          38 /s → 38 /s    (100 % kept)              ±0
   system               420 /s → 380 /s   (DEBUG dropped)          −0.01 MB/s
   ────────────────────────────────────────────────────────────────────────
   before  26,091 lines/s   8.2 MB/s   ≈ 708 GB/day raw
   after    1,043 lines/s   0.33 MB/s  ≈  28 GB/day raw
                                        ≈ 3.5 GB/day stored @ 8× gzip

   REDUCTION: 25×.  And the ERROR lines — the ones you actually open —
   are kept at 100 %, so forensic capability went UP, not down.

   14-day retention: 3.5 GB/day × 14 ≈ 49 GB in object storage
   at ~$0.021/GB-month                ≈ $1/month for the hot tier
   ⇒ storage is not the bill. The INGESTERS and the agent CPU are.
```

**That last line is the recurring lesson of this module.** As with metrics (lesson 03) and traces (lesson 05), retention is cheap and the always-on tier is expensive. Optimise the always-on tier.

---

**Step 5 — the alerts that tell you the pipeline is lying to you.**

```yaml
groups:
  - name: logging-pipeline
    rules:
      # Loki is rejecting data — and WHY. Never aggregate away `reason`.
      - alert: LokiDiscardingSamples
        expr: sum by (tenant, reason) (rate(loki_discarded_samples_total[5m])) > 0
        for: 5m
        labels: { severity: page }
        annotations:
          summary: 'Loki dropping {{ $labels.tenant }} logs, reason={{ $labels.reason }}'
          runbook: |
            reason=stream_limit         → label schema regressed; find the new label
            reason=rate_limited         → raise ingestion_rate_mb or find the hot stream
            reason=per_stream_rate_limit→ ONE stream is hot; find it with
                                          topk(5, sum by (stream)(rate(...)))
            reason=line_too_long        → max_line_size; enable truncation
            reason=structured_metadata_too_large → a stack trace in an attribute

      # Stream count against the budget, before the wall.
      - alert: LokiStreamCountApproachingLimit
        expr: |
          sum by (tenant) (loki_ingester_memory_streams)
            > 0.7 * sum by (tenant) (loki_ingester_limits_max_global_streams_per_user)
        for: 15m
        labels: { severity: ticket }

      # The agent is the last line of defence — is its buffer draining?
      - alert: VectorDiskBufferFilling
        expr: |
          vector_buffer_byte_size / vector_buffer_max_byte_size > 0.7
        for: 10m
        labels: { severity: page }

      # Vector exits on disk I/O error. Free space is a first-class alert.
      - alert: VectorBufferVolumeLowSpace
        expr: |
          node_filesystem_avail_bytes{mountpoint="/var/lib/vector"}
            / node_filesystem_size_bytes{mountpoint="/var/lib/vector"} < 0.2
        for: 5m
        labels: { severity: page }
        annotations:
          summary: 'Vector will FORCE-EXIT if it cannot write its disk buffer'
```

## Practice

Feeds the [fleet observability design](../practice/fleet-observability/README.md).

Design the logging layer of the fleet observability system. Deliver three artifacts.

**(a) The Loki label schema** — a table with every field classified as **label**, **structured metadata**, or **log line body**, with a one-line justification each. Then:
1. The stream-count arithmetic for the naive schema and for yours, showing the ratio, and the ingester-memory figure that follows (streams × open-chunk overhead × replication factor).
2. The `trace_id`-as-label counterexample computed explicitly — how many streams that would produce and how quickly the limit fires.
3. Every tenant limit you raise from its default, with the arithmetic that justifies the new value: `max_global_streams_per_user`, `ingestion_rate_mb`, `per_stream_rate_limit`, `max_line_size` and its truncation flag.
4. A note on whether you enable bloom filters, referencing the >75 TB/month guidance and the microservices-mode requirement.

**(b) The LogQL straggler query** — with the stages annotated against §6's cost model, showing which stage each clause runs in and roughly what fraction of lines survives it. Then write the *wrong* version (parser first) and state the expected cost multiplier.

**(c) The Loki-vs-ELK decision memo** — one page. It must contain the workload characterisation (label-scoped filtering vs unpredictable-field forensics), the failure-mode asymmetry stated concretely, and, if you choose Elasticsearch, the mapping strategy (`"dynamic": "strict"`, `total_fields.limit`, and where you use the `flattened` type). If you choose Loki but have a forensic requirement, say how you serve it.

Additionally, write short answers to:

5. **The measurement.** How you would produce the §4 volume table for your own fleet, and what you expect the top-two sources to be.
6. **The promotion list.** Which currently-logged signals you would promote to metrics, with NCCL as the worked case, and why sampling is structurally the wrong tool for that shape of redundancy.
7. **The collection tier.** Vector, Fluent Bit or the OTel Collector, with the deciding property. If Vector, state your disk-buffer sizing and your answer to the force-exit-on-I/O-error behaviour.
8. **The alert set** — including the `loki_discarded_samples_total` alert broken out **by reason**, with a runbook line per reason.

**Acceptance criteria.** Done when (i) every stream count is arithmetic rather than assertion, (ii) each of the three tiers has at least three fields in it and each placement is justified, (iii) the volume reduction is computed end to end with a before/after, and (iv) the decision memo would survive someone arguing the opposite choice.

## Common pitfalls

- **"Structured logging solves the cardinality problem."** *Symptom:* everything is JSON and Loki still falls over. *Mechanism:* structuring makes fields *available*; it says nothing about which fields become stream labels. The two decisions are independent, and structuring alone is necessary but not sufficient. The cardinality decision is the three-way classification in §3.

- **"High-cardinality fields have to go in the log body."** *Symptom:* every trace-ID lookup is a full scan and a JSON parse. *Mechanism:* this advice predates **structured metadata**, which is the correct home for high-cardinality fields you filter on — no index cost, no stream cost, bloom-accelerated. Body is for what you read; structured metadata is for what you filter on; labels are for what you select on.

- **"Loki has no full-text search, so it can't do ad-hoc queries."** *Symptom:* teams reject Loki for a workload it would serve fine. *Mechanism:* `|=` and `|~` do brute-force scans over the label-selected chunks at roughly GB/s per core. It is slow for wide-open queries across many streams, not impossible — the constraint is that scan cost is linear in *selected chunk volume*, so a tight label selector makes ad-hoc search perfectly practical.

- **"Only high-volume streams cost memory."** *Symptom:* ingesters OOM on a fleet with modest log volume. *Mechanism:* every active stream holds an open chunk in ingester RAM until `chunk_target_size` (1.5 MiB compressed), `chunk_idle_period` (30 m) or `max_chunk_age` (2 h) fires. Ten thousand sparse streams cost more memory than a hundred busy ones, and produce thousands of tiny objects in the bucket that make queries slow forever after.

- **"ELK's dynamic mapping is a config default you turn off."** *Symptom:* fields silently missing from search after "fixing" the mapping explosion. *Mechanism:* `"dynamic": false` **drops** unmapped fields silently; `"dynamic": "strict"` **rejects the document**. Turning dynamic mapping off without a bounded explicit mapping behind it converts a loud explosion into a silent data-loss bug. Use `strict`, plus `flattened` for the genuinely unpredictable parts.

- **"Sampling fixes NCCL log volume."** *Symptom:* volume down 100×, and the straggler's line is not in the sample. *Mechanism:* the 4,000 lines per collective are one event described 4,000 times. Sampling reduces volume proportionally without reducing redundancy, and it gives no guarantee that the anomalous rank survived. Promote the timing to a metric and turn the debug flag off.

- **"Errors stopped, so the service is healthy."** *Symptom:* an incident where the error logs simply are not there. *Mechanism:* several limits reject lines *selectively* — `max_line_size` rejects the long ones (stack traces), `max_structured_metadata_size` rejects lines with big attributes (also stack traces), and `per_stream_rate_limit` throttles the hottest stream (usually the failing component). The lines you lose correlate with the lines you need. Check `loki_discarded_samples_total` by `reason` first.

- **"The agent's buffer defaults are fine."** *Symptom:* a 20-minute Loki outage becomes 20 minutes of permanently lost logs. *Mechanism:* Vector's default sink buffer is **in-memory, 500 events** — seconds of coverage. Fluent Bit's default storage is memory. Both need explicit durable configuration (`type = "disk"` / `storage.type filesystem`) to survive a backend outage, and Vector additionally needs free-space monitoring because it **force-exits** on a disk-buffer I/O error.

## Self-check

**Why is `trace_id` catastrophic as a Loki label, acceptable as structured metadata, and mediocre in the log body?**
As a **label** it defines the stream, so every distinct trace mints a new stream — with 10,000 base streams and ~10⁶ traces/day, stream count is effectively unbounded, `max_global_streams_per_user` (default 5000) rejects writes almost immediately, and every stream holds an open chunk in ingester RAM. As **structured metadata** it costs nothing in stream count, is stored per line separately from the body, is filterable with `| trace_id="…"`, and is what bloom filters accelerate — so a needle-in-a-haystack lookup can skip chunks that statistically cannot contain it. In the **body** it works but every query must decode the JSON of every line in range with no acceleration, which is one to two orders of magnitude more expensive per byte than a line filter.

**When do you choose Elasticsearch over Loki, and what makes it a confident choice rather than a fallback?**
When ad-hoc full-text search over fields nobody predicted is the *primary* workload — security forensics, compliance discovery, incident archaeology where you need an IOC hash that turned up in an unexpected header six weeks ago. Loki has no index over line content, so a wide-open search scans every chunk in the label-selected set; the premise of the forensic requirement is that you did not know to index the field, so no label schema could have helped. That is a structural mismatch, not a tuning problem. Scope the cost with `"dynamic": "strict"`, an explicit bounded mapping, `index.mapping.total_fields.limit`, and the `flattened` type for genuinely unpredictable attribute bags.

**Same cardinality failure, two systems: name the mechanism and the failure mode of each, and say which fails better.**
In **Loki**: an unbounded label value multiplies stream count until `max_global_streams_per_user` is reached, at which point writes are rejected with an explicit "maximum active stream limit exceeded" and counted in `loki_discarded_samples_total`. Loud, immediate, bounded to one tenant, and rolled back by removing the label. In **Elasticsearch**: unbounded JSON keys under dynamic mapping create unbounded indexed fields, growing the cluster state that every node holds; mapping updates are serialised through the master, so the master saturates, cluster-state publication stalls, indexing blocks and nodes fail in sequence. Quiet, then catastrophic, cluster-wide, and recovered by reindexing. Loki's failure is strictly better operationally, and that asymmetry is a legitimate selection criterion.

**Why does a low-volume stream cost as much ingester memory as a high-volume one?**
Because memory is held per *stream*, not per byte. Every active stream keeps an open chunk in ingester RAM with an uncompressed head block, and it is only flushed when one of three triggers fires: `chunk_target_size` (1.5 MiB compressed), `chunk_idle_period` (30 minutes with no writes), or `max_chunk_age` (2 hours). A stream writing one line an hour holds its chunk for up to 30 minutes just as a busy stream does. It is also worse for storage: it flushes a nearly-empty chunk, producing thousands of tiny objects in the bucket, each with an index entry and each requiring its own GET at query time.

**Why does placing `|=` before `| json` matter, and by roughly how much?**
LogQL is a staged pipeline with steeply increasing per-byte cost: the stream selector uses the index (cheap, and the only indexed stage); the line filter matches substrings against raw decompressed bytes at roughly GB/s per core; the parser fully decodes JSON for every surviving line at one to two orders of magnitude more cost per byte; the label filter then compares typed fields. Putting the line filter first discards non-matching lines before the expensive parse. On a query where the filter eliminates 99.9% of lines, the difference is commonly 100–1000× in query time and in the bytes-scanned budget.

**Why can't sampling fix the NCCL debug-log problem, and what does the general case look like?**
Because the redundancy is structural rather than volumetric: 4,000 ranks in a synchronous collective each log the same event within milliseconds, so 4,000 lines describe one event. Sampling at 1-in-100 leaves 40 descriptions of that one event and provides no guarantee that the *straggler's* line — the only informative one — survived. The durable fix is to change what is emitted: a `nccl_collective_duration_seconds` histogram makes the straggler query exact and cheap, and lets you turn `NCCL_DEBUG` off entirely in steady state. Generalise the shape: whenever N processes log the same event because they participate in one synchronous operation — SPMD training, leader-follower replication, fan-out to identical workers — sampling is the wrong tool and metric extraction is the right one.

**Your service "stopped logging errors" during an incident. What do you check first, and why is this failure correlated with what you needed?**
Check `sum by (tenant, reason) (rate(loki_discarded_samples_total[5m]))` before believing the application. Several limits reject *selectively* in ways that correlate with failure: `max_line_size` (256 KB default, and `max_line_size_truncate` is `false`, so oversize lines are rejected outright) hits long lines, which are stack traces; `max_structured_metadata_size` (64 KB) and `max_structured_metadata_entries_count` (128) reject OTel records carrying a big `exception.stacktrace` attribute; and `per_stream_rate_limit` (3 MB/s default) throttles the hottest stream, which during an incident is usually the failing component. The lines you lose are disproportionately the lines that describe the failure.

**Compare Vector and Fluent Bit on what happens during a two-hour backend outage.**
**Vector** with `buffer.type = "disk"` writes every event to a write-ahead-log-style buffer before it is read back, fsyncing on a 500 ms interval, in append-only files up to 128 MiB (minimum total buffer ~256 MiB) with per-event checksums and partial recovery on corruption. `when_full = "block"` propagates backpressure through transforms to sources rather than dropping. The critical caveat is that on a disk I/O error during flush Vector **forcefully stops the process**, because after an I/O error it cannot honour its durability guarantee — so free space on the buffer volume is a first-class alert. **Fluent Bit** with `storage.type filesystem` writes 2 MB MessagePack chunks to disk, holds at most `storage.max_chunks_up` (default 128, ≈256 MB) in memory, and bounds recovery memory with `storage.backlog.mem_limit` (default 100M) so a restart after a long outage does not OOM on the backlog. When an input's `Mem_Buf_Limit` is reached the input is **paused** — for a tail input that means it stops reading, which is graceful if the file persists and lossy if the container is deleted first.

## Connections & what's next

This lesson is lesson 01's cardinality constraint reappearing in two more indexes — Loki's stream index and Elasticsearch's field mapping — and structured metadata is the log-world sibling of the exemplar and the info-metric join from lessons 01 and 05. The sampling discipline is lesson 05's outcome-based selection applied to a signal where the "outcome" is the log level. The agent tier here is the same tier lesson 04 configured for traces and metrics, which is why "one Collector fleet for three signals" is a live option rather than a slogan. And the volume arithmetic closes the module's recurring finding: retention is cheap, the always-on tier is expensive, and metric promotion is the highest-leverage lever in every signal.

The NCCL metric extraction in §9 is the direct input to lesson 09's straggler detection, where `nccl_collective_duration_seconds` becomes a fleet-wide goodput signal.

Next: [07 · SLOs and alerting](07-slos-and-alerting.md), which builds multi-window multi-burn-rate alerting from first principles — the layer that decides which of these signals actually pages someone.

## References & further reading

**Primary sources — read directly from the repositories**
- Grafana Loki (`grafana/loki`, `main`, August 2026), `docs/sources/shared/configuration.md` — every default quoted in §4: `max_global_streams_per_user: 5000`, `max_streams_per_user: 0`, `max_label_names_per_series: 15`, `max_label_name_length: 1024`, `max_label_value_length: 2048`, `per_stream_rate_limit: 3MB`, `per_stream_rate_limit_burst: 15MB`, `ingestion_rate_mb: 4`, `ingestion_burst_size_mb: 6`, `max_line_size: 256KB`, `max_line_size_truncate: false`, `max_chunks_per_query: 2000000`, `max_query_series: 500`; and the chunk settings `chunk_idle_period: 30m`, `chunk_block_size: 262144`, `chunk_target_size: 1572864`, `chunk_encoding: gzip`, `max_chunk_age: 2h`.
- Grafana Loki, `docs/sources/get-started/labels/structured-metadata.md` — structured metadata as a third tier, chunk format V4 / schema ≥ 13, `allow_structured_metadata`, the intended use cases (OTel ingestion, pod names, process IDs, container IDs), and the hard limits `max_structured_metadata_size: 64KB` and `max_structured_metadata_entries_count: 128` with the `structured_metadata_too_large` / `structured_metadata_too_many` discard reasons and the `exception.stacktrace` trigger. **This is new material relative to the previous version of this lesson**, which treated the decision as binary (label vs log line).
- Grafana Loki, `docs/sources/get-started/labels/cardinality.md` — the default 15-index-label limit, the combinatorial-labels example, the "huge index and thousands of tiny chunks" failure description, and the `logcli series '{}' --analyze-labels` diagnostic.
- Grafana Loki, `docs/sources/operations/bloom-filters.md` — bloom filters as **experimental**, the >75 TB/month intended scale, the requirement for Bloom Planner/Builder and Bloom Gateway components (unsupported in single-binary mode), and the fact that blooms accelerate **structured-metadata** lookups specifically.
- Vector 0.58.0 (`vectordotdev/vector`), `website/content/en/docs/architecture/buffering-model.md` — the 100-event inter-component buffer, the 500-event default sink buffer, disk buffers as a write-ahead log with a **500 ms** fsync interval, the **~256 MiB** minimum buffer size and **128 MiB** append-only files, per-event checksums with partial recovery, and the explicit statement that Vector **forcefully stops itself** on a disk I/O error during flush, making free storage space an operator-monitored resource.
- Vector 0.58.0, `website/content/en/docs/architecture/arc.md` — Adaptive Request Concurrency as a TCP-congestion-control-inspired feedback loop replacing static rate limits.
- Fluent Bit 5.1.1 (`fluent/fluent-bit`), `CHUNKS.md` — the chunk model, MessagePack encoding, Tag-based grouping, the chunkio on-disk layout with CRC32 and metadata header, and `storage.type filesystem`.
- Fluent Bit 5.1.1, `include/fluent-bit/flb_storage.h` — `FLB_STORAGE_MAX_CHUNKS_UP = 128` and `FLB_STORAGE_BL_MEM_LIMIT = "100M"`; `include/fluent-bit/flb_input_chunk.h` — `FLB_INPUT_CHUNK_FS_MAX_SIZE = 2048000` (2 MB); `src/flb_storage.c` — the `mem_buf_limit` enforcement path.
- [Grafana Loki LogQL documentation](https://grafana.com/docs/loki/latest/query/) — the stream-selector / line-filter / parser / label-filter pipeline used as the cost model in §6.
- [Grafana Loki configuration reference](https://grafana.com/docs/loki/latest/configure/) — the prose version of the limits above.
- [Elasticsearch mapping documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-mapping.html) — `dynamic` modes (`true`/`false`/`strict`), `index.mapping.total_fields.limit`, and the `flattened` field type referenced in §5.

**Real-world engineering write-ups**
- Cloudflare, [An overview of Cloudflare's logging pipeline](https://blog.cloudflare.com/an-overview-of-cloudflares-logging-pipeline/)
- Cloudflare, [Adopting OpenTelemetry for our logging pipeline](https://blog.cloudflare.com/adopting-opentelemetry-for-our-logging-pipeline/)
- Chris Siebenmann (University of Toronto CS), [Grafana Loki and what can go wrong with label cardinality](https://utcc.utoronto.ca/~cks/space/blog/sysadmin/GrafanaLokiCardinalityProblem) — a practitioner's first-hand account of hitting the stream-cardinality wall in a small operated deployment.

**Sources consulted but not relied upon.** Several vendor documentation domains are unreachable from this environment's egress proxy; where a page could not be fetched, the fact was verified against the cloned upstream repository as itemised above, or omitted. The Elasticsearch mapping details in §5 are stated from the product's long-standing documented behaviour and should be confirmed against your cluster's major version before being quoted in a design review — field-limit defaults in particular have moved between versions. Compression ratios (8×) and object-storage pricing (~$0.021/GB-month) in the worked example are representative figures used to make the arithmetic runnable; substitute measured values from your own fleet.
