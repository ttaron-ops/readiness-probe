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
sources: 10
---

# A01.8 · A repeatable system-design method

> **Concept.** A time-boxed framework you *lead*: the back-of-the-envelope estimate chooses the architecture, and staff signal is naming — and quantifying — the tradeoff before the interviewer asks.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits
[Lessons 01–07](../README.md) built the individual concepts one at a time: consistency models, consensus and quorums, replication and partitioning, caching, queueing and backpressure, failure and resilience, and — most recently — data-intensive patterns and delivery semantics. Each of those lessons answered a narrow question well. This lesson is the synthesis: a single, repeatable method that pulls every one of those concepts into a bounded design under a clock, so that "what consistency model?" and "what happens when this fails?" and "is this pipeline effectively-once?" stop being separate lessons and become questions the method forces you to ask, in order, on any prompt you're handed.

## Why this matters
At staff level the design interview is not "can you draw boxes" — it's "can you drive an ambiguous problem to a defended architecture on a clock, connecting every choice to a stated requirement." The differentiator is estimation discipline: the numbers *choose* the design ("fits in RAM" vs "must shard"), and for GPU systems the scarce unit is GPU-seconds and HBM-GB, not QPS — so most candidates' web-shaped estimates are simply wrong.

## What's new here (calibration)
- **Skip (you already know):** "ask clarifying questions," draw a load balancer, put a cache in front, mention a CDN. These are table stakes, not signal.
- **Skip:** the vocabulary of "high-level design" and "API design" as interview-prep buzzwords — you've built real APIs and real high-level architectures; the mechanics aren't new to you.
- **New depth:** why the estimation step, not the diagram, is the actual design decision — treated here as a forcing function, with the GPU-specific units (KV-cache-GB, GPU-seconds, weight-load bandwidth) that make a web-shaped BOTE simply wrong for this domain.
- **New depth:** what this method looks like when it's not an interview exercise at all — a real, peer-reviewed staff-level design document (MAST) and a real narrative arc through estimate → bottleneck → fix → new bottleneck (OpenAI's Kubernetes scaling posts), so you can calibrate against the genuine artifact, not just the interview simulation of it.

## Core concepts
**The framework — eight steps you drive, time-boxed (~45 min):**

1. **Requirements.** Functional first (what it does), then non-functional: availability target, latency SLO (p50/p99), consistency needs, durability (RPO/RTO). Then the **scale envelope**: QPS, data size, growth rate. Explicitly state what's *out of scope* — bounding the problem is a staff move.

2. **Estimation / BOTE discipline.** Derive QPS, storage, bandwidth, memory, and machine count from first principles. Anchor on **Jeff Dean's numbers everyone should know**: L1 ≈ 0.5 ns, main memory ≈ 100 ns, SSD random read ≈ 16 µs, within-DC round trip ≈ 0.5 ms, disk seek ≈ ms, cross-region ≈ tens–hundreds of ms. The estimate is the load-bearing step: it *chooses the architecture*. "150 GB working set → fits in RAM on a few nodes" is a different system than "15 TB → must shard and hit SSD."

3. **API** — the contract. A few endpoints/signatures pin down what you're actually building and expose the read/write asymmetry.

4. **Data model** — access patterns *first*, then schema and **partition key**. The partition key is chosen from the dominant query, not the entity diagram.

5. **High-level design** — the boxes, now that the numbers justify them.

6. **Scale it** — go straight to the first bottleneck the estimate exposed. Shard / cache / replicate *deliberately*, each move traced to a number.

7. **Failure & operations** — what breaks, blast radius, backpressure, degraded mode, RPO/RTO. Where senior candidates stop and staff candidates start.

8. **Tradeoffs** — state the axes you chose **and the ones you rejected**. Staff is demonstrated in the rejected branch and its quantification.

