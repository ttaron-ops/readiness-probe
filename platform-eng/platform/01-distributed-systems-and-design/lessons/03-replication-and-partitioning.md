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
sources: 10
---

# A01.3 · Replication and partitioning

> **Concept.** Replication buys durability and read scale at a tail-latency and RPO cost; partitioning buys write scale but makes skew the default failure mode.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 02 turned "a write needs a majority" into an exact number: `leader_fsync(WAL) + RTT_to_slowest_majority + follower_fsync`, and showed a quorum's job is to stay linearizable while surviving node loss. Replication generalizes that single "majority for a strongly consistent log" into a full spectrum — sync, async, tunable quorum (W+R>N), chain — each buying a different point on the durability/availability/latency curve, most of them without paying for full Raft-style consensus at all. Partitioning then adds a concern replication alone never touches: once one copy of the data can't keep up with write volume, you have to split the data itself, and that split brings its own failure mode — skew — that has nothing to do with how many copies you keep. This lesson covers both axes and sets up lesson 04, which asks what happens when a "replica" is built for latency instead of durability.

## Why this matters

At staff level the interesting question is never "should we replicate/shard" — it's *which point on the guarantee/cost curve*, stated as a number, and *what breaks when the assumption behind that number breaks*. Every replication choice is a bet on the correlation of failures and the acceptable RPO; every partition choice is a bet on key distribution that skew will call. The metadata plane of a GPU fleet (the etcd/K8s-adjacent store holding jobs, quotas, node inventory) is where these bets get made, and where a bad one shows up as a single hot shard throttling the whole control plane during a submission storm.

## What's new here (calibration)

- **Skip (carried over):** leader/follower mechanics; "sharding spreads load"; hashing a key to pick a shard — you've built this before.
- **New here — the guarantee as a number.** RPO stated as `replication_lag × write_rate` (records at risk), not "some risk of data loss on failover"; the exact mechanism (sloppy quorums + hinted handoff) that voids W+R>N's read-your-writes guarantee, and why that's a deliberate availability trade, not a bug.
- **New here — the production partitioning default.** A fixed large partition count mapped to nodes as metadata (Kafka, Elasticsearch, Uber Schemaless) versus the textbook `hash mod N`, with the exact data-movement cost each pays on rebalance.
- **New here — keeping two failure models apart.** The metadata-plane sharding failure model (graceful, partial availability) stays sharply distinct from the training-plane parameter-sharding failure model (gang, all-or-nothing) — the module's own three-planes distinction, applied concretely to this topic.

## Core concepts

**Replication spectrum — the guarantee/cost curve.**
- **Sync**: commit waits for replica ack. Guarantee = zero data loss on single-node failure (RPO=0). Cost = commit tail is the *slowest* required replica's tail; one gray-failing/GC-pausing replica stalls all commits. Mitigate with a quorum so you don't wait for the straggler.
- **Async**: commit returns before replica ack. Cost = **RPO > 0** — failover loses the un-shipped tail (replication lag window). Formalize it: `RPO = replication_lag × write_rate` — literally the number of records at risk if you fail over right now. A 200 ms replication lag at 5,000 writes/s is 1,000 records at risk on every failover, not an abstract "some loss."
- **Quorum (W+R > N)**: the tunable middle. Pick W and R to trade write vs read availability against N; W+R>N guarantees a read quorum overlaps the last write quorum → read-your-writes *if* nothing weakens it.
- **What weakens W+R>N**: **sloppy quorums + hinted handoff** (Dynamo) accept writes on *substitute* nodes during a partition to preserve availability. The W nodes that ack may not be in the R set that later reads → the overlap guarantee no longer holds; you trade consistency for availability explicitly. Read-repair + anti-entropy (Merkle trees) close the gap eventually, not immediately.
- **Chain replication**: order replicas in a chain, writes flow head→tail, reads served by tail. Gives strong consistency with throughput near async (each node does one send) — the throughput-optimized alternative to majority quorum, at the cost of higher write latency (full chain traversal) and a reconfiguration cost on any node failure.

**Partitioning strategies — and the one production actually uses.**
- **Range**: good scans/ordered queries; **hot-spot prone** — the *monotonic-key trap* (timestamp/auto-increment key sends all writes to the tail partition).
- **Consistent hashing**: even spread, kills range scans; virtual nodes smooth the ring.
- **The production default — fixed large partition count mapped to nodes** (Elasticsearch shards, Kafka partitions, Citus): choose e.g. 1024 partitions up front, map partitions→nodes as *metadata*. Rebalancing moves whole partitions (a metadata change), never rehashes keys. **Never `hash mod N`** — changing node count reshuffles ~all keys.

