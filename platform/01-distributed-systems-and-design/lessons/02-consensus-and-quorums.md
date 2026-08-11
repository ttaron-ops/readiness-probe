---
lesson: "A01.2"
title: "Consensus and quorums"
module: "A-01"
concept: "raft-quorum-fsync"
status: not-started
est_time: "4 hrs"
prev: "01-consistency-models.md"
next: "03-replication-and-partitioning.md"
artifacts: ["etcd-sizing-worksheet", "write-latency-budget"]
sources: 10
---

# A01.2 · Consensus and quorums

> **Concept.** Consensus commit latency is `leader fsync(WAL) + 1 RTT to the slowest majority + follower fsync`; etcd is disk-bound, and a slow disk — not a network partition — is the usual cause of "scheduling is mysteriously slow across the whole fleet."
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 01 established the vocabulary: linearizability as a real-time-recency guarantee, PACELC's "Else" as the everyday latency-vs-consistency tax, and the specific claim that **etcd is linearizable by default while the K8s watch cache sitting on top of it is only eventually consistent**. That lesson stated the guarantee; it didn't explain the machine that produces it. This lesson goes inside etcd's own write path — Raft — to show *why* etcd can make that linearizable claim and, more importantly, *what it costs in disk I/O and network round trips every single time a write commits*. Where Lesson 01 gave you the vocabulary to name a guarantee, this lesson gives you the formula to price it.

## Why this matters

A senior can stand up a 3-node etcd cluster and explain that writes need a majority. A staff engineer gets paged because API-server p99 latency tripled and scheduling stalled fleet-wide, and has to say — within minutes — "this is `wal_fsync` p99 blowing past 10 ms because another tenant is hammering the same disk; etcd is missing heartbeats and churning elections." That diagnosis requires knowing the *exact* commit-latency formula, which metric names to watch, and why even-numbered clusters and cross-AZ links inflate the quorum RTT without buying fault tolerance. Consensus is where "I can build it" becomes "I can bound it and tell you the one number that decides."

## What's new here (calibration)

**Skip (you already know this):**
- What a leader is, and that Raft has leader election plus log replication.
- That a write needs a majority to commit.

**The genuinely new depth this lesson adds:**
- The exact commit-latency formula (`leader fsync + RTT to slowest majority + follower fsync`) and the two Prometheus metric names to watch — turning "etcd feels slow" into a number you can alert on.
- The internal machinery beyond "majority wins": `ReadIndex`/leader-lease reads, `PreVote`/`CheckQuorum`, and the learner-then-promote membership-change pattern, each tied to a concrete failure mode it prevents.
- Real production numbers — from OpenAI's GPU-fleet etcd tuning and Roblox's Consul/BoltDB outage — showing the disk-bound argument isn't theoretical, it has taken down hyperscale fleets.

## Core concepts

**Raft internals worth naming.**

- **Terms + commitIndex.** Each election bumps the *term*; `commitIndex` advances only when an entry is acknowledged by a majority *in the leader's current term*. The "two leaders" illusion is resolved by term ordering — a write from an old-term leader can never commit because followers reject its lower term.
- **Read paths without a log append.** **ReadIndex**: leader records its current commit index, confirms it is still leader via a heartbeat round, then serves once applied — a linearizable read with no disk write. **Leader lease**: leader serves reads locally within a time-bounded lease, trading a clock assumption for zero round trips. Both avoid appending a no-op to the log per read, and both are still fully **linearizable** — they optimize away the log append, not the guarantee.
- **CheckQuorum + PreVote.** PreVote makes a candidate confirm it *could* win before incrementing its term, so a flapping/partitioned node can't inflate its term and disrupt a healthy leader on rejoin. CheckQuorum makes a leader step down if it can't reach a majority. Together they kill spurious elections from transient partitions.
- **Two safety pillars beyond "majority wins."** Majority acknowledgment alone isn't sufficient for correctness — Raft leans on two more mechanisms: the **Log Matching Property** (if two logs contain an entry with the same index and term, every entry before it is identical too — so logs never silently diverge and merge back wrong), and the **election restriction** (a candidate can only win if its log is at least as up-to-date as a majority of voters, which guarantees every committed entry survives into every future leader's log). These two rules, not just "count the votes," are what make Raft's safety proof hold.
- **Membership changes.** etcd adds a **learner** (non-voting, catches up the log) *before* promoting it to voter, and does **single-server** add/remove rather than arbitrary joint-consensus jumps — so a reconfiguration never transiently loses quorum. Adding a fresh voter directly to a 3-node cluster briefly makes it a 4-node cluster whose new member has an empty log, enlarging the majority the cluster must reach while the newcomer is useless; the learner step avoids that window.

