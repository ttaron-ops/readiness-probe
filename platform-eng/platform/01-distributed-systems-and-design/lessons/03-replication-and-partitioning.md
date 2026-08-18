---
lesson: "A01.3"
title: "Replication and partitioning"
module: "A-01"
concept: "replication spectrum, partition schemes, hot shards"
status: not-started
est_time: "5 hrs"
prev: "02-consensus-and-quorums.md"
next: "04-caching.md"
artifacts: ["metadata-store shard design + rebalance plan"]
sources: 14
---

# A01.3 · Replication and partitioning

> **Concept.** Replication buys durability and read scale at a tail-latency and RPO cost; partitioning buys write scale but makes skew the default failure mode.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 02 priced one specific replication scheme: a fixed majority, every write durable on a quorum, `leader_fsync + RTT + follower_fsync + apply`. That is the most expensive point on the curve, and you buy linearizability with it. Most of your fleet's data does not need it and cannot afford it.

This lesson maps the rest of the curve. Replication becomes a spectrum — synchronous, asynchronous, tunable quorum, chain — where each scheme trades durability, availability and latency differently, and most of them never run a consensus round at all. Then partitioning adds a concern replication never touches: once one copy of the data cannot absorb the write volume, you have to split the data itself, and the split brings a failure mode that has nothing to do with how many copies you keep. Copies protect against *node* failure. Partitions protect against *volume*. Skew defeats partitioning the way a slow disk defeats consensus — quietly, and by an order of magnitude.

## Why this matters

At staff level the interesting question is never "should we replicate or shard." It is *which point on the guarantee/cost curve*, stated as a number, and *what breaks when the assumption behind that number breaks*. Every replication choice is a bet on failure correlation and an acceptable RPO. Every partition choice is a bet on key distribution that skew will call.

The metadata plane of a GPU fleet — the store holding jobs, quotas, node inventory, allocation records — is where those bets get made. A bad one shows up as a single hot shard throttling the whole control plane during a submission storm, or as an "RPO of 5 seconds" that nobody converted into records, or as a scale-out that browns out the system it was meant to relieve. All three are arithmetic errors, and all three are avoidable in a design review by someone who does the arithmetic out loud.

## What's new here (calibration)

**Skip (carried over):** leader/follower mechanics; "sharding spreads load"; hashing a key to pick a shard.

**New here:**
- **The guarantee as a number.** RPO expressed as `replication_lag × write_rate` — records at risk, not "some risk of data loss." Read availability as `1 − (1−a)^R`. Fan-out tail latency as `F(x)^k`, which is the arithmetic that explains why a 100-shard scatter-gather has a p99 nothing like its shards' p99.
- **Exactly what `W + R > N` buys and exactly what voids it.** The overlap argument drawn out, the failure modes it still permits even when it holds, and the precise mechanism — sloppy quorums with hinted handoff — that suspends it on purpose.
- **Chain replication** as the third option most engineers cannot name: strong consistency at near-async throughput, and the specific cost that keeps it out of most designs.
- **The production partitioning default**, with the data-movement arithmetic for each scheme: `hash mod N` versus a consistent-hashing ring with virtual nodes versus a fixed partition count mapped through metadata — and what each moves on a node join.
- **Skew as the expected case**, with the mitigation menu costed: salting, dedicated shards, shuffle sharding.
- **A live control-plane instance**: KEP-5866's server-side sharded LIST/WATCH, which shards the Kubernetes watch stream by hash range — partitioning applied to the exact system you operate.

Version note: Cassandra, Kafka and Kubernetes defaults below were read from `apache/cassandra`, `apache/kafka` and `kubernetes/enhancements` on GitHub in August 2026. Papers and vendor engineering blogs were not fetchable in this environment; every one of them is marked in References, and no number in this lesson depends on an unfetched source unless it is explicitly attributed as recalled.

## Core concepts

### Two axes, two different problems

Keep these separate or every subsequent argument will be muddled:

| | **Replication** | **Partitioning (sharding)** |
|---|---|---|
| Problem solved | A node dies / reads exceed one node's capacity | Writes or data exceed one node's capacity |
| Unit | A copy of the *same* data | A disjoint *subset* of the data |
| Adding more gives you | Durability, read throughput, locality | Write throughput, total capacity |
| Does not give you | Write throughput (every replica takes every write) | Durability (one copy of each shard is still one copy) |
| Characteristic failure | Divergence, stale reads, lost tail on failover | **Skew** — one shard hot while the rest idle |
| The number that decides | RPO, and read-vs-write availability | Key distribution, and rebalance cost |

Almost every production store does both: partition the keyspace, then replicate each partition. Kafka: topic → partitions → replicas. Cassandra: token ranges → replication factor. Elasticsearch: shards → replicas. The two decisions are independent and you must justify both.

### The replication spectrum: where the acknowledgement happens

Every scheme is a choice about **when the client hears "yes"** relative to when the data is safe elsewhere.

```
  One write. Leader L, replicas R1, R2. Time flows right. ✓ = client sees success.

  ASYNCHRONOUS
    client ──▶ L: append, ✓ immediately
                 └─▶ R1 …later…            RPO WINDOW ▓▓▓▓▓▓▓▓▓▓▓▓▓▓
                 └─▶ R2 …later…            ← if L dies in here, this data
    latency: 1 local write.                  is GONE. It was acknowledged.
    failure : lose the un-shipped tail.

  SYNCHRONOUS (all replicas)
    client ──▶ L: append
                 ├─▶ R1 ack ─────┐
                 └─▶ R2 ack ──────────────────────┐   ← R2 is GC-pausing
                                                   ▼
                                                   ✓ (waits for the SLOWEST)
    latency: max over ALL replicas. RPO = 0.
    failure : one sick replica stalls every write. Availability = AND of all.

  QUORUM (W of N)   ← the tunable middle
    client ──▶ L: append
                 ├─▶ R1 ack ─────┐
                 └─▶ R2 ack ──────────────────────  (still going, ignored)
                                 ▼
                                 ✓ as soon as W acks land
    latency: the W-th fastest, not the slowest. RPO = 0 for W ≥ 2.
    failure : tolerates N−W slow/dead replicas per write.

  CHAIN
    client ──▶ HEAD ──▶ MIDDLE ──▶ TAIL ──✓ (tail acks AND serves all reads)
    latency: sum of the whole chain (worse than quorum).
    throughput: each node sends ONE copy downstream, not N — so the head is
                not a fan-out bottleneck. Strong consistency, near-async
                throughput, and reads never need coordination because the
                tail is by construction the most conservative node.
    failure : any node failure requires reconfiguring the chain.
```

