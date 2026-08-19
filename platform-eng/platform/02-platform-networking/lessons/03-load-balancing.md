---
lesson: "A02.3"
title: "Load balancing"
module: "A-02"
concept: "LB as failure domain"
status: not-started
est_time: "3.5 hrs"
prev: "02-dns-and-service-discovery.md"
next: "04-cloud-networking.md"
artifacts: ["thundering-herd-sim", "maglev-table-exercise"]
sources: 11
---

# A02.3 · Load balancing

> **Concept.** The load balancer is a failure domain and a stateful control point, not a dumb distributor — its health-check logic, hash table, and draining decide whether one slow backend becomes an outage or gets contained.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Where this fits
Lesson 02 left you at DNS's cache topology — how a name resolves and how that resolution's TTL/staleness interacts with backend churn. This lesson picks up exactly where that hands off: once a client has an IP, the LB is the next stateful control point on the path, and its internal state (health verdicts, hash table, connection table, drain state) determines whether the backend churn DNS just exposed you to turns into a contained blip or a fleet-wide outage. It unlocks Lesson 04's cloud-networking cost model (cross-zone LB is one of the two dominant egress culprits) and Lesson 06's service mesh (Envoy's retry-budget and outlier-detection knobs are the same mechanisms, one layer up the stack).

## Why this matters
At staff scope you own the blast radius, not the box. Every LB decision — how it hashes, how aggressively it health-checks, how it ramps a cold backend, how clients retry — is a lever on whether a transient blip stays a blip or amplifies into a fleet-wide outage. The interview question is never "what is round-robin"; it's "your p99 spiked to timeouts across three services behind one LB tier — walk me through why." Everything below is the mechanism underneath that walk-through, and it's exactly where inference-serving in front of GPU pods diverges hard from stateless web.

## What's new here (calibration)
- **Skipped:** L4 vs L7 in principle, round-robin/least-conn, what a health check is, sticky sessions, anycast basics, and the high-level TLS-termination-location tradeoff.
- **New depth — the algorithm family:** Maglev isn't a one-off trick; it's one instance of "minimal disruption + bounded load imbalance," alongside rendezvous hashing (HRW) and ring/ketama hashing. You will build the Maglev table by hand here, from the same offset/skip permutation the production implementations use.
- **New depth — the actual defaults.** Every health-check, outlier-detection, panic-mode, and slow-start number in this lesson is read from Envoy's own API protos rather than from memory, because these are the numbers you argue about in a review.
- **New depth — the hardware layer underneath the LB:** ECMP at the router is itself a consistent-hash problem with its own reshuffle-on-topology-change failure mode, and it's *why* Maglev's stateless-instance design exists in the first place.
- **New depth — connection budget arithmetic:** ephemeral-port exhaustion at an SNAT-ing LB is a capacity calculation you can do in advance, and almost nobody does.
- **New depth — GPU-inference routing:** least-outstanding-requests is table stakes; queue-depth-, KV-cache-, and prefix-cache-aware routing (the Gateway API Inference Extension / llm-d direction) is the current frontier, with a documented model-server metrics contract you can read.

## Core concepts

### 1. An LB is three pieces of state, not an algorithm

The algorithm ("round robin," "least connections") is the least interesting thing about a load balancer, because it only decides *where a new request goes when everything is fine*. What decides your blast radius is the state the LB carries:

| State | What it holds | How it goes wrong |
|---|---|---|
| **Health verdict** | which backends are eligible right now | a shared-dependency blip fails everyone's check at once, and the LB ejects the majority |
| **Flow/connection table** | which existing flows are pinned to which backend | it is lost on LB restart or reshuffled on ECMP change → mass RSTs |
| **Drain state** | what happens to in-flight work when a backend leaves | wrong ordering resets live connections during a routine deploy |

Reason about an LB the way you reason about a database: it has state, that state has failure modes, and those failure modes are **correlated across everything behind it**. A bug in a single backend affects that backend's users. A bug in the LB's health logic affects everyone.

### 2. L4 versus L7 on the same request: what each can and cannot see

This distinction is usually taught as "layer 4 is faster, layer 7 is smarter," which is true and useless. The operational version is: **what information does the balancer have at the moment it must choose a backend, and what does it consequently break?**

```
   ONE HTTPS REQUEST, TWO BALANCERS.  Client 203.0.113.9 → api.example.com
   ══════════════════════════════════════════════════════════════════════════

   ┌─────────────────────────── L4 (NLB / IPVS / Katran / Maglev ) ───────────┐
   │                                                                          │
   │  What arrives:  TCP SYN                                                  │
   │   ┌───────────────────────────────────────────────────────┐              │
   │   │ IP  src 203.0.113.9  dst 198.51.100.7                 │  ◀ VISIBLE   │
   │   │ TCP src 51234        dst 443       [SYN]              │  ◀ VISIBLE   │
   │   ├───────────────────────────────────────────────────────┤              │
   │   │ (nothing — TLS has not started, there is no payload)  │              │
   │   └───────────────────────────────────────────────────────┘              │
   │                                                                          │
   │  CAN decide on:  5-tuple hash, source IP, dst port, packet rate,         │
   │                  and — with TLS passthrough + SNI peeking — the          │
   │                  ClientHello's SNI, but ONLY after the handshake starts  │
   │  CANNOT see:     URL path, method, headers, cookies, request body,       │
   │                  response status, request COST                           │
   │  Decision made:  ONCE, per connection. Every request multiplexed on      │
   │                  that connection goes to the same backend — which is     │
   │                  why HTTP/2 and gRPC break L4 balancing: ONE long-lived  │
   │                  connection carries thousands of requests to ONE pod.    │
   │  Costs:          ~microseconds; no TLS termination; DSR possible         │
   │  Failure it hides: a backend returning 500s at full speed looks HEALTHY  │
   │                  (the TCP connection succeeds) unless you add passive    │
   │                  detection, which L4 cannot do because it sees no status │
   └──────────────────────────────────────────────────────────────────────────┘

   ┌─────────────────────────── L7 (Envoy / ALB / NGINX) ─────────────────────┐
   │                                                                          │
   │  What arrives:  a terminated TLS session and a parsed HTTP request       │
   │   ┌───────────────────────────────────────────────────────┐              │
   │   │ :method POST      :path /v1/chat/completions          │  ◀ VISIBLE   │
   │   │ :authority api.example.com                            │  ◀ VISIBLE   │
   │   │ x-request-id, x-tenant-id, cookie, auth header        │  ◀ VISIBLE   │
   │   │ body: {"model":"llama-70b","messages":[... 8 KB ...]} │  ◀ VISIBLE   │
   │   └───────────────────────────────────────────────────────┘              │
   │                                                                          │
   │  CAN decide on:  path, header, method, tenant, body (with ext_proc),     │
   │                  per-request outstanding count, observed response codes  │
   │  Decision made:  PER REQUEST — so one HTTP/2 connection's 1000 streams   │
   │                  can be spread across 1000 backends                      │
   │  Costs:          TLS termination CPU, HTTP parsing, buffering, one more  │
   │                  TCP connection on the backend side, and the LB is now   │
   │                  in the TLS trust boundary (it holds the cert)           │
   │  New powers:     retries, timeouts, circuit breaking, outlier ejection   │
   │                  by status code, header-based canary, per-tenant limits  │
   │  New failure:    the retry it performs is invisible to the client, so    │
   │                  the LB itself can amplify a blip (see §9)               │
   └──────────────────────────────────────────────────────────────────────────┘

   THE SENTENCE TO REMEMBER: an L4 balancer balances CONNECTIONS; an L7
   balancer balances REQUESTS. Everything else follows from that — including
   why an L4 LB in front of gRPC produces a perfectly even connection count
   and a wildly uneven CPU distribution.
```

**The gRPC/HTTP-2 consequence is worth stating separately** because it is the most common real-world instance. A gRPC client opens one HTTP/2 connection to the VIP and multiplexes every RPC over it. An L4 LB pinned that connection to pod A at SYN time and cannot revisit the decision. If you scale from 3 to 30 pods, the existing clients keep talking to their original 3. The standard answers, in increasing order of intrusiveness: an L7 proxy that balances per stream; client-side balancing over a headless Service (lesson 02 §9's RRset, with its staleness cost); `MAX_CONNECTION_AGE` on the server so connections are periodically cycled and rebalance naturally; or a mesh sidecar (lesson 06).

### 3. The selection algorithms, and what each actually optimises

| Policy | Decision cost | State needed | Disruption when the backend set changes | Balance quality | Use it when |
|---|---|---|---|---|---|
| Round robin | O(1) | a counter | none (no affinity to lose) | perfect by *count*, arbitrary by *cost* | request costs are uniform |
| Weighted RR | O(1) | counters + weights | none | proportional by count | backends differ in capacity |
| **Least request (P2C)** | O(1) — **2 random probes**, not a full scan | in-flight counter per host | none | very good, and cost-aware | request costs vary — **the sane default** |
| Ring hash (ketama) | O(log R) | ring of R points (Envoy: **min 1024**, max 8 M, `XX_HASH` by default) | ≈1/N of keys move | uneven unless R ≫ N | you need affinity and can afford a big ring |
| **Maglev** | O(1) — one array index | table of size M (Envoy default **65537**, must be prime, ≤ 5000011) | ≈1/N of table slots move | within ~1 % across backends | you need affinity *and* even load *and* O(1) |
| Rendezvous (HRW) | O(N) per lookup | none — nothing shared | exactly 1/N, optimal | perfect | N is small and you want zero shared state |

*(Envoy defaults from `api/envoy/config/cluster/v3/cluster.proto`: `RingHashLbConfig.minimum_ring_size` "Defaults to 1024 entries, and limited to 8M"; `MaglevLbConfig.table_size` "must be prime number limited to 5000011. If it is not specified, the default is 65537"; `LeastRequestLbConfig.choice_count` "Defaults to 2 so that we perform two-choice selection if the field is not set".)*

**The least-request detail that surprises people.** Envoy's "least request" is not a global minimum. It picks `choice_count` hosts **at random** (default 2) and sends the request to whichever of those two has the fewest active requests. This is the *power of two random choices*: it costs O(1) instead of O(N), and it avoids the herding failure that a true global minimum produces — with a global minimum, every concurrently-arriving request sees the same "least loaded" host and they all pile onto it simultaneously. Two random choices gets you exponentially better balance than random, without the herd.

### 4. Maglev, derived and worked by hand

**The problem.** You want a stateless mapping from flow → backend such that (a) every LB instance computes the same answer without coordinating, (b) adding or removing one backend moves as few flows as possible, and (c) load is evenly spread. Modulo hashing gets (a) and (c) and fails (b) catastrophically. Ring hashing gets (a) and (b) and needs an enormous ring for (c).

**The construction.** Build a lookup table of size `M` (prime). Each backend gets a **permutation of the table's slots**, generated from two hashes of its name:

```
   offset(b) = h1(name_b) mod M
   skip(b)   = (h2(name_b) mod (M − 1)) + 1        ← +1 so skip is never 0
   perm_b[j] = (offset(b) + j × skip(b)) mod M     ← for j = 0, 1, 2, …
```

Because `M` is prime and `1 ≤ skip < M`, `perm_b` visits **every slot exactly once** before repeating — that is the reason M must be prime. Then fill the table by round-robining over backends, each taking the next slot in its own permutation that is still empty, until the table is full.

*(This is exactly what production implementations do. Envoy's `source/extensions/load_balancing_policies/maglev/maglev_lb.cc` builds each entry as `xxHash64(key) % table_size_` for the offset and `(xxHash64(key, 1) % (table_size_ - 1)) + 1` for the skip, sorts hosts first so the build order is stable, and fills with `while (table_[c] != nullptr) { c += skip; if (c >= size) c -= size; }`. The stable sort matters: an unstable host ordering changes the assignment even with an unchanged host set.)*

**Worked by hand, M = 7, three backends.** Suppose the hashes give:

```
   backend   offset   skip     permutation (offset + j·skip mod 7)
   ───────   ──────   ────     ───────────────────────────────────
   A            3       4      3, 0, 4, 1, 5, 2, 6
   B            0       2      0, 2, 4, 6, 1, 3, 5
   C            3       1      3, 4, 5, 6, 0, 1, 2

   FILL — round-robin A, B, C; each takes its next UNCLAIMED slot:

   round 1:  A wants 3 → free      → table[3] = A
             B wants 0 → free      → table[0] = B
             C wants 3 → TAKEN(A)  → next is 4, free → table[4] = C
   round 2:  A wants 0 → TAKEN(B)  → 4 TAKEN(C) → 1 free → table[1] = A
             B wants 2 → free      → table[2] = B
             C wants 5 → free      → table[5] = C
   round 3:  A wants 5 → TAKEN(C)  → 2 TAKEN(B) → 6 free → table[6] = A

                 slot:   0    1    2    3    4    5    6
   TABLE (N=3):          B    A    B    A    C    C    A
                        └── A:3   B:2   C:2  ── perfectly balanced at M=7 ──┘

   NOW ADD BACKEND D (offset 5, skip 3 → perm 5, 1, 4, 0, 3, 6, 2) AND REBUILD:

   round 1:  A→3, B→0, C→(3 taken)→4, D→5
   round 2:  A→(0 taken, 4 taken)→1, B→2, C→(5 taken)→6, D→(1 taken)→4? taken →0? taken →3? taken →6? taken →2? taken … D already has 5
   ...
                 slot:   0    1    2    3    4    5    6
   TABLE (N=4):          B    A    B    A    C    D    C

   FLOWS THAT MOVED:  slot 5 (C→D) and slot 6 (A→C)  =  2 of 7  ≈ 29 %
   COMPARE MODULO:    hash%3 vs hash%4 — a key stays only when h%3 == h%4,
                      which for uniform h happens about 1 time in 4 →
                      roughly 75 % of ALL flows remap.

   At M = 65537 and N = 100 → 101 the same construction moves ≈ 1/101 ≈ 1 %
   of slots, and every backend ends up within ~1 % of M/N entries.
```

**Two production caveats.**

- **Maglev bounds disruption; it does not eliminate it.** Envoy's own proto says so: "Maglev aims for 'minimal disruption' rather than an absolute guarantee." Established connections are protected by a *separate* mechanism — the LB's connection-tracking table — which is why the table is only the answer for *new* flows.
- **M must be ≫ N.** If M is close to N, the permutations cannot spread and the balance guarantee degrades toward modulo. 65537 against a few hundred backends leaves ~200–650 slots per backend, which is plenty; a hand-rolled implementation with M = 1024 and N = 500 is not Maglev, it is noise.

### 5. The tier underneath: ECMP and DSR

One LB box is a bandwidth ceiling and a failure domain of one. The escape is a *tier* of identical stateless LB instances, all advertising the same VIP, with the router spraying across them by ECMP hash.

```
   THE FULL L4 PATH, WITH DSR
   ══════════════════════════════════════════════════════════════════════════

                          client 203.0.113.9
                                  │  dst = VIP 198.51.100.7
                                  ▼
                        ┌──────────────────┐
                        │  router / fabric │  ECMP hash over the 5-tuple picks
                        │                  │  ONE of the equal-cost next hops.
                        └───┬───┬───┬───┬──┘  Every LB advertises the VIP via
                            │   │   │   │     BGP with the same metric.
                  ┌─────────┘   │   │   └─────────┐
                  ▼             ▼   ▼             ▼
             ┌────────┐   ┌────────┐ ┌────────┐ ┌────────┐
             │ LB-1   │   │ LB-2   │ │ LB-3   │ │ LB-4   │  ← STATELESS.
             │ Maglev │   │ Maglev │ │ Maglev │ │ Maglev │    Same backend set
             │ table  │   │ table  │ │ table  │ │ table  │    ⇒ same table
             └───┬────┘   └───┬────┘ └───┬────┘ └───┬────┘    ⇒ same answer.
                 │            │          │          │
                 │  encapsulate (IPIP/GUE) or rewrite MAC; the
                 │  INNER destination is still the VIP
                 └──────┬─────┴──────────┴──────────┘
                        ▼
                 ┌──────────────┐   backend has the VIP bound on a loopback/
                 │  backend pod │   dummy interface with ARP suppressed, so it
                 │  VIP on lo   │   accepts VIP-addressed packets but never
                 └──────┬───────┘   answers ARP for the VIP
                        │
                        │  REPLY: src = VIP, dst = client.
                        │  The LB is NOT on this path.
                        ▼
                    client 203.0.113.9

   WHY IT MATTERS, QUANTITATIVELY: for a typical request/response workload the
   response is 10–100× the request. For LLM inference the asymmetry is more
   extreme — a 2 KB prompt produces tens of KB of streamed tokens. Keeping the
   LB tier off the return path means its sizing is driven by REQUEST bytes and
   packet rate, not by total throughput.

   WHAT IT COSTS — and this is a security decision, not just a speed trick:
     • the backend sources packets from an address it does not own → cloud
       anti-spoofing (AWS source/dest check, GCP equivalent) drops them unless
       explicitly disabled for those instances
     • stateful firewalls and NAT devices in the return path see a flow whose
       forward direction they never observed → they drop it
     • conntrack on the backend must accept it; on a Kubernetes node this
       interacts with the rules from 01b.7
     • MTU: IPIP/GUE encapsulation adds 20–36 bytes, so the inner MTU shrinks
       (the same class of bug as the overlay MTU issue in lesson 05)
```

**The ECMP reshuffle failure, precisely.** A router's ECMP group hashes the 5-tuple into a bucket and maps buckets to next hops. When the number of active next hops changes — a link flaps, an LB instance withdraws its BGP route — a naive implementation recomputes `hash mod (number of live paths)`, which is exactly the modulo problem from §4 one layer down: **flows that had nothing to do with the failed path get moved anyway.** From inside the LB software this is invisible; it looks like "some connections reset for no reason." Two mitigations: **resilient ECMP hashing** (the switch keeps the bucket→next-hop map and only reassigns the buckets belonging to the dead path — check your silicon supports it and that it is enabled), and **stateless LB instances** so that a flow arriving at a different instance still computes the same backend. The second is why Maglev's design is what it is; the first is a switch feature you have to ask for.

### 6. Health checking: the mechanism and the cascade

**Active health checking** probes each backend on a schedule. Envoy's `HealthCheck` message requires you to set `timeout`, `interval`, `unhealthy_threshold`, and `healthy_threshold` explicitly — there are no defaults, which is itself a design statement. Two behaviours worth knowing:

- `no_traffic_interval` defaults to **60 s**: a cluster receiving no traffic is probed far less often, so a health-checked-but-idle backend can be stale by up to a minute when traffic arrives.
- For HTTP checks, a status outside both `expected_statuses` and `retriable_statuses` marks the host **immediately unhealthy**, ignoring `unhealthy_threshold` entirely. Set `retriable_statuses` if you want a 503 to count toward the threshold rather than eject on the first occurrence.
- During startup, a **single** successful check marks a host healthy regardless of `healthy_threshold`.

**Passive health checking (outlier detection)** ejects backends based on the responses real traffic gets. Envoy's defaults, read from `api/envoy/config/cluster/v3/outlier_detection.proto`:

| Field | Default | What it means |
|---|---|---|
| `consecutive_5xx` | **5** | consecutive server-side errors before ejection |
| `enforcing_consecutive_5xx` | **100** | % chance the verdict is acted on |
| `consecutive_gateway_failure` | **5** | consecutive 502/503/504 |
| `enforcing_consecutive_gateway_failure` | **0** | **off by default** |
| `interval` | **10 s** | ejection-analysis sweep period |
| `base_ejection_time` | **30 s** | ejection duration = base × (times ejected) |
| `max_ejection_time` | **300 s** (or `base_ejection_time` if larger) | cap on the above |
| `max_ejection_time_jitter` | **0 s** | jitter added to un-ejection, to avoid a synchronized return |
| `max_ejection_percent` | **10 %** | ceiling on how much of the cluster may be ejected |
| `success_rate_minimum_hosts` | **5** | fewer hosts than this → success-rate detection is off entirely |
| `success_rate_request_volume` | **100** | fewer requests than this in one interval → this host is excluded from the statistic |
| `success_rate_stdev_factor` | **1900** (÷1000 → **1.9**) | eject if success rate < mean − 1.9 × stdev |
| `enforcing_success_rate` | **100** | % chance a success-rate verdict is acted on |
| `failure_percentage_threshold` | **85 %** | eject if this host's failure rate ≥ 85 % |
| `enforcing_failure_percentage` | **0** | **off by default** |
| `failure_percentage_minimum_hosts` | **5** | |
| `failure_percentage_request_volume` | **50** | |

**Correction worth flagging:** there is no `min_request_amount` field in Envoy's outlier detection. The knobs that prevent low-traffic false positives are **`success_rate_request_volume` (100)** and **`success_rate_minimum_hosts` (5)** — a host that saw 12 requests in the 10-second interval is simply not included in the success-rate statistic, and a cluster with 4 hosts never runs success-rate detection at all. Together with `enforcing_*` (a percentage dial for "how often do we actually act on this verdict"), those are the sensitivity controls.

**The cascade, and why panic mode exists.**

```
   SHARED-DEPENDENCY CASCADE, WITH AND WITHOUT PANIC MODE
   ═══════════════════════════════════════════════════════════════════════

   Setup: 20 backends, all talking to one Postgres primary. p50 latency 8 ms.

   t=0     Postgres failover begins. Every backend's /healthz — which does a
           SELECT 1 — starts timing out. This is a SHARED dependency, so the
           failures are perfectly CORRELATED, not independent.

   t=0..15 Active checks fail. With unhealthy_threshold=3 and interval=5s,
           hosts start dropping out at t≈15 s.

   WITHOUT PANIC MODE  ─────────────────────────────────────────────────────
   t=15    18 of 20 hosts marked unhealthy. Two stragglers remain "healthy"
           only because their check happened to land between failures.
   t=15+   100 % of traffic → 2 hosts.  Each now carries 10× its normal load.
           Both saturate within seconds and fail their own checks.
   t=20    0 healthy hosts. The LB returns 503 for everything.
   t=45    Postgres recovers. All 20 hosts pass checks again — SIMULTANEOUSLY,
           because their check schedules were never de-synchronised.
   t=45+   All 20 return at once, all clients retry at once (see §9).
           The recovered database gets a 20× connection burst and falls over.

   WITH PANIC MODE (healthy_panic_threshold, default 50 %) ─────────────────
   t=15    healthy fraction 2/20 = 10 % < 50 % → PANIC.
           Envoy IGNORES health status entirely and balances across ALL 20.
   t=15+   Each host carries 1× its normal load — degraded, because Postgres
           is down, but nothing is being CRUSHED, and the two survivors are
           not being asked to do 10× work.
   t=45    Postgres recovers, health returns above 50 %, panic exits.
           `max_ejection_time_jitter` (set it! default is 0) de-synchronises
           the un-ejections so the recovery burst is smeared.

   THE BET PANIC MODE MAKES: a half-degraded fleet serving degraded responses
   beats a healthy minority being guaranteed to collapse. If that bet is wrong
   for your service — if a degraded response is worse than no response —
   set `fail_traffic_on_panic: true`, and Envoy will fail everything instead.
```

### 7. Slow start: the exact ramp

A backend that just passed its readiness probe is not ready for its share. Caches are cold, JIT is cold, connection pools are empty, and — on a GPU inference replica — model weights may be loaded but the CUDA graphs, the KV-cache allocator, and the first batch's kernels are not warm. Give it 1/N of traffic instantly and it queues, its latency spikes, outlier detection ejects it, and you have an oscillation.

Envoy's slow start scales the endpoint's weight over a configured window (`api/envoy/config/cluster/v3/cluster.proto`, `SlowStartConfig`):

```
   new_weight = weight × max( min_weight_percent , time_factor ^ (1 / aggression) )

     time_factor = time_since_host_start_seconds / slow_start_window_seconds
     aggression  = 1.0 by default → LINEAR ramp
     min_weight_percent = 10 % by default → a host never gets ~zero traffic,
                          which matters because zero traffic means zero
                          warming and zero passive health signal
```

Worked, for a 300-second model warm-up window with `aggression: 1.0`:

```
   t = 30 s   → time_factor 0.10 → weight ×0.10  (the 10 % floor also gives 0.10)
   t = 60 s   → 0.20 → ×0.20
   t = 150 s  → 0.50 → ×0.50
   t = 300 s  → 1.00 → ×1.00, slow start over

   With aggression = 2.0, weight = time_factor^0.5:
   t = 30 s   → 0.10^0.5 = 0.32   ← ramps FASTER early
   t = 150 s  → 0.50^0.5 = 0.71
   With aggression = 0.5, weight = time_factor^2:
   t = 30 s   → 0.10^2   = 0.01 → clamped to the 10 % floor
   t = 150 s  → 0.50^2   = 0.25   ← ramps SLOWER early
```

**Pick the window from the measured warm-up, not from a round number.** For an inference replica, that is the time from `Ready` to steady-state tokens/second — measure it with a synthetic request loop against a fresh replica. For a JVM service it is the time to JIT compilation steady state, typically tens of seconds to a few minutes.

Slow start applies only to `ROUND_ROBIN` and `LEAST_REQUEST` policies (it is a weight modifier, and the hash-based policies do not use weights the same way) — a fact that quietly rules out combining Maglev affinity with a gradual ramp.

### 8. Draining: the ordering that prevents resets

Removing a backend cleanly is an ordering problem, and Kubernetes makes it harder than it looks because **endpoint removal and pod termination are concurrent, not sequential**.

```
   WHAT ACTUALLY HAPPENS ON `kubectl delete pod` — TWO RACING TIMELINES
   ═══════════════════════════════════════════════════════════════════════

   API server marks pod Terminating (deletionTimestamp set)
        │
        ├──────────────► PATH 1: endpoint removal (the SLOW one)
        │                  EndpointSlice controller observes → updates slice
        │                  → kube-proxy on every node observes → rewrites rules
        │                  → LB control plane observes → updates its cluster
        │                  → LB data plane stops sending new requests
        │                TOTAL: hundreds of ms to SEVERAL SECONDS on a big
        │                cluster, and it is not bounded by anything you set.
        │
        └──────────────► PATH 2: pod termination (the FAST one)
                           kubelet runs preStop hook
                           kubelet sends SIGTERM
                           app exits
                        TOTAL: as fast as your app shuts down.

   IF PATH 2 FINISHES FIRST — which is the DEFAULT for a well-behaved app
   that exits promptly on SIGTERM — traffic is still being sent to a socket
   that is gone.  Symptom: a burst of connection-refused / RST at every
   deploy, proportional to your request rate × the propagation delay.

   THE FIX IS TO DELIBERATELY SLOW PATH 2:

     lifecycle:
       preStop:
         exec:
           command: ["/bin/sh","-c","sleep 15"]     # ← do NOT skip this
     terminationGracePeriodSeconds: 45              # > sleep + drain time

   Sequence with the hook:
     t=0    Terminating; preStop starts; endpoint removal begins in parallel
     t=0-5  LB control plane converges; new requests stop arriving
     t=15   preStop returns; SIGTERM delivered
     t=15+  app stops accepting, finishes in-flight requests, exits
     t=45   SIGKILL if it has not (grace period expired)

   The `sleep` is not a hack for a missing feature — it is the only portable
   way to hold the pod alive while an EVENTUALLY CONSISTENT control plane
   converges. Size it from your measured endpoint-propagation p99, not from
   folklore.
```

On the LB side, the complementary knob is **connection draining**: stop routing new requests to the host, but let existing ones complete up to a deadline. For an L7 proxy this also means sending `GOAWAY` on HTTP/2 connections so clients open a fresh connection elsewhere rather than continuing to multiplex onto a draining backend. AWS target groups call the same idea "deregistration delay"; Envoy calls it drain, and a mesh adds `DRAINING` health status via EDS.

### 9. Retries, budgets, and the jitter arithmetic

**The mechanism you must internalise:** an LB spreads load; it does not reduce it. Clients decide how much load exists. A perfectly tuned LB in front of undisciplined clients still collapses.

**Worked: the thundering herd.** 1000 clients, each with a request in flight, backend blips for 2 seconds.

```
   MODEL A — fixed exponential backoff, no jitter
   ──────────────────────────────────────────────
   All 1000 fail at t = 0 (they were all in flight when the blip started).
   attempt 1 at t = 1 s : 1000 requests in ~one RTT window (say 10 ms)
                          → instantaneous rate 1000 / 0.010 = 100,000 rps
                          against a service sized for, say, 2,000 rps.
                          → all fail again (overload), and the backend's
                            recovery is prevented by the load itself.
   attempt 2 at t = 3 s : another 1000-request spike
   attempt 3 at t = 7 s : another
   Offered load = clients × (1 + retries), concentrated in instants.
   THE BACKEND NEVER GETS A QUIET WINDOW.  Exponential growth does NOT
   de-synchronise anything: everyone's clock started at the same t=0, so
   everyone's t+1, t+3, t+7 are the same instants.

   MODEL B — full jitter:  sleep = random(0, min(cap, base × 2^attempt))
   ────────────────────────────────────────────────────────────────────
   attempt 1: each client sleeps uniform(0, 1 s)
              → 1000 retries smeared over 1 s → mean rate 1000 rps,
                peak within any 10 ms window ≈ 10 requests
              → the backend, sized for 2,000 rps, absorbs it
   Clients that succeed STOP RETRYING, so attempt 2's population is a
   fraction of attempt 1's. The herd dissolves instead of re-forming.

   PEAK REDUCTION: 100,000 rps → ~1,000 rps.  Two orders of magnitude,
   from one call to random().
```

**The proxy-side bound: a retry budget.** Per-client jitter requires every client to be correct. A retry budget enforces the bound at the proxy, where you control it: cap retries at a percentage of *active requests* to that cluster (Envoy's `RetryBudget`, with `budget_percent` and `min_retry_concurrency`). At 20 %, a cluster serving 1,000 concurrent requests permits at most 200 concurrent retries no matter what the clients do — so total offered load is bounded at 1.2×, not at (1 + max_retries)×.

**The multiplication trap.** Retries compose *multiplicatively* across layers. A → B → C → D, each configured with "3 attempts," means one client request can become 3 × 3 × 3 = **27** requests at D. The rules that follow: enable retries at exactly one layer (usually the edge-most one that can safely retry), set a retry budget rather than a retry count wherever you can, and never retry non-idempotent requests without an idempotency key.

### 10. The connection budget: port exhaustion, worked

An LB that SNATs (rewrites the source address to its own before forwarding) is limited by a resource nobody puts on a dashboard: the **ephemeral port space per (source IP, destination IP, destination port) tuple**.

```
   THE CONSTRAINT
   ══════════════════════════════════════════════════════════════════════
   A TCP connection is unique on the 5-tuple:
        (proto, src IP, src port, dst IP, dst port)
   The LB fixes proto=TCP, src IP=its own, dst IP=the backend, dst port=8080.
   The ONLY free field is src port.

   Linux default range:  $ sysctl net.ipv4.ip_local_port_range
                           net.ipv4.ip_local_port_range = 32768 60999
                         → 60999 − 32768 + 1 = 28,232 ports

   → 28,232 SIMULTANEOUS connections from one LB IP to one backend
     socket. Not per LB. Per (LB IP, backend IP, backend port) PAIR.

   WORKED — an inference gateway in front of a GPU pool
   ══════════════════════════════════════════════════════════════════════
   Given:  120,000 requests/second through the gateway
           mean backend service time 250 ms (LLM: prefill + decode)
           NO connection reuse (a fresh TCP connection per request —
           the common accident when a client library disables keep-alive)
           16 backend pods

   Little's Law:  concurrency = rate × service time
                              = 120,000 × 0.250 = 30,000 concurrent

   Per backend:   30,000 / 16 = 1,875 concurrent connections
                  → 1,875 < 28,232.  Fine so far.

   BUT: a closed connection does not free its port immediately. It sits in
   TIME_WAIT for 2×MSL, which Linux hardcodes at 60 s (TCP_TIMEWAIT_LEN in
   include/net/tcp.h) and does NOT expose as a sysctl.

   Connection CHURN = 120,000/s ÷ 16 backends = 7,500 closes/s per backend
   Ports held in TIME_WAIT = 7,500 × 60 s = 450,000

   450,000  ≫  28,232.  PORT EXHAUSTION, roughly 16× over budget.

   SYMPTOM: connect() fails with EADDRNOTAVAIL, which surfaces as
   sporadic 503 UF ("upstream connection failure") in Envoy access logs
   and as `nstat TcpExtTCPTimeWaitOverflow` / a flat ceiling on connection
   rate that does not respond to adding CPU.

   THE FIXES, in order of leverage
   ══════════════════════════════════════════════════════════════════════
   1. KEEP-ALIVE / connection pooling.  100 pooled connections per backend
      serving 7,500 rps each is 0 churn. This removes the problem rather
      than enlarging it, and it removes the DNS load from lesson 02 §3 and
      the conntrack pressure from 01b.7 at the same time. ALWAYS DO THIS FIRST.
   2. More destination tuples. 16 backends → 64 backends multiplies the
      budget by 4 because the tuple includes the destination IP.
   3. More source IPs on the LB.  Each additional source IP is another
      full 28,232-port space per destination.
   4. Widen the range:  net.ipv4.ip_local_port_range = 10240 65535
      → 55,296 ports. Less than a 2× gain; do not confuse it with a fix.
   5. net.ipv4.tcp_tw_reuse = 1 lets the kernel reuse a TIME_WAIT socket
      for a new OUTGOING connection when timestamps (RFC 7323) prove the
      new one is later. This is safe for a client/LB; `tcp_tw_recycle`
      was the dangerous one and was REMOVED from Linux in 4.12.
   6. DSR (§5) removes the LB from the return path but NOT from this
      budget, because the forward path still needs a tuple — unless the
      LB is doing pure L3 forwarding without SNAT, in which case the
      client's own address and port are preserved and the budget is
      the client population's, not the LB's.
```

The same arithmetic applies unchanged to a Kubernetes node doing masquerade for pod egress, to a NAT gateway (lesson 04), and to a mesh sidecar (lesson 06). **Do this calculation before you deploy, not during the incident.**

### 11. Routing for GPU inference: why the whole model changes

Everything above assumes requests are roughly interchangeable. LLM inference breaks that assumption harder than any workload you have balanced before:

- **Cost variance is ~100×.** A 20-token completion and a 4,000-token generation on the same replica differ by two orders of magnitude in occupancy. Round robin equalises *counts*, which is precisely the wrong invariant.
- **The cost is not knowable from the request alone.** Output length is not in the request. Prompt length is, and correlates with prefill cost, but decode cost depends on how long the model decides to talk.
- **State is sticky and valuable.** A replica that has already processed a request sharing a long system prompt holds that prefix in its KV cache. Routing a subsequent request with the same prefix to the *same* replica skips recomputing it. This makes affinity a *performance* optimisation, not just a session-management one — the opposite of the stateless web case where affinity is a liability.
- **Backpressure is visible if you ask.** Model servers export their own queue depth and KV-cache utilisation, so the balancer can route on the backend's *actual* state instead of inferring it from response times.

The current standard for doing this on Kubernetes is the **Gateway API Inference Extension**: an Envoy `ext_proc` filter (the "Endpoint Picker," EPP) that intercepts the request, scrapes model-server metrics, and tells the gateway which endpoint to use. Its model-server protocol requires these metrics, with the concrete vLLM names (from `docs/proposals/003-model-server-protocol/README.md` in `kubernetes-sigs/gateway-api-inference-extension`):

| Metric | Type | vLLM name | Used for |
|---|---|---|---|
| `TotalQueuedRequests` | gauge | `vllm:num_requests_waiting` | queue-depth-aware routing — the direct backpressure signal |
| `TotalRunningRequests` | gauge | `vllm:num_requests_running` | in-flight batch occupancy |
| `KVCacheUtilization` | gauge | `vllm:kv_cache_usage_perc` | a replica near KV-cache capacity will start preempting/recomputing |
| `BlockSize` (optional) | gauge | `vllm:cache_config_info{block_size}` | input to the prefix-cache scorer |
| `NumGPUBlocks` (optional) | gauge | `vllm:cache_config_info{num_gpu_blocks}` | input to the prefix-cache scorer |
| LoRA adapter state | gauge | `vllm:lora_requests_info{max_lora, running_lora_adapters, waiting_lora_adapters}` | adapter-affinity routing: send a request to a replica that already has its adapter resident |

Prefix-cache-aware scheduling has been in the EPP since v0.4.0 and depends on the model server implementing prefix-cache reuse (vLLM's automatic prefix caching). The project has since partnered with llm-d, and the EPP itself has moved to `llm-d/llm-d-router` while the `InferencePool` API and conformance tests stay in the Gateway API repository — worth knowing so you look in the right place.

**The one-line summary for a design review:** for stateless web, prefer least-request with two choices and treat affinity as a cost; for LLM inference, prefer explicit queue-depth and KV-cache signals from the model server and treat prefix affinity as a *saving*, because recomputing a shared 8 KB system prompt on the wrong replica costs more than any routing sophistication you could add.

## Perspectives

**Distributed-systems theory.** Maglev is one point in a design space, not the answer. The space is (lookup cost) × (shared state) × (disruption bound) × (balance quality): HRW gets optimal disruption and perfect balance with zero shared state at O(N) per lookup; ring hashing gets O(log R) with a big ring and mediocre balance; Maglev buys O(1) and ~1 % balance for a build step and M entries of memory. A staff engineer names which member a system uses and why, and knows that all three bound disruption to ≈1/N — none of them eliminates it, which is why every one of them is paired with a connection table.

**Client behaviour.** LB-side degradation logic (panic mode, outlier detection, slow start) and client-side discipline (jitter, budgets, idempotency) are two halves of one amplification problem. The LB decides *where* load goes; the clients decide *how much* exists. An interviewer asking "why did the outage get worse after the first blip" is testing whether you see both halves — and the answer is almost always synchronized retries plus a health-check verdict that removed capacity at the moment capacity was scarcest.

**Hardware and ECMP.** The failure domain extends one layer below the LB process, into the routing fabric's own hashing. Lose a link and a naive ECMP implementation rehashes flows that never touched it. That reshuffle is invisible from inside the LB and reads as "some connections reset for no reason." Knowing to ask "is resilient ECMP hashing enabled on this switch model?" is the difference between knowing Maglev and knowing why Maglev's stateless-instance design exists.

**GPU-inference economics.** Request cost variance turns routing into scheduling. The dollar framing: an 8-GPU node costs on the order of $30–$40/hr on demand (verify your provider and region — these move), so a routing policy that leaves 20 % of a 64-node inference fleet idle while the rest queues is burning roughly the cost of 12 nodes continuously. Queue-depth-aware routing is not a refinement; it is the difference between buying 64 nodes and buying 80.

## Real-world use cases

- **Envoy's outlier-detection defaults as a design document.** Read directly from `api/envoy/config/cluster/v3/outlier_detection.proto`. What it shows: the *safe* detectors are on by default (`consecutive_5xx = 5`, enforced at 100 %) while the *aggressive* ones are shipped off (`enforcing_consecutive_gateway_failure = 0`, `enforcing_failure_percentage = 0`), and every statistical detector has a volume floor (`success_rate_request_volume = 100`, `success_rate_minimum_hosts = 5`). That is an explicit statement that low-traffic false positives are considered worse than slow ejection — a judgement you may need to reverse for a high-traffic tier, and should reverse deliberately.
- **Katran (Meta), an XDP/eBPF L4 load balancer** (https://github.com/facebookincubator/katran) — a production Maglev-style consistent-hashing balancer running in the XDP hook described in lesson 01 §14, i.e. before `sk_buff` allocation. What it shows: the table-based approach and the DSR return path are convergent production engineering across at least three independent hyperscale implementations, not a single paper's idea. *(Repository referenced; the source was not read in this pass — the Maglev construction described in §4 is verified against Envoy's implementation instead.)*
- **The Gateway API Inference Extension model-server protocol** (`kubernetes-sigs/gateway-api-inference-extension`, `docs/proposals/003-model-server-protocol/README.md`), read directly. What it shows: the industry has converged on a *specific* set of routing signals for LLM serving — queued requests, running requests, KV-cache utilisation, block size, GPU block count, and LoRA adapter residency — with concrete metric names across vLLM, TensorRT-LLM, trtllm-serve, and SGLang. If you are designing an inference tier, this is the contract to build against rather than inventing one.

## Worked example

**Scenario.** An inference gateway fronts 16 vLLM replicas on 8×H100 nodes. During a routine node-pool upgrade (four nodes drained one at a time), p99 goes from 4 s to timeouts, and the incident lasts 25 minutes rather than the expected few seconds per node. Reconstruct the mechanism, then fix each contributing factor with a number.

**Step 1 — establish what changed and when.** Three things happen when a node drains: pods terminate, endpoints are removed, and replacement pods start cold. Check whether the damage tracks removals or arrivals.

```bash
$ kubectl get events --sort-by=.lastTimestamp -n inference | tail
… Killing    pod/vllm-7c9d-mnk2   Stopping container vllm
… Scheduled  pod/vllm-7c9d-x4l8   Successfully assigned to node-42
… Started    pod/vllm-7c9d-x4l8   Started container vllm

# Envoy's own view: which response flags are set on the failures?
$ kubectl logs -n gateway deploy/envoy | grep -oP '"\K(UF|UO|URX|UH|NR|DC)(?=")' \
  | sort | uniq -c | sort -rn
   4821 UO      ← upstream overflow: a circuit breaker was tripped
   1902 UF      ← upstream connection failure
    311 UH      ← no healthy upstream
```

`UO` dominating means requests are being rejected by a **circuit breaker**, not failing on the network. That points at concurrency limits, not connectivity.

**Step 2 — the cold-replica half.** A fresh vLLM replica passes its readiness probe when the HTTP server is up. Measure when it is actually fast:

```bash
$ for i in $(seq 1 20); do
>   curl -s -o /dev/null -w '%{time_total}\n' -X POST \
>     http://$NEW_POD:8000/v1/completions \
>     -d '{"model":"llama-3-70b","prompt":"hello","max_tokens":16}'
>   sleep 10
> done
9.84   ← Ready, but the first requests are ~40× steady state
7.21
4.03
1.15
0.31
0.26   ← steady state reached at roughly t+50 s after Ready
0.25
...
```

**The gap between `Ready` and useful is ~50 seconds, and the balancer knows nothing about it.** During that window this replica is receiving 1/16 of 120,000 rps while serving at a fraction of its rate, so its queue grows without bound and the gateway's per-endpoint concurrency limit trips — hence `UO`.

**Step 3 — the draining half.**

```bash
$ kubectl get deploy -n inference vllm -o jsonpath='{.spec.template.spec}' | jq '{
    preStop: .containers[0].lifecycle.preStop,
    grace: .terminationGracePeriodSeconds }'
{
  "preStop": null,
  "grace": 30
}
```

No `preStop` hook. So the moment a pod is marked Terminating, vLLM gets SIGTERM and stops accepting — while the gateway is still sending it traffic for as long as endpoint propagation takes. Measure that:

```bash
$ kubectl delete pod -n inference vllm-7c9d-mnk2 &
$ while true; do
>   date +%s.%N; kubectl get endpointslice -n inference -l kubernetes.io/service-name=vllm \
>     -o jsonpath='{.items[*].endpoints[*].addresses[0]}' | tr ' ' '\n' | wc -l
>   sleep 0.2
> done | head -20
1770000000.10  16
1770000000.31  16
...
1770000002.94  15      ← 2.8 s from delete to endpoint removal
```

**2.8 seconds × 120,000 rps ÷ 16 endpoints = ~21,000 requests routed to a dead socket per pod removal**, and four pods were drained. That is the `UF` count's origin.

**Step 4 — the retry half, which turns 25,000 failures into an outage.**

```yaml
# The gateway's route config, as found:
retryPolicy:
  numRetries: 3
  retryOn: 5xx,reset,connect-failure
# and NO retry budget.
```

```
   Failed requests from steps 2–3, per drained node:  ~21,000 (UF) + backlog
   With numRetries: 3 and no budget, each becomes up to 4 attempts:
        21,000 × 4 = 84,000 requests
   All of them arrive within the ~3 s propagation window, at the exact
   moment capacity is 15/16 of normal and one replica is cold.
   Offered load during the window:
        120,000 × (84,000/21,000 amplification on the failing fraction)
        → an effective ~1.5–2× spike on 15/16 of the capacity
   → more timeouts → more retries → the next node drain starts before the
     previous one has recovered → the 25-minute duration.
```

**Step 5 — the fix, with a number attached to each part.**

```yaml
# (a) Give the gateway the replica's real warm-up curve.
#     Envoy cluster (or Istio DestinationRule warmupDurationSecs).
loadBalancingPolicy:
  policies:
    - typedExtensionConfig:
        name: envoy.load_balancing_policies.least_request
        typedConfig:
          "@type": type.googleapis.com/envoy.extensions.load_balancing_policies.least_request.v3.LeastRequest
          choiceCount: 2                    # power-of-two choices (this IS the default)
          slowStartConfig:
            slowStartWindow: 60s            # measured: ~50 s to steady state, +20 % margin
            aggression: { defaultValue: 1.0 }   # linear
            minWeightPercent: { value: 10 }     # never zero — it needs traffic to warm
```

```yaml
# (b) Stop sending traffic to a socket that is going away.
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 90     # > preStop + longest in-flight generation
      containers:
        - name: vllm
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh","-c","sleep 8"]   # measured p99 propagation 2.8 s ×~3
```

```yaml
# (c) Bound the amplification at the proxy rather than trusting clients.
retryPolicy:
  numRetries: 2
  retryOn: reset,connect-failure            # NOT 5xx — a 503 from an overloaded
                                            # GPU replica must not be retried;
                                            # retrying it is how you keep it overloaded
  retryBackOff:
    baseInterval: 0.1s                      # Envoy applies full jitter over
    maxInterval: 2s                         # [0, base×2^n) capped at maxInterval
  retryBudget:
    budgetPercent: { value: 20 }            # ≤20 % of active requests may be retries
    minRetryConcurrency: { value: 3 }
```

```yaml
# (d) Make outlier detection stop ejecting a cold replica.
outlierDetection:
  consecutive5xx: 5
  interval: 10s
  baseEjectionTime: 30s
  maxEjectionTime: 300s
  maxEjectionTimeJitter: 10s        # default is 0 — set it, so recovered hosts
                                    # do not all return in the same instant
  maxEjectionPercent: 10            # never eject more than 10 % of a 16-replica pool
  successRateRequestVolume: 100     # a cold replica serving 40 req/interval is
                                    # excluded from the success-rate statistic anyway
commonLbConfig:
  healthyPanicThreshold: { value: 50 }   # spread over everyone rather than crush a few
```

**Step 6 — the result, and the residual risk.**

```
   Requests to dead sockets per drain:  ~21,000  →  ~0
        (preStop 8 s > measured 2.8 s propagation p99)
   Cold-replica overload:               UO 4,821 →  ~0
        (60 s linear ramp; at t=10 s the replica gets 17 % of a share
         instead of 100 %, matching its ~20 % of steady-state throughput)
   Retry amplification on the failing fraction:  4×  →  ≤1.2×
        (budget 20 %, and 5xx removed from retryOn)
   Drain duration per node:             ~6 min  →  ~90 s

   RESIDUAL RISK, stated: the preStop sleep is calibrated to a MEASURED
   propagation p99 on today's cluster size. EndpointSlice propagation
   scales with cluster size and Service count (lesson 05), so re-measure
   after any large growth, and alert on the gap between "pod Terminating"
   and "endpoint removed" rather than assuming 8 s is permanent.
```

## Practice
<feeds [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)>

**Task A — build the Maglev table.** Implement the construction from §4 in any language: `offset = h1(name) mod M`, `skip = (h2(name) mod (M−1)) + 1`, round-robin fill. Then:

1. With M = 65537 and N = 100 backends, hash 100,000 synthetic flow keys and record each key's backend.
2. Add one backend, rebuild, and count how many keys changed backend. Compare against the theoretical ≈1/101.
3. Do the same with `hash(key) mod N` for N = 100 → 101 and record the remap fraction.
4. Report the per-backend entry counts' min, max, and standard deviation as a percentage of M/N — this is the "~1 % balance" claim, tested.
5. Repeat with M = 127 and N = 100 and show the balance guarantee degrading, to demonstrate why M ≫ N matters.

**Task B — the thundering-herd simulation.** Simulate 1,000 clients against a backend with a hard capacity of 2,000 rps that is unavailable for 2 seconds. Implement three retry policies: fixed interval, exponential without jitter, and full jitter (`sleep = random(0, min(cap, base × 2^attempt))`). Plot offered load per 10 ms bucket for each. Report the peak instantaneous rate and the time to full recovery for all three. Then add a proxy-side retry budget of 20 % of active requests and show what it does to the fixed-interval case — which is the realistic scenario, because you cannot fix every client.

**Task C — measure your endpoint propagation.** On a real cluster, delete a pod and time the gap between `deletionTimestamp` being set and the address disappearing from the EndpointSlice, sampling at 200 ms. Repeat 20 times and report p50/p99. Then size a `preStop` sleep at ≈3× p99 and demonstrate, with a load generator running, that connection-refused errors go to zero.

**Task D — the port-budget worksheet.** For a service you actually run, compute: requests/second, mean service time, concurrency by Little's Law, connections per backend, connection churn per second, TIME_WAIT occupancy at 60 s, and the resulting port headroom against `ip_local_port_range`. State which of the six mitigations in §10 you would apply and in what order, with the multiplier each buys.

**Acceptance criteria**

- [ ] A Maglev implementation plus a table of remap percentages (Maglev vs modulo) and per-backend balance statistics for at least two table sizes.
- [ ] A herd simulation with plotted offered load for three retry policies and a stated peak-reduction factor.
- [ ] Measured endpoint-propagation p50/p99 from a real cluster, with a `preStop` value derived from it and a before/after error count.
- [ ] A completed port-budget worksheet with the arithmetic shown and a prioritised mitigation list.
- [ ] A runbook decision section for a GPU inference tier: routing policy (with the vLLM metric names it consumes), slow-start window justified by a measured warm-up curve, panic threshold, outlier-detection volume floors, and retry budget — each choice defended against a shared-dependency blip.

## Common pitfalls
- **"Consistent hashing means zero disruption."** It bounds disruption to ≈1/N of flows per membership change. Envoy's own Maglev documentation says the goal is "minimal disruption rather than an absolute guarantee." Established flows survive only because a *separate* connection table protects them.
- **"Least request finds the least loaded host."** Envoy's `LEAST_REQUEST` samples `choice_count` hosts at random — **default 2** — and picks the better of them. That is deliberate: a true global minimum causes every concurrently-arriving request to herd onto the same host. If you were relying on a global minimum, you were relying on something that does not exist.
- **"Health checks alone determine rotation."** Below the panic threshold (default **50 %**), Envoy ignores health status entirely and balances across every host. Health status governs the normal case, not the collapse case — and if degraded responses are worse than none for your service, you must opt in to `fail_traffic_on_panic`.
- **Tuning `min_request_amount` in Envoy outlier detection.** That field does not exist. The volume floors are `success_rate_request_volume` (default **100** requests per 10 s interval) and `success_rate_minimum_hosts` (default **5**), plus the `enforcing_*` percentages. Configuring a field that is not in the schema either errors or is silently ignored depending on your control plane.
- **"Round-robin is fair."** It is fair in request *count*, not request *cost*. For LLM inference, where a 20-token completion and a 4,000-token generation differ ~100× in occupancy, equal counts produce wildly unequal load — and the visible symptom is GPUs that are simultaneously queued and underutilised.
- **"Exponential backoff is enough."** Exponential growth does not de-correlate clients that all failed at the same instant; their t+1, t+2, t+4 are the same instants. Only randomisation breaks the synchronisation. Full jitter also reduces *total* work, because early successes leave the retry population.
- **Retrying 503s from an overloaded backend.** A 503 caused by overload is the backend telling you it has no capacity; retrying it adds load to the thing that is out of load. Retry `reset` and `connect-failure`; be extremely deliberate about retrying `5xx`.
- **"DSR just means faster."** DSR requires backends to source packets from an address they do not own. Cloud anti-spoofing checks, stateful firewalls, and NAT devices in the return path will silently drop that traffic. It is a trust-boundary decision with a performance benefit, not a performance knob.
- **Deploying without a `preStop` hook because "the app handles SIGTERM properly."** Handling SIGTERM properly makes it *worse*: the app exits promptly while the eventually-consistent endpoint propagation is still in flight, so traffic arrives at a closed socket. The hook exists to hold the pod alive during that window.

## Self-check

**1. Why does least-outstanding-requests matter far more for GPU inference than for a stateless web tier, and what exactly does Envoy's `LEAST_REQUEST` do?**

Because request cost variance is ~100× for LLM inference (a 20-token completion versus a 4,000-token generation differ by two orders of magnitude in replica occupancy), so equalising request *counts* — which is all round robin does — produces wildly unequal load, and the symptom is a fleet that is simultaneously queued and underutilised. Least-request tracks in-flight work rather than counts. The mechanism detail: Envoy's `LEAST_REQUEST` is not a global minimum; it samples `choice_count` hosts at random (**default 2**) and picks the one with fewer active requests — the power-of-two-choices algorithm, chosen because it is O(1) and because a true global minimum makes all concurrently-arriving requests herd onto the same host. For inference specifically, even that is a proxy signal; the better signal is the model server's own `vllm:num_requests_waiting` and `vllm:kv_cache_usage_perc`, consumed by an endpoint picker.

**2. What does Envoy's panic threshold do, what cascade does it prevent, and when should you disable it?**

When the fraction of healthy hosts falls below `healthy_panic_threshold` (**default 50 %**), Envoy ignores health status entirely and balances across all hosts. It prevents the shared-dependency cascade: a common backend (a database, an auth service) blips, every host fails its health check simultaneously because the failures are *correlated*, the LB ejects the majority, and the surviving minority receives 100 % of traffic and collapses — converting a degraded service into a total outage. Panic mode bets that a half-degraded fleet serving degraded responses beats a healthy minority guaranteed to be crushed. You disable that bet with `fail_traffic_on_panic: true` when a degraded response is *worse* than no response — for example, a payment authorisation that would return a wrong answer, or a service whose partial results poison a downstream cache.

**3. Why is a newly-`Ready` GPU inference replica not ready for full traffic, and what exactly does slow start do about it?**

`Ready` means a health endpoint responded. It does not mean CUDA graphs are captured, the KV-cache allocator is primed, the first batch's kernels are warm, or (for a JVM) that JIT has reached steady state — in the worked example above the measured gap was ~50 seconds, during which the first requests were ~40× steady-state latency. Envoy's slow start scales the endpoint's weight by `max(min_weight_percent, time_factor^(1/aggression))` where `time_factor = time_since_start / slow_start_window`, with `aggression` defaulting to **1.0** (a linear ramp) and `min_weight_percent` defaulting to **10 %** so the host still gets some traffic — which it needs, both to warm up and to produce a passive health signal. Size the window from the measured warm-up curve, not a round number, and note that slow start applies to weight-based policies (round robin, least request), not to the hash-based ones.

**4. Why must LB instances behind an ECMP tier be stateless, and what does a dead upstream router do to flows that never traversed it?**

An ECMP group hashes the flow tuple into a bucket and maps buckets to next hops. When the live-path count changes — a link flaps, an instance withdraws its BGP route — a naive implementation recomputes the mapping over the surviving paths, which is the modulo problem one layer down: **flows that were never on the failed path can be reassigned to a different LB instance.** A stateful LB has no record of a flow that arrives at it mid-connection and will drop or mishandle it. A stateless, Maglev-table-driven LB computes the same backend from the same host set no matter which instance the packet reaches, so the reassignment is invisible. The mitigation on the network side is **resilient ECMP hashing**, where the switch retains the bucket→next-hop map and reassigns only the buckets belonging to the dead path — verify your silicon supports it *and* that it is enabled, because the failure it prevents looks like "random connection resets" and is easily blamed on the application.

**5. Why does production Maglev use a table size M that is much larger than N, and why must it be prime?**

**Prime** is a correctness requirement, not a preference. Each backend's permutation is `perm[j] = (offset + j × skip) mod M` with `1 ≤ skip < M`. When M is prime, `skip` is necessarily coprime with M, so the sequence visits every one of the M slots exactly once before repeating — which is what guarantees every backend can always find a free slot and that the fill terminates. With a composite M, a `skip` sharing a factor with M cycles through only a subset of slots, and the fill can stall or badly skew. **M ≫ N** is a quality requirement: the near-uniform balance emerges from each backend having a long, well-spread preference list relative to how many slots it will claim (M/N of them). At M = 65537 and a few hundred backends that is hundreds of slots each and balance lands within about 1 %; at M close to N there is no room to spread and balance degrades toward the modulo case. Envoy's default is 65537 with a hard cap of 5000011, both prime.

**6. Your gateway SNATs to 16 backends and handles 120,000 rps with a 250 ms mean service time and no connection reuse. Will it work? Show the arithmetic.**

No. Concurrency by Little's Law is `120,000 × 0.250 = 30,000`, which is 1,875 per backend — comfortably under the ~28,232 ephemeral ports in Linux's default `ip_local_port_range` of 32768–60999. The killer is churn, not concurrency: at 120,000 closes/second across 16 backends that is 7,500 closes/s per backend, and each closed connection holds its port in `TIME_WAIT` for 60 seconds (`TCP_TIMEWAIT_LEN`, hardcoded in `include/net/tcp.h`, not a sysctl). That is `7,500 × 60 = 450,000` ports needed against 28,232 available — about 16× over budget. The symptom is `connect()` returning `EADDRNOTAVAIL`, surfacing as sporadic `UF` in Envoy's access log and a connection-rate ceiling that does not respond to more CPU. The fix order matters: **connection pooling first** (it removes the churn entirely and simultaneously cuts DNS load and conntrack pressure), then more destination tuples (more backends multiply the budget because the tuple includes the destination IP), then more source IPs on the gateway, then `tcp_tw_reuse=1`, and only then widening the port range — which buys less than 2× and is often mistaken for a fix.

**7. A node drain causes a burst of connection-refused errors even though the application shuts down cleanly on SIGTERM. Why does shutting down *cleanly* make it worse, and what is the fix?**

Because endpoint removal and pod termination run **concurrently**, not in sequence. Setting `deletionTimestamp` starts two independent races: the EndpointSlice controller updating the slice, kube-proxy and the LB control plane observing it, and the data plane converging (hundreds of milliseconds to several seconds, unbounded by any setting you control); and the kubelet sending SIGTERM to the container. An application that exits *promptly* on SIGTERM wins that race, so its socket is gone while the load balancer is still sending it traffic. The fix is to deliberately delay path 2 with a `preStop` hook (`sleep N`) sized from your measured endpoint-propagation p99 — in the worked example, 2.8 s measured, 8 s configured — with `terminationGracePeriodSeconds` set larger than the sleep plus the longest in-flight request. Measure it rather than copying a number: propagation time scales with cluster size and Service count, so alert on the gap between "pod Terminating" and "endpoint removed" instead of trusting a constant.

## Connections & what's next
This lesson's mechanisms recur throughout the module: **Lesson 05** (Kubernetes networking) shows how a Service VIP is resolved by the dataplane and why EndpointSlice propagation — the thing your `preStop` hook is waiting on — has the latency it has; **Lesson 06** (service mesh) reuses outlier detection, panic thresholds, and retry budgets at the sidecar layer, where they are configured through Istio's `DestinationRule` rather than raw Envoy; **Lesson 07** (GPU/RDMA networking) is where request-tier routing hands off to the fabric and where you will see why none of this machinery may sit on the collective path. Module 09's GPU-fabric artifact is the reference for that fabric-level picture — this lesson deliberately stayed at the request tier.

Next: **[04-cloud-networking.md](04-cloud-networking.md)** — the LB tier's cross-zone spray is one of the two dominant egress-cost culprits, so this is where the LB discussion turns into a dollar figure on the monthly bill.

## References & further reading

**Primary sources — read directly for this lesson**

1. `api/envoy/config/cluster/v3/cluster.proto`, Envoy `main` (https://github.com/envoyproxy/envoy/blob/main/api/envoy/config/cluster/v3/cluster.proto) — source of every load-balancing default in §3 and §7: `MaglevLbConfig.table_size` default **65537**, must be prime, capped at 5000011; `RingHashLbConfig.minimum_ring_size` default **1024**, max 8 M, `XX_HASH` default hash function; `LeastRequestLbConfig.choice_count` default **2** (power-of-two choices); `SlowStartConfig` with `aggression` default 1.0, `min_weight_percent` default 10 %, and the formula `new_weight = weight × max(min_weight_percent, time_factor^(1/aggression))`; `CommonLbConfig.healthy_panic_threshold` default **50 %** and `fail_traffic_on_panic`; `ZoneAwareLbConfig.min_cluster_size` default 6.
2. `api/envoy/config/cluster/v3/outlier_detection.proto`, Envoy `main` — the complete outlier-detection default table in §6. **Correction to the previous version of this lesson:** it referred to a `min_request_amount` field, which does not exist in this schema; the volume floors are `success_rate_request_volume` (100) and `success_rate_minimum_hosts` (5), with `failure_percentage_request_volume` (50) and `failure_percentage_minimum_hosts` (5) for the failure-percentage detector, and the `enforcing_*` percentages as the sensitivity dials.
3. `api/envoy/config/core/v3/health_check.proto`, Envoy `main` — that `timeout`, `interval`, `unhealthy_threshold`, and `healthy_threshold` are all **required** with no defaults; `no_traffic_interval` default **60 s**; that an HTTP status outside `expected_statuses` and `retriable_statuses` marks a host immediately unhealthy regardless of the threshold; and that a single successful check suffices during startup.
4. `source/extensions/load_balancing_policies/maglev/maglev_lb.cc`, Envoy `main` — the Maglev construction verified in code and reproduced in §4: `offset = xxHash64(key) % table_size`, `skip = (xxHash64(key, 1) % (table_size − 1)) + 1`, the stable sort of hosts before building (an unstable order changes assignments even with an unchanged host set), the weighted round-robin fill with `target_weight_`/`max_normalized_weight`, and the linear probe `while (table_[c] != nullptr) { c += skip; if (c >= size) c -= size; }`.
5. `docs/proposals/003-model-server-protocol/README.md`, `kubernetes-sigs/gateway-api-inference-extension` (https://github.com/kubernetes-sigs/gateway-api-inference-extension) — the model-server metrics contract in §11 with the exact vLLM names (`vllm:num_requests_waiting`, `vllm:num_requests_running`, `vllm:kv_cache_usage_perc`, `vllm:cache_config_info{block_size,num_gpu_blocks}`, `vllm:lora_requests_info{max_lora,running_lora_adapters,waiting_lora_adapters}`) and the equivalents for Triton TensorRT-LLM, trtllm-serve, and SGLang; plus the requirement that model servers implement the OpenAI Completions/Chat APIs and support prefix-cache reuse.
6. `README.md`, `kubernetes-sigs/gateway-api-inference-extension` — the architecture in §11: the Endpoint Picker as an Envoy `ext_proc` external processor, prefix-cache-aware scheduling since v0.4.0, the llm-d partnership, and the migration of the EPP and associated APIs to `llm-d/llm-d-router` while `InferencePool` and conformance tests remain in this repository.
7. `include/net/tcp.h`, Linux master — `TCP_TIMEWAIT_LEN` (60 s) used in §10's port-exhaustion arithmetic, and the fact that it is a compile-time constant rather than a sysctl.
8. `Documentation/networking/ip-sysctl.rst`, Linux master — `ip_local_port_range` semantics and `tcp_tw_reuse`; also the record that `tcp_tw_recycle` was removed (it is absent from current documentation and source, having been dropped in Linux 4.12).

**Sources named but not fetched in this pass — do not treat the wording as verified**

9. Eisenbud et al., "Maglev: A Fast and Reliable Software Network Load Balancer," NSDI '16 (https://www.usenix.org/conference/nsdi16/technical-sessions/presentation/eisenbud) — the origin of the table construction. `usenix.org` was not reachable from this environment, so the algorithm in §4 is derived from and verified against Envoy's implementation instead, and the ~1 % balance figure is stated as what the construction produces at M ≫ N rather than quoted from the paper.
10. Katran (https://github.com/facebookincubator/katran), Cloudflare's Unimog write-up, and the Netflix edge-load-balancing post — cited as convergent production implementations of the same design. The Katran repository is reachable but its source was not read in this pass; the Cloudflare and Netflix blogs are blocked by this environment's egress policy. No figures from any of them are asserted here.
11. AWS Builders' Library, "Timeouts, retries, and backoff with jitter" (https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) — the canonical treatment of full jitter. `aws.amazon.com` is blocked by this environment's egress policy; the jitter arithmetic in §9 is derived from first principles here (uniform smearing of a synchronized population over the backoff interval) rather than quoted, and the `random(0, min(cap, base × 2^attempt))` form is the standard full-jitter definition.
