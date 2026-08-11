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
sources: 8
---

# A01.4 · Caching

> **Concept.** A cache is a modal system: its behavior — and the load it imposes on the origin — changes discontinuously between hit and miss, so you design for the miss and the mode transition, not the hit.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 03 laid out replication as a spectrum — sync, async, quorum, chain — where every scheme is built around the same assumption: a replica is supposed to eventually hold *correct* data, and the whole design effort goes into bounding how wrong or stale it's allowed to get in the meantime. A cache is a different kind of replica. It's a copy of data optimized purely for *latency*, and it's explicitly allowed to be wrong, stale, or simply absent — the thing replication schemes work so hard to avoid is a cache's normal operating condition. That relaxed contract is exactly what makes caching cheap and fast, but it buys a failure mode replication doesn't have: instead of degrading gracefully like a replica losing one node, a cache flips discontinuously between two entirely different operating regimes — hit and miss — and the origin behind it has to survive both. This lesson is about designing for that transition, not the steady state.

## Why this matters

The cache's hit-mode numbers are the easy ones and the ones that lie. The system that kills you is the *transition*: a flush, a synchronized TTL expiry, or a fleet fault-restart flips you into miss mode and the origin — sized for hit-mode load — takes traffic it was never provisioned for. On a GPU fleet this is two literal, first-order problems: KV-cache under memory pressure (drives TTFT), and multi-TB weight distribution on cold start. Both are distributed-cache problems with real money attached, and both fail as stampedes.

## What's new here (calibration)

- **Skip (carried over):** TTL vs write-through/around/back; "caches cut latency"; LRU exists.
- **New here — the modal-system test.** The framing that a cache has genuinely different hit-mode and miss-mode operating regimes, and the operational test that follows from it: load-test *cold* (0% hit ratio), because steady-state benchmarks hide the exact failure mode that kills you.
- **New here — stampede as math, not folklore.** A peak-load formula for cache-stampede recovery, not just "use single-flight" as a slogan.
- **New here — two real (not analogical) GPU-serving-plane instances.** KV-cache and weight distribution are the same caching problem restated with an HBM-bound eviction unit and a bandwidth/topology cold-start problem, respectively — tying directly to the module's serving-plane framing.

## Core concepts

**Caches are modal systems (Brooker/AWS).** Two operating modes with different origin load. The dangerous mode is **cold-cache-after-flush**: hit ratio → 0, origin sees full request rate it wasn't sized for, latency climbs, retries pile on, and you can be *unable to recover* because the origin can't serve the miss storm long enough to refill the cache (a metastable failure). Consequences:
- **Design for the miss, not the hit.** The question is never "how fast at 95% hit ratio" but "does the origin survive at 0% hit ratio, and for how long."
- **Hit ratio is a first-class SLI**, alarmed and dashboarded. A cache that improves the mean while hiding an origin that can't survive a flush has *reduced* availability by adding a mode.
- **Load-test cold.** The steady-state benchmark is the useless one; test the flush.

**Thundering herd / cache stampede.** A hot key expires; every concurrent request misses simultaneously and hits the origin at once. Fixes, in rough order:
- **Request coalescing / single-flight**: one origin fetch per key; concurrent callers wait on that in-flight fetch. Collapses N simultaneous misses to 1.
- **Probabilistic early expiration (XFetch)**: recompute *before* expiry with probability rising as TTL approaches, so one lucky request refreshes while the rest still hit — no synchronized cliff.
- **Staggered/jittered TTLs**: never expire a cohort of keys on the same tick.
- **Stale-while-revalidate**: serve stale, refresh in background — the miss never reaches the user path.
- **Negative caching**: cache "not found" to stop miss-storms hammering the origin for keys that don't exist.

**Formalizing the stampede peak.** The peak concurrent recompute load a stampede produces is approximately:

```
peak_replicas_recomputing ≈ N / max(1, single_flight_sharing_factor) / jitter_window_ticks
```

where `N` is the number of replicas/callers whose cache entries expire together, `single_flight_sharing_factor` is how many callers per unique key collapse onto one in-flight fetch (1 = no coalescing), and `jitter_window_ticks` is how many distinct expiry ticks the TTLs are spread across (1 = fully synchronized, no jitter). With no defenses (`sharing_factor=1`, `jitter_window_ticks=1`), peak = N — the full herd hits the origin at once. Single-flight alone divides by the sharing factor; jitter alone divides by the window; the two compose, which is why production defenses stack them rather than picking one.