**The three sentences to take away.** Synchronous replication makes your write availability the *conjunction* of every replica's availability — you have made things worse in exchange for RPO 0. A quorum makes it a *majority function*, which is the entire reason quorums exist. Chain replication makes throughput independent of replica count, at the cost of latency proportional to chain length and a reconfiguration on every failure — which is why it shows up in storage systems (where throughput dominates) and rarely in metadata systems (where latency does).

### RPO is not a duration, it is a record count

"Our RPO is 5 seconds" is not a specification; it is half of one. Convert it:

```
  RPO_records = replication_lag × write_rate

  Metadata store, async replica, measured lag p99 = 200 ms, 5,000 writes/s:
      0.2 s × 5,000 /s = 1,000 records at risk on any failover.

  Same system during a submission storm, lag 5 s, 20,000 writes/s:
      5 s × 20,000 /s = 100,000 records at risk.

  Note the multiplication: lag and write rate rise TOGETHER under load, because
  the replica falls behind exactly when there is more to ship. RPO is worst
  precisely when a failover is most likely. Quote the product at peak, never
  the lag at idle.
```

Then ask the question that actually matters: *which* 1,000 records? For a job-metadata store, the most recent writes are the newly-submitted jobs and the just-completed allocations — the highest-value, least-reconstructible records in the system. "We might lose the last second of writes" sounds tolerable until you notice the last second of writes is the only record that a GPU was handed out.

**Kafka has a beautiful worked instance of getting this wrong on purpose.** The producer's `acks` default is `all` (verified in `ProducerConfig`), which sounds like RPO 0 — but "all" means all *in-sync* replicas, and the broker's `min.insync.replicas` default is **1** (verified in `ServerLogConfigs`). If replicas fall out of the ISR, "all" can mean "the leader alone," and an acknowledged write dies with that leader. The guarantee you think you configured is the conjunction of two settings, and the default of the second one silently disarms the first. **Every durability setting has a second setting that decides what it means.** Find it.

### Quorums: the overlap argument, drawn

```
  N = 3 replicas. W = 2 (write must ack on 2). R = 2 (read must gather 2).
  W + R = 4 > N = 3  ⇒ every read set intersects every completed write set.

     write(x=v2), W=2:            read(x), R=2:
     ┌────┬────┬────┐             ┌────┬────┬────┐
     │ A  │ B  │ C  │             │ A  │ B  │ C  │
     │ v2 │ v2 │ v1 │             │ ██ │    │ ██ │   ← reader picks A and C
     └────┴────┴────┘             └────┴────┴────┘
       ack  ack  (behind)          gets v2 and v1 → takes the HIGHER VERSION
                                   ⇒ sees v2. Correct.

  Why intersection is forced: |W| + |R| = 4 tokens into 3 slots ⇒ at least one
  slot is claimed twice (pigeonhole). That slot has the newest write.

  W + R ≤ N breaks it. W=1, R=1, N=3:
     ┌────┬────┬────┐
     │ v2 │ v1 │ v1 │  writer acked on A only; reader happens to ask B ⇒ v1.
     └────┴────┴────┘  Stale, silently, with no error anywhere.
```

Choosing W and R is choosing which operation degrades when replicas are down:

| Config (N=3) | Write available while | Read available while | Latency shape | Use when |
|---|---|---|---|---|
| W=3, R=1 | all 3 up | any 1 up | slow writes, fast reads | read-heavy, writes rare and precious |
| W=2, R=2 | 2 up | 2 up | balanced | the default; symmetric degradation |
| W=1, R=3 | any 1 up | all 3 up | fast writes, slow reads | write-heavy ingest, tolerant readers |
| W=1, R=1 | any 1 up | any 1 up | fastest, **no overlap** | logs, metrics, anything reconstructible |

**Now the part that gets skipped, and that lesson 01 warned about: `W + R > N` is not linearizability.** Three things it still permits:

1. **Concurrent writes with no ordering.** Two clients write v2 and v3 concurrently, each to a different pair. A later read sees both. Which wins? Last-write-wins by timestamp silently discards one (and clock skew decides which); version vectors surface both as siblings and make the application choose. Neither is linearizability — the quorum system has no serialisation point for the two writes.
2. **A partially-completed write.** The writer got 1 ack of the required 2 and returned an error to its client. The value is now on one replica. A subsequent R=2 read may see it and return it — a *failed* write becoming visible. No rollback exists.
3. **Non-monotonic reads across attempts.** Read 1 hits {A,B} and sees v2; read 2 hits {B,C} and sees v1 if the repair has not run yet. The reader goes backwards, exactly the anomaly lesson 01 named.

To get from quorum overlap to something you can reason about you need three more machines, and production systems ship all three:

- **Read repair** — when a read sees divergent versions, write the winner back to the stale replicas synchronously on the read path. Fixes what you touch.
- **Anti-entropy** — background reconciliation using **Merkle trees**: each replica builds a hash tree over its key ranges; two replicas compare root hashes, and if they differ, descend only into differing subtrees. Comparing two 1-million-key replicas costs O(log n) hash exchanges to locate the divergent keys, not a full scan. Fixes what nobody touches.
- **Hinted handoff** — see below, and note it fixes availability while *breaking* the overlap.

### What voids `W + R > N` on purpose: sloppy quorums and hinted handoff

```
  N=3, home replicas for key k are {A,B,C}. W=2. A partition isolates B and C.

  STRICT quorum: only A is reachable ⇒ 1 < W ⇒ the write FAILS.
                 Consistent, unavailable. (This is the PC in PACELC.)

  SLOPPY quorum: take the next healthy nodes on the ring as stand-ins.
                 Write lands on A and D, where D is NOT a home replica for k.
                 D stores it with a HINT: "this belongs to B; deliver when B returns."
                 Client sees success. W satisfied — by the wrong nodes.

     home replicas   {A, B, C}          write set actually used {A, D}
     later read      {B, C}   ← R=2 satisfied, and the intersection with the
                                write set is EMPTY. Overlap guarantee: gone.
                                The read returns the OLD value, with no error.

  Convergence comes later, out of band:
     · hinted handoff replays D → B when B is reachable
       (Cassandra: hints on by default, max_hint_window 3h — after that the
        hint is DROPPED and only anti-entropy repair will fix it)
     · read repair fixes replicas you happen to read
     · anti-entropy/Merkle repair fixes the rest
```

This is a deliberate trade, not a bug: durability of the *write* preserved, recency of the *read* suspended, for the duration of the partition plus the repair lag. It is the mechanism that lets a Dynamo-style store claim "always writeable." The hint window is the number to know — in Cassandra, `max_hint_window: 3h` by default (verified in `conf/cassandra.yaml`), after which hints are discarded and only a full repair restores consistency. **A node down longer than the hint window and never repaired is a permanent silent divergence.** That is why "run repair within `gc_grace_seconds`" is an operational commandment in Cassandra shops, not a suggestion.

