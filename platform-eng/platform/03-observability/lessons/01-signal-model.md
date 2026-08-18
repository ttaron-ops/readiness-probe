---
lesson: "A03.1"
title: "The signal model"
module: "A-03"
concept: "cardinality-as-constraint"
status: not-started
est_time: "3.5 hrs"
prev: null
next: "02-prometheus-and-promql.md"
artifacts: ["cardinality budget worksheet"]
sources: 12
---

# A03.1 · The signal model

> **Concept.** Cardinality is the master constraint that decides which signal a question belongs to — and on a GPU fleet, the wrong choice is a series-count bomb.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

This is lesson one, so it opens the module's through-line rather than following from anything: **cardinality is the master constraint, and delivered work (goodput) is the master SLI.** Every later lesson is a variation on "which signal, at what cost, answers this question" — lesson 2's PromQL traps are what happens when you query a signal whose semantics you assumed; lesson 3 is what happens when the cardinality budget you set here is exceeded at fleet scale; lesson 6 is the same governance failure transplanted into log-stream labels; lesson 9 turns the label classifications you make here into a concrete DCGM relabel config for 10,000 GPUs.

What this lesson adds over the intuition you already run on daily is the *physical* version of the constraint. Not "high cardinality is bad" but: here is the exact data structure a series occupies, here is what each of its fields costs in bytes, here is why the cost is paid per series rather than amortised, and here is the arithmetic that turns a label set into a RAM number you can put in a design doc.

Everything below is checked against the **Prometheus 3.14.0** tree (`main`, August 2026) — specifically `tsdb/head.go`, `tsdb/head_append.go`, `tsdb/index/postings.go`, `tsdb/exemplar.go`, `model/labels/labels_stringlabels.go`, and `docs/storage.md`. Where a number depends on your label set or workload, that is said explicitly and the formula is given so you can recompute it.

## Why this matters

At senior level you pick metrics vs logs vs traces by habit and it usually works. At staff level the question flips: you own the *budget*, and the budget is denominated in active time series, not gigabytes. A single badly-chosen label on a widely-scraped metric can multiply your TSDB head by two orders of magnitude, push the process past its memory limit, and turn a routine restart into a thirty-minute outage while the write-ahead log replays. The staff engineer is the person who can look at a proposed metric and say "that dimension has 10k values in prod, it does not belong on a label" *before* it ships — and justify it with a number, in a design review, against someone who wants the label.

The failure is asymmetric in a way that makes it worse. Adding a label is a one-line PR that passes tests. Removing it later is a breaking change to every dashboard, every recording rule, and every alert that ever used it, and it does not retroactively free the memory you already spent — the series stay in the head until they age out. **Cardinality decisions are cheap to make and expensive to unmake**, which is precisely the profile of a decision that needs a gate.

On a GPU fleet this is not academic. `gpu_uuid`, `job_id`, `pod`, and `mig_instance` are all naturally high-cardinality, and DCGM will happily emit them as labels. At 10k GPUs times up to 7 MIG slices times per-tenant dimensions, naive labelling is a bomb that goes off the first time a large training job churns pod names. Deciding *here* what is a bounded label versus an exemplar reference is what keeps the fleet's own monitoring from becoming the fleet's biggest tenant — and it is the difference between a monitoring bill that scales with GPUs and one that scales with GPUs × jobs × tenants.

## What's new here (calibration)

- **Skip (you already know):** the four signals exist (metrics/logs/traces + profiles/events); push vs pull collection; the RED and USE method names and when each applies; that high cardinality is bad.
- **New:** the *physical* anatomy of a time series — the `memSeries` struct, the packed label string, the head chunk, the postings lists, the stripe hashmap — and a byte-by-byte derivation of where the "1–3 KB per series" rule of thumb actually comes from, including which term you control.
- **New:** that cardinality is not one budget but **five** — head RAM, WAL replay time, index/postings, query fan-out, and remote-write throughput — that each has a different scaling exponent, and that a fix which helps one can hurt another.
- **New:** cardinality as a *time-varying* property of a label. The realistic incident is a label that *was* bounded becoming unbounded, not a label that was always a bomb. That shape determines what you instrument.
- **New:** the enforcement layer — CI linting, `metric_relabel_configs` as scrape-time backstop, Collector-side attribute limits — written out as real config, because a rule with no enforcement point is a wiki page.
- **New:** exemplars as an actual storage mechanism with a size, a retention behaviour and a failure mode, rather than as a hand-wave for "you can still get to the trace."

## Core concepts

### 1. What a time series physically is

Start with the thing that is being counted. "Cardinality" is a count of *series*, and until you know what a series occupies you cannot reason about its cost.

A Prometheus scrape returns lines like this:

```
DCGM_FI_DEV_SM_ACTIVE{gpu="3",UUID="GPU-1a2b3c4d-...",device="nvidia3",modelName="NVIDIA H100 80GB HBM3",Hostname="gpu-node-0417",namespace="team-vision",pod="train-clip-7b-worker-12",container="trainer"} 0.83
```

The parser turns that into three things: an ordered label set (including `__name__`), a float64 value, and an int64 millisecond timestamp. The label set — *all* of it, `__name__` included — **is** the series identity. Change one character of one label value and you have created a different series with a different lifetime and a different memory allocation.

Here is what the head does with it. This is the structural picture to hold in your head for the rest of the module.

```
   ONE SAMPLE → THE STRUCTURES IT TOUCHES IN THE PROMETHEUS HEAD BLOCK
   ══════════════════════════════════════════════════════════════════════════════

   scrape response line
   DCGM_FI_DEV_SM_ACTIVE{gpu="3",UUID="GPU-1a2b…",pod="train-clip-7b-worker-12",…} 0.83
            │
            │  1. parse → labels.Labels + value + timestamp
            ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  labels.Labels  (build tag: stringlabels — the default since 3.0)    │
   │  ONE flat Go string, length-prefixed name/value pairs, names sorted: │
   │    [8]__name__[21]DCGM_FI_DEV_SM_ACTIVE[3]gpu[1]3[4]UUID[40]GPU-1a…  │
   │  → 1 allocation per series, NOT 2 per label.                          │
   │  → but the bytes are NOT shared between series: every series that     │
   │    carries modelName="NVIDIA H100 80GB HBM3" pays for those 21 bytes. │
   └───────────────────────────────┬──────────────────────────────────────┘
            │  2. xxhash64 of the label string
            ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  stripeSeries — 16384 stripes (DefaultStripeSize = 1<<14)            │
   │    hashes[stripe]  : map[uint64] → list of *memSeries with that hash │
   │    series[stripe]  : map[HeadSeriesRef] → *memSeries                 │
   │  Sharded so 16384 goroutines can append without one global lock.     │
   └───────────────────────────────┬──────────────────────────────────────┘
            │  hit?  ──yes──▶ append to existing series (cheap path)
            │  miss? ──────▶ 3. CREATE A NEW SERIES  ← this is "cardinality"
            ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  memSeries                                                            │
   │    ref            HeadSeriesRef (monotonic uint64)                    │
   │    lset           the packed label string above                       │
   │    headChunks     → memChunk → chunkenc.XORChunk  (being filled)      │
   │    mmappedChunks  []*mmappedChunk (older, on disk, mmap'd back)       │
   │    nextAt         timestamp at which to cut the next chunk            │
   │    app            chunkenc.Appender for the open chunk                │
   │    …plus ~10 more fields (mutex, ooo, lastValue, txs, …)              │
   └───────────────────────────────┬──────────────────────────────────────┘
            │  4. register in the inverted index
            ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  memPostings   map[labelName]map[labelValue][]storage.SeriesRef      │
   │    ""            → ""                    → [ …all series refs… ]     │
   │    "__name__"    → "DCGM_FI_DEV_SM_ACTIVE"→ [ …refs… ]               │
   │    "gpu"         → "3"                    → [ …refs… ]               │
   │    "pod"         → "train-clip-7b-worker-12" → [ this one ref ]  ◀── │
   │    …one entry per (name,value) pair the series carries               │
   │  A selector {gpu="3",pod=~"train.*"} = intersect these lists.        │
   └───────────────────────────────┬──────────────────────────────────────┘
            │  5. durability
            ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  WAL (wal/, 128 MB segments, 32 KB pages)                            │
   │    · a "series" record the first time the label set is seen           │
   │    · a "samples" record for every sample thereafter                   │
   │  Replayed on restart to rebuild EVERYTHING above.                     │
   └──────────────────────────────────────────────────────────────────────┘
```

