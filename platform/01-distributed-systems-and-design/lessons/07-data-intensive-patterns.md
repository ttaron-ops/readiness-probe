---
lesson: "A01.7"
title: "Data-intensive patterns"
module: "A-01"
concept: "log-as-truth, CDC, delivery semantics"
status: not-started
est_time: "4 hrs"
artifacts: ["gpu-cost-usage-pipeline.md"]
---

# A01.7 · Data-intensive patterns

> **Concept.** The log is the source of truth; everything else is a materialized view. Delivery semantics have exact boundaries and exact costs — name them, then choose the cheapest one that holds.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Why this matters
On a GPU platform the two highest-blast-radius data pipelines are fleet telemetry (a gray failure here blinds every autoscaler and on-call) and GPU-second billing (a double-count here ships wrong invoices to customers). Both live or die on delivery semantics you can state precisely and defend by cost. Staff signal is refusing the "exactly-once, everywhere, for free" hand-wave and instead naming the exact guarantee, its price, and where it degrades.

## Core notes
**Skip (you already know):** Kafka is a durable partitioned log; events decouple producers from consumers; "make the consumer idempotent"; batch-vs-stream at the tutorial level.

**The log as source of truth.** Model the system "database inside-out": an immutable, append-only, totally-ordered (per-partition) log is truth; tables, indexes, caches, and search are *derived materialized views* you can rebuild by replaying the log. This is why log-first simplifies replication and CDC — every consumer is just a deterministic fold over the same ordered stream, and a new replica is "start from offset 0." **Log compaction** keeps the latest value per key indefinitely (tombstones expire deletes), giving you a compacted changelog that doubles as a bootstrap snapshot without unbounded retention. The offset *is* the consumer's position; lag is measurable, replay is free.

**CDC: log-based vs query-based.** Log-based CDC (Debezium reading the DB WAL / MySQL binlog) is the staff default: low overhead (no query load on the source), preserves commit order, and — critically — **captures deletes** and intra-poll-interval updates. Query-based CDC (polling `WHERE updated_at > last`) misses deletes entirely, misses multiple updates between polls, adds read load, and its staleness is bounded only by poll frequency (which you pay for in load). Reach for query-based only when you cannot get at the WAL.

**Delivery semantics — the exact boundaries.**
- *At-most-once:* fire and forget; drops on failure. Fine for lossy metrics.
- *At-least-once:* retry until acked; duplicates on the retry path. The default transport contract.
- *Exactly-once:* **no free lunch.** True exactly-once *across a boundary* (transport → sink) requires either a distributed transaction / 2PC spanning both, or an **idempotent sink** that absorbs the duplicates at-least-once inevitably produces. There is no third option.
- **Kafka EOS** = idempotent producer (per-`<producer,partition>` sequence numbers dedup broker-side on retry) **+** transactions (atomic multi-partition writes coordinated by a transaction coordinator; `read_committed` consumers skip aborted records). This gives exactly-once *within the Kafka boundary* (read-process-write Kafka→Kafka), not automatically to your external DB.
- **Effectively-once** = at-least-once delivery + an idempotent / dedup'd sink (keyed upsert, or a dedup table on a business key). Usually far cheaper than EOS and what most teams actually ship.
- **Flink end-to-end exactly-once** = Chandy–Lamport barrier (asynchronous distributed snapshot) checkpoints for consistent operator state + a **2-phase-commit sink** (pre-commit on checkpoint, commit on checkpoint-complete). The deciding cost: checkpoint interval sets recovery replay and adds barrier-alignment latency.

**Kappa vs Lambda; why you still keep a batch path.** Kappa = one streaming code path, reprocess by replaying the log from an earlier offset. Lambda = streaming layer for fresh/approximate + batch layer for correct/complete, reconciled. Even in a Kappa shop you keep a **batch backfill path** — for reprocessing after a logic bug, for late/out-of-order data beyond your watermark, and for cold-start rebuilds of a view. The reconciliation job that recomputes truth from the log is your correctness backstop.

**GPU ties.** (1) *Fleet telemetry* — DCGM/GPU metrics are high-cardinality (per-GPU × per-node × per-job) *and* high-volume. Ship at-least-once, aggregate idempotently, downsample early, and apply backpressure so the observability pipeline never becomes the fleet-wide gray failure that hides the outage it should surface. (2) *Cost/usage streams* — GPU-second billing must be effectively-once; double-counting is a customer-visible billing bug. Design an idempotency key (`job_id + interval`) and a dedup'd/exactly-once rollup.

## Worked example
**Design: GPU-minute cost pipeline, 10,000 GPUs.** Each GPU emits a usage event per minute → ~10k events/min ≈ 167/s steady, bursty on job churn. Transport is at-least-once, so expect duplicates and out-of-order arrival (a node's buffered events flush late after a network blip).

*Idempotency key.* `(job_id, gpu_uuid, minute_bucket)` — one canonical billable unit. The event carries `{key, gpu_minutes, emit_ts}`.

*What breaks naively.* A streaming `SUM(gpu_minutes) GROUP BY job_id` over an at-least-once stream double-counts every redelivered event → invoice inflates. Out-of-order arrival makes a windowed SUM close a window before late events land → invoice deflates. Both are silent.

*The fix — keyed upsert (effectively-once).* Land raw events into a table keyed by the idempotency key with `UPSERT` (last-writer-wins on identical key; a redelivery overwrites its own row, contributing exactly once). The monthly rollup is `SUM` over the *deduplicated keyed table*, not over the stream. Late events simply upsert into their bucket and are picked up on the next rollup pass. Result: at-least-once transport + idempotent keyed sink = exactly-once invoice, no 2PC, no distributed transaction. Choose a bucket-close grace period (e.g. bill T+24h) so late arrivals settle before the invoice is cut — that grace window is the deciding number trading freshness for correctness.

## Practice
Write up the GPU cost/usage pipeline as `gpu-cost-usage-pipeline.md`: the idempotency key, the dedup/upsert model, the SUM-over-keyed-table rollup, the grace window, and an explicit statement of which guarantee (effectively-once) you chose and why not EOS. Feeds the [staff design portfolio](../practice/staff-design-portfolio/README.md).

## Self-check
- Why can at-least-once + an idempotent sink be *cheaper* than Kafka EOS while delivering an exactly-once business result? **Answer:** It moves dedup to a keyed upsert on a business key, avoiding the transaction coordinator, `read_committed` overhead, and the throughput cost of transactional writes — you pay only for the sink's upsert, and duplicates are absorbed rather than prevented.
- What does log-based CDC catch that query-based polling structurally cannot, and why? **Answer:** Deletes (and every intermediate update between polls) — because it reads the WAL/binlog, which records the actual delete and every committed change in commit order, whereas `WHERE updated_at > x` only ever sees currently-present rows at poll time.
- In Flink end-to-end exactly-once, what are the two mechanisms and what sets recovery cost? **Answer:** Chandy–Lamport barrier checkpoints for consistent operator state plus a 2-phase-commit sink (pre-commit on checkpoint, commit on complete); the checkpoint interval sets how much you replay on recovery and adds barrier-alignment latency.

## References
- https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/
- https://www.confluent.io/blog/transactions-apache-kafka/
- https://debezium.io/documentation/
- https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/
- https://nightlies.apache.org/flink/flink-docs-stable/docs/learn-flink/fault_tolerance/