| Scheme | Rebalance cost | Scan support | Hot-key resilience | Production example |
|---|---|---|---|---|
| Range partitioning | Low if splits are targeted, but monotonic-key workloads concentrate rebalances on the tail partition | Excellent — ordered scans are native | Poor by default — the monotonic-key trap; needs an explicit reverse/hash prefix | HBase, CockroachDB ranges |
| Consistent hashing (ring + vnodes) | Moderate — join/leave moves ~1/N of the ring's data via vnode reassignment | None — hashing destroys order | Moderate — spreads *keys*, not skewed *load*; still needs salting for celebrity keys | Cassandra, Riak, DynamoDB (internal) |
| Fixed partition count → metadata map | Low — only the partition→node map changes; key→partition is stable | Depends — can layer range within fixed partitions | Same as hashing for spreading; still needs salting for skew | Kafka, Elasticsearch, Citus, Uber Schemaless |

**Hot shards / celebrity problem.** Skew is the default, not the exception — a Zipfian tenant/key distribution means one shard runs hot regardless of a "uniform" hash, because the *input* isn't uniform. Mitigations: **key salting/splitting** (append a bucket to the hot key: `tenant#0..k`), **request coalescing**, **dedicated shards** for known whales, **shuffle sharding** for blast-radius isolation.

**Rebalancing.** Must move *minimal* data (fixed-partition schemes do this by construction) and **throttle** — an un-throttled rebalance during scale-out is a self-inflicted overload: the copy traffic competes with live traffic and you brown out the thing you were trying to relieve.

**GPU tie — keep these two distinct:**
1. *Real*: sharding the scheduler/metadata store (jobs, quotas, node inventory). Range vs hash choice; hot shards when one tenant floods submissions. This is a genuine partial-availability system.
2. *Analogy (labeled as such)*: parameter sharding in distributed training — data/tensor/pipeline parallelism + ZeRO stages shard optimizer/gradient/parameter state across ranks; all-reduce is a "quorum-like" all-to-all. Pedagogically useful, but the **failure model differs**: training is all-or-nothing gang (one rank dies → step dies → restart from checkpoint), not the graceful partial-availability of a replicated store. Don't reason about training resilience with database-replication intuitions.

## Perspectives

