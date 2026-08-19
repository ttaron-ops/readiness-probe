---
lesson: "A03.10"
title: "The telemetry lakehouse — SQL over months of fleet telemetry"
module: "A-03"
concept: "analytics path for telemetry"
status: not-started
est_time: "6 hrs"
prev: "09-gpu-and-ml-observability.md"
next: null
artifacts: ["two-path telemetry architecture diagram", "$/useful-GPU-hour SQL over a quarter", "hot-vs-cold retention + cost model"]
sources: 21
---

# A03.10 · The telemetry lakehouse — SQL over months of fleet telemetry

> **Concept.** Prometheus answers *"is this GPU on fire right now?"*. It structurally cannot
> answer *"what was team X's $/useful-GPU-hour last quarter, and why did it move?"*. That second
> question is a FinOps question, it is the one Module 11 and the capstone are built on, and it
> needs a second path off the same telemetry source: Kafka → columnar object storage → SQL.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

Lessons 1–9 built one path and built it properly: cardinality as the master constraint (L1),
PromQL you can trust (L2), a metrics system that survives 4,000 nodes (L3), collector-side
enrichment (L4), traces and exemplars (L5), logs (L6), burn-rate SLOs (L7), continuous profiling
(L8), and the fleet-wide GPU/ML synthesis (L9). Every one of those optimises the same thing:
**answering a question about the last few hours, in under a second, without going bankrupt on
active series.**

That optimisation has a price, and L1 and L3 already named it — you dropped `pod`, `workload_id`
and `gpu_uuid` from your metric labels to stay inside the cardinality budget. This lesson is
where that bill comes due. The labels you correctly refused to keep are exactly the labels a
cost question needs to join on. The fix is not a bigger Prometheus; it is a **second path with
different physics**, and this lesson is mostly about what those physics *are* — why a column
store charges almost nothing for a dimension that would bankrupt a TSDB, and what that costs you
in latency instead.

Everything below is checked against the **Apache Parquet format spec** (`apache/parquet-format`,
`master` — `README.md`, `Encodings.md`, `Compression.md`), the **Apache Iceberg table spec**
(`apache/iceberg`, `main` — `format/spec.md`), the **ClickHouse documentation**
(`ClickHouse/clickhouse-docs`, `main` — `docs/best-practices/partitioning_keys.mdx`,
`selecting_an_insert_strategy.md`, `docs/use-cases/observability/build-your-own/managing-data.md`),
and the **AWS Price List API** (`us-east-1`, retrieved 2026-08-18) for every dollar figure. The
rendered documentation sites and vendor blogs are unreachable from this environment; sources are
marked accordingly in the References.

## Why this matters

Your capstone deliverable is a GPU cost/efficiency controller, and its whole claim is
per-team, per-month attribution. Try to serve that from the hot stack and you get one of two
failures, both of which a staff interviewer will find in about ninety seconds:

- **Keep the labels.** `pod`, `namespace`, `workload_id` and `gpu_uuid` on every DCGM series →
  millions of active series → Prometheus OOMs on the head block. This is the failure mode you
  already learned to size for in L3. You traded it away on purpose.
- **Drop the labels and pre-compute.** Recording rules for every rollup you think you'll want →
  it works, until someone asks a question you didn't pre-aggregate ("split that by accelerator
  SKU and region"), and the answer is "I'd need to add a rule and wait a quarter for data."

The staff move is recognising that these are not two bad options, they are **one missing
component**. "We run two paths off one source: a hot path tuned for alerting and a cold path
tuned for accounting" is a complete, correct answer. "We'll add more Prometheus" is not.

There is also a plain reliability argument: **billing questions must be auditable**. A
downsampled 1-hour rollup is fine for a trend line and unusable for a chargeback dispute, where
someone will ask you to reproduce a specific team's number for a specific week six months ago —
including the misattribution window that L9 §10 warned you about at every job boundary.

## What's new here (calibration)

- You already know Prometheus/Thanos/Mimir, remote-write, downsampling, and why cardinality kills
  a metrics system (L1, L3) — none of that is re-taught. You also already know the FOCUS billing
  spec from Module 11 L10.
- **New: the two-path model as an architecture**, not a workaround — what each path is physically
  good at, and the *seam* (Kafka) that lets them share one source without coupling.
- **New: why cardinality is nearly free in a column store**, derived from dictionary encoding and
  RLE rather than asserted — including the ~20,000× cost ratio for one high-cardinality dimension.
- **New: Parquet physically** — row groups, column chunks, pages, the footer written last, the
  encodings (`RLE_DICTIONARY`, `DELTA_BINARY_PACKED`, `BYTE_STREAM_SPLIT`), the recommended
  sizes (512 MB–1 GB row groups, 8 KB pages), and per-column encoding choices for GPU telemetry.
- **New: Iceberg physically** — metadata → snapshot → manifest list → manifest → data file, the
  manifest-entry statistics that make file pruning work, the partition transform table with the
  exact bucket hash, inclusive projection during scan planning, partition evolution, and snapshot
  expiry.
- **New: the cost model per tier with verified list prices**, including the request-count cost of
  over-partitioning — which is where a lakehouse actually goes wrong.
- **New: ClickHouse as the middle rung**, with its real constraints (partition cardinality,
  "too many parts", TTL and tiered storage, insert batching).
- **New: event-time and late data.** Prometheus quietly ignores this. An accounting ledger cannot.
- **New: when *not* to build one** — the honest answer for most fleets, with the break-even
  arithmetic rather than a shrug.

## Core concepts

### 1 · Two paths, one source

```
   THE TWO PATHS, WITH QUERY ROUTES AND COST PER TIER
   ═══════════════════════════════════════════════════════════════════════════════════
                        dcgm-exporter / OTel Collector / app SDKs
                                        │
                ┌───────────────────────┴───────────────────────┐
                ▼                                               ▼
   ╔══ HOT PATH (L1–L9) ══════════════╗          ╔══ COLD PATH (this lesson) ══════════╗
   ║  Prometheus / agent              ║          ║  Kafka  (durable seam, replayable)  ║
   ║      │ remote_write              ║          ║      │                              ║
   ║      ▼                           ║          ║      ▼  stream job (Flink/Spark)    ║
   ║  Mimir / Thanos ingesters        ║          ║  Parquet files in object storage    ║
   ║      │ head block in RAM         ║          ║      │ Iceberg / Delta table layer  ║
   ║      ▼ 2h blocks → object store  ║          ║      ▼                              ║
   ║  Grafana · alert rules           ║          ║  Trino / ClickHouse / DuckDB / Spark║
   ╚══════════════════════════════════╝          ╚═════════════════════════════════════╝
        "is it on fire NOW?"                          "what did it COST last quarter?"
        latency: milliseconds                         latency: seconds → minutes
        PRICED IN: RAM per ACTIVE SERIES              PRICED IN: TB stored + TB SCANNED
        ≈ $0.00077 per series-month  (§10)            ≈ $23.5 per TB-month (S3 Standard)
        RETENTION DOES NOT CHANGE THE PRICE           + $5 per TB scanned (Athena)
        CARDINALITY IS THE PRICE                      CARDINALITY IS ~FREE (§2)

   ── the same question, priced both ways ────────────────────────────────────────────
     "per-namespace GPU-hours, last 24h"      hot: recording rule, ~10 ms,   $0
     "per-POD GPU-hours, last 24h"            hot: IMPOSSIBLE (cardinality)
                                              cold: 24 GB scanned, ~8 s,     $0.12
     "$/useful-GPU-hour by team, last quarter, joined to a time-versioned rate card"
                                              hot: IMPOSSIBLE (no join, no labels)
                                              cold: 25 GB scanned, ~30 s,    $0.12
```

| Concern | Hot path (Prometheus) | Cold path (lakehouse) |
|---|---|---|
| Query latency | sub-second | seconds → minutes |
| Retention | 15s raw → 5m → 1h, weeks | raw for days, rollups for **years** |
| Cardinality | tight — millions of active series | loose — **billions of rows is fine** |
| Joins | label-matching, single source | **SQL joins** to org chart, rate card, registry |
| Late data | dropped | first-class, bounded by watermark |
| Language | PromQL | SQL |
| Priced in | **RAM** (active series) | **storage + bytes scanned** |
| Failure shape | a cliff (OOM) | a slope (a bill) |
| User | on-call SRE, alert rules | FinOps, capacity, leadership |

> **The mental model.** Prometheus is a **circular buffer with math**. The lakehouse is a
> **ledger with joins**. Don't ask the buffer to do accounting; don't ask the ledger to page you.

The five question classes the hot path structurally cannot serve: **long history** (beyond
retention, or lost to downsampling), **complex SQL** (cross-signal, multi-window), **joins to
business data** (org chart, rate card, model registry), **ad-hoc analytical** (asked by people who
write SQL, not PromQL), and **forensic/compliance** (reproduce this exact number from eight months
ago). Every one of those is a Module 11 question.

### 2 · Why cardinality is nearly free in a column store

This is the physics the whole lesson rests on, and it is worth deriving rather than asserting.

**In a TSDB, a distinct label value creates a distinct object.** A new `(metric, label set)`
combination is a new `memSeries` struct, a new packed label string, a new open head chunk, and a
new entry in every postings list it belongs to — roughly 0.5–1.4 KB of resident memory before GC
headroom (L1 §2). The index is an inverted index from label pairs to series references, and the
*number of objects* is the product of distinct values. Cardinality is not a data-volume problem;
it is an **object-count problem**, and objects cost RAM whether or not anyone queries them.

