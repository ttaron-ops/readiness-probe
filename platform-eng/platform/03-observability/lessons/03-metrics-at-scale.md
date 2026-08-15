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
sources: 7
---

# A03.3 · Metrics at scale

> **Concept.** A metrics stack dies from head-block cardinality against RAM, not disk — and the fix is sharding scrape + remote-write into Mimir/Thanos, not federation.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits
Lesson 02 gave you the PromQL traps that ship a wrong single-Prometheus dashboard. This lesson is what happens when that single Prometheus is asked to hold a fleet: the same cardinality pressure that silently corrupts a query now silently kills the process, and recovery from that kill is itself an outage. Once you can size and shard the storage tier correctly, the next question is how telemetry actually gets *into* it at the collection edge across hundreds of heterogeneous services — that's lesson 04's Collector architecture.

## Why this matters
Every senior can stand up Prometheus. What separates staff is knowing the exact shape of the wall it hits at fleet scale, naming the alert that fires before the outage, and choosing between Thanos and Mimir with a defensible reason rather than a vibe. On a thousands-of-node GPU fleet this is not academic: a single Prometheus falls over, and the recovery path (WAL replay) is itself a multi-tens-of-minutes outage. This is a standard CoreWeave/Lambda-tier interview scenario and a real 3am page.

## What's new here (calibration)
- The growth *model* behind a cardinality number, not just a point-in-time series count — churn, not node count, is usually the dominant term on a scheduled fleet.
- WAL replay time reframed as a stated RTO you design retention/cardinality limits backward from, not an incident-time surprise.
- Per-tenant series limits as a cost-allocation lever, not just a blast-radius control.
- The actual (non-technical) reason Cortex forked into Mimir — a real staff-interview question with a real answer.

## Core concepts
**Skip (you already know):** single-Prometheus scrape and PromQL, local TSDB on disk, that you eventually need some remote long-term storage.

**How the system actually falls over.** It's almost never disk. The failure is **cardinality → RAM**. Prometheus keeps an in-memory **head block** holding one in-RAM series object per active series, plus a write-ahead log (WAL). A cardinality spike — a bad deploy that starts emitting a `pod_hash` or `request_id` label — multiplies active series, ballooning the head and WAL until the process is **OOM-killed**. Recovery is the second trap: on restart Prometheus **replays the WAL** before it can scrape or serve queries, and a bloated WAL means tens of minutes of hard downtime. So the outage is cardinality-triggered but the *duration* is WAL-replay-bound.

**WAL replay is a hidden SLA.** WAL replay time scales roughly linearly with WAL size since the last checkpoint (default checkpoint interval: 2h), and that size is roughly `(active series churn rate) × (time since last checkpoint) × (bytes/sample)`. A cardinality spike doesn't just blow up the head — it grows replay cost proportionally, which is the actual mechanism behind "recovery makes the outage worse." The staff move is to invert this: state a target WAL-replay budget (e.g. under 5 minutes, because that's your real RTO after *any* Prometheus restart — a routine deploy, a node eviction, an OOM, not just a cardinality incident) and work backward to the retention/cardinality ceiling that guarantees it. This turns cardinality budgeting from a "keep it reasonable" guideline into an availability decision with a number attached.

**The second failure mode is silent:** remote-write **queue back-pressure**. When the remote endpoint (Mimir/Thanos) slows, the remote-write shards saturate, the pending-samples queue fills, and samples are **dropped silently** — your global view goes wrong with no error in the app.

**Compaction is a third, quieter failure mode.** The Thanos/Mimir compactor is a stateful, single-writer-per-tenant background process that merges and downsamples blocks in object storage. At very high block counts it can itself become the bottleneck — compaction falls behind, query-time block fan-out grows, and store-gateway memory climbs. It doesn't page like an OOM does; it shows up weeks later as "why are historical range queries suddenly slow."

**Alert on the leading indicators, not the OOM:**
- `prometheus_tsdb_head_series` — and specifically its *growth rate*; a sharp derivative is a cardinality incident in progress.
- `prometheus_remote_storage_samples_pending` (and `_failed_total`) — rising pending = back-pressure before drops.
- `prometheus_tsdb_wal_replay_duration_seconds` — governs your recovery-time budget; alert when trend threatens your stated WAL-replay SLA, not just on the value after the fact.

