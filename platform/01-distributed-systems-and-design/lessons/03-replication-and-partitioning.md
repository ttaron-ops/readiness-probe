---
lesson: "A01.3"
title: "Replication and partitioning"
module: "A-01"
concept: "replication spectrum, partition schemes, hot shards"
status: not-started
est_time: "3 hrs"
artifacts: ["metadata-store shard design + rebalance plan"]
---

# A01.3 · Replication and partitioning

> **Concept.** Replication buys durability and read scale at a tail-latency and RPO cost; partitioning buys write scale but makes skew the default failure mode.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Why this matters

At staff level the interesting question is never "should we replicate/shard" — it's *which point on the guarantee/cost curve*, stated as a number, and *what breaks when the assumption behind that number breaks*. Every replication choice is a bet on the correlation of failures and the acceptable RPO; every partition choice is a bet on key distribution that skew will call. The metadata plane of a GPU fleet (the etcd/K8s-adjacent store holding jobs, quotas, node inventory) is where these bets get made, and where a bad one shows up as a single hot shard throttling the whole control plane during a submission storm.

## Core notes

**Skip (you already know):** leader/follower mechanics; "sharding spreads load"; hashing a key to pick a shard.

**Replication spectrum — the guarantee/cost curve.**
- **Sync**: commit waits for replica ack. Guarantee = zero data loss on single-node failure (RPO=0). Cost = commit tail is the *slowest* required replica's tail; one gray-failing/GC-pausing replica stalls all commits. Mitigate with a quorum so you don't wait for the straggler.
- **Async**: commit returns before replica ack. Cost = **RPO > 0** — failover loses the un-shipped tail (replication lag window). The deciding number is *acceptable data loss in seconds* × *write rate* = records at risk.
- **Quorum (W+R > N)**: the tunable middle. Pick W and R to trade write vs read availability against N; W+R>N guarantees a read quorum overlaps the last write quorum → read-your-writes *if* nothing weakens it.
- **What weakens W+R>N**: **sloppy quorums + hinted handoff** (Dynamo) accept writes on *substitute* nodes during a partition to preserve availability. The W nodes that ack may not be in the R set that later reads → the overlap guarantee no longer holds; you trade consistency for availability explicitly. Read-repair + anti-entropy (Merkle trees) close the gap eventually, not immediately.
- **Chain replication**: order replicas in a chain, writes flow head→tail, reads served by tail. Gives strong consistency with throughput near async (each node does one send) — the throughput-optimized alternative to majority quorum, at the cost of higher write latency (full chain traversal) and a reconfiguration cost on any node failure.

**Partitioning strategies — and the one production actually uses.**
- **Range**: good scans/ordered queries; **hot-spot prone** — the *monotonic-key trap* (timestamp/auto-increment key sends all writes to the tail partition).
- **Consistent hashing**: even spread, kills range scans; virtual nodes smooth the ring.
- **The production default — fixed large partition count mapped to nodes** (Elasticsearch shards, Kafka partitions, Citus): choose e.g. 1024 partitions up front, map partitions→nodes as *metadata*. Rebalancing moves whole partitions (a metadata change), never rehashes keys. **Never `hash mod N`** — changing node count reshuffles ~all keys.

**Hot shards / celebrity problem.** Skew is the default, not the exception — a Zipfian tenant/key distribution means one shard runs hot regardless of a "uniform" hash, because the *input* isn't uniform. Mitigations: **key salting/splitting** (append a bucket to the hot key: `tenant#0..k`), **request coalescing**, **dedicated shards** for known whales, **shuffle sharding** for blast-radius isolation.

**Rebalancing.** Must move *minimal* data (fixed-partition schemes do this by construction) and **throttle** — an un-throttled rebalance during scale-out is a self-inflicted overload: the copy traffic competes with live traffic and you brown out the thing you were trying to relieve.

