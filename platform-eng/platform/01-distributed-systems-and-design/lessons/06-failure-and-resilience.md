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
sources: 22
---

# A01.6 · Failure and resilience

> **Concept.** Failure detectors are fundamentally imperfect and health checks lie; on a GPU fleet the deciding failures are gray (SDC) and correlated (gang-scheduled), so resilience is checkpoint math and client-vantage probing, not replica redundancy.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 05 gave you the theory of overload: shed cheap and early, don't defer, and watch for the sustaining feedback loop that turns a spike into a metastable outage. All of that assumed one thing you were never asked to question — that you can *tell* the system is overloaded, or a node is dead, or a request needs to be shed. That assumption is the subject of this lesson, and it turns out to be provably imperfect. Deciding *when* to shed a request, retry a call, or fail a node over all reduce to the same underlying question: is the thing on the other end slow, or is it dead? This lesson is the layer underneath everything in lesson 05 — the failure-detection substrate that every backpressure, retry, and shedding decision is built on top of, and the reason none of those decisions can ever be made with certainty.

It is also where the module's numbers stop being about steady-state throughput and start being about *expected loss*: how much work a failure costs you, how far a failure spreads, and how much compute you are willing to burn in advance to bound both.

## Why this matters

The redundancy math every senior engineer carries — "two replicas, MTBF squared" — silently assumes independent, fail-stop failures observable by a health check. On a large training run all three assumptions break: failures are correlated (one NIC kills a 1,024-GPU gang), they are gray (a GPU passes `nvidia-smi` while corrupting gradients), and the detector cannot tell slow from dead. The staff bound here is the checkpoint interval — a single number from `√(2·δ·MTBF)` — that sets how much compute you burn to survive a fleet whose real MTBF is days, not years.

The other half is cheaper to state and more expensive to get wrong: **a retry is a load multiplier, and multipliers compose.** Three tiers that each retry twice turn one client request into up to 27 requests at the bottom of the stack. That is the arithmetic behind most of the "the site went down and stayed down after we fixed the trigger" incidents in the industry, and it is fixable with three specific mechanisms — jittered backoff, retry budgets, and a correctly-parameterised breaker — that this lesson gives you in implementation-level detail rather than as names.

## What's new here (calibration)

**Skip (you already know):** retries and timeouts exist; circuit breakers are a pattern; redundancy raises availability; `kubectl get nodes` shows `NotReady` when a kubelet stops heartbeating.

The genuinely new depth this lesson adds, past that baseline:

- **Why timeouts can never be perfect** — not an engineering gap you can close with a smarter algorithm, but a formal impossibility result (FLP) that tells you exactly what tradeoff you're stuck making, plus the two properties (completeness, accuracy) that let you say *which* imperfection you bought.
- **The exact jitter formulas** — full, equal, and decorrelated jitter as they are actually written in AWS's own published simulator, with a re-run of that simulator's numbers so you can see the size of the effect rather than take "add jitter" on faith.
- **Real circuit-breaker state machines with real thresholds** — Hystrix and resilience4j side by side, including the half-open behaviour (one probe vs. ten), and Envoy's per-host variant with its default ejection percentages.
- **Retry budgets as arithmetic** — gRPC's token-bucket throttle with its exact accounting rules, and the amplification factor a budget replaces.
- **Metastable failure as a two-threshold system** — why a system can stay down after the trigger is gone, and the only two exits from the loop.
- **Differential observability / gray failure** as the organizing idea for GPU-fleet resilience — the specific reason "the health check is green" stops meaning "the node is fine" at scale.
- **Why GPUs specifically produce silent corruption rather than clean crashes** — the hardware/silicon mechanisms, not just the abstract "gray failure" label.
- **Checkpoint economics as a pure cost calculus** (Young/Daly), derived rather than quoted, so you can re-run it with your own `δ` and MTBF.

Version note: implementation defaults below were read from source in August 2026 — Netflix Hystrix `HystrixCommandProperties`/`HystrixCircuitBreaker` (archived, 1.5.x line), resilience4j `CircuitBreakerConfig` (master), Envoy `api/envoy/config/cluster/v3/{outlier_detection,circuit_breaker}.proto`, gRPC gRFC A6, Kubernetes `pkg/kubelet` and `client-go/util/flowcontrol` (master). Defaults move; the mechanisms don't.

## Core concepts

### 1 · The detection problem, stated precisely

Every resilience mechanism in this lesson is downstream of one question: **process P sent a request to process Q and has not heard back. Is Q dead?**

In 1985 Fischer, Lynch, and Paterson proved you cannot answer it. Their result is about consensus, but the reason consensus is impossible is the reason detection is impossible: in a **fully asynchronous** system — no upper bound on message delay, no upper bound on process step time, no synchronised clocks — a message that has not arrived is indistinguishable from a message that will arrive later. Any deterministic protocol that must terminate can be forced into an infinite indecisive execution by an adversary that delays exactly the right message, even if only one process may crash. It is a proof, not a limitation of 1985 hardware.

Real systems escape by weakening an assumption. The standard escape is **partial synchrony** (Dwork, Lynch, Stockmeyer 1988): assume that message delay is bounded by some unknown Δ *after some unknown time* — the network is eventually well-behaved, you just don't know when or by how much. Under that assumption consensus becomes solvable, and the price is that your algorithm's *liveness* depends on a timeout whose correct value you cannot compute. That timeout is where FLP's impossibility gets deposited. Raft's election timeout, etcd's heartbeat interval, and your HTTP client's `ResponseHeaderTimeout` are all the same parameter wearing different names.

**What a timeout actually buys and costs.** Model it as a classifier over the distribution of response times:

```
             observed response-time distribution of a HEALTHY dependency
   density
     │        ╭──╮
     │       ╱    ╲                             timeout T
     │      ╱      ╲___                             │
     │    ╱             ╲──────────────────╮        │
     │  ╱                                   ╲───────┼─────────────▶ time
     └──┴────────┴──────────────┴───────────────────┴──────────────
       p50      p90            p99                 p99.9

     T set here (≈p99)   ──▶  ~1% of HEALTHY calls are declared failures
                              (false positives; each one costs a retry,
                               and retries are load multipliers)

     T set here (≈p99.9) ──▶  0.1% false positives, but a genuinely dead
                              dependency is only noticed after T, and
                              every in-flight request holds a thread/
                              connection/slot for the full T

   The curve is not fixed: it moves right under load, which is exactly when
   you can least afford either error. That is why a static timeout tuned in
   January is a latent incident in June.
```

Two costs, one knob:

- **False positive rate** — the fraction of healthy-but-slow calls you kill. Each one is a retry, an ejected host, or a failover. At scale, an aggressive timeout during a mild latency excursion *creates* the outage it was meant to detect, because the retries it generates are load.
- **Detection latency** — how long a genuinely dead peer keeps looking "maybe alive". During that window every request routed to it is lost, and every caller holds a resource for the timeout duration. If your service has a 200-request concurrency limit and a dependency with a 30 s timeout goes dark, you have ~200 slots ÷ 30 s = under 7 req/s of throughput left. That is how one dead dependency takes down a healthy service.

There is no setting of `T` that zeroes both. You are choosing a point on a curve, and the staff move is to say which point and why — "timeout at p99.9 of the healthy distribution plus 50 ms, so we accept ~1-in-1000 false kills to keep detection under a second" — rather than inheriting a framework default.

### 2 · Completeness and accuracy: naming which imperfection you bought

Chandra and Toueg (1996) gave the vocabulary that makes "imperfect" precise. A failure detector is a module at each process that outputs a list of suspected processes, and it is graded on two orthogonal properties:

| Property | Informal statement | What violating it looks like |
|---|---|---|
| **Completeness** | Every process that actually crashes is eventually suspected by every correct process. | A dead node stays in the routing table forever; requests keep being sent into a black hole. |
| **Accuracy** | A correct (live) process is not suspected. | A GC pause or a slow disk gets a healthy node evicted, killing capacity you needed. |