Two properties of that picture do all the work later:

**The label bytes are paid per series, not once.** With the `stringlabels` implementation (`model/labels/labels_stringlabels.go`, the default build since Prometheus 3.0 — the file is gated `//go:build !slicelabels && !dedupelabels`), a label set is one flat Go string with each name and value preceded by its length: one byte for lengths 0–254, otherwise a 0xFF sentinel plus three little-endian bytes. That was a real win over the 2.x `[]Label` layout, which paid two 16-byte string headers per label. What it did *not* do is deduplicate across series. Two million DCGM series each carrying `modelName="NVIDIA H100 80GB HBM3"` each hold their own copy of those bytes. **Long label values are a per-series tax**, which is why `modelName` and `device` cost real money at fleet scale even though they have only a handful of distinct values.

(The `dedupelabels` build tag exists precisely to fix this by interning label strings in a symbol table, at the cost of a pointer chase on every read. It is not the default; if you build your own Prometheus for a very-high-series deployment it is the knob to know about.)

**The postings index is per (name, value) pair, not per series.** `memPostings` is `map[string]map[string][]storage.SeriesRef` (`tsdb/index/postings.go`). A query like `{namespace="team-vision", gpu="3"}` is answered by intersecting two sorted lists of series refs. That is why selectors are fast — and why a label with a million distinct values creates a million map entries each holding a one-element slice, which is the worst possible shape for a Go map: maximum overhead, minimum payload.

### 2. Deriving the per-series cost, honestly

The number everyone quotes is "1–3 KB per active series." It is a useful planning figure, but quoting it without knowing where it comes from means you cannot tell when it is wrong — and it *is* wrong, in both directions, depending on your label sets. Derive it.

For one active series in the head on a 64-bit build:

| Component | Bytes | Where it comes from |
|---|---:|---|
| `memSeries` struct | ~180–220 | ~20 fields: `ref` (8), `meta` ptr (8), `shardHash` (8), `sync.Mutex` (8), `lset` string header (16), `mmappedChunks` slice header (24), `headChunks` ptr (8), `firstChunkID` (8), `ooo` ptr (8), `mmMaxTime` (8), `nextAt` (8), `lastValue` (8), two histogram ptrs (16), `app` interface (16), `txs` ptr (8), counters/bools (~8) + Go size-class rounding |
| Packed label string | **50–500** | the actual text of every name and value, plus 1 byte per length prefix. **This is the term you control.** |
| Open head chunk | ~150–500 | XOR-compressed samples: Prometheus averages **1–2 bytes per sample** after compression (`docs/storage.md`); a chunk targets `DefaultSamplesPerChunk = 120` samples, and the backing `[]byte` grows by doubling, so there is up to 2× slack mid-chunk |
| Postings entries | 8 × (L+1) | one `storage.SeriesRef` (8 B) appended per label pair, plus the all-postings list `""/""`. L=10 labels → 88 B |
| `stripeSeries` map entries | ~80–120 | two Go maps hold a pointer to every series (by ref and by hash) |
| **Total** | **~500 B – 1.4 KB** | before Go heap fragmentation and GC headroom |

Now add the operational reality: Go's garbage collector will not run at 100% heap utilisation, allocation is bucketed into size classes, and you need headroom for query evaluation. The **1–3 KB/series** figure people quote is that derived 0.5–1.4 KB with a realistic 1.5–2× multiplier for live heap plus GC headroom. It is a range because the label-string term genuinely varies by an order of magnitude between a lean `up{job,instance}` series and a fully-labelled DCGM series.

A worked instance. Take the DCGM series from §1 and count its label bytes:

```
  __name__            8 +  21   = 29
  gpu                 3 +   1   =  4
  UUID                4 +  40   = 44
  device              6 +   8   = 14
  modelName           9 +  21   = 30
  Hostname            8 +  14   = 22
  namespace           9 +  11   = 20
  pod                 3 +  24   = 27
  container           9 +   7   = 16
  job                 3 +  13   = 16
  instance            8 +  20   = 28
                            ─────────
  name+value bytes                250
  length prefixes (22 × 1 B)  +    22
                            ─────────
  packed label string             272 bytes / series
```

So this series is roughly `200 (struct) + 272 (labels) + 300 (chunk, mid-fill) + 96 (postings) + 100 (maps)` ≈ **970 bytes**, call it ~1.9 KB with GC headroom. Ten million of them is **~19 GB of resident memory before a single query runs.** That arithmetic is the whole reason this lesson is first.

And note which line dominates the *controllable* part. Dropping `UUID` and `modelName` from the label set removes 74 bytes/series — 7.6% — while dropping them removes nothing from the *series count*. Dropping a label that had 200 distinct values removes 99.5% of the series. **Trimming label text is a linear saving; trimming label dimensionality is a multiplicative one.** Always attack the exponent first.

### 3. The identity, and the two ways it multiplies

For one metric name:

```
series(metric) = ∏ over labels L of  |distinct values of L|
```

For a scrape target or a whole fleet:

```
series(total) = Σ over metric names of  ∏ over labels of |distinct values|
```

Both halves matter, and conflating them is the most common sizing error:

- **Within a metric, cardinality is combinatorial.** Two individually-reasonable labels — `tenant` at 200, `model` at 30 — are a 6,000× multiplier, and that rides on top of whatever fleet fan-out (nodes × GPUs) you already have.
- **Across metrics, cost is additive.** Each metric name has its own independent series space. A target exposing ten metrics contributes the *sum* of their products, not a product across all of them. People who multiply across metrics overestimate wildly and, worse, lose the ability to identify *which* metric is the offender.

