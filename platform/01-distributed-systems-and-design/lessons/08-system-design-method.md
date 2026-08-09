---
lesson: "A01.8"
title: "A repeatable system-design method"
module: "A-01"
concept: "driven framework, BOTE in GPU units"
status: not-started
est_time: "4 hrs"
artifacts: ["inference-gateway-design.md"]
---

# A01.8 · A repeatable system-design method

> **Concept.** A time-boxed framework you *lead*: the back-of-the-envelope estimate chooses the architecture, and staff signal is naming — and quantifying — the tradeoff before the interviewer asks.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Why this matters
At staff level the design interview is not "can you draw boxes" — it's "can you drive an ambiguous problem to a defended architecture on a clock, connecting every choice to a stated requirement." The differentiator is estimation discipline: the numbers *choose* the design ("fits in RAM" vs "must shard"), and for GPU systems the scarce unit is GPU-seconds and HBM-GB, not QPS — so most candidates' web-shaped estimates are simply wrong.

## Core notes
**Skip (you already know):** "ask clarifying questions," draw a load balancer, put a cache in front, mention a CDN. These are table stakes, not signal.

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

## Worked example
**Full-framework rep: "design an inference gateway/router for a multi-model GPU fleet."**

*Requirements:* route requests to the right model replica; SLO p99 TTFT < 500 ms; availability 99.9%; models range 7B–70B; out of scope: training, fine-tuning.

*Estimation (the deciding step):* Suppose 5k req/min ≈ 83 req/s, mean 400 output tokens. If a 70B replica sustains ~2,000 tok/s aggregate decode, one replica serves ~5 req/s at that length → need ~17 replicas *just for throughput*, before headroom. Now the real limiter: **KV-cache**. If one concurrent 70B request at 4k context needs ~1.2 GB of KV and an 80 GB card spends ~40 GB on weights, ~40 GB remains → ~30 concurrent requests/card *ceiling*. That number, not QPS, sizes the fleet and sets the admission-control threshold.

*API:* `POST /v1/generate {model, prompt, max_tokens, stream}` → streamed tokens.

*Data model / routing:* model→replica placement table; partition/route by `model_id`; consistency for placement is eventual (a stale route costs one retry, not correctness) → cache placement aggressively, invalidate on replica churn.

*Scale it:* first bottleneck is KV-cache-bound concurrency → admission control + queue per replica with a bounded wait; overflow sheds or routes to a warm replica.

*Failure & backpressure:* on replica loss, drain its queue to siblings; when fleet KV is saturated, apply backpressure (429 + Retry-After) rather than accept and blow the TTFT SLO — degraded mode caps `max_tokens` before dropping requests.

*Tradeoffs:* chose eventual placement consistency (cheap, one-retry cost) over a consensus-backed router (strong, but adds latency and a failure domain); chose per-replica queue with shedding over unbounded buffering (protects p99 at the cost of visible 429s).

## Practice
Produce `inference-gateway-design.md` as a portfolio write-up running all eight steps, with the KV-cache and replica-count math shown explicitly and the tradeoff section naming both chosen and rejected axes. Feeds the [staff design portfolio](../practice/staff-design-portfolio/README.md).

## Self-check
- Why is the estimation step, not the diagram, the load-bearing part of the framework? **Answer:** Because the numbers choose the architecture — working-set size decides RAM vs shard, and for GPU serving the KV-cache-per-request number sets the concurrency ceiling and thus the fleet size and admission threshold; the boxes just render a decision the math already made.
- For a GPU inference system, why is QPS the wrong primary scaling unit, and what replaces it? **Answer:** The card is limited by HBM — specifically KV-cache GB per concurrent request (plus weights) — long before it's limited by request rate; you size on GPU-seconds and KV-cache-bound concurrency, with bandwidth (NVLink/RDMA, weight-load egress) as the second axis.
- What is the concrete staff signal that separates a senior answer from a staff answer in this framework? **Answer:** Naming the tradeoff before the interviewer asks *and quantifying it* — e.g. stating that sync cross-region replication buys RPO≈0 but adds ~80 ms to write p99, then choosing async with a bounded RPO — plus a genuine failure/backpressure/degraded-mode section rather than stopping at the happy path.

## References
- https://github.com/donnemartin/system-design-primer
- https://static.googleusercontent.com/media/research.google.com/en//people/jeff/stanford-295-talk.pdf
- https://sre.google/sre-book/table-of-contents/
- https://blog.vllm.ai/2023/06/20/vllm.html
- https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/