**GPU tie — keep these two distinct:**
1. *Real*: sharding the scheduler/metadata store (jobs, quotas, node inventory). Range vs hash choice; hot shards when one tenant floods submissions. This is a genuine partial-availability system.
2. *Analogy (labeled as such)*: parameter sharding in distributed training — data/tensor/pipeline parallelism + ZeRO stages shard optimizer/gradient/parameter state across ranks; all-reduce is a "quorum-like" all-to-all. Pedagogically useful, but the **failure model differs**: training is all-or-nothing gang (one rank dies → step dies → restart from checkpoint), not the graceful partial-availability of a replicated store. Don't reason about training resilience with database-replication intuitions.

## Worked example

Shard a metadata store: **500M job records, 50K writes/s, one tenant = 30% of traffic.**

*Naive scheme — hash(tenant_id) into 256 partitions:* all of one tenant's rows land on one partition. That tenant = 30% × 50K = **15K writes/s on a single shard**, while the average shard carries 50K/256 ≈ 195 writes/s. The hot shard is ~**77× the mean** — it saturates first and throttles that tenant (and any co-located tenant) while 255 shards idle. Hashing didn't help because the *input* is skewed; hashing preserves the skew.

*Fix — composite-key salting for whales:* key = `hash(tenant_id, salt)` where `salt ∈ 0..S-1` for flagged large tenants (S=32 for the 30% tenant), `salt=0` otherwise. The whale's 15K/s now spreads across 32 sub-partitions ≈ **470 writes/s each** — within ~2.4× of the mean, tolerable. Reads for that tenant fan out to 32 partitions (scatter-gather cost — the price of the split), so salt only tenants that need it.

*Layout:* 1024 fixed partitions → N nodes via a metadata map, not `mod N`. Scaling N=8→10 moves 1024×(1/8−1/10)≈**205 partitions** (~20% of data), not a full rehash.

*Throttled rebalance:* cap copy bandwidth at, say, 20% of per-node write-path capacity; move partitions serially/small-batch; watch p99 on live traffic and back off if it climbs. Target: rebalance completes in hours without the live SLI moving — a slow correct rebalance beats a fast brownout.

## Practice

*Feeds [staff design portfolio](../practice/staff-design-portfolio/README.md).*

Write a 2-page design for the GPU-fleet **scheduler metadata store**: (a) state your N, W, R and the exact RPO/tail-latency you're buying; (b) pick a partition scheme and justify range-vs-fixed-hash against the store's dominant query (point lookup by job-id vs scan by tenant+time); (c) identify the hot-shard scenario for *your* workload and give the salting/dedicated-shard plan with per-shard numbers; (d) a throttled rebalance runbook with the SLI you watch and the abort condition. Include one paragraph explicitly contrasting this system's partial-availability failure model with training's gang/all-or-nothing model — to prove you won't conflate them.

## Self-check

- Why can W+R>N still return a stale read in practice? **Answer:** Sloppy quorums with hinted handoff accept writes on substitute nodes during a partition, so the W nodes that acked may not intersect the R nodes later read — the overlap guarantee that W+R>N relies on is voided until anti-entropy/read-repair converges. It's a deliberate availability-for-consistency trade, not a bug.
- You have a uniform hash and still get a hot shard. Why, and what's the fix? **Answer:** A uniform hash spreads *keys* evenly but preserves *load skew* in the input (Zipfian tenants/celebrity keys) — 30% of traffic on one key is 30% on one shard no matter the hash. Fix is to split that key with salting (`key#0..S-1`) or give it a dedicated shard, accepting scatter-gather read cost for the split keys only.
- Why do Elasticsearch/Kafka use a fixed large partition count instead of `hash mod N`? **Answer:** With `mod N`, changing node count remaps almost every key → a full data reshuffle. A fixed partition count decouples key→partition (stable) from partition→node (a metadata map), so scaling moves only whole partitions (~1/old−1/new of data) as cheap metadata changes, and rebalancing never rehashes.

## References

- https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf
- https://aws.amazon.com/builders-library/workload-isolation-using-shuffle-sharding/
- https://dataintensive.com/ (DDIA Ch. 5–6)
- https://www.cs.cornell.edu/home/rvr/papers/OSDI04.pdf (Chain Replication)
