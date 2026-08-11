---
lesson: "A01.5"
title: "Queueing and backpressure"
module: "A-01"
concept: "Little's Law, load shedding, metastability"
status: not-started
est_time: "5 hrs"
prev: "04-caching.md"
next: "06-failure-and-resilience.md"
artifacts: ["admission-control drop-threshold calc", "Kueue fair-share allocation model"]
sources: 10
---

# A01.5 · Queueing and backpressure

> **Concept.** A queue converts rejection into latency; at a fixed concurrency limit you cannot queue your way out of overload — you can only shed.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 04 established that a cache is a **modal system**: hit-mode and miss-mode impose completely different load on the origin, and the dangerous event is the *transition* — a flush or synchronized expiry that flips the origin from serving a trickle of misses to serving 100% of traffic it was never sized for. That's a specific, vivid instance of a much larger problem. Any system whose offered load can exceed its serving capacity has this problem, whether the trigger is a cache flush, a flash sale, a fault-restart storm, or nothing more than organic growth past a hard ceiling. Queueing theory is the general vocabulary for that larger problem: what a queue can and cannot do for you, when shedding is the only sound response, and why a system can stay overloaded long after the thing that overloaded it is gone. This lesson gives you that vocabulary — Little's Law, load shedding, metastability — precisely enough to reason about *any* queue in the fleet: the inference batcher, the admission path in front of a training data loader, the retry queue in front of etcd.

## Why this matters

Every "add a queue to absorb the spike" instinct hides a guarantee-cost-failure triple that decides whether a GPU fleet degrades gracefully or collapses into a metastable outage that survives the removal of its own trigger. The staff bound is a single number: the arrival rate at which the queue-implied wait crosses the SLO. Below it you serve; above it you must reject cheaply or you amplify. Getting this wrong is the difference between shedding 5% of traffic for 90 seconds and a 40-minute retry-storm death spiral that no amount of added capacity clears.

## What's new here (calibration)

**Skip (you already know):** FIFO decouples producer/consumer; unbounded queues are a memory bomb; "add a buffer for bursts" is the naive reflex.

The genuinely new depth this lesson adds, past that baseline:

- **Little's Law as an identity you cannot design around**, not a formula you recite — it forecloses entire classes of "just add a bigger queue" fixes before you write any code.
- **Coordinated omission** — the reason a load test that "proves" your capacity can be quietly lying to you about the exact failure mode this lesson is about, and the open-loop fix.
- **Metastability as a control-theory phenomenon** with a documented base rate in real production incidents (not a theoretical curiosity) — and why "the spike is over" doesn't mean "the system recovers."
- **Kueue's borrow/preempt mechanics** as a live, running instance of the fairness-vs-utilization-vs-starvation tension, not an abstract tradeoff.

## Core concepts

### Little's Law: an identity, not a design choice

The whole spine of this lesson is one equation: **L = λ·W**, where `L` is the average number of requests in the system (queued + being served), `λ` is the average arrival/throughput rate, and `W` is the average time a request spends in the system (queue wait + service time). Little proved this in 1961 for *any* stable queueing system, regardless of arrival distribution, service-time distribution, or scheduling discipline — it holds for a bank line, a disk I/O queue, or a GPU inference batcher equally, as long as the system is in steady state (nothing is drifting to infinity).

The reason this matters for design, not just analysis: at a **fixed concurrency limit** — a bounded worker pool, a fixed batch-slot count, a fixed number of connections — `L` is capped by construction. Rearranging the law, `λ = L / W`. If `L` is capped at `L_max`, then throughput is capped too: `λ ≤ L_max / W_service`. Push more arrivals in past that point and the identity still has to hold — but `L` cannot rise past its cap, so the *slack* has to go somewhere. It goes into `W`: queue wait grows, average time-in-system grows, and `λ` stays pinned exactly where it was. This is the formal version of Brooker's observation that **deferring load doesn't work; only shedding does** — a queue never raises your ceiling, it only decides whose latency absorbs the excess. Once offered load exceeds the concurrency ceiling, `W → ∞` roughly linearly in queue depth while `λ` stays flat at capacity. You cannot queue your way out of overload; you can only convert rejection into latency, and past some point that latency itself becomes a rejection (the client gives up or breaches its own deadline).

### Bounded queue + load shedding is the only stable overload response