**Driving the interview.** Manage the clock (don't sink 20 min into requirements), surface tradeoffs proactively, and connect every decision back to a stated requirement. The staff tell: **name the tradeoff before the interviewer probes, and put a number on it** ("cross-region sync replication buys RPO≈0 but adds ~80 ms to every write p99 — I'll use async + a bounded RPO instead").

**GPU divergence — the estimation step is where GPU designs split from web designs.** The scarce resource is GPU-seconds and HBM-GB, not QPS. Two math tracks dominate:
- *Bandwidth math* — NVLink / RDMA / InfiniBand for collectives, and weight-load egress (cold-loading a 70B model in bf16 ≈ 140 GB across the network is a real, sizeable transfer that gates cold-start).
- *Memory math* — **KV-cache GB per concurrent request** is the capacity limiter for serving. KV-cache bytes ≈ 2 (K and V) × layers × 2 (bf16) × hidden_dim × seq_len; per concurrent request this can be hundreds of MB to multiple GB, and it — not FLOPs — is what caps concurrency on a card. Practice BOTE in these units until they're reflexive.

**What the method looks like when a real bottleneck fights back — a second worked anchor.** Jeff Dean's latency numbers are the classic anchor for step 2, but they're static constants. It helps to also anchor the *whole* framework against a real system that hit a wall and had to change architecture because of a number, live. OpenAI's own account of scaling a single Kubernetes cluster first to [2,500 nodes](https://openai.com/index/scaling-kubernetes-to-2500-nodes/) and later to [7,500 nodes](https://openai.com/index/scaling-kubernetes-to-7500-nodes/) is exactly that: at roughly 500 nodes they hit etcd write-latency spikes even on strong hardware — the fix was moving the etcd data directory onto local SSD and splitting high-churn Kubernetes Events into a separate etcd cluster so its write load couldn't degrade the primary one. That is step 6 ("scale it — go straight to the first bottleneck the estimate exposed") happening for real, with a specific component (etcd's fsync path) as the binding constraint, exactly the way [Lesson 02](02-consensus-and-quorums.md) frames etcd as fsync-bound.

**Design docs as the real-world artifact this method produces.** The eight-step framework isn't an interview invention — it's a compressed version of what a real design document contains. Malte Ubl's account of [design docs at Google](https://www.industrialempathy.com/posts/design-docs-at-google/) describes the standard shape: a problem statement, explicit goals and *non-goals* (this lesson's step 1 "state what's out of scope"), a survey of existing solutions, the proposed design, and — critically — alternatives considered with their tradeoffs (this lesson's step 8). The document exists specifically so reviewers can evaluate the tradeoffs *before* code is written; a design doc that skips the rejected alternatives is considered incomplete by the same reviewers who'd flag a candidate for the same omission in an interview. Practicing this framework is, quite literally, practicing how to write (and defend) the artifact senior engineers write before every non-trivial project.

## Perspectives
**The estimator/quant view.** The back-of-the-envelope step is not throat-clearing before the "real" design work — it *is* the design decision. Every later step (data model, high-level design, scale-out) is bookkeeping that follows from what the estimate already determined. A candidate who draws the architecture before doing the math is, structurally, guessing.

**The interviewer/evaluator view.** From the other side of the table, staff signal looks like: clarifying scope quickly and moving on, producing a number unprompted and using it to justify a design choice, naming a rejected alternative before being asked "why not X," and spending real time on failure modes rather than treating them as an afterthought if the clock allows. A senior candidate produces a working design; a staff candidate produces a working design *and* a legible trail of why it isn't a different one.

**The real-design-doc/organizational view.** This eight-step method mirrors, almost step for step, what gets written and peer-reviewed at a real engineering organization before a system is built — goals and non-goals, a proposed design, and alternatives considered with tradeoffs, as described in Google's own design-doc culture. Practicing the interview version and writing the real artifact are the same skill at two different clock speeds.

**The GPU-unit-economics view.** QPS-first thinking is a category error for GPU systems, not just an oversimplification. A web service's cost scales with request count; a GPU serving system's cost scales with HBM-GB reserved per concurrent request and GPU-seconds consumed per request, and a fleet can be QPS-underloaded while completely full on KV-cache. Estimating in QPS on a GPU prompt produces a design sized for the wrong resource entirely — the worked example below shows the ceiling flip explicitly.

## Real-world use cases
- **Meta, "MAST: Global Scheduling of ML Training across Geo-Distributed Datacenters at Hyperscale," OSDI '24** — https://www.usenix.org/system/files/osdi24-choudhury.pdf — a real, published, staff-level design write-up in miniature: states the problem with a number (the worst-region GPU demand/supply ratio was 2.63 before MAST, 0.98 after), states the design principle (temporal decoupling — a fast-path real-time scheduler plus a slow-path continuous reoptimization loop), and ties every architectural choice back to that number. Lead with this one as what a staff-level design doc that survived peer review and shipped at hyperscale actually looks like.
- **OpenAI, "Scaling Kubernetes to 2,500 nodes"** — https://openai.com/index/scaling-kubernetes-to-2500-nodes/ — **and "Scaling Kubernetes to 7,500 nodes"** — https://openai.com/index/scaling-kubernetes-to-7500-nodes/ — a real narrative arc through estimate → bottleneck → architectural fix → new bottleneck, at a real GPU-fleet operator: etcd write-latency spikes past ~500 nodes, fixed by moving etcd onto local SSD and isolating high-churn Events traffic into its own etcd cluster; the sequel post repeats the pattern at 3x the scale with a different binding constraint (pod networking, API server load).
- **CoreWeave, "A Slurm on Kubernetes Implementation for HPC and Large Scale AI" (SUNK)** — https://www.coreweave.com/blog/sunk-slurm-on-kubernetes-implementations — a real requirements-driven design tradeoff from a named GPU-cloud employer: rather than forcing a Slurm-vs-Kubernetes choice, CoreWeave built an integration layer (SUNK) that runs Slurm's batch scheduling model on top of Kubernetes' container orchestration. "State what's out of scope, then design to the actual constraint" (this lesson's step 1 and step 6), done for real.
- **Malte Ubl (ex-Google Principal Engineer), "Design Docs at Google"** — https://www.industrialempathy.com/posts/design-docs-at-google/ — a genuine, widely-cited account of how the artifact this lesson trains you to produce is actually used, reviewed, and relied on inside a hyperscaler.

## Worked example
**Full-framework rep: "design an inference gateway/router for a multi-model GPU fleet."**

*Requirements:* route requests to the right model replica; SLO p99 TTFT < 500 ms; availability 99.9%; models range 7B–70B; out of scope: training, fine-tuning.

*Estimation (the deciding step):* Suppose 5k req/min ≈ 83 req/s, mean 400 output tokens. If a 70B replica sustains ~2,000 tok/s aggregate decode, one replica serves ~5 req/s at that length → need ~17 replicas *just for throughput*, before headroom. Now the real limiter: **KV-cache**. If one concurrent 70B request at 4k context needs ~1.2 GB of KV and an 80 GB card spends ~40 GB on weights, ~40 GB remains → ~30 concurrent requests/card *ceiling*. That number, not QPS, sizes the fleet and sets the admission-control threshold. (This is the same ceiling-flip named in the GPU-unit-economics perspective above: a QPS-only estimate would size ~17 replicas and miss the KV-cache wall entirely.)

*API:* `POST /v1/generate {model, prompt, max_tokens, stream}` → streamed tokens.

*Data model / routing:* model→replica placement table; partition/route by `model_id`; consistency for placement is eventual (a stale route costs one retry, not correctness) → cache placement aggressively, invalidate on replica churn.

*Scale it:* first bottleneck is KV-cache-bound concurrency → admission control + queue per replica with a bounded wait; overflow sheds or routes to a warm replica.

*Failure & backpressure:* on replica loss, drain its queue to siblings; when fleet KV is saturated, apply backpressure (429 + Retry-After) rather than accept and blow the TTFT SLO — degraded mode caps `max_tokens` before dropping requests.

*Tradeoffs:* chose eventual placement consistency (cheap, one-retry cost) over a consensus-backed router (strong, but adds latency and a failure domain); chose per-replica queue with shedding over unbounded buffering (protects p99 at the cost of visible 429s).

**A second anchor, at hyperscale: MAST's numbers.** The same "estimate exposes the bottleneck, architecture follows" logic scales up to a whole training fleet. Before MAST, Meta's most-contended region had a GPU demand/supply ratio of 2.63 (more than 2.5x oversubscribed relative to what a single-region scheduler could serve well); after MAST's global, geo-distributed scheduling, that ratio fell to 0.98. The architectural principle that got them there — *temporal decoupling*, a fast real-time scheduling path plus a slower continuously-reoptimizing path running in the background — is a direct answer to a step-6 "scale it" question, at the scale of an entire company's training fleet rather than one interview prompt. The number (2.63 → 0.98) is exactly the kind of "put a number on the tradeoff" move this lesson's step 8 asks a staff candidate to make.

## Practice
Produce `inference-gateway-design.md` as a portfolio write-up running all eight steps, with the KV-cache and replica-count math shown explicitly and the tradeoff section naming both chosen and rejected axes. Feeds the [staff design portfolio](../practice/staff-design-portfolio/README.md).

## Common pitfalls
1. **"Start by drawing the boxes."** The estimate should come *before* the diagram — the numbers choose which boxes are even needed. Both MAST's ratio numbers and OpenAI's latency/scale numbers precede their architecture in the real write-ups; draw the diagram after the math, not instead of it.
2. **"QPS is the universal scale unit."** For GPU serving, KV-cache-GB-per-request is usually the binding constraint before request rate is (the worked example's ~30-concurrent-requests-per-card ceiling); for training, GPU-seconds and interconnect bandwidth dominate. Sizing a GPU system on QPS alone produces the wrong architecture.
3. **"Requirements-gathering is where you spend most of the clock."** Over-indexing on requirements at the expense of estimation and the failure/ops section is exactly the senior-vs-staff gap this lesson is built to close — budget the clock so estimation and failure/ops both get real time.
4. **"Naming a tradeoff after being asked is the same as naming it unprompted."** It isn't. Staff signal is specifically volunteering the tradeoff, with a number attached, before the interviewer has to probe for it.
5. **"A clean, symmetric design is a sign of good design."** Real systems that actually ship — MAST's two-speed fast/slow scheduling split, CoreWeave's SUNK hybrid rather than a pure Slurm-or-Kubernetes choice — are often asymmetric and pragmatic precisely because they were driven by a real, specific constraint rather than aesthetic symmetry.

## Self-check
- Why is the estimation step, not the diagram, the load-bearing part of the framework? **Answer:** Because the numbers choose the architecture — working-set size decides RAM vs shard, and for GPU serving the KV-cache-per-request number sets the concurrency ceiling and thus the fleet size and admission threshold; the boxes just render a decision the math already made.
- For a GPU inference system, why is QPS the wrong primary scaling unit, and what replaces it? **Answer:** The card is limited by HBM — specifically KV-cache GB per concurrent request (plus weights) — long before it's limited by request rate; you size on GPU-seconds and KV-cache-bound concurrency, with bandwidth (NVLink/RDMA, weight-load egress) as the second axis.
- What is the concrete staff signal that separates a senior answer from a staff answer in this framework? **Answer:** Naming the tradeoff before the interviewer asks *and quantifying it* — e.g. stating that sync cross-region replication buys RPO≈0 but adds ~80 ms to write p99, then choosing async with a bounded RPO — plus a genuine failure/backpressure/degraded-mode section rather than stopping at the happy path.
- MAST's headline number is a GPU demand/supply ratio going from 2.63 to 0.98. Which step of the framework does that number belong to, and what design principle does it justify? **Answer:** It's a step-2 (estimation) number that motivates step 6 (scale it): the pre-MAST ratio quantifies the actual bottleneck (single-region scheduling leaving one region badly oversubscribed while others had slack), and the post-MAST ratio is the evidence that justifies the chosen architecture — temporal decoupling (fast real-time scheduling + slow continuous reoptimization) — over a simpler single-speed global scheduler.

## Connections & what's next
This lesson is the synthesis point for the whole module: [Lesson 01](01-consistency-models.md)'s consistency models, [Lesson 02](02-consensus-and-quorums.md)'s consensus and quorum costs, [Lesson 03](03-replication-and-partitioning.md)'s replication and partitioning, [Lesson 04](04-caching.md)'s caching, [Lesson 05](05-queueing-and-backpressure.md)'s queueing and backpressure, [Lesson 06](06-failure-and-resilience.md)'s failure and blast-radius framing, and [Lesson 07](07-data-intensive-patterns.md)'s delivery-semantics guarantees are not background knowledge here — each one becomes a concrete tradeoff axis the method forces you to surface in steps 6 through 8. The method is inert on paper, though: [Lesson 09](09-design-rehearsal.md) is where it becomes a reflex, run as timed reps against the prompts a platform engineer actually gets, with the binding constraint deliberately varied so you learn to find the *real* bottleneck instead of pattern-matching one canned shape.

## References & further reading

**Primary sources**
- Dean, J., *Numbers Everyone Should Know* (Stanford EE380 slides) — https://static.googleusercontent.com/media/research.google.com/en//people/jeff/stanford-295-talk.pdf — the canonical latency figures anchoring step 2's estimation discipline.
- Choudhury, A. et al., *MAST: Global Scheduling of ML Training across Geo-Distributed Datacenters at Hyperscale*, OSDI '24 — https://www.usenix.org/system/files/osdi24-choudhury.pdf — the full paper behind the 2.63 → 0.98 worked anchor and the temporal-decoupling design.
- Google, *Site Reliability Engineering* (the SRE Book) — https://sre.google/sre-book/table-of-contents/ — read for the failure/operations vocabulary (blast radius, degraded mode, RPO/RTO) step 7 draws on.

**Real-world engineering blogs**
- OpenAI, *Scaling Kubernetes to 2,500 Nodes* — https://openai.com/index/scaling-kubernetes-to-2500-nodes/ — the etcd write-latency bottleneck and its fix; a real estimate → bottleneck → architecture-change arc.
- OpenAI, *Scaling Kubernetes to 7,500 Nodes* — https://openai.com/index/scaling-kubernetes-to-7500-nodes/ — the same fleet, 3x the scale, a new binding constraint.
- CoreWeave, *A Slurm on Kubernetes Implementation for HPC and Large Scale AI* — https://www.coreweave.com/blog/sunk-slurm-on-kubernetes-implementations — a real requirements-driven hybrid-architecture tradeoff from a GPU-cloud operator.
- vLLM Team, *vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention* — https://blog.vllm.ai/2023/06/20/vllm.html — the production reference for how KV-cache memory management actually works, underlying this lesson's KV-cache math.

**Deeper dives**
- *The System Design Primer* — https://github.com/donnemartin/system-design-primer — broad reference coverage of classic system-design building blocks.
- Ubl, M., *Design Docs at Google* — https://www.industrialempathy.com/posts/design-docs-at-google/ — how the artifact this method produces is actually written and reviewed inside a hyperscaler.
- Kleppmann, M., *Designing Data-Intensive Applications* — https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ — the deeper theory behind the data-model and scale-out steps.
