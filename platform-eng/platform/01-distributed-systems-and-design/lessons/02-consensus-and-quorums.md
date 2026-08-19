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
sources: 16
---

# A01.2 · Consensus and quorums

> **Concept.** Consensus commit latency is `leader fsync(WAL) + 1 RTT to the slowest majority + follower fsync`; etcd is disk-bound, and a slow disk — not a network partition — is the usual cause of "scheduling is mysteriously slow across the whole fleet."
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 01 established the vocabulary and made one specific claim: etcd is linearizable, while the watch cache above it is not, and the correctness of the whole control plane rests on where writes get serialised rather than on how fresh reads are. That lesson named the guarantee. This lesson opens the machine that produces it.

You will come out able to do three things a senior engineer usually cannot. Write the commit-latency formula from first principles and defend each term. Read an etcd Raft trace — actual message types, actual fields — and say what the cluster is doing. And answer "3 or 5 nodes?" with arithmetic instead of instinct. Lesson 01's "amortise the proof, not the payload" trick reappears here in its original form: Raft's `ReadIndex`, which is where the Kubernetes API server's consistent-read-from-cache design got the idea.

## Why this matters

A senior can stand up a 3-node etcd cluster and explain that writes need a majority. A staff engineer gets paged because API-server p99 tripled and scheduling stalled fleet-wide, and has to say — within minutes — "`etcd_disk_wal_fsync_duration_seconds` p99 is at 800 ms on member 2, we're past the mixin's critical threshold, the leader is logging `leader failed to send out heartbeat on time`, and this is a disk-contention problem, not a network one."

That diagnosis requires knowing the exact commit path, the metric names, the shipped alert thresholds, and the causal chain from a stalled `fdatasync` to a fleet-wide scheduling delay. It also requires knowing what does *not* follow — a 40 ms fsync spike does not cause an election, and saying it does will cost you credibility with anyone who has actually read the tick configuration. Consensus is where "I can build it" becomes "I can bound it and tell you the one number that decides."

## What's new here (calibration)

**Skip (you already know this):**
- What a leader is, and that Raft has leader election plus log replication.
- That a write needs a majority to commit.

**The genuinely new depth this lesson adds:**
- The actual protocol: the per-server state, the RPC field lists, and the exact message types etcd puts on the wire — enough that a Raft trace becomes readable rather than mysterious.
- The two safety rules that do the real work beyond "count the votes": the election restriction (with the precise up-to-date comparison) and the rule that a leader may only commit by counting replicas of an entry **from its own term** — with the five-server scenario that proves why.
- `ReadIndex` in its actual implementation, including the gate that makes a freshly-elected leader defer reads, and why it is the same mechanism as Kubernetes' consistent-read-from-cache one layer up.
- `PreVote` and `CheckQuorum` as concrete code paths with concrete failure modes, not as acronyms.
- Real defaults and real alert thresholds from etcd's own configuration and its shipped Prometheus mixin — including a correction to the numbers commonly quoted for fsync health.
- Availability arithmetic: what a 5-member cluster actually buys you in probability terms, and why 3 members across 2 availability zones is a trap that 3 members across 3 AZs is not.

Version note: all etcd internals, defaults, metric names and alert thresholds below were read directly from the `etcd-io/etcd`, `etcd-io/raft` and `etcd-io/website` repositories on GitHub in August 2026 (etcd.io itself is blocked by this environment's egress proxy). Raft protocol claims come from the extended Raft paper PDF in the `raft/raft.github.io` repository, cross-checked against the etcd implementation. Where a figure could not be read (the paper's Figure 2 is rendered as an image), that is stated in References.

## Core concepts

### The problem: turning a log into a state machine everyone agrees on

Consensus exists to solve one problem: **get a set of machines to agree on a single, totally ordered sequence of commands, such that every machine applies the same commands in the same order and therefore ends up in the same state.** That is the replicated state machine model. etcd's state machine is an MVCC key-value store; the commands are puts, deletes and transactions; the ordered sequence is the Raft log.

Two things make this hard, and knowing which is which tells you where to look when it breaks:

- **Safety must never depend on timing.** Messages may be delayed, duplicated, or reordered; nodes may crash and restart with stale state; clocks may drift. None of that may produce a wrong answer. Raft states this outright: safety must not depend on timing.
- **Liveness must depend on timing, because it cannot not.** FLP (Fischer, Lynch, Paterson, 1985) proves that in a fully asynchronous system with even one crash fault, no deterministic protocol can guarantee both safety and termination. Every real consensus system therefore buys liveness with a timing assumption — a timeout — and randomises it to break symmetry. Raft's randomised election timeout *is* the FLP escape hatch.

Hold on to that split. When etcd is unavailable, it is almost always a liveness problem (timeouts, elections, a stalled disk). When someone claims etcd lost data, they are claiming a safety violation, which is a far stronger and far rarer claim.

### Per-server state and the wire protocol

Raft's basic algorithm needs only two RPCs, plus a third for snapshots. This is the state and message layout, from the paper's Figure 2, with the field names cross-checked against etcd's implementation:

**State on every server**

| Field | Persistent? | Meaning |
|---|---|---|
| `currentTerm` | **yes, fsynced** | Latest term the server has seen; monotonically increasing |
| `votedFor` | **yes, fsynced** | Candidate that received this server's vote in `currentTerm`, or null |
| `log[]` | **yes, fsynced** | Entries, each `{term, index, command}` |
| `commitIndex` | no | Highest index known committed |
| `lastApplied` | no | Highest index applied to the state machine |
| `nextIndex[peer]` | no (leader only) | Next index the leader will send this follower |
| `matchIndex[peer]` | no (leader only) | Highest index known replicated on this follower |

The three persistent fields are why consensus is disk-bound. A vote that is not durable can be cast twice across a crash, which breaks Election Safety. An entry acknowledged but not durable can vanish on restart, which breaks Leader Completeness. This is not an implementation choice; it is the correctness requirement, and it is the reason `fdatasync` sits on the critical path of every write. etcd's implementation is explicit about it, deferring the send of any "voting" response — `MsgAppResp`, `MsgVoteResp`, `MsgPreVoteResp` — until the corresponding state is durably on local disk, citing Raft thesis §3.8.

**AppendEntries** — replicate entries; also the heartbeat when `entries[]` is empty.

| Argument | Meaning |
|---|---|
| `term` | Leader's term |
| `leaderId` | So followers can redirect clients |
| `prevLogIndex` | Index of the entry immediately preceding the new ones |
| `prevLogTerm` | Term of that entry — the consistency check |
| `entries[]` | Entries to store (empty = heartbeat) |
| `leaderCommit` | Leader's `commitIndex` |

Returns `{term, success}`. A follower returns `success=false` if `term < currentTerm`, or if its log has no entry at `prevLogIndex` with `prevLogTerm`.

**RequestVote** — solicit a vote.

| Argument | Meaning |
|---|---|
| `term` | Candidate's term |
| `candidateId` | Who is asking |
| `lastLogIndex` | Index of the candidate's last entry |
| `lastLogTerm` | Term of the candidate's last entry |

Returns `{term, voteGranted}`. The voter grants only if it has not voted in this term **and** the candidate's log is at least as up to date as its own.

**InstallSnapshot** — for a follower so far behind that the leader has already compacted the entries it needs: `{term, leaderId, lastIncludedIndex, lastIncludedTerm, offset, data[], done}`.

In etcd this is all one flat message type, which is what you see in traces and tests. From `raftpb/raft.proto`:

```
message Message {
  MessageType type      //  MsgHup MsgBeat MsgProp MsgApp MsgAppResp MsgVote MsgVoteResp
                        //  MsgSnap MsgHeartbeat MsgHeartbeatResp MsgUnreachable MsgSnapStatus
                        //  MsgCheckQuorum MsgTransferLeader MsgTimeoutNow MsgReadIndex
                        //  MsgReadIndexResp MsgPreVote MsgPreVoteResp MsgStorageAppend
                        //  MsgStorageAppendResp MsgStorageApply MsgStorageApplyResp
                        //  MsgForgetLeader
  uint64  to, from, term
  uint64  logTerm       // = prevLogTerm on MsgApp
  uint64  index         // = prevLogIndex on MsgApp
  Entry   entries[]     // each {Term, Index, Type, Data}
  uint64  commit        // = leaderCommit
  uint64  vote
  Snapshot snapshot     // populated only for MsgSnap
  bool    reject        // = !success
  uint64  rejectHint    // follower's hint for how far back to rewind nextIndex
  bytes   context       // carries the ReadIndex request id / heartbeat context
}
```

Two of those fields are worth pausing on because they are pure engineering, not protocol:

- **`rejectHint`.** The paper's basic recovery is "on rejection, decrement `nextIndex` and retry" — one round trip per conflicting entry. The optimisation, which etcd implements, is for the follower to return the term of the conflicting entry and the first index it holds for that term, so the leader can skip an entire term per round trip. The paper explicitly doubts this optimisation is necessary; at Kubernetes log volumes it is.
- **`context`.** The correlation token for read-only requests: it is what lets a heartbeat round acknowledge a specific batch of pending reads (see `ReadIndex` below).

### Leader election, and what actually goes wrong

```
 3-member cluster {A,B,C}. Time flows down. Boxes are disk syncs.

  A (leader, term 5)            B (follower)            C (follower)
     │                              │                       │
     │──MsgHeartbeat(term=5)───────▶│                       │
     │──MsgHeartbeat(term=5)────────────────────────────────▶│
     │◀─MsgHeartbeatResp────────────│                       │
     │◀─MsgHeartbeatResp────────────────────────────────────│
     │        (every heartbeat-interval = 100 ms default)
     │
     ✗ A dies (or its disk stalls for longer than election-timeout)
     │
     ·  B and C each count down a RANDOMISED election timeout.
     ·  Randomisation is the whole trick: if both timed out together every
     ·  time, they would split the vote forever. Paper's example range is
     ·  150–300 ms; etcd derives ticks from election-timeout=1000 ms default.
     │
     │                              │ timeout first
     │                              │ [fsync currentTerm=6, votedFor=B]
     │                              │──MsgVote(term=6,
     │                              │    index=lastLogIndex,
     │                              │    logTerm=lastLogTerm)──────────▶│
     │                              │                       │ is B's log at least
     │                              │                       │ as up-to-date as mine?
     │                              │                       │ [fsync votedFor=B]
     │                              │◀────MsgVoteResp(granted)─────────│
     │                              │ has 2 of 3 (itself + C) = majority
     │                              │ BECOMES LEADER term 6
     │                              │ appends an empty entry for term 6
     │                              │──MsgApp(term=6, entries=[noop])──▶│
```

The two knobs and their real constraints:

| Setting | etcd default | Constraint | Source |
|---|---|---|---|
| `--heartbeat-interval` | 100 ms | Set to ~0.5–1.5× the peer RTT | etcd tuning docs |
| `--election-timeout` | 1000 ms | Must be ≥ 10× RTT; etcd rejects configs where `5 × heartbeat > election`; hard maximum 50,000 ms | etcd tuning docs; `server/embed/config.go` |
| Raft-library ratio | — | `ElectionTick` must exceed `HeartbeatTick`; library suggests **10×** | `etcd-io/raft` `Config` |

The paper's version of the same constraint is the inequality worth memorising, because it generalises to any failure-detector design:

```
   broadcastTime  ≪  electionTimeout  ≪  MTBF
   (0.5–20 ms,        (10–500 ms,          (months per server)
    dominated by       chosen by you)
    a disk sync)
```

Left inequality violated → spurious elections, because a leader cannot reliably get heartbeats out in time. Right inequality violated → the cluster spends its life electing. **This is why "just lower the election timeout to fail over faster" is a trap:** you are pushing `electionTimeout` toward `broadcastTime`, and `broadcastTime` includes a disk sync whose tail you do not control.

### The two safety rules that do the real work

Majority counting alone is not sufficient. Raft's five guaranteed properties are Election Safety, Leader Append-Only, Log Matching, Leader Completeness, and State Machine Safety. Two mechanisms carry most of the weight.

**1. The election restriction.** A candidate wins only if its log is at least as up to date as the voter's. "Up to date" is defined precisely, and etcd implements it in four lines:

```go
// etcd-io/raft, log.go
func (l *raftLog) isUpToDate(their entryID) bool {
    our := l.lastEntryID()
    return their.term > our.term || their.term == our.term && their.index >= our.index
}
```

Higher last term wins; same last term, longer log wins. Because a candidate must reach a majority, and any committed entry is on a majority, the two majorities intersect — so a candidate that could not have that entry cannot win. **This is why Raft never needs to ship missing entries *to* a new leader: log entries only ever flow leader → follower.**

**2. A leader may not commit an old-term entry by counting replicas.** This is the subtle one and the classic interview probe. The paper's Figure 8 scenario, redrawn:

```
 Five servers. Each column is one server's log; the number in a box is the ENTRY'S TERM.
 Index:        1   2   3

 (a) S1 leader in term 2, partially replicates index 2:
     S1 [1][2]        S2 [1][2]      S3 [1]      S4 [1]      S5 [1]

 (b) S1 crashes. S5 wins term 3 with votes from S3, S4 and itself
     (their logs end at index 1 term 1, so S5's log is "up to date" enough),
     and accepts a DIFFERENT entry at index 2:
     S1 (down)        S2 [1][2]      S3 [1]      S4 [1]      S5 [1][3]

 (c) S5 crashes. S1 restarts, wins term 4, and keeps replicating the old
     term-2 entry. Index 2 now sits on S1,S2,S3 — a MAJORITY:
     S1 [1][2][4]     S2 [1][2]      S3 [1][2]   S4 [1]      S5 (down)
                          ^^^ on 3 of 5. Tempting to call this committed. DO NOT.

 (d) If S1 crashes here, S5 can still win term 5 (votes from S2,S3,S4 —
     its last entry is term 3, higher than their term 2) and OVERWRITE index 2:
     S5 [1][3][3][3]…    ← the "committed" entry is gone. Safety violated.

 (e) The fix: if S1 instead replicates an entry from ITS OWN TERM (term 4) to a
     majority BEFORE crashing, then index 2 is committed — and everything before
     it is committed too, because S5 can no longer win (its log is now behind).
```

The rule that falls out: **a leader marks an entry committed only when an entry from its own current term has been replicated to a majority.** Old entries commit implicitly, dragged along by a newer one. etcd enforces exactly this in `maybeCommit`:

```go
// etcd-io/raft, log.go — note the matchTerm() call: index alone is not enough
func (l *raftLog) maybeCommit(at entryID) bool {
    if at.term != 0 && at.index > l.committed && l.matchTerm(at) {
        l.commitTo(at.index)
        return true
    }
    return false
}
```

Two practical consequences you can observe:

- A newly elected leader appends an **empty (no-op) entry** for its term immediately. That is not bookkeeping; it is the mechanism that lets it commit everything inherited from previous terms and start serving.
- Therefore a leader cannot serve a linearizable read until that no-op commits — which is exactly the gate in etcd's read path (next section). **In an election, the write stall and the linearizable-read stall have the same cause and the same duration.**

### `ReadIndex`: linearizable reads without touching the disk

A naive linearizable read appends a read command to the log — correct, and absurdly expensive: a disk fsync on a majority to answer a question that changes nothing. `ReadIndex` gets the same guarantee for one network round trip and zero disk writes.

The leader must establish two things: (1) it is *still* the leader right now, so it is not serving from a superseded state; (2) it has applied everything committed as of the moment the read arrived.

