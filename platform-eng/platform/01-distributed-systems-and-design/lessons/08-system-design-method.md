---
lesson: "A01.8"
title: "A repeatable system-design method"
module: "A-01"
concept: "driven framework, BOTE in GPU units"
status: not-started
est_time: "6 hrs"
prev: "07-data-intensive-patterns.md"
next: "09-design-rehearsal.md"
artifacts: ["inference-gateway-design.md"]
sources: 12
---

# A01.8 · A repeatable system-design method

> **Concept.** A time-boxed framework you *lead*: the back-of-the-envelope estimate chooses the architecture, and staff signal is naming — and quantifying — the tradeoff before the interviewer asks.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits

[Lessons 01–07](../README.md) built the concepts one at a time: consistency models, consensus and quorums, replication and partitioning, caching, queueing and backpressure, failure and resilience, and delivery semantics. Each answered a narrow question well. This lesson is the synthesis — a single repeatable method that pulls all of them into a bounded design under a clock, so "what consistency model?", "what happens when this fails?", and "is this pipeline effectively-once?" stop being separate lessons and become questions the method *forces* you to ask, in order, on any prompt you are handed.

The specific thing this lesson adds that no earlier lesson could: **a procedure with a clock attached**. Knowing PACELC does not help if you spend 18 of your 45 minutes on requirements and never reach failure modes. Half of what is being graded is sequencing.

## Why this matters

At staff level the design interview is not "can you draw boxes" — it is "can you drive an ambiguous problem to a defended architecture on a clock, connecting every choice to a stated requirement." The differentiator is estimation discipline: the numbers *choose* the design ("fits in RAM" vs "must shard"), and for GPU systems the scarce unit is HBM-GB and memory bandwidth, not QPS — so a web-shaped estimate is not merely imprecise, it sizes the wrong resource entirely and produces an architecture that cannot work.

The stakes outside the interview are the same. This method is a compressed design document, and a design document is the artifact that gets a quarter of engineering time committed. Getting the estimate wrong there is not a rejected candidacy; it is a rack of GPUs bought against the wrong bottleneck.

## What's new here (calibration)

- **Skip (you already know):** "ask clarifying questions," draw a load balancer, put a cache in front, mention a CDN. Table stakes, not signal.
- **Skip:** the vocabulary of "high-level design" and "API design" as interview-prep buzzwords — you have built real APIs and real architectures.
- **New — the clock as a first-class artifact.** A minute-by-minute structure with a stated deliverable per block, and the rule for what to cut when you fall behind.
- **New — estimation as a dependency graph, not a ritual.** Which number feeds which decision, so you compute the three numbers that *change the architecture* and skip the ones that do not.
- **New — the GPU estimation formulas, derived.** KV-cache bytes per token from model config, decode throughput from memory bandwidth rather than FLOPs, prefill cost from parameter count, and the point where the two contend. Most candidates' GPU estimates are wrong in a *structural* way: they size on compute when decode is bandwidth-bound.
- **New — the guarantees & non-guarantees table** as the concrete deliverable of the failure step, and availability composition arithmetic as the thing that kills "just add another service".
- **New — a full-length worked design**, end to end, on a GPU-platform prompt, with every number shown — not a sketch with the arithmetic elided.

## Core concepts

### 1 · What is actually being graded

Before the method, the rubric it is built to satisfy. From the interviewer's side, a staff-level design answer is scored on five things, and only one of them is the diagram:

| Signal | Senior answer | Staff answer |
|---|---|---|
| **Scoping** | Answers the question as asked | Bounds it: states non-goals, picks the sub-problem worth 30 minutes, says why |
| **Estimation** | Produces numbers when asked | Produces a number *unprompted* and uses it to eliminate an architecture |
| **Bottleneck** | Designs the first component that comes to mind | Names the resource that caps the system and designs *at* it |
| **Failure** | Covers it if the clock allows | Spends real time there; produces a guarantees/non-guarantees table |
| **Tradeoffs** | Explains the chosen design | Names the *rejected* alternative and quantifies why it lost |

The asymmetry is worth internalising: a senior candidate produces a working design; a staff candidate produces a working design **and a legible trail of why it isn't a different one**. Everything below is machinery for producing that trail on a clock.

### 2 · The clock

Forty-five minutes is the common shape (some loops give 60; the block proportions hold). The failure mode is not running out of ideas, it is running out of time before reaching the sections that carry the most signal — which are the last two.

```
  THE 45-MINUTE STRUCTURE — what to PRODUCE in each block
  ══════════════════════════════════════════════════════════════════════════

  min  0 ─────5 ────────12 ──15 ──20 ──────30 ──────38 ──────45
       │  REQ  │   EST    │API│ DATA│  HLD +  │ FAILURE │ TRADE │
       │       │          │   │     │  SCALE  │ & OPS   │ OFFS  │
       └───────┴──────────┴───┴─────┴─────────┴─────────┴───────┘
         5 min    7 min    3m   5m     10 min     8 min    7 min

  ┌──────────────┬───────────────────────────────────────────────────────┐
  │ BLOCK        │ THE ARTIFACT ON THE BOARD WHEN THE BLOCK ENDS         │
  ├──────────────┼───────────────────────────────────────────────────────┤
  │ REQUIREMENTS │ 3–5 functional bullets · an SLO with a number ·        │
  │ 0–5          │ an availability target · a scale envelope ·            │
  │              │ an explicit NON-GOALS list                            │
  ├──────────────┼───────────────────────────────────────────────────────┤
  │ ESTIMATION   │ 3–5 derived numbers, units carried, each labelled     │
  │ 5–12         │ with the decision it makes. ONE of them must be       │
  │              │ the binding constraint, said out loud.                 │
  ├──────────────┼───────────────────────────────────────────────────────┤
  │ API          │ 2–4 endpoints with request/response shapes.           │
  │ 12–15        │ Reveals the read/write asymmetry.                     │
  ├──────────────┼───────────────────────────────────────────────────────┤
  │ DATA MODEL   │ Access patterns FIRST, then entities, then the        │
  │ 15–20        │ PARTITION KEY with its justification.                 │
  ├──────────────┼───────────────────────────────────────────────────────┤
  │ HLD + SCALE  │ The boxes — drawn only now — with the estimate's      │
  │ 20–30        │ numbers annotated on the edges. Then go STRAIGHT to   │
  │              │ the bottleneck and fix it. Each fix traced to a number.│
  ├──────────────┼───────────────────────────────────────────────────────┤
  │ FAILURE &    │ A GUARANTEES / NON-GUARANTEES table. Blast radius.    │
  │ OPS  30–38   │ Degraded mode. RPO/RTO. What pages, what doesn't.     │
  ├──────────────┼───────────────────────────────────────────────────────┤
  │ TRADEOFFS    │ 2–3 axes, each with the CHOSEN end, the REJECTED end, │
  │ 38–45        │ and a NUMBER that decided it.                         │
  └──────────────┴───────────────────────────────────────────────────────┘

  WHEN YOU FALL BEHIND — cut in this order, and SAY that you are cutting:
    1st cut  the second half of the data model  ("I'll assume a keyed
             store; the interesting part is the partition key, which is X")
    2nd cut  breadth in the high-level design   ("three more components
             exist; none is on the critical path — I'll focus here")
    3rd cut  the API                            ("two endpoints, POST
             /generate and GET /models; moving on")
  NEVER cut: the estimate, the failure section, or the tradeoff section.
  Those are the three blocks the score actually comes from.
```

Two disciplines make the clock work:

**Announce the block.** "I'll take five minutes on requirements, then do the estimate, and I'd like to spend real time on failure modes at the end." This does three things at once: it shows you know the shape, it gets the interviewer to help you keep time, and it pre-authorises you to move on when they are still asking requirement questions at minute 8.

**Timestamp out loud.** "That's twelve minutes; I have my numbers, moving to the API." Interviewers reliably mark up candidates who self-manage the clock, because it is the same skill as running a design review.

### 3 · Step 1 — Requirements, and the three you must not skip

Functional requirements are the easy half: what does it do. Enumerate 3–5 and stop; a longer list is a scoping failure, not thoroughness.

The non-functional half is where the design is actually decided, and there are exactly three questions that change the architecture. Ask them even if the interviewer did not offer them:

1. **What is the latency SLO, at which percentile?** "p99 TTFT < 500 ms" and "p50 < 500 ms" are different systems: the first forces admission control and queue bounds, the second permits a queue. A number at a percentile, or you have no SLO.
2. **What is the consistency requirement, and for which data?** Almost every system has both a small strongly-consistent core (quota ledgers, placement decisions, leader identity) and a large eventually-consistent bulk (routing tables, caches, metrics). Naming the split *per data class* rather than for the system is the [Lesson 01](01-consistency-models.md) move.
3. **What is the durability requirement — RPO and RTO?** RPO ("how much data may we lose") sets replication synchrony; RTO ("how fast must we be back") sets checkpoint/snapshot cadence and standby warmth. These are the two numbers that decide the storage tier, and candidates almost never ask for them.

Then the **scale envelope**: request rate, data volume, growth rate, and the size of the largest single unit (the largest model, the largest job, the largest tenant). That last one is the skew question and it is the one that breaks naive designs.

Finally — and this is the highest-signal, lowest-cost move in the entire method — **state non-goals explicitly**:

> "Out of scope: training and fine-tuning, multi-region failover, and per-token billing. I'll assume a single region and a separate billing pipeline. If you'd rather I cover multi-region, I'll trade it against the failure section."

That sentence bounds the problem, demonstrates you know what you excluded, and hands the interviewer a control. Google's published account of design-doc practice makes the same move structural: goals *and non-goals* are a mandatory section, precisely because a reviewer cannot evaluate a design without knowing what it declined to do.

### 4 · Step 2 — Estimation, as a dependency graph

The estimate is not throat-clearing. **It is the design decision**; every later step is bookkeeping that follows from it. But most candidates estimate ritualistically — computing QPS and storage because the template says to — and then draw an architecture that owes nothing to either number.

The fix is to work backwards from the decisions. Only compute a number if you can say which fork it chooses.

```
  ESTIMATION AS A DEPENDENCY GRAPH
  ═══════════════════════════════════════════════════════════════════════

  INPUTS (from requirements)          DERIVED NUMBER          DECISION IT MAKES
  ──────────────────────────          ──────────────          ─────────────────

  users × actions/user/day  ──┐
  peak-to-average ratio     ──┴──▶  peak QPS  ─────────────▶  how many stateless
                                        │                     front-ends; whether
                                        │                     a queue is needed
                                        │
  payload size ─────────────────┬──▶ ingress/egress ───────▶  NIC saturation?
                                │    bandwidth               CDN? compression?
                                │
  records/day × record size ──┬─┴──▶ hot dataset size ─────▶  ★ FITS IN RAM
  retention × replication   ──┘         │                       vs MUST SHARD
                                        │                     — the single most
                                        │                       architecture-
                                        │                       changing number
                                        ▼
                              total storage ────────────────▶  ★ SHARD COUNT
                                                                = ceil(total /
                                                                  per-shard cap)

  ── GPU-SPECIFIC BRANCH (this is where web-shaped estimates go wrong) ──

  model params × bytes/param ─────▶ weight footprint ──────▶  ★ TENSOR-PARALLEL
                                        │                       DEGREE (how many
                                        │                       GPUs per replica)
                                        ▼
  HBM per GPU − weights − overhead ─▶ KV budget ────┐
                                                     ├──▶ ★ CONCURRENCY CEILING
  2·layers·kv_heads·head_dim·dtype ─▶ KV bytes/token │      per replica
   × context length              ──▶ KV per request ─┘      (NOT QPS-derived)

  HBM bandwidth ÷ bytes read per ──▶ decode tokens/s ──────▶  ★ REPLICA COUNT
   decode step                          per replica
                                        │
  peak QPS × tokens/request ──────▶ required tokens/s ──────┘

  2 × params × prompt_tokens ─────▶ prefill FLOPs ─────────▶  does prefill
                                        │                     contend with
  achievable TFLOPS (≈80% of peak) ─────┘                     decode? →
                                                              chunked prefill or
                                                              disaggregation

  Little's Law: L = λ·W  ─────────▶ concurrent requests ───▶  CROSS-CHECK:
                                                              does the concurrency
                                                              ceiling actually
                                                              cover L?
```

**Anchor numbers.** You need a small set of magnitudes memorised so you can do this without a calculator. Jeff Dean's "numbers everyone should know" are the canonical set, and the right way to hold them is as *ratios*, since the absolute values have drifted with hardware since they were published:

| Operation | Order of magnitude | Ratio to the one above |
|---|---:|---|
| L1 cache reference | ~0.5 ns | — |
| L2 cache reference | ~7 ns | ~14× |
| Main memory reference | ~100 ns | ~14× |
| SSD random read | ~16 µs | ~160× |
| Round trip within a datacenter | ~0.5 ms | ~30× |
| Disk seek (spinning) | ~10 ms | ~20× |
| Round trip CA → Netherlands | ~150 ms | ~15× |

The load-bearing facts are the gaps, not the values: **memory is ~100× faster than SSD, SSD is ~30× faster than a datacenter round trip, and a cross-continent round trip is ~300× a local one.** Those three ratios decide caching, sharding, and multi-region questions respectively.

**GPU anchors**, which the classic table predates entirely:

| Quantity | Value | Provenance |
|---|---:|---|
| H100 SXM HBM3 capacity | 80 GB | NVIDIA H100 spec |
| H100 SXM HBM3 bandwidth | 3.35 TB/s | NVIDIA H100 spec |
| H100 SXM BF16 dense peak | 989 TFLOPS | derived and confirmed: 1830 MHz × 512 FMA/TC/clock × 2 × 528 TCs |
| H100 SXM BF16 *achievable* matmul | ~794 TFLOPS (~80 % of peak) | measured, `stas00/ml-engineering` MAMF benchmark |
| A100 SXM BF16 dense peak | 312 TFLOPS | same derivation, 1410 MHz × 256 × 2 × 432 |
| bf16 bytes per parameter | 2 B | — |

**Always estimate at ~70–80 % of peak, not at peak.** The measured max-achievable-matmul figure for the H100 at BF16 is about 80 % of the datasheet number, and real serving workloads sit below even that. A candidate who divides by peak TFLOPS gets a replica count that is 25 % too optimistic before any other error.

**The rules that keep an estimate fast and honest:**

- **Carry units through every step.** `req/s × B/req = B/s`. Most estimation errors are unit errors, and carrying units catches them for free.
- **Round aggressively, in the direction you can defend.** 86,400 s/day → 10⁵. 30 days → 3×10⁶ s. Say "I'm rounding up, so this is an upper bound."
- **State every assumption as you use it.** "I'll assume 2,000 input tokens and 400 output — tell me if that's wrong for your workload." This converts a guess into a shared premise, and if the interviewer corrects you, they have just handed you the real number.
- **Announce the binding constraint.** The estimate is not finished until you have said "so the thing that caps this system is X." If you cannot say that, you computed numbers but did not estimate.

### 5 · Step 3 — API

Three to four endpoints, with shapes. This is a three-minute block and its purpose is not the API — it is that writing the signatures forces you to commit to what the system *is*, and it exposes the **read/write asymmetry** that drives the rest of the design.

Ask of each endpoint: is it read or write? Is it synchronous or does it return a handle? Is it idempotent, and if so what is the key ([Lesson 07](07-data-intensive-patterns.md), §4)? What is its individual latency budget within the SLO?

The asymmetry is the payload. "10,000 reads per write" means a cache and read replicas. "Writes are 10× reads" means the write path is the system and reads are an afterthought. Say which you have.

### 6 · Step 4 — Data model, and the partition key

**Access patterns first, schema second.** List the queries the system must serve, in order of frequency, before drawing a single entity. Then design the schema so the dominant query is a single-partition lookup.

The **partition key is the single highest-leverage decision in the data model**, and it is chosen from the dominant query, never from the entity diagram. Say all four of these out loud:

1. **The key.** "Partition by `job_id`."
2. **What it makes cheap.** "Every read is a single-partition scan."
3. **What it makes expensive.** "Cross-job queries fan out to every partition — I'll serve those from a separate aggregate rather than fanning out."
4. **The skew it risks.** "The largest job is 82 % of the fleet's events, so `job_id` alone gives a 10× hot partition. I'll use `hash(job_id, gpu_uuid)` and give up per-job ordering, which is safe because the sink deduplicates by key rather than by position."

Point 4 is the staff move. Every partition key has a skew story; naming it unprompted is the difference between a design and a diagram.

### 7 · Step 5–6 — High-level design, then straight at the bottleneck

**Draw the boxes only now**, and annotate the edges with the numbers from the estimate. An unlabelled architecture diagram carries almost no information; the same diagram with "33,000 tok/s" and "128 concurrent/replica" on its edges is an argument.

Then — and this is the whole content of the scale step — **go directly to the constraint you named at the end of the estimate.** Do not scale the system uniformly. Do not add a cache "because caches are good". Every scaling move must be traceable to a number:

| Move | Justified by | Cost you must state |
|---|---|---|
| Add a cache | read:write ratio + hit-rate estimate | staleness window; stampede risk ([L04](04-caching.md)) |
| Shard | dataset size ÷ per-shard capacity | cross-shard queries; rebalancing ([L03](03-replication-and-partitioning.md)) |
| Add replicas | required throughput ÷ per-replica throughput | replication lag; read-your-writes ([L01](01-consistency-models.md)) |
| Add a queue | burst ratio vs. sustained capacity | it raises latency, not throughput ([L05](05-queueing-and-backpressure.md)) |
| Batch | per-item overhead vs. batch overhead | tail latency; the batch-size/TPOT tradeoff |

**The "and then?" discipline.** After you fix the first bottleneck, immediately ask what the *new* one is. Real scaling narratives are a chain of these, and being able to run two links of the chain unprompted is a strong signal. OpenAI's published account of scaling a single Kubernetes cluster is exactly this shape: past roughly 500 nodes they hit etcd write-latency spikes even on good hardware, fixed by moving the etcd data directory onto local SSD and splitting high-churn Kubernetes Events into a separate etcd cluster so their write load could not degrade the primary — and then the 7,500-node sequel hits a *different* binding constraint (pod networking, API-server load) at 3× the scale. That is step 6 happening for real, with etcd's fsync path as the constraint, exactly as [Lesson 02](02-consensus-and-quorums.md) frames it.

### 8 · Step 7 — Failure and operations, and the table it produces

This is where senior candidates stop and staff candidates start. The block has a concrete deliverable: **a guarantees and non-guarantees table.**

| | Guarantee | Mechanism | Non-guarantee (say this out loud) |
|---|---|---|---|
| Example row | "An accepted request either completes or returns 429/503 within the SLO" | admission control + bounded queue + deadline propagation | "We do not guarantee that a *rejected* request would have failed if accepted" |

Four things go in this block, in this order:

**(a) Failure modes, each with a blast radius.** Not "a node could fail" but "a node failure takes out 128 in-flight requests and 1/13th of capacity; the router drains its queue to siblings within one health-check interval." Draw on [Lesson 06](06-failure-and-resilience.md): correlated failure, gray failure, and the fact that your detector is imperfect.

**(b) Availability composition.** This is the arithmetic that kills "we'll just add a service". Serial dependencies multiply:

```
  A_total = ∏ A_i

  Critical path: LB (99.99) → gateway (99.95) → router (99.99) → replica pool (99.97)
  A = 0.9999 × 0.9995 × 0.9999 × 0.9997 = 0.9990
    = 99.90 %  →  0.001 × 8,760 h/yr ≈ 8.8 h/yr ≈ 43 min/month

  Add ONE more 99.95 % service on the critical path:
  A = 0.9990 × 0.9995 = 0.9985  →  13.1 h/yr  →  +4.4 h/yr from one component

  ⇒ The design move is not "make each service more available" but
    "get components OFF the critical path" — make the dependency soft
    (cache the answer, default it, degrade without it).
```

And redundancy does *not* square your availability if the replicas share anything: with `u = u_independent + u_common`, the common-mode term is a floor no number of replicas crosses ([Lesson 06](06-failure-and-resilience.md), §12).

**(c) Degraded mode.** What does the system do when it cannot meet the SLO? The answer must be a *designed behaviour*, not a collapse: cap `max_tokens`, serve a smaller model, drop optional enrichment, return a cached answer, shed with `429 + Retry-After`. Naming the degraded mode is a strong signal precisely because most candidates only design the happy path and the total-failure path.

**(d) RPO/RTO, and what pages.** State the recovery-point and recovery-time objectives, and then state which alerts wake a human. "Queue depth above X for 5 minutes pages; a single replica loss does not" tells the interviewer you have operated something.

### 9 · Step 8 — Tradeoffs, and the rejected branch

The final block is where staff signal concentrates, and it has a specific shape. For each of 2–3 axes:

> **Axis · chosen end · rejected end · the number that decided it.**

Worked examples of the shape:

- "This is a **durability vs. write-latency** call. I chose async replication with a bounded RPO of 30 seconds over synchronous cross-region, because sync adds roughly 80 ms to every write p99 at that distance and the product tolerates 30 seconds of loss. If the RPO requirement were zero I'd flip it and pay the 80 ms."
- "This is a **freshness vs. load** call on the placement table. I chose a 5-second cached view over a consensus read, because a stale route costs one retry — about 40 ms — while a quorum read adds ~2 ms to every one of 83 requests per second *and* puts the router in etcd's failure domain. If a stale route could cause a double-allocation instead of a retry, I'd flip it."
- "This is **blast radius vs. efficiency**. I chose 4 cells of 4 replicas over one pool of 13, accepting ~23 % more idle headroom, because it caps a bad-model-version rollout at 25 % of traffic."

Three rules for this block:

1. **Volunteer it.** A tradeoff named after the interviewer probes for it scores as a *recovery*, not as a signal. Say it before the question.
2. **Quantify it.** "Adds latency" is a hedge; "adds ~80 ms to write p99" is an argument.
3. **State the flip condition.** "If X were true I'd choose the other end" proves you understand the axis rather than having memorised a preference. It is also the single most reliable way to demonstrate senior-vs-staff separation in one sentence.

### 10 · Driving: the verbal patterns

The method is a script; delivering it is a separate skill. Five patterns, all of them cheap:

- **Narrate the decision, not the conclusion.** "Because the KV budget gives 128 concurrent per replica and I need 1,100 concurrent, I need at least 9 replicas — I'll take 13 for headroom" is worth ten times "I'll use 13 replicas."
- **Ask for the number you need, once.** "Do you have a real figure for average output length, or should I assume 400 tokens?" Then commit and move. Asking twice reads as stalling.
- **Take corrections as inputs.** If the interviewer says "actually it's 10× that," say "then the KV ceiling moves from 128 to 12 per replica and I need 130 replicas, which makes weight-loading time the new problem" — and keep going. The recalculation *is* the test.
- **Handle "what if 10×" by naming which number breaks first.** Not "we'd add more servers" but "at 10× the concurrency ceiling is unchanged per replica, so it is linear in replicas until the router's placement table stops fitting in memory at around N — that is where the architecture changes."
- **When you don't know, say what you'd measure.** "I don't know the real p99 arrival delay for that source; I'd instrument it for a week and set the grace window from the max observed plus margin." That is a better answer than a confident guess and interviewers score it that way.

### 11 · Why the GPU branch is structurally different

The most common failure on a GPU prompt is not arithmetic — it is estimating the wrong quantity. Three specific divergences:

**(a) Decode is memory-bandwidth-bound, not compute-bound.** In autoregressive decode, generating one token requires reading the *entire* weight matrix set from HBM, plus the KV cache for every active sequence, and doing comparatively little arithmetic with it. The step time is therefore:

```
  t_step ≈ (bytes_read_per_step) / (HBM bandwidth × achievable efficiency)

  bytes_read_per_step = weights_on_this_GPU + Σ over active seqs of their KV
```

FLOPs barely enter it. A candidate who computes "70B params × 2 FLOPs × 400 tokens ÷ 989 TFLOPS" has computed something true and irrelevant, and will size the fleet wrong by an order of magnitude. The worked example below shows decode using ~17 % of the available FLOPs while saturating bandwidth — that ratio *is* the argument.

