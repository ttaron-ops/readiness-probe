---
lesson: "A01.5"
title: "Queueing and backpressure"
module: "A-01"
concept: "Little's Law, load shedding, metastability"
status: not-started
est_time: "3 hrs"
artifacts: ["admission-control drop-threshold calc", "Kueue fair-share allocation model"]
---

# A01.5 · Queueing and backpressure

> **Concept.** A queue converts rejection into latency; at a fixed concurrency limit you cannot queue your way out of overload — you can only shed.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Why this matters
Every "add a queue to absorb the spike" instinct hides a guarantee-cost-failure triple that decides whether a GPU fleet degrades gracefully or collapses into a metastable outage that survives the removal of its own trigger. The staff bound is a single number: the arrival rate at which the queue-implied wait crosses the SLO. Below it you serve; above it you must reject cheaply or you amplify. Getting this wrong is the difference between shedding 5% of traffic for 90 seconds and a 40-minute retry-storm death spiral that no amount of added capacity clears.

## Core notes
**Skip (you already know):** FIFO decouples producer/consumer; unbounded queues are a memory bomb; "add a buffer for bursts" is the naive reflex.

**Little's Law is the whole spine: L = λ·W.** With `L` = concurrent in-flight requests, `λ` = throughput, `W` = latency. The staff corollary: at a *fixed concurrency limit* `L` (a bounded worker pool, a fixed batch slot count), λ and W trade off **directly and inversely** — pushing more arrivals in cannot raise λ past `L / W_service`; it only inflates `W` via queue wait. You never queue your way out of overload; you convert rejection into latency (Brooker: deferring load doesn't work, only shedding does). Once offered load exceeds the concurrency ceiling, `W → ∞` linearly in queue depth while λ stays pinned at capacity.

**Bounded queue + load shedding is the only stable overload response.** Fail fast and *cheap* at the edge — a rejected request must cost orders of magnitude less than a served one, or the shedding path itself becomes the overload. Shed *before* the request touches the expensive resource (the GPU), not after.

**LIFO under overload.** When the queue is deep, serve the *freshest* requests: old queued work has often already been abandoned by the client (its deadline expired), so FIFO spends capacity producing responses nobody is waiting for. LIFO keeps goodput high by preferentially completing work still inside its deadline.

**Deadline / latency propagation.** Stamp each request with a remaining-time budget and drop it the moment queue-implied wait would blow the SLO — *before* it consumes downstream work. A request that cannot meet its deadline is pure waste; killing it early is capacity returned to requests that can still succeed.

**Backpressure propagation vs naive drop.** Pull-based / credit-based flow control (consumer advertises capacity, producer blocks) is strictly better than blind tail-drop because it pushes the shed decision to the *cheapest* point and preserves ordering/fairness. Naive drop at a full buffer sheds randomly and late.

**Coordinated omission** — the reason your load-test latency numbers lie. A closed-loop load generator that waits for each response before sending the next *stops sending during a stall*, so the requests that would have hit the slow window never get measured. Your p99 looks fine while production is on fire. Fix: open-loop generation at a fixed rate, or correct for the omitted samples.

**Metastable failure.** A system that stays overloaded *after* the trigger is removed because a sustaining feedback loop — retries, a full queue, cache-miss storms — keeps offered load above capacity on its own. The queue is the amplifier: it holds the elevated load in place. Escape requires actively *draining* (shed hard, drop retries) not merely removing the original spike.

## Worked example
**Inference pool.** 8 replicas × max batch-concurrency 32 → `L_max = 256` in-flight token-steps. Mean service time `W_service = 40 ms/token-step`. Max sustainable throughput: `λ_max = L_max / W_service = 256 / 0.040 s = 6,400 steps/s`. At λ_max the pool is saturated; any additional arrivals sit in queue. For a **500 ms TTFT SLO** with 40 ms/step service, the queue budget is `500 − 40 = 460 ms` of wait → at saturation a request can tolerate `460 / 40 ≈ 11` requests ahead of it *per slot*, i.e. a global queue depth ≈ `11 × 256 ≈ 2,800`. **Admission-control drop threshold:** reject when `current_queue_depth / λ_max > 0.46 s`, i.e. queue depth > ~2,900; practically set it lower (~2,000) to hold margin against variance and to keep p99, not mean, inside 500 ms. Above threshold: reject at the edge with a cheap 429, LIFO the survivors.

**Kueue fair share.** Two tenants, 60/40 target share, 100 GPUs. Steady state with both saturated: A = 60, B = 40 GPUs. Kueue orders admission by *usage-weighted* fair sharing, so if B is idle, A **borrows** B's unused quota up to the cohort limit and runs at 100. When B then bursts, Kueue sees A over its fair share and B under; new B jobs are admitted and A's borrowed jobs become **preemption candidates** until allocation converges back toward 60/40. The staff tension: preempting A's borrowed job wastes its partial progress (checkpoint-bound), so the real knob is *how far over fair share* borrowing is allowed before preemption fires — utilization (let A keep borrowing) vs fairness (reclaim for B fast) vs starvation (never let B wait unboundedly).

## Practice
Feeds [staff design portfolio](../practice/staff-design-portfolio/README.md). Model an inference tier: given target QPS, batch concurrency, and per-token service time, derive `λ_max` via Little's Law, compute the queue depth at which p99 TTFT breaches SLO, and specify the admission-control drop threshold plus the shed mechanism (edge 429 + LIFO). Then model a two-tenant Kueue cohort: compute steady-state allocation, the borrow behavior when one tenant is idle, and the preemption trigger when it bursts. Write the one-page decision: fair-share vs utilization vs starvation, with the deciding number for each.

## Self-check
- At a fixed concurrency limit, what happens to throughput when you increase the queue size? **Answer:** Nothing — throughput stays pinned at `L / W_service`; only latency (queue wait) grows. Little's Law: extra `L` you can't serve just inflates `W`. You've converted rejection into latency, not added capacity.
- Why prefer LIFO over FIFO once the queue is deep under overload? **Answer:** Old FIFO-queued requests have likely already breached their deadline and been abandoned by the client, so serving them produces responses nobody awaits. LIFO serves the freshest requests still inside their deadline, maximizing goodput.
- What makes a failure *metastable* rather than just an overload spike? **Answer:** A sustaining feedback loop (retries, a full queue, cache-miss storms) keeps offered load above capacity even after the original trigger is gone. The queue amplifies and holds the load in place, so recovery requires active draining/shedding, not just removing the spike.

## References
- https://sre.google/sre-book/handling-overload/
- https://brooker.co.za/blog/2022/08/11/backoff.html
- https://kueue.sigs.k8s.io/docs/concepts/fair_sharing/
- https://www.usenix.org/conference/osdi22/presentation/huang-lexiang
- https://encore.dev/blog/queueing