```
 Client            etcd server (leader)                     Followers
   │  Range(key)
   │────────────▶ LinearizableReadNotify()
   │              │
   │              │  ① record readIndex := raftLog.committed   ← the "index" in ReadIndex
   │              │  ② GATE: has an entry from MY term already
   │              │     committed? If not, park this read in
   │              │     pendingReadIndexMessages and wait.
   │              │     (freshly-elected leader, no-op not yet committed)
   │              │  ③ broadcast heartbeats carrying `context`
   │              │────── MsgHeartbeat(ctx=readIndex) ──────────▶ B
   │              │────── MsgHeartbeat(ctx=readIndex) ──────────▶ C
   │              │◀───── MsgHeartbeatResp(ctx) ─────────────────  B
   │              │  ④ quorum of acks (leader self-acks) ⇒ confirmed
   │              │     still leader as of readIndex
   │              │  ⑤ wait until appliedIndex ≥ readIndex
   │              │     (ApplyWait — the state machine may lag the log)
   │◀─────────────│  ⑥ serve from the local MVCC store
   │   value

  Cost: 1 network RTT to a majority. ZERO disk writes. Fully LINEARIZABLE.
  Batching: one loop confirmation releases every read waiting at or below
  that index — so a burst of 1,000 concurrent reads costs one heartbeat round,
  not 1,000. If no response arrives, etcd retries the ReadIndex after 500 ms
  and logs "waiting for ReadIndex response took too long, retrying".
```

Every step above is in the source: `readOnly.addRequest(commitIndex, req)` records the index, `recvAck` collects heartbeat acknowledgements, `maybeAdvance` releases requests once the acks form a quorum, and `Read.LinearizableReadLoop` does the `AppliedIndex`/`ApplyWait` step before serving. The retry constant is `readIndexRetryTime = 500 * time.Millisecond`.

The alternative etcd also implements is **`ReadOnlyLeaseBased`**: the leader serves reads locally, without any round trip, as long as it is within a lease derived from the election timeout. That converts a network cost into a **clock assumption** — if this leader's clock runs slow relative to a partitioned follower's, it can serve stale data after a new leader has been elected. The library refuses to enable it without `CheckQuorum`, and etcd's default is the safe option. **Say that out loud in an interview: leases trade a round trip for a clock-drift assumption, and the assumption is not free.**

etcd exposes the third option to clients too: a **serializable** read (`clientv3.WithSerializable()`, `etcdctl --consistency=s`) is served by whichever member you asked, from its local state, with no quorum check at all — fast and possibly stale. etcd's published benchmark on a 3-member GCE cluster puts the difference at 0.7 ms vs 0.3 ms per request single-client, and 141,578 vs 185,758 QPS at 1,000 clients (etcd 3.2.0, 8 vCPU/16 GB/SSD — a dated snapshot, but the *ratio* is the durable lesson).

### `PreVote` and `CheckQuorum`

Two mechanisms that exist purely to stop a sick minority from disrupting a healthy majority.

