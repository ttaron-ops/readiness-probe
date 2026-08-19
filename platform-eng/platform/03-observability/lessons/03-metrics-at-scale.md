---
lesson: "A03.3"
title: "Metrics at scale"
module: "A-03"
concept: "cardinality, sharding, long-term storage"
status: not-started
est_time: "4 hrs"
prev: "02-prometheus-and-promql.md"
next: "04-opentelemetry.md"
artifacts: ["fleet-observability design"]
sources: 12
---

# A03.3 · Metrics at scale

> **Concept.** A metrics stack dies from head-block cardinality against RAM, not disk — and the fix is sharding scrape + remote-write into Mimir/Thanos, not federation.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 02 gave you the PromQL traps that ship a wrong single-Prometheus dashboard. This lesson is what happens when that single Prometheus is asked to hold a fleet: the same cardinality pressure that silently corrupts a query now silently kills the process, and recovery from that kill is itself the outage. Once you can size and shard the storage tier correctly, the next question is how telemetry actually gets *into* it at the collection edge across hundreds of heterogeneous services — that is lesson 04's Collector architecture.

Everything below is checked against **Prometheus 3.14.0** (`main`, August 2026), **Grafana Mimir 3.2.0-rc.0** (`main`), and **Thanos 0.43.0-dev** (`main`) — read from the cloned repositories, specifically Mimir's `docs/sources/mimir/manage/run-production-environment/planning-capacity.md`, `references/architecture/hash-ring/index.md`, `configure/configuration-parameters/index.md`, and Thanos's `docs/components/compact.md`. Numbers are quoted with their source; where a figure depends on your label sets, that is said.

## Why this matters

Every senior can stand up Prometheus. What separates staff is knowing the exact shape of the wall it hits at fleet scale, naming the alert that fires *before* the outage, and choosing between Thanos and Mimir with a defensible reason rather than a preference. On a thousands-of-node GPU fleet this is not academic: a single Prometheus falls over, and the recovery path — WAL replay — is itself a multi-tens-of-minutes outage during which nothing is scraped, nothing is queried, and no alert evaluates. The samples missed during replay are **permanently lost**, because Prometheus pulls: there is no buffer at the target to catch up from.

The second reason is economic and slower-moving. A fleet metrics tier is one of the largest line items in a platform budget after compute itself, and almost all of it is decided by three numbers you set in a design doc: active series, retention per resolution, and replication factor. Getting those wrong by 2× is a six-figure annual difference at fleet scale, and the mistake is invisible for months because the system works fine — it just costs more than it should.

The third is that "design a metrics system for 4,000 GPU nodes" is a standard interview scenario at every GPU-cloud and large-platform employer, and the discriminating detail is always the same: can you name the fall-over mode, the leading indicator, and the recovery-time budget — or do you only know the component names?

## What's new here (calibration)

- **Skip (you already know):** single-Prometheus scrape and PromQL; local TSDB on disk; that you eventually need remote long-term storage; that Thanos and Mimir exist.
- **New:** the fall-over sequence traced through the actual structures — head block → WAL growth → OOM → replay — with the arithmetic for how long each stage takes, so WAL replay becomes a stated RTO you design backward from rather than an incident-time surprise.
- **New:** the **growth model** behind a cardinality number. Churn, not node count, is usually the dominant term on a scheduled fleet, and a sizing exercise re-run only when node count changes will consistently under-provision.
- **New:** Mimir's *two* architectures — the classic hash-ring/RF=3 write path and the newer Kafka-backed **ingest storage** path — because the operational model, the failure modes and the quorum semantics differ, and the classic-only mental model is now incomplete.
- **New:** real capacity constants from the upstream sizing guide (CPU/GB per 300k series in an ingester, 13 KB/series of store-gateway disk, one compactor per 20 M series) so the worked example is a calculation rather than an assertion.
- **Corrected:** downsampling does **not** save storage. Thanos's own compactor documentation states it increases object-storage footprint roughly 3× when all three resolutions are kept, because each downsampled block stores five aggregations. Downsampling buys *query speed over long ranges*, which is a different justification.
- **Corrected:** the RF=3 write quorum is `floor(RF/2) + 1 = 2`, not `ceil(RF/2) + 1` (which would be 3). The previous version of this lesson had the formula wrong even though the answer was right.

## Core concepts

### 1. How it actually falls over — the sequence, with times

It is almost never disk. The failure is **cardinality → RAM → OOM → replay**, and the fourth stage is the one that hurts.

```
   THE FALL-OVER SEQUENCE, WITH WHERE THE TIME GOES
   ═══════════════════════════════════════════════════════════════════════════

   ① STEADY STATE                                    RAM: 24 GiB / 64 GiB limit
      head:  12 M active series × ~1.9 KB    ≈ 23 GiB
      WAL:   samples appended continuously; checkpointed every ~2 h when the
             head is truncated. Segments are 128 MiB.
      ─────────────────────────────────────────────────────────────────────────

   ② CARDINALITY EVENT (a churn change, not a metric change — see lesson 1 §8)
      new series created at 9,000/s instead of 40/s.
      head grows: +9,000 series/s × 1.9 KB   ≈ +17 MiB/s  ≈ +1 GiB/min
      WAL grows FASTER than usual too: every new series writes a series
      record (label set, ~300 B) in addition to its samples.
      ─────────────────────────────────────────────────────────────────────────
                                    ~35 min later
   ③ OOMKill                                          RAM: 64 GiB → container dies
      Duration of this step: ~3 seconds. Kubernetes restarts the pod.
      This is NOT the outage. It only looks like the interesting part.
      ─────────────────────────────────────────────────────────────────────────

   ④ WAL REPLAY  ← THE OUTAGE
      On start, Prometheus must reconstruct the entire head from the WAL
      before it opens the HTTP listener for queries or starts scraping:
        · read every WAL segment since the last checkpoint
        · for each SERIES record: allocate a memSeries, insert into the
          16,384-stripe hashmap, append a ref to every postings list
        · for each SAMPLES record: append into the right chunk
        · re-mmap the head chunk files in chunks_head/
      Cost is dominated by series-record replay, i.e. by CARDINALITY —
      the same quantity that caused the OOM.

      ┌──────────────────────────────────────────────────────────────────┐
      │  observed order of magnitude: single-digit GiB of WAL per minute │
      │  → 19.4 M series + samples ≈ 40–60 GiB of WAL                     │
      │  → replay ≈ 15–30 minutes on a modern host                        │
      │  MEASURE YOURS: prometheus_tsdb_data_replay_duration_seconds      │
      └──────────────────────────────────────────────────────────────────┘

      During ④:  no scrapes   → samples permanently lost (pull model)
                 no queries   → dashboards blank
                 no rules     → NO ALERTS EVALUATE AT ALL
      ─────────────────────────────────────────────────────────────────────────

   ⑤ CRASH LOOP
      Replay completes, head is rebuilt at 19.4 M series, RAM is back at 64 GiB,
      OOM again in minutes. Now every cycle costs another full replay.
      Exits: raise the limit (bridging), drop the label at scrape (real fix,
      but existing series persist until head truncation), or delete the WAL
      and accept the loss.
```

**The one number to internalise:** the OOM is seconds and the replay is tens of minutes. Everything about how you design this tier follows from making stage ④ short — which means making the head small, which means bounding cardinality, which is why lesson 1 comes first.

**Why replay cannot be parallelised away.** Prometheus does parallelise WAL replay across shards, but the work is inherently bounded by re-creating series and re-populating the postings index, which is memory-allocation-heavy and GC-heavy. There is no shortcut that skips it, because the head is not persisted — only the WAL is. (The head *chunks* are m-mapped to `chunks_head/`, which is why replay does not have to re-decompress every sample; but the series index is pure RAM and must be rebuilt.)

### 2. The second failure: silent remote-write back-pressure

The first failure is loud. The second is not.

When Prometheus remote-writes to Mimir/Thanos, a `QueueManager` reads from the WAL and ships batches. It is a sharded pipeline with defaults that matter (`docs/configuration/configuration.md`, `queue_config`):