**Quorum math.** An `N`-node cluster tolerates `⌊(N−1)/2⌋` failures:

| N | Majority | Failures tolerated |
| --- | --- | --- |
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

**Even sizes buy nothing.** 4 tolerates the same 1 failure as 3 but needs a larger majority (3 acks vs 2) → more latency, more disk fsyncs, no extra fault tolerance. Always odd.

**Commit latency formula.** A write is acked only after it is durable on a majority:

```
commit_latency ≈ leader_fsync(WAL) + RTT_to_slowest_majority_node + follower_fsync
```

Every proposal must `fsync` the WAL on the leader **and** on enough followers to form a majority before the client sees success. That is why **etcd is disk-bound, not CPU-bound.**

**Why etcd hates slow disks — the metrics.** Watch:

- `etcd_disk_wal_fsync_duration_seconds` — p99 should be **< 10 ms**.
- `etcd_disk_backend_commit_duration_seconds` — p99 should be **< 25 ms**.

Causal chain when the disk degrades: slow `fsync` → the leader can't ack proposals or send heartbeats in time → followers time out and start an election → leader churn → API-server writes stall → **scheduling stalls across the entire fleet.** A **noisy-neighbor** stealing disk IOPS is the classic trigger, and the symptom ("scheduling is slow everywhere") points nowhere near the disk unless you know this chain.

**Heartbeat-to-election-timeout ratio.** Raft implementations, etcd included, run heartbeats far more often than the election timeout to avoid spurious elections from a single missed beat — a common rule of thumb is roughly **1:10** (e.g., a 100 ms heartbeat interval against a ~1 s election timeout). Too tight a ratio (heartbeats close to the timeout) causes elections on ordinary jitter; too loose wastes failover time. This ratio is a tuning knob, not a constant — but 1:10 is the number to start from and the one to sanity-check in any etcd config review.

**Split-brain and failure modes.** A minority partition **cannot elect a leader** (safe — no split-brain writes) but also **cannot serve linearizable reads or writes** (unavailable — this is the **PC** in PACELC, the same PACELC introduced in Lesson 01). Danger zones: even-node clusters; **cross-AZ latency inflating the quorum RTT** (every commit pays the slowest-majority link); and the "two leaders in different terms" mirage, which is not real split-brain because term ordering prevents the stale leader from committing.

**Paxos contrast — the staff war story.** Multi-Paxos solves the same problem; Google's *"Paxos Made Live"* is the canonical "we implemented the paper and it broke in production" account — the paper omits snapshotting, disk corruption handling, master leases, and membership churn, all of which dominate real operational cost. The lesson: consensus correctness is the easy 20%; durability, leases, and reconfiguration are the 80% that pages you. Raft was explicitly designed as a reaction to this: Ongaro & Ousterhout optimized for *understandability* over Paxos's minimal message complexity, on the theory that an algorithm engineers can actually reason about in production is worth more than one that's marginally more elegant on paper.

