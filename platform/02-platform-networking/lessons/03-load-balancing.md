---
lesson: "A02.3"
title: "Load balancing"
module: "A-02"
concept: "LB as failure domain"
status: not-started
est_time: "3 hrs"
artifacts: ["thundering-herd-sim", "maglev-table-exercise"]
---

# A02.3 · Load balancing

> **Concept.** The load balancer is a failure domain and a stateful control point, not a dumb distributor — its health-check logic, hash table, and draining decide whether one slow backend becomes an outage or gets contained.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Why this matters
At staff scope you own the blast radius, not the box. Every LB decision — how it hashes, how aggressively it health-checks, how it ramps a cold backend, how clients retry — is a lever on whether a transient blip stays a blip or amplifies into a fleet-wide outage. The interview question is never "what is round-robin"; it's "your p99 spiked to timeouts across three services behind one LB tier — walk me through why." Everything below is the mechanism underneath that walk-through, and it's exactly where inference-serving in front of GPU pods diverges hard from stateless web.

## Core notes
**Skip (you already know):** L4 vs L7 in principle, round-robin/least-conn, what a health check is, sticky sessions, anycast basics, and the high-level TLS-termination-location tradeoff.

**The LB is a failure domain.** Three pieces of LB state decide blast radius: the *health-check verdict* (which backends are in rotation), the *connection table* (which flows are pinned where), and the *drain state* (what happens to in-flight work on removal). Get any wrong and the LB turns one slow backend into an outage — or contains it. Reason about an LB the way you reason about a database: it has state, that state has failure modes, and the failure modes are correlated across everything behind it.

