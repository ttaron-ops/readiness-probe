---
lesson: "A01.9"
title: "Design rehearsal"
module: "A-01"
concept: "system-design reps"
status: not-started
est_time: "4 hrs"
artifacts: ["a set of timed design-rep write-ups"]
---

# A01.9 · Design rehearsal

> **Concept.** Design skill is a muscle, not a memory: run timed reps on the prompts you actually get, varying the binding constraint so you name the right tradeoff axis by reflex instead of pattern-matching one canned shape.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Why this matters
The method in A01.8 is inert until it is automatic under a clock and out loud. In a staff loop — and in a real design review the week you ship — you get 40 minutes, an ambiguous prompt, and an audience that judges you on whether you found the *real* bottleneck and named its tradeoff, not on whether you recited a template. Reps are how you convert the method into a reflex, and how you stop over-fitting to one system shape. The failure mode of a strong senior is a beautiful answer to the wrong axis: designing for throughput a system whose binding constraint is tail latency, or quorum-syncing a store whose actual requirement was restart RTO. Reps under varied constraints are the only fix.

## Core notes
**Skip (you already know):** that practice helps; memorizing one canned "design Twitter" answer.

**How to run one rep (35–45 min, out loud, one prompt).** Timebox hard. Speak the whole thing — silent design hides the gaps. Then self-score against a fixed rubric before you look at any reference:

- **Guarantees stated?** Did I write down the consistency / durability / availability contract *before* drawing boxes? (What does a client observe after a write; what survives a node loss; what's the SLO.)
- **Estimated?** Back-of-envelope QPS, bytes/s, fan-out, working-set size, GPU-hours. A number changes the design; a hand-wave does not.
- **Real bottleneck named?** Not the first component — the one that actually caps the system. Usually a queue depth, a replication fsync, a cache miss cliff, or a scheduler's fragmentation.
- **Blast radius named?** What breaks when this fails, how far does it spread, what's the cell / shard boundary that contains it.

Score each 0/1, log the misses, and let the misses pick your next prompt.

**Deliberately vary the binding constraint.** The single highest-leverage habit. Same system, different dominant constraint, and the architecture flips. Rotate reps across:
- **latency-bound** (serving path, KV-cache locality) →
- **storage-bound** (checkpoint store, telemetry retention) →
- **consistency-bound** (scheduler quota, leader election, config plane).

If every rep optimizes the same axis you are training a pattern-match, not a method.

**Name the tradeoff axis — the staff tell.** A senior lists options; a staff engineer says "this is a **PACELC** call and here we favor L," or "this is **fair-share vs utilization vs starvation** and we cap the queue to bound the third." The axes worth rehearsing until you reach for them by name:

| Axis | Name it as | Where it bites |
| --- | --- | --- |
| consistency ↔ latency (no partition) | **PACELC** (the *else-latency* half) | config/metadata reads, KV-cache routing |
| availability ↔ consistency (under partition) | **CAP** / partition behavior | leader election, quota ledger |
| throughput ↔ tail latency | **Little's Law**, batching, queue cap | inference batching, ingest |
| fair-share ↔ utilization ↔ starvation | scheduling / gang + preemption | GPU scheduler, quota |
| durability ↔ write latency | **sync/quorum vs async** replication | checkpoint store, weight registry |
| blast radius ↔ efficiency | **cells / shuffle-sharding** vs shared pool | any multi-tenant plane |
| freshness ↔ load | cache **TTL** / invalidation | weight distribution, metrics rollups |

**The canonical prompt set (this IS the GPU tie).** Build your reps around the prompts a platform engineer actually gets. Each maps back to an earlier lesson and lives in one of the three planes:

| Prompt | Plane | Binding constraint | Ties to |
| --- | --- | --- | --- |
| GPU scheduler / quota + fair-share (Kueue-shaped) | control | consistency + fairness | L2 consensus, L5 queueing |
| Distributed-training checkpoint store | training | durability ↔ write latency | L3 replication, L6 failure |
| Inference gateway / KV-cache-locality router | serving | tail latency | L1 consistency, L4 caching, L5 queueing |
| Fleet metrics / telemetry pipeline | control | cardinality + backpressure | L7 data-intensive |
| Model-weight distribution (cold-start herd) | serving/training | freshness ↔ load, thundering herd | L4 caching, L6 failure |
| Classics reframed: rate limiter, distributed lock / leader election (→ etcd), object storage for weights | control | consistency / quotas | L2, L3 |

**The three-planes throughline.** Before you design anything, say which plane the question lives in — **control** (etcd/K8s: small, strongly-consistent, quorum, config-shaped), **training** (gang-scheduled, checkpoint-bound, all-or-nothing, throughput over latency), **serving** (SLO-driven, queue + KV-cache, tail-latency over throughput). Naming the plane pre-selects the *right* answer for consistency, failure semantics, and queueing — and makes visible when a prompt spans planes (a scheduler's ledger is control-plane consistent even though it schedules training jobs). That is the staff move: the same word ("consistency," "failure," "queue") means a different thing in each plane, and you say which.

## Worked example
**Two contrasting reps, back-to-back, on the *same* system — a distributed-training checkpoint store — under different binding constraints.** The point is to feel the architecture and the *named axis* flip.

**Rep A — optimize for restart RTO (minimize time to resume after a crash).**
- *Plane:* training. *Guarantee stated:* "on failure, resume within ~30s from a checkpoint at most N steps stale; losing the last checkpoint is acceptable."
- *Estimate:* 512 GPUs, 200 GB model+optimizer state, checkpoint every 10 min. Full state = 200 GB; per-rank shard ≈ 400 MB. Restart budget dominated by *read* bandwidth, not write.
- *Real bottleneck:* the *read* path on restart — pulling shards back onto 512 ranks. So write cheap, read fast.
- *Architecture:* asynchronous, tiered — write shard to **local NVMe** first (near-instant), background-flush to shared/object tier. Restart reads from nearest surviving local/peer copy; **async replication**, no quorum on the hot path. Keep only last 1–2 checkpoints hot.
- *Named axis:* **durability ↔ write-latency**, and we *favor latency/RTO* — we accept losing the most-recent checkpoint (bounded step loss) to make write and restart cheap.
- *Blast radius:* a node loss costs at most the async-un-flushed delta; contained to that rank's shard.

**Rep B — same store, optimize for zero-loss durability (no committed step may be lost).**
- *Guarantee stated:* "a checkpoint acknowledged as committed survives any single-node (and ideally single-rack) loss; readers never see a torn checkpoint."
- *Estimate:* same 200 GB, but now every checkpoint must be durably fanned out *before* ack — write bandwidth and fsync/quorum latency now dominate the step budget, so checkpoints get rarer or writes get striped harder.
- *Real bottleneck:* the *write* commit path — synchronous replication / erasure-coding fan-out and the fsync barrier before ack.
- *Architecture:* **synchronous quorum or EC** across fault domains, write-ahead + atomic manifest swap so a checkpoint is all-or-nothing, no local-only tier on the commit path. Restart is now the cheap side.
- *Named axis:* same **durability ↔ write-latency** axis — but we *favor durability*, paying write latency (and lower checkpoint frequency) to guarantee zero committed loss.
- *Blast radius:* wider fan-out per write (more domains touched) is the *cost* we accept for containment of data loss to zero.

**The rep's whole payoff:** one system, one axis, opposite ends — and articulating *which end and why* is the reflex you are training. Write both up; the diff between them is the lesson.

## Practice
Do a timed set (aim for 5 reps) drawn from the canonical prompt set above, one prompt per rep, 35–45 min each, out loud, self-scored on the 4-point rubric. Force constraint variety: at least one latency-bound, one storage-bound, one consistency-bound; do at least one "same system, two constraints" pair as in the worked example. Write each up (guarantees → estimate → API/data → scale-out → **named axis** → blast radius) and log your rubric misses so the next set targets them. Feeds the [staff design portfolio](../practice/staff-design-portfolio/README.md).

## Self-check
- Why vary the binding constraint across reps instead of doing more reps of the same prompt? **Answer:** Because skill transfers by *axis*, not by *shape*. Repeating one prompt trains a pattern-match to one architecture; varying the constraint (latency- vs storage- vs consistency-bound) forces you to re-derive the real bottleneck and name a different tradeoff each time, which is the actual reusable skill. Same reason the worked example runs the *same* checkpoint store under RTO vs zero-loss: the flip is the lesson.
- A prompt says "design a GPU quota and fair-share scheduler." Which plane, and what's the first tradeoff axis you name? **Answer:** Control plane. The quota ledger is small, strongly-consistent config-shaped state (quorum-backed, etcd-shaped), so the first axis is **fair-share ↔ utilization ↔ starvation** — you cap queue depth and define preemption/gang semantics to bound starvation — with a secondary **CAP** call on the ledger under partition (favor consistency: refuse to double-allocate rather than over-admit).
- You produced a clean design but scored 0 on "real bottleneck named." What most likely went wrong, and how do reps fix it? **Answer:** You designed the first component that came to mind rather than the one that caps the system — typically a queue depth, an fsync/replication barrier, a cache-miss cliff, or scheduler fragmentation. Reps fix it by making "estimate first, then find the cap" a scored, timed reflex: the back-of-envelope number surfaces the true bottleneck before you draw boxes, and self-scoring the miss steers your next prompt at the weakness.

## References
- System Design Primer — https://github.com/donnemartin/system-design-primer
- Jepsen analyses (real distributed-systems failure modes) — https://jepsen.io/analyses
- Google SRE Book — https://sre.google/sre-book/table-of-contents/
- Kueue (job queueing / quota / fair-share on Kubernetes) — https://kueue.sigs.k8s.io/docs/concepts/
- Kubernetes scheduling framework — https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/
