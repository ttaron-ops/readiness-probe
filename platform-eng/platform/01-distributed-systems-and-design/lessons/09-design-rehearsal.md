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
sources: 12
---

# A01.9 · Design rehearsal

> **Concept.** Design skill is a muscle, not a memory: run timed reps on the prompts you actually get, varying the binding constraint so you name the right tradeoff axis by reflex instead of pattern-matching one canned shape.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits

[Lesson 08](08-system-design-method.md) gave you the method — eight steps on a 45-minute clock, from requirements through the estimate to the guarantees table and the rejected branch. That method is inert as a checklist; this lesson is where it becomes a reflex you reach for cold, under time pressure, without consulting the list. It is deliberately the module's last lesson: everything from L1's PACELC classification through L8's estimation discipline supplies an axis or a mechanism, and this lesson's only job is to force you to re-derive and recombine them under constraints you did not choose.

The specific addition here is a **training protocol** — a rep structure, a scored rubric, a constraint-rotation schedule, and a way of selecting your next prompt from your own misses. There is no lesson 10; what comes after this is the module's [checkpoint](../checkpoint.md), which grades exactly this skill: bounding any named system in under five minutes, unaided.

## Why this matters

The method in [Lesson 08](08-system-design-method.md) is inert until it is automatic, under a clock, out loud. In a staff loop — and in a real design review the week you ship — you get 40 minutes, an ambiguous prompt, and an audience that judges you on whether you found the *real* bottleneck and named its tradeoff, not on whether you recited a template.

The failure mode of a strong senior engineer is specific and expensive: **a beautiful answer to the wrong axis.** Designing for throughput a system whose binding constraint is tail latency. Quorum-replicating a store whose actual requirement was restart RTO. Sharding for capacity a store that was bound by fault domains. Each of those produces a coherent, defensible, *wrong* architecture, and reading more does not fix it, because the error is not in the knowledge — it is in the reflex that selects which knowledge applies. Reps under varied constraints are the only fix, and the worked example below is built to make one such misidentification happen to you on purpose.

## What's new here (calibration)

- **Skip (you already know):** that practice helps in general, and that memorising one canned "design Twitter" answer is a dead end — that critique of interview-prep culture is not news to you.
- **New — the constraint is the training signal, not the prompt.** Most "practice" drills the same shape repeatedly and trains pattern-matching. The claim here is narrower and sharper: varying the *binding constraint* on the *same or similar systems* is what transfers, because it forces you to re-derive the bottleneck instead of recalling it.
- **New — a scored rubric with anchors.** Not "did I name the bottleneck" but a 0/1/2 scale with a written description of what each score looks like, because a binary self-score is too easy to award yourself.
- **New — self-scoring as a separately trained skill**, with the specific biases that make it unreliable and the mechanics (write before you look; score the transcript, not the memory) that correct for them.
- **New — the three-planes classification as an explicit, tabulated pre-filter**, applied *per sub-decision* rather than once per prompt, with what each plane pre-selects for consistency, failure semantics, queueing, and capacity unit.
- **New — a full-length contrast pair**, executed end to end with numbers, where the same system under two different guarantees produces two different architectures — including one deliberate trap where the obvious bottleneck is not the real one.

## Core concepts

### 1 · Why reps, and what actually transfers

The uncomfortable finding from skill-acquisition research is that time-on-task is a poor predictor of expertise. What predicts it is *deliberate* practice: work at the edge of current competence, with immediate and specific feedback on what failed, repeated with correction. Ericsson's formulation is the standard citation, and the operational consequence for you is blunt: **ten reps on a latency-bound serving path build a latency-bound reflex, not a general one.**

The mechanism is worth being precise about, because it determines how you schedule your reps. What you are training is not knowledge of architectures — you already have that — it is a **classifier**: given an ambiguous prompt, which axis is binding? That classifier is trained by examples that are *diverse in the label* (the binding constraint) rather than diverse in the input (the system). Two reps on a checkpoint store under different constraints train it better than five reps on five different systems that all happen to be latency-bound.

This gives the one rule that organises everything else in this lesson:

> **Rotate the constraint, not the prompt.** If you cannot say what constraint this rep was about, the rep did not train anything.

### 2 · The rep protocol

One rep is 35–45 minutes on one prompt, plus 10 minutes of scoring. The scoring is not optional; a rep you did not score is exercise, not practice.

```
   THE REP LOOP — and why the miss-log is the part that makes it work
   ══════════════════════════════════════════════════════════════════════

   ┌──────────────────────────────────────────────────────────────────┐
   │ 0 · SELECT                                                       │
   │   Pick a prompt from the canonical set (§6) AND a binding         │
   │   constraint from the rotation (§5). The constraint is chosen     │
   │   FIRST and independently — that is what stops you re-running     │
   │   the shape you already know.                                    │
   └───────────────────────────┬──────────────────────────────────────┘
                               ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │ 1 · CLASSIFY  (2 min, before anything else)                      │
   │   Which plane? control / training / serving — and say it per      │
   │   sub-system, not once for the whole prompt (§4).                │
   └───────────────────────────┬──────────────────────────────────────┘
                               ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │ 2 · RUN  (35–45 min, TIMED, OUT LOUD, recorded)                  │
   │   The eight steps from Lesson 08, on the clock. Out loud is       │
   │   non-negotiable: silent design hides exactly the gaps you are    │
   │   trying to find, because your inner monologue skips the steps    │
   │   it cannot do.                                                  │
   └───────────────────────────┬──────────────────────────────────────┘
                               ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │ 3 · SCORE  (10 min, from the RECORDING/artifact, BEFORE           │
   │            looking at any reference answer)                       │
   │   Six criteria, 0/1/2 each, against written anchors (§3).         │
   └───────────────────────────┬──────────────────────────────────────┘
                               ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │ 4 · LOG THE MISS  (2 min)                                        │
   │   One line: "scored 0 on bottleneck — designed the write path     │
   │   when the estimate said the read path was 5% of RTO."            │
   └───────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │ 5 · SELECT NEXT FROM THE MISS  ◀── THE FEEDBACK EDGE             │
   │   The next rep's constraint is chosen to re-test the criterion    │
   │   you scored lowest on, on a DIFFERENT system.                    │
   │   Same criterion + different system = transfer.                   │
   │   Same system + same criterion = memorisation.                    │
   └───────────────────────────┬──────────────────────────────────────┘
                               │
                               └──────────▶ back to 0

   Without edge 5, this is a list of practice problems. With it, it is a
   training loop that converges on your actual weakness.
```

Three mechanics that people skip and shouldn't:

- **Out loud, recorded.** Silent design lets you skip the step you cannot do without noticing. Speaking forces every step to exist. Recording lets you score the transcript instead of your memory of it, which is the single biggest improvement available to self-study.
- **A hard timebox with a visible clock.** Reps that run long train the wrong thing — the whole difficulty is compression.
- **Score before you look at any reference.** Once you have seen a better answer, your memory of what you said drifts toward it. This is not a character flaw; it is how memory works, and the fix is procedural.

### 3 · The rubric, with anchors

A four-point yes/no rubric is too easy to award yourself. Use six criteria on a 0/1/2 scale with written anchors, and score against the transcript.

| # | Criterion | 0 | 1 | 2 |
|---|---|---|---|---|
| 1 | **Plane & scope named** | Started designing immediately | Named the plane once for the whole system | Named it **per sub-decision**, and stated explicit non-goals |
| 2 | **Guarantees stated before boxes** | Guarantees never stated, or only when asked | Stated qualitatively ("consistent", "durable") | An SLO at a percentile, a consistency class **per data class**, and RPO/RTO |
| 3 | **Estimated with units** | No numbers, or numbers with no units | Numbers computed but not used to decide anything | Every number carries units, and at least one **eliminated an architecture** |
| 4 | **Real bottleneck named** | Designed the first component that came to mind | Named a plausible bottleneck without deriving it | Derived it from the estimate, and said what the **next** one is |
| 5 | **Blast radius & failure** | Happy path only | Listed failure modes | Guarantees/**non-guarantees** table, blast radius per mode, degraded-mode ladder |
| 6 | **Axis named, quantified, flippable** | Listed options | Named an axis | Named it, attached a number, and stated the **flip condition** |

**12 is a strong staff rep. Below 8 and the rep found something real; log it.**

Two scoring rules that keep it honest:

- **Score the artifact, not the intention.** "I would have mentioned that" is a 0. If it is not in the transcript, it did not happen — which is exactly the standard the interview applies.
- **Criterion 4 is the one to be harshest on.** It is the easiest to fool yourself about, because a design can be internally coherent and complete while being aimed at a component that was never the constraint. The test: can you point at the line in your own estimate that *forced* the bottleneck you named? If the bottleneck came from intuition rather than from a number, that is a 1 at best.

### 4 · The three planes as a pre-filter, applied per sub-decision

Before designing anything, say which plane the question lives in. This is not taxonomy for its own sake — the plane **pre-selects the default answer** for four separate questions, so naming it correctly saves you five minutes and naming it incorrectly costs you the whole rep.

```
   THE PRE-FILTER — run this in the first two minutes, per SUB-SYSTEM
   ══════════════════════════════════════════════════════════════════════

            "What is the unit of work, and what is scarce?"
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
   ┌─────────────┐    ┌──────────────┐     ┌────────────────┐
   │  CONTROL    │    │   TRAINING   │     │    SERVING     │
   │             │    │              │     │                │
   │ small state │    │ gang-        │     │ per-request,   │
   │ read/write  │    │ scheduled,   │     │ SLO-bound,     │
   │ by many;    │    │ all-or-      │     │ latency is the │
   │ correctness │    │ nothing;     │     │ product        │
   │ >> volume   │    │ long-running │     │                │
   └──────┬──────┘    └──────┬───────┘     └───────┬────────┘
          │                  │                     │
          ▼                  ▼                     ▼
  ┌──────────────────────────────────────────────────────────────┐
  │ WHAT EACH PLANE PRE-SELECTS                                  │
  ├──────────────┬──────────────┬──────────────┬─────────────────┤
  │              │ CONTROL      │ TRAINING     │ SERVING         │
  ├──────────────┼──────────────┼──────────────┼─────────────────┤
  │ consistency  │ linearizable │ per-job      │ eventual +      │
  │              │ (quorum,     │ sequential;  │ read-your-      │
  │              │ etcd-shaped) │ no cross-job │ writes where    │
  │              │              │ ordering     │ users notice    │
  ├──────────────┼──────────────┼──────────────┼─────────────────┤
  │ failure      │ fail-closed; │ checkpoint + │ shed; degrade;  │
  │ semantics    │ refuse       │ restart;     │ never queue     │
  │              │ rather than  │ elasticity   │ past the SLO    │
  │              │ double-      │              │                 │
  │              │ allocate     │              │                 │
  ├──────────────┼──────────────┼──────────────┼─────────────────┤
  │ queueing     │ tiny queues; │ deep queues  │ bounded queue + │
  │              │ admission is │ are FINE —   │ LIFO + deadline │
  │              │ the product  │ jobs wait    │ propagation     │
  │              │              │ hours        │                 │
  ├──────────────┼──────────────┼──────────────┼─────────────────┤
  │ capacity     │ writes/s and │ GPU-seconds, │ HBM-GB per      │
  │ unit         │ watch fan-out│ interconnect │ concurrent req; │
  │              │              │ GB/s         │ tokens/s        │
  ├──────────────┼──────────────┼──────────────┼─────────────────┤
  │ the number   │ fsync p99    │ MTBF/N and   │ KV ceiling and  │
  │ that decides │ and quorum   │ checkpoint δ │ TPOT at batch B │
  │              │ RTT          │              │                 │
  └──────────────┴──────────────┴──────────────┴─────────────────┘

  ★ MOST REAL PROMPTS SPAN PLANES. Classify PER SUB-DECISION:

     "GPU scheduler with quota"
        ├─ the quota LEDGER            → CONTROL  (linearizable, fail-closed)
        ├─ the JOBS it admits          → TRAINING (gang, checkpoint-bound)
        └─ the scheduler's own API     → CONTROL  (watch fan-out is the cost)

     "checkpoint store"
        ├─ the manifest / `latest` ptr → CONTROL  (CAS, all-or-nothing)
        └─ the shard BLOBS             → TRAINING (bandwidth-bound, immutable)

     "inference gateway"
        ├─ the tenant quota ledger     → CONTROL  (strong, small)
        ├─ the placement table         → SERVING  (eventual, cached)
        └─ the replicas                → SERVING  (HBM-bound)
```

**Saying it per sub-decision is the specific staff tell this lesson trains toward.** "It's a control-plane problem" is a senior answer. "The ledger is control-plane so it's linearizable and fails closed; the jobs it admits are training-plane so their failure semantics are checkpoint-and-restart, which means the ledger has to hold a reservation across a restart without leaking it" is a staff answer — and note that the second one produced a *design requirement* the first one could not see.

### 5 · Rotating the binding constraint

The constraint is chosen before the prompt and independently of it. Six to rotate through, with the tell that identifies each and what flips when it binds:

| Binding constraint | How you can tell | What the architecture does | The number that proves it |
|---|---|---|---|
| **Latency-bound** | The SLO is a percentile and the product dies without it | admission control, bounded queues, LIFO, deadline propagation, replicate for tail | p99 vs. p50 gap; queue-implied wait |
| **Throughput-bound** | Sustained demand approaches sustained capacity | batching, pipelining, horizontal scale, backpressure | offered rate ÷ per-unit capacity |
| **Storage-bound** | Total bytes ÷ per-node capacity > 1 | sharding, tiering, retention policy, compaction | dataset size ÷ per-shard usable |
| **Bandwidth-bound** | The bytes moved per unit of work dominate the compute | locality, caching, compression, co-location, disaggregation is *rejected* | bytes/op ÷ link GB/s |
| **Consistency-bound** | A wrong answer costs more than a slow one | quorum, single-writer, CAS, fencing tokens | quorum RTT; conflict rate |
| **Fault-domain-bound** | The replication scheme needs more independent domains than you have | placement groups, EC width, cells, spread constraints | domains required vs. domains available |

The last one is under-rehearsed and appears in the worked example below. It is the constraint that decides shard *count* in storage systems more often than capacity does, and almost nobody reaches for it.

**Rotation discipline.** Over any five reps, cover at least three distinct constraints, and include at least one **contrast pair** — the same system run twice under two different constraints, back to back. If your last three reps were all latency-bound, your next one is not allowed to be.

### 6 · The canonical prompt set — and the seed number for each

Build reps around the prompts a GPU-platform engineer actually gets. Each maps to earlier lessons, lives in one or more planes, and has **one estimate that decides the whole design** — the seed. Doing the seed calculation first is the habit that keeps a rep from becoming box-drawing.

| Prompt | Plane(s) | Usual binding constraint | The seed number | Ties to |
|---|---|---|---|---|
| GPU scheduler / quota + fair-share (Kueue-shaped) | control (ledger) + training (jobs) | consistency + fairness | quota-ledger writes/s and the gang size distribution | L2, L5 |
| Distributed-training checkpoint store | control (manifest) + training (blobs) | **fault-domain**, often mistaken for bandwidth | state bytes = params × bytes/param(optimizer); δ and the RTO decomposition | L3, L6 |
| Inference gateway / KV-locality router | serving | latency (KV ceiling) | KV bytes/token × context ÷ free HBM | L1, L4, L5, L8 |
| Fleet metrics / telemetry pipeline | control | cardinality + backpressure | active series = GPUs × metrics × label cardinality | L7 |
| Model-weight distribution (cold-start herd) | serving/training | bandwidth + thundering herd | model GB ÷ per-node link GB/s × simultaneous pullers | L4, L6 |
| GPU-second billing pipeline | control | correctness | records/day and the duplicate rate × unit price | L7 |
| Classics reframed: rate limiter, distributed lock / leader election, object store for weights | control | consistency / quotas | quorum RTT; lease duration vs. clock skew | L2, L3 |

**Do the seed out loud in the first three minutes of every rep.** If you cannot compute the seed for a prompt, that is the gap, and it is a knowledge gap rather than a reflex gap — go back to the relevant lesson rather than doing more reps.

### 7 · Contrast reps: the highest-density exercise

A contrast rep is the same system designed twice, back to back, under two different binding constraints or two different guarantees. It is the highest-value exercise in this lesson, for a mechanical reason: **you cannot hide behind "it depends" when you have to produce two concrete architectures and explain the delta in one sentence each.**

The protocol:

1. Run rep A fully (35–45 min), score it.
2. Change exactly one thing — the guarantee, or the constraint — and state it precisely.
3. Run rep B **from the requirements again**, not by patching A. Patching A trains the wrong reflex; re-deriving trains the right one.
4. Produce a **delta table**: what changed, what stayed, and — the payoff — *which numbers moved and by how much*.

Two ready-made contrast pairs you do not have to invent:

- **Scale-driven, from production.** Discord's storage stack went through three real generations (MongoDB → Cassandra → ScyllaDB) as it grew from hundreds of millions to trillions of messages, and each migration was forced by a genuinely *different* binding constraint (index-fits-in-RAM, then maintenance cost at ~177 nodes, then p99 tail latency). Design for the 2017 scale, then for the 2022 scale, and write down what changed in the binding constraint at each step. The constraint genuinely changed rather than being invented for practice, which is what makes it a good drill.
- **Guarantee-driven, constructed.** The checkpoint store below: RTO-optimised versus zero-loss-durable. Same system, same numbers, two architectures.

The module's [design-drill ladder](../practice/staff-design-portfolio/design-drills.md) is the third form: the same prompt at one rack, one fleet, and multi-region, with an explicit *flip table* recording which decision changes at which number. Use it when you want the scale axis rather than the guarantee axis.

### 8 · Naming the axis — the tradeoff table

A senior lists options; a staff engineer names the axis. Rehearse these until they are reflexive, with the number that usually decides each:

| Axis | Name it as | Where it bites on a GPU platform | The number that usually decides it |
|---|---|---|---|
| consistency ↔ latency (no partition) | **PACELC**, the *else-latency* half | placement tables, KV-cache routing, config reads | quorum RTT × request rate vs. cost of one stale-route retry |
| availability ↔ consistency (under partition) | **CAP** / partition behaviour | quota ledger, leader election | cost of double-allocation vs. cost of refusing work |
| throughput ↔ tail latency | **Little's Law**, batching, queue cap | inference batching, telemetry ingest | TPOT at batch B vs. the p99 SLO |
| fair-share ↔ utilisation ↔ starvation | gang scheduling + preemption | GPU scheduler, quota, cohorts | max queue age vs. fragmentation % |
| durability ↔ write latency | sync/quorum vs. async replication | checkpoint store, weight registry | δ blocking time vs. bytes at risk |
| blast radius ↔ efficiency | **cells / shuffle-sharding** vs. one pool | any multi-tenant plane | idle headroom % vs. fraction of tenants a failure reaches |
| freshness ↔ load | cache **TTL** / invalidation | weight distribution, metrics rollups | staleness window vs. origin QPS |
| cost ↔ everything | committed vs. on-demand vs. spot | fleet sizing | $/GPU-hour × utilisation vs. preemption rate |

Meta's MAST scheduler is a published instance of this table in action: it splits scheduling into a fast, less-optimal path for time-sensitive placement and an asynchronous, more-optimal "slow path" — an explicit optimality-vs-latency split, made and defended in production. That asymmetry is itself a good discussion prompt: *why* an asymmetric split rather than one scheduler tuned in the middle, and which axis is actually being traded?

### 9 · The self-scoring problem

Scoring your own rep honestly is a separate skill from designing, and it is the piece most self-study skips entirely. Three specific biases and their procedural fixes:

| Bias | What it feels like | Fix |
|---|---|---|
| **Hindsight** | "I knew the bottleneck was the fault domains" — after reading the reference | Score **before** looking at anything. Write the scores down; a written score resists revision in a way a felt one does not. |
| **Coherence-as-correctness** | The design was clean and complete, so it felt strong | Score criterion 4 by asking "which line of my estimate forced this bottleneck?" A clean design aimed at the wrong component scores 0 there, and it should. |
| **Recall smoothing** | Your memory of the rep is more articulate than the rep | Score the **recording or the artifact**, never the memory. This is the highest-value 10 minutes in the whole loop. |

A useful calibration: after scoring, write one sentence you *actually said* that you would not want an interviewer to hear. If you cannot find one in 45 minutes of talking, you are not scoring honestly.

### 10 · A four-week schedule

Reps decay without spacing. A workable programme, ~5 hours a week:

| Week | Reps | Constraint rotation | Deliverable |
|---|---|---|---|
| 1 | 3 | latency-bound, consistency-bound, storage-bound | 3 write-ups + miss log |
| 2 | 3 (one contrast pair) | fault-domain-bound, bandwidth-bound, + repeat of week 1's worst criterion | 2 write-ups + 1 delta table |
| 3 | 3 | drawn from the miss log; different systems, same weak criteria | 3 write-ups |
| 4 | 2 + the drill ladder | any; plus one prompt at T1/T2/T3 scale | 2 write-ups + a flip table |

By the end you have 10–12 rep write-ups, of which the best 5–6 clean up into the [staff design portfolio](../practice/staff-design-portfolio/README.md). **The portfolio is a by-product of the reps, not a separate project** — which is the point of writing each rep up rather than just talking through it.

## Perspectives

**The deliberate-practice view.** Repeating a task in a form you have already mastered does not build capability; the signal that trains skill is practice just past the edge of current competence with fast, specific feedback. Applied here, the lever is not "more reps", it is "reps whose binding constraint you have not already automated" — which is why the constraint is selected before the prompt, and why the miss log picks the next rep.

**The self-assessment view.** Your after-the-fact impression of how a rep went is the least reliable signal in the room. It is easy to feel a rep "went fine"; it is much harder to look at a transcript and mark a 0 next to "real bottleneck named" when you drew a clean diagram and never found the cap. Treat the rubric as external and non-negotiable, and score from the artifact.

**The three-planes view.** The first move in any rep is classification, because it pre-selects the right consistency model, failure semantics, queueing regime, and capacity unit before you have drawn a box. It also exposes the prompts that do not classify cleanly — and those are the interesting ones, because the interface between two planes (a control-plane ledger holding reservations for training-plane jobs across a restart) is where the real design problems live.

**The contrast-rep view.** Running the *same* system twice with the guarantee flipped forces the axis into the open. You cannot say "it depends" when you have to produce two different architectures from the same requirements and defend the delta in one sentence each. This is the muscle the method alone cannot train, because the method runs once per prompt; the contrast rep runs it twice and makes you justify the difference.

**The interviewer's view.** Loops are calibrated: multiple interviewers see many candidates on the same prompt, so what stands out is not novelty but *derivation* — a candidate who arrives at a common architecture via a stated number is scored above one who arrives at a clever architecture via taste. Reps train the derivation, which is the part that is legible from the other side of the table.

## Real-world use cases

- **Meta, "MAST: Global Scheduling of ML Training across Geo-Distributed Datacenters at Hyperscale" (OSDI '24)** — <https://www.usenix.org/system/files/osdi24-choudhury.pdf> — *What it shows:* a gradable reference answer for the "GPU scheduler / quota" prompt, including a published fast-path/slow-path split (an explicit optimality-vs-latency trade) and a headline number that motivates the whole design (worst-region demand/supply ratio 2.63 → 0.98). Score your own scheduler rep against it.
- **Discord, "How Discord Stores Trillions of Messages"** — <https://discord.com/blog/how-discord-stores-trillions-of-messages> — *What it shows:* a ready-made contrast pair where the constraint genuinely changed. Three storage generations (MongoDB, then Cassandra at ~12 nodes and billions of messages in 2017, then ScyllaDB at ~177 nodes and trillions by 2022), each migration forced by a different binding constraint — index-fits-in-RAM, then maintenance cost at scale, then p99 tail latency. Design for the 2017 scale, then the 2022 scale, and name what changed.
- **Roblox, "Return to Service — the 73-hour outage report (Oct 2021)"** — <https://blog.roblox.com/2022/01/roblox-return-to-service-10-28-10-31-2021/> — *What it shows:* a control-plane rep and a blast-radius rep in one incident. A Consul upgrade under simultaneous high read and write load produced contention that took down fleet-wide service discovery, and the tooling needed to diagnose it depended on the failing system. Good source for "design the coordination plane, then say what containment would have bounded *this specific* failure."
- **Modal, "GPU Memory Snapshots"** — <https://modal.com/blog/gpu-mem-snapshots> — *What it shows:* the weight-distribution / cold-start prompt with real numbers attached. Snapshotting full GPU state (weights in VRAM plus the CUDA context) cut median cold start from roughly two minutes to around ten seconds on one tested model — a concrete freshness-vs-load anchor for your own estimate. *(2026 snapshot; treat the figures as dated to that report.)*

## Worked example

**A contrast pair on one system: a distributed-training checkpoint store, designed twice.** Rep A optimises restart RTO; Rep B optimises zero-loss durability. Both are run end to end. Rep A contains a deliberate trap — the obvious bottleneck is not the real one — and finding it is the point of the exercise.

### Shared setup (both reps)

**Classify first.** Two planes in one prompt: the **manifest / `latest` pointer is control-plane** (small, must be all-or-nothing, linearizable), and the **shard blobs are training-plane** (large, immutable, bandwidth-shaped). Design them differently.

**Envelope.**

```
  Job                    512 × H100 (64 nodes × 8), FSDP/ZeRO-3 sharded
  Model                  13B parameters
  Training state         weights bf16 2 B + fp32 master 4 B
                         + Adam m 4 B + Adam v 4 B          = 14 B/param
                         13×10⁹ × 14 B = 182 GB
                         + RNG/dataloader/scheduler state    ≈ 200 GB total
  Per-rank shard         200 GB / 512 = 390 MB
  Step time              2.5 s  (assumption — state it)
  Job MTBF               per-GPU 20,000 h ÷ 512 = 39 h      (L06, §12)
  Local NVMe             ~5 GB/s write per node → 320 GB/s aggregate
  Shared object tier     ~20 GB/s aggregate                  (assumption)
```

### REP A — optimise restart RTO

**Guarantee stated (before boxes).** "After a failure, training resumes within 60 seconds from a checkpoint at most 10 minutes stale. Losing the single most recent checkpoint is acceptable; losing more than 10 minutes of progress is not."

**Estimate.**

```
  δ (blocking write, local-first):
     per node writes 8 ranks × 390 MB = 3.1 GB at 5 GB/s   = 0.62 s
     + barrier and metadata                                 ≈ 1 s   → δ = 1 s

  Optimal interval (Young, L06 §13):
     τ = √(2·δ·M) = √(2 × (1/3600) h × 39 h) = √0.02167 = 0.147 h = 8.8 min
     → round to 10 min (240 steps)

  Waste = √(2δ/M) = √(2 × 0.000278 / 39) = 0.0038 = 0.38 %

  Background flush to the shared tier: 200 GB ÷ 20 GB/s = 10 s, overlapped
  with training, so it costs nothing on the critical path.
```

**Now decompose RTO — this is the trap.** The obvious move is to optimise the read path, because "restart is dominated by reading the checkpoint back". Check it:

```
  RTO BUDGET, measured end to end
  ───────────────────────────────────────────────────────────────
  detect the failure (heartbeat + timeout, L06 §1)      30    s
  reschedule / acquire a replacement node               20–120 s
  container start + CUDA context init                   ~30   s
  NCCL rendezvous across 512 ranks                      30–60 s
  READ 200 GB back:
      from peer local NVMe, 320 GB/s aggregate           0.6  s
      from the shared object tier, 20 GB/s              10    s
  ───────────────────────────────────────────────────────────────
  TOTAL                                                110–250 s
  READ SHARE OF RTO                                    0.5 – 9 %

  ★ THE READ PATH IS NOT THE BOTTLENECK. It is under 10 % of RTO even
    in the worst case. An architecture that optimises it — a faster
    object store, a fancier prefetch — is optimising 9 % of the number
    the guarantee is written against, and cannot reach the 60 s target
    no matter how good it gets.

  ★ THE REAL BOTTLENECK IS PROCESS AND COLLECTIVE RE-INITIALISATION,
    plus node acquisition. Those are 90 %+ of RTO.
```

**Architecture, aimed at the actual constraint.**

```
   REP A — RTO-OPTIMISED
   ══════════════════════════════════════════════════════════════════════

   64 training nodes                    ┌──────────────────────────────┐
   ┌────────────────────────┐           │ HOT SPARE POOL: 4 nodes,     │
   │ rank writes 390 MB     │           │ containers RUNNING, CUDA     │
   │ → LOCAL NVMe (0.6 s)   │           │ context up, weights-free.    │
   │ → returns to training  │           │ ← this is what buys the RTO  │
   └───────────┬────────────┘           └──────────────┬───────────────┘
               │ async, overlapped                     │ swap-in on failure
               ▼                                       ▼
   ┌────────────────────────┐           ┌──────────────────────────────┐
   │ peer replication:      │           │ ELASTIC RECONFIGURATION:     │
   │ each shard also lands  │           │ the collective is rebuilt    │
   │ on 1 peer node's NVMe  │           │ WITHOUT a full job restart;  │
   │ (async, 390 MB)        │           │ surviving ranks keep their   │
   └───────────┬────────────┘           │ state in HBM                 │
               │                        └──────────────────────────────┘
               ▼ background flush, 10 s, off the critical path
   ┌───────────────────────────────────────────────────────────────────┐
   │ SHARED OBJECT TIER — 20 GB/s, holds the durable history           │
   │ manifest committed LAST (CAS on `latest`) → no torn checkpoint    │
   └───────────────────────────────────────────────────────────────────┘

   Restart reads from, in order: local NVMe (survived) → peer NVMe →
   shared tier. The last is the only one that costs 10 s, and it is the
   rarest path.
```

**API.**

```
POST /v1/jobs/{job}/checkpoints            → {ckpt_id, step, upload_targets[512]}
PUT  <upload_target>                        (one per rank, content-addressed)
POST /v1/jobs/{job}/checkpoints/{id}/commit → CAS publish; 409 if step regressed
GET  /v1/jobs/{job}/checkpoints/latest      → manifest (committed only)
GET  /v1/jobs/{job}/checkpoints/{id}/manifest
```

**Data model.** Manifest = `{job_id, step, wall_clock, world_size, parallelism{tp,pp,dp}, shards:[{rank, uri, bytes, sha256}], state}`. Blobs are **content-addressed by SHA-256**, which gives free dedup across consecutive checkpoints for any tensor that did not change and gives a corruption check on read (the SDC defence from [Lesson 06](06-failure-and-resilience.md), §11). Manifests partition by `job_id` (a few thousand rows — never shard); blobs partition by content hash (uniform by construction).

**Named axis:** **durability ↔ write latency**, favouring latency. We accept losing the newest, not-yet-flushed checkpoint in exchange for a 1-second blocking write and a 10-minute interval.

**Blast radius:** a node loss costs the un-flushed delta for its 8 ranks — bounded by the flush cadence — plus whatever the elastic reconfiguration cannot absorb.

**Rep A score, honestly:** criterion 4 is the whole rep. A version of this design that concluded "so we need a faster object store" scores **0** there, because the estimate says the read is under 10 % of RTO. The design is only correct because the RTO decomposition forced it.

### REP B — same system, optimise zero-loss durability

**Change exactly one thing.** New guarantee: *"a checkpoint acknowledged as committed survives any single node loss and any single rack loss; a reader never observes a torn checkpoint."* Re-derive from requirements; do not patch Rep A.

**Estimate.**

```
  Commit must now be durable across fault domains BEFORE ack.
  Erasure coding EC(6,3): 6 data + 3 parity chunks, tolerates 3 losses,
  storage overhead 1.5× (vs 3× for triple replication).

  δ (blocking): write 200 GB × 1.5 = 300 GB at 20 GB/s   = 15 s
                + fsync barrier + manifest CAS            ≈ 2 s
                                                     → δ ≈ 17 s, round 20 s

  τ = √(2 × (20/3600) × 39) = √0.4333 = 0.658 h = 39 min → 40 min (960 steps)

  Waste = √(2 × 0.00556 / 39) = √2.85×10⁻⁴ = 1.69 %
```

**The shard-count arithmetic — and the third distinct binding constraint.**

```
  Retention: 3 most recent + hourly for 24 h + daily for 30 d = 57 checkpoints
  Raw       = 57 × 200 GB                       = 11.4 TB
  With EC(6,3) at 1.5×                          = 17.1 TB

  (a) capacity-bound : 17.1 TB ÷ 30 TB usable/node          =  1 node
  (b) write-bound    : 20 GB/s burst ÷ 5 GB/s per node      =  4 nodes
  (c) read-bound     : 20 GB/s restart ÷ 5 GB/s per node    =  4 nodes
  (d) FAULT-DOMAIN-bound : EC(6,3) needs 9 independent
      placement groups, and no rack may hold > 3 chunks
      → ≥ 9 domains, spread over ≥ 3 racks                  =  9 nodes

  shard/node count = max(1, 4, 4, 9) = 9  →  round to 12
                     (4 nodes in each of 3 racks: even EC spread,
                      and one node may be down without breaking placement)

  ★ FAULT-DOMAIN-BOUND. Not capacity, not bandwidth. The replication
    scheme's independence requirement sets the node count, and any
    argument about disk size or throughput for this store is irrelevant.
```

**Architecture.**

```
   REP B — ZERO-LOSS DURABILITY
   ══════════════════════════════════════════════════════════════════════

   64 training nodes
   ┌────────────────────────────────────────────────────────────────┐
   │ ALL 512 ranks write shards SYNCHRONOUSLY. No local-only tier    │
   │ on the commit path — a local write is not a durable write.      │
   └──────────────────────────┬─────────────────────────────────────┘
                              │ 300 GB (EC-expanded), 15 s BLOCKING
                              ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ CHECKPOINT STORE — 12 nodes, 3 racks × 4                        │
   │  EC(6,3), ≤ 3 chunks per rack, fsync before ack                 │
   │  ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
   │  │ rack A   │ │ rack B   │ │ rack C   │  chunk placement is     │
   │  │ 3 chunks │ │ 3 chunks │ │ 3 chunks │  the CONSTRAINT, not    │
   │  └──────────┘ └──────────┘ └──────────┘  an optimisation        │
   └──────────────────────────┬─────────────────────────────────────┘
                              │ all 512 shards durable AND fsynced
                              ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ MANIFEST COMMIT — CAS on `latest`, control-plane semantics.     │
   │ ATOMIC MANIFEST SWAP is what makes "no torn checkpoint" true:   │
   │ readers resolve `latest` → manifest → shards, and a manifest    │
   │ only exists once every shard it names is durable.               │
   │ A straggler rank past the deadline ⇒ ABORT the whole checkpoint │
   │ (never commit a partial). All-or-nothing is the guarantee.      │
   └────────────────────────────────────────────────────────────────┘

   Restart is now the CHEAP side: read 200 GB at 20 GB/s = 10 s, and
   every committed checkpoint is guaranteed complete and verified by
   its per-shard SHA-256.
```

**Named axis:** the *same* **durability ↔ write-latency** axis — but favouring durability. We pay 20 s of blocking write and a 40-minute interval to guarantee zero committed loss.

**Failure modes, with the guarantees table:**

| | Guarantee | Mechanism | Non-guarantee — say it |
|---|---|---|---|
| Completeness | A committed checkpoint names only durable shards | manifest CAS *after* all fsyncs | An **aborted** checkpoint leaves orphan blobs; GC reclaims them, not the commit path |
| Fault tolerance | Survives any 3 chunk losses, incl. one whole rack | EC(6,3), ≤ 3 chunks/rack | Two simultaneous rack losses lose the checkpoint; that is out of scope by declaration |
| Integrity | Corruption on read is detected | per-shard SHA-256 verified on restore | Corruption is *detected*, not repaired, below the EC threshold |
| Freshness | At most 40 min of training is at risk | τ = 40 min from Young | 40 min is **more** work at risk than Rep A's 10 min — see the delta |

### The delta table — the payoff of the pair

| | Rep A (RTO) | Rep B (zero-loss) | Why it moved |
|---|---|---|---|
| Blocking write δ | 1 s | 20 s | durability must be established before ack |
| Optimal interval τ | 10 min | 40 min | `τ = √(2δM)`; δ rose 20×, τ rose √20 ≈ 4.5× |
| Checkpoint waste | 0.38 % | 1.69 % | `√(2δ/M)`; also √20 ≈ 4.5× |
| Work at risk per failure | ~5 min (τ/2) | ~20 min (τ/2) | **the durable design loses MORE work per failure** |
| Committed-checkpoint loss | possible (newest, unflushed) | never | the actual guarantee that changed |
| Restart RTO | 110–250 s, init-dominated | 120–260 s, unchanged | RTO was never storage-dominated |
| Store node count | 4 (bandwidth-bound) | 12 (**fault-domain-bound**) | EC(6,3) needs 9 independent domains |
| Storage overhead | ~1× (async, few copies) | 1.5× (EC) | parity |

**The sentence the whole pair exists to produce:** *durability of the artifact and minimality of lost work are different objectives, and optimising one can worsen the other.* Rep B never loses a committed checkpoint and yet loses four times as much training work per failure, because the cost of committing forced the interval out. A candidate who says that unprompted has demonstrated the thing this module is for.

**And the trap, restated:** in both reps the storage read path is under 10 % of RTO. Two full designs of a "checkpoint store" and the storage-read bandwidth never once binds. If your rep concluded that the answer was a faster object store, log the miss against criterion 4 and pick a different system with the same criterion for the next rep.

## Practice

Run a timed set of **five reps**, one prompt each, 35–45 minutes each, out loud and recorded, scored on the six-criterion rubric before consulting any reference.

**Constraints on the set:**

1. **Constraint variety.** At least three distinct binding constraints across the five, including at least one that is *not* latency- or throughput-bound (fault-domain, consistency, or bandwidth).
2. **One contrast pair.** Same system, one thing changed, re-derived from requirements rather than patched — either your own (as in the worked example) or Discord's real two-generation redesign.
3. **The seed first.** Compute each prompt's seed number (§6) in the first three minutes, out loud, before drawing anything.
4. **Plane per sub-decision.** Every rep must name at least two sub-systems and classify each independently.
5. **Feedback edge closed.** Each rep after the first must target the criterion you scored lowest on previously, **on a different system**.

**Each write-up contains:** plane classification per sub-decision · guarantees (SLO at a percentile, consistency per data class, RPO/RTO) · the estimate with units and the seed identified · API · data model with the partition key and its skew story · an ASCII architecture diagram annotated with the estimate's numbers · the binding constraint, derived, plus the *next* one · a guarantees/non-guarantees table with blast radius · a degraded-mode ladder · ≥ 3 tradeoff axes with chosen end, rejected end, the deciding number, and the flip condition · your rubric scores with one sentence per miss.

*Acceptance:* your miss log shows a criterion that improved between reps 1 and 5, and the contrast pair has a delta table in which at least one number moved in a direction you did not expect. Cleaned up, the best 5–6 of these become the [staff design portfolio](../practice/staff-design-portfolio/README.md); for the scale axis instead of the guarantee axis, run the [design-drill ladder](../practice/staff-design-portfolio/design-drills.md).

## Common pitfalls

1. **"Ten reps of the same prompt builds the skill."** *Symptom:* fluent on serving prompts, lost on a control-plane one. *Mechanism:* what you are training is a classifier over *binding constraints*, and it is trained by diversity in the label, not in the input. Repetition on one shape trains a pattern-match to that shape. Rotate the constraint before choosing the prompt.
2. **Designing the storage layer because the prompt says "store".** *Symptom:* a checkpoint-store rep that optimises read bandwidth. *Mechanism:* the prompt names a component; the estimate names the constraint, and they routinely disagree — in the worked example the read path is under 10 % of RTO while process and collective re-initialisation are 90 %. Decompose the number the guarantee is written against before choosing what to optimise.
3. **"A design that covers everything is a strong rep."** *Symptom:* uniform detail across every component and no bottleneck named. *Mechanism:* uniform coverage is the signature of never having found the constraint — if you had, you would have spent your minutes there. Score on "was the actual constraint identified and derived", never on breadth.
4. **"Self-scoring is easy once you know the rubric."** *Symptom:* every rep scores 10–12 and nothing improves. *Mechanism:* hindsight bias, coherence-as-correctness, and recall smoothing all inflate self-scores, and all three are procedural rather than moral failings. Score from the recording, before looking at a reference, in writing.
5. **"The prompt tells you the plane."** *Symptom:* a scheduler design that gives the quota ledger training-plane failure semantics, and consequently leaks reservations across job restarts. *Mechanism:* most real prompts span planes, and the interface between them is where the design problems live. Classify per sub-decision and state what each plane pre-selects.
6. **Patching rep A instead of re-deriving for rep B.** *Symptom:* a contrast pair whose two architectures differ by one component. *Mechanism:* patching preserves rep A's framing, including whatever was wrong with it; re-deriving from requirements is what surfaces that a changed guarantee moves δ, which moves τ, which moves the work at risk in a direction you did not predict. The delta table is the artifact and it only exists if you re-derived.
7. **"If I can explain the design, I've done the rep."** *Symptom:* a confident walkthrough with no numbers in it. *Mechanism:* explanation is necessary and not sufficient; the six rubric criteria are the operational definition of "done", and any of them can be missing from an account that sounds complete. Criterion 3 in particular — a number that *eliminated an architecture* — is absent from most fluent explanations.
8. **Practising without writing up.** *Symptom:* twelve reps done, no portfolio. *Mechanism:* the write-up is where the design gets pinned down enough to be scored and reused, and the portfolio is meant to be a by-product of the reps rather than a separate project. A rep you did not write up cannot be scored honestly a week later and cannot become evidence.

## Self-check

- **Why vary the binding constraint rather than doing more reps of the same prompt?** Because what you are training is a classifier from an ambiguous prompt to "which axis binds", and a classifier is trained by diversity in the *label*, not the input. Ten reps on latency-bound serving paths build a latency-bound reflex; two reps on one system under two different constraints train the general one. The worked example is exactly this: one checkpoint store, RTO versus zero-loss durability, producing two different architectures and — the payoff — a delta table showing that the durable design loses *more* training work per failure (τ/2 of 20 min vs 5 min) despite never losing a committed checkpoint.
- **A prompt says "design a GPU quota and fair-share scheduler." Classify it and name the first axis.** It spans planes, so classify per sub-decision: the **quota ledger is control-plane** — small, linearizable, quorum-backed, fails closed (refuse to admit rather than double-allocate) — while the **jobs it admits are training-plane** — gang-scheduled, all-or-nothing, checkpoint-bound, and content to queue for hours. That split immediately generates a design requirement the single-label answer cannot see: the ledger must hold a reservation across a training-plane restart without leaking it, which means reservations need leases and a fencing token rather than plain rows. First axis: **fair-share ↔ utilisation ↔ starvation**, with max queue age as the number; secondary axis: the ledger's **CAP** behaviour under partition, resolved toward consistency.
- **You produced a clean design but scored 0 on "real bottleneck named". What went wrong, and what is the fix?** You designed the component the prompt named rather than the one the estimate constrains — the checkpoint-store trap, where "store" pulls you to the read path while the RTO decomposition shows the read is under 10 % of the number and process/collective re-initialisation is 90 %. The fix is procedural: decompose the specific quantity your guarantee is written against (RTO, p99 TTFT, waste fraction, invoice error) into its terms, *then* optimise the largest term. The scoring test is "which line of my own estimate forced this bottleneck?" — if the answer is intuition rather than a number, it is a 1 at best.
- **Give the shard-count arithmetic for the durable checkpoint store and name the binding constraint.** Retention of 57 checkpoints × 200 GB = 11.4 TB raw, × 1.5 for EC(6,3) = 17.1 TB. Capacity: 17.1 ÷ 30 TB usable per node = 1. Write burst: 20 GB/s ÷ 5 GB/s per node = 4. Restart read: same, 4. Fault domains: EC(6,3) needs 9 independent placement groups with no more than 3 chunks per rack, so ≥ 9 nodes across ≥ 3 racks. `max(1, 4, 4, 9) = 9`, rounded to **12** (4 nodes in each of 3 racks, so one node can be down without breaking placement). The store is **fault-domain-bound** — which means any argument about disk capacity or throughput for it is irrelevant, and the growth lever is the EC width, not the hardware.
- **Which of the three worked designs in this module was bound by what, and why does that matter?** Lesson 07's billing pipeline partition count was **skew-bound** (keying by `job_id` gave a 10× hot partition; keying by `hash(job_id, gpu_uuid)` fixed it at the cost of per-job ordering). Lesson 08's request-log shard count was **storage-bound** (`ceil(6.2 TB ÷ 2 TB usable) = 4`, with write and read constraints each demanding only 1). This lesson's checkpoint store is **fault-domain-bound** (9 placement groups). Three sharding decisions, three different constraints — which is the whole argument for rotating constraints rather than prompts: "how do I shard this?" has no general answer, only a `max()` over constraints, and the interesting engineering is knowing which term wins.
- **What does the three-planes table pre-select, and give a case where naming the plane once is wrong.** It pre-selects four defaults: consistency model, failure semantics, queueing regime, and capacity unit — plus the number that usually decides (fsync p99 and quorum RTT for control; MTBF/N and checkpoint δ for training; the KV ceiling and TPOT at batch B for serving). Naming it once is wrong on any prompt that spans planes, which is most of them: a checkpoint store's manifest is control-plane (CAS, all-or-nothing, small) while its blobs are training-plane (immutable, bandwidth-shaped, huge), and designing both with one set of defaults produces either an unnecessarily expensive blob path or a manifest that can be observed torn.
- **How do you score a rep honestly, and why is that a separate skill?** Score from the recording or the written artifact, never from memory; score before consulting any reference; write the scores down. Three biases make the unaided version unreliable — hindsight (after reading a better answer you remember having known it), coherence-as-correctness (a clean design *feels* right even when aimed at the wrong component), and recall smoothing (your memory of what you said is more articulate than what you said). The calibration check: find one sentence you actually said that you would not want an interviewer to hear. If 45 minutes of talking produced none, you are not scoring, you are reminiscing.
- **Design a contrast rep for the inference gateway and predict which numbers move.** Rep A: p99 TPOT < 50 ms (the Lesson 08 design — batch 128, 3,902 tok/s per replica, 13 replicas). Rep B: change one thing, p99 TPOT < 25 ms for an interactive coding product. Re-derive: the step-time table forces batch ≈ 64 (23.9 ms), which drops per-replica throughput to 2,678 tok/s, so replica count rises from 13 to `33,333 ÷ 2,678 ÷ 0.7 ≈ 18` — about 20 more H100s for a latency the first product never asked for. The concurrency ceiling per replica (128, set by HBM) does not move at all, because it is a memory property and not a batching one — so the fleet ends up with *more* headroom on concurrency and *less* on throughput, which flips which constraint binds. That inversion is the delta, and predicting it before running the numbers is the reflex being trained.

## Connections & what's next

This is the module's last lesson and its job is to be a hub, not a new topic. Every prior lesson supplies an axis or a mechanism a rep draws on: [L01](01-consistency-models.md)'s PACELC classification and per-data-class consistency, [L02](02-consensus-and-quorums.md)'s quorum and etcd mechanics, [L03](03-replication-and-partitioning.md)'s replication and partitioning math, [L04](04-caching.md)'s stampede and locality patterns, [L05](05-queueing-and-backpressure.md)'s Little's Law and shed-vs-defer discipline, [L06](06-failure-and-resilience.md)'s checkpoint-interval formula, availability composition, and gray-failure detection — all three of which appear directly in the worked example — [L07](07-data-intensive-patterns.md)'s idempotency and guarantees-table shape, and [L08](08-system-design-method.md)'s eight-step method, which every rep runs end to end.

There is no lesson 10. What comes next is the module's [checkpoint](../checkpoint.md), whose pass criterion 8 tests exactly this skill — drive a design on a GPU-platform prompt, bounding it in under five minutes, unaided — and the [staff design portfolio](../practice/staff-design-portfolio/README.md), where these timed reps, cleaned up to 2–4 pages each, become the 5–6 write-ups the deliverable requires. Use the [design-drill ladder](../practice/staff-design-portfolio/design-drills.md) when you want the scale axis rather than the guarantee axis; the flip table it produces is the same artifact as this lesson's delta table, taken along a different dimension.

## References & further reading

**Primary sources**

1. **Choudhury, A. et al., "MAST: Global Scheduling of ML Training across Geo-Distributed Datacenters at Hyperscale," OSDI '24** — <https://www.usenix.org/system/files/osdi24-choudhury.pdf> — a gradable reference design for the GPU-scheduler prompt, including the published fast-path/slow-path split and the 2.63 → 0.98 demand/supply headline. *Not fetched from this environment (egress-restricted); figures carried forward from the previous revision of this lesson and from lesson 08.*
2. **Ericsson, K.A., Krampe, R.T., Tesch-Römer, C. (1993), "The Role of Deliberate Practice in the Acquisition of Expert Performance,"** *Psychological Review* 100(3) — the standard citation for the practice model in §1: work at the edge of competence with immediate, specific feedback, rather than time-on-task. *Not fetched from this environment; cited for the model, which §1 states in its own terms.*
3. **Young, J.W. (1974), "A First Order Approximation to the Optimum Checkpoint Interval,"** *Communications of the ACM* 17(9) — <https://cacm.acm.org/research/a-first-order-approximation-to-the-optimum-checkpoint-interval/> — the `τ = √(2δM)` and waste `= √(2δ/M)` used throughout the worked example. Derived from scratch in [Lesson 06](06-failure-and-resilience.md), §13, so the paper is optional depth. *Not fetched from this environment.*
4. **Kueue documentation** — <https://kueue.sigs.k8s.io/docs/concepts/> — the real fair-share, cohort, and ClusterQueue model the "GPU scheduler / quota" prompt is shaped after. Read it before that rep so your design is calibrated against a shipping API rather than an invented one. *Not fetched from this environment.*
5. **Kubernetes scheduling framework** — <https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/> — the extension points a Kueue-shaped scheduler plugs into; useful for making the scheduler rep concrete about *where* a policy actually executes. *Not fetched from this environment.*
6. **`stas00/ml-engineering`, "Accelerator" chapter** — <https://github.com/stas00/ml-engineering/blob/master/compute/accelerator/README.md> — the H100/A100 capacity, bandwidth, and achievable-TFLOPS figures that anchor the estimates in the worked example and in [Lesson 08](08-system-design-method.md) (H100 SXM 80 GB HBM3 at 3.35 TB/s; 989 TFLOPS BF16 theoretical vs ~794 measured). *Fetched and read in this environment, August 2026.*

**Real-world engineering write-ups**

7. **Discord, "How Discord Stores Trillions of Messages"** — <https://discord.com/blog/how-discord-stores-trillions-of-messages> — the three-generation storage redesign (MongoDB → Cassandra → ScyllaDB), usable as a ready-made contrast pair where the binding constraint genuinely changed at each step. *Not fetched from this environment; summary carried forward from the previous revision of this lesson.*
8. **Roblox, "Return to Service | 10/28 – 10/31 2021"** — <https://blog.roblox.com/2022/01/roblox-return-to-service-10-28-10-31-2021/> — a 73-hour outage from a control-plane coordination failure, with diagnostic tooling inside the failure domain. A real blast-radius rep. *Not fetched from this environment.*
9. **Modal, "GPU Memory Snapshots"** — <https://modal.com/blog/gpu-mem-snapshots> — cold-start numbers (median ~2 min → ~10 s on one tested model via full GPU-state snapshotting) for the weight-distribution prompt. *Not fetched from this environment; figures carried forward from the previous revision of this lesson and dated to that report.*

**Deeper dives**

10. **Google, *Site Reliability Engineering* (the SRE Book)** — <https://sre.google/sre-book/table-of-contents/> — the fuller treatment behind the shed-vs-defer, error-budget, and cascading-failure ideas the reps rehearse in miniature. *Not fetched from this environment (egress-restricted).*
11. **Jepsen analyses** — <https://jepsen.io/analyses> — real distributed databases' consistency claims tested and frequently broken. Read two before any consistency-bound rep; they are the best available calibration for what a *precise* guarantee statement sounds like, which is criterion 2 on the rubric. *Not fetched from this environment.*
12. **The System Design Primer** — <https://github.com/donnemartin/system-design-primer> — a broad armory of canonical system shapes a rep prompt can draw from. Useful for recognising shapes; it will not train the constraint classifier, which is what this lesson is for.
