---
lesson: "A01.7"
title: "Data-intensive patterns"
module: "A-01"
concept: "log-as-truth, CDC, delivery semantics"
status: not-started
est_time: "6 hrs"
prev: "06-failure-and-resilience.md"
next: "08-system-design-method.md"
artifacts: ["gpu-cost-usage-pipeline.md"]
sources: 8
---

# A01.7 · Data-intensive patterns

> **Concept.** The log is the source of truth; everything else is a materialized view. Delivery semantics have exact boundaries and exact costs — name them, then choose the cheapest one that holds.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits
[Lesson 06](06-failure-and-resilience.md) established that failure detectors lie, timeouts force retries, and retries — even done right, with jitter and budgets — produce **duplicates** on the wire. That lesson stopped at containment: cells, shuffle-sharding, checkpoint intervals to bound how much a failure costs. It never asked what happens *after* a retried, possibly-duplicated event lands in a downstream data pipeline. This lesson answers exactly that: once duplicates and out-of-order arrivals are a fact of life (not a bug to eliminate), what precise guarantee can a data pipeline offer, at what cost, and how do you build one — a billing pipeline, a telemetry stream — that survives them without corrupting the numbers it produces.

## Why this matters
On a GPU platform the two highest-blast-radius data pipelines are fleet telemetry (a gray failure here blinds every autoscaler and on-call) and GPU-second billing (a double-count here ships wrong invoices to customers). Both live or die on delivery semantics you can state precisely and defend by cost. Staff signal is refusing the "exactly-once, everywhere, for free" hand-wave and instead naming the exact guarantee, its price, and where it degrades.

## What's new here (calibration)
- **Skip (you already know):** Kafka is a durable partitioned log; events decouple producers from consumers; "make the consumer idempotent"; batch-vs-stream at the tutorial level.
- **Skip:** consumer groups, partition assignment, and offset commits — you already run these in production and don't need the mechanics re-taught.
- **New depth:** the precise boundary between what a distributed transaction (2PC) buys you and what an idempotent sink buys you — most engineers can name "exactly-once" as a goal but not the two mechanisms that are the *only* ways to approach it.
- **New depth:** how log-based CDC actually bootstraps a full initial snapshot of a live table *without locking it* (the watermark mechanism), and what a money-grade idempotency key looks like end to end, including its expiry lifecycle.

## Core concepts
**The log as source of truth.** Model the system "database inside-out": an immutable, append-only, totally-ordered (per-partition) log is truth; tables, indexes, caches, and search are *derived materialized views* you can rebuild by replaying the log. This is why log-first simplifies replication and CDC — every consumer is just a deterministic fold over the same ordered stream, and a new replica is "start from offset 0." **Log compaction** keeps the latest value per key indefinitely (tombstones expire deletes), giving you a compacted changelog that doubles as a bootstrap snapshot without unbounded retention. The offset *is* the consumer's position; lag is measurable, replay is free.

**CDC: log-based vs query-based.** Log-based CDC (Debezium reading the DB WAL / MySQL binlog) is the staff default: low overhead (no query load on the source), preserves commit order, and — critically — **captures deletes** and intra-poll-interval updates. Query-based CDC (polling `WHERE updated_at > last`) misses deletes entirely, misses multiple updates between polls, adds read load, and its staleness is bounded only by poll frequency (which you pay for in load). Reach for query-based only when you cannot get at the WAL.

**How log-based CDC bootstraps without locking the source — the watermark mechanism.** A new CDC consumer needs two things: the ongoing stream of changes *and* the full existing state of every row, not just rows that change after it starts. The naive way to get the full state is a `SELECT *` snapshot, which either locks the table (unacceptable on a live production DB) or risks missing/duplicating rows that change mid-scan. Netflix's [DBLog](https://netflixtechblog.com/dblog-a-generic-change-data-capture-framework-69351fb9099b) framework solves this with a **watermark**: it interleaves log-tailing with chunked table scans. Before scanning a chunk of rows, it writes a low-watermark marker into the log itself (a synthetic event); after the chunk is read, it writes a high-watermark marker. Any real change events for rows in that chunk that appear *between* the two watermarks in the log are known to be newer than the snapshot read, so the merge logic keeps the log's version and discards the (now-stale) snapshot value for that row. The result: a consistent full-table snapshot assembled from small, resumable chunk reads, fully interleaved with live log events, with **no lock held on the source table at any point**. This is a structurally different mechanism from polling — it is the difference between "eventually catch up by re-querying" and "provably converge to a consistent state via ordering against the log." DBLog has run in production across tens of Netflix microservices since around 2018.