Given the above, the only sound response to sustained overload is to **fail fast and cheap** at the edge, before the request touches the expensive resource (the GPU, the database write path). "Cheap" is not decorative — a rejected request must cost orders of magnitude less to produce than a served one, or the shedding path itself becomes the next overload (this is exactly the mechanism behind several of the metastable incidents cited below: the "reject" path was itself expensive enough to sustain the outage). Shed at the point where the marginal cost of saying no is lowest — typically a load balancer or admission-control layer in front of the resource, not deep inside the call graph.

### LIFO under overload

When the queue is deep, prefer serving the *freshest* requests, not the oldest. Old FIFO-queued requests have often already been abandoned by the client — its own timeout or deadline expired while it waited — so completing them produces a response nobody is waiting for: pure wasted capacity. LIFO keeps goodput high by preferentially finishing work that is still inside its deadline. This is a genuine inversion of the intuition that FIFO is "fair," and it is not universally correct — see the Shopify case below for the counter-consideration.

### Deadline / latency propagation

Stamp each request with a remaining-time budget at ingress, and drop it the instant the queue-implied wait would blow the SLO — *before* it consumes any downstream work. A request that cannot possibly meet its deadline is pure waste if you let it run; killing it early returns that capacity to a request that still can succeed. This is the mechanism that makes LIFO-under-overload principled rather than arbitrary: you're not guessing which requests are stale, you're measuring it.

### Backpressure propagation vs. naive drop

Pull-based / credit-based flow control — the consumer advertises remaining capacity, the producer blocks or slows until credit is available — is strictly better than blind tail-drop at a full buffer, because it pushes the shed decision to the *cheapest* point in the pipeline and preserves ordering and fairness upstream. Naive drop at a full buffer sheds whatever happens to arrive last, indiscriminately and late (the buffer is already full, meaning latency has already been paid before the drop happens). Backpressure prevents the buffer from filling in the first place.

### Coordinated omission — why your load test lies to you

This is the load-generator/measurement failure mode that makes the rest of this lesson dangerous to skip: a **closed-loop** load generator — one that waits for each response before issuing the next request for that "virtual user" — *stops sending during a stall*. If the system under test hangs for 2 seconds, the generator simply doesn't send anything for those 2 seconds; the requests that *would* have arrived during the stall, and that would have measured the worst of it, never get sent and never get measured. The result: your p99 latency number looks fine, sometimes excellent, while a system in the same state in production — where real users keep arriving on their own schedule regardless of how the server is doing — is on fire. Gil Tene coined the term; the fix is **open-loop load generation** at a fixed target rate (send at rate `λ_target` regardless of response timing) or, if closed-loop is unavoidable, explicitly correcting the recorded latencies for the omitted samples. HdrHistogram ships a *corrected-for-coordinated-omission* recording mode specifically for this — if your load-testing tool doesn't advertise open-loop generation or CoCo correction, treat every percentile it reports above p50 with real suspicion.

### Metastable failure

A system is **metastable** when it stays overloaded *after the original trigger is removed*, because a **sustaining feedback loop** — retries, a permanently-full queue, a cache-miss storm re-triggering itself — keeps offered load above capacity on its own, independent of the trigger. The queue (or the retry logic, or the cache) is the *amplifier*: it holds the elevated load in place even once the spike that caused it is long gone. This is why "wait for the traffic to die down" is not a recovery strategy for a metastable incident — there is no external traffic sustaining it anymore; the system is sustaining itself. Escaping requires actively **draining**: shed hard, kill in-flight retries, possibly restart components to clear the amplifying state, *then* let load back in slowly. Section "Real-world use cases" below has the production evidence that this is common, not exotic.