**Coherence & invalidation (one of the two hard problems).** Write-invalidate (drop the entry, next read repopulates — simple, adds a miss) vs write-update (push new value — more traffic, avoids the miss). Either alone has a **staleness window**: the interval where a replica serves the old value. Production answer is usually **TTL + explicit invalidate** — TTL bounds worst-case staleness even if an invalidate is lost; explicit invalidate cuts the *common-case* window to near zero. **Tiered caches** (L1 in-process, L2 shared Redis/memcached): faster L1 but now you have *inclusion/consistency* across tiers — an L2 invalidate doesn't reach L1s, so L1 needs its own short TTL or a pub/sub invalidation bus.

**GPU tie — two strong, real ties (not analogies):**
1. **KV-cache in inference is literally a distributed-cache problem.** Prefix/prompt caching and radix-tree sharing (SGLang RadixAttention) reuse prior attention state; **Mooncake** disaggregates the KV store, spilling GPU HBM → CPU DRAM → SSD with a scheduler that replicates and evicts KV blocks. Hit ratio drives **TTFT** (a hit skips prefill). Eviction under memory pressure *is* a cache-coherence/replacement decision — evict the wrong block and you pay a full prefill recompute.
2. **Model-weight caching across the fleet.** Multi-GB/TB weights pulled to nodes; **cold-start stampede** when hundreds of nodes fault-restart and all pull the same checkpoint from object store, hitting the egress cap. Thundering-herd fix is **p2p/torrent distribution** (nodes seed each other) + local NVMe cache with **jittered warmup**.

**KV-cache eviction — the unit and the cost of a miss are different from a generic cache.** A generic cache miss costs a network round trip to the origin; a KV-cache miss costs GPU-seconds of recompute. That changes which eviction policy is worth its complexity:

| Policy | Eviction unit | What it captures | Cost of a wrong eviction |
|---|---|---|---|
| LRU | Whole cache entry (e.g. per-request KV block) | Recency only | A full prefill recompute (GPU-seconds) for that request's context, even if a near-identical prefix was evicted a moment earlier |
| Radix-tree / prefix-aware (RadixAttention-style) | Shared token-prefix subtree | Recency *and* sharing — a prefix used by many requests is protected even if any single request using it is "old" | Lower — shared prefixes are evicted last, so the common case (many requests sharing a system prompt) rarely repays a full prefill |
| Priority-by-reuse-distance | Per-block, ranked by predicted next-use | Recency, sharing, and a forward-looking reuse estimate | Lowest in principle, at the cost of tracking/predicting reuse distance — worth it only when the workload's reuse pattern is stable enough to predict |

The lesson generalizes: **the eviction unit should match the thing that's actually shared** (a token prefix, not an opaque blob), because in this domain the cost of getting it wrong is measured in GPU-seconds, not milliseconds.

## Perspectives

**The systems/modal-behavior view (Brooker).** This is the anchor framing of the lesson: hit-mode and miss-mode are, functionally, two different systems wearing the same interface. A cache that's "99% hit rate, p50 latency 2ms" in one mode can be an origin-melting outage in the other, and nothing about the hit-mode metrics predicts that — the only way to know is to deliberately measure the miss-mode system, which most teams never do until it happens in production.