**In a column store, a distinct value is an entry in a dictionary and an integer in a row.**
Parquet encodes a column chunk with `RLE_DICTIONARY` (`Encodings.md`, encoding 8, superseding
`PLAIN_DICTIONARY` = 2): the distinct values are written once into a **dictionary page — which
the spec requires to be the first page of the column chunk, at most one per chunk** — and the
data pages hold bit-packed indices into it, run-length encoded when values repeat.

Price the same dimension both ways:

```
   ONE DIMENSION WITH k DISTINCT VALUES, PRICED IN BOTH SYSTEMS
   ═══════════════════════════════════════════════════════════════════════════
   Setting: the fleet from L3/L9 — 32,000 GPUs, ~27 DCGM metrics,
            ≈ 1,000,000 active series before the new dimension.
   New dimension: `workload_id`, k = 200 concurrent distinct values,
            orthogonal to the existing label set.

   ── HOT PATH ───────────────────────────────────────────────────────────────
     series after      = 1,000,000 × 200            = 200,000,000 active series
     at ≈ $0.00077 per series-month (§10's derivation)
                                                    ≈ $154,000 / MONTH
     …and it does not survive the first scrape: 200 M active series at
     ~2.5 GB per 300 k series (Mimir sizing, L3 §10) is 1.7 TB of RAM
     before replication. The real answer is "it OOMs", not "it costs".

   ── COLD PATH ──────────────────────────────────────────────────────────────
     dictionary: 200 values × ~36 B                 = 7.2 KB PER COLUMN CHUNK
       (so ~7 KB per row group — of which there are a few thousand per month)
     per row: an RLE-packed index. 200 distinct values needs 8 bits, and
       telemetry rows arrive sorted by host so the value repeats in long runs.
       Assume the pessimistic non-run case: 1 B/row.
     rows/month (§4)  = 5.53e9/day × 30             = 1.66e11 rows
     added bytes      = 1.66e11 × 1 B               = 166 GB/month
     at $0.023/GB-month (S3 Standard, verified)     ≈ $3.82 / MONTH
     scan cost when the column IS read: +166 GB     ≈ $0.83 at $5/TB (Athena)
     scan cost when the column is NOT read          = $0  (columnar projection)

   ── THE RATIO ──────────────────────────────────────────────────────────────
     $154,000 vs $3.82  ≈  40,000×   (and the hot number is hypothetical,
     because that system does not run at all at 200 M active series.)
```

**Three consequences follow, and together they are the whole design rationale:**

1. **In the hot path, a dimension costs whether you query it or not.** The RAM is allocated at
   ingest. In the cold path, an unread column costs only its bytes at rest, and columnar
   projection means a query that does not name it reads none of it.
2. **Retention behaves oppositely too.** Hot-path cost is a function of *active series* and is
   essentially flat in retention — keeping 30 days versus 90 does not change the head block.
   Cold-path cost is linear in retention and independent of cardinality. **So the two systems'
   cost drivers are orthogonal, which is precisely why running both is cheaper than running
   either one stretched to cover both jobs.**
3. **The lakehouse's cost driver is bytes *scanned*, so its equivalent of a cardinality
   catastrophe is an unpartitioned query**, not a high-cardinality column. Different failure,
   different control (§6, §7).

### 3 · The tee, and why the seam is Kafka

You do **not** fork the scrape. You tee **after** collection, and the tee point is a durable log:

1. `dcgm-exporter` / the OTel Collector emits as normal — the hot path is untouched.
2. A sidecar or Collector exporter also publishes a **richer** event stream to Kafka.
3. A stream job (Flink/Spark/Beam, or plain consumers) writes columnar files to object storage.

Kafka earns its place for three reasons, and each has an operational number attached:

- **Decoupling.** The lake writer can be down without losing telemetry, bounded by the topic's
  retention. Size it deliberately: at 64k events/s (§4) and ~120 B/event on the wire, the raw
  topic ingests `64,000 × 120 B ≈ 7.7 MB/s ≈ 664 GB/day`. A 3-day retention with replication
  factor 3 is `664 × 3 × 3 ≈ 6 TB` of broker disk — that is your outage tolerance, priced.
- **Fan-out.** Multiple independent consumers from one publish: the lake writer, a real-time
  anomaly job, a per-team usage exporter, and a replay-into-staging job for testing.
- **Replay.** When you find a bug in the rollup job you reprocess from the log instead of losing
  the quarter. This is the property that makes the cold path *correctable*, and it is the single
  biggest reason not to write Parquet straight from the collector.

This is also the one place the cold path is *allowed* to diverge from the hot path: it can carry
`pod`, `workload_id` and `gpu_uuid` — **the labels the hot path had to drop.** That divergence is
the entire point.

**Topic layout** worth copying:

```
gpu_telemetry_raw          # every DCGM field, per device, every scrape
gpu_telemetry_rollup_1m    # pre-aggregated (see §5)
gpu_workload_events        # pod/job lifecycle joined to GPU UUID
gpu_hardware_events        # XID, ECC retirement, fabric errors — low-rate, high-value
```

Hardware events get their own topic precisely because they are low-rate: you do not want an XID
48 stuck behind a two-hour backlog of utilisation samples.

**Partition key:** `hash(cluster + ":" + hostname)`. This keeps a host's samples ordered in one
partition so per-host rollups are cheap and so the writer can produce host-sorted files (which,
per §4, is worth real money in compression). Keying by `device_uuid` is the tempting mistake — it
8×'s your partition count and destroys host locality for no ordering gain.

### 4 · Schema rules that save you a backfill

Use Avro or Protobuf with a schema registry. JSON at this volume is a tax on storage and on every
consumer's CPU. Four rules matter more than the rest:

1. **`device_uuid` is the join key — not `hostname` + `gpu_index`.** Hosts get reimaged and GPU
   indices shift underneath you; UUIDs survive. Carry both, join on the UUID.
2. **`ts_ms` is event-time, set by the exporter** — never Kafka ingest-time. An accounting record
   stamped with when it happened to arrive is not an accounting record (§8).
3. **`workload_id` is opaque to the exporter.** The sidecar copies it from a pod label or the
   scheduler processor (L9 §10); the lake does not care what it means, only that it is stable.
4. **Column order and sort order are a compression decision, not cosmetics.** Parquet compresses
   per column chunk, and dictionary + RLE encoding rewards *runs* of repeated values. Sorting
   rows by `(cluster, hostname, device_uuid, ts)` before writing turns the three identity columns
   into long runs — often 10–50× smaller than the same data in arrival order — and leaves `ts`
   monotonically increasing within a file, which is exactly what `DELTA_BINARY_PACKED` wants.

Per-column encoding choices for the GPU telemetry table, using the encodings the spec defines:

| Column | Type | Encoding | Why |
|---|---|---|---|
| `ts` | `INT64` (millis) | `DELTA_BINARY_PACKED` (5) | monotonic within a sorted file; deltas are tiny and pack to a few bits |
| `cluster` | `BYTE_ARRAY` | `RLE_DICTIONARY` (8) | ~4 distinct values, huge runs after sorting |
| `hostname` | `BYTE_ARRAY` | `RLE_DICTIONARY` | 4,000 distinct; runs of thousands of rows each |
| `device_uuid` | `BYTE_ARRAY` (fixed 36) | `RLE_DICTIONARY` | 32,000 distinct; the §2 argument, made concrete |
| `gpu_index` | `INT32` | `RLE_DICTIONARY` | 8 distinct |
| `namespace`, `pod`, `workload_id` | `BYTE_ARRAY` | `RLE_DICTIONARY` | high cardinality, cheap anyway |
| `sm_active`, `pipe_tensor_active` | `FLOAT` | `BYTE_STREAM_SPLIT` (9) | splits mantissa/exponent bytes into separate streams so a general codec finds structure — designed for exactly this |
| `power_w`, `temp_c` | `FLOAT` / `INT32` | `BYTE_STREAM_SPLIT` / `DELTA_BINARY_PACKED` | slowly varying |
| `xid` | `INT32`, mostly null | `RLE_DICTIONARY` | nulls cost a definition level, ~0 bits when uniform |

Codec: **ZSTD** for cold data (the spec lists `UNCOMPRESSED`, `SNAPPY`, `GZIP`, `LZO`, `BROTLI`,
`LZ4`, `ZSTD`, `LZ4_RAW`). ZSTD's compression/CPU trade dominates GZIP for this shape of data, and
Snappy is the choice only when the query engine is CPU-bound rather than IO-bound.

### 5 · Pre-aggregation, and the scan math that forces it

For your 4,000-node fleet from L3:

```
32,000 GPUs × 30 metrics × 4 samples/min  ≈  3.84M events/min  ≈  64k/s
                                          ≈  5.5B rows/day
                                          ≈  497B rows per 90 days
```

Kafka handles 64k/s without noticing. The problem is the **query**: "average host utilisation per
day for the last 90 days" against raw is a 497-billion-row scan. Roll up to host-grain 1-minute
first and the same question scans `4,000 hosts × 1,440 min × 90 days ≈ 518M rows` — **~960× less
data for a bit-identical answer.**

That ratio is why every real deployment emits multiple grains from one stream:

| Table | Grain | Rows / 90 days | Typical question |
|---|---|---:|---|
| `gpu_metrics_raw` | (device, metric, 15s) | 497 B | forensic drill-down; keep days, not months |
| `workload_util_by_device_1m` | (device, workload, minute) | 4.15 B | per-job efficiency, chargeback |
| `workload_util_by_host_1m` | (host, minute) | 518 M | host saturation, stranding |
| `workload_util_by_namespace_1m` | (cluster, namespace, minute) | 26 M | team-level utilisation |
| `workload_util_by_cluster_5m` | (cluster, 5 min) | 104 K | fleet capacity, forecasting |