**Retry budgets, made concrete.** The practical guard against a retry-fed metastable loop is a **retry budget**: cap total retries, fleet-wide, at a small fixed fraction of base request volume — a common number is **~10%** — enforced with a token bucket that refills at that rate. Concretely: if base traffic is 10,000 req/s, the retry budget allows at most ~1,000 retries/s regardless of how many individual clients think they need to retry; once the bucket is empty, further retries fail immediately instead of queueing. This is the mechanism that would have (and, per Slack's own writeup below, subsequently did) bound a retry storm even when every individual client's backoff-with-jitter implementation is textbook-correct — the budget caps the *aggregate*, which no amount of well-behaved per-client backoff can do on its own.

### Fair-share scheduling as a live instance

Everything above is about *time* (queue wait). The same shed/hold/starve tension shows up in *capacity* allocation, and Kueue's fair-sharing model is a clean, running example — worked through in detail below.

## Perspectives

**The queueing-theory view.** Little's Law is not a design lever; it's an identity that always holds for a stable system, which means it *forbids* certain moves rather than suggesting them. You cannot raise `λ_max` by adding queue depth at a fixed `L`; the only two levers that actually move `λ_max` are raising `L` (more concurrency — more replicas, more GPU slots) or lowering `W_service` (faster per-request work). A "bigger buffer" changes neither, so it cannot be the fix, no matter how it's marketed internally.

**The load-generator / measurement view.** You cannot verify any of the above with a naive benchmark. Coordinated omission means a closed-loop load test systematically under-reports tail latency exactly during the failure window this lesson is about — the worse your system's stall, the more that stall is hidden from the report. A capacity number derived from a closed-loop test is not conservative; it is actively misleading in the direction that hurts you.

**The feedback-systems / control-theory view.** Metastability is not a distributed-systems-specific mystery; it's the systems-theory pattern of a **sustaining loop** overpowering a **trigger**. The trigger (a spike, a flush) pushes the system past a threshold; past that threshold, a loop internal to the system (retries, cache misses, queue backlog) keeps regenerating the overload condition on its own. Removing the trigger doesn't break the loop — you have to intervene on the loop directly (drain, shed, restart) to bring the system back under the threshold, at which point it becomes stable again on its own.

**The fairness/scheduling view.** Kueue's borrow-and-preempt behavior is capacity-allocation backpressure: an idle tenant's quota is "credit" a busy tenant can borrow, and the return of that credit (preemption) is the shed decision, applied to jobs instead of requests. The same three-way tension — fairness (give B its share), utilization (let A keep borrowing while B is idle), starvation (never let B wait unboundedly) — is structurally the same tension between LIFO throughput-maximization and FIFO fairness in the request-queue case above, just at a different timescale and unit of work.

## Real-world use cases

- **Slack, "Slack's Incident on 2-22-22"** — <https://slack.engineering/slacks-incident-on-2-22-22/> — A load spike on a Vitess (sharded MySQL) keyspace, driven by an inefficient query on group-DM data, triggered a retry storm that sustained the overload. The notable detail: Slack's clients were already using exponential backoff with jitter — the textbook-correct retry pattern — and it still wasn't sufficient to prevent sustained overload at fleet scale, because per-client-correct backoff says nothing about the *aggregate* retry rate across the whole fleet. Recovery matched "shed, don't defer": throttle new connections at the edge (the client boot throttle) and fix the load-amplification source (the query pattern hitting every shard on every miss) before letting traffic back in. Lead with this one — it's the clearest real "even correct retry design isn't sufficient on its own" case available.
- **Shopify, "Surviving Flashes of High-Write Traffic Using Scriptable Load Balancers" (Parts I & II)** — <https://shopify.engineering/surviving-flashes-of-high-write-traffic-using-scriptable-load-balancers-part-i> and <https://shopify.engineering/surviving-flashes-of-high-write-traffic-using-scriptable-load-balancers-part-ii> — Flash-sale admission control implemented at the edge with scriptable Nginx/Lua load balancers, throttling write traffic before it reaches the backend. Shopify's own account describes correcting an early version of this system where fairness for customers already queued wasn't preserved. That's a useful complication to this lesson's LIFO-under-overload point: LIFO maximizes throughput for a machine-to-machine queue with hard deadlines, but in a **human-facing** queue (a customer who has been waiting in a checkout line), serving the newest arrivals first instead of the person who arrived first can be experienced as actively unfair, even though it's throughput-optimal. The "prefer freshest work" rule from Core concepts is a default for deadline-bound machine work, not a universal law — check which kind of queue you're actually shedding from.
- **Huang, L. et al., "Metastable Failures in the Wild" (OSDI '22)** — <https://www.usenix.org/conference/osdi22/presentation/huang-lexiang> — This is the paper behind the "metastable failure" term used above, already the source of Kueue-adjacent framing in this module; worth pulling one concrete number into the body rather than leaving it as an abstract citation. The authors' analysis of public incident reports found that **at least 4 of the last 15 major AWS outages** followed the metastable-failure pattern, and their in-depth study covers **22 metastable incidents across 11 different organizations**. This is real evidence the pattern is a recurring, cross-company production reality, not a theoretical curiosity confined to one team's postmortem.

## Worked example

**Inference pool.** 8 replicas × max batch-concurrency 32 → `L_max = 256` in-flight token-steps. Mean service time `W_service = 40 ms/token-step`. Max sustainable throughput: `λ_max = L_max / W_service = 256 / 0.040 s = 6,400 steps/s`. At λ_max the pool is saturated; any additional arrivals sit in queue. For a **500 ms TTFT SLO** with 40 ms/step service, the queue budget is `500 − 40 = 460 ms` of wait → at saturation a request can tolerate `460 / 40 ≈ 11` requests ahead of it *per slot*, i.e. a global queue depth ≈ `11 × 256 ≈ 2,800`. **Admission-control drop threshold:** reject when `current_queue_depth / λ_max > 0.46 s`, i.e. queue depth > ~2,900; practically set it lower (~2,000) to hold margin against variance and to keep p99, not mean, inside 500 ms. Above threshold: reject at the edge with a cheap 429, LIFO the survivors.

**Kueue fair share.** Two tenants, 60/40 target share, 100 GPUs. Steady state with both saturated: A = 60, B = 40 GPUs. Kueue orders admission by *usage-weighted* fair sharing, so if B is idle, A **borrows** B's unused quota up to the cohort limit and runs at 100. When B then bursts, Kueue sees A over its fair share and B under; new B jobs are admitted and A's borrowed jobs become **preemption candidates** until allocation converges back toward 60/40. The staff tension: preempting A's borrowed job wastes its partial progress (checkpoint-bound), so the real knob is *how far over fair share* borrowing is allowed before preemption fires — utilization (let A keep borrowing) vs fairness (reclaim for B fast) vs starvation (never let B wait unboundedly).

## Practice

Feeds [staff design portfolio](../practice/staff-design-portfolio/README.md). Model an inference tier: given target QPS, batch concurrency, and per-token service time, derive `λ_max` via Little's Law, compute the queue depth at which p99 TTFT breaches SLO, and specify the admission-control drop threshold plus the shed mechanism (edge 429 + LIFO). Then model a two-tenant Kueue cohort: compute steady-state allocation, the borrow behavior when one tenant is idle, and the preemption trigger when it bursts. Write the one-page decision: fair-share vs utilization vs starvation, with the deciding number for each. As a stretch goal, specify the retry budget (token-bucket rate as a fraction of base traffic) you'd put in front of the tier, and state which of the "Common pitfalls" below it's meant to close.

## Common pitfalls

1. **"Add a bigger queue to absorb the burst."** At a fixed concurrency limit, queue depth only converts rejection into latency (Little's Law) — it never raises max throughput. The fix is more concurrency or less service time, not more buffer.
2. **"Exponential backoff with jitter is sufficient to prevent retry storms."** Slack's incident shows correct client-side backoff can still sustain fleet-wide overload — per-client-correct behavior doesn't bound the aggregate. You need server-side retry budgets / token-bucket caps and edge-level shedding too, not just well-behaved clients.
3. **"My load test proves we can handle X QPS."** Closed-loop load generators suffer coordinated omission — they stop sending during a stall, hiding exactly the tail behavior that matters. Use open-loop generation, or correct for the omitted samples.
4. **"Once the traffic spike passes, the system will recover on its own."** Metastable failures persist because a sustaining loop (retries, a full queue, cache-miss storms) keeps offered load above capacity independent of the original trigger. Recovery requires active draining, not patience.
5. **"FIFO is always the fair choice."** Under deep-queue overload, FIFO serves requests whose deadline has likely already passed client-side — wasted work. LIFO maximizes goodput for deadline-bound machine work — but per Shopify's account, in a human-facing fairness context (a customer who arrived first), LIFO can feel unfair even though it's throughput-optimal for a machine-to-machine deadline queue. Know which kind of queue you're shedding from before applying the rule.

## Self-check

- At a fixed concurrency limit, what happens to throughput when you increase the queue size? **Answer:** Nothing — throughput stays pinned at `L / W_service`; only latency (queue wait) grows. Little's Law: extra `L` you can't serve just inflates `W`. You've converted rejection into latency, not added capacity.
- Why prefer LIFO over FIFO once the queue is deep under overload, and when is that the wrong call? **Answer:** Old FIFO-queued requests have likely already breached their deadline and been abandoned by the client, so serving them produces responses nobody awaits; LIFO serves the freshest requests still inside their deadline, maximizing goodput. It's the wrong call for a human-facing, first-come-first-served context (Shopify's flash-sale queue) where serving newer arrivals ahead of people who've been waiting reads as unfair even though it's throughput-optimal.
- What makes a failure *metastable* rather than just an overload spike? **Answer:** A sustaining feedback loop (retries, a full queue, cache-miss storms) keeps offered load above capacity even after the original trigger is gone. The queue amplifies and holds the load in place, so recovery requires active draining/shedding, not just removing the spike.
- Why can a load test report a clean p99 while the same system falls over in production under similar load? **Answer:** Coordinated omission — a closed-loop generator stops issuing requests during a stall, so exactly the requests that would have measured the worst latency never get sent or recorded. The fix is open-loop load generation at a fixed rate, or correcting recorded latencies for the omitted samples (e.g. HdrHistogram's corrected-for-coordinated-omission mode).

## Connections & what's next

This lesson generalizes lesson 04's cache-flush miss-storm into the broader theory of overload: the flush was one trigger, queueing/backpressure/metastability is the mechanism that determines whether *any* trigger turns into a graceful shed or an unrecoverable spiral. It also sets up two lessons ahead: lesson 07 will show how retry storms — the sustaining loop discussed here — routinely produce duplicate-delivery bugs downstream, so the shedding discipline here is also a delivery-semantics precondition. Next: **[06 · Failure and resilience](06-failure-and-resilience.md)** — queueing and shedding both assume you can *tell* the system is overloaded; failure detection is the layer underneath that assumption, and it turns out to be provably imperfect.

## References & further reading

**Primary sources**
- **Little, J.D.C. (1961), "A Proof for the Queuing Formula: L = λW,"** *Operations Research* 9(3):383–387, DOI [10.1287/opre.9.3.383](https://doi.org/10.1287/opre.9.3.383) — the original proof behind the identity this whole lesson is built on; read for the exact stationarity conditions under which it holds.
- **Huang, L. et al. (2022), "Metastable Failures in the Wild,"** OSDI '22 — <https://www.usenix.org/conference/osdi22/presentation/huang-lexiang> — read for the formal definition of metastability and the incident-survey methodology behind the "4 of 15 AWS outages" figure.
- **Kueue, "Fair Sharing"** — <https://kueue.sigs.k8s.io/docs/concepts/fair_sharing/> — read for the exact borrow/preemption mechanics behind the worked example.

**Real-world engineering blogs**
- **Slack, "Slack's Incident on 2-22-22"** — <https://slack.engineering/slacks-incident-on-2-22-22/> — what it shows: correct client-side backoff-with-jitter still contributing to sustained overload at fleet scale, and the shed-first recovery playbook.
- **Shopify, "Surviving Flashes of High-Write Traffic Using Scriptable Load Balancers" (Parts I & II)** — <https://shopify.engineering/surviving-flashes-of-high-write-traffic-using-scriptable-load-balancers-part-i> · <https://shopify.engineering/surviving-flashes-of-high-write-traffic-using-scriptable-load-balancers-part-ii> — what it shows: edge admission control for flash-sale traffic, and a real correction to a fairness-for-already-queued-customers bug.

**Deeper dives**
- **Brooker, M., "Metastability and Distributed Systems"** — <https://brooker.co.za/blog/2021/05/24/metastable.html> — a decade of hyperscale-operations experience distilled into the general theory of the sustaining-loop pattern.
- **Brooker, M., "What is Backoff For?"** — <https://brooker.co.za/blog/2022/08/11/backoff.html> — what backoff and jitter do and don't do; useful complement to the Slack case above.
- **Google SRE Book, "Handling Overload"** — <https://sre.google/sre-book/handling-overload/> — the load-shedding and degraded-response playbook this lesson's "shed cheap, shed early" advice is drawn from.
- **Encore, "Handling growth: an introduction to queueing"** — <https://encore.dev/blog/queueing> — an accessible, example-driven walkthrough of the queueing-theory ideas in Core concepts.
