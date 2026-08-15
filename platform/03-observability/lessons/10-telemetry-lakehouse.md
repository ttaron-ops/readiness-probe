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
sources: 12
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
enrichment (L4), traces and exemplars (L5), logs (L6), burn-rate SLOs (L7), profiling (L8), and
the fleet-wide GPU synthesis (L9). Every one of those optimises the same thing: **answering a
question about the last few hours, in under a second, without going bankrupt on active series.**

That optimisation has a price, and L1 and L3 already named it — you dropped `pod`, `workload_id`
and `gpu_uuid` from your metric labels to stay inside the cardinality budget. This lesson is
where that bill comes due. The labels you correctly refused to keep are exactly the labels a
cost question needs to join on. The fix is not a bigger Prometheus; it is a **second path with
different physics**.

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
someone will ask you to reproduce a specific team's number for a specific week six months ago.

## What's new here (calibration)

- You already know Prometheus/Thanos/Mimir, remote-write, downsampling, and why cardinality kills
  a metrics system (L1, L3) — none of that is re-taught. You also already know the FOCUS billing
  spec from Module 11 L10.
- **New: the two-path model as an architecture**, not a workaround — what each path is physically
  good at, and the *seam* (Kafka) that lets them share one source without coupling.
- **New: columnar table formats** — Parquet plus Iceberg/Delta/Hudi, the catalog, ACID over object
  storage, and the small-files/compaction problem that is the number-one way these get slow.
- **New: event-time and late data.** Prometheus quietly ignores this. An accounting ledger cannot.
- **New: the cost calculus.** The hot path is priced in **RAM per active series**; the cold path is
  priced in **storage plus bytes scanned**. Different failure modes, different levers.
- **New: when *not* to build one** — the honest answer for most fleets, and a strong interview signal.

## Core concepts

### 1 · Two paths, one source

