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
sources: 16
---

# A01.7 · Data-intensive patterns

> **Concept.** The log is the source of truth; everything else is a materialized view. Delivery semantics have exact boundaries and exact costs — name them, then choose the cheapest one that holds.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits

[Lesson 06](06-failure-and-resilience.md) established that failure detectors lie, timeouts force retries, and retries — even done right, with full jitter and a token-bucket budget — produce **duplicates on the wire**. That lesson stopped at containment: cap the multiplier, bound the blast radius, bound the rework. It never asked what happens *after* a retried, possibly-duplicated event lands in a downstream data pipeline. This lesson answers exactly that: once duplicates and out-of-order arrivals are a fact of life rather than a bug to eliminate, what precise guarantee can a data pipeline offer, at what cost, and how do you build one — a billing pipeline, a telemetry stream — that survives them without corrupting the numbers it produces.

It also completes a thread that has run since [Lesson 01](01-consistency-models.md). Consistency models told you what a *reader* observes. Delivery semantics tell you what a *writer's* retry does to state. They are the same question — "how many times did this effect happen?" — asked from the two ends of the wire.

## Why this matters

On a GPU platform the two highest-blast-radius data pipelines are fleet telemetry (a gray failure here blinds every autoscaler and every on-call) and GPU-second billing (a double-count here ships wrong invoices to paying customers). Both live or die on delivery semantics you can state precisely and defend by cost.

Put a number on it. A 10,000-GPU fleet emitting one usage event per GPU per minute produces 14.4 million billable records a day. If the transport is at-least-once with a modest 0.5 % duplicate rate and the rollup is a naive `SUM`, the invoice is inflated by 0.5 % — on a $5 M monthly spend that is **$25,000 of over-billing per month**, generated silently, discovered by a customer. The engineering that prevents it is one composite unique key and an upsert. Knowing exactly which mechanism buys which guarantee, and what each costs, is the difference between shipping that and shipping a distributed transaction you did not need.

Staff signal here is refusing the "exactly-once, everywhere, for free" hand-wave and instead naming the exact guarantee, its price, and where it degrades.

## What's new here (calibration)

- **Skip (you already know):** Kafka is a durable partitioned log; events decouple producers from consumers; "make the consumer idempotent"; batch-vs-stream at the tutorial level.
- **Skip:** consumer groups, partition assignment, and offset commits — you run these in production and don't need the mechanics re-taught.
- **New depth — why exactly-once *delivery* is impossible and exactly-once *effect* is not.** The argument is two lines long and it settles every subsequent design question.
- **New depth — Kafka's EOS at the level of the fields on the wire.** Producer ID, epoch, per-partition sequence number, the five batches the broker retains per producer, and what `OutOfOrderSequenceException` actually means when you see it in a log.
- **New depth — the idempotency key as a schema, not a slogan.** Unique constraints, request fingerprints, cached error responses, the concurrent-duplicate case, and the expiry lifecycle.
- **New depth — the outbox and the saga as concrete sequences**, including the exact table schema Debezium's outbox router expects and the compensation ordering a saga executes on failure.
- **New depth — how log-based CDC bootstraps a full snapshot of a live table without locking it**, and its shipped implementation (chunk size, snapshot window, collision buffer).
- **New depth — where a batch/stream pipeline decides correctness and where it decides lateness.** They are two different places, and conflating them is why "the window closed" silently deflates numbers.

Version note: all Kafka, Debezium, and Flink defaults below were read from those projects' source or shipped documentation in August 2026 (`apache/kafka` master, `debezium/debezium` master docs, `apache/flink` master docs). Defaults move between releases; check yours.

## Core concepts

### 1 · The log as the source of truth — and what that actually means mechanically

The framing is Kleppmann's "database inside-out": take a database apart and you find a **write-ahead log** (the durable record of every mutation, in commit order), plus a set of structures *derived* from it — B-tree indexes, materialized views, caches, the replication stream. In a monolithic database those derived structures are private. Turn the database inside out and you make the log the public interface: an immutable, append-only, per-partition totally-ordered stream, with every table, index, cache, and search index a **materialized view** that any consumer can rebuild by replaying from offset 0.

That is not a metaphor. It changes four operational properties:

| Property | Table-as-truth | Log-as-truth |
|---|---|---|
| Adding a new derived view | Backfill script, bespoke, error-prone | Start a consumer at offset 0 |
| "What did this look like at 14:00?" | Requires temporal tables or backups | Replay to the offset at 14:00 |
| Consumer state | Wherever the consumer put it | One integer: the offset |
| Fixing a logic bug in a view | Mutate the view in place and hope | Delete the view, replay, rebuild |

**Mechanically**, an append-only log is a sequence of segment files. Kafka's defaults (from `storage/.../log/LogConfig.java`, master): `segment.bytes = 1 GiB`, `segment.ms = 7 days` — a segment is closed when either bound is hit — and `retention.ms = 7 days` with `cleanup.policy = delete`. Appends are sequential writes to the active segment; reads are sequential scans from an offset, resolved through a sparse index. Sequential-only access is the reason a log sustains throughput a random-access store cannot: the disk (or the page cache) never seeks.

**Log compaction is the mechanism that makes a log usable as a snapshot.** With `cleanup.policy = compact`, a background cleaner rewrites closed segments keeping only the *latest* record per key, forever. A `null` value is a **tombstone**: it deletes the key, and is itself retained for `delete.retention.ms = 24 h` (default) so that a consumer that was offline briefly still observes the deletion before it disappears. The cleaner only bothers with a partition when the "dirty" (uncompacted) fraction exceeds `min.cleanable.dirty.ratio = 0.5`, i.e. it waits until half the log is garbage before doing work.

The consequence worth carrying: **a compacted topic is a changelog that doubles as a full snapshot with bounded storage.** A new consumer reading a compacted topic from offset 0 sees exactly one record per live key — the current state — and then transitions seamlessly into the live tail. That is how Kafka Streams restores state stores, and it is the pattern to reach for whenever you need "give me the whole current state, then keep me updated" without a separate snapshot API.

**What the log does not give you: global order.** A Kafka topic guarantees total order *within a partition*, not across partitions. Order is therefore a function of your partitioning key. If two events must be ordered relative to each other, they must share a key. This is the single most common design error in event pipelines: choosing a key for load balance and then depending on an ordering the key does not provide.

### 2 · Why exactly-once *delivery* is impossible, and what is possible instead

Here is the whole argument, and it takes two paragraphs.

A sender S sends a message to receiver R and waits for an acknowledgement. The ack does not arrive. **S cannot distinguish "the message was lost" from "the message arrived and the ack was lost."** No amount of protocol design fixes this; it is the Two Generals problem, and any fix requires an unbounded number of further messages, each of which has the same problem.

So S has exactly two options, and they are the two delivery semantics:

- **Don't resend** → the message may never arrive → **at-most-once**.
- **Resend** → the message may arrive twice → **at-least-once**.

There is no third branch. **Exactly-once delivery does not exist.** What *does* exist is exactly-once **effect**: the message may be delivered many times, but the observable state change happens once. And there are exactly two mechanisms for achieving it:

1. **An atomic transaction spanning the boundary** — 2PC, or a single system that owns both the offset and the sink so the "commit" is one operation. Cost: a coordinator, blocking on coordinator failure, and availability equal to the *least* available participant.
2. **An idempotent sink** — the effect of applying the same message twice is identical to applying it once. Cost: a key, a uniqueness constraint, and storage for the dedup state.

Everything in the rest of this lesson is one of those two, wearing a name.

| Guarantee | What it means | Mechanism | Typical cost | Use when |
|---|---|---|---|---|
| **At-most-once** | May be lost, never duplicated | Fire and forget; no ack | ~0 | Lossy metrics, debug traces |
| **At-least-once** | Never lost, may be duplicated | Retry until acked | Retry load (lesson 06) | The default transport contract |
| **Effectively-once** | Delivered ≥ 1, applied once | At-least-once + idempotent sink | One unique index; dedup storage | **Most production pipelines, including billing** |
| **Exactly-once (2PC)** | Atomic across the boundary | Coordinator + prepare/commit | Coordinator, blocking, latency | When the sink genuinely cannot be made idempotent |
| **Kafka EOS** | Exactly-once *within Kafka* | Idempotent producer + transactions | Transaction coordinator, `read_committed` latency | Kafka→Kafka read-process-write |

