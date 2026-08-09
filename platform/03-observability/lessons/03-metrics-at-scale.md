---
lesson: "A03.3"
title: "Metrics at scale"
module: "A-03"
concept: "cardinality, sharding, long-term storage"
status: not-started
est_time: "3 hrs"
artifacts: ["fleet-observability design"]
---

# A03.3 · Metrics at scale

> **Concept.** A metrics stack dies from head-block cardinality against RAM, not disk — and the fix is sharding scrape + remote-write into Mimir/Thanos, not federation.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Why this matters
Every senior can stand up Prometheus. What separates staff is knowing the exact shape of the wall it hits at fleet scale, naming the alert that fires before the outage, and choosing between Thanos and Mimir with a defensible reason rather than a vibe. On a thousands-of-node GPU fleet this is not academic: a single Prometheus falls over, and the recovery path (WAL replay) is itself a multi-tens-of-minutes outage. This is a standard CoreWeave/Lambda-tier interview scenario and a real 3am page.

## Core notes
**Skip (you already know):** single-Prometheus scrape and PromQL, local TSDB on disk, that you eventually need some remote long-term storage.

**How the system actually falls over.** It's almost never disk. The failure is **cardinality → RAM**. Prometheus keeps an in-memory **head block** holding one in-RAM series object per active series, plus a write-ahead log (WAL). A cardinality spike — a bad deploy that starts emitting a `pod_hash` or `request_id` label — multiplies active series, ballooning the head and WAL until the process is **OOM-killed**. Recovery is the second trap: on restart Prometheus **replays the WAL** before it can scrape or serve queries, and a bloated WAL means tens of minutes of hard downtime. So the outage is cardinality-triggered but the *duration* is WAL-replay-bound.

The **second failure mode** is silent: remote-write **queue back-pressure**. When the remote endpoint (Mimir/Thanos) slows, the remote-write shards saturate, the pending-samples queue fills, and samples are **dropped silently** — your global view goes wrong with no error in the app.

**Alert on the leading indicators, not the OOM:**
- `prometheus_tsdb_head_series` — and specifically its *growth rate*; a sharp derivative is a cardinality incident in progress.
- `prometheus_remote_storage_samples_pending` (and `_failed_total`) — rising pending = back-pressure before drops.
- WAL replay duration / `prometheus_tsdb_wal_replay_duration_seconds` — governs your recovery-time budget.

**The three long-term-storage architectures.**
- **Thanos** — Prometheus stays the source of truth; a **sidecar** ships completed TSDB blocks to object storage; **Store Gateway** serves historical blocks and **Querier** fans out across sidecars/stores and **dedups** HA replica pairs. Cheap to bolt onto an existing Prometheus estate. Weak point: **query latency at scale** (fan-out over many stores/blocks).
- **Mimir** — modern Cortex fork; Prometheus becomes a **dumb remote-write forwarder**. Path is **distributor → ingester → store-gateway**, sharded over a **hash ring**. Best raw scale, best query performance, native **multi-tenancy** with per-tenant limits. Cost: materially more infra to run.
- **Cortex** — the ancestor Mimir forked from. Know it exists and that Mimir superseded it; **don't pick it for anything new.**

**Sharding.** Two axes: **functional sharding** (a Prometheus pair per team/tenant/region — natural blast-radius boundaries) and **hashmod scrape sharding** (`hashmod` on `__address__` across N Prometheis so each scrapes ~1/N of targets). Both remote-write into Mimir/Thanos so you still get a single global view. **Downsampling** (Thanos 5m and 1h resolutions; Mimir compactor equivalents) is what makes multi-year retention actually queryable instead of timing out.

**Federation is not scaling.** Hierarchical federation is designed to pull **aggregates** up a tree, not raw series. Reaching for `/federate` to move whole workloads' series to a parent is a well-known anti-pattern — it just moves the cardinality-vs-RAM wall up one level. Scale with remote-write + sharding; use federation only for genuine cross-Prometheus rollups.

