---
lesson: "A01.9"
title: "Design rehearsal"
module: "A-01"
concept: "system-design reps"
status: not-started
est_time: "5 hrs"
prev: "08-system-design-method.md"
next: null
artifacts: ["a set of timed design-rep write-ups"]
sources: 9
---

# A01.9 · Design rehearsal

> **Concept.** Design skill is a muscle, not a memory: run timed reps on the prompts you actually get, varying the binding constraint so you name the right tradeoff axis by reflex instead of pattern-matching one canned shape.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits
Lesson 08 gave you the repeatable method — eight steps you drive on a clock, from requirements through the tradeoff axis to the failure mode. That method is inert as a checklist; this lesson is where it becomes a reflex you reach for cold, under time pressure, without consulting the list. It is deliberately the module's last lesson: everything from L1's PACELC class through L8's estimation discipline supplies an axis or a mechanism, and this lesson's only job is to force you to re-derive and recombine them under constraints you didn't choose. There is no lesson 10 after this — what comes next is the module's [checkpoint](../checkpoint.md), which grades exactly this skill: bounding any named system in under five minutes, unaided.

## Why this matters
The method in A01.8 is inert until it is automatic under a clock and out loud. In a staff loop — and in a real design review the week you ship — you get 40 minutes, an ambiguous prompt, and an audience that judges you on whether you found the *real* bottleneck and named its tradeoff, not on whether you recited a template. Reps are how you convert the method into a reflex, and how you stop over-fitting to one system shape. The failure mode of a strong senior is a beautiful answer to the wrong axis: designing for throughput a system whose binding constraint is tail latency, or quorum-syncing a store whose actual requirement was restart RTO. Reps under varied constraints are the only fix.

## What's new here (calibration)
- **Skip (you already know):** that practice helps in general, and that memorizing one canned "design Twitter" or "design Netflix" answer is a dead end — that critique of interview-prep culture is not news to you.
- **New depth — the constraint is the training signal, not the prompt.** Most "practice" drills the same shape repeatedly (another chat app, another URL shortener) and trains pattern-matching, not judgment. This lesson's actual claim is narrower and sharper: varying the *binding constraint* on the *same or similar systems* is what transfers, because it forces you to re-derive the bottleneck instead of recalling it.
- **New depth — self-scoring is a separate, trained skill from designing.** Grading your own rep honestly against a fixed rubric, before you consult any reference answer, is a distinct capability from doing the design itself — and it's the piece most self-study skips entirely.
- **New depth — the three-planes classification as an explicit pre-filter.** Naming which plane (control / training / serving) a prompt lives in, *before* you draw anything, and re-naming it per sub-decision when a prompt spans planes, is the specific staff tell this lesson trains toward.

## Core concepts
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

**A ready-made contrast pair, drawn from production.** You don't have to invent both halves of a contrast rep yourself — some companies lived it. Discord's storage stack went through three real generations (MongoDB → Cassandra → ScyllaDB) as scale grew from hundreds of millions to trillions of messages, and each migration was forced by a genuinely different binding constraint (index-fits-in-RAM, then maintenance-cost-at-177-nodes, then p99-tail-latency). Treat their public account as a second contrast-rep pair, alongside the checkpoint-store example below: design for the 2017 scale, then redesign for the 2022 scale, and write down what changed in the binding constraint at each step. See Real-world use cases below for the link.

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

Meta's MAST scheduler (see Real-world use cases) is a real, published instance of this table in action at hyperscale: it splits scheduling into an asynchronous, more-optimal "slow path" and a fast, less-optimal path for time-sensitive placement decisions — an explicit consistency/optimality-vs-latency split, made and named by a team that had to defend it in production. That's a good discussion prompt in its own right: why an *asymmetric* split, and which axis is actually being traded.

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

## Perspectives
**The deliberate-practice view.** Skill-acquisition research is consistent on one point: repeating a task in a form you've already mastered does not build capability — the signal that trains skill is practice just past the edge of current competence, with fast feedback on exactly where it broke down. Applied here, that means the lever is not "more reps," it's "reps whose binding constraint you haven't already automated." Ten reps on a latency-bound serving path build a latency-bound reflex, not a general one; the canonical prompt table exists specifically to push you across control, training, and serving so no rep repeats a shape you've already solved.

**The self-assessment/rubric view.** Scoring your own rep honestly against the four-point rubric is a different skill from doing the design, and it's the one most self-study skips. It's easy to feel a rep "went fine" from memory; it's much harder to look at what you actually said out loud and mark a 0 next to "bottleneck named" when you drew a clean diagram but never found the actual cap. Treat the rubric as an external, non-negotiable checklist, precisely because your own after-the-fact impression of how a rep went is the least reliable signal in the room.