Pick the grains from the questions you must answer, then **retain raw shorter than rollups** —
inverted from most people's instinct, and the single biggest cost lever here.

**One correctness rule that governs every rollup, and it is the same one module 05 makes for
GPU-hours:** a rollup of a *ratio* must carry enough information to be re-aggregated. Storing only
`avg(sm_active)` per minute is fine to average across minutes (equal-width windows) and **wrong**
to average across devices that were not present for the whole window. Store `sum_of_samples` and
`sample_count` alongside the mean, and every downstream aggregation becomes
`sum(sum_of_samples) / sum(sample_count)` — exact, order-independent, and immune to the
`avg_over_time × window_hours` inflation that the hot path has to be careful about. Two extra
`INT64` columns; they compress to almost nothing and they remove an entire class of dispute.

### 6 · Parquet, physically

**Parquet is the file; it is not the table.** Knowing its layout is what lets you reason about why
a query scans what it scans.

```
   PARQUET FILE LAYOUT — AND WHAT EACH LEVEL LETS A READER SKIP
   ═══════════════════════════════════════════════════════════════════════════
   ┌────────────────────────────────────────────────────────────────────────┐
   │ "PAR1"                                       4-byte magic              │
   ├────────────────────────────────────────────────────────────────────────┤
   │ ROW GROUP 0        ← horizontal slice of rows; spec RECOMMENDS 512MB–1GB│
   │   ┌──────────────────────────────────────────────────────────────────┐ │
   │   │ COLUMN CHUNK: ts          ← all of one column, CONTIGUOUS        │ │
   │   │   [dictionary page]  ← MUST be first; at most one per chunk      │ │
   │   │   [data page 0][data page 1]…   ← spec RECOMMENDS ~8 KB pages    │ │
   │   │        each page: header (min/max/null count) + encoded values   │ │
   │   │        each page individually CRC32-checksummable & compressed   │ │
   │   ├──────────────────────────────────────────────────────────────────┤ │
   │   │ COLUMN CHUNK: hostname                                           │ │
   │   ├──────────────────────────────────────────────────────────────────┤ │
   │   │ COLUMN CHUNK: sm_active                                          │ │
   │   └──────────────────────────────────────────────────────────────────┘ │
   ├────────────────────────────────────────────────────────────────────────┤
   │ ROW GROUP 1 … ROW GROUP M                                              │
   ├────────────────────────────────────────────────────────────────────────┤
   │ [optional COLUMN INDEX / OFFSET INDEX]  ← page-level min/max → page skip│
   ├────────────────────────────────────────────────────────────────────────┤
   │ FILE METADATA (Thrift, TCompactProtocol)                               │
   │   schema · row-group list · per-column-chunk byte offsets & statistics │
   │   ⚠ WRITTEN LAST, so a single-pass writer never has to seek back —     │
   │     and so a truncated file is UNREADABLE: no footer, no file.         │
   ├────────────────────────────────────────────────────────────────────────┤
   │ 4-byte footer length (LE)  ·  "PAR1"                                   │
   └────────────────────────────────────────────────────────────────────────┘

   READ PATH, and the three independent prunings it performs:
     1. read the last 8 bytes → footer length → read + parse FILE METADATA
     2. PROJECTION  : read only the column chunks named in SELECT
     3. ROW-GROUP   : skip row groups whose column statistics exclude the predicate
     4. PAGE (opt.) : with the page index, skip pages within a surviving chunk
     ⇒ a query touching 4 of 40 columns over 1 of 90 days reads roughly
       (4/40) × (1/90) of the bytes — TWO independent factors, multiplied.

   FAILURE GRANULARITY (from the spec's own error-recovery notes):
     corrupt file metadata → THE FILE IS LOST
     corrupt column metadata → that column chunk lost (other row groups fine)
     corrupt page header → the rest of that chunk lost
     corrupt page data → that page lost
   ⇒ smaller row groups = more resilience, worse sequential IO. The spec's
     512 MB–1 GB recommendation is an IO-throughput choice, not a safety one.
```

**The practical consequence people miss:** the row-group recommendation of 512 MB–1 GB is about
making sequential reads large, and it is in direct tension with the streaming writer's desire to
commit often. A writer that commits every 30 seconds at this fleet's rate produces files of
`64,000 events/s × 30 s × ~12 B/row ≈ 23 MB` — twenty to forty times smaller than the
recommendation, and that is *before* per-partition fan-out divides it further. That tension is the
small-files problem, and §7 prices it.

### 7 · Iceberg, physically — and where the query time actually goes

**Iceberg (or Delta, or Hudi) is the table layer over a pile of Parquet files.** It adds what a
directory of files cannot provide: atomic commits, schema and partition evolution, hidden
partitioning, time travel, and — the part that matters for query speed — **per-file statistics
that let the planner skip files without opening them.**

```
   ICEBERG METADATA TREE — ONE COMMIT, AND WHAT A SCAN READS
   ═══════════════════════════════════════════════════════════════════════════
   catalog (Glue / Hive / REST / Unity)
     └─▶ table metadata JSON        ← current schema, partition specs, sort
          │                            orders, snapshot list, current snapshot.
          │                            Each change writes a NEW metadata file,
          │                            swapped in by ONE atomic operation.
          └─▶ SNAPSHOT (a point in time)
               └─▶ MANIFEST LIST (avro)      ← one row per manifest, with
                    │                          partition summaries + counts
                    ├─▶ MANIFEST (avro)      ← one manifest_entry per data file
                    │     status: 0 EXISTING | 1 ADDED | 2 DELETED
                    │     snapshot_id, sequence_number, file_sequence_number
                    │     data_file {
                    │        file_path, file_format, partition tuple,
                    │        record_count, file_size_in_bytes,
                    │        column_sizes{}, value_counts{},
                    │        null_value_counts{}, nan_value_counts{},
                    │        lower_bounds{}, upper_bounds{},   ← THE PRUNING KEY
                    │        split_offsets[]  (= Parquet row-group offsets),
                    │        sort_order_id
                    │     }
                    └─▶ MANIFEST … (one per partition spec; a spec change
                                    starts new manifests, old ones stay)

   SCAN PLANNING, in order:
     1. read table metadata → current snapshot
     2. read manifest list; skip manifests whose PARTITION SUMMARIES cannot
        match (this is where most of the skipping happens, cheaply)
     3. read surviving manifests; convert the query predicate into a
        PARTITION predicate by INCLUSIVE PROJECTION, then use lower/upper
        bounds per column to drop individual files
     4. open the surviving Parquet files; use split_offsets to read row groups
   ⇒ planning cost ∝ MANIFEST bytes read, not data bytes. Over-partitioning
     inflates step 3 until PLANNING dominates EXECUTION (§7's arithmetic).
```

**Hidden partitioning and inclusive projection, which is the feature people cite without knowing
what it does.** A table declares a partition spec such as `day(ts)`; the spec transforms are:

| Transform | Result | Note |
|---|---|---|
| `identity` | the value | any primitive except geometry/geography |
| `bucket[N]` | `int` | `(murmur3_x86_32(value) & Integer.MAX_VALUE) % N` — sign bit discarded |
| `truncate[W]` | source type | `v − (v % W)` for numerics; first `L` code points for strings |
| `year` / `month` | `int` | years/months since 1970 |
| `day` | `date` | days since 1970-01-01 (readers must also accept `int`) |
| `hour` | `int` | hours since 1970-01-01T00:00 |
| `void` | null | used to retire a partition field in v1 tables |

The user writes `WHERE ts >= TIMESTAMP '2026-04-01'`; Iceberg derives the partition predicate
`ts_day >= day('2026-04-01')` itself. That derivation is the **inclusive projection**: if a row
matches the scan predicate, the partition predicate must match that row's partition — so it can
over-select (files at the boundary contain both matching and non-matching rows) but never
under-select. The practical payoff is that **you never write partition predicates by hand and
never get them wrong**, and the practical trap is that this only works when you filter on the
*source column*; filtering on a derived expression the transform cannot see defeats it entirely.

**Partition evolution** is possible because each manifest records the spec it was written with:
old files stay in old manifests under the old spec, new files go into new manifests, and scan
planning applies each manifest's own spec. So changing `day(ts)` to `hour(ts)` does not rewrite
history — but it does mean a query spanning the change reads two sets of manifests with different
partition predicates, and the older half prunes at day granularity.

**Snapshot expiry is a real retention control, and it is not optional.** Retention is governed by
`min-snapshots-to-keep`, `max-snapshot-age-ms` and `max-ref-age-ms`; the expiry procedure keeps
each branch's referenced snapshot plus ancestors until they are both older than
`max-snapshot-age-ms` and beyond `min-snapshots-to-keep`. Until a snapshot expires, **the data
files it references cannot be deleted** — so a table with no expiry policy keeps every version of
every compacted file forever, and your storage bill grows with your *compaction rate* rather than
with your data. That is one of the two ways a working lakehouse quietly becomes expensive.

**One design rule specific to telemetry:** avoid row-level deletes. Iceberg v2+ supports position
deletes, equality deletes and deletion vectors, and every one of them adds a file that must be
read and applied at scan time. Telemetry is append-only by nature; make corrections by writing a
restatement partition (§8) rather than by deleting rows, and your scan path stays a pure read.