| Parameter | Default | What it controls |
|---|---:|---|
| `capacity` | 10000 | samples buffered *per shard* before WAL reading blocks |
| `min_shards` | 1 | starting concurrency |
| `max_shards` | 50 | ceiling on concurrency |
| `max_samples_per_send` | 2000 | batch size |
| `batch_send_deadline` | 5s | max wait before sending a partial batch |
| `min_backoff` / `max_backoff` | 30ms / 5s | retry backoff, doubling |
| `retry_on_http_429` | false | **whether to retry when the remote says "slow down"** |
| `sample_age_limit` | 0s (disabled) | drop samples older than this rather than retrying forever |

Two of those defaults produce the classic silent-loss incident:

**`retry_on_http_429: false`.** Mimir's distributor returns **HTTP 429** when a tenant exceeds `ingestion_rate` (default 10,000 samples/s per tenant) or `request_rate`. By default Prometheus does **not** retry a 429 — it drops the batch and moves on. So the correct, well-behaved backpressure signal from your storage tier becomes silent data loss at the sender. Mimir's own documentation calls this out and recommends `retry_on_http_429: true`.

**Unbounded retry without `sample_age_limit`.** The opposite failure: with retries on and no age limit, a long remote outage makes Prometheus retry increasingly stale samples forever, the queue never drains, and the WAL cannot be truncated — so the WAL grows, and now you are back in §1 stage ④ by a different road. Setting `sample_age_limit` (say 2h) bounds it: samples older than the limit are dropped deliberately rather than accidentally.

**The metrics that show it, before it becomes loss:**

```promql
# 1. Queue depth. Rising and not draining = backpressure.
prometheus_remote_storage_samples_pending

# 2. Shard saturation. If desired == max, the sender is at its ceiling
#    and cannot go faster no matter what.
prometheus_remote_storage_shards_desired
  >= prometheus_remote_storage_shards_max

# 3. THE LAG QUERY — the single best remote-write health signal.
#    How far behind real time is the newest sample we successfully sent?
(
  prometheus_remote_storage_highest_timestamp_in_seconds
    - ignoring(remote_name, url)
  prometheus_remote_storage_queue_highest_sent_timestamp_seconds
) > 120

# 4. Actual loss. Anything non-zero here is data you no longer have.
rate(prometheus_remote_storage_samples_dropped_total[5m]) > 0
rate(prometheus_remote_storage_samples_failed_total[5m]) > 0
```

Query 3 is the one to put on the wall. It is in seconds, it is intuitive, and it goes bad before anything is dropped.

### 3. The growth model — churn dominates node count

A point-in-time series count is a snapshot, not a plan. Model it:

```
   active_series ≈  Σ over metrics of
                    ( fleet_fanout × independent_label_product )
                  + ( churning_series_per_unit_time × head_retention_window )

   fleet_fanout            = nodes × devices_per_node
   head_retention_window   ≈ 2–3 h (DefaultBlockDuration = 2h; the head is
                             compacted once it spans > 1.5 × the chunk range)
```

The second term is the one people forget, and on a scheduled GPU fleet it usually dominates. Compare two fleets:

```
   FLEET A — grows, but stable                FLEET B — flat, but churns
   ────────────────────────────               ─────────────────────────────
   4,000 → 8,000 nodes over a year            4,000 nodes, constant
   pods live for weeks                        pods restart every 4 min
                                              (preemption + short jobs)

   base series (25 metrics × 8 GPU):          base series: 800,000
     year 0:  800,000                         churn term:
     year 1:  1,600,000                          32,000 GPUs × 25 metrics
                                                 × (180 min / 4 min)
   growth: 2× in a year                          = 36,000,000
                                              → 45× the base, appearing in HOURS
```

Fleet B, which "isn't growing," produces a 45× series count in an afternoon. **Size for your churn rate, not your node count**, and instrument the churn rate directly:

```promql
# Series creation rate — the leading indicator (lesson 1 §8).
rate(prometheus_tsdb_head_series_created_total[5m])

# Its ratio to the standing count tells you the head's turnover.
# > ~0.001/s sustained means your head is being replaced hourly.
rate(prometheus_tsdb_head_series_created_total[5m]) / prometheus_tsdb_head_series
```

The design consequence: **the labels that churn must be concentrated in as few metrics as possible.** That is exactly the info-metric pattern from lesson 1 — one `dcgm_gpu_info` series per GPU absorbs all the pod churn, instead of 25 metrics each absorbing it.

### 4. Sharding: two axes, and what each buys

**Functional sharding.** A Prometheus (pair) per team, per service class, or per region. Blast radius follows an organisational boundary: team X's cardinality incident kills team X's Prometheus. Cloudflare runs 900+ instances on this model. Cheap to reason about, and the failure domain matches the ownership domain — which is what makes the on-call story work.

**Hashmod scrape sharding.** N Prometheis, each scraping `1/N` of the targets, selected by hashing the target address:

```yaml
# Shard 2 of 4. Deploy four identical configs with SHARD=0..3.
scrape_configs:
  - job_name: dcgm-exporter
    kubernetes_sd_configs:
      - role: endpoints
        namespaces: { names: [gpu-operator] }
    relabel_configs:
      # Keep only the targets whose address hashes into this shard.
      - source_labels: [__address__]
        modulus: 4
        target_label: __tmp_shard
        action: hashmod
      - source_labels: [__tmp_shard]
        regex: '2'                  # ← the only line that differs per shard
        action: keep
      # Standard k8s enrichment after the shard filter, so the cheap
      # filter runs first.
      - source_labels: [__meta_kubernetes_pod_node_name]
        target_label: Hostname
```

Two properties worth stating because they surprise people:

- **hashmod shards on the target, not on the series.** All series from one node land on one shard. That is what you want — a per-node query is answered by one shard — but it means an unusually fat node lands entirely on one shard, so shards are not perfectly balanced. Check `prometheus_tsdb_head_series` per shard and rebalance the modulus if the spread exceeds ~20%.
- **Adding a shard reshuffles everything.** `hashmod` is plain modulus, not consistent hashing: going from 4 to 5 shards moves roughly 80% of targets. In practice you double (4 → 8) so that half the targets stay put, and you accept a brief period of duplicate or missing scrapes during the rollout. If gapless cutover matters, run the new shard set in parallel writing to the same Mimir tenant and let deduplication handle the overlap.

**Both axes compose.** The standard fleet design is functional sharding at the top (a GPU-infra Prometheus estate separate from the app-team estate) and hashmod within each function.

**Vertical sharding — the third axis people forget.** You can also split by *metric*, not by target: one Prometheus scrapes only DCGM, another only cAdvisor/kubelet, another only app metrics. This is useful precisely because the churn characteristics differ per metric family: the DCGM estate can have a tight cardinality limit and short retention, while a low-churn node-exporter estate can hold more.

### 5. Federation is not scaling — mechanically, why

`/federate` is an HTTP endpoint on Prometheus that returns the current value of a selected set of series, in exposition format. The parent Prometheus **scrapes** it exactly like any other target.

That last sentence is the whole argument:

```
   WHY FEDERATION MOVES THE WALL INSTEAD OF REMOVING IT
   ═══════════════════════════════════════════════════════════════════════════

   ┌── child A ──┐  ┌── child B ──┐  ┌── child C ──┐  ┌── child D ──┐
   │ 3 M series  │  │ 3 M series  │  │ 3 M series  │  │ 3 M series  │
   │  /federate  │  │  /federate  │  │  /federate  │  │  /federate  │
   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
          │                │                │                │
          └────────────────┴───────┬────────┴────────────────┘
                                   │  parent SCRAPES each child
                                   ▼
                    ┌──────────────────────────────────┐
                    │  parent Prometheus                │
                    │  head must hold whatever it       │
                    │  scraped — SAME memSeries, SAME   │
                    │  postings, SAME WAL, SAME replay  │
                    └──────────────────────────────────┘

   match[]={__name__=~".+"}          → parent holds 12 M series.
                                       You have built a bigger single
                                       point of failure, with one extra
                                       hop of staleness.

   match[]={__name__=~"cluster:.*"}  → parent holds a few thousand
                                       AGGREGATE series. This is what
                                       federation is FOR.

   Additional failure modes federation adds:
     · the parent's scrape_timeout must exceed the child's serialisation
       time for the whole match set — a slow child times out and you lose
       ALL of its series for that interval, not some
     · sample timestamps are the CHILD's, so a slow federate scrape
       produces samples out of order relative to the parent's own clock
     · one scrape_interval of extra staleness on every federated series
     · no deduplication: two HA children federate two copies
```