```
                       dcgm-exporter / OTel Collector
                                   │
                ┌──────────────────┴──────────────────┐
                ▼                                     ▼
        HOT PATH (L1–L9)                      COLD PATH (this lesson)
   Prometheus → Thanos/Mimir              Kafka → stream job → Parquet/Iceberg
        → Grafana, alerts                      → Trino / ClickHouse / BigQuery
                │                                     │
       "is it on fire now?"                "what did it cost last quarter?"
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
| User | on-call SRE, alert rules | FinOps, capacity, leadership |

> **The mental model.** Prometheus is a **circular buffer with math**. The lakehouse is a
> **ledger with joins**. Don't ask the buffer to do accounting; don't ask the ledger to page you.

The five question classes the hot path structurally cannot serve: **long history** (beyond
retention, or lost to downsampling), **complex SQL** (cross-signal, multi-window), **joins to
business data** (org chart, rate card, model registry), **ad-hoc analytical** (asked by people who
write SQL, not PromQL), and **forensic/compliance** (reproduce this exact number from eight months
ago). Every one of those is a Module 11 question.

### 2 · The tee, and why the seam is Kafka

You do **not** fork the scrape. You tee **after** collection, and the tee point is a durable log:

1. `dcgm-exporter` / the OTel Collector emits as normal — the hot path is untouched.
2. A sidecar or Collector exporter also publishes a **richer** event stream to Kafka.
3. A stream job (Flink/Spark/Beam, or plain consumers) writes columnar files to object storage.

Kafka earns its place for three reasons: it **decouples** producer rate from writer rate (the
lake writer can be down for an hour without losing telemetry), it lets **multiple consumers** fan
out from one publish, and it gives you **replay** — when you find a bug in the rollup job, you
reprocess from the log instead of losing the quarter.

This is also the one place the cold path is *allowed* to diverge from the hot path: it can carry
`pod`, `workload_id` and `gpu_uuid` — **the labels the hot path had to drop.** That divergence is
the entire point.

**Topic layout** worth copying:

```
gpu_telemetry_raw          # every DCGM field, per device, every scrape
gpu_telemetry_rollup_1m    # pre-aggregated (see §4)
gpu_workload_events        # pod/job lifecycle joined to GPU UUID
gpu_hardware_events        # XID, ECC retirement, fabric errors — low-rate, high-value
```

Hardware events get their own topic precisely because they are low-rate: you do not want an XID
48 stuck behind a two-hour backlog of utilisation samples.

### 3 · Schema rules that save you a backfill

Use Avro or Protobuf with a schema registry. JSON at this volume is a tax on storage and on every
consumer's CPU. Three rules matter more than the rest:

1. **`device_uuid` is the join key — not `hostname` + `gpu_index`.** Hosts get reimaged and GPU
   indices shift underneath you; UUIDs survive. Carry both, join on the UUID.
2. **`ts_ms` is event-time, set by the exporter** — never Kafka ingest-time. An accounting record
   stamped with when it happened to arrive is not an accounting record.
3. **`workload_id` is opaque to the exporter.** The sidecar copies it from a pod label; the lake
   does not care what it means, only that it is stable.

**Partition key:** `hash(cluster + ":" + hostname)`. This keeps a host's samples ordered in one
partition so per-host rollups are cheap. Keying by `device_uuid` is the tempting mistake — it 8×'s
your partition count and destroys host locality for no ordering gain.

### 4 · Pre-aggregation, and the scan math that forces it

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

| Table | Grain | Typical question |
|---|---|---|
| `gpu_metrics_raw` | (device, 15s) | forensic drill-down; keep days, not months |
| `workload_util_by_device_1m` | (device, workload, minute) | per-job efficiency, chargeback |
| `workload_util_by_host_1m` | (host, minute) | host saturation, stranding |
| `workload_util_by_namespace_1m` | (cluster, namespace, minute) | team-level utilisation |
| `workload_util_by_cluster_5m` | (cluster, 5 min) | fleet capacity, forecasting |

Pick the grains from the questions you must answer, then **retain raw shorter than rollups** —
inverted from most people's instinct, and the single biggest cost lever here.

### 5 · Table formats: Parquet is the file, Iceberg is the table

**Parquet** (or ORC) is the columnar *file* format: column-major layout, per-column compression,
and statistics per row group so an engine can skip entire files. Telemetry compresses
extraordinarily well because adjacent values repeat — vendors routinely report 10–20× on log and
metric data.

**Iceberg / Delta / Hudi** are *table* formats layered on top. They add what a pile of Parquet
files lacks: an atomic manifest (so a reader never sees a half-written commit), schema evolution,
partition evolution, hidden partitioning, and time travel. A **catalog** (Glue, Hive, Iceberg
REST, Unity) maps table names to those manifests.

Two operational realities decide whether yours is fast:

- **The small-files problem.** A stream job committing every 30 seconds produces thousands of tiny
  files a day; query planning then spends more time listing files than reading data. You need a
  **compaction** job. This is the most common way a working lakehouse becomes a slow one.
- **Partitioning is the whole ballgame.** Partition by `day` plus a low-cardinality dimension you
  always filter on (`cluster`, `region`). Partitioning by `device_uuid` recreates the small-files
  problem with extra steps.

**Engines** read the same tables: Trino/Presto (federated, ad-hoc), Spark (heavy batch),
ClickHouse (fastest interactive, can now read *and* write Iceberg/Delta), BigQuery/Snowflake
(external tables, if your warehouse already lives there). The point of the open format is that
this choice is reversible — the data is not locked in an engine's proprietary store.

### 6 · Event-time, watermarks, and late data

A GPU node partitions off for ten minutes and its samples arrive late. Three questions you must
answer explicitly, because silence here is how ledgers go wrong:

- **How late is acceptable?** A watermark — say 30 minutes — after which a partition is sealed.
- **What happens to later-than-that data?** A quarantine table you can reconcile from, never a
  silent drop, and never an unbounded window (which prevents the job from ever finalising).
- **Is a sealed day immutable?** For chargeback it must be. Restatements get a new version and an
  audit note, exactly like finance.

### 7 · The join that makes it worth building

The cold path's superpower is joining telemetry to data that is not telemetry: the **org chart**
(namespace → team → cost centre), the **rate card** (SKU × region × time → $/hour, including
committed-use tiers), the **device inventory** (UUID → SKU, node, NVLink island), and the
**model registry**. None of these live in Prometheus, and all of them are required to say a
dollar number out loud.

Time-versioning the rate card is the subtle part: rates change, and a query over Q2 must apply
the rate that was in effect on each day, not today's. That is a `BETWEEN valid_from AND valid_to`
join — trivial in SQL, impossible in PromQL.

### 8 · When *not* to build one

Say this out loud in an interview; it is a maturity signal. A lakehouse is a **data platform** —
Kafka, a stream job, compaction, a catalog, schema governance, and someone on call for all of it.
Do not build one if:

- Your retention question is answered by Thanos/Mimir downsampling (often it is).
- Your fleet is small enough that a single ClickHouse instance with 12-month retention does the
  whole job — **this is the right answer for most teams**, and it is the "start here" path.
- You have no consumer who writes SQL. A lakehouse with no analyst is a very expensive archive.

The escalation ladder is: **Prometheus → +Thanos/Mimir downsampling → one ClickHouse →
lakehouse with open table formats.** Most teams should stop at step three. Knowing *where* to
stop is the skill.

## Perspectives

**The SRE's view.** Nothing changes on the hot path, and that is the design goal. The lakehouse
must never be in the alerting path, must never be a dependency of a dashboard someone opens at
3 a.m., and must be allowed to be down for hours. If it is load-bearing for on-call, it has been
built wrong.

**The FinOps view.** This is the path that makes a dollar number defensible. It supports the three
things chargeback actually requires and PromQL cannot give: joins to the rate card and org chart,
immutable sealed periods, and the ability to answer a question nobody pre-aggregated. Module 11's
FOCUS-shaped output is a table in this lake.

**The data-platform view.** None of this is observability technology. It is Kafka, Parquet, a
table format, a catalog, and compaction — the same stack a data team already runs. The productive
framing is that **telemetry has become a new data domain**, which usually means the right move is
to reuse your company's existing lake rather than build a second one inside the observability team.

**The economics view.** The two paths fail differently, and that is why you keep both. The hot
path's cost is **RAM, and it fails as an OOM** — a cliff. The cold path's cost is **storage plus
bytes scanned, and it fails as a bill** — a slope. Object storage at a few cents per GB-month
makes multi-year retention genuinely cheap; the runaway risk moves to unpartitioned queries doing
full-table scans, which is why partitioning and rollups are cost controls, not performance tuning.

## Real-world use cases

> *The proxy in this environment blocks these hosts, so the URLs below are cited from search
> results and my own reading history rather than fetched here. Figures attributed to vendor posts
> are the vendors' own claims — treat them as directional and verify before quoting.*

- **ClickHouse's own LogHouse platform** — [scaling observability past 100 PB](https://clickhouse.com/blog/scaling-observability-beyond-100pb-wide-events-replacing-otel).
  Reported ingesting over a million log lines/second and storing ~19 PiB over six months,
  compressed to ~1.13 PiB (≈17×). *What it shows:* compression ratio, not raw storage price, is
  what makes long-retention telemetry affordable — and it is a property of the columnar format.
- **Cloudflare's logging pipeline** — [overview](https://blog.cloudflare.com/an-overview-of-cloudflares-logging-pipeline/)
  and the [tagged ClickHouse archive](https://blog.cloudflare.com/tag/clickhouse/). Over a hundred
  petabytes across dozens of clusters, after an Elasticsearch pipeline stopped keeping up with
  ingest. *What it shows:* the migration trigger is almost always **ingest**, not query.
- **Cloudflare's billing-pipeline stall** — [a hidden ClickHouse query-plan bottleneck](https://blog.cloudflare.com/clickhouse-query-plan-contention/).
  A partitioning change on a petabyte-scale cluster caused lock contention in the query planner
  and stalled critical **billing** jobs. *What it shows:* exactly the failure mode that matters for
  your capstone — the analytics path *is* the billing path, so its partitioning scheme is a
  production concern, not a data-modelling detail. This is the single most relevant story here.
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
      -- rows are 1-minute means of a 15s-sampled ratio, so a mean of means
      -- over equal windows is correct here; it would not be over ragged ones.
      avg(sm_active)         AS mean_sm_active,
      count(*) / 60.0        AS allocated_hours
  FROM workload_util_by_device_1m
  WHERE ts >= DATE '2026-04-01'          -- partition pruning: always filter the
    AND ts <  DATE '2026-07-01'          -- partition column, or you scan the lake
  GROUP BY 1, 2, 3
)
SELECT
    u.day,
    t.team,
    sum(u.allocated_hours)                            AS allocated_gpu_hours,
    sum(u.allocated_hours * u.mean_sm_active)         AS useful_gpu_hours,
    sum(u.allocated_hours * r.usd_per_hour)           AS spend_usd,
    sum(u.allocated_hours * r.usd_per_hour)
      / nullif(sum(u.allocated_hours * u.mean_sm_active), 0)
                                                      AS usd_per_useful_gpu_hour
FROM util u
JOIN device_inventory d ON d.device_uuid = u.device_uuid
JOIN team_mapping     t ON t.namespace   = u.namespace
JOIN rate_card        r ON r.sku    = d.sku
                       AND r.region = d.region
                       AND u.day BETWEEN r.valid_from AND r.valid_to   -- the subtle one
GROUP BY 1, 2
ORDER BY usd_per_useful_gpu_hour DESC;
```