The line to say out loud in an interview: **"exactly-once is not a delivery mode, it is a property of the sink."**

### 3 · Kafka's exactly-once semantics, at the level of the wire

Kafka's EOS is the best-documented production implementation of both mechanisms, so it is worth knowing precisely rather than by slogan. It has two independent layers.

**Layer 1 — the idempotent producer** (deduplication of *producer retries*, per partition):

- On `initProducerId`, the broker assigns the producer a **PID** and an **epoch**.
- Every record batch the producer sends carries `(PID, epoch, base sequence number)`. Sequence numbers are **per `(PID, partition)`** and increase monotonically.
- The broker keeps, per producer per partition, the metadata of the last **5** batches — `ProducerStateEntry.NUM_BATCHES_TO_RETAIN = 5` in `apache/kafka` master. On append it calls `findDuplicateBatch`; if the incoming batch's sequence matches a retained one, the broker **returns the original offset and success without writing anything**. The retry is absorbed silently.
- If the sequence is neither the expected next one nor one of the retained 5, the broker throws **`OutOfOrderSequenceException`**. Seeing this in a log means a batch was lost between two accepted batches (or the producer state expired) — the producer's sequence stream has a hole, and Kafka refuses to guess.
- This is exactly why `max.in.flight.requests.per.connection` must be **≤ 5** when idempotence is on: more in-flight batches than the broker retains metadata for, and a retry of an old batch can no longer be recognised. `ProducerConfig` enforces this and throws `ConfigException` otherwise.

The current defaults, from `clients/.../producer/ProducerConfig.java` (master), matter because they mean **you are already running the idempotent producer whether you asked for it or not**:

| Producer config | Default | Note |
|---|---|---|
| `enable.idempotence` | `true` | On by default since Kafka 3.0 |
| `acks` | `all` | Required by idempotence |
| `retries` | `Integer.MAX_VALUE` | Bounded by `delivery.timeout.ms`, not by count |
| `delivery.timeout.ms` | `120000` (2 min) | The real retry bound |
| `max.in.flight.requests.per.connection` | `5` | Ceiling for idempotence |
| `linger.ms` | `5` | |
| `batch.size` | `16384` | |
| `transaction.timeout.ms` | `60000` (1 min) | Only with `transactional.id` |
| `transactional.id` | `null` | Setting it enables layer 2 |

**Layer 2 — transactions** (atomicity across *multiple partitions*, and fencing across producer restarts):