**(b) Concurrency is capped by HBM, not by request rate.** The number of requests a replica can hold is `KV_budget ÷ KV_per_request`, and `KV_budget = HBM − weights − activations − fragmentation`. This is a hard ceiling: exceed it and requests are evicted or queued, regardless of how much CPU or network you have. Sizing on QPS alone produces a fleet that is simultaneously idle on request rate and completely full on memory.

**KV cache per token, derived from the model config** — you should be able to write this from memory:

```
  KV bytes/token = 2 (K and V)
                 × num_hidden_layers
                 × num_key_value_heads          ← the GQA number, NOT num_attention_heads
                 × head_dim                     ← hidden_size / num_attention_heads
                 × bytes_per_element            ← 2 for bf16/fp16, 1 for fp8

  For a 70B-class GQA decoder (80 layers, 8 KV heads, head_dim 128, bf16):
      2 × 80 × 8 × 128 × 2 B = 327,680 B = 320 KiB per token

  Read num_hidden_layers / num_key_value_heads / hidden_size / num_attention_heads
  out of the model's own config.json — do not memorise them per model.
```

**The GQA factor is where people go wrong.** Using `num_attention_heads` (64) instead of `num_key_value_heads` (8) overstates KV by 8× and produces a design with an imaginary memory crisis. Grouped-query attention exists precisely to shrink this term.

**(c) Prefill and decode are different workloads competing for the same silicon.** Prefill processes the whole prompt in parallel and is compute-bound (`≈ 2 × params × prompt_tokens` FLOPs). Decode is bandwidth-bound and produces one token per step. Running them on the same replica means a long prompt's prefill stalls every in-flight decode — the classic "TTFT is fine but TPOT spikes when someone pastes a document" symptom. Naming this, and naming the two mitigations (chunked prefill, or disaggregating prefill and decode onto separate replica pools), is a strong GPU-specific signal.

**(d) Cold start is a first-class constraint.** Loading a 70B bf16 model means moving 140 GB. Over a 200 Gb/s link at a realistic 20 GB/s you get ~7 s; over 100 Gb/s at ~10 GB/s, ~14 s; from local NVMe at ~6 GB/s, ~23 s. Any of those makes reactive autoscaling on a 60-second window useless — the capacity arrives after the burst is over. The design consequence is warm pools and pre-pulled weights, and stating that consequence *from* the number is the move.

### 12 · The same method, outside the interview

This eight-step framework is a compressed design document, and knowing that changes how seriously to take it. Google's published account of design-doc culture describes the standard shape: context and scope, **goals and non-goals**, the proposed design, **alternatives considered with their tradeoffs**, and cross-cutting concerns. Those map one-to-one onto steps 1, 5–7, and 8 here. The document exists so reviewers can evaluate the tradeoffs *before* code is written — and a design doc that omits the rejected alternatives is considered incomplete by the same reviewers who would mark a candidate down for the same omission.

Two published artifacts worth reading as calibration for what "good" looks like at the top of the range: Meta's MAST paper, which states its problem with a number (the worst-region GPU demand/supply ratio was 2.63 before, 0.98 after), states one design principle (temporal decoupling: a fast real-time scheduling path plus a slow continuously-reoptimising path), and ties every architectural choice back to that number; and CoreWeave's SUNK write-up, which resolves a Slurm-vs-Kubernetes framing by refusing it and building the integration layer instead — "state what's out of scope, then design to the actual constraint" done for real by a GPU-cloud operator.

## Perspectives

**The estimator's view.** The back-of-the-envelope step is not preamble to the "real" design work — it *is* the design decision, and every later step is bookkeeping that follows from it. A candidate who draws the architecture before doing the math is structurally guessing, and an interviewer can tell because none of the boxes will be justified by anything.

**The interviewer's view.** From the other side of the table, staff signal looks like: scoping fast and moving on, producing a number unprompted and using it to eliminate an option, naming a rejected alternative before being asked "why not X", and spending real minutes on failure modes rather than treating them as a bonus round. The rubric in §1 is roughly what gets written on the feedback form.

**The organisational view.** The interview version and the real design doc are the same skill at two clock speeds. Practising this framework is practising how to write and defend the artifact that gets a quarter of engineering time committed — which is why the portfolio deliverable for this module is written as design docs, not as interview transcripts.

**The GPU-unit-economics view.** QPS-first thinking is a category error for GPU systems, not merely an oversimplification. A web service's cost scales with request count; a GPU serving system's cost scales with HBM-GB reserved per concurrent request and with GPU-seconds consumed, and a fleet can be QPS-underloaded while completely full on KV cache. Estimating in QPS on a GPU prompt sizes the wrong resource entirely — the worked example makes the ceiling-flip explicit.

## Real-world use cases

- **Meta, "MAST: Global Scheduling of ML Training across Geo-Distributed Datacenters at Hyperscale," OSDI '24** — <https://www.usenix.org/system/files/osdi24-choudhury.pdf> — *What it shows:* a staff-level design write-up in miniature. It states the problem with a number (worst-region GPU demand/supply ratio 2.63 → 0.98), states one design principle (temporal decoupling — a fast-path real-time scheduler plus a slow-path continuous reoptimisation loop), and ties every architectural choice back to that number. Read it as the calibration target for step 8.
- **OpenAI, "Scaling Kubernetes to 2,500 nodes"** and **"Scaling Kubernetes to 7,500 nodes"** — <https://openai.com/index/scaling-kubernetes-to-2500-nodes/> · <https://openai.com/index/scaling-kubernetes-to-7500-nodes/> — *What it shows:* the "and then?" discipline from §7 running for real at a GPU-fleet operator. etcd write-latency spikes past ~500 nodes, fixed by local SSD for the etcd data directory plus isolating high-churn Events into a separate etcd cluster; the sequel repeats the pattern at 3× the scale with a different binding constraint.
- **CoreWeave, "A Slurm on Kubernetes Implementation for HPC and Large Scale AI" (SUNK)** — <https://www.coreweave.com/blog/sunk-slurm-on-kubernetes-implementations> — *What it shows:* step 1 done well. Rather than answering "Slurm or Kubernetes", CoreWeave scoped the actual constraint and built an integration layer that runs Slurm's batch model on Kubernetes' orchestration. Refusing a false binary, with a stated reason, is a legitimate and strong interview move.
- **Malte Ubl, "Design Docs at Google"** — <https://www.industrialempathy.com/posts/design-docs-at-google/> — *What it shows:* that steps 1 and 8 are not interview inventions. Goals/non-goals and alternatives-considered are mandatory sections of the real artifact, for the same reason they are graded here — a reviewer cannot evaluate a design without knowing what it declined to do and what it rejected.

## Worked example

**Prompt: "Design an inference gateway and router for a multi-model GPU fleet."** Run all eight steps, on the clock, with every number shown. This is the shape the portfolio artifact should take.

### Block 1 (0–5 min) — Requirements

**Functional.** (1) Accept a generation request naming a model, stream tokens back. (2) Route it to a replica that serves that model, preferring one that already holds the conversation's prefix. (3) Enforce per-tenant rate and concurrency limits. (4) Expose which models are available and their status.

**Non-functional.**

| Requirement | Value | Consequence |
|---|---|---|
| Latency SLO | p99 **TTFT < 500 ms**; p99 **TPOT < 50 ms** | TTFT bounds prefill + queue; TPOT bounds batch size |
| Availability | 99.9 % monthly (≈ 43 min) | budget the critical-path chain |
| Consistency — placement table | eventual, ≤ 5 s stale | a stale route costs one retry, not correctness |
| Consistency — tenant quota ledger | strong | double-admission is a real cost |
| Durability | request/response log RPO 5 min, RTO 30 min | async ship to object storage is fine |
| Models | 7B, 13B, 70B classes, bf16 | 70B dominates the sizing |

**Scale envelope.** 5,000 requests/min peak; mean 2,000 input tokens, 400 output tokens; 200 tenants; largest tenant is 40 % of traffic.

