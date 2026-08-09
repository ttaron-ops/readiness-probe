---
lesson: "A01.4"
title: "Caching"
module: "A-01"
concept: "modal systems, stampede, coherence, KV-cache"
status: not-started
est_time: "3 hrs"
artifacts: ["KV-cache stampede sizing + p2p weight-warmup design"]
---

# A01.4 · Caching

> **Concept.** A cache is a modal system: its behavior — and the load it imposes on the origin — changes discontinuously between hit and miss, so you design for the miss and the mode transition, not the hit.
>
> Module: [🧩 Distributed systems & system design](../README.md) · Track A — Platform excellence

## Why this matters

The cache's hit-mode numbers are the easy ones and the ones that lie. The system that kills you is the *transition*: a flush, a synchronized TTL expiry, or a fleet fault-restart flips you into miss mode and the origin — sized for hit-mode load — takes traffic it was never provisioned for. On a GPU fleet this is two literal, first-order problems: KV-cache under memory pressure (drives TTFT), and multi-TB weight distribution on cold start. Both are distributed-cache problems with real money attached, and both fail as stampedes.

## Core notes

**Skip (you already know):** TTL vs write-through/around/back; "caches cut latency"; LRU exists.

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

**Coherence & invalidation (one of the two hard problems).** Write-invalidate (drop the entry, next read repopulates — simple, adds a miss) vs write-update (push new value — more traffic, avoids the miss). Either alone has a **staleness window**: the interval where a replica serves the old value. Production answer is usually **TTL + explicit invalidate** — TTL bounds worst-case staleness even if an invalidate is lost; explicit invalidate cuts the *common-case* window to near zero. **Tiered caches** (L1 in-process, L2 shared Redis/memcached): faster L1 but now you have *inclusion/consistency* across tiers — an L2 invalidate doesn't reach L1s, so L1 needs its own short TTL or a pub/sub invalidation bus.

**GPU tie — two strong, real ties (not analogies):**
1. **KV-cache in inference is literally a distributed-cache problem.** Prefix/prompt caching and radix-tree sharing (SGLang RadixAttention) reuse prior attention state; **Mooncake** disaggregates the KV store, spilling GPU HBM → CPU DRAM → SSD with a scheduler that replicates and evicts KV blocks. Hit ratio drives **TTFT** (a hit skips prefill). Eviction under memory pressure *is* a cache-coherence/replacement decision — evict the wrong block and you pay a full prefill recompute.
2. **Model-weight caching across the fleet.** Multi-GB/TB weights pulled to nodes; **cold-start stampede** when hundreds of nodes fault-restart and all pull the same checkpoint from object store, hitting the egress cap. Thundering-herd fix is **p2p/torrent distribution** (nodes seed each other) + local NVMe cache with **jittered warmup**.

## Worked example

**Part A — KV-cache stampede on a shared system prompt.** An inference fleet caches KV for a shared system prompt across **200 replicas**; a single TTL expiry makes them all recompute prefill at once. Say the shared prefix is 2K tokens and prefill costs ~X ms of GPU per request. At synchronized expiry, 200 replicas each re-run prefill in the same window → a **200×** spike in prefill GPU demand precisely when serving traffic also needs those GPUs, spiking TTFT for *all* in-flight requests. *Fix:* single-flight per prefix-key (one replica computes, shares the KV block via the disaggregated store) collapses 200 recomputes → **1**; add jittered TTL so even without sharing the recomputes smear across a window (e.g. ±30s) instead of a spike. Peak prefill load drops from 200× to ~1× (single-flight) or to `200 / jitter_window_ticks` (jitter alone).

**Part B — cold weight pull.** **1,000 nodes** cold-restart and each pull a **200 GB** weight set from S3, but aggregate egress is capped at **5 GB/s**. Naive: total = 1000 × 200 GB = 200 TB ÷ 5 GB/s = **40,000 s ≈ 11 hours** of serialized warmup — the fleet is down for half a day, and every node retries against the same cap. *Fix — p2p:* seed a handful of nodes from S3, then nodes replicate to each other (torrent-style); with fan-out the checkpoint propagates in ~log(N) rounds instead of N serial pulls, and S3 egress is touched ~once. Add local NVMe cache so a *reboot* (not a version change) is a local read, and jitter the warmup start so the initial seeders don't themselves stampede S3. Warmup drops from ~11 h to minutes, bounded by intra-fleet bandwidth, not the S3 cap.

## Practice

*Feeds [staff design portfolio](../practice/staff-design-portfolio/README.md).*

Design the KV-cache + weight-distribution layer for the serving plane: (a) name your hit-ratio SLI and the alarm threshold, and describe the cold-flush load test that proves the prefill origin survives 0% hit ratio; (b) pick a stampede defense for the shared-prefix KV case and show the peak-load math before/after; (c) specify the KV eviction policy under HBM pressure and state, in tokens/latency, what a wrong eviction costs (the recompute); (d) design the weight cold-start: p2p topology, NVMe cache key (version-addressed), jittered warmup, and the egress number you're protecting. Explicitly separate the *coherence* concern (KV correctness across the disaggregated tiers) from the *load* concern (stampede) — a staff answer treats them as two problems.

## Self-check

- What does "a cache is a modal system" mean operationally, and what's the one test that follows? **Answer:** The cache has distinct hit and miss modes that impose *different* load on the origin; the mode can flip discontinuously (flush, mass expiry, fleet restart) into miss mode where the origin sees full traffic it wasn't sized for and may not recover (metastable). The test that follows: load-test *cold* (0% hit ratio) and confirm the origin survives — steady-state benchmarks are the misleading ones.
- Single-flight and probabilistic early expiration both fight stampede — when do you prefer each? **Answer:** Single-flight collapses simultaneous misses on an *already-expired* key to one origin fetch — it's the fix when the herd has already formed. Probabilistic early expiration (XFetch) prevents the herd forming at all by refreshing *before* expiry with rising probability, so there's never a synchronized cliff. Use early expiration for known-hot keys with predictable recompute cost; single-flight as the general backstop for any key.
- Why TTL *and* explicit invalidate rather than either alone? **Answer:** Explicit invalidate cuts the common-case staleness window to near zero but is unreliable — a lost/missed invalidate leaves a stale entry forever. TTL bounds the *worst-case* staleness even when an invalidate is dropped, but alone forces a choice between long windows or high miss rates. Together: invalidate handles the common case fast, TTL caps the failure case — belt and suspenders for the "hard problem."

## References

- https://aws.amazon.com/builders-library/caching-challenges-and-strategies/
- https://brooker.co.za/blog/2021/08/27/caches.html
- https://arxiv.org/abs/2407.00079 (Mooncake: disaggregated KV cache)
- https://arxiv.org/abs/2312.07104 (SGLang / RadixAttention)