### Partitioning scheme 1: range

Assign contiguous key ranges to nodes: `[a–f) → node 1`, `[f–m) → node 2`, and so on.

- **Buys:** ordered scans are native. `WHERE tenant = X AND ts BETWEEN …` reads one contiguous run. Splits are cheap and local: a hot range splits in two.
- **Costs:** the **monotonic-key trap.** If your key starts with a timestamp or an auto-increment ID, *every* write goes to the last range. You have built a distributed system with one active node and N−1 spectators, and no amount of adding nodes helps, because the new writes still land at the tail.
- **Fix:** prefix the key with something high-entropy — a hash bucket, a reversed timestamp, a tenant ID — accepting that you have just destroyed the global scan you chose range partitioning for. You can keep scans *within* a prefix: `(tenant_id, ts)` scans one tenant's history efficiently while spreading tenants across ranges. That composite is usually the right answer.
- **Used by:** HBase, Bigtable, CockroachDB, TiKV, Spanner (splits).

### Partitioning scheme 2: consistent hashing with virtual nodes

The problem it solves is stated most clearly by the thing it replaces. Naive `shard = hash(key) mod N`:

```
  N = 4 → 5.  Key k stays put only if hash(k) mod 4 == hash(k) mod 5.
  For uniform hashes that is true for about 1/5 of keys.
  ⇒ roughly 80% of ALL DATA moves for one node added. Unusable in production.
```

Consistent hashing (Karger et al., 1997) maps both keys and nodes onto the same circular hash space; a key belongs to the first node clockwise from it. Adding a node steals only the arc between it and its predecessor — on average `1/(N+1)` of the keyspace, and only from **one** neighbour.

That "only from one neighbour" is both the point and the flaw, and virtual nodes are the fix:

```
 WITHOUT virtual nodes — 4 physical nodes on the ring (positions from hash(node_id)):

     0 ────────────────────── 2^32
     │   A        B    C        D           │
     ├───┴────────┴────┴────────┴───────────┤
      ↑ A owns the arc from D's position to A's, i.e. everything hashing in
        between. Arc sizes are RANDOM: with n random points on a circle the
        largest arc is ~ln(n)/n of the circle, so with 4 nodes one node can
        easily own 40%+ of the keyspace. Load imbalance is structural.

     Node C dies:   its entire arc goes to D alone.
     ⇒ D now serves ~2× its load. Recovery traffic all comes FROM D's
       predecessor and lands ON D. One node absorbs a whole node's failure.

 WITH virtual nodes — each physical node claims V positions (V=4 shown; real
 systems use 16–256):

     0 ─────────────────────────────────────────────────── 2^32
     │ A₁  C₂  B₁  D₃  A₂  B₄  C₁  D₁  B₂  A₃  D₂  C₃  A₄ … │
     ├──┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┤
       Each node owns MANY small arcs, interleaved with every other node.

     Node C dies: its arcs C₁…C₄ each go to a DIFFERENT successor.
     ⇒ C's load is absorbed by A, B and D roughly equally, and the recovery
       stream is sourced from many nodes in parallel instead of one.

     Load smoothing: the standard deviation of per-node load falls like
     ~1/√V. V=16 → roughly ±25% spread; V=256 → roughly ±6%.
     Cost of large V: the ring metadata every node gossips grows as N×V, and
     range scans/repairs get more fragmented.
```

Cassandra's shipped default is `num_tokens: 16` with `allocate_tokens_for_local_replication_factor: 3` (verified in `conf/cassandra.yaml`), which is a deliberate move away from the old 256: fewer tokens with a *placement-aware allocator* beats many random tokens, because the allocator optimises replicated load directly instead of relying on the law of large numbers. **That is the modern lesson: virtual nodes smooth randomness, but choosing positions deliberately beats smoothing more randomness.**

What consistent hashing costs you: range scans are gone — adjacent keys are deliberately scattered — and it spreads *keys*, never *load* (see skew, below).

### Partitioning scheme 3: a fixed partition count mapped through metadata (the production default)

```
  Two-level mapping. THE FIRST LEVEL NEVER CHANGES.

    key ──hash──▶ partition_id ∈ [0, 1024)      ← fixed at creation, immutable
    partition_id ──lookup──▶ node               ← a metadata table you rewrite

  8 nodes, 1024 partitions = 128 partitions/node:

    ┌──────── partition → node map (the ONLY thing that moves) ────────┐
    │ p0..p127→n1  p128..p255→n2  …  p896..p1023→n8                    │
    └──────────────────────────────────────────────────────────────────┘

  Scale 8 → 10 nodes: target 1024/10 ≈ 102 partitions each.
    n1..n8 each give up 128−102 = 26 partitions; n9 and n10 receive ~102 each.
    Data moved = 1024 × (1/8 − 1/10) = 1024 × 0.025 ≈ 26 partitions per old
    node ≈ 205 partitions ≈ 20% of the data — and NOT ONE KEY IS REHASHED.

  Migration of one partition, live:
    1. n9 starts as a follower of p5 on n1, streaming a snapshot then the tail
    2. when caught up, flip the map entry p5 → n9  (a metadata write, atomic)
    3. clients with a stale map hit n1, which returns "moved" and they refresh
    4. n1 drops its copy of p5
  Rollback at any point before step 2 is free: nothing has changed.
```

Choosing the partition count is the one irreversible decision. Rules of thumb: high enough that a single partition fits comfortably on one node at your projected maximum size (a partition is the unit of movement, so a 500 GB partition is a 500 GB migration); high enough to divide evenly among your maximum node count; low enough that per-partition overhead — file handles, memory, heartbeats, metadata rows — stays reasonable. Powers of two between 256 and 4,096 are the usual landing zone. Kafka's `num.partitions` default is **1** (verified), which is right for a default and wrong for everything you will actually run.

| Scheme | Data moved on node join | Scans | Hot-key resilience | Metadata cost | Production examples |
|---|---|---|---|---|---|
| `hash mod N` | ~`1 − 1/max(N,N+1)` ≈ **most of it** | none | none | zero | Nothing that has ever scaled |
| Range | Only the split range | **Excellent, native** | Poor by default (monotonic-key trap) | Range map, grows with splits | HBase, Bigtable, CockroachDB, Spanner |
| Consistent hashing + vnodes | ~`1/(N+1)`, sourced from many peers | None | Moderate — spreads keys, not load | Ring positions gossiped, O(N×V) | Cassandra, Riak, Dynamo |
| Fixed partitions + metadata map | ~`1/N_old − 1/N_new` of data, whole partitions only | Within a partition | Same as hashing | One table, O(P) rows | Kafka, Elasticsearch, Citus, Uber Schemaless (reported) |

