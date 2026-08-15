---
lesson: "A01.1"
title: "Consistency models"
module: "A-01"
concept: "consistency-hierarchy"
status: not-started
est_time: "4 hrs"
prev: null
next: "02-consensus-and-quorums.md"
artifacts: ["consistency-placement-matrix", "etcd-stale-read-blast-radius-note"]
sources: 11
---

# A01.1 · Consistency models

> **Concept.** Linearizability and serializability are orthogonal axes, CAP is the degenerate corner of PACELC, and the K8s watch cache is the eventual-consistency trap that sits under every GPU scheduler.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits

This is the module's opening lesson. The module's through-line is that **a senior engineer can build a distributed system; a staff engineer can bound it** — state the exact guarantee, its cost, and the failure mode when it's violated — across the **three planes** every GPU platform is made of: control (etcd/K8s), training (gang-scheduled jobs), and serving (SLO-bound inference). Consistency models are where that bounding starts: before you can reason about a scheduler, a checkpoint store, or an inference cache, you need precise vocabulary for "how stale can a read be" and "what does a write actually promise." Every later lesson in this module — consensus (02), replication (03), caching (04) — assumes you can name a system's consistency model on sight and quote the cost of tightening it. Get this lesson's vocabulary solid and the rest of the module reads as applications of it, not new theory.

## Why this matters

At senior level you pick "strong" or "eventual" per datastore and move on. At staff level someone asks *why the scheduler double-bound a GPU* and the answer is a consistency argument: which read path was stale, what guarantee it actually offered, and the exact number of milliseconds you'd pay to close the gap. You cannot bound a control plane you can only describe as "strongly consistent." You have to name the model, the isolation axis, and the read path — because those three decide whether a stale read can corrupt state or merely waste a scheduling cycle.

The single most interview-relevant fact in this whole module: the Kubernetes API server serves reads from a watch cache that is only eventually consistent. Every controller, scheduler, and informer in the fleet is reading potentially-stale data by default, and that is *by design* — the correctness comes from the write path, not the read path.

## What's new here (calibration)

**Skip (you already know this):**
- Eventual vs. strong consistency as a binary label.
- That CAP forces a choice, and that it only forces it during an actual partition.
- The one-line intuition that "linearizable = looks like a single copy respecting real-time order."

**The genuinely new depth this lesson adds:**
- Linearizability and serializability are **not** one scale from weak to strong — they're orthogonal axes, and both off-diagonal combinations (serializable-not-linearizable, linearizable-not-serializable) show up in systems you already run. Knowing which axis a guarantee lives on is the staff-level move.
- PACELC's **Else** clause as the tax you pay on every normal request, not just during rare partitions — with a concrete RTT number attached, not a hand-wave.
- The exact mechanics of the etcd-linearizable / K8s-watch-cache-eventual split (`resourceVersion=0`, optimistic-concurrency writes) as a worked, numbered example rather than a slogan.

## Core concepts

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

**Formal definition, one line (Herlihy & Wing, 1990).** An execution is linearizable if each operation appears to take effect atomically at some single instant between its invocation and its response — and that instant defines a legal, real-time-respecting sequential order. That "atomic point somewhere between call and return" is the precise thing every hand-wavy "looks instantaneous" description is trying to say.

**PACELC — CAP is a corner case, teach this instead (Abadi).** *If* **P**artition, choose **A**vailability or **C**onsistency; **E**lse (no partition), choose **L**atency or **C**onsistency. The whole point is the **Else** clause: 99.99% of the time there is no partition, and you are *still* trading consistency for latency — a quorum read costs a network round trip that a local cache read does not. CAP only describes the rare partition; PACELC describes every normal day.

| System | PACELC | Reading |
| --- | --- | --- |
| Dynamo, Cassandra | **PA/EL** | give up C for availability under partition, and for latency normally |
| Classic single-node RDBMS | **PC/EC** | consistency both under partition and normally |
| etcd, Spanner | **PC/EC** | quorum/consensus always; pay latency for consistency |

**Session guarantees — the practical middle ground apps actually need.** Between "fully linearizable" (expensive) and "eventual" (surprising) sits a set of per-client guarantees a system can offer cheaply because they only bind *one client's own view*, not global recency (Terry et al., 1994, from the Bayou project):