**The three long-term-storage architectures.**
- **Thanos** — Prometheus stays the source of truth; a **sidecar** ships completed TSDB blocks to object storage; **Store Gateway** serves historical blocks and **Querier** fans out across sidecars/stores and **dedups** HA replica pairs. Cheap to bolt onto an existing Prometheus estate. Weak point: **query latency at scale** (fan-out over many stores/blocks).
- **Mimir** — modern Cortex fork; Prometheus becomes a **dumb remote-write forwarder**. Path is **distributor → ingester → store-gateway**, sharded over a **hash ring**. Best raw scale, best query performance, native **multi-tenancy** with per-tenant limits. Cost: materially more infra to run.
- **Cortex** — the ancestor Mimir forked from. Know it exists and that Mimir superseded it; **don't pick it for anything new.**

**Why Mimir exists as a separate project from Cortex at all** is a real staff-interview question, and the honest answer is business/licensing, not pure technology: the fork was driven by open-source sustainability and licensing economics (Grafana Labs relicensing its stack, including replacing Cortex with Mimir under Apache 2.0 to control the commercial terms of its own product), not by Cortex being technically broken. If you're asked "why not just improve Cortex," the correct answer references governance and licensing, not a laundry list of bugs.

**Sharding.** Two axes: **functional sharding** (a Prometheus pair per team/tenant/region — natural blast-radius boundaries) and **hashmod scrape sharding** (`hashmod` on `__address__` across N Prometheis so each scrapes ~1/N of targets). Both remote-write into Mimir/Thanos so you still get a single global view. **Downsampling** (Thanos 5m and 1h resolutions; Mimir compactor equivalents) is what makes multi-year retention actually queryable instead of timing out.

**Cardinality defense is layered — and both layers matter.** Cardinality limiting at the distributor (Mimir's `max_global_series_per_user`) and cardinality limiting at scrape time (`relabel_configs` dropping labels/series before they're ever scraped) are not redundant — they're two different failure boundaries. Relabeling prevents a runaway series from ever entering the pipeline at all (cheapest, but only catches what you anticipated). The distributor limit is the backstop that rejects whatever gets through relabeling anyway and exceeds a tenant's quota, protecting the shared ingester fleet from *any* tenant's mistake, anticipated or not. A design that specifies only one layer is missing half the defense.

**The growth model behind a series-count number.** A point-in-time series count is a snapshot, not a plan. On a GPU fleet, active series scales as roughly `node count × GPU density × label-churn rate`, and **churn** — pods restarting, training jobs turning over, autoscaling — is very often the dominant term, not raw node growth. A fleet that stays flat at 4000 nodes but runs high job turnover (many short-lived training/inference pods, each contributing new `pod_hash`/`job_id`-flavored series before the old ones expire) can grow active series faster than a fleet that doubles in node count with stable long-running pods. Sizing for headroom means sizing for your churn rate, not just your node count.

**Federation is not scaling.** Hierarchical federation is designed to pull **aggregates** up a tree, not raw series. Reaching for `/federate` to move whole workloads' series to a parent is a well-known anti-pattern — it just moves the cardinality-vs-RAM wall up one level. Scale with remote-write + sharding; use federation only for genuine cross-Prometheus rollups.