### Skew is the default case, not the exception

A uniform hash gives every *key* an equal chance of landing on a shard. Real traffic is not uniform across keys — it is Zipfian, and the top few keys or tenants carry a share of load that no hash function can dilute. **Hashing spreads keys; it preserves load skew exactly.** If one tenant is 30% of your writes and all its rows share a partition key, one shard is 30% of your writes no matter how good the hash.

The mitigation menu, with what each costs:

| Mitigation | Mechanism | Cost | When |
|---|---|---|---|
| **Key salting / splitting** | Append a bucket to the hot key: `tenant#0` … `tenant#S-1`, chosen randomly on write | Reads must scatter-gather across all S buckets, which drags in the fan-out tail-latency math below | A few known-hot keys, write-dominated |
| **Dedicated shards** | Route named whales to their own physical shards | Manual, and it is a routing table someone must maintain and get right | A handful of large, stable tenants |
| **Request coalescing** | Collapse concurrent identical reads into one backend fetch | Only helps read-hot keys; adds a synchronisation point | Read-dominated hot keys (see lesson 04) |
| **Shuffle sharding** | Each tenant gets a pseudo-random *subset* of shards rather than one | Does not reduce the hot tenant's load; it bounds who else that tenant can hurt | Multi-tenant blast-radius containment |
| **Adaptive splitting** | The store detects a hot range and splits it automatically | Only works for range partitioning; can thrash | Range-partitioned stores |

**Shuffle sharding deserves the extra sentence** because it solves a different problem than the others. With 8 shards and each tenant assigned 2, there are C(8,2) = 28 distinct pairs. One abusive tenant degrades its 2 shards; another tenant collides with it completely only if it drew the same pair — probability 1/28 ≈ 3.6% — and partially (one shard shared) with probability about 21%. It does not make the fleet faster; it makes one bad tenant's blast radius a computable fraction instead of "everyone."

You have already seen this arithmetic in a system you operate, incidentally: Kubernetes' API Priority and Fairness assigns request flows to queues by shuffle sharding, and the published table gives the exact collision probabilities — with hand size 8 and 64 queues, one "elephant" flow squishes a given "mouse" flow with probability ~2.3×10⁻¹⁰, rising to ~0.36 with 16 elephants. Lesson 05 uses this properly; here it is the proof that shuffle sharding is a production primitive, not an AWS blog-post idea.

### Fan-out and the tail: why scatter-gather changes your latency model

Every mitigation that splits a key across shards, and every query that must touch many shards, buys into this arithmetic:

```
  A request fans out to k shards and must wait for ALL of them.
  If per-shard latency is IID with CDF F, then

      P(request ≤ x) = F(x)^k

  Per-shard p99 = 10 ms  ⇒ F(10 ms) = 0.99.

      k = 1   → P(≤10 ms) = 0.99      99% of requests under 10 ms
      k = 10  → 0.99^10  = 0.904      only 90% under 10 ms
      k = 32  → 0.99^32  = 0.725
      k = 100 → 0.99^100 = 0.366      ⇒ 63% of requests exceed the per-shard p99

  Turn it around: to hold a p99 for the whole request at k = 100, each shard
  must satisfy  F(x)^100 = 0.99  ⇒  F(x) = 0.99^(1/100) = 0.9999.
  Each shard needs its p99.99 — not its p99 — inside your budget.
```

Three design consequences follow directly. **Do not fan out further than you must** — salt only the tenants that need it, and pick the smallest S that fixes the hot shard. **Hedged requests** (send a duplicate to a second replica after a delay near p95, take the first answer) convert a per-shard tail into a per-replica-pair tail and are the standard fix. **Partial results** — return what k−1 shards produced and mark the response incomplete — are often better product behaviour than making every user wait for the slowest shard.

### Rebalancing without causing the outage you were preventing

Rebalancing moves data while the system serves traffic, using the same disks, the same NICs and the same page cache. Uncontrolled, it is an act of self-harm: you add capacity to relieve pressure and the copy traffic browns out the system before the new capacity is usable.

A rebalance plan needs five things, and a design review should ask for all five by name:

1. **A rate limit**, expressed as a fraction of a resource you measure — e.g. copy bandwidth capped at 20% of per-node write-path capacity, or a fixed MB/s per source node. Not "as fast as possible."
2. **A concurrency limit** — how many partitions move at once. One at a time is often correct and rarely feels fast enough.
3. **An SLI to watch and an abort condition** — "if p99 read latency on the source node exceeds X for Y minutes, pause." Automatic, not "someone is watching Grafana."
4. **Atomic cutover with rollback** — the map flip is one write; before it, abandoning costs nothing.
5. **Explicit ownership of the double-write window** — during migration, does the destination take writes, or does the source stay authoritative until cutover? Both are valid; ambiguity is not.

Do the arithmetic before you start: moving 20% of a 4 TB dataset at 200 MB/s takes `800 GB ÷ 0.2 GB/s = 4,000 s ≈ 67 min` of continuous copying — plus the tail catch-up. If that is unacceptable, the fix is a lower rate over a longer window, not a higher rate.

### Where this lands on a GPU platform

**1. The Kubernetes control plane deliberately does not partition — and that is why it has a size limit.** etcd is one Raft group; every key is on every member. There is no sharding, so the ceiling is one machine's capacity and one consensus group's write rate, which is exactly why Kubernetes publishes node and pod limits (5,000 nodes, 150,000 pods) rather than "add more etcd." Operators partition *around* it instead: separate etcd clusters per resource class (the Events split), or more clusters rather than bigger ones. **A design that "shards etcd" is either running multiple independent clusters or is confused.**

**2. Sharding is arriving at the watch stream, and the design is worth reading.** KEP-5866 (*Server-side sharded LIST and watch*) adds a selector of the form `shardRange(object.metadata.uid, start, end)`: the API server hashes the field and dispatches only events falling in the requested range. The motivation is the exact thing this lesson is about — client-side sharding (as `kube-state-metrics` does today) does not reduce ingress, since every replica still receives and deserialises the full stream and discards what is not its. The KEP is explicit that coordination and resharding are *not* in scope: clients own the range assignment. That is the honest shape of every partitioning design — the mapping is the easy half, and rebalancing is the half that takes years.

**3. Your metadata store is the real partitioning exercise.** Jobs, quotas, allocations, node inventory. It is multi-tenant (so Zipfian), it is queried both by point lookup (job ID) and by scan (tenant + time), and it has one tenant that is 10× the next. This is the system the Worked example designs.