**Non-goals, stated out loud.** Training and fine-tuning; multi-region failover (single region assumed); per-token billing (separate pipeline — that is [Lesson 07](07-data-intensive-patterns.md)'s design); model fine-tune storage. Also out: the inference engine itself — I assume a paged-KV engine per replica and design the layer above it.

### Block 2 (5–12 min) — Estimation

**Request rate.**

```
  5,000 req/min ÷ 60 = 83.3 req/s peak
  output token rate = 83.3 req/s × 400 tok = 33,333 tok/s   ← the demand number
  input token rate  = 83.3 req/s × 2,000 tok = 166,667 tok/s (prefill)
```

**Replica shape for a 70B model.** Weights = 70×10⁹ params × 2 B = **140 GB**. One H100 has 80 GB, so a replica needs ≥ 2 GPUs; take **TP = 4** (4×H100 = 320 GB) for headroom and lower TPOT.

```
  Per GPU:  weights shard      140 / 4          = 35 GB
            activations + frag  ~5 GB (assumed, engine-dependent)
            KV budget           80 − 35 − 5     = 40 GB
  Per replica KV budget:        4 × 40          = 160 GB
```

**KV cache per request.** Assume a 70B-class GQA decoder: 80 layers, 8 KV heads, head_dim 128, bf16 — read from `config.json`, not memorised.

```
  KV per token   = 2 × 80 × 8 × 128 × 2 B = 327,680 B = 320 KiB
  KV per request at 4,096-token context
                 = 320 KiB × 4,096 = 1,310,720 KiB = 1.25 GiB

  ★ CONCURRENCY CEILING per replica = 160 GB / 1.25 GiB ≈ 128 concurrent requests
```

**Decode throughput, from bandwidth.** Per decode step each GPU reads its weight shard plus its share of the KV for every active sequence.

```
  per-GPU KV shard per sequence = 1.25 GiB / 4 = 0.328 GB
  bytes read per step (batch B) = 35 GB + 0.328·B GB
  effective bandwidth           = 3.35 TB/s × 0.70 = 2,345 GB/s   (70 % achievable)

  B =  32 : (35 + 10.5) / 2345 = 19.4 ms/step  →  32/0.0194  = 1,649 tok/s
  B =  64 : (35 + 21.0) / 2345 = 23.9 ms/step  →  64/0.0239  = 2,678 tok/s
  B = 128 : (35 + 42.0) / 2345 = 32.8 ms/step  → 128/0.0328  = 3,902 tok/s
```

Note what the table says: **throughput rises with batch, TPOT worsens with batch, and the SLO picks the row.** p99 TPOT < 50 ms admits B = 128 (32.8 ms) with margin; if the SLO were 25 ms we would be forced to B = 64 and would need ~46 % more replicas.

**Is decode compute-bound? Check, don't assume.**

```
  decode FLOPs at B=128: 2 × 70×10⁹ × 128 tokens/step ÷ 0.0328 s = 546 TFLOPS
  replica compute available: 4 × 794 TFLOPS (achievable)          = 3,176 TFLOPS
  → decode uses 17 % of the FLOPs while saturating bandwidth.
  ⇒ CONFIRMED memory-bandwidth-bound. Sizing on TFLOPS would be wrong.
```

**Replica count, two independent ways — take the max.**

```
  (a) Throughput:  33,333 tok/s ÷ 3,902 tok/s per replica = 8.54  → 9 replicas

  (b) Concurrency (Little's Law, L = λ·W):
        W = TTFT + 400 × TPOT ≈ 0.35 s + 400 × 0.0328 s = 13.5 s
        L = 83.3 req/s × 13.5 s = 1,124 concurrent requests
        replicas = 1,124 / 128 = 8.8  → 9 replicas

  Both give 9 — reassuring, and expected, since the two are linked through B.
  At 9 replicas utilisation is 1,124/1,152 = 97.6 % — no headroom at all.

  ★ Target 70 % utilisation for burst absorption and single-replica loss:
        1,124 / 0.70 = 1,606 concurrent → 1,606 / 128 = 12.5 → 13 REPLICAS
        = 13 × 4 = 52 H100s for the 70B tier
```

**Prefill contention — the finding that changes the design.**

```
  prefill FLOPs per request = 2 × 70×10⁹ × 2,000 tokens = 2.8×10¹⁴ = 280 TFLOPs
  fleet prefill demand      = 83.3 req/s × 280 TFLOPs   = 23,333 TFLOPS
  fleet compute available   = 13 replicas × 3,176       = 41,288 TFLOPS

  ⇒ prefill wants 56 % of all fleet FLOPs, on the same silicon that is
    supposed to be running decode. A single 2,000-token prefill takes
    280 / 3,176 = 88 ms of pure compute on a replica — during which every
    in-flight decode on that replica stalls, blowing p99 TPOT.

  ★ THE BINDING CONSTRAINT IS NOT THROUGHPUT, IT IS PREFILL/DECODE INTERFERENCE.
```

**Auxiliary numbers.**

```
  Weight load (cold start): 140 GB ÷ ~20 GB/s (200 Gb/s, realistic)  ≈ 7 s
                            140 GB ÷ ~6 GB/s (local NVMe)            ≈ 23 s
  ⇒ reactive autoscaling on a 60 s window is useless; keep warm replicas.

  Request/response log: 83.3 req/s × ~9.6 KB ≈ 800 KB/s ≈ 69 GB/day
                        30-day retention × RF 3            ≈ 6.2 TB

  Placement table: ~40 replicas × ~300 B = 12 KB. Fits in a CPU cache line
  budget, let alone RAM ⇒ never shard it; the only question is freshness.
```

### Block 3 (12–15 min) — API

```
POST /v1/generate
  { "model": "llama-70b", "messages": [...], "max_tokens": 400,
    "stream": true, "session_id": "s-8f2c" }        ← session_id enables prefix routing
  → 200, Server-Sent Events stream of token deltas
  → 429 { "retry_after_ms": 1200 }                  ← the shed path, first-class
  → 503 { "reason": "no_capacity_for_model" }

GET  /v1/models
  → [ { "id": "llama-70b", "ready_replicas": 13, "queue_depth_p50": 4 } ]

GET  /v1/generate/{request_id}      ← recovery for a dropped stream
POST /v1/admin/replicas/{id}/drain  ← operator control, used by deploys
```

**Asymmetry:** this is a write-shaped, long-lived-connection system — one expensive streaming call, essentially no reads. So there is no read cache to add; the caching that matters is *inside* the replica (the KV prefix cache), which is why `session_id` is in the request and why routing is affinity-aware. Saying that here saves five minutes later.

### Block 4 (15–20 min) — Data model

**Access patterns, in frequency order:**

1. "Which replicas serve model M, and which one holds session S's prefix?" — 83/s, must be < 1 ms.
2. "Has tenant T exceeded its concurrency limit?" — 83/s, must be strongly consistent.
3. "What happened to request R?" — rare, offline, must be durable.

**Three stores, because they have three different requirements:**

| Store | Contents | Size | Consistency | Where |
|---|---|---:|---|---|
| Placement table | replica → {model, address, health, load, prefix-cache summary} | ~12 KB | eventual, ≤ 5 s | in-process cache in every gateway, fed by a watch |
| Tenant quota ledger | tenant → {concurrency limit, in-flight count} | ~200 tenants × 100 B = 20 KB | strong | single-writer per tenant, sharded by `tenant_id`; a CAS store |
| Request log | request/response, token counts, latencies | 6.2 TB / 30 d | none (append) | object storage, partitioned by hour |

**Partition keys.** Session-affinity routing: `hash(session_id)` with **rendezvous (HRW) hashing** rather than `hash mod N`, so that adding or removing a replica remaps only ~1/N of sessions instead of nearly all of them ([Lesson 03](03-replication-and-partitioning.md)). Quota ledger: `tenant_id` — skew accepted, because the largest tenant is 40 % of traffic but the ledger operation is a counter increment costing microseconds, so a 40 % hot key is irrelevant. **Say the skew and say why it does not matter**; that is the same move as calling out a skew that does.

**Request log shard count — the arithmetic:**

```
  Total    = 6.2 TB (30 days, RF 3)
  Per node = 4 TB NVMe × 50 % (compaction + growth headroom) = 2 TB usable

  (a) storage-bound   : ceil(6.2 / 2)                    = 4 shards
  (b) write-bound     : 800 KB/s ÷ (200 MB/s per shard)  = 1 shard
  (c) read-bound      : ~1 lookup/s                      = 1 shard

  shard count = max(a, b, c) = 4  →  round to 4 (already a power of two,
  which keeps future splits a clean 4 → 8 doubling)

  ⇒ STORAGE-bound, not throughput-bound. Any argument in this design that
    appeals to write throughput for the log is wrong.
```

### Block 5 (20–30 min) — High-level design, then the bottleneck

```
   INFERENCE GATEWAY — annotated with the estimate's numbers
   ═══════════════════════════════════════════════════════════════════════

   clients
     │  83 req/s peak, SSE streams, ~13.5 s mean hold time
     ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │  GATEWAY (stateless, N=6 for 99.95 %; each holds ~190 open streams) │
  │   1. authn + tenant lookup                                          │
  │   2. QUOTA CHECK ── CAS against ledger (strong) ── 429 if over      │
  │   3. ADMISSION: predicted queue wait > TTFT budget?  → 429 early    │
  │   4. ROUTE: HRW(session_id) over replicas serving `model`,          │
  │             weighted by (free KV blocks, queue depth)               │
  └──────────────┬──────────────────────────────────────────────────────┘
                 │  placement table: 12 KB, watch-fed, ≤5 s stale
                 │  (stale route ⇒ one retry ≈ 40 ms, NOT an error)
                 ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │  70B TIER — 13 replicas × (4 × H100 TP=4) = 52 GPUs                   │
  │  ┌───────────────────────────────────────────────────────────────┐    │
  │  │ replica: KV budget 160 GB │ 1.25 GiB/req @4k │ CEILING 128    │    │
  │  │          decode 3,902 tok/s @ B=128, TPOT 32.8 ms             │    │
  │  │          bounded queue: 32 slots, LIFO under load (L05)       │    │
  │  └───────────────────────────────────────────────────────────────┘    │
  │  utilisation target 70 % ⇒ 1,606 concurrent capacity vs 1,124 demand  │
  └───────────────────────────────────────────────────────────────────────┘
       │ 7B / 13B tiers sized by the same three formulas (smaller weights
       │ ⇒ larger KV budget ⇒ far higher concurrency ceiling per GPU)
       ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │  request log → object storage, 4 shards, 800 KB/s, 6.2 TB / 30 d      │
  └───────────────────────────────────────────────────────────────────────┘

   ★ FIRST BOTTLENECK (from the estimate): prefill/decode interference.
     A 2,000-token prefill costs 88 ms of compute and stalls every decode
     on that replica; at 83 req/s of prefill demand this happens constantly.
```

**Fix the bottleneck — two options, and the choice:**

| Option | Mechanism | Cost | Verdict |
|---|---|---|---|
| **Chunked prefill** | Split a prefill into ~512-token chunks and interleave them with decode steps in the same batch | Prefill latency rises (~4 chunks × scheduling overhead); TTFT worsens by roughly 15–25 % | **Chosen.** Keeps one homogeneous fleet; TTFT budget has room (88 ms of prefill in a 500 ms budget) |
| **Disaggregated prefill/decode** | Separate replica pools; transfer KV from prefill node to decode node over the fabric | Must move 1.25 GiB of KV per request across the fabric — at 25 GB/s that is 50 ms and 83×1.25 GiB/s = 104 GB/s of sustained fabric traffic | Rejected at *this* scale; revisit if prompt length grows past ~8k, where prefill dominates |

**Then the next bottleneck.** With prefill chunked, the constraint becomes the concurrency ceiling itself, and the lever is the KV footprint: FP8 KV cache halves `KV per token` from 320 KiB to 160 KiB, doubling the ceiling from 128 to 256 per replica and cutting the replica count from 13 to 7 — at an accuracy cost that must be measured, not assumed. **Name it as the next move and say what evidence would justify it.**

### Block 6 (30–38 min) — Failure and operations

**Guarantees and non-guarantees:**

| | Guarantee | Mechanism | Non-guarantee — stated explicitly |
|---|---|---|---|
| Admission | An accepted request completes or returns an error within its deadline | deadline stamped at ingress; dropped when queue-implied wait exceeds it | A 429 does not mean the fleet was full — it means *this route* was; a retry may succeed |
| Routing | Best-effort session affinity | HRW over a ≤ 5 s-stale placement table | Affinity is not guaranteed; a miss costs a prefill, not a wrong answer |
| Quota | No tenant exceeds its concurrency limit | CAS on a strongly-consistent ledger | Limits are *concurrency*, not tokens/s; a tenant with long generations uses more GPU-seconds within the same limit |
| Durability | Request log RPO 5 min | async batched ship to object storage | An in-flight generation is lost on gateway crash; the client must retry |
| Streams | A dropped SSE stream is resumable for 60 s | `GET /v1/generate/{id}` replays buffered tokens | Beyond 60 s the request is gone |

**Failure modes with blast radius:**

- *One replica lost.* 128 in-flight requests fail (~11 % of concurrent load); capacity drops 7.7 %. Gateway removes it on the next health interval; its queue is redistributed. **Bounded and survivable because we sized at 70 % utilisation** — the whole reason that headroom number exists.
- *One gateway lost.* ~190 open streams drop; clients retry. Stateless, so no state loss.
- *Quota ledger unavailable.* Two options, and this is a real CAP call: fail closed (reject all traffic — safe, catastrophic) or fail open with locally-cached limits (over-admission possible, service continues). **Chosen: fail open with the last-known limits and a hard fleet-wide concurrency cap as the backstop**, because over-admission costs money while fail-closed costs the product. State the flip: if the ledger governed *money* rather than concurrency, fail closed.
- *Correlated failure.* All 13 replicas run the same model version and the same driver. A bad model push is a fleet-wide event that no amount of replication protects against. **Mitigation: 4 cells of 3–4 replicas, deploys advance one cell at a time** — capping a bad push at ~25 % of traffic, for ~23 % more idle headroom.
- *Gray failure.* A GPU whose bandwidth has degraded 30 % keeps serving, slowly, and passes every liveness check. **Detection is differential** ([Lesson 06](06-failure-and-resilience.md), §10): compare per-replica p50 TPOT against the fleet median and eject beyond 1.9σ, with a cap of 10 % of the fleet ejected at once.

**Availability composition:**

```
  LB 99.99 × gateway 99.95 × quota ledger 99.99 × replica pool 99.97
      = 0.9990  →  99.90 %  ≈ 43 min/month     ✓ meets the target, barely

  Note the quota ledger is ON the critical path and contributes 0.01 %.
  Failing open (above) takes it OFF the critical path for availability
  purposes, buying back ~53 min/year for the cost of possible over-admission.
  ⇒ The availability arithmetic is what turns "fail open vs fail closed"
    from a philosophical argument into a numbers argument.
```

**Degraded mode, in escalating order:** (1) reduce max batch size to protect TPOT; (2) cap `max_tokens` fleet-wide; (3) route overflow from 70B to 13B with a response header saying so; (4) shed with `429 + Retry-After` — **never** accept a request that cannot meet its SLO ([Lesson 05](05-queueing-and-backpressure.md)).

**What pages:** fleet concurrency > 90 % of ceiling for 5 min; p99 TTFT > SLO for 10 min; any replica ejected for numerical mismatch. **What does not page:** a single replica loss; a queue spike under 2 minutes.

### Block 7 (38–45 min) — Tradeoffs

1. **Freshness ↔ load, on the placement table.** *Chosen:* a ≤ 5 s-stale watch-fed cache in each gateway. *Rejected:* a consensus read per request. *The number:* a stale route costs one retry ≈ 40 ms on a small fraction of requests; a quorum read costs ~2 ms on **every** request and puts the gateway inside the ledger's failure domain, converting a ledger blip into a total outage. *Flip condition:* if a stale route caused a double-allocation instead of a retry, the cost model reverses and I would pay for the strong read.
2. **Throughput ↔ tail latency, on batch size.** *Chosen:* B = 128, giving 3,902 tok/s and 32.8 ms TPOT. *Rejected:* B = 64, giving 2,678 tok/s and 23.9 ms TPOT. *The number:* the SLO is 50 ms TPOT, so B = 128 fits with 34 % margin while B = 64 would require 46 % more replicas — about 24 additional H100s — for latency the product does not ask for. *Flip condition:* an interactive-coding product with a 25 ms TPOT requirement flips this immediately.
3. **Blast radius ↔ efficiency, on cells.** *Chosen:* 4 cells of 3–4 replicas. *Rejected:* one pool of 13. *The number:* cells cap a bad model push at 25 % of traffic and cost ~23 % more idle headroom, because each cell must independently absorb burst and one replica loss. *Flip condition:* if deploys were validated by a shadow fleet, the containment value drops and the single pool wins.

**Explicitly not chosen, and why:** disaggregated prefill/decode (fabric cost of 104 GB/s sustained KV transfer exceeds its benefit at 2k prompts); FP8 KV cache (would halve the replica count — deferred pending an accuracy measurement, not rejected); multi-region (declared a non-goal in block 1).

## Practice

Produce `inference-gateway-design.md` as a portfolio write-up running all eight steps on this prompt, in your own numbers. It must contain, at minimum:

1. A **requirements** section with an SLO at a percentile, an availability target, a per-data-class consistency statement, RPO/RTO, and an explicit **non-goals** list.
2. An **estimation** section showing, with units carried: request rate and token rate; weight footprint and the TP degree it forces; KV bytes/token derived from `2 × layers × kv_heads × head_dim × dtype_bytes`; the concurrency ceiling; decode step time from bandwidth (with an achievable-efficiency factor, not peak); the FLOPs check proving decode is bandwidth-bound; replica count computed **two ways** (throughput and Little's Law) and reconciled; and the utilisation-headroom multiplier.
3. An **API** of 3–4 endpoints including the shed path.
4. A **data model** with access patterns first, a partition key with its skew story, and one **shard-count derivation** taking the max over storage, write, and read constraints.
5. A **high-level design diagram in ASCII**, with the estimate's numbers annotated on it.
6. A **scale** section that goes straight to the binding constraint, evaluates at least two fixes in a table with their costs, and names the *next* bottleneck.
7. A **failure** section containing a **guarantees / non-guarantees table**, blast radius per failure mode, an **availability composition calculation**, a degraded-mode ladder, and what pages.
8. A **tradeoffs** section with ≥ 3 axes, each with chosen end, rejected end, the number that decided it, and the **flip condition**.

*Acceptance:* every box in the diagram traces to a number in section 2; the phrase "it depends" appears nowhere without a following "on X, and here the value of X is Y"; and a reader who disagrees with one of your assumptions can re-run the arithmetic and get a different design. Then do it once **on a clock, out loud, in 45 minutes**, and compare — the gap between the written version and the timed version is the thing [Lesson 09](09-design-rehearsal.md) trains.

Feeds the [staff design portfolio](../practice/staff-design-portfolio/README.md).

## Common pitfalls

1. **Drawing the boxes first.** *Symptom:* a clean architecture that the interviewer immediately dismantles with "why that many?" *Mechanism:* the numbers choose the components; without them every box is a guess, and none of them can be defended. MAST's ratio numbers and OpenAI's scale numbers precede their architectures in the real write-ups too. Draw after the math.
2. **Estimating in QPS on a GPU prompt.** *Symptom:* a fleet sized at 9 replicas that turns out to hold 128 concurrent requests each while the workload needs 1,600. *Mechanism:* the card is capped by HBM — KV bytes per concurrent request plus weights — long before it is capped by request rate. Size on the concurrency ceiling and on bandwidth-derived token throughput, then cross-check with Little's Law.
3. **Sizing decode on FLOPs.** *Symptom:* an estimate that says one H100 serves thousands of concurrent 70B requests. *Mechanism:* decode reads the entire weight set from HBM every step and does little arithmetic with it; in the worked example it uses 17 % of available FLOPs while saturating bandwidth. Divide bytes-read-per-step by bandwidth, not FLOPs by TFLOPS — and use ~70–80 % of peak, since measured achievable matmul throughput on an H100 at BF16 is about 80 % of the datasheet figure.
4. **Using `num_attention_heads` in the KV formula.** *Symptom:* an 8× overstatement of KV cache and a design that solves an imaginary memory crisis. *Mechanism:* grouped-query attention shares K/V across query heads; the KV term scales with `num_key_value_heads` (8 for a 70B-class model), not with the 64 query heads. Read both out of `config.json`.
5. **Spending the clock on requirements.** *Symptom:* minute 18 and no numbers on the board. *Mechanism:* the score comes from estimation, failure, and tradeoffs; requirements are a five-minute framing block. Announce your time budget at the start so that moving on is expected rather than abrupt.
6. **Naming a tradeoff only after being asked.** *Symptom:* every tradeoff in the transcript is preceded by an interviewer question. *Mechanism:* a volunteered tradeoff demonstrates that you searched the design space; a prompted one demonstrates only that you can answer. Attach a number and a flip condition, and say it before the probe.
7. **Treating the failure section as a bonus round.** *Symptom:* "and if we had more time I'd talk about failure modes." *Mechanism:* that section is where the senior/staff line is drawn, so cutting it cuts exactly the signal you were there to produce. Cut the data model or the API instead — and say which you are cutting and why.
8. **Preferring the symmetric design.** *Symptom:* one uniform pool, one uniform consistency model, one uniform replication scheme. *Mechanism:* real systems that ship are asymmetric because real constraints are — MAST's fast-path/slow-path split, CoreWeave's hybrid rather than a Slurm-or-Kubernetes choice, this design's three stores with three different consistency requirements. Symmetry is an aesthetic preference; constraints are the input.

## Self-check

- **Why is the estimate, not the diagram, the load-bearing part of the method?** Because the numbers eliminate architectures. "150 GB working set" and "15 TB working set" are different systems, and no amount of drawing decides between them. On the GPU branch specifically, the KV-cache-per-request number sets the concurrency ceiling per replica, which sets the fleet size and the admission threshold; the diagram then renders a decision the arithmetic already made. Operationally: if you cannot say which fork a number chooses, do not compute it — and the estimate is not finished until you have said out loud what the binding constraint is.
- **Write the KV-cache formula and compute it for a 70B-class model.** `KV bytes/token = 2 (K and V) × num_hidden_layers × num_key_value_heads × head_dim × bytes_per_element`, with `head_dim = hidden_size / num_attention_heads`. For 80 layers, 8 KV heads (GQA — *not* the 64 query heads), head_dim 128, bf16: `2 × 80 × 8 × 128 × 2 = 327,680 B = 320 KiB/token`, so a 4,096-token context is **1.25 GiB per request**. With 140 GB of weights across TP = 4 on 4×H100 (35 GB/GPU), ~5 GB/GPU of activations and fragmentation, the KV budget is 4 × 40 = 160 GB, giving a **concurrency ceiling of 128 requests per replica**. Halving the element size (FP8 KV) doubles that to 256.
- **Show that decode is bandwidth-bound rather than compute-bound, with numbers.** Per decode step a GPU reads its weight shard (35 GB) plus its share of KV for every active sequence (0.328 GB × B). At B = 128 that is 77 GB; at 3.35 TB/s × 70 % achievable = 2,345 GB/s, the step takes 32.8 ms, so the replica emits 128/0.0328 = 3,902 tok/s. The FLOPs used are `2 × 70×10⁹ × 128 ÷ 0.0328 s = 546 TFLOPS` against 4 × 794 = 3,176 TFLOPS achievable — **17 % of the compute, 100 % of the bandwidth**. Hence step time is `bytes_read / (bandwidth × efficiency)` and TFLOPS is the wrong denominator.
- **Size a fleet two independent ways and reconcile them.** (a) Throughput: demand 83.3 req/s × 400 output tokens = 33,333 tok/s; ÷ 3,902 tok/s per replica = 8.54 → 9 replicas. (b) Little's Law: `W = TTFT + 400 × TPOT ≈ 0.35 + 13.1 = 13.5 s`; `L = λ·W = 83.3 × 13.5 = 1,124` concurrent; ÷ 128 per replica = 8.8 → 9 replicas. They agree because both are linked through the batch size. Nine replicas is 97.6 % utilisation, which leaves nothing for burst or for a single replica loss — so target 70 % utilisation: `1,124/0.7 = 1,606` concurrent → **13 replicas, 52 H100s**. The headroom multiplier is not decoration; it is what makes the "one replica lost" failure mode survivable.
- **Ten dependencies on the critical path, each 99.9 %. What do you tell the interviewer?** `0.999¹⁰ = 0.99005` → 99.0 %, about 87 hours of downtime a year. The design consequence is not "make each service more available" — it is **get components off the critical path**: cache the answer, default it, or fail open so the dependency's availability stops multiplying into yours. In the worked design, failing open on the quota ledger removes its 99.99 % from the product and buys back ~53 minutes a year, at the cost of possible over-admission. And redundancy does not rescue you if the replicas share a failure domain: `u ≈ u_independent^k + u_common`, so the common-mode term is a floor.
- **Derive a shard count properly.** Take the max over independent constraints, then round to a number that splits cleanly. For the request log: storage `ceil(6.2 TB / 2 TB usable per node) = 4`; write throughput `800 KB/s ÷ 200 MB/s per shard = 1`; read `~1 lookup/s = 1`. Max is 4, already a power of two so a future 4 → 8 split is clean. The output of the exercise is not just "4" — it is **"storage-bound, not throughput-bound"**, which tells you that any later argument about write capacity for this store is irrelevant, and that the growth lever is retention, not hardware.
- **What is the concrete staff signal that separates a senior answer from a staff answer?** Volunteering the tradeoff, quantifying it, and stating its flip condition — "I chose a 5-second-stale placement cache over a consensus read because a stale route costs one retry (~40 ms) on a small fraction of requests while a quorum read costs ~2 ms on every request and puts the gateway in the ledger's failure domain; if a stale route caused a double-allocation instead of a retry I'd flip it" — plus spending real minutes on failure and producing a guarantees/non-guarantees table rather than stopping at the happy path.
- **MAST's headline number is a GPU demand/supply ratio going 2.63 → 0.98. Which step does it belong to, and what does it justify?** It is a step-2 (estimation) number that motivates step 6 (scale it). The pre-MAST ratio quantifies the actual bottleneck — single-region scheduling leaving one region badly oversubscribed while others had slack — and the post-MAST ratio is the evidence justifying the chosen architecture, temporal decoupling (a fast real-time path plus a slow continuously-reoptimising path), over a simpler single-speed global scheduler. It is also the model for step 8: a tradeoff with a number attached, in a document that survived peer review and shipped.

## Connections & what's next

This lesson is the synthesis point for the module. [Lesson 01](01-consistency-models.md)'s per-data-class consistency shows up in block 1 and in the three-store data model; [Lesson 02](02-consensus-and-quorums.md)'s quorum cost is the number behind rejecting a consensus read on the hot path; [Lesson 03](03-replication-and-partitioning.md)'s rendezvous hashing and skew analysis is the partition-key block; [Lesson 04](04-caching.md)'s caching supplies the freshness↔load axis; [Lesson 05](05-queueing-and-backpressure.md)'s Little's Law is the second, independent way the fleet gets sized and the reason admission control exists rather than an unbounded queue; [Lesson 06](06-failure-and-resilience.md)'s availability composition, blast-radius controls, and gray-failure detection are the entire failure block; and [Lesson 07](07-data-intensive-patterns.md)'s guarantees table is the template for the guarantees/non-guarantees table here. None of them is background — each becomes a concrete tradeoff axis the method forces you to surface.

The method is inert on paper. **[Lesson 09](09-design-rehearsal.md)** is where it becomes a reflex: timed reps against the prompts a platform engineer actually gets, with the binding constraint deliberately varied so you learn to *find* the real bottleneck instead of pattern-matching one canned shape — and with a scoring rubric, because the gap between a written design and a spoken one under a clock is exactly what the loop measures.

## References & further reading

**Primary sources**

1. **Dean, J., *Numbers Everyone Should Know* (Stanford EE380 slides)** — <https://static.googleusercontent.com/media/research.google.com/en//people/jeff/stanford-295-talk.pdf> — the latency anchors in §4. Hold them as *ratios* rather than absolutes; the values have drifted with hardware, the gaps have not. *Not fetched from this environment (egress-restricted); the table reproduces widely-published order-of-magnitude figures and is presented as such.*
2. **Choudhury, A. et al., *MAST: Global Scheduling of ML Training across Geo-Distributed Datacenters at Hyperscale*, OSDI '24** — <https://www.usenix.org/system/files/osdi24-choudhury.pdf> — the 2.63 → 0.98 anchor and the temporal-decoupling design. Read it as the calibration target for what a defended tradeoff looks like at hyperscale. *Not fetched from this environment; figures carried forward from the previous revision of this lesson.*
3. **`stas00/ml-engineering`, "Accelerator" chapter** — <https://github.com/stas00/ml-engineering/blob/master/compute/accelerator/README.md> — source of the GPU anchor table in §4: the theoretical-TFLOPS derivation (`clock × FMAs/TC/cycle × 2 × TensorCores`, giving 989 TFLOPS for the H100 SXM at 1830 MHz × 512 × 2 × 528 and 312 for the A100 SXM), and the **MAMF** measured-matmul benchmark showing the H100 SXM reaching 794.5 TFLOPS against a 989 theoretical peak (80.3 % efficiency). *Fetched and read in this environment, August 2026.* This is the basis for the "estimate at 70–80 % of peak" rule.
4. **`meta-llama/llama-models`, Llama 3.1 model card** — <https://github.com/meta-llama/llama-models> — confirms the 8B/70B/405B family uses grouped-query attention and a 128k context length, the two facts the KV-cache formula depends on. *Fetched and read.* Specific layer/head counts should be read from the model's own `config.json` rather than memorised; the worked example labels them as assumptions for that reason.
5. **Google, *Site Reliability Engineering* (the SRE Book)** — <https://sre.google/sre-book/table-of-contents/> — the vocabulary step 7 draws on: error budgets, blast radius, graceful degradation, and the cascading-failure chapter behind the degraded-mode ladder. *Not fetched from this environment (egress-restricted).*

**Real-world engineering write-ups**

6. **OpenAI, *Scaling Kubernetes to 2,500 Nodes*** — <https://openai.com/index/scaling-kubernetes-to-2500-nodes/> — the etcd write-latency bottleneck past ~500 nodes and its fix (local SSD for the data directory; Events split into their own cluster). A real estimate → bottleneck → architecture-change arc. *Not fetched from this environment; summary carried forward from the previous revision of this lesson.*
7. **OpenAI, *Scaling Kubernetes to 7,500 Nodes*** — <https://openai.com/index/scaling-kubernetes-to-7500-nodes/> — the same fleet at 3× the scale with a different binding constraint. *Not fetched from this environment.*
8. **CoreWeave, *A Slurm on Kubernetes Implementation for HPC and Large Scale AI*** — <https://www.coreweave.com/blog/sunk-slurm-on-kubernetes-implementations> — a requirements-driven hybrid architecture from a GPU-cloud operator; the model for refusing a false binary in step 1. *Not fetched from this environment.*
9. **vLLM Team, *vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention*** — <https://blog.vllm.ai/2023/06/20/vllm.html> — the production reference for paged KV-cache management, which is what makes the "KV budget ÷ KV per request" ceiling in §11 a real number rather than an approximation (paging is what removes the fragmentation waste that would otherwise dominate). *Not fetched from this environment.*
10. **Malte Ubl, *Design Docs at Google*** — <https://www.industrialempathy.com/posts/design-docs-at-google/> — how the artifact this method produces is written and reviewed: context/scope, goals **and non-goals**, proposed design, **alternatives considered**, cross-cutting concerns. *Not fetched from this environment; summary carried forward from the previous revision of this lesson.*

**Deeper dives**

11. **Kleppmann, M., *Designing Data-Intensive Applications*** — <https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/> — the theory behind the data-model and scale-out steps; Chapters 5–6 are the partition-key block, Chapter 1 the guarantees framing.
12. **The System Design Primer** — <https://github.com/donnemartin/system-design-primer> — broad reference coverage of canonical building blocks. Useful as a checklist of shapes you should recognise; it is not a substitute for the estimation discipline in §4, which is the part that actually decides designs.