**The rule:** federate *recording-rule output* (`cluster:*`, `job:*`) up a hierarchy. Never federate raw series. To move raw series, use remote-write, which is a streaming push with backpressure, retries and deduplication — all the properties a scrape does not have.

### 6. Thanos — Prometheus stays the source of truth

Thanos bolts long-term storage onto an existing Prometheus estate. The components and how a query is answered:

```
   THANOS — READ AND WRITE PATHS
   ═══════════════════════════════════════════════════════════════════════════

   WRITE (block-shipping model)
   ───────────────────────────
   Prometheus (unchanged)                    every 2 h, the head is compacted
     └── local TSDB ──▶ [ 2 h block ]        into a persisted block on disk
                            │
                     Thanos SIDECAR (same pod, reads the TSDB dir)
                            │  uploads the completed block
                            ▼
                    ┌──────────────────┐
                    │  OBJECT STORAGE   │  S3/GCS/Azure. Blocks are immutable.
                    │  <ULID>/          │  meta.json + index + chunks/
                    └────────┬─────────┘
                             │
                    Thanos COMPACTOR   (singleton per tenant/bucket!)
                      · vertical + horizontal compaction of small blocks
                      · downsampling: 5m for blocks >40 h, 1h for >10 d
                      · retention enforcement per resolution
                      · writes a bucket index the stores/queriers read

   READ (fan-out model)
   ───────────────────
                        Thanos QUERIER  (stateless, speaks PromQL)
                          │  fan-out via the Store API (gRPC)
       ┌──────────────────┼──────────────────┬─────────────────┐
       ▼                  ▼                  ▼                 ▼
   SIDECAR A          SIDECAR B         STORE GATEWAY     RULER (optional)
   (last ~2 h,        (last ~2 h,       (everything in    evaluates rules
    from Prom A)       from Prom B)      object storage)  against the Querier

   Querier DEDUPLICATES by a replica label (--query.replica-label=prometheus_replica)
   using a penalty-based merge: it prefers one replica's series and only
   switches when that replica has a gap. This is how an HA pair of Prometheis
   becomes one clean series.

   ── THE COST: every query touches EVERY component that might hold data.
      A 30-day range query fans out to the store gateway, which must open
      the index-header of every block in range, do the postings lookups,
      and stream chunks. Latency ∝ number of blocks × series matched.
      This is Thanos's characteristic weakness.
```

