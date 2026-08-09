---
lesson: "A01.1"
title: "Consistency models"
module: "A-01"
concept: "consistency-hierarchy"
status: not-started
est_time: "3 hrs"
artifacts: ["consistency-placement-matrix", "etcd-stale-read-blast-radius-note"]
---

# A01.1 · Consistency models

> **Concept.** Linearizability and serializability are orthogonal axes, CAP is the degenerate corner of PACELC, and the K8s watch cache is the eventual-consistency trap that sits under every GPU scheduler.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Why this matters

At senior level you pick "strong" or "eventual" per datastore and move on. At staff level someone asks *why the scheduler double-bound a GPU* and the answer is a consistency argument: which read path was stale, what guarantee it actually offered, and the exact number of milliseconds you'd pay to close the gap. You cannot bound a control plane you can only describe as "strongly consistent." You have to name the model, the isolation axis, and the read path — because those three decide whether a stale read can corrupt state or merely waste a scheduling cycle.

The single most interview-relevant fact in this whole module: the Kubernetes API server serves reads from a watch cache that is only eventually consistent. Every controller, scheduler, and informer in the fleet is reading potentially-stale data by default, and that is *by design* — the correctness comes from the write path, not the read path.

## Core notes

**Skip (you already know):** eventual vs strong definitions; that CAP forces a choice only during a partition; that "linearizable = looks like a single copy respecting real-time order."

**The hierarchy (strict superset chain, strongest first).**

```
strict-serializable ⊃ linearizable ⊃ sequential ⊃ causal ⊃ (PRAM / read-your-writes) ⊃ eventual
```

Each level is a strict superset of guarantees below it: satisfying the stronger model automatically satisfies the weaker one, never the reverse.

**Two orthogonal axes — this is the staff distinction.** Linearizability and serializability are *not* points on one line; they are different axes:

- **Linearizability** is a *single-object, real-time recency* guarantee. It governs one register/key: once a write commits, every later read sees it (or newer), ordered by wall-clock. Says nothing about multi-key atomicity.
- **Serializability** is a *multi-object transaction isolation* guarantee. Transactions appear to execute in *some* serial order — but that order need not respect real time. Says nothing about recency.
- **Strict-serializable** = both axes at once: serial transaction order that also respects real-time. This is Spanner's `TrueTime` external consistency and the strongest useful model.

Because they are orthogonal, both off-diagonal corners exist and matter:

- **Serializable but *not* linearizable:** snapshot-isolation-style reads served from a consistent past snapshot. Multi-key atomic, but you can read stale — the snapshot is "in the past."
- **Linearizable but *not* serializable:** a single-register compare-and-swap. Perfectly real-time-recent on that one key, but there is no multi-key transaction, so serializability is not even in scope.

**PACELC — CAP is a corner case, teach this instead (Abadi).** *If* **P**artition, choose **A**vailability or **C**onsistency; **E**lse (no partition), choose **L**atency or **C**onsistency. The whole point is the **Else** clause: 99.99% of the time there is no partition, and you are *still* trading consistency for latency — a quorum read costs a network round trip that a local cache read does not. CAP only describes the rare partition; PACELC describes every normal day.

| System | PACELC | Reading |
| --- | --- | --- |
| Dynamo, Cassandra | **PA/EL** | give up C for availability under partition, and for latency normally |
| Classic single-node RDBMS | **PC/EC** | consistency both under partition and normally |
| etcd, Spanner | **PC/EC** | quorum/consensus always; pay latency for consistency |

**Real placements — the etcd / K8s split.** etcd is **linearizable by default**: reads go through Raft (`ReadIndex` or a quorum round trip), so a read reflects every committed write. But the **K8s API server serves list/watch from an in-memory watch cache that is only eventually consistent.** A `list` with `resourceVersion=0` explicitly asks for "any cached version, possibly stale" — fast, local memory, no quorum. To force freshness you must request a quorum read (omit `resourceVersion` → served through etcd). This staleness is the documented root cause of double-scheduling and stale-informer bugs.

**GPU frame — control plane.** Under a GPU fleet the K8s control plane is the control plane: schedulers and controllers read from fast-but-eventual informer caches while the source of truth (etcd) stays linearizable-but-slow. A scheduler trusting a stale cache can **double-bind a GPU** — bind pod B to a node whose GPU was already claimed by pod A in a write the cache hasn't caught up to. The correct fix is **not** "make reads stronger" (that just adds quorum latency to the hot path). The fix is **optimistic concurrency on the write**: the binding write carries the `resourceVersion` it read; etcd does a compare-and-swap and rejects the write if the object changed underneath. Stale reads are cheap and tolerable *because the write is CAS-guarded.*