**The disease without `PreVote`:** a node partitioned away from the cluster times out, increments its term, times out again, increments again — for ten minutes it climbs to term 4,000 while the healthy cluster sits at term 6. When the partition heals, its first `MsgVote(term=4000)` forces the legitimate leader to step down (Raft's universal rule: any message with a higher term makes you a follower), triggering an election the disrupting node cannot even win, because its log is behind. Pure, pointless unavailability.

**`PreVote`** inserts a dry run. The candidate sends `MsgPreVote` for `term+1` **without incrementing its own term**, and peers answer as they would for a real vote (including the up-to-date check) but — critically — **never update their term in response to a pre-vote**. Only when a majority says "yes, you would win" does the node increment its term and campaign for real. A partitioned node's pre-votes are refused by nobody it can reach, so its term never moves and healing costs the cluster nothing.

**`CheckQuorum`** is the mirror image, on the leader:

```go
// etcd-io/raft, raft.go — leader step, MsgCheckQuorum (fires each election timeout)
if !r.trk.QuorumActive() {
    r.logger.Warningf("%x stepped down to follower since quorum is not active", r.id)
    r.becomeFollower(r.Term, None)
}
// then mark every peer inactive again, so the next interval measures fresh contact
```

A leader that has not heard from a majority within an election timeout demotes itself. Without it, a leader isolated from its followers keeps believing it is leader — harmless for writes (they can never commit) but fatal for lease-based reads, which is precisely why the library requires `CheckQuorum` before it will allow `ReadOnlyLeaseBased`.

**And the thing they do *not* prevent.** Two nodes can genuinely believe they are leader at the same wall-clock moment, in different terms. That is not split brain, and it is not a bug. The old-term leader's `MsgApp` is rejected by every follower that has moved on, so it can never advance `commitIndex`; it will discover a higher term in the rejection and step down. **Split brain requires two leaders both able to commit, and quorum intersection makes that impossible — two majorities of the same set always share at least one member, and that member's persisted `votedFor` for the term stops the second election.**

### Membership changes without losing quorum

Switching directly from configuration C_old to C_new is unsafe: servers adopt the new configuration at different times, so there is an interval in which a majority of C_old and a majority of C_new can each elect a leader for the same term (paper, Figure 10). Raft offers two safe approaches, and etcd implements both plus a practice on top:

- **Joint consensus** — a transitional configuration in which decisions require majorities of *both* the old and new configurations. Visible in etcd's `ConfState`, which carries `voters`, `voters_outgoing`, `learners` and `learners_next`, and in `ConfChangeV2`'s transition modes (`Auto`, `JointImplicit`, `JointExplicit`).
- **Single-server changes** — add or remove one server at a time, which is safe because any two majorities of configurations differing by one member necessarily intersect.
- **Learners.** etcd adds a new member as a **learner** first: it receives the log but does not vote and does not count toward quorum. Once it has caught up, you promote it. Skipping this step is the documented footgun: adding a voter to a healthy 3-member cluster makes it a 4-member cluster whose quorum is now **3**, while the newcomer's log is empty and it contributes nothing — so you have gone from tolerating one failure to tolerating one failure *with a larger, slower majority*, and if the new member was misconfigured and never joins, you now have 2 up and 2 down, need 3 votes to change membership, and **cannot undo your own change**. etcd defends against this with `strict-reconfig-check` (on by default), which rejects reconfigurations that would cost quorum.

The operational rule from etcd's own FAQ: **when replacing a failed member, remove first, then add.** Removing takes 3 → 2 members with quorum still 2; adding takes it back to 3 with quorum still 2. Adding first takes quorum to 3 while you are already down a node.

### Quorum arithmetic, and what a fifth node actually buys

Quorum for `n` members is `(n/2)+1`; the cluster tolerates `⌊(n−1)/2⌋` failures:

| Cluster size | Majority | Failures tolerated |
|:-:|:-:|:-:|
| 1 | 1 | 0 |
| 2 | 2 | 0 |
| 3 | 2 | 1 |
| 4 | 3 | 1 |
| 5 | 3 | 2 |
| 6 | 4 | 2 |
| 7 | 4 | 3 |

Even sizes are strictly worse: same fault tolerance, larger majority, more machines that can fail, more fsyncs per proposal. etcd's guidance is odd sizes, and no more than seven members — beyond that, write performance degrades because every proposal must be replicated more widely for no fault-tolerance gain that anyone actually needs.

**Now do it in probabilities, because "tolerates 2 failures" is not a decision input.** Let `p` be the probability that any given member is unavailable at a random instant, failures independent. The cluster is down when a majority is down:

```
  3 members, quorum 2 → down when ≥2 are down
     P(down) = 3p²(1−p) + p³ ≈ 3p²         for small p

  5 members, quorum 3 → down when ≥3 are down
     P(down) = 10p³(1−p)² + 5p⁴(1−p) + p⁵ ≈ 10p³   for small p

  Worked, with p = 0.01 (each member unavailable ~7 hours/month):
     3 members: ≈ 3 × 10⁻⁴      → ~4.3 minutes of quorum loss per month
     5 members: ≈ 1 × 10⁻⁵      → ~8 seconds of quorum loss per month
     Improvement: ~30×.

  Now with CORRELATED failure — one rack, one power domain, one bad kernel
  rollout, one AZ: the independence assumption collapses and BOTH numbers
  become approximately P(that domain fails). Five members in one rack is a
  more expensive 1-member cluster.
```

That last line is the whole point. The fifth node buys ~30× against *independent* failures and approximately nothing against correlated ones. So the real sizing question is not "how many nodes" but **"how many independent failure domains, and does a majority survive losing the largest one?"**

```
  3 members across 2 AZs — the trap:
       AZ-A: [m1][m2]        AZ-B: [m3]
       lose AZ-A  → 1 of 3 alive, quorum 2 → DEAD
       lose AZ-B  → 2 of 3 alive           → alive
     You have bought AZ tolerance in exactly one of two directions.
     (This is the same reason a 2-member cluster tolerates 0 failures.)

  3 members across 3 AZs:
       AZ-A: [m1]   AZ-B: [m2]   AZ-C: [m3]
       lose any one AZ → 2 of 3 alive, quorum 2 → alive ✓
       cost: every commit crosses an AZ boundary (see the latency budget)

  5 members across 3 AZs (2 + 2 + 1):
       lose the 1-member AZ → 4 alive, quorum 3 → alive ✓
       lose a 2-member AZ   → 3 alive, quorum 3 → alive, ZERO further margin ✓
```

### Why it is disk-bound: the commit path with the clock running

```
 One write. Leader L, followers F1, F2. Time flows right. ▓▓ = fdatasync.

  client   ──put──▶ L
                    │
  L: append to WAL  ├─▓▓▓▓▓▓▓▓▓▓▓─┐  local fsync (must be durable before
                    │             │  L may count itself in the majority)
  L→F1: MsgApp      ├──────────╲  │
  L→F2: MsgApp      ├─────────╲ ╲ │   (sent in PARALLEL, and etcd may send
                    │          ╲ ╲│    them before its own fsync completes —
                    │           ╲ │    but a follower's ACK is what counts)
  F1: append+fsync  │      ┌─────▓▓▓▓▓▓▓▓▓─┐
  F1→L: MsgAppResp  │      │              ╱
  F2: append+fsync  │  ┌─────▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓┐   ← the slow one
  F2→L: MsgAppResp  │  │                        ╲
                    ▼  ▼                         ▼
  L: commitIndex advances as soon as a MAJORITY (L + F1) has it durably.
     F2 is NOT waited for. That is the whole value of a quorum.
                    │
  L: apply to MVCC  ├──┐   (backend commit — the boltdb write, batched)
  L→client: OK      ├──┘
                    ▼

  commit_latency ≈ leader_fsync(WAL)
                 + RTT to the SLOWEST member of the fastest majority
                 + that member's fsync(WAL)
                 + apply time
```

Every term in that formula is measurable, and etcd exports each one:

| Metric | What it measures | etcd mixin alert threshold (p99 over 5 m, for 10 m) |
|---|---|---|
| `etcd_disk_wal_fsync_duration_seconds` | The `fdatasync` on the Raft WAL | `> 0.5 s` warning, `> 1 s` **critical** |
| `etcd_disk_backend_commit_duration_seconds` | The boltdb backend commit | `> 0.25 s` warning |
| `etcd_network_peer_round_trip_time_seconds` | Peer RTT | `> 0.15 s` warning |
| `grpc_server_handling_seconds` (unary) | Client-visible request latency | `> 0.15 s` **critical** |
| `etcd_server_proposals_failed_total` | Proposals that never committed | `rate[15m] > 5` warning |
| `etcd_server_leader_changes_seen_total` | Elections | — (rate > 0 is the signal) |
| `etcd_server_heartbeat_send_failures_total` | Heartbeats sent later than 2× the interval | — |
| `etcd_server_slow_read_indexes_total` | `ReadIndex` rounds that had to retry | — |
| `etcd_mvcc_db_total_size_in_bytes` / `etcd_server_quota_backend_bytes` | Backend size vs quota | `> 95%` **critical** |

**A correction worth internalising.** The figures usually quoted in blog posts and in the previous version of this lesson — "`wal_fsync` p99 under 10 ms, `backend_commit` p99 under 25 ms" — are reasonable *health targets* for a well-provisioned cluster, and they are roughly what local NVMe delivers, but they are **not** etcd's shipped alert thresholds, which are 50× and 10× looser (0.5 s and 0.25 s). If you page on 10 ms you will page constantly; if you only page at etcd's defaults you will discover the problem at 500 ms, long after your API server p99 has moved. Set your SLO near the tight number, your page near the loose one, and know which is which. The related log lines to grep for, with their hard-coded thresholds in the source:

| Log message | Fires when | Constant |
|---|---|---|
| `slow fdatasync` | a single WAL fsync exceeds 1 s | `warnSyncDuration = time.Second` |
| `apply request took too long` | applying a request exceeds 100 ms | `DefaultWarningApplyDuration = 100 ms` (`--warning-apply-duration`) |
| `leader failed to send out heartbeat on time; took too long, leader is overloaded likely from slow disk` | a heartbeat send exceeds 2× `heartbeat-interval` | `2 * r.heartbeat` |
| `dropped MsgProp … since streamMsg's sending buffer is full` | peer traffic starved by client traffic | — |

**What kind of disk.** etcd's hardware guidance is the most misread page in its documentation: *"Typically 50 sequential IOPS is required. For heavily loaded clusters, 500 sequential IOPS is recommended. Note that most cloud providers publish concurrent IOPS rather than sequential IOPS; the published concurrent IOPS can be 10x greater than the sequential IOPS."* Read that twice. A volume advertising 3,000 IOPS may deliver 300 sequential-sync IOPS, and etcd's workload is a serial chain of `fdatasync` calls where each one waits for the last. **Throughput on the spec sheet tells you nothing about the number etcd cares about.** Measure it with `fio` before you trust it. Typical `fdatasync` latency is ~10 ms on a spinning disk and under 1 ms on SSD, per etcd's own performance documentation.

Other defaults that shape the write path, from `server/embed/config.go` and `server/storage/quota.go`:

| Setting | Default | Why it matters |
|---|---|---|
| `--quota-backend-bytes` | 2 GB (8 GB suggested maximum) | Exceeding it raises an alarm and etcd rejects writes with `mvcc: database space exceeded` |
| `--snapshot-count` | 10,000 entries | How often the log is compacted into a snapshot |
| `--max-request-bytes` | 1.5 MB | The single-request ceiling; oversized objects fail here |
| `--max-txn-ops` | 128 | Operations per transaction |
| `--max-wals` / `--max-snapshots` | 5 / 5 | Files retained on disk |
| `--auto-compaction-retention` | off | Without it, MVCC history grows until the quota alarm fires |

### The causal chain, stated correctly

Here is the chain from a disk problem to a fleet-wide scheduling stall — with the magnitudes that actually apply, because the version of this story that says "a 40 ms fsync causes an election" is wrong and will be caught:

```
  A noisy neighbour steals IOPS from the shared volume
        │
        ▼
  wal fdatasync latency rises: 1 ms → 40 ms → 400 ms → multi-second stalls
        │
        ├─▶ at TENS of ms:   commit latency rises proportionally. Every K8s write
        │                    (pod status, lease renewal, event) slows. No elections.
        │                    Symptom: apiserver p99 up, "apply request took too long"
        │                    in etcd logs, nothing looks broken.
        │
        ├─▶ at HUNDREDS of ms: the leader's raft loop — which does the WAL sync
        │                    inline — delays heartbeats. `heartbeat_send_failures_total`
        │                    climbs; the "leader is overloaded likely from slow disk"
        │                    warning appears. Still no election, because a follower
        │                    needs to miss ~10 consecutive heartbeats (election-timeout
        │                    1000 ms ÷ heartbeat-interval 100 ms).
        │
        └─▶ at MULTI-SECOND stalls: followers exceed the 1 s election timeout →
                                    election → the new leader must commit a no-op
                                    before it can serve linearizable reads or commit
                                    anything → writes AND linearizable reads stall
                                    for the election plus that no-op round trip →
                                    if the disk is shared cluster-wide, the new
                                    leader hits the same wall → LEADER CHURN.
        │
        ▼
  kube-apiserver writes stall (leases, pod status, bindings)
        │
        ▼
  · controller leader elections start failing (leases are etcd writes)
  · scheduler binds queue up behind the write path
  · kubelet status updates back up; nodes drift toward NotReady
        │
        ▼
  "Scheduling is slow across the entire fleet" — with no storage alert anywhere,
  because the volume is inside its throughput budget the whole time.
```

**Under a GPU platform, apply the same reasoning at the write-rate end.** The reason OpenAI-scale clusters move Kubernetes Events into a *separate* etcd cluster is that Events are high-volume, low-value writes that share the WAL with the scheduling-critical objects; splitting them means an event storm cannot lengthen the fsync queue in front of a pod binding. The general rule: **anything that shares etcd's WAL shares etcd's commit latency.** That includes another tenant's disk, another controller's chatty status writes, and your own metrics pipeline if you were unwise enough to put it in CRDs.

## Perspectives

**The algorithm-designer view.** Raft's contribution is not a better bound; Multi-Paxos gets equivalent results. Its contribution is decomposition — leader election, log replication, and safety as three separately-understandable pieces, plus a strong leader so all entries flow one way. Paxos's canonical operational account, Google's *Paxos Made Live*, makes the point from the other side: the algorithm was the small part, and snapshotting, disk corruption, master leases, membership churn and testing dominated the real cost. Raft's design goal — an algorithm engineers can implement correctly under pressure — is why essentially every consensus system built after 2014 (etcd, Consul, CockroachDB, TiKV) chose it.

**The operator / on-call view.** Four signals answer nearly every etcd page: `etcd_disk_wal_fsync_duration_seconds` p99 (disk), `etcd_network_peer_round_trip_time_seconds` p99 (network), `etcd_server_leader_changes_seen_total` rate (stability), and backend size against quota (capacity). The chain to memorise runs disk → heartbeat delay → election → write stall → fleet-wide scheduling delay, and the magnitudes matter: tens of milliseconds is a latency problem, seconds is an availability problem. Add one habit: when etcd looks slow, check *which member*. Raft's commit waits on the slowest member of the fastest majority, so one degraded machine in a five-member cluster is invisible until it becomes the third-fastest.

**The hardware view.** etcd's bottleneck is sync-write *latency*, not bandwidth. That single fact invalidates most cloud storage marketing, which quotes concurrent IOPS — up to 10× the sequential figure, per etcd's own documentation. Network-attached storage adds a network round trip inside every `fdatasync`, on the critical path of every write, in a serial chain. It also explains what looks like a paradox: etcd can be latency-starved on a volume that is nowhere near its throughput limit, because it is not asking for throughput. Bandwidth does matter in exactly one place — member recovery, where etcd's guidance is that ~10 MB/s recovers 100 MB of data in about 15 seconds.

**The cost / risk view.** Sizing is a purchase with a stated price: going 3 → 5 buys roughly a 30× reduction in quorum-loss probability against *independent* failures, and costs one extra cross-domain RTT on every commit (the majority now includes the second-slowest member) plus two more fsyncs per proposal. Against *correlated* failure it buys nothing, which is why the honest framing is failure domains, not node counts. And there is a floor under all of it: correlated failure is not a tail risk in a GPU fleet — one power domain, one top-of-rack switch, one kernel rollout, one cluster-wide config push. Five members in one rack is a more expensive three-member cluster.

## Real-world use cases

- **etcd issue #15220 — the progress-notification race** (verified through KEP-2340, which quotes it in detail). etcd's watch progress notification could, in versions before v3.4.25 / v3.5.8, be delivered *before* an event at the same revision, inverting the ordering guarantee callers depend on. Kubernetes' consistent-read-from-cache design (lesson 01) rests entirely on that ordering, so the KEP describes the bug as causing "silent corruption that cannot be automatically detected prior to acting upon the corrupted data" and adds a startup safeguard that refuses to enable the feature against an etcd too old to have the fix. **What it shows:** a consensus system's *auxiliary* guarantees — progress notifications, leases, watch ordering — are where the safety bugs actually live, exactly as *Paxos Made Live* predicted. The core commit protocol was never the risky part.
- **etcd's shipped alert thresholds** (`contrib/mixin/alerts/alerts.libsonnet`, read from the repository). `etcdHighFsyncDurations` at p99 > 0.5 s (warning) and > 1 s (critical); `etcdHighCommitDurations` at p99 > 0.25 s; `etcdGRPCRequestsSlow` at p99 > 0.15 s critical; `etcdHighNumberOfFailedProposals` at a rate above 5 per 15 minutes; `etcdDatabaseQuotaLowSpace` above 95% of quota. **What it shows:** the gap between a health target and a paging threshold, and a ready-made starting point you can copy rather than invent.
- **etcd's own sizing table**, which is stated in Kubernetes terms: a "small" cluster (≈50 K8s nodes, <100 clients, <200 req/s, <100 MB) maps to an AWS `m4.large`; "large" (≈1,000 K8s nodes, <1,500 clients, <10,000 req/s, <1 GB) to an `m4.2xlarge`; "xLarge" (≈3,000 K8s nodes, >10,000 req/s) to an `m4.4xlarge` with 16 vCPU and 64 GB. **What it shows:** the resource that scales with cluster size is RAM and disk latency, not CPU — etcd needs "two to four cores" for typical clusters, and eight to sixteen only for very heavy deployments.
- **OpenAI, *Scaling Kubernetes to 2,500 nodes* and *to 7,500 nodes*.** The publicly reported findings that match this lesson's thesis: etcd write latency was pathological on network-attached SSD despite ample rated IOPS, and moving the etcd data directory to local SSD fixed it; at larger scale they separated Kubernetes Events into their own etcd cluster and ran five etcd members alongside five API servers. **Not re-fetched this pass** (openai.com is blocked by this environment's egress proxy) — treat the specific figures as remembered rather than verified, and re-check before quoting them; the *mechanism* is independently confirmed by etcd's own sequential-vs-concurrent-IOPS guidance above.
- **Roblox, *Return to Service 10/28–10/31/2021*** — a 73-hour outage whose root cause was in the consensus storage layer: a Consul streaming feature under load hit a pathological performance case in BoltDB, the same storage engine family etcd uses, compounded by freelist handling that kept the on-disk file from shrinking, and by a monitoring stack that depended on the failed cluster. **Not re-fetched this pass** (blog.roblox.com blocked) — cited for the shape of the failure, not for its numbers. The transferable lessons are: the storage engine under your consensus log is part of your consensus system's reliability, and monitoring that depends on the thing it monitors is not monitoring.
- **Jepsen, *etcd 3.4.3*.** Reported etcd's key-value operations holding to their strict-serializable claim under partitions, pauses and clock skew, while finding the *lock* API unsafe under asynchronous conditions. **Not re-fetched this pass** (jepsen.io blocked). The durable lesson matches lesson 01: guarantees are per-operation, and the layers built *on top of* consensus are where they leak.