**Consistent hashing done right — Maglev.** Naive `hash(key) % N` reshuffles *every* flow when N changes (one backend added → ~all flows remap → mass connection resets + cold caches everywhere). Ring/ketama consistent hashing bounds churn to ~1/N of flows on a membership change, but pays with load imbalance (uneven arc lengths → hot backends) unless you add many virtual nodes. **Maglev** (Google's L4 LB) builds a fixed-size *lookup table* via a permutation-fill algorithm that is simultaneously *minimally disruptive* (a single backend change perturbs only its share of table entries) **and** near-perfectly balanced (entries spread within ~1% across backends). It layers *connection tracking* on top so existing flows survive backend-set changes even when the table entry moves — the table decides new flows, the conn-track table protects established ones. That combination — minimal disruption + even balance + flow survival — is the whole point, and it's the standard staff answer to "how do stateless LBs keep flows stable under autoscaling."

**DSR + ECMP — scaling past one box's bandwidth.** A single LB is a bandwidth ceiling. The escape: the router does **ECMP** (equal-cost multipath) hashing to spray packets across a *tier* of stateless LB instances — any instance can handle any packet because Maglev tables are deterministic from the same backend set. Each LB then does **DSR** (direct server return): it encapsulates/rewrites the packet to a chosen backend, but the backend replies *directly to the client*, source-addressed as the VIP, bypassing the LB on the return path. Since responses are the bulk of the bytes (and for inference, the *generated tokens*), the LB tier only sees the thin request path — you scale throughput horizontally by adding LB instances behind ECMP without any single box carrying return traffic.

**Health checking is subtle.** *Active* checks probe endpoints; *passive* checks (outlier detection) eject backends that return errors on real traffic. The cascading-failure risk: aggressive checks against a *shared dependency* — a backend DB blips, all backends fail their check together, the LB ejects a majority, the survivors get crushed, they fail too. The guard is the **panic threshold** (Envoy default 50%): if fewer than X% of hosts are healthy, the LB *ignores health status entirely* and load-balances across all hosts — betting that a half-degraded backend beats sending 100% of traffic to a collapsing minority. Knowing panic mode exists (and why) is a reliable staff-vs-senior tell.

**Thundering herd, slow-start, and drain ordering.** On *scale-up*, a fresh backend has cold caches, cold JIT, empty connection pools — give it full share the instant it's `Ready` and it tips over. Envoy's `slow_start_window` linearly ramps its weight from near-zero over the window. On *scale-down*, resets are prevented by ordering: `preStop` hook → endpoint removal → **connection draining** (finish in-flight, refuse new) → `terminationGracePeriod` → SIGKILL; get the ordering wrong and you reset live connections. And the classic: after any blip, *synchronized retries* create a thundering herd — 1000 clients that all failed at T=0 all retry at T=1. **Full-jitter exponential backoff** (`sleep = random(0, min(cap, base·2^attempt))`) is non-negotiable precisely because it de-correlates that spike; plain exponential backoff still lets everyone retry in lockstep.

## Worked example
**Thundering-herd amplification.** 1000 clients, one backend blips for 2s. Model synchronized retry with fixed exponential backoff (1s, 2s, 4s...) vs full jitter.

- *No jitter:* all 1000 fail at T=0, all retry at T=1 (spike = 1000 req in one instant), the retries themselves overload the recovering backend → they fail again → retry at T=3 (another 1000-spike). Retry amplification: offered load = `clients × (1 + retries)` concentrated in instants. The backend never gets a quiet window to recover — a self-sustaining retry storm.
- *Full jitter:* each client sleeps `random(0, 2^attempt)`. The 1000 retries smear uniformly across a 1–2s window → instantaneous rate drops from 1000/instant to ~1000/window ≈ a flat, absorbable trickle. This is the AWS Builders' Library simulation: full jitter both flattens the peak *and* reduces total work, because early successes stop retrying before the herd re-forms.

**Maglev vs modulo remap count.** 1000 flow-keys, backends 3→4.
- *Modulo:* `hash(k) % 3` vs `hash(k) % 4`. A key stays put only if `h%3 == h%4`; that overlap is small, so expect **~700–800 of 1000 flows remap** (mass reset + cold caches everywhere).
- *Maglev:* build the lookup table (size M, a prime like 65537 in prod; use e.g. 7 for a hand exercise) via permutation fill for 3 backends, then for 4. A key remaps only if its table slot changed owner. Adding the 4th backend claims ~M/4 of slots evenly from the existing three → expect **~250 of 1000 flows remap** (≈1/N), and *those* are protected by conn-tracking if already established. Count both, tabulate the delta — that delta is why every serious L4 LB uses a Maglev-style table.

## Practice
<feeds [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)>

Build two artifacts for the runbook: (1) a small simulation (any language) that runs the 1000-client thundering-herd model with and without full jitter and plots offered-load-per-instant over time — capture the peak-flattening number. (2) A `maglev-table` script that fills the lookup table for a backend set, then recomputes for N+1 and N−1, and reports the exact percentage of flow-keys that remap vs the modulo baseline. In the runbook, add a decision section: given an inference tier in front of GPU pods, specify the routing policy (least-outstanding-requests / queue-depth-aware), the slow-start window sized to model-load warmup, and the panic threshold — and justify each against a shared-dependency blip scenario. Reference the GPU-fabric artifact (module 09) for the intra-node fabric picture; this lesson stays at the request-tier LB.

## Self-check
- Why does least-*outstanding-requests* routing matter far more for GPU inference than for a stateless web tier, and what failure does round-robin/least-conn produce there? **Answer:** GPU inference requests have wildly variable cost — a 20-token completion vs a 4000-token generation can differ 100× in occupancy time. Round-robin ignores cost entirely and least-*connections* only counts open connections, not work-in-flight, so both let long generations pile onto some H100s while others sit idle — you get simultaneously queued *and* underutilized GPUs. Least-outstanding-requests (or explicit queue-depth-aware routing) tracks actual in-flight work per replica, so it steers new requests to the genuinely least-loaded GPU and keeps the expensive fleet evenly saturated.
- What does Envoy's panic threshold do, and what cascading failure is it designed to prevent? **Answer:** When the fraction of healthy hosts drops below the threshold (default 50%), Envoy stops honoring health status and load-balances across *all* hosts. It exists to prevent a shared-dependency cascade: if a common backend blips and most hosts fail their health check simultaneously, ejecting the majority would slam 100% of traffic onto a tiny surviving set that then also collapses. Panic mode bets that spreading load over a half-degraded fleet beats guaranteeing the healthy minority gets crushed.
- Why is a newly-`Ready` GPU inference replica not actually ready for full traffic, and how does slow-start map to that? **Answer:** `Ready` (passing a health check) does not mean the model weights are loaded, the CUDA context is warm, the KV-cache allocator is primed, or JIT/graph capture is done — a replica can pass a liveness probe seconds before it can serve at full rate. Give it full share instantly and it queues/tips over. `slow_start_window` ramps the replica's traffic weight up linearly over a window sized to the model-load warmup, so it absorbs the cold-start penalty at low load and reaches full share only once genuinely warm.

## References
- https://www.usenix.org/conference/nsdi16/technical-sessions/presentation/eisenbud
- https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/
- https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/slow_start
- https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier
- https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/panic_threshold
