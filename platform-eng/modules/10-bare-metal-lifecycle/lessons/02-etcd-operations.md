---
lesson: "10.2"
title: "etcd operations: disk, quorum, and the 2am restore"
module: "10"
concept: "etcd operations: disk, quorum, and the 2am restore"
status: not-started
est_time: "12h"
prev: "01-cluster-provisioning.md"
next: "03-control-plane-ha.md"
artifacts: []
sources: 10
---

# 10.2 · etcd operations: disk, quorum, and the 2am restore

> **Concept.** You know what etcd is — now you own its disk, its quorum, its NOSPACE alarm, and its disaster-recovery runbook.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Where this fits

Lesson 10.1 got you a working control plane and the PKI to prove it: you can now name every cert
between the apiserver and etcd and trace a write end to end. This lesson is the **anchor of the
whole module** — per the module README, it's "the thing that pages you" — because everything else
in this module (HA topology in 10.3, declarative fleets in 10.4, even the health-remediation loop
in 10.6) assumes the datastore underneath the control plane stays up. 10.1 built the pipe between
apiserver and etcd; this lesson is everything that can go wrong on the etcd side of that pipe once
real write traffic hits it — quota exhaustion, disk latency, quorum loss, and how you get the
cluster writable again under time pressure.

## Why this matters

etcd is the single stateful thing in the cluster and the thing that will actually page you. On
EKS/GKE, AWS and Google owned etcd's disk, its backups, and its quorum — you never got that page.
On bare metal you do. A full etcd turns the **entire cluster read-only** (no deploys, no
scale-ups, no GPU-job scheduling); a lost quorum stops the API cold; a corrupt data directory
means restoring from a snapshot under time pressure, and a restore is not a file copy — it mints a
new Raft identity, which is why people who have only read about it get it wrong on the night.

This is not hypothetical: a documented kOps incident (below) shows a real team hitting a NOSPACE
alarm in production and working the exact compact/defrag/disarm drill this lesson teaches, live,
under an active outage. The module's checkpoint gates on recovering etcd from a quota-exceeded
read-only state **in under 30 minutes**, timed, unaided — and the interview probe is the same
question. On a GPU fleet the cost is measurable: a frozen control plane means no new jobs bind
while every rented or owned H100 keeps burning power and depreciation.

## What's new here (calibration)

Per this module's calibration (see the [README](../README.md#calibrated-to-your-background---what-we-skip)):
Module 02 taught etcd's **role** — the single source of truth, the only thing the apiserver
persists to, watch/revision semantics behind informers — plus a first pass at quorum, MVCC, and
the compaction/defragmentation distinction. It answered *what etcd is for* and gave you the
headline numbers. You also already have 04's GPU Operator experience, 05's XID/NPD concepts, and
general on-prem fluency; none of that is re-taught.

This lesson is **operations**, and everything below goes past where 02 stopped:

- **How a write physically lands** — the gRPC quota preflight, the Raft proposal, the WAL `fsync`
  that sets the latency floor, and the bbolt commit — so "etcd hates slow disks" becomes a
  causal chain rather than a slogan.
- **Raft as a message sequence**, including what a partition actually does to a three-member
  cluster and why the minority side goes read-only rather than serving stale writes.
- **The on-disk layout** — WAL segments, Raft snapshots, and the single bbolt file — and where
  compaction, defragmentation, Raft snapshotting, and `etcdctl snapshot save` each act on it.
  Four different things share two words; conflating them is the classic mistake.
- **The quota mechanism at source level** — the exact cost formula, why the check uses the
  *physical* file size, and therefore why defragmenting can clear a NOSPACE that compaction alone
  cannot.
- **Exactly which operations an alarm blocks** — NOSPACE and CORRUPT block very different things,
  and knowing the difference is what lets you recover from one and not make the other worse.
- **Worked capacity math** — database growth from object count, size and churn, and time-to-quota
  when compaction stops.
- **What a restore actually does to the data directory**, and why "restore on one member and let
  the others catch up" cannot work.