Three things to notice, because each is an interview answer:

1. **`allocated_hours` and `useful_gpu_hours` are different columns**, and their ratio is the
   allocated-vs-utilised spine of Modules 05 and 11. The lake is where that becomes a dollar.
2. **The rate-card join is time-versioned.** Applying today's rate to April is the most common
   silent error in home-grown chargeback.
3. **`nullif(...)` guards the divide.** A team that allocated GPUs and used none has
   `useful_gpu_hours = 0` — infinitely bad efficiency, and a division by zero if you are careless.
   That row is the most interesting one on the report; do not let it crash the query.

**Sizing sanity check.** Ninety days at device-1m grain is
`32,000 × 1,440 × 90 ≈ 4.1B rows`. With `day` partitioning and column pruning the engine reads a
handful of columns from ~90 partitions — seconds to low minutes, and a few hundred dollars a month
of object storage. The same question against raw is a 497B-row scan you should never issue.

## Practice

Extend the **[fleet observability design](../practice/fleet-observability/README.md)** with a
fourth part — a two-path architecture, built against the
**[fake GPU fleet](../../../modules/04-gpu-on-kubernetes/practice/fake-gpu-fleet/README.md)** so
none of this needs real hardware:

1. **The architecture diagram + the seam.** One page: where the tee happens, the Kafka topic
   layout, which labels the cold path carries that the hot path drops, and the explicit statement
   that the lake is never in the alerting path.
