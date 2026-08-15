---
lesson: "A01.6"
title: "Failure and resilience"
module: "A-01"
concept: "Imperfect detectors, gray failure, checkpoint math"
status: not-started
est_time: "5 hrs"
prev: "05-queueing-and-backpressure.md"
next: "07-data-intensive-patterns.md"
artifacts: ["job-level MTBF + optimal checkpoint interval calc", "client-vantage health-check design"]
sources: 10
---

# A01.6 · Failure and resilience

> **Concept.** Failure detectors are fundamentally imperfect and health checks lie; on a GPU fleet the deciding failures are gray (SDC) and correlated (gang-scheduled), so resilience is checkpoint math and client-vantage probing, not replica redundancy.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 05 gave you the theory of overload: shed cheap and early, don't defer, and watch for the sustaining feedback loop that turns a spike into a metastable outage. All of that assumed one thing you were never asked to question — that you can *tell* the system is overloaded, or a node is dead, or a request needs to be shed. That assumption is the subject of this lesson, and it turns out to be provably imperfect. Deciding *when* to shed a request, retry a call, or fail a node over all reduce to the same underlying question: is the thing on the other end slow, or is it dead? This lesson is the layer underneath everything in lesson 05 — the failure-detection substrate that every backpressure, retry, and shedding decision is built on top of, and the reason none of those decisions can ever be made with certainty.

## Why this matters

The redundancy math every senior engineer carries — "two replicas, MTBF squared" — silently assumes independent, fail-stop failures observable by a health check. On a large training run all three assumptions break: failures are correlated (one NIC kills a 1,024-GPU gang), they are gray (a GPU passes `nvidia-smi` while corrupting gradients), and the detector cannot tell slow from dead. The staff bound here is the checkpoint interval — a single number from `√(2·δ·MTBF)` — that sets how much compute you burn to survive a fleet whose real MTBF is days, not years.

## What's new here (calibration)

**Skip (you already know):** retries and timeouts exist; circuit breakers are a pattern; redundancy raises availability.

The genuinely new depth this lesson adds, past that baseline:

- **Why timeouts can never be perfect** — not an engineering gap you can close with a smarter algorithm, but a formal impossibility result (FLP) that tells you exactly what tradeoff you're stuck making.
- **Differential observability / gray failure** as the organizing idea for GPU-fleet resilience — the specific reason "the health check is green" stops meaning "the node is fine" at scale.
- **Why GPUs specifically produce silent corruption rather than clean crashes** — the hardware/silicon mechanisms (marginal HBM cells, cosmic-ray bit flips, thermal-throttling miscounts), not just the abstract "gray failure" label.
- **Checkpoint economics as a pure cost calculus** (Young/Daly) — wasted compute from checkpointing is a real, budgetable percentage, not a vague "do it more often to be safe."

## Core concepts

### Failure detectors are fundamentally imperfect (FLP)

Start from the hard limit, because everything else in this lesson is a response to it. In 1985, Fischer, Lynch, and Paterson proved that in a fully **asynchronous** system — one with no upper bound on message delay — no deterministic consensus protocol can guarantee termination if even one process can fail, because no process can ever distinguish "that process is slow" from "that process is dead" with certainty. A message that hasn't arrived yet might arrive next millisecond, or never. This is not an engineering shortfall waiting for a cleverer timeout algorithm; it is a **provable impossibility** under the stated assumptions. Every real system works around it by trading away one of those assumptions (adding a partial-synchrony assumption, a failure-detector oracle, or randomization) — but the underlying tension never disappears, it just gets pushed into a parameter you have to choose.

Concretely, that parameter is the timeout, and every choice of it trades a **false positive** (you declare a merely-slow node dead, kill it or fail it over, and pay the cost of that mistake) against **detection latency** (you wait longer to be sure, and a genuinely dead node keeps looking "maybe alive" for longer while the system suffers). There is no timeout value that eliminates both costs simultaneously; you are always choosing a point on that curve.