The practical consequence of "combinatorial within, additive across" is that **the cheapest fix is almost always to split one over-labelled metric into two under-labelled ones.** If you need `DCGM_FI_DEV_SM_ACTIVE` sliced by `gpu` (8/node) and separately by `tenant` (200), do not put both on one metric (8 × 200 per node); emit the per-GPU series raw and derive the per-tenant view with a recording rule (8 + 200 per node's worth of aggregation). The sum beats the product every time.

The dimensionality rule of thumb, and the one sentence to memorise: **any dimension that could take more than ~10³–10⁴ distinct values in prod does not belong on a metric label — if it could have 10k values, it is a span attribute, a log field, or an exemplar.** `user_id`, `request_id`, `trace_id`, `gpu_uuid`, `email`, full URL paths, error messages, and `pod` (whenever pods churn) are all on the wrong side of that line.

### 4. Cardinality is five budgets, not one

Here is the part that a single "series count" number hides. The same cardinality drives five distinct resources, with different exponents and different failure signatures. Knowing which one you are about to hit determines the fix.

| Budget | What scales with cardinality | Scaling | The metric that shows it | Failure signature |
|---|---|---|---|---|
| **Head RAM** | one `memSeries` + labels + open chunk per active series | linear in *active* series | `prometheus_tsdb_head_series` vs `process_resident_memory_bytes` | OOMKill, usually mid-scrape |
| **WAL replay** | every series record must be re-read and re-created on restart | linear in series × WAL segments retained | `prometheus_tsdb_data_replay_duration_seconds` | a restart that should take 20 s takes 25 min; *this* is the outage |
| **Index / postings** | one map entry per distinct (name,value); one ref per series per label | linear in distinct pairs; Go map overhead is worst at 1-element lists | `prometheus_tsdb_symbol_table_size_bytes`, block `index` file size | slow `label_values()`, slow autocomplete, fat blocks |
| **Query fan-out** | a selector must merge N postings lists and decompress N chunk streams | linear in *matched* series per query | `prometheus_engine_query_duration_seconds`, samples-loaded counters | dashboards time out; one bad panel starves the engine |
| **Remote-write** | series must be batched, relabelled and shipped | linear in samples/s = series × 1/scrape_interval | `prometheus_remote_storage_samples_pending`, `..._shards` | queue backpressure, dropped samples, ingester 429s downstream |

Two things fall out of this table that people get wrong:

**Retention does not fix cardinality.** `--storage.tsdb.retention.time` controls how long *persisted blocks* live on disk. The head block holds roughly the last 2–3 hours regardless (`DefaultBlockDuration = 2h`; the head is compacted when it spans more than 1.5× the chunk range). A cardinality spike kills the process in minutes, long before any retention policy is consulted. If someone proposes shortening retention to fix a memory problem, they have mistaken the disk budget for the RAM budget.

**Increasing the scrape interval barely helps either.** Doubling `scrape_interval` halves your *sample* rate — which helps remote-write and disk — but the series still exist, so head RAM, postings and query fan-out are unchanged. It actually makes the chunk term slightly *worse* per unit time, because a 120-sample chunk now covers 30 minutes instead of 15, so a given series holds an open chunk for longer. **Sample rate and series count are orthogonal axes. Diagnose which one you are on before you turn a knob.**

The one knob that *does* move all five at once is reducing distinct label values, which is why the whole lesson points there.

### 5. The signal-fit matrix — read it as "what gets indexed"

Each signal answers a different question and has a different cost profile. The useful way to compare them is not "cheap vs expensive" but **what does this system build an index over, and what does that index cost per distinct value.**

| Signal | Answers | What is indexed | Cardinality tolerance | Cost driver |
|---|---|---|---|---|
| **Metrics** (Prometheus/Mimir) | aggregate trend, "is it healthy", alerting | *every* label pair, always, in RAM | **bounded** — the constraint | active series count |
| **Traces** (Tempo/Jaeger) | causality, latency attribution, "where did the time go" | trace ID (always); span attributes only if you opt in | per-event, **sampled** | spans stored × sampling rate |
| **Logs, index-light** (Loki) | arbitrary context, "what exactly happened to *this* request" | **only the stream labels** — the body is not indexed | unbounded in the body, **bounded in the labels** | streams × chunk churn, then bytes scanned at query time |
| **Logs, index-heavy** (Elasticsearch/OpenSearch) | same, plus fast arbitrary field search | every mapped field, inverted, on disk | field *count* is the constraint (mapping explosion) | index size ≈ 1–2× raw; heap for fielddata |
| **Profiles** (Pyroscope/Parca) | "where do CPU cycles / allocations go" across the fleet | stack-trace symbol table + a small label set per profile series | continuous, whole-fleet | symbol table + samples/s × stack depth |
| **Events** (K8s events, XID, deploys) | discrete state changes | usually nothing; they are annotations | discrete | negligible; they *explain* the other signals |

The row that catches people is Loki: it looks like "cheap logs", and it is — **as long as you keep its stream labels as bounded as a metric's labels.** A Loki stream is defined by its label set exactly the way a Prometheus series is, and putting `request_id` in a Loki label is the same bomb in a different building. Lesson 6 does the full arithmetic; note the shape here.

Events deserve their own line because they are the connective tissue. A deploy marker, an XID error, a preemption, a MIG reconfiguration — each is a discrete annotation that *explains* a step change you can see in metrics and lets you jump to the trace or log for the "why." They are nearly free and chronically under-instrumented.

### 6. The cost/value inversion, in bytes

The signals order **inversely** on value-per-byte and cost-per-byte. Make that concrete rather than rhetorical.

Take one HTTP service handling **10,000 requests/second**, and ask what each signal costs to answer "what is p99 latency?" over a day.

```
  ONE QUESTION — "what is p99 latency right now?" — PRICED PER SIGNAL
  ══════════════════════════════════════════════════════════════════════════

  METRICS (histogram, 12 buckets + _sum + _count = 14 series per label combo)
     14 series × 1 combo, 15 s scrape
     samples/day  = 14 × (86400/15)          = 80,640
     bytes/day    = 80,640 × 1.5 B/sample    ≈ 121 KB          ← per day
     query cost   = read 14 series           ≈ microseconds
     ─ answers the question exactly, forever, for a rounding error.

  TRACES (1 span per request, 100 % sampled, ~500 B/span compressed)
     spans/day    = 10,000 × 86,400          = 864,000,000
     bytes/day    = 864e6 × 500 B            ≈ 432 GB/day
     at 1 % head sampling                    ≈ 4.3 GB/day
     ─ answers it, but only if you keep enough spans, and the p99 you
       compute from a 1 % sample has real sampling error at the tail.

  LOGS (1 access-log line per request, ~300 B)
     lines/day    = 864,000,000
     bytes/day    ≈ 259 GB/day raw, ~26–52 GB compressed at 5–10×
     query cost   = scan all of it to compute one percentile
     ─ answers it, at ~5 orders of magnitude more storage than the metric,
       and the query takes minutes instead of microseconds.
```

Three orders of magnitude of storage and five of query cost, for the same answer. That is the inversion:

```
   value-per-byte:   metrics  >  profiles  >  traces  >  logs
   cost-per-byte:    metrics  <  profiles  <  traces  <  logs
```

**The staff move is to demote every question to the cheapest signal that can still answer it, then use exemplars to keep the expensive signals reachable for the events that actually matter.** "What is p99?" is a metric question. "Why was *this specific request* at p99?" is a trace question — and you get from one to the other by following an exemplar, not by making the metric high-cardinality.

The corollary that is easy to miss: this ordering is about *bytes*, not about *usefulness*. Logs are last on cost-per-byte and first on "can answer literally anything." That is why the answer is never "delete logs" — it is "stop paying log prices for questions a metric answers."

### 7. Exemplars: the mechanism that makes demotion safe

If you are going to keep telling people "that goes on a span, not a label," you owe them the bridge. Exemplars are that bridge, and they are a real storage system with real limits — not a hand-wave.

**What an exemplar is.** In the OpenMetrics exposition format, a histogram bucket line may carry a trailing `#` followed by a label set, a value and an optional timestamp:

```
http_request_duration_seconds_bucket{le="2.5"} 84 # {trace_id="4bf92f3577b34da6a3ce929d0e0e4736"} 2.31 1755504312.451
```

That says: of the 84 observations in this bucket, here is *one* of them, it took 2.31 s, and its trace is `4bf92f35…`. The metric stays bounded — `trace_id` is **not** a label, it is a payload attached to the bucket.

**How Prometheus stores them.** Enabled with `--enable-feature=exemplar-storage` and sized in the config file:

```yaml
storage:
  exemplars:
    max_exemplars: 100000     # total across ALL series, not per series
```

The implementation (`tsdb/exemplar.go`, `NewCircularExemplarStorage`) is a **single fixed-size circular buffer shared by every series in the process.** Not a buffer per series — one global ring. Prometheus's own docs put an exemplar carrying just a `trace_id` at **roughly 100 bytes** of memory, so a 100k-exemplar ring is on the order of 10 MB. Exemplars are also appended to the WAL, so they survive a restart for as long as the WAL does.

**The failure mode this creates**, and it is the one people hit: because the ring is global and overwritten oldest-first, **a high-throughput service can evict a low-throughput service's exemplars.** Your latency panel on the busy service always has an exemplar to click; the panel on the quiet service that only sees one slow request an hour usually has nothing, because the ring wrapped. If exemplar coverage matters for a specific service, the fix is to raise `max_exemplars` (linear memory cost) or to scrape that service into a Prometheus with less exemplar traffic — not to add a label.

**Getting them out.** `/api/v1/query_exemplars` takes an expression and a time range and returns exemplars for the matching series. Grafana wires this to the "exemplar" dots on a graph panel; the data-source config maps the exemplar's `trace_id` label to a trace data source, which is how you go from "the p99 line moved" to "here is the trace of one request that made it move" in two clicks.

**One important scope limit:** exemplars attach to *counter and histogram* observations. They are not a general escape hatch for gauges. A DCGM gauge cannot carry an exemplar in the classic exposition path; the equivalent GPU move is a low-churn **info/mapping metric** (`dcgm_gpu_info{gpu, UUID, pod, namespace} 1`) joined at query time with `on(...) group_left(...)`, which lesson 9 develops in full. The pattern is the same — keep the identity off the hot series, resolve it at query time — but the mechanism differs.

### 8. Cardinality is time-varying — the shape of the real incident

The bomb almost never arrives as an obviously stupid label. It arrives as a label that was fine and stopped being fine. Hold this timeline; it is what the alerting in §9 is built to catch.

```
  A CARDINALITY INCIDENT, FROM DEPLOY TO OUTAGE
  ═══════════════════════════════════════════════════════════════════════════
  Fleet: 4,000 GPU nodes.  Prometheus limit: 64 GiB.  Baseline head: 12M series.

  t+0m    Scheduler config change lands: bin-packing made more aggressive.
          Nothing observability-related changed. No metric was edited.
             head_series ▏████████                       12.0 M   RAM  23 GiB

  t+0-20m Preemption rate rises. Training pods now restart every ~4 min
          instead of every ~6 h. Each restart = a NEW pod name = a NEW
          value of the `pod` label on every DCGM series for that node.
          Old series don't vanish: they stay in the head until the head is
          truncated, ~2–3 h later.
             head_series ▏█████████████                  19.4 M   RAM  36 GiB
             ↑ rate of series creation, not the absolute count, is the tell:
               prometheus_tsdb_head_series_created_total goes from
               ~40/s baseline to ~9,000/s

  t+35m   Head RAM crosses the container limit during a scrape burst.
          ⚠ OOMKilled. Container restarts in 3 seconds. Looks benign.
             head_series ▏                                0     RAM   1 GiB

  t+35m   ── WAL REPLAY BEGINS ──────────────────────────────────────────────
          The WAL still contains every one of those 19.4 M series records.
          Replay must re-create each memSeries, re-insert into stripeSeries,
          re-append to every postings list. Single restart, single-threaded
          per-shard work.
             replay progress ▏███▏                        ~2 GiB/min of WAL

  t+35–58m  NO SCRAPES ARE SERVED. NO QUERIES ARE SERVED. NO ALERTS EVALUATE.
          This is the outage. The OOM was 3 seconds; the replay is 23 minutes.
          And the 23 minutes of missing samples are gone permanently —
          Prometheus pulls, so an unscraped interval is not recoverable.

  t+58m   Replay completes. Head is rebuilt at 19.4 M series.
          RAM is right back where it was. It OOMs again in 6 minutes.
          ── CRASH LOOP ──
             The only exits: raise the memory limit, or drop the label
             (metric_relabel_configs, requires a restart to take effect
             but takes effect on the NEXT scrape, so the head drains at
             the next head truncation ~2 h later), or delete the WAL and
             accept the data loss.
  ═══════════════════════════════════════════════════════════════════════════
```

Read the four lessons off that timeline:

1. **The trigger was not an observability change.** It was a scheduler change. Cardinality is coupled to workload behaviour you do not own.
2. **The leading indicator is a rate, not a level.** `rate(prometheus_tsdb_head_series_created_total[5m])` moved 20 minutes before `prometheus_tsdb_head_series` looked alarming, because the absolute count climbs slowly at first. Alert on the derivative.
3. **The outage is the recovery, not the crash.** OOM is 3 seconds. WAL replay is tens of minutes and scales with the very thing that caused the OOM — a positive feedback loop. This is why lesson 3 treats replay time as a first-class capacity number.
4. **Fixing it is slow even once you know the fix.** Dropping the label stops *new* bad series, but the existing ones live in the head until truncation. Plan for a bridging action (raise the limit) as well as the real fix.

### 9. Governance: the three enforcement points

The 10³–10⁴ rule holds in practice only if something enforces it. Three places, from cheapest to last-resort.

**(a) CI-time linting, in the PR that adds the metric.** The cheapest possible gate: reject the label before it exists. `pint` (Cloudflare's Prometheus rule linter) checks rule files; for instrumentation, a simple repo-level check on metric-definition sites catches the obvious offenders:

```yaml
# .github/workflows/metric-lint.yml  (excerpt)
- name: reject unbounded metric labels
  run: |
    # Any new prometheus metric declaring a label from the deny-list fails the build.
    DENY='user_id|request_id|trace_id|span_id|session_id|email|uuid|full_url|pod_name'
    if git diff origin/main...HEAD -U0 -- '*.go' '*.py' \
       | grep -E '^\+' | grep -Ei "(LabelNames|labelnames|labels=)\[?.*($DENY)"; then
      echo "::error::high-cardinality label proposed; use a span attribute or exemplar"
      exit 1
    fi
```

Crude, and it will have false positives. It is still worth more than every wiki page you will ever write, because it puts the objection in front of the author at the moment the decision is cheap to reverse.

**(b) `metric_relabel_configs` — the scrape-time backstop.** This runs in Prometheus *after* the scrape and *before* ingestion, so it protects you from a label you do not control (a vendor exporter, a third-party chart) without a code change on their side. This is the config that actually saves you at 3am:

```yaml
scrape_configs:
  - job_name: dcgm-exporter
    kubernetes_sd_configs: [{ role: endpoints }]
    metric_relabel_configs:
      # 1. Drop the unbounded identity labels entirely.
      #    action: labeldrop takes a regex over LABEL NAMES.
      - action: labeldrop
        regex: 'UUID|DCGM_FI_DRIVER_VERSION|container_id'

      # 2. Keep pod, but only for the metrics that actually need per-pod slicing.
      #    Everything else loses it.
      - source_labels: [__name__]
        regex: 'DCGM_FI_DEV_(FB_USED|FB_FREE)'
        action: keep
        # (paired with a second scrape_config that drops `pod` for the rest)

      # 3. Hard stop: refuse the whole sample if a label value looks like a UUID.
      #    Cheap insurance against a new label appearing in an exporter upgrade.
      - source_labels: [pod]
        regex: '.*-[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}'
        action: drop

      # 4. Collapse a bounded-but-noisy label into buckets you actually query.
      - source_labels: [modelName]
        regex: 'NVIDIA (H100|H200|A100).*'
        target_label: gpu_family
        replacement: '${1}'
      - action: labeldrop
        regex: 'modelName'
```

Three mechanics worth stating explicitly, because they are the ones people get wrong:

- `labeldrop`/`labelkeep` match against **label names**; `drop`/`keep` match against the concatenated **values** in `source_labels`. Mixing them up produces a config that silently does nothing.
- The regex is **fully anchored** — `regex: 'pod'` means `^pod$`, not "contains pod."
- `metric_relabel_configs` runs per sample on every scrape. It is not free: a complex regex chain across millions of samples shows up in `prometheus_target_scrape_pool_sync_total` latency and in CPU. Prefer `labeldrop` (cheap) over value regexes (expensive) where both work.

**(c) Collector-side limits — the pipeline safety net.** In an OpenTelemetry pipeline you can cap attribute cardinality before it ever becomes a series, which lesson 4 develops:

```yaml
processors:
  # Delete or hash the attributes that must never become labels.
  transform:
    metric_statements:
      - context: datapoint
        statements:
          - delete_key(attributes, "request.id")
          - delete_key(attributes, "user.id")
          - replace_pattern(attributes["http.route"], "/[0-9a-f]{8,}", "/{id}")
```

None of the three is optional at fleet scale, and they are complementary: (a) stops new mistakes, (b) stops mistakes you do not own, (c) stops mistakes in the OTel path where there is no scrape to relabel.

**Governance also means a number.** A budget with no denominator is a preference. Write it as: *"The GPU subsystem may hold at most 1.5 M active series across the fleet. Per-metric budgets are allocated in `cardinality-budget.yaml` and enforced by a recording rule that alerts when any metric exceeds 120% of its allocation for 30 minutes."* Then the recording rule:

```yaml
groups:
  - name: cardinality-governance
    interval: 1m
    rules:
      # Per-metric active series. count by (__name__) is expensive; run it
      # as a rule at 1m, never ad hoc in a dashboard.
      - record: job:series_count:by_metric
        expr: count by (__name__, job) ({__name__=~".+"})

      - alert: MetricOverCardinalityBudget
        expr: |
          job:series_count:by_metric
            > on (__name__) group_left()
          (cardinality_budget_series * 1.2)
        for: 30m
        labels: { severity: ticket }
        annotations:
          summary: '{{ $labels.__name__ }} is {{ $value }} series, over its budget'

      # The leading indicator from §8 — series CREATION rate, not level.
      - alert: SeriesCreationRateSpike
        expr: |
          rate(prometheus_tsdb_head_series_created_total[5m])
            > 10 * avg_over_time(rate(prometheus_tsdb_head_series_created_total[5m])[6h:5m])
        for: 10m
        labels: { severity: page }
```

Note `cardinality_budget_series` is a metric you publish yourself — a tiny exporter or a static file of `cardinality_budget_series{__name__="DCGM_FI_DEV_SM_ACTIVE"} 40000` lines. Putting the budget *in* the monitoring system is what makes it enforceable rather than aspirational.

### 10. GPU-fleet tie

On the fleet the fatal high-cardinality labels are `gpu_uuid`/`UUID`, `job_id`, `pod`, and `mig_instance`. At 10k GPUs × up to 7 MIG instances × per-tenant labels, naive DCGM labelling is a bomb — and unlike a web service, the churn is driven by a scheduler you do not control (see §8).

The decision to make here, which lesson 9 resolves into config, is per-dimension:

| Dimension | Distinct values @ 10k GPUs | Verdict | Why |
|---|---:|---|---|
| `Hostname` / `node` | 1,250 nodes | **bounded label** | stable, you slice on it constantly, churn only on node replacement |
| `gpu` (index 0–7) | 8 | **bounded label** | you need per-device granularity for reclaim decisions |
| `UUID` | 10,000 | **drop → info metric** | stable per device but 10k-wide; carry it once in `dcgm_gpu_info`, join at query time |
| `modelName` | ~5 | **collapse → `gpu_family`** | few values but 21 bytes each × every series; map to a short token |
| `namespace` | ~200 | **bounded label** | this is your chargeback dimension; you must have it |
| `pod` | unbounded, churns | **drop from hot series** | the §8 bomb; resolve via the pod-resources join at query time |
| `mig_instance` | 7 per GPU | **bounded label, but multiply it in** | 8 × 7 = 56 per node, not 8 — budget for it |
| `job_id` / run ID | unbounded | **exemplar / log field / lakehouse column** | belongs in the analytics path (lesson 10), not the hot path |

The GPU-specific trap worth flagging now, because the rest of the module leans on it: **`DCGM_FI_DEV_GPU_UTIL` is a presence metric, not an intensity metric** — it reports the fraction of a short driver sample window during which at least one kernel was resident, which a batch-1 decode server pins at 100 while using a fraction of a percent of the tensor throughput. The `05-gpu-observability` module derives that from the NVML counter semantics; it matters *here* because it changes the signal model. If your headline GPU health signal is structurally unable to distinguish "busy" from "wasted", no amount of cardinality budget buys you a useful dashboard. The signal-fit question is not only "which signal type" but "which *field*, and does it measure the thing the question is about."

## Perspectives

**Systems-internals.** The per-series cost is dominated by two terms you can actually see in the source: the packed label string (per series, not shared — `labels_stringlabels.go`) and the open head chunk (up to 2× slack because the backing slice doubles). Everything else is small and fixed. That means a memory problem has exactly two shapes: too many series (attack the label *dimensionality*), or fat series (attack the label *text*). Diagnose which by dividing `process_resident_memory_bytes` by `prometheus_tsdb_head_series` — if bytes/series is above ~3 KB you have a label-text problem; if it is normal and the count is huge you have a dimensionality problem. Two different fixes.

**Economics.** Cardinality is a line item with a dollar figure, and the pricing models of commercial vendors are the clearest evidence that it is *the* cost driver rather than a technicality. Datadog's custom-metrics billing has historically been per unique series per month; the 2026 repricing that moves toward billing by metric *name* is a vendor redesigning its business model around exactly the constraint this lesson teaches. Treat every proposed label as a recurring cost with a multiplier attached, and price it before the design review, not after the invoice.

**Org-design.** The rule is toothless without an enforcement point, and the enforcement point has to be in the path the mistake actually travels. Mistakes from your own engineers travel through CI (gate (a)). Mistakes from vendors and third-party charts travel through the scrape (gate (b)). Mistakes from OTel-instrumented services bypass both (gate (c)). A "cardinality policy" that names only one of the three has an unguarded door, and the mistake will find it. Staff leverage here is not knowing the rule; it is owning the three gates and the budget file they enforce.

**Failure-mode.** Model cardinality as a time-varying property of a label, coupled to workload behaviour you do not own. The alert that saves you is on the *derivative* (`rate(prometheus_tsdb_head_series_created_total[5m])`), not the level, because the level climbs slowly at first and the recovery cost (WAL replay) is already committed by the time the level looks bad. Corollary: your post-incident action item is never only "we dropped the label" — it is "we dropped the label *and* we now alert on series-creation rate *and* we added it to the budget file."

**Query-engine.** Everything above is about ingest, but cardinality is also a query-time constraint that shows up later and hurts differently. A selector's cost is linear in *matched* series, so one dashboard panel doing `sum(rate(DCGM_FI_PROF_PIPE_TENSOR_ACTIVE[5m]))` across 32,000 series decompresses 32,000 chunk streams per evaluation. Six such panels on a wall-mounted dashboard refreshing every 10 s is a sustained query load that will starve rule evaluation — and rule evaluation is what your alerts depend on. The connection people miss: **a heavy dashboard degrades alerting**, because they share one query engine.

## Real-world use cases

- **Cloudflare, "How Cloudflare runs Prometheus at scale."** Roughly 900+ Prometheus instances holding on the order of 4.9 billion active time series across the edge fleet, with metric review and rule-linting (their open-source `pint` tool) as a first-class, enforced part of the pipeline rather than a convention. **What it shows:** at planet scale the technical fix (shard, federate, cap) is necessary but not sufficient — what keeps it alive is that a human or a linter says no to a label before it ships. It is the strongest available evidence that cardinality is a governance problem wearing a technical costume.

- **Uber, "M3: Uber's Open Source, Large-scale Metrics Platform for Prometheus."** Uber built and open-sourced M3 because their metrics ingest and cardinality outgrew what a Prometheus-per-cluster topology could hold, at a scale they described in the hundreds of millions of metrics per second aggregated. **What it shows:** there is a real cost cliff, and past it the answer is not "tune Prometheus" but "run a distributed TSDB with a different per-series cost model" — dictionary-encoded, sharded, with the labels index amortised across nodes rather than paid in full on each. It also shows the migration is a *year-scale* project, which is the argument for getting the label decisions right before you need it.

- **Datadog, "Infinite Cardinality Metrics."** A vendor publicly restructuring how custom metrics are priced — moving away from charging per unique series toward charging per metric name. **What it shows:** the vendor's own economics were dominated by unique series, exactly as this lesson's arithmetic predicts, and the fix required changing the product rather than the customers' behaviour. Read it as market confirmation of the identity `series = ∏(distinct label values)` being the thing that costs money.

- **The generic "pod-label" postmortem.** This one has no single canonical write-up because every large Kubernetes shop has lived it: a `pod` (or `pod_name`, or `instance` derived from pod IP) label that was safe at low churn becomes unbounded when deploy cadence, autoscaling aggressiveness, or preemption rate changes. The characteristic signature is the one in §8 — the trigger is a scheduler or deploy-pipeline change, not an observability change, and the outage is the WAL replay rather than the OOM. **What it shows:** the mechanism is more valuable to memorise than any specific incident, because you will meet the mechanism repeatedly and never meet the same incident twice.

## Worked example

**Cardinality budget for the GPU-fleet DCGM signal set.** Fleet: **4,000 nodes × 8 GPUs = 32,000 GPUs**, 200 tenants (namespaces), 5 GPU models. Prometheus (or Mimir ingester) memory target: keep the GPU subsystem under **1.5 M active series**.

**Step 1 — price the naive labelling.** dcgm-exporter's default `dcp-metrics-included.csv` plus the Kubernetes pod-resources enrichment gives every metric these labels: `gpu`, `UUID`, `device`, `modelName`, `Hostname`, `namespace`, `pod`, `container`, plus `job`/`instance` from the scrape. Take one metric first:

```
  DCGM_FI_DEV_SM_ACTIVE, naive labelling
  ────────────────────────────────────────────────────────────────
  Hostname            4,000     (one per node)
  gpu                     8     (index within node)
  UUID                     -    (functionally determined by Hostname×gpu → ×1)
  device                   -    (functionally determined → ×1)
  modelName                -    (functionally determined by node → ×1)
  namespace             200     (but only ~1 active per GPU at a time)
  pod                      ?    (see below)
  container                -    (≈1 per pod)

  Functional dependency matters: labels determined by other labels do NOT
  multiply the series count. UUID adds no series beyond Hostname×gpu — it
  adds BYTES (40 chars × every series). Different budget, same lesson.

  Steady state, no churn:
      series = 4,000 × 8 = 32,000                      ← one series per GPU
  With pod churn at the §8 rate (restart every 4 min, head holds ~3 h):
      distinct pod values per GPU over a head lifetime
                 = 180 min / 4 min                     = 45
      series = 32,000 × 45                             = 1,440,000
```

**One metric, one behavioural change, and you have consumed the entire fleet budget.** And dcgm-exporter's default CSV exports on the order of 20–30 fields, so multiply by that for the real total: **~29–43 M series.** At the ~1.9 KB/series derived in §2, that is **55–82 GB of head** for GPU metrics alone. Non-starter.

**Step 2 — classify each label.**

| Label | Distinct | Contributes | Verdict | Justification |
|---|---:|---|---|---|
| `Hostname` | 4,000 | ×4,000 | **keep** | primary slice dimension; churns only on node replacement |
| `gpu` | 8 | ×8 | **keep** | per-device is the unit of reclaim decisions |
| `UUID` | 32,000 | ×1 (dependent) | **drop → info metric** | +44 B/series for identity you resolve rarely |
| `device` | 8 | ×1 (dependent) | **drop** | derivable from `gpu`; pure byte cost |
| `modelName` | 5 | ×1 (dependent) | **collapse → `gpu_family`** | 21 B/series → 4 B; keeps the one query you actually run |
| `namespace` | 200 | ×1 (≈1 active/GPU) | **keep** | chargeback dimension; does not multiply because a GPU has one tenant at a time |
| `pod` | unbounded | **×45 and rising** | **drop → join** | the bomb; resolve via `dcgm_gpu_info` at query time |
| `container` | ~1/pod | ×1 | **drop** | rides on `pod`, no independent value |

**Step 3 — recompute.**

```
  Post-fix, per metric:
      series = Hostname(4,000) × gpu(8) × namespace(1 active) = 32,000

  Across the exported field set (say 25 fields kept after the counter-set audit):
      total = 25 × 32,000                                     = 800,000 series

  Plus the identity mapping metric, one series per GPU:
      dcgm_gpu_info{Hostname,gpu,UUID,gpu_family,namespace,pod} 1
      total = 32,000 series      ← ALL the churn is concentrated here

  Plus recording-rule outputs (per-tenant, per-family rollups):
      ~200 tenants × ~10 rollups + 5 families × ~10             ≈ 2,050 series

  GRAND TOTAL ≈ 834,000 series          ← under the 1.5 M budget, 44 % headroom
```

From ~29–43 M to ~834 k — **a 35–50× reduction** — by removing dependent labels (bytes) and unbounded ones (dimensionality). Memory: 834,000 × 1.9 KB ≈ **1.6 GB of head** for the entire GPU subsystem.

**Step 4 — check what you gave up, and price the workaround.** You can no longer write `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE{pod="train-clip-7b-worker-12"}` directly. The replacement is a vector-matching join against the info metric:

```promql
# Tensor-pipe activity for one specific training pod's GPUs.
DCGM_FI_PROF_PIPE_TENSOR_ACTIVE
  * on (Hostname, gpu) group_left (pod, namespace, UUID)
  dcgm_gpu_info{namespace="team-vision", pod="train-clip-7b-worker-12"}
```

Read the join: `on (Hostname, gpu)` says the two sides are matched by node and device index. `group_left (pod, namespace, UUID)` says the left side (the activity metric) may have many series per right-side series, and copy those three labels from the right onto the result. The output carries `pod` even though no stored series does.

**The cost you accepted, stated honestly:**

1. The join is evaluated at query time, so per-pod panels are slower than per-node panels — one extra postings intersection plus a hash join over 32,000 right-hand series.
2. `dcgm_gpu_info` itself churns (its `pod` label changes on every restart), so it carries the cardinality you removed from 25 other metrics — **but only once instead of 25 times**, which is a 25× saving and the entire point of the info-metric pattern.
3. Historical queries degrade: because the info metric's `pod` label changes over time and the join is evaluated per timestamp, a range query correctly attributes each interval to whichever pod held the GPU then — which is *better* than a static label would be, but surprises people who expect a stable series.
4. Ad-hoc "which pod is on GPU 3 right now" from a dashboard variable now needs a `label_values(dcgm_gpu_info, pod)` query rather than reading it off the metric.

**Step 5 — the enforcement.** The budget is a number in a file, and something must enforce it:

```yaml
# cardinality-budget.yaml → exported as cardinality_budget_series{...}
DCGM_FI_DEV_SM_ACTIVE:            40000    # 32k + 25 % headroom
DCGM_FI_PROF_PIPE_TENSOR_ACTIVE:  40000
DCGM_FI_DEV_FB_USED:              40000
dcgm_gpu_info:                    50000    # churn lives here; wider budget
```

paired with the `MetricOverCardinalityBudget` alert from §9. Re-run the arithmetic yourself with your own fleet numbers — **the multiplication is the lesson**, and the classification table is the artifact you will defend in a design review.

## Practice

Feeds the [fleet observability design](../practice/fleet-observability/README.md).

Build a **cardinality budget worksheet** for the fleet's core GPU signals. For each of `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_DEV_SM_ACTIVE`, `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`, `DCGM_FI_DEV_FB_USED`, `DCGM_FI_DEV_POWER_USAGE`, and one XID-error event stream:

1. **Enumerate labels and estimate distinct values** at 10k GPUs / up to 7 MIG instances / N tenants. For each label also record whether it is *independent* (multiplies the series count) or *functionally dependent* on another label (costs bytes only). This distinction is the one most worksheets miss.
2. **Compute the packed label-string length** for one representative series, the way §2 does it — name bytes + value bytes + one length prefix per field. This is your bytes-per-series term.
3. **Compute `series = ∏(independent distinct values)`** for the naive labelling, and then the head-RAM figure as `series × (struct 200 B + label string + chunk 300 B + 8×(L+1) + maps 100 B) × 1.8` for GC headroom.
4. **Classify every label** as *bounded keep*, *collapse* (map many values to few), *recording-rule aggregate*, or *drop → info-metric join / exemplar / log field*, with a one-line justification each and the 10³–10⁴ rule as your cutoff.
5. **Recompute post-fix** and check it lands under a stated fleet budget (e.g. 1.5 M active series for the GPU subsystem). Show both numbers and the ratio.
6. **Write the queries you lost.** For at least three questions that a dropped label used to answer directly, write the replacement PromQL — the `group_left` join, the exemplar hop, or the LogQL query — and note which is slower and by roughly how much.
7. **Write the enforcement.** Produce (a) the `metric_relabel_configs` block that implements your drops and collapses, (b) the `cardinality-budget.yaml` entries, and (c) the two alerting rules from §9 (budget overrun and series-creation-rate spike) with your own thresholds derived from your baseline. A worksheet with no enforcement plan is a wish list.
8. **Write the WAL-replay estimate.** Given your post-fix series count, estimate restart time and state the two mitigations you would put in the runbook. Lesson 3 gives you the model; committing a number here is what makes lesson 3 land.

**Acceptance criteria.** The worksheet is done when a peer can read it and (i) re-derive every series count from the label tables without asking you a question, (ii) find, for each dropped label, the exact query that replaces it, and (iii) point at the config file and the alert that stop the classification from silently reverting.

Carry the label/exemplar decisions forward; lesson 9 turns them into the concrete DCGM relabel config for a 10k-GPU fleet, and lesson 10 picks up every dimension you exiled to "the analytics path."

## Common pitfalls

- **"Cardinality only matters for metrics."** *Symptom:* a Loki deployment that ingests a tenth of the bytes of your Elasticsearch cluster but falls over anyway. *Mechanism:* it is the same failure wherever a dimension gets *indexed* — Loki stream labels define a stream exactly as Prometheus labels define a series, Elasticsearch mapping explosions come from field count, and trace backends that index arbitrary span attributes have the same problem in the same shape. The constraint is about indexes, not about Prometheus.

- **"Shorten retention and the cardinality problem goes away."** *Symptom:* someone drops retention from 15 d to 3 d and the OOMs continue unchanged. *Mechanism:* retention governs deletion of *persisted blocks* on disk. The head block holds ~2–3 h of data regardless (`DefaultBlockDuration = 2h`, head compacted past 1.5× that), and it is the head that OOMs. The disk budget and the RAM budget are different budgets with different knobs.

- **"Increase the scrape interval to save memory."** *Symptom:* interval doubled from 15 s to 30 s, memory unchanged or slightly worse. *Mechanism:* series count is unchanged, so every per-series structure is unchanged; only sample rate drops, which helps disk and remote-write. Worse, a 120-sample chunk now spans 60 minutes instead of 30, so each series holds an open, partly-slack chunk for twice as long.

- **"High cardinality is bad, full stop."** *Symptom:* a team refuses to add per-request detail anywhere, and incidents take hours because nobody can see individual requests. *Mechanism:* unbounded cardinality is *the point* of logs and traces — they are designed for it. It is specifically bad on an *indexed, always-resident* dimension. The correct instinct is not "less detail" but "the same detail, in the signal built to hold it."

- **"Exemplars solve cardinality for free."** *Symptom:* the busy service's panels have exemplar dots, the quiet service's never do. *Mechanism:* `tsdb/exemplar.go` implements one **global** circular buffer sized by `max_exemplars`, overwritten oldest-first across all series. High-throughput series evict low-throughput ones. Exemplars are cheap (~100 B each) and the right pattern, but they are a fixed-size ring, not a database.

- **"Dependent labels are free because they don't multiply."** *Symptom:* a fleet whose series count is fine but whose bytes-per-series is 4 KB. *Mechanism:* a label functionally determined by another (`UUID` by `Hostname`×`gpu`) adds no series, but `labels_stringlabels.go` stores the full text **per series**, uninterned. 44 bytes × 800,000 series is 35 MB for one label you never query. Free in the exponent, not free in the constant.

- **"We'll add the label now and clean it up later."** *Symptom:* a two-year-old label nobody dares remove. *Mechanism:* the label becomes load-bearing for dashboards, recording rules and alerts written by people who have left, and removing it is a breaking change with no test coverage. And removal does not retroactively free memory — the series persist until head truncation. The asymmetry between adding and removing is the reason the gate belongs in CI.

## Self-check

**Why is series count, not sample rate or byte volume, the first-order cost driver for a metrics backend?**
Because every distinct series allocates its own set of live structures in the head — a `memSeries` struct (~200 B), its own uninterned copy of the packed label string (50–500 B), an open head chunk with up to 2× slack (~150–500 B), one 8-byte postings ref per label pair, and two hashmap entries — for a derived ~0.5–1.4 KB and an operational 1–3 KB with GC headroom. Samples append into an *existing* chunk at 1–2 bytes each after XOR compression, so throughput is cheap. Series count also multiplies combinatorially as `∏(distinct label values)` while sample rate is merely linear in `1/scrape_interval`. Halving the scrape interval halves samples and touches nothing else; adding one 200-value label multiplies everything.

**A teammate wants to add `request_id` as a label on an HTTP request-rate metric "so we can find slow requests." What do you tell them, and where does that data belong?**
No — `request_id` is unbounded, orders of magnitude past the 10³–10⁴ cutoff, and every distinct value permanently allocates a series that lives until head truncation. It belongs as a span attribute on a trace (searchable, sampled) or a log field. The bridge they actually want is an **exemplar**: instrument the latency histogram to attach `trace_id` to bucket observations, enable `--enable-feature=exemplar-storage`, and wire Grafana's exemplar link to the trace data source. They then click the p99 dot and land on the trace of a real slow request, with the metric still at 14 bounded series.

**State the cost/value inversion and the staff move it implies. Give the order-of-magnitude numbers.**
Value-per-byte runs metrics > profiles > traces > logs; cost-per-byte runs the inverse. For a 10k-req/s service answering "what is p99?": a histogram costs ~121 KB/day and answers in microseconds; 100%-sampled traces cost ~432 GB/day; access logs cost ~259 GB/day raw and require a full scan to compute the percentile. Roughly six orders of magnitude of storage for the same answer. The staff move is to demote every question to the cheapest signal that still answers it, and to use exemplars to keep the expensive signals reachable only for the specific events that mattered.

**A scrape target exposes ten metrics, each with its own label set. How do you combine their cardinality costs, and what's the common mistake?**
Additively: `Σ over metrics of ∏(label cardinalities for that metric)`, because each metric name has an independent series space. The common mistake is multiplying across metrics as if they shared one combinatorial space, which overstates the total wildly and — more damagingly — destroys your ability to identify which single metric is the offender. The same insight is also the standard fix: splitting one over-labelled metric into two under-labelled ones converts a product into a sum.

**A label passed cardinality review six months ago and is still on the metric today. Is it still safe?**
Not necessarily. Cardinality is a time-varying property coupled to workload behaviour you do not own. A `pod` label at 200 stable pods is fine; the same label becomes a bomb the moment preemption rate, autoscaling aggressiveness or deploy cadence changes — and that change will land in a scheduler PR, not an observability one. Instrument the *derivative*: alert on `rate(prometheus_tsdb_head_series_created_total[5m])` against its own 6-hour baseline, because the absolute count climbs slowly at first while the WAL is already accumulating the series records that will make your recovery slow.

**Prometheus OOMs and restarts in 3 seconds. Why is that a 25-minute outage?**
Because the crash is not the outage; the WAL replay is. The write-ahead log still contains a series record for every series that was in the head, and replay must re-parse each one, re-create the `memSeries`, re-insert into the 16,384-stripe hashmap and re-append to every postings list before the process serves a single scrape or query. Replay time scales with the same series count that caused the OOM, so the failure mode is self-reinforcing. During replay nothing is scraped, and because Prometheus pulls, the unscraped interval is permanently lost. Watch `prometheus_tsdb_data_replay_duration_seconds` and treat it as a capacity number, not a curiosity.

**You have two Prometheis with identical `prometheus_tsdb_head_series`, but one uses 3× the RAM. What is the most likely cause and how do you confirm it?**
Label *text*, not label dimensionality. Because `stringlabels` stores an uninterned copy of the full packed label string per series, long values (`UUID`, `modelName`, full URLs, Java class names, container IDs) inflate bytes-per-series without changing the count. Confirm by computing `process_resident_memory_bytes / prometheus_tsdb_head_series` on both — above ~3 KB/series points at text — and then by inspecting `/api/v1/status/tsdb`, which reports the top label names and values by cardinality and by series count. The fix is `labeldrop`, or collapsing long values to short tokens with a relabel `replacement`.

**Why does a heavy Grafana dashboard put your alerting at risk?**
Because dashboards and rule evaluation share one query engine and one CPU budget. A selector's cost is linear in matched series, so a panel doing `sum(rate(X[5m]))` over 32,000 series decompresses 32,000 chunk streams per evaluation; several such panels refreshing every 10 s is a sustained load. When the engine saturates, rule evaluation queues, `prometheus_rule_evaluation_duration_seconds` and `prometheus_rule_group_iterations_missed_total` climb, and alerts fire late or not at all. The mitigation is exactly the one from §3 — pre-aggregate the fleet-wide panels into recording rules so the dashboard reads a few hundred series instead of tens of thousands.

## Connections & what's next

This lesson's cardinality budget is the constraint every later lesson inherits. Lesson 2 shows what happens when the query semantics on top of a correctly-budgeted signal are wrong; the `group_left` join and recording rules introduced here are its raw material. Lesson 3 is the fall-over mode of §4 and §8 at genuine fleet scale, and turns WAL replay from an anecdote into a sizing number. Lesson 4 adds the third enforcement gate (Collector-side attribute limits) and the pipeline where relabelling no longer applies. Lesson 6 applies the identical governance failure to Loki stream labels. Lesson 7 consumes bounded, correct signals as SLIs. Lesson 9 turns the classification table in §10 into deployable DCGM relabel config for 10k GPUs. Lesson 10 picks up every dimension you exiled from the hot path — `job_id`, `pod`, `UUID`, per-request detail — and gives it a home where cardinality is free and latency is not.

Next: [02 · Prometheus and PromQL](02-prometheus-and-promql.md) — the concrete query-semantics traps that quietly violate the signal-fit and cost decisions made here.

## References & further reading

**Primary sources — read directly from the tree**
- Prometheus 3.14.0 source, `tsdb/head.go` — `memSeries`, `stripeSeries`, `DefaultStripeSize = 1<<14`, `DefaultSamplesPerChunk = 120`, and the full list of `prometheus_tsdb_*` metrics quoted in §4. Verified against the `main` branch, August 2026.
- Prometheus 3.14.0 source, `model/labels/labels_stringlabels.go` — the packed-string label encoding (length-prefixed, 1 byte for 0–254 else 0xFF + 3 bytes LE) and its `//go:build !slicelabels && !dedupelabels` guard confirming it is the default build. This is the basis for the "label bytes are paid per series" claim in §2, which contradicts the common assumption that label values are interned.
- Prometheus 3.14.0 source, `tsdb/index/postings.go` — `memPostings` as `map[string]map[string][]storage.SeriesRef`, the basis for the postings-cost term.
- Prometheus 3.14.0 source, `tsdb/exemplar.go` — `NewCircularExemplarStorage`, confirming a **single global** ring rather than per-series buffers. This corrects the previous version of this lesson, which stated that Prometheus "keeps a small fixed-size buffer per series by default."
- [Prometheus storage documentation](https://prometheus.io/docs/prometheus/latest/storage/) — on-disk layout, 2-hour blocks, 512 MB chunk segments, 128 MB WAL segments, the "1–2 bytes per sample" figure and the `needed_disk_space` formula. Also mirrored in the repo at `docs/storage.md`.
- [Prometheus feature flags documentation](https://prometheus.io/docs/prometheus/latest/feature_flags/) — `exemplar-storage`, including the "roughly 100 bytes per exemplar" figure and the fact that exemplars are appended to the WAL.
- [Prometheus metric and label naming practices](https://prometheus.io/docs/practices/naming/) — the upstream position on what belongs in a label.
- [Prometheus relabeling configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config) — the `metric_relabel_configs` actions used in §9, including the anchored-regex and `labeldrop`-matches-names semantics.
- [OpenMetrics 1.0 specification, Exemplars section](https://github.com/OpenMetrics/OpenMetrics/blob/v1.0.0/specification/OpenMetrics.md#exemplars) — the wire format for the exemplar line shown in §7.
- [OpenTelemetry signals concepts](https://opentelemetry.io/docs/concepts/signals/) — the collection-convergence framing referenced in §5.

**Real-world engineering write-ups**
- Cloudflare, [How Cloudflare runs Prometheus at scale](https://blog.cloudflare.com/how-cloudflare-runs-prometheus-at-scale/) — the 900-instance / ~4.9 B series figures and the `pint` linting workflow discussed in Real-world use cases.
- Uber, [M3: Uber's Open Source, Large-scale Metrics Platform for Prometheus](https://www.uber.com/en-IN/blog/m3/) — the cost-cliff argument for moving off single-node Prometheus.
- Datadog, [Infinite Cardinality Metrics](https://www.datadoghq.com/blog/infinite-cardinality-metrics/) — vendor repricing as evidence that unique series, not metric names, drive cost.

**Deeper dives**
- Last9, [How to Manage High-Cardinality Metrics in Prometheus](https://last9.io/blog/how-to-manage-high-cardinality-metrics-in-prometheus/) — practical triage using `/api/v1/status/tsdb`.

**Sources consulted but not relied upon.** Several vendor documentation domains referenced in the observability ecosystem are unreachable from this environment's egress proxy. Where a fact could not be confirmed against a fetched page, it was instead verified against the upstream Git repository (cloned and read locally, as noted above) or omitted. No figure in this lesson is sourced from a page that could not be read.