**Version note.** Every flag, default, constant, error string, alarm behaviour and command surface
below was read from the `etcd-io/etcd` source tree: branch `release-3.6` (tip `v3.6.14`, August
2026) for etcd itself, with `release-3.5` used for the before/after comparisons, and the
`release-1.36` branch of `kubernetes/kubernetes` for the apiserver-side compaction logic. etcd.io
and kubernetes.io are both unreachable from this environment's egress proxy, so the documentation
sites were **not** consulted; see [References](#references--further-reading). Transcripts are
representative — formatted exactly as the real tools format them, with realistic values — not
literal captures.

**Which etcd you actually have.** kubeadm's `DefaultEtcdVersion` by Kubernetes release
(`cmd/kubeadm/app/constants/constants.go`): **1.33 → etcd 3.5.24**, **1.34 → 3.6.5**, **1.35 and
1.36 → 3.6.8**. So etcd 3.6 became kubeadm's default at **Kubernetes 1.34**, not 1.33 — a
distinction that matters because the 3.5→3.6 tooling split below breaks runbooks. kubeadm 1.36
still accepts an external etcd as old as `3.5.24-0` (`MinExternalEtcdVersion`). Check yours with
`kubectl -n kube-system get pod etcd-<node> -o jsonpath='{.spec.containers[0].image}'`.

## Core concepts

### What etcd actually stores

etcd v3 is a flat, ordered, **multi-version** key–value store. Two facts drive everything else.

**Fact 1: every mutation gets a new, cluster-global revision number.** A revision is a monotonic
64-bit counter incremented once per mutating transaction across the whole store — not per key.
Every version of every key is addressable as `(main revision, sub revision)`. Nothing is ever
overwritten in place; a delete writes a **tombstone**, which is itself a new revision.

**Fact 2: there are two indexes, one in memory and one on disk.**

- The **keyIndex** is an in-memory B-tree mapping *user key* → the list of revisions at which that
  key changed. It exists so a `Get` on a key can find the right revision without scanning.
- The **backend** is a single [bbolt](https://github.com/etcd-io/bbolt) file (`db`) — a
  memory-mapped B+tree — whose main bucket (`key`) is keyed by the **encoded revision**, with the
  serialised `KeyValue` protobuf as the value.

So a read is: look up the key in the in-memory keyIndex to get the revision, then look up that
revision in bbolt to get the bytes. And a write is: append a new revision to bbolt, append the
revision to the keyIndex.

The consequence that will define your operational life: **the size of the backend is not the size
of your data. It is the size of your data plus every superseded version that has not yet been
compacted, plus every free page that has not yet been defragmented.** Kubernetes is an unusually
bad workload in this respect, because a huge share of its writes are updates to objects that
already exist — Node status, Leases, Events, pod status — so history accumulates far faster than
the live object set grows.

Kubernetes keys look like this (from `k8s.io/apiserver/pkg/storage/etcd3`):

```
/registry/pods/default/trainer-0
/registry/leases/kube-node-lease/gpu-node-041
/registry/events/default/trainer-0.17b3c9...
/registry/deployments/prod/api
```

You can read them straight out of etcd with `apiserver-etcd-client.crt` from lesson 10.1, which is
the fastest way to see what is actually consuming your database:

```console
$ etcdctl get /registry/ --prefix --keys-only \
  | sed -n 's|^/registry/\([^/]*\)/.*|\1|p' | sort | uniq -c | sort -rn | head
  41822 events
   9310 pods
   4021 leases
   3155 endpointslices
   1204 configmaps
    812 replicasets
    401 nodes
```

### The write path, stage by stage

```
  CLIENT (kube-apiserver)                       LEADER etcd                    FOLLOWERS
  ───────────────────────                       ───────────                    ─────────
  Put(/registry/pods/…, 6 KiB)
        │ gRPC over TLS (apiserver-etcd-client.crt)
        ▼
  ┌──────────────────────────────────────────────────┐
  │ 1. quotaKVServer.Put → qa.check()                │  <-- BEFORE Raft. Purely local.
  │    cost = 256 + len(key) + len(value)            │      If it fails: ResourceExhausted
  │    ok if backend.Size() + cost < quota           │      + a cluster-wide NOSPACE alarm
  └──────────────────────┬───────────────────────────┘      is ACTIVATED via Raft.
                         ▼
  ┌──────────────────────────────────────────────────┐
  │ 2. Raft propose: append entry to the leader's    │
  │    in-memory log, broadcast MsgApp to followers  │ ──── MsgApp ────▶ append to
  └──────────────────────┬───────────────────────────┘                   their logs
                         ▼                                                    │
  ┌──────────────────────────────────────────────────┐                        ▼
  │ 3. PERSIST: append to WAL segment + fsync()      │                  WAL append + fsync
  │    ── this is the latency floor of the whole     │                        │
  │       Kubernetes control plane ──                │ ◀─── MsgAppResp ───────┘
  └──────────────────────┬───────────────────────────┘
                         ▼
  ┌──────────────────────────────────────────────────┐
  │ 4. COMMIT when a QUORUM (incl. the leader) has   │
  │    fsynced. Advance commitIndex.                 │
  └──────────────────────┬───────────────────────────┘
                         ▼
  ┌──────────────────────────────────────────────────┐
  │ 5. APPLY: uberApplier → (corrupt?) → (capped?) → │  <-- alarms are enforced HERE too,
  │    auth → quota → backend. Assign revision N+1,  │      so an entry already committed
  │    write into the bbolt batch buffer.            │      can still fail at apply time.
  └──────────────────────┬───────────────────────────┘
                         ▼
  ┌──────────────────────────────────────────────────┐
  │ 6. Respond to the client. The bbolt transaction  │
  │    is committed asynchronously (batched), which  │
  │    is why etcd_disk_backend_commit_duration is   │
  │    a separate, coarser metric from wal_fsync.    │
  └──────────────────────────────────────────────────┘
```

Three things to take from that diagram.

**The quota is checked twice, in two different places.** Once in the gRPC layer before the request
enters Raft (`server/etcdserver/api/v3rpc/quota.go`), and once at apply time
(`server/etcdserver/apply/apply.go`). The first is why a client sees an immediate rejection; the
second is why an alarm raised by one member is honoured by all of them — the alarm itself is
replicated through Raft.

**Durability costs one `fsync` per Raft commit, on a quorum of machines.** Not one per key, but
one per batch of proposals the leader can group — so under load etcd amortises, and under light
load it does not. This is why a lightly-loaded cluster on a bad disk feels *worse* than a busy one.

**The reply to the client happens after apply, but the bbolt commit is batched.** The default
batching parameters mean the on-disk B+tree lags the applied state by up to a batch interval;
`consistent_index` in the backend is what makes crash recovery replay exactly the missing entries
from the WAL.

### Raft: election, replication, and what a partition really does

etcd embeds `go.etcd.io/raft` (v3.6.0 in etcd 3.6). Timing parameters, from
`server/embed/config.go`:

| Flag | Default | Meaning | Validation |
|---|---|---|---|
| `--heartbeat-interval` | `100` ms | one Raft "tick"; leader sends heartbeats every tick | must be > 0 |
| `--election-timeout` | `1000` ms | a follower with no heartbeat for this long starts an election | must be ≥ 5 × heartbeat, and ≤ 50000 ms |

`ElectionTicks = election-timeout ÷ heartbeat-interval` = 10 by default. `raft.Config` is built in
`server/etcdserver/bootstrap.go` with `HeartbeatTick: 1`, `ElectionTick: ElectionTicks`,
`CheckQuorum: true`, `PreVote: <--pre-vote, default true>`, `MaxSizePerMsg: 1 MiB`,
`MaxInflightMsgs: 512`.

Two mechanisms in there are worth understanding because they are the difference between "etcd
handles partitions" and "etcd handles partitions *gracefully*":

- **Randomised election timeout.** `resetRandomizedElectionTimeout()` in `raft.go` sets
  `randomizedElectionTimeout = electionTimeout + rand.Intn(electionTimeout)` — so each member waits
  a uniformly random 10–20 ticks (1000–2000 ms at defaults) before campaigning. Without this,
  every follower would campaign at the same instant, split the vote, and repeat forever.
- **PreVote.** A member that has been partitioned away keeps incrementing its term while trying to
  win elections it cannot win. When the partition heals, its higher term would force the healthy
  leader to step down — a real, disruptive outage caused by a node that was never in a position to
  lead. PreVote makes a candidate first ask "would you vote for me?" *without* incrementing its
  term; only if a quorum says yes does it start a real election. **CheckQuorum** is the mirror
  image: a leader that stops hearing from a quorum steps down on its own rather than continuing to
  serve as a leader nobody follows.

Here is the whole thing as a message sequence across three members, including a partition. Read
the middle block carefully; the asymmetry between the two sides is the entire point.

```
 TIME   m1 (leader, term 7)          m2 (follower)              m3 (follower)
 ─────  ────────────────────         ──────────────             ──────────────
 t+0    MsgHeartbeat ──────────────▶ ack                        ack
        (every 100 ms)      └──────────────────────────────────▶
 t+0.2  client PUT k=v
        append to log[42], WAL fsync
        MsgApp{idx 42} ────────────▶ append log[42], WAL fsync  (same)
                            └──────────────────────────────────▶
 t+0.3  ◀── MsgAppResp{42} ──────────┘                          │
        QUORUM = 2 of 3 reached (m1 itself + m2). commitIndex=42.
        apply → revision N+1 → respond OK to the apiserver.
        m3's ack arrives later and changes nothing.  ◀──────────┘

 ══════════════════ NETWORK PARTITION: {m1, m2} | {m3} ══════════════════

 t+1.0  MsgHeartbeat ──────────────▶ ack                        ✗ (dropped)
 t+1.0  client PUT k=v2
        MsgApp{43} ────────────────▶ fsync, ack
        commit 43 with 2/3.  WRITES CONTINUE NORMALLY on the majority side.

 t+1.0  MAJORITY SIDE {m1,m2}                MINORITY SIDE {m3}
        ────────────────────────             ───────────────────
        • has quorum (2 ≥ ⌊3/2⌋+1)           • election timeout fires at
        • keeps its leader                     1000–2000 ms
        • serves reads AND writes            • PreVote: "would you vote for me
        • unaffected                           in term 8?" → nobody answers
                                             • never increments its term,
                                               never becomes a candidate
                                             • has no leader:
                                               etcd_server_has_leader = 0
                                             • linearizable reads BLOCK
                                               (ReadIndex needs a quorum ack)
                                             • serializable reads still work
                                               and RETURN STALE DATA

 ══════════════════ PARTITION HEALS at t+5.0 ══════════════════

 t+5.0  MsgHeartbeat{term 7} ─────────────────────────────────▶ m3
        m3 sees term 7 == its own term (PreVote kept it at 7), accepts m1
        as leader, replies MsgAppResp with its lagging index 42.
 t+5.1  m1 sends MsgApp for entries 43..99.
        If 43 has already been discarded from m1's in-memory Raft log
        (compacted after a snapshot — see --snapshot-count below), m1
        instead sends MsgSnap: the whole bbolt state.  That is expensive
        and is why --snapshot-catchup-entries exists.
 t+5.3  m3 caught up. Cluster is 3/3 again.
```

The row to memorise: **a minority partition does not serve stale writes and does not elect a rival
leader; it stops.** Linearizable reads on the minority side block because etcd's `ReadIndex`
implementation requires the leader to confirm its leadership with a quorum before answering. That
is a correctness guarantee, and it is also why "etcd is up but `kubectl get` hangs" is a real
symptom of a partition rather than a contradiction.

**Quorum arithmetic**, which falls straight out of "a commit needs a majority":

| Members N | Quorum ⌊N/2⌋+1 | Failures tolerated N−quorum | Verdict |
|---|---|---|---|
| 1 | 1 | 0 | dev only; any restart is an outage |
| 2 | 2 | 0 | strictly worse than 1 — twice the failure surface, no tolerance |
| 3 | 2 | 1 | the standard |
| 4 | 3 | 1 | same tolerance as 3, more machines, slower writes |
| 5 | 3 | 2 | large or critical clusters; survives a failure *during* maintenance |
| 6 | 4 | 2 | same as 5, worse |
| 7 | 4 | 3 | write latency starts to hurt |

Even counts are pure downside: you pay for another machine, another `fsync` in the critical path,
and another thing that can fail, and you tolerate no more failures than the odd count below you.

The write-latency cost is not just "more members": a commit needs the **quorum-th fastest**
member. At N=3 the leader waits for the faster of the two followers; at N=5 it waits for the
second-fastest of four. That is more forgiving than it sounds — one slow member out of five is
invisible — which is the real argument for 5 on a big fleet: **it lets you take a member down for
maintenance and still tolerate an unplanned failure.** At N=3, a planned reboot means you are
running at zero fault tolerance until it comes back.

### The on-disk layout, and the four things called "snapshot" or "compaction"

This is the diagram to internalise, because every operational decision in this lesson is an
operation on one specific box in it.

```
  /var/lib/etcd/                                   (kubeadm's --data-dir)
  └── member/
      ├── wal/                                     WRITE-AHEAD LOG — the durability record
      │   ├── 0000000000000005-000000000004e2c1.wal   64 MB, PREALLOCATED (SegmentSizeBytes
      │   ├── 0000000000000006-0000000000061a80.wal   = 64*1000*1000), fsync'd on every commit
      │   └── ...                                     kept: --max-wals (default 5) beyond the
      │                                               last snapshot; older ones are purged
      │
      ├── snap/                                    RAFT SNAPSHOTS — a checkpoint of Raft state
      │   ├── 0000000000000006-00000000000186a0.snap  written every --snapshot-count applied
      │   ├── ...                                     entries (kubeadm sets 10000); lets etcd
      │   │                                           purge WAL segments before it and lets a
      │   │                                           far-behind follower be caught up in one
      │   │                                           MsgSnap.  kept: --max-snapshots (5)
      │   │
      │   └── db  ────────────────────────────────  THE BACKEND: one bbolt (B+tree) file, mmap'd.
      │           Buckets: "key" (revision → KeyValue), "meta" (consistent_index,
      │           confState, storage version), "lease", "auth", "members", …
      │           THIS is what --quota-backend-bytes measures.
      │           THIS is what `etcdctl snapshot save` copies.
      │
      └── (that's it — everything else is derived)
```

Now the four operations, which people merge into two because they share words:

| Operation | Acts on | Removes | Shrinks the `db` file? | Blocking? | Who triggers it |
|---|---|---|---|---|---|
| **MVCC compaction** (`etcdctl compact <rev>`, `--auto-compaction-*`) | the `key` bucket inside `db` | superseded key versions and tombstones below `<rev>` | **No** — freed bbolt pages go on the free list | Brief pauses only | the apiserver, every 5 min, by default |
| **Defragmentation** (`etcdctl defrag`, `etcdutl defrag --data-dir`) | the whole `db` file | nothing logical — it rewrites the B+tree densely | **Yes** — returns free pages to the filesystem | **Yes, fully blocks that member** | you, manually |
| **Raft snapshot** (`--snapshot-count`) | `snap/*.snap` + the in-memory Raft log | old Raft log entries and the WAL segments before it | No | No | etcd, automatically |
| **`etcdctl snapshot save`** | produces a *new file elsewhere* | nothing | No | No (streams over gRPC) | your backup cron |

The mnemonic that actually survives an interview: **compaction frees space *inside* the file;
defragmentation gives space *back to the disk*; a Raft snapshot truncates the *log*, not the data;
and `snapshot save` is a backup, not any of the above.**

Here is the same thing as a picture of the file, which makes the "why doesn't the file shrink"
question answer itself:

```
  db file, 1.9 GiB physical, right after a burst of Lease churn
  ┌────────────────────────────────────────────────────────────────────────┐
  │ live │ old │ live │ old │ old │ live │ tombstone │ old │ live │ old   │
  └────────────────────────────────────────────────────────────────────────┘
   DbSize      = 1.9 GiB  (what the QUOTA is checked against)
   DbSizeInUse = 1.9 GiB  (nothing compacted yet)

                    ── etcdctl compact <rev> ──▶

  ┌────────────────────────────────────────────────────────────────────────┐
  │ live │ FREE│ live │ FREE│ FREE│ live │   FREE    │ FREE│ live │ FREE  │
  └────────────────────────────────────────────────────────────────────────┘
   DbSize      = 1.9 GiB  ← UNCHANGED. Still over quota. Still read-only.
   DbSizeInUse = 0.4 GiB  ← the logical content shrank
   free pages are reusable by etcd, but are NOT returned to the filesystem

                    ── etcdctl defrag (one member) ──▶
       (writes a fresh db.tmp.* next to it, copies live pages into it in
        batches of 10 000 keys, closes both, rename(2)s over the original,
        reopens.  Holds the batch-tx lock AND the read-tx lock the whole
        time, so this member answers NOTHING until it finishes.)

  ┌────────────────────┐
  │ live │ live │ live │
  └────────────────────┘
   DbSize      = 0.4 GiB  ← NOW under quota
   DbSizeInUse = 0.4 GiB
```

etcd 3.6 shows you all of this directly. `endpoint status -w table` gained four columns over 3.5 —
**storage version, in use, percentage not in use, and quota** — so you no longer have to compute
fragmentation by hand:

```console
$ etcdctl --endpoints=https://10.10.0.11:2379,https://10.10.0.12:2379,https://10.10.0.13:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    endpoint status -w table
+--------------------------+------------------+---------+-----------------+---------+--------+----------------------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT         |        ID        | VERSION | STORAGE VERSION | DB SIZE | IN USE | PERCENTAGE NOT IN USE|  QUOTA  | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------------+------------------+---------+-----------------+---------+--------+----------------------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://10.10.0.11:2379  | 91bc3c398fb3c146 |  3.6.8  |      3.6.0      | 1.9 GB  | 402 MB |                  78% | 2.1 GB  |      true |      false |         7 |    4812993 |            4812993 |        |
| https://10.10.0.12:2379  | fd422379fda50e48 |  3.6.8  |      3.6.0      | 1.9 GB  | 402 MB |                  78% | 2.1 GB  |     false |      false |         7 |    4812993 |            4812993 |        |
| https://10.10.0.13:2379  | 8211f1d0f64f3269 |  3.6.8  |      3.6.0      | 411 MB  | 402 MB |                   2% | 2.1 GB  |     false |      false |         7 |    4812993 |            4812993 |        |
+--------------------------+------------------+---------+-----------------+---------+--------+----------------------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

Read that line by line, because it is a complete diagnosis:

- `DB SIZE` 1.9 GB against a `QUOTA` of 2.1 GB (`humanize` renders 2 GiB as "2.1 GB") on two
  members: **you are at 90% of quota and about to go read-only.**
- `IN USE` 402 MB on all three: the *logical* data is small. Compaction has already happened.
  Space is not the problem — fragmentation is.
- `PERCENTAGE NOT IN USE` 78%: computed as `100 − (in_use × 100 ÷ db_size)`. Three-quarters of
  the file is free pages waiting for a defrag.
- `10.10.0.13` at 411 MB and 2%: **that member has already been defragmented.** Defrag is
  per-member and per-file; it is not replicated. This row is what a half-finished maintenance
  window looks like.
- `RAFT INDEX` identical across all three, `ERRORS` empty: replication is healthy and no alarm is
  set. If an alarm were active, `ERRORS` would carry it (the `Status` RPC appends
  `alarm.String()` and `etcdserver: no leader` into that column).

On 3.5, this table has only `endpoint, ID, version, db size, is leader, is learner, raft term,
raft index, raft applied index, errors` — no `in use`, no `quota`. If your runbook says "compare
db size to db size in use," it was written for `-w json` or for 3.6.

### The space quota: how it trips, mechanically

**The problem.** bbolt is memory-mapped and etcd keeps the whole keyspace addressable. An
unbounded database means unbounded memory, unbounded startup time, and an unbounded restore time
during the exact incident where you can least afford it. So etcd enforces a hard ceiling.

**The constants** (`server/storage/quota.go`, etcd 3.6):

```go
DefaultQuotaBytes = int64(2 * 1024 * 1024 * 1024)   // 2 GiB  — used when --quota-backend-bytes is 0/unset
MaxQuotaBytes     = int64(8 * 1024 * 1024 * 1024)   // 8 GiB  — a WARNING threshold, not a hard cap
const leaseOverhead = 64
const kvOverhead    = 256
```

**The cost function**, verbatim in structure:

```go
func costPut(r *pb.PutRequest) int { return kvOverhead + len(r.Key) + len(r.Value) }
// a Txn costs the larger of its success and failure branches
// a LeaseGrant costs leaseOverhead (64)

func (b *BackendQuota) Available(v any) bool {
    cost := b.Cost(v)
    if cost == 0 { return true }                       // pure reads and deletes are free
    return b.be.Size()+int64(cost) < b.maxBackendBytes  // <-- be.Size() is the PHYSICAL FILE SIZE
}
```

Two lines in there are the whole lesson:

1. **`cost == 0` passes unconditionally.** `Cost()` only has cases for `PutRequest`, `TxnRequest`
   and `LeaseGrantRequest`. A `DeleteRange` has no cost, so **deletes are never blocked by the
   quota.** That is deliberate and it is your escape hatch.
2. **`b.be.Size()` is the physical file size, not `SizeInUse`.** The quota is checked against how
   big the file is on disk, including free pages. This is precisely why the recovery order is
   compact-*then*-defrag: compaction alone moves `SizeInUse` and leaves `Size` untouched, so the
   very next write re-trips the alarm.

Three flag behaviours worth knowing (`NewBackendQuota`):

| `--quota-backend-bytes` | Behaviour |
|---|---|
| `0` (unset, the kubeadm default) | 2 GiB, logged as `enabled backend quota with default value` |
| negative | **quota disabled entirely** (`passthroughQuota`) — every check returns true |
| > 8 GiB | still applied, but logged: `quota exceeds the maximum value` with `quota-maximum-size-bytes: 8589934592` |

That last row is the nuance most people get wrong in both directions. 8 GiB is **not** a hard
technical wall — etcd will run with a larger quota and simply warn. It is a recommendation whose
justification is recovery time: a bigger database takes longer to snapshot, longer to transfer to
a lagging follower as a `MsgSnap`, longer to `defrag`, and longer to restore. Raising it is a
legitimate engineering decision *if you write down the MTTR you are trading away*. Setting it
negative to "make the problem go away" is not; you have removed the only thing standing between a
leaking controller and an unbootable etcd.

### When the alarm trips: exactly what stops working

The alarm is raised in one of two places — the gRPC preflight (`quota.go`) or the apply path
(`server.go`, `"message exceeded backend quota; raising alarm"`) — and is then **replicated
through Raft**, so it is cluster-wide and survives a leader change. It does **not** clear itself,
even if space is freed. That is deliberate: an alarm that flapped would let a runaway workload
oscillate a cluster in and out of read-only.

What the alarm does is swap in a different applier. From
`server/etcdserver/apply/uber_applier.go`:

```go
a.applyV3 = a.applyV3base
if noSpaceAlarms { a.applyV3 = newApplierV3Capped(a.applyV3) }
if corruptAlarms { a.applyV3 = newApplierV3Corrupt(a.applyV3) }
```

And those two appliers are very different animals:

| Request | Normal | Under **NOSPACE** (`applierV3Capped`) | Under **CORRUPT** (`applierV3Corrupt`) |
|---|---|---|---|
| `Range` (read) | ✅ | ✅ **works** | ❌ `ErrCorrupt` |
| `Put` | ✅ | ❌ `ErrNoSpace` | ❌ `ErrCorrupt` |
| `DeleteRange` | ✅ | ✅ **works** | ❌ `ErrCorrupt` |
| `Txn` | ✅ | ❌ if `Cost(txn) > 0`, else ✅ | ❌ `ErrCorrupt` |
| `Compaction` | ✅ | ✅ **works** | ❌ `ErrCorrupt` |
| `LeaseGrant` | ✅ | ❌ `ErrNoSpace` | ❌ `ErrCorrupt` |
| `LeaseRevoke` | ✅ | ✅ | ❌ `ErrCorrupt` |

**Under NOSPACE you can still read, delete, and compact.** That is not an accident — it is exactly
the set of operations you need to dig yourself out, and it is why the recovery procedure works at
all. **Under CORRUPT even reads fail**, because the whole premise of a corruption alarm is that
you cannot trust what you would read.

The wire errors, from `api/v3rpc/rpctypes/error.go`:

```
NOSPACE : codes.ResourceExhausted, "etcdserver: mvcc: database space exceeded"
CORRUPT : codes.DataLoss,          "etcdserver: corrupt cluster"
```

And here is what a NOSPACE-frozen cluster looks like from every angle at once:

```console
$ etcdctl alarm list
memberID:10501334649042878790 alarm:NOSPACE
memberID:18248076129603518024 alarm:NOSPACE

$ etcdctl put /probe hello
{"level":"warn","msg":"retrying of unary invoker failed",
 "error":"etcdserver: mvcc: database space exceeded"}
Error: etcdserver: mvcc: database space exceeded

$ kubectl create deployment probe --image=nginx
error: failed to create deployment: etcdserver: mvcc: database space exceeded

$ kubectl get nodes            # reads still work — this is why it looks fine on a dashboard
NAME   STATUS   ROLES           AGE   VERSION
cp1    Ready    control-plane   41d   v1.36.1
...

$ kubectl get nodes -o wide --watch   # …until node leases stop renewing
# ~40s later, nodes begin flipping to NotReady: the kubelet's Lease update is a Put,
# and Puts are what the alarm blocks.
```

That last step is the part people do not anticipate. **A NOSPACE alarm does not stay a "can't
deploy" problem.** Node heartbeats are Lease writes; when they fail, node-lifecycle-controller
starts marking nodes `NotReady` after `--node-monitor-grace-period` (40 s by default), and once it
would evict pods you are one step from a self-inflicted mass eviction — except that the eviction
writes also fail, so the cluster instead sits in a frozen, half-degraded state until you fix it.
On a GPU fleet this is a good outcome disguised as a bad one: nothing gets rescheduled, so nothing
disturbs your running training jobs. Kubelets keep running the pods they already know about;
containerd does not care that the API is frozen.

### The recovery procedure, in order, with the reason for each step

Every step in this order exists because the step before it is a precondition. Run it out of order
and you will re-trip the alarm and lose the time you were trying to save.

```bash
# 0. CONFIRM it is actually NOSPACE, not lost quorum and not a slow disk.
etcdctl alarm list                              # -> memberID:… alarm:NOSPACE
etcdctl endpoint status --cluster -w table      # -> DB SIZE vs QUOTA vs IN USE, per member

# 1. STOP THE BLEEDING (optional but usually right).
#    Find what is writing. Under NOSPACE, reads and deletes still work, so you CAN
#    delete the offender's objects even while the cluster is read-only.
etcdctl get /registry/ --prefix --keys-only | sed -n 's|^/registry/\([^/]*\)/.*|\1|p' \
  | sort | uniq -c | sort -rn | head
#    Classic culprits: events (unbounded), leases, a controller hot-looping on status.
etcdctl del /registry/events/ --prefix          # allowed: DeleteRange has zero quota cost

# 2. COMPACT to the current revision. Frees the superseded history INSIDE the file.
rev=$(etcdctl endpoint status -w json | jq -r '.[0].Status.header.revision')
etcdctl compact "$rev" --physical
#    --physical waits until old revisions are physically removed from the backend,
#    so you know step 3 will actually reclaim something.

# 3. DEFRAG, ONE MEMBER AT A TIME, followers first, leader last.
#    Each defrag fully blocks that member: no reads, no writes, no heartbeat responses.
for ep in https://10.10.0.13:2379 https://10.10.0.12:2379 https://10.10.0.11:2379; do
  etcdctl --endpoints="$ep" defrag
  etcdctl --endpoints="$ep" endpoint health     # wait for healthy before moving on
done

# 4. DISARM. Only now — the alarm is sticky and will not clear itself.
etcdctl alarm disarm
etcdctl alarm list                              # -> (empty)

# 5. VERIFY the control plane is writable again.
kubectl create configmap nospace-probe --from-literal=t=$(date +%s)
kubectl delete configmap nospace-probe
```

Why the order is not negotiable:

- **Delete before compact**: compaction only drops versions that are *already superseded*. If the
  offending keys are still live, compaction cannot touch them. Deleting first turns them into
  tombstones that the compaction can then remove.
- **Compact before defrag**: defragmentation rewrites whatever is logically present. Defragging
  before compacting faithfully rewrites all your garbage into a tightly-packed file of the same
  size, and you will have taken the outage for nothing.
- **Defrag before disarm**: the quota is checked against the *physical file size*. Disarming a
  1.9 GiB file under a 2 GiB quota buys you a handful of writes before the alarm re-arms.
- **Followers before the leader**: defrag blocks the member completely, including its Raft
  responses. Blocking a follower costs you nothing at N=3 (the leader still has itself plus the
  other follower). Blocking the leader stalls every write for the duration and usually triggers an
  election. If you must defrag the leader, move leadership off it first:
  `etcdctl move-leader <target-member-hex-id>`.
- **Never in parallel**: two blocked members out of three is a lost quorum, self-inflicted, in the
  middle of an incident.

**How long will the defrag take?** It is proportional to the *live* data, not the file size,
because `defragdb()` copies live keys into a fresh bbolt file in batches of 10 000
(`defragLimit`). The metric `etcd_disk_backend_defrag_duration_seconds` has buckets starting at
0.1 s and the source comment records the rule of thumb the buckets were sized from: **~1 second
per 100 MB.** So 400 MB of live data ≈ 4 s per member on decent NVMe, and a whole 3-member cluster
is under a minute of sequential, per-member blocking. Watch `etcd_disk_defrag_inflight` (1 while
running) if you automate it.

### CORRUPT is a different animal

`CORRUPT` is raised by etcd's corruption detection, not by you. Two mechanisms:

- **Initial corrupt check** — kubeadm turns this on explicitly with
  `--feature-gates=InitialCorruptCheck=true` in the etcd static pod. On startup a member compares
  its hash with its peers and refuses to join a cluster it disagrees with, rather than silently
  serving divergent data.
- **Periodic compact-hash check** — `DefaultCompactHashCheckTime = 1 minute`; members compare the
  hash of their state at the last compacted revision.

If it fires, **do not `alarm disarm`**. A CORRUPT alarm means two members have genuinely different
data at the same revision, which is a correctness failure, not a capacity one. The procedure is:
identify the diverging member with `etcdctl endpoint hashkv --cluster` (identical `HASH` values at
the same `HASH_REVISION` mean agreement), remove it from the cluster
(`etcdctl member remove <id>`), wipe its data directory, and re-add it as a **learner** so it
re-syncs from scratch. Disarming instead just re-enables writes on a cluster whose members
disagree about what is in it.

The canonical instance of this failure class is etcd's own **v3.5 data-inconsistency
postmortem** (`Documentation/postmortems/v3.5-data-inconsistency.md` in the etcd repo): a bug in
how `consistent_index` was persisted meant that a member which crashed at the wrong moment could
replay entries it had already applied, producing durable divergence between members that nothing
detected until much later. The fixes were both engineering and operational — correcting the index
handling, and making corruption checking a first-class, on-by-default feature. That is why
kubeadm sets `InitialCorruptCheck` for you, and why "etcd said CORRUPT" is a sentence that should
make you stop and think rather than reach for `disarm`.

### Worked math: database growth and time-to-quota

You cannot size etcd by feel. Here is the model, then two runs of it.

**The model.** Every mutating request adds `256 + len(key) + len(value)` bytes of *new* content to
the backend (the `kvOverhead` constant plus the encoded key and value). Superseded versions stay
until compaction. So:

```
  steady-state DbSizeInUse  ≈  live_bytes  +  write_rate × cost_per_write × compaction_window
  DbSize (physical)         ≈  high-water mark of the above, since bbolt never returns pages
                               on its own
  time_to_quota (if compaction stops)
                            ≈  (quota − current DbSize) ÷ (write_rate × cost_per_write)
```

**Run 1: steady state for a 400-node GPU fleet.** Assume 8 pods per node and typical serialised
sizes (measure yours: `etcdctl get <key> --print-value-only | wc -c`).

| Resource | Count | Avg cost/object | Live bytes |
|---|---|---|---|
| Pods | 3 200 | 256 + 45 + 6 000 ≈ 6.3 KiB | 20.2 MiB |
| Nodes (status is large) | 400 | 256 + 40 + 12 000 ≈ 12.0 KiB | 4.8 MiB |
| Leases (one per node, plus CP) | 410 | 256 + 55 + 400 ≈ 0.7 KiB | 0.3 MiB |
| EndpointSlices | 600 | 256 + 60 + 2 500 ≈ 2.7 KiB | 1.6 MiB |
| ConfigMaps / Secrets | 1 200 | 256 + 55 + 3 000 ≈ 3.2 KiB | 3.8 MiB |
| Events (1 h retention) | 20 000 | 256 + 90 + 700 ≈ 1.0 KiB | 20.5 MiB |
| **live_bytes** | | | **≈ 51 MiB** |

Now the churn term. Node Leases dominate, because every kubelet renews every 10 s by default:

```
  lease renewals  = 400 nodes ÷ 10 s               =  40 writes/s
  cost per renewal ≈ 256 + 55 + 400                =  711 B
  lease churn                                       =  28.4 KiB/s

  node status updates (roughly every 10 s under normal conditions)
                   = 400 ÷ 10 s = 40 writes/s × 12.3 KiB  = 492 KiB/s   <-- the real monster
  pod status, events, endpointslices, misc          ≈  60 KiB/s
  ──────────────────────────────────────────────────────────────────
  total write throughput                            ≈ 580 KiB/s
```

With the apiserver compacting every 5 minutes, the retained history is roughly one compaction
window plus a margin:

```
  history ≈ 580 KiB/s × 300 s ≈ 170 MiB
  DbSizeInUse ≈ 51 MiB + 170 MiB ≈ 221 MiB
  DbSize (physical) settles at the high-water mark, typically 1.5–3× that after a few
  churn cycles → expect 350–650 MiB and plan a defrag when it exceeds ~50% not-in-use.
```

**221 MiB of logical data against a 2 GiB quota.** A 400-node fleet is comfortable — which is the
correct and reassuring answer, and it tells you that if your etcd is near quota, *something is
wrong*, and raising the quota is treating a symptom.

**Run 2: what happens when compaction stops.** This is the failure that produces the 2am page. The
apiserver's compactor can stop for real reasons: every apiserver restarting in a loop, someone
setting `--etcd-compaction-interval=0` (which disables it), or an operator disabling etcd's
own auto-compaction on the assumption that "Kubernetes handles it" while also running standalone
etcd. Add one hot-looping controller — a genuinely common failure, e.g. an operator writing status
on every reconcile and triggering its own watch — and you get:

```
  hot-loop write rate                    = 200 writes/s
  object size                            = 3 KiB
  cost per write = 256 + 60 + 3072       = 3 388 B
  growth from the hot loop               = 200 × 3388  = 677.6 KB/s
  plus the 580 KiB/s baseline (≈594 KB/s)              = 1 271 KB/s ≈ 1.27 MB/s

  headroom = 2 GiB − 400 MB current      = 2 147 483 648 − 400 000 000 ≈ 1.747 GB

  time_to_quota = 1.747e9 B ÷ 1.271e6 B/s ≈ 1 375 s ≈ 23 minutes
```

**Twenty-three minutes from "compaction stopped" to "the cluster is read-only."** Without the hot
loop, at the 594 KB/s baseline alone, it is `1.747e9 ÷ 5.94e5 ≈ 2 941 s ≈ 49 minutes`. Either way
it is far too fast for a daily capacity review to catch, and it is why the alert you actually want
is not "DB size > 70% of quota" but **the derivative**: `etcd_debugging_mvcc_compact_revision`
not advancing, or `predict_linear(etcd_mvcc_db_total_size_in_bytes[15m], 3600) > quota`.

Re-run both with your own numbers. The two inputs worth measuring rather than assuming are the
**average serialised object size** for your top three resource types, and the **actual write rate**
(`rate(etcd_mvcc_put_total[5m]) + rate(etcd_mvcc_txn_total[5m])`).

### Compaction in a Kubernetes cluster: who is actually doing it

There are two independent compactors, and running both or neither are both mistakes.

**The apiserver's compactor** (`k8s.io/apiserver/pkg/storage/etcd3/compact.go`) is the one that is
on by default. Its algorithm is more interesting than "compact every 5 minutes," and the detail
explains the history window you actually get:

- Every `--etcd-compaction-interval` (default **5m0s**), each apiserver wakes up.
- It runs a **compare-and-swap transaction** on a special key, `compact_rev_key`: "if your
  `Version` is still the number I last saw, set your value to the revision I observed one interval
  ago." Exactly one apiserver in an HA set wins per interval; the losers read the winner's value,
  update their local view, and try again next interval.
- Because it compacts to *the revision it saw one interval ago*, the retained history is one full
  interval, not zero. The source comment states the guarantee precisely: **"in normal cases, the
  interval is 5 minutes; in failover, the interval is >5m and <10m."**

That is the mechanical origin of "you get about five minutes of watch history," and therefore of
the `410 Gone: too old resource version` behaviour that informers handle for you. Setting
`--etcd-compaction-interval=0` disables it entirely — and now nothing compacts unless you have
configured etcd itself to.

**etcd's own auto-compactor** (`server/etcdserver/api/v3compactor/`) is **off by default under
kubeadm**, because `DefaultAutoCompactionRetention = "0"` means disabled:

| Flag | Default | Behaviour |
|---|---|---|
| `--auto-compaction-mode` | `periodic` | `periodic` = retain a duration of history; `revision` = retain a number of revisions |
| `--auto-compaction-retention` | `"0"` | `0` disables it. `periodic` accepts `1h`, `24h`, or a bare number meaning hours. `revision` accepts a count, e.g. `1000` |

The two modes work differently and it matters:

- **`revision` mode** runs on a fixed 5-minute timer (`revInterval`) and compacts to
  `currentRevision − retention`.
- **`periodic` mode** samples the current revision every `period ÷ 10` (that is `retryDivisor`,
  capped so the sampling interval never exceeds 6 minutes for periods over an hour), keeps a
  sliding window of those samples, and compacts to the oldest one — i.e. to the revision as of
  `period` ago. It deliberately does nothing at all for the first full period after startup.

**Configure exactly one of the two.** If the apiserver compacts every 5 minutes and etcd is also
set to `--auto-compaction-retention=1h`, the apiserver's aggressive compaction wins and etcd's
does nothing — harmless but misleading. If you disable the apiserver's compactor and forget to
enable etcd's, you have built Run 2 above. For a stacked kubeadm cluster the right answer is
usually: leave the apiserver's default alone, and set etcd's `--auto-compaction-mode=revision
--auto-compaction-retention=1000` as a **backstop** so that a control plane whose apiservers are
all crash-looping still cannot fill its own disk.

### Why etcd hates slow disks — the causal chain, not the slogan

Every Raft commit requires an `fsync` to the WAL before it can be acknowledged. That single
constraint propagates all the way up:

```
  slow fsync
     │  WAL append blocks
     ▼
  follower MsgAppResp is late
     │  leader cannot reach quorum on this entry
     ▼
  commitIndex stalls
     │  apiserver's etcd call exceeds its deadline
     ▼
  apiserver returns 500 / "context deadline exceeded"
     │  AND: the leader's own apply loop is behind, so its heartbeats go out late
     ▼
  a follower's randomized election timeout (1000–2000 ms) fires
     │
     ▼
  election → new leader → all in-flight proposals are lost and retried
     │  (etcd_server_leader_changes_seen_total increments)
     ▼
  every controller's watch reconnects, every client retries, load spikes,
  the new leader is now behind on the same slow disk → repeat
```

This is why the failure looks like a *cascade* rather than a slowdown: past a threshold, latency
turns into election churn, and election churn multiplies the load that caused it.

The metrics, with their real Prometheus names and where the thresholds come from:

| Signal | Metric | Threshold | Why |
|---|---|---|---|
| WAL fsync latency | `etcd_disk_wal_fsync_duration_seconds` | p99 sustained **> 10 ms** | Histogram buckets are exponential from **1 ms** (`ExponentialBuckets(0.001, 2, 14)`, top bucket 8.192 s), i.e. the design point is single-digit milliseconds. Above ~10 ms you are inside the election-timeout budget. |
| Backend commit latency | `etcd_disk_backend_commit_duration_seconds` | p99 sustained **> 25 ms** | Same bucket family; this is the batched bbolt commit, expected to be coarser than fsync. |
| Election churn | `etcd_server_leader_changes_seen_total` | any sustained rate | Should be flat for weeks. |
| No leader | `etcd_server_has_leader` | `== 0` | Quorum lost or partitioned. Page immediately. |
| Physical DB size | `etcd_mvcc_db_total_size_in_bytes` | > 70% of quota, or rising trend | The quota is checked against this. |
| Logical DB size | `etcd_mvcc_db_total_size_in_use_in_bytes` | compare to the above | The gap is your fragmentation, i.e. your defrag signal. |
| Compaction alive | `etcd_debugging_mvcc_compact_revision` | must keep advancing | A flat line here is Run 2 starting. |
| Proposal failures | `etcd_server_proposals_failed_total` | rising | Proposals dropped, usually during elections. |
| Defrag running | `etcd_disk_defrag_inflight` | `== 1` | That member is blocked right now. |
| Slow apply warnings | log line `apply request took too long` | — | Emitted past `--warning-apply-duration`, default **100 ms**. Grep for it; it is the cheapest early warning there is. |

The fix is almost always physical: **a dedicated low-latency NVMe for `--data-dir`, not shared
with container images, container logs, or anything else that does bulk I/O.** etcd's write pattern
is small, sequential, latency-bound and durability-bound — the exact opposite of what a
throughput-optimised shared volume is good at. If you genuinely cannot move it, raising
`--heartbeat-interval` and `--election-timeout` (keeping the ≥5× ratio, e.g. 500 ms / 5000 ms)
buys tolerance at the cost of slower failure detection. That is a last resort, not a tuning knob.

On a GPU fleet the economics are not close: a few hundred dollars of enterprise NVMe against an
hour of a 64-H100 fleet sitting idle because nothing can schedule. Put that line in the 10.8 model.

### The tuning surface, with real defaults

Everything you might set, with the value etcd uses if you do not, read from
`server/embed/config.go` and `server/etcdserver/server.go` (3.6):

| Flag | Default | What it controls | When to change it |
|---|---|---|---|
| `--quota-backend-bytes` | `0` → 2 GiB | backend size ceiling; negative disables | raise deliberately with an MTTR budget; warns above 8 GiB |
| `--auto-compaction-mode` | `periodic` | `periodic` \| `revision` | set with retention if etcd must self-compact |
| `--auto-compaction-retention` | `"0"` (disabled) | history to keep | `1000` in `revision` mode as a backstop under Kubernetes |
| `--heartbeat-interval` | `100` ms | Raft tick | raise on high-RTT or slow-disk clusters |
| `--election-timeout` | `1000` ms | election trigger | must be ≥ 5× heartbeat, ≤ 50 000 ms |
| `--snapshot-count` | `10000` | applied entries between Raft snapshots (kubeadm sets this explicitly, matching the default) | lower = more frequent snapshots, shorter WAL replay on restart, more I/O |
| `--snapshot-catchup-entries` | `5000` | Raft log entries kept after a snapshot so a lagging follower can catch up without a full `MsgSnap` | raise if followers frequently need full snapshots |
| `--max-wals` | `5` | WAL segment files retained | rarely |
| `--max-snapshots` | `5` | `.snap` files retained | rarely |
| `--max-request-bytes` | `1.5 MiB` | largest single request | this is the real reason a huge ConfigMap fails with `etcdserver: request is too large` |
| `--max-txn-ops` | `128` | operations per transaction | rarely |
| `--warning-apply-duration` | `100 ms` | threshold for the `apply request took too long` log | lower it temporarily when hunting slow requests |
| `--pre-vote` | `true` | PreVote algorithm | leave on |
| `--listen-metrics-urls` | (unset; kubeadm sets `http://127.0.0.1:2381`) | plain-HTTP metrics/health endpoint | needed for probes and Prometheus without client certs |
| apiserver `--etcd-compaction-interval` | `5m0s` | apiserver-driven compaction; `0` disables | almost never; if you do, enable etcd's own |

### Members, learners, and safe reconfiguration

Adding a voting member to a busy cluster is riskier than it looks. The new member starts empty; it
must receive the entire state (potentially a multi-hundred-megabyte `MsgSnap`) before it can
usefully vote — but it counts toward quorum from the moment it is added. Go from 3 members to 4
and quorum jumps from 2 to 3, so the cluster now depends on a member that is still syncing.

**Learners** exist for exactly this. A learner receives the Raft log but does not vote and does not
count toward quorum:

```console
$ etcdctl member add cp4 --learner --peer-urls=https://10.10.0.14:2380
Member 9bf4f2b3d0a1c877 added as learner to cluster ef37ad9dc622a7c4

ETCD_NAME="cp4"
ETCD_INITIAL_CLUSTER="cp1=https://10.10.0.11:2380,cp2=https://10.10.0.12:2380,cp3=https://10.10.0.13:2380,cp4=https://10.10.0.14:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
# ^ paste these into the new member's config. --initial-cluster-state=existing is
#   mandatory: "new" would make it try to bootstrap a fresh cluster.

$ etcdctl member list -w table
+------------------+---------+------+--------------------------+--------------------------+------------+
|        ID        | STATUS  | NAME |        PEER ADDRS        |       CLIENT ADDRS       | IS LEARNER |
+------------------+---------+------+--------------------------+--------------------------+------------+
| 8211f1d0f64f3269 | started | cp3  | https://10.10.0.13:2380  | https://10.10.0.13:2379  |      false |
| 91bc3c398fb3c146 | started | cp1  | https://10.10.0.11:2380  | https://10.10.0.11:2379  |      false |
| 9bf4f2b3d0a1c877 | started | cp4  | https://10.10.0.14:2380  | https://10.10.0.14:2379  |       true |
| fd422379fda50e48 | started | cp2  | https://10.10.0.12:2380  | https://10.10.0.12:2379  |      false |
+------------------+---------+------+--------------------------+--------------------------+------------+

# Wait until the learner's raft applied index has caught up, then:
$ etcdctl member promote 9bf4f2b3d0a1c877
Member 9bf4f2b3d0a1c877 promoted in cluster ef37ad9dc622a7c4
```

`member promote` **refuses** if the learner is too far behind — the server checks its progress
against the leader before allowing the promotion, which is exactly the safety property you want.
Cluster API's `KubeadmControlPlane` builds on this: its preflight checks explicitly refuse to
scale when "there are etcd members still in learner mode," and when it deletes a control-plane
machine it first moves etcd leadership off that member (`ForwardEtcdLeadership`) and then removes
the member — the sequence you would run by hand. Lesson 10.4 comes back to that.

Two more rules for reconfiguration:

- **Remove before you destroy.** If you are decommissioning a node, run `etcdctl member remove
  <id>` *first*, while the member is still healthy. Powering it off and removing it later means
  running at reduced quorum in between.
- **`--strict-reconfig-check` is on by default** (`DefaultStrictReconfigCheck = true`) and refuses
  reconfigurations that would immediately break quorum. Do not turn it off to "make the command
  work."

### Backup: what `snapshot save` actually captures

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%F-%H%M).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

Mechanically, this opens a gRPC `Snapshot` stream against **one specific member** and streams that
member's bbolt file, with a **SHA-256 digest appended to the end**. Consequences:

- **It is a point-in-time copy of one member.** Specify exactly one endpoint. Pointing it at a
  comma-separated list is meaningless — you get whichever one the client picked.
- **It contains the data, the membership bucket, and the storage version — not the WAL.** Which is
  why restoring it has to synthesise a new Raft history (next section).
- **The appended hash is what `snapshot status` verifies.** A file copied straight out of
  `member/snap/db` has no such hash, which is why restoring one requires `--skip-hash-check`.
- **The revision in the snapshot is your RPO boundary.** Everything written after it is gone.

Verify every snapshot. An unverified backup is a rumour:

```console
$ etcdutl snapshot status /backup/etcd-2026-08-18-0300.db -w table
+----------+----------+------------+------------+---------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE | VERSION |
+----------+----------+------------+------------+---------+
| 3fa1c40e |  4812993 |      61402 |     411 MB |   3.6.0 |
+----------+----------+------------+------------+---------+
```

`snapshot status` does more than print numbers: it opens the file read-only with bbolt and runs
`tx.Check()`, a full structural integrity check of the B+tree, failing with `snapshot file
integrity check failed` if the page structure is damaged. It then walks every bucket to compute
the CRC and counts distinct live keys (skipping tombstones), so `TOTAL KEYS` is the live key
count, not the revision count. `VERSION` is the storage version, and it is empty for snapshots
taken from servers older than 3.6.

Backup policy, not backup command:

- **Cron it, and ship it off the node.** A snapshot sitting on the node whose disk just died is
  not a backup. Object storage, another failure domain, ideally another site.
- **Choose an RPO and state it.** Every-5-minutes means you can lose 5 minutes of object writes.
  For a cluster where the durable state also lives in Git (10.4's GitOps pattern), that is
  usually fine — you lose Leases, Events, and pod status, all of which regenerate. For a cluster
  where etcd holds the only copy of something, it is not.
- **Rehearse the restore on a throwaway VM, on a schedule.** The failure mode of an untested
  restore runbook is that you discover it was written for etcd 3.5 at 02:40 on a Sunday.

### Restore: why it is not "copy the file back"

This is the section that separates people who have done it from people who have read about it.

`etcdutl snapshot restore` builds a **brand-new data directory with a brand-new Raft identity.**
Here is what it actually does, step by step, from `etcdutl/snapshot/v3_snapshot.go`:

```
  INPUT: snapshot.db  (bbolt file + 32-byte SHA-256 tail)
         --name cp1 --initial-cluster "cp1=https://10.10.0.11:2380,cp2=…,cp3=…"
         --initial-advertise-peer-urls https://10.10.0.11:2380
         --initial-cluster-token <token>            (default: "etcd-cluster")
         --data-dir /var/lib/etcd-restored

  1. copyAndVerifyDB()
     • read the last 32 bytes as the expected SHA-256
     • copy the file to <data-dir>/member/snap/db
     • truncate the 32-byte hash tail off the copy
     • re-hash the truncated copy and compare  → mismatch = abort
       (a file with no hash tail is only accepted with --skip-hash-check)

  2. TrimMembershipFromBackend()
     • DELETE the entire members bucket from the copied database.
       The old cluster's membership is discarded, on purpose.

  3. compute a NEW cluster ID and NEW member IDs
     • derived from --initial-cluster-token + the --initial-cluster URL map.
       ── this is why every member must restore with the SAME token and the
          SAME --initial-cluster string, or they will compute different
          cluster IDs and refuse to talk to each other ──

  4. saveWALAndSnap()
     • create a brand-new, EMPTY WAL directory
     • synthesise one ConfChangeAddNode entry per member, all at Term 1,
       Index 1..N
     • write HardState{ Term: 1, Vote: <first member>, Commit: N }
     • write a Raft snapshot at { Index: N, Term: 1 } with the new ConfState

  5. updateCIndex(commit, term)
     • set consistent_index in the backend to match the synthetic Raft state

  RESULT: a data directory whose DATA is from your backup but whose RAFT LOG
          starts at term 1, index N, with new member IDs and a new cluster ID.
```

Every operational rule about restore falls out of that:

- **You cannot restore one member and let replication fix the others.** The restored member has a
  different cluster ID from the surviving ones. They will reject each other with `cluster ID
  mismatch`. Replication is not a repair mechanism here.
- **You restore the *same snapshot file* on *every* member**, each with its own `--name` and
  `--initial-advertise-peer-urls`, but an **identical `--initial-cluster`** and identical
  `--initial-cluster-token`, and then start them together. They independently compute the same
  cluster ID and the same member IDs, agree on the same synthetic Raft history, and form a cluster.
- **Never mix restored and surviving members.** Take everything down first.
- **The revision counter resumes from the snapshot's revision**, which is *lower* than the
  revision clients last saw. Clients that cached a higher `resourceVersion` will see revisions go
  backwards. Kubernetes informers recover (they get a `410 Gone` and re-LIST), but if you need to
  avoid it entirely, `--bump-revision <N> --mark-compacted` exists precisely to fast-forward the
  revision counter past what clients remember. Use it when restoring a cluster with external
  consumers of the revision.
- **`--data-dir` must not already exist (or must be empty)**; restore refuses to overwrite, which
  is a feature.

### Lost quorum: what it takes, and the last resort

Quorum is lost when fewer than ⌊N/2⌋+1 members can reach each other. At N=3 that means **two
members gone**, not one. The symptoms are unambiguous: `etcd_server_has_leader == 0`,
`endpoint status` times out against the dead members, apiserver logs fill with `context deadline
exceeded` talking to etcd, and `kubectl` hangs on both reads and writes.

Your options, best first:

1. **Bring a member back.** Nine times out of ten one of the "dead" members is a machine that
   rebooted, a full disk, or a network flap. A member with an intact data directory rejoins and
   restores quorum with no data loss. Always spend the first five minutes here.
2. **Restore from a snapshot onto all members.** The full procedure below. You lose everything
   after the snapshot revision.
3. **`--force-new-cluster` from a survivor.** The genuine last resort: start one surviving member
   with `--force-new-cluster`, which rewrites its own membership to a single-member cluster (it
   truncates the ConfState to just itself), making it a leader of one immediately. You then
   re-add the others as fresh members. Use it when a member has *more recent data than your
   newest snapshot* and you would rather keep that data than restore. Understand the risk: it
   unilaterally declares one member's view to be the truth, so if that member was behind, you have
   silently rolled back everything the others had committed. Take a snapshot of the survivor's data
   directory before you do it.

### The DR runbook

For a stacked kubeadm control plane where etcd data is gone or corrupt. Print this; do not
improvise it at 02:00.

```bash
# 1. STOP THE APISERVERS on every control-plane node, so nothing writes half-state.
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
crictl ps | grep kube-apiserver          # confirm it is gone

# 2. STOP ETCD on every member. Restore is an offline operation.
mv /etc/kubernetes/manifests/etcd.yaml /tmp/
crictl ps | grep etcd                    # confirm

# 3. MOVE THE OLD DATA ASIDE (do not delete — it may be your best copy).
mv /var/lib/etcd /var/lib/etcd.broken.$(date +%s)

# 4. PICK AND VERIFY THE SNAPSHOT. Note the revision: that is your RPO boundary.
etcdutl snapshot status /backup/etcd-2026-08-18-0300.db -w table

# 5. RESTORE ON EVERY MEMBER — same file, same --initial-cluster, same token,
#    per-member --name and --initial-advertise-peer-urls.
#    On cp1:
etcdutl snapshot restore /backup/etcd-2026-08-18-0300.db \
  --name cp1 \
  --initial-cluster cp1=https://10.10.0.11:2380,cp2=https://10.10.0.12:2380,cp3=https://10.10.0.13:2380 \
  --initial-cluster-token k8s-restore-2026-08-18 \
  --initial-advertise-peer-urls https://10.10.0.11:2380 \
  --data-dir /var/lib/etcd
#    On cp2 and cp3: identical except --name and --initial-advertise-peer-urls.

# 6. START ETCD on all members together and let them form the new cluster.
mv /tmp/etcd.yaml /etc/kubernetes/manifests/
etcdctl endpoint status --cluster -w table    # 3 members, one leader, matching raft index
etcdctl endpoint health --cluster

# 7. START THE APISERVERS.
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
kubectl get nodes
kubectl create configmap dr-probe --from-literal=t=$(date +%s) && kubectl delete cm dr-probe

# 8. RECONCILE THE DRIFT. Everything created after the snapshot revision is gone.
#    Re-apply from GitOps / your manifests. Expect:
#      - pods that existed will be re-created by their controllers
#      - pods created after the snapshot are simply absent; the kubelets still
#        RUNNING them will be told to delete them (they are no longer in the API)
#      - Leases regenerate within ~10s; nodes flip Ready again
#      - Events are gone; that is fine
#    If the data loss involved a stolen data directory or snapshot, ROTATE SECRETS.
```

Two things about step 8 that surprise people the first time. **Kubelets keep running pods the
restored apiserver has never heard of** — the kubelet's local state is authoritative until it is
told otherwise — so for a few seconds after the restore you have containers running that the API
does not know about, and then they are cleaned up. And **static pods survive a restore entirely**,
which is exactly why the control plane can come back at all.

### Triage: three checks, under a minute

The symptom is always the same — `kubectl` hangs or errors, controllers stall — and there are
three very different causes. Check them in this order, because it discriminates fastest:

```
  kubectl is failing / hanging
        │
        ▼
  1. etcdctl alarm list
        │
        ├── NOSPACE ──────▶ writes fail, reads work.
        │                   FIX: delete → compact → defrag (one at a time) → disarm
        │
        ├── CORRUPT ──────▶ reads fail too.  DO NOT DISARM.
        │                   FIX: hashkv to find the diverging member, remove it,
        │                        wipe it, re-add as learner
        │
        └── (empty)
              │
              ▼
  2. etcd_server_has_leader  /  etcdctl endpoint status --cluster
        │
        ├── 0, or endpoints timing out ──▶ QUORUM LOST.
        │                                   FIX: revive a member; else restore;
        │                                   last resort --force-new-cluster
        │
        └── 1, all members present
              │
              ▼
  3. etcd_disk_wal_fsync_duration_seconds p99   +  grep "apply request took too long"
        │
        ├── p99 ≫ 10 ms ──▶ SLOW DISK.
        │                   FIX: move --data-dir to dedicated NVMe; find the
        │                   noisy neighbour; as a stopgap raise heartbeat/election
        │
        └── normal ───────▶ it is not etcd. Look at apiserver APF (429s),
                            admission webhooks timing out, or the network.
```

Memorise the order, not just the checks. `alarm list` is one RPC and gives a definitive yes/no;
`has_leader` is a single gauge; fsync latency requires interpreting a histogram. Cheapest and most
decisive first.

## Perspectives

**Developer / app-owner view.** From inside a Deployment, a NOSPACE-frozen cluster looks like
`kubectl apply` hanging or erroring with a message about "mvcc" that means nothing to them. They
open a "deploy is stuck" ticket with no idea the cause is two hops away in the datastore. Part of
running this well is making the alarm and quorum state visible on the platform status page
*before* it reaches that ticket — a single panel with `etcd_server_has_leader`, the alarm count,
and DB size against quota answers 90% of those tickets pre-emptively.

**Operator / on-call view.** This lesson is written for the person holding the pager. The triage
order (`alarm list` → `has_leader` → `wal_fsync`) exists because under time pressure you do not
get to read every metric; you need one ordered checklist that discriminates three failure classes
in under a minute. Everything else — the snapshot cron, the verified restores, the defrag job — is
work you do on a Tuesday so that Sunday is boring.

**Hardware / disk view.** Every operational property in this lesson traces to one physical fact:
`fsync` on a contended or spinning disk takes milliseconds to tens of milliseconds, and Raft
cannot acknowledge a write until it is durable on a quorum. The quota exists because a
memory-mapped B+tree has to be recoverable in bounded time. The defrag is disruptive because
rewriting a mmap'd file requires exclusive access. Buying a dedicated NVMe is not a nice-to-have;
it is the fix for the root cause of most etcd pages.

**Economics view.** The 8 GiB warning threshold is explicitly an MTTR trade: bigger database →
longer snapshot, longer `MsgSnap` to a lagging follower, longer defrag, longer restore. On a GPU
fleet, control-plane downtime converts directly into idle-but-still-depreciating hardware, so the
"cheap" move of raising the quota to avoid ops work can be the expensive one. Frame it explicitly
in the capex-vs-cloud writeup: etcd disk quality and quota discipline are cheap insurance against
very expensive idle compute, and the number you need is (minutes of control-plane downtime) ×
(fleet $/GPU-hour) × (GPU count).

## Real-world use cases

- **"Kubernetes ETCD Out of Space on kOps: A Real-Life Incident and Recovery"** —
  <https://medium.com/@caue._/kubernetes-etcd-out-of-space-on-kops-a-real-life-incident-and-recovery-a1857f3e0998>
  — a documented production NOSPACE incident on a kOps cluster, worked live. What it shows: the
  exact sequence this lesson teaches — the cluster going read-only while reads kept working, the
  `mvcc: database space exceeded` error surfacing through `kubectl`, and compact → defrag →
  `alarm disarm` restoring write capability. Read it as the narrative version of Drill 1.
- **etcd v3.5 data-inconsistency postmortem** (in-repo:
  `Documentation/postmortems/v3.5-data-inconsistency.md`) —
  <https://github.com/etcd-io/etcd/blob/main/Documentation/postmortems/v3.5-data-inconsistency.md>
  — etcd's own postmortem of the bug where mishandled `consistent_index` persistence could let a
  member replay already-applied entries after a crash, producing durable divergence between
  members. What it shows: why corruption detection is a first-class feature rather than paranoia,
  and why kubeadm sets `--feature-gates=InitialCorruptCheck=true` on the etcd static pod. It is
  also the best available argument for why you should never `alarm disarm` a CORRUPT.
- **CNCF "The Kubernetes Surgeon's Handbook: Precision Recovery from etcd Snapshots"** —
  <https://www.cncf.io/blog/2025/05/08/the-kubernetes-surgeons-handbook-precision-recovery-from-etcd-snapshots/>
  — a walkthrough of recovering specific objects from a snapshot without a full-cluster restore
  (restore the snapshot into a scratch data directory, start a single-member etcd against it, read
  the keys you need, apply them back to the live cluster). What it shows: the useful middle option
  between "lose the object" and "take the whole cluster down," and a practical demonstration that
  a restored data directory is just an etcd you can start anywhere.
- **CNCF "Making etcd incidents easier to debug in production Kubernetes"** —
  <https://www.cncf.io/blog/2026/03/12/making-etcd-incidents-easier-to-debug-in-production-kubernetes/>
  — introduces `etcd-diagnosis`, tooling built for triaging real production etcd incidents. What
  it shows: that the triage tree above is common enough to have been productised, and where to
  look for tooling beyond raw `etcdctl` and Prometheus.

## Worked example — trace of the two drills on a stacked kubeadm node

Both drills on a single-node stacked kubeadm cluster, with the real output at each step. Run them;
the transcripts below are what you should see.

### Drill 1 — fill the quota, watch the cluster freeze, recover it

**Setup.** Shrink the quota so you can fill it in seconds rather than days. Edit
`/etc/kubernetes/manifests/etcd.yaml` and add to the container's `command`:

```yaml
    - --quota-backend-bytes=16777216      # 16 MiB, for the drill only
```

The kubelet notices the file change and restarts the pod within a few seconds.

**Fill it.**

```console
$ VAL=$(head -c 65536 /dev/urandom | base64 -w0)
$ for i in $(seq 1 400); do etcdctl put "/drill/junk-$i" "$VAL" >/dev/null 2>&1 || \
    { echo "failed at $i"; break; }; done
failed at 189
```

189 puts × (256 + 15 + ~87 000 bytes of base64) ≈ 16.5 MB — the quota, as predicted by the cost
formula.

**Observe the freeze.**

```console
$ etcdctl alarm list
memberID:10276657743932975437 alarm:NOSPACE

$ etcdctl endpoint status -w table
+------------------------+------------------+---------+-----------------+---------+--------+-----------------------+--------+-----------+------------+-----------+------------+--------------------+-------------------------+
|        ENDPOINT        |        ID        | VERSION | STORAGE VERSION | DB SIZE | IN USE | PERCENTAGE NOT IN USE | QUOTA  | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX |         ERRORS          |
+------------------------+------------------+---------+-----------------+---------+--------+-----------------------+--------+-----------+------------+-----------+------------+--------------------+-------------------------+
| https://127.0.0.1:2379 | 8e9e05c52164694d |  3.6.8  |      3.6.0      |  17 MB  | 17 MB  |                    0% | 17 MB  |      true |      false |         3 |      21486 |              21486 | memberID:1027…alarm:NOSPACE |
+------------------------+------------------+---------+-----------------+---------+--------+-----------------------+--------+-----------+------------+-----------+------------+--------------------+-------------------------+

$ kubectl create deployment probe --image=nginx
error: failed to create deployment: etcdserver: mvcc: database space exceeded

$ kubectl get nodes             # reads unaffected
NAME   STATUS   ROLES           AGE   VERSION
cp1    Ready    control-plane   3d    v1.36.1
```

Note `PERCENTAGE NOT IN USE = 0%`: nothing has been compacted yet, so every byte in the file is
still logically live. This is the "compaction has not run" shape, distinct from the "compaction
ran but nobody defragged" shape (78%) shown earlier.

**Recover, and watch each step do exactly one thing.**

```console
$ etcdctl del /drill/ --prefix                     # allowed: deletes have zero quota cost
185

$ etcdctl endpoint status -w json | jq -r '.[0].Status.header.revision'
21491

$ etcdctl compact 21491 --physical
compacted revision 21491

$ etcdctl endpoint status -w table | awk -F'|' '{print $6, $7, $8}'
 DB SIZE   IN USE   PERCENTAGE NOT IN USE
 17 MB     528 kB   96%
#          ^^^^^^ logical content collapsed…      ^^^ …but DB SIZE has not moved.
#          Writes STILL FAIL here. This is the step people stop at.

$ etcdctl defrag
Finished defragmenting etcd member[127.0.0.1:2379]

$ etcdctl endpoint status -w table | awk -F'|' '{print $6, $7, $8}'
 DB SIZE   IN USE   PERCENTAGE NOT IN USE
 561 kB    528 kB   5%

$ etcdctl put /probe still-alarmed
{"level":"warn","msg":"retrying of unary invoker failed",...}
Error: etcdserver: mvcc: database space exceeded
#  ^^ the alarm is STICKY. Space is free, but the alarm has not cleared itself.

$ etcdctl alarm disarm
memberID:10276657743932975437 alarm:NOSPACE

$ etcdctl alarm list
$ kubectl create deployment probe --image=nginx
deployment.apps/probe created
```

Four distinct observations to write into your runbook: deletes worked while writes did not;
compaction moved `IN USE` and not `DB SIZE`; defrag moved `DB SIZE`; and the alarm survived all
three until explicitly disarmed. **Remove the temporary `--quota-backend-bytes` line when you are
done.**

### Drill 2 — destroy the data directory, restore from a snapshot

```console
# Take and verify a backup.
$ etcdctl snapshot save /backup/drill.db
{"level":"info","msg":"saved","path":"/backup/drill.db"}
Snapshot saved at /backup/drill.db
Server version 3.6.8

$ etcdutl snapshot status /backup/drill.db -w table
+----------+----------+------------+------------+---------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE | VERSION |
+----------+----------+------------+------------+---------+
| a71cc3b8 |    21502 |       1187 |     561 kB |   3.6.0 |
+----------+----------+------------+------------+---------+

# Create a MARKER after the snapshot, so you can prove the RPO boundary is real.
$ kubectl create cm dr-marker --from-literal=t=$(date +%s)
configmap/dr-marker created

# Destroy etcd.
$ mv /etc/kubernetes/manifests/etcd.yaml /tmp/
$ rm -rf /var/lib/etcd/member

# The apiserver crash-loops within seconds.
$ crictl logs $(crictl ps -a --name kube-apiserver -q | head -1) 2>&1 | tail -3
W0818 11:02:14.882  Error from storage: context deadline exceeded
F0818 11:02:19.104  Error creating self-signed certificates: ... etcdserver: no leader
$ kubectl get nodes
The connection to the server 10.10.0.11:6443 was refused - did you specify the right host or port?

# Restore.  NOTE: etcdutl, not etcdctl — see the tooling table below.
$ etcdutl snapshot restore /backup/drill.db \
    --name cp1 \
    --initial-cluster cp1=https://10.10.0.11:2380 \
    --initial-cluster-token k8s-drill \
    --initial-advertise-peer-urls https://10.10.0.11:2380 \
    --data-dir /var/lib/etcd
{"level":"info","msg":"restoring snapshot","path":"/backup/drill.db",
 "wal-dir":"/var/lib/etcd/member/wal","data-dir":"/var/lib/etcd",
 "snap-dir":"/var/lib/etcd/member/snap"}
{"level":"info","msg":"restored snapshot","path":"/backup/drill.db"}

$ mv /tmp/etcd.yaml /etc/kubernetes/manifests/

# Verify — and prove the RPO.
$ kubectl get nodes
NAME   STATUS   ROLES           AGE   VERSION
cp1    Ready    control-plane   3d    v1.36.1

$ kubectl get cm dr-marker
Error from server (NotFound): configmaps "dr-marker" not found
#  ^^ EXACTLY the point: it was created after revision 21502 and is gone.
#     That gap is your RPO, expressed as an object you can point at.

$ etcdctl endpoint status -w table | awk -F'|' '{print $2, $11, $12}'
 ENDPOINT                 RAFT TERM  RAFT INDEX
 https://127.0.0.1:2379   2          14
#  ^^ term and index RESET. This is the synthetic Raft history the restore built.
```

That last line is the whole "restore is not a file copy" lesson in one observation: the data came
back, and the Raft log did not — it was rebuilt from scratch at term 1 with one entry per member.

### The etcd 3.5 → 3.6 tooling split

The command surface moved, and a runbook written for 3.5 fails on 3.6 in ways that cost you
minutes you do not have. Verified by diffing `etcdctl/ctlv3/command/` and `etcdutl/etcdutl/`
between the `release-3.5` and `release-3.6` branches:

| Operation | etcd 3.5 | etcd 3.6 |
|---|---|---|
| Take a backup | `etcdctl snapshot save` | `etcdctl snapshot save` (unchanged) |
| Inspect a snapshot | `etcdctl snapshot status` (worked, printed `Deprecated: Use etcdutl snapshot status instead`) | **`etcdutl snapshot status`** — removed from `etcdctl` |
| Restore a snapshot | `etcdctl snapshot restore` (worked, printed `Deprecated: Use etcdutl snapshot restore instead`) | **`etcdutl snapshot restore`** — removed from `etcdctl` |
| Defrag a **running** member | `etcdctl defrag [--cluster]` | `etcdctl defrag [--cluster]` (unchanged) |
| Defrag a **stopped** data dir | `etcdctl defrag --data-dir <dir>` | **`etcdutl defrag --data-dir <dir>`** — the `--data-dir` flag no longer exists on `etcdctl defrag` |
| Compare member hashes | `etcdctl endpoint hashkv` | `etcdctl endpoint hashkv` (plus `etcdutl hashkv` for an offline dir) |

The clean mental model: **`etcdctl` talks to a running cluster over gRPC; `etcdutl` operates on
files on disk.** `snapshot save` stays in `etcdctl` because it must ask a live member for a
stream. Everything that touches a data directory or a snapshot file directly moved to `etcdutl`.
Since 3.6 arrived as kubeadm's default at **Kubernetes 1.34**, a cluster on 1.33 still has 3.5's
surface — check before you trust the runbook.

## Practice

On a 1-VM (drills 1–3) or 3-VM (drill 4) cluster; stacked kubeadm etcd is fine. Time yourself
against the checkpoint's 30-minute target.

**Drill 1 — fill and recover.** Set `--quota-backend-bytes=16777216` in the etcd static pod, fill
past it, and capture: the exact `etcdctl put` error, the exact `kubectl` error, `alarm list`
output, and an `endpoint status -w table` before and after each of delete / compact / defrag /
disarm. **You must show four separate state changes** — deletes succeeding under the alarm,
`IN USE` dropping without `DB SIZE` moving, `DB SIZE` dropping, and the alarm surviving until
disarmed. Then leave the alarm armed and try `disarm` *first*, to see it re-trip; that negative
result belongs in the writeup too.

**Drill 2 — destroy and restore.** `etcdctl snapshot save`, verify with `etcdutl snapshot status`,
create a marker ConfigMap **after** the snapshot, `rm -rf /var/lib/etcd/member`, watch the
apiserver crash-loop (`crictl logs`, `journalctl -u kubelet`), restore with **`etcdutl snapshot
restore`**, and prove recovery: apiserver healthy, nodes `Ready`, marker **absent**, and `RAFT
TERM`/`RAFT INDEX` reset. Record the revision gap explicitly as your measured RPO.

**Drill 3 — the tooling split.** On a 3.6 cluster, run `etcdctl snapshot restore` and
`etcdctl defrag --data-dir /var/lib/etcd` and capture the errors. Then do the same operations with
`etcdutl`. Confirm your own runbook uses the right tool for the etcd version you actually run
(`kubectl -n kube-system get pod etcd-<node> -o jsonpath='{.spec.containers[0].image}'`).

**Drill 4 (3 VMs) — quorum and learners.** Build a 3-member stacked cluster (this is 10.3's
worked example; do it once here for the etcd half). Then:
(a) stop one member and confirm writes continue and `endpoint status` shows two healthy members;
(b) stop a second and confirm `etcd_server_has_leader` goes to 0 and `kubectl` hangs on reads as
well as writes — quorum loss, not just write loss;
(c) restart one and watch quorum return;
(d) add a fourth member with `--learner`, observe `IS LEARNER true` in `member list -w table`,
promote it, and confirm quorum went from 2 to 3.

**Acceptance:** both failure drills documented with **exact commands, the etcd tool used (and the
3.5-vs-3.6 difference), the observed output at each step, and what the apiserver did** — plus your
own version of the growth math from the Core-concepts section using measured object sizes and
write rates from your cluster. This is the etcd half of the deliverable, alongside 10.1's cert
table.

## Common pitfalls

- **Disarming before reclaiming.** `alarm disarm` frees nothing; the very next `Put` re-trips it
  because the quota is checked against the physical file size. Order is always
  delete → compact → defrag → disarm.
- **Compacting and stopping there.** Compaction moves `DbSizeInUse`, not `DbSize`, and the quota
  reads `DbSize`. You will see the space "freed" in one column and the cluster still frozen. The
  3.6 `endpoint status` table shows both columns side by side specifically so you can see this.
- **Defragging all members in parallel.** Each defrag holds that member's batch-transaction *and*
  read-transaction locks for the whole rewrite, so the member answers nothing — not even Raft.
  Two blocked members out of three is a self-inflicted quorum loss during an incident. One at a
  time, health-check between each, leader last (or `move-leader` first).
- **Restoring one member and expecting replication to fix the rest.** Restore mints a new cluster
  ID and new member IDs from `--initial-cluster-token` + `--initial-cluster`. A restored member and
  a surviving member will refuse each other with a cluster-ID mismatch. Restore the same file on
  every member with identical `--initial-cluster` and token, then start them together.
- **Hard-coding etcd 3.5 commands in a 3.6 runbook.** `etcdctl snapshot restore`, `etcdctl
  snapshot status` and `etcdctl defrag --data-dir` are all gone in 3.6. And note the version
  boundary: etcd 3.6 became kubeadm's default at Kubernetes **1.34**, not 1.33.
- **Treating 8 GiB as a hard limit — or as no limit.** It is a warning threshold whose real
  content is restore time. Raising the quota above it is a legitimate decision if you write down
  the MTTR you are buying; setting `--quota-backend-bytes` negative to disable the quota is how
  you turn a recoverable incident into an unbootable member.
- **`alarm disarm` on a CORRUPT.** NOSPACE is a capacity problem you can disarm your way out of
  after fixing it. CORRUPT means members genuinely disagree about the data; disarming just
  re-enables writes on top of divergence. Find the diverging member with `endpoint hashkv`, remove
  it, wipe it, re-add it as a learner.
- **Backing up to the node's own disk.** A snapshot on the machine whose disk failed is not a
  backup. Ship it off-node and verify it with `etcdutl snapshot status`, which also runs a full
  bbolt integrity check.
- **Adding a voting member to a busy cluster.** Going 3 → 4 raises quorum from 2 to 3 immediately,
  while the new member is still receiving a potentially huge `MsgSnap`. Add as `--learner`, wait,
  then `member promote` — which itself refuses if the learner is too far behind.

## Self-check

**(a) 3-member vs 5-member etcd — how many simultaneous member failures does each tolerate, and
why?**
**Answer:** 3 tolerates **1**, 5 tolerates **2**. A Raft commit needs a majority, quorum =
⌊N/2⌋+1, so quorum is 2 and 3 respectively and you can lose `N − quorum` members and still form a
majority. Even sizes never help: 4 also needs 3 and so also tolerates only 1, at the cost of an
extra machine and an extra `fsync` in the commit path. The real argument for 5 on a large fleet is
not raw availability but **maintenance**: at N=3, taking one member down for a reboot leaves you
at zero fault tolerance, whereas at N=5 you can lose an unplanned member during planned work.

**(b) Compaction vs defragmentation — what does each reclaim?**
**Answer:** **Compaction** discards superseded MVCC key versions and tombstones below a chosen
revision. It shrinks the *logical* content (`DbSizeInUse` /
`etcd_mvcc_db_total_size_in_use_in_bytes`) and returns pages to bbolt's internal free list, but it
does **not** shrink the file. Under Kubernetes the apiserver drives it every 5 minutes via a CAS
lease on the `compact_rev_key` key, compacting to the revision it saw one interval ago; etcd's own
`--auto-compaction-*` is disabled by default (`retention "0"`). **Defragmentation** rewrites the
bbolt file into a fresh, densely-packed copy and `rename(2)`s it over the original, returning free
pages to the filesystem and shrinking `DbSize` /
`etcd_mvcc_db_total_size_in_bytes`. It fully blocks the member it runs against (it holds both the
batch-tx and read-tx locks), is never automatic, and must be run one member at a time. The reason
the distinction is operationally load-bearing: **the quota is checked against the physical file
size**, so compaction alone never clears a NOSPACE alarm.

**(c) Why does etcd hate slow disks — what latency dominates, and what metric warns you first?**
**Answer:** A Raft entry is not committed until it has been appended to the WAL and `fsync`ed on a
quorum of members, so **WAL fsync latency is the floor on every write** — not CPU, not network,
not the size of the data. When fsync stalls: commits stall → the apiserver's calls time out →
the leader's heartbeats go out late → a follower's randomised election timeout (1000–2000 ms at
defaults) fires → a new leader is elected → in-flight proposals are lost and retried → every
watcher reconnects → load spikes on the same slow disk. It is a cascade, not a slowdown.
First warning: **`etcd_disk_wal_fsync_duration_seconds`** p99 — keep it well under ~10 ms; the
histogram's buckets start at 1 ms, which tells you the design point. Backed up by
`etcd_disk_backend_commit_duration_seconds` (p99 under ~25 ms),
`etcd_server_leader_changes_seen_total` (should be flat), and the `apply request took too long`
log line emitted past `--warning-apply-duration` (default 100 ms). Fix: a dedicated low-latency
NVMe for `--data-dir`.

**(d) What changed in etcd 3.6's tooling, and why does it matter for your runbook?**
**Answer:** The rule is now **`etcdctl` = talk to a running cluster over gRPC; `etcdutl` = operate
on files on disk.** So `snapshot save` stays in `etcdctl` (it needs a live member to stream from),
while **`snapshot restore` and `snapshot status` moved to `etcdutl`** and were removed from
`etcdctl` — in 3.5 they still worked there but printed a deprecation notice. Separately,
`etcdctl defrag` lost its `--data-dir` flag: defragmenting a *stopped* member's directory is now
`etcdutl defrag --data-dir <dir>`, while `etcdctl defrag [--cluster]` still defrags running
members over their client endpoints. It matters because a 3.5-era runbook fails at the worst
possible moment — and the version boundary is easy to get wrong: kubeadm shipped etcd **3.5.24 in
Kubernetes 1.33** and only moved to **3.6.5 in 1.34** (3.6.8 in 1.35/1.36).

**(e) You are paged: `kubectl apply` is frozen. What three checks triage it in under a minute, and
what fixes each?**
**Answer:** In this order. (1) **`etcdctl alarm list`.** `NOSPACE` → writes fail but reads and
deletes work; fix is delete the offender → `compact` → `defrag` one member at a time → `alarm
disarm`. `CORRUPT` → reads fail too; **do not disarm**; find the diverging member with `endpoint
hashkv --cluster`, remove it, wipe its data dir, re-add as a learner. (2) **`etcd_server_has_leader`
/ `endpoint status --cluster`.** `0` or timing-out endpoints → quorum lost; first try to revive a
member with an intact data directory (no data loss), then a full snapshot restore on all members,
and only as a last resort `--force-new-cluster` from a survivor — which unilaterally declares one
member's view to be the truth. (3) **`etcd_disk_wal_fsync_duration_seconds` p99** plus `grep 'apply
request took too long'`. Elevated with a leader present and no alarm → the disk is too slow; move
`--data-dir` to dedicated NVMe. If all three are clean, it is not etcd — look at apiserver
flow-control 429s, a hung admission webhook, or the network.

**(f) A restore has to be done on all three members with the same `--initial-cluster` and the same
`--initial-cluster-token`. Why?**
**Answer:** Because `etcdutl snapshot restore` does not restore a Raft cluster — it **builds a new
one**. It copies the bbolt file into `member/snap/db`, verifies the appended SHA-256, then
**deletes the members bucket entirely**, derives a new cluster ID and new member IDs from
`--initial-cluster-token` plus the `--initial-cluster` URL map, and synthesises a fresh WAL
containing one `ConfChangeAddNode` entry per member at term 1, with `HardState{Term:1, Commit:N}`
and a Raft snapshot at index N. Two members that restore with different tokens or different
cluster strings compute *different cluster IDs* and refuse to talk to each other; a restored
member and a surviving member likewise mismatch. Hence: same file, same token, same
`--initial-cluster` everywhere, differing only in `--name` and
`--initial-advertise-peer-urls`, all started together, with no surviving members mixed in. A side
effect worth knowing: the revision counter resumes at the snapshot's revision, so revisions appear
to go backwards to clients — `--bump-revision N --mark-compacted` exists to avoid that when
external consumers track revisions.

## Connections & what's next

This lesson assumed the PKI from **10.1** already works — you cannot reach etcd to break it on
purpose without `apiserver-etcd-client` signed by the etcd CA. It feeds directly into **10.3
(control-plane HA)**: the quorum math and the partition behaviour here are the reasoning behind
that lesson's stacked-vs-external topology decision and its "why an even number of control-plane
nodes is worse than useless" rule, and the pre-upgrade snapshot in 10.3's upgrade runbook is this
lesson's DR discipline applied one level up. It underpins **10.4 (declarative fleets)**, where
`KubeadmControlPlane` automates precisely the member-management sequence you just did by hand —
forward etcd leadership off a machine, remove the member, then delete the machine — and refuses to
scale while any member is still a learner. It underpins **10.6 (hardware health and RMA)**: a node
RMA'd out from under a stacked etcd member is a quorum event, not just a compute event. And the
quota-versus-MTTR trade is a direct input to the **10.8 capex-vs-cloud** model: etcd disk quality
is cheap insurance measured against idle-GPU cost.

Next: **[10.3 · Control-plane HA](03-control-plane-ha.md)** — you now know how a single etcd
member fails and recovers; the next lesson grows this to a 3-node quorum behind a VIP and adds the
version-skew-safe upgrade order that keeps that quorum intact while you patch it.

## References & further reading

**Primary sources**

- **`etcd-io/etcd`, branch `release-3.6` (tip `v3.6.14`, Aug 2026)** —
  <https://github.com/etcd-io/etcd/tree/release-3.6> — read directly and the authority for every
  default, constant, error string and behaviour here. The files that matter:
  `server/embed/config.go` (flag defaults and validation), `server/storage/quota.go` (the cost
  formula and quota constants), `server/etcdserver/api/v3rpc/quota.go` +
  `server/etcdserver/server.go` (alarm activation),
  `server/etcdserver/apply/{uber_applier,apply,corrupt}.go` (exactly what NOSPACE and CORRUPT
  block), `server/storage/backend/backend.go` (defrag and its locks),
  `server/etcdserver/api/v3compactor/` (auto-compaction), `etcdutl/snapshot/v3_snapshot.go`
  (save/status/restore internals), `etcdctl/ctlv3/command/printer.go` (`endpoint status`
  columns), `server/storage/*/metrics.go` (metric names and buckets). **Note:** etcd.io is
  unreachable from this environment's egress proxy, so the published maintenance/recovery/tuning
  guides were **not** fetched or relied upon — everything here came from this tree. Read for:
  checking any claim against the version you actually run.
- **`etcd-io/etcd`, branch `release-3.5`** —
  <https://github.com/etcd-io/etcd/tree/release-3.5> — used only to diff
  `etcdctl/ctlv3/command/{snapshot,defrag}_command.go` and `printer.go` against 3.6 for the
  tooling-split and column-set comparison. Read for: what your 3.5 runbook was written against.
- **`etcd-io/raft` v3.6.0** — <https://github.com/etcd-io/raft/tree/v3.6.0> — the Raft library
  etcd embeds; `raft.go`'s `resetRandomizedElectionTimeout()` is the source of the 1000–2000 ms
  randomised election window. Read for: PreVote and CheckQuorum semantics in detail.
- **etcd v3.5 data-inconsistency postmortem** (in-repo) —
  <https://github.com/etcd-io/etcd/blob/main/Documentation/postmortems/v3.5-data-inconsistency.md>
  — read for: why corruption checking exists, and what a CORRUPT alarm is telling you.
- **`kubernetes/kubernetes`, branch `release-1.36`** —
  <https://github.com/kubernetes/kubernetes/tree/release-1.36> — the apiserver side:
  `staging/src/k8s.io/apiserver/pkg/storage/etcd3/compact.go` (the `compact_rev_key` CAS-lease
  algorithm and its "normal 5 m, failover >5 m and <10 m" guarantee),
  `.../storagebackend/config.go` (`DefaultCompactInterval = 5m`),
  `cmd/kubeadm/app/phases/etcd/local.go` (etcd static-pod flags), and
  `cmd/kubeadm/app/constants/constants.go` (`DefaultEtcdVersion` per release). **kubernetes.io is
  likewise blocked here**, so "Operating etcd clusters" and the other kubernetes.io pages were
  **not relied upon**. Read for: what your Kubernetes version is doing to your etcd.

**Real-world engineering blogs**

- **"Kubernetes ETCD Out of Space on kOps: A Real-Life Incident and Recovery"** —
  <https://medium.com/@caue._/kubernetes-etcd-out-of-space-on-kops-a-real-life-incident-and-recovery-a1857f3e0998>
  — what it shows: a real production NOSPACE incident worked through the same
  compact/defrag/disarm sequence taught here, with the read-only-but-looks-healthy symptom.
- **CNCF "The Kubernetes Surgeon's Handbook: Precision Recovery from etcd Snapshots"** —
  <https://www.cncf.io/blog/2025/05/08/the-kubernetes-surgeons-handbook-precision-recovery-from-etcd-snapshots/>
  — what it shows: restoring a snapshot into a scratch data directory to recover individual
  objects, as a middle option between losing data and a full-cluster restore.
- **CNCF "Making etcd incidents easier to debug in production Kubernetes"** —
  <https://www.cncf.io/blog/2026/03/12/making-etcd-incidents-easier-to-debug-in-production-kubernetes/>
  — what it shows: purpose-built `etcd-diagnosis` tooling for triaging production etcd incidents
  faster than manual metric reading.

**Deeper dives**

- **bbolt** — <https://github.com/etcd-io/bbolt> — the single-file B+tree that *is* etcd's backend.
  Its README explains page allocation, the free list, and why a mmap'd B+tree never returns space
  to the filesystem on its own — which is the mechanical reason defragmentation has to exist as a
  separate, disruptive operation.
- **`etcdctl --help` / `etcdutl --help` on the binary you actually run** — the only source that is
  guaranteed to describe your version rather than someone's memory of a different one. Run both
  before you trust any runbook, including this one.