**GPU-fleet tie (reference the DCGM/util-lie/MFU artifact, don't re-derive).** A thousands-of-node GPU fleet is precisely where single Prometheus dies: DCGM exporters emit many series per GPU per node. Staff pattern: **hashmod-shard the DCGM scrape** across a Prometheus pool, **remote-write into Mimir** with **per-tenant `max_global_series_per_user`** limits so one team's cardinality spike can't OOM the shared plane, push **recording rules** for the fleet rollups (per-cluster utilization, allocation), and **downsample** for capacity-planning history.

## Perspectives
**Capacity-planning.** The worked example's sizing (2.1M series → 4 shards → 6 Mimir ingesters) is a point-in-time snapshot. Staff depth means owning the growth model behind it: series count scales with node count × GPU density × label churn, and on a scheduled GPU fleet, churn from job turnover and pod restarts is often the term that grows fastest — meaning a static sizing exercise re-run only when node count changes will consistently under-provision.

**Operability.** WAL replay time is a hidden SLA. It determines your actual RTO after *any* Prometheus restart — not just a cardinality incident, but every routine deploy, node eviction, or OOM. State a target replay budget (e.g. under 5 minutes) and derive your retention/cardinality ceiling from it; this reframes "keep cardinality under control" from a hygiene guideline into an availability commitment with a number.

**Multi-tenancy / economics.** Mimir's `max_global_series_per_user` is simultaneously a blast-radius control and a cost-allocation mechanism. Per-tenant series limits let you charge back cardinality the same way you'd charge back compute — directly relevant when GPU tenants are internal cost centers and "your team's label explosion degraded the shared plane" needs to become a bill, not a postmortem action item.

**Architecture-history.** Why Cortex forked into Mimir is a real staff-interview angle with a real, non-technical answer: licensing and open-source business sustainability (Grafana Labs moving its stack to Apache 2.0 licensing it controls), not a technical rewrite driven by Cortex being broken. Knowing this distinguishes "I read the architecture doc" from "I understand why the ecosystem looks the way it does."

## Real-world use cases
- **Cloudflare, "How Cloudflare runs Prometheus at scale"** — https://blog.cloudflare.com/how-cloudflare-runs-prometheus-at-scale/ — runs 900+ Prometheus instances functionally-sharded by team/service; a real production example of the "functional sharding" axis at GPU-fleet-comparable scale.
- **Uber, "M3: Uber's Open Source, Large-scale Metrics Platform for Prometheus"** — https://www.uber.com/en-IN/blog/m3/ — aggregates 500M metrics/sec and persists 20M/sec with replication-factor-3 quorum writes, directly comparable to the RF=3 Mimir-ingester sizing math in this lesson's worked example.
- **Groww Engineering, "Migration from Thanos to Grafana Mimir"** — https://tech.groww.in/migration-from-thanos-to-grafana-mimir-cab21eed7311 — a concrete production migration write-up covering the distributor→ingester→store-gateway cutover and historical block import, useful as a template for a real Thanos-to-Mimir migration plan.

## Worked example
**Scenario: 4000 GPU nodes, 30-day hi-res + 2-year downsampled.**

1. **Series count.** Assume 8 GPUs/node and DCGM emitting ~40 series/GPU (util, mem, SM clocks, power, ECC, XID, temp, per-GPU labels), plus ~200 node-level series. Per node ≈ 8×40 + 200 ≈ 520. Fleet ≈ 4000 × 520 ≈ **2.1M active series** for GPU/node telemetry alone — before app metrics. A single Prometheus head at a few million series with churn is squarely in OOM territory.

2. **Shard count.** Budget a conservative ~1M active series per Prometheus shard (leaving headroom for spikes and WAL). 2.1M ÷ ~1M ⇒ **hashmod across 3–4 scrape shards**; round to **4** for spike headroom and even blast radius. Each shard scrapes ~1000 nodes and remote-writes to Mimir.

3. **Mimir ingesters.** Ingesters hold the recent write path in-memory with replication factor 3. Sizing ~1.5–2M series per ingester and RF=3 over 2.1M base ⇒ ~6.3M series-slots ÷ ~1.5M ⇒ **~4–5 ingester replicas minimum**, provision **6** for rollout/headroom. Set **`max_global_series_per_user`** per tenant so the fleet tenant is capped (e.g. 4M) and a runaway `pod_hash` label is rejected at the distributor instead of OOMing an ingester.

4. **The RF=3 quorum math.** With replication factor N=3, the distributor requires a write quorum of `⌈N/2⌉+1 = 2` successful writes to acknowledge. That means the ingester fleet tolerates losing `⌊N/2⌋ = 1` replica without write failures — lose 2 of the 3 replicas holding a given series and writes to it start failing even though reads may still partially succeed. This is why "provision 6, need ~4–5" isn't just headroom for growth — it's headroom so that a single ingester failure during a rolling deploy doesn't put any series one loss away from write failure.

5. **WAL replay budget.** Target: WAL replay under 5 minutes per shard after any restart. At ~1M active series per shard with a 2h default checkpoint interval, back-calculate the churn rate you can tolerate before replay blows the budget — if churn (from job turnover) is trending up, either shrink the checkpoint interval, add shards, or tighten per-tenant limits before the next restart turns into a 20-minute outage instead of a 5-minute one.

6. **Retention.** 30-day raw in ingester→store-gateway/object storage; **Thanos/Mimir downsampling to 5m and 1h** for the 2-year capacity-history tier so `range` queries over years stay fast, and so the compactor doesn't fall permanently behind on raw-block volume.

7. **The two fall-over alerts.**
   - `rate(prometheus_tsdb_head_series[10m])` on each shard — page on sustained sharp growth (cardinality incident).
   - `prometheus_remote_storage_samples_pending` trending up (and `_failed_total > 0`) — page before samples are silently dropped into Mimir.

## Practice
<feeds [fleet observability design](../practice/fleet-observability/README.md)>

Design the metrics tier for the 4000-node fleet: pick functional vs hashmod sharding (or both), choose Mimir vs Thanos with an explicit justification tied to query performance and multi-tenancy, write the per-tenant limit config (both the relabel-time and distributor-time layers), define the recording rules for fleet rollups, set the downsampling/retention tiers, state your WAL-replay RTO target and the cardinality ceiling it implies, and specify the exact two leading-indicator alerts with thresholds. Deliver it as the metrics section of the fleet-observability design doc.

## Common pitfalls
- **"Thanos and Mimir solve the same problem the same way, pick either."** They don't. Thanos keeps Prometheus as the source of truth and bolts on long-term storage via sidecar — lower operational lift, weaker query performance at scale. Mimir replaces Prometheus's role with a sharded ingest path — higher operational lift, much better scale and multi-tenancy. The choice is a real tradeoff, not a coin flip.
- **"Federation gives us horizontal scale."** It doesn't. The scraping Prometheus still has to hold and expose all the federated series in its own head first — federation *relocates* the cardinality pressure up one level, it doesn't reduce it anywhere.
- **"Remote-write failures are loud."** They aren't. Samples dropped due to queue back-pressure fail *silently* — you must actively alert on `prometheus_remote_storage_samples_pending`/`_failed_total`, because nothing in the app or the dashboard will tell you samples are missing.
- **"More retention just costs more disk."** It doesn't stop there. Longer retention also drives compaction/downsampling CPU and store-gateway memory — long retention without downsampling makes multi-year range queries time out even when disk itself is cheap.

## Self-check
- What resource does a Prometheus actually run out of first under a cardinality spike, and why does recovery make the outage worse? **Answer:** RAM — the in-memory head block plus WAL balloon with active series and the process is OOM-killed; recovery is worse because restart replays the (now bloated) WAL before serving, adding tens of minutes of downtime on top.
- Why is `/federate` the wrong tool for scaling out a large fleet's metrics, and what do you use instead? **Answer:** Federation is built to pull aggregates up a hierarchy, not raw series; using it to move whole workloads just relocates the cardinality-vs-RAM wall to the parent. Scale with hashmod/functional scrape sharding + remote-write into Mimir/Thanos, with downsampling for retention.
- You need best-in-class query performance and native per-tenant limits on a shared GPU plane — Thanos or Mimir, and what's the tradeoff? **Answer:** Mimir — Prometheus becomes a dumb remote-write forwarder into a hash-ring-sharded distributor→ingester→store-gateway pipeline with native multi-tenancy and `max_global_series_per_user`; the tradeoff is materially more infrastructure to operate than bolting a Thanos sidecar onto existing Prometheis.
- Why treat WAL replay duration as an SLA rather than an incident-time detail, and what do you derive from it? **Answer:** Because it governs your real recovery time after *any* Prometheus restart, not just cardinality incidents — a routine deploy or node eviction hits the same replay cost. Stating a target (e.g. under 5 minutes) lets you work backward to the retention/churn ceiling that guarantees it, turning cardinality budgeting into an availability decision.
- Why do you need both relabel-time and distributor-time cardinality limits instead of just one? **Answer:** They defend different boundaries — relabeling prevents anticipated runaway series from ever being scraped (cheapest, but only catches what you anticipated), while the distributor's `max_global_series_per_user` rejects whatever slips through and exceeds a tenant's quota regardless of cause, protecting the shared ingester fleet from any tenant's unanticipated mistake.

## Connections & what's next
Builds on the PromQL correctness traps in [02 — Prometheus and PromQL](02-prometheus-and-promql.md); the per-tenant limits and recording rules here are the metrics-tier half of what [09 — GPU and ML observability at fleet scale](09-gpu-and-ml-observability.md) synthesizes with tracing/logging/profiling into a single fleet observability posture. Next: [04 — OpenTelemetry](04-opentelemetry.md), which covers how telemetry actually gets collected and shipped into this storage tier at fleet scale.

## References & further reading
**Primary sources**
- Mimir architecture: https://grafana.com/docs/mimir/latest/references/architecture/
- Thanos design: https://thanos.io/tip/thanos/design.md/
- Prometheus federation (and why it isn't scaling): https://prometheus.io/docs/prometheus/latest/federation/
- Prometheus remote-write tuning: https://prometheus.io/docs/practices/remote_write/

**Real-world engineering blogs**
- Cloudflare, "How Cloudflare runs Prometheus at scale": https://blog.cloudflare.com/how-cloudflare-runs-prometheus-at-scale/
- Uber, "M3: Uber's Open Source, Large-scale Metrics Platform for Prometheus": https://www.uber.com/en-IN/blog/m3/
- Groww Engineering, "Migration from Thanos to Grafana Mimir": https://tech.groww.in/migration-from-thanos-to-grafana-mimir-cab21eed7311
