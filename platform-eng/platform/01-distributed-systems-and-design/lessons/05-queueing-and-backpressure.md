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
sources: 14
---

# A01.5 · Queueing and backpressure

> **Concept.** A queue converts rejection into latency; at a fixed concurrency limit you cannot queue your way out of overload — you can only shed.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 04 ended on a specific failure: an origin sized for 1% of traffic suddenly receiving 100% of it, unable to refill the cache that would relieve it, stuck. That is one instance of a general pattern, and this lesson is the general pattern.

Any system whose offered load can exceed its service capacity has this problem, whatever the trigger — a cache flush, a fault-restart storm, a submission burst, a slow disk under etcd (lesson 02) halving the write path's service rate, a hot shard (lesson 03) that salting did not fully absorb, or nothing more exotic than organic growth past a ceiling nobody computed. Queueing theory is the vocabulary for that whole class: what a queue can and cannot do, why shedding is the only sound response to sustained overload, and why a system can stay overloaded long after the thing that overloaded it has gone away.

The tools here apply to every queue in the fleet: the inference batcher, the admission path in front of a training data loader, the API server's request queues, the retry storm in front of etcd, and the job queue Kueue manages.

## Why this matters

"Add a queue to absorb the spike" is the most common instinct in system design and one of the most consequential. Sometimes it is right. When it is wrong, it is wrong in a way that converts a five-minute capacity problem into a forty-minute outage that adding capacity does not fix.

The staff bound is a single number: **the arrival rate at which queue-implied wait crosses the SLO.** Below it you serve; above it you must reject cheaply, or you amplify. Getting that number wrong is the difference between shedding 5% of traffic for ninety seconds and a retry-storm death spiral that survives the removal of its own trigger.

There is a second, quieter reason. The capacity number you are defending is probably wrong, because the load test that produced it almost certainly suffered coordinated omission — and it is wrong in the optimistic direction, systematically, and worst exactly during the stall you are trying to characterise.

## What's new here (calibration)

**Skip (you already know):** FIFO decouples producer from consumer; unbounded queues are a memory bomb; "add a buffer for bursts" is the reflex.

**New here:**
- **Little's Law as an identity that forecloses options**, not a formula to recite — plus the utilisation curve and Kingman's approximation, which together explain why "we're only at 80% CPU" is not the reassurance it sounds like, and why *variability* costs as much as *load*.
- **The overload response ranked by cost**, with the specific requirement that a rejection must be orders of magnitude cheaper than a service — including what happens when it is not.
- **Coordinated omission** with the arithmetic showing exactly how much your p99 is understated, and the open-loop fix.
- **Metastability as a state machine** with a sustaining loop, a trigger, and an explicit escape procedure — plus the retry-budget arithmetic that bounds the most common loop.
- **Two production instances you operate**: Kubernetes API Priority and Fairness (seats, queues, shuffle sharding, borrowing, and the choice between `Reject` and `Queue`) and Kueue's fair sharing (weighted dominant-resource share, borrowing, preemption strategies) — both read from their source documentation, not paraphrased.

Version note: Kubernetes APF details are from `kubernetes/website` (`content/en/docs/concepts/cluster-administration/flow-control.md` and the `kube-apiserver` reference) and Kueue details from `kubernetes-sigs/kueue` (`site/content/en/docs/concepts/`), read on GitHub in August 2026. Queueing results are standard and derived here rather than cited; the papers and engineering blogs were not fetchable in this environment and are marked in References.

## Core concepts

### Little's Law: an identity, not a design choice

**L = λ · W.** The average number of items in a system equals the average arrival rate times the average time each spends there.

Little proved this in 1961 for any system in steady state, with no assumptions about arrival distribution, service distribution, or scheduling discipline. It holds for a bank queue, a disk I/O queue, a GPU batcher and a Kubernetes admission path equally. The only requirement is stationarity — nothing drifting to infinity.

Its value is not that it computes things. It is that it **forbids** things:

```
   L = λ · W        ⇒        λ = L / W

   Now suppose L is CAPPED by construction — a fixed worker pool, a fixed
   number of batch slots, a fixed connection limit, a fixed seat count.
   Call the cap L_max, and let W_service be the time actually spent being
   served (as opposed to waiting).

        λ_max = L_max / W_service

   That is your throughput ceiling, and it contains no term you can improve
   by adding buffer. Push arrivals past λ_max and the identity still has to
   hold, but L cannot rise past L_max — so the slack goes into W:

        arrivals ↑ ,  L pinned at L_max  ⇒  W ↑ , λ pinned at λ_max

   A queue does not raise your ceiling. It decides WHOSE LATENCY absorbs
   the excess, and for how long, before the client gives up anyway.
```

Only two levers move `λ_max`: raise `L_max` (more concurrency — replicas, slots, connections, GPUs) or lower `W_service` (faster work per item — a better kernel, a smaller batch, a cheaper query). Queue depth is neither. **Every "we'll add a bigger buffer" proposal is a proposal to convert rejections into latency, and it should be evaluated as one.**

Worked, on a system you run:

```
  kube-apiserver default concurrency (with APF enabled) is the sum of
  --max-requests-inflight (400) and --max-mutating-requests-inflight (200):

      L_max = 600 seats

  If the average request occupies its seat for W_service = 20 ms:
      λ_max = 600 / 0.020 s = 30,000 requests/s

  If a badly-written controller starts issuing unpaginated LISTs of every pod
  in a 5,000-node cluster, and each takes 2 s while holding MULTIPLE seats
  (APF charges list requests seats proportional to the estimated object count):
      one such request at, say, 10 seats × 2 s consumes 20 seat-seconds —
      the same capacity as 1,000 ordinary 20 ms requests.

  A handful of those per second does not "slow down the API server." It
  DELETES most of its capacity, and everything else queues behind them.
```

### Utilisation is not a linear dial

Little's Law tells you the ceiling. It does not tell you how bad life gets as you approach it. For that, the single most useful result in applied queueing is the utilisation curve. For an M/M/1 queue (Poisson arrivals, exponential service, one server) with utilisation `ρ = λ/μ`:

```
   W_total = 1 / (μ − λ) = W_service / (1 − ρ)
   W_queue = W_service × ρ/(1 − ρ)

   ρ      1/(1−ρ)     total time in system, as a multiple of service time
   ────   ────────    ────────────────────────────────────────────────────
   0.50     2.0×      █▌
   0.70     3.3×      ██▌
   0.80     5.0×      ████
   0.90    10.0×      ████████
   0.95    20.0×      ████████████████
   0.99   100.0×      ████████████████████████████████████████████████████…

   Read the last two rows as an operational statement: going from 90% to 95%
   utilisation — which looks like "a bit more headroom used" on any dashboard
   — DOUBLES latency. Going from 95% to 99% quintuples it again.
```