**Phi-accrual detectors**, in one line: instead of a hard boolean "alive/dead" cutoff, track the observed distribution of heartbeat inter-arrival times and compute a continuous suspicion level, `φ = −log10(P(heartbeat this late))` — the longer a heartbeat is overdue relative to its historical distribution, the higher `φ` climbs. This lets you set an explicit, tunable false-positive rate (e.g. "act once φ implies less than a 1-in-10,000 chance this is still a healthy node") instead of guessing a fixed millisecond constant. It's a better *parameterization* of the FLP tradeoff — it does not remove the tradeoff itself.

### Timeouts, retries, jitter — and their limits

Static timeouts fossilize a latency assumption that drifts as the system changes; adaptive timeouts that track the moving p99 age better. Exponential backoff **with full jitter** is the baseline — naked exponential backoff without jitter synchronizes clients into retry waves that hit the recovering system in lockstep; jitter spreads the retries out in time. The staff-level upgrade, as lesson 05 covered in depth, is a **retry budget / token-bucket** cap on total retries (commonly ~10% of base request volume) so that even a partial outage cannot multiply itself into a full retry storm through perfectly well-behaved individual clients.

Brooker's skeptical take on **circuit breakers** is worth internalizing precisely because it cuts against a widely-held "of course you should have one" assumption: a circuit breaker adds a *mode* (open / closed / half-open), and that mode itself has to be correctly triggered and tested, which means it can misfire — tripping open on a healthy subset of traffic during a partial outage and turning a 20%-degraded system into a 100%-down one for the clients behind that breaker. Continuous load-shedding plus retry budgets degrade smoothly as conditions worsen; a circuit breaker's binary open/closed state does not. This doesn't mean never use one — it means treat "add a circuit breaker" as a design decision with its own failure modes, not a hygiene checkbox.

### Gray failure and differential observability

This is the single most important resilience concept for a GPU fleet, and it comes from Huang et al.'s HotOS '17 paper on cloud-scale outages. A **gray failure** is a fault where the system's *own* health check reports healthy while clients actually suffer: a NIC silently dropping 2% of packets, a disk that's degraded but still spinning and still answering probes, a GPU with a marginal HBM cell that returns wrong bits under specific access patterns. The core idea is **differential observability**: the server's vantage point says "I'm fine," and the client's vantage point says "this is broken," and the gap between those two views is exactly where outages hide, because standard monitoring is built almost entirely from the server's vantage point. The fix is structural, not a smarter check: observe from the **client's** vantage, and probe with **active compute**, not a status read. A status register can be green while the thing it's supposed to represent is wrong.

### Why GPUs specifically produce silent corruption, not clean crashes

Gray failure is a general cloud-systems idea; on a GPU fleet it has a specific, physical cause worth naming rather than treating as an abstraction. Three concrete mechanisms:

- **Marginal HBM cells.** At the density and speed modern HBM stacks run at, individual memory cells can be right at the edge of reliably holding a value — under particular voltage, temperature, or access-pattern conditions they flip a bit without triggering any error-correction alarm, silently corrupting the tensor stored there.
- **Cosmic-ray-induced bit flips.** High-energy particles from cosmic radiation can flip a bit in memory or in a compute unit's logic; at the scale of a fleet with tens of thousands of GPUs running continuously, the aggregate rate of these single-event upsets becomes a real, non-negligible source of silent corruption rather than a lab curiosity.
- **Thermal throttling miscounts and marginal silicon.** A chip running close to its thermal or power limits can produce a subtly wrong result — a miscounted cycle, a corrupted intermediate value — without crashing, especially on marginal units that passed factory testing but drifted closer to their limits under sustained load.

None of these produce a crash, an exception, or a failed health check. They produce a **wrong number**, silently propagated into a gradient or an activation, which is exactly what "silent data corruption" (SDC) means, and exactly why it needs a fundamentally different detection strategy than "does the process respond."

### Cascading & metastable failure, blast radius, and correlated failure