## Worked example

**Setup.** Size and budget etcd for a GPU-fleet control plane. Three availability zones, cross-AZ RTT 2 ms, same-AZ RTT 0.3 ms. Local NVMe with measured `wal_fsync` p99 = 3 ms. Target: survive the loss of any one AZ.

**Step 1 — failure domains before node counts.** The requirement "survive one AZ" plus three AZs forces at least one member per AZ and a majority that survives losing any single AZ. A 3-member cluster with one member per AZ satisfies it: lose any AZ, 2 of 3 remain, quorum is 2. A 5-member cluster (2+2+1) also satisfies it and additionally survives one AZ *plus* one extra machine — but note the tight case: losing a 2-member AZ leaves exactly 3 of 5, exactly quorum, zero further margin.

**Step 2 — commit latency budget for each option.**

```
  3 members, 1 per AZ. Fastest majority = leader + nearest follower.
    commit ≈ leader_fsync + RTT_nearest_follower + follower_fsync + apply
           ≈ 3 ms        + 2 ms                 + 3 ms           + ~1 ms
           ≈ 9 ms p99 per write

  5 members, 2+2+1. Fastest majority = leader + 2 followers, so the commit
  waits on the SECOND-nearest follower — which, if the leader's own AZ holds
  a peer, may still be local:
    best case  (peer in leader's AZ): ≈ 3 + 0.3 + 3 + 1 ≈ 7.3 ms
    worst case (leader in the 1-member AZ, both acks cross AZ boundaries):
                                       ≈ 3 + 2   + 3 + 1 ≈ 9 ms, with a fatter
                                       tail because it is the max of two
                                       independent cross-AZ RTTs, not one.

  Tail intuition: waiting on the max of k independent samples pushes you further
  into the distribution's tail. Two 2 ms links with a p99 of 8 ms give a combined
  p99 nearer 9–10 ms, because you need BOTH to be fast. Wider majorities cost
  tail latency even when they do not cost mean latency.
```

