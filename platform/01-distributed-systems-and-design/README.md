---
id: "A-01"
track: "A — Platform excellence"
title: "Distributed systems & system design"
notion: null                # repo-native module (added in the 12–15mo rebuild), not from the original Notion plan
phase: "Track A · deepen module"
effort: "8–10 weeks ≈ ~38 hrs @ 10–12 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: []
unlocks: []
started: null
completed: null
---

# 🧩 Distributed systems & system design

> **Goal.** Deepen the distributed-systems foundation under real platform design and the
> system-design interview — to a **senior/staff bar**.
>
> **Track A — Platform excellence.** A deepen module: you cleared the senior bar here already;
> this takes it to staff depth and interview-ready fluency.

## Through-line

**A senior can *build* the systems; a staff engineer can *bound* them** — state the exact
guarantee, the cost of that guarantee, the failure mode when it's violated, and the one number
that decides the tradeoff. Every lesson converts a familiar topic into (a) a precise
guarantee/formula, (b) a named real-system placement, and (c) a failure mode with a blast-radius
story. The recurring frame is **three planes** — *control* (etcd/K8s: linearizable, PC/EC,
fsync-bound), *training* (gang-scheduled, all-or-nothing, correlated failure, checkpoint-bound),
and *serving* (SLO + queue + KV-cache, latency-bound) — and naming which plane a question lives in
is the staff move.

## Calibrated to your background — what we skip

You already build distributed systems, so we **skip the definitions** and spend only on the
**staff delta**: the precise guarantee vs the cost, the real-system placement, and the failure
mode with a number attached. The GPU tie is not decoration — the control plane, the training job,
and the inference path are three *different* points in the CAP/PACELC/Little's-Law design space.

## Lessons

| # | Lesson | Staff delta |
|---|--------|-------------|
| 01 | [Consistency models](lessons/01-consistency-models.md) | linearizable vs serializable; PACELC; the K8s stale-cache bug |
| 02 | [Consensus and quorums](lessons/02-consensus-and-quorums.md) | Raft ReadIndex/PreVote; why etcd is fsync-bound |
| 03 | [Replication and partitioning](lessons/03-replication-and-partitioning.md) | W+R>N; fixed-partition rebalancing; hot shards |
| 04 | [Caching](lessons/04-caching.md) | caches as modal systems; stampede; KV-cache & weight distribution |
| 05 | [Queueing and backpressure](lessons/05-queueing-and-backpressure.md) | Little's Law; shed-don't-defer; metastable failure |
| 06 | [Failure and resilience](lessons/06-failure-and-resilience.md) | gray failure / SDC; retry budgets; checkpoint-interval math |
| 07 | [Data-intensive patterns](lessons/07-data-intensive-patterns.md) | exactly-once reality; CDC; the billing-pipeline idempotency key |
| 08 | [A repeatable system-design method](lessons/08-system-design-method.md) | BOTE in GPU units; drive the interview; name tradeoffs |
| 09 | [Design rehearsal](lessons/09-design-rehearsal.md) | timed reps on the real GPU-platform prompts |

Total ≈ **38 hrs ≈ 8–10 weeks**. **Spine:** L1 (PACELC), L2 (etcd), L6 (gray failure),
L8 (the method).

## Deliverable & checkpoint

- Build the **[staff design portfolio](practice/staff-design-portfolio/)** — 5–6 staff-level
  design write-ups (2–4 pages each) on GPU-platform prompts (GPU scheduler/quota, training
  checkpoint store, inference gateway/router, fleet telemetry pipeline, model-weight
  distribution), each following the Lesson 08 framework with an explicit **guarantees &
  non-guarantees** table, one **back-of-envelope** estimate, and a **failure & blast-radius**
  section. These double as interview artifacts and checkpoint evidence.
- The [**checkpoint**](checkpoint.md) is the gate — for any named system, in < 5 min: state its
  PACELC class and where reads/writes land; do the BOTE for its dominant resource and find the
  first bottleneck; name one gray/correlated failure mode and the blast-radius control that bounds it.

## Directory layout

| Path | What goes here |
|------|----------------|
| [`lessons/`](lessons/) | One page per concept — notes, worked example, practice, self-check. |
| [`practice/`](practice/) | Design write-ups, labs, diagrams — the buildable output. |
| [`resources/`](resources/) | Saved references, papers, link index. |
| [`checkpoint.md`](checkpoint.md) | Checkpoint answers (the completion gate). |