**GPU frame — control plane.** etcd *is* the control plane under CoreWeave / Lambda-style GPU fleets. Operational rules that follow directly from the formula: put etcd on **dedicated NVMe**, **isolate its disk** from noisy neighbors, size at **3 or 5** (never even, rarely 7 — the extra RTT isn't worth it), and internalize that a slow etcd disk manifests as *fleet-wide scheduling latency*, not as an obvious storage alert.

## Perspectives

**The algorithm-designer view.** Raft deliberately trades a small amount of theoretical elegance for understandability compared to Paxos. A strong leader (all writes flow through one node, simplifying reasoning about ordering), the Log Matching Property, and term-based election restrictions are Raft's answer to "how do we get a protocol engineers can actually implement correctly." Paxos is arguably more general (Multi-Paxos allows more flexible leaderless variants), but Ongaro & Ousterhout's stated design goal — and the reason Raft displaced Paxos in nearly every new system built after 2014 (etcd, Consul, CockroachDB) — was that understandable-and-correct beats minimal-and-error-prone in production.

**The operator / on-call view.** The metrics that matter are narrow and specific: `wal_fsync` p99 and `backend_commit` p99. The runbook chain an on-call engineer needs memorized is: disk latency spike → missed heartbeats → election churn → write stalls → fleet-wide scheduling delay. Without that chain pre-loaded, "scheduling is slow everywhere" reads as a networking or scheduler-code problem, and the actual root cause (a noisy neighbor on the etcd disk) goes uninvestigated for hours.

**The hardware / disk view.** Why NVMe vs. network-attached storage is the whole ballgame: etcd's bottleneck is **sync-write latency**, not raw throughput. A cloud block-storage volume can advertise thousands of IOPS and still be unusable for etcd if each `fsync` round-trips over the network — throughput numbers on a spec sheet say nothing about fsync latency. "Fast SSD" marketing claims lie by omission unless you specifically check sync-write IOPS/latency, which is exactly the number that determines etcd's commit floor.

**The cost / latency view.** 3 vs. 5 vs. 7 nodes is a pure purchase: each additional pair of nodes buys tolerance for one more simultaneous failure, priced in milliseconds of added commit-tail latency per write (because the majority now must include a slower node) plus more fsyncs per proposal. Framing cluster sizing as "how many milliseconds am I willing to pay per write, for how many extra simultaneous failures I can survive" turns a vibes-based "5 feels safer" decision into an explicit tradeoff a staff engineer can defend in a design review.

## Real-world use cases

- **OpenAI — "Scaling Kubernetes to 2,500 nodes"**: https://openai.com/index/scaling-kubernetes-to-2500-nodes/ — OpenAI found etcd write latency spiking to hundreds of milliseconds even on a P30 SSD rated for 5,000 IOPS; benchmarking showed etcd was only using ~10% of available IOPS because it does sequential, latency-bound (not throughput-bound) sync writes. Moving the etcd data directory to **local (not network-attached) SSD** cut write latency to ~200µs. Direct, named confirmation of this lesson's core thesis — etcd is disk-bound, and *which kind* of disk matters more than its rated IOPS. (2023 dated snapshot.)
- **OpenAI — "Scaling Kubernetes to 7,500 nodes"**: https://openai.com/index/scaling-kubernetes-to-7500-nodes/ — the follow-up: etcd and API servers moved to dedicated nodes, Kubernetes **Events split into a separate etcd cluster** to isolate write-heavy, low-value traffic from scheduling-critical objects; their largest clusters run **5 etcd + 5 API servers**. A real, production quorum-sizing and workload-isolation decision at GPU-fleet scale. (2023 dated snapshot.)
- **Roblox — "Roblox Return to Service 10/28–10/31/2021"** (73-hour outage postmortem): https://blog.roblox.com/2022/01/roblox-return-to-service-10-28-10-31-2021/ — root cause: a new Consul streaming feature under high load triggered a pathological performance bug in **BoltDB**, the storage engine backing Consul's Raft WAL, and outdated freelist handling meant deleted log space never shrank on disk. The added twist: the monitoring stack itself depended on the broken Consul cluster, extending the outage. A hyperscale example of exactly the consensus-storage-layer failure mode this lesson teaches. (2021/2022 dated postmortem.)
- **etcd.io — "Latest Jepsen Results against etcd 3.4.3"**: https://etcd.io/blog/2020/jepsen-343-results/ — the etcd team's own response to the Jepsen findings (cross-referenced from Lesson 01): confirms strict-serializable KV operations held under fault injection, and documents the fix path for the lock-service issue Jepsen found. Good primary-adjacent companion to the raw Jepsen analysis.

## Worked example

**Setup.** Size an etcd cluster for a GPU-fleet control plane. Cross-AZ link RTT = 2 ms. Measured `wal_fsync` p99 = 8 ms.

**Q1 — 3 vs 5 vs 7?**

- **3:** tolerates 1 failure, majority = 2 (leader + 1 follower). Quorum RTT = the one link to the nearest follower.
- **5:** tolerates 2 failures, majority = 3 (leader + 2 followers). Quorum RTT = the link to the **2nd-slowest** follower — so every commit now waits on the slower of two AZ hops, adding ~2 ms of tail versus 3-node. Extra durability cost too: more nodes fsync per proposal.
- **7:** tolerates 3, majority = 4. Rarely justified — the added quorum breadth keeps inflating commit latency for fault tolerance you almost never need.

**Decision:** default **3** for a control plane where fast recovery/replacement is available; **5** only if you genuinely need to survive 2 simultaneous node losses (e.g., 2-AZ-loss tolerance) and can absorb the extra ~2 ms commit tail — this is exactly OpenAI's real-world choice at their largest cluster sizes (see Real-world use cases). The one deciding number is *how many simultaneous failures you must survive*, not raw node count.

**Q2 — Write-latency budget.**

```
commit ≈ leader_fsync + RTT_slowest_majority + follower_fsync
       ≈ 8 ms         + 2 ms                 + 8 ms
       ≈ 18 ms p99 per write
```

Healthy: `wal_fsync` p99 (8 ms) is under the 10 ms target, so ~18 ms commits are expected. Now inject a **40 ms fsync spike** (noisy neighbor): the leader can't fsync its append or send heartbeats in time, followers hit their election timeout (default ~1 s, but the heartbeat interval is ~100 ms so misses accumulate fast — recall the ~1:10 heartbeat:timeout ratio), miss heartbeats, and **trigger an election**. Each election blocks writes until a new leader stabilizes → API-server write latency spikes → scheduling stalls fleet-wide. A single-disk pathology, invisible as "storage," surfaces as a control-plane-wide outage — the same shape of failure that took Roblox down for 73 hours. That is the entire argument for dedicated, isolated NVMe under etcd.

## Practice

*Feeds [staff design portfolio](../practice/staff-design-portfolio/README.md).*

1. **etcd sizing worksheet.** For your real fleet, justify 3 vs 5 in writing: state the failure count you must survive, your cross-AZ RTT, and the commit-latency delta 5 adds over 3. Make the deciding number explicit.
2. **Write-latency budget.** Pull `etcd_disk_wal_fsync_duration_seconds` and `etcd_disk_backend_commit_duration_seconds` p99 from your cluster (or estimate), compute expected commit latency from the formula, and set alert thresholds at the 10 ms / 25 ms lines with a runbook entry linking "fsync spike → election churn → fleet scheduling stall."
3. Write a half-page "Paxos Made Live"-style retro on one consensus system you operate: which of snapshotting / disk corruption / leases / membership churn actually caused incidents versus the core protocol.

## Common pitfalls

1. **"Odd node count = more fault tolerance, so always go bigger."** 4 nodes buys nothing over 3 (same 1-failure tolerance, worse majority math — 3 acks instead of 2). Always odd, and size for how many *simultaneous* failures you must actually survive, not for a bigger-feels-safer instinct.
2. **"Two leaders" in the metrics means split-brain.** Term ordering makes the old leader's writes uncommittable — a stale leader that will step down or get rejected, not a concurrent-writer corruption event.
3. **"Network-attached / cloud block storage is fine for etcd because it's 'SSD'."** What matters is sync-write (fsync) latency and IOPS, not marketed throughput. This is literally what broke Roblox's Consul cluster and what OpenAI fixed by moving etcd to local SSD.
4. **"ReadIndex / leader-lease reads are 'eventually consistent, so cheaper but weaker'."** ReadIndex *is* linearizable — it avoids a log append (and thus a disk fsync) on the read path, not the consistency guarantee itself. Don't confuse "no disk write" with "weaker guarantee."
5. **"Adding a new voter to a 3-node cluster directly is a safe, simple scale-up."** A fresh voter with an empty log transiently enlarges the majority the cluster must reach while contributing zero useful acknowledgments — a real availability dip. The learner-then-promote pattern (etcd's default) avoids this window.

## Self-check

- Your etcd `wal_fsync` p99 jumps to 40 ms and the whole fleet's scheduling slows. Trace the causal chain and name the likely root cause. **Answer:** Slow WAL fsync → leader can't durably append/ack proposals or send heartbeats in time → followers miss heartbeats and hit election timeout → leader churn/re-election blocks writes → API-server write latency spikes → scheduling stalls fleet-wide. Likely root cause: disk I/O contention, classically a **noisy neighbor** stealing IOPS from a shared disk; the fix is dedicated, isolated NVMe for etcd. It is a disk problem masquerading as a control-plane outage.
- Why does going from a 3-node to a 5-node etcd cluster increase write latency, and what fault-tolerance do you get for it? **Answer:** Commit requires a majority fsync + ack. At 3 the majority is 2 (leader + nearest follower); at 5 it is 3 (leader + 2 followers), so every commit now waits on the *2nd-slowest* follower link and more nodes fsync per proposal — adding tail latency (≈ another cross-AZ RTT). In return you tolerate 2 simultaneous failures instead of 1. You pay latency for surviving one more concurrent node loss.
- A partitioned minority etcd node keeps timing out and trying to become leader. Which two Raft mechanisms prevent it from disrupting the healthy majority, and how? **Answer:** **PreVote** — the node must confirm it could actually win an election before incrementing its term, so it can't inflate its term while partitioned and force the real leader to step down on rejoin. **CheckQuorum** — a leader that can't reach a majority steps down on its own, preventing a stale leader from serving. Together they suppress spurious elections and term inflation from a flapping minority.
- Is a `ReadIndex` read linearizable or only eventually consistent, and why do people conflate the two? **Answer:** Linearizable. `ReadIndex` confirms the leader is still leader (via a heartbeat round) and waits for the recorded commit index to be applied before serving — that satisfies the same real-time recency guarantee as a full quorum round trip. People conflate it with "eventual" because it skips a log append and disk fsync, which reads like a shortcut on consistency; in fact it's a shortcut on *mechanism* (no WAL write needed for a read), not on the guarantee.

## Connections & what's next

This lesson cashes in Lesson 01's claim that etcd is linearizable: `ReadIndex` and the majority-commit protocol shown here are the actual mechanism behind that guarantee, and the commit-latency formula is the concrete price of PACELC's "Else" tax named there. Forward, **Lesson 03** (replication and partitioning) builds on the same majority-quorum mechanics — the W+R>N overlap argument is Dynamo-style replication's answer to the same "how many nodes must agree" question Raft answers here, just with weaker guarantees and lower cost. **Lesson 06** (failure and resilience) revisits the failure-detection assumptions underneath PreVote/CheckQuorum — how a system decides "is that node actually dead" is the same question gray-failure detection has to answer at a larger scale. Carry forward: *if consensus's cost is dominated by disk fsync, what does replication's cost curve look like when the bottleneck shifts to network fan-out instead?*

## References & further reading

**Primary sources**
- Ongaro, D. & Ousterhout, J. — *In Search of an Understandable Consensus Algorithm* (Raft): https://raft.github.io/raft.pdf — read for the Log Matching Property, election restriction, and the understandability-over-Paxos design rationale.
- etcd — *Tuning guide* (disk, heartbeat, election timeouts, fsync metrics): https://etcd.io/docs/v3.5/tuning/ — read for the official `wal_fsync`/`backend_commit` thresholds and heartbeat-interval tuning guidance used in Core concepts.
- etcd — *Operations / hardware recommendations*: https://etcd.io/docs/v3.5/op-guide/hardware/ — read for the official disk-isolation and NVMe guidance behind the GPU-frame recommendations.
- Burrows, M. (2006). *The Chubby Lock Service for Loosely-Coupled Distributed Systems*, OSDI '06. https://research.google/pubs/the-chubby-lock-service-for-loosely-coupled-distributed-systems/ — read as a historical/operational contrast to Raft: a Paxos-based lock service built for availability over raw performance, and the operational lessons Google drew from running it at scale.

**Real-world engineering blogs**
- OpenAI — *Scaling Kubernetes to 2,500 nodes*: https://openai.com/index/scaling-kubernetes-to-2500-nodes/ — what it shows: etcd write latency is fsync/latency-bound, not throughput-bound; local SSD cut write latency 40x.
- OpenAI — *Scaling Kubernetes to 7,500 nodes*: https://openai.com/index/scaling-kubernetes-to-7500-nodes/ — what it shows: real production quorum sizing (5 etcd + 5 API servers) and workload isolation (Events in a separate etcd cluster).
- Roblox — *Roblox Return to Service 10/28-10/31/2021*: https://blog.roblox.com/2022/01/roblox-return-to-service-10-28-10-31-2021/ — what it shows: a consensus-storage-layer disk pathology (BoltDB freelist) causing a 73-hour, hyperscale outage.
- etcd.io — *Latest Jepsen Results against etcd 3.4.3*: https://etcd.io/blog/2020/jepsen-343-results/ — what it shows: the etcd team's own accounting of what Jepsen's fault injection confirmed and what it broke.
- Jepsen — *etcd 3.4.3 analysis*: https://jepsen.io/analyses/etcd-3.4.3 — what it shows: independent fault-injection verification of etcd's strict-serializable claim (and the lock-service exception).

**Deeper dives**
- Chandra, T., Griesemer, R. & Redstone, J. — *Paxos Made Live*: https://static.googleusercontent.com/media/research.google.com/en//archive/paxos_made_live.pdf — the "we implemented the paper and it broke in production" war story behind this lesson's 80/20 argument.