Perfect accuracy is unattainable under asynchrony (that's FLP again). So real detectors aim for **eventually strong** (`◇S`) behaviour: strong completeness plus *eventual* weak accuracy — after some unknown point, at least one correct process stops being suspected. That is exactly enough to make consensus solvable, which is why it is the class Raft-style systems implicitly assume.

Practically, this table tells you where to spend. **Completeness is cheap** — lengthen the timeout, you get it. **Accuracy is expensive** — it requires either a longer timeout (worse detection latency) or a smarter model of what "late" means. The smarter model is the accrual detector.

### 3 · Phi-accrual detectors: the tunable version of the same tradeoff

A boolean detector forces you to pick one millisecond constant that must serve both "the network hiccupped" and "the machine is on fire". A **φ-accrual** detector (Hayashibara et al., 2004) instead outputs a continuous suspicion level, and lets each consumer pick its own threshold.

The mechanism, concretely:

1. Keep a sliding window of the last *N* heartbeat inter-arrival times (Akka default `max-sample-size = 1000`).
2. Fit a distribution to that window — the classic implementation assumes normal, with mean `μ` and standard deviation `σ` estimated from the samples, and a floor on `σ` so a suspiciously regular sender doesn't make the detector hair-trigger (Akka: `min-std-deviation = 100 ms`).
3. Let `t_diff` be the time since the last heartbeat arrived. Compute the probability that a heartbeat would legitimately be at least this late: `P_later(t_diff) = 1 − F(t_diff)`, where `F` is the fitted CDF.
4. Output **`φ = −log₁₀(P_later(t_diff))`**.

φ is a log-odds of being wrong. **φ = 8 means "if I convict now, the chance that this node is actually healthy is about 10⁻⁸."** That is not a coincidence of two projects: Apache Cassandra ships `phi_convict_threshold: 8` in `cassandra.yaml`, and Akka Cluster ships `akka.cluster.failure-detector.threshold = 8.0`. Akka additionally adds `acceptable-heartbeat-pause = 3 s` to the mean before computing φ, explicitly to survive a GC pause without convicting — a direct, legible encoding of "buy accuracy with detection latency".

Worked: heartbeat interval 1 s, observed `μ = 1.0 s`, `σ = 0.15 s`, normal fit. At `t_diff = 1.5 s`, that's 3.3σ out, `P_later ≈ 4.7×10⁻⁴`, φ ≈ 3.3 — suspicious, not convicted. At `t_diff = 2.0 s` (6.7σ), `P_later ≈ 1.3×10⁻¹¹`, φ ≈ 10.9 — well past 8, convict. Now let the network degrade so `σ` rises to 0.5 s: the same 2.0 s gap is only 2σ out, `P_later ≈ 0.023`, φ ≈ 1.6 — **the detector automatically became more patient because the network got noisier.** That adaptivity is the whole point; a fixed 1.8 s timeout would have started convicting the entire cluster.

φ does not repeal FLP. It re-parameterises the same tradeoff into a unit you can reason about ("what false-conviction rate am I buying?") instead of a millisecond constant you guessed.

### 4 · Health checks lie in a specific, predictable way

Kubernetes gives you three probe types and they encode three different questions:

| Probe | Question | Failure action | Classic misuse |
|---|---|---|---|
| `livenessProbe` | Is this process wedged? | Kill the container. | Pointing it at a dependency, so a database blip restarts every pod in the fleet simultaneously. |
| `readinessProbe` | Should this pod receive traffic now? | Remove from Service endpoints. | Making it expensive, so it fails under the load it was meant to shield you from. |
| `startupProbe` | Has slow initialisation finished? | Suppress the other two until it passes. | Omitting it on a model server that takes 4 minutes to load weights, so liveness kills it at 3. |

The structural trap is the **deep health check**: a probe that validates the dependency graph rather than the process. Deep checks have excellent completeness and terrible correlated accuracy — when the shared dependency degrades, *every* instance fails the check *at the same instant*, and the health-check system removes 100% of your capacity in response to a partial degradation. This is the mechanism behind "the load balancer took the whole fleet out of rotation."

Production systems defend against it by capping how much of a fleet the ejection machinery may remove. Envoy's outlier detection — the same code path Istio's `outlierDetection` configures — has this as a first-class default:

| Envoy `outlier_detection` field | Default | What it does |
|---|---|---|
| `consecutive_5xx` | 5 | Consecutive server-side errors before ejection. |
| `interval` | 10 s | How often the ejection sweep runs (ejects *and* un-ejects). |
| `base_ejection_time` | 30 s | Ejection duration; multiplied by the number of times this host has been ejected. |
| `max_ejection_time` | 300 s | Cap on that multiplication. |
| **`max_ejection_percent`** | **10 %** | **Hard cap on the fraction of the cluster ejected at once.** |
| `enforcing_consecutive_5xx` | 100 | Percent of the time an eligible ejection is actually enforced (a dial to ramp detection in). |
| `success_rate_minimum_hosts` | 5 | Below this cluster size, statistical outlier detection is off. |
| `success_rate_request_volume` | 100 | Per-host requests needed in one interval to include it in the statistics. |
| `success_rate_stdev_factor` | 1900 | Divided by 1000 → eject hosts more than 1.9σ below the cluster mean success rate. |
| `failure_percentage_request_volume` | 50 | Per-host volume needed for percentage-based ejection. |

Read the last three rows together and you have the actually-interesting algorithm: Envoy computes the **cluster-wide mean and standard deviation of per-host success rate** each interval, and ejects hosts whose success rate is below `mean − 1.9σ`. That is a differential check — a host is unhealthy *relative to its peers*, not relative to an absolute threshold — which is precisely the shape that survives a fleet-wide degradation without ejecting the fleet. And `max_ejection_percent: 10` is the backstop for when even that reasoning is wrong.

**The generalisable rule: a health-check system needs a bound on its own blast radius.** Any mechanism that removes capacity in response to a signal must refuse to remove more than a fixed fraction, because the signal can be wrong about all of them at once.

### 5 · Retries are a load multiplier, and multipliers compose

Here is the arithmetic that decides most cascading-failure incidents.

Let a tier make at most `a` total attempts per logical call (one initial + `a−1` retries). A single client request that traverses `d` tiers, each retrying independently, can produce up to `a^d` requests at the bottom. That exponent is the entire problem.

```
        RETRY AMPLIFICATION — three tiers, 3 attempts each (a=3, d=3)

  clients        gateway            scheduler API        etcd / metadata store
  ───────        ───────            ─────────────        ─────────────────────
                    │                     │                      │
   1,000 req/s ────▶│  a=3          ┌─────▼─────┐          ┌──────▼──────┐
                    │──────────────▶│  3,000/s  │─────────▶│  9,000/s    │
                    │  on timeout   │           │  a=3     │             │
                    │               └───────────┘          └──────┬──────┘
                    │                                             │ a=3
                    │                                      ┌──────▼──────┐
                    │                                      │  27,000/s   │  ← 27x
                    │                                      └─────────────┘
                                                            capacity 12,000/s
                                                            ═══ SATURATED ═══
                                                                  │
        ┌─────────────────────────────────────────────────────────┘
        │  saturation ⇒ latency ⇒ upstream timeouts fire ⇒ MORE retries
        ▼
   the amplification is not a one-off spike; it is a positive feedback term


        THE SAME PATH WITH THE THREE FIXES APPLIED

  clients        gateway            scheduler API        etcd / metadata store
   1,000 req/s ────▶│                     │                      │
                    │ retry budget 10%    │ NO retry here        │
                    │ full-jitter backoff │ (retry at ONE layer) │
                    │ breaker per host    │                      │
                    │──────────────▶ 1,100/s ─────────────▶ 1,100/s   ← 1.1x
                    │                                      capacity 12,000/s
                    │                                      ══ 9% utilised ══
                    │
                    └─ breaker OPEN for a host ⇒ that host's share fails fast
                       (cheap 503 at the edge, not a queued request deep in)
```

Three fixes, in order of leverage:

**(a) Retry at exactly one layer.** If the gateway retries and the scheduler client retries and the SDK retries, you have three multipliers. Pick the layer with the most context about whether a retry can help — usually the one closest to the client that still knows the request is idempotent — and make every other layer return the error. In gRPC terms, a service config `retryPolicy` on the outermost channel and none below it.

**(b) Cap the aggregate, not the individual.** Per-client backoff bounds *one* client's behaviour and says nothing about the fleet's. That is exactly the finding in Slack's 2-22-22 write-up: their clients were already doing exponential backoff with jitter — textbook-correct — and the retry storm still sustained the overload, because correct per-client behaviour does not bound the sum. See §7.

**(c) Make the failure cheap.** A retry that fails at the edge in 2 ms costs ~0 capacity; a retry that fails after a 30 s timeout deep in the call graph holds a slot the whole time. This is the same "shed cheap" principle from lesson 05, applied to the retry path.

**Worked amplification factors** (steady state, every call failing and exhausting its attempts):

| Config | Multiplier at the bottom tier | 1,000 req/s becomes |
|---|---:|---:|
| 3 tiers × 3 attempts | 3³ = 27 | 27,000/s |
| 3 tiers × 2 attempts | 2³ = 8 | 8,000/s |
| 1 tier × 3 attempts | 3 | 3,000/s |
| 1 tier, 10 % retry budget | 1.10 | 1,100/s |
| 3 tiers, 10 % budget each | 1.10³ = 1.331 | 1,331/s |

The last row is the one to remember: **budgets compose multiplicatively too, but 1.1³ is 1.33, not 27.** A budget converts an exponential blow-up into a rounding error.

### 6 · Backoff and jitter: the exact formulas, and what they buy

Everyone knows "exponential backoff with jitter". Almost nobody can write the three variants. Here they are, transcribed from the algorithm definitions in AWS's own published backoff simulator (`aws-samples/aws-arch-backoff-simulator`, `src/backoff_simulator.py`), which is the code behind the 2015 AWS Architecture Blog post on the subject:

```python
# Shared: exponential envelope, capped.
def expo(n):                       # n = attempt number, 1-based
    return min(CAP, BASE * 2**n)

# 1. Exponential, no jitter — every client sleeps the SAME duration.
def backoff_none(n):
    return expo(n)

# 2. Full jitter — sleep uniformly anywhere in [0, envelope].
def backoff_full(n):
    return random.uniform(0, expo(n))

# 3. Equal jitter — half the envelope guaranteed, half randomised.
def backoff_equal(n):
    v = expo(n)
    return v/2 + random.uniform(0, v/2)

# 4. Decorrelated jitter — a random walk anchored to the PREVIOUS sleep,
#    not to the attempt number. `sleep` is per-client state, init to BASE.
def backoff_decorr(n):
    global sleep
    sleep = min(CAP, random.uniform(BASE, sleep * 3))
    return sleep
```

The simulator's own parameters are `BASE = 5 ms`, `CAP = 2000 ms`, a network modelled as `|Normal(μ=10 ms, σ=2 ms)|`, and *C* clients all contending to write one optimistically-concurrency-controlled row — each client retries until its own write wins, so contention is the failure mode. Each configuration is averaged over 100 runs.

Re-running that published simulator (Python-3 port: `xrange`→`range`, integer division made explicit; no algorithmic change) gives:

| Clients | | No backoff | | Exponential | | Equal jitter | | Full jitter | | Decorrelated |
|---:|---|---:|---|---:|---|---:|---|---:|---|---:|
| | | **time / calls** | | **time / calls** | | **time / calls** | | **time / calls** | | **time / calls** |
| 10 | | 383 / 51 | | 3,442 / 50 | | 782 / 42 | | 481 / 39 | | 442 / 37 |
| 20 | | 594 / 150 | | 14,796 / 151 | | 1,679 / 107 | | 1,003 / 100 | | 824 / 100 |
| 50 | | 1,142 / 693 | | 36,259 / 624 | | 4,198 / 346 | | 3,005 / 331 | | 2,275 / 374 |
| 100 | | 2,032 / 2,422 | | 63,840 / 1,864 | | 6,539 / 811 | | 4,842 / 794 | | 4,565 / 1,002 |
| 150 | | 2,878 / 5,144 | | 85,526 / 3,545 | | 8,270 / 1,322 | | 6,395 / 1,318 | | 6,365 / 1,763 |
| 190 | | 3,540 / 8,011 | | 101,155 / 5,166 | | 9,469 / 1,762 | | 7,375 / 1,772 | | 8,035 / 2,429 |

*(time = simulated ms until all clients have succeeded; calls = total server calls. Re-run in this environment in August 2026 from the unmodified upstream algorithm code — the AWS blog post itself was not reachable from here, so treat these as a faithful re-execution of the published simulator rather than a transcription of the published table.)*

Read four things out of it:

1. **Plain exponential backoff is catastrophically slow at scale and does not even save work.** At 100 clients it takes 63,840 ms versus full jitter's 4,842 ms — 13× longer — while issuing *more* calls (1,864 vs 794). Without jitter, every client's envelope is the same, so the clients stay synchronised: they all sleep, all wake together, all collide again, and all double together. Backoff without jitter does not spread a herd, it *paces* a herd.
2. **Jitter is the mechanism, backoff is just the envelope.** Full jitter cuts server calls by ~57 % versus plain exponential at 100 clients and finishes 13× sooner. The randomisation is what decorrelates arrivals; the exponential part only bounds the worst case.
3. **Full vs equal jitter is a real but second-order choice.** Equal jitter guarantees a minimum sleep (`v/2`), which is useful when you need a hard floor between attempts — a rate-limited API with a `Retry-After` contract, say. It pays for that floor with ~35 % more completion time at 100 clients (6,539 vs 4,842 ms) at essentially the same call count.
4. **Decorrelated jitter trades work for latency, and the trade flips.** It is the fastest option at low contention (10–100 clients) because its random walk can jump back down to `BASE` instead of monotonically doubling. But it issues noticeably more calls under heavy contention (2,429 vs 1,772 at 190 clients) and by then has lost its latency edge too. Use it when the server has spare capacity and you care about tail completion time; use full jitter when you are protecting a server that is already the bottleneck.

**The default you should reach for is full jitter**, and the one-line reason is: it is the only variant whose expected sleep grows with the envelope *and* whose arrival process is memoryless enough to spread a synchronised herd.

**A worked cautionary case from a system you already run.** Kubernetes' kubelet backs off container restarts with `client-go`'s `util/flowcontrol.Backoff`. The constants, from source: `initialCrashLoopBackOff = 10 s`, `MaxContainerBackOff = 300 s`, doubling per event, and the entry resets to the initial value after `2 × maxDuration` = 10 minutes without an event. The jitter factor is a constructor argument — and `kubelet.go` builds it with `flowcontrol.NewBackOff(base, boMax)`, which sets `maxJitterFactor = 0.0`, i.e. **no jitter at all**. So the familiar 10 s → 20 s → 40 s → … → 5 min `CrashLoopBackOff` ladder is perfectly synchronised across every pod that crashed at the same moment. That is fine for its purpose (the retry target is the local container runtime, not a shared remote service) and instructive as a rule: *unjittered backoff is acceptable exactly when the retries do not contend for a shared resource.* The moment they do — a fleet of pods all re-pulling the same image, all re-resolving the same DNS name, all re-registering with the same API — you need the jitter.

### 7 · Retry budgets: capping the aggregate

Per-client backoff is a local property. A retry budget is a global one, and it is the only mechanism in this lesson that directly bounds the multiplier from §5.

**gRPC's `retryThrottling`** (gRFC A6) is the cleanest published specification, and it is a token bucket with unusual accounting. Per *server name* (not per method, not per service), the client keeps `token_count ∈ [0, maxTokens]`, initialised to `maxTokens`:

```
config:  { "retryThrottling": { "maxTokens": 10, "tokenRatio": 0.1 } }

on every RPC completion:
    failure  →  token_count -= 1
    success  →  token_count += tokenRatio          (clamped to maxTokens)

threshold = maxTokens / 2                          (= 5 with the config above)

retry (or a non-first hedged attempt) is allowed  ⟺  token_count > threshold
when not allowed: the retry is NOT queued or delayed — it is cancelled and the
                  failure is returned to the application immediately
```

Three details that carry the design:

- **The asymmetry is the budget.** A failure costs 1 token; a success earns 0.1. Steady state therefore holds at a **10 % failure ratio** — ten successes pay for one failure. Push the failure rate above that and the bucket drains; below it, the bucket refills to `maxTokens` and retries are unrestricted. `tokenRatio` *is* the retry budget expressed as a ratio.
- **Only "real" failures count.** The spec is explicit that only statuses that qualify as retryable (or a server pushback saying "don't retry") decrement the bucket. `INVALID_ARGUMENT` from a malformed request does not, because a client bug should not disable retries for everyone sharing that channel.
- **Draining is fail-fast, not fail-slow.** Once throttled, retries are cancelled outright. A budget that *delayed* retries would just move the queue, which is lesson 05's "defer doesn't work" in a new costume.

**Envoy's retry budget** solves the same problem in the concurrency domain rather than the rate domain: `circuit_breakers.thresholds[].retry_budget.budget_percent` defaults to **20 %**, meaning "concurrent retries may not exceed 20 % of (active requests + active pending requests)". When set, it overrides the older fixed `max_retries` threshold (whose own default is 3 concurrent retries per priority). Same idea, different unit: gRPC bounds retries as a fraction of *completed calls over time*, Envoy as a fraction of *in-flight work right now*.

**Pick your number like this.** Budget = the fraction of extra load your dependency can absorb without degrading. If the dependency runs at 60 % utilisation, a 10 % retry budget takes it to 66 % — fine. A retry storm without a budget takes it to 27× — not fine. The industry-common 10 % is not magic; it is "a rounding error on capacity" for typical utilisation targets. If you run your GPU scheduler's etcd at 80 % of its fsync budget, a 10 % budget is already 88 % and you should pick 5 %.

### 8 · Circuit breakers: the real state machines

A circuit breaker is a **stateful, per-dependency cache of the answer to "is this call worth attempting?"** Its job is not to make failures rarer; it is to make failures *cheap and fast* so that (a) callers stop holding resources during a long timeout, and (b) the failing dependency gets a chance to drain.

The three states are always the same; the interesting engineering is entirely in the transition rules.

```
                       CIRCUIT BREAKER STATE MACHINE
                       (transition labels = actual defaults)

              failure statistics over the sliding window
              exceed the threshold
      ┌────────────────────────────────────────────────────────┐
      │                                                        ▼
 ┌────┴─────┐                                             ┌──────────┐
 │  CLOSED  │                                             │   OPEN   │
 │          │                                             │          │
 │ calls    │                                             │ calls    │
 │ pass     │                                             │ fail     │
 │ through; │                                             │ instantly│
 │ outcomes │                                             │ (no      │
 │ recorded │                                             │ network  │
 └────▲─────┘                                             │ I/O)     │
      │                                                   └────┬─────┘
      │ probe(s) succeeded                                     │ wait duration
      │                                                        │ elapsed
      │              ┌───────────────┐                         │
      └──────────────┤  HALF-OPEN    │◀────────────────────────┘
                     │               │
                     │ a LIMITED     │────────────────┐
                     │ number of     │  probe(s)      │
                     │ probe calls   │  failed        │
                     │ allowed       │                ▼
                     └───────────────┘        back to OPEN, wait
                                              window restarts

   ── Hystrix (1.5.x) ──────────────────────────────────────────────
   window:      10 s rolling, 10 buckets of 1 s
   trip when:   ≥ 20 requests in the window  AND  error % ≥ 50
   OPEN wait:   5,000 ms  (circuitBreakerSleepWindowInMilliseconds)
   HALF-OPEN:   EXACTLY ONE probe call is admitted; all others rejected
                success → CLOSED *and the metrics stream is reset to zero*
                failure → OPEN, sleep window restarts from now

   ── resilience4j (master) ────────────────────────────────────────
   window:      COUNT_BASED, slidingWindowSize = 100 calls
   arm when:    minimumNumberOfCalls = 100 recorded
   trip when:   failureRateThreshold ≥ 50 %   OR
                slowCallRateThreshold ≥ 100 % of calls slower than
                slowCallDurationThreshold = 60 s
   OPEN wait:   60 s (waitDurationInOpenState)
   HALF-OPEN:   permittedNumberOfCallsInHalfOpenState = 10 probes;
                the SAME failure-rate threshold is evaluated over those 10
                → ≥ 5 failures re-opens, otherwise CLOSED
   ── Envoy outlier detection (per upstream HOST, not per dependency) ──
   trip when:   5 consecutive 5xx, or success rate < mean − 1.9σ
   OPEN wait:   base_ejection_time 30 s × (times ejected), cap 300 s
   HALF-OPEN:   implicit — the host is simply returned to the pool at the
                next 10 s sweep and judged on live traffic
   SAFETY:      max_ejection_percent 10 % of the cluster, always
```

The comparison that matters for design:

| Dimension | Hystrix | resilience4j | Envoy outlier detection |
|---|---|---|---|
| Unit of protection | logical command | logical call | one upstream host |
| Window | time (10 s) | count (100 calls) or time | interval sweep (10 s) |
| Minimum volume | 20 requests | 100 calls | 100 req/host (statistical mode) |
| Half-open probes | 1 | 10 | whole host, live traffic |
| Blast-radius cap | none | none | **10 % of cluster** |
| Slow-call trip | via 1 s execution timeout | explicit `slowCallRate` | via 5xx only |

**Three failure modes of breakers, with mechanisms:**

1. **The breaker converts a partial outage into a total one.** This is Marc Brooker's objection, and the mechanism is precise: suppose 20 % of a dependency's shards are down. Every caller sees ~20 % errors. With `errorThresholdPercentage = 50` you are safe — but set it to 15 %, or let the failing shard be hot enough to push the observed rate over 50 %, and the breaker opens for *all* traffic, including the 80 % that was working. You have taken a 20 %-degraded system to 100 % down for everyone behind that breaker. Continuous mechanisms — load shedding, retry budgets, per-host ejection — degrade proportionally; a breaker is a step function.
2. **The mode is never tested.** OPEN and HALF-OPEN are code paths that run only during incidents. The fallback behind an open breaker is usually the least-exercised branch in the service. Brooker's broader point about fallbacks applies: a fallback path that has not carried production traffic is not known to work.
3. **Synchronised half-open probing.** With Hystrix's single probe there is at most one in-flight test per breaker instance — but a fleet of 500 pods all opened their breakers within the same second and will all probe 5 s later, simultaneously. The recovering dependency gets a 500-request spike precisely when it is least able to take it. **The fix is jitter on the sleep window**, which neither Hystrix nor resilience4j does by default. If you deploy breakers at fleet scale, randomise `waitDurationInOpenState` per instance (e.g. `60 s × Uniform(0.5, 1.5)`).

**The staff position:** prefer *per-host* ejection with a blast-radius cap (Envoy-style) plus a retry budget over a *per-dependency* breaker, because both degrade continuously and both have a bound on how much damage a wrong decision does. Reach for a classic breaker when the failure is genuinely all-or-nothing — a dependency that is either up or down, with an expensive timeout — and then jitter the sleep window and rehearse the open path.

### 9 · Metastable failure: why the system stays down after you fix the trigger

Lesson 05 introduced the term. Here is the state machine that makes it precise, because "the retries kept it down" is a description, not a mechanism.

A metastable failure needs three ingredients (Bronson et al., HotOS '21, and the follow-up empirical study, Huang et al., OSDI '22):

- a **trigger** — the spike, deploy, or dependency blip that first pushes load past capacity;
- an **amplification mechanism** — retries, a full queue, a cache that started missing, a re-connect storm — that converts *degraded service* back into *increased load*;
- a **sustaining effect** — the amplified load alone is enough to keep the system degraded, with the trigger gone.

The consequence is that the system has **two stable states and hysteresis**: it enters the bad state at offered load `L_trigger`, but it will not leave the bad state until offered load drops to `L_recover`, and `L_recover < L_trigger` — often far below it. That gap is why "traffic is back to normal, why are we still down?" is such a common incident-channel message.

```
      METASTABLE FAILURE — the loop, and the only two exits

                    ┌──────────────────────────────────┐
                    │      offered load  L             │
                    │   (client demand + retries)      │
                    └───────────────┬──────────────────┘
                                    │  L > capacity
                                    ▼
                    ┌──────────────────────────────────┐
                    │  queues fill · service time ↑    │
                    │  goodput ↓ (work completes after │
                    │  the caller already gave up)     │
                    └───────────────┬──────────────────┘
                                    │
                                    ▼
                    ┌──────────────────────────────────┐
                    │  callers hit their timeouts       │
                    └───────────────┬──────────────────┘
                                    │
                                    ▼
                    ┌──────────────────────────────────┐
                    │  each timeout emits a RETRY       │
                    │  ⇒ offered load grows            │──┐
                    └──────────────────────────────────┘  │
                                    ▲                     │
                                    └─────────────────────┘
                                    THE LOOP IS SELF-FEEDING:
                                    removing the trigger does nothing

    ── EXIT 1 · cut the load below L_recover ───────────────────────
       shed aggressively at the edge (reject > 90 % if needed),
       cancel in-flight retries, drop the queue backlog, THEN ramp
       admission back up slowly. You must go *below* L_recover, not
       back to normal — hysteresis means normal load re-enters the loop.

    ── EXIT 2 · break the amplifier ────────────────────────────────
       stop the mechanism that converts degradation into load:
       disable retries fleet-wide (or drain the retry budget), bound
       the queue and drop instead of buffering, restart components
       whose in-memory state (connection storms, cold caches) is the
       amplifier. This raises L_recover back toward L_trigger.

    Note what is NOT an exit: adding capacity. Capacity raises the
    trigger threshold for NEXT time; it does not break a loop that is
    already amplifying, because the amplifier scales with the new
    capacity too.
```

The OSDI '22 empirical study is the reason to treat this as an operating condition rather than a curiosity: analysing public incident reports, the authors found **at least 4 of the last 15 major AWS outages** followed the metastable pattern, and their in-depth study covers **22 metastable incidents across 11 different organizations**.

**The design implication is a runbook item, not just a diagram.** For every service you own, you should be able to answer: *what is my amplifier, and what is the switch that turns it off?* A fleet-wide retry kill-switch, a queue-drain command, and an admission-control dial that can go to 99 % rejection are the three controls that let you take exit 1 or exit 2 in minutes rather than hours.

### 10 · Gray failure and differential observability

Everything so far assumed the failure is at least *visible* to somebody. Gray failure is the class where it isn't.

Huang et al. (HotOS '17) define a **gray failure** as a fault where the system's own failure detectors do not observe a problem that applications are nonetheless suffering. Their model has three parties:

```
        ┌──────────────┐                 ┌──────────────┐
        │  APPLICATION │                 │   OBSERVER   │
        │  (client)    │                 │  (the health │
        │              │                 │   checker)   │
        └──────┬───────┘                 └───────┬──────┘
               │  experiences                    │  measures
               │  "this is broken"               │  "this is fine"
               ▼                                 ▼
        ╔══════════════════════════════════════════════════╗
        ║              SYSTEM under a fault                ║
        ║   (NIC dropping 2% of packets; HBM cell flipping ║
        ║    under a specific access pattern; a disk at    ║
        ║    10x normal latency but still answering)       ║
        ╚══════════════════════════════════════════════════╝

        DIFFERENTIAL OBSERVABILITY = app's view ≠ observer's view.
        The gap is not a monitoring bug. It is structural: the observer
        was built from the SERVER's vantage, and asks a cheaper question
        than the application does.
```

The reason this is the organizing concept for a GPU fleet rather than a curiosity: **almost all standard monitoring is server-vantage and status-shaped.** `nvidia-smi` reads status registers. A kubelet heartbeat proves the kubelet's goroutine is scheduled. A `readinessProbe` on `/healthz` proves an HTTP handler returned 200. None of them execute the workload's actual computation, and none of them measure from where the workload sits.

Two structural fixes:

1. **Move the vantage point.** Measure from where the client is — synthetic probes issued by a sidecar on a *different* node, per-caller success rates aggregated by *callee* (Envoy's success-rate outlier detection is exactly this, cheaply), end-to-end request tracing that attributes latency to a host rather than a service.
2. **Change the question from status to result.** Run an actual computation with a known answer and check both the answer and the time it took. A status register can be green while the thing it represents is wrong; a matrix multiply that returns the wrong product cannot lie.

### 11 · Why GPUs fail silently: the physical mechanisms

Gray failure on a GPU fleet has specific physical causes, and naming them is what lets you design the right probe.

- **Marginal HBM cells.** At the density and clock rates of modern HBM stacks, individual cells sit close to the edge of reliably retaining charge. Under a particular combination of voltage droop, temperature, and access pattern (refresh timing interacting with a hot row), a cell returns the wrong value. ECC catches single-bit errors and reports them — that path is *not* silent, and you should be scraping `DCGM_FI_DEV_ECC_SBE_VOL_TOTAL` / `DCGM_FI_DEV_ECC_DBE_VOL_TOTAL`. The silent path is corruption that ECC does not cover: errors in logic and datapath rather than in protected memory arrays.
- **Single-event upsets.** High-energy particles flip bits in memory or in combinational logic. Per-device the rate is tiny; multiply by tens of thousands of devices running continuously and it stops being negligible. This is the classic argument for why fleet-scale reliability engineering differs in kind, not degree, from single-machine reliability.
- **Marginal silicon under thermal and power stress.** A part that passed factory test can drift as it ages and as it runs sustained near its power limit. The result is not a crash but an occasionally-wrong arithmetic result — the defect is data-dependent and often reproducible only under specific input patterns, which is precisely why it survives manufacturing test and every subsequent health check.

None of these raise an exception. They produce **a wrong number**, silently folded into a gradient or an activation. The detection strategy that follows is active and numerical:

- **Deterministic replay.** Re-run a suspect step with fixed seeds and deterministic kernels on different hardware; compare bitwise. Google's published account of SDC at Gemini training scale describes exactly this, plus proactive SDC scanners run on otherwise-idle machines, with SDC events observed roughly **every one to two weeks at ~10,000-chip scale**.
- **In-band invariants.** Gradient-norm and loss spike detectors, checksummed all-reduce results, periodic self-consistency checks on a known input. Cheap, and they catch the corruption that matters (the one that reaches the model) rather than every corruption.
- **Out-of-band burn-in.** A short numerical test suite run on every node before it re-enters the schedulable pool after any maintenance event.

### 12 · Correlated failure, and the availability arithmetic that exposes it

The redundancy intuition — "two replicas, so unavailability squares" — is arithmetic that assumes independence. Here is what the arithmetic actually says, and where independence goes.

**Serial composition (a dependency chain).** If your request must traverse *n* components, each available `A_i`, then availability is the product:

```
A_total = ∏ A_i

10 dependencies, each 99.9 %:
    A = 0.999^10 = 0.99005
    unavailability = 0.995 %  →  0.00995 × 8,760 h/yr ≈ 87 h/yr ≈ 3.6 days
```

Ten "three nines" dependencies compose to barely two nines. **This is the single most useful piece of availability arithmetic in a design interview**, because it immediately kills the "I'll just add another service" reflex and forces the question of which dependencies are on the critical path versus which can be made soft (cached, defaulted, skipped).

**Parallel composition (redundancy), assuming independence.** *k* replicas each with unavailability `u`:

```
u_total = u^k

2 replicas at 99.9 %  →  u = 10⁻³,  u_total = 10⁻⁶  →  99.9999 %
```

**Parallel composition with correlation.** Now split each replica's unavailability into an independent part and a common-mode part: `u = u_ind + u_com`, where `u_com` is the share caused by something both replicas share (same rack, same PDU, same top-of-rack switch, same NIC firmware, same config push, same certificate expiry). Common-mode failures don't multiply — they hit together:

```
u_total ≈ u_ind^k + u_com

Same 2 replicas, u = 10⁻³, with just 1 % of that being common-mode:
    u_ind = 0.99 × 10⁻³,   u_com = 10⁻⁵
    u_total ≈ (0.99×10⁻³)² + 10⁻⁵ = 9.8×10⁻⁷ + 10⁻⁵ ≈ 1.1×10⁻⁵
                                                      → 99.9989 %
```

**A 1 % common-mode share made the redundant pair 11× less available than the independence model predicts**, and no amount of adding replicas fixes it — `u_com` is a floor. This is the number to reach for whenever someone proposes N+1 within a single failure domain. The design question is never "how many replicas" but "what do these replicas share, and what is the residual `u_com` after I remove it?"

**Gang-scheduled jobs are the degenerate case.** A 1,024-GPU training job is all-or-nothing: any one GPU failing kills the job. That is serial composition over 1,024 components, and it means job-level MTBF is `MTBF_gpu / N` — redundancy has nothing to protect, because there is no request to reroute. Worked in the example section below.

**Blast-radius controls: cells and shuffle sharding.** If you cannot make failures independent, bound how many tenants each one reaches.

*Cells*: partition the fleet into *m* independent stacks with no shared control plane, and pin each tenant to one. A failure takes out `1/m` of tenants. Simple, and the cost is `m` copies of every operational burden.

*Shuffle sharding*: give each tenant a random subset of *k* workers out of *n*. Two tenants collide completely only if they drew the same subset. The arithmetic is a binomial coefficient and it is startlingly good:

```
n = 100 workers, k = 5 per tenant
    distinct shards = C(100,5) = 100·99·98·97·96 / 120 = 75,287,520

    P(a given other tenant shares ALL 5 of your workers) = 1 / 75,287,520

n = 8 workers, k = 2 per tenant       (the small-fleet intuition)
    distinct shards = C(8,2) = 28
    P(full overlap with a given tenant) = 1/28 ≈ 3.6 %
    P(at least one shared worker) = 1 − C(6,2)/C(8,2) = 1 − 15/28 ≈ 46 %
```

Note what the numbers say: *partial* overlap is common (46 % in the small case), *total* overlap is rare. Shuffle sharding therefore does not prevent a bad tenant from touching you — it prevents a bad tenant from being able to take you fully down, provided your client retries across its own shard members. That proviso is load-bearing: shuffle sharding without client-side retry across the shard buys you nothing.

### 13 · Checkpoint economics: deriving the interval instead of quoting it

Given that failures on a large fleet are frequent, correlated, and partly undetectable in real time, the practical lever for a long training job is not "prevent failure" but **bound the cost of failure**. That is a one-variable optimisation, and you should be able to derive it on a whiteboard.

Set up the cost. Let:

- `δ` = wall-clock cost of writing one checkpoint (hours),
- `M` = mean time between job-level interruptions (hours),
- `τ` = the checkpoint interval you are choosing (hours).

Over a long run, two things waste time, and only two:

1. **Checkpoint overhead.** You pay `δ` every `τ` hours → wasted fraction `δ/τ`. Decreasing in `τ`.
2. **Rework after a failure.** A failure lands uniformly at random within an interval, so on average you lose half an interval, `τ/2`, and failures arrive at rate `1/M` → wasted fraction `(τ/2)/M = τ/(2M)`. Increasing in `τ`.

Total wasted fraction:

```
f(τ) = δ/τ + τ/(2M)

df/dτ = −δ/τ² + 1/(2M) = 0
      ⇒ τ² = 2δM
      ⇒ τ_opt = √(2·δ·M)                          ← Young (1974)

and substituting back:

f(τ_opt) = δ/√(2δM) + √(2δM)/(2M)
         = √(δ/(2M)) + √(δ/(2M))
         = 2·√(δ/(2M)) = √(2δ/M)                  ← the waste AT the optimum
```

Two facts fall out that are worth more than the formula:

- **At the optimum, the two costs are exactly equal.** Checkpoint overhead equals expected rework, each contributing `√(δ/(2M))`. If your monitoring says you spend 3 % of wall-clock writing checkpoints and lose 12 % to rework, you are checkpointing too rarely — and you can see that without recomputing anything.
- **The waste at the optimum is `√(2δ/M)`, so it improves with the *square root* of checkpoint cost.** Halving `δ` (async, sharded, or overlapped checkpointing) reduces waste by only 1/√2 ≈ 29 %, not 50 %. That tempers how much engineering to spend on faster checkpoint writes, and it is why elasticity — changing `M` and the failure's *scope* — is the complementary lever rather than a redundant one.

Daly (2006) refines the model to account for restart/recovery time and for failures that strike during the checkpoint write itself; in the regime `δ ≪ M` — which is where you want to be operating anyway — the correction is small and of order `δ`, so the Young form is the one to carry in your head. Use Daly's when `δ` approaches a meaningful fraction of `M`, which is the signal that your checkpoint path itself is the problem.

**Elasticity is a different term in the same equation.** Checkpointing bounds the *rework per failure*. Elastic training bounds the *scope of a failure*: a localised GPU or node failure drops those units out of the job and the rest continues at reduced scale, rather than the whole gang halting. Google's Gemini-scale account describes continuing at roughly **97 % of prior throughput** on fewer chips after a localised failure instead of a full restart. In the cost model, that converts a full `τ/2` rework event into a small throughput haircut — it attacks the `τ/(2M)` term without touching `δ`. The two levers stack.

## Perspectives

**The developer's view.** Every timeout, retry, and breaker in your code is a guess about a distribution you probably have not measured. The highest-value change most services can make is to *measure the healthy latency distribution of each dependency* and set the timeout from it (p99.9 + margin) instead of from a framework default — then make sure exactly one layer owns the retry. If you cannot say which layer that is, you have an amplification factor you have not computed.

**The operator's view.** You need three switches you can throw during an incident, and you need to have thrown them in a game day: a fleet-wide retry kill-switch, an admission-control dial that reaches 99 % rejection, and a queue-drain. Metastability means recovery is an *action*, not a wait; without those switches your only exit is a rolling restart of everything.

**The hardware view.** SDC on GPUs is not a software bug waiting to be patched; it traces to physical phenomena at the edge of what current silicon and memory density can guarantee. That is why detection has to be active and numerical rather than passive and status-based — and why the fleet needs a *quarantine* workflow: a suspect node is cordoned and burned in, not rebooted and returned.

**The economics view.** Young/Daly turns "how resilient should this job be" into a cost minimisation with a closed-form answer. On the worked numbers below, checkpointing costs ~10 % of a large training run's compute — a seven-figure annual line item on a 1,024×H100 job, which is what makes async checkpointing and elastic recovery fundable projects. Equally, the `√` in `√(2δ/M)` is a budgeting fact: halving checkpoint time buys ~29 % of the waste back, not 50 %.

**The theory view.** FLP is a mathematical fact about asynchronous systems with one faulty process, not an engineering gap. Internalising it changes the question from "how do we build a perfect detector" to "what false-positive rate can we afford, and what detection latency can we tolerate" — the answerable question, and the one Chandra–Toueg's completeness/accuracy pair lets you answer precisely.

## Real-world use cases

- **Google, "Training in Turmoil: Silent Data Corruption in Systems at Scale"** — <https://research.google/pubs/training-in-turmoil-silent-data-corruption-in-systems-at-scale/> — Google's own account of SDC at Gemini training scale: SDC events observed roughly **every one to two weeks** at ~10,000-chip scale, detected via **deterministic-replay** training (rerun the suspect step deterministically and compare) plus **proactive SDC scanners** run on otherwise-idle machines. Also describes **elastic** recovery continuing at roughly 97 % of prior throughput on fewer chips after a localized failure, rather than a full restart. Lead with this one alongside the Llama 3 paper below.
- **Meta, "The Llama 3 Herd of Models"** — <https://arxiv.org/abs/2407.21783> — the primary source for the SDC statistics this lesson cites: over a 54-day, ~16K-H100 training run, Meta recorded **466 total job interruptions** (419 unexpected, 47 planned), sustained **greater than 90 % effective training time**, and attributed roughly **78 % of unexpected interruptions** to confirmed or suspected hardware issues — GPU-related issues alone accounted for the majority of those. Six interruptions were specifically attributed to silent data corruption. What it shows for this lesson: the job-level MTBF arithmetic in §12 is not a pessimistic model, it is what a real 16K-GPU run measures.
- **Slack, "Slack's Incident on 2-22-22"** — <https://slack.engineering/slacks-incident-on-2-22-22/> — a load spike on a Vitess keyspace triggered a retry storm that sustained the overload. The decisive detail for §5–§7: Slack's clients were **already using exponential backoff with jitter** — the textbook-correct per-client pattern — and it was not sufficient, because per-client correctness says nothing about the aggregate. Recovery was "shed, don't defer": throttle new connections at the edge, fix the load-amplification source, then let traffic back in. This is the empirical case for retry *budgets* over retry *backoff*.
- **Huang, L. et al., "Metastable Failures in the Wild" (OSDI '22)** — <https://www.usenix.org/conference/osdi22/presentation/huang-lexiang> — the empirical grounding for §9: **at least 4 of the last 15 major AWS outages** analysed followed the metastable pattern, and the study covers **22 metastable incidents across 11 organizations**. What it shows: the two-threshold hysteresis model is a description of how large systems actually fail, not a theoretical construct.
- **Roblox, "Roblox Return to Service 10/28–10/31 2021"** — <https://blog.roblox.com/2022/01/roblox-return-to-service-10-28-10-31-2021/> — a 73-hour outage whose root cause sat deep inside Consul's use of BoltDB under a specific load pattern, and whose diagnosis was made dramatically harder because the monitoring needed to see the problem depended on the failing system. What it shows: differential observability applied to the *operator's* vantage — when the observer shares a failure domain with the system, you lose the vantage exactly when you need it.
- **PagerDuty, "August 28 Kafka Outages – What Happened and How We're Improving"** — <https://www.pagerduty.com/eng/august-28-kafka-outages-what-happened-and-how-were-improving/> — a bug creating a new Kafka producer per API request flooded the broker fleet; observability gaps delayed root-cause identification, and the recovery itself produced **duplicate webhooks** as the stale backlog drained. What it shows: the recovery path of a cascading failure is where delivery-semantics bugs surface — the direct bridge into lesson 07.

## Worked example

Run the whole chain on one system: a 1,024-GPU training job on a fleet whose control plane you also own.

### Step 1 — Job-level MTBF

```
GPUs in the gang            N       = 1,024
Per-GPU MTBF (vendor/fleet) M_gpu   = 20,000 h
Gang semantics: any single GPU failure kills the job (serial composition)

    M_job = M_gpu / N = 20,000 / 1,024 = 19.5 h
```

Now add the failures that are not clean GPU crashes. Take the Llama 3 measured split as the calibration: hardware issues account for ~78 % of unexpected interruptions, so non-GPU-hardware causes (host, network, storage, software) add roughly another 28 % on top of the GPU term (`0.78 → 1.0` implies dividing by 0.78, a 1.28× interruption rate). And SDC — the interruptions that require a *replay* to even detect — adds a small but non-zero rate at this scale.

```
    interruption rate  λ = 1/19.5 × 1.28 ≈ 0.0656 /h
    M_eff = 1/λ ≈ 15.2 h        →  call it 15 h
```

**Sanity-check against reality before believing it.** Meta's 16K-GPU run had 419 unexpected interruptions in 54 days = 419/1,296 h → one every ~3.1 h at 16× the GPU count. Scaling that to 1,024 GPUs (interruption rate is roughly linear in N) gives ~3.1 × 16 = ~50 h between interruptions — *better* than our 15 h estimate. The gap is the point: a vendor per-GPU MTBF of 20,000 h is conservative relative to what a well-run fleet achieves, and the estimate should be stated as a range. **Use `M_eff` = 15 h for the pessimistic plan and 50 h for the optimistic one, and show both.** That range-with-provenance is the staff move; a single confident number is not.

### Step 2 — Optimal checkpoint interval and the waste it implies

Take `δ` = 5 min = 0.083 h (a synchronous, full-state checkpoint to a shared store).

```
τ_opt = √(2·δ·M)

 pessimistic  M = 15 h :  τ = √(2 × 0.083 × 15)  = √2.50  = 1.58 h  (≈ 95 min)
 optimistic   M = 50 h :  τ = √(2 × 0.083 × 50)  = √8.30  = 2.88 h  (≈ 173 min)

waste  f = √(2δ/M)

 M = 15 h :  f = √(0.167/15) = √0.0111 = 10.5 %
 M = 50 h :  f = √(0.167/50) = √0.0033 =  5.8 %
```

Decompose the 10.5 % to check the "two halves are equal" property: overhead `δ/τ = 0.083/1.58 = 5.3 %`, rework `τ/(2M) = 1.58/30 = 5.3 %`. Equal, as derived. If your dashboard shows them unequal, your `τ` is wrong and the direction of the inequality tells you which way to move it.

### Step 3 — Sensitivity: what does faster checkpointing actually buy?

```
 M = 15 h fixed; vary δ

 δ = 5 min (0.083 h)   τ_opt = 1.58 h   waste = 10.5 %
 δ = 2 min (0.033 h)   τ_opt = 1.00 h   waste =  6.7 %
 δ = 1 min (0.017 h)   τ_opt = 0.71 h   waste =  4.7 %
 δ = 20 s  (0.0056 h)  τ_opt = 0.41 h   waste =  2.7 %

 5 min → 1 min is a 5× engineering win on δ and a 2.24× win on waste (√5).
```

Costed on a 1,024-GPU H100 fleet at an indicative $2/GPU-hour rental: 10.5 % of 1,024 GPU-hours/hour ≈ 107 wasted GPU-hours per wall-clock hour ≈ **$215/hour, ≈ $1.9 M/year**. Getting `δ` from 5 min to 1 min recovers about 59 GPU-hours/hour ≈ $118/hour ≈ **$1.0 M/year**. That is the business case for asynchronous, sharded checkpointing, stated in the unit the funding decision is made in.

### Step 4 — Retry amplification on the control plane serving this job

The training job's launcher, the scheduler, and the node agents all talk to the same metadata store. Compute the exposure:

```
 base control-plane request rate            R  = 1,000 req/s
 tiers that currently retry                 d  = 3 (SDK, gateway, scheduler client)
 attempts each                              a  = 3

 worst-case bottom-tier load = R · a^d = 1,000 × 27 = 27,000 req/s
 measured metadata-store capacity            = 12,000 req/s
 ⇒ 2.25× over capacity from retries alone, with zero increase in user demand

 After the fix — retries only in the gateway, 10 % budget (gRPC
 retryThrottling maxTokens=10, tokenRatio=0.1):

 bottom-tier load = R × 1.10 = 1,100 req/s   →  9 % of capacity
```

The intermediate case is worth showing too, because it is the one teams actually ship: keeping retries at all three tiers but budgeting each at 10 % gives `1.1³ = 1.331` → 1,331 req/s. Still fine. **Budgets are robust to the organisational reality that you will not successfully remove every retry.**

### Step 5 — The client-vantage health check that catches what the above misses

None of the above detects a node whose NIC drops 2 % of packets or whose GPU silently miscomputes. Design the probe explicitly:

| Design element | Choice | Why |
|---|---|---|
| **Probe workload** | A small fixed-shape NCCL all-reduce (e.g. 256 MB, ring across the node's 8 GPUs and out over the fabric to one peer node) with a deterministic input | Exercises HBM, SMs, NVLink, the NIC, and the fabric — the whole path the training job uses — instead of a status register. |
| **Assertions** | (1) result matches the precomputed expected tensor **bitwise**; (2) achieved bus bandwidth ≥ 95 % of the fleet's rolling median for the same shape | (1) catches SDC; (2) catches gray degradation. Bandwidth is compared to *peers*, not to a constant — the same differential logic as Envoy's success-rate ejection. |
| **Vantage** | Run by a DaemonSet, but the peer half of the collective runs on a *different* node, and the pass/fail verdict is computed by the peer | A node cannot be the sole judge of its own health; that is the server-vantage trap. |
| **Cadence** | Every 15 min on idle nodes; on every node at job boundaries; immediately on any node implicated by a training-loss spike | Cheap when idle, mandatory before a node re-enters a gang. |
| **Action** | Below bandwidth threshold → `cordon` + drain + queue for burn-in. Bitwise mismatch → `cordon` immediately, mark the node quarantined, page | Bandwidth degradation is a capacity problem; a bitwise mismatch is a correctness problem and must never silently re-enter the pool. |
| **Blast-radius cap** | Refuse to cordon more than 5 % of the fleet in any 10-minute window; beyond that, alert instead of acting | The §4 lesson: the checker must bound its own damage, because a fabric-wide event will fail every node's probe at once. |

**Why a status-read health check misses all of it:** `nvidia-smi` returns the GPU's own view of its own registers; a persistent XID error would show, but a marginal cell that flips under one access pattern will not, because nothing asked the GPU to *compute* anything. Likewise a link-state check on the NIC reports "up" for a NIC dropping 2 % of packets — link state and packet integrity are different questions, and only the second one is the one your all-reduce cares about.

## Practice

Feeds the [staff design portfolio](../practice/staff-design-portfolio/README.md). Produce one document with four parts:

**1 · The reliability calculator.** For a stated GPU count `N`, per-GPU MTBF, and checkpoint cost `δ`, compute job-level MTBF, `τ_opt = √(2δM)`, the waste `√(2δ/M)` decomposed into its overhead and rework halves, and the annual dollar cost at a stated $/GPU-hour. Add a sensitivity table over ≥ 4 values of `δ` and a second over an optimistic/pessimistic `M` range with the provenance of each bound. *Acceptance:* the two halves of the waste are equal at your `τ_opt`, and every number carries units.

**2 · The amplification audit.** For one real call path you own, enumerate every layer that retries (including SDKs and service meshes you did not write), state `a` for each, compute `a^d`, and compare it to the measured capacity of the bottom tier. Then specify the fix: which single layer keeps retries, what the budget is (a `maxTokens`/`tokenRatio` pair or an Envoy `budget_percent`), and the residual multiplier. *Acceptance:* a before/after multiplier with a capacity number next to it.

**3 · The breaker or the budget.** For one dependency, choose between a per-dependency circuit breaker and per-host ejection + retry budget. State actual threshold values (window size, minimum volume, trip threshold, open duration, half-open probe count, ejection cap), and the specific partial-outage scenario in which your choice degrades better than the rejected one. *Acceptance:* the rejected option is named and its failure mode is quantified.

**4 · The client-vantage probe design.** Following the table in the worked example, specify the probe workload, both assertions, where the verdict is computed, the cadence, the cordon/drain/quarantine actions, and the blast-radius cap on the checker itself. Add a paragraph on what a status-read health check would miss and why. *Acceptance:* the probe executes real compute and its verdict is rendered from a vantage other than the node under test.

## Common pitfalls

1. **"We just need a better timeout value."** *Symptom:* a recurring incident where healthy-but-slow nodes get killed, alternating with incidents where a dead node is not noticed for minutes. *Mechanism:* FLP — under asynchrony no value of `T` separates slow from dead, so tuning `T` slides you along the false-positive/detection-latency curve rather than off it. You choose a point deliberately (and let φ-accrual or a differential success-rate check adapt the point as conditions move); you do not find a value that has no cost.
2. **"We use exponential backoff, so we're safe from retry storms."** *Symptom:* a retry storm despite every client doing textbook backoff — Slack's 2-22-22 exactly. *Mechanism:* backoff bounds one client's rate; the aggregate is the sum over clients and is unbounded by any per-client rule. Only a budget (gRPC `retryThrottling`, Envoy `retry_budget`) caps the sum. And backoff *without jitter* does not even spread the herd — the AWS simulator re-run shows plain exponential taking 13× longer and issuing more calls than full jitter at 100 clients.
3. **"Health checks passing means the node is healthy."** *Symptom:* a job produces NaNs or throughput mysteriously 20 % below plan while every dashboard is green. *Mechanism:* gray failure — the observer's vantage and question differ from the application's. Status registers and liveness handlers answer "is the process scheduled", not "does this hardware compute correctly". Only an active, numerical, client-vantage probe closes the gap.
4. **"Deep health checks are more thorough, so they're better."** *Symptom:* a dependency degrades slightly and 100 % of your fleet leaves the load-balancer pool simultaneously. *Mechanism:* deep checks are perfectly correlated across instances — when the shared dependency wobbles, every instance fails at the same instant, and the health system's response is to remove all capacity. Cap it (`max_ejection_percent: 10`) and prefer differential checks (success rate vs. peer mean) over absolute ones.
5. **"Two replicas double our effective MTBF."** *Symptom:* a rack-level or config-push event takes out an ostensibly redundant pair. *Mechanism:* `u_total ≈ u_ind^k + u_com`; the common-mode term is a floor that replicas cannot cross. A 1 % common-mode share costs 11× the availability the independence model promised. And on a gang-scheduled job, redundancy protects nothing at all — the job is serial composition over `N` GPUs, so `M_job = M_gpu/N` regardless of spares.
6. **"Circuit breakers are strictly good hygiene."** *Symptom:* a 20 %-degraded dependency becomes 100 % unavailable to everything behind the breaker. *Mechanism:* a breaker is a step function on an aggregate error rate; it cannot distinguish "20 % of shards are down" from "the dependency is down", so a mis-set threshold converts partial into total. It also adds two code paths (OPEN, HALF-OPEN) that only execute during incidents and are therefore the least-tested branches you own. Prefer continuous mechanisms; if you use a breaker, jitter the sleep window so 500 pods don't probe in lockstep.
7. **"Traffic is back to normal, so we should recover shortly."** *Symptom:* the system stays saturated for hours after the trigger is gone. *Mechanism:* metastability — the amplifier (retries, a full queue, a cold cache) is now the load source, and the system has hysteresis: it will not leave the bad state until offered load falls below `L_recover`, which is well under the `L_trigger` that got it there. Exit by shedding hard below `L_recover`, or by disabling the amplifier. Adding capacity is not an exit.
8. **"More frequent checkpoints are always safer."** *Symptom:* a job spends 20 % of wall-clock writing checkpoints and still loses meaningful work. *Mechanism:* `f(τ) = δ/τ + τ/(2M)` has a genuine interior minimum; over-checkpointing wastes exactly as much as under-checkpointing, just through the `δ/τ` term instead of the `τ/(2M)` one. At the optimum the two halves are equal — measure both and let the inequality tell you which way to move.

## Self-check

- **Why can't a timeout distinguish a slow node from a dead one, and what does that force you to trade?** FLP: under asynchrony there is no bound separating "slow" from "crashed", so any timeout is a calibrated guess. It trades false positives (killing/failing over a healthy-but-slow node, which generates retry load precisely when the system is fragile) against detection latency (a dead node keeps absorbing requests, and every caller holds a slot for the full timeout — a 200-slot service with a 30 s timeout to a dead dependency has under 7 req/s of throughput left). Chandra–Toueg name the two sides as *accuracy* and *completeness*; φ-accrual re-parameterises the tradeoff into a false-conviction probability (φ = 8 ⇒ ~10⁻⁸) instead of a millisecond constant, but does not remove it.
- **Write the three jitter formulas and say which you would default to and why.** With `v = min(cap, base·2ⁿ)`: *full jitter* = `uniform(0, v)`; *equal jitter* = `v/2 + uniform(0, v/2)`; *decorrelated jitter* = `sleep = min(cap, uniform(base, sleep·3))`, carrying `sleep` as per-client state. Default to full jitter: re-running AWS's own simulator, at 100 contending clients it completes in 4,842 ms with 794 server calls versus plain exponential's 63,840 ms and 1,864 calls — 13× faster *and* 57 % less server work, because randomising the whole interval is what decorrelates a synchronised herd. Equal jitter buys a guaranteed minimum gap at ~35 % more completion time; decorrelated jitter is fastest at low contention but issues ~37 % more calls at high contention.
- **Three tiers each retry twice on failure. What is the load multiplier at the bottom, and what are the two fixes?** Attempts per tier `a = 3`, depth `d = 3` → `3³ = 27×`. 1,000 req/s of user demand becomes 27,000 req/s at the metadata store. Fix one: retry at exactly one layer (multiplier → 3×). Fix two: a retry budget capping retries at ~10 % of base volume (multiplier → 1.1×, or 1.1³ = 1.33× if you budget all three tiers and remove none). gRPC implements the budget as a token bucket: failures cost 1 token, successes earn `tokenRatio` (0.1), retries are permitted only while `token_count > maxTokens/2`, and blocked retries are cancelled rather than delayed.
- **Give the half-open behaviour of two real circuit breakers and say why the difference matters.** Hystrix admits **exactly one** probe after `circuitBreakerSleepWindowInMilliseconds = 5000`; success closes the breaker and resets the metrics stream to zero, failure re-opens it and restarts the window. resilience4j admits **`permittedNumberOfCallsInHalfOpenState = 10`** probes after `waitDurationInOpenState = 60 s` and applies the same `failureRateThreshold = 50 %` over those 10, so ≥ 5 failures re-open. One probe minimises the load put on a recovering dependency but makes the close/re-open decision on a single sample (a flaky recovery flaps); ten probes give a statistically meaningful verdict but put 10× the load on something that just came back. Neither jitters its wait duration, so a fleet of N instances that tripped together will probe together — add jitter yourself.
- **Draw the metastable loop and name both exits.** Load exceeds capacity → queues fill and latency rises → callers hit timeouts → each timeout emits a retry → load rises further. The loop is self-sustaining, so removing the trigger changes nothing, and the system has hysteresis (`L_recover < L_trigger`). Exit 1: cut offered load below `L_recover` — shed hard at the edge, cancel in-flight retries, drop the backlog, then ramp back slowly. Exit 2: break the amplifier — kill retries fleet-wide, bound and drain queues, restart the component whose in-memory state is doing the amplifying. Adding capacity is not an exit; the amplifier scales with capacity.
- **Ten dependencies at 99.9 %. What is your availability, and what does a 1 % common-mode share do to a redundant pair?** Serial: `0.999¹⁰ = 0.99005` → 99.0 %, ≈ 87 hours of downtime a year. That is the arithmetic that kills "just add one more service on the critical path". Parallel with independence: two replicas at `u = 10⁻³` give `u² = 10⁻⁶` (six nines). With 1 % of that unavailability common-mode: `u_total ≈ (0.99×10⁻³)² + 10⁻⁵ ≈ 1.1×10⁻⁵` — **11× worse than the independence model**, and adding replicas cannot go below the `u_com` floor. Find and remove the shared thing (rack, PDU, ToR, firmware version, config push, certificate) or the replicas are decoration.
- **Derive the optimal checkpoint interval and state what its two halves tell you operationally.** Waste `f(τ) = δ/τ + τ/(2M)`; `f'(τ) = −δ/τ² + 1/(2M) = 0` ⇒ `τ_opt = √(2δM)`, and `f(τ_opt) = √(2δ/M)`. At the optimum the overhead half and the rework half are exactly equal, each `√(δ/(2M))` — so if your dashboards show 3 % checkpoint overhead and 12 % rework, you are checkpointing too rarely, no recomputation needed. And because waste goes as `√δ`, halving checkpoint cost recovers only ~29 % of the waste, which is why elasticity (shrinking the *scope* of a failure, e.g. Google's ~97 %-throughput continuation) is a complementary lever rather than a redundant one.
- **What is gray failure, why does a normal health check miss it, and what specifically causes it on GPUs?** A fault the application suffers while the system's own detectors report healthy — differential observability, the gap between the observer's vantage/question and the application's. Status-shaped, server-vantage checks (`nvidia-smi` registers, kubelet heartbeats, `/healthz`) answer "is the process scheduled", not "does this hardware compute correctly". On GPUs the physical causes are marginal HBM cells that flip only under specific voltage/thermal/access-pattern conditions, single-event upsets that are negligible per-device and material across tens of thousands of devices, and aged silicon running near its power limit producing data-dependent arithmetic errors. None crash; all produce a wrong number. Detection must be active and numerical: deterministic replay, bitwise-checked collectives, in-band gradient-norm invariants, proactive scanners on idle machines.

## Connections & what's next

This lesson is the substrate under lesson 05: every shedding, retry, and failover decision there assumes you can detect overload or failure, and this lesson shows that detection is provably imperfect — the pair reads as "how to act under overload" and "how to know what is actually happening while you act". The metastable-loop diagram here is the mechanism behind lesson 05's Little's-Law ceiling, and the retry-budget arithmetic is the aggregate control that lesson 05's per-client backoff could not provide. It also picks up [Lesson 02](02-consensus-and-quorums.md)'s partial-synchrony assumption and shows you where the timeout it hides in actually lives, and [Lesson 03](03-replication-and-partitioning.md)'s replication math, which the correlated-failure arithmetic here corrects. Forward, it feeds [Lesson 08](08-system-design-method.md)'s "failure & operations" step directly — availability composition, blast-radius controls, and checkpoint economics are three of the numbers that step is supposed to produce. Next: **[07 · Data-intensive patterns](07-data-intensive-patterns.md)** — starting from exactly the seam this lesson leaves open: what happens to *correctness* when a retry or a recovery, taken under uncertainty about what actually failed, causes a message to be delivered more than once.

## References & further reading

**Primary sources — theory**

1. **Fischer, M., Lynch, N., Paterson, M. (1985), "Impossibility of Distributed Consensus with One Faulty Process,"** *Journal of the ACM* 32(2) — <https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf> — the FLP result. Read for the precise asynchrony assumptions the impossibility depends on; everything in §1 is downstream of them. *Not fetched from this environment (egress-restricted); the statement of the result used here is standard and load-bearing claims are limited to it.*
2. **Dwork, C., Lynch, N., Stockmeyer, L. (1988), "Consensus in the Presence of Partial Synchrony,"** *Journal of the ACM* 35(2) — the escape hatch every real system uses: bounds that hold eventually, after an unknown time. *Not fetched from this environment; cited for the model only.*
3. **Chandra, T., Toueg, S. (1996), "Unreliable Failure Detectors for Reliable Distributed Systems,"** *Journal of the ACM* 43(2) — <https://dl.acm.org/doi/10.1145/226643.226647> — the completeness/accuracy vocabulary in §2 and the `◇S` class. *Not fetched from this environment; cited for the definitions only.*
4. **Hayashibara, N., Défago, X., Yared, R., Katayama, T. (2004), "The φ Accrual Failure Detector,"** SRDS '04 — the φ formulation in §3. *Not fetched from this environment; the formula and behaviour described here were cross-checked against two independent open-source implementations (Cassandra, Akka — see below), which is the verification actually relied upon.*
5. **Young, J.W. (1974), "A First Order Approximation to the Optimum Checkpoint Interval,"** *Communications of the ACM* 17(9):530–531 — <https://cacm.acm.org/research/a-first-order-approximation-to-the-optimum-checkpoint-interval/> — the `√(2δM)` result. §13 derives it from scratch, so the paper is optional depth. *Not fetched from this environment.*
6. **Daly, J.T. (2006), "A higher order estimate of the optimum checkpoint interval for restart dumps"** — <https://www.usenix.org/legacy/events/fast09/tech/full_papers/daly/daly.pdf> — the refinement that accounts for restart time and for failures during the checkpoint write. Use it when `δ` is not small relative to `M`. *Not fetched from this environment.*
7. **Huang, P. et al. (2017), "Gray Failure: The Achilles' Heel of Cloud-Scale Systems,"** HotOS '17 — <https://www.microsoft.com/en-us/research/publication/gray-failure-achilles-heel-cloud-scale-systems/> — the paper that coined differential observability and the app/observer/system model reproduced in §10. *Publication page fetched and used.*
8. **Bronson, N. et al. (2021), "Metastable Failures in Distributed Systems,"** HotOS '21 — the trigger / amplification / sustaining-effect decomposition in §9. *Not fetched from this environment; the decomposition is corroborated by the OSDI '22 study below.*
9. **Huang, L. et al. (2022), "Metastable Failures in the Wild,"** OSDI '22 — <https://www.usenix.org/conference/osdi22/presentation/huang-lexiang> — source of the "≥ 4 of the last 15 major AWS outages" and "22 incidents across 11 organizations" figures. *Not fetched from this environment; figures carried forward from lesson 05, where they were verified.*

**Implementations — read from source, and the actual basis for the defaults in this lesson**

10. **`aws-samples/aws-arch-backoff-simulator`** — <https://github.com/aws-samples/aws-arch-backoff-simulator> — the simulator behind AWS's exponential-backoff-and-jitter post. `src/backoff_simulator.py` contains the four algorithms transcribed in §6 (`ExpoBackoff`, `ExpoBackoffEqualJitter`, `ExpoBackoffFullJitter`, `ExpoBackoffDecorr`) with `base = 5 ms`, `cap = 2000 ms`, `Net(10, 2)`, 100 trials per point. *Cloned and re-run in this environment (Python-3 port, no algorithmic change); the results table in §6 is that re-run, not a transcription of the blog's table, which was unreachable.*
11. **Netflix Hystrix** — <https://github.com/Netflix/Hystrix> — `HystrixCommandProperties.java` (rolling window 10,000 ms / 10 buckets; `circuitBreakerRequestVolumeThreshold = 20`; `circuitBreakerErrorThresholdPercentage = 50`; `circuitBreakerSleepWindowInMilliseconds = 5000`; `executionTimeoutInMilliseconds = 1000`) and `HystrixCircuitBreaker.java` (the single-probe half-open logic, `markSuccess`/`markNonSuccess`). *Cloned and read.* The project is archived; the mechanics remain the reference implementation of the pattern.
12. **resilience4j** — <https://github.com/resilience4j/resilience4j> — `CircuitBreakerConfig.java` (`DEFAULT_FAILURE_RATE_THRESHOLD = 50`, `DEFAULT_SLOW_CALL_RATE_THRESHOLD = 100`, `DEFAULT_WAIT_DURATION_IN_OPEN_STATE = 60 s`, `DEFAULT_PERMITTED_CALLS_IN_HALF_OPEN_STATE = 10`, `DEFAULT_MINIMUM_NUMBER_OF_CALLS = 100`, `DEFAULT_SLIDING_WINDOW_SIZE = 100`, `DEFAULT_SLOW_CALL_DURATION_THRESHOLD = 60 s`) and `internal/CircuitBreakerStateMachine.java` (the half-open permit counter). *Cloned and read.*
13. **Envoy** — <https://github.com/envoyproxy/envoy> — `api/envoy/config/cluster/v3/outlier_detection.proto` (all defaults in the §4 table, including `max_ejection_percent: 10` and `success_rate_stdev_factor: 1900`) and `circuit_breaker.proto` (`max_retries: 3`; `RetryBudget.budget_percent` default 20 %). *Cloned and read.* This is also the code path Istio's `outlierDetection` configures.
14. **gRPC gRFC A6, "Client Retries"** — <https://github.com/grpc/proposal/blob/master/A6-client-retries.md> — the `retryThrottling` token-bucket specification quoted in §7 (`maxTokens`, `tokenRatio`, threshold `maxTokens/2`, failure −1 / success +`tokenRatio`, throttled retries cancelled not delayed, only retryable statuses counted). *Fetched and read.*
15. **Kubernetes** — <https://github.com/kubernetes/kubernetes> — `pkg/kubelet/kubelet.go` (`initialCrashLoopBackOff = 10 s`, `MaxCrashLoopBackOff = v1beta1.MaxContainerBackOff`), `pkg/kubelet/apis/config/v1beta1/defaults.go` (`MaxContainerBackOff = 300 s`), and `staging/src/k8s.io/client-go/util/flowcontrol/backoff.go` (doubling with optional jitter factor; `NewBackOff` sets it to 0, so kubelet's restart backoff is unjittered; entries reset after `2 × maxDuration`). *Cloned and read.*
16. **Apache Cassandra `conf/cassandra.yaml`** — <https://github.com/apache/cassandra> — `phi_convict_threshold: 8`. *Fetched and read.*
17. **Akka `akka-cluster/src/main/resources/reference.conf`** — <https://github.com/akka/akka> — `PhiAccrualFailureDetector` with `threshold = 8.0`, `heartbeat-interval = 1 s`, `max-sample-size = 1000`, `min-std-deviation = 100 ms`, `acceptable-heartbeat-pause = 3 s`, `monitored-by-nr-of-members = 9`. *Fetched and read.* The clearest production instance of "buy accuracy with detection latency".

**Real-world engineering write-ups**

18. **Google, "Training in Turmoil: Silent Data Corruption in Systems at Scale"** — <https://research.google/pubs/training-in-turmoil-silent-data-corruption-in-systems-at-scale/> — weekly-scale SDC events at ~10,000 chips, deterministic-replay detection, proactive scanners, ~97 %-throughput elastic continuation. *Not fetched from this environment; figures carried forward from the previous revision of this lesson.*
19. **Meta, "The Llama 3 Herd of Models"** — <https://arxiv.org/abs/2407.21783> — 466 interruptions over 54 days at ~16K H100s, > 90 % effective training time, ~78 % hardware-attributed. *Not fetched from this environment (arXiv is egress-restricted); figures carried forward from the previous revision of this lesson — verify against the paper's infrastructure section before quoting externally.*
20. **Slack, "Slack's Incident on 2-22-22"** — <https://slack.engineering/slacks-incident-on-2-22-22/> — correct per-client backoff-with-jitter that still produced a sustained retry storm; the empirical case for aggregate budgets. *Not fetched from this environment; summary carried forward from lesson 05.*
21. **Roblox, "Roblox Return to Service 10/28–10/31 2021"** — <https://blog.roblox.com/2022/01/roblox-return-to-service-10-28-10-31-2021/> — a 73-hour outage where the observability needed to diagnose the failure shared a failure domain with it. *Not fetched from this environment; summary carried forward from the previous revision of this lesson.*
22. **PagerDuty, "August 28 Kafka Outages – What Happened and How We're Improving"** — <https://www.pagerduty.com/eng/august-28-kafka-outages-what-happened-and-how-were-improving/> — a producer-per-request bug that destabilised a broker fleet, and a recovery that emitted duplicate webhooks. The bridge into lesson 07. *Not fetched from this environment; summary carried forward from the previous revision of this lesson.*