**The sidecar's structural limitation:** blocks are only uploaded once the head is compacted, which is **every 2 hours**. So a Prometheus that dies has up to 2 hours of un-uploaded data that exists only in its local TSDB and WAL. Thanos does not make Prometheus's local disk expendable; it makes it *eventually* expendable. (Thanos **Receive** exists precisely to close this — it accepts remote-write directly into a Thanos-native ingestion tier, which makes Thanos structurally much closer to Mimir. If you are choosing Thanos-with-Receive, you are choosing something with most of Mimir's operational weight.)

**The compactor is a singleton and that is a real constraint.** Running two compactors against the same bucket without proper sharding corrupts blocks — they will both try to compact the same source blocks and produce overlapping outputs. Thanos guards this with a lock, but the operational rule stands: one compactor per bucket, or shard by tenant with explicit relabelling. When the compactor falls behind, nothing pages: block count grows, query fan-out grows, store-gateway memory grows, and eight weeks later "historical queries got slow."

**Downsampling: what it is actually for.** This is worth stating precisely, because it is widely misunderstood. Thanos's compactor documentation is explicit: downsampling **does not save space** — it *adds* two more blocks per raw block, and keeping all three resolutions increases object-storage footprint by roughly **3×**. Each downsampled chunk is an `AggrChunk` holding **five** aggregations — `count`, `sum`, `min`, `max`, `counter` — so that `rate()`, `avg_over_time()`, `max_over_time()` and friends remain mathematically correct on the downsampled series.

What downsampling buys is **query speed over long ranges**. A one-year range query at raw 15 s resolution would load ~2.1 M samples per series; at 1 h resolution it loads 8,760. That is the entire justification, and it is a good one — but budget the storage as an increase, not a saving. The timing rules:

| Pass | Applies to | Trigger age | Result |
|---|---|---:|---|
| 1 | raw blocks | **> 40 hours** | 5 m resolution `AggrChunk`s |
| 2 | 5 m blocks | **> 10 days** | 1 h resolution `AggrChunk`s |

And the retention trap that follows from it: **retention per resolution must be at least as long as the age threshold for the next pass.** If `--retention.resolution-raw` is 24 h, raw blocks are deleted before they are 40 h old and the 5 m downsample never happens. The upstream rule of thumb is to set all three retentions equal, and longer than 10 days.

### 7. Mimir — Prometheus becomes a dumb forwarder

Mimir (a Grafana Labs fork of Cortex) inverts the model: Prometheus (or Grafana Alloy, or the OTel Collector) does nothing but scrape and remote-write; all storage, querying and rule evaluation happen in Mimir. Two architectures ship today and you need both in your head.

**Classic architecture — hash ring, RF=3.**

```
   MIMIR CLASSIC WRITE PATH
   ═══════════════════════════════════════════════════════════════════════════

   Prometheus/Alloy ──remote-write──▶ DISTRIBUTOR  (stateless, behind an L7 LB)
                                        │
              validates: label count/length, metadata length, sample age,
                         exemplar labels (≤128)
              enforces:  per-tenant request_rate, ingestion_rate → HTTP 429
              dedups:    HA pairs, via the HA tracker
                                        │
                        hash each series' labels with fnv32a → 32-bit token
                                        │
                        look up the token in the INGESTER RING;
                        the owner is the instance with the smallest
                        token greater than the series token;
                        walk clockwise for the next RF−1 owners
                                        │
              ┌─────────────────────────┼─────────────────────────┐
              ▼                         ▼                         ▼
         INGESTER #2               INGESTER #3               INGESTER #4
         (authoritative)            (replica)                 (replica)
              └─────────────────────────┴─────────────────────────┘
                                        │
                    WRITE SUCCEEDS ON A QUORUM OF ⌊RF/2⌋+1 = 2 of 3
                    ⇒ tolerates losing 1 ingester with no write errors
                    ⇒ losing 2 of the 3 owners of a series fails writes
                                        │
              each ingester holds series in memory (its own TSDB head)
              and periodically ships blocks to OBJECT STORAGE
                                        │
                                        ▼
                    STORE-GATEWAY serves historical blocks;
                    COMPACTOR compacts and deduplicates them.
```

**Ingest storage architecture — Kafka in the middle.** The newer model replaces distributor→ingester replication with distributor→**Kafka partition**→ingester consumption. Replication becomes Kafka's problem. Sharding uses two rings: a **partitions ring** (which series goes to which Kafka partition) and an **ingesters ring** (which ingester owns which partition, for service discovery on the read path). Critically, **read consistency is a quorum of 1** — each partition need only be queried once, from any healthy ingester that owns it, because Kafka already guaranteed the write. Partitions have a lifecycle (`Pending` → `Active` → `Inactive`) managed by the ingesters that own them, which is what makes scale-down safe.

Why it matters for a design review: the classic architecture's write amplification is 3× at the ingester tier (every series is in RAM three times); the ingest-storage architecture moves durability into Kafka and lets you run fewer ingester replicas per partition, at the cost of operating Kafka. If someone asks "how do you scale Mimir writes," the current answer names both.

**The multi-tenancy that makes Mimir the fleet answer.** Every request carries an `X-Scope-OrgID` header. Limits are per tenant, and the defaults are deliberately low so you set them consciously (`configure/configuration-parameters/index.md`):

| Limit | Default | Effect when exceeded |
|---|---:|---|
| `max_global_series_per_user` | **150,000** | series rejected at the distributor |
| `max_global_series_per_metric` | 0 (disabled) | per-metric-name cap; catches one runaway metric |
| `ingestion_rate` | **10,000** samples/s | HTTP 429 |
| `ingestion_burst_size` | **200,000** samples | token-bucket burst |
| `request_rate` | 0 (disabled) | HTTP 429 |
| `max_fetched_series_per_query` | — | protects the read path from one huge query |

**Note the interaction with §2.** `ingestion_rate` defaulting to 10,000 samples/s is *nothing* at fleet scale — 800,000 series at a 15 s scrape is ~53,000 samples/s for one tenant. If you do not raise it, you get 429s; and if `retry_on_http_429` is left at Prometheus's default `false`, those 429s are silent data loss. **Those two defaults are adversarial by accident and this pairing is the single most common Mimir onboarding failure.**

**Shuffle sharding** is the other multi-tenancy primitive worth knowing. Instead of spreading a tenant's series across all ingesters, Mimir can assign each tenant a random *subset* of instances. Two tenants then share few or no instances, so one tenant's expensive queries or write spikes degrade a bounded set of others. It is the difference between "one bad tenant slows everyone" and "one bad tenant slows a computable handful."

### 8. Choosing — Thanos vs Mimir vs staying put

| | **Prometheus + remote-write to a managed service** | **Thanos** | **Mimir** |
|---|---|---|---|
| Source of truth | vendor | **your Prometheus** | Mimir ingesters |
| Ingest model | remote-write | block upload via sidecar (2 h lag) | remote-write, sharded by hash ring or Kafka |
| Multi-tenancy | vendor's | weak (bucket/label conventions) | **native, per-tenant limits, shuffle sharding** |
| Query at scale | vendor's | fan-out over sidecars + store gateways; **the weak point** | query-frontend + scheduler + query sharding |
| Ops burden | lowest | **moderate** — sidecar, store GW, compactor | **highest** — 8+ component types, a ring, caches |
| Data loss window on Prom death | seconds (WAL-backed queue) | **up to 2 h** (unshipped block) | seconds |
| Best when | small estate, no compliance blocker | you already run many Prometheis and want cheap history | one fleet-scale multi-tenant plane |

**The honest decision rule.** Under ~1–2 M active series and one team: stay on sharded Prometheus with a long local retention and skip both. From ~2 M to ~20 M with an existing Prometheus estate and no hard multi-tenancy requirement: Thanos, because the incremental cost is a sidecar and a bucket. Above that, or the moment you need per-tenant limits and chargeback on a shared plane: Mimir. **Do not pick Cortex for anything new** — Mimir superseded it, and Mimir ships a documented migration path (`docs/sources/helm-charts/mimir-distributed/migration-guides/migrate-from-cortex.md`).

**Why Mimir exists as a separate project at all** is a genuine interview question with a non-technical answer: the fork was driven by open-source licensing and business sustainability — Grafana Labs consolidating its stack under a licence it controls — not by Cortex being technically broken. Answering "because Cortex had bugs" signals you read a comparison blog post; answering "governance and licensing, and then the projects diverged technically afterwards" signals you understand why the ecosystem looks the way it does.

### 9. Layered cardinality defence

Neither layer is sufficient alone; they guard different boundaries.

**Layer 1 — scrape-time relabelling (lesson 1 §9).** Prevents the series from ever being created. Cheapest possible, but only catches what you anticipated, and it lives in the config of the thing doing the scraping — so it does not protect you from a tenant that pushes directly via OTLP.

**Layer 2 — distributor limits.** Rejects whatever gets through, regardless of cause, and protects the *shared* ingester fleet from any tenant's mistake. This is the layer that turns "your team broke the observability plane" into "your team got 429s," which is a completely different conversation.

**Layer 3 — the read path.** Easy to forget: `max_fetched_series_per_query` and `max_fetched_chunk_bytes_per_query` stop one badly-written dashboard panel from OOMing a querier. A read-path limit is a cardinality control too.

Concretely, per-tenant overrides in Mimir's runtime config:

```yaml
# runtime.yaml — reloaded without restart
overrides:
  gpu-infra:
    max_global_series_per_user: 4000000      # the fleet tenant
    max_global_series_per_metric: 200000     # catches ONE runaway metric name
    ingestion_rate: 200000                   # samples/s — 800k series @ 15 s ≈ 53k, 4× headroom
    ingestion_burst_size: 2000000
    max_fetched_series_per_query: 500000
    max_global_exemplars_per_user: 500000

  team-vision:
    max_global_series_per_user: 250000
    ingestion_rate: 25000
    ingestion_burst_size: 250000

  team-nlp:
    max_global_series_per_user: 250000
    ingestion_rate: 25000
    ingestion_burst_size: 250000
```

and the matching sender-side config, which is where the 429 interaction is resolved:

```yaml
# prometheus.yml on each scrape shard
remote_write:
  - url: http://mimir-distributor.mimir.svc:8080/api/v1/push
    headers:
      X-Scope-OrgID: gpu-infra
    queue_config:
      capacity: 10000
      max_shards: 200            # default 50 is too low for fleet volume
      min_shards: 10             # avoid the slow ramp after a restart
      max_samples_per_send: 2000
      batch_send_deadline: 5s
      retry_on_http_429: true    # ← WITHOUT THIS, 429s ARE SILENT LOSS
      sample_age_limit: 2h       # ← bound the retry window; protects the WAL
    write_relabel_configs:
      # Last-chance drop before anything leaves the process.
      - source_labels: [__name__]
        regex: 'go_gc_duration_seconds.*|promhttp_.*'
        action: drop
```

### 10. Sizing constants you can actually calculate with

From Mimir's upstream capacity-planning guide. These are stated as rough production estimates by the project, with a recommendation to run **50% extra** memory and disk for peaks:

| Component | Driver | CPU | Memory | Disk |
|---|---|---|---|---|
| Distributor | samples/s | 1 core / 25,000 samples/s | 1 GB / 25,000 samples/s | — |
| **Ingester** | **in-memory series (after RF)** | **1 core / 300,000 series** | **2.5 GB / 300,000 series** | **5 GB / 300,000 series** |
| Query-frontend | queries/s | 1 core / 250 q/s | 1 GB / 250 q/s | — |
| Query-scheduler | queries/s | 1 core / 500 q/s | 100 MB / 500 q/s | — |
| Querier | queries/s | 1 core / 10 q/s | 1 GB / 10 q/s | — |
| Store-gateway | q/s and active series | 1 core / 10 q/s | 1 GB / 10 q/s | **13 GB / 1 M active series** |
| Compactor | active series (pre-replication) | 1 core | 4 GB | 300 GB |
| Alertmanager | firing alerts | 1 core / 100 notifications/s | 1 GB / 5,000 firing alerts | — |

Two derived rules from the same source:

- **Ingester series count is `active_series × replication_factor`.** With RF=3, 800,000 active series means 2.4 M in-memory series across the ingester fleet — that is what you size against, not 800,000.
- **Run at least one compactor per 20 M active series** (pre-replication), and remember it is *not* a per-tenant singleton in Mimir the way it is in Thanos — Mimir's compactors shard via their own hash ring.
- The store-gateway's 13 KB/series disk figure assumes 2 bytes/sample in compacted blocks, an index-header at ~0.10% of block size, a 15 s scrape, 1-year retention, and store-gateway RF=3. Change any of those and rescale.

### 11. WAL replay as a stated RTO

The staff move is to invert the whole lesson: **pick a recovery-time budget first, then derive the cardinality ceiling that guarantees it.**

```
   DERIVING A CARDINALITY CEILING FROM AN RTO
   ═══════════════════════════════════════════════════════════════════════════
   Requirement:  after ANY Prometheus restart — routine deploy, node drain,
                 OOM, node failure — the shard is serving again within 5 min.

   Step 1  Measure YOUR replay rate. Restart a shard at a known series count
           and read the histogram:
              prometheus_tsdb_data_replay_duration_seconds
           Suppose 1.0 M series replays in 95 s.
              ⇒ ~10,500 series/s of replay throughput on this hardware.

   Step 2  Budget. 5 min = 300 s, keep 40 % margin for a cold page cache:
              usable = 180 s
              ceiling = 180 s × 10,500 series/s ≈ 1.9 M series per shard

   Step 3  Add the churn headroom from §3. If the head can grow 2× in an
           hour under normal job turnover:
              steady-state ceiling ≈ 950,000 series per shard

   Step 4  Shard count = total_series / ceiling, rounded up, then rounded
           to a power of two so future doubling only moves half the targets.

   Step 5  Turn the ceiling into ENFORCEMENT, not a hope:
              · per-tenant max_global_series_per_user in Mimir (§9)
              · a per-shard alert on prometheus_tsdb_head_series
              · a per-shard alert on the SERIES CREATION RATE (leading)
```

That is the difference between "keep cardinality reasonable" and an availability commitment with a number, an alert, and an enforcement point.

### 12. GPU-fleet tie

The fleet metrics tier, assembled from the pieces above (lesson 9 makes it concrete):

- **Vertical split first.** A dedicated Prometheus estate for DCGM, separate from kubelet/cAdvisor and separate from app metrics. Different churn profiles, different retention, different cardinality ceilings, independent blast radii.
- **Hashmod within the DCGM estate**, sized from §11, with the shard count a power of two.
- **Remote-write into Mimir** with `X-Scope-OrgID: gpu-infra`, `retry_on_http_429: true`, `max_shards: 200`, `sample_age_limit: 2h`.
- **Per-tenant limits** so a training team's label explosion becomes their 429s, not the shared plane's OOM.
- **Recording rules for every fleet rollup** — per-cluster, per-tenant, per-GPU-family — evaluated in Mimir's ruler rather than in each shard, so the rollups exist once and are computed over the deduplicated global view.
- **Downsampling with eyes open:** budget ~3× object storage for keeping raw + 5 m + 1 h, and justify it as long-range query speed for capacity planning, not as a saving.
- **The two fall-over alerts** (§13 of the worked example), plus the remote-write lag query from §2.

The GPU-specific amplifier is that a DCGM exporter emits *many* metrics per GPU, so the per-node fan-out is 25–40× larger than a typical node exporter, and MIG multiplies it again by up to 7 per physical GPU. A fleet of 4,000 GPU nodes is, in series terms, comparable to a few hundred thousand ordinary nodes. Size accordingly.

## Perspectives

**Capacity-planning.** A sizing exercise anchored on node count will under-provision on a scheduled fleet, because the churn term (§3) grows independently of node count and much faster. The staff artefact is not a number; it is a *model* with churn as an input, re-evaluated when scheduler behaviour changes — which means the observability team needs to be in the review loop for scheduler and autoscaler changes. That is an org-design consequence of an arithmetic fact.

**Operability.** WAL replay time is a hidden SLA that applies to *every* restart, not just incidents. Every routine deploy of Prometheus, every node drain, every kubelet eviction pays it. State a replay budget, measure it with `prometheus_tsdb_data_replay_duration_seconds`, and derive the cardinality ceiling from it (§11). A team that has never measured its own replay rate does not know its own RTO.

**Multi-tenancy and economics.** Mimir's per-tenant limits are simultaneously a blast-radius control and a cost-allocation mechanism. Once a tenant has a series quota, cardinality becomes chargeable the way CPU is: "your team's label explosion degraded the shared plane" turns into "your team is at 94% of its series quota, here is the request form." That converts a recurring postmortem action item into a standing process — which is the only way this scales past a handful of teams.

**Architecture-history.** Cortex → Mimir was a licensing and governance fork, not a technical rescue; the technical divergence came after. Knowing the difference matters because it predicts the future: projects that fork for governance reasons tend to diverge in *operational* surface (config formats, deployment modes, tooling) faster than in core algorithms, which is exactly what happened — Mimir's headline differences from Cortex are its config model, its Helm/Jsonnet tooling and its defaults, more than its storage engine.

**Reliability engineering.** The most dangerous property of this tier is that its failures are self-reinforcing. Cardinality causes an OOM; the OOM causes a replay; the replay's cost scales with the cardinality that caused it; and during the replay no alerts evaluate — including the alert that would have told you about the cardinality. Any subsystem with that shape needs an *external* observer: a small, boring, low-cardinality Prometheus (or a managed endpoint) whose only job is to scrape the big ones and alert on their health. Monitoring your monitoring from inside your monitoring is a single point of failure with extra steps.

## Real-world use cases

- **Cloudflare, "How Cloudflare runs Prometheus at scale."** 900+ Prometheus instances, functionally sharded by team and service, with metric review and `pint` rule linting in the pipeline. **What it shows:** functional sharding at a scale comparable to a large GPU fleet, and — more importantly — that the sharding topology follows the *ownership* topology. Blast radius that matches an on-call rotation is worth more than blast radius that is merely small.

- **Uber, "M3: Uber's Open Source, Large-scale Metrics Platform for Prometheus."** A purpose-built distributed TSDB with quorum writes at replication factor 3, aggregating on the order of hundreds of millions of metrics per second. **What it shows:** the same RF=3 quorum arithmetic as Mimir's classic path arrived at independently, which is a good sign it is the right trade-off for this workload class; and that past a certain scale the answer is a distributed TSDB, not a bigger Prometheus.

- **Groww Engineering, "Migration from Thanos to Grafana Mimir."** A production write-up of the distributor→ingester→store-gateway cutover, including importing historical Thanos blocks. **What it shows:** the migration is mostly a *block-format and tenancy* exercise rather than a data-loss risk — Thanos and Mimir both write Prometheus TSDB blocks to object storage — which is the concrete reason "start with Thanos, move to Mimir if you need tenancy" is a defensible sequencing rather than a rewrite.

- **The 429 silent-loss pattern.** Not a single named postmortem but a repeated onboarding failure: Mimir's `ingestion_rate` default of 10,000 samples/s meets Prometheus's `retry_on_http_429: false` default, and a new tenant's data disappears with no error visible in any dashboard the tenant owns. **What it shows:** the most dangerous defaults are the ones that are individually reasonable and jointly catastrophic. When integrating two systems, enumerate the *pairs* of defaults at the seam, not just each system's defaults in isolation.

## Worked example

**Scenario: 4,000 GPU nodes, 8 GPUs each. 30-day high-resolution retention, 2-year downsampled retention. Design the metrics tier.**

---

**Step 1 — series count, from the label model (not a guess).**

Using the lesson-1 classification (per-GPU labels `Hostname`, `gpu`, `namespace`; `UUID`/`pod` demoted to an info metric):

```
   GPUs                = 4,000 nodes × 8              = 32,000
   DCGM fields kept    (after a counter-set audit)    = 25
   per-GPU series      = 32,000 × 25                  = 800,000
   info metric         = 1 per GPU                    =  32,000
   node-level (node-exporter, kubelet, cAdvisor)
                       ≈ 900 series/node × 4,000      = 3,600,000
   ─────────────────────────────────────────────────────────────
   TOTAL ACTIVE SERIES                                ≈ 4,432,000
```

Note which term dominates: **node-level metrics, not GPU metrics.** cAdvisor and kubelet are the cardinality hogs on any Kubernetes fleet. This is an argument for the vertical split in §12 — the DCGM estate and the node estate have different growth curves and should not share a failure domain.

For sizing, take the GPU estate (832,000 series) and the node estate (3.6 M series) separately.

---

**Step 2 — sample rate.**

```
   GPU estate:   832,000 series × (1 sample / 15 s)   ≈ 55,467 samples/s
   Node estate:  3,600,000 series × (1 sample / 30 s) ≈ 120,000 samples/s
   ─────────────────────────────────────────────────────────────────────
   TOTAL                                              ≈ 175,467 samples/s
```

Immediately: Mimir's default `ingestion_rate` of 10,000 samples/s per tenant is **17× too low**. Set it, and set `retry_on_http_429: true`, before anything else.

---

**Step 3 — shard count, from the RTO (§11).**

```
   Measured replay throughput on the target instance type: 10,500 series/s
   (from prometheus_tsdb_data_replay_duration_seconds after a test restart)

   RTO 5 min, 40 % margin        → 180 s usable
   ceiling per shard             = 180 × 10,500          ≈ 1,890,000 series
   churn headroom (2× in an hour)→ steady-state ceiling  ≈   945,000 series

   GPU estate:   832,000 / 945,000  = 0.88  →  round to  2 shards (HA pair)
   Node estate:  3,600,000 / 945,000 = 3.81 →  round to  8 shards
                                              (power of two, so the next
                                               doubling moves half the targets)

   Per-shard load:
     GPU shard:  416,000 series → RAM ≈ 416,000 × 1.9 KB ≈ 0.8 GiB head
                                  request 4 GiB, limit 8 GiB (query + GC headroom)
     Node shard: 450,000 series → RAM ≈ 0.9 GiB head
                                  request 4 GiB, limit 8 GiB
```

Both estates are HA pairs (2× each), so 4 GPU-estate processes and 16 node-estate processes. Mimir's HA tracker deduplicates the pairs at the distributor using `__replica__` and `cluster` labels.

---

**Step 4 — Mimir sizing, from the upstream constants (§10).**

```
   ACTIVE SERIES (post-dedup, pre-replication)         = 4,432,000
   INGESTER IN-MEMORY SERIES = active × RF(3)          = 13,296,000

   INGESTERS
     memory: 13.3 M / 300,000 × 2.5 GB   = 110.8 GB   → +50 % = 166 GB
     cpu:    13.3 M / 300,000 × 1 core   = 44.3 cores → +50 % = 67 cores
     disk:   13.3 M / 300,000 × 5 GB     = 221.6 GB   → +50 % = 333 GB
     Choose instance shape: 12 ingesters × (16 GB RAM, 6 cores, 30 GB disk)
       ⇒ 192 GB RAM, 72 cores, 360 GB — clears all three with margin,
         and 12 replicas means losing 1 during a rolling deploy still
         leaves every series with ≥2 of its 3 owners (quorum holds).

   DISTRIBUTORS
     175,467 samples/s / 25,000 = 7.0 cores and 7.0 GB
     Choose 6 × (2 cores, 4 GB) — stateless, scale on CPU, sit behind an L7 LB.
     (L7, not L4: remote-write reuses one TCP connection per shard, so an
      L4 service balances connections, not requests, and skews distributors.)

   STORE-GATEWAY
     disk: 4.43 M / 1 M × 13 GB = 57.6 GB  → ×RF3 already included in the
           13 GB figure; add 50 % → 86 GB
     Assume ~30 queries/s reaching the store gateway at peak:
       cpu 3 cores, memory 3 GB → +50 % → 4.5 cores, 4.5 GB
     Choose 3 × (4 cores, 8 GB, 100 GB disk) for HA and shard spread.

   COMPACTOR
     4.43 M active series → 1 instance per 20 M ⇒ 1 needed.
     Run 2 for availability: each 1 core, 4 GB, 300 GB disk.

   QUERIER / QUERY-FRONTEND
     Assume 40 q/s sustained (dashboards + rulers):
       queriers: 40/10 = 4 cores, 4 GB → run 6 × (2 cores, 4 GB)
       query-frontend: 40/250 → tiny; run 2 for HA
       query-scheduler: 2 replicas
```

---

**Step 5 — the RF=3 quorum property, stated correctly.**

With `-ingester.ring.replication-factor: 3`, the distributor considers a write successful once a **quorum of `⌊3/2⌋ + 1 = 2`** ingesters accept each series. Therefore:

- Losing **1** of the 3 owners of a series: writes still succeed. ✔
- Losing **2** of the 3 owners: writes for that series fail. ✘

That is why the answer to "how many ingesters" is never just "enough for the memory." During a rolling deploy one replica is always down; if a second fails at that moment, any series whose token range is owned by both is unwritable. Twelve ingesters instead of the six that memory alone would justify buys the probability of that overlap down to near zero, and gives the ring enough tokens to spread evenly.

Under the **ingest storage** architecture this arithmetic changes: durability is Kafka's, and read consistency is a **quorum of 1** — the querier reads each partition from any one healthy owning ingester. You size ingesters for consumption throughput and query fan-out, not for replication.

---

**Step 6 — retention and the honest storage cost.**

```
   RAW, 30 DAYS
     samples/day  = 175,467 samples/s × 86,400        = 15.16 e9
     bytes/day    = 15.16 e9 × 2 B/sample (compacted) = 30.3 GB/day
     30 days                                          = 909 GB
     × RF at rest? NO — object storage holds ONE copy after compaction.
                                                        ─────────
                                                        ≈ 0.9 TB

   5-MINUTE RESOLUTION, 2 YEARS
     samples/series/day = 288
     4.43 M series × 288 × 2 B                        = 2.55 GB/day
     730 days                                         = 1.86 TB

   1-HOUR RESOLUTION, 2 YEARS
     4.43 M series × 24 × 2 B                         = 0.21 GB/day
     730 days                                         = 155 GB

   ── BUT: Thanos/Mimir downsampled blocks are AggrChunks holding
      count, sum, min, max, counter — five series' worth of data, not one.
      Multiply the two downsampled tiers by ~5:
        5m tier  ≈ 1.86 TB × 5   ≈  9.3 TB
        1h tier  ≈ 155 GB × 5    ≈ 775 GB
   ─────────────────────────────────────────────────────────────────────
   TOTAL OBJECT STORAGE ≈ 0.9 + 9.3 + 0.8                ≈ 11 TB

   At a representative $0.021/GB-month for standard object storage:
        11,000 GB × $0.021                              ≈ $231/month
   ── the storage is CHEAP. The ingesters' RAM is the expensive part:
        192 GB RAM + 72 cores, always on, is the real bill.
```

**Read the shape of that result.** The instinct that "long retention costs a lot" is wrong for metrics: two years of downsampled history is a couple of hundred dollars a month. What costs money is the *hot* tier — in-memory series in ingesters — which is a function of **active series**, i.e. cardinality, i.e. lesson 1. Retention is cheap; cardinality is expensive. Design accordingly.

**The retention configuration that avoids the §6 trap:**

```yaml
# Thanos compactor — all three retentions must exceed the downsample thresholds.
--retention.resolution-raw=30d      # > 40 h, so the 5m pass can run
--retention.resolution-5m=2y        # > 10 d, so the 1h pass can run
--retention.resolution-1h=2y
```

```yaml
# Mimir equivalent — per-tenant, in runtime overrides.
overrides:
  gpu-infra:
    compactor_blocks_retention_period: 2y
```

---

**Step 7 — the two fall-over alerts, plus the third nobody writes.**

```yaml
groups:
  - name: metrics-tier-fallover
    interval: 30s
    rules:
      # ① LEADING INDICATOR — series creation rate against its own baseline.
      #    Fires ~20 min before head_series looks alarming.
      - alert: PrometheusSeriesCreationSpike
        expr: |
          rate(prometheus_tsdb_head_series_created_total[5m])
            > 10 * avg_over_time(
                rate(prometheus_tsdb_head_series_created_total[5m])[6h:5m]
              )
        for: 10m
        labels: { severity: page }
        annotations:
          summary: '{{ $labels.instance }} creating {{ $value | printf "%.0f" }} series/s'
          runbook: 'Check recent scheduler/deploy changes; find the metric with
                    topk(10, count by (__name__)({__name__=~".+"}))'

      # ② HARD CEILING — head series against the RTO-derived limit from §11.
      - alert: PrometheusHeadSeriesOverCeiling
        expr: prometheus_tsdb_head_series > 945000
        for: 15m
        labels: { severity: page }
        annotations:
          summary: 'head at {{ $value }} series; WAL replay will exceed the 5 min RTO'

      # ③ THE SILENT ONE — remote-write lag, in seconds behind real time.
      - alert: RemoteWriteLagging
        expr: |
          (
            prometheus_remote_storage_highest_timestamp_in_seconds
              - ignoring(remote_name, url)
            prometheus_remote_storage_queue_highest_sent_timestamp_seconds
          ) > 120
        for: 10m
        labels: { severity: page }
        annotations:
          summary: 'remote-write {{ $labels.url }} is {{ $value | printf "%.0f" }}s behind'

      # ④ ACTUAL LOSS — should never be non-zero.
      - alert: RemoteWriteDroppingSamples
        expr: rate(prometheus_remote_storage_samples_dropped_total[5m]) > 0
        for: 5m
        labels: { severity: page }

      # ⑤ THE RTO ITSELF — measured, not assumed.
      - alert: WALReplayExceedsRTO
        expr: |
          histogram_quantile(0.99,
            rate(prometheus_tsdb_data_replay_duration_seconds_bucket[7d])
          ) > 300
        labels: { severity: ticket }
        annotations:
          summary: 'p99 WAL replay is {{ $value | printf "%.0f" }}s; RTO budget is 300s'
```

Alert ⑤ is the one nobody writes and everybody needs: it turns the RTO from a design-doc sentence into a continuously-verified property.

## Practice

Feeds the [fleet observability design](../practice/fleet-observability/README.md).

Write the **metrics-tier section of the fleet observability design doc** for the 4,000-node fleet. It must contain, each with its arithmetic shown:

1. **A series model, not a series number.** Break the total into per-metric-family terms with the label products from lesson 1, plus an explicit churn term with its assumption stated (restart rate × head window). State which term dominates and what would change that.
2. **The sharding topology.** Choose vertical splits, functional splits and hashmod within them; justify the modulus; and explain your rollout procedure for going from N to 2N shards without a gap.
3. **The RTO derivation.** Measure (or state as an assumption with units) your replay throughput in series/s, pick an RTO, and derive the per-shard series ceiling and therefore the shard count. Show the margin you kept and why.
4. **Thanos vs Mimir, decided.** One paragraph naming the deciding property for *your* situation — not a feature table. If you choose Thanos, state your answer to the 2-hour unshipped-block window. If you choose Mimir, state your answer to the operational surface.
5. **Both cardinality-defence layers as real config.** The scrape-time `metric_relabel_configs`, the per-tenant `overrides` block, and the read-path limits. Include the `retry_on_http_429` / `ingestion_rate` pairing and say in one sentence why they must be set together.
6. **Component sizing** from the constants in §10, carried through with units, including the RF multiplication for ingesters and the +50% headroom. State the instance shapes you would actually request.
7. **Retention and its true cost.** Three tiers, the downsampling thresholds, the ~5× `AggrChunk` inflation, and a dollar figure. Then state which line dominates the bill and what that implies about where to spend engineering effort.
8. **The alert set** — all five from the worked example, with your own thresholds derived from your own baselines, plus a one-line runbook annotation each.
9. **The external observer.** Describe the small independent Prometheus (or managed endpoint) that watches the big ones, what it scrapes, and why it must not live in the same failure domain.

**Acceptance criteria.** Done when (i) every number in the doc has an arithmetic derivation or a cited constant next to it, (ii) a reviewer can change one input — node count, churn rate, scrape interval, RF — and re-run the whole thing, (iii) the alert thresholds are traceable to the sizing decisions rather than round numbers, and (iv) the Thanos/Mimir choice paragraph would survive someone arguing the other side.

## Common pitfalls

- **"Thanos and Mimir solve the same problem the same way."** *Symptom:* a design doc that picks one on operator preference. *Mechanism:* Thanos keeps Prometheus as the source of truth and ships completed 2-hour blocks — lower lift, weaker query performance at fan-out, and a 2-hour data-loss window if a Prometheus dies. Mimir replaces Prometheus's storage role with a sharded ingest path — higher lift, native multi-tenancy, seconds-scale loss window. Different systems with different failure modes.

- **"Federation gives us horizontal scale."** *Symptom:* a parent Prometheus that OOMs a month after the children stopped OOMing. *Mechanism:* the parent *scrapes* `/federate`, so it must hold every federated series in its own head, with its own WAL and its own replay. Federation relocates the wall and adds a scrape-timeout failure mode where a slow child loses *all* its series for that interval.

- **"Remote-write failures are loud."** *Symptom:* a gap in Mimir that nobody noticed for a week. *Mechanism:* `retry_on_http_429` defaults to `false`, so the storage tier's correct backpressure signal becomes a silent drop at the sender. Alert on the lag query and on `samples_dropped_total`, because nothing in the application or the dashboard will tell you.

- **"Downsampling saves storage."** *Symptom:* a capacity plan that budgets *less* object storage for long retention. *Mechanism:* each downsampled chunk is an `AggrChunk` holding count, sum, min, max and counter, and all three resolutions are retained by default — Thanos's own docs put the increase at roughly 3×. Downsampling buys long-range query speed. Budget it as a cost with a benefit, not a saving.

- **"Retention is the expensive part."** *Symptom:* an aggressive retention cut that saves a few hundred dollars and costs a capacity-planning capability. *Mechanism:* compacted samples are ~2 bytes; two years of downsampled history for a 4.4 M-series fleet is on the order of 10 TB, i.e. low hundreds of dollars a month. The expensive resource is ingester RAM, which scales with *active series* × RF. Cardinality is the bill.

- **"More shards is always safer."** *Symptom:* 32 shards, each at 5% utilisation, and a query path that fans out to 32 endpoints for every panel. *Mechanism:* shard count trades ingest blast radius against query fan-out and operational surface. Each shard is a process to deploy, a rule set to keep in sync, and an endpoint the querier must contact. Size from the RTO ceiling (§11), then stop.

- **"`hashmod` is consistent hashing."** *Symptom:* a shard-count change that reshuffles the whole fleet and produces a scrape gap. *Mechanism:* `hashmod` is `hash(value) mod N` — changing N moves ~`(N−1)/N` of the targets. Double the shard count so half stay put, and overlap the old and new sets during cutover.

- **"One compactor is one compactor."** *Symptom:* corrupted or overlapping blocks in the bucket. *Mechanism:* Thanos's compactor is a single-writer per bucket; running two unsharded against the same bucket produces conflicting compaction outputs. Mimir's compactors shard via their own hash ring and are safe to run many — the two systems differ here, and the Thanos rule does not transfer.

## Self-check

**What resource does Prometheus run out of first under a cardinality spike, and why does recovery make the outage worse?**
RAM. The head block holds one `memSeries` plus its label string, open chunk and postings refs per active series; a churn spike multiplies active series and the process is OOMKilled — in about three seconds. The outage is the *recovery*: on restart Prometheus must replay the write-ahead log to rebuild the head before it opens the query listener or starts scraping, and replay cost scales with the same series count that caused the OOM. Tens of minutes during which nothing is scraped (samples permanently lost, because Prometheus pulls), nothing is queried, and no rules evaluate — including the rule that would have alerted on the cardinality. Then the head is rebuilt at the same size and it OOMs again: a crash loop where every cycle costs a full replay. Measure yours with `prometheus_tsdb_data_replay_duration_seconds`.

**Why is `/federate` the wrong tool for scaling, and what do you use instead?**
Because federation is a *scrape*: the parent Prometheus ingests whatever it federates into its own head, with its own `memSeries`, postings, WAL and replay cost. `match[]={__name__=~".+"}` across four 3-M-series children gives you a 12-M-series parent — a bigger single point of failure with one extra scrape interval of staleness, no deduplication of HA pairs, and a new failure mode where a slow child times out and you lose *all* of its series for that interval. Federation is for pulling *recording-rule aggregates* (`cluster:*`, `job:*`) up a hierarchy. To move raw series, use remote-write: a streaming push with backpressure, retries, deduplication and a WAL-backed queue.

**You need best-in-class query performance and native per-tenant limits on a shared GPU plane. Thanos or Mimir, and what is the tradeoff?**
Mimir. Prometheus becomes a remote-write forwarder; the distributor validates and shards by `fnv32a` hash into a ring (classic) or into Kafka partitions (ingest storage); ingesters hold recent series; a query-frontend, scheduler and queriers serve reads with query sharding and result caching; and per-tenant `max_global_series_per_user`, `ingestion_rate` and read-path limits are first-class. Thanos, by contrast, keeps Prometheus authoritative, ships completed 2-hour blocks via a sidecar (so up to 2 hours are at risk if a Prometheus dies), fans queries out over sidecars and store gateways, and has only convention-level multi-tenancy. The tradeoff is operational surface: Mimir is eight-plus component types, a hash ring, memcached tiers and a runtime-overrides file, versus Thanos's sidecar-plus-bucket increment on an estate you already run.

**Derive a per-shard cardinality ceiling from a 5-minute RTO.**
Measure replay throughput by restarting a shard at a known series count and reading `prometheus_tsdb_data_replay_duration_seconds` — say 1.0 M series in 95 s, i.e. ~10,500 series/s. Budget 300 s, keep ~40% margin for a cold page cache and a busy host: 180 s usable. Ceiling = 180 × 10,500 ≈ 1.89 M series. Then subtract churn headroom: if the head can double in an hour under normal job turnover, the steady-state ceiling is ~945,000. Shard count = total series ÷ ceiling, rounded up and then to a power of two so the next doubling only moves half the targets. Finally, enforce it: a per-shard `head_series` alert, a series-creation-rate alert, and a Mimir `max_global_series_per_user` backstop.

**Why do you need both scrape-time relabelling and distributor-side limits?**
They guard different boundaries. `metric_relabel_configs` prevents the series from ever being created — cheapest possible, and it works without any downstream cooperation — but it only catches what you anticipated, it lives in the scraper's config, and it does not protect you from a tenant pushing OTLP directly. `max_global_series_per_user` rejects whatever gets through, regardless of cause or path, and protects the *shared* ingester fleet from any tenant's unanticipated mistake. There is a third layer people forget: read-path limits (`max_fetched_series_per_query`) stop one dashboard panel from OOMing a querier. A design that names only one layer has an unguarded door.

**Explain the RF=3 quorum property and why it changes your ingester count.**
With replication factor 3, the distributor hashes each series, finds its authoritative owner in the ring and the next two instances clockwise, and considers the write successful once a quorum of `⌊3/2⌋ + 1 = 2` ingesters accept it. So losing one owner is transparent; losing two makes that series unwritable. During a rolling deploy one replica is always down by design, so you provision enough ingesters that a second concurrent failure is unlikely to hit the same token range — which is why the answer is never "the minimum the memory arithmetic allows." Under Mimir's ingest-storage architecture the arithmetic changes: Kafka owns durability and read consistency is a quorum of 1, so you size ingesters for consumption throughput and query fan-out instead.

**Does downsampling save storage? Justify it correctly.**
No. Thanos's compactor documentation states that downsampling adds two more blocks per raw block and increases object-storage footprint roughly 3× when all three resolutions are retained, because each downsampled chunk is an `AggrChunk` carrying five aggregations — count, sum, min, max, counter — so that `rate`, `avg_over_time` and `max_over_time` stay mathematically correct. The justification is **query latency over long ranges**: a one-year range at raw 15 s resolution loads ~2.1 M samples per series, at 1 h resolution ~8,760. The associated trap is retention: each resolution's retention must exceed the age threshold for the next downsampling pass (40 h for raw→5 m, 10 d for 5 m→1 h), or the source blocks are deleted before the downsample can run.

**Your Mimir tenant is silently missing data. Name the two defaults that caused it.**
Mimir's `ingestion_rate` defaults to 10,000 samples/s per tenant, which is far below fleet volume (800,000 series at a 15 s scrape is ~53,000 samples/s), so the distributor returns HTTP 429. Prometheus's `retry_on_http_429` defaults to `false`, so the sender drops the batch instead of retrying. Individually both defaults are defensible; together they convert correct backpressure into silent loss with no error surfaced to the tenant. The fix is both: raise `ingestion_rate` and `ingestion_burst_size` per tenant, set `retry_on_http_429: true`, and bound the retry window with `sample_age_limit` so a long outage cannot prevent WAL truncation and start a second incident.

## Connections & what's next

This lesson operationalises the cardinality budget from [01 · The signal model](01-signal-model.md) — the head-RAM and WAL-replay budgets there become shard counts and per-tenant limits here — and it is where the recording rules from [02 · Prometheus and PromQL](02-prometheus-and-promql.md) move from a single Prometheus into Mimir's ruler so they are evaluated once over a deduplicated global view. The per-tenant limits and rollup rules are the metrics-tier half of what [09 · GPU and ML observability at fleet scale](09-gpu-and-ml-observability.md) synthesises with tracing, logging and profiling into one fleet posture. The storage-cost arithmetic in the worked example is also the first half of the argument in lesson 10: metrics retention is cheap, metrics *cardinality* is expensive, and the questions that need unbounded cardinality over months belong on the second path.

Next: [04 — OpenTelemetry](04-opentelemetry.md), which covers how telemetry actually gets collected and shipped into this storage tier at fleet scale — and adds the third cardinality-enforcement gate for the paths where there is no scrape to relabel.

## References & further reading

**Primary sources — read directly from the repositories**
- Grafana Mimir 3.2.0-rc.0, `docs/sources/mimir/manage/run-production-environment/planning-capacity.md` — every capacity constant in §10 (1 core + 2.5 GB + 5 GB per 300,000 in-memory ingester series; 13 GB store-gateway disk per 1 M active series and the assumptions behind it; one compactor per 20 M active series; the +50% headroom recommendation).
- Grafana Mimir 3.2.0-rc.0, `docs/sources/mimir/references/architecture/hash-ring/index.md` — `fnv32a` series hashing, ring ownership and clockwise replica walk, the classic-architecture quorum of 2 at RF=3, and the ingest-storage architecture's partitions ring, ingesters ring and quorum-of-1 read path. **This corrects the previous version of this lesson**, which gave the quorum formula as `⌈N/2⌉+1` (= 3 for RF=3) rather than `⌊N/2⌋+1` (= 2).
- Grafana Mimir 3.2.0-rc.0, `docs/sources/mimir/references/architecture/components/distributor.md` — validation checks, per-tenant request/ingestion rate limiting, the HTTP 429 behaviour, the HA tracker, and the L7-load-balancer requirement.
- Grafana Mimir 3.2.0-rc.0, `docs/sources/mimir/configure/configuration-parameters/index.md` — `max_global_series_per_user` (default 150,000), `max_global_series_per_metric` (0), `ingestion_rate` (10,000 samples/s), `ingestion_burst_size` (200,000).
- Thanos 0.43.0-dev, `docs/components/compact.md` — the 40-hour and 10-day downsampling thresholds, the `AggrChunk` protobuf with its five aggregations, the explicit statement that downsampling **increases** storage ~3× rather than saving it, and the retention-vs-threshold rule. **This corrects** the previous version's implication that downsampling is a storage optimisation.
- Prometheus 3.14.0, `docs/configuration/configuration.md` — the complete `queue_config` defaults table in §2 (`capacity: 10000`, `max_shards: 50`, `min_shards: 1`, `max_samples_per_send: 2000`, `batch_send_deadline: 5s`, `min_backoff: 30ms`, `max_backoff: 5s`, `retry_on_http_429: false`, `sample_age_limit: 0s`).
- Prometheus 3.14.0, `storage/remote/queue_manager.go` and `storage/remote/write.go` — the `prometheus_remote_storage_*` metric names used in §2, including `samples_pending`, `samples_dropped_total`, `shards_desired`, `queue_highest_sent_timestamp_seconds` and `highest_timestamp_in_seconds`.
- Prometheus 3.14.0, `tsdb/db.go`, `tsdb/head.go`, `tsdb/wlog/wlog.go` — `DefaultBlockDuration = 2h`, the head-compaction trigger at 1.5× the chunk range, 128 MiB WAL segments with 32 KiB pages, and the `prometheus_tsdb_*` metric names.
- [Prometheus federation documentation](https://prometheus.io/docs/prometheus/latest/federation/) — the hierarchical and cross-service use cases, and the `match[]` semantics behind §5. Repo: `docs/federation.md`.
- [Grafana Mimir architecture reference](https://grafana.com/docs/mimir/latest/references/architecture/) — the component overview; read alongside the repo files above.
- [Thanos design documentation](https://thanos.io/tip/thanos/design.md/) — the component model and Store API.

**Real-world engineering write-ups**
- Cloudflare, [How Cloudflare runs Prometheus at scale](https://blog.cloudflare.com/how-cloudflare-runs-prometheus-at-scale/)
- Uber, [M3: Uber's Open Source, Large-scale Metrics Platform for Prometheus](https://www.uber.com/en-IN/blog/m3/)
- Groww Engineering, [Migration from Thanos to Grafana Mimir](https://tech.groww.in/migration-from-thanos-to-grafana-mimir-cab21eed7311)

**Sources consulted but not relied upon.** Several vendor documentation domains are unreachable from this environment's egress proxy. Where a page could not be fetched, the fact was verified against the cloned upstream repository instead (as itemised above) or omitted. The object-storage price used in the worked example (~$0.021/GB-month) is a representative standard-tier figure for arithmetic purposes only — substitute your provider's current published rate before quoting a cost in a design review.