**Delivery semantics — the exact boundaries.**
- *At-most-once:* fire and forget; drops on failure. Fine for lossy metrics.
- *At-least-once:* retry until acked; duplicates on the retry path. The default transport contract.
- *Exactly-once:* **no free lunch.** True exactly-once *across a boundary* (transport → sink) requires either a distributed transaction / 2PC spanning both, or an **idempotent sink** that absorbs the duplicates at-least-once inevitably produces. There is no third option.
- **Kafka EOS** = idempotent producer (per-`<producer,partition>` sequence numbers dedup broker-side on retry) **+** transactions (atomic multi-partition writes coordinated by a transaction coordinator; `read_committed` consumers skip aborted records). This gives exactly-once *within the Kafka boundary* (read-process-write Kafka→Kafka), not automatically to your external DB.
- **Effectively-once** = at-least-once delivery + an idempotent / dedup'd sink (keyed upsert, or a dedup table on a business key). Usually far cheaper than EOS and what most teams actually ship.
- **Flink end-to-end exactly-once** = Chandy–Lamport barrier (asynchronous distributed snapshot) checkpoints for consistent operator state + a **2-phase-commit sink** (pre-commit on checkpoint, commit on checkpoint-complete). The deciding cost: checkpoint interval sets recovery replay and adds barrier-alignment latency.

**Idempotency-key design and lifecycle — the production template.** Stripe's public [idempotency design](https://stripe.com/blog/idempotency) is the canonical worked reference: the client generates a unique key (Stripe recommends a V4 UUID) and sends it in an `Idempotency-Key` header on every mutating request. The server, on first receipt, executes the request and **persists the resulting status code and body** — including error responses — keyed by that idempotency key. Any retry with the same key, within the retention window, short-circuits straight to the stored response without re-executing the side effect. Stripe prunes keys after roughly 24 hours; a retry that arrives after the key is pruned is treated as a brand-new request. Two details matter for your own designs: (1) errors are cached too, so a client that retries after a definitive failure gets the same failure back, not a second attempt at a possibly-now-inconsistent operation; (2) reusing a key with a *different* request body is treated as a conflict, not silently accepted — this is what stops a key collision from masking a bug. This is the exact shape the worked example below reuses for GPU billing.

**Kappa vs Lambda; why you still keep a batch path.** Kappa = one streaming code path, reprocess by replaying the log from an earlier offset. Lambda = streaming layer for fresh/approximate + batch layer for correct/complete, reconciled. Even in a Kappa shop you keep a **batch backfill path** — for reprocessing after a logic bug, for late/out-of-order data beyond your watermark, and for cold-start rebuilds of a view. The reconciliation job that recomputes truth from the log is your correctness backstop.

**GPU ties.** (1) *Fleet telemetry* — DCGM/GPU metrics are high-cardinality (per-GPU × per-node × per-job) *and* high-volume. Ship at-least-once, aggregate idempotently, downsample early, and apply backpressure so the observability pipeline never becomes the fleet-wide gray failure that hides the outage it should surface. (2) *Cost/usage streams* — GPU-second billing must be effectively-once; double-counting is a customer-visible billing bug. Design an idempotency key (`job_id + interval`) and a dedup'd/exactly-once rollup.

## Perspectives
**The log/storage-engine view.** The mechanism that makes log-based CDC possible is the database's own write-ahead log — the same structure the storage engine uses for crash recovery. Reading it is fundamentally different from querying the table: the WAL records every committed mutation, including deletes and intermediate states, in exact commit order, with no additional query load on the live table. Query-based polling has no equivalent structure to read; it can only re-derive current state, never the history of how it got there.

**The delivery-semantics / distributed-transactions view.** "Exactly-once" is not a delivery mode a broker can just switch on; it names a boundary problem. Every system that claims it is really doing one of two things: coordinating a transaction across the boundary (2PC — expensive, and only as available as its weakest participant), or making the *effect* of duplicate delivery a no-op at the sink (idempotency — cheaper, and the far more common production choice). Kafka's own "exactly-once semantics" is scoped to Kafka-to-Kafka; the instant your pipeline writes to an external database, you are back to choosing between these two mechanisms.

**The billing/financial-correctness view.** A dashboard metric that's 0.1% off for five minutes is invisible. An invoice that's 0.1% inflated is a support ticket, a refund, and — at scale — a trust problem. This asymmetry is why billing pipelines get the stricter treatment: an explicit idempotency key tied to the exact billable unit, a persisted-response or keyed-upsert mechanism, and a deliberate grace window before the number is considered final. "Close enough" is a valid engineering answer for telemetry and an unacceptable one for money.