**The data-modeler view.** The partition key should come from the dominant access pattern, not the entity-relationship diagram. Is the query a point lookup by ID (favors hash or fixed-partition), a range/time-ordered scan (favors range partitioning), or a per-tenant scan (favors co-locating a tenant's rows — which then makes that tenant a hot-shard risk)? Picking a key because "that's the primary key on the ER diagram" is how a scheme ends up clean on paper and hot in production.

**The operator/rebalancer view.** The distinction that matters operationally is *metadata-map* (fixed partition count, rebalance = reassign partition→node pointers) versus *rehash* (`hash mod N`, rebalance = recompute almost every key's location). Live migration in the metadata-map world is: dual-write or copy-then-cutover a partition, verify, flip the map entry, retire the old copy — all throttled against a live-traffic SLI. In the rehash world there's no cheap partial move; you're forced into either downtime or a much more elaborate live-rehash scheme, which is exactly why production systems avoid it.

**The multi-tenancy/skew view.** Real traffic follows a Zipfian power law, not a uniform distribution — a handful of tenants or keys carry a disproportionate share of load no matter how well the hash spreads keys. The toolkit is salting/splitting a hot key, dedicated shards for known whales, and shuffle sharding (each customer gets a pseudo-random *subset* of shards, so one customer's overload can only ever collide with a fraction of others) to bound blast radius even when you can't eliminate skew.

**The failure-model view.** Replication and partitioning give **graceful partial availability**: lose one shard or one replica and you degrade a slice of the keyspace, not the whole system. Contrast this explicitly with GPU training's **all-or-nothing gang-failure model**: one rank dies, the whole step dies, and recovery means restarting from the last checkpoint — there is no "partition still serving while another is down." Conflating the two is a real staff-level mistake: it leads to under-provisioning training-job resilience because "distributed systems degrade gracefully" doesn't hold there.

## Real-world use cases

*(These URLs are the canonical, live links to the named posts. This session's web-fetch tool was blocked by the environment's network egress proxy, so they were not independently re-fetched this pass — flagged per the sourcing rules rather than silently cited as verified.)*

- **Discord — "How Discord Stores Trillions of Messages"** (https://discord.com/blog/how-discord-stores-trillions-of-messages; mirror: https://blog.bytebytego.com/p/how-discord-stores-trillions-of-messages) — a real three-generation partitioning story: MongoDB → Cassandra (12 nodes in 2017, growing to 177 nodes by 2022) → ScyllaDB with a shard-per-core architecture, adopted specifically to fix coordination bottlenecks at that node count. Lead with this one — it's the clearest "partitioning scheme changes as scale and hot-shard pain grow" case available.
- **Uber Engineering — "The Architecture of Schemaless, Uber Engineering's Trip Datastore Using MySQL"** (https://www.uber.com/en-BG/blog/schemaless-part-two-architecture/) — matches this lesson's "production default" almost exactly: a fixed shard count (4096) mapped to nodes via metadata, never `hash mod N`.
- **Amazon, via Werner Vogels — "Amazon's Highly Available Key-value Store" (Dynamo)** (https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html) — the origin of sloppy quorums + hinted handoff, the exact mechanism this lesson names as what weakens W+R>N.
- **Vitess/YouTube — "Cloud Native MySQL Sharding with Vitess and Kubernetes"** (https://vitess.io/blog/2015-10-06-cloud-native-mysql-sharding-with-vitess-and-kubernetes/) — YouTube's live, online resharding of MySQL (splitting one shard into many with a zero-downtime cutover) — a real "throttled rebalance without a live-SLI brownout" case study, the production version of this lesson's own worked example.

## Worked example

Shard a metadata store: **500M job records, 50K writes/s, one tenant = 30% of traffic.**

*Naive scheme — hash(tenant_id) into 256 partitions:* all of one tenant's rows land on one partition. That tenant = 30% × 50K = **15K writes/s on a single shard**, while the average shard carries 50K/256 ≈ 195 writes/s. The hot shard is ~**77× the mean** — it saturates first and throttles that tenant (and any co-located tenant) while 255 shards idle. Hashing didn't help because the *input* is skewed; hashing preserves the skew.

*Fix — composite-key salting for whales:* key = `hash(tenant_id, salt)` where `salt ∈ 0..S-1` for flagged large tenants (S=32 for the 30% tenant), `salt=0` otherwise. The whale's 15K/s now spreads across 32 sub-partitions ≈ **470 writes/s each** — within ~2.4× of the mean, tolerable. Reads for that tenant fan out to 32 partitions (scatter-gather cost — the price of the split), so salt only tenants that need it.

*Layout:* 1024 fixed partitions → N nodes via a metadata map, not `mod N`. Scaling N=8→10 moves 1024×(1/8−1/10)≈**205 partitions** (~20% of data), not a full rehash.

*Throttled rebalance:* cap copy bandwidth at, say, 20% of per-node write-path capacity; move partitions serially/small-batch; watch p99 on live traffic and back off if it climbs. Target: rebalance completes in hours without the live SLI moving — a slow correct rebalance beats a fast brownout.

## Practice

*Feeds [staff design portfolio](../practice/staff-design-portfolio/README.md).*

Write a 2-page design for the GPU-fleet **scheduler metadata store**: (a) state your N, W, R and the exact RPO/tail-latency you're buying; (b) pick a partition scheme and justify range-vs-fixed-hash against the store's dominant query (point lookup by job-id vs scan by tenant+time); (c) identify the hot-shard scenario for *your* workload and give the salting/dedicated-shard plan with per-shard numbers; (d) a throttled rebalance runbook with the SLI you watch and the abort condition. Include one paragraph explicitly contrasting this system's partial-availability failure model with training's gang/all-or-nothing model — to prove you won't conflate them.

## Common pitfalls

1. **"A uniform hash function solves hot shards."** Hashing spreads *keys*, not *load* — a Zipfian/celebrity input distribution puts 30% of traffic on 30% of shards no matter the hash (Discord's story and this lesson's own worked example). Fix the input skew (salting, dedicated shards), not the hash function.
2. **`hash mod N` is "just sharding."** It couples key→shard to node count, so any scale-out reshuffles nearly all data. Production systems (Uber Schemaless, Elasticsearch, Kafka) instead fix the partition count and remap only partition→node.
3. **"Sync replication is safer, full stop."** It's safer for durability, but the commit tail is bound by the *slowest required* replica — a single gray-failing or GC-pausing replica stalls every write unless you quorum instead of requiring all.
4. **"Sloppy quorums are a bug/edge case."** They're a deliberate design choice (Dynamo) trading away W+R>N's read-your-writes guarantee for availability during a partition — a documented, intentional trade, not a defect to be "fixed."
5. **"Training's parameter/tensor sharding and database sharding are the same idea."** Database sharding is partial-availability; training sharding (ZeRO, tensor/pipeline parallelism) is gang, all-or-nothing. Don't reason about training resilience with replication-store intuition — this is the module's own explicitly flagged distinction, and it's worth keeping prominent.

## Self-check

- Why can W+R>N still return a stale read in practice? **Answer:** Sloppy quorums with hinted handoff accept writes on substitute nodes during a partition, so the W nodes that acked may not intersect the R nodes later read — the overlap guarantee that W+R>N relies on is voided until anti-entropy/read-repair converges. It's a deliberate availability-for-consistency trade, not a bug.
- You have a uniform hash and still get a hot shard. Why, and what's the fix? **Answer:** A uniform hash spreads *keys* evenly but preserves *load skew* in the input (Zipfian tenants/celebrity keys) — 30% of traffic on one key is 30% on one shard no matter the hash. Fix is to split that key with salting (`key#0..S-1`) or give it a dedicated shard, accepting scatter-gather read cost for the split keys only.
- Why do Elasticsearch/Kafka use a fixed large partition count instead of `hash mod N`? **Answer:** With `mod N`, changing node count remaps almost every key → a full data reshuffle. A fixed partition count decouples key→partition (stable) from partition→node (a metadata map), so scaling moves only whole partitions (~1/old−1/new of data) as cheap metadata changes, and rebalancing never rehashes.
- Async replication's RPO is quoted as "5 seconds." What does that actually bound, and what number turns it into records at risk? **Answer:** "5 seconds" bounds the *replication lag* — how far behind the replica can fall before a failover. It is not itself a record count. `RPO = replication_lag × write_rate` converts it: at 5s lag and 5,000 writes/s, up to 25,000 records can be lost on failover. The lag number alone is meaningless without the write-rate multiplier.

## Connections & what's next

This lesson generalizes lesson 02's quorum mechanics: "majority" stops being a fixed consensus requirement and becomes a tunable dial (W, R, sync/async/chain) traded against durability, availability, and latency. It also introduces the module's recurring failure-model warning — graceful partial availability versus training's gang failure — that resurfaces whenever a design conflates a data-store shard with a training rank. Forward, lesson 04 shows that a cache is a specialized replica optimized for latency rather than durability, reusing the replication instincts built here with a different objective function; and a hot shard that salting and dedicated shards don't fully absorb doesn't just run slow — it creates the queueing pressure and shed/admit decisions that lesson 05 formalizes.

## References & further reading

**Primary sources**
- DeCandia, G. et al. (2007), *Dynamo: Amazon's Highly Available Key-value Store*, SOSP '07 — https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf (read for: the original sloppy-quorum/hinted-handoff/vector-clock design).
- Elhemali, M. et al. (2022), *Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service*, USENIX ATC '22 — https://www.usenix.org/system/files/atc22-elhemali.pdf (read for: how a production quorum store evolved past the original Dynamo design at massive scale).
- Karger, D. et al. (1997), *Consistent Hashing and Random Trees*, STOC '97 — DOI https://doi.org/10.1145/258533.258660 (read for: the consistent-hashing construction underlying ring-based partitioning).
- van Renesse, R. & Schneider, F. (2004), *Chain Replication for Supporting High Throughput and Availability*, OSDI '04 — https://www.cs.cornell.edu/home/rvr/papers/OSDI04.pdf (read for: the chain-replication protocol and its throughput/latency tradeoff vs quorum replication).

**Real-world engineering blogs**
- Discord — *How Discord Stores Trillions of Messages* — https://discord.com/blog/how-discord-stores-trillions-of-messages — what it shows: a three-generation partitioning-scheme evolution driven by hot-shard and coordination pain.
- Uber Engineering — *The Architecture of Schemaless* — https://www.uber.com/en-BG/blog/schemaless-part-two-architecture/ — what it shows: the fixed-partition-count-plus-metadata-map pattern in production.
- Amazon, via Werner Vogels — *Amazon's Highly Available Key-value Store* — https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html — what it shows: sloppy quorums and hinted handoff explained by the system that introduced them.
- Vitess/YouTube — *Cloud Native MySQL Sharding with Vitess and Kubernetes* — https://vitess.io/blog/2015-10-06-cloud-native-mysql-sharding-with-vitess-and-kubernetes/ — what it shows: live, throttled, zero-downtime resharding at YouTube's scale.
- AWS Builders' Library — *Workload isolation using shuffle-sharding* — https://aws.amazon.com/builders-library/workload-isolation-using-shuffle-sharding/ — what it shows: shuffle sharding as a production blast-radius-containment pattern for multi-tenant skew.

**Deeper dives**
- Kleppmann, M., *Designing Data-Intensive Applications*, Ch. 5–6 — https://dataintensive.com/ — the fuller treatment of replication and partitioning this lesson compresses.