2. **The schema.** A `GpuSample` Protobuf/Avro record honouring the three rules (§3), plus the
   grain table (§4) with a one-line justification per grain tied to a question you must answer.
3. **Run it small.** You do not need Kafka to learn this. Have the synthetic exporter also write
   Parquet to local disk partitioned by day, then query it with **DuckDB** — it reads Parquet and
   Iceberg directly and runs in-process. Reproduce the worked-example SQL end to end.
4. **The cost model.** Hot-path RAM for the label set you *would* have needed, versus cold-path
   storage + scan for the same questions. Two numbers, stated, with assumptions.
5. **The stop-here recommendation.** Close with a paragraph naming where *this* fleet should stop
   on the escalation ladder (§8) and why. Recommending against the thing you just designed, with
   a reason, is the strongest sentence in the artifact.

## Common pitfalls

- **"We'll just extend retention in Prometheus."** Retention is disk; the wall is **RAM for active
  series**, and it does not move when you add disk. This conflation is L3's lesson resurfacing.
- **Ingest-time instead of event-time.** Works perfectly until the first network partition, then
  silently misattributes a node's GPU-hours to the wrong hour — and you find out during a
  chargeback dispute.
- **No compaction job.** The lakehouse works beautifully for three weeks and then queries slow to
  a crawl. It is thousands of tiny files, every time.