**The operability/backfill view.** Even a team that has fully committed to a Kappa (streaming-only) architecture keeps a batch path alive, because streaming exactly-once guarantees only ever cover the failure modes you anticipated when you wrote the pipeline. A logic bug discovered three weeks later, data that arrives a month late, or a brand-new materialized view that needs to be built from scratch all require replaying history at a scale and pace no streaming operator is built for. The batch reconciliation job — recompute truth straight from the log — is the correctness backstop that makes the streaming path's guarantees trustworthy in the first place.

## Real-world use cases
- **Netflix, "DBLog: A Generic Change-Data-Capture Framework"** — https://netflixtechblog.com/dblog-a-generic-change-data-capture-framework-69351fb9099b — a real, detailed log-based CDC system built and run in production since around 2018 across tens of Netflix microservices; shows exactly how log-based CDC bootstraps a full initial snapshot of a live table without ever locking it, via interleaved watermarks. This is the mechanism reference for the "log-based vs query-based" framing above.
- **Stripe, "Designing Robust and Predictable APIs with Idempotency"** — https://stripe.com/blog/idempotency — the canonical production idempotency-key design: a client-generated key in an `Idempotency-Key` header, the server persisting the first response (including errors) and replaying it on retry, with a 24-hour key retention window. Direct real-world template for this lesson's own `job_id + interval` billing idempotency key.
- **PagerDuty, "August 28 Kafka Outages — What Happened and How We're Improving"** — https://www.pagerduty.com/eng/august-28-kafka-outages-what-happened-and-how-were-improving/ — a runaway Kafka producer bug (millions of extra producer connections per hour) destabilized their cluster; in recovering from the backlog, some customers received **duplicate webhooks**. A concrete, dated (2025) cost of a delivery-semantics gap surfacing under a real incident — ties directly back to [Lesson 06](06-failure-and-resilience.md)'s cascading-failure framing and to this lesson's billing-pipeline stakes.