**Now the arithmetic that decides whether your table is fast — the small-files and
over-partitioning cost, priced with real S3 request rates:**

```
   OVER-PARTITIONING, PRICED  (S3 us-east-1 list, verified 2026-08-18:
     PUT/COPY/POST/LIST $0.005 per 1,000 · GET and all others $0.0004 per 1,000)
   ═══════════════════════════════════════════════════════════════════════════
   ✗ PARTITION BY device_uuid, day    (the "we always join on it" mistake)
       partitions per day        = 32,000
       stream job commits        = every 30 s ⇒ 2/min
       files written per minute  = 32,000 × 2                = 64,000
       PUTs per month            = 64,000 × 60 × 24 × 30     = 2.76e9
       PUT COST                  = 2.76e9 / 1000 × $0.005    ≈ $13,800/MONTH
       …in request charges alone, before storing a single byte.
       Each file also holds ~170 rows ⇒ ~2 KB of data with a ~1 KB footer.
       Manifest rows per day     = 32,000 × 2,880            = 92 M
       ⇒ scan PLANNING reads gigabytes of manifests to skip kilobytes of data.

   ✓ PARTITION BY day, cluster        (filter dimensions, low cardinality)
       partitions per day        = 1 × 4                     = 4
       files written per minute  = 4 × 2                     = 8
       PUTs per month            = 8 × 43,200                = 345,600
       PUT COST                  = 345,600/1000 × $0.005     ≈ $1.73/MONTH
       …then COMPACTION rewrites the day into ~512 MB files:
       daily data ≈ 5.5e9 rows × 12 B ≈ 66 GB ⇒ ~130 files/day
       compaction PUTs           = 130 × 30                  = 3,900/month ≈ $0.02
       compaction GETs (read the small files back)
                                 = 8 × 43,200                = 345,600 ≈ $0.14
       Manifest rows per day     ≈ 130  ⇒ planning is microseconds.

   ⇒ THE RULE: PARTITION BY WHAT YOU FILTER ON, NOT BY WHAT YOU JOIN ON.
     File-level lower/upper bounds handle the join key for free — that is
     what the manifest statistics are FOR. And run compaction regardless:
     the un-compacted table is 8× more objects and reads 8× more footers.
```

### 8 · Event-time, watermarks, and late data

A GPU node partitions off for ten minutes and its samples arrive late. Three questions you must
answer explicitly, because silence here is how ledgers go wrong:

- **How late is acceptable?** A watermark — say 30 minutes — after which a partition is sealed.
- **What happens to later-than-that data?** A quarantine table you can reconcile from, never a
  silent drop, and never an unbounded window (which prevents the job from ever finalising).
- **Is a sealed day immutable?** For chargeback it must be. Restatements get a new version and an
  audit note, exactly like finance.

```
   THE SEALING PROTOCOL — ONE PARTITION'S LIFE
   ═══════════════════════════════════════════════════════════════════════════
   event-time day D
   ├─ D 00:00 ──────────────────────── D 23:59   OPEN
   │     stream job appends; each Iceberg commit is an atomic snapshot;
   │     readers always see a consistent set of files, never a half-write
   │
   ├─ D+1 00:00 → 00:30                          GRACE (watermark = 30 min)
   │     late-arriving events with ts in D are still accepted into D
   │
   ├─ D+1 00:30                                  SEAL
   │     · run compaction for D (small files → ~512 MB files)
   │     · compute and store a partition CHECKSUM: row count, sum of
   │       sample_count, distinct device count, min/max ts
   │     · publish `partition_status(D) = SEALED, version = 1`
   │     · anything with ts ∈ D arriving after this goes to QUARANTINE
   │
   ├─ D+3  someone finds a bug in the enrichment (e.g. the L9 §10 scheduler
   │       cache misattributed 40 minutes of one job)
   │     · REPLAY from Kafka into a staging table
   │     · write `partition_status(D) = SEALED, version = 2` + an audit row
   │       naming the reason, the query used, and the delta per team
   │     · version 1 remains readable via Iceberg TIME TRAVEL, which is what
   │       makes "reproduce the number we billed in April" answerable
   └─────────────────────────────────────────────────────────────────────────
   ⇒ The invariant to state out loud in a design review: A SEALED PARTITION
     IS NEVER MUTATED IN PLACE. Corrections are new versions with an audit
     trail. That single rule is what separates a ledger from a data dump.
```

**Two mechanisms make this cheap rather than heroic.** Iceberg's atomic snapshot swap means a
reader mid-compaction sees either the old files or the new ones, never a mix — so compaction can
run against a live table. And time travel is free: an expired-but-retained snapshot *is* the April
version, provided your `max-snapshot-age-ms` policy is longer than your dispute window. **Set the
snapshot retention from the chargeback dispute window, not from a default** — that is the one
place where an accounting requirement dictates a storage setting.

### 9 · The join that makes it worth building

The cold path's superpower is joining telemetry to data that is not telemetry: the **org chart**
(namespace → team → cost centre), the **rate card** (SKU × region × time → $/hour, including
committed-use tiers), the **device inventory** (UUID → SKU, node, NVLink island), and the
**model registry**. None of these live in Prometheus, and all of them are required to say a
dollar number out loud.

Time-versioning the rate card is the subtle part: rates change, and a query over Q2 must apply
the rate that was in effect on each day, not today's. That is a `BETWEEN valid_from AND valid_to`
join — trivial in SQL, impossible in PromQL. Build the dimension tables as **slowly-changing
dimensions, type 2**: never update a row, close it and insert a successor.

```sql
-- rate_card: type-2 SCD. One row per (sku, region, commitment) per validity window.
CREATE TABLE rate_card (
    sku          VARCHAR,      -- 'H100-80GB-SXM'
    region       VARCHAR,
    commitment   VARCHAR,      -- 'on-demand' | '1yr-reserved' | 'spot-avg'
    usd_per_hour DECIMAL(10,4),
    valid_from   DATE,
    valid_to     DATE,         -- exclusive; DATE '9999-12-31' for the open row
    source_ref   VARCHAR       -- WHERE this number came from: contract, invoice,
);                             -- price-list snapshot. Auditors ask; answer in the schema.

-- The invariant that must be enforced by a test, not by hope:
--   for every (sku, region, commitment), validity windows are
--   NON-OVERLAPPING and GAP-FREE across the whole reporting period.
-- A gap silently drops rows from the join and understates spend; an overlap
-- silently doubles it. Both are invisible in the output.
```

The same discipline applies to `team_mapping` (people move between teams, and last quarter's cost
belongs to last quarter's team) and to `device_inventory` (a UUID that was an A100 and is now an
H100 after an RMA is *two* rows, not an update).

### 10 · The cost model, tier by tier

Every figure below is from the AWS Price List API for `us-east-1`, retrieved 2026-08-18. Prices
change and differ by region and by negotiated discount — re-run the arithmetic, do not copy the
conclusion.

```
   VERIFIED UNIT PRICES (us-east-1, list, 2026-08-18)
   ═══════════════════════════════════════════════════════════════════════════
     S3 Standard storage       $0.023 /GB-mo (first 50 TB) · $0.022 (next 450 TB)
                                             · $0.021 (over 500 TB)
     S3 Standard-IA            $0.0125/GB-mo  + $0.01/GB retrieved
     S3 Glacier Instant Retr.  $0.0040/GB-mo  + $0.03/GB retrieved
     S3 requests               PUT/COPY/POST/LIST $0.005 /1,000
                               GET and all other  $0.0004/1,000
       (Standard-IA: PUT $0.01/1,000, GET $0.001/1,000;
        Glacier IR:  PUT $0.02/1,000, GET $0.01 /1,000)
     EBS gp3                   $0.08  /GB-mo
     EC2 r7i.4xlarge (16 vCPU / 128 GiB), on-demand Linux   $1.0584/hour
     EC2 r7i.8xlarge (32 vCPU / 256 GiB), on-demand Linux   $2.1168/hour
     Athena                    $5.00 per TB scanned
```

**Tier 1 — the hot TSDB, priced per active series.** Using Mimir's published sizing (L3 §10:
ingester ≈ 1 core and 2.5 GB per 300,000 in-memory series, with in-memory = active × RF):

```
   1,000,000 active series, RF = 3  ⇒ 3,000,000 in-memory series
     CPU    3.0e6 / 3.0e5 × 1 core            = 10 cores
     RAM    3.0e6 / 3.0e5 × 2.5 GB            = 25 GB
   On r7i.4xlarge (16 vCPU, 128 GiB) the binding constraint is CPU:
     instances = 10/16 = 0.625  ⇒ $1.0584 × 0.625 × 730 h ≈ $483/month
   Add ~60 % for distributors, queriers, store-gateways, compactor:
                                              ≈ $770/month per 1 M active series
     ⇒ ≈ $0.00077 per active series-month.   ← the number used in §2
   Samples produced: 1e6 × 4/min × 43,200 min = 1.73e11 samples/month
     at ~2 B/sample compressed                ≈ 345 GB/month
   IF you insist on a $/TB figure:            ≈ $2,230 per TB-month
     ⚠ but this is a CATEGORY ERROR: the cost does not move if you keep the
       data twice as long, and it doubles if you double the series at the
       same volume. Price the hot path per SERIES, not per byte.
```

**Tier 2 — ClickHouse (or any single-node/small-cluster column store), priced per TB stored.**

