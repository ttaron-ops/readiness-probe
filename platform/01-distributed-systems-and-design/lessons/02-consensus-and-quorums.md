---
lesson: "A01.2"
title: "Consensus and quorums"
module: "A-01"
concept: "raft-quorum-fsync"
status: not-started
est_time: "3 hrs"
artifacts: ["etcd-sizing-worksheet", "write-latency-budget"]
---

# A01.2 · Consensus and quorums

> **Concept.** Consensus commit latency is `leader fsync(WAL) + 1 RTT to the slowest majority + follower fsync`; etcd is disk-bound, and a slow disk — not a network partition — is the usual cause of "scheduling is mysteriously slow across the whole fleet."
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Why this matters

A senior can stand up a 3-node etcd cluster and explain that writes need a majority. A staff engineer gets paged because API-server p99 latency tripled and scheduling stalled fleet-wide, and has to say — within minutes — "this is `wal_fsync` p99 blowing past 10 ms because another tenant is hammering the same disk; etcd is missing heartbeats and churning elections." That diagnosis requires knowing the *exact* commit-latency formula, which metric names to watch, and why even-numbered clusters and cross-AZ links inflate the quorum RTT without buying fault tolerance. Consensus is where "I can build it" becomes "I can bound it and tell you the one number that decides."

## Core notes

**Skip (you already know):** what a leader is; that Raft has leader election plus log replication; that a write needs a majority.

**Raft internals worth naming.**

- **Terms + commitIndex.** Each election bumps the *term*; `commitIndex` advances only when an entry is acknowledged by a majority *in the leader's current term*. The "two leaders" illusion is resolved by term ordering — a write from an old-term leader can never commit because followers reject its lower term.
- **Read paths without a log append.** **ReadIndex**: leader records its current commit index, confirms it is still leader via a heartbeat round, then serves once applied — a linearizable read with no disk write. **Leader lease**: leader serves reads locally within a time-bounded lease, trading a clock assumption for zero round trips. Both avoid appending a no-op to the log per read.
- **CheckQuorum + PreVote.** PreVote makes a candidate confirm it *could* win before incrementing its term, so a flapping/partitioned node can't inflate its term and disrupt a healthy leader on rejoin. CheckQuorum makes a leader step down if it can't reach a majority. Together they kill spurious elections from transient partitions.
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

**Split-brain and failure modes.** A minority partition **cannot elect a leader** (safe — no split-brain writes) but also **cannot serve linearizable reads or writes** (unavailable — this is the **PC** in PACELC). Danger zones: even-node clusters; **cross-AZ latency inflating the quorum RTT** (every commit pays the slowest-majority link); and the "two leaders in different terms" mirage, which is not real split-brain because term ordering prevents the stale leader from committing.

**Paxos contrast — the staff war story.** Multi-Paxos solves the same problem; Google's *"Paxos Made Live"* is the canonical "we implemented the paper and it broke in production" account — the paper omits snapshotting, disk corruption handling, master leases, and membership churn, all of which dominate real operational cost. The lesson: consensus correctness is the easy 20%; durability, leases, and reconfiguration are the 80% that pages you.