**The three-planes view.** The first move in any rep is classifying which plane — control, training, serving — the prompt lives in, because that classification pre-selects the right consistency model, failure semantics, and queueing regime before you've drawn a single box. It also exposes the prompts that don't classify cleanly: a GPU scheduler's quota ledger is control-plane (strongly consistent, quorum-backed) even though the workloads it admits are training-plane (gang-scheduled, checkpoint-bound). Naming the plane *per sub-decision*, not once for the whole system, is the specific move that separates a staff answer from a senior one.

**The contrast-rep view.** The highest-density exercise here isn't any single rep — it's running the *same* system twice with the binding constraint flipped, the way the worked example does with the checkpoint store. A contrast rep forces the axis into the open: you can't hide behind "it depends" when you have to produce two different architectures from the same requirements and explain, in one sentence each, why they differ. This is the muscle lesson 08's method alone can't train, because the method runs once per prompt — the contrast rep runs it twice, back to back, and makes you defend the delta.

## Real-world use cases
- **Meta's MAST global ML-training scheduler (OSDI '24)** — https://www.usenix.org/system/files/osdi24-choudhury.pdf — *What it shows:* a real, published reference design for the "GPU scheduler / quota + fair-share" rep in the canonical prompt table above — a hyperscale team's actual guarantees, estimate, and named tradeoffs (including a fast-path/slow-path split) to compare your own rep against.
- **Discord, "How Discord Stores Trillions of Messages"** — https://discord.com/blog/how-discord-stores-trillions-of-messages — *What it shows:* a storage-bound rep made real. Discord's three-generation evolution (MongoDB, then Cassandra at ~12 nodes/billions of messages in 2017, then ScyllaDB at ~177 nodes/trillions of messages by 2022) is a genuine "redo this design under a different constraint" story — design for the 2017 scale, then the 2022 scale, and name what changed.
- **Roblox, "Return to Service" — the 73-hour outage report (Oct 2021)** — https://blog.roblox.com/2022/01/roblox-return-to-service-10-28-10-31-2021/ — *What it shows:* a consistency/control-plane rep and a failure/blast-radius rep in one incident. A Consul upgrade under simultaneous high read and write load produced channel contention that took down fleet-wide service discovery, and the same outage tooling depended on the failing system, which is a blast-radius lesson in its own right — a good source for "design the control/coordination plane, then explain what containment would have bounded this specific failure."
- **Modal, "GPU Memory Snapshots"** — https://modal.com/blog/gpu-mem-snapshots — *What it shows:* the serving/training weight-distribution, cold-start prompt with real numbers attached. Snapshotting full GPU state (weights in VRAM, CUDA context) cut median cold start from roughly two minutes to around ten seconds on one tested model — a concrete freshness-vs-load tradeoff to anchor your own rep's estimate against (2026 snapshot; treat the exact figures as dated to that report).

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
Do a timed set (aim for 5 reps) drawn from the canonical prompt set above, one prompt per rep, 35–45 min each, out loud, self-scored on the 4-point rubric. Force constraint variety: at least one latency-bound, one storage-bound, one consistency-bound; do at least one "same system, two constraints" pair as in the worked example — or borrow Discord's real two-redesign story (2017-scale vs 2022-scale) as that pair instead of inventing one. Write each up (guarantees → estimate → API/data → scale-out → **named axis** → blast radius) and log your rubric misses so the next set targets them. Feeds the [staff design portfolio](../practice/staff-design-portfolio/README.md).

## Common pitfalls
1. **"Doing 10 reps of the same prompt builds the skill."** Repetition on one shape trains pattern-matching to that shape, not the transferable skill — re-deriving the bottleneck under a *different* binding constraint. Discord's real three-generation redesign is a good "forced" version of this exercise, since the constraint genuinely changed at each stage rather than being invented for practice.
2. **"A design that covers everything is a strong rep."** An unscoped, uniformly-detailed answer usually means the real bottleneck was never found — a strong rep is narrow and pointed, not exhaustive. Score explicitly on "was the actual constraint identified," not on coverage.
3. **"Self-scoring honestly is easy once you know the rubric."** It's a distinct, practiced skill — silently reviewing your own design hides the same gaps that silent (never-spoken) design hides. Score out loud or in writing, immediately after the rep, before memory smooths over the misses.
4. **"The prompt tells you the plane."** Some prompts span planes — a scheduler's quota ledger is control-plane-consistency-bound even though it schedules training-plane work. Naming which part of a multi-plane system you're in *per sub-decision*, the way MAST's real fast-path/slow-path split does, is the actual staff move.
5. **"If I can explain the design, I've done the rep."** Explaining is necessary but not sufficient — the rubric's four checks (guarantees stated, estimated, bottleneck named, blast radius named) are the operational definition of "done," and any one of them can be missing from an explanation that still sounds complete.