Retry storms feed the death spiral described in lesson 05; containment on the resilience side is architectural — **cells** and **shuffle sharding** bound how many customers or jobs any single failure can reach, so a gray-failing component takes down a bounded slice instead of the whole fleet. And correlated failure is exactly what breaks the naive redundancy calculation: two "independent" replicas sharing a rack, a PDU, a top-of-rack switch, or a NIC firmware version are not statistically independent, and their joint failure probability sits far above the product of their individual failure probabilities. The MTBF-squared intuition assumes independence that correlated infrastructure quietly removes.

### Checkpoint economics (Young/Daly) as a cost-minimization problem

Given that failures on a large fleet are frequent, correlated, and partly undetectable in real time, the practical resilience lever for a long training job is not "prevent failure" but **bound the cost of failure** — and that is a pure optimization problem. Young (1974) derived the first-order approximation for the checkpoint interval that minimizes total wasted compute: `interval ≈ √(2·δ·MTBF)`, where `δ` is the checkpoint write cost and `MTBF` is the mean time between failures for the job as a whole (Daly's later work refines this for more general failure/overhead models, but the Young form is the one worth carrying in your head). The intuition: checkpoint too rarely and you lose a lot of work on each failure (rework cost dominates); checkpoint too often and you spend too much time writing checkpoints that mostly never get used (overhead dominates). The square root falls out of minimizing the sum of those two costs — worked with real numbers below.

**Elasticity is the second lever, and it's a different mechanism.** Checkpoint interval reduces the *rework* after a failure. A separate lever reduces the *blast radius* of a failure in the first place: **elastic training**, where a localized GPU or node failure drops those units out of the job and the remaining units continue at reduced scale rather than the whole job halting. Google's account of Gemini-scale training (below) describes continuing at roughly **97% of prior throughput** with fewer chips after a localized failure, rather than a full-job restart. Checkpoint interval and elasticity are complementary, not substitutes: checkpointing bounds how much you lose *when* something fails; elasticity bounds how much of the job is affected *by* the failure.

## Perspectives

**The theory / impossibility view.** FLP is not an engineering constraint you can eventually satisfy with a good enough implementation — it's a mathematical fact about asynchronous systems with even one faulty process. Every timeout, circuit breaker, and health check in this lesson is a negotiated compromise with that fact, not a solution to it. Internalizing this changes the question from "how do we build a perfect detector" to "what false-positive rate can we afford, and what detection latency can we tolerate" — which is the actual, answerable engineering question.

**The observability / gray-failure view.** Differential observability reframes almost every hard-to-diagnose production incident as a gap between what the server thinks is true and what the client experiences. The organizing move for any resilience design on this lesson's topic is to ask: where is my monitoring measuring from the server's vantage when I need the client's?

**The hardware / silicon view.** Silent data corruption on GPUs is not a software bug waiting to be patched; it traces to physical phenomena at the edge of what current silicon and memory density can reliably guarantee — marginal cells, cosmic-ray events, thermal-limit drift. That means detection has to be active and numerical (does this computation produce the right answer) rather than passive and status-based (is the process still running), because the hardware itself is not going to tell you it's wrong.

**The economics / checkpoint view.** Young/Daly turns "how resilient should this job be" into a pure cost-minimization calculus with a closed-form answer, not a judgment call. Wasted compute is a real, budgetable percentage of the total training run — the worked example below puts it at roughly 10%, which on a large GPU fleet is a line item worth optimizing (via faster checkpointing or elasticity), not an accepted cost of doing business.

## Real-world use cases

- **Google, "Training in Turmoil: Silent Data Corruption in Systems at Scale"** — <https://research.google/pubs/training-in-turmoil-silent-data-corruption-in-systems-at-scale/> — Google's own account of SDC at Gemini training scale: SDC events observed roughly **every one to two weeks** at ~10,000-chip scale, detected via **deterministic-replay** training (rerun the suspect step deterministically and compare) plus **proactive SDC scanners** run on otherwise-idle machines. Also describes **elastic** recovery continuing at roughly 97% of prior throughput on fewer chips after a localized failure, rather than a full restart. A second hyperscaler's independent confirmation of the pattern, with a concrete detection-and-mitigation architecture. Lead with this one alongside the Llama 3 paper below.
- **Meta, "The Llama 3 Herd of Models"** — <https://arxiv.org/abs/2407.21783> — the primary source for the SDC statistics this lesson cites: over a 54-day, ~16K-H100 training run, Meta recorded **466 total job interruptions** (419 unexpected, 47 planned), sustained **greater than 90% effective training time**, and attributed roughly **78% of unexpected interruptions** to confirmed or suspected hardware issues — GPU-related issues alone accounted for the majority of those. Six of those interruptions were specifically attributed to silent data corruption, consistent with the ~1.4% figure cited in Core concepts. Cite the paper itself for these numbers, not a secondary summary.
- **Roblox, "Roblox Return to Service 10/28–10/31 2021"** — <https://blog.roblox.com/2022/01/roblox-return-to-service-10-28-10-31-2021/> — a 73-hour outage whose root cause was buried deep inside Consul's use of BoltDB (a pathological performance interaction under specific load conditions), and whose diagnosis was made dramatically harder because the monitoring systems that would normally have shown the problem themselves depended on the very system that was failing. A textbook gray-failure diagnosis story: the fault wasn't visible from the vantage points the team could actually observe from.
- **PagerDuty, "August 28 Kafka Outages – What Happened and How We're Improving"** — <https://www.pagerduty.com/eng/august-28-kafka-outages-what-happened-and-how-were-improving/> — a bug that created a new Kafka producer per API request flooded the broker fleet and destabilized it; observability gaps delayed pinpointing the root cause during the first incident, and the recovery process itself produced **duplicate webhooks** for some customers as the backlog of stale messages drained. A good bridge into lesson 07: a cascading failure's recovery path is exactly where delivery-semantics bugs (duplicates, at-least-once leaking into "exactly-once" assumptions) tend to surface.

## Worked example

**Job-level MTBF.** 1,024 GPUs, per-GPU MTBF 20,000 h. Under the independent-failure approximation, the job (gang-scheduled, all-or-nothing — any one GPU down kills it) has `MTBF_job ≈ MTBF_gpu / N = 20,000 / 1,024 ≈ 19.5 h`. Layer in a **silent data corruption** event roughly every week or two across a fleet at Google's reported scale (10,000 chips), plus Meta's reported ~1.4% SDC share of unexpected interruptions, and the effective mean time between *interruptions* (not just clean crashes) drops further. Call the effective interrupt interval ~12–16 h for this worked example.

**Optimal checkpoint interval (Young/Daly).** `interval ≈ √(2·δ·MTBF)` where `δ` = checkpoint write cost. With `δ = 5 min = 0.083 h` and `MTBF = 15 h`: `interval ≈ √(2 × 0.083 × 15) ≈ √2.5 ≈ 1.58 h`. Wasted compute ≈ checkpoint overhead + expected rework: roughly `δ/interval + interval/(2·MTBF) ≈ 0.083/1.58 + 1.58/30 ≈ 5.3% + 5.3% ≈ 10–11%`. That ~10% is the price of surviving a days-MTBF fleet with checkpointing alone; halving `δ` (async/sharded checkpointing) is the lever that moves it directly. The other lever, per Core concepts, is **elasticity** — Google's reported ~97%-throughput continuation after a localized failure reduces the *N* affected by a given failure rather than the rework per failure, which is a different term in the same cost equation and can be pursued independently or in combination.

**Client-vantage health check.** To catch a gray-failing NIC that `nvidia-smi` reports healthy: don't poll status — run an *active* differential probe. Periodically execute a small all-reduce / collective across each node's NICs and *check the numerical result and the achieved bandwidth* against the healthy baseline; a node that returns wrong bytes or is 2% below line rate is cordoned and drained even though every local status read is green. The health signal comes from a client-side compute result, not a server-side status register.

## Practice

Feeds [staff design portfolio](../practice/staff-design-portfolio/README.md). For a stated GPU count and per-GPU MTBF plus an SDC event rate, compute job-level MTBF, the Young/Daly optimal checkpoint interval, and the wasted-compute percentage; then vary `δ` to show the sensitivity, and separately estimate how much an elastic-recovery capability (continue at reduced scale rather than full restart) would change the effective wasted-compute number. Separately, design a client-vantage / differential-observability health check that catches a gray-failing NIC or an SDC-producing GPU that `nvidia-smi` calls healthy — specify the active probe, the baseline, the cordon/drain trigger, and why a status-read health check would miss it.

## Common pitfalls

1. **"We just need a better timeout value."** FLP says no timeout value distinguishes slow-from-dead under asynchrony — it's an inherent tradeoff (false positives vs. detection latency) you tune deliberately, not a bug a smarter constant fixes.
2. **"Health checks passing means the node is healthy."** Gray failure is precisely a passing health check on a degraded component. Only active, client-vantage compute probes — not status reads — catch it.
3. **"Two replicas double our effective MTBF."** This assumes independent failure. Replicas sharing a rack, PDU, top-of-rack switch, or NIC firmware version are correlated, not independent — and on a 1,024-GPU gang-scheduled job, effective MTBF is `MTBF_gpu / N` regardless of redundancy, because any single failure kills the whole gang.
4. **"Circuit breakers are strictly good hygiene."** Per Brooker's critique, a circuit breaker adds a mode that can itself misfire and worsen a partial outage by tripping on a healthy subset of traffic. Continuous load-shedding plus retry budgets often degrade more gracefully than a binary open/closed breaker.
5. **"More frequent checkpoints are always safer."** Checkpoint overhead scales with frequency; Young/Daly shows a real optimum. Over-checkpointing wastes as much compute as under-checkpointing does, just through a different mechanism (write overhead instead of rework).

## Self-check

- Why can't a timeout distinguish a slow node from a dead one, and what does that force you to trade? **Answer:** FLP impossibility — under asynchrony there's no bound that separates "slow" from "crashed," so any timeout is a calibrated guess. It forces a trade between false positives (killing/failing-over a healthy-but-slow node) and detection latency (waiting longer while a real outage runs). Phi-accrual detectors parameterize the tradeoff explicitly; they don't remove it.
- On a 1,024-GPU gang-scheduled job, why doesn't adding replica redundancy help, and what does? **Answer:** The job is all-or-nothing and failures are correlated — one GPU or NIC kills the whole gang, so the blast radius is the entire job and replica redundancy has nothing to protect there. What actually helps is checkpoint frequency plus fast restart (bounding rework via Young/Daly), elastic recovery (bounding how many chips a given failure removes from the job), and reducing correlated dependencies (shared racks/PDUs/firmware).
- What is gray failure and why does a normal health check miss it? **Answer:** A component that degrades — a NIC dropping 2% of packets, a GPU silently corrupting data — while its own status reads still report healthy: differential observability, where the server's vantage says fine and the client's vantage suffers. A status-poll health check reads the healthy register; only an active compute probe from the client's vantage, checked against a baseline, catches it.
- Why does GPU hardware specifically tend to fail silently (SDC) rather than crashing cleanly, and what detection strategy does that require? **Answer:** Mechanisms like marginal HBM cells near their reliability edge, cosmic-ray-induced bit flips, and thermal-throttling-related miscounts on marginal silicon all produce a *wrong number* rather than a crash or exception — nothing trips a status alarm. Detection has to be active and numerical: run a known computation (a checkpoint replay, a collective op) and check the result and performance against a healthy baseline, rather than relying on process-alive or status-register checks.

## Connections & what's next

This lesson depends directly on lesson 05: every shedding, retry, and failover decision in the queueing/backpressure lesson assumes you can detect overload or failure, and this lesson shows that detection is provably imperfect — the two lessons together are "how to act under overload" and "how to know what's actually happening while you act." It also sets up two lessons ahead: lesson 07, where a cascading failure's recovery path (per the PagerDuty case above) routinely produces delivery-semantics bugs — duplicates leaking through a system that assumed exactly-once — and lesson 08's design method, whose "failure & operations" step draws directly on this lesson's gray-failure and checkpoint-economics framing. Next: **[07 · Data-intensive patterns](07-data-intensive-patterns.md)** — starting from exactly the seam this lesson leaves open: what happens to correctness when a retry or a recovery, done under uncertainty about what actually failed, causes a message to be delivered more than once.

## References & further reading

**Primary sources**
- **Young, J.W. (1974), "A First Order Approximation to the Optimum Checkpoint Interval,"** *Communications of the ACM* 17(9):530–531 — <https://cacm.acm.org/research/a-first-order-approximation-to-the-optimum-checkpoint-interval/> — the original derivation of the `√(2·δ·MTBF)` formula that Daly's later work (below) builds on; read for where the square root actually comes from.
- **Fischer, M., Lynch, N., Paterson, M. (1985), "Impossibility of Distributed Consensus with One Faulty Process,"** *Journal of the ACM* 32(2) — <https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf> — the FLP result this lesson invokes by name; read for the precise asynchrony assumptions the impossibility depends on.
- **Meta (2024), "The Llama 3 Herd of Models,"** arXiv:2407.21783 — <https://arxiv.org/abs/2407.21783> — primary source for the interruption and SDC statistics used throughout this lesson; read the infrastructure/reliability section directly rather than a secondary summary.
- **Huang, P. et al. (2017), "Gray Failure: The Achilles' Heel of Cloud-Scale Systems,"** HotOS '17 — <https://www.microsoft.com/en-us/research/publication/gray-failure-achilles-heel-cloud-scale-systems/> — the paper that coined differential observability; read for the taxonomy of gray-failure causes.
- **Daly, J.T. (2006), "A higher order estimate of the optimum checkpoint interval for restart dumps,"** — <https://www.usenix.org/legacy/events/fast09/tech/full_papers/daly/daly.pdf> — the refinement of Young's formula used in production checkpoint-interval calculators.

**Real-world engineering blogs**
- **Google, "Training in Turmoil: Silent Data Corruption in Systems at Scale"** — <https://research.google/pubs/training-in-turmoil-silent-data-corruption-in-systems-at-scale/> — what it shows: a second hyperscaler's independent confirmation of weekly-scale SDC events, with a concrete deterministic-replay detection and elastic-recovery architecture.
- **Roblox, "Roblox Return to Service 10/28–10/31 2021"** — <https://blog.roblox.com/2022/01/roblox-return-to-service-10-28-10-31-2021/> — what it shows: a 73-hour outage where the failure and the monitoring needed to diagnose it were entangled — a real gray-failure diagnosis story.
- **PagerDuty, "August 28 Kafka Outages – What Happened and How We're Improving"** — <https://www.pagerduty.com/eng/august-28-kafka-outages-what-happened-and-how-were-improving/> — what it shows: observability gaps delaying root-cause detection, and a recovery process that itself produced duplicate deliveries — the direct bridge into lesson 07.
- **Meta, "How Meta keeps its AI hardware reliable"** — <https://engineering.fb.com/2025/07/22/data-infrastructure/how-meta-keeps-its-ai-hardware-reliable/> — what it shows: production tooling (ServiceLab, Fleetscanner) built specifically to catch SDC and performance-outlier hardware faults at hyperscale.

**Deeper dives**
- **AWS Builders' Library, "Timeouts, retries, and backoff with jitter"** — <https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/> — a practitioner-level walkthrough of choosing timeout values against a false-positive budget, directly downstream of the FLP tradeoff discussed above.