**GPU-fleet tie (reference the DCGM/util-lie/MFU artifact, don't re-derive).** A thousands-of-node GPU fleet is precisely where single Prometheus dies: DCGM exporters emit many series per GPU per node. Staff pattern: **hashmod-shard the DCGM scrape** across a Prometheus pool, **remote-write into Mimir** with **per-tenant `max_global_series_per_user`** limits so one team's cardinality spike can't OOM the shared plane, push **recording rules** for the fleet rollups (per-cluster utilization, allocation), and **downsample** for capacity-planning history.

## Worked example
**Scenario: 4000 GPU nodes, 30-day hi-res + 2-year downsampled.**

1. **Series count.** Assume 8 GPUs/node and DCGM emitting ~40 series/GPU (util, mem, SM clocks, power, ECC, XID, temp, per-GPU labels), plus ~200 node-level series. Per node ≈ 8×40 + 200 ≈ 520. Fleet ≈ 4000 × 520 ≈ **2.1M active series** for GPU/node telemetry alone — before app metrics. A single Prometheus head at a few million series with churn is squarely in OOM territory.

2. **Shard count.** Budget a conservative ~1M active series per Prometheus shard (leaving headroom for spikes and WAL). 2.1M ÷ ~1M ⇒ **hashmod across 3–4 scrape shards**; round to **4** for spike headroom and even blast radius. Each shard scrapes ~1000 nodes and remote-writes to Mimir.

3. **Mimir ingesters.** Ingesters hold the recent write path in-memory with replication factor 3. Sizing ~1.5–2M series per ingester and RF=3 over 2.1M base ⇒ ~6.3M series-slots ÷ ~1.5M ⇒ **~4–5 ingester replicas minimum**, provision **6** for rollout/headroom. Set **`max_global_series_per_user`** per tenant so the fleet tenant is capped (e.g. 4M) and a runaway `pod_hash` label is rejected at the distributor instead of OOMing an ingester.

4. **Retention.** 30-day raw in ingester→store-gateway/object storage; **Thanos/Mimir downsampling to 5m and 1h** for the 2-year capacity-history tier so `range` queries over years stay fast.

5. **The two fall-over alerts.**
   - `rate(prometheus_tsdb_head_series[10m])` on each shard — page on sustained sharp growth (cardinality incident).
   - `prometheus_remote_storage_samples_pending` trending up (and `_failed_total > 0`) — page before samples are silently dropped into Mimir.

## Practice
<feeds [fleet observability design](../practice/fleet-observability/README.md)>

Design the metrics tier for the 4000-node fleet: pick functional vs hashmod sharding (or both), choose Mimir vs Thanos with an explicit justification tied to query performance and multi-tenancy, write the per-tenant limit config, define the recording rules for fleet rollups, set the downsampling/retention tiers, and specify the exact two leading-indicator alerts with thresholds. Deliver it as the metrics section of the fleet-observability design doc.

## Self-check
- What resource does a Prometheus actually run out of first under a cardinality spike, and why does recovery make the outage worse? **Answer:** RAM — the in-memory head block plus WAL balloon with active series and the process is OOM-killed; recovery is worse because restart replays the (now bloated) WAL before serving, adding tens of minutes of downtime on top.
- Why is `/federate` the wrong tool for scaling out a large fleet's metrics, and what do you use instead? **Answer:** Federation is built to pull aggregates up a hierarchy, not raw series; using it to move whole workloads just relocates the cardinality-vs-RAM wall to the parent. Scale with hashmod/functional scrape sharding + remote-write into Mimir/Thanos, with downsampling for retention.
- You need best-in-class query performance and native per-tenant limits on a shared GPU plane — Thanos or Mimir, and what's the tradeoff? **Answer:** Mimir — Prometheus becomes a dumb remote-write forwarder into a hash-ring-sharded distributor→ingester→store-gateway pipeline with native multi-tenancy and `max_global_series_per_user`; the tradeoff is materially more infrastructure to operate than bolting a Thanos sidecar onto existing Prometheis.

## References
- Mimir architecture: https://grafana.com/docs/mimir/latest/references/architecture/
- Thanos design: https://thanos.io/tip/thanos/design.md/
- Prometheus federation (and why it isn't scaling): https://prometheus.io/docs/prometheus/latest/federation/
- Prometheus remote-write tuning: https://prometheus.io/docs/practices/remote_write/