## Self-check
- Why vary the binding constraint across reps instead of doing more reps of the same prompt? **Answer:** Because skill transfers by *axis*, not by *shape*. Repeating one prompt trains a pattern-match to one architecture; varying the constraint (latency- vs storage- vs consistency-bound) forces you to re-derive the real bottleneck and name a different tradeoff each time, which is the actual reusable skill. Same reason the worked example runs the *same* checkpoint store under RTO vs zero-loss: the flip is the lesson.
- A prompt says "design a GPU quota and fair-share scheduler." Which plane, and what's the first tradeoff axis you name? **Answer:** Control plane. The quota ledger is small, strongly-consistent config-shaped state (quorum-backed, etcd-shaped), so the first axis is **fair-share ↔ utilization ↔ starvation** — you cap queue depth and define preemption/gang semantics to bound starvation — with a secondary **CAP** call on the ledger under partition (favor consistency: refuse to double-allocate rather than over-admit).
- You produced a clean design but scored 0 on "real bottleneck named." What most likely went wrong, and how do reps fix it? **Answer:** You designed the first component that came to mind rather than the one that caps the system — typically a queue depth, an fsync/replication barrier, a cache-miss cliff, or scheduler fragmentation. Reps fix it by making "estimate first, then find the cap" a scored, timed reflex: the back-of-envelope number surfaces the true bottleneck before you draw boxes, and self-scoring the miss steers your next prompt at the weakness.
- A prompt spans two planes — e.g. "design the ledger a GPU scheduler uses to track quota for training jobs." Why doesn't "call it control plane and move on" fully answer the classification question, and what should you say instead? **Answer:** Because the ledger itself (small, strongly-consistent, quorum-backed state) and the workload it schedules (gang-scheduled, checkpoint-bound training jobs) sit in genuinely different planes even though they're one system. Naming the plane once for the whole thing hides that split; the staff move is naming it per sub-decision — "the quota ledger is control-plane-consistent; the jobs it admits are training-plane, so their failure semantics differ from the ledger's" — which mirrors the asymmetric fast-path/slow-path split MAST's real scheduler uses at hyperscale.

## Connections & what's next
This is the module's last lesson, and its job is to be a hub, not a new topic: every prior lesson supplies an axis or mechanism a rep draws on — L1's PACELC classification, L2's quorum/etcd mechanics, L3's replication and partitioning math, L4's cache stampede and locality patterns, L5's Little's Law and shed-vs-defer discipline, L6's gray-failure and checkpoint-interval math, L7's exactly-once and idempotency-key patterns, and L8's 8-step method that every single rep runs from end to end. There is no lesson 10 — what's next is the module's own [checkpoint](../checkpoint.md), whose pass criterion 8 tests exactly the skill this lesson trains (drive a design, bounding any named system in under five minutes, unaided), and the [staff design portfolio](../practice/staff-design-portfolio/README.md) deliverable, where these timed reps — cleaned up and written to 2–4 pages each — become the 5–6 design write-ups the portfolio requires.

## References & further reading

**Primary sources**
- System Design Primer — https://github.com/donnemartin/system-design-primer — read for the broad armory of canonical system shapes and patterns a rep prompt draws from.
- Kueue docs (job queueing / quota / fair-share on Kubernetes) — https://kueue.sigs.k8s.io/docs/concepts/ — read for the real fair-share/quota API the "GPU scheduler" prompt in the canonical table is shaped after.
- Kubernetes scheduling framework — https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/ — read for the real extension points a Kueue-shaped scheduler plugs into.
- Meta, "MAST: Global Scheduling of ML Training across Geo-Distributed Datacenters at Hyperscale" (OSDI '24) — https://www.usenix.org/system/files/osdi24-choudhury.pdf — read for a real, gradable reference design for the GPU-scheduler rep, including a published fast-path/slow-path consistency-vs-latency split.

**Real-world engineering blogs**
- Discord, "How Discord Stores Trillions of Messages" — https://discord.com/blog/how-discord-stores-trillions-of-messages — what it shows: a real three-generation storage redesign (MongoDB → Cassandra → ScyllaDB), usable as a ready-made contrast-rep pair.
- Roblox, "Return to Service | 10/28 – 10/31 2021" — https://blog.roblox.com/2022/01/roblox-return-to-service-10-28-10-31-2021/ — what it shows: a 73-hour outage from a control-plane coordination failure (Consul) with a single point of failure and diagnostic blindness — a real blast-radius rep.
- Modal, "GPU Memory Snapshots" — https://modal.com/blog/gpu-mem-snapshots — what it shows: a real freshness-vs-load, cold-start tradeoff for the model-weight-distribution prompt, with before/after latency numbers.

**Deeper dives**
- Google SRE Book — https://sre.google/sre-book/table-of-contents/ — the fuller treatment behind the shed-vs-defer and error-budget ideas this module rehearses in miniature.
- Jepsen analyses (real distributed-systems failure modes) — https://jepsen.io/analyses — read a few before a consistency-bound rep; these are real production databases' consistency claims tested and often broken, the exact texture a "guarantees stated" answer needs.