Two immediate consequences. First, **there is no such thing as running a latency-sensitive system at 95% utilisation**; the standard operating point for anything with a tail SLO is 50–70%, and the "wasted" 30–50% is not waste, it is the latency budget. Second, **utilisation targets must be set per-resource**: a GPU fleet at 90% *allocation* may be at 99% on some other resource (the scheduler's write path, the image pull bandwidth, one hot shard), and that resource sets your tail.

Kingman's approximation extends this to realistic (non-Poisson) traffic and is the version worth memorising, because it contains the term the M/M/1 formula hides:

```
   E[W_queue]  ≈   ρ/(1−ρ)   ×   (c_a² + c_s²)/2   ×   E[S]
                   ─────────     ───────────────       ────
                   utilisation    VARIABILITY         service
                   term           term                time

     c_a = coefficient of variation of interarrival times (σ/mean)
     c_s = coefficient of variation of service times

   Perfectly regular arrivals and service (c_a = c_s = 0) ⇒ NO QUEUE AT ALL,
   at any utilisation below 1. All queueing is caused by variability.

   Worked: ρ = 0.8, E[S] = 20 ms.
     smooth traffic, uniform work  (c_a=0.5, c_s=0.5):
        4 × 0.25 × 20 ms = 20 ms of queueing
     Poisson arrivals, uniform work (c_a=1, c_s=0.5):
        4 × 0.625 × 20 ms = 50 ms
     Poisson arrivals, HIGHLY VARIABLE work (c_a=1, c_s=3 — e.g. a mix of
     20 ms point reads and 2 s full LISTs):
        4 × 5 × 20 ms = 400 ms of queueing at the SAME 80% utilisation.

   ⇒ Reducing variability is as powerful as reducing load. That is the formal
     justification for separating workload classes (small requests away from
     large ones), for capping request size, and for APF's entire design.
```

### The queue diagram, with the numbers on it

```
                      offered load λ                 service capacity μ·c
                            │                                │
   clients ──────▶ ┌────────▼─────────┐        ┌─────────────▼──────────────┐
                   │  ADMISSION       │        │  SERVICE                   │
                   │  (cheap: a few   │        │  c concurrent slots        │
                   │   µs per reject) │        │  W_service each            │
                   │                  │        │  ⇒ λ_max = c / W_service   │
                   │  ① SHED HERE     │        └────────────┬───────────────┘
                   └────────┬─────────┘                     │
                            │ admitted                      ▼ responses
                            ▼
                   ┌──────────────────────────┐
                   │        QUEUE  (bounded!)  │
                   │  depth q, wait = q/λ_max │
                   │  ② deadline check here   │
                   │  ③ LIFO here under load  │
                   └────────┬─────────────────┘
                            │
                            ▼ workers pull when a slot frees
                       ④ BACKPRESSURE flows UPSTREAM as "no credit"

   BACKLOG GROWTH when λ > λ_max:
       dq/dt = λ − λ_max                (a constant-slope ramp, not a curve)

       λ = 1.2 × λ_max , λ_max = 5,000/s  ⇒ dq/dt = 1,000 items/s
       queue budget at a 500 ms SLO with λ_max = 5,000/s:
           q_max = 0.5 s × 5,000/s = 2,500 items
       time to breach from empty:   2,500 ÷ 1,000 = 2.5 seconds.

   ⇒ A 20% overload eats a HALF-SECOND latency budget in under three seconds.
     Whatever your shed decision is, it has to be automatic. There is no
     version of this where a human is in the loop.

   WHERE TO SHED — ① is the only good answer:
     ① at admission        cost ≈ µs   ✓ cheapest, protects everything downstream
     ② in the queue        cost = the wait already paid, wasted
     ③ at the worker       cost = partial service, wasted
     ④ at the origin       cost = full service for a response nobody wants
   Shed at ①, and make sure ① cannot itself be overloaded (it must not
   authenticate, look up quota in a database, or log to a synchronous sink).
```

### Bounded queue plus shedding is the only stable overload response

Given the above, the only sound response to *sustained* overload is to refuse work, cheaply, as early as possible. Three requirements make it work, and each one is a real failure when violated:

1. **The queue must be bounded.** An unbounded queue is not a queue, it is a memory leak that also destroys latency: every item admitted past `q_max` is an item that will be served after its deadline and therefore is pure waste, plus the memory to hold it.
2. **The rejection must be orders of magnitude cheaper than the service.** If saying no costs 30% of saying yes, then at 3× overload your shed path alone consumes your entire capacity and you have merely changed which component falls over. This is not hypothetical: it is the mechanism behind several documented metastable outages, where the "error" path involved a database lookup, a full log line, or a synchronous metric write.
3. **The client must be told something it can act on.** `429` with `Retry-After` is actionable; a connection reset is not — it teaches clients nothing and invites immediate retry.

The response ladder, cheapest first — and note that shedding is the *last* resort, not the first:

| Response | What it costs | When |
|---|---|---|
| **Serve degraded** (cached, stale, partial, smaller model) | Some quality | Always try this first: 100% of users get a worse answer instead of 30% getting none |
| **Prioritise** (drop low-value classes first) | Fairness within a class | When your traffic has genuinely different value: health checks and leader elections must never be shed |
| **Shed at admission** with `429` + `Retry-After` | The dropped requests | Sustained overload past the ladder above |
| **Backpressure upstream** (refuse to accept, let the producer block) | Producer-side latency | Closed systems where the producer can actually slow down |
| **Queue** | Latency, memory | **Bursts only** — bursts shorter than `q_max / (λ − λ_max)` |

That last row is the honest statement of what a queue is for: **absorbing bursts whose duration is shorter than the time it takes to fill the queue.** A queue sized to absorb a two-second burst is doing exactly the job it should. The same queue facing a two-minute overload is a latency amplifier.

### Under overload, prefer the freshest work

Once the queue is deep, FIFO becomes actively harmful. The item at the head has been waiting the longest, which means it is the item most likely to have already been abandoned by its client:

```
  Client timeout 1 s. Queue wait currently 3 s. λ_max = 5,000/s.

  FIFO: workers dequeue items that arrived 3 s ago. Every client that
        submitted them gave up 2 s ago. GOODPUT = 0. The system is at
        100% utilisation producing responses nobody is waiting for, and
        it stays there forever, because it never catches up.

  LIFO: workers dequeue items that arrived milliseconds ago, which are
        still inside their deadline. Goodput ≈ λ_max. The items at the
        BOTTOM of the stack starve, but they were already dead.
```

**LIFO under overload maximises goodput, and it does it by giving up on requests that were already lost.** Two important caveats:

- It is for **deadline-bound machine work**. In a human-facing queue — a checkout line, a signup flow — serving the newest arrival first is experienced as unfair even when it is throughput-optimal, and "fairness for people already waiting" is a real product requirement that overrides the math.
- It is a *proxy* for the real rule, which is deadline propagation. LIFO guesses that older means expired. Deadlines *measure* it.

### Deadline propagation makes it principled

Stamp every request with a deadline at ingress and propagate it through every hop:

```
  gateway   receives request, budget = 500 ms, deadline = now + 500 ms
     │  passes deadline in metadata (gRPC does this natively via the context
     │  deadline; HTTP needs a header convention and discipline)
     ▼
  router    now + 30 ms elapsed → 470 ms remain
     │  BEFORE enqueuing: is (current queue wait) > (remaining budget)?
     │      current queue wait = q / λ_max = 2,500/5,000 = 500 ms
     │      500 ms > 470 ms  ⇒  REJECT NOW, do not enqueue.
     ▼
  worker    checks the deadline again before starting expensive work,
            and again before each downstream call
     │
     ▼
  downstream services inherit the REMAINING budget, not a fresh one
```

Three properties follow, and all three matter:

- **You never start work you cannot finish in time.** Capacity spent on a doomed request is capacity stolen from a live one.
- **Cancellation propagates.** When the client disconnects, everything downstream can stop — which is why gRPC's context cancellation and Go's `context.Context` are load-shedding infrastructure, not just tidy plumbing.
- **Retry storms get self-limiting.** A retried request inherits the *remaining* budget, so a request that has already burned 400 ms of its 500 ms budget cannot spawn a full-cost retry.

The anti-pattern this replaces is the one almost every service ships with: each hop has its own fixed timeout, so a three-hop call with 1 s timeouts each can take 3 s while the client gave up at 1 s, and every hop keeps working on it.

### Backpressure beats dropping at a full buffer

Dropping at a full buffer is shedding *after* the latency has already been paid. Credit-based flow control prevents the buffer filling at all:

```
  TAIL-DROP (what a naive bounded queue does)
    producer ──▶ [████████████████████] queue FULL ──▶ drop whatever arrives next
    · the dropped item is chosen by arrival timing, not by value
    · everything already in the queue has already waited, so the latency cost
      is sunk before the drop decision is made
    · producers get no signal until they are already over the line

  CREDIT / PULL-BASED BACKPRESSURE
    consumer advertises: "I have room for 32 more"
    producer sends at most 32, then STOPS until credit returns
    · the shed decision moves UPSTREAM to the cheapest possible point, where
      the producer can choose what not to send, or slow its own source
    · the queue never fills, so queueing latency stays bounded by design
    · ordering and fairness are preserved because nothing is dropped mid-stream
    Real instances: TCP's receive window; gRPC/HTTP-2 flow-control windows;
    Kafka consumer pull semantics; reactive-streams `request(n)`.

  THE LIMITATION worth naming: backpressure only works if the producer can
  actually slow down. Against the open internet it cannot — you cannot ask
  the world to send fewer requests — so at the EDGE you shed, and INSIDE the
  system you backpressure. Getting this backwards (trying to backpressure the
  internet, or dropping internal messages you could have paced) is a common
  and expensive design error.
```

### Coordinated omission: why your capacity number is wrong

This is the measurement failure that makes everything above dangerous to trust.

```
  CLOSED-LOOP load generator: each virtual user waits for its response before
  sending the next request. 100 users, target 1 request each per 10 ms.

   normal:  u1 ├─10ms─┤├─10ms─┤├─10ms─┤├─10ms─┤   (4 samples, all 10 ms)

   with a 2 s stall at t=1s:
            u1 ├─10ms─┤├──────── 2,000 ms ────────┤├─10ms─┤
                       ▲                           ▲
                       │ ONE sample of 2,000 ms    │ back to normal
                       │
              during the stall, u1 sends NOTHING. The ~200 requests it
              WOULD have sent (one per 10 ms) are never issued, never
              measured, and never counted.

  What gets recorded: 1 slow sample among ~400 fast ones ⇒ p99 looks like…
  10 ms. The 99.75th percentile is fine! The system stalled for two seconds.

  What SHOULD have been recorded (open-loop, requests arrive on schedule
  regardless of server state — as real users do):
      request arriving at t=1.00 s waits 2,000 ms
      request arriving at t=1.01 s waits 1,990 ms
      …
      request arriving at t=2.99 s waits    10 ms
      ⇒ ~200 samples averaging ~1,000 ms.
      Now the p99 over the whole run is DOMINATED by the stall, as it should be.

  Magnitude of the error, worked:
      run length 60 s at 10,000 req/s = 600,000 requests
      one 2 s stall omits ~20,000 samples (3.3% of the run)
      those omitted samples are ALL in the worst 3.3% of the distribution
      ⇒ everything at and above p96.7 is fabricated. Your p99 and p99.9 are
        measuring the healthy period only.
```

The rule: **a closed-loop load test cannot measure the failure mode this lesson is about, and it fails silently and optimistically, in proportion to how bad the stall was.** The fixes, in order of preference: use an **open-loop** generator that issues at a fixed rate regardless of response timing (wrk2, `vegeta`, k6 with a constant-arrival-rate executor, and `ghz` in that mode); or record with a histogram that corrects for the omission (HdrHistogram's `recordValueWithExpectedInterval`, which synthesises the missing samples); at minimum, cross-check your generator's reported rate against the *intended* rate — if the achieved rate dipped during the test, that dip is exactly the omitted mass.

Gil Tene coined the term. The practical heuristic: **if your load-testing tool cannot tell you the difference between "response time" and "service time," treat every percentile above p50 that it reports as decoration.**

### Metastable failure: the loop that outlives its trigger

```
   ┌─────────────┐   trigger    ┌────────────────────┐   drain     ┌──────────┐
   │   STABLE    │─────────────▶│   METASTABLE       │────────────▶│  STABLE  │
   │             │  (spike,     │  (overloaded, and  │  (shed hard,│          │
   │ λ < capacity│   flush,     │   STAYING there)   │  kill in-   │          │
   │ queue ≈ 0   │   deploy,    │                    │  flight     │          │
   └─────────────┘   node loss) │  ┌──────────────┐  │  retries,   └──────────┘
        ▲                       │  │ SUSTAINING   │  │  restart,        │
        │                       │  │    LOOP      │  │  admit slowly)   │
        │  removing the trigger │  │ ↻ retries    │  │                  │
        └───── DOES NOT ────────┤  │ ↻ empty cache│  ├──────────────────┘
               work             │  │ ↻ full queue │  │
                                │  │ ↻ timeouts   │  │
                                │  └──────────────┘  │
                                └────────────────────┘

   The defining property: offered load is now sustained ABOVE capacity by a
   loop internal to the system. The original spike is long gone. "Wait for
   traffic to die down" is not a recovery strategy, because there is no
   external traffic holding it up any more — the system is feeding itself.
```

The three canonical loops, each with its own escape:

| Loop | Mechanism | Escape |
|---|---|---|
| **Retries** | Each timeout produces `r` more requests; offered load becomes `λ(1+r)`, which raises latency, which causes more timeouts | Retry budget (below), circuit breakers, deadline propagation so retries inherit a shrinking budget |
| **Cold cache** | Origin cannot serve the miss storm, so the cache never refills, so the miss storm continues (lesson 04) | Shed until the origin can serve enough misses to refill; pre-warm; admit traffic in stages |
| **Queue backlog** | Every request now waits `q/λ_max`, exceeding client timeouts, so clients retry, so `q` grows | Drop the queue entirely (yes, all of it), then admit slowly |

**Retry amplification, with the arithmetic.** Suppose a fraction `f` of requests fail (timeout) and each failure is retried up to `r` times:

```
   offered load  ≈  λ × (1 + f·r + (f·r)² + …)  =  λ / (1 − f·r)   for f·r < 1

   f = 0.2, r = 3  ⇒  f·r = 0.6  ⇒  offered load = λ / 0.4 = 2.5 λ
   f = 0.4, r = 3  ⇒  f·r = 1.2  ⇒  the series DIVERGES: there is no
                                     equilibrium. Load grows without bound
                                     until something else breaks.

   ⇒ The failure rate at which retries become self-sustaining is f = 1/r.
     With three retries that is a 33% failure rate — a threshold a struggling
     service crosses easily, and after which it cannot recover on its own no
     matter how much the external traffic falls.
```

**The retry budget** is the fleet-wide fix, and it works where per-client backoff does not:

```
   Token bucket, refilled at 10% of the observed success rate.
   base traffic 10,000 req/s ⇒ budget ≈ 1,000 retries/s, fleet-wide.
   A retry consumes a token; if the bucket is empty, the retry is REFUSED
   IMMEDIATELY (the caller sees the original error instead).

   Why this and not backoff: exponential backoff with full jitter is correct
   PER CLIENT and says nothing about the AGGREGATE. Ten thousand perfectly
   well-behaved clients each retrying politely still produce a retry storm,
   because no client can observe the total. The budget caps the sum, which is
   the only quantity that matters to the server. Implement it server-side or
   in a shared client library — and note that gRPC's retry policy config
   includes exactly this (`retryThrottling` with `maxTokens`/`tokenRatio`)
   for the same reason.
```

**The escape procedure**, because "restart it" is not a plan:

1. **Stop the loop first.** Shed at the edge — aggressively, far below capacity. Disable retries fleet-wide if you have the switch. This is the step people skip because it looks like making the outage worse.
2. **Drain the amplifying state.** Empty the queues. Kill in-flight requests that are past their deadlines. Restart components holding poisoned state if that is the only way.
3. **Verify you are below capacity** before re-admitting — measure, do not assume.
4. **Admit in stages**, watching latency between steps. Going straight back to 100% re-triggers the loop, and now you have taught everyone that the fix does not work.

### Fair sharing: the same problem, in capacity instead of time

Everything above concerns *time* (queue wait). The identical shed/hold/starve tension appears in *capacity allocation*, and two systems you run implement it explicitly.

**Kubernetes API Priority and Fairness** is queueing theory shipped as a control plane:

```
  Total concurrency = --max-requests-inflight (400) + --max-mutating-requests-inflight (200)
                    = 600 SEATS

  ┌──────────────── PriorityLevelConfigurations (default set) ──────────────┐
  │ exempt          system:masters — never queued, never limited            │
  │ node-high       node health updates                                      │
  │ system          kubelets (system:nodes)                                  │
  │ leader-election leases from controller-manager / scheduler               │
  │ workload-high   built-in controllers                                     │
  │ workload-low    other service accounts                                   │
  │ global-default  everything else                                          │
  │ catch-all       very small share; the backstop                           │
  └─────────────────────────────────────────────────────────────────────────┘
    Each level has a NOMINAL share of the 600 seats, and levels LEND unused
    concurrency to busy levels and BORROW within configured bounds — the
    limits are recomputed periodically from observed demand.

  Within a level, requests are assigned to a FLOW (FlowSchema name +
  distinguisher: user, or namespace, or nothing) and then to a QUEUE by
  SHUFFLE SHARDING — each flow draws `handSize` queues out of `queues`.

  Published collision probabilities (probability a given "mouse" flow shares
  every one of its queues with some "elephant"):

     handSize  queues   1 elephant     4 elephants    16 elephants
        12       32     4.4e-09        0.114          0.994
        10       64     6.6e-12        0.00046        0.500
         8       64     2.3e-10        0.00049        0.359
         8      128     7.0e-13        3.4e-06        0.0275
         6      512     4.1e-14        5.0e-09        2.3e-05

  Read the trade in that table: a LARGER hand size makes two individual flows
  less likely to collide, but makes it easier for a few flows to occupy every
  queue — and it raises the latency one heavy flow can inflict, since a single
  flow can hold up to handSize × queueLengthLimit queued requests.

  SEATS ARE NOT UNIFORM. A list request the server estimates will return many
  objects is charged seats PROPORTIONAL TO THE ESTIMATE — this is Kingman's
  variability term implemented as an admission-control policy: expensive,
  high-variance work is made to cost what it actually costs.

  And the shed decision is a literal enum: a PriorityLevelConfiguration of
  type `Reject` returns HTTP 429 immediately; type `Queue` queues with fair
  queueing and shuffle sharding, bounded by queueLengthLimit.
```

**Kueue's fair sharing** is the same tension one level up, in GPU-seconds instead of seats:

```
  Cohort = a set of ClusterQueues that share borrowable resources.
  Each ClusterQueue has NOMINAL quota; unused nominal quota in the cohort is
  BORROWABLE by others, up to borrowingLimit.

  Kueue assigns each ClusterQueue a weighted SHARE VALUE summarising its
  borrowed-resource usage relative to the cohort (a dominant-resource share,
  weighted by .spec.fairSharing.weight). It is observable:
      .status.fairSharing.weightedShare
      metric: kueue_cluster_queue_weighted_share

  · admission:  prefer the ClusterQueue with the LOWEST share value
  · preemption: prefer to preempt from the ClusterQueue with the HIGHEST share

  preemptionStrategies (Kueue Configuration) bound when a preemption may fire:
      LessThanOrEqualToFinalShare — preemptor's share after admitting must not
                                    exceed the target's share before preemption
      LessThanInitialShare        — preemptor's share after admitting must be
                                    strictly below the target's current share
  These constraints are what make the system provably loop-free: Kueue's own
  documentation proves that if A preempts B, B cannot then preempt A, because
  both preemptions would require DRS_A < DRS_B and DRS_B < DRS_A.

  THE STAFF TENSION, in one line: preempting a borrowed GPU job destroys its
  progress since the last checkpoint (lesson 06's arithmetic), so the real
  tuning knob is HOW FAR over fair share borrowing is allowed to run before
  reclamation fires — utilisation (let A keep borrowing) versus fairness
  (give B its share now) versus starvation (never let B wait unboundedly),
  with checkpoint interval as the hidden fourth term.
```

Note the structural identity to the request queue: fair queueing is LIFO-vs-FIFO's argument at a different timescale, borrowing is queueing, preemption is shedding, and the loop-freedom proof is the equivalent of ensuring your shed path cannot itself cause more work.

## Perspectives

**The queueing-theory view.** Little's Law is an identity, so it *forbids* rather than suggests: at a fixed concurrency limit, buffer size cannot move `λ_max`. Kingman's formula adds the term that most capacity plans omit — variability multiplies queueing as surely as utilisation does, so mixing 20 ms point reads with 2 s scans in one queue costs you more than the extra load does. That is the formal case for workload classes, for request-size caps, and for charging expensive requests more seats.

**The measurement view.** A capacity number derived from a closed-loop load test is not conservative; it is optimistic in exactly the regime you care about, and by an amount proportional to how badly the system stalled. Before trusting any p99, ask two questions: was the generator open-loop, and did the achieved rate ever dip below the target? A dip *is* the omitted mass.

**The control-theory view.** Metastability is not a distributed-systems curiosity; it is a sustaining loop overpowering the removal of a trigger. That reframing tells you exactly what to do — intervene on the loop, not on the trigger — and it tells you why the intervention feels wrong: you must shed harder than the current load seems to justify, because the load you see is the loop's output, not the customers' input.

**The scheduling/fairness view.** Every allocation system eventually implements the same three-way tension: utilisation (let whoever is here use everything), fairness (reserve for whoever is not here yet), and starvation avoidance (bound how long the absent party waits). APF resolves it with seats, shuffle-sharded queues and borrowing; Kueue with weighted dominant-resource shares and bounded preemption. Neither invents a new answer, and neither can — the tension is structural. What a staff engineer contributes is naming the deciding number: for APF it is `handSize`/`queues` against your flow count; for Kueue it is the borrowing overshoot allowed before reclamation, priced against the checkpoint interval it will destroy.

## Real-world use cases

- **Kubernetes API Priority and Fairness** (verified, read from `kubernetes/website`). The shuffle-sharding collision table reproduced above; the seat model including proportional seat charging for large list requests and extra seat-time charged to writes for the watch notifications they will trigger; the default priority levels that guarantee an ill-behaved Pod cannot starve leader election; and the `Reject` (immediate 429) versus `Queue` (fair queueing, bounded by `queueLengthLimit`) choice. **What it shows:** every idea in this lesson, running in your cluster today, with metrics (`apiserver_flowcontrol_current_executing_seats`, `apiserver_flowcontrol_demand_seats`, `apiserver_flowcontrol_current_inqueue_seats`, and per-priority-level queue-length histograms) you can read right now.
- **Kueue fair sharing** (verified, read from `kubernetes-sigs/kueue`). Weighted share values exposed at `.status.fairSharing.weightedShare` and as `kueue_cluster_queue_weighted_share`; admission ordered by lowest share and preemption by highest; the `LessThanOrEqualToFinalShare` / `LessThanInitialShare` strategies; and the published proof that these constraints prevent two workloads in different ClusterQueues from preempting each other in a loop. **What it shows:** a production GPU scheduler treating fairness as an explicitly-computed quantity with an anti-thrashing proof — and a correction worth noting: it is a **dominant-resource share**, not a simple usage ratio, which matters when tenants request different resource mixes.
- **Slack, *Slack's Incident on 2-22-22*** — a load spike on a Vitess keyspace, driven by an inefficient query pattern, that turned into a sustained retry storm. The detail that makes it worth reading: Slack's clients were already doing exponential backoff **with jitter** — the textbook-correct pattern — and it was still insufficient, because per-client-correct behaviour says nothing about aggregate retry rate. Recovery followed the shed-first playbook: throttle new connections at the edge, fix the load-amplifying query, then let traffic back in. **Not fetched this pass** (slack.engineering blocked); the mechanism is corroborated by the retry-amplification arithmetic above and by gRPC's inclusion of retry throttling as a first-class config.
- **Huang, L. et al., *Metastable Failures in the Wild* (OSDI 2022)** — the paper behind the terminology, with a survey of public incident reports. The figures previously cited in this lesson — that at least 4 of 15 studied AWS outages followed the metastable pattern, and that the study covers 22 incidents across 11 organisations — are **recalled and not verified this pass** (usenix.org is blocked); treat them as approximate and re-check before quoting. The taxonomy (trigger, sustaining effect, the gap between the load that causes the transition and the much lower load required to recover) is what makes the paper worth reading in full.
- **Shopify, *Surviving Flashes of High-Write Traffic Using Scriptable Load Balancers*** — flash-sale admission control at the edge, in the load balancer, before traffic reaches any backend, plus a documented correction where an early version failed to preserve fairness for customers already queued. **Not fetched this pass** (shopify.engineering blocked). It is the counterweight to this lesson's LIFO advice: for a human-facing queue, "serve the freshest" is throughput-optimal and unacceptable.
- **Google SRE, *Handling Overload* and *Addressing Cascading Failures*** — the operational playbook this lesson's ladder follows: graceful degradation before shedding, per-customer limits, client-side throttling, and the observation that a system in cascading failure usually needs load reduced *below* the level that triggered it before it recovers. **Not fetched this pass** (sre.google blocked).

## Worked example

### Part A — the inference tier's admission threshold

**Setup.** 8 replicas, each running continuous batching with 32 concurrent sequence slots. Mean per-step service time is 40 ms. TTFT SLO is 500 ms at p99. Arrival rate is bursty: `c_a ≈ 1`, and service is fairly uniform, `c_s ≈ 0.5`.

**A1 — The ceiling, from Little's Law.**

```
   L_max = 8 replicas × 32 slots         = 256 concurrent sequences
   W_service = 40 ms per step
   λ_max = L_max / W_service = 256 / 0.040 s = 6,400 steps/s

   Sanity check the units: "steps/s," not "requests/s." Convert with your
   measured steps-per-request (prefill + decode). At an average 200 steps per
   request: 6,400 / 200 = 32 requests/s. THIS IS THE NUMBER, and it is far
   smaller than the step-rate figure that flatters the slide.
```

**A2 — The operating point, from the utilisation curve.**

```
   At ρ = 0.95:  W_total = 40 ms / (1−0.95) = 800 ms  ⇒ SLO ALREADY BREACHED
   At ρ = 0.90:  40 / 0.10 = 400 ms                   ⇒ inside 500 ms, barely
   At ρ = 0.80:  40 / 0.20 = 200 ms                   ⇒ comfortable

   With Kingman's variability term at ρ=0.8, c_a=1, c_s=0.5:
      E[W_queue] ≈ (0.8/0.2) × ((1 + 0.25)/2) × 40 ms = 4 × 0.625 × 40 = 100 ms
      total ≈ 140 ms.  Still comfortable.
   At ρ=0.9:
      E[W_queue] ≈ 9 × 0.625 × 40 = 225 ms, total ≈ 265 ms — and this is the
      MEAN. The p99 is several times it, which is why the target operating
      point is 0.8 and not 0.9.

   ⇒ Plan capacity at ρ = 0.8 ⇒ target λ = 0.8 × 32 = ~26 requests/s,
     and treat the 20% headroom as the SLO's cost, not as waste.
```

**A3 — The drop threshold.**

```
   Queue budget = SLO − service = 500 ms − 40 ms = 460 ms of allowable wait.
   Wait implied by a queue of depth q at saturation:  W_queue = q / λ_max.

   q_max = 0.460 s × 6,400 steps/s ≈ 2,944 steps in queue
         ≈ 2,944 / 200 steps-per-request ≈ 15 requests queued fleet-wide.

   Set the actual threshold LOWER — the derivation used the mean, and you owe
   the SLO at p99. A 40% margin is a reasonable starting point:
        drop threshold ≈ 0.6 × 2,944 ≈ 1,750 steps ≈ 9 queued requests

   Implementation: reject at the edge with 429 + Retry-After when the measured
   queue-implied wait exceeds the remaining deadline. Prefer measuring the wait
   directly (timestamp on enqueue, compare on dequeue) over inferring it from
   depth — depth is a proxy, wait is the thing the SLO is about.
```

**A4 — How fast does it go wrong?**

```
   A 20% overload: λ = 1.2 × λ_max ⇒ dq/dt = 0.2 × 6,400 = 1,280 steps/s
   time to fill the 2,944-step budget from empty:  ≈ 2.3 seconds.

   ⇒ Shedding must be automatic and must react within a second or two.
     A five-minute autoscaler is not a load-shedding mechanism — it is a
     capacity mechanism, and it operates two orders of magnitude too slowly
     to protect this SLO. You need BOTH, and you must not confuse them.
```

**A5 — The retry guard.**

```
   Without a budget, at 20% failure with 3 retries: offered load = λ/(1−0.6)
   = 2.5λ — from a 20% overload you have manufactured a 150% overload.
   At 33% failure the series diverges and no external traffic reduction saves you.

   With a 10% retry budget: max additional load = 0.1λ, so the worst case is
   1.1λ regardless of how many clients decide to retry. The budget is what
   turns an unbounded amplifier into a bounded 10% surcharge.
```

### Part B — the Kueue cohort

**Setup.** Two tenants share 100 GPUs. Tenant A's ClusterQueue has 60 nominal, tenant B's has 40; both are in one cohort, so unused nominal quota is borrowable. Jobs checkpoint every 30 minutes.

```
  STATE 1 — both saturated.
    A = 60, B = 40. Nobody borrows. Weighted shares equal (each at its nominal).
    Admission order is irrelevant; the system is at its fair point.

  STATE 2 — B idle, A has a queue.
    B's 40 GPUs are unused ⇒ borrowable. A borrows up to borrowingLimit and
    runs at up to 100 GPUs. Utilisation 100%; A's weighted share is now high.
    THIS IS THE POINT OF THE COHORT — idle quota does no work.

  STATE 3 — B bursts. The interesting one.
    Kueue admits B's workloads preferring the LOWEST share value (B's is 0),
    and looks for preemption targets in the HIGHEST-share queue (A's borrowed
    jobs), subject to the configured preemptionStrategies.

    THE COST NOBODY PUTS IN THE SLIDE:
      A borrowed job preempted 20 minutes into a 30-minute checkpoint interval
      loses 20 minutes × (GPUs held) of work.
      Preempting 40 GPUs' worth of A's jobs at a mean of 15 minutes since
      checkpoint destroys 40 × 0.25 h = 10 GPU-hours.
      At a notional $2/GPU-hour that is $20 per reclamation event — small,
      until you notice it recurs every time B's usage oscillates.

    THE ACTUAL KNOB: how far over fair share A may run before reclamation
    fires, and how long B waits.
      · reclaim instantly  → maximum fairness, maximum wasted work, and
                             thrashing if B's demand oscillates
      · reclaim after a grace period ≥ the checkpoint interval → B waits up to
        30 minutes; A loses at most the tail since its last checkpoint
      · never reclaim (no preemption) → B can starve indefinitely: unacceptable

    THE DESIGN ANSWER: set the reclamation grace period from the CHECKPOINT
    INTERVAL, not from a fairness intuition — and shorten the checkpoint
    interval for jobs that run in borrowed capacity, so borrowed work is cheap
    to reclaim. That reframing (make preemption cheap rather than rare) is the
    staff move, and it is only visible if you cost the preemption in GPU-hours.
```

### Part C — reading your own API server as a queue

```
  A one-off exercise with real numbers, on any cluster you run:

   1. seats in use vs limit, per priority level:
        sum(apiserver_flowcontrol_current_executing_seats) by (priority_level)
        / apiserver_flowcontrol_nominal_limit_seats
      ⇒ your ρ per class. Anything sustained above 0.8 is a latency problem
        already, whether or not anyone has complained.

   2. queue lengths:
        apiserver_flowcontrol_current_inqueue_requests by priority_level
      ⇒ if this is non-zero at steady state, you are queueing, i.e. converting
        rejection into latency, i.e. spending your SLO.

   3. rejections:
        rate(apiserver_flowcontrol_rejected_requests_total[5m]) by (reason)
      ⇒ `queue-full` means your queueLengthLimit is the binding constraint;
        `time-out` means requests aged out; `cancelled` means clients gave up
        first — which is coordinated omission happening to you in production.

   4. THE ONE TO ACT ON: a single flow dominating a priority level. Compare
        apiserver_flowcontrol_demand_seats (a histogram, per level)
      against the nominal limit. A level whose demand persistently exceeds its
      limit is a level that needs either more seats (borrowing), its own
      FlowSchema, or a fixed client. This is precisely how a chatty operator
      that lists all pods every second is found — and it is the same lesson-03
      hot-shard problem, wearing an API-server costume.
```

## Practice

*Feeds the [staff design portfolio](../practice/staff-design-portfolio/README.md).*

1. **Admission-control drop-threshold calculation (artifact).** For an inference tier you own or design: derive `L_max` and `λ_max` from Little's Law with real slot counts and measured service time; state your target `ρ` and justify it against the utilisation curve *and* Kingman's variability term (measure `c_a` and `c_s`, do not assume 1); compute the queue budget from the SLO and the drop threshold, including the margin you added and why; and compute the time-to-breach at a 20% overload, which is your required shed reaction time. State explicitly which mechanism reacts on that timescale — it will not be your autoscaler.
2. **Kueue fair-share allocation model (artifact).** Model a two-tenant cohort: steady-state allocation, borrowing behaviour when one tenant is idle, the preemption trigger when it returns, and — the part that makes it a staff artifact — the **cost of a reclamation in GPU-hours**, computed from the checkpoint interval. Give the reclamation grace period you would set and derive it from the checkpoint interval rather than from a fairness intuition.
3. **Prove coordinated omission on your own stack.** Run the same load test twice against a service you can stall (add a 2-second sleep behind a feature flag): once closed-loop, once open-loop at a fixed rate. Record both p99s. Then compute the fraction of the run the stall occupied and check it against the percentile above which the closed-loop numbers are fabricated. Put both numbers in your design doc — this single comparison changes how a room treats every latency number thereafter.
4. **Write the retry budget.** Specify a token bucket for one service: refill rate as a fraction of success rate, bucket size, what happens when it is empty, and where it lives (server-side, client library, or service mesh). Then compute the amplification factor `1/(1 − f·r)` for your current retry policy at `f` = 10%, 20% and 33%, and state which of those your current configuration survives.
5. **Read your API server.** Run the four PromQL queries in Part C against a real cluster and write a paragraph on what you found: your per-class utilisation, whether you are queueing at steady state, and whether any single flow is dominating a priority level. If something is, find the client.

## Common pitfalls

1. **"Add a bigger queue to absorb the burst."** Symptom: latency degrades and throughput does not improve. Mechanism: at fixed `L_max`, `λ_max = L_max / W_service` contains no buffer term; extra depth goes entirely into `W`. A queue is for bursts shorter than `q_max / (λ − λ_max)` — compute that duration and check whether your burst is actually shorter than it.
2. **Running at 90–95% utilisation because "there's still headroom."** Symptom: p99 that behaves nothing like p50, and cliff-edge behaviour on small load increases. Mechanism: `W ∝ 1/(1−ρ)` — 90% → 95% doubles latency. Target 50–70% for latency-sensitive paths and call the remainder the SLO's cost.
3. **Ignoring variability.** Symptom: two services at the same utilisation with wildly different tails. Mechanism: Kingman's `(c_a² + c_s²)/2` term — mixing 20 ms and 2 s work in one queue multiplies queueing time even at unchanged load. Separate the classes, cap request size, or charge expensive requests more (which is exactly what APF's proportional seat accounting does).
4. **An expensive shed path.** Symptom: adding load shedding did not help, or made things worse. Mechanism: if rejecting costs a meaningful fraction of serving, the shed path saturates too. A rejection must not authenticate, query a database, write a log line synchronously, or allocate much. Measure the cost of your `429`.
5. **"Exponential backoff with jitter is enough."** Symptom: a retry storm despite textbook-correct client behaviour (Slack's 2-22-22). Mechanism: backoff is per-client and bounds nothing in aggregate; ten thousand polite clients still produce a storm. You need a fleet-wide retry budget, which is the only thing that caps the sum.
6. **Trusting a closed-loop load test.** Symptom: "we tested 10,000 rps and it was fine" followed by an outage at 6,000. Mechanism: coordinated omission — the generator stops sending during stalls, so every percentile above `1 − stall_fraction` is fabricated. Use open-loop generation or CO-corrected recording.
7. **"The spike passed, it'll recover."** Symptom: an outage that outlives its cause by an hour. Mechanism: a sustaining loop (retries, cold cache, queue backlog) is now holding load above capacity by itself. Recovery requires intervening on the loop — shed below the trigger level, drain state, re-admit in stages.
8. **FIFO under deep overload.** Symptom: 100% utilisation and near-zero goodput. Mechanism: the head of a FIFO queue is the item most likely already abandoned. Use deadlines to drop provably-dead work, and LIFO as the ordering heuristic for machine-to-machine deadline-bound queues — but not for human-facing ones, where arrival order is a fairness contract.
9. **Per-hop timeouts instead of propagated deadlines.** Symptom: a client that gave up 2 s ago while four services still work on its request. Mechanism: each hop restarting the clock means total time is the *sum* of the timeouts. Propagate the remaining budget and check it before every expensive step.
10. **Confusing autoscaling with load shedding.** Symptom: an autoscaling policy cited as the overload plan. Mechanism: overload fills the queue budget in seconds; autoscaling reacts in minutes and needs the capacity to exist. You need shedding for the seconds and autoscaling for the minutes, and saying so explicitly in a design review is a fast way to find out whether anyone has done the arithmetic.

## Self-check

- **At a fixed concurrency limit, what does increasing queue size do to throughput?** **Answer:** Nothing. `λ_max = L_max / W_service`, and buffer appears in neither term. Extra depth raises `L` only up to the concurrency cap; beyond that the excess goes into `W`, so latency rises and throughput stays pinned. You have converted rejection into latency — and past the client's timeout, that latency *is* a rejection, only after you have paid for it. The only levers on `λ_max` are more concurrency or less service time per item.

- **Your service is at 80% utilisation and someone proposes running it at 90% to save money. What do you say?** **Answer:** That 90% is not "10% more load," it is twice the latency: `W ∝ 1/(1−ρ)` gives 5× service time at 0.8 and 10× at 0.9. And that is the mean — the tail degrades faster. Then bring in Kingman: if the workload mixes short and long requests, the `(c_a²+c_s²)/2` factor multiplies the whole thing, so measured `c_s` may make 0.9 far worse than the textbook curve suggests. The 20% headroom is not idle capacity, it is what buys the p99 in the SLO. If the money matters, reduce `W_service` or separate the workload classes; do not spend the headroom.

- **Why prefer LIFO under deep overload, and when is it wrong?** **Answer:** At a queue wait longer than the client timeout, FIFO dequeues items whose clients gave up long ago, so goodput approaches zero while utilisation stays at 100% — and the system never catches up. LIFO serves the freshest items, which are the ones still inside their deadline, so goodput approaches `λ_max`. It is wrong in two cases: for human-facing queues, where arrival order is a fairness contract customers can perceive (Shopify's flash-sale correction is the documented example); and as a substitute for deadlines, since LIFO only *guesses* that older means expired. Propagate deadlines and drop provably-dead work; use LIFO as the ordering heuristic on top.

- **What makes a failure metastable, and what is the escape?** **Answer:** A sustaining loop internal to the system holds offered load above capacity after the original trigger is gone — retries feeding on timeouts, an empty cache the origin cannot refill, a queue backlog that pushes every request past its client's timeout. The signature is that removing the trigger does not help and the load required to *stay* in the bad state is much lower than the load required to *enter* it. Escape: shed at the edge below the level that triggered it (counter-intuitive, and the step people skip), drain the amplifying state (empty queues, kill deadline-expired in-flight work, restart if state is poisoned), verify you are below capacity by measurement, then re-admit in stages. Retry amplification gives the arithmetic: offered load ≈ `λ/(1 − f·r)`, which diverges at `f = 1/r` — a 33% failure rate with three retries.

- **Why can a load test report a clean p99 while the same system falls over in production?** **Answer:** Coordinated omission. A closed-loop generator sends the next request only after the previous response, so during a stall it sends nothing and the requests that would have arrived — the ones that would have measured the worst latency — are never issued or recorded. A 2-second stall in a 60-second run at 10,000 rps omits ~20,000 samples, 3.3% of the run, and they are precisely the worst 3.3%: everything from p96.7 upwards is fabricated. Fixes: open-loop generation at a fixed arrival rate (wrk2, vegeta, k6 constant-arrival-rate), or CO-corrected recording (HdrHistogram's expected-interval variant). Quick smell test: if the achieved rate ever dipped below the target rate, that dip is the omitted mass.

- **Backoff versus a retry budget — why do you need both?** **Answer:** Backoff with jitter is per-client and does two things well: it spreads correlated retries in time and it reduces one client's pressure on a struggling server. It bounds nothing in aggregate — ten thousand perfectly-behaved clients still sum to a storm, because no client can observe the total. A retry budget is a server-side or shared-library token bucket refilled at a small fraction (commonly 10%) of the success rate, so total retries are capped fleet-wide no matter how many clients want to retry; when the bucket is empty, retries fail immediately. Backoff shapes the distribution; the budget caps the integral. gRPC's `retryThrottling` config exists for exactly this reason.

- **Explain APF as a queueing system, in its own vocabulary.** **Answer:** Total concurrency is `--max-requests-inflight` (400) plus `--max-mutating-requests-inflight` (200) = 600 seats — that is `L_max`. Seats are partitioned across priority levels by nominal share, with borrowing between levels so idle capacity is not wasted. Within a level, requests map to flows (FlowSchema plus distinguisher) and flows map to queues by shuffle sharding — `handSize` queues drawn from `queues`, with published collision probabilities that let you trade isolation against a single flow's ability to dominate. A level configured `Reject` returns 429 immediately; `Queue` queues up to `queueLengthLimit`, so a single flow can hold at most `handSize × queueLengthLimit` queued requests. Crucially, seats are not uniform: large LIST requests are charged seats proportional to their estimated result size, and writes are charged extra seat-time for the watch notifications they generate — which is Kingman's variability term implemented as admission policy.

## Connections & what's next

This lesson generalises lesson 04's cache-flush miss storm into the theory of overload: the flush was one trigger among many, and queueing, backpressure and metastability determine whether *any* trigger produces a graceful shed or an unrecoverable spiral. It also closes a loop with lesson 02 — a slow etcd disk raises `W_service` on the control plane's write path, and everything in this lesson then applies mechanically: utilisation rises, the queue fills, controllers time out and retry, and you have a metastable control plane whose root cause is a disk.

Next, **[06 · Failure and resilience](06-failure-and-resilience.md)**: every mechanism here assumes you can *tell* that the system is overloaded or that a component is down. Failure detection is the layer beneath that assumption, and it is provably imperfect — a slow node and a dead node are indistinguishable from the outside, which is precisely why timeouts, and therefore this lesson's entire apparatus, are heuristics. Lesson 06 also supplies the checkpoint-interval arithmetic that Part B of the Worked example treated as a given. **Lesson 07** picks up the other consequence: retries — the sustaining loop here — produce duplicate delivery downstream, which makes the shedding discipline in this lesson a precondition for the delivery-semantics work there.

Carry forward: *if you cannot distinguish a slow component from a failed one, how should that change the timeouts, retries and shed thresholds you just computed?*

## References & further reading

**Primary sources — verified against upstream Git repositories this pass**

1. **Kubernetes, *API Priority and Fairness*** — `content/en/docs/concepts/cluster-administration/flow-control.md` in <https://github.com/kubernetes/website> (kubernetes.io is blocked here). **Source of** the seat model and the statement that total concurrency is the sum of the two inflight flags; proportional seat charging for large list requests and the extra seat-time charged to writes for watch notifications; the flow/FlowSchema/distinguisher model; shuffle sharding with the full `handSize`/`queues` collision-probability table reproduced above and its stated trade-off; the `Reject` (429) versus `Queue` semantics and `queueLengthLimit`; the `handSize × queueLengthLimit` bound on one flow's queued requests; the default priority levels (`exempt`, `node-high`, `system`, `leader-election`, `workload-high`, `workload-low`, `global-default`, `catch-all`); and the metric names used in Part C.
2. **Kubernetes, `kube-apiserver` reference** — `content/en/docs/reference/command-line-tools-reference/kube-apiserver.md`, same repository. **Source of** `--max-requests-inflight` default 400 and `--max-mutating-requests-inflight` default 200.
3. **Kueue, *Fair Sharing* and *Preemption*** — `site/content/en/docs/concepts/fair_sharing.md` and `preemption.md` in <https://github.com/kubernetes-sigs/kueue>. **Source of** the cohort/borrowable-quota model, the weighted share value at `.status.fairSharing.weightedShare` and the `kueue_cluster_queue_weighted_share` metric, admission by lowest share and preemption by highest, the `LessThanOrEqualToFinalShare` and `LessThanInitialShare` strategies, `borrowingLimit`/`reclaimWithinCohort`/`borrowWithinCohort`, and the published proof (via dominant-resource-share inequalities) that two workloads in different ClusterQueues cannot preempt each other in a loop. **Correction to the previous version of this lesson:** it described Kueue's ordering as "usage-weighted fair sharing"; the implementation is a **weighted dominant-resource share**, which differs materially when tenants request different resource mixes.

**Queueing results used and derived here rather than cited**

4. **Little's Law (`L = λW`)**, the M/M/1 waiting-time result (`W = W_service/(1−ρ)`), and **Kingman's approximation** (`E[W_q] ≈ ρ/(1−ρ) × (c_a²+c_s²)/2 × E[S]`) are standard results, restated and worked above rather than quoted. **Little, J.D.C. (1961), *A Proof for the Queuing Formula: L = λW*, Operations Research 9(3):383–387**, DOI <https://doi.org/10.1287/opre.9.3.383>, is the original proof and is worth reading for the exact stationarity conditions — **not fetched this pass** (the DOI resolver and journal are blocked here). **Kingman, J.F.C. (1961), *The single server queue in heavy traffic*, Math. Proc. Cambridge Phil. Soc.** — likewise not fetched.

**Real-world engineering — not fetchable this pass**

5. **Huang, L. et al. (2022), *Metastable Failures in the Wild*, OSDI** — <https://www.usenix.org/conference/osdi22/presentation/huang-lexiang>. The formal definition, the trigger/sustaining-effect taxonomy, and the incident survey. **Blocked.** The figures previously quoted in this lesson (4 of 15 AWS outages; 22 incidents across 11 organisations) are recalled, not verified — re-check before citing.
6. **Bronson, N., Aghayev, A., Charapko, A. & Zhu, T. (2021), *Metastable Failures in Distributed Systems*, HotOS** — the shorter paper that introduced the framing, and the easier first read. **Not fetched.**
7. **Slack Engineering, *Slack's Incident on 2-22-22*** — <https://slack.engineering/slacks-incident-on-2-22-22/>. Correct per-client backoff proving insufficient at fleet scale, and a shed-first recovery. **Blocked.**
8. **Shopify Engineering, *Surviving Flashes of High-Write Traffic Using Scriptable Load Balancers*, parts I and II** — <https://shopify.engineering/surviving-flashes-of-high-write-traffic-using-scriptable-load-balancers-part-i>. Edge admission control, and the fairness correction that qualifies this lesson's LIFO advice. **Blocked.**
9. **Google SRE Book, *Handling Overload* and *Addressing Cascading Failures*** — <https://sre.google/sre-book/handling-overload/>. Graceful degradation, per-customer limits, client-side throttling, and the observation that recovery generally requires dropping load *below* the level that triggered the failure. **Blocked.**
10. **Brooker, M., *Metastability and Distributed Systems*** and ***What is Backoff For?*** — <https://brooker.co.za/blog/2021/05/24/metastable.html>, <https://brooker.co.za/blog/2022/08/11/backoff.html>. The sustaining-loop framing and a precise account of what backoff does and does not accomplish. **Blocked.**
11. **Tene, G., *How NOT to Measure Latency*** — the talk that named coordinated omission, and the reference for the correction implemented in **HdrHistogram** (`recordValueWithExpectedInterval`). **Not fetched**; the arithmetic above is derived, and the mechanism is directly observable by running the two-test comparison in Practice item 3.

**Deeper dives**

12. **Netflix, `concurrency-limits`** — <https://github.com/Netflix/concurrency-limits>. TCP-congestion-control-style adaptive concurrency limits (Gradient/Vegas algorithms) that *discover* `L_max` from observed latency instead of requiring you to configure it — the natural next step once you have done this lesson's arithmetic by hand and watched it drift.
13. **Envoy adaptive concurrency and circuit breaking** — the same ideas as a sidecar feature: concurrency limits, outlier detection, and per-upstream circuit breakers, configurable without touching application code.
14. **Gunther, N., *Guerrilla Capacity Planning*** — the Universal Scalability Law, which extends the utilisation curve with a *coherency* term for the cost of coordination between workers. It is the tool for the case this lesson does not cover: throughput that gets **worse** as you add concurrency, which is what you are seeing when adding replicas makes the system slower.
