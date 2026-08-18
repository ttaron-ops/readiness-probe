---
lesson: "A01.4"
title: "Caching"
module: "A-01"
concept: "modal systems, stampede, coherence, KV-cache"
status: not-started
est_time: "4.5 hrs"
prev: "03-replication-and-partitioning.md"
next: "05-queueing-and-backpressure.md"
artifacts: ["KV-cache stampede sizing + p2p weight-warmup design"]
sources: 15
---

# A01.4 · Caching

> **Concept.** A cache is a modal system: its behavior — and the load it imposes on the origin — changes discontinuously between hit and miss, so you design for the miss and the mode transition, not the hit.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 03 laid out replication as a spectrum, and every scheme on it shared one assumption: a replica is *supposed* to hold correct data, and the entire design effort goes into bounding how wrong or how stale it may be in the meantime.

A cache is a replica that abandons that assumption. It is a copy optimised purely for latency, explicitly permitted to be stale, wrong, or simply absent — the condition replication schemes work hardest to avoid is a cache's normal operating state. That relaxed contract is what makes caching cheap, and it buys a failure mode replication does not have. A replica losing a node degrades gracefully: less margin, same behaviour. A cache losing its contents does not degrade — it *switches modes*, from serving 99% of traffic itself to serving none of it, and the system behind it inherits a load it was never sized for, instantly.

This lesson is about that transition. It also picks up two threads from lesson 03: the hot key you salted for write throughput is the same key that will stampede, and the fan-out arithmetic that ruined your scatter-gather tail is the same arithmetic that governs a herd.

## Why this matters

Hit-mode numbers are the easy ones and the ones that lie. A cache with a 99% hit rate and a 2 ms p50 can be sitting in front of an origin that cannot survive ten seconds at 0% hit rate, and nothing in the steady-state dashboard says so. The event that kills you is the transition — a flush, a synchronised TTL expiry, a deploy that changes a cache key prefix, a fleet restart — and the system's behaviour on the other side of it is a different system.

On a GPU platform this is not an analogy, it is two first-order production problems with money attached. **KV-cache** under HBM pressure directly determines time-to-first-token, and a wrong eviction costs GPU-seconds of prefill recompute rather than a millisecond round trip. **Model-weight distribution** turns every cold start into a multi-terabyte fan-out problem where the naive design serialises your entire fleet behind one object-store egress cap. Both are distributed-cache problems. Both fail as stampedes.

## What's new here (calibration)

**Skip (carried over):** TTL versus write-through/around/back; "caches cut latency"; LRU exists.

**New here:**
- **The modal framing made quantitative** — the hit-rate-to-origin-load curve, why it is brutally non-linear near 100%, and the operational test that follows (load-test cold, not warm).
- **Stampede as arithmetic**, with each defence expressed as a term you can compute: single-flight, jittered TTLs, probabilistic early expiration with its actual recompute rule, stale-while-revalidate, and negative caching.
- **Coherence with the races drawn out** — write-invalidate versus write-update versus write-through against the same write, including the specific interleaving that leaves a stale value in a cache *forever* despite a correct invalidation.
- **KV-cache as a real cache with a published implementation** — vLLM's block hashing, reference counting, free-queue LRU and the deliberate reverse-order free that makes eviction match the sharing structure, plus the memory arithmetic that tells you how many concurrent contexts you actually have.
- **Weight distribution as a bandwidth-topology problem**, with the fan-out arithmetic that turns 11 hours into minutes.

Version note: vLLM internals below are from `vllm-project/vllm` (`docs/design/prefix_caching.md` and `vllm/config/cache.py`) read on GitHub in August 2026. Model-architecture arithmetic is computed from published model configurations and marked as such — re-run it against your own `config.json`. Vendor engineering blogs and the papers cited were not fetchable in this environment and are marked in References.

## Core concepts

### A cache is a mode switch wearing an interface

The framing that makes everything else fall out: a cache does not have *a* behaviour, it has two, and the system's real capacity requirement is set by the second one.

```
  ORIGIN LOAD as a function of hit ratio h, at fixed arrival rate λ = 10,000 req/s

     origin_rps = λ × (1 − h)

  h      1−h      origin_rps     vs. h=0.99
  ────   ─────    ──────────     ──────────
  0.99   0.01        100          1×      ← what your dashboard shows
  0.98   0.02        200          2×      ← one point of hit rate = DOUBLE
  0.95   0.05        500          5×
  0.90   0.10      1,000         10×
  0.50   0.50      5,000         50×
  0.00   1.00     10,000        100×      ← cold cache: the mode you never test

  10000 ┤●                                        The curve is a straight line
        │ ●                                       in h — but you live at the far
  origin│  ●                                      right end of it, where every
   rps  │    ●                                    small change in h is a LARGE
   5000 ┤      ●                                  multiple of origin load.
        │         ●                               d(origin)/dh = −λ, so at
        │              ●                          λ=10k, one point of hit rate
        │                    ●                    is ±100 rps in absolute terms
      0 ┤                          ●●●●●●●●●●●●●● and ±100% in relative terms
        └┬────┬────┬────┬────┬────┬────┬────┬───  when you are at 99%.
         0   0.2  0.4  0.6  0.8  0.9  0.95 0.99 1.0
                        hit ratio h
```

**The number that matters is not `h`, it is `1 − h`, and you should plot and alarm on it.** A hit-rate dashboard showing 99% → 98% looks like a rounding error; a miss-rate dashboard showing 1% → 2% looks like what it is, a doubling.

Three consequences:

1. **Design for the miss.** The question is never "what is p50 at 99% hit rate." It is "does the origin survive at `h = 0`, and for how long, and what does it do when it cannot?"
2. **Hit ratio is a first-class SLI**, dashboarded and alarmed, because it is the only leading indicator of an origin overload that has not happened yet. **A cache in front of an origin that cannot survive a flush has *reduced* your availability** — you have added a component whose failure mode is a load multiplier.
3. **Load-test cold.** The steady-state benchmark measures the mode that is not dangerous. Flush and measure; that is the number that belongs in the capacity plan.

And the failure that makes this urgent rather than academic: if the origin cannot serve the miss storm, the cache never refills, so the miss storm never ends. The system is stuck in the expensive mode with no path back — a **metastable failure**, which lesson 05 formalises. The cache is not a passive victim here; it is the amplifier.

### The stampede, and each defence as a term you can compute

A **stampede** (thundering herd, dog-pile) happens when many concurrent requests miss the same key at the same instant and all proceed to the origin. It has two independent causes, and the defences map onto them one-to-one:

- **Concurrency** — N requests for one key are in flight simultaneously; the first miss has not finished populating the cache when the others check.
- **Synchronisation** — many keys (or many replicas' copies of one key) expire at the same instant, because they were all populated at the same instant with the same TTL.

```
  BASELINE: hot key, TTL expires at t=0, 200 concurrent callers, origin recompute = 800 ms

   t=0        expiry
   caller 1   ├── miss ──▶ origin ─────────────────────── 800 ms ──▶ set
   caller 2   ├── miss ──▶ origin ───────────────────────────────▶
   caller 3   ├── miss ──▶ origin ───────────────────────────────▶
     …            (all 200 miss inside the 800 ms window, because the cache
   caller 200 ├── miss ──▶ origin  stays empty until the FIRST one finishes)
                            ▲
                            └── 200 concurrent recomputes of the SAME value.
                                Origin sees 200× its steady-state load for 800 ms.

  DEFENCE 1 — SINGLE FLIGHT (request coalescing)
   caller 1   ├── miss ──▶ acquires the in-flight slot ──▶ origin ── 800 ms ──▶ set
   caller 2   ├── miss ──▶ finds slot taken ──▶ WAITS on it ──────────────────┤
     …                                                                        │
   caller 200 ├── miss ──▶ WAITS ─────────────────────────────────────────────┤
                                                        all 200 answered ─────┘
   origin sees 1 request. Latency for all 200 ≈ 800 ms.
   Cost: 199 callers now block on one origin call — you converted a load
   problem into a latency problem, and you created a single point of failure
   for that key (if the flight fails, all 200 fail together — so cap the wait,
   and decide whether a timeout means "serve stale" or "error").

  DEFENCE 2 — JITTERED TTL (fixes synchronisation, not concurrency)
   TTL = base ± rand(0, J). With base 300 s and J = 60 s, the 200 replicas'
   copies expire spread across a 60 s window instead of one instant.
   Herd size per instant ≈ 200 × (recompute_window / jitter_window)
                          = 200 × (0.8 s / 60 s) ≈ 2.7 concurrent recomputes.
   Cost: nothing. There is no reason not to jitter every TTL you set.

  DEFENCE 3 — STALE-WHILE-REVALIDATE (removes the miss from the user path)
   at expiry:   caller 1 gets the STALE value immediately, and triggers an
                async refresh; callers 2..200 also get the stale value.
   user-visible latency: unchanged (a hit).
   staleness: bounded by the refresh time, and by a hard stale-if-error limit.
   Cost: you must be willing to serve a known-expired value — a correctness
   decision, not a performance one. Combine with single-flight so the async
   refresh happens once.
```

**Probabilistic early expiration (XFetch)** attacks the synchronisation cause from a different angle: instead of expiring at a fixed instant, each reader independently decides, with rising probability as expiry approaches, to refresh *early* while still serving the cached value. The published rule (Vattani, Chierichetti & Lowenstein, VLDB 2015 — **not fetched this pass**, see References) is:

```
   refresh now  if   now − delta × beta × ln(rand())  ≥  expiry

     delta  = measured time the last recompute took  (so expensive keys
              start trying earlier — the cost is self-tuning)
     beta   = tuning constant, ~1 by default; >1 refreshes earlier
     rand() = uniform in (0,1), so −ln(rand()) is exponentially distributed
              with mean 1 — the "coin flip" gets heavier as `now` approaches
              `expiry`

   Effect: exactly one reader (usually) refreshes shortly BEFORE expiry, while
   every other reader still gets a hit. There is never an instant at which the
   key is absent, so the herd cannot form.
```

The whole family, and when each is the right tool:

| Defence | Attacks | Cost | Use when |
|---|---|---|---|
| Single flight / coalescing | Concurrency | 199 callers wait on 1; correlated failure per key | Always — the general backstop |
| Jittered TTL | Synchronisation | None | Always. There is no argument against it |
| Probabilistic early expiration | Synchronisation | Slightly more recomputes overall; needs `delta` tracked | Known-hot keys with expensive, measurable recompute |
| Stale-while-revalidate | Both, from the user's side | Serves a knowingly stale value | When staleness is cheaper than latency — most read paths |
| Negative caching | Miss storms for keys that do not exist | A create is invisible until the negative entry expires — keep this TTL short | Any key space clients can probe (IDs, lookups) |
| Pre-warming | The cold start itself | Complexity, and it must be part of the deploy | Deploys and scale-outs where you control the timing |

**Peak recompute load, as one expression:**

```
  peak_concurrent_recomputes ≈  N
                              ─────────────────────────────────────────
                               sharing_factor × jitter_spread

   N               = callers/replicas whose entry expires in the same window
   sharing_factor  = callers collapsed onto one in-flight fetch (1 = none)
   jitter_spread   = jitter_window ÷ recompute_time, i.e. how many
                     non-overlapping recompute windows the expiries spread over
                     (1 = fully synchronised)

  Worked: N=200, recompute 800 ms.
    no defences:                    200 / (1 × 1)                = 200
    jitter only, J = 60 s:          200 / (1 × 75)               ≈ 2.7
    single-flight only:             200 / (200 × 1)              = 1
    both:                                                          1, with the
      jitter also protecting you when single-flight's scope is per-process
      rather than per-fleet — which it usually is.
```

That last parenthesis is the one people get wrong in production. `golang.org/x/sync/singleflight`, `functools.lru_cache`, and every in-process coalescing library dedupe **within one process**. With 200 replicas you have reduced 200 × C concurrent misses to 200. **Fleet-wide coalescing needs a shared lock** — a Redis `SET key NX EX 10` acting as a lease, where the winner recomputes and the losers poll or serve stale — and now you have to reason about what happens when the lease holder dies (the TTL on the lock bounds it) and whether a lost lock means error or stale.

### Coherence: three strategies against one write

The second hard problem. Here is the same write under the three common strategies, with the race in each one drawn rather than asserted.

```
  Setup: cache C (holds x=1), database D (holds x=1). Writer W sets x=2.
  Reader R reads x throughout.

 ── WRITE-THROUGH ────────────────────────────────────────────────────────────
   W ──▶ C: set x=2 ──▶ D: write x=2 ──▶ ✓
   Cache and DB updated in one path; C is never behind D.
   Race: if the D write fails after the C write succeeds, the cache now holds
   a value the database never accepted. Ordering matters: write D FIRST, then
   C, and a failure leaves you merely stale rather than fabricated.
   Cost: every write pays cache-write latency; caches every write whether or
   not it will ever be read (bad for write-heavy, low-reuse data).

 ── WRITE-AROUND + INVALIDATE (the common production choice) ─────────────────
   W ──▶ D: write x=2 ──▶ C: DELETE x ──▶ ✓
   next reader misses, loads x=2 from D, populates C.
   Cost: the first reader after every write takes a miss.
   THE RACE THAT BITES — a read and a write interleaved:

     R: read x → MISS in C
     R: ──▶ D: read x        ────────────▶ returns 1   (slow read, e.g. GC pause)
                    W: ──▶ D: write x=2
                    W: ──▶ C: DELETE x                (deletes nothing; C empty)
     R: ──▶ C: SET x = 1     ←──────────── populates the cache with the OLD value
                                            AFTER the invalidation has run.

     Result: C holds x=1, D holds x=2, and NO FURTHER EVENT WILL FIX IT
     until the TTL expires. The invalidation was correct, well-ordered, and
     delivered — and it still lost, because it arrived while a stale read was
     in flight.

     This is why memcached-scale systems add LEASES: on a miss, the cache hands
     the reader a token; an invalidation between issue and fill INVALIDATES THE
     TOKEN, so the reader's fill is rejected. It is also why "TTL as a backstop"
     is not laziness — it is the bound on exactly this race.

 ── WRITE-UPDATE (push the new value) ────────────────────────────────────────
   W ──▶ D: write x=2 ──▶ C: SET x=2 ──▶ ✓
   No subsequent miss; the cache is warm immediately.
   Race: two concurrent writers W1(x=2) and W2(x=3) can reach D and C in
   DIFFERENT orders — D ends at 3 while C ends at 2, permanently, again until
   TTL. Invalidate does not have this problem, because "delete" commutes with
   "delete" while "set 2" does not commute with "set 3".
   Cost: pushes values nobody will read; amplifies traffic for write-heavy keys.

 ── AND THE TIERED-CACHE PROBLEM ─────────────────────────────────────────────
   in-process L1 (per replica, 200 of them)
         │
      shared L2 (Redis / memcached)
         │
       origin D

   An invalidation sent to L2 does NOT reach 200 L1s. Each L1 will keep serving
   the old value for up to its own TTL. Options:
     · very short L1 TTLs (seconds) — simple, costs L2 traffic
     · a pub/sub invalidation bus L1s subscribe to — correct, but now you own a
       delivery problem: a missed message is a permanently stale L1
     · versioned keys: put a version in the key itself, so a "new version" is a
       new key and nothing needs invalidating anywhere. Old entries age out.
   Versioned keys are the only one of the three with no invalidation race,
   which is why content-addressed and version-addressed caching wins whenever
   you can afford the extra key churn.
```

**The production default is TTL *and* explicit invalidation, and now you know why it is not belt-and-braces cargo cult.** Explicit invalidation cuts the common-case staleness window to near zero but cannot be relied on: it races (as drawn), it can be lost in transit, and it does not reach every tier. The TTL is the bound on every one of those failures. Choose the TTL as "how long am I willing to be wrong when an invalidation is lost," not as "how long is this value probably good for."

### Eviction: the policy question is what unit is shared

Eviction is usually taught as an algorithm menu (LRU, LFU, ARC, 2Q, S3-FIFO, TinyLFU). The more useful framing for a system designer:

**The eviction unit should be the unit that is actually shared, and the policy should be weighted by the cost of a miss on that unit.** A generic cache treats entries as interchangeable and misses as equal-cost, so recency is a reasonable proxy. Neither assumption holds for the caches on a GPU platform.

| Cache | Unit | What a miss costs | Right policy |
|---|---|---|---|
| Redis in front of Postgres | One key | one round trip + one query, ~ms | LRU/TinyLFU, cost-blind is fine |
| CDN edge | One object | one origin fetch, size-dependent | Size-aware (GDSF-style) — evicting one 1 GB object beats evicting a thousand 1 KB ones |
| **LLM KV cache** | A block of tokens **within a prefix chain** | **GPU-seconds of prefill recompute**, proportional to the tokens lost | Prefix/reference-aware LRU (below) |
| Model weights on node NVMe | One version of one model | a multi-GB network pull, tens of seconds | Rarely evict; pin by scheduling policy, not by recency |

### The KV cache, as actually implemented

This is a real distributed-cache problem with a published implementation, so learn it from the code rather than the metaphor. vLLM's prefix cache works like this:

```
  Blocks are FIXED-SIZE runs of tokens (DEFAULT_BLOCK_SIZE = 16 tokens).
  A block is cacheable only when FULL. Its identity is a HASH CHAIN:

     block_hash(i) = H( parent_hash = block_hash(i−1),
                        block_tokens = the 16 token ids in this block,
                        extra_keys   = LoRA id, multimodal input hashes,
                                       cache_salt )

  Prompt: "You are a helpful assistant. …"   block size 16
   ┌──────────────┬──────────────┬──────────────┬──────────────┐
   │  tokens 0-15 │ tokens 16-31 │ tokens 32-47 │ tokens 48-52 │
   │  hash h1     │ hash h2      │ hash h3      │  PARTIAL —   │
   │  =H(∅,t0-15) │ =H(h1,t16-31)│ =H(h2,t32-47)│  not cached  │
   └──────────────┴──────────────┴──────────────┴──────────────┘
        ▲               ▲              ▲
        └───────────────┴──────────────┘
     Any request whose prompt starts with the same tokens produces the SAME
     hashes for as many leading blocks as it shares. Prefix sharing is therefore
     automatic and exact — no tree walk, just hash lookups down the chain until
     the first miss.

  MEMORY MANAGEMENT — the parts that matter operationally:
   · KVCacheBlock { block_id, block_hash, ref_cnt, prev_free, next_free }
     All blocks are preallocated into a pool at startup.
   · ref_cnt counts requests currently using the block. A cached block still
     sitting in the free queue can be reused: on a hit it is "touched" —
     ref_cnt++ and it is REMOVED from the free queue so it cannot be evicted.
   · The free queue is a doubly linked list = LRU. Eviction pops the HEAD.
   · When a request finishes, its blocks return to the TAIL of the free queue
     IN REVERSE ORDER. That is deliberate: the LAST block of a request encodes
     the most tokens of context and is therefore the LEAST likely to be shared
     with anyone else, so it should be evicted first. The FIRST blocks — the
     system prompt everybody shares — end up furthest from the eviction head.

  ⇒ The policy is LRU, but the DATA STRUCTURE encodes the sharing structure,
    so the effect is prefix-aware without a separate priority scheme. This is
    the general lesson restated: match the eviction unit and ordering to what
    is actually shared, and a simple policy becomes a good one.

  Multi-tenancy note: `cache_salt` is mixed into the hash so that cache reuse
  is limited to requests that agree on the salt. Without it, prefix sharing
  across tenants is a side channel — a tenant can detect that another tenant
  has already prefilled a given prefix by observing its own TTFT.
```

**How much KV cache do you actually have?** The arithmetic, carried through with units:

```
  bytes_per_token = 2 (K and V)
                  × num_layers
                  × num_kv_heads          ← GQA: this is the KV head count,
                  × head_dim                NOT the query head count
                  × dtype_bytes

  Worked for a 70B-class model with GQA (80 layers, 8 KV heads, head_dim 128,
  fp16 — computed from the published architecture; check your own config.json):

     per layer per token = 2 × 8 × 128 × 2 B = 4,096 B = 4 KiB
     × 80 layers                            = 327,680 B ≈ 320 KiB per token

  A 2,048-token shared system prompt therefore occupies
     2,048 × 320 KiB ≈ 640 MiB of KV — cluster-wide, once, if it is shared.
     Without prefix sharing: 640 MiB PER CONCURRENT REQUEST.

  Capacity on one 8×80 GB node, tensor-parallel:
     total HBM                    640 GB
     × gpu_memory_utilization 0.92  ≈ 589 GB usable by the engine
     − weights (fp16, 70B)         ≈ 140 GB
     − activations/workspace       ≈  30 GB (workload dependent)
     ⇒ KV budget                   ≈ 419 GB
     ÷ 320 KiB per token           ≈ 1.34 M tokens of KV
     ÷ 2,048 tokens per context    ≈ 650 concurrent 2K-token contexts

  Now the point: if 500 of those requests share one 2,048-token system prompt,
  prefix caching stores it ONCE (640 MiB) instead of 500 times (312 GB).
  Prefix sharing is not a latency optimisation here — it is the difference
  between fitting your concurrency target and not.
```

**What a KV-cache miss costs** is the other half. A miss means re-running prefill over the lost tokens: compute proportional to `tokens × model FLOPs/token`, occupying the same GPUs that are trying to serve decode for everyone else. So a KV eviction does not just slow one request — it converts memory pressure into *compute* pressure on a shared resource, which shows up as TTFT degradation for requests that had nothing to do with the eviction. **That coupling is why KV-cache eviction is a capacity-planning decision, not a tuning knob.**

### Weight distribution: a cache problem where the policy is irrelevant

The other GPU-platform cache is model weights on node-local storage. Here no eviction algorithm helps, because the problem is not what you keep — it is how the bytes get there.

```
  NAIVE: every node pulls from object storage independently.

     ┌──────────┐
     │  S3 /GCS │  aggregate egress cap: 5 GB/s
     └────┬─────┘
      ┌───┴────┬────────┬────────┬── … ──┐      1,000 nodes
      ▼        ▼        ▼        ▼       ▼      200 GB each
    node1    node2    node3    node4   node1000

    total bytes = 1,000 × 200 GB = 200 TB
    time        = 200 TB ÷ 5 GB/s = 40,000 s ≈ 11.1 hours
    …and every node is retrying against the same cap, so the effective
    throughput is worse than the cap, not better.

  P2P / TREE FAN-OUT: seed a few from the origin, then replicate node-to-node.

     S3 ──▶ n1 ──┬──▶ n2 ──┬──▶ n4 …        each round DOUBLES the number of
                 │         └──▶ n5 …        nodes holding the weights
                 └──▶ n3 ──┬──▶ n6 …
                           └──▶ n7 …

    rounds needed   = ⌈log2(1000)⌉ = 10
    per-round time  = 200 GB ÷ per-node send bandwidth
                    = 200 GB ÷ 10 GB/s ≈ 20 s     (≈80 Gbps effective NIC)
    total           ≈ 10 × 20 s = 200 s ≈ 3.3 minutes
    origin egress   = 200 GB, ONCE.

    Real torrent-style systems beat the pure tree by chunking: a node starts
    serving chunk 1 while still receiving chunk 2, so the pipeline fills and
    the constant factor drops well below the naive round model.

  Then add the two cheap wins:
    · CONTENT- OR VERSION-ADDRESSED local NVMe cache. Key the local copy by
      model version/digest, never by "latest". A reboot is then a local read
      (seconds), and a version change is the only event that costs network.
      This is the versioned-key idea from the coherence section: no
      invalidation protocol exists, because a new version is a new key.
    · JITTERED WARM-UP. Without it, your seeders stampede the origin at t=0
      exactly like any other synchronised expiry.
```

The design rule: **when the cache miss is bandwidth-bound rather than compute-bound, the fix lives in the topology, not the policy.** Adding an eviction algorithm to an 11-hour warm-up changes nothing; changing the fan-out shape changes it by two orders of magnitude.

## Perspectives

**The modal-systems view.** Hit mode and miss mode are two different systems sharing an interface, and every metric you routinely look at describes the first one. A cache is a *load transformer*: it converts `λ` into `λ(1−h)`, and its failure mode is that transformation reverting instantly. Design reviews should ask, for every cache: what is the origin's capacity at `h = 0`, how long can it sustain that, and what does it do when it cannot? If the answer to the last is "queue and retry," you have a metastable failure waiting for a trigger, and lesson 05 is about it.

**The coherence view.** Every invalidation strategy has a race, and the races are not exotic — a slow read racing a well-formed invalidation leaves a stale entry that nothing but the TTL will fix. That is why real systems layer defences (leases or tokens on fills, TTL as a backstop, versioned keys where affordable) rather than picking one strategy. Coherence is not a feature you ship once; Meta's public account of taking TAO's cache-consistency from "six nines" to "ten nines" describes years of dedicated invalidation engineering, which is the honest scale of the problem.

**The inference-serving view.** KV cache is not a cache in front of a database; it is a cache competing for the same physical memory as the weights and activations that serve the *next* request. Its miss cost is denominated in GPU-seconds of prefill on a shared resource, so an eviction externalises cost onto unrelated requests as TTFT. Its unit of sharing is a token prefix, not an opaque key. And it has a multi-tenancy dimension a Redis cache does not: shared prefixes across tenants are a measurable side channel, which is why implementations expose a cache salt.

**The fleet-operations view.** Getting hundreds of gigabytes onto hundreds of nodes quickly is a bandwidth and topology problem, and it has nothing in common with cache-policy tuning. The levers are fan-out shape (tree/torrent instead of N independent pulls), locality (version-addressed local NVMe so reboots are free), scheduling (place work where the weights already are — a cache-aware scheduler beats a faster cache), and jitter (so your warm-up does not stampede its own origin). The instinct "just add a cache" misses that the constraint is the pipe.

## Real-world use cases

- **vLLM's prefix-cache implementation** (verified, read from `vllm-project/vllm`). Block size default 16 tokens; only full blocks are cached; block identity is a hash chain of `(parent_hash, block_tokens, extra_keys)` with `sha256` the default algorithm since v0.11; `enable_prefix_caching` defaults to true; `gpu_memory_utilization` defaults to 0.92; eviction is LRU over a doubly linked free queue, with finished requests returned to the queue **in reverse block order** so the least-shareable blocks are evicted first; `cache_salt` isolates cache reuse between trust groups. **What it shows:** a production cache where the data structure encodes the sharing structure, making a simple policy behave like a smart one — and a concrete instance of the "cheaper hash is a security trade" decision (the docs explicitly warn that non-cryptographic hashing raises collision risk that "can cause undefined behavior or even leak private information in multi-tenant environments").
- **Meta, *Cache made consistent* (TAO)** — the account of raising TAO's cache-consistency guarantee from roughly six nines to roughly ten nines through sustained invalidation engineering, including tooling to *measure* inconsistency, which is the part most teams never build. **Not fetched this pass** (engineering.fb.com blocked); cited for the shape of the argument — coherence is a multi-year investment, not a checkbox.
- **Facebook/Meta, *Scaling Memcache at Facebook* (NSDI 2013)** — the origin of the **lease** mechanism described in the coherence section: on a miss the cache returns a token, and an invalidation between token issue and fill rejects the fill, closing exactly the stale-set race drawn above. Also the source of the classic tiered/regional memcache architecture and of "gutter" pools for absorbing failure. **Not fetched this pass** (usenix.org blocked); the lease mechanism is described from standard knowledge.
- **Modal, *GPU Memory Snapshots*** and **CoreWeave's Tensorizer** — two vendors attacking cold start from opposite ends: snapshot and restore full GPU state (weights in VRAM, CUDA context) versus stream-deserialise weights from object storage faster than a naive loader. Both report large multiples of improvement on cold start. **Not fetched this pass** (both domains blocked); the reported figures — roughly 10× and greater-than-5× respectively — are recalled from the previous version of this lesson and should be re-verified before you cite them. The *mechanism* each attacks is exactly the cold-weight-pull problem the arithmetic above quantifies.
- **AWS Builders' Library, *Caching challenges and strategies*** — a practitioner catalogue of the failure modes in this lesson (modal behaviour, stampede, staleness, cache-as-a-load-multiplier) drawn from AWS service operations. **Not fetched this pass**; listed because it is the best single operational companion to this material.

## Worked example

### Part A — sizing an inference tier's prefix cache and its stampede risk

**Setup.** 200 replicas serve an agent workload. Every request shares a 2,048-token system prompt. Prefill for that prefix costs ~800 ms of GPU on one replica. The prefix's cached KV is invalidated whenever the prompt template is redeployed. Steady-state arrival is 4,000 requests/s across the fleet, and the TTFT SLO is 500 ms.

**A1 — What does the shared prefix save?**

```
  KV per token (70B-class, GQA, fp16, from the arithmetic above)  ≈ 320 KiB
  2,048-token prefix                                              ≈ 640 MiB

  Without prefix sharing, per replica with 650 concurrent contexts:
      650 × 640 MiB = 416 GB of KV spent re-storing the SAME prefix
      — which does not fit; the practical effect is that concurrency collapses
        to whatever does fit.
  With prefix sharing: 640 MiB once. The rest of the KV budget goes to the
      per-request suffixes, which is what you actually want to spend it on.

  And in compute: at 800 ms of prefill per unshared prefix, 4,000 req/s would
  need 4,000 × 0.8 = 3,200 GPU-seconds per second of prefill for the prefix
  alone. Prefix caching takes that to ~0. Prefix caching is not an
  optimisation for this workload — it is the reason the workload is feasible.
```

**A2 — The template redeploy, which is a flush.**

Redeploying the prompt template changes the token ids, so every block hash changes, so **every replica's cached prefix is instantly worthless.** This is a synchronised, fleet-wide, 100%-miss event on the hottest key in the system. With no defences:

```
  N = 200 replicas, each immediately re-prefilling the new prefix
  peak = N / (sharing_factor × jitter_spread) = 200 / (1 × 1) = 200
       ⇒ 200 concurrent 800 ms prefills, all at t=0

  Those prefills contend with in-flight decode on the same GPUs. TTFT for
  every request in flight — including ones that need nothing new — spikes,
  because prefill and decode share the SM and the memory bandwidth. The blast
  radius is the whole tier, not just the requests that missed.
```

**A3 — Defences, costed.**

```
  1. Staged rollout of the template (the real fix).
     Roll the new prompt to 5% of replicas, then 25%, then 100%.
     peak = 200 × 0.05 = 10 concurrent prefills in the first wave.
     Cost: nothing but deploy machinery you should already have.
     ⇒ jitter_spread ≈ 20 by construction.

  2. Fleet-wide single flight on the prefix, if you run a shared KV store.
     One replica prefills; the KV blocks are published to the shared tier;
     the rest fetch them over the network instead of recomputing.
     peak = 200 / 200 = 1 prefill.
     Cost: you now need a KV transfer path, and fetching 640 MiB over the
     network must be faster than 800 ms of prefill — at 10 GB/s that is 64 ms,
     so yes, comfortably, PROVIDED the fabric is not already saturated.

  3. Pre-warm before cutover.
     Prefill the new prefix on each replica while the old one is still serving,
     then switch. peak = 0 at the transition; the cost is transient double
     KV occupancy (2 × 640 MiB), which is 0.3% of a 419 GB budget.

  Combined: peak ≈ 1–10 concurrent prefills instead of 200, and the SLO holds.
```

**A4 — The alarm that would have caught it.** Not "hit ratio," which is a lagging indicator once you are already in miss mode, but **prefix-cache miss *rate* (1−h) per replica**, alarmed on a multiplier of its own baseline, plus **prefill GPU-seconds/second** as the load signal. Both move at t=0 of the redeploy, before TTFT breaches.

### Part B — cold-start weight distribution

**Setup.** 1,000 nodes fault-restart simultaneously (a rack power event, or a cluster-wide rollout). Each needs a 200 GB weight set. Object-store aggregate egress is capped at 5 GB/s. Intra-fleet NIC bandwidth is ~10 GB/s per node.

```
  NAIVE
    bytes  = 1,000 × 200 GB = 200 TB
    time   = 200 TB ÷ 5 GB/s = 40,000 s ≈ 11.1 h
    Actual behaviour is worse: 1,000 clients contend for the cap, each gets
    ~5 MB/s, every one of them times out and retries, and the retries consume
    the cap that the original requests needed. This is a metastable failure
    with a cache trigger — the exact pattern lesson 05 names.

  TREE / TORRENT FAN-OUT
    rounds = ⌈log2(1000)⌉ = 10
    time   ≈ 10 × (200 GB ÷ 10 GB/s) = 10 × 20 s = 200 s ≈ 3.3 min
    origin egress = 200 GB once  (vs 200 TB)
    speed-up ≈ 200×

  WITH VERSION-ADDRESSED LOCAL NVMe
    A restart that does NOT change the model version reads from local disk:
      200 GB ÷ ~2 GB/s NVMe ≈ 100 s, ZERO network, ZERO origin load.
    ⇒ the fan-out cost is paid once per model version, not once per restart.
    This is the single highest-leverage change, and it is a cache-key design
    decision: key on the version digest, never on "latest".

  WITH JITTERED WARM-UP
    Without jitter the seed nodes hit the origin simultaneously; with a random
    0–60 s start offset they do not. Free, and it is the difference between the
    first round taking 20 s and taking as long as the origin's queue.

  CAPACITY CHECK — is 3.3 minutes good enough?
    1,000 nodes × 3.3 min of unavailability = 55 node-hours of lost capacity
    per full cold start. At a notional $2/GPU-hour and 8 GPUs/node that is
    ~$880 per event. Compare the naive 11.1 h: ~$178,000 of idle capacity
    per event. THAT is the number that funds the p2p work.
```

### Part C — the coherence decision, made explicitly

The same tier caches per-tenant quota state in Redis, read on every admission decision (4,000/s) and written when a job starts or finishes (~50/s).

```
  Option 1: TTL only, 60 s.
    Staleness window: up to 60 s. At 50 writes/s that is up to 3,000
    unreflected quota changes — a tenant can exceed quota for a minute.
    Verdict: unacceptable for admission control.

  Option 2: write-around + invalidate, no TTL.
    Common-case staleness: milliseconds. Worst case: FOREVER, via the
    stale-set race drawn in Core concepts, or a dropped invalidation.
    Verdict: unacceptable — the failure is unbounded and silent.

  Option 3: invalidate + short TTL (5 s) + version check on the decision path.
    Common case: milliseconds (invalidate).
    Bounded failure: 5 s (TTL) for a lost invalidation.
    For the decision that must not be wrong — the actual admission — read the
    authoritative counter with a compare-and-swap rather than trusting the
    cache (lesson 01's rule: name the object whose CAS serialises the write).
    Cost: 50 authoritative reads/s on the write path instead of 4,000 on the
    read path — 1.25% of the traffic pays for correctness.
    Verdict: ship this.
```

That last line is the transferable pattern, and it is the same one lesson 01 reached from the other direction: **cache the evaluation, never the decision.** Serve the fast, possibly-stale value to everything that is scoring, filtering or displaying, and pay for the authoritative read exactly once, at the point where being wrong changes state.

## Practice

*Feeds the [staff design portfolio](../practice/staff-design-portfolio/README.md).*

Design the KV-cache and weight-distribution layer for the serving plane. Required sections:

1. **Modal analysis.** State your `λ` and steady-state `h`; compute origin load at `h`, at `h − 0.01`, and at `h = 0`. State the origin's measured capacity and how long it survives at `h = 0`. Name your hit-rate SLI, express it as miss rate `1−h`, and give the alarm threshold as a multiple of baseline.
2. **The cold-flush test.** Describe the load test that proves the origin survives 0% hit ratio: how you flush, what you measure, what the pass criterion is. Run it if you can, and record what actually happened — this is the section that separates a design from a hope.
3. **Stampede plan.** Pick your defences and show the peak-load arithmetic before and after, using `N / (sharing_factor × jitter_spread)`. State explicitly whether your single-flight is per-process or fleet-wide, and if it is per-process, say what your real fleet-wide peak is.
4. **KV eviction.** State the eviction unit, the policy, and — in tokens and GPU-seconds — what a wrong eviction costs. Compute your KV budget from `2 × layers × kv_heads × head_dim × dtype_bytes` and your GPU memory, and state how many concurrent contexts of your typical length that gives you, with and without prefix sharing.
5. **Weight cold start.** Fan-out topology, local cache key (version-addressed — say so explicitly), jittered warm-up window, and the origin egress figure you are protecting. Compute naive time vs fan-out time vs local-hit time, and convert the difference into idle-GPU cost per event.
6. **Coherence, separated from load.** One paragraph that treats correctness (which values may be stale, for how long, bounded by what) entirely separately from load (stampede, capacity). A staff answer never lets these two share a paragraph, because their mitigations are unrelated.

## Common pitfalls

1. **Benchmarking at steady-state hit ratio.** Symptom: a capacity plan that says the tier handles 10,000 rps, and an outage the first time the cache is flushed. Mechanism: origin load is `λ(1−h)`, so the benchmark measured 1% of the load the failure mode produces. Test cold.
2. **Alarming on hit ratio instead of miss ratio.** Symptom: nobody notices 99% → 98%. Mechanism: that is a doubling of origin load displayed as a one-point change. Plot `1−h`.
3. **"Single-flight solves stampede."** It solves the *concurrency* cause, within one process. Symptom: 200 replicas each make exactly one origin call at the same instant and the origin still falls over. Mechanism: in-process coalescing does not coordinate across replicas; synchronised expiry is untouched by it. Add jitter always, and a shared lease if you need fleet-wide dedup.
4. **Never jittering TTLs.** Symptom: periodic, precisely-spaced origin load spikes matching your TTL. Mechanism: everything populated together expires together. There is no cost to jitter; treat an unjittered TTL as a defect.
5. **"TTL alone is enough."** TTL bounds *worst-case* staleness only, so making it short enough for correctness makes it short enough to destroy your hit rate. Symptom: a tug-of-war between the correctness people and the performance people. Mechanism: you need invalidation for the common case and TTL as the bound on invalidation's failures. Use both, and pick the TTL as "how long may I be wrong when an invalidate is lost."
6. **Assuming a correct invalidation cannot lose.** Symptom: a stale value that survives long after a verified-delivered invalidation. Mechanism: the stale-set race — a read that missed before the write repopulates the cache after the delete. Fix with leases/tokens on fill, or with versioned keys that never need invalidating.
7. **Write-update for concurrent writers.** Symptom: cache and database permanently disagree with no invalidation ever having failed. Mechanism: two writers can reach the store and the cache in different orders; `set` does not commute, `delete` does. Prefer invalidate.
8. **Forgetting the L1 tier when invalidating.** Symptom: a fix that works in staging (one replica) and fails in production (200). Mechanism: invalidating L2 does not reach in-process L1s. Short L1 TTLs, an invalidation bus, or versioned keys.
9. **"KV-cache eviction is just LRU, it's fine."** It is LRU, but the free-queue ordering is what makes it prefix-aware, and a wrong eviction costs GPU-seconds of prefill that degrade TTFT for *unrelated* requests sharing the device. Symptom: TTFT regressions with no change in request mix. Mechanism: memory pressure converted into compute pressure on a shared resource.
10. **Treating p2p weight distribution as premature optimisation.** Symptom: an 11-hour cold start discovered during an incident. Mechanism: N independent pulls against a fixed egress cap serialise, and the retries make it worse. The arithmetic — `N × size ÷ cap` versus `log₂(N) × size ÷ per-node-bandwidth` — is two orders of magnitude at 1,000 nodes and is worth doing before you need it.

## Self-check

- **What does "a cache is a modal system" mean operationally, and what test follows?** **Answer:** The cache has two operating modes that impose completely different load on the origin — `λ(1−h)` at steady state versus `λ` when cold — and it can switch between them discontinuously (flush, synchronised expiry, key-prefix change, fleet restart). Nothing in hit-mode metrics predicts miss-mode behaviour. The test that follows: load-test with the cache cold and empty, measure whether the origin survives, and for how long. If the origin cannot serve the miss storm, the cache never refills and the system cannot self-recover — a metastable failure in which the cache is the amplifier, not the victim.

- **Origin load is 100 rps at 99% hit rate. What is it at 98%, and why is that scarier than it sounds?** **Answer:** `λ(1−h)` with `λ = 10,000`: 200 rps — double. Near `h = 1`, the relative change in origin load equals the relative change in *miss* rate, which is enormous for small absolute changes in hit rate. That is why the SLI should be `1−h`, and why capacity must be planned against `h = 0` (10,000 rps here, 100× steady state) rather than against a hit-mode benchmark.

- **Single-flight and probabilistic early expiration both fight stampede — when do you use each?** **Answer:** Single-flight attacks *concurrency*: many callers missing the same key at once are collapsed onto one origin fetch. It is the general backstop and it acts after the herd has formed, so all callers pay the recompute latency and share its failure. Probabilistic early expiration attacks *synchronisation*: each reader independently decides, with probability rising as expiry approaches (`now − delta × beta × ln(rand()) ≥ expiry`), to refresh early while the cached value is still being served — so there is never an instant when the key is absent and the herd cannot form. Use early expiration for known-hot keys with an expensive, measurable recompute (`delta` is the measured recompute time, which makes the mechanism self-tuning); use single-flight everywhere as the backstop; and jitter every TTL regardless, since it costs nothing.

- **Why TTL *and* explicit invalidation, rather than either alone?** **Answer:** Invalidation gives a near-zero common-case staleness window but cannot be trusted: it races a concurrent slow read (the reader repopulates with the pre-write value *after* the delete lands, leaving a stale entry nothing will fix), it can be lost in transit, and it does not reach in-process L1 tiers. TTL bounds every one of those failures, but alone it forces a choice between a long staleness window and a destroyed hit rate. Together: invalidation handles the common case in milliseconds; TTL caps the failure case. Choose the TTL as "how long am I willing to be wrong when an invalidate is lost." Stronger options where you can afford them: leases/tokens on fill (rejecting a fill whose token was invalidated mid-flight) or versioned keys, which eliminate invalidation entirely.

- **500 replicas share a KV prefix that expires with no jitter and no coalescing. Compute the spike and name a 10× fix.** **Answer:** `peak = N / (sharing_factor × jitter_spread) = 500 / (1 × 1) = 500` concurrent recomputes at the expiry instant — and for a KV prefix each is a full prefill occupying GPUs shared with in-flight decode, so TTFT degrades fleet-wide, not just for the missing requests. A 10× fix with no new infrastructure: spread expiry over a jitter window ten times the recompute duration (`jitter_spread = 10` → peak ≈ 50), or equivalently stage the rollout that caused the invalidation to 10% of replicas at a time. Adding fleet-wide single-flight via a shared KV store takes it to ≈1, at the cost of a KV transfer path — which is worth it only if fetching the blocks is faster than recomputing them (640 MiB at 10 GB/s ≈ 64 ms versus 800 ms of prefill: yes).

- **Why is a KV-cache miss a different kind of miss?** **Answer:** Three ways. The unit is a token-block in a prefix chain, not an opaque key, so what is shared is a *prefix* and the eviction ordering must respect that — vLLM does it by returning a finished request's blocks to the free queue in reverse order, so the deepest (least shareable) blocks are evicted first while the shared system prompt sits furthest from the eviction head. The miss cost is GPU-seconds of prefill rather than a network round trip, and that prefill runs on the same devices serving everyone else's decode, so the cost is externalised onto unrelated requests as TTFT. And the cache competes for the same HBM as weights and activations, so eviction policy is a capacity decision: at ~320 KiB of KV per token for a 70B-class GQA model, a 2,048-token shared prefix is 640 MiB shared versus 640 MiB *per request* unshared.

- **1,000 nodes cold-start needing 200 GB each behind a 5 GB/s egress cap. Give the two numbers and the fix.** **Answer:** Naive: 200 TB ÷ 5 GB/s = 40,000 s ≈ 11.1 hours, and in practice worse because a thousand clients contending on one cap all time out and retry, consuming the cap with retries. Tree/torrent fan-out: ⌈log₂1000⌉ = 10 rounds × (200 GB ÷ 10 GB/s) ≈ 200 s ≈ 3.3 minutes, with origin egress of 200 GB once. Add a version-addressed local NVMe cache so a restart that does not change the model version is a local read (~100 s, zero network), and jitter the warm-up so the seeders do not stampede the origin. The key insight is that no eviction policy helps: the constraint is the pipe and the topology, not what you keep.

## Connections & what's next

This lesson finishes the replication spectrum from lesson 03 by walking off the end of it: a copy that has given up on correctness entirely and buys pure latency. The staleness window that lesson 01 asked you to bound and lesson 03 asked you to minimise is here the *normal operating condition*, and the engineering moves to bounding the damage rather than the staleness.

Forward, **lesson 05** takes the mode transition seriously as a general phenomenon. The stampede you just costed is one instance of a class: offered load exceeding capacity, with a feedback loop (retries, an empty cache, a full queue) that keeps it there after the trigger is gone. The `N / (sharing × jitter)` formula is the same shape as the admission-control arithmetic there, and "the origin cannot refill the cache because it is busy serving misses" is the canonical metastable failure. **Lesson 06** picks up the other half of what you saw here: a cache that is *up* but quietly wrong — a stale entry that survived a correct invalidation — is a gray failure, and detecting it is much harder than detecting a cache that is down.

Carry forward: *when offered load exceeds capacity, what does a queue actually do for you — and what makes an overload persist after its cause is gone?*

## References & further reading

**Primary sources — verified against upstream Git repositories this pass**

1. **`vllm-project/vllm` — `docs/design/prefix_caching.md`** — <https://github.com/vllm-project/vllm>. **Source of** the block hash-chain construction (`parent_hash`, `block_tokens`, `extra_keys` including LoRA id, multimodal hashes and `cache_salt`), the "only full blocks are cached" rule, the `KVCacheBlock` structure with `ref_cnt` and doubly-linked free-queue pointers, the preallocated block pool, the touch-on-hit behaviour, the LRU eviction from the free-queue head, the reverse-order free with its stated rationale ("the last block of a request must hash more tokens and is less likely to be reused… it should be evicted first"), the `sha256` default since v0.11 and the explicit security warning about non-cryptographic hash algorithms in multi-tenant deployments.
2. **`vllm-project/vllm` — `vllm/config/cache.py`** — same repository. **Source of** `DEFAULT_BLOCK_SIZE = 16`, `enable_prefix_caching: bool = True`, `gpu_memory_utilization: float = 0.92`, `prefix_caching_hash_algo` options, and the `prefix_match_unit` granularity control.
3. **Model KV-cache arithmetic** — `bytes_per_token = 2 × layers × kv_heads × head_dim × dtype_bytes`, worked here for an 80-layer, 8-KV-head, 128-head-dim, fp16 model (≈320 KiB/token). **Computed, not cited.** Re-run it against your model's `config.json` (`num_hidden_layers`, `num_key_value_heads`, `hidden_size / num_attention_heads`) before using it in a capacity plan; the GQA distinction between query heads and KV heads is where this calculation is usually got wrong by a factor of 8.

**Primary sources — not fetchable in this environment, not relied on for any number above**

4. **Nishtala, R. et al. (2013), *Scaling Memcache at Facebook*, NSDI** — <https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf>. The lease mechanism (token issued on miss, invalidated by a concurrent write, rejecting the stale fill) that closes the stale-set race drawn in Core concepts, plus the regional/tiered architecture and gutter pools. **Blocked by this environment's egress proxy**; the lease mechanism is described from standard knowledge.
5. **Vattani, A., Chierichetti, F. & Lowenstein, K. (2015), *Optimal Probabilistic Cache Stampede Prevention*, VLDB 8(8)** — the XFetch rule `now − delta × beta × ln(rand()) ≥ expiry`. **Not fetched**; the rule is reproduced from standard usage and should be checked against the paper (or against an implementation such as the ones in common Redis client libraries) before you rely on the constants.
6. **Qin, R. et al. (2024), *Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving*** — <https://arxiv.org/abs/2407.00079>. Disaggregating the KV cache across GPU HBM, CPU DRAM and SSD with a dedicated scheduler — the reference design for the "fleet-wide single flight on a KV prefix" option in the Worked example. **Not fetched** (arxiv.org blocked).
7. **Zheng, L. et al. (2023), *SGLang / RadixAttention*** — <https://arxiv.org/abs/2312.07104>. Prefix sharing via a radix tree with an LRU eviction that is explicitly structure-aware — the design vLLM's hash-chain-plus-ordered-free-queue reaches by different means. **Not fetched.**

**Real-world engineering — not fetchable this pass**

8. **Meta, *Cache made consistent* (TAO)** — <https://engineering.fb.com/2022/06/08/core-infra/cache-made-consistent/>. Coherence as a multi-year investment, including building the measurement tooling first. **Blocked.**
9. **AWS Builders' Library, *Caching challenges and strategies*** — <https://aws.amazon.com/builders-library/caching-challenges-and-strategies/>. The operational catalogue: modal behaviour, stampedes, the cache-as-availability-reducer argument, and the "always test cold" discipline. **Blocked.**
10. **Modal, *GPU Memory Snapshots: Supercharging sub-second startup*** — <https://modal.com/blog/gpu-mem-snapshots>; and **CoreWeave, *Decrease PyTorch Model Load Times with Tensorizer*** — <https://www.coreweave.com/blog/coreweaves-tensorizer-decrease-pytorch-model-load-times>. Two production attacks on cold start. **Blocked**; the reported multiples (~10× and >5×) are recalled from this lesson's previous version and are flagged as unverified.
11. **Brooker, M., *Caches, modes, and metastable failures*** — <https://brooker.co.za/blog/2021/08/27/caches.html>. The conceptual framing this lesson's modal-system section is built on. **Blocked**; lesson 05 continues the argument with the metastability literature.

**Deeper dives**

12. **Kleppmann, M., *Designing Data-Intensive Applications*** — chapter 1's discussion of caches as derived data, and chapter 11's treatment of derived state and change data capture, which is the principled alternative to invalidation: rather than telling caches what to forget, build them from a log they consume.
13. **Nginx caching directives** — `proxy_cache_lock` (single-flight at the reverse proxy), `proxy_cache_use_stale updating` (serve stale while one request updates), and `proxy_cache_background_update` — the stampede defences in this lesson implemented as three lines of configuration, worth knowing exist before you write your own.
14. **RFC 5861, *HTTP Cache-Control Extensions for Stale Content*** — `stale-while-revalidate` and `stale-if-error` as protocol-level standards, and the correct vocabulary to use when arguing for serving stale data.
15. **Einziger, Friedman & Manes, *TinyLFU: A Highly Efficient Cache Admission Policy*** — the admission-policy idea (decide what is worth *admitting*, not only what to evict) behind Caffeine and several CDN caches; the right next step if your workload's problem is cache pollution rather than stampede.