- Setting `transactional.id` makes the producer's identity durable. On restart, `initProducerId` **bumps the epoch**, and the transaction coordinator aborts any transaction left open by the previous incarnation and **fences** it: writes from the old epoch are rejected. This is the piece that makes a crashed-and-restarted stream processor safe.
- A transaction spans `beginTransaction()` → sends to N partitions → `sendOffsetsToTransaction()` (the consumer's offsets are written into the transaction too, as a record in `__consumer_offsets`) → `commitTransaction()`.
- Commit is two-phase: the coordinator writes a `PREPARE_COMMIT` to its own log (`__transaction_state`), then writes **transaction markers** into every partition the transaction touched, then writes `COMPLETE_COMMIT`.
- Consumers with `isolation.level = read_committed` do not read past the **LSO** (last stable offset) — the offset of the earliest still-open transaction — so uncommitted records are invisible, and aborted records are skipped using the abort index.

**The cost, stated plainly:** `read_committed` consumers cannot read records belonging to an open transaction, so **end-to-end latency becomes a function of the producer's commit cadence**, not of the producer's send cadence. A stream job committing every 100 ms adds ~100 ms of unavoidable latency. That is the price of "exactly-once inside Kafka".

**And the boundary:** all of this covers Kafka→Kafka. The instant your pipeline writes to Postgres, S3, or a billing ledger, the transaction does not extend there, and you are back to the two mechanisms from §2 — 2PC across Kafka and the external store, or an idempotent sink. In practice, choose the sink.

### 4 · Idempotency keys as a schema

"Make the consumer idempotent" is advice, not a design. Here is the design, in the shape Stripe published and in the shape you would actually build it.

**The contract.** The *caller* generates a key that names the unit of work — Stripe recommends a V4 UUID, sent in an `Idempotency-Key` header. The server, on first receipt, executes the work and **persists the response — status code and body, including error responses** — keyed by that key. Any later request with the same key replays the stored response without re-executing the side effect. Keys are pruned after roughly 24 hours; a request arriving after the prune is treated as brand new.

**The schema and the state machine**, which is where the real engineering is:

```sql
CREATE TABLE idempotency_keys (
    key                TEXT        PRIMARY KEY,      -- caller-supplied
    request_fingerprint TEXT       NOT NULL,         -- hash of method+path+body
    state              TEXT        NOT NULL,         -- 'in_progress' | 'completed'
    locked_at          TIMESTAMPTZ,                  -- for stuck-request recovery
    response_status    INT,                          -- persisted, including 4xx/5xx
    response_body      JSONB,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON idempotency_keys (created_at);       -- for the 24h reaper
```

```
  REQUEST WITH Idempotency-Key: 8f2c…
        │
        ▼
  INSERT ... ON CONFLICT DO NOTHING          ← the uniqueness constraint IS the lock
        │
        ├── inserted (we are first) ────────▶ state='in_progress'
        │                                     execute the side effect
        │                                     UPDATE state='completed',
        │                                            response_status/body
        │                                     return the response
        │
        └── conflict (a row already exists)
                 │
                 ├── fingerprint DIFFERS  ──▶ 422 / 409: "key reused with a
                 │                            different request body"
                 │                            (this catches client bugs instead
                 │                             of silently masking them)
                 │
                 ├── state='completed'    ──▶ replay stored status + body verbatim
                 │                            (INCLUDING a stored 4xx — a retry
                 │                             after a definitive failure must not
                 │                             become a second attempt)
                 │
                 └── state='in_progress'  ──▶ 409 Conflict, "request in progress,
                                              retry shortly"
                                              (concurrent duplicate — do NOT block
                                               holding a connection; make the
                                               client come back)
```

Five details that separate a working implementation from a demo:

1. **The uniqueness constraint is the concurrency control.** `INSERT ... ON CONFLICT DO NOTHING` is atomic; a separate `SELECT` then `INSERT` is a race that produces double charges under exactly the retry storm you built this for.
2. **Cache errors too.** If the first attempt returned a definitive `402 card_declined`, the retry must get `402` again, not a fresh attempt against state that may have changed.
3. **Compare the request fingerprint.** Same key, different body means a client bug. Returning a conflict surfaces it; silently returning the old response hides it forever.
4. **`in_progress` needs a timeout.** A process that crashes mid-execution leaves a row that would block the key forever. Sweep rows with `locked_at < now() - interval '90 seconds'` back to a retryable state — and note that this is only safe if the underlying side effect is itself transactional or separately idempotent.
5. **The retention window is a product decision.** Stripe's ~24 h matches the outer bound of a sane client retry campaign. Too short and a legitimately late retry re-executes; too long and the table grows without bound. Size it from your own measured retry-arrival distribution, not from a default.

**Choosing the key.** Two families:

- **Caller-generated opaque key** (a UUID). Correct when the caller alone knows that two requests are the same intent — "this is the same *button press*, not a second one."
- **Derived natural key** (`job_id + gpu_uuid + minute_bucket`). Correct when the unit of work is *intrinsically identifiable* from its content. This is the family billing and telemetry pipelines want, because it makes the key reproducible by any producer, including a replay from the log three weeks later.

The billing example below uses the second, and that choice is what makes the whole pipeline replayable.

### 5 · The dual-write problem, and the outbox pattern as its answer

Every service that both persists state and publishes an event has this bug unless it has explicitly fixed it:

```go
tx.Commit()                    // (1) the order row is durable
producer.Send(orderCreated)    // (2) the event is published
```

These are two systems and there is no atomicity between them. Four interleavings, two of them wrong:

| # | (1) DB commit | (2) publish | Result |
|---|---|---|---|
| 1 | ok | ok | correct |
| 2 | ok | **crash before send** | **DB has the order; no one downstream knows. Silent divergence.** |
| 3 | **fails** | ok | **Event published for state that does not exist. Ghost downstream.** |
| 4 | fails | not attempted | correct (nothing happened) |

Swapping the order does not help; it just swaps which of the two bad cases you get. Retrying the publish does not help either, because the process may not survive to retry.

**The outbox pattern removes the second system from the critical path.** Write the event into the *same database, in the same transaction*, as a row in an `outbox` table. Now there is exactly one commit, so cases 2 and 3 are structurally impossible. A separate relay — log-based CDC — tails the outbox table and publishes.

```
  APPLICATION                       DATABASE (ONE transaction)              KAFKA
  ───────────                       ──────────────────────────              ─────

  BEGIN
   ├─ INSERT INTO gpu_reservations  ┌──────────────────────────┐
   │    (job_id, gpus, state)       │  gpu_reservations        │
   │                                │  ┌────────────────────┐  │
   ├─ INSERT INTO outbox            │  │ job-7  8  RESERVED │  │
   │    (id, aggregatetype,         │  └────────────────────┘  │
   │     aggregateid, type,         │                          │
   │     payload)                   │  outbox                  │
  COMMIT  ◀── atomic ───────────────│  ┌────────────────────┐  │
                                    │  │ id=406c… Reservation│ │
                                    │  │ aggregateid=job-7   │ │
                                    │  │ type=GpusReserved   │ │
                                    │  │ payload={...}       │ │
                                    │  └─────────┬──────────┘  │
                                    └────────────┼─────────────┘
                                                 │
                                    WAL / binlog │  (the commit is already durable;
                                                 │   the relay reads the SAME log the
                                                 ▼   DB uses for crash recovery)
                                        ┌────────────────┐
                                        │ Debezium +     │  outbox event router SMT:
                                        │ outbox router  │  topic  ← aggregatetype
                                        │                │  key    ← aggregateid
                                        └───────┬────────┘  value  ← payload
                                                │           header id ← outbox id
                                                ▼
                                    ┌───────────────────────────┐
                                    │ topic: outbox.event.      │
                                    │        Reservation        │
                                    │ key: "job-7"              │
                                    │ headers: id=406c…         │
                                    └───────────┬───────────────┘
                                                │  at-least-once
                                                ▼
                                    ┌───────────────────────────┐
                                    │ CONSUMER with an          │
                                    │ idempotent sink keyed on  │
                                    │ the header `id`           │
                                    └───────────────────────────┘

  GUARANTEE: exactly-once EFFECT end to end.
  Composed from: one atomic DB commit  +  at-least-once relay  +  idempotent sink.
  NOT from a distributed transaction. There is no coordinator anywhere in this picture.
```

The table shape is not hypothetical — this is the schema Debezium's outbox event router expects by default:

```
Column        |          Type          | Modifiers
--------------+------------------------+-----------
id            | uuid                   | not null   ← becomes the Kafka header `id`;
aggregatetype | character varying(255) | not null      the consumer's dedup key
aggregateid   | character varying(255) | not null   ← becomes the Kafka message key
type          | character varying(255) | not null      (so per-aggregate order holds)
payload       | jsonb                  |            ← becomes the message value
```

Three properties to state when you propose this:

- **The relay is at-least-once, always.** It can crash after publishing and before recording its position, and republish. That is fine and expected — the `id` column is the consumer's idempotency key, and §4 tells you what to do with it.
- **Ordering is per aggregate.** `aggregateid` becomes the Kafka key, so all events for one job land in one partition in commit order. Across aggregates there is no order, and you should not want one.
- **Rows can be deleted immediately after insert.** A common trick: `INSERT ... ; DELETE ...` in the same transaction. The WAL records both, so CDC still sees the insert, and the table stays empty. Costs nothing at read time and avoids a reaper job.

### 6 · Sagas: transactions across services without a coordinator

The outbox fixes "one service, two systems". A saga fixes "one business operation, many services" — where 2PC would be the textbook answer and is usually the wrong one, because a 2PC coordinator makes your availability the product of every participant's and blocks holding locks while a participant is down.

A **saga** (Garcia-Molina & Salem, 1987) decomposes a long-lived transaction into a sequence of local transactions `T₁ … Tₙ`, each with a **compensating transaction** `C₁ … Cₙ` that semantically undoes it. If `Tₖ` fails, the saga executes `Cₖ₋₁, Cₖ₋₂, … , C₁` — in reverse order.

A GPU-platform saga, concretely: *admit a training job*.

```
  FORWARD PATH (each step is a LOCAL transaction, committed independently)

  T1  quota-service      reserve 512 GPU-hours from team budget      ✔ commit
       │  emits QuotaReserved (via outbox)
       ▼
  T2  scheduler          create gang reservation for 64×8 GPUs       ✔ commit
       │  emits GangReserved
       ▼
  T3  storage-service    provision 20 TB scratch volume              ✔ commit
       │  emits ScratchProvisioned
       ▼
  T4  fleet-controller   bind pods to the reserved nodes             ✘ FAILS
                         (a node failed a health probe — lesson 06)

  COMPENSATION PATH (reverse order, each also a local transaction)

  C3  storage-service    delete scratch volume, release capacity     ✔
  C2  scheduler          release the gang reservation                ✔
  C1  quota-service      credit 512 GPU-hours back to the budget     ✔

  End state: the job is REJECTED and every resource is released.
  No lock was held across services at any point. No coordinator exists.
```

The three things that make this hard, and their standard countermeasures:

1. **Compensation is semantic, not syntactic.** `C1` is not "roll back the transaction" — that transaction committed hours ago. It is "credit 512 GPU-hours", a *new* transaction with its own row in the ledger. Some operations have no compensation (you cannot un-send an email); for those, order the saga so the irreversible step is **last**, once every reversible step has succeeded.
2. **There is no isolation.** Between `T1` and `C1`, another job can see the reduced quota and make a decision on it — a dirty read that a real transaction would have prevented. The standard countermeasures: a **semantic lock** (mark the reservation `PENDING` so other readers know it is provisional), **commutative updates** (increment/decrement instead of set, so concurrent changes compose), and **re-read-and-verify** before the irreversible step.
3. **Compensations must be idempotent and must not fail permanently.** `C1` will be retried; crediting the quota twice must not double-credit. Same mechanism as §4: key the compensation on the saga ID.

**Choreography vs orchestration.** In *choreography*, each service listens for the previous step's event and emits its own — no central component, but the control flow exists only implicitly across N services and is genuinely hard to debug. In *orchestration*, one saga orchestrator holds the state machine and issues commands. For anything with more than ~3 steps or any compensation logic, orchestrate: the orchestrator's state table *is* your answer to "where is job-7 stuck?", and it is worth the extra component.

**When not to use a saga:** if all the state lives in one database, use a transaction. Sagas are the price of a service boundary, not a design goal.

### 7 · CDC: log-based vs query-based, and the snapshot that doesn't lock

**Query-based CDC** polls: `SELECT * FROM t WHERE updated_at > :last_seen`. It is easy and it is structurally broken in four ways:

1. **It cannot see deletes.** A deleted row is simply absent from the next poll. Nothing distinguishes "deleted" from "never matched the predicate".
2. **It loses intermediate states.** Two updates between polls collapse into one observation. If a downstream consumer cares about the transition (an audit log, a state machine), it is wrong.
3. **`updated_at` is a lie under concurrency.** A transaction that started before your high-water mark and committed after it writes a row with an `updated_at` *below* the mark — and you will never see it. This is not theoretical; it is the standard failure mode of poll-based sync, and it silently drops rows.
4. **It costs the source.** Every poll is a query against the live table, competing with production traffic. Freshness is bought in query load.

**Log-based CDC** reads the database's own write-ahead log — PostgreSQL's WAL via a logical replication slot, MySQL's binlog. That log is not an add-on; it is the structure the storage engine already maintains for crash recovery and replication. It records **every committed mutation, including deletes, in exact commit order**, and reading it costs the source almost nothing beyond retaining the log segments. All four problems above disappear, and one new operational concern appears: **an unconsumed replication slot pins WAL and will fill the source's disk.** Monitor slot lag as if it were a disk-space alert, because it is one.

**The bootstrap problem.** A new CDC consumer needs the ongoing change stream *and* the full existing state — rows that never change after it starts still need to exist downstream. The naive answer, `SELECT *` under a table lock, is unacceptable on a live database.

Netflix's DBLog framework introduced the fix, and Debezium ships it as **incremental snapshots**. The mechanism, with its shipped parameters:

```
  TIME ────────────────────────────────────────────────────────────────────▶

  transaction log (streamed continuously, never paused):
     …  U(k=3)   ┃          U(k=7)      U(k=3)          ┃   D(k=9)   …
                 ┃                                      ┃
                 ┃                                      ┃
        write LOW WATERMARK                    write HIGH WATERMARK
        (a signal row → appears in the log)     (same)
                 ┃                                      ┃
                 ┗━━━━━━━━━ SNAPSHOT WINDOW ━━━━━━━━━━━━┛
                            (chunk of 1,024 rows, ordered by PK,
                             read with a plain SELECT — NO LOCK)

  Inside the window, the connector BUFFERS the chunk's READ events and
  compares primary keys against the live stream:

      streamed event for k=7  →  no buffered READ for k=7  →  emit it directly
      streamed event for k=3  →  buffered READ for k=3 EXISTS
                                 ⇒ DISCARD the buffered READ (it is stale;
                                    the log event is logically newer)
                                 ⇒ emit the streamed event

  At HIGH WATERMARK the buffer holds only READ events for rows that were
  NOT touched during the window. Emit those. Advance to the next chunk.

  RESULT: a consistent full-table snapshot, assembled from small resumable
  chunks, fully interleaved with live change capture, with no lock ever held.
```

Shipped details worth knowing: the default `incremental.snapshot.chunk.size` is **1,024 rows**; the snapshot is triggered by writing a signal row (or a message to a signalling Kafka topic) rather than by restarting the connector; it is **resumable** — an interrupted snapshot continues from the last completed chunk rather than restarting; and for read-only sources Debezium can use the database's current in-progress transaction ID as the watermark pair instead of writing signal rows (PostgreSQL 13+, MySQL, MariaDB).

The conceptual point to take into a design discussion: this is not "poll and eventually converge". It is **ordering the snapshot against the log** so that convergence is provable, chunk by chunk, without ever taking a lock.

### 8 · Batch and stream: where correctness is decided, and where lateness is

Every serious pipeline eventually needs both a fast path and a correct path. The two architectures for that are Lambda and Kappa, and the useful way to hold them is not "which is better" but **where each one places the two decisions that actually matter**.

```
   LAMBDA — two code paths, reconciled at read time
   ═══════════════════════════════════════════════════════════════════════

                    ┌──────────────────────────────────────┐
                    │      IMMUTABLE APPEND-ONLY LOG       │
                    │      (Kafka, retention = R)          │
                    └───────┬───────────────────┬──────────┘
                            │                   │
              ┌─────────────▼──────┐   ┌────────▼─────────────────┐
              │  SPEED LAYER       │   │  BATCH LAYER             │
              │  streaming job     │   │  scheduled recompute     │
              │  latency: seconds  │   │  latency: hours          │
              │                    │   │                          │
              │  ◆ LATENESS        │   │  ◆ CORRECTNESS decided   │
              │    decided here:   │   │    here: reprocesses the │
              │    watermark +     │   │    FULL window from the  │
              │    allowed         │   │    log, so late data and │
              │    lateness        │   │    logic bugs are fixed  │
              └─────────┬──────────┘   └────────┬─────────────────┘
                        │  approximate           │  authoritative
                        └────────┬───────────────┘
                                 ▼
                         ┌───────────────┐
                         │ SERVING LAYER │  batch result WINS where it exists;
                         │  (merge)      │  speed result fills the recent gap
                         └───────────────┘
                COST: the same business logic written and maintained TWICE,
                      in two engines, with two sets of bugs.


   KAPPA — one code path; "batch" is just a replay
   ═══════════════════════════════════════════════════════════════════════

                    ┌──────────────────────────────────────┐
                    │      IMMUTABLE APPEND-ONLY LOG       │
                    └───────┬──────────────────────┬───────┘
                            │ live tail            │ replay from offset X
                            ▼                      ▼
              ┌──────────────────────┐   ┌──────────────────────┐
              │ streaming job v1     │   │ streaming job v2     │
              │ (serving now)        │   │ (rebuilding a new    │
              │                      │   │  view from history)  │
              │ ◆ LATENESS decided   │   │ ◆ CORRECTNESS decided│
              │   here (watermark +  │   │   here — SAME code,  │
              │   allowed lateness)  │   │   more history, no    │
              │                      │   │   deadline pressure  │
              └──────────┬───────────┘   └──────────┬───────────┘
                         │                          │
                         └────────► cut over ◄──────┘
                                   (atomic view swap)

                COST: log retention R must exceed the longest history you
                      will ever need to replay, and reprocessing throughput
                      must be ≫ live throughput or a rebuild never catches up.
```

**The two decisions, precisely:**

- **Correctness** is decided by *what data the computation is allowed to see*. A batch (or replay) pass over a closed period sees everything that ever arrived for that period, so it is correct by construction. A streaming pass sees only what has arrived so far.
- **Lateness** is decided by the **watermark** and the **allowed lateness**. A watermark is the pipeline's assertion "I believe I have seen all events with event-time ≤ W". It is a heuristic — usually `max(event_time seen) − allowed_out_of_orderness`. When the watermark passes the end of a window, the window fires. Events arriving after that are **late**, and what happens to them is a separate, explicit setting: Flink's `allowedLateness` defaults to **0**, meaning late events are *silently dropped* unless you attach a side output.

That default is the trap. `allowedLateness = 0` plus a 5-minute out-of-orderness tolerance means an event delayed 6 minutes by a network blip is dropped with no error, no metric, and no log line — the aggregate is simply smaller than the truth. **Always route late events to a side output and count them.** A "late events dropped" counter that is normally zero is one of the highest-value metrics in a data pipeline, because it is the only signal that your lateness assumption stopped being true.

**Even in a Kappa shop you keep a batch/replay path**, for three reasons that never go away: a logic bug found three weeks later, data that arrives a month late from a source you don't control, and a brand-new materialized view that must be built from scratch. The reconciliation job that recomputes truth straight from the log is the correctness backstop that makes the streaming path's guarantees trustworthy in the first place.

### 9 · Flink's end-to-end exactly-once, and what it costs

Flink is the clearest production example of mechanism 1 from §2 (transaction across the boundary), so it is worth knowing what it actually does.

**Barriers.** The source injects numbered **stream barriers** into the data stream. Barriers flow in line with records and never overtake them, so a barrier partitions the stream into "before checkpoint *n*" and "after checkpoint *n*". Multiple barriers for different checkpoints can be in flight at once.

**Alignment.** An operator with multiple inputs waits until it has seen barrier *n* on *every* input before snapshotting its state; records arriving on an already-aligned input are buffered. This is what makes the snapshot a consistent cut across the whole dataflow (Chandy–Lamport). The cost is the alignment stall — the Flink docs state it is usually a few milliseconds but note that outliers can be noticeably worse, which is why Flink offers a switch to **skip alignment** and fall back to at-least-once for latency-critical jobs.

**The 2PC sink.** For end-to-end exactly-once the sink participates in the checkpoint as a transaction:

```
  checkpoint n barrier arrives at the sink
        │
        ├─▶ PRE-COMMIT: flush everything written since checkpoint n−1 into an
        │   external transaction, and make that transaction's handle part of
        │   the checkpointed state (so a recovery can find it again)
        │
        ▼
  JobManager reports "checkpoint n complete" to every operator
        │
        └─▶ COMMIT: the sink commits the external transaction

  FAILURE BETWEEN THE TWO:  on recovery the sink restores the transaction
  handle from state and re-attempts the commit. The external system must
  therefore support (a) transactions that survive a client restart, and
  (b) an idempotent commit of an already-committed transaction.
  Kafka provides both via transactional.id + epoch fencing (§3).
```

**The cost, in one sentence:** data is only visible downstream *after a checkpoint completes*, so **your end-to-end latency floor is the checkpoint interval**, and your recovery replay is bounded by it too. A 30-second checkpoint interval means 30 seconds of latency and up to 30 seconds of reprocessing on recovery. That single number is the tradeoff dial, and being able to say it is the difference between citing "Flink does exactly-once" and understanding it.

### 10 · Choosing, for a GPU platform

Two pipelines, two different correct answers — which is exactly why the guarantee is a design decision and not a default.

**Fleet telemetry (DCGM/GPU metrics).** High cardinality (per-GPU × per-node × per-job × per-metric) and high volume. Correctness tolerance is loose: a dashboard 0.1 % off for five minutes is invisible. Ship at-least-once, aggregate idempotently where it is cheap, **downsample early** (at the node, not at the store), and apply backpressure so the observability pipeline never becomes the fleet-wide gray failure that hides the outage it should surface (lesson 06, §10 — an observer that shares a failure domain with the system is not an observer). Guarantee: **at-least-once, best-effort dedup**.

**GPU-second billing.** Correctness tolerance is zero: an invoice 0.1 % inflated is a support ticket, a refund, and at scale a trust problem. Guarantee: **effectively-once**, via a derived natural key and an idempotent keyed sink. Explicitly *not* Kafka EOS — the sink is an external ledger, so EOS would not cover the boundary anyway, and the coordinator would buy nothing the unique index does not already provide. Worked end to end below.

## Perspectives

**The storage-engine view.** Log-based CDC works because the database already keeps the thing you want: the WAL is written for crash recovery and physical replication, records every committed mutation in commit order including deletes, and imposes no query load on the live table. Query-based polling has no equivalent structure to read — it can only re-derive *current state*, never *the history of how it got there*. That asymmetry, not vendor preference, is why log-based CDC is the staff default.

**The delivery-semantics view.** "Exactly-once" names a boundary problem, not a broker feature. Every system that claims it is doing one of two things: coordinating a transaction across the boundary (expensive, availability equal to the weakest participant), or making duplicate delivery a no-op at the sink (cheap, and the far more common production choice). Kafka's own EOS is scoped to Kafka-to-Kafka; the instant your pipeline writes elsewhere you are choosing again.

**The financial-correctness view.** The asymmetry between telemetry and billing is not a matter of care, it is a matter of the cost function. Metrics have a symmetric, forgiving error cost; money has an asymmetric one where over-billing is a trust event and under-billing is revenue you never recover. That asymmetry is what justifies the extra machinery — a key tied to the exact billable unit, a persisted keyed row, and a deliberate grace window before the number is final. "Close enough" is a valid engineering answer for one and unacceptable for the other, and saying which is which out loud is the design decision.

**The operability/backfill view.** Streaming guarantees only cover the failure modes you anticipated when you wrote the job. A logic bug discovered three weeks later, data that arrives a month late, a new view that must be built from scratch — none of those are failures the pipeline can recover from; they are recomputations. The batch reconciliation path is what makes the streaming path's guarantees trustworthy, and it is the reason log retention is a correctness parameter rather than a storage-cost parameter.

**The interview view.** This topic is where candidates most often assert instead of explain. The three sentences that signal depth: "exactly-once delivery is impossible — the sender can't distinguish a lost message from a lost ack"; "so you either coordinate a transaction across the boundary or make the sink idempotent, and for this system the sink is cheaper because *X*"; "and here is the key: `(job_id, gpu_uuid, minute_bucket)`, with a unique index and an upsert." Then draw the grace window and say what number closes it.

## Real-world use cases

- **Netflix, "DBLog: A Generic Change-Data-Capture Framework"** — <https://netflixtechblog.com/dblog-a-generic-change-data-capture-framework-69351fb9099b> — the origin of the watermark-based lock-free snapshot in §7, in production across tens of Netflix microservices since around 2018. *What it shows:* that "consistent full-table snapshot" and "never lock the source" are compatible, if you order the snapshot against the log instead of against a lock.
- **Debezium incremental snapshots** — <https://debezium.io/documentation/> — the shipped implementation of that mechanism, with the parameters quoted in §7 (default chunk size 1,024 rows; signal-triggered; resumable; read-only variant using in-progress transaction IDs as watermarks). *What it shows:* the algorithm is not research — it is a config flag on a connector you can run today.
- **Stripe, "Designing Robust and Predictable APIs with Idempotency"** — <https://stripe.com/blog/idempotency> — the canonical idempotency-key contract: client-generated V4 UUID in an `Idempotency-Key` header, server persists the first response *including errors*, ~24-hour retention, key reuse with a different body treated as a conflict. *What it shows:* the production shape the schema in §4 implements, and the reason each of its unusual details exists.
- **PagerDuty, "August 28 Kafka Outages — What Happened and How We're Improving"** — <https://www.pagerduty.com/eng/august-28-kafka-outages-what-happened-and-how-were-improving/> — a runaway producer bug destabilised the broker fleet, and **recovering from the backlog produced duplicate webhooks** for some customers. *What it shows:* the concrete, customer-visible cost of a delivery-semantics gap, surfacing exactly where lesson 06 predicts it will — on the recovery path, not during the failure.
- **Apache Kafka's own source** — <https://github.com/apache/kafka> — `ProducerStateEntry.NUM_BATCHES_TO_RETAIN = 5`, `findDuplicateBatch`, `OutOfOrderSequenceException`, and the `ProducerConfig` defaults table in §3. *What it shows:* that "the idempotent producer dedups retries" is a five-element ring buffer of batch metadata per producer per partition — small, cheap, and with a precisely-defined failure mode when its assumptions are violated.

## Worked example

**Design: a GPU-minute cost pipeline for a 10,000-GPU fleet, end to end, with the numbers that decide each choice.**

### Step 1 — Volume, and what it rules in or out

```
 GPUs                                   N       = 10,000
 Emission rate                                  = 1 usage event / GPU / minute
 Event rate                             λ       = 10,000 / 60 s = 166.7 events/s
 Daily records                                  = 10,000 × 1,440 = 14.4 M/day
 Monthly records                                = ~432 M/month

 Event payload (JSON, measured shape):
   job_id 36 B + gpu_uuid 40 B + node 24 B + minute_bucket 20 B
   + gpu_seconds 8 B + sku 12 B + emit_ts 20 B + JSON overhead ~90 B
                                        ≈ 250 B, call it 300 B on the wire

 Ingest bandwidth   = 166.7 × 300 B     ≈ 50 KB/s   ≈ 4.3 GB/day
 Kafka storage      = 4.3 GB/day × 7 d retention × RF 3
                                        ≈ 91 GB
```

**What that rules out:** at 50 KB/s and 167 events/s this is a *tiny* stream. Any argument in this design that appeals to throughput is wrong. Do not propose Flink with RocksDB state, do not propose a Lambda architecture, do not shard anything for load. **The hard problems here are correctness and lateness, not scale** — and saying so explicitly is the first staff move in the design, because it kills three plausible-sounding but unjustified architectures in one sentence.

### Step 2 — The arithmetic that decides the partition count

Kafka partition count is decided by three independent constraints; take the max.

```
 (a) Throughput.    A partition comfortably sustains ≫ 10 MB/s.
                    Need 50 KB/s.                          →  1 partition

 (b) Consumer parallelism. One consumer instance handles ~5,000 upserts/s
     against Postgres with batching. Need 167/s, plus 10x replay headroom
     for a backfill (1,670/s).                             →  1 partition
     Operationally we want ≥ 3 consumers for rolling deploys and failure
     tolerance.                                            →  3 partitions

 (c) Key skew.      If we key by job_id: the largest training job is
     8,192 GPUs = 82 % of the fleet's events on ONE partition.
     With P partitions, expected share = 1/P, actual share = 0.82.
     Skew factor = 0.82 · P. At P=12 that is a 9.8x hot partition.
                                                           →  job_id is a BAD key

     If we key by (job_id, gpu_uuid): 10,000 distinct keys, hashed.
     Expected per-partition share = 1/P ± O(1/√(10,000/P)).
     At P=12: 833 keys/partition, ±3.5 % — flat.           →  good key
```

**Decision: 12 partitions, keyed by `hash(job_id || gpu_uuid)`.** Twelve rather than three, because partition count can only be increased (and increasing it rehashes keys, breaking per-key ordering), so buy headroom now; and because 12 divides evenly by 1, 2, 3, 4, 6, 12 consumers.

**The tradeoff you must state:** keying by `(job_id, gpu_uuid)` gives up per-job event ordering. That is *deliberately* fine here, because the sink deduplicates by key rather than by position, and the rollup is a commutative `SUM`. If the sink had been a state machine — "job started, job ended" — this key would be wrong and `job_id` would be right despite the skew, with the hot job handled by a dedicated partition or a separate topic. **Ordering requirements choose the key; load balance only chooses among the keys that satisfy them.**

### Step 3 — The idempotency key, and what breaks without it

**Key: `(job_id, gpu_uuid, minute_bucket)`.** This is a *derived natural key* (§4): it names the exact billable unit, and any producer — the live agent, a replay from the log, a batch reconciliation three weeks later — computes the identical key from the same event. That reproducibility is what makes the whole pipeline replayable.

What goes wrong with a naive `SUM(gpu_seconds) GROUP BY job_id` over the raw stream:

```
 Duplicate rate on an at-least-once transport with lesson 06's retry budget:
   measured redelivery ≈ 0.5 % of events

 Over-count       = 0.5 % of 432 M monthly records ≈ 2.16 M duplicated records
 Invoice inflation = +0.5 %
 On a $5 M monthly GPU spend                       = +$25,000/month over-billed

 And the opposite error, from lateness:
   a windowed SUM that fires on watermark with allowedLateness = 0 drops
   every event delayed past the watermark. Measured p99.9 arrival delay on
   this fleet: 6 h (a node buffers locally through a network partition).
   Events dropped = the tail beyond the window ⇒ invoice DEFLATED.

 Both errors are silent. Neither raises an exception. One loses money,
 the other loses customers.
```

### Step 4 — The sink: keyed upsert, sized

```sql
CREATE TABLE gpu_usage_minutes (
    job_id        TEXT        NOT NULL,
    gpu_uuid      TEXT        NOT NULL,
    minute_bucket TIMESTAMPTZ NOT NULL,
    gpu_seconds   NUMERIC(8,3) NOT NULL,
    sku           TEXT        NOT NULL,
    first_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (job_id, gpu_uuid, minute_bucket)     -- ← the dedup mechanism
) PARTITION BY RANGE (minute_bucket);                 -- daily partitions

-- the consumer's only write:
INSERT INTO gpu_usage_minutes (job_id, gpu_uuid, minute_bucket, gpu_seconds, sku)
VALUES (:job_id, :gpu_uuid, :minute_bucket, :gpu_seconds, :sku)
ON CONFLICT (job_id, gpu_uuid, minute_bucket) DO UPDATE
    SET gpu_seconds = EXCLUDED.gpu_seconds,
        sku         = EXCLUDED.sku;
```

A redelivery of an already-applied record overwrites its own row with identical values and contributes exactly once. **At-least-once transport + idempotent keyed sink = effectively-once invoice, with no 2PC and no transaction coordinator anywhere in the design.**

Sizing the dedup state — this is the cost of the guarantee, and you should be able to state it:

```
 Row width: job_id 36 + gpu_uuid 40 + timestamptz 8 + numeric 9 + sku 12
            + Postgres row header 24 + alignment            ≈ 130 B
 Primary-key index entry (84 B key + 16 B overhead)         ≈ 100 B
 Total per record                                           ≈ 230 B

 Per day    : 14.4 M × 230 B                                ≈ 3.3 GB
 Per month  : 432 M × 230 B                                 ≈ 99 GB
 Retain 35 days of minute-grain detail (invoice + dispute window)
            : 14.4 M × 35 × 230 B                           ≈ 116 GB

 Beyond 35 days, roll up to (job_id, day, sku) and drop the minute rows:
            10,000 GPUs → ~500 jobs/day × 1 sku ≈ 500 rows/day ≈ 0.1 MB/day
            → 5 years of billing history in < 200 MB.
```

**116 GB on one Postgres instance with daily partitions.** No sharding, no distributed database. The estimate chose the architecture — and the fact that it chose the *boring* one is the answer, not a failure of ambition.

### Step 5 — The grace window, derived rather than guessed

The rollup must eventually declare a month final. When?

```
 Measure the arrival-delay distribution (emit_ts → sink write) over 90 days:

   p50    2.1 s
   p99    47 s
   p99.9  6.2 h        ← a node buffering through a network partition
   max    21.7 h       ← a node down for maintenance, replaying its spool

 Choose the close: max observed + margin, rounded up
                 = 21.7 h + 2.3 h  →  T + 24 h

 Residual risk at T+24h: events arriving later than any observed sample.
 Handle them, do not pretend they cannot happen:
   → they still upsert into gpu_usage_minutes (the table has no window)
   → a nightly reconciliation job re-sums each CLOSED day and writes a
     `billing_adjustments` row for any delta
   → adjustments below $1.00 are absorbed; above that they appear on the
     next invoice as a line item
```

This is the whole reason the sink is a keyed table and not a windowed streaming aggregate: **a table has no window to close, so a late event is just an upsert.** The grace window is a *business* decision about when to bill, not a *technical* deadline after which data is lost. That inversion — moving the deadline out of the pipeline and into the invoice — is the design move worth defending.

### Step 6 — Why not Kafka EOS, stated for the interviewer

Three reasons, in order of force:

1. **It does not cover the boundary.** Kafka's transactions are atomic across Kafka partitions and consumer offsets. The billing ledger is Postgres. EOS stops at the broker; the write to Postgres is outside it. To get atomicity you would need XA/2PC between Kafka and Postgres — a coordinator, blocking, and availability equal to the *less* available of the two.
2. **The unique index already does the job.** The upsert makes duplicates a no-op. EOS would prevent duplicates that the sink already absorbs — paying a coordinator to avoid work that costs one index lookup.
3. **It adds latency for no benefit here.** `read_committed` consumers stall at the LSO until a transaction commits, tying end-to-end latency to the commit cadence. Billing has a 24-hour grace window; buying milliseconds is meaningless.

**Where the answer would flip:** if the sink genuinely could not be made idempotent — an append-only external ledger with no natural key, or a third-party API with no idempotency-key support — then the coordinator earns its cost, and you would use a 2PC sink (Flink-style, §9) or an outbox on the sink side. Naming that flip condition unprompted is the staff move; asserting "we chose effectively-once" without it is not.

## Practice

Write up the GPU cost/usage pipeline as `gpu-cost-usage-pipeline.md`, following the worked example's structure but with your own stated assumptions. It must contain:

1. **A volume estimate** — event rate, payload size, ingest bandwidth, log retention storage with replication factor — and an explicit sentence saying which architectures the numbers *rule out*.
2. **The partition-count arithmetic**, computed from all three constraints (throughput, consumer parallelism, key skew), with the chosen key and a statement of what ordering guarantee that key gives up.
3. **The idempotency key**, named as caller-generated or derived-natural, with the reason.
4. **The sink schema** as real DDL, including the uniqueness constraint that performs the dedup and the upsert statement, plus a sizing calculation for the dedup state over your retention window.
5. **The grace window**, derived from a stated (or assumed, and labelled) arrival-delay distribution, plus what happens to an event arriving after it — a reconciliation path, not "it is dropped".
6. **An explicit guarantees table** — for each stage: the guarantee it provides, the mechanism, and the failure mode when the mechanism's assumption is violated.
7. **A "why not EOS / why not 2PC" section** with the three reasons and the condition under which the answer flips.

*Acceptance:* every architectural choice traces to a number in section 1 or 2; the word "exactly-once" appears only alongside the boundary it applies to; and there is no component in the design whose cost you cannot state. Feeds the [staff design portfolio](../practice/staff-design-portfolio/README.md).

## Common pitfalls

1. **Treating "exactly-once" as a feature some systems just have.** *Symptom:* a design doc that says "we use Kafka EOS so duplicates are handled" while writing to an external database. *Mechanism:* delivery is at-most-once or at-least-once and nothing else — the sender cannot distinguish a lost message from a lost ack. Exactly-once *effect* comes from a transaction spanning the boundary or an idempotent sink, and Kafka's transactions span Kafka only. Naming the boundary is the fix.
2. **Assuming a partition key that balances load also provides ordering.** *Symptom:* a state machine downstream that occasionally sees "ended" before "started". *Mechanism:* Kafka orders within a partition only; two events are ordered relative to each other **only if they share a key**. Ordering requirements choose the key; load balance chooses among the keys that satisfy them, never the other way round.
3. **Assuming query-based CDC is "basically the same" as log-based.** *Symptom:* a replica that slowly diverges, missing deleted rows and occasional updates. *Mechanism:* `WHERE updated_at > :x` cannot see deletes (an absent row is indistinguishable from a never-matching one), collapses intermediate states, and misses rows written by transactions that began before the high-water mark and committed after it. The WAL has none of those blind spots because it records committed mutations in commit order.
4. **Forgetting that a replication slot is a disk-space liability.** *Symptom:* the source database fills its disk and stops accepting writes; the outage is on the *source*, not the pipeline. *Mechanism:* an unconsumed logical replication slot pins WAL segments so they cannot be recycled. Alert on slot lag bytes with the same severity as free disk.
5. **Assuming a windowed streaming aggregate is safe once the window closes.** *Symptom:* monthly totals that are quietly a fraction of a percent low, discovered by a customer. *Mechanism:* the watermark is a heuristic, and Flink's `allowedLateness` defaults to **0** — late events are dropped with no exception and no metric. Route late events to a side output, count them, and prefer a keyed table (which has no window to close) over a window whenever the number is money.
6. **Doing a dual write and calling it eventual consistency.** *Symptom:* rows in the database with no corresponding downstream event, discovered months later during a reconciliation. *Mechanism:* `commit()` then `send()` are two systems with no atomicity; a crash between them loses the event permanently and no retry loop can fix it, because the process that would retry is gone. The outbox makes it one commit; the relay's at-least-once redelivery is then absorbed by the consumer's idempotency key.
7. **Reaching for a saga when a transaction would do.** *Symptom:* a five-service choreography, with compensations, for state that all lives in one database. *Mechanism:* sagas exist to avoid holding locks across service boundaries and they cost you isolation — dirty reads between `Tₖ` and `Cₖ` are inherent, and every compensation is new code with its own idempotency requirements. If there is no service boundary, there is no reason to pay that.
8. **Assuming "at-least-once + idempotent sink" is weaker than EOS.** *Symptom:* a proposal to add a transaction coordinator to a pipeline whose sink already has a unique index. *Mechanism:* for the outcome that matters — no double-billing, no double-counted metric — the two deliver the same result, and the sink-side version skips the coordinator, the `read_committed` latency floor, and an entire failure domain. It is usually the *stronger* engineering choice, not the compromise.

## Self-check

- **Why is exactly-once delivery impossible, and what replaces it?** The sender cannot distinguish "my message was lost" from "my message arrived and the ack was lost" — the Two Generals problem, and every proposed fix needs another message with the same problem. So the sender either does not resend (at-most-once) or resends (at-least-once); there is no third branch. What replaces it is exactly-once *effect*, achievable in exactly two ways: a transaction spanning the boundary (2PC, or a single system that owns both offset and sink), or an idempotent sink where applying a message twice is indistinguishable from applying it once. Naming which of the two you are using — and where the boundary is — is the whole answer.
- **Describe Kafka's idempotent producer at the level of the wire, and say what `OutOfOrderSequenceException` means.** The broker assigns a producer a PID and an epoch; every batch carries `(PID, epoch, base sequence)` with sequences monotonic per `(PID, partition)`. The broker retains metadata for the last **5** batches per producer per partition (`NUM_BATCHES_TO_RETAIN = 5`) and, on a batch whose sequence matches a retained one, returns the original offset and success **without writing** — the retry is absorbed. `OutOfOrderSequenceException` means the incoming sequence is neither the expected next one nor one of the retained five: a batch is missing from the sequence stream, so the broker refuses to append rather than silently create a gap. This is also why `max.in.flight.requests.per.connection` must be ≤ 5 with idempotence on — more in-flight batches than the broker remembers and a retry becomes unrecognisable.
- **What breaks if you write to your database and then publish to Kafka, and what fixes it?** Two systems, no atomicity: crash after commit and before send loses the event permanently (the DB has state nobody downstream knows about); publish succeeding while the commit fails creates a ghost event for state that does not exist. Reordering swaps which bad case you get; retrying does not help if the process dies. The **outbox pattern** fixes it by writing the event into the same database in the same transaction — one commit, so both bad interleavings are structurally impossible — and having log-based CDC tail the outbox table. The relay is at-least-once, and the outbox row's `id` becomes the consumer's idempotency key. Debezium's outbox router expects `(id, aggregatetype, aggregateid, type, payload)`, mapping `aggregatetype`→topic, `aggregateid`→message key, `payload`→value, `id`→header.
- **Sketch a saga and name the two things it gives up relative to a distributed transaction.** A sequence of local transactions `T₁…Tₙ`, each with a compensation `Cᵢ`; if `Tₖ` fails, run `Cₖ₋₁ … C₁` in reverse. Example: reserve quota → reserve gang → provision scratch → bind pods; if binding fails, delete scratch, release the gang, credit the quota. It gives up (1) **isolation** — between `T₁` and `C₁` other actors can read the provisional state, so you need semantic locks, commutative updates, or re-read-and-verify — and (2) **syntactic rollback**: compensations are new business transactions with their own idempotency requirements, and some operations have no compensation at all, so the irreversible step must be ordered last. In exchange you never hold a lock across a service boundary and your availability is not the product of every participant's.
- **How does log-based CDC build a full consistent snapshot without locking the source?** Interleave chunked `SELECT`s with continuous log tailing, and order the two against each other using watermarks. Write a low-watermark marker into the log, read a chunk (Debezium default 1,024 rows, ordered by primary key) into a memory buffer, write a high-watermark marker. During that window, compare streamed change events against buffered `READ` events by primary key: if a streamed event matches a buffered row, discard the buffered `READ` (it is stale — the log event is logically newer) and emit the streamed event. At the high watermark the buffer holds only rows nothing touched during the window; emit those. Repeat per chunk. The result is a provably consistent snapshot, resumable, assembled without ever holding a lock.
- **In a pipeline, where is correctness decided and where is lateness decided?** Correctness is decided by *what data the computation is allowed to see*: a batch pass or a replay over a closed period sees everything that ever arrived, so it is correct by construction; a streaming pass sees only what has arrived so far. Lateness is decided by two separate knobs: the **watermark** (the heuristic assertion "I have seen everything with event-time ≤ W", typically `max seen − allowed out-of-orderness`), which decides when a window fires, and the **allowed lateness**, which decides what happens to events arriving after that. Flink's `allowedLateness` defaults to 0, so late events are silently dropped — always attach a side output and a counter. Conflating the two is why aggregates deflate without any error surfacing.
- **Design the billing key for a 10,000-GPU fleet and justify every part of it.** Key: `(job_id, gpu_uuid, minute_bucket)`, as the primary key of a partitioned table, written with `INSERT … ON CONFLICT DO UPDATE`. It is a *derived natural* key so any producer — live agent, log replay, or a reconciliation job weeks later — computes the identical key from the same event, which is what makes the pipeline replayable. `minute_bucket` is the billable granularity, so a redelivered record overwrites its own row and contributes once. The uniqueness constraint *is* the dedup mechanism, so no coordinator exists in the design. Sizing: ~230 B/record including the index, 14.4 M records/day → ~3.3 GB/day, ~116 GB for a 35-day minute-grain window on one Postgres instance, rolled up to `(job_id, day, sku)` beyond that. And because the sink is a table rather than a window, a late event is just an upsert — the 24-hour grace period is a decision about when to *invoice*, not a deadline after which data is lost.
- **When does the answer flip from "idempotent sink" to "2PC"?** When the sink genuinely cannot be made idempotent: an append-only external ledger with no natural key to constrain on, or a third-party API that offers no idempotency-key mechanism and whose side effect is irreversible. Then a coordinator earns its cost — a Flink-style 2PC sink that pre-commits on the checkpoint barrier and commits on checkpoint-complete, with the transaction handle stored in checkpointed state so a recovery can re-attempt the commit. The price you accept is that end-to-end latency becomes the checkpoint interval, recovery replays up to one interval, and the external system must support transactions that survive a client restart plus an idempotent commit of an already-committed transaction.

## Connections & what's next

This lesson closes the loop [Lesson 06](06-failure-and-resilience.md) opened: imperfect failure detectors force retries, retries produce duplicates on the wire (PagerDuty's duplicate webhooks are that exact sequence landing on customers), and this lesson names the two mechanisms — a transaction across the boundary, or an idempotent sink — that stop those duplicates from corrupting a downstream number. It also picks up [Lesson 01](01-consistency-models.md)'s consistency vocabulary from the writer's side, [Lesson 03](03-replication-and-partitioning.md)'s partitioning (the key-choice arithmetic in the worked example is the same reasoning applied to a log), and [Lesson 05](05-queueing-and-backpressure.md)'s backpressure, which is what stops a telemetry pipeline from becoming the fleet-wide gray failure it was built to prevent. Forward: [Lesson 08](08-system-design-method.md) makes this a *component* of the design method rather than a standalone topic — the moment a prompt touches metrics, billing, audit, or any other pipeline, steps 4 (data model) and 7 (failure & operations) have to produce exactly the guarantees table and the grace window this lesson just built. The fleet-telemetry-pipeline prompt in the design portfolio is where the two meet.

## References & further reading

**Implementations — read from source or shipped docs, and the basis for every default quoted above**

1. **Apache Kafka** — <https://github.com/apache/kafka> — `clients/src/main/java/org/apache/kafka/clients/producer/ProducerConfig.java` (the §3 defaults table: `enable.idempotence=true`, `acks=all`, `retries=Integer.MAX_VALUE`, `delivery.timeout.ms=120000`, `max.in.flight.requests.per.connection=5` with the ≤ 5 validation for idempotence, `transaction.timeout.ms=60000`, `linger.ms=5`, `batch.size=16384`); `storage/.../log/ProducerStateEntry.java` (`NUM_BATCHES_TO_RETAIN = 5`, `findDuplicateBatch`) and `ProducerAppendInfo.java` (`OutOfOrderSequenceException`); `storage/.../log/LogConfig.java` (`DEFAULT_SEGMENT_BYTES = 1 GiB`, `DEFAULT_SEGMENT_MS`/`DEFAULT_RETENTION_MS = 7 days`, `DEFAULT_DELETE_RETENTION_MS = 24 h`, `DEFAULT_MIN_CLEANABLE_DIRTY_RATIO = 0.5`). *Cloned and read in this environment, August 2026.*
2. **Debezium documentation** — <https://debezium.io/documentation/> — `documentation/modules/ROOT/partials/.../con-connector-incremental-snapshot.adoc` (the §7 watermark/snapshot-window mechanism, default chunk size 1,024 rows, buffering and collision resolution, resumability) and `documentation/modules/ROOT/pages/transformations/outbox-event-router.adoc` (the §5 outbox table schema and the field-to-Kafka mapping). *Cloned from the `debezium/debezium` repository and read in this environment; the rendered docs site itself is egress-restricted here.*
3. **Apache Flink documentation** — <https://nightlies.apache.org/flink/flink-docs-stable/docs/learn-flink/fault_tolerance/> — `docs/content/docs/concepts/stateful-stream-processing.md` for the §9 barrier and alignment semantics, including the exactly-once-vs-at-least-once alignment switch and its stated latency characteristics. *Read from the `apache/flink` repository in this environment; the published docs site is egress-restricted here.*

**Primary sources**

4. **Garcia-Molina, H., Salem, K. (1987), "Sagas,"** SIGMOD '87 — the original decomposition into local transactions with compensations, and the source of the term. *Not fetched from this environment; cited for the model, which §6 develops from first principles.*
5. **Akidau, T. et al. (2015), "The Dataflow Model,"** VLDB — the what/where/when/how decomposition and the watermark/allowed-lateness separation used in §8. *Not fetched from this environment; the mechanics described here are cross-checked against Flink's shipped documentation instead.*
6. **Chandy, K.M., Lamport, L. (1985), "Distributed Snapshots: Determining Global States of Distributed Systems"** — the asynchronous-snapshot algorithm Flink's barrier checkpointing implements. *Not fetched from this environment; cited for the algorithm.*
7. **Kleppmann, M., "Turning the Database Inside-Out"** — <https://martin.kleppmann.com/2015/11/05/database-inside-out-at-strange-loop.html> — the §1 framing: log as truth, everything else a materialized view. *Not fetched from this environment; the framing is developed independently in §1 and does not rely on unverified specifics.*
8. **Kreps, J., "The Log: What every software engineer should know about real-time data's unifying abstraction"** — <https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying> — the log-as-integration-substrate argument. *Not fetched from this environment.*
9. **Kreps, J., "Questioning the Lambda Architecture"** — <https://www.oreilly.com/radar/questioning-the-lambda-architecture/> — the origin of the Kappa argument in §8 (the maintenance cost of two implementations of the same logic). *Not fetched from this environment.*

**Real-world engineering write-ups**

10. **Netflix TechBlog, "DBLog: A Generic Change-Data-Capture Framework"** — <https://netflixtechblog.com/dblog-a-generic-change-data-capture-framework-69351fb9099b> — the watermark-based lock-free snapshot; in production across tens of Netflix microservices. *Not fetched from this environment; the mechanism described in §7 is taken from Debezium's shipped implementation of it, which was read directly.*
11. **Stripe Engineering, "Designing Robust and Predictable APIs with Idempotency"** — <https://stripe.com/blog/idempotency> — the idempotency-key contract in §4. *Not fetched from this environment; the contract (client-generated key, persisted responses including errors, ~24 h retention, conflict on key reuse with a different body) is carried forward from the previous revision of this lesson, and the schema and state machine in §4 are this lesson's own construction.*
12. **Confluent, "Exactly-once Semantics Are Possible: Here's How Apache Kafka Does It"** — <https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/> — the narrative introduction to the idempotent-producer and transaction mechanics. *Not fetched from this environment; §3's mechanics were read from the Kafka source instead.*
13. **Confluent, "Transactions in Apache Kafka"** — <https://www.confluent.io/blog/transactions-apache-kafka/> — the transaction-coordinator internals (`__transaction_state`, markers, LSO). *Not fetched from this environment.*
14. **PagerDuty, "August 28 Kafka Outages — What Happened and How We're Improving"** — <https://www.pagerduty.com/eng/august-28-kafka-outages-what-happened-and-how-were-improving/> — a real incident whose *recovery* produced duplicate customer-facing webhooks. *Not fetched from this environment; summary carried forward from the previous revision of this lesson and from lesson 06.*

**Deeper dives**

15. **Kleppmann, M., *Designing Data-Intensive Applications*** — <https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/> — Chapters 5 (replication), 7 (transactions), 11 (stream processing) and 12 (the future of data systems) are the book-length treatment of everything in this lesson, and Chapter 11's "exactly-once execution" discussion is the closest published parallel to §2.
16. **`harut8/system-design`, `databases/` track** — see [`docs/EXTERNAL-DEPTH.md`](../../../../docs/EXTERNAL-DEPTH.md) — chapters on WAL internals and replication are the storage-engine layer beneath §7's log-based CDC.