**The coherence/correctness view.** Every invalidation strategy has a race: write-invalidate can drop the entry after a stale read has already started; write-update can push a new value while an in-flight write to the origin is still pending, so which one "wins" is a timing question. Tiered caches multiply this — an L2 invalidation doesn't automatically reach every L1, so an L1 can be authoritative-looking and wrong. None of this is solved once; it's a standing engineering investment (see Meta's TAO story below).

**The inference-serving/hardware view.** KV-cache lives in GPU HBM, a resource that is both scarce and shared with the model weights and activation memory needed to serve the *next* request — eviction isn't a policy tuning knob in isolation, it's competing directly with throughput for the same physical memory. The eviction unit is a token (or a shared prefix of tokens), not a generic key-value pair, and a miss's cost is denominated in GPU-seconds of recompute, not network round-trip milliseconds — a completely different cost model from a Redis cache in front of a database.

**The fleet-operations/cold-start view.** Getting hundreds of GB of weights onto hundreds of nodes fast is not a caching-*policy* problem — no eviction algorithm helps you here — it's a bandwidth-and-topology problem. The fix lives in how data moves (p2p/torrent fan-out instead of N independent pulls from one object store), not in what you keep or evict. Treating it as "just add a cache" misses that the bottleneck is the pipe, not the policy.

## Real-world use cases

*(These URLs are the canonical, live links to the named posts. This session's web-fetch tool was blocked by the environment's network egress proxy, so they were not independently re-fetched this pass — flagged per the sourcing rules rather than silently cited as verified.)*

- **Modal — "GPU Memory Snapshots: Supercharging sub-second startup"** (https://modal.com/blog/gpu-mem-snapshots) — CUDA-level checkpoint/restore of full GPU state (weights in VRAM, kernel objects, execution context); one workload's cold start dropped from ~118s to ~12s (~10x), another from 20s to 2s. The sharpest, most current, most GPU-specific case for the weight-caching/cold-start section, from a serverless GPU inference provider — the exact category of employer this module targets. Lead with this one.
- **CoreWeave — "Decrease PyTorch Model Load Times with CoreWeave's Tensorizer"** (https://www.coreweave.com/blog/coreweaves-tensorizer-decrease-pytorch-model-load-times) — fast S3/HTTP-streamed model deserialization, reported >5x faster loads than naive Hugging Face loading when scaling from zero — a named GPU-cloud operator's real fix for exactly the cold-weight-pull problem in this lesson's Worked Example Part B.
- **Meta — "Cache made consistent" (TAO)** (https://engineering.fb.com/2022/06/08/core-infra/cache-made-consistent/) — Meta's account of taking TAO's cache-consistency guarantee from "six nines" to "ten nines" over years of dedicated invalidation engineering — the best available real story that coherence is one of the two hard problems, and that it's a long deliberate investment, not a one-time fix.

## Worked example

**Part A — KV-cache stampede on a shared system prompt.** An inference fleet caches KV for a shared system prompt across **200 replicas**; a single TTL expiry makes them all recompute prefill at once. Say the shared prefix is 2K tokens and prefill costs ~X ms of GPU per request. At synchronized expiry, 200 replicas each re-run prefill in the same window → a **200×** spike in prefill GPU demand precisely when serving traffic also needs those GPUs, spiking TTFT for *all* in-flight requests. Using the peak formula: with no defenses, `sharing_factor=1, jitter_window_ticks=1` → peak = 200. *Fix:* single-flight per prefix-key (one replica computes, shares the KV block via the disaggregated store) sets `sharing_factor=200` → peak ≈ 1; add jittered TTL (e.g. ±30s window) so even without sharing, `jitter_window_ticks` grows and the recomputes smear across the window instead of a spike (e.g. a 30-tick window drops peak to ~7). Peak prefill load drops from 200× to ~1× (single-flight) or to `200 / jitter_window_ticks` (jitter alone) — and the two stack for defense in depth.

**Part B — cold weight pull.** **1,000 nodes** cold-restart and each pull a **200 GB** weight set from S3, but aggregate egress is capped at **5 GB/s**. Naive: total = 1000 × 200 GB = 200 TB ÷ 5 GB/s = **40,000 s ≈ 11 hours** of serialized warmup — the fleet is down for half a day, and every node retries against the same cap. *Fix — p2p:* seed a handful of nodes from S3, then nodes replicate to each other (torrent-style); with fan-out the checkpoint propagates in ~log(N) rounds instead of N serial pulls, and S3 egress is touched ~once. Add local NVMe cache so a *reboot* (not a version change) is a local read, and jitter the warmup start so the initial seeders don't themselves stampede S3. Warmup drops from ~11 h to minutes, bounded by intra-fleet bandwidth, not the S3 cap — directionally consistent with Modal's and CoreWeave's real, publicly reported 5–10x cold-start improvements from attacking this exact bottleneck (2024/2025 figures — treat as dated snapshots).

## Practice

*Feeds [staff design portfolio](../practice/staff-design-portfolio/README.md).*

Design the KV-cache + weight-distribution layer for the serving plane: (a) name your hit-ratio SLI and the alarm threshold, and describe the cold-flush load test that proves the prefill origin survives 0% hit ratio; (b) pick a stampede defense for the shared-prefix KV case and show the peak-load math before/after, using the `N / sharing_factor / jitter_window_ticks` formula; (c) specify the KV eviction policy under HBM pressure and state, in tokens/latency, what a wrong eviction costs (the recompute); (d) design the weight cold-start: p2p topology, NVMe cache key (version-addressed), jittered warmup, and the egress number you're protecting. Explicitly separate the *coherence* concern (KV correctness across the disaggregated tiers) from the *load* concern (stampede) — a staff answer treats them as two problems.

## Common pitfalls

1. **"Benchmark at steady-state hit ratio to know if the cache is sized right."** The number that matters is whether the origin survives a cold flush (0% hit ratio) — steady-state numbers hide exactly the failure mode that kills you.
2. **"TTL alone is enough for correctness."** TTL only bounds worst-case staleness; pair it with explicit invalidation. Meta's multi-year TAO investment (six nines → ten nines) is real evidence this is genuinely hard, not a checkbox.
3. **"KV-cache eviction is just LRU."** A wrong eviction in inference costs a full prefill recompute (GPU-seconds), not just a slower read — that changed cost model is why a radix-tree/prefix-aware policy usually beats naive LRU here.
4. **"Single-flight alone solves stampede."** It collapses concurrent misses on an already-expired key, but does nothing to prevent synchronized expiry in the first place; pair it with jitter/probabilistic early expiration so the herd never forms.
5. **"P2P/torrent weight distribution is exotic and unnecessary below huge scale."** Even hundreds of nodes pulling from a shared object-store egress cap can serialize into hours (this lesson's own 11-hour naive-warmup math). Modal's and CoreWeave's real, published numbers (~10x, >5x) show mainstream GPU-cloud providers already treat this as first-order infrastructure, not a large-scale-only optimization.

## Self-check

- What does "a cache is a modal system" mean operationally, and what's the one test that follows? **Answer:** The cache has distinct hit and miss modes that impose *different* load on the origin; the mode can flip discontinuously (flush, mass expiry, fleet restart) into miss mode where the origin sees full traffic it wasn't sized for and may not recover (metastable). The test that follows: load-test *cold* (0% hit ratio) and confirm the origin survives — steady-state benchmarks are the misleading ones.
- Single-flight and probabilistic early expiration both fight stampede — when do you prefer each? **Answer:** Single-flight collapses simultaneous misses on an *already-expired* key to one origin fetch — it's the fix when the herd has already formed. Probabilistic early expiration (XFetch) prevents the herd forming at all by refreshing *before* expiry with rising probability, so there's never a synchronized cliff. Use early expiration for known-hot keys with predictable recompute cost; single-flight as the general backstop for any key.
- Why TTL *and* explicit invalidate rather than either alone? **Answer:** Explicit invalidate cuts the common-case staleness window to near zero but is unreliable — a lost/missed invalidate leaves a stale entry forever. TTL bounds the *worst-case* staleness even when an invalidate is dropped, but alone forces a choice between long windows or high miss rates. Together: invalidate handles the common case fast, TTL caps the failure case — belt and suspenders for the "hard problem."
- 500 replicas share a KV-cache prefix that expires with no jitter and no single-flight. Using the peak formula, what's the recompute spike, and name one change that cuts it 10x? **Answer:** With `sharing_factor=1` and `jitter_window_ticks=1`, peak = 500/1/1 = 500 — a full 500x spike in prefill demand at the expiry instant. Spreading the TTL over a 10-tick jitter window (`jitter_window_ticks=10`) alone cuts peak to ~50, a 10x reduction, without requiring any KV-sharing infrastructure; adding single-flight on top would cut it further.

## Connections & what's next

This lesson treats a cache as lesson 03's replication spectrum specialized for one objective — latency instead of durability — reusing the same replica instincts (staleness windows, invalidation races) but relaxing the correctness contract that replication is built to guarantee. Forward, a cache stampede is really a queueing/backpressure problem wearing a cache-specific name: the peak-load formula derived here is the same shape as the admission-control math lesson 05 formalizes with Little's Law, and that's the thread lesson 05 picks up directly. Cache coherence failures (a stale KV block silently served, a lost invalidation) are also a resilience concern in the gray-failure sense lesson 06 covers — a cache that's "up" but quietly wrong is a harder failure to detect than one that's simply down.

## References & further reading

**Primary sources**
- Nishtala, R. et al. (2013), *Scaling Memcache at Facebook*, NSDI '13 — https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf (read for: leases as a stampede control at hyperscale, and the tiered-cache-cluster architecture behind Facebook's memcache deployment).
- Qin, R. et al. (2024), *Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving* — https://arxiv.org/abs/2407.00079 (read for: disaggregating the KV cache across GPU HBM / CPU DRAM / SSD with a dedicated scheduler).
- Zheng, L. et al. (2023), *Efficiently Programming Large Language Models using SGLang* (RadixAttention) — https://arxiv.org/abs/2312.07104 (read for: the radix-tree prefix-sharing eviction structure this lesson's eviction table references).

**Real-world engineering blogs**
- Modal — *GPU Memory Snapshots: Supercharging sub-second startup* — https://modal.com/blog/gpu-mem-snapshots — what it shows: full-GPU-state checkpoint/restore cutting cold start ~10x at a serverless GPU inference provider.
- CoreWeave — *Decrease PyTorch Model Load Times with CoreWeave's Tensorizer* — https://www.coreweave.com/blog/coreweaves-tensorizer-decrease-pytorch-model-load-times — what it shows: streamed model deserialization as a production fix for the cold-weight-pull problem.
- Meta — *Cache made consistent* (TAO) — https://engineering.fb.com/2022/06/08/core-infra/cache-made-consistent/ — what it shows: cache-coherence engineering as a multi-year investment, not a one-time fix.
- AWS Builders' Library — *Caching challenges and strategies* — https://aws.amazon.com/builders-library/caching-challenges-and-strategies/ — what it shows: a production catalogue of cache failure modes and mitigations across AWS services.

**Deeper dives**
- Brooker, M., *Caches, modes, and metastable failures* — https://brooker.co.za/blog/2021/08/27/caches.html — the conceptual framing this lesson's "modal system" idea is built on.
