# Depth map — platform/01 · Distributed systems & system design

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **The `databases/` track is the hidden match for this module.** It is 24 chapters, 1,000–3,600
> lines each, and it is where the *precise* version of every guarantee in these lessons lives —
> isolation levels, quorum arithmetic, WAL durability, failure detectors. This module teaches you
> to **bound** a system; those chapters are where the bounds are stated exactly.

| Lesson | Go deeper in | Why |
|---|---|---|
| 01 Consistency models | [`databases/05-transactions-and-concurrency`](https://github.com/harut8/system-design/blob/main/databases/05-transactions-and-concurrency.md) | 2,600 lines: isolation levels, the anomalies each permits, and **serializable vs linearizable** stated precisely rather than gestured at |
| 01 Consistency models | [`databases/18-concurrency-control-and-scheduling`](https://github.com/harut8/system-design/blob/main/databases/18-concurrency-control-and-scheduling.md) | schedules, conflict serializability, OCC vs MVCC vs 2PL — the theory the models are defined over |
| 02 Consensus & quorums | [`databases/16-failure-detection-and-leader-election`](https://github.com/harut8/system-design/blob/main/databases/16-failure-detection-and-leader-election.md) | phi-accrual detectors, timeouts, split-brain and fencing — why leader election is hard *before* Raft |
| 02 Consensus & quorums | [`kubernetes/04-etcd-internals`](https://github.com/harut8/system-design/blob/main/kubernetes/04-etcd-internals.md) | the named real-system placement: Raft in production, MVCC revisions, and the fsync bound |
| 02 Consensus & quorums | [`databases/14-write-ahead-log-internals`](https://github.com/harut8/system-design/blob/main/databases/14-write-ahead-log-internals.md) | **why** etcd is fsync-bound — group commit, durability vs latency, the actual mechanism |
| 03 Replication & partitioning | [`databases/12-replication-and-distributed-storage`](https://github.com/harut8/system-design/blob/main/databases/12-replication-and-distributed-storage.md) | 2,800 lines: sync/async/quorum replication, W+R>N, rebalancing strategies, hot shards |
| 03 Replication & partitioning | [`databases/19-distributed-databases-deep-dive`](https://github.com/harut8/system-design/blob/main/databases/19-distributed-databases-deep-dive.md) | Spanner/Cockroach/Yugabyte-class systems — where the theory got built |
| 04 Caching | [`databases/10-in-memory-databases`](https://github.com/harut8/system-design/blob/main/databases/10-in-memory-databases.md) | eviction, persistence and the memory-bound design space behind "caches are modal systems" |
| 04 Caching | [`databases/00-os-and-hardware-internals`](https://github.com/harut8/system-design/blob/main/databases/00-os-and-hardware-internals.md) | the page cache — the cache you already have and usually forget to reason about |
| 05 Queueing & backpressure | [`databases/17-latches-and-locks-internals`](https://github.com/harut8/system-design/blob/main/databases/17-latches-and-locks-internals.md) | 3,600 lines on contention: where queueing shows up *inside* a single process, and why convoying is a metastable failure in miniature |
| 06 Failure & resilience | [`databases/16-failure-detection-and-leader-election`](https://github.com/harut8/system-design/blob/main/databases/16-failure-detection-and-leader-election.md) | the fail-stop vs fail-slow distinction, made rigorous — the core of the gray-failure argument |
| 07 Data-intensive patterns | [`databases/13-lsm-trees-and-compaction`](https://github.com/harut8/system-design/blob/main/databases/13-lsm-trees-and-compaction.md) | write/read/space amplification as a tradeoff triangle you can quote numbers from |
| 07 Data-intensive patterns | [`databases/08-olap-databases`](https://github.com/harut8/system-design/blob/main/databases/08-olap-databases.md) · [`09-htap-databases`](https://github.com/harut8/system-design/blob/main/databases/09-htap-databases.md) | the OLTP/OLAP split, which is the same two-path argument as `platform/03` L10 in a different domain |
| 08 System-design method | [`SYSTEM-DESIGN-GUIDE.md`](https://github.com/harut8/system-design/blob/main/SYSTEM-DESIGN-GUIDE.md) | a compact 6-step framework with a back-of-envelope section and a common-mistakes list — compare against your own method and keep whichever step ordering you can actually execute under pressure |
| 09 Design rehearsal | [`solutions/`](https://github.com/harut8/system-design/tree/main/solutions) · [`implementation/`](https://github.com/harut8/system-design/tree/main/implementation) | four worked design docs, each implemented at four scale tiers — the raw material for the [design drills](../practice/staff-design-portfolio/design-drills.md) |

## What this seeded

The [design-drill ladder](../practice/staff-design-portfolio/design-drills.md) — designing the same
system at one-rack / one-fleet / multi-region scale with a "what flips, and at what number" table.
That method comes straight from how `implementation/` is organised, re-aimed at GPU-platform prompts.

## Read with care

[`distributed-systems/README.md`](https://github.com/harut8/system-design/blob/main/distributed-systems/README.md)
looks like the obvious match by name, but it is a **roadmap only** — a reading map with no chapters
behind it. Useful as a book/paper list; not a substitute for the `databases/` track, which is where
the actual depth ended up.