## Worked example

**Setup.** 3-node etcd cluster. A custom scheduler lists pods and nodes, decides a GPU binding, and writes a `Binding`. It reads via the API server watch cache with `resourceVersion=0`.

**Q1 — What can a stale (rv=0) read cause, and what can it *not*?**

- **Can cause: double-schedule / double-bind.** The cache shows a node's GPU as free because the prior `Binding` write hasn't propagated to the cache yet. Scheduler picks it again → two pods target one GPU. This is a *read* staleness bug and it is real.
- **Cannot cause: a lost write.** The `Binding` write is a CAS against etcd on `resourceVersion`. If the object moved, etcd returns a conflict (HTTP 409) and the scheduler retries with fresh state. Committed writes are never silently overwritten — the linearizable write path holds even though the read path is eventual.

So the blast radius of a stale read is bounded to *wasted work + a conflict retry*, unless the code skips the CAS — then the stale read becomes a corrupting double-bind. The guarantee is only as strong as the write's optimistic-concurrency check.

**Q2 — The latency delta you're trading.**

- **Cache read (rv=0):** local process memory. ≈ tens of microseconds, no network, no disk. No quorum.
- **Quorum / linearizable read:** `ReadIndex` confirms leadership + waits for the commit index to apply. ≈ **1 RTT to a majority + apply time**. On a 2 ms cross-AZ link that is ~2 ms + apply — roughly **50–100× the cache read**, per read, on the scheduler's hot path.

A scheduler doing thousands of reads per second cannot pay quorum latency on every one. So the design is: **read cheap and eventual, write guarded and linearizable.** You buy correctness at the one write, not at the million reads.

## Practice

*Feeds [staff design portfolio](../practice/staff-design-portfolio/README.md).*

1. **Placement matrix.** Build a table classifying 8 systems you actually run (etcd, your primary RDBMS, Cassandra/Dynamo if present, Redis, Kafka, S3, Spanner/Cloud SQL, your K8s API server) on *both* axes — (a) PACELC class, (b) linearizable? serializable? both? neither? — with one sentence of evidence each. Flag every one where the read path and write path have different guarantees.
2. **Stale-read blast-radius note.** For your own scheduler or a controller you own, write the two-column "stale read *can* cause / *cannot* cause" analysis. Identify precisely which writes are CAS-guarded and which are blind writes (the blind ones are your corruption surface).
3. Instrument or estimate the cache-read vs quorum-read latency delta in your environment and state the read-QPS above which quorum-on-every-read is infeasible.

## Self-check

- Two systems are both called "strongly consistent," but one is serializable-not-linearizable and the other linearizable-not-serializable. Give a concrete example of each and the observable difference. **Answer:** Serializable-not-linearizable = snapshot-isolation reads: multi-key atomic, but a read can return a consistent *past* snapshot (stale). Linearizable-not-serializable = a single-key CAS register: real-time-recent on that key, but there is no multi-key transaction so serializability isn't in scope. Observable difference: the first can read stale-but-atomic across keys; the second is always fresh but only for one key.
- A K8s scheduler double-bound a GPU. Is the fix a stronger read path or something else, and why? **Answer:** Not a stronger read. The stale read came from the eventually-consistent watch cache (`resourceVersion=0`), and forcing quorum reads on the scheduler's hot path would add ~1 RTT per read at 50–100× cost. The fix is optimistic concurrency on the *write*: carry the read `resourceVersion` into the `Binding` write so etcd does a compare-and-swap and rejects (409) the second bind. Reads stay cheap; correctness lives at the guarded write.
- Why is PACELC's "Else" clause where most design decisions actually live, and what does it cost concretely? **Answer:** Partitions are rare; the "Else" (no-partition) case is nearly always in effect, and even then you trade consistency against latency — a quorum/linearizable read costs a network round trip (~1 RTT + apply, e.g. ~2 ms cross-AZ) versus a local cache read (~microseconds). CAP only describes the rare partition corner; PACELC's Else describes the everyday latency-vs-consistency tax.

## References

- Jepsen — Consistency Models (the map of the hierarchy): https://jepsen.io/consistency/models
- Kyle Kingsbury (Aphyr) — Strong consistency models: https://aphyr.com/posts/313-strong-consistency-models
- Daniel Abadi — Consistency Tradeoffs in Modern Distributed Database System Design (PACELC): http://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf
- Kubernetes — API concepts, resourceVersion and cache semantics: https://kubernetes.io/docs/reference/using-api/api-concepts/