```
   Raw telemetry: 5.53e9 rows/day × ~12 B/row (encoded+ZSTD)  ≈ 66 GB/day
   90-day retention                                            ≈ 5.9 TB
     on gp3 at $0.08/GB-mo:  5,900 × 0.08                     ≈ $472/month
     compute: 2 × r7i.8xlarge (32 vCPU, 256 GiB) @ $2.1168    ≈ $3,090/month
                                                               ─────────────
                                                               ≈ $3,560/month
     per TB stored                                            ≈ $600/TB-month
     ⇒ compute-dominated, like the profile store in L8. This is the tier
       you pick when queries are FREQUENT and the dataset is single-digit TB.
```

**Tier 3 — the lakehouse, priced per TB stored plus per TB scanned.**

```
   Same 5.9 TB in Parquet/Iceberg on S3 Standard:
     5,900 GB × $0.023                                        ≈ $136/month
   Add rollups (device-1m for 2 years):
     4.15e9 rows/90d ⇒ 1.68e10 rows/year × ~10 B ≈ 168 GB/yr
     2 years                                                  ≈ 336 GB ≈ $8/month
   Requests (compacted layout, §7)                            ≈ $2/month
   Query engine, serverless (Athena) at 200 queries/month
     averaging 25 GB scanned: 200 × 0.025 TB × $5             ≈ $25/month
                                                               ─────────────
                                                               ≈ $171/month
     per TB stored                                            ≈ $27/TB-month
```

**The comparison table that belongs in the design doc:**

| Tier | Priced in | This fleet, 90 days | $/TB-month | Latency | Cardinality cost | Retention cost |
|---|---|---:|---:|---|---|---|
| Prometheus/Mimir hot | active series | $770 per 1M series | ~$2,230* | ms | **brutal** | ~free |
| ClickHouse | storage + compute | $3,560/month | ~$600 | 0.1–5 s | free | linear |
| Lakehouse (S3 + Athena) | storage + scan | $171/month | ~$27 | 5–120 s | free | linear |
| S3 Standard-IA (rollups >90d) | storage + retrieval | $0.0125/GB-mo | ~$12.8 | +retrieval | free | linear |
| S3 Glacier Instant (archive >1y) | storage + retrieval | $0.004/GB-mo | ~$4.1 | +$0.03/GB read | free | linear |

*\* the hot-path $/TB figure is included only for comparison and is misleading on its own — see
Tier 1's warning. The honest hot-path unit is $0.00077 per active series-month.*

**Two derived break-evens worth memorising, because they are the whole "when to build one"
answer in numeric form:**

```
   BREAK-EVEN 1 — WHEN DOES THE LAKEHOUSE PAY FOR ITSELF ON COST ALONE?
     Cold-path infrastructure has a fixed floor: Kafka (3 brokers ≈
     3 × r7i.4xlarge ≈ $2,320/mo) + a stream job (≈ $700/mo) + an engineer's
     attention. Call the floor ≈ $3,000/month before any data.
     It replaces hot-path spend only if it lets you REMOVE series. Removing
     a dimension worth 200,000 active series saves 200,000 × $0.00077 ≈ $154/mo.
     ⇒ You would need to remove ~4 MILLION active series for the lakehouse to
       pay for itself on infrastructure cost alone.
     ⇒ THEREFORE: a lakehouse is almost never justified by cost saving. It is
       justified by QUESTIONS THAT CANNOT OTHERWISE BE ANSWERED. Say that in
       the design review before someone builds a $3k/month cost-saving project
       that saves $154.

   BREAK-EVEN 2 — CLICKHOUSE vs LAKEHOUSE
     ClickHouse total ≈ $3,560/mo at 5.9 TB, and it is COMPUTE-dominated:
     the marginal TB costs $0.08 × 1,000 = $80/month.
     Lakehouse marginal TB = $23 storage + scan-when-read.
     ⇒ crossover on storage alone at roughly 60–80 TB, but the REAL crossover
       is organisational: multiple consuming teams, a need to join to
       warehouse data, and open-format portability. Below that, one
       ClickHouse is less machinery for the same answers.
```

### 11 · ClickHouse as the middle rung

For most fleets the correct answer is "one ClickHouse," so it deserves its constraints stated
properly rather than as a footnote.

**MergeTree stores data in parts and merges them in the background.** An insert creates at least
one part per distinct partition-key value among the inserted rows; background merges combine parts
into larger ones. **Merges never cross partitions.** Everything below follows from those two
sentences.

- **Partition-key cardinality is a hard operational constraint.** ClickHouse's own best-practice
  documentation recommends a **low-cardinality partitioning key — fewer than 100–1,000 distinct
  values** — because parts cannot merge across partitions, so a high-cardinality key produces
  parts that can never be merged away. Exceed the limits and inserts fail with the
  "too many parts" error (`parts_to_throw_insert` per partition, `max_parts_in_total` overall).
  Note this is the *same* rule as Iceberg's §7, arrived at from completely different mechanics.
- **Partitioning is a data-management tool, not a query optimisation.** The docs are explicit:
  its primary value is dropping/moving/archiving whole partitions as a metadata operation.
  ClickHouse *does* build MinMax indexes on partition columns automatically, so filtering on the
  partition expression prunes — but querying *across* many partitions can be slower than an
  unpartitioned table because of part fragmentation.
- **Inserts must be batched.** The recommendation is **at least 1,000 rows, ideally
  10,000–100,000**, and roughly **one insert query per second**, because each insert creates parts
  that background merging must keep up with. At 64k events/s that is one batch of 64,000 rows per
  second — comfortably inside the recommended range, which is not a coincidence: the guidance is
  built around exactly this shape of ingest. If you cannot batch client-side, asynchronous inserts
  move the batching server-side.
- **TTL is the retention mechanism, and one setting makes it cheap.** With
  **`ttl_only_drop_parts = 1`** ClickHouse drops a whole part once all its rows have expired,
  instead of running a mutation to delete rows — which is why the TTL interval should be a
  multiple of the partition period (partition by day, TTL in whole days). TTL merges are
  scheduled, not immediate: **`merge_with_ttl_timeout` defaults to 14,400 s (4 hours)** as the
  minimum delay before repeating a delete-TTL merge, and expiry can be forced with
  `ALTER TABLE … MATERIALIZE TTL`.
- **Tiered storage is TTL with a destination.** Recent partitions on SSD, older ones moved to S3
  by a `TTL … TO VOLUME` rule — the hot/cold split inside one engine, which is often all the
  "lakehouse" a fleet needs.
- **Codecs are a per-column, per-age decision.** `ZSTD(1)` is the general recommendation for
  observability data, and a `TTL Timestamp + INTERVAL 4 DAY RECOMPRESS CODEC(ZSTD(3))` rule
  re-compresses ageing data harder, trading query CPU on rarely-read data for storage.

A schema for the GPU telemetry table that respects all of the above:

```sql
CREATE TABLE gpu_metrics_1m
(
    ts             DateTime CODEC(DoubleDelta, ZSTD(1)),
    cluster        LowCardinality(String),
    hostname       LowCardinality(String),
    device_uuid    String  CODEC(ZSTD(1)),
    gpu_index      UInt8,
    namespace      LowCardinality(String),
    workload_id    String  CODEC(ZSTD(1)),
    sm_active_sum  Float32 CODEC(Gorilla, ZSTD(1)),   -- sum of samples (§5)
    sample_count   UInt16,                             -- and their count
    power_w_avg    Float32 CODEC(Gorilla, ZSTD(1)),
    xid            UInt16  CODEC(ZSTD(1))
)
ENGINE = MergeTree
PARTITION BY toDate(ts)              -- 1 value/day: ~365/yr, inside 100–1,000
ORDER BY (cluster, hostname, device_uuid, ts)  -- ascending cardinality; also the
                                               -- sort that makes ZSTD effective
TTL ts + INTERVAL 90 DAY DELETE,
    ts + INTERVAL 4  DAY RECOMPRESS CODEC(ZSTD(3))
SETTINGS ttl_only_drop_parts = 1;
```

Every clause is load-bearing: the partition key is low-cardinality and aligns with the TTL period
so whole parts drop; the sort order puts low-cardinality columns first (better compression, and
the prefix a query filters on); `LowCardinality(String)` is ClickHouse's dictionary encoding, the
same mechanism as Parquet's `RLE_DICTIONARY`; and the sum/count pair preserves re-aggregability.

### 12 · When *not* to build one

Say this out loud in an interview; it is a maturity signal. A lakehouse is a **data platform** —
Kafka, a stream job, compaction, snapshot expiry, a catalog, schema governance, and someone on
call for all of it. Do not build one if:

- Your retention question is answered by Thanos/Mimir downsampling (often it is).
- Your fleet is small enough that a single ClickHouse instance with 12-month retention does the
  whole job — **this is the right answer for most teams**, and it is the "start here" path.
- You have no consumer who writes SQL. A lakehouse with no analyst is a very expensive archive.
- You are justifying it on cost saving. Break-even 1 in §10 says you would need to remove ~4 M
  active series before it pays for its own floor.

The escalation ladder, with the trigger for each rung:

| Rung | Build when | Stop here if |
|---|---|---|
| 1 · Prometheus + recording rules | always | your questions are all "last few hours, pre-aggregated" |
| 2 · + Thanos/Mimir downsampling | you need months of trend at 5m/1h grain | nobody asks for per-pod or per-job history |
| 3 · **one ClickHouse** | you need per-pod/per-job history, arbitrary SQL, and joins | dataset < ~50 TB and one team consumes it |
| 4 · lakehouse (open table format) | multiple teams consume it, you must join to warehouse data, dataset is tens of TB+, or open-format portability is a requirement | — |