- **Over-partitioning.** `PARTITION BY device_uuid` feels right because you join on it. It creates
  32,000 partitions per day, and the planner spends longer listing files than reading them.
  Partition on what you **filter** by (`day`, `cluster`), not what you **join** on.
- **Building the lake before you have a consumer.** If no one writes SQL against it within a
  month, you built an archive with a Kafka cluster attached. Ship one report first, then the
  platform.
- **Letting it become load-bearing for on-call.** The moment an alert or a 3 a.m. dashboard depends
  on the cold path, you have taken a batch system's availability and put it in the incident path.

## Self-check

- Why can't you answer *"$/useful-GPU-hour by team for last quarter"* from Prometheus, even with
  Thanos giving you a year of retention? **Answer:** Two independent reasons. **Cardinality** — the
  question needs `namespace`/`workload_id`/`device_uuid` as dimensions, and keeping those as metric
  labels across 32,000 GPUs blows the active-series budget (L1/L3); you dropped them on purpose.
  **Joins** — the dollar figure requires joining to a time-versioned rate card, a device inventory,
  and an org chart, none of which are time series, and PromQL has no join to non-series data.
  Long retention fixes neither; it only fixes the third, smaller problem (downsampling loses
  fidelity).
- The scan math says a host-1m rollup answers "daily host utilisation over 90 days" with ~960×
  less data than raw. So why keep raw at all? **Answer:** Because rollups only answer the questions
  their grain anticipated. Raw is for **forensics** — reconstructing exactly what one device did
  during a specific incident, at 15-second resolution, including dimensions no rollup carries.
  The resolution is asymmetric retention: keep raw for days and rollups for years. Keeping raw for
  years is the expensive mistake; keeping none is the un-debuggable one.
- Your stream job partitions the table by `device_uuid` because every query joins on it. Query
  planning is now slower than query execution. What went wrong, and what is the rule? **Answer:**
  32,000 devices × daily commits produces an enormous number of small partitions and files, so the
  engine spends its time listing metadata rather than reading data — the small-files problem. The
  rule is **partition by what you filter on** (`day`, plus a low-cardinality `cluster`/`region`),
  **not by what you join on**; column statistics and file-level pruning handle the join key
  efficiently without partitioning by it. And run compaction regardless.
- Why must the lakehouse be allowed to be down for two hours, and what does that constraint buy
  you? **Answer:** Because it is a batch accounting system, not an incident-response system —
  keeping it off the alerting path is what lets you run it on cheap object storage, tolerate late
  data with watermarks, and reprocess from Kafka when the rollup job has a bug. If an on-call
  dashboard depends on it, you have coupled a 3 a.m. page to a system with batch-grade
  availability, and you lose the freedom to replay and restate — the exact properties that make it
  a trustworthy ledger.
- An interviewer asks you to design telemetry analytics for a 200-node GPU fleet. What is the
  correct answer? **Answer:** Almost certainly *not* a lakehouse. Start with Prometheus plus
  Thanos/Mimir downsampling; if that can't answer the FinOps questions, put one ClickHouse instance
  with 12-month retention behind the same telemetry stream. A full Kafka + Iceberg + compaction +
  catalog platform is justified by fleet scale, multiple consuming teams, and a real need to join
  to warehouse data — none of which a 200-node fleet has yet. Naming the escalation ladder and
  where this fleet stops on it is the answer; jumping to the biggest architecture is the trap.

## Connections & what's next

This closes the module by naming the boundary of everything before it: L1–L9 make the **hot path**
correct and affordable, and this lesson says what to do with the questions that path deliberately
cannot serve. The cardinality budget from L1 and the sizing work from L3 are what *create* the need
for this lesson — read it as the other half of that trade, not a new topic.

