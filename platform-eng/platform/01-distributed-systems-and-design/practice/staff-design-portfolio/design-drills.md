# Design drills — the same system, three times, at three scales

Companion to the [portfolio README](README.md). The portfolio is **six systems designed once**.
This is **one system designed three times**, and it trains a different and harder reflex.

> **Method adapted from** the `tasks/` → `solutions/` → `implementation/` structure in
> [`harut8/system-design`](https://github.com/harut8/system-design), which solves each design
> problem at four scale tiers (10k → 100k → 1m → 10m users) with a separate architecture per tier.
> Re-aimed here at GPU-platform prompts. See [`docs/EXTERNAL-DEPTH.md`](../../../../../docs/EXTERNAL-DEPTH.md).

## Why the ladder beats one-shot reps

A one-shot design rep answers *"can you design X?"*. Interviewers at staff level are asking
something else: **"do you know which decisions are scale-dependent, and can you name the number
where each one flips?"**

Designing the same system at 10× and 100× forces that out, because you cannot fake it. Either you
know that fan-out-on-write becomes correct somewhere between tier 1 and tier 2, and you can say
roughly where and why — or you have memorised one architecture and will apply it everywhere.

The failure mode this drill exists to kill is **premature Kafka**: reaching for the 10m-tier
architecture on a 10k-tier problem. In the source repo's ladder, the 10k tier of a feed system is a
FastAPI monolith on one PostgreSQL doing fan-out-on-read, and the 10m tier is microservices with
Kafka, Cassandra and a Redis cluster. Both are correct **for their tier**. Proposing the second for
the first is the single most common way a strong engineer fails a design interview — it reads as
someone who has read about scale rather than operated it.

There is a second, quieter payoff: it is the same skill as
[Lesson 10's escalation ladder](../../../03-observability/lessons/10-telemetry-lakehouse.md) —
knowing where to **stop**.

## The three tiers

Define them by the resource that actually binds, not by user count — GPU platforms don't have users.

| Tier | Envelope | The binding constraint | Architecture instinct |
|---|---|---|---|
| **T1 · one rack** | 8–64 GPUs, 1 cluster, 1 team | nothing is scarce except the GPUs themselves | a single controller, one etcd, in-process state, no queue |
| **T2 · one fleet** | 1k–4k GPUs, 1–3 clusters, ~20 teams | contention, fairness, and the control plane | queueing, quota, cohorts, sharded scrape, a real scheduler |
| **T3 · multi-region** | 30k+ GPUs, many clusters/providers, org-wide | coordination cost, blast radius, heterogeneity | cells, federation, async everything, per-region autonomy |

## The drill (90 minutes, timed)

For **one** prompt, in one sitting:

1. **T1, 20 min.** Design the *simplest thing that is actually correct*. Write down the specific
   scale assumption each simplification depends on. Ban yourself from queues, sharding, and
   eventual consistency unless you can defend them at 8 GPUs.
2. **T2, 30 min.** Re-derive from the numbers, do not patch T1. Then answer explicitly: **which T1
   decisions broke, and at what number?** This list is the entire point of the exercise.
3. **T3, 25 min.** Same again. Here the interesting failures stop being throughput and start being
   **coordination and blast radius** — a global anything becomes the outage.
4. **The flip table, 15 min.** One table, filled from memory, in this shape:

   | Decision | T1 | T2 | T3 | Flips at | Because |
   |---|---|---|---|---|---|
   | GPU state store | in-memory in the controller | etcd via the API server | per-region etcd + async rollup | ~1k GPUs / >1 cluster | watch fan-out and etcd write amplification |
   | Admission | first-come, `Pending` | Kueue ClusterQueue + cohorts | per-region queues, global fair-share async | multi-team | starvation and gang deadlock appear with contention |
   | Cost attribution | a PromQL query | recording rules + per-team rollups | lakehouse + sealed monthly periods | ~1k GPUs / audit need | cardinality, then the join to a rate card |

   The **"Flips at"** and **"Because"** columns are the deliverable. Boxes are worthless; the
   threshold and its cause are what a staff interviewer is listening for.

## Prompt set

Reuse the portfolio prompts — the point is the ladder, not new problems:

| Prompt | Where the interesting flip lives |
|---|---|
| **GPU scheduler / quota & fair-share** | T1 has no queue at all; T2 needs gang admission and borrowing; T3 cannot do global fair-share synchronously |
| **Distributed-training checkpoint store** | T1 writes to local NVMe; T2 needs shared storage and async upload; T3 needs the checkpoint-interval math against a much worse failure rate |
| **Inference gateway / router** | T1 is one vLLM behind a Service; T2 needs KV-cache-aware routing; T3 needs regional capacity and cross-region overflow policy |
| **Fleet telemetry pipeline** | T1 is one Prometheus; T2 is sharded scrape + Mimir; T3 forces the hot/cold split from `platform/03` L10 |
| **Model-weight distribution** | T1 pulls from a registry; T2 hits the thundering herd; T3 is a CDN/P2P and egress-cost problem |

## Do two contrast reps

Beyond the tiers, run the portfolio's contrast rep at **fixed** scale — same system, same tier,
**different binding constraint**:

- **Checkpoint store at T2**, once for restart-RTO (fast local + async upload) and once for
  zero-loss durability (synchronous quorum). Note how the architecture *and* the named tradeoff
  axis flip while the scale stays put.
- **Inference gateway at T2**, once for p99 TTFT and once for min cost-per-million-tokens.

Scale is not the only axis that moves a design, and an interviewer who changes the *constraint*
rather than the *scale* catches anyone who only drilled the ladder.

## Acceptance criteria

- [ ] **Three tiers, three genuinely different architectures** for at least two prompts — not one
      architecture with components crossed out.
- [ ] **A flip table per prompt** with a number in "Flips at" and a mechanism in "Because" for
      every row. No hand-waving thresholds.
- [ ] **T1 is embarrassingly simple** and you can defend it — no Kafka, no sharding, no eventual
      consistency, each omission justified by a stated assumption.
- [ ] **Two contrast reps** at fixed scale under different binding constraints.
- [ ] **One written "we should stop here" recommendation** — pick a real fleet size and argue
      against the bigger architecture. This is the sentence that separates senior from staff.

## Suggested layout

```
staff-design-portfolio/
├── design-drills.md              # this file — the method
└── drills/
    ├── gpu-scheduler-ladder.md   # T1/T2/T3 + flip table
    ├── checkpoint-store-ladder.md
    └── inference-gateway-ladder.md
```

## Depth reading

Only when a tier is blocked on internals you don't have — see
[`docs/EXTERNAL-DEPTH.md`](../../../../../docs/EXTERNAL-DEPTH.md):

- [`SYSTEM-DESIGN-GUIDE.md`](https://github.com/harut8/system-design/blob/main/SYSTEM-DESIGN-GUIDE.md)
  — the requirements → estimation → API → high-level → deep-dive → tradeoffs framework, and a
  back-of-envelope section worth reading alongside Lesson 08.
- [`solutions/`](https://github.com/harut8/system-design/tree/main/solutions) — four fully worked
  design documents (distributed counter, feed, search, RBAC). Consumer-scale rather than
  GPU-scale, so read them for **the shape of a complete write-up**, not for the architectures.
- [`implementation/`](https://github.com/harut8/system-design/tree/main/implementation) — the same
  problems built at each tier. Read one problem across all four tiers back-to-back; that reading
  *is* the drill, done by someone else.