**Most teams should stop at rung 3. Knowing where to stop is the skill.**

## Perspectives

**The SRE's view.** Nothing changes on the hot path, and that is the design goal. The lakehouse
must never be in the alerting path, must never be a dependency of a dashboard someone opens at
3 a.m., and must be allowed to be down for hours. If it is load-bearing for on-call, it has been
built wrong — and the property you lose by coupling it is the ability to replay and restate, which
is exactly what makes it trustworthy as a ledger.

**The FinOps view.** This is the path that makes a dollar number defensible. It supports the three
things chargeback actually requires and PromQL cannot give: joins to a time-versioned rate card and
org chart, immutable sealed periods with an audit trail, and the ability to answer a question
nobody pre-aggregated. Module 11's FOCUS-shaped output is a table in this lake, and L9 §10's
misattribution window becomes a quantified, reconcilable line item here rather than a caveat.

**The data-platform view.** None of this is observability technology. It is Kafka, Parquet, a
table format, a catalog, and compaction — the same stack a data team already runs, with the same
failure modes (small files, missing compaction, unexpired snapshots, a partition key chosen for
joins). The productive framing is that **telemetry has become a new data domain**, which usually
means the right move is to reuse your company's existing lake rather than build a second one
inside the observability team.

**The economics view.** The two paths fail differently, and that is why you keep both. The hot
path's cost is **RAM as a function of cardinality, and it fails as an OOM** — a cliff, and one
that retention cannot move. The cold path's cost is **storage plus bytes scanned as a function of
retention, and it fails as a bill** — a slope. Because the drivers are orthogonal, the two-path
architecture is genuinely cheaper than either system stretched to cover both jobs. The runaway
risk moves to unpartitioned queries doing full-table scans and to over-partitioned writes doing
millions of PUTs, which is why partitioning and rollups are cost controls, not performance tuning.

**The format-neutrality view.** Parquet plus an open table format means the engine choice is
reversible: Trino today, ClickHouse reading Iceberg tomorrow, DuckDB on a laptop for a one-off.
That optionality is worth real money in a market where every vendor's pricing changes annually —
and it is the one argument for rung 4 that does not depend on scale at all.

## Real-world use cases

> *Vendor engineering blogs are blocked by the proxy in this environment. The entries below are
> cited from prior reading rather than fetched here, and figures attributed to vendor posts are
> the vendors' own claims — directional, not audited. The Parquet, Iceberg, ClickHouse and AWS
> pricing facts used throughout this lesson **were** verified from primary sources (see References).*

- **Cloudflare's billing-pipeline stall** — [a hidden ClickHouse query-plan bottleneck](https://blog.cloudflare.com/clickhouse-query-plan-contention/).
  A partitioning change on a petabyte-scale cluster caused lock contention in the query planner
  and stalled critical **billing** jobs. *What it shows:* exactly the failure mode that matters for
  your capstone — the analytics path *is* the billing path, so its partitioning scheme is a
  production concern, not a data-modelling detail. This is the single most relevant story here,
  and it is the operational counterpart to §7's and §11's partitioning arithmetic.
- **ClickHouse's own LogHouse platform** — [scaling observability past 100 PB](https://clickhouse.com/blog/scaling-observability-beyond-100pb-wide-events-replacing-otel).
  Reported ingesting over a million log lines/second and storing ~19 PiB over six months,
  compressed to ~1.13 PiB (≈17×). *What it shows:* compression ratio, not raw storage price, is
  what makes long-retention telemetry affordable — and it is a property of the columnar format
  and the sort order (§4), not of the vendor.