Downstream, this is load-bearing for
**[modules/11-gpu-cost-economics](../../../modules/11-gpu-cost-economics/README.md)**: the
allocated-vs-utilised attribution, the FOCUS-shaped output (M11 L10), and every chargeback number
are tables and queries in this lake. It also feeds
**[modules/12-capstone-interview](../../../modules/12-capstone-interview/README.md)** — the
flagship controller emits the hot-path metrics, but its per-team monthly report comes from here.
And it pairs with **[modules/05-gpu-observability](../../../modules/05-gpu-observability/README.md)**,
whose DCGM semantics define what `sm_active` in the worked example actually means.

There is no lesson 11. What's next is the **[deliverable](../practice/fleet-observability/README.md)**
and the **[checkpoint](../checkpoint.md)** — item 7 is this lesson stated as a pass criterion.

## References & further reading

**Primary sources**
- Apache Parquet documentation — https://parquet.apache.org/docs/ — *read for the columnar layout, row groups, and the statistics that make file pruning work.*
- Apache Iceberg table spec — https://iceberg.apache.org/spec/ — *read for manifests, snapshots, hidden partitioning, and schema/partition evolution.*
- Delta Lake documentation — https://docs.delta.io/latest/index.html — *read as the main alternative table format; the transaction-log design contrasts usefully with Iceberg's manifests.*
- Apache Kafka design documentation — https://kafka.apache.org/documentation/#design — *read for the durability/replay properties that make it the right seam.*
- OpenTelemetry Collector — https://opentelemetry.io/docs/collector/ — *read for the exporter fan-out that implements the tee.*
- Prometheus remote-write specification — https://prometheus.io/docs/specs/remote_write_spec/ — *read for the alternative tee point, and its limits.*
- DuckDB documentation — https://duckdb.org/docs/ — *read for the practice task: Parquet and Iceberg querying in-process, no cluster.*
- FOCUS billing specification — https://focus.finops.org/ — *the output schema Module 11 targets; the lake is where you produce it.*

**Real-world engineering blogs** *(cited, not fetched — proxy-blocked in this environment)*
- Cloudflare, "Our billing pipeline was suddenly slow: a hidden bottleneck in ClickHouse" — https://blog.cloudflare.com/clickhouse-query-plan-contention/ — *what it shows: partitioning choices on the analytics path are a billing-availability concern.*
- Cloudflare, "An overview of Cloudflare's logging pipeline" — https://blog.cloudflare.com/an-overview-of-cloudflares-logging-pipeline/ — *what it shows: ingest, not query, is what forces the migration.*
- ClickHouse, "Scaling our observability platform beyond 100 PB" — https://clickhouse.com/blog/scaling-observability-beyond-100pb-wide-events-replacing-otel — *what it shows: compression ratio is the affordability lever for long retention.*
- ClickHouse, "Are open table formats + lakehouses the future of observability?" — https://clickhouse.com/blog/lakehouses-path-to-low-cost-scalable-no-lockin-observability — *what it shows: the hot/cold dual-write pattern, argued by an interested party — read it for the architecture, discount the conclusion.*

**Deeper dives**
- `harut8/system-design`, "35 — Telemetry Lakehouse" — https://github.com/harut8/system-design/blob/main/sre-observability/35-telemetry-lakehouse.md — *the general-purpose version of this lesson, at ~660 lines: formats, engines, schema-on-read vs write, anti-patterns.*
- `harut8/system-design`, "17 — Telemetry Lakehouse & SQL Analytics" — https://github.com/harut8/system-design/blob/main/gpu-observability/17-telemetry-lakehouse-and-sql-analytics.md — *the GPU-specific version: the Kafka contract, the `GpuSample` schema, and the rollup grains this lesson's §3–§4 are built on.*
- Grafana Mimir documentation — https://grafana.com/docs/mimir/latest/ — *read to establish where the hot path's downsampling genuinely suffices, so you can defend not building a lake.*