**Step 3 — availability arithmetic.** With `p = 0.01` per member (independent), quorum-loss probability is ≈ 3p² ≈ 3×10⁻⁴ for 3 members and ≈ 10p³ ≈ 1×10⁻⁵ for 5 — about 4.3 minutes vs 8 seconds per month. But the requirement is AZ survival, and AZ failure is correlated, so the honest statement is: *both* configurations meet the stated requirement; the 5-member cluster adds ~30× margin against uncorrelated machine failure and one extra machine's worth of margin during an AZ outage, in exchange for a fatter commit tail and two more fsyncs per proposal.

**Decision: 3 members, one per AZ**, unless you must survive an AZ loss *concurrently* with a machine failure — for example during a rolling kernel upgrade, when one member is deliberately down and your effective margin is already spent. That is the deciding number: **how many simultaneous independent failures must you survive *while* a planned maintenance is in flight?**

**Step 4 — the write budget, end to end.** A 9 ms p99 commit is the floor under every Kubernetes write. Sanity-check it against fleet write volume: at 2,000 writes/s of pod status, leases and bindings, and etcd batching proposals, a 3-member cluster on 8 vCPU is well inside the published envelope (etcd's own benchmark reached 44,341 writes/s at 22 ms average latency with 1,000 concurrent clients on comparable hardware). **Your constraint is latency, not throughput** — which is exactly why the fix for a slow control plane is a faster disk rather than a bigger cluster.

**Step 5 — inject the fault, with correct magnitudes.** A noisy neighbour drives `wal_fsync` p99 from 3 ms to 400 ms:

- Commit latency goes from ~9 ms to ~800 ms (two fsyncs on the path). Every API-server write inherits it; `apply request took too long` starts appearing at the 100 ms threshold.
- `etcd_server_heartbeat_send_failures_total` climbs and the "leader is overloaded likely from slow disk" warning appears, since heartbeats now exceed 2 × 100 ms.
- **No election yet**: a follower needs 1,000 ms without any heartbeat, i.e. roughly ten consecutive missed beats.
- Kubernetes-side symptom: `kube-controller-manager` and `kube-scheduler` lease renewals (etcd writes on a default 15 s lease with a 10 s renew deadline) start missing, controllers begin flapping their leadership, and scheduling throughput collapses — while every storage dashboard shows the volume comfortably inside its throughput budget.
- Push to multi-second stalls and you cross into elections. Each election costs the timeout plus the new leader's no-op commit before it can serve linearizable reads. If the disk problem is cluster-wide, the new leader hits it too: **leader churn**, visible as `etcd_server_leader_changes_seen_total` climbing continuously.

**Step 6 — the alerts you write from this.** Page on `histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) > 0.05` for 10 m (a tenth of etcd's own warning threshold, chosen because your commit budget is 9 ms and 50 ms is already 5× degraded), on `rate(etcd_server_leader_changes_seen_total[15m]) > 0` for 15 m, and on `etcd_server_heartbeat_send_failures_total` increasing. Link all three to one runbook whose first line is: **check per-member fsync before you look at the network.**

## Practice

*Feeds the [staff design portfolio](../practice/staff-design-portfolio/README.md).*

1. **etcd sizing worksheet (artifact: `etcd-sizing-worksheet`).** For your real fleet, justify 3 vs 5 in writing. Required contents: your failure domains and which one is largest; the majority that survives losing it; the independent-failure arithmetic (`3p²` vs `10p³`) with your own `p` and where you got it; the commit-latency delta from your measured cross-domain RTT; and one sentence naming the deciding number. Include the maintenance case — what your margin is *while* one member is down for patching.
2. **Write-latency budget (artifact: `write-latency-budget`).** Pull p99 for `etcd_disk_wal_fsync_duration_seconds`, `etcd_disk_backend_commit_duration_seconds` and `etcd_network_peer_round_trip_time_seconds`, per member. Compute expected commit latency from the formula and compare it against measured `grpc_server_handling_seconds` p99 for `Txn`/`Put`. If the measured value is much larger than the computed one, you have found queueing rather than disk latency — say so and name where. Ship alert rules with two tiers (your SLO, etcd's threshold) and the runbook chain.
3. **Prove your disk.** Run `fio` in the sync-write mode etcd actually uses (small sequential writes with `fdatasync` per write) against your etcd volume, and compare the result to the volume's advertised IOPS. Write down both numbers and the ratio. If the ratio is anywhere near 10×, you have just demonstrated the concurrent-vs-sequential IOPS trap on your own hardware.
4. **Read the trace.** Run a three-member etcd locally, `kill -STOP` the leader for two seconds, and read the logs of all three members side by side. Identify: the missed heartbeats, the term increment, the `MsgVote` exchange, the new leader's first entry, and the moment reads start succeeding again. Write the timeline with real timestamps. Then repeat with `--pre-vote=false` if your version allows, partition one member for a minute, and compare the term numbers on rejoin.
5. **A *Paxos Made Live* retro.** For one consensus system you operate, write half a page on which incidents came from the core protocol versus from snapshotting, disk exhaustion, leases, membership changes, or client behaviour. The ratio will make the paper's argument better than the paper does.

## Common pitfalls

1. **"Bigger cluster, more fault tolerance."** 4 members tolerate the same single failure as 3, with a majority of 3 instead of 2 — more fsyncs, more tail, more machines that can fail. Symptom: someone "temporarily" adds a member and the cluster becomes both slower and more fragile. Mechanism: fault tolerance is `⌊(n−1)/2⌋`, which is flat from odd `n` to `n+1`.
2. **"Add the replacement member first, then remove the dead one."** Adding to a 3-member cluster raises quorum to 3 while one member is already dead; if the newcomer fails to join, you have 2 up, 2 down, and need 3 votes to undo it — permanent quorum loss requiring a restore from snapshot. Mechanism: quorum is computed over configured members, not live ones. Remove first, then add, and use a learner.
3. **"Two leaders in the logs means split brain."** Two leaders in *different terms* is normal during a partition. The stale one cannot commit anything: followers reject its lower term, and it steps down on discovering a higher one. Split brain requires two leaders that can both commit, which quorum intersection makes impossible.
4. **"A 40 ms fsync spike causes elections."** It does not. With a 100 ms heartbeat interval and a 1,000 ms election timeout, a follower must go ~10 consecutive heartbeats without contact. Tens of milliseconds is a *latency* problem (real, user-visible, and worth fixing); elections need stalls approaching the election timeout. Getting this magnitude wrong in a review is a tell.
5. **"The volume is SSD and rated for thousands of IOPS, so it's fine for etcd."** etcd needs *sequential sync-write* IOPS — 50 minimum, 500 for heavy clusters — and cloud providers publish *concurrent* IOPS, which its own docs say can be 10× higher. Measure with `fio` before believing a spec sheet.
6. **"`ReadIndex` is a weaker, cheaper read."** It is fully linearizable. It removes the disk write, not the guarantee: it confirms leadership with a heartbeat round and waits for the applied index to catch up. The genuinely weaker option is a *serializable* read (`WithSerializable`), served locally with no quorum check, and the genuinely riskier one is a lease-based read, which substitutes a clock assumption for the round trip.
7. **"Consensus makes my data safe, so I don't need backups."** Raft protects against crash faults, not against operator error, correlated corruption or a bad client. etcd's own quota alarm — `mvcc: database space exceeded` at 2 GB by default — is a failure mode that stops all writes and has nothing to do with consensus. Set `--auto-compaction-retention`, watch the quota metric, and take snapshots.

## Self-check

- **Write the commit-latency formula and justify every term.** **Answer:** `commit ≈ leader_fsync(WAL) + RTT to the slowest member of the fastest majority + that member's fsync(WAL) + apply`. The leader's fsync is required because a vote or entry that is not durable can be lost across a crash and break safety; the RTT is unavoidable because a majority must actually receive the entry; the follower's fsync is required for the same durability reason before its acknowledgement means anything; apply is the backend commit into the MVCC store before the value is readable. The formula shows why the cluster is disk-bound: two fsyncs and one round trip, and on local NVMe the fsyncs typically dominate a same-AZ RTT.

- **Your `wal_fsync` p99 jumps from 3 ms to 400 ms. What happens, in order, and what does *not*?** **Answer:** Commit latency roughly doubles the fsync increase (two on the path), so every Kubernetes write slows to hundreds of milliseconds; `apply request took too long` appears past 100 ms. The leader's raft loop delays heartbeats, so `etcd_server_heartbeat_send_failures_total` climbs and the "leader is overloaded likely from slow disk" warning appears once sends exceed 2× the 100 ms interval. Controller and scheduler leader leases start missing renewals; scheduling throughput collapses. What does **not** happen at 400 ms is an election — that needs ~1,000 ms of silence, ten missed heartbeats. Push to multi-second stalls and you get elections, and if the disk problem is fleet-wide, leader churn. Most likely root cause: I/O contention on a shared volume; the fix is dedicated, isolated storage, and the tell is that the volume is inside its throughput budget the whole time because etcd is latency-bound, not throughput-bound.

- **Why can't a leader commit an entry from a previous term by counting replicas?** **Answer:** Because a majority holding an old-term entry does not prevent a *different* server, whose last entry has a higher term, from winning a later election and overwriting it — the paper's Figure 8. The election restriction compares last-entry term first and index only as a tiebreak, so a server with a higher-term tail beats a server with a longer lower-term log. The rule is therefore: a leader commits by counting replicas only for an entry from **its own current term**, which implicitly commits everything before it. etcd implements this as the `matchTerm` check inside `maybeCommit`. The practical consequence is the no-op entry a new leader appends immediately — and, because linearizable reads are gated on having committed an entry in the current term, the fact that reads and writes stall together after an election.

- **Explain `ReadIndex` end to end, and why it is not weaker than a log-append read.** **Answer:** The leader records its current `commitIndex` as the read index; if it has not yet committed an entry in its own term, it parks the read until it has. It then broadcasts heartbeats carrying a context token, and once a majority acknowledges that context, it knows it was still leader at that index. It waits for `appliedIndex ≥ readIndex` and serves from local state. That satisfies linearizability: no write that completed before the read began can be missing, and no superseded leader can answer. It costs one network round trip and zero disk writes, and it batches — one confirmation releases every read waiting at or below that index. etcd retries after 500 ms if no response arrives, counting it in `etcd_server_slow_read_indexes_total`. Note the structural identity with Kubernetes' consistent-read-from-cache: confirm a revision cheaply, then wait for local state to reach it.

- **A partitioned member keeps timing out. Which two mechanisms protect the healthy majority, and how does each work?** **Answer:** `PreVote` — before incrementing its term, the node asks whether it *would* win, and peers answer without updating their own terms; a partitioned node therefore never inflates its term, so on rejoin it cannot force the legitimate leader to step down. `CheckQuorum` — a leader that has not heard from a majority within an election timeout steps down on its own (`becomeFollower`), so an isolated leader cannot keep believing it is one. `CheckQuorum` is mandatory before the library will allow lease-based reads, because a stale leader serving from a lease is the one case where believing you are leader is actually dangerous.

- **3 vs 5 members: give the availability arithmetic and the case where it does not apply.** **Answer:** With independent per-member unavailability `p`, quorum loss is ≈ `3p²` for 3 members and ≈ `10p³` for 5. At `p = 0.01` that is ~3×10⁻⁴ vs ~1×10⁻⁵ — roughly 4.3 minutes vs 8 seconds of quorum loss per month, a ~30× improvement, paid for with one extra cross-domain RTT on the commit path (the majority now includes the second-slowest member) and two more fsyncs per proposal. It does not apply when failures are correlated: one rack, one power domain, one AZ, one kernel rollout. There both numbers collapse to the probability that the shared domain fails, and five members in one failure domain is just an expensive three-member cluster. The right question is how many independent failure domains you have and whether a majority survives losing the largest.

- **Why is "just lower the election timeout for faster failover" wrong?** **Answer:** Because the safe operating region is `broadcastTime ≪ electionTimeout ≪ MTBF`, and `broadcastTime` includes a disk sync whose tail you do not control. etcd requires `election-timeout ≥ 5 × heartbeat-interval` and its documentation recommends at least 10× the peer RTT; pushing the timeout down toward the broadcast time makes ordinary disk or network jitter look like leader death, and every spurious election costs a write stall plus the new leader's no-op commit. Faster failover that triggers on noise is slower availability.

## Connections & what's next

This lesson cashes in lesson 01's claim: `ReadIndex` and the majority-commit protocol are the actual machinery behind etcd's linearizable label, and the commit-latency formula is the concrete price of PACELC's Else tax. The lesson-01 point that "correctness lives at the write" now has a number attached — roughly 9 ms per write on well-provisioned local NVMe across three AZs, and the fsync is the dominant term.

Forward, **lesson 03** asks what happens when you refuse to pay this price: replication with a *tunable* quorum (`W + R > N`) instead of a fixed majority, sloppy quorums that trade the overlap guarantee for availability, and partitioning, where the bottleneck moves from disk to network fan-out and skew becomes the default failure mode. **Lesson 05** picks up the queueing consequence of a slow commit path — a write path with a hard concurrency limit and rising service time is exactly the setup for the metastable failure it formalises. **Lesson 06** revisits the failure-detection assumption underneath `PreVote` and `CheckQuorum`: deciding "is that member actually dead" is the same problem gray-failure detection faces at fleet scale, and it is provably imperfect.

Carry forward: *if consensus's cost is dominated by disk fsync on a majority, what does the cost curve look like when you stop requiring a majority at all — and what exactly do you give up?*

## References & further reading

**Primary sources — verified against upstream Git repositories this pass**

1. **Ongaro, D. & Ousterhout, J., *In Search of an Understandable Consensus Algorithm* (Raft), extended version** — `raft.pdf` in <https://github.com/raft/raft.github.io> (raft.github.io itself is blocked here; the PDF was read from the repository). **Source of** the five safety properties, the election restriction and its up-to-date definition, the Log Matching Property and the `AppendEntries` consistency check, the Figure 8 scenario reproduced above, the randomised 150–300 ms election-timeout example, the `broadcastTime ≪ electionTimeout ≪ MTBF` inequality with its 0.5–20 ms and 10–500 ms ranges, the joint-consensus configuration change and the Figure 10 argument for why a direct switch is unsafe. **Caveat:** the paper's Figure 2 (the state and RPC specification) is rendered as an image and could not be extracted as text in this environment; the field tables above were reconstructed from the paper's prose and cross-checked field by field against the etcd implementation below.
2. **`etcd-io/raft`** — <https://github.com/etcd-io/raft>. **Source of** the `raftpb.Message` field list and the `MessageType` enum reproduced above, `isUpToDate` and `maybeCommit` (quoted verbatim), the `readOnly` structure with `addRequest`/`recvAck`/`maybeAdvance`, `sendMsgReadIndexResponse` and the `ReadOnlySafe` vs `ReadOnlyLeaseBased` options (including the validation that `CheckQuorum` must be enabled for lease-based reads), the `MsgCheckQuorum` step-down path, the `PreVote` term-handling comments, the `ElectionTick > HeartbeatTick` validation with its suggested 10× ratio, and the `ConfState`/`ConfChangeV2` joint-consensus fields.
3. **`etcd-io/etcd`** — <https://github.com/etcd-io/etcd>. **Source of** the linearizable read loop (`server/etcdserver/read/read.go`: `LinearizableReadNotify`, `requestCurrentIndex`, the `ApplyWait` step, `readIndexRetryTime = 500 ms`), the heartbeat-send-failure warning and its 2× heartbeat threshold (`server/etcdserver/raft.go`), the `slow fdatasync` warning at `warnSyncDuration = 1 s` (`server/storage/wal/wal.go`), `DefaultWarningApplyDuration = 100 ms` and the `apply request took too long` warning, and the defaults in `server/embed/config.go` and `server/storage/quota.go`: `heartbeat-interval` 100 ms, `election-timeout` 1000 ms with the `5 × heartbeat` validation and 50 s maximum, `snapshot-count` 10,000, `max-request-bytes` 1.5 MB, `max-txn-ops` 128, `max-wals`/`max-snapshots` 5, quota 2 GB default and 8 GB suggested maximum.
4. **`etcd-io/etcd` — `contrib/mixin/alerts/alerts.libsonnet`.** **Source of** every alert threshold in the metrics table: `etcdHighFsyncDurations` 0.5 s warning / 1 s critical, `etcdHighCommitDurations` 0.25 s, `etcdMemberCommunicationSlow` 0.15 s, `etcdGRPCRequestsSlow` 0.15 s, `etcdHighNumberOfFailedProposals` rate > 5, `etcdDatabaseQuotaLowSpace` 95%. **Correction to the previous version of this lesson:** it gave "`wal_fsync` p99 < 10 ms, `backend_commit` p99 < 25 ms" as *the* thresholds to watch. Those are defensible health targets but they are not etcd's shipped alert thresholds, which are 0.5 s and 0.25 s; the lesson now distinguishes SLO from page.
5. **etcd documentation — Tuning** (`content/en/docs/v3.6/tuning.md` in <https://github.com/etcd-io/website>; etcd.io is blocked here). **Source of** the 100 ms / 1000 ms defaults with rationale, "heartbeat interval around 0.5–1.5× the round-trip time," "election timeouts must be at least 10 times the round-trip time," the 50 s upper limit and its global-deployment rationale, the requirement that all members share the same values, `--snapshot-count` tuning, the `ionice` disk-priority recommendation, and the `dropped MsgProp … sending buffer is full` symptom with its traffic-control fix.
6. **etcd documentation — Hardware recommendations** (same repository). **Source of** "50 sequential IOPS … 500 sequential IOPS for heavily loaded clusters," the warning that published *concurrent* IOPS can be 10× the sequential figure, the `fio`/`diskbench` recommendation, the ~10 MB/s recovers 100 MB in 15 s guidance, the CPU (2–4 cores typical, 8–16 heavy) and memory (8 GB typical, 16–64 GB heavy) guidance, and the small/medium/large/xLarge sizing table mapped to 50 / 250 / 1,000 / 3,000-node Kubernetes clusters.
7. **etcd documentation — FAQ** (same repository). **Source of** the quorum formula `(n/2)+1`, the cluster-size/majority/failure-tolerance table reproduced above, the argument that adding a member to an odd cluster is worse and riskier, the remove-then-add rule, `strict-reconfig-check`, the "no more than seven members" guidance and the Chubby-suggests-five comparison, the 2 GB/8 GB quota numbers, the `mvcc: database space exceeded` alarm, and the statement that applying a request should normally take under 50 ms with a warning above 100 ms.
8. **etcd documentation — Performance** (same repository). **Source of** the baseline benchmark numbers quoted above (3-member GCE cluster, 8 vCPU/16 GB/50 GB SSD, etcd 3.2.0): 583 writes/s at 1.6 ms single-client, 44,341 writes/s at 22 ms with 1,000 clients; linearizable vs serializable reads at 1,353 vs 2,909 QPS single-client and 141,578 vs 185,758 QPS at 1,000 clients. Dated snapshot — the ratios transfer, the absolute numbers should be re-measured on your hardware with the bundled `benchmark` tool.
9. **KEP-2340, *Consistent Reads from Cache*** — <https://github.com/kubernetes/enhancements>. **Source of** the etcd #15220 progress-notification race, its fix in etcd v3.4.25 / v3.5.8, and the description of its consequence as silent, undetectable corruption. Also the reason lesson 01's consistent-read path depends on this lesson's machinery.

**Primary sources — not fetchable in this environment, not relied on for any number above**

10. **Fischer, M., Lynch, N. & Paterson, M. (1985), *Impossibility of Distributed Consensus with One Faulty Process*, JACM 32(2)** — the FLP result behind "liveness needs a timing assumption." Not fetched; stated as attributed.
11. **Lamport, L. (1998), *The Part-Time Parliament*, and (2001) *Paxos Made Simple*** — the original Paxos papers. Not fetched. Read *Paxos Made Simple* first; the 1998 paper's Greek-parliament framing is the historical artefact, not the content.
12. **Chandra, T., Griesemer, R. & Redstone, J. (2007), *Paxos Made Live — An Engineering Perspective*, PODC** — <https://static.googleusercontent.com/media/research.google.com/en//archive/paxos_made_live.pdf>. The canonical account of the gap between a consensus paper and a running system: snapshotting, disk corruption, master leases, membership churn, and testing. Not fetched; cited for its argument, which the etcd #15220 entry above independently illustrates.
13. **Burrows, M. (2006), *The Chubby Lock Service for Loosely-Coupled Distributed Systems*, OSDI** — the operational contrast case, and the source of the "five members" convention etcd's FAQ cites. Not fetched.
14. **Ongaro, D. (2014), *Consensus: Bridging Theory and Practice* (PhD dissertation)** — the extended treatment of membership changes, log compaction, client interaction, and the leadership-transfer and pre-vote extensions. The chapter on cluster membership is the definitive treatment of the learner/joint-consensus material summarised above. Not fetched.

**Real-world engineering — not fetchable this pass**

15. **OpenAI, *Scaling Kubernetes to 2,500 nodes* and *Scaling Kubernetes to 7,500 nodes*** — <https://openai.com/index/scaling-kubernetes-to-2500-nodes/>, <https://openai.com/index/scaling-kubernetes-to-7500-nodes/>. Blocked by the egress proxy; the local-SSD and separate-Events-cluster findings are recalled, not verified. Re-check before quoting figures.
16. **Roblox, *Return to Service 10/28–10/31/2021*** — <https://blog.roblox.com/2022/01/roblox-return-to-service-10-28-10-31-2021/>; and **Jepsen, *etcd 3.4.3*** — <https://jepsen.io/analyses/etcd-3.4.3> with the etcd team's response at <https://etcd.io/blog/2020/jepsen-343-results/>. All blocked; cited for the shape of their findings, not their numbers.