**4. Training-parameter sharding is an analogy, and the failure model is different — do not mix them.** Data, tensor and pipeline parallelism, and ZeRO's sharding of optimizer state, gradients and parameters across ranks, do partition state across machines, and all-reduce does look like an all-to-all quorum. But a replicated store degrades *gracefully*: lose one shard and you lose access to a slice of the keyspace while the rest serves. A training step is **gang-scheduled and all-or-nothing**: one rank dies and the step dies, the collective hangs, and recovery means restarting from the last checkpoint. Reasoning about training resilience with database-replication intuition ("we'll just lose a shard") systematically under-provisions checkpointing and over-estimates resilience. Lesson 06 does the checkpoint-interval math; the point here is to keep the two failure models apart.

## Perspectives

**The data-modeller's view.** The partition key comes from the dominant access pattern, not the entity-relationship diagram. Point lookup by ID favours hash or fixed partitions; time-ordered scans favour range; per-tenant scans favour co-locating a tenant's rows — which immediately makes that tenant a hot-shard candidate. Composite keys are how you get both: `(tenant_id, ts)` spreads tenants while keeping each tenant's history contiguous. Picking a key because it is the primary key on the ER diagram is how a scheme ends up clean on paper and hot in production.

**The operator's view.** The distinction that decides your on-call life is *metadata map* versus *rehash*. In the metadata-map world, moving data is: copy, verify, flip one map entry, retire the old copy — throttled, resumable, and reversible until the flip. In the `hash mod N` world there is no cheap partial move, so you are choosing between downtime and an elaborate bespoke live-rehash. This is the entire reason production systems fix the partition count up front. Corollary: your rebalance tooling should be exercised routinely, not first used during an emergency scale-out.

**The multi-tenancy view.** Assume Zipf. A handful of tenants will carry a disproportionate share of every resource you measure, and that is normal customer behaviour, not abuse. Design for it explicitly: know your top-10 tenants by write rate, have a salting or dedicated-shard plan on the shelf before you need it, and use shuffle sharding to bound how much of the fleet any one tenant can affect. Then measure the actual distribution rather than assuming it — the ratio between your p50 and p99 tenant is the number that decides whether you need any of this.

**The failure-model view.** Replication and partitioning give **graceful partial availability**: lose a replica and you lose margin; lose a shard and you lose a slice of the keyspace. This is a *different* failure model from a gang-scheduled training job, where losing one rank kills the whole step, and from a quorum system, where losing a majority stops everything. A staff engineer names which model applies before proposing a mitigation, because the mitigations do not transfer: retries and partial results are right for the first, checkpointing is right for the second, and quorum sizing is right for the third.

## Real-world use cases

- **KEP-5866, *Server-side sharded LIST and watch*** (verified, read from `kubernetes/enhancements`). Adds `selector=shardRange(object.metadata.uid, start, end)` so the API server filters the watch stream server-side by hash range. The motivation section is a clean statement of why client-side sharding is not sharding: "every replica still receives the full stream of events, paying the CPU and network cost to deserialize everything, only to discard items not belonging to their shard… Functionally, this makes horizontal scaling of the watch stream impossible." **What it shows:** partitioning applied to a system you run daily, and an explicit acknowledgement that coordination and resharding are the hard parts, deliberately left out of scope.
- **Cassandra's shipped defaults** (verified in `apache/cassandra`, `conf/cassandra.yaml`). `num_tokens: 16` with `allocate_tokens_for_local_replication_factor: 3`; `hinted_handoff_enabled: true` with `max_hint_window: 3h`. **What it shows:** the modern virtual-node position — a modest token count plus a replication-aware allocator beats a large random token count — and a concrete, checkable bound on how long a sloppy quorum's repair debt is retained before it becomes a silent divergence only anti-entropy can fix.
- **Kafka's `acks=all` with `min.insync.replicas=1`** (both verified in `apache/kafka`). **What it shows:** the general pattern that a durability guarantee is the conjunction of at least two settings, and that shipped defaults optimise for "it works out of the box," not for "your data survives." The fix is `min.insync.replicas=2` with `replication.factor=3`, which is the only combination where `acks=all` means what people think it means.
- **Dynamo (DeCandia et al., SOSP 2007)** — the origin of the sloppy quorum, hinted handoff, vector-clock reconciliation and Merkle-tree anti-entropy described above, and of the `N`/`W`/`R` vocabulary itself. **Not fetched this pass** (allthingsdistributed.com and the ACM DL are blocked here); the mechanisms are described from standard knowledge and cross-checked against Cassandra's implementation, which is a direct descendant. Do not quote specific Dynamo numbers from memory — including the frequently-repeated `N=3, W=2, R=2` configuration and the 99.9th-percentile SLA — without re-reading the paper.
- **Uber Engineering, *The Architecture of Schemaless*** — the canonical public example of a fixed shard count (reported as 4,096) mapped to nodes through metadata rather than `hash mod N`. **Not fetched this pass** (uber.com blocked); treat the shard count as recalled. The pattern itself is independently visible in Kafka, Elasticsearch and Citus.
- **Discord, *How Discord Stores Trillions of Messages*** — a three-generation partitioning story (MongoDB → Cassandra → ScyllaDB) driven by hot-partition and coordination pain at growing node counts. **Not fetched this pass** (discord.com blocked); cited for the shape of the evolution, not for its figures.
- **Vitess / YouTube resharding** and **AWS Builders' Library, *Workload isolation using shuffle-sharding*** — the reference accounts of live throttled resharding and of shuffle sharding as a blast-radius primitive. **Not fetched this pass**; the shuffle-sharding arithmetic used above is derived from first principles and cross-checked against Kubernetes' published APF collision table, which *was* verified.

## Worked example

**Design the GPU-fleet metadata store.** Requirements: 500 M job records; 50,000 writes/s at peak; the largest tenant is 30% of traffic; dominant queries are (a) point lookup by `job_id` and (b) "my jobs in the last hour" by `(tenant_id, ts)`; a lost allocation record is a billing incident.

**Step 1 — replication: pick N, W, R and state what they buy.**

Choose `N = 3`, `W = 2`, `R = 2`. Overlap holds (`4 > 3`). Writes survive one replica down; reads survive one replica down; commit latency is the second-fastest replica, not the slowest. RPO for an acknowledged write is 0 — two durable copies before the client hears success.