- **Cloudflare's logging pipeline** — [overview](https://blog.cloudflare.com/an-overview-of-cloudflares-logging-pipeline/)
  and the [tagged ClickHouse archive](https://blog.cloudflare.com/tag/clickhouse/). Over a hundred
  petabytes across dozens of clusters, after an Elasticsearch pipeline stopped keeping up with
  ingest. *What it shows:* the migration trigger is almost always **ingest**, not query.
- **The open-table-format argument, stated by a vendor with an interest in it** —
  [ClickHouse on lakehouses for observability](https://clickhouse.com/blog/lakehouses-path-to-low-cost-scalable-no-lockin-observability),
  describing the dual-write pattern (MergeTree hot, Iceberg/Delta cold). *What it shows:* the
  now-standard hot/cold split — and a useful exercise in reading a vendor's architecture argument
  while discounting for the fact that they sell the hot half.

## Worked example — `$/useful-GPU-hour` by team, for a quarter

The question Module 11 needs and PromQL cannot answer. Three joins do the work: a device
inventory, a **time-versioned** rate card, and the team mapping.

```sql
WITH util AS (
  SELECT
      date_trunc('day', ts)  AS day,
      namespace,
      device_uuid,
      -- Re-aggregable form (§5): sum the sums and the counts, divide once.
      -- This is exact regardless of how many devices were present for how
      -- long, which is the same correctness point module 05 makes about
      -- sum_over_time vs avg_over_time for GPU-hours.
      sum(sm_active_sum)     AS sm_active_sum,
      sum(sample_count)      AS samples,
      sum(sample_count) * 60.0 / 3600.0   AS allocated_hours  -- 1 sample = 60 s
  FROM workload_util_by_device_1m
  WHERE ts >= DATE '2026-04-01'          -- partition pruning: always filter the
    AND ts <  DATE '2026-07-01'          -- partition column, or you scan the lake
  GROUP BY 1, 2, 3
)
SELECT
    u.day,
    t.team,
    sum(u.allocated_hours)                                   AS allocated_gpu_hours,
    sum(u.sm_active_sum * 60.0 / 3600.0)                     AS useful_gpu_hours,
    sum(u.allocated_hours * r.usd_per_hour)                  AS spend_usd,
    sum(u.allocated_hours * r.usd_per_hour)
      / nullif(sum(u.sm_active_sum * 60.0 / 3600.0), 0)      AS usd_per_useful_gpu_hour
FROM util u
JOIN device_inventory d ON d.device_uuid = u.device_uuid
                       AND u.day BETWEEN d.valid_from AND d.valid_to   -- RMAs happen
JOIN team_mapping     t ON t.namespace   = u.namespace
                       AND u.day BETWEEN t.valid_from AND t.valid_to   -- people move
JOIN rate_card        r ON r.sku    = d.sku
                       AND r.region = d.region
                       AND u.day BETWEEN r.valid_from AND r.valid_to   -- the subtle one
GROUP BY 1, 2
ORDER BY usd_per_useful_gpu_hour DESC;
```

Four things to notice, because each is an interview answer:

1. **`allocated_hours` and `useful_gpu_hours` are different columns**, and their ratio is the
   allocated-vs-utilised spine of Modules 05 and 11. The lake is where that becomes a dollar.
2. **Every dimension join is time-versioned**, not just the rate card. Applying today's team map
   to April moves a whole team's spend to whoever inherited their namespace; applying today's
   inventory ignores every RMA. These are the errors nobody catches until a dispute.
3. **The utilisation aggregation is sum-of-sums over sum-of-counts**, not a mean of means, so it
   is exact even when devices enter and leave mid-window.
4. **`nullif(...)` guards the divide.** A team that allocated GPUs and used none has
   `useful_gpu_hours = 0` — infinitely bad efficiency, and a division by zero if you are careless.
   That row is the most interesting one on the report; do not let it crash the query.

**Sizing and cost check, end to end.**

```
   ROWS
     device-1m grain, 90 days: 32,000 × 1,440 × 90        = 4.147e9 rows
   BYTES ACTUALLY SCANNED (columnar projection + partition pruning)
     columns read: ts, namespace, device_uuid, sm_active_sum, sample_count
     encoded+ZSTD ≈ 6 B/row across those five columns
     4.147e9 × 6 B                                        ≈ 24.9 GB
   COST
     Athena at $5.00/TB scanned: 0.0249 TB × 5            ≈ $0.12 per run
     Requests: ~130 files/day × 90 days = 11,700 GETs
       11,700 / 1000 × $0.0004                            ≈ $0.005
   LATENCY  seconds to low minutes on a partitioned, compacted table.

   THE SAME QUESTION AGAINST RAW
     497e9 rows × ~6 B                                    ≈ 2.98 TB
     0.0249 TB → 2.98 TB is a 120× increase              ≈ $14.90 per run
   ⇒ The rollup is not an optimisation; it is the difference between a report
     someone runs daily and one they run once and never again.

   THE SAME QUESTION AGAINST AN UNPARTITIONED TABLE
     No partition pruning ⇒ every file's footer is read, then every row group.
     At 90 days of device-partitioned small files (§7's ✗ layout):
       2.88 M files ⇒ 2.88 M GETs ≈ $1.15 in requests alone, plus a planner
       that reads ~92 M manifest rows per day of range.
   ⇒ THE PARTITIONING CHOICE MOVES THE COST BY ~100×, TWICE, INDEPENDENTLY:
     once through bytes scanned, once through request count and planning time.
```

## Practice

Extend the **[fleet observability design](../practice/fleet-observability/README.md)** with a
fourth part — a two-path architecture, built against the
**[fake GPU fleet](../../../modules/04-gpu-on-kubernetes/practice/fake-gpu-fleet/README.md)** so
none of this needs real hardware:

1. **The architecture diagram + the seam.** One page: where the tee happens, the Kafka topic
   layout with a sized retention (events/s × bytes/event × days × RF), which labels the cold path
   carries that the hot path drops, and the explicit statement that the lake is never in the
   alerting path.
2. **The schema.** A `GpuSample` Protobuf/Avro record honouring the four rules (§4), a per-column
   Parquet encoding table with a one-line justification each, and the sort order you will write
   files in — plus the grain table (§5) with a question tied to each grain, and the sum/count pair
   that keeps every ratio re-aggregable.
3. **The physical layout.** State your partition spec (`day` plus one low-cardinality dimension),
   your target file size after compaction, your compaction cadence, and your snapshot-expiry
   policy — and derive the last one from your chargeback dispute window rather than a default.
4. **Run it small.** You do not need Kafka to learn this. Have the synthetic exporter also write
   Parquet to local disk partitioned by day, then query it with **DuckDB** — it reads Parquet and
   Iceberg directly and runs in-process. Reproduce the worked-example SQL end to end, then
   deliberately write a second copy partitioned by `device_uuid` and measure the difference in
   file count, planning time and query time. Report all three numbers.
5. **The cost model, both paths.** Hot-path cost for the label set you *would* have needed
   (series × $/series-month, with your own sizing), versus cold-path storage + scan for the same
   questions. Then compute both break-evens from §10 for your fleet: how many active series the
   lakehouse would have to remove to pay for its own floor, and at what dataset size ClickHouse
   loses to object storage.
6. **The sealing protocol.** Write it as a runbook: watermark, seal time, the checksum you store
   per partition, where late data goes, and the exact procedure for a restatement including the
   audit record. One paragraph on how you would reproduce last April's number after a restatement.
7. **The dimension tables.** `rate_card`, `team_mapping` and `device_inventory` as type-2 SCDs,
   plus the test that asserts non-overlapping, gap-free validity windows. State what a gap and an
   overlap each do to the output, and why neither is visible in the result.
8. **The stop-here recommendation.** Close with a paragraph naming where *this* fleet should stop
   on the escalation ladder (§12) and why, using your own break-even numbers. Recommending against
   the thing you just designed, with arithmetic, is the strongest page in the artifact.

**Acceptance criteria:** every cost figure traceable to a stated unit price and a stated
assumption; a partition spec justified by filter patterns rather than join keys; measured (not
assumed) numbers for the over-partitioned comparison in step 4; a sealing protocol with an
immutability invariant; and a stop-here recommendation with a break-even behind it.

## Common pitfalls

- **"We'll just extend retention in Prometheus."** Retention is disk; the wall is **RAM for active
  series**, and it does not move when you add disk. Symptom: a retention increase that does
  nothing for the question you actually had (per-pod history), because that question was never a
  retention problem. Mechanism: hot-path cost is a function of cardinality and is flat in
  retention — §2 and §10 make that explicit.
- **Ingest-time instead of event-time.** Works perfectly until the first network partition, then
  silently misattributes a node's GPU-hours to the wrong hour — and you find out during a
  chargeback dispute. Mechanism: the stamp records when the pipeline saw the event, not when it
  happened, so the error is proportional to backlog and is largest exactly when the fleet is
  unhealthy.
- **No compaction job.** The lakehouse works beautifully for three weeks and then queries slow to
  a crawl. Mechanism: a 30-second commit interval produces ~23 MB files against a spec
  recommendation of 512 MB–1 GB row groups, so the reader spends its time on footers and manifests
  instead of data.
- **No snapshot expiry.** Storage grows with your *compaction rate*, not your data volume, because
  every superseded file is still referenced by an old snapshot and cannot be deleted. Symptom: a
  table whose storage doubles while its row count is flat. Fix: `max-snapshot-age-ms` /
  `min-snapshots-to-keep`, set from your dispute window.
- **Over-partitioning.** `PARTITION BY device_uuid` feels right because you join on it. It creates
  32,000 partitions per day, ~$13.8k/month in PUT requests at a 30-second commit interval, and a
  planner that reads gigabytes of manifests to skip kilobytes of data. Partition on what you
  **filter** by (`day`, `cluster`), not what you **join** on — file-level lower/upper bounds
  handle the join key for free.
- **The same mistake in ClickHouse, arrived at differently.** A high-cardinality `PARTITION BY`
  produces parts that can never merge (merges do not cross partitions), and inserts eventually
  fail with "too many parts." Keep partition-key cardinality in the 100–1,000 range.
- **Un-batched inserts into ClickHouse.** Small, frequent inserts create parts faster than
  background merging can consume them. Symptom: rising part counts, then insert failures. Fix:
  batch to 10,000–100,000 rows at roughly one insert per second, or enable asynchronous inserts.
- **Averaging averages in a rollup.** Storing only the per-minute mean makes every downstream
  aggregation an approximation, and a wrong one whenever devices are present for different
  durations. Fix: store sum and count; divide once at the end.
- **Un-versioned dimension tables.** Applying today's rate card, team map or device inventory to
  last quarter is the most common silent error in home-grown chargeback, and it always
  under- or over-states one team specifically. Fix: type-2 SCDs with gap-free, non-overlapping
  validity windows, enforced by a test.
- **Building the lake before you have a consumer.** If no one writes SQL against it within a
  month, you built an archive with a Kafka cluster attached. Ship one report first, then the
  platform.
- **Letting it become load-bearing for on-call.** The moment an alert or a 3 a.m. dashboard depends
  on the cold path, you have taken a batch system's availability and put it in the incident path —
  and you have lost the freedom to replay and restate, which is what made it trustworthy.

## Self-check

- **Why can't you answer *"$/useful-GPU-hour by team for last quarter"* from Prometheus, even with
  Thanos giving you a year of retention?** — Two independent reasons. **Cardinality:** the question
  needs `namespace`/`workload_id`/`device_uuid` as dimensions, and keeping those as metric labels
  across 32,000 GPUs blows the active-series budget (L1/L3); you dropped them on purpose, and the
  cost is per-series-per-month whether or not anyone queries them. **Joins:** the dollar figure
  requires joining to a time-versioned rate card, a device inventory and an org chart, none of
  which are time series, and PromQL has no join to non-series data. Long retention fixes neither;
  it only addresses the third, smaller problem (downsampling loses fidelity).

- **Why is a high-cardinality dimension almost free in Parquet and ruinous in a TSDB? Give the
  mechanism and an order-of-magnitude.** — In a TSDB each distinct label combination is a distinct
  *object*: a `memSeries` struct, a packed label string, an open head chunk and postings entries,
  ~0.5–1.4 KB of RAM each, allocated at ingest regardless of queries. In Parquet the distinct
  values are written once into a dictionary page (required to be the chunk's first page, at most
  one per chunk) and each row stores a bit-packed, run-length-encoded index — bytes per *row*,
  not an object per *value*, and zero bytes read if the query does not project that column. For a
  200-value dimension on a 1 M-series fleet the hot path implies 200 M active series (≈$154k/month
  at $0.00077 per series-month, and in practice an OOM), versus ~166 GB/month ≈ $3.82 in object
  storage — roughly a 40,000× ratio.

- **Your Iceberg table is partitioned by `device_uuid` because every query joins on it. Query
  planning is now slower than execution and your S3 bill has a large request line. What went
  wrong, what is the rule, and what is the cost?** — 32,000 partitions × two commits per minute is
  64,000 files/minute — 2.76 × 10⁹ PUTs/month ≈ $13,800/month in request charges alone at
  $0.005/1,000, plus ~92 M manifest rows per day for the planner to read in order to skip kilobytes
  of data. The rule is **partition by what you filter on** (`day`, plus a low-cardinality
  `cluster`/`region`), **not by what you join on**: the manifest's per-file `lower_bounds` /
  `upper_bounds` prune on the join key for free. Repartitioning to `day, cluster` with compaction
  takes the same workload to ~$2/month in requests and ~130 files/day.

- **What does a partition-sealing protocol have to guarantee, and how do you serve "reproduce the
  number we billed in April" after a restatement?** — It must guarantee that a sealed partition is
  never mutated in place: after the watermark expires, compact, store a checksum (row count, sum of
  sample counts, distinct device count, min/max ts), publish a version, and route anything later
  into quarantine rather than dropping it. A correction is a *new version* plus an audit record
  naming the reason and the per-team delta. The April figure remains answerable because the prior
  snapshot is still referenced — which means your snapshot-expiry policy
  (`max-snapshot-age-ms` / `min-snapshots-to-keep`) must be set from the chargeback dispute window,
  not left at a default.

- **The scan math says a host-1m rollup answers "daily host utilisation over 90 days" with ~960×
  less data than raw. So why keep raw at all, and what must the rollup store to be trustworthy?**
  — Rollups only answer questions their grain anticipated; raw is for **forensics** —
  reconstructing exactly what one device did during a specific incident at 15-second resolution,
  including dimensions no rollup carries. The resolution is asymmetric retention: raw for days,
  rollups for years (keeping raw for years is the expensive mistake; keeping none is the
  un-debuggable one). And the rollup must store `sum_of_samples` and `sample_count`, not just a
  mean, so downstream aggregation is `sum(sums)/sum(counts)` — exact regardless of which devices
  were present for which part of the window.

- **An interviewer asks you to design telemetry analytics for a 200-node GPU fleet. What is the
  correct answer, and what break-even backs it?** — Almost certainly *not* a lakehouse. Start with
  Prometheus plus Thanos/Mimir downsampling; if that cannot answer the FinOps questions, put one
  ClickHouse instance with 12-month retention behind the same telemetry stream, partitioned by day
  with `ttl_only_drop_parts = 1`. The break-even: a lakehouse's fixed floor (Kafka, a stream job,
  compaction, a catalog, an on-call owner) is on the order of $3,000/month before any data, while
  the hot-path saving from removing series is ~$0.00077 per active series-month — so it would need
  to remove roughly 4 million active series to pay for itself on cost alone. It is justified by
  questions that cannot otherwise be answered, by multiple consuming teams, and by open-format
  portability — none of which a 200-node fleet has yet. Naming the ladder and where this fleet
  stops on it *is* the answer; jumping to the biggest architecture is the trap.

- **Why is the two-path architecture genuinely cheaper than one system doing both jobs, rather
  than just being two bills?** — Because the two systems' cost drivers are **orthogonal**. The hot
  path is priced in RAM per active series and is essentially flat in retention; the cold path is
  priced in bytes stored and bytes scanned and is essentially flat in cardinality. Forcing the hot
  path to carry accounting dimensions multiplies its only expensive axis; forcing the cold path to
  serve alerts means paying for latency it is not built to deliver and coupling on-call to a batch
  system. Splitting lets each system be cheap along the axis it is cheap on, and the seam (Kafka)
  costs a fixed, sizeable, but *bounded* amount — which is exactly what the break-even in §10
  makes you check before you build it.

## Connections & what's next

This closes the module by naming the boundary of everything before it: L1–L9 make the **hot path**
correct and affordable, and this lesson says what to do with the questions that path deliberately
cannot serve. The cardinality budget from L1 and the sizing work from L3 are what *create* the need
for this lesson — read it as the other half of that trade, not a new topic. L9's join-key spine
determines which columns exist here at all, and L9 §10's attribution-cache misattribution window is
the specific error this path exists to quantify and reconcile.

Downstream, this is load-bearing for
**[modules/11-gpu-cost-economics](../../../modules/11-gpu-cost-economics/README.md)**: the
allocated-vs-utilised attribution, the FOCUS-shaped output (M11 L10), and every chargeback number
are tables and queries in this lake. It also feeds
**[modules/12-capstone-interview](../../../modules/12-capstone-interview/README.md)** — the
flagship controller emits the hot-path metrics, but its per-team monthly report comes from here.
And it pairs with **[modules/05-gpu-observability](../../../modules/05-gpu-observability/README.md)**,
whose DCGM semantics define what `sm_active` in the worked example actually means, and whose
GPU-hours integral is the same correctness argument this lesson makes with sums and counts.

There is no lesson 11. What's next is the **[deliverable](../practice/fleet-observability/README.md)**
and the **[checkpoint](../checkpoint.md)** — item 7 is this lesson stated as a pass criterion.

## References & further reading

**Primary sources (read from upstream repositories and APIs; the rendered documentation sites are unreachable from this environment)**
- Apache Parquet — format specification: `apache/parquet-format`, `README.md`. *Verified: the file layout (magic `PAR1`, row groups → column chunks → pages, file metadata written last in Thrift `TCompactProtocol`), the glossary definitions, the recommended **512 MB–1 GB row groups** and **~8 KB data pages**, the requirement that a dictionary page be first and at most one per column chunk, the optional column/offset index for page-level skipping, per-page CRC32 checksums, and the error-recovery granularity table.*
- Apache Parquet — `Encodings.md`. *Verified encoding identifiers and semantics used in §4: `PLAIN` (0), `PLAIN_DICTIONARY` (2) / `RLE_DICTIONARY` (8), `RLE` (3), `DELTA_BINARY_PACKED` (5), `DELTA_LENGTH_BYTE_ARRAY` (6), `DELTA_BYTE_ARRAY` (7), `BYTE_STREAM_SPLIT` (9).*
- Apache Parquet — `Compression.md`. *Verified the codec list: `UNCOMPRESSED`, `SNAPPY`, `GZIP`, `LZO`, `BROTLI`, `LZ4`, `ZSTD`, `LZ4_RAW`.*
- Apache Iceberg — table specification: `apache/iceberg`, `format/spec.md`. *Verified: the metadata → snapshot → manifest list → manifest → data file tree; `manifest_entry` fields (`status` 0/1/2, `snapshot_id`, `sequence_number`, `file_sequence_number`) and the `data_file` struct (`record_count`, `file_size_in_bytes`, `column_sizes`, `value_counts`, `null_value_counts`, `nan_value_counts`, `lower_bounds`, `upper_bounds`, `split_offsets`, `sort_order_id`); the partition-transform table including `bucket[N] = (murmur3_x86_32(v) & Integer.MAX_VALUE) % N` and the `truncate[W]` rules; scan planning via inclusive projection; partition evolution semantics; and the snapshot-retention properties `min-snapshots-to-keep`, `max-snapshot-age-ms`, `max-ref-age-ms`.*
- Delta Lake — protocol specification: `delta-io/delta`, `PROTOCOL.md`. *Consulted as the main alternative table format; its transaction-log design contrasts usefully with Iceberg's manifest tree. Not relied upon for any figure in this lesson.*
- ClickHouse — `ClickHouse/clickhouse-docs`, `docs/best-practices/partitioning_keys.mdx`. *Verified: partitioning is primarily a data-management technique; merges never cross partitions; the "too many parts" error with `parts_to_throw_insert` / `max_parts_in_total`; the recommendation of a **low-cardinality partitioning key with fewer than 100–1,000 distinct values**; and automatic MinMax indexes on partition columns.*
- ClickHouse — `docs/best-practices/selecting_an_insert_strategy.md` and `_snippets/_bulk_inserts.md`. *Verified: batch **at least 1,000 rows, ideally 10,000–100,000**, roughly **one insert query per second**; insert deduplication making synchronous inserts idempotent on retry; asynchronous inserts as the server-side alternative.*
- ClickHouse — `docs/use-cases/observability/build-your-own/managing-data.md`. *Verified: table- and column-level TTL; the recommendation to use **`ttl_only_drop_parts = 1`** and to align the TTL period with the partition period; **`merge_with_ttl_timeout` default 14,400 s (4 hours)**; `ALTER TABLE … MATERIALIZE TTL`; `ZSTD(1)` as the general observability recommendation with `TTL … RECOMPRESS CODEC(ZSTD(3))` for ageing data; and TTL-driven movement between storage tiers.*
- AWS Price List API — `AmazonS3`, `AmazonEC2` and `AmazonAthena`, `us-east-1`, retrieved 2026-08-18. *Verified every price in §7 and §10: S3 Standard $0.023 / $0.022 / $0.021 per GB-month by tier; Standard-IA $0.0125 + $0.01/GB retrieval; Glacier Instant Retrieval $0.004 + $0.03/GB retrieval; requests PUT/COPY/POST/LIST $0.005 per 1,000 and GET $0.0004 per 1,000 (IA $0.01 / $0.001; GIR $0.02 / $0.01); EBS gp3 $0.08/GB-month; `r7i.4xlarge` $1.0584/hour and `r7i.8xlarge` $2.1168/hour on-demand Linux; Athena $5.00 per TB scanned.*
- Apache Kafka — design documentation: https://kafka.apache.org/documentation/#design — *read for the durability/replay properties that make it the right seam. Not fetched in this environment; no figure in this lesson depends on it beyond the sizing arithmetic, which uses your own event rate.*
- OpenTelemetry Collector — https://opentelemetry.io/docs/collector/ — *the exporter fan-out that implements the tee; L4's territory. Not fetched here.*
- Prometheus remote-write specification — https://prometheus.io/docs/specs/remote_write_spec/ — *the alternative tee point and its limits. Not fetched here (the site is proxy-blocked); L3's territory.*
- DuckDB documentation — https://duckdb.org/docs/ — *for the practice task: Parquet and Iceberg querying in-process, no cluster. Not fetched here.*
- FOCUS billing specification — https://focus.finops.org/ — *the output schema Module 11 targets; the lake is where you produce it. Not fetched here.*

**Real-world engineering blogs** *(cited from prior reading, not fetched — proxy-blocked in this environment; vendor figures are the vendors' own claims and are not relied upon for any calculation above)*
- Cloudflare, "Our billing pipeline was suddenly slow: a hidden bottleneck in ClickHouse" — https://blog.cloudflare.com/clickhouse-query-plan-contention/
- Cloudflare, "An overview of Cloudflare's logging pipeline" — https://blog.cloudflare.com/an-overview-of-cloudflares-logging-pipeline/
- ClickHouse, "Scaling our observability platform beyond 100 PB" — https://clickhouse.com/blog/scaling-observability-beyond-100pb-wide-events-replacing-otel
- ClickHouse, "Are open table formats + lakehouses the future of observability?" — https://clickhouse.com/blog/lakehouses-path-to-low-cost-scalable-no-lockin-observability

**Deeper dives**
- `harut8/system-design`, "35 — Telemetry Lakehouse" — https://github.com/harut8/system-design/blob/main/sre-observability/35-telemetry-lakehouse.md — *the general-purpose version of this lesson: formats, engines, schema-on-read vs write, anti-patterns.*
- `harut8/system-design`, "17 — Telemetry Lakehouse & SQL Analytics" — https://github.com/harut8/system-design/blob/main/gpu-observability/17-telemetry-lakehouse-and-sql-analytics.md — *the GPU-specific version: the Kafka contract, the `GpuSample` schema, and the rollup grains §3–§5 are built on.*
- Grafana Mimir documentation — https://grafana.com/docs/mimir/latest/ — *read to establish where the hot path's downsampling genuinely suffices, so you can defend not building a lake. The per-component sizing constants used in §10's Tier 1 derivation come via lesson 03.*