## Worked example
**Design: GPU-minute cost pipeline, 10,000 GPUs.** Each GPU emits a usage event per minute → ~10k events/min ≈ 167/s steady, bursty on job churn. Transport is at-least-once, so expect duplicates and out-of-order arrival (a node's buffered events flush late after a network blip).

*Idempotency key.* `(job_id, gpu_uuid, minute_bucket)` — one canonical billable unit. The event carries `{key, gpu_minutes, emit_ts}`. This is structurally the same move as Stripe's `Idempotency-Key`: a caller-determined key that names the exact unit of work being billed, so a redelivery of the same unit is recognizable as such rather than as new work.

*What breaks naively.* A streaming `SUM(gpu_minutes) GROUP BY job_id` over an at-least-once stream double-counts every redelivered event → invoice inflates. Out-of-order arrival makes a windowed SUM close a window before late events land → invoice deflates. Both are silent.

*The fix — keyed upsert (effectively-once).* Land raw events into a table keyed by the idempotency key with `UPSERT` (last-writer-wins on identical key; a redelivery overwrites its own row, contributing exactly once). The monthly rollup is `SUM` over the *deduplicated keyed table*, not over the stream. Late events simply upsert into their bucket and are picked up on the next rollup pass. Result: at-least-once transport + idempotent keyed sink = exactly-once invoice, no 2PC, no distributed transaction. Choose a bucket-close grace period (e.g. bill T+24h — the same order of magnitude as Stripe's 24-hour idempotency-key retention, chosen for the same reason: give the system enough time to see the stragglers before declaring the number final) so late arrivals settle before the invoice is cut — that grace window is the deciding number trading freshness for correctness.

## Practice
Write up the GPU cost/usage pipeline as `gpu-cost-usage-pipeline.md`: the idempotency key, the dedup/upsert model, the SUM-over-keyed-table rollup, the grace window, and an explicit statement of which guarantee (effectively-once) you chose and why not EOS. Feeds the [staff design portfolio](../practice/staff-design-portfolio/README.md).

## Common pitfalls
1. **Treating "exactly-once" as a solved, free feature some systems just "have."** It is always either a distributed transaction (2PC) across the boundary, or an idempotent sink absorbing the duplicates that at-least-once delivery inevitably produces. Kafka's own EOS covers Kafka→Kafka only, not your external DB — the instant you write outside Kafka, you're back to choosing.
2. **Assuming query-based CDC (`WHERE updated_at > x`) is "basically the same" as log-based.** It is structurally blind to deletes and to any intermediate updates that happen between polls, and its staleness bound is only as good as (expensive) poll frequency lets it be.
3. **Assuming "at-least-once + idempotent sink" is a weaker guarantee than Kafka EOS.** For the business outcome that matters (no double-billing, no double-counted metric), it delivers the same effectively-once result — usually at lower cost, since it skips the transaction coordinator entirely.
4. **Assuming windowed streaming aggregates are safe once "the window closes."** Out-of-order or late arrivals after the window closes silently deflate results (this lesson's own worked example hits this exactly) — you need an explicit grace period or a keyed-upsert model that doesn't depend on a window ever truly "closing."
5. **Assuming a Kappa (streaming-only) architecture removes the need for batch.** Even Kappa shops keep a batch reconciliation/backfill path, for logic-bug reprocessing, very-late data, and cold-start rebuilds of a view — it's the correctness backstop, not a legacy leftover.

## Self-check
- Why can at-least-once + an idempotent sink be *cheaper* than Kafka EOS while delivering an exactly-once business result? **Answer:** It moves dedup to a keyed upsert on a business key, avoiding the transaction coordinator, `read_committed` overhead, and the throughput cost of transactional writes — you pay only for the sink's upsert, and duplicates are absorbed rather than prevented.
- What does log-based CDC catch that query-based polling structurally cannot, and why? **Answer:** Deletes (and every intermediate update between polls) — because it reads the WAL/binlog, which records the actual delete and every committed change in commit order, whereas `WHERE updated_at > x` only ever sees currently-present rows at poll time.
- In Flink end-to-end exactly-once, what are the two mechanisms and what sets recovery cost? **Answer:** Chandy–Lamport barrier checkpoints for consistent operator state plus a 2-phase-commit sink (pre-commit on checkpoint, commit on complete); the checkpoint interval sets how much you replay on recovery and adds barrier-alignment latency.
- How does DBLog's watermark mechanism let a CDC tool build a full, consistent table snapshot without ever locking the source table? **Answer:** It interleaves log-tailing with chunked table scans, writing a low-watermark and high-watermark marker into the log around each chunk read; any real change event for a row in that chunk appearing between the two markers is known to be newer than the snapshot value, so the merge keeps the log's version — giving a consistent snapshot assembled from lock-free chunk reads, ordered against the live log rather than a held lock.

## Connections & what's next
This lesson closes the loop that [Lesson 06](06-failure-and-resilience.md) opened: retries and imperfect failure detectors produce duplicates on the wire (the PagerDuty story above shows exactly that, landing as duplicate customer-facing webhooks), and this lesson is where you name the exact mechanism — 2PC or an idempotent sink — that stops those duplicates from corrupting a downstream pipeline. It also sets up [Lesson 08](08-system-design-method.md): data-intensive patterns aren't a standalone topic in a system-design interview, they're a component the design method has to reason about the moment a prompt touches metrics, billing, or any other pipeline — the fleet-telemetry-pipeline prompt in this module's design portfolio is exactly where this lesson and the next one meet.

## References & further reading

**Primary sources**
- Debezium documentation — https://debezium.io/documentation/ — official CDC connector docs; read for the exact WAL/binlog capture mechanics behind log-based CDC.
- Apache Flink docs, *Fault Tolerance via State Snapshots* — https://nightlies.apache.org/flink/flink-docs-stable/docs/learn-flink/fault_tolerance/ — read for the Chandy–Lamport barrier checkpoint and 2PC-sink mechanics referenced above.

**Real-world engineering blogs**
- Netflix TechBlog, *DBLog: A Generic Change-Data-Capture Framework* — https://netflixtechblog.com/dblog-a-generic-change-data-capture-framework-69351fb9099b — the watermark-based, lock-free CDC snapshot mechanism; in production across tens of Netflix microservices.
- Stripe Engineering, *Designing Robust and Predictable APIs with Idempotency* — https://stripe.com/blog/idempotency — the canonical idempotency-key design and lifecycle used as the template for the billing worked example.
- Confluent, *Exactly-once Semantics Are Possible: Here's How Apache Kafka Does It* — https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/ — the idempotent-producer + transactions mechanism behind Kafka EOS.
- Confluent, *Transactions in Apache Kafka* — https://www.confluent.io/blog/transactions-apache-kafka/ — the transaction-coordinator internals for multi-partition atomic writes.
- PagerDuty, *August 28 Kafka Outages — What Happened and How We're Improving* — https://www.pagerduty.com/eng/august-28-kafka-outages-what-happened-and-how-were-improving/ — a real incident where recovery produced duplicate customer-facing webhooks; the concrete cost of a delivery-semantics gap.

**Deeper dives**
- Kleppmann, M., *Designing Data-Intensive Applications* — https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ — the book-length treatment of log-as-truth, CDC, and delivery-semantics tradeoffs this lesson is built on.
