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
sources: 20
---

# A01.1 · Consistency models

> **Concept.** Linearizability and serializability are orthogonal axes, CAP is the degenerate corner of PACELC, and the K8s watch cache is the eventual-consistency trap that sits under every GPU scheduler.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits

This is the module's opening lesson. The module's through-line is that **a senior engineer can build a distributed system; a staff engineer can bound it** — state the exact guarantee, its cost, and the failure mode when it's violated — across the **three planes** every GPU platform is made of: control (etcd/K8s), training (gang-scheduled jobs), and serving (SLO-bound inference). Consistency models are where that bounding starts: before you can reason about a scheduler, a checkpoint store, or an inference cache, you need precise vocabulary for "how stale can a read be" and "what does a write actually promise." Every later lesson in this module — consensus (02), replication (03), caching (04) — assumes you can name a system's consistency model on sight and quote the cost of tightening it. Get this lesson solid and the rest of the module reads as applications of it, not new theory.

This lesson does the definitional work *properly*, from operation histories up, because the rest of the module leans on it. Lesson 02 shows the machine (Raft) that produces linearizability and what it costs per write. Lesson 03 shows what you get when you replace consensus with a tunable quorum. Lesson 04 shows a replica that has abandoned correctness entirely in exchange for latency. All three are points in the space this lesson maps.

## Why this matters

At senior level you pick "strong" or "eventual" per datastore and move on. At staff level someone asks *why the scheduler double-booked a GPU* and the answer is a consistency argument: which read path was stale, what guarantee it actually offered, where the write was serialised, and the exact number of milliseconds you'd pay to close the gap. You cannot bound a control plane you can only describe as "strongly consistent." You have to name the model, the isolation axis, and the read path — because those three decide whether a stale read can corrupt state or merely waste a scheduling cycle.

The single most interview-relevant fact in this whole module: **the Kubernetes API server does not serve most reads from etcd.** It serves them from an in-memory watch cache, and every controller, scheduler and informer in your fleet reads from a *further* copy of that cache inside its own process. That is by design — the correctness comes from where writes are serialised, not from the freshness of reads. Kubernetes SIG API Machinery is explicit about the consequence: "every controller, at all times, is operating on a potentially outdated view of the cluster's state" (KEP-5647, *Stale controller handling*, motivation section). If you cannot say precisely what that outdated view can and cannot cause, you cannot review a controller design.

## What's new here (calibration)

**Skip (you already know this):**
- Eventual vs. strong consistency as a binary label.
- That CAP forces a choice, and that it only forces it during an actual partition.
- The one-line intuition that "linearizable = looks like a single copy respecting real-time order."

**The genuinely new depth this lesson adds:**
- Consistency models defined the way the literature defines them — as **sets of legal operation histories** — so that "is this read legal?" becomes a question you can answer mechanically instead of by intuition, with one history drawn under four models.
- Linearizability and serializability are **not** one scale from weak to strong — they're orthogonal axes, and both off-diagonal combinations show up in systems you already run. Also: linearizability is **composable** and serializability is not, which is *why* the distinction survives in practice.
- PACELC's **Else** clause as the tax you pay on every normal request, with the arithmetic attached.
- The actual mechanics of the etcd/Kubernetes split: etcd's MVCC revisions, how `resourceVersion` is derived from them, the full `resourceVersion` / `resourceVersionMatch` semantics table, and — the part most people's mental model is now three years out of date on — the fact that since **Kubernetes v1.31 a consistent LIST is served from the watch cache**, not from a quorum read, using etcd progress notifications.
- Where a Kubernetes write is *actually* serialised (an etcd transaction with a `ModRevision` compare, plus a guard inside the update function), what the scheduler's `Binding` request does and does not carry, and therefore which double-scheduling failures the API server prevents for you and which it does not.

Version note: Kubernetes behaviour below is stated against the current documented API semantics in `kubernetes/website` and the SIG API Machinery KEPs, and against `kubernetes/kubernetes` master source, read directly from the upstream Git repositories in August 2026. etcd behaviour is from the `etcd-io/etcd` and `etcd-io/raft` sources at the same date. Sources that could not be fetched in this environment are marked as such in References and are not relied on for any number here.

## Core concepts

### A consistency model is a set of legal histories, not an adjective

The reason "strongly consistent" arguments go in circles is that people argue about adjectives. The literature does not. A consistency model is defined over **histories**: sequences of operation *invocations* and *responses*, each stamped with the client that issued it and a real-time instant.

Three facts follow from that framing, and they are the whole toolkit:

1. **Every operation has duration.** An operation is not a point; it is an interval `[invocation, response]`. Anything that happens inside that interval is *concurrent* with it. All the interesting freedom in every weak model lives in these intervals.
2. **A model is a predicate over histories.** "Is this system linearizable?" means "for every history it can produce, does there exist an ordering of operations satisfying the model's rules?" This is why consistency checking (Jepsen's Knossos/Elle, Porcupine) is a search problem, and why it is NP-hard in general for linearizability.
3. **Models compose into a hierarchy** because their rule sets are nested. Every linearizable history is sequentially consistent; not the reverse.

The hierarchy, strongest first:

```
strict serializable ⊃ linearizable ⊃ sequential ⊃ causal ⊃ read-your-writes/PRAM ⊃ eventual
```

Each is a strict superset of the *guarantees* below it and a strict subset of the *legal histories*. A stronger model forbids more histories; that is exactly what makes it expensive.

### One history, four models, four different legal answers

Here is the diagram to hold in your head for the rest of the module. One register `x`, initially `0`. Three clients. Operation boxes span from invocation to response; real time runs left to right.

```
 real time ─────────────────────────────────────────────────────────────────────▶
            t0      t1      t2      t3      t4      t5      t6      t7      t8

 C1  (writer)   ┌──────── write(x=1) ────────┐
                │ invoked t1        returns t3│
                └─────────────────────────────┘

 C2  (writer)                   ┌────────── write(x=2) ──────────┐
                                │ invoked t2            returns t5│
                                └─────────────────────────────────┘

 C3  (reader)                                     ┌─ read(x) ─┐      ┌─ read(x) ─┐
                                                  │ t6     t6.5│      │ t7     t8 │
                                                  └────────────┘      └───────────┘

 Facts from the picture:
   · write(x=1) and write(x=2) OVERLAP in real time  → they are concurrent
   · both writes RETURNED before C3's first read was INVOKED (t5 < t6)
   · C3's two reads do not overlap each other: read#1 returns before read#2 starts

 What may each read legally return?

 ┌──────────────────┬──────────────┬──────────────┬────────────────────────────────┐
 │ model            │ read#1 (t6)  │ read#2 (t7)  │ why                            │
 ├──────────────────┼──────────────┼──────────────┼────────────────────────────────┤
 │ linearizable     │ 1 or 2       │ = read#1     │ both writes completed before   │
 │                  │ (concurrent  │ or newer     │ read#1 began, so 0 is illegal; │
 │                  │  writes may  │ (never       │ the two writes may linearize   │
 │                  │  order       │  1 after     │ in either order, but once a    │
 │                  │  either way) │  seeing 2)   │ reader observes an order, real │
 │                  │              │              │ time forbids going back        │
 ├──────────────────┼──────────────┼──────────────┼────────────────────────────────┤
 │ sequential       │ 0, 1 or 2    │ ≥ read#1 in  │ program order per client is    │
 │                  │              │ the chosen   │ respected, REAL TIME IS NOT.   │
 │                  │              │ total order  │ Legal to place both writes     │
 │                  │              │              │ *after* both reads in the      │
 │                  │              │              │ single order → 0, 0 is legal   │
 ├──────────────────┼──────────────┼──────────────┼────────────────────────────────┤
 │ causal           │ 0, 1 or 2    │ ≥ read#1 on  │ the writes are concurrent (no  │
 │                  │              │ any causal   │ happens-before between them),  │
 │                  │              │ chain C3 has │ so different readers may see    │
 │                  │              │ observed     │ them in different orders — but │
 │                  │              │              │ C3 may not un-see what it saw  │
 ├──────────────────┼──────────────┼──────────────┼────────────────────────────────┤
 │ eventual         │ 0, 1 or 2    │ 0, 1 or 2 —  │ no ordering guarantee at all;  │
 │                  │              │ INCLUDING    │ only a liveness promise that   │
 │                  │              │ going        │ if writes stop, replicas       │
 │                  │              │ backwards    │ converge at some unspecified   │
 │                  │              │              │ time                           │
 └──────────────────┴──────────────┴──────────────┴────────────────────────────────┘
```

Read the table twice. Three things in it are the whole lesson:

- **Linearizability's power comes from real time**, and only from real time. It forbids `0` at t6 *because both writes returned at t5*. No weaker model can make that inference, because no weaker model looks at the clock.
- **Sequential consistency is not "linearizability minus a bit."** It permits a reader to see `0` arbitrarily long after both writes acknowledged, as long as *some* single global order exists that respects each client's own program order. A cache that lags by an hour but never reorders one client's operations is sequentially consistent. That is why "sequentially consistent" is a much weaker sales pitch than it sounds.
- **Eventual consistency has no safety property.** Every other row constrains what may be returned. The eventual row constrains nothing about any individual read; it promises only that *if writes stop*, replicas converge. "Reads may go backwards" is not a bug report, it is the specification. This is precisely the anomaly Kubernetes SIG API Machinery calls "going back in time" (KEP-3157, *Watch-List*): a client's LIST served from a stale watch cache can return data *older than what that client already observed*.

### Linearizability, precisely

Herlihy and Wing's 1990 definition: a history is linearizable if each operation appears to take effect atomically at a single instant — its **linearization point** — somewhere between its invocation and its response, and the resulting sequential history is legal for the object's sequential specification.

Two consequences engineers routinely miss:

**It is a single-object property.** "Is my system linearizable" is not a well-formed question. Linearizability is defined per object. etcd's key-value operations were reported linearizable by Jepsen while, in the same analysis, etcd's *lock* API was found unsafe under process pauses. Same system, same version, two different answers, because they are two different objects with two different sequential specifications. **State the guarantee per operation, not per system.**

**It is composable (the "locality" property), and this is why it survives.** If every individual object in a system is linearizable, the system as a whole is linearizable. No other strong model has this. It means you can build a linearizable system out of independently-implemented linearizable parts and reason locally. Serializability has no such property: composing two serializable databases gives you something that is not serializable across them, which is why cross-shard transactions need an extra protocol (2PC, or a global timestamp authority) rather than falling out for free.

The cost of linearizability is a *communication* cost, not a bookkeeping cost. To know that no write is in flight that must precede your read, the node answering the read has to hear from enough of the cluster to rule it out. That is one round trip to a majority, on every read, unless you buy your way out with a lease (a time bound, i.e. a clock assumption) or with a cached revision you can prove is fresh enough (which is the trick Kubernetes now uses — see below).

### Sequential consistency, and the anomaly it permits

Lamport's 1979 definition, written for multiprocessors: the result of any execution is the same as if the operations of all processors were executed in *some* sequential order, and the operations of each individual processor appear in that order in the sequence in the order the program issued them.

Note what is absent: any reference to real time. That single omission is the entire difference from linearizability, and it is why sequential consistency is not composable and why it permits unbounded staleness.

The concrete anomaly, in your own fleet: a controller reads from its informer cache, which is a sequentially-consistent-ish view (it applies the watch stream in order, so it never reorders events, but it can be arbitrarily far behind). It reads `Node/gpu-042` and sees `status.allocatable["nvidia.com/gpu"] = 8`. A human drained and cordoned that node forty seconds ago and the API call returned before the controller's read began. Under linearizability that read is illegal. Under sequential consistency it is perfectly legal — there exists a global order in which the drain happens after this read — and it is what your fleet actually does every day.

### Causal consistency and the mechanism underneath it

Causal consistency says: operations related by **happens-before** must be seen in that order by every process; concurrent operations may be seen in any order, and different processes may pick different orders.

Happens-before (Lamport, 1978) is the transitive closure of three rules: (a) within one process, earlier operations happen-before later ones; (b) a send happens-before the matching receive; (c) transitivity. Two operations neither of which happens-before the other are **concurrent**.

The mechanism that enforces it is a **version vector**: one counter per replica, carried on every write and every message.

```
 Three replicas, vector = [A, B, C]. Each replica increments its own slot on a local write.

   R_A: write x=1        vector [1,0,0]   ──────────┐
                                                     │ replicated
   R_B: receives x=1, merges → [1,0,0]  ◀────────────┘
        then writes y=2 (causally after x=1) → [1,1,0]

   R_C: receives y=2 with vector [1,1,0]
        checks: do I have everything y=2 depends on?
           incoming[A]=1 > local[A]=0  → I am MISSING x=1
        → BUFFER y=2, do not apply it yet
        later receives x=1 [1,0,0] → apply, local becomes [1,0,0]
        → now incoming[A]=1 == local[A] and incoming[B]=local[B]+1 → apply y=2

   Comparison rules:
     V1 ≤ V2  iff  V1[i] ≤ V2[i] for all i          → V1 happens-before V2
     neither V1 ≤ V2 nor V2 ≤ V1                     → CONCURRENT (a real conflict,
                                                       hand it to the application:
                                                       LWW, merge, or sibling values)
```

That buffering step is the whole implementation: a replica delays applying an update until its causal dependencies are locally present. The cost is metadata (one counter per replica per object, so version vectors do not scale to millions of writers — production systems use dotted version vectors, or a per-client "last seen" token instead) and the possibility of unbounded buffering when a dependency is lost.

Why care in a control plane? Because the anomalies causal consistency prevents are exactly the ones that look like corruption in a UI or an audit log: a `Job` object referencing a `ConfigMap` revision that "doesn't exist yet," a status update visible before the spec change that caused it, a reply visible before the comment it replies to. And there is a theoretical reason this level is special: a line of results (Mahajan–Alvisi–Dahlin 2011; Attiya, Ellen, Morrison 2015) establishes that causal consistency is essentially **the strongest model you can provide while remaining available under partition** — everything above it in the hierarchy requires giving up availability during a partition. That is the sharp version of "CAP," and it is more useful than the theorem itself: it tells you the ceiling, not just that a ceiling exists.

### Eventual consistency has no safety property at all

Formally: if no new writes are made to an object, eventually all replicas that can communicate will converge on the same value. That is a **liveness** property with no time bound. It says nothing about any read you actually perform.

This matters because "eventual" in a design document usually means "bounded by something we measured once." Sometimes the bound is published and enforced — Cloudflare Workers KV documents a propagation window of up to roughly a minute between locations. Sometimes it is not bounded at all: a Kubernetes watch cache lagging because the watch stream is clogged has no documented ceiling — which is exactly why KEP-2340 had to add a *timeout and fallback* path when the cache fails to catch up, and why KEP-5647 exists at all. **Always ask for the bound and how it is enforced. "Eventual" without a number and an enforcement mechanism is an unbounded staleness window wearing a nice word.**

### The comparison table to memorise

| Model | What it constrains | Anomaly it still permits | What it costs |
|---|---|---|---|
| **Strict serializable** | Multi-object transaction order *and* real time | Nothing observable; only latency | Consensus + a global time authority (Spanner's commit-wait on TrueTime uncertainty) |
| **Linearizable** | Single-object real-time recency | Nothing on that object; no multi-object atomicity — two linearizable keys can be read at different logical instants | 1 RTT to a majority per read (or a lease, i.e. a clock assumption) |
| **Sequential** | A global total order respecting each client's program order | Unbounded staleness; a read can miss a write acknowledged long ago | Ordered delivery; no round trip needed |
| **Causal** | Happens-before order | Concurrent writes seen in different orders by different readers | Version-vector metadata + dependency buffering |
| **Read-your-writes / PRAM** | One client's own view | Any other client's view; two clients disagree freely | Sticky routing or a client-carried version token |
| **Eventual** | Nothing per-read | Reads going backwards; arbitrary staleness; conflicting values until convergence | Effectively free |

### The other axis: serializability

Serializability comes from the transaction-isolation literature, not the concurrent-objects literature, and it constrains something different: a set of *multi-object transactions* is serializable if the outcome is equal to some serial execution of those transactions. Real time never enters the definition.

Because the two definitions quantify over different things — one object with real time vs. many objects without it — a system can satisfy either without the other:

```
                        LINEARIZABLE  (single-object, real-time recency)
                          no                                yes
                     ┌──────────────────────────┬──────────────────────────────┐
              no     │  MySQL READ COMMITTED    │  etcd / ZooKeeper KV,        │
                     │  DynamoDB eventual reads │  a single CAS register,      │
  S                  │  a bare Redis replica    │  a Raft-backed counter       │
  E                  │                          │                              │
  R                  │  weak on both axes:      │  fresh, but no multi-key     │
  I                  │  stale AND non-atomic    │  atomicity: two keys read    │
  A                  │                          │  at two different instants   │
  L                  ├──────────────────────────┼──────────────────────────────┤
  I           yes    │  Postgres SERIALIZABLE   │  Spanner (external           │
  Z                  │  on a read replica;      │  consistency), CockroachDB,  │
  A                  │  snapshot-based SSI      │  FoundationDB, etcd's        │
  B                  │  reading a consistent    │  STM transactions            │
  L                  │  PAST snapshot           │                              │
  E                  │                          │  = STRICT SERIALIZABLE       │
                     │  atomic across keys,     │  atomic across keys AND      │
                     │  but the snapshot can    │  never stale                 │
                     │  be arbitrarily stale    │                              │
                     └──────────────────────────┴──────────────────────────────┘
```

The two off-diagonal boxes are the interview answer. Bottom-left: a serializable-but-not-linearizable system can hand you a perfectly self-consistent view of the world as it existed ten seconds ago. Top-right: a linearizable-but-not-serializable system always tells you the truth about one key and cannot tell you a coherent story about two.

And the isolation ladder underneath serializability matters for the same reason: **snapshot isolation is not serializable**, and the counter-example is write skew. Two transactions each read the same set of rows, each check an invariant that currently holds, and each write a *different* row. Neither writes what the other read, so no first-committer-wins check fires, and both commit — leaving the invariant violated. The canonical instance: two on-call engineers each check "at least one person is on call," each see two people on call, each remove themselves. In your fleet: two quota controllers each read "cluster has 12 free GPUs," each admit an 8-GPU job into different namespaces, and now 16 GPUs are promised out of 12. Serializable isolation (Postgres SSI, or an explicit `SELECT … FOR UPDATE` on a shared row) prevents it; snapshot isolation does not.

### Session guarantees: the middle ground, and how they are actually implemented

Between "linearizable everywhere" and "eventual" sits a set of guarantees that bind only *one client's own view*. They are cheap precisely because they need no global agreement (Terry et al., the Bayou project, 1994):

| Guarantee | Promise | Bug it prevents | Typical implementation |
|---|---|---|---|
| **Read-your-writes** | A client sees its own prior writes | User posts a comment, refreshes, comment is gone — looks like data loss | Route the client's reads to the replica it wrote to (sticky session), or carry the write's version token and require the reader to be at least that fresh |
| **Monotonic reads** | Successive reads never go backwards | Counter shows 50, then 48, then 50 — looks like corruption | Sticky routing to one replica; or the client remembers the highest version it has seen and refuses/retries older |
| **Monotonic writes** | A client's writes apply in issue order | `status=shipped` then `status=processing` land out of order; final state wrong | Per-client sequence numbers applied in order at the replica |
| **Writes-follow-reads** | A write issued after reading V is ordered after V | A reply is ordered before the comment it replies to | Attach the read's version as a causal dependency of the write |

The reason to teach these with mechanism attached: **you already run one.** Kubernetes' `resourceVersionMatch=NotOlderThan` is exactly a client-carried version token implementing monotonic reads. A controller that stores the last `resourceVersion` it observed and passes it on the next LIST is buying monotonic reads for a fraction of the cost of a consistent read, and the API server documentation actively recommends this over an unset `resourceVersion` "unless you have strong consistency requirements," because `NotOlderThan` is served from the watch cache while an unset `resourceVersion` historically required a quorum read.

Most application bugs blamed on eventual consistency are missing *session* guarantees, not a missing global one. Reaching for linearizability to fix a read-your-writes bug buys correctness the app did not need at a latency cost every request then pays.

### PACELC, with the arithmetic

CAP's actual statement (Gilbert & Lynch's formalisation of Brewer's conjecture) is narrow: in an asynchronous network where messages can be lost, no implementation of a read/write register can guarantee both availability and atomic (linearizable) consistency. It says nothing about the 99.99% of time when there is no partition.

Abadi's PACELC fixes that: *if* **P**artition, choose **A**vailability or **C**onsistency; **E**lse, choose **L**atency or **C**onsistency.

The Else clause is where your money goes. Price it:

```
  A read served from the local process cache (informer / watch cache):
      no network, no disk                                    ≈ 10–100 µs

  A read that must prove freshness through consensus:
      1 RTT to a majority + apply time
      same-AZ RTT ~0.2–0.5 ms, cross-AZ RTT ~1–2 ms          ≈ 1–3 ms typical

  Ratio: 20× to 200×, per read, on every hot path.
```

Now attach it to a fleet. A moderately busy control plane sustains 5,000 reads/s across schedulers, controllers, and kubelets. At 2 ms per consistent read versus 50 µs per cached read, the delta is `5,000 × (2 ms − 0.05 ms) = 9.75 s` of added latency accrued **per wall-clock second**. You would need roughly ten additional seconds of concurrency every second to absorb it — i.e. it is not a tuning question, it is an architecture question. That arithmetic is why the Kubernetes API server has a watch cache at all, and why the answer to a stale-read bug is almost never "make the reads consistent."

| System | PACELC | What that means concretely |
|---|---|---|
| etcd, ZooKeeper | **PC/EC** | Minority partition refuses service; every consistent read pays a majority round trip (or a lease) |
| Spanner | **PC/EC** | Consensus per split, plus commit-wait on TrueTime uncertainty (single-digit ms) for external consistency |
| Dynamo, Cassandra (default) | **PA/EL** | Accepts writes on either side of a partition; normally serves from the nearest replica and reconciles later |
| Kubernetes watch cache | **PA/EL** on the read path, **PC/EC** on the write path | Reads keep being served (possibly stale) when etcd is unreachable; writes stop entirely |

That last row is the shape of every good control plane: cheap eventual reads, expensive linearizable writes, and correctness placed at the write.

### The GPU control plane, mechanism by mechanism

Now the concrete stack you operate. Everything below is a real mechanism with real defaults.

```
   ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
   │  kube-scheduler  │   │ device-plugin    │   │  your operator   │
   │  informer cache  │   │ controller       │   │  informer cache  │
   │  (in-process,    │   │ informer cache   │   │                  │
   │   eventually     │   └────────┬─────────┘   └────────┬─────────┘
   │   consistent)    │            │                      │
   └────────┬─────────┘            │                      │
            │  LIST rv=0 (bootstrap) + WATCH (incremental) │
            └──────────────┬───────┴──────────────────────┘
                           │
              ┌────────────▼──────────────────────────────────────────┐
              │                   kube-apiserver                      │
              │                                                       │
              │   ┌───────────────────────────────────────────────┐   │
              │   │ watch cache — one per resource type           │   │
              │   │  · full object store + a ring buffer of       │   │
              │   │    recent events (targets ~75 s of history)   │   │
              │   │  · fed by ONE etcd watch per resource         │   │
              │   │  · serves: rv=0 (Any), NotOlderThan,          │   │
              │   │            and (since v1.31) consistent LIST  │   │
              │   └──────────────▲────────────────────┬───────────┘   │
              │                  │ watch events        │ progress req  │
              │                  │                     │ (every 100 ms │
              │                  │                     │  while a      │
              │                  │                     │  consistent   │
              │                  │                     │  read waits)  │
              │   ┌──────────────┴─────────────────────▼───────────┐   │
              │   │ storage layer: Range / Txn / Watch over gRPC   │   │
              │   └──────────────────────┬─────────────────────────┘   │
              └──────────────────────────│─────────────────────────────┘
                                         │
                        ┌────────────────▼──────────────────┐
                        │            etcd cluster           │
                        │  MVCC: one monotonic revision per │
                        │  committed write, cluster-wide.   │
                        │  Each key holds CreateRevision,   │
                        │  ModRevision, Version.            │
                        │  Reads: serializable (local) or   │
                        │  linearizable (ReadIndex) —       │
                        │  see lesson 02 for the machinery. │
                        └───────────────────────────────────┘
```

**Where `resourceVersion` comes from.** etcd assigns one monotonically increasing **revision** per committed write, cluster-wide. A key's `ModRevision` is the revision at which it was last modified. The Kubernetes storage layer surfaces an object's `metadata.resourceVersion` as that key's `ModRevision`, and a *list's* `metadata.resourceVersion` as the store revision at which the list was constructed. Hence the standing advice: `resourceVersion` is an opaque token — compare for equality, use it to resume, do not do arithmetic on it. (Since v1.35 the API defines an ordering comparison within a single resource type, with a `CompareResourceVersion` helper in apimachinery; it is still not a timestamp and still not comparable across resource types.)

**The read-path semantics table**, reproduced from the Kubernetes API concepts reference so you never have to open it again:

*get:*

| `resourceVersion` unset | `resourceVersion="0"` | `resourceVersion="<other>"` |
|---|---|---|
| Most recent | Any | Not older than |

*list (with `resourceVersionMatch`):*

| `resourceVersionMatch` | paging | rv unset | rv="0" | rv="\<other\>" |
|---|---|---|---|---|
| *unset* | no limit | Most recent | Any | Not older than |
| *unset* | limit set, no continue | Most recent | Any | Exact |
| *unset* | continue token | Continuation | Continuation | Invalid (400) |
| `Exact` | any | Invalid | Invalid | Exact |
| `NotOlderThan` | any | Invalid | Any | Not older than |

And what those four words mean:

- **Any** — served from the watch cache, no freshness guarantee at all. May return data *older than the client has already seen*, especially across API server replicas behind a load balancer, each with an independently lagging cache.
- **Most recent** — a consistent read. Historically this meant a quorum read to etcd. **This is the part most engineers' model is stale on:** with `ConsistentListFromCache` (alpha v1.28, beta and on by default since **v1.31**, requiring etcd ≥ v3.4.31 or ≥ v3.5.13), a consistent **LIST** is served from the watch cache instead. Consistent **GET** still goes to etcd, per KEP-2340.
- **Not older than** — served from the watch cache, but the API server blocks until the cache has reached at least the supplied revision. This is the monotonic-reads token.
- **Exact** — pinned to one revision; `410 Gone` if it has been compacted away. With `ListFromCacheSnapshot` the API server can answer from an in-memory snapshot rather than etcd.

**How a consistent LIST is served from a cache without lying.** This is the mechanism worth knowing cold, because it is the general trick for "cheap reads with a real guarantee," and it is the same trick as Raft's `ReadIndex` (lesson 02) one layer up:

1. The API server asks etcd for the current revision `R` for that resource.
2. If the watch cache has already applied everything up to `R`, serve immediately from memory. Done — no quorum read of the actual data, which for a 150k-pod cluster is the difference between shipping megabytes out of etcd and reading an in-memory index.
3. If not, register a "waiting read" and block. A background goroutine sends etcd a `WatchProgressRequest` **every 100 ms** until the watch stream delivers a progress notification at or beyond `R`. etcd's contract for a progress notification is exactly the guarantee needed: all events with revision ≤ the notified revision have already been delivered on this watch.
4. If the cache does not catch up within the timeout, **fall back to etcd**. The fallback rate is exported as `apiserver_watch_cache_consistent_read_total{fallback="true"|"false"}`, and the wait time as `apiserver_watch_cache_read_wait` — the KEP's own target was p99 wait under 200 ms, i.e. two progress-poll periods.

Two operational notes fall straight out. First, this is why the etcd version requirement is hard: etcd issue #15220 was a race that could deliver a progress notification *before* the event at the same revision, which would let the API server serve a consistent read that silently missed a write. Fixed in etcd v3.4.25 / v3.5.8; Kubernetes refuses to enable the feature against older etcd. Second, `apiserver_watch_cache_consistent_read_total{fallback="true"}` climbing is a *leading indicator* that your watch caches are falling behind, and it will show up before anything user-visible does.

**The staleness bounds you can actually quote:**

| Bound | Value | Where it comes from |
|---|---|---|
| Watch-cache event history | targets ~75 s of events | API server watch cache ring buffer / snapshot retention |
| etcd compaction interval | 5 m default | `kube-apiserver --etcd-compaction-interval=5m0s` |
| Progress-poll interval while a consistent read waits | 100 ms | KEP-2340 algorithm |
| Informer resync period | whatever your controller sets (commonly 10 m or 0 = never) | client-go `SharedInformerFactory` |
| Watch cache lag | **unbounded** | no mechanism bounds it; hence the fallback path and KEP-5647 |

That last row is the one to say out loud in a design review. There is no guaranteed ceiling on informer-cache lag. `410 Gone` on a watch resume is the *cure* (the client must re-LIST), not the bound.

### What a stale read can and cannot cause — with the interleaving

Now the payoff. Here is the failure, drawn as a timeline, with the exact place the system is saved and the exact place it is not.

```
 Two schedulers (default kube-scheduler + a custom GPU scheduler, or one scheduler
 restarting and losing its assumed-pod cache). Node gpu-042 has 8 GPUs, 8 already
 requested by pod-A which was bound 300 ms ago.

  t=0     S1  bind(pod-A → gpu-042)
              POST /api/v1/namespaces/ns/pods/pod-A/binding
              body: Binding{ObjectMeta{Namespace,Name,UID}, Target: Node/gpu-042}
                    ── note: UID is set; resourceVersion is NOT ──
  t=0+ε   apiserver → etcd Txn:
              compare  ModRevision(key) == <read revision>     ← the real CAS
              guard    if pod.Spec.NodeName != "" → reject
                       ("pod pod-A is already assigned to node gpu-042")
              then     set pod.Spec.NodeName = "gpu-042"
          COMMIT at revision 90210

  t=5ms   apiserver watch cache applies rev 90210
  t=?ms   S2's informer receives the event ............ SOMETIME. Unbounded.

  t=20ms  S2 scores nodes from ITS cache, which still shows gpu-042 with
          8 free GPUs (it has not applied rev 90210 yet)
  t=25ms  S2  bind(pod-B → gpu-042)
              apiserver Txn: pod-B's NodeName is "" → guard PASSES
              COMMIT at revision 90211           ← ✗ NOT PREVENTED

  t=60ms  kubelet on gpu-042 sees pod-B, runs admission:
              requested nvidia.com/gpu=8, allocatable 8, already allocated 8
          → REJECT: Pod status Failed, reason "OutOfnvidia.com/gpu"
                    (kubelet builds the reason as "OutOf" + resource name)
          → ✓ contained here, at the node, not at the API server
```

Read that carefully, because the popular version of this story is wrong in a way that matters:

- **What the API server's compare-and-swap prevents:** two writes to *the same object* racing. The etcd transaction compares the key's `ModRevision`, so a lost update on the pod is impossible; and the binding handler additionally refuses any bind to a pod whose `spec.nodeName` is already set, and refuses to bind a pod that is being deleted or that still has scheduling gates. The scheduler's `Binding` also carries the pod's **UID** as a precondition, which is what stops a bind landing on a *different* pod that was deleted and recreated with the same name.
- **What it does not prevent:** two *different* pods being bound to the same node's exhausted resources. Nothing in the pod-A transaction constrains the pod-B transaction — they are different keys. Kubernetes has no cross-object transaction here.
- **Where over-commitment is actually serialised:** in two places, neither of them etcd. (1) Inside a single scheduler process, `assumeAndReserve` writes the assumed pod into the scheduler's own cache *before* the bind API call returns, so a single scheduler never double-books against its own decisions. (2) At the node, where kubelet admission and the device manager are the final arbiter and fail the pod with `OutOfnvidia.com/gpu`.
- **Therefore the real blast radius of a stale read here is a wasted scheduling cycle plus a failed pod** — bad, visible, recoverable. It becomes *corruption* only where you have built a resource with no node-level arbiter: an external quota counter, a leased IP range, a licence server, a "GPU-hours consumed" ledger. Those are the objects where you must supply the serialisation point yourself, with a compare-and-swap on a single object that all writers contend on, or with a lease.

**The general rule to carry out of this lesson:** *for every write in your design, name the single object whose compare-and-swap serialises it.* If you cannot name one, the write is not serialised, and no amount of read consistency will save it.

## Perspectives

**The theory / algorithm-designer view.** Linearizability and serializability are distinct correctness conditions with different quantifiers — one object plus real time versus many objects without real time — which is why both off-diagonal combinations exist and why only linearizability composes. Composability is not an academic nicety: it is the reason you can reason about a system of a thousand linearizable keys one key at a time, and the reason a cross-shard transaction needs an explicit protocol bolted on. Treating "strong consistency" as one dial is the most common conceptual bug brought into a staff-level design interview.

**The operator / SRE view.** Staleness is measurable, and the measurements have names. Watch for `apiserver_watch_cache_consistent_read_total{fallback="true"}` (consistent reads that had to give up on the cache and hit etcd), `apiserver_watch_cache_read_wait` (how long they blocked), `etcd_server_slow_read_indexes_total` and `etcd_debugging_mvcc_db_total_size_in_bytes`, and `resourceVersion` skew between API server replicas behind the load balancer — each replica's watch cache lags independently, so a client that round-robins across replicas can observe time going backwards even when every replica is individually healthy. An SRE with those four signals localises a "stale scheduling" incident in minutes instead of hypothesising a network partition.

**The application / controller-author view.** Most bugs blamed on eventual consistency are session-guarantee bugs. If your controller writes an object and then immediately re-reads it from its own informer, you have written a read-your-writes bug; the fix is to read from the API (a `Get` with `resourceVersion` unset is "most recent") or to track the version you wrote and refuse to act until the informer catches up — which is precisely what KEP-5647 proposes to standardise. Do not fix it by making every read consistent.

**The economics / latency view.** PACELC's Else branch has a per-request price and therefore a capacity price. At 5,000 reads/s a 2 ms consistent read versus a 50 µs cached read costs ~9.75 s of aggregate latency per second of wall clock — real fleet capacity. Consistent-LIST-from-cache is interesting precisely because it changes this arithmetic: you pay one small revision round trip and reuse it across every waiting reader, instead of streaming the data itself out of the consensus group. Cheap guarantees are usually built this way — amortise the proof, not the payload.

## Real-world use cases

- **Kubernetes issue #59848 — "vulnerable to stale reads, violating critical pod safety guarantees"** (<https://github.com/kubernetes/kubernetes/issues/59848>). Referenced as an explicit goal to resolve by two separate KEPs, so its substance is verifiable even without the issue thread: KEP-3157 (*Watch-List*) states that "going back in time" can happen when a reflector's initial LIST "is served from a stale watch cache with data much older than the reflector has previously observed or if the api-server or etcd are partitioned," and notes that the watch cache did not support `resourceVersion=""` and was therefore "vulnerable to stale reads." KEP-2340 lists resolving it as goal #1. The failure is a monotonic-reads violation at the informer boundary: a client that already saw revision N restarts its reflector, LISTs with `rv=0`, and receives state from before N — and then acts on it. This is the mechanism, not an anecdote.
- **Why Kubernetes kept the stale path for so long — the scale number.** KEP-2340's motivation gives the arithmetic that made "just do consistent reads" impossible: in a 5,000-node cluster at 30 pods/node, each kubelet LISTing its own pods forces the API server to read **150,000 pods from etcd** and filter down to the ~30 it needs — for each of 5,000 kubelets. Served from the watch cache with a built-in index, the same request touches only the 30. That ratio is why the eventual-consistency read path exists, and why the fix was to make the cache provably fresh rather than to bypass it.
- **KEP-5647, *Stale controller handling* (SIG API Machinery).** The current, live acknowledgement that this is unsolved at the controller layer: "A change event might arrive within milliseconds, or under other circumstances, could be delayed by seconds or even minutes… operators currently have no visibility into this lag." The proposal is opt-in: expose the cache's resource version to the controller so it can refuse to reconcile until its cache is at least as new as its own last write. If you own a controller that binds scarce resources, this is the pattern to implement yourself today rather than wait for.
- **Spanner's external consistency (Google Cloud).** The reference implementation of the top-right box: strict serializability bought with TrueTime — a bounded-uncertainty clock — plus **commit-wait**, where a transaction deliberately delays its commit until the uncertainty interval has passed so that its timestamp is guaranteed to be in the past everywhere. It is the clearest demonstration that real-time order is purchasable, and that its price is latency measured in the width of your clock uncertainty. *(Not re-fetched this pass — see References.)*
- **Cloudflare Workers KV.** A production store that publishes its staleness window rather than hiding it: writes are immediately visible at the writing location and propagate globally within a documented window on the order of a minute. Useful as the contrast case — an eventual system with a *stated* bound is a design; one without is a hazard. *(Not re-fetched this pass — see References.)*
- **Jepsen's etcd 3.4.3 analysis.** Reported etcd's key-value operations as holding up to its strict-serializable claim under partitions, pauses and clock skew, while separately finding the *lock* API unsafe in an asynchronous network. The lesson is the per-operation rule: "etcd is linearizable" was true of the KV API and false of the lock API at the same version. *(Not re-fetched this pass — see References.)*

## Worked example

**Setup.** A GPU fleet: 3-node etcd, 3 API server replicas behind a load balancer, ~150,000 pods, and a custom GPU scheduler that maintains an informer cache and binds pods with the standard `Binding` subresource. Cross-AZ RTT is 2 ms; the scheduler sustains ~4,000 reads/s against its cache during a submission burst.

**Q1 — Classify every read and write path in one table.**

| Path | Call | Model in force | Bound on staleness |
|---|---|---|---|
| Scheduler scoring nodes | informer cache, in-process | eventual (monotonic within one informer) | none |
| Informer bootstrap | `LIST rv=0` → "Any" | eventual, may go backwards | none |
| Informer bootstrap, hardened | `LIST rv=<last seen>, resourceVersionMatch=NotOlderThan` | monotonic reads | blocks until cache ≥ token |
| Pre-bind sanity check | `GET` with `resourceVersion` unset → "Most recent" | linearizable per object (etcd quorum read) | 0 |
| Controller full scan | `LIST` with `resourceVersion` unset → "Most recent" | linearizable snapshot of that collection, served from cache via progress notification (v1.31+) | 0, or falls back to etcd |
| Bind | `POST …/binding` | linearizable write: etcd `Txn` with `ModRevision` compare, `UID` precondition, `nodeName==""` guard | n/a |

**Q2 — What can the stale read cause, and what can it not?**

*Can cause.* (a) A wasted scheduling cycle — the scheduler picks a node whose GPUs are already claimed by a bind it has not observed; kubelet fails the pod with `OutOfnvidia.com/gpu` and it re-queues. (b) A "going back in time" event after a scheduler restart: the new leader LISTs with `rv=0`, gets a view older than the state it had already acted on, and re-attempts binds for pods it had already assumed. (c) A stale `Node` object leading to scheduling onto a node that was cordoned before the decision started.

*Cannot cause.* (a) A lost update on a pod — the etcd transaction compares `ModRevision`, so a concurrent writer cannot be silently overwritten. (b) A bind landing on the wrong incarnation of a name — the `Binding` carries the pod `UID` as a precondition, so a delete-and-recreate between read and write fails the write rather than binding the new pod. (c) A double *bind of the same pod* — the handler's `spec.nodeName != ""` guard rejects it with `pod <name> is already assigned to node <node>`.

*The gap that is on you.* Any resource whose exhaustion is **not** re-checked by a node-level arbiter — an external quota ledger, an IP or licence pool, a "GPU-hours" counter — has no second line of defence. For those, the design rule is: one object per contended pool, and every writer performs a compare-and-swap against that object's `resourceVersion` (or an equivalent CAS in whatever store holds it). If two writers can succeed without contending on a single key, your invariant is decoration.

**Q3 — Price the three candidate fixes.**

*Option A — make the scheduler's reads consistent.* Every scoring cycle reads through to a quorum. Cost: at 4,000 reads/s and ~2 ms per consistent read versus ~50 µs cached, `4,000 × 1.95 ms = 7.8 s` of added latency per second of wall clock, plus 4,000 extra ops/s of load on a 3-node etcd whose own commit path is disk-bound (lesson 02). **Rejected on arithmetic**, before any design argument.

*Option B — bound the staleness where it matters.* Keep cheap cache reads for scoring, but before the bind, issue one consistent `GET` on the target `Node` and the pod. Cost: **one** consistent read per binding decision, not per scoring read. At 50 binds/s that is `50 × 2 ms = 100 ms/s` — 0.01% of a core-second per second, i.e. free. This is the general shape of the right answer: *push the strong read to the decision point, not the evaluation loop.*

*Option C — fix the restart hazard.* Have the scheduler persist the highest `resourceVersion` it has acted on and bootstrap with `resourceVersion=<that>&resourceVersionMatch=NotOlderThan` instead of `rv=0`. Cost: the API server blocks the LIST until its watch cache reaches that revision — in a healthy cluster, single-digit milliseconds; the exposure is that a genuinely lagging API server replica now makes you wait rather than lie. **That trade is correct for a scheduler** and is exactly the mechanism KEP-5647 proposes to generalise.

**Q4 — Availability arithmetic on the read path.** Three API server replicas, each with an independent watch cache. Suppose each replica's cache is more than 5 s behind for 0.1% of the time, independently. A client that pins to one replica sees a stale-by-5s view 0.1% of the time. A client that round-robins across all three has `1 − (1 − 0.001)³ ≈ 0.3%` chance that *some* read in a three-read sequence is served from a lagging replica — and, worse, its reads can now go *backwards* between replicas even though each replica is monotonic on its own. **The fix is not more replicas; it is a session token.** `NotOlderThan` with the client's high-water-mark makes the replicas' independent lag invisible, because any replica that is behind blocks instead of answering.

**Q5 — What does 5 s of watch-cache lag cost in scheduling terms?** With a burst of 500 pod submissions and a 5 s lag, the scheduler makes decisions against a node view up to 5 s old. At 50 binds/s that is up to 250 binds' worth of unobserved capacity change — enough to over-subscribe an entire 8-GPU node many times over. The number that matters for your alert threshold is therefore not the lag in seconds but `lag × bind_rate` = decisions made blind. Set the alert on the product, and you will alert on the thing that hurts.

## Practice

*Feeds the [staff design portfolio](../practice/staff-design-portfolio/README.md).*

1. **Placement matrix (artifact: `consistency-placement-matrix`).** Build a table classifying at least 8 systems you actually run — etcd, your primary RDBMS, Redis, Kafka, S3, the K8s API server, any Cassandra/Dynamo-family store, your object-storage-backed checkpoint store — on *both* axes: (a) PACELC class, (b) linearizable? serializable? both? neither? — with one sentence of evidence each. **Then add two columns most people forget:** the read path's staleness bound *and how it is enforced*, and the single object whose compare-and-swap serialises writes. Flag every row where the read path and write path have different guarantees; those rows are where your incidents will come from.
2. **Stale-read blast-radius note (artifact: `etcd-stale-read-blast-radius-note`).** For a controller or scheduler you own, write the two-column "a stale read *can* cause / *cannot* cause" analysis in the form used in the Worked example. Requirements: name the specific API call and its `resourceVersion` semantics for each read; for each write, name the object whose CAS serialises it, or write "none — unserialised" and treat that line as a finding. Finish with the node-level or downstream arbiter that catches the failure, or state that there isn't one.
3. **Measure the delta.** On a test cluster, time `kubectl get pods -A` (consistent LIST) against a `resourceVersion=0` LIST issued with `kubectl get --raw`. Then scrape `apiserver_watch_cache_consistent_read_total` and `apiserver_watch_cache_read_wait` and record the fallback rate. State the read-QPS above which consistent-everywhere is infeasible for your fleet, showing the arithmetic.
4. **Break it deliberately.** Write a two-client script against any store you run that produces a monotonic-reads violation (read 50, then 48) by pinning reads to different replicas. Then fix it twice — once with sticky routing, once with a client-carried version token — and note which one you would actually ship and why.

## Common pitfalls

1. **Treating "strongly consistent" as one dial.** It collapses two orthogonal axes: single-object real-time recency (linearizability) and multi-object transaction atomicity (serializability). Symptom: a design review where nobody can say whether two keys are read at the same instant. Mechanism: the two properties are defined over different quantifiers, so neither implies the other. Ask "linearizable on what object, serializable across what set."
2. **"CAP means you always trade C against A."** CAP binds only during an actual partition, which is rare. The everyday trade is PACELC's Else: consistency against latency, on every request. Symptom: a team that has never priced its read path arguing about partitions it has not had.
3. **"Kubernetes consistent reads are quorum reads."** True until v1.31. Since then, a consistent **LIST** is served from the watch cache using etcd progress notifications, with a fallback to etcd on timeout; only a consistent **GET** still goes through to etcd. If your mental cost model still says "consistent LIST = quorum read of every object," you will over-estimate the cost of correctness and reach for `rv=0` where you did not need to.
4. **"Eventual consistency means roughly fine."** It is a liveness property with no per-read guarantee and, in the Kubernetes watch cache, no bound at all. Symptom: an "impossible" bug where a controller acts on state older than what it had already seen. Mechanism: reflector re-LIST with `rv=0` after a restart or a `410 Gone`, served by a lagging replica.
5. **"The optimistic-concurrency check on the write makes stale reads safe."** It makes them safe *for that object*. It does nothing for an invariant spanning objects — two different pods bound to the same exhausted node pass their own CAS checks independently. Symptom: over-commitment that no `409 Conflict` ever reported. Mechanism: no cross-object transaction; the guard lives in one key's transaction.
6. **"Snapshot isolation is serializable."** It is not; write skew is the counter-example, and it is the exact shape of a quota-admission bug: two transactions read the same free-capacity view, check the same invariant, and write different rows. Symptom: quota exceeded with no conflicting write in the audit log.
7. **"A quorum read (R of N) is automatically linearizable."** `W + R > N` guarantees only that the read set intersects the last completed write set. Without per-key versioning and a repair step it gives you regular-register semantics, not linearizability — and sloppy quorums void even that. Lesson 03 does this arithmetic properly.

## Self-check

- **Two systems are both "strongly consistent," one serializable-not-linearizable and one linearizable-not-serializable. Give an example of each and the observable difference.**
  **Answer:** Serializable-not-linearizable: a snapshot-isolation/SSI read on a Postgres replica, or any consistent-snapshot read. Multi-key atomic, but the snapshot may be arbitrarily far in the past, so a value written and acknowledged a minute ago may be invisible. Linearizable-not-serializable: a single etcd key under compare-and-swap, or a Raft-backed counter. Always reflects the latest committed write on that key, but there is no multi-key transaction, so reading two keys gives you two different logical instants and no invariant across them. Observable difference: the first can return a self-consistent view of a stale world; the second returns the truth about one key and cannot tell a coherent story about two. Bonus point for saying that only linearizability is composable — a system of linearizable objects is linearizable, whereas composing serializable stores is not serializable across them.

- **Draw the four-model table for one history, in your own words.** **Answer:** Two writes overlap in real time and both return; a reader then reads twice, non-overlapping. Linearizable: read#1 may return either write's value but not the initial value, because both writes returned before it was invoked; read#2 may not go backwards. Sequential: read#1 may return the initial value, because there exists a total order (respecting each client's program order) that places both writes after both reads — real time is not consulted. Causal: same freedom, because the two writes are concurrent (no happens-before), but one reader may not observe them in one order and then the other. Eventual: anything, including read#2 returning a value older than read#1; the only promise is that if writes stop, replicas converge, with no time bound.

- **A GPU scheduler double-booked a node. Trace where each defence sits, and which one failed.** **Answer:** Scoring read comes from an eventually-consistent informer cache with no staleness bound, so the scheduler can score against a view that predates a recent bind. The bind is a `POST` to the pod's `binding` subresource; the API server executes an etcd transaction that compares the pod key's `ModRevision`, enforces the `Binding`'s `UID` precondition, and rejects if `spec.nodeName` is already set — so lost updates, wrong-incarnation binds and double-binding *the same pod* are all impossible. What is *not* prevented is two different pods binding to the same exhausted node, because those are two transactions on two different keys with no cross-object constraint. The remaining defences are the scheduler's own assumed-pod cache (only effective within one process, so a restart or a second scheduler defeats it) and kubelet admission, which fails the pod with reason `OutOfnvidia.com/gpu`. So the failure is contained at the node, and the cost is a wasted cycle plus a failed pod — unless the contended resource has no node-level arbiter, in which case you must supply the single-object CAS yourself.

- **Why is making the scheduler's reads consistent the wrong fix, in numbers?** **Answer:** A cached read is ~10–100 µs; a consistent read is ~1–3 ms (one majority round trip on a 2 ms cross-AZ link, plus apply). At 4,000 scoring reads/s the delta is roughly `4,000 × 1.95 ms ≈ 7.8 s` of added latency per wall-clock second, plus thousands of extra ops/s against a disk-bound etcd. The right fix moves the strong read to the *decision* point: one consistent `GET` per bind at ~50 binds/s costs ~100 ms/s total. Additionally, harden the bootstrap with `resourceVersionMatch=NotOlderThan` carrying the highest revision the scheduler has already acted on, which converts an unbounded stale-read hazard into a bounded wait.

- **What exactly does `resourceVersionMatch=NotOlderThan` buy, and what does it cost?** **Answer:** It is a client-carried version token implementing monotonic reads: the API server serves from its watch cache but blocks until that cache has applied at least the supplied revision, so the client can never see a view older than one it already observed — including across API server replicas with independently lagging caches. It does not give you the *latest* state, only "not older than the token." Cost: a wait proportional to that replica's lag, and a failure mode where a badly lagging replica makes you slow instead of wrong. It is the cheap fix for the class of "informer went back in time after a restart" bugs.

- **How can a consistent LIST be served from a cache without lying?** **Answer:** Amortise the proof, not the payload. The API server asks etcd for the current revision `R` for that resource; if the watch cache has already applied through `R`, it answers from memory. If not, it registers a waiting read and polls etcd with `WatchProgressRequest` every 100 ms until the watch stream reports progress at or past `R` — etcd's progress notification guarantees every event at or below that revision has already been delivered. On timeout it falls back to a real etcd read. Alpha in v1.28, beta and default-on in v1.31, requires etcd ≥3.4.31/3.5.13 because an earlier progress-notification race (etcd #15220) could deliver the notification before the event at the same revision, which would silently corrupt the guarantee. Observability: `apiserver_watch_cache_consistent_read_total{fallback}` and `apiserver_watch_cache_read_wait`. Note the structural similarity to Raft's `ReadIndex` (lesson 02): confirm a revision cheaply, then wait for local state to reach it.

- **An app shows a user their own comment disappearing right after posting. Which guarantee is missing, and why doesn't the fix need linearizability?** **Answer:** Read-your-writes. It is a *client-scoped* guarantee: route that client's reads to the replica that accepted its write, or have the client carry the write's version token and require any serving replica to be at least that fresh. Both cost approximately nothing and bind only one client's view. Global linearizability would also fix it, at a per-request round-trip cost paid by every client for a guarantee only this one needed.

## Connections & what's next

This lesson is the axis every later lesson measures against. **Lesson 02** shows how etcd *earns* the linearizable label — the Raft machinery, the real RPCs, and why `ReadIndex` is the same "amortise the proof" trick you just saw in the API server's consistent-read-from-cache path — and what it costs in fsyncs and round trips on every write. **Lesson 03** takes the `W + R > N` claim from the pitfalls list and does the arithmetic, showing exactly when quorum overlap does and does not add up to a real guarantee. **Lesson 04** takes the eventual end of the hierarchy to its logical extreme: a replica that has given up on correctness entirely in exchange for latency, and what that does to the system behind it. **Lesson 05** picks up the queueing consequence of every "wait until the cache catches up" decision you just made.

Carry one question into Lesson 02: *if etcd is linearizable "by default," what mechanism inside Raft makes that true, what does it cost on every write, and why is the cost measured in disk fsyncs rather than network hops?*

## References & further reading

**Primary sources — verified against upstream Git repositories this pass**

1. **Ongaro, D. & Ousterhout, J., *In Search of an Understandable Consensus Algorithm* (Raft), extended version** — `raft.pdf` in <https://github.com/raft/raft.github.io>. Read directly from the repository (the raft.github.io website itself is blocked by this environment's egress proxy). Used here only for the terminology of terms and commitment; lesson 02 uses it in depth.
2. **Kubernetes API concepts reference** — `content/en/docs/reference/using-api/api-concepts.md` in <https://github.com/kubernetes/website>. Read from the repository (kubernetes.io is blocked here). **Source of** the `resourceVersion` / `resourceVersionMatch` semantics tables reproduced above, the Any / Most recent / Not older than / Exact definitions, the statement that "most recent" reads are served from the watch cache for etcd ≥3.4.31/3.5.13 with Kubernetes ≥1.31, and the ~75 s snapshot-retention window.
3. **KEP-2340, *Consistent Reads from Cache*** — `keps/sig-api-machinery/2340-Consistent-reads-from-cache/README.md` in <https://github.com/kubernetes/enhancements>. **Source of** the four-step algorithm, the 100 ms `WatchProgressRequest` poll interval, the fallback-to-etcd path, the `apiserver_watch_cache_consistent_read_total` / `apiserver_watch_cache_read_wait` metrics, the etcd #15220 progress-notification race and its v3.4.25/v3.5.8 fix, the 5,000-node × 30-pods = 150,000-object motivation figure, and the 1.28-alpha / 1.31-beta graduation.
4. **KEP-3157, *Watch-List*** — same repository. **Source of** the "going back in time" description of issue #59848 and the statement that the watch cache did not support `resourceVersion=""`.
5. **KEP-5647, *Stale controller handling*** — same repository. **Source of** the quoted motivation that every controller operates on a potentially outdated view with no operator visibility into lag, and of the proposed opt-in read-your-writes mechanism for controllers.
6. **`kubernetes/kubernetes` master source** — <https://github.com/kubernetes/kubernetes>. **Source of** the binding write path (`pkg/registry/core/pod/storage/storage.go`: `GuaranteedUpdate` with optional `UID`/`resourceVersion` preconditions, the `pod %v is already assigned to node %q` guard, the deletion and scheduling-gate rejections), the scheduler's `assumeAndReserve`/`assume` ordering (`pkg/scheduler/schedule_one.go`), the `Binding` object the default binder actually sends — namespace, name and **UID**, no `resourceVersion` (`pkg/scheduler/framework/plugins/defaultbinder/default_binder.go`) — and the kubelet's `InsufficientResourcePrefix = "OutOf"` admission-failure reason (`pkg/kubelet/lifecycle/predicate.go`). **Correction to the previous version of this lesson:** it claimed the scheduler's binding write "carries the `resourceVersion` it read" so that etcd performs a compare-and-swap on it. The binding carries the pod **UID**, not a `resourceVersion`; the protection against double-binding comes from the in-transaction `spec.nodeName != ""` guard plus etcd's `ModRevision` compare inside `GuaranteedUpdate`, and it does **not** prevent two *different* pods binding to the same exhausted node.
7. **`kube-apiserver` command-line reference** — `content/en/docs/reference/command-line-tools-reference/kube-apiserver.md` in <https://github.com/kubernetes/website>. **Source of** `--etcd-compaction-interval` default `5m0s`, `--watch-cache` default true, and the `--watch-cache-sizes` semantics.
8. **`etcd-io/etcd` master source** — <https://github.com/etcd-io/etcd>. **Source of** the MVCC revision model and the linearizable-read path (`server/etcdserver/read/read.go`) referenced here and developed in lesson 02.

**Primary sources — not fetchable in this environment, and therefore not relied on for any number above**

9. **Herlihy, M. & Wing, J. (1990), *Linearizability: A Correctness Condition for Concurrent Objects*, ACM TOPLAS 12(3):463–492** — DOI <https://doi.org/10.1145/78969.78972>. The formal linearization-point definition and the locality/composability theorem. ACM is blocked by this environment's proxy; the definition is restated here from standard usage, not quoted.
10. **Lamport, L. (1979), *How to Make a Multiprocessor Computer That Correctly Executes Multiprocess Programs*, IEEE Trans. Computers C-28(9)** — the sequential-consistency definition. Not fetched.
11. **Lamport, L. (1978), *Time, Clocks, and the Ordering of Events in a Distributed System*, CACM 21(7)** — happens-before and logical clocks, the basis of the causal section. Not fetched.
12. **Gilbert, S. & Lynch, N. (2002), *Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services*** — <https://groups.csail.mit.edu/tds/papers/Gilbert/Brewer2.pdf>. The formal CAP result this lesson deliberately de-emphasises in favour of PACELC. Not fetched.
13. **Terry, D. et al. (1994), *Session Guarantees for Weakly Consistent Replicated Data*** (Bayou) — the four session guarantees. Not fetched.
14. **Abadi, D., *Consistency Tradeoffs in Modern Distributed Database System Design* (PACELC)** — <http://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf>. Not fetched.
15. **Mahajan, Alvisi & Dahlin (2011), *Consistency, Availability, and Convergence*; Attiya, Ellen & Morrison (2015), *Limitations of Highly-Available Eventually-Consistent Data Stores*** — the results behind "causal is the ceiling under partition." Not fetched; the claim is stated as attributed, not derived here.

**Real-world engineering — not fetchable this pass**

16. **Google Cloud, *Strict Serializability and External Consistency in Spanner*** — <https://cloud.google.com/blog/products/databases/strict-serializability-and-external-consistency-in-spanner>. TrueTime plus commit-wait as the price of the top-right box. Blocked by the egress proxy; cited for the mechanism only.
17. **Cloudflare, *Building With Workers KV*** — <https://blog.cloudflare.com/building-with-workers-kv/>. A published, bounded eventual-consistency window. Blocked; the ~1-minute figure is quoted as approximate and should be re-checked against current Cloudflare documentation before you cite it in a design review.
18. **Jepsen, *etcd 3.4.3*** — <https://jepsen.io/analyses/etcd-3.4.3>, and the etcd team's response at <https://etcd.io/blog/2020/jepsen-343-results/>. Blocked; used only for the per-operation point (KV vs. lock API), which lesson 02 revisits.

**Deeper dives**

19. **Kleppmann, M., *Designing Data-Intensive Applications*, chapters 5, 7 and 9** — the fuller treatment of replication, isolation levels and consistency/consensus. Chapter 7's write-skew material is the source of the isolation ladder summarised above.
20. **Jepsen, *Consistency Models*** — <https://jepsen.io/consistency> — the interactive map of the hierarchy, and **Kyle Kingsbury, *Strong consistency models*** — <https://aphyr.com/posts/313-strong-consistency-models>. Both blocked here; both are the best visual companions to the table in Core concepts.