**GPU frame — control plane.** etcd *is* the control plane under CoreWeave / Lambda-style GPU fleets. Operational rules that follow directly from the formula: put etcd on **dedicated NVMe**, **isolate its disk** from noisy neighbors, size at **3 or 5** (never even, rarely 7 — the extra RTT isn't worth it), and internalize that a slow etcd disk manifests as *fleet-wide scheduling latency*, not as an obvious storage alert.

## Worked example

**Setup.** Size an etcd cluster for a GPU-fleet control plane. Cross-AZ link RTT = 2 ms. Measured `wal_fsync` p99 = 8 ms.

**Q1 — 3 vs 5 vs 7?**

- **3:** tolerates 1 failure, majority = 2 (leader + 1 follower). Quorum RTT = the one link to the nearest follower.
- **5:** tolerates 2 failures, majority = 3 (leader + 2 followers). Quorum RTT = the link to the **2nd-slowest** follower — so every commit now waits on the slower of two AZ hops, adding ~2 ms of tail versus 3-node. Extra durability cost too: more nodes fsync per proposal.
- **7:** tolerates 3, majority = 4. Rarely justified — the added quorum breadth keeps inflating commit latency for fault tolerance you almost never need.

**Decision:** default **3** for a control plane where fast recovery/replacement is available; **5** only if you genuinely need to survive 2 simultaneous node losses (e.g., 2-AZ-loss tolerance) and can absorb the extra ~2 ms commit tail. The one deciding number is *how many simultaneous failures you must survive*, not raw node count.

**Q2 — Write-latency budget.**

```
commit ≈ leader_fsync + RTT_slowest_majority + follower_fsync
       ≈ 8 ms         + 2 ms                 + 8 ms
       ≈ 18 ms p99 per write
```

Healthy: `wal_fsync` p99 (8 ms) is under the 10 ms target, so ~18 ms commits are expected. Now inject a **40 ms fsync spike** (noisy neighbor): the leader can't fsync its append or send heartbeats in time, followers hit their election timeout (default ~1 s, but the heartbeat interval is ~100 ms so misses accumulate fast), miss heartbeats, and **trigger an election**. Each election blocks writes until a new leader stabilizes → API-server write latency spikes → scheduling stalls fleet-wide. A single-disk pathology, invisible as "storage," surfaces as a control-plane-wide outage. That is the entire argument for dedicated, isolated NVMe under etcd.

## Practice

*Feeds [staff design portfolio](../practice/staff-design-portfolio/README.md).*

1. **etcd sizing worksheet.** For your real fleet, justify 3 vs 5 in writing: state the failure count you must survive, your cross-AZ RTT, and the commit-latency delta 5 adds over 3. Make the deciding number explicit.
2. **Write-latency budget.** Pull `etcd_disk_wal_fsync_duration_seconds` and `etcd_disk_backend_commit_duration_seconds` p99 from your cluster (or estimate), compute expected commit latency from the formula, and set alert thresholds at the 10 ms / 25 ms lines with a runbook entry linking "fsync spike → election churn → fleet scheduling stall."
3. Write a half-page "Paxos Made Live"-style retro on one consensus system you operate: which of snapshotting / disk corruption / leases / membership churn actually caused incidents versus the core protocol.

## Self-check

- Your etcd `wal_fsync` p99 jumps to 40 ms and the whole fleet's scheduling slows. Trace the causal chain and name the likely root cause. **Answer:** Slow WAL fsync → leader can't durably append/ack proposals or send heartbeats in time → followers miss heartbeats and hit election timeout → leader churn/re-election blocks writes → API-server write latency spikes → scheduling stalls fleet-wide. Likely root cause: disk I/O contention, classically a **noisy neighbor** stealing IOPS from a shared disk; the fix is dedicated, isolated NVMe for etcd. It is a disk problem masquerading as a control-plane outage.
- Why does going from a 3-node to a 5-node etcd cluster increase write latency, and what fault-tolerance do you get for it? **Answer:** Commit requires a majority fsync + ack. At 3 the majority is 2 (leader + nearest follower); at 5 it is 3 (leader + 2 followers), so every commit now waits on the *2nd-slowest* follower link and more nodes fsync per proposal — adding tail latency (≈ another cross-AZ RTT). In return you tolerate 2 simultaneous failures instead of 1. You pay latency for surviving one more concurrent node loss.
- A partitioned minority etcd node keeps timing out and trying to become leader. Which two Raft mechanisms prevent it from disrupting the healthy majority, and how? **Answer:** **PreVote** — the node must confirm it could actually win an election before incrementing its term, so it can't inflate its term while partitioned and force the real leader to step down on rejoin. **CheckQuorum** — a leader that can't reach a majority steps down on its own, preventing a stale leader from serving. Together they suppress spurious elections and term inflation from a flapping minority.

## References

- Ongaro & Ousterhout — In Search of an Understandable Consensus Algorithm (Raft): https://raft.github.io/raft.pdf
- etcd — Tuning guide (disk, heartbeat, election timeouts, fsync metrics): https://etcd.io/docs/v3.5/tuning/
- Jepsen — etcd 3.4.3 analysis: https://jepsen.io/analyses/etcd-3.4.3
- Chandra, Griesemer & Redstone — Paxos Made Live: https://static.googleusercontent.com/media/research.google.com/en//archive/paxos_made_live.pdf
- etcd — Operations / hardware recommendations: https://etcd.io/docs/v3.5/op-guide/hardware/