| Guarantee | What it promises | App bug it prevents |
| --- | --- | --- |
| Read-your-writes | A client always sees its own prior writes | User submits a form, refreshes, and sees the *old* value — looks like data loss |
| Monotonic reads | Successive reads never go backward in time | A client sees a friend-count of 50, then 48, then 50 again — looks like flapping/corruption |
| Monotonic writes | A client's writes are applied in the order it issued them | Writing "set status=shipped" then "set status=processing" lands out of order — final state is wrong |
| Writes-follow-reads | A write issued after seeing value V is ordered after V | Replying to a comment you just read gets applied *before* the comment it replies to |

Note these are all **client-scoped**, not system-wide — cheap to offer (usually via sticky sessions or client-carried version tokens) precisely because they don't require a global quorum.

**Real placements — the etcd / K8s split.** etcd is **linearizable by default**: reads go through Raft (`ReadIndex` or a quorum round trip), so a read reflects every committed write. But the **K8s API server serves list/watch from an in-memory watch cache that is only eventually consistent.** A `list` with `resourceVersion=0` explicitly asks for "any cached version, possibly stale" — fast, local memory, no quorum. To force freshness you must request a quorum read (omit `resourceVersion` → served through etcd). This staleness is the documented root cause of double-scheduling and stale-informer bugs (see [Real-world use cases](#real-world-use-cases) below).

**GPU frame — control plane.** Under a GPU fleet the K8s control plane is the control plane: schedulers and controllers read from fast-but-eventual informer caches while the source of truth (etcd) stays linearizable-but-slow. A scheduler trusting a stale cache can **double-bind a GPU** — bind pod B to a node whose GPU was already claimed by pod A in a write the cache hasn't caught up to. The correct fix is **not** "make reads stronger" (that just adds quorum latency to the hot path). The fix is **optimistic concurrency on the write**: the binding write carries the `resourceVersion` it read; etcd does a compare-and-swap and rejects the write if the object changed underneath. Stale reads are cheap and tolerable *because the write is CAS-guarded.*

## Perspectives

**The theory / algorithm-designer view.** Linearizability and serializability are formally distinct correctness conditions, not two points on a strength dial. Herlihy & Wing define linearizability purely in terms of one object's operation history — an atomic point between invocation and response. Serializability (from the database-transactions literature) is defined purely in terms of *multiple* objects and *equivalence to some serial schedule* — real time never enters the definition. Because the two definitions quantify over different things (one object vs. many; real-time order vs. any legal order), a system can satisfy either without the other. Treating them as one axis is the single most common conceptual bug engineers bring into a system-design interview.

**The operator / SRE view.** Staleness isn't abstract on a dashboard — it shows up as specific, nameable signals: **watch-cache lag** (the gap between the API server's in-memory `resourceVersion` and etcd's latest committed one), **informer resync interval** (the periodic full-relist controllers use as a staleness backstop, typically minutes), and **resourceVersion skew** across API server replicas behind a load balancer (each replica's watch cache can lag independently). An SRE who knows to check `apiserver_watch_cache_events_dispatched_total` and resourceVersion skew between replicas can localize a "stale scheduling" incident in minutes instead of chasing a network-partition theory.

**The client / application-developer view.** Most application bugs blamed on "eventual consistency" are actually missing **session guarantees**, not a need for full linearizability. A user who doesn't see their own comment after posting it needs read-your-writes, not a linearizable datastore. Reaching for global strong consistency to fix a session-scoped bug is the classic overcorrection — it buys correctness the app didn't need at a latency cost every request now pays.

**The economics / latency view.** PACELC's "Else" branch has a dollar-and-millisecond price tag. If a quorum read costs an extra ~2 ms over a local cache read (see the Worked example), and a service does 5,000 reads/sec, choosing linearizable-everywhere costs **10 seconds of aggregate added latency per second of wall-clock time** — a real capacity and cost line, not a footnote. Staff-level design explicitly prices this: read cheap by default, pay the quorum tax only where a stale read is genuinely unsafe.

## Real-world use cases

- **Kubernetes GitHub issue #59848 — "Kubernetes is vulnerable to stale reads, violating critical pod safety guarantees"** (confirmed live): https://github.com/kubernetes/kubernetes/issues/59848 — a `resourceVersion=0` reflector read let a partitioned/lagging API server return pre-deletion pod state after a kubelet restart, so two nodes briefly ran a pod with the same name simultaneously. This is the textbook production instance of exactly the stale-read mechanism this lesson teaches — read it first.
- **Google Cloud — "Strict Serializability and External Consistency in Spanner"**: https://cloud.google.com/blog/products/databases/strict-serializability-and-external-consistency-in-spanner — walks through Spanner's commit-wait + TrueTime protocol, showing concretely how a real system buys *both* axes (serial order + real-time recency) at once, and what it costs (bounded commit-wait on clock uncertainty, sub-millisecond in practice).
- **Cloudflare — "Building With Workers KV, a Fast Distributed Key-Value Store"**: https://blog.cloudflare.com/building-with-workers-kv/ — a production key-value store with an explicit, bounded eventual-consistency window: writes are visible immediately at the writing location but can take up to ~60 seconds to propagate globally. A good second data point that "staleness window" is a designed, disclosed number, not an excuse.
- **Jepsen — etcd 3.4.3 analysis** (cross-referenced from Lesson 02's references): https://jepsen.io/analyses/etcd-3.4.3 — confirmed etcd's strict-serializable claim held under process pauses, partitions, and clock skew, but separately found etcd's **lock service** was not safe under asynchronous networks. A real example of why you must state the guarantee per-*operation*, not per-*system* — "etcd is linearizable" is true for KV ops and was false for the lock API at the time.

## Worked example

**Setup.** 3-node etcd cluster. A custom scheduler lists pods and nodes, decides a GPU binding, and writes a `Binding`. It reads via the API server watch cache with `resourceVersion=0`.

**Q1 — What can a stale (rv=0) read cause, and what can it *not*?**

- **Can cause: double-schedule / double-bind.** The cache shows a node's GPU as free because the prior `Binding` write hasn't propagated to the cache yet. Scheduler picks it again → two pods target one GPU. This is a *read* staleness bug and it is real — it's the exact mechanism behind K8s issue #59848 above.
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

## Common pitfalls

1. **"Strongly consistent" is treated as one dial.** This collapses two orthogonal axes — recency (linearizability) and multi-key atomicity (serializability). Ask "linearizable on what, serializable across what," not "how strong."
2. **"CAP means you always trade C vs A."** CAP only binds during an actual partition, which is rare. PACELC's "Else" branch — consistency vs. latency — is what you pay on every normal request, partition or not.
3. **"A quorum read (R of N) is automatically linearizable."** W+R>N (Dynamo-style) only guarantees the read set overlaps the write set — that gives you regular/read-your-writes consistency, not linearizability, unless it's paired with per-key versioning or vector clocks. (This bridges directly into Lesson 03's replication math.)
4. **"Eventual consistency means 'roughly fine, ignore it'."** The staleness window is a designed number — some systems bound it explicitly (Cloudflare KV, ~60s) and others leave it effectively unbounded under partition or cache lag (K8s watch cache). Always ask for the bound.
5. **"Serializable implies linearizable, or vice versa."** They don't imply each other in either direction: snapshot-isolation reads are serializable-not-linearizable; a single-key compare-and-swap is linearizable-not-serializable.

## Self-check

- Two systems are both called "strongly consistent," but one is serializable-not-linearizable and the other linearizable-not-serializable. Give a concrete example of each and the observable difference. **Answer:** Serializable-not-linearizable = snapshot-isolation reads: multi-key atomic, but a read can return a consistent *past* snapshot (stale). Linearizable-not-serializable = a single-key CAS register: real-time-recent on that key, but there is no multi-key transaction so serializability isn't in scope. Observable difference: the first can read stale-but-atomic across keys; the second is always fresh but only for one key.
- A K8s scheduler double-bound a GPU. Is the fix a stronger read path or something else, and why? **Answer:** Not a stronger read. The stale read came from the eventually-consistent watch cache (`resourceVersion=0`), and forcing quorum reads on the scheduler's hot path would add ~1 RTT per read at 50–100× cost. The fix is optimistic concurrency on the *write*: carry the read `resourceVersion` into the `Binding` write so etcd does a compare-and-swap and rejects (409) the second bind. Reads stay cheap; correctness lives at the guarded write.
- Why is PACELC's "Else" clause where most design decisions actually live, and what does it cost concretely? **Answer:** Partitions are rare; the "Else" (no-partition) case is nearly always in effect, and even then you trade consistency against latency — a quorum/linearizable read costs a network round trip (~1 RTT + apply, e.g. ~2 ms cross-AZ) versus a local cache read (~microseconds). CAP only describes the rare partition corner; PACELC's Else describes the everyday latency-vs-consistency tax.
- An app shows a user their own comment disappearing right after they post it, even though the backing store is "eventually consistent by design." Which session guarantee is missing, and why doesn't fixing it require full linearizability? **Answer:** Missing read-your-writes — the client's own subsequent read should always reflect its own prior write. This is a *client-scoped* guarantee (route the client's reads to the replica it wrote to, or carry a version token), not a global recency guarantee, so it can be fixed cheaply without paying for system-wide linearizability.

## Connections & what's next

This lesson is the axis every later lesson in the module measures against: **Lesson 02** shows how etcd actually *earns* its linearizable label — the Raft write path underneath the guarantee named here — and what that guarantee costs in disk and network terms. **Lesson 03** revisits the W+R>N quorum-overlap idea from the pitfalls section above and shows exactly when it does (and doesn't) add up to linearizability. Carry forward one question into Lesson 02: *if etcd is linearizable "by default," what mechanism inside Raft makes that true, and what does it cost on every write?*

## References & further reading

**Primary sources**
- Herlihy, M. & Wing, J. (1990). *Linearizability: A Correctness Condition for Concurrent Objects*, ACM TOPLAS 12(3):463–492. DOI: https://doi.org/10.1145/78969.78972 — read for the formal atomic-point-between-invocation-and-response definition.
- Gilbert, S. & Lynch, N. (2002). *Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services*. https://groups.csail.mit.edu/tds/papers/Gilbert/Brewer2.pdf — read for the formal proof behind "CAP," the theorem this lesson deliberately de-emphasizes in favor of PACELC.
- Terry, D. et al. (1994). *Session Guarantees for Weakly Consistent Replicated Data* (the Bayou project). http://www.cs.utexas.edu/~lorenzo/corsi/cs380d/papers/SessionGuaranteesBayou.pdf — read for the four session guarantees (read-your-writes, monotonic reads, monotonic writes, writes-follow-reads) used in Core concepts.
- Daniel Abadi — *Consistency Tradeoffs in Modern Distributed Database System Design* (PACELC): http://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf — read for the PACELC formalism this lesson is built around.
- Kubernetes — *API concepts, resourceVersion and cache semantics*: https://kubernetes.io/docs/reference/using-api/api-concepts/ — read for the official semantics of `resourceVersion=0` and quorum reads.

**Real-world engineering blogs**
- Kubernetes GitHub issue #59848 — stale-reads pod-safety bug: https://github.com/kubernetes/kubernetes/issues/59848 — what it shows: the exact double-schedule failure mode this lesson's worked example is built on.
- Google Cloud — *Strict Serializability and External Consistency in Spanner*: https://cloud.google.com/blog/products/databases/strict-serializability-and-external-consistency-in-spanner — what it shows: a real system buying both consistency axes at once, and the commit-wait cost of doing so.
- Cloudflare — *Building With Workers KV*: https://blog.cloudflare.com/building-with-workers-kv/ — what it shows: a production system with an explicit, bounded (~60s) eventual-consistency propagation window.
- Jepsen — etcd 3.4.3 analysis: https://jepsen.io/analyses/etcd-3.4.3 — what it shows: strict-serializable KV operations confirmed under real fault injection, but a per-operation exception (the lock service) that proves "state the guarantee per operation."

**Deeper dives**
- Jepsen — *Consistency Models* (interactive map of the hierarchy): https://jepsen.io/consistency/models
- Kyle Kingsbury (Aphyr) — *Strong consistency models*: https://aphyr.com/posts/313-strong-consistency-models
