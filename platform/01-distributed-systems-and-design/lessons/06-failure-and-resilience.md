---
lesson: "A01.6"
title: "Failure and resilience"
module: "A-01"
concept: "Imperfect detectors, gray failure, checkpoint math"
status: not-started
est_time: "3 hrs"
artifacts: ["job-level MTBF + optimal checkpoint interval calc", "client-vantage health-check design"]
---

# A01.6 · Failure and resilience

> **Concept.** Failure detectors are fundamentally imperfect and health checks lie; on a GPU fleet the deciding failures are gray (SDC) and correlated (gang-scheduled), so resilience is checkpoint math and client-vantage probing, not replica redundancy.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Why this matters
The redundancy math every senior engineer carries — "two replicas, MTBF squared" — silently assumes independent, fail-stop failures observable by a health check. On a large training run all three assumptions break: failures are correlated (one NIC kills a 1,024-GPU gang), they are gray (a GPU passes `nvidia-smi` while corrupting gradients), and the detector cannot tell slow from dead. The staff bound here is the checkpoint interval — a single number from `√(2·δ·MTBF)` — that sets how much compute you burn to survive a fleet whose real MTBF is days, not years.

## Core notes
**Skip (you already know):** retries and timeouts exist; circuit breakers are a pattern; redundancy raises availability.

**Failure detectors are fundamentally imperfect (FLP).** A timeout cannot distinguish a slow node from a dead one — this is not an engineering gap, it's an impossibility result under asynchrony. Every timeout trades a **false-positive** (kill/failover a healthy-but-slow node) against **detection latency** (wait longer, tolerate more of a real outage). Phi-accrual detectors replace the boolean with a *suspicion probability* over observed heartbeat inter-arrival, letting you set the false-positive rate explicitly instead of guessing a timeout constant — but they don't remove the tradeoff, they parameterize it.

**Timeouts, retries, jitter done right.** Static timeouts fossilize a latency assumption; adaptive timeouts track the moving p99. Exponential backoff **with full jitter** — naked backoff still synchronizes clients into retry waves; jitter spreads them. The staff upgrade is **retry budgets / token-bucket retries**: cap total retries at a small fraction (e.g. 10%) of base load so a partial outage can't multiply into a retry storm. Brooker's skeptical take on **circuit breakers**: they add a *mode* (open/closed/half-open) that itself must be tested and can worsen a partial outage by tripping on a healthy subset — prefer load-based shedding + budgets, which degrade continuously rather than flipping.

**Gray failure & differential observability** (Huang et al., HotOS '17) — the most important resilience concept for a GPU fleet. The system's *own* health check passes while clients suffer: a NIC dropping 2% of packets, a disk degraded but spinning, a GPU with a marginal HBM cell. The server's vantage says healthy; the client's vantage says broken. The gap between them (*differential observability*) is where outages hide. The only fix is to observe from the client's vantage, and to probe with *active compute*, not status reads.

**Cascading & metastable failure, blast radius.** Retry storms feed the death spiral; containment is architectural — **cells / shuffle sharding** bound how many customers any single failure can reach. And correlated failure kills the redundancy math: two "independent" replicas sharing a rack, PDU, top-of-rack switch, or NIC firmware version are not independent; their joint failure probability is far above the product of the marginals.

## Worked example
**Job-level MTBF.** 1,024 GPUs, per-GPU MTBF 20,000 h. Under the independent-failure approximation, the job (gang-scheduled, all-or-nothing — any one GPU down kills it) has `MTBF_job ≈ MTBF_gpu / N = 20,000 / 1,024 ≈ 19.5 h`. Layer in a **silent data corruption** event roughly every few days across the fleet and the effective mean time between *interruptions* drops further — Meta attributed ~1.4% of Llama-3 unexpected interruptions to SDC; Google reports an SDC event every week or two in Gemini training. Call the effective interrupt interval ~12–16 h.

**Optimal checkpoint interval (Young/Daly).** `interval ≈ √(2·δ·MTBF)` where `δ` = checkpoint write cost. With `δ = 5 min = 0.083 h` and `MTBF = 15 h`: `interval ≈ √(2 × 0.083 × 15) ≈ √2.5 ≈ 1.58 h`. Wasted compute ≈ checkpoint overhead + expected rework: roughly `δ/interval + interval/(2·MTBF) ≈ 0.083/1.58 + 1.58/30 ≈ 5.3% + 5.3% ≈ 10–11%`. That ~10% is the price of surviving a days-MTBF fleet; halving `δ` (async/sharded checkpointing) is the lever that moves it.

**Client-vantage health check.** To catch a gray-failing NIC that `nvidia-smi` reports healthy: don't poll status — run an *active* differential probe. Periodically execute a small all-reduce / collective across each node's NICs and *check the numerical result and the achieved bandwidth* against the healthy baseline; a node that returns wrong bytes or 2% below line rate is cordoned and drained even though every local status read is green. The health signal comes from a client-side compute result, not a server-side status register.

## Practice
Feeds [staff design portfolio](../practice/staff-design-portfolio/README.md). For a stated GPU count and per-GPU MTBF plus an SDC event rate, compute job-level MTBF, the Young/Daly optimal checkpoint interval, and the wasted-compute percentage; then vary `δ` to show the sensitivity. Separately, design a client-vantage / differential-observability health check that catches a gray-failing NIC or an SDC-producing GPU that `nvidia-smi` calls healthy — specify the active probe, the baseline, the cordon/drain trigger, and why a status-read health check would miss it.

## Self-check
- Why can't a timeout distinguish a slow node from a dead one, and what does that force you to trade? **Answer:** FLP impossibility — under asynchrony there's no bound that separates "slow" from "crashed," so any timeout is a guess. It forces a trade between false-positives (killing a healthy-but-slow node) and detection latency (waiting longer while a real outage runs). Phi-accrual parameterizes the tradeoff; it doesn't remove it.
- On a 1,024-GPU gang-scheduled job, why doesn't adding replica redundancy help, and what does? **Answer:** The job is all-or-nothing and failures are correlated — one GPU/NIC kills the whole gang, so the blast radius is the entire job and replica redundancy has nothing to protect. What helps is checkpoint frequency + fast restart (bounding rework via Young/Daly) and reducing correlated dependencies.
- What is gray failure and why does a normal health check miss it? **Answer:** A component that degrades (a NIC dropping 2% of packets, a GPU silently corrupting data) while its own status reads report healthy — differential observability: server vantage says fine, client vantage suffers. A status-poll health check reads the healthy register; only an active compute probe from the client's vantage, checked against a baseline, catches it.

## References
- https://www.microsoft.com/en-us/research/publication/gray-failure-achilles-heel-cloud-scale-systems/
- https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/
- https://engineering.fb.com/2025/07/22/data-infrastructure/how-meta-keeps-its-ai-hardware-reliable/
- https://www.usenix.org/legacy/events/fast09/tech/full_papers/daly/daly.pdf
- https://raftconsensus.github.io/