Now state the failure honestly: **if you enable sloppy quorums, the overlap guarantee is suspended during a partition** and a read can miss an acknowledged write until hinted handoff replays. For a billing-relevant store, disable sloppy quorums (accept write unavailability during a partition — this is a PC/EC choice, in lesson 01's vocabulary) or route allocation records through a separate strict path. **Pick per-table, not per-cluster.**

**Step 2 — the partition key, and why the obvious one is wrong.**

*Candidate A: `hash(job_id) → 1024 partitions`.* Perfect spread for writes and point lookups. Query (b) becomes a 1,024-way scatter-gather — and by the fan-out arithmetic, with per-shard p99 = 10 ms the whole query is over 10 ms about 99.996% of the time. Unusable for the tenant dashboard.

*Candidate B: `hash(tenant_id) → 1024 partitions`.* Query (b) is one partition. But the 30% tenant is now one partition:

```
  Total peak writes             50,000 /s
  Whale tenant (30%)            15,000 /s  → lands on ONE partition
  Mean per partition            50,000 / 1024 ≈ 49 /s
  Hot/mean ratio                15,000 / 49 ≈ 307×
```

That partition saturates first, throttles the whale *and* every tenant co-resident on it, while 1,023 partitions idle. Hashing did not help because the input was skewed and hashing preserves skew exactly.

*Candidate C — the answer: composite key with salting for flagged whales.*

```
  partition_key = hash(tenant_id, salt)   where
      salt ∈ {0}            for ordinary tenants  (S = 1)
      salt ∈ {0 … 31}       for flagged whales    (S = 32, chosen at write time)
  clustering key = (ts, job_id)            ← keeps query (b) ordered inside a partition
  secondary index / lookup table: job_id → partition_key   ← keeps query (a) a point lookup

  Whale load after salting:  15,000 / 32 ≈ 469 /s per sub-partition
  Ratio to the mean (49/s):  ≈ 9.6×  — still the hottest, but now within one
                             node's capacity rather than 307× over it.
  Choose S from the target, not from a round number:
      S ≥ whale_rate / (acceptable_multiple × mean_rate)
        = 15,000 / (10 × 49) ≈ 31   → S = 32.

  Query (b) for the whale now fans out to 32 sub-partitions:
      per-shard p99 10 ms ⇒ P(all 32 ≤ 10 ms) = 0.99^32 = 0.725
      ⇒ 27% of whale dashboard queries exceed 10 ms.
      Mitigate with hedged requests, or accept 32 as the cost of not being down.
  Ordinary tenants are unaffected: S = 1, one partition, no fan-out.
```

**Step 3 — layout and growth.**

1,024 fixed partitions mapped to nodes through a metadata table. Start at 8 nodes = 128 partitions each; a partition holds ~500 M/1,024 ≈ 490 K records. Scaling 8 → 10 nodes moves `1024 × (1/8 − 1/10) ≈ 26` partitions off each existing node, ~205 partitions total, ~20% of the data — and rehashes nothing. Compare with `hash mod N`, which would move ~80% and require a full rewrite.

**Step 4 — the rebalance plan, with the abort condition.**

```
  Data to move:      20% of 4 TB = 800 GB
  Rate cap:          200 MB/s aggregate (≈20% of per-node write-path capacity)
  Copy time:         800 GB ÷ 0.2 GB/s = 4,000 s ≈ 67 min, plus tail catch-up
  Concurrency:       4 partitions in flight, ≤1 per source node
  SLI watched:       p99 read latency per source node, and write-path saturation
  Abort condition:   p99 > 1.5× the 24 h baseline for 5 min ⇒ pause, alert,
                     do not auto-resume
  Cutover:           per-partition metadata flip, atomic; stale clients get a
                     "moved" response and refresh their map
  Rollback:          free before the flip (destination copy is discardable)
```

Run it during the low-water period, and expect it to take a shift rather than a coffee break. **A slow correct rebalance beats a fast brownout**, and the number that makes that argument in a review is the 67 minutes above, not the sentiment.

**Step 5 — availability arithmetic, so the design has a number.**

```
  Per-replica availability a = 0.995 (unavailable ~3.6 h/month).

  A partition with N=3, R=2 (need any 2 of 3 up to serve a read):
      P(available) = a³ + 3a²(1−a) = 0.999925…
      ⇒ ~3.2 s of unavailability per partition per month.

  The store as a whole for query (b) — one partition, ordinary tenant:
      same as a partition: ≈ 99.9925%.

  The whale's dashboard query — needs ALL 32 sub-partitions:
      0.999925^32 ≈ 0.99760  ⇒ ~1.7 h/month for that ONE query type.

  This is the lesson in one line: FAN-OUT MULTIPLIES UNAVAILABILITY the same
  way it multiplies tail latency. The whale you salted for throughput now has
  the worst availability of any tenant. Fix it with partial results (answer
  from the sub-partitions that responded, flag the response incomplete) rather
  than by adding replicas, which is far more expensive per nine.
```

**Step 6 — the RPO line for the disaster-recovery replica.**

The cross-region replica is asynchronous — synchronous across regions would put a 30–80 ms RTT inside every commit. Measured lag p99 = 800 ms at peak:

```
  RPO_records = 0.8 s × 50,000 /s = 40,000 records at risk on a region failover.
```

If 40,000 job records is unacceptable, the options are: raise replica throughput to cut the lag (buys a linear improvement), make *allocation* records synchronous while leaving the rest async (buys the improvement only where it matters — usually the right answer), or accept it and write down the reconciliation procedure. **What you may not do is quote "sub-second RPO" and stop.**

## Practice

*Feeds the [staff design portfolio](../practice/staff-design-portfolio/README.md).*

Write a two-page design for the GPU-fleet **scheduler metadata store**. Required sections:

1. **Replication.** State N, W, R and the exact RPO you are buying, in records, at peak — showing `lag × write_rate` with your own numbers. Say what happens during a partition and whether you have enabled sloppy quorums; if you have, state which tables can tolerate the suspended overlap guarantee and which cannot, and give the hint-window bound.
2. **Partitioning.** Pick a scheme and justify it against your dominant queries — point lookup versus tenant+time scan. Show the data-movement arithmetic for a node-count change under your scheme, and under `hash mod N`, so the difference is on the page.
3. **Skew.** Identify your hot-shard scenario with real per-shard numbers. Derive `S` for salting from a target hot/mean ratio rather than picking a round number, and show the fan-out cost your salting introduces — both the tail-latency term `F(x)^S` and the availability term `p^S`.
4. **Rebalance runbook.** Data volume, rate cap, expected duration (computed), concurrency, the SLI you watch, the numeric abort condition, the cutover step, and what rollback costs before and after cutover.
5. **Failure-model paragraph.** One paragraph explicitly contrasting this system's partial-availability model with training's gang/all-or-nothing model, naming one mitigation that is right for each and wrong for the other.

Stretch: take the fan-out table (`F(x)^k` for k = 1, 10, 32, 100) and compute it with your *own* measured per-shard latency distribution rather than assuming a clean p99. The gap between the two is usually where the surprise lives.

## Common pitfalls

1. **"A uniform hash fixes hot shards."** Hashing distributes *keys* uniformly and preserves *load* skew exactly. Symptom: one shard at 300% of mean with a hash function that tests perfectly uniform. Mechanism: the input distribution is Zipfian; hashing is a bijection on keys, not a redistribution of traffic. Fix the input (salting, dedicated shards), not the hash.
2. **`hash mod N` as "just sharding."** It couples key→shard to node count, so any change reshuffles most of the data. Symptom: "we can't scale out without downtime." Mechanism: `hash(k) mod N` and `hash(k) mod (N+1)` agree for only ~`1/(N+1)` of keys. Fix the partition count up front and remap partition→node instead.
3. **"Synchronous replication is safer, full stop."** It gives RPO 0 and makes write availability the conjunction of every replica's availability, with commit latency bound by the *slowest* replica. Symptom: one GC-pausing or gray-failing replica stalls every write cluster-wide. Mechanism: waiting for all means waiting for the max. Use a quorum unless you genuinely need every copy.
4. **Quoting RPO as a duration.** "5 seconds" is not a loss budget. Symptom: a failover that loses far more than anyone expected. Mechanism: `records = lag × write_rate`, and both terms peak together under load. Always quote the product at peak, and say which records they are.
5. **"`W + R > N` means my reads are correct."** It guarantees set intersection with *completed* writes only. It still permits concurrent writes with no ordering, visible partially-failed writes, and non-monotonic reads between attempts — and sloppy quorums suspend it entirely by design. Symptom: a stale read that no metric reported. Fix: know whether sloppy quorums are on, and pair the quorum with read repair and scheduled anti-entropy.
6. **Treating sloppy quorums as a bug.** They are a deliberate availability trade with a documented repair window (Cassandra's `max_hint_window`, 3 h by default). Symptom: a node down longer than the window, never repaired, silently divergent forever. Mechanism: hints are dropped after the window; only anti-entropy repair fixes it after that. Schedule repair.
7. **Rebalancing at full speed.** Symptom: adding capacity causes the outage the capacity was meant to prevent. Mechanism: copy traffic competes with live traffic for the same disks and NICs on the *source*, which is already the loaded node. Always cap the rate, cap the concurrency, and set a numeric abort condition.
8. **Ignoring the fan-out multiplier.** Symptom: a scatter-gather query whose p99 is nothing like its shards' p99, and whose availability is worse than any single shard's. Mechanism: `F(x)^k` for latency and `a^k` for availability — waiting for all of k things is exponentially worse in k. Fix with hedged requests, partial results, or fewer shards touched per query.
9. **Reasoning about training sharding with database intuition.** Symptom: an under-provisioned checkpoint interval justified by "distributed systems degrade gracefully." Mechanism: a gang-scheduled step is all-or-nothing; there is no partial availability to degrade into. Keep the models separate.

## Self-check

- **Why can `W + R > N` still return a stale read?** **Answer:** Two reasons. First, by design: sloppy quorums with hinted handoff accept writes on substitute nodes during a partition, so the W nodes that acknowledged may not intersect the R nodes later read — the pigeonhole argument requires both sets to be drawn from the same N home replicas, and a sloppy write is not. Convergence comes later via hinted handoff (bounded by the hint window — 3 h by default in Cassandra), read repair and Merkle-tree anti-entropy. Second, even with strict quorums, overlap only relates a read to *completed* writes: concurrent writes have no order, a partially-completed write can become visible, and two successive reads hitting different replica subsets can go backwards before repair runs.

- **Derive the data movement for a node join under each scheme.** **Answer:** `hash mod N`: a key stays put only if `hash(k) mod N == hash(k) mod (N+1)`, true for about `1/(N+1)` of keys, so roughly `N/(N+1)` — most of the data — moves. Consistent hashing: the new node's ring positions claim the arcs between them and their predecessors, about `1/(N+1)` of the keyspace, sourced from one neighbour per position (which is why virtual nodes matter: with V positions the load comes from up to V different peers in parallel). Fixed partitions with a metadata map: partitions move whole, `P × (1/N_old − 1/N_new)` of them, and no key is ever rehashed — so 8 → 10 nodes with 1,024 partitions moves ~205 partitions ≈ 20% of the data as a series of independently abortable copies.

- **What do virtual nodes actually fix, and what does the token count trade?** **Answer:** Two things. Load balance: with N random points on a ring the arc sizes are highly variable (the largest is ~ln(N)/N of the circle), so a few physical nodes give a badly skewed distribution; V positions per node reduce the per-node load spread as roughly 1/√V. And failure absorption: without virtual nodes, a dead node's whole arc lands on its single successor, doubling that node's load and sourcing all recovery traffic from one peer; with virtual nodes the arcs scatter across many successors and recovery streams in parallel. The trade is metadata size (O(N×V) ring positions to gossip) and fragmentation of scans and repairs. Cassandra's shipped default of `num_tokens: 16` plus a replication-aware allocator reflects the modern view that *deliberate* placement beats *more* randomness.

- **Your dashboard query fans out to 100 shards, each with a 10 ms p99. What is the query's p99?** **Answer:** Not 10 ms. `P(all 100 ≤ 10 ms) = 0.99^100 ≈ 0.366`, so about 63% of requests exceed 10 ms — 10 ms is roughly the p37, not the p99. To hold a real 10 ms p99 at k = 100 you need each shard at `0.99^(1/100) = 0.9999`, i.e. its p99.99 inside the budget. Practical fixes: reduce k (touch fewer shards per query), hedge (duplicate the request to another replica after a p95-ish delay and take the first response), or return partial results. The same exponent applies to availability: `a^k` means fan-out multiplies unavailability too.

- **Async replication with 5 s lag: what is your actual exposure?** **Answer:** The lag alone is not a loss budget. `RPO_records = lag × write_rate`, so 5 s at 5,000 writes/s is 25,000 records; at a 20,000 writes/s peak it is 100,000. Worse, lag and write rate rise together — the replica falls behind precisely when there is most to lose — so the number to quote is the product at peak, not at idle. Then name *which* records: the most recent writes are the newest allocations and job submissions, typically the least reconstructible data in the system.

- **`acks=all` is set. Is the write durable?** **Answer:** Not necessarily. "All" means all replicas currently *in sync*, and Kafka's `min.insync.replicas` default is 1, so if followers have fallen out of the ISR, "all" can mean the leader alone and an acknowledged write dies with it. The combination that means what people intend is `replication.factor=3`, `min.insync.replicas=2`, `acks=all` — then a write needs two durable copies and the partition goes read-only rather than accepting under-replicated writes. The general rule: every durability setting has a second setting that defines it; find the second one.

- **How would you shard a Kubernetes controller, and what does the platform now give you?** **Answer:** Historically you could only shard client-side: every replica watches everything and discards what is not its shard, which spends the same network and CPU per replica and therefore does not scale the watch stream at all (`kube-state-metrics` is the standard example). KEP-5866 adds server-side sharding via a selector like `shardRange(object.metadata.uid, start, end)`: the API server hashes the field and dispatches only matching events, so each replica's ingress falls proportionally. Note what the KEP explicitly leaves out — coordination between replicas and resharding — because that is the hard half of every partitioning system: assigning ranges is easy, moving them safely while everything is running is not.

## Connections & what's next

This lesson generalises lesson 02's fixed majority into a dial. "Quorum" stops being a consensus requirement and becomes `W` and `R`, chosen against durability, availability and latency — and lesson 01's warning is now proven: quorum overlap is not linearizability, and sloppy quorums suspend even the overlap.

Forward, **lesson 04** takes the spectrum one step past async: a cache is a replica that has given up on correctness entirely in exchange for latency, which turns the staleness window from a bound to be minimised into the normal operating condition. The hot-key problem you just salted for reappears there as the stampede, and the fan-out arithmetic reappears as the herd. **Lesson 05** formalises what happens when a hot shard that salting did not fully absorb keeps taking traffic: queue growth, admission control, and the metastable failure. **Lesson 06** returns to the failure-correlation assumption underneath every availability number computed here — the arithmetic assumed independence, and gray and correlated failures are exactly where that assumption dies.

Carry forward: *if a replica optimised for durability costs you an fsync and a round trip, what does a replica optimised purely for latency cost you — and what does the system behind it do when that replica disappears?*

## References & further reading

**Primary sources — verified against upstream Git repositories this pass**

1. **KEP-5866, *Server-side sharded LIST and watch*** — `keps/sig-api-machinery/5866-server-side-sharded-list-and-watch/README.md` in <https://github.com/kubernetes/enhancements>. **Source of** the `shardRange(object.metadata.uid, start, end)` selector grammar, the kube-state-metrics motivating story, the quoted argument that client-side sharding cannot scale the watch stream, and the explicit non-goals (coordination, resharding, sharding kube-controller-manager).
2. **`apache/cassandra`, `conf/cassandra.yaml`** — <https://github.com/apache/cassandra>. **Source of** `num_tokens: 16`, `allocate_tokens_for_local_replication_factor: 3` and the comment explaining that the allocator optimises replicated load rather than relying on random placement, `hinted_handoff_enabled: true`, and `max_hint_window: 3h`.
3. **`apache/kafka`** — <https://github.com/apache/kafka>. **Source of** `num.partitions` default 1 (`ServerLogConfigs.NUM_PARTITIONS_DEFAULT`, and `config/server.properties`), `min.insync.replicas` default 1 (`ServerLogConfigs.MIN_IN_SYNC_REPLICAS_DEFAULT`), and the producer `acks` default of `all` (`ProducerConfig`).
4. **Kubernetes API Priority and Fairness documentation** — `content/en/docs/concepts/cluster-administration/flow-control.md` in <https://github.com/kubernetes/website>. **Source of** the shuffle-sharding collision-probability table cited above (hand size × queues versus 1, 4 and 16 elephants). Lesson 05 uses this document in depth.

**Primary sources — not fetchable in this environment, not relied on for any number above**

5. **DeCandia, G. et al. (2007), *Dynamo: Amazon's Highly Available Key-value Store*, SOSP** — <https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf>. The origin of `N`/`W`/`R`, sloppy quorums, hinted handoff, vector-clock reconciliation and Merkle-tree anti-entropy. **Blocked by this environment's egress proxy.** The mechanisms above are described from standard knowledge and cross-checked against Cassandra's implementation; the paper's specific configuration and SLA numbers are deliberately not quoted.
6. **Karger, D. et al. (1997), *Consistent Hashing and Random Trees*, STOC** — DOI <https://doi.org/10.1145/258533.258660>. The ring construction. **Not fetched** (ACM DL blocked). The `1/(N+1)` movement result and the largest-arc behaviour are standard results restated here, not quoted.
7. **van Renesse, R. & Schneider, F. (2004), *Chain Replication for Supporting High Throughput and Availability*, OSDI** — <https://www.cs.cornell.edu/home/rvr/papers/OSDI04.pdf>; and **Terrace, J. & Freedman, M. (2009), *Object Storage on CRAQ*, USENIX ATC**, which extends chain replication to allow reads at any node using clean/dirty version tracking. **Not fetched.** Read CRAQ second — it is the version that makes chain replication practical for read-heavy workloads.
8. **Elhemali, M. et al. (2022), *Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service*, USENIX ATC** — <https://www.usenix.org/system/files/atc22-elhemali.pdf>. How a production quorum store evolved past the original Dynamo design, including adaptive capacity for hot partitions — directly relevant to the skew section. **Not fetched** (usenix.org blocked).
9. **Dean, J. & Barroso, L. (2013), *The Tail at Scale*, CACM 56(2)** — the source of the fan-out tail-latency argument and of hedged requests as the standard mitigation. **Not fetched.** The `F(x)^k` arithmetic above is derived from first principles and can be re-derived by anyone; the paper's contribution is the production techniques.

**Real-world engineering — not fetchable this pass**

10. **Uber Engineering, *The Architecture of Schemaless*** — <https://www.uber.com/en-BG/blog/schemaless-part-two-architecture/>. The fixed-shard-count-plus-metadata-map pattern in production; shard count reported as 4,096. **Blocked**; treat the number as recalled and re-check before citing.
11. **Discord, *How Discord Stores Trillions of Messages*** — <https://discord.com/blog/how-discord-stores-trillions-of-messages>. MongoDB → Cassandra → ScyllaDB, driven by hot-partition and coordination pain. **Blocked**; cited for the shape of the evolution only.
12. **Vitess, *Cloud Native MySQL Sharding with Vitess and Kubernetes*** — <https://vitess.io/blog/2015-10-06-cloud-native-mysql-sharding-with-vitess-and-kubernetes/>. Live, throttled, zero-downtime resharding at YouTube scale — the production version of this lesson's rebalance plan. **Blocked.**
13. **AWS Builders' Library, *Workload isolation using shuffle-sharding*** — <https://aws.amazon.com/builders-library/workload-isolation-using-shuffle-sharding/>. **Blocked**; the combinatorial argument above is derived directly and cross-checked against Kubernetes' verified APF table.

**Deeper dives**

14. **Kleppmann, M., *Designing Data-Intensive Applications*, chapters 5 and 6** — the fuller treatment of replication (leader-based, multi-leader, leaderless, and the read-your-writes/monotonic-reads consequences) and partitioning (by key range, by hash, secondary indexes, rebalancing strategies, request routing). Chapter 6's "rebalancing strategies" section is the direct ancestor of this lesson's scheme comparison; read it alongside chapter 5's "problems with replication lag."
