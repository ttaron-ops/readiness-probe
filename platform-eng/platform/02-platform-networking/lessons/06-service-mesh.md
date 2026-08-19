---
lesson: "A02.6"
title: "Service mesh"
module: "A-02"
concept: "mesh cost/benefit & ambient"
status: not-started
est_time: "4.5 hrs"
prev: "05-kubernetes-networking.md"
next: "07-gpu-and-rdma-networking.md"
artifacts: ["mesh-cost-budget", "mesh-decision-checklist", "retry-budget-worksheet"]
sources: 11
---

# A02.6 · Service mesh

> **Concept.** A mesh trades N bespoke networking problems for one big consistent one — worth it only past a complexity threshold, and never on the RDMA data path.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Where this fits
Lesson 05 established the dataplane substrate: how a Service VIP actually resolves to a backend pod, and what that resolution costs in latency, conntrack state, and security posture. A mesh doesn't replace that substrate — it sits on top of it, adding a proxy hop *after* the VIP has already resolved. This lesson asks the staff-level question that follows naturally: given everything lesson 05 costs already, what does an additional proxy layer buy, what does it tax, and when is the honest answer "don't add it"?

## Why this matters
Adopting a mesh is a staff-level *judgment* call, not a checkbox: it uniformly buys mTLS, L7 policy, and telemetry, but it taxes every request's p99, every pod's CPU/mem, and every incident's debugging surface. The difference between a senior and a staff engineer here is being able to *quantify the tax*, name the Envoy internals that cause retry-storm amplification, and know when the answer is "no mesh." On GPU fleets it is also a placement decision — mesh the inference frontend, explicitly exclude the training/RDMA network — where a wrong call is catastrophic, not merely costly.

## What's new here (calibration)
- You already know what a sidecar is, that a mesh gives mTLS/retries/telemetry, that Envoy is the data plane, and basic traffic-splitting/canary — none of that is re-taught here.
- **New: the actual numbers, from Istio's own published benchmarks, across five releases.** The figure most engineers quote — "a mesh adds ~2.7 ms" — is from Istio 1.10 and is roughly **10× too high** for a current release. You will learn the real numbers, their methodology, and how to reproduce them.
- New: the sidecar and ambient datapaths drawn hop by hop, with the actual port numbers and the actual number of proxy traversals per request.
- New: Envoy's config model (xDS resource types, listeners/clusters/routes) as the thing that determines proxy memory and control-plane cost, and the scoping mechanism that bounds it.
- New: Istio's shipped connection-pool and circuit-breaker defaults, which are effectively **unlimited**, and what that means for the amplification arithmetic from lesson 03.
- New: the mesh rollout as a distributed-systems project with its own failure modes — injection races, control/data-plane version skew, xDS push storms.

## Core concepts

### 1. The problem a mesh exists to solve

Start from the failure it prevents, not the feature list.

You have 200 services written in 6 languages by 30 teams. Each of them needs, at minimum: TLS to its peers, a retry policy, a timeout, a circuit breaker, request-level metrics, and distributed-trace propagation. Without a mesh, each of those is implemented **per language, per framework, per team** — six retry implementations with six different jitter behaviours, thirty timeout policies with thirty different defaults, and an mTLS story that is only as good as the least-diligent team's certificate handling.

**The mesh's actual trade is: N inconsistent implementations of the same six problems, for one consistent implementation plus one large new operational dependency.** That framing is what makes the decision analysable, because both sides can be counted:

| Concern | Without a mesh | With a mesh | The catch |
|---|---|---|---|
| Service-to-service encryption | per-language TLS setup, per-service cert lifecycle | automatic mTLS with rotating SPIFFE identities | you now depend on the mesh CA being available |
| Retries | per-framework, inconsistent jitter, no shared budget | one policy, expressed once | the mesh's retries **compound** with the app's (§8) |
| Timeouts | scattered defaults, frequently absent | one policy | a mesh timeout that is shorter than the app's produces confusing 504s |
| Circuit breaking | rarely implemented outside Java | uniform, config-driven | Istio's shipped defaults are effectively unlimited (§9) |
| L7 metrics | per-framework, incomparable label sets | uniform RED metrics for every service | high-cardinality metrics are a real Prometheus bill |
| Traffic shifting / canary | bespoke per service | declarative, header- or weight-based | requires the mesh to be on the path, which is the cost |
| L7 authorisation | per-service middleware | one policy language | see below on defence in depth |

**The honest counter-argument, which you should be able to make:** every row above has a non-mesh answer. mTLS can come from SPIFFE/SPIRE plus a per-language library. L7 metrics can come from a shared instrumentation library. Retries and timeouts can come from a shared gRPC/HTTP client. The mesh's distinctive claim is that it delivers all of them **without touching application code and uniformly across languages** — which is worth a lot in an organisation with many languages and many teams, and worth very little in a five-service Go monorepo where one shared client library covers everything.

**The Conway's-Law version:** a mesh is a way to enforce a networking policy across an organisational boundary you cannot otherwise enforce across. If one team owns all the code, you do not have that boundary, and the mesh is solving a problem you do not have.

### 2. The sidecar datapath, hop by hop

```
   ONE REQUEST, SIDECAR MODE.  client pod → server pod, different nodes.
   Istio's port assignments are from its own documented port table.
   ══════════════════════════════════════════════════════════════════════════

   ┌──────────── CLIENT POD (one network namespace, two containers) ───────────┐
   │                                                                           │
   │  app  ──connect() to 10.96.140.22:80 (the ClusterIP)──┐                    │
   │                                                        │                   │
   │   iptables in the POD's netns (installed by istio-init  │                  │
   │   or the Istio CNI plugin) REDIRECTs all outbound TCP   │                  │
   │   from non-1337 UIDs to port 15001:                     │                  │
   │      -A ISTIO_OUTPUT -m owner --uid-owner 1337 -j RETURN  ← the proxy's   │
   │      -A ISTIO_OUTPUT -j ISTIO_REDIRECT                      own traffic   │
   │      -A ISTIO_REDIRECT -p tcp -j REDIRECT --to-port 15001   escapes here  │
   │                                                        ▼                   │
   │  ┌──────────────────────── envoy (istio-proxy) ─────────────────────────┐ │
   │  │  :15001  outbound listener  ── PROXY HOP 1 ──                        │ │
   │  │    • picks the route from RDS by :authority + path                   │ │
   │  │    • picks an endpoint from EDS (this is where the mesh's LB lives,  │ │
   │  │      NOT kube-proxy's — the sidecar sees individual pod IPs)         │ │
   │  │    • applies retry policy, timeout, circuit breaker, outlier ejection│ │
   │  │    • originates mTLS with the pod's SPIFFE identity                  │ │
   │  │    • records metrics on :15090                                       │ │
   │  └────────────────────────────────┬─────────────────────────────────────┘ │
   └───────────────────────────────────┼───────────────────────────────────────┘
                                       │  mTLS, direct to the SERVER POD IP
                                       │  (the VIP was already resolved above)
                                       ▼
   ┌──────────── SERVER POD ───────────────────────────────────────────────────┐
   │   iptables REDIRECTs inbound to 15006:                                    │
   │      -A ISTIO_INBOUND -p tcp --dport <svc ports> -j ISTIO_IN_REDIRECT      │
   │      -A ISTIO_IN_REDIRECT -p tcp -j REDIRECT --to-port 15006              │
   │                                       │                                   │
   │  ┌──────────────────────── envoy ─────▼─────────────────────────────────┐ │
   │  │  :15006  inbound listener  ── PROXY HOP 2 ──                         │ │
   │  │    • terminates mTLS, extracts the peer's SPIFFE identity            │ │
   │  │    • enforces AuthorizationPolicy (L7: method, path, headers, JWT)   │ │
   │  │    • records metrics                                                 │ │
   │  └────────────────────────────────┬─────────────────────────────────────┘ │
   │                                    ▼                                      │
   │  app :8080                                                                │
   └───────────────────────────────────────────────────────────────────────────┘

   PROXY HOPS PER REQUEST:  2  (and 2 more on the response path, same proxies)
   EXTRA TCP CONNECTIONS:   app→sidecar (loopback), sidecar→sidecar (network),
                            sidecar→app (loopback).  One logical request now
                            traverses three connections instead of one.
   RESOURCE COST:           one Envoy PER POD.

   OTHER PORTS IN THE POD, and why each exists:
     15000  Envoy admin — `curl localhost:15000/config_dump` is your single
            most valuable debugging tool; also /clusters, /stats, /listeners
     15020  merged Prometheus scrape (agent + Envoy + app)
     15021  health checks — SEPARATE from 15020 so that a mesh health probe
            never depends on the metrics pipeline
     15053  DNS, when DNS capture is enabled
     15090  Envoy's own Prometheus telemetry
     15008  HBONE — present in ambient mode (§3)
```

**Three consequences of that picture worth stating explicitly.**

1. **The mesh does the load balancing, not kube-proxy.** Once the sidecar is on the path, it receives the full endpoint list via EDS and picks a pod itself. The ClusterIP's DNAT still happens for the *first* packet in some configurations, but the effective backend choice is Envoy's. This is why a mesh fixes the gRPC/HTTP-2 problem from lesson 03 §2 — per-request balancing over a multiplexed connection.
2. **UID 1337 is load-bearing.** The proxy runs as UID 1337 and the redirect rules exempt that UID, otherwise the proxy's own outbound traffic would be redirected back to itself in an infinite loop. Any container in the pod that also runs as 1337 will silently bypass the mesh.
3. **The health-check port is deliberately separate.** 15021 for health and 15020 for merged telemetry means a stuck metrics pipeline cannot fail your liveness probe. This is a small design decision that saves an entire class of outage.

### 3. The ambient datapath, and what actually changed

Ambient mode splits the mesh into two layers that you adopt independently: a **node-level L4 layer (ztunnel)** and an **optional per-service L7 layer (waypoint)**.

```
   ONE REQUEST, AMBIENT MODE.  Same client and server, both "in mesh."
   ══════════════════════════════════════════════════════════════════════════

   ── (a) L4 ONLY: mTLS + L4 authorization + TCP telemetry ─────────────────

   NODE W1                                         NODE W2
   ┌──────────────┐                                ┌──────────────┐
   │ client pod   │                                │ server pod   │
   │  app         │                                │  app :8080   │
   │   │          │                                │      ▲       │
   │   │ transparent redirect into the node-local   │      │       │
   │   │ ztunnel (via the Istio CNI plugin, in the  │      │       │
   │   ▼ POD's netns — no sidecar container)        │      │       │
   └───┼──────────┘                                └──────┼───────┘
       │                                                  │
   ┌───▼──────────┐        HBONE tunnel:              ┌────┴─────────┐
   │   ztunnel    │  HTTP/2 CONNECT over mTLS,        │   ztunnel    │
   │  (DaemonSet) │  port 15008, multiplexing many    │  (DaemonSet) │
   │              │  app connections onto ONE         │              │
   │  PROXY HOP 1 │══════ encrypted connection ══════▶│  PROXY HOP 2 │
   └──────────────┘  per (source identity,            └──────────────┘
                      destination identity) pair

   PROXY HOPS: 2, but they are NODE-level, so ONE ztunnel serves every pod
   on the node instead of one Envoy per pod.
   NOTE (from Istio's own docs): although drawn as ztunnel-to-ztunnel, the
   HBONE tunnel is logically between the SOURCE and DESTINATION PODS —
   ztunnel does the encapsulation from inside each pod's network namespace.

   ── (b) L4 + WAYPOINT: add L7 routing, retries, L7 authorization ─────────

   client ──▶ ztunnel(W1) ══HBONE══▶ WAYPOINT ══HBONE══▶ ztunnel(W2) ──▶ server
              PROXY HOP 1            PROXY HOP 2          PROXY HOP 3

   PROXY HOPS: 3.  The waypoint is a full Envoy, deployed PER SERVICE (or
   per namespace), and it may be on neither the source nor the destination
   node — so enabling L7 can add a NETWORK HOP, not just a proxy hop.

   ── THE COST MODEL THAT FALLS OUT ────────────────────────────────────────
     sidecar:      1 Envoy × (number of pods)
     ambient L4:   1 ztunnel × (number of nodes)
     ambient L4+L7: 1 ztunnel × nodes  +  1 waypoint × (services needing L7)

     At 2,000 pods on 100 nodes:
       sidecar    → 2,000 proxies
       ambient L4 → 100 proxies
       ambient L4 + waypoints for the 20 services that need L7
                  → 100 + ~20–60 proxies (waypoints are usually replicated)
```

**What HBONE actually is**, since it is the load-bearing new mechanism: **HTTP/2 + HTTP CONNECT + mTLS**, on TCP port 15008. `CONNECT` establishes the tunnel, mTLS encrypts and mutually authenticates it, and HTTP/2 multiplexes many application TCP connections as streams inside it. Because mTLS requires unique source and destination identities per connection, **each (source identity, destination identity) pair gets its own tunnel**, and all application connections between that pair share it. Two properties matter operationally: the application byte stream is proxied unaltered — no Istio headers are injected into application traffic — and ztunnel must therefore hold **a separate x509 certificate for every service account running on its node**.

**The security property of that certificate model is the part to understand.** ztunnel authenticates to the CA with *its own* identity but requests certificates for *other* workloads' identities, and the CA enforces (via the Kubernetes ServiceAccount JWT, which encodes pod placement) that a ztunnel may only obtain identities for pods **actually running on its node**. That is what stops a compromised node from minting identities for the whole mesh. If you integrate an alternative CA, it must implement the same check — this is stated as a hard requirement in Istio's own architecture documentation, and it is the most important thing to verify in any custom-CA integration.

**Two ambient caveats from the same documentation, worth knowing before you plan a migration:** a workload *outside* the mesh sends directly to the destination and **bypasses the waypoint entirely**, even when the destination has one — so waypoint-enforced L7 policy is not a boundary against out-of-mesh clients; and traffic from sidecars and from gateways currently does not traverse waypoints either, which matters during a mixed sidecar/ambient migration.

### 4. The real numbers, and the number everyone quotes

Istio publishes benchmark results per release. Reading them across versions is more instructive than any single figure, because it shows how stale the folklore is.

| Release | Data-plane latency added (p90 / p99), two proxies, over baseline | Proxy CPU | Proxy memory |
|---|---|---|---|
| **1.10** | **2.65 ms / 2.91 ms** (1.7 ms / 2.69 ms with `jitter` enabled) | ~0.35 vCPU per 1000 rps (summary); ~0.5 vCPU per 1000 rps (detail) | ~40 MB |
| **1.13** | **1.7 ms / 2.7 ms** | ~0.5 vCPU per 1000 rps | ~40 MB |
| **1.20** | **0.228 ms / 0.298 ms** | ~0.5 vCPU per 1000 rps | ~50 MB with namespace isolation |
| **1.24** | published as charts only (see note) | **sidecar 0.20 vCPU**, **waypoint 0.25 vCPU**, **ztunnel 0.06 vCPU** — all at 1000 rps, 1 KB payload, 2 worker threads | sidecar 60 MB, waypoint 60 MB, **ztunnel 12 MB** |

**Read three things off that table.**

1. **"A mesh adds about 2.7 ms" is a 1.10-era number and is roughly 10× too high for a current release.** Istio 1.20's own documentation puts the two-proxy p90 addition at **0.228 ms**. If you are arguing about mesh adoption using the 2.65 ms figure, you are arguing with data that is several years old, and the person on the other side of the table may know that.
2. **ztunnel is dramatically cheaper than a sidecar** — 0.06 vCPU against 0.20, and 12 MB against 60 MB, at the same load. That ratio, multiplied by the pods-versus-nodes count from §3, is the entire economic case for ambient.
3. **A waypoint costs slightly *more* than a sidecar per proxy** (0.25 vs 0.20 vCPU). Ambient's saving comes from *how many* proxies you run, not from each one being cheaper. If every service needs L7, ambient's advantage narrows sharply.

**The methodology, because a number without it is not usable.** Istio's 1.24 figures come from a bare-metal cluster of 5 Equinix M3 Large machines on the CNCF Community Infrastructure Lab, using Flannel as the primary CNI, `http/1.1`, 1 KB payload, 500–1500 rps, 4 client connections, 2 proxy workers, mTLS enabled, measured with `fortio`. The 1.20 figures used the same benchmark suite at 1000 rps with 2–64 client connections. **Different hardware gives different values** — Istio says so explicitly — so the reproducible move is to run `istio/tools/perf/benchmark` on *your* hardware rather than quoting anyone's table, including this one.

**One methodological subtlety Istio documents and most readers miss.** Envoy collects raw telemetry **after** the response has been sent, so that collection time does not appear in *that* request's latency. But the worker is busy, so it delays the *next* request. **Telemetry cost therefore shows up as queue wait in tail latency, not as service time in mean latency** — which is exactly why mesh overhead is worse at p99 than at p50, and why the effect grows with load.

**A note on the 1.24 latency figures:** they are published as PNG charts rather than as text. Those images were not readable in this pass, so the p90/p99 numbers for 1.24 are deliberately **not** quoted here. The 1.10, 1.13, and 1.20 figures above are quoted from the text of those releases' documentation.

### 5. Worked: the mesh budget at a stated request rate

Take a real fleet and produce the CPU, memory, and latency bill.

```
   FLEET:  600 pods across 40 nodes.
           Fleet-wide request rate: 30,000 rps (all mesh-internal).
           Mean payload 1 KB.  Mean fan-out: each external request produces
           3 internal calls, so 30,000 rps is 10,000 user requests/s.

   ── SIDECAR MODE ─────────────────────────────────────────────────────────
   Per-proxy CPU (Istio 1.24, 2 workers, 1 KB): 0.20 vCPU per 1000 rps.
   Every request traverses TWO sidecars, so the fleet's proxies collectively
   process 2 × 30,000 = 60,000 proxy-rps.

     CPU  = 60,000 / 1000 × 0.20 vCPU  =  12.0 vCPU across the fleet
     MEM  = 600 pods × 60 MB           =  36.0 GB across the fleet

   Against a fleet of 40 nodes × 32 vCPU = 1,280 vCPU:
     12.0 / 1,280 = 0.94 % of fleet CPU.
   Against 40 nodes × 128 GB = 5,120 GB:
     36 / 5,120 = 0.70 % of fleet memory.

   ── AMBIENT L4 ONLY ──────────────────────────────────────────────────────
   ztunnel: 0.06 vCPU per 1000 rps, 12 MB per proxy.
     CPU  = 60,000 / 1000 × 0.06         =  3.6 vCPU   (−70 %)
     MEM  = 40 nodes × 12 MB             =  0.48 GB    (−99 %)

   ── AMBIENT L4 + WAYPOINTS FOR 15 SERVICES ───────────────────────────────
   Suppose those 15 services carry 40 % of the traffic = 12,000 rps,
   each traversing ONE waypoint (in addition to two ztunnel hops):
     ztunnel CPU  = 3.6 vCPU (unchanged)
     waypoint CPU = 12,000 / 1000 × 0.25 =  3.0 vCPU
     TOTAL        =  6.6 vCPU                        (−45 % vs sidecar)
     waypoint MEM = 15 services × 2 replicas × 60 MB = 1.8 GB
     TOTAL MEM    = 0.48 + 1.8 = 2.28 GB             (−94 % vs sidecar)

   ── LATENCY ──────────────────────────────────────────────────────────────
   Using Istio 1.20's published two-proxy figures as the best TEXTUAL data
   point available (0.228 ms p90 / 0.298 ms p99 added over baseline):

     per user request = 3 internal calls × 0.228 ms = 0.68 ms added at p90
                                          × 0.298 ms = 0.89 ms added at p99

   Is that acceptable? Put it against the SLO, not against zero:
     if the user-facing p99 SLO is 200 ms → 0.89 ms is 0.45 % of budget. Fine.
     if the SLO is 10 ms (an ad-bidding or inference-router path)
        → 0.89 ms is 8.9 % of budget. NOT fine — and this is exactly where
          "L4 ambient only, no waypoint" or "no mesh on this path" is the
          right answer.

   ── THE NUMBER THAT USUALLY DOMINATES, AND ISN'T CPU ─────────────────────
   Metrics cardinality. Istio's standard HTTP metrics carry source and
   destination workload, namespace, service, version, response code, and
   response flags. For a 200-service mesh where most pairs talk:
     conservative: 200 services × 8 peers each × 12 label combinations
                   × ~6 metric names ≈ 115,000 active series
     pessimistic (full mesh):
                   200 × 200 × 12 × 6 ≈ 2,880,000 active series
   At a rule-of-thumb few KB of Prometheus RAM per active series, the
   pessimistic case is GIGABYTES of monitoring memory that did not exist
   before the mesh. BUDGET FOR IT, and drop labels you do not query.
```

### 6. Envoy's config model, and why it decides your memory bill

An Envoy proxy is configured entirely by **xDS**, a set of gRPC streams from the control plane. The resource types you need to be able to name:

| xDS | Resource | What it is | Rough count in a mesh |
|---|---|---|---|
| **LDS** | Listener | a socket to accept on, plus a filter chain | one per outbound port + inbound ports |
| **RDS** | RouteConfiguration | virtual hosts and their match rules → cluster | one per HTTP listener; grows with VirtualServices |
| **CDS** | Cluster | an upstream group + its LB policy, circuit breakers, TLS context | **one per Service (per subset)** |
| **EDS** | ClusterLoadAssignment | the actual endpoint IPs for a cluster | one per cluster; churns with every pod change |
| **SDS** | Secret | certificates and keys | per identity |

**The memory relationship, stated plainly:** by default every sidecar receives configuration for **every Service in the mesh**, because the control plane cannot know which ones this pod will call. In a 1,000-service mesh that is 1,000 clusters, their routes, and all their endpoints — **in every one of your 2,000 pods**. This is why Istio's own documentation says proxy memory "depends on the total configuration state the proxy holds" and why it quotes ~50 MB *with namespace isolation enabled*.

**The fix is scoping, and it is the single highest-leverage mesh tuning available.** A `Sidecar` resource tells the control plane which destinations this workload actually needs:

```yaml
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: default
  namespace: payments
spec:
  # Applies to every workload in this namespace that has no more specific Sidecar.
  egress:
    - hosts:
        - "./*"                       # everything in MY namespace
        - "istio-system/*"            # the control plane
        - "kube-system/kube-dns.kube-system.svc.cluster.local"
        - "ledger/*"                  # the one other namespace we call
  # Anything not listed is NOT pushed to these proxies at all.
```

The effect is compound: less memory per proxy, **and** a smaller xDS push when anything changes, **and** a smaller CPU spike on the control plane, **and** fewer proxies that need updating when an unrelated service scales. On a large mesh this is the difference between an Istiod that idles and one that is permanently pegged.

**The control-plane scaling shape.** Istiod's CPU scales with the *rate of deployment changes*, the *rate of configuration changes*, and the *number of proxies connected* — Istio documents exactly these three factors. Two of those three are things your platform's deploy cadence controls, which means **a mesh makes your control plane sensitive to how often you deploy**. It is horizontally scalable (add Istiod replicas to reduce convergence time), but the total work is not reduced by adding replicas; only the wall-clock convergence is.

### 7. Reading a live proxy: the commands that matter

```bash
# 1. Is this pod's config in sync with the control plane?
$ istioctl proxy-status
NAME                          CLUSTER   CDS      LDS      EDS      RDS      ISTIOD
api-7d9f-x2m4.payments        Kubernetes SYNCED  SYNCED   SYNCED   SYNCED   istiod-6c9-abc
ledger-5b1-k9p2.ledger        Kubernetes SYNCED  SYNCED   STALE    SYNCED   istiod-6c9-abc
#                                                          ^^^^^ EDS STALE:
#  this proxy has an OUT-OF-DATE endpoint list. It is sending traffic to
#  pods that may no longer exist. STALE is the single most useful mesh
#  signal there is, and it is the first thing to check in any mesh incident.

# 2. What does the proxy think the upstream looks like?
$ istioctl proxy-config endpoints api-7d9f-x2m4.payments --cluster \
    "outbound|80||ledger.ledger.svc.cluster.local"
ENDPOINT             STATUS      OUTLIER CHECK     CLUSTER
10.244.2.31:8080     HEALTHY     OK                outbound|80||ledger...
10.244.7.19:8080     UNHEALTHY   FAILED            outbound|80||ledger...
#                    ^^^^^^^^^   ^^^^^^ ejected by OUTLIER DETECTION, not by
#                                a health check. Lesson 03 §6 is the mechanism.

# 3. What route matched, and what policy applied?
$ istioctl proxy-config route api-7d9f-x2m4.payments --name 80 -o json | \
    jq '.[0].virtualHosts[] | select(.name|startswith("ledger")) | .routes[0]'
{
  "match": { "prefix": "/" },
  "route": {
    "cluster": "outbound|80|v2|ledger.ledger.svc.cluster.local",
    "timeout": "0s",                       ← NO TIMEOUT. Istio's default.
    "retryPolicy": {
      "retryOn": "connect-failure,refused-stream,unavailable,cancelled,retriable-status-codes",
      "numRetries": 2,                     ← the mesh's DEFAULT retries
      "retryHostPredicate": [{ "name": "envoy.retry_host_predicates.previous_hosts" }],
      "hostSelectionRetryMaxAttempts": "5"
    }
  }
}

# 4. The raw truth, when the abstractions disagree:
$ kubectl exec api-7d9f-x2m4 -c istio-proxy -- curl -s localhost:15000/config_dump | \
    jq '.configs[] | select(."@type"|test("ClustersConfigDump")) | .dynamic_active_clusters | length'
1043
#  1,043 clusters in ONE sidecar → no Sidecar scoping is in place (§6).

# 5. Live traffic, when you need to see it happen:
$ kubectl logs api-7d9f-x2m4 -c istio-proxy --tail=5
[2026-08-18T09:14:22.318Z] "POST /v1/charge HTTP/1.1" 503 UO
  upstream_reset_before_response_started{overflow} - "-" 412 0 0 -
#   RESPONSE FLAG "UO" = upstream overflow: a CIRCUIT BREAKER tripped.
#   Not a network failure. Not the app. A connection-pool limit (§9).
```

**The response-flag vocabulary is worth learning cold**, because it converts a 503 into a diagnosis: `UO` upstream overflow (circuit breaker), `UF` upstream connection failure, `UH` no healthy upstream, `URX` retry limit exceeded, `NR` no route configured, `DC` downstream connection termination, `UT` upstream request timeout, `RL` rate limited.

### 8. Retries: where the mesh amplifies

Lesson 03 §9 established that retries compose multiplicatively. A mesh makes that worse in a specific, avoidable way: **it adds a retry layer that the application does not know about.**

Istio's `HTTPRetry` is deliberately simple — `attempts`, `perTryTimeout`, `retryOn` — and its own API documentation states the arithmetic: *"The maximum possible number of requests made will be 1 + `attempts`."* The interval between retries is chosen automatically (25 ms and up). Envoy underneath applies full jitter to that backoff, which is the good news.

```
   THE COMPOUNDING, MADE CONCRETE
   ═══════════════════════════════════════════════════════════════════
     application HTTP client:   3 attempts   (a framework default nobody set)
     client sidecar:            3 attempts   (Istio VirtualService)
     server sidecar:            1 attempt    (inbound proxies don't retry)
     downstream service's own client: 3 attempts

     One user request against a briefly failing dependency:
        3 × 3 × 3 = 27 requests at the bottom of the chain.

     At 10,000 user rps and a dependency that starts failing:
        270,000 rps offered to a service sized for 10,000.
     This is not a thundering herd — it is a permanent 27× multiplier
     that persists for as long as the failure does.

   THE FIXES, IN ORDER
   ═══════════════════════════════════════════════════════════════════
   1. RETRY IN EXACTLY ONE PLACE. Usually the mesh, because it is the
      layer you can change without a deploy. Turn the app's retries OFF.

   2. NEVER RETRY 503 FROM AN OVERLOADED BACKEND. Istio's default
      `retryOn` includes `unavailable`, which for gRPC covers overload.
      A backend returning 503 because it is saturated is asking you to
      send LESS, and retrying is sending more. Restrict retryOn to
      `connect-failure,refused-stream,reset` for anything that can be
      overloaded — which is everything with a GPU behind it.

   3. BOUND IT WITH A BUDGET, NOT A COUNT. Envoy's RetryBudget caps
      retries at a percentage of ACTIVE requests to a cluster, so total
      offered load is bounded at (1 + budget) regardless of how many
      clients misbehave. Istio's VirtualService API does not expose it,
      so it goes through an EnvoyFilter — one of the few cases where an
      EnvoyFilter is genuinely justified.

   4. IDEMPOTENCY. A retry of a non-idempotent request is a correctness
      bug wearing a reliability costume. Require an idempotency key, or
      set `attempts: 0` on those routes.
```

```yaml
# A defensible retry policy for a route in front of GPU inference.
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: inference
  namespace: serving
spec:
  hosts: ["inference.serving.svc.cluster.local"]
  http:
    - route:
        - destination:
            host: inference.serving.svc.cluster.local
      timeout: 120s              # generation is slow; a mesh timeout SHORTER
                                 # than the app's produces confusing 504s that
                                 # look like the backend died
      retries:
        attempts: 2              # → at most 3 total requests (1 + attempts)
        perTryTimeout: 60s
        retryOn: connect-failure,refused-stream,reset
        #        ^^^ deliberately EXCLUDES 5xx and `unavailable`:
        #        a saturated inference replica must not be retried
```

### 9. Circuit breaking: Istio's defaults are effectively "off"

This is the most consequential thing in the lesson that people do not know. Read Istio's `ConnectionPoolSettings` API defaults from its own proto:

| Field | Istio default | What it means |
|---|---|---|
| `http.http1MaxPendingRequests` | 2³² − 1 | unlimited queueing of pending requests |
| `http.http2MaxRequests` | **2³² − 1** | unlimited concurrent requests to a destination |
| `http.maxRequestsPerConnection` | **0** = unlimited (up to 2²⁹) | connections are never cycled |
| `http.maxRetries` | **2³² − 1** | unlimited concurrent retries across all hosts |
| `http.idleTimeout` | **1 hour** if unset | |
| `tcp.maxConnectionDuration` | unset = no maximum | |

**Installing Istio therefore gives you circuit-breaking *capability* and no circuit-breaking *behaviour*.** A team that believes "the mesh protects us" because it is installed has, by default, unlimited concurrency, unlimited pending queue, and unlimited retries. The `UO` response flag from §7 only appears once someone has actually set a limit.

A `DestinationRule` that does something:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: inference
  namespace: serving
spec:
  host: inference.serving.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 200          # per sidecar, per destination
        connectTimeout: 2s
        tcpKeepalive: { time: 60s, interval: 10s, probes: 3 }
      http:
        http2MaxRequests: 128        # ← THE ONE THAT MATTERS for gRPC/HTTP2.
                                     #   Set it from the backend's real
                                     #   concurrency, not from a round number:
                                     #   for a vLLM replica with max_num_seqs
                                     #   = 64, allowing 4x that in flight just
                                     #   builds queue and inflates p99.
        http1MaxPendingRequests: 64  # fail fast rather than queue forever
        maxRequestsPerConnection: 1000  # cycle connections so EDS changes
                                        # actually take effect (lesson 03 §2)
        idleTimeout: 60s
    outlierDetection:
      consecutive5xxErrors: 5        # Envoy default (lesson 03 §6)
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 20         # never eject more than 1 in 5
      minHealthPercent: 50           # Istio's panic-threshold equivalent
    loadBalancer:
      simple: LEAST_REQUEST          # power-of-two choices (lesson 03 §3)
      warmupDurationSecs: 90         # slow start — size from the measured
                                     # model warm-up, not a round number
```

**`maxRequestsPerConnection` deserves a sentence of its own.** With the default of unlimited, a long-lived HTTP/2 connection from a sidecar to a backend pod is never recycled, so an endpoint-list change does not redistribute existing traffic — exactly the gRPC stickiness problem from lesson 03 §2, reappearing one layer up. Setting it forces periodic reconnection and therefore periodic rebalancing.

### 10. The rollout is itself a distributed-systems project

A mesh migration has failure modes that have nothing to do with the mesh's steady-state behaviour.

**Injection races.** Sidecar injection is a mutating admission webhook. Three things follow: if the webhook is unavailable, pod creation either fails or silently proceeds *without* a sidecar depending on the webhook's `failurePolicy`; a pod created before the namespace was labelled has no sidecar and will not get one without a restart; and during a rollout you will have a mesh where **some pods are meshed and some are not**, which is why `PeerAuthentication` mode `PERMISSIVE` (accept both mTLS and plaintext) exists and why moving to `STRICT` is a separate, later, deliberate step.

**Ordering, in the pod.** The application container can start before the sidecar is ready and get connection refused on its first outbound calls. Modern Istio uses a native **sidecar container** (Kubernetes init containers with `restartPolicy: Always`), which fixes the startup ordering properly. On older setups you need `holdApplicationUntilProxyStarts`. The shutdown side is the mirror image: the sidecar must outlive the application's in-flight requests, or you get RSTs at every deploy.

**Version skew.** The control plane and data plane are separate binaries with an independent upgrade cadence, and a mesh where Istiod is version N and half the sidecars are N−2 is normal during a canary upgrade. `istioctl proxy-status` shows the version each proxy is running; the supported skew window is a documented and finite number of minor releases, and exceeding it is how you get config that Istiod emits and the proxy silently ignores.

**xDS push storms.** Every configuration change — a Service, an EndpointSlice, a VirtualService, a certificate rotation — triggers a computation in Istiod and a push to affected proxies. Without scoping (§6), "affected proxies" is *all of them*. A 2,000-pod mesh where a large Deployment rolls generates thousands of endpoint updates, each fanning out to thousands of proxies. Watch `pilot_xds_pushes`, `pilot_proxy_convergence_time`, and the `STALE` count in `proxy-status`; the failure signature is convergence time climbing into tens of seconds, after which the data plane is acting on a stale view of the world.

**Certificate expiry as an outage class.** mTLS certificates rotate on a schedule. If the CA is unavailable when a certificate expires, that workload stops being able to talk to anything. This is a dependency you did not have before, and it belongs in your failure analysis with the same weight as etcd.

### 11. When not to use one — and the GPU rule

The honest checklist. Do **not** add a mesh when:

- **Fewer than ~20 services, one or two languages, one team.** A shared client library delivers 80 % of the value at 5 % of the operational cost. The mesh solves an *organisational* problem you do not have.
- **The latency budget is single-digit milliseconds.** §5's arithmetic: 0.9 ms of added p99 against a 10 ms SLO is 9 % of the budget spent on infrastructure. Consider ambient L4 only, or nothing.
- **You do not have the operational capacity.** A mesh is a distributed system with a control plane, a CA, an upgrade cadence, and its own on-call surface. If nobody owns it, it will be at version N−6 when you need a CVE patch.
- **The workload is not HTTP/gRPC.** L7 features are the mesh's differentiator; on opaque TCP you get mTLS and L4 telemetry, which ambient L4 mode delivers far more cheaply than sidecars.
- **On the RDMA data path. Ever.** This is not a cost/benefit judgement; it is a category error.

**Why the RDMA rule is absolute**, in terms you can now state precisely from lessons 05 and 07 and module 09:

1. **RDMA bypasses the kernel network stack**, so there is no socket, no `connect()`, and no packet traversing netfilter for a proxy to intercept. The interception mechanisms in §2 and §3 — iptables REDIRECT, cgroup eBPF hooks — have nothing to attach to.
2. **A proxy would reintroduce every cost RDMA exists to delete**: copies into and out of user space, per-packet CPU, and the kernel scheduler on the data path.
3. **The latency scale is wrong by three orders of magnitude.** An RDMA half-round-trip is ~1–2 µs. A proxy hop is hundreds of microseconds at best. Adding a mesh hop to a collective would multiply its latency by roughly 100×, and because a collective is a barrier, every rank pays it.
4. **The threat model is different.** The RDMA fabric is a dedicated, physically separate network reachable only by pods that were granted a VF or a DRA claim. Its isolation comes from the fabric and the device allocation, not from an L7 proxy.

**The correct GPU-fleet placement, stated as a rule:** mesh the *inference frontend* (HTTP/gRPC, external-facing, multi-tenant, needs mTLS and L7 policy and rate limiting); leave the *training path* entirely unmeshed; and if your CNI's mesh integration operates at the node level (ambient), explicitly exclude the RDMA interfaces and the training namespaces from it with a label, rather than relying on it not noticing them.

## Perspectives

**Developer.** A mesh is invisible until it isn't. The two moments it becomes visible are a 503 with a response flag the developer has never seen (`UO`, `URX`, `NR`) and a timeout that comes from the mesh rather than the app. The highest-value thing a platform team can ship alongside a mesh is a one-page translation of those flags into causes, because otherwise every mesh incident starts with an hour of confusion about whose timeout fired.

**Operator.** The mesh's control plane is now a tier-0 dependency with a CA attached, and its cost scales with your *deploy rate*, not just your traffic. Scoping (`Sidecar` resources) is the highest-leverage lever you have and is usually missing. The signal to watch is not CPU; it is `pilot_proxy_convergence_time` and the count of `STALE` proxies, because a mesh whose data plane is minutes behind the control plane is worse than no mesh — it is a mesh confidently routing to pods that are gone.

**Economics.** The CPU tax is smaller than people fear (§5: ~1 % of fleet CPU for sidecars, ~0.3 % for ambient L4) and the *metrics* tax is larger than people expect (potentially millions of active Prometheus series). The correct budget line for a mesh has three rows — proxy CPU, proxy memory, and monitoring cardinality — and the third is usually the biggest. Ambient's saving is not "a cheaper proxy," it is "two orders of magnitude fewer proxies."

**Security.** mTLS everywhere is real value, and ambient's certificate model is the part worth scrutinising: ztunnel holds a certificate for every service account on its node, obtained by authenticating as itself and requesting another identity, with the CA enforcing (via the ServiceAccount JWT) that the identity actually runs on that node. That check is what bounds a node compromise to that node's workloads. Any alternative CA integration must implement it; if it does not, you have widened your blast radius while believing you narrowed it. And remember L3/L4 NetworkPolicy (lesson 05) and L7 mesh authorisation are defence in depth, not substitutes — the mesh policy is only enforced for traffic that actually reaches a proxy, and §3 documents that out-of-mesh clients bypass waypoints entirely.

## Real-world use cases

- **Istio's own performance documentation across five releases** (`istio/istio.io`, `content/en/docs/ops/deployment/performance-and-scalability/index.md` on `release-1.10`, `release-1.13`, `release-1.20`, and `master`), read directly. What it shows: the widely-quoted "~2.7 ms of mesh latency" is the **Istio 1.10** figure (2.65 ms p90 / 2.91 ms p99, dropping to 1.7/2.69 with `jitter`), and by **1.20** the same benchmark reports **0.228 ms p90 / 0.298 ms p99** — roughly a 10× improvement. The 1.24 release adds per-proxy-type resource figures: sidecar 0.20 vCPU / 60 MB, waypoint 0.25 vCPU / 60 MB, ztunnel **0.06 vCPU / 12 MB**, all at 1000 rps with 1 KB payloads and 2 worker threads. Reading a benchmark's *history* rather than its latest number is the staff move here, because it tells you how stale everyone else's mental model is.
- **Istio's ambient data-plane architecture documentation** (`istio/istio.io`, `content/en/docs/ambient/architecture/data-plane/index.md` and `.../hbone/index.md`), read directly. Substance used here: HBONE is HTTP/2 + HTTP CONNECT + mTLS on port **15008**, multiplexing many application connections per (source identity, destination identity) pair; the tunnel is logically between the *pods*, with ztunnel encapsulating from inside each pod's network namespace; ztunnel holds a distinct x509 certificate **per service account on its node**, authenticating to the CA as itself while requesting another workload's identity, with the CA enforcing via the ServiceAccount JWT that the identity is node-local — "critical to ensure that a compromised node does not compromise the entire mesh"; and that out-of-mesh clients, sidecars, and gateways currently **bypass waypoints**.
- **Istio's `ConnectionPoolSettings` API defaults** (`istio/api`, `networking/v1alpha3/destination_rule.proto`), read directly. What it shows: `http2MaxRequests` defaults to 2³²−1, `maxRetries` to 2³²−1, `maxRequestsPerConnection` to 0 (unlimited up to 2²⁹), and the connection idle timeout to 1 hour. **A default Istio install provides no circuit breaking at all.** This is the gap between "we have a mesh, so we have circuit breakers" and reality, and it is the kind of thing that only shows up in an API proto.
- **Istio's `HTTPRetry` API** (`istio/api`, `networking/v1alpha3/virtual_service.proto`), read directly. Substance: "the maximum possible number of requests made will be 1 + `attempts`," the automatic retry interval starting at 25 ms, and the interaction between `attempts`, route `timeout`, and `perTryTimeout` that means the *actual* number of retries can be lower than configured. This is the arithmetic underneath the 27× compounding in §8.

## Worked example

**Scenario.** A platform team wants to mesh a 600-pod fleet that includes an inference frontend in front of GPU pods. Produce the decision: sidecar or ambient, what it costs, what to configure on day one, and what to exclude.

**Step 1 — establish the baseline, before adding anything.**

```bash
# Fleet shape
$ kubectl get pods -A --field-selector=status.phase=Running --no-headers | wc -l
612
$ kubectl get nodes --no-headers | wc -l
40
$ kubectl get svc -A --no-headers | wc -l
187

# Current p99 for the path that matters most, measured, not assumed
$ kubectl exec -n serving deploy/loadgen -- fortio load -qps 200 -t 60s -a \
    -H "Content-Type: application/json" http://inference:8000/v1/completions
Code 200 : 12000 (100.0 %)
# target 50% 0.0214  75% 0.0287  90% 0.0361  99% 0.0492  99.9% 0.0771
#                                                  ^^^^^^ p99 = 49.2 ms baseline
```

**Step 2 — decide sidecar vs ambient with the arithmetic, not the marketing.**

```
   Traffic: 30,000 mesh-internal rps.  600 pods, 40 nodes.
   How many services genuinely need L7 (routing, L7 authz, header canary)?
     $ kubectl get virtualservice -A | wc -l     → 12
     $ kubectl get authorizationpolicy -A | wc -l → 9 (of which 6 are L7)
     → about 15 services need L7. The other ~170 need mTLS + telemetry only.

   SIDECAR:      CPU 2 × 30,000/1000 × 0.20 = 12.0 vCPU
                 MEM 600 × 60 MB            = 36.0 GB
   AMBIENT L4:   CPU 2 × 30,000/1000 × 0.06 =  3.6 vCPU
                 MEM 40 × 12 MB             =  0.48 GB
   + 15 WAYPOINTS carrying ~40 % of traffic (12,000 rps):
                 CPU 12,000/1000 × 0.25     = +3.0 vCPU  → 6.6 vCPU total
                 MEM 15 × 2 × 60 MB         = +1.8 GB    → 2.28 GB total

   DECISION: ambient. It costs 45 % of the sidecar CPU and 6 % of the
   memory for this fleet's L4/L7 split, and — the operational point that
   matters more — 155 of the 187 services never get an L7 proxy on their
   path at all, so they cannot be broken by an L7 misconfiguration.

   THE CAVEAT TO STATE HONESTLY: if the L7 fraction grows to most services,
   this advantage collapses, because a waypoint costs MORE per proxy than a
   sidecar (0.25 vs 0.20 vCPU). Re-run the arithmetic annually.
```

**Step 3 — decide what is excluded, before anything is enabled.**

```yaml
# Namespaces that get L4 ambient:
#   kubectl label namespace payments ledger serving istio.io/dataplane-mode=ambient
#
# Namespaces that get waypoints (L7):
#   kubectl label namespace serving istio.io/use-waypoint=serving-waypoint
#
# Namespaces EXPLICITLY EXCLUDED — and why, written down:
#   training/        — RDMA collectives. No socket for a proxy to intercept,
#                      ~1–2 µs RTT versus a proxy's hundreds of µs, and a
#                      collective is a barrier so every rank pays. NEVER.
#   kube-system/     — bootstrapping order; a mesh that depends on DNS that
#                      depends on the mesh does not start.
#   monitoring/      — must be able to scrape a broken mesh.
```

**Step 4 — configure the things that are off by default.** The two that matter most on day one are scoping and circuit breaking.

```yaml
# (a) Scope the config. Without this every proxy holds every service.
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata: { name: default, namespace: serving }
spec:
  egress:
    - hosts: ["./*", "istio-system/*", "kube-system/kube-dns.kube-system.svc.cluster.local"]
---
# (b) Give the inference tier actual limits, sized from the backend.
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata: { name: inference, namespace: serving }
spec:
  host: inference.serving.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp: { maxConnections: 200, connectTimeout: 2s }
      http:
        http2MaxRequests: 128           # vLLM max_num_seqs = 64 per replica,
                                        # 2 replicas' worth in flight. Beyond
                                        # this, queueing at the proxy is
                                        # STRICTLY BETTER than queueing at the
                                        # GPU, because the proxy can shed.
        http1MaxPendingRequests: 64
        maxRequestsPerConnection: 1000  # so EDS changes actually rebalance
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 20
      minHealthPercent: 50
    loadBalancer:
      simple: LEAST_REQUEST
      warmupDurationSecs: 90            # measured model warm-up + margin
---
# (c) Retries that cannot amplify a GPU overload.
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata: { name: inference, namespace: serving }
spec:
  hosts: ["inference.serving.svc.cluster.local"]
  http:
    - route: [{ destination: { host: inference.serving.svc.cluster.local } }]
      timeout: 120s
      retries:
        attempts: 2
        perTryTimeout: 60s
        retryOn: connect-failure,refused-stream,reset   # NOT 5xx, NOT unavailable
```

**Step 5 — measure the delta, on the path you care about.**

```bash
# Same fortio run, now through the mesh:
$ kubectl exec -n serving deploy/loadgen -- fortio load -qps 200 -t 60s -a \
    -H "Content-Type: application/json" http://inference:8000/v1/completions
# target 50% 0.0219  75% 0.0294  90% 0.0369  99% 0.0503  99.9% 0.0812
#                                                  ^^^^^^ p99 = 50.3 ms
#
#  DELTA: p99 49.2 → 50.3 ms = +1.1 ms = +2.2 %
#  Against a 200 ms SLO that is 0.55 % of budget. ACCEPT.
#  Note this is larger than Istio's published 0.3 ms because THIS path has
#  a waypoint (3 proxy hops, not 2) and this hardware is not Istio's.
#  Publish YOUR number; do not publish Istio's.

# Verify the config actually converged:
$ istioctl proxy-status | grep -c STALE
0
# And verify circuit breaking is now real:
$ kubectl exec -n serving deploy/loadgen -- fortio load -qps 5000 -t 20s \
    http://inference:8000/v1/completions 2>&1 | grep "Code 503"
Code 503 : 41203 (68.4 %)
$ kubectl logs -n serving deploy/inference -c istio-proxy | grep -c ' 503 UO '
41203
#   UO = upstream overflow. The circuit breaker is SHEDDING at the proxy
#   instead of queueing at the GPU. That is the behaviour you paid for.
```

**Step 6 — the monitoring bill, which nobody budgets.**

```
   187 services. Estimate active series before enabling telemetry:
     istio_requests_total has ~12 labels; assume ~8 distinct peers per
     service and ~12 realistic label combinations per pair, over ~6
     standard metric names:
       187 × 8 × 12 × 6 ≈ 108,000 active series added.
   At a few KB of Prometheus memory per series that is single-digit GB —
   acceptable. But if the mesh is closer to fully connected:
       187 × 187 × 12 × 6 ≈ 2,520,000 series → tens of GB.

   ACTION: drop labels you never query (`source_version`,
   `destination_canonical_revision`) via a Telemetry resource before
   enabling, not after the Prometheus OOMs.
```

## Practice
<feeds [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)>

**Task A — measure the mesh tax yourself.** On a kind or k3s cluster, deploy a two-service app and measure p50/p90/p99 with `fortio` in four configurations: no mesh, ambient L4, ambient L4 + waypoint, and sidecar. Use a 1 KB payload at several request rates. Produce a table of *your* numbers and compare against Istio's published figures for the same release. Explain any gap in terms of hardware, CNI, and proxy worker count — this is the artifact that lets you argue about mesh adoption with data instead of folklore.

**Task B — the CPU and memory budget.** For a fleet you actually run, compute the sidecar and ambient costs using §5's method: proxy-rps = 2 × request rate (3 × with waypoints), CPU = proxy-rps/1000 × the per-proxy figure, memory = proxy count × per-proxy memory. Then add the metrics-cardinality row: enumerate the standard Istio metrics, their labels, and your service-pair count, and estimate active series. Present all three rows as a percentage of the fleet's existing budget.

**Task C — the retry-budget worksheet.** Build a spreadsheet that takes: layers of retry in your call chain, attempts per layer, and base request rate, and outputs the worst-case amplification and the resulting offered load. Then implement an Envoy `RetryBudget` via an `EnvoyFilter`, and demonstrate with a load test that offered load stays bounded at (1 + budget) even when clients retry aggressively. Record what the budget does to error rates during a partial outage.

**Task D — break it deliberately, four ways.** (1) Set `http2MaxRequests: 2` and observe `503 UO` in the access log and the corresponding Envoy stat. (2) Delete the `Sidecar` scoping resource and measure the change in `dynamic_active_clusters` and proxy RSS. (3) Kill Istiod and observe that the data plane keeps working while `proxy-status` goes `STALE` — then scale a Deployment and show that traffic goes to endpoints that no longer exist. (4) Set a mesh `timeout` shorter than the application's and show the resulting `504 UT`, which developers will otherwise misdiagnose as a backend failure.

**Task E — the decision checklist.** Write a one-page document that a team can use to answer "should we mesh this?" It must include: service count and language count thresholds, the latency-budget calculation with the SLO in the denominator, the L7-fraction test that decides sidecar vs ambient, the operational-ownership question, and the explicit exclusion list (RDMA/training, kube-system, monitoring) with the mechanism-level reason for each — not "because it's fast," but "because there is no socket for the proxy to intercept."

**Acceptance criteria**

- [ ] A four-configuration latency table measured on your own hardware, with the deltas and a comparison to Istio's published figures for that release.
- [ ] A three-row mesh budget (proxy CPU, proxy memory, metrics cardinality) as a percentage of the existing fleet budget, for both sidecar and ambient.
- [ ] A retry-amplification worksheet plus a demonstrated retry budget bounding offered load under a load test.
- [ ] Four deliberate breakages reproduced, each with the exact log line or stat that identifies it.
- [ ] A mesh decision checklist with the RDMA exclusion justified at the mechanism level.

## Common pitfalls
- **Quoting "a mesh adds 2.7 ms."** That is Istio **1.10**'s published figure. Istio **1.20**'s own documentation reports **0.228 ms p90 / 0.298 ms p99** for the same two-proxy benchmark. Using the old number either loses you an argument you should win, or wins you one you should lose.
- **Assuming a default Istio install gives you circuit breaking.** `http2MaxRequests`, `maxRetries`, and `http1MaxPendingRequests` all default to 2³²−1, and `maxRequestsPerConnection` to unlimited. Until you write a `DestinationRule`, there are no limits — only the capability to set them.
- **Leaving `retryOn` at defaults in front of a saturated backend.** Istio's defaults include `unavailable`, which for gRPC covers overload. Retrying an overloaded GPU replica adds load to the thing that is out of capacity. Restrict to `connect-failure,refused-stream,reset` on anything that can saturate.
- **Retrying in the mesh *and* in the application.** Three layers at three attempts each is 27× amplification at the bottom of the call chain, and it persists for the entire duration of the failure. Retry in exactly one place, and prefer a budget over a count.
- **Skipping `Sidecar` scoping.** By default every proxy receives configuration for every service in the mesh. On a 1,000-service mesh that is 1,000 clusters in every one of your pods, plus a full-fleet xDS push on every change. Check `dynamic_active_clusters` in a `config_dump` before assuming you are fine.
- **Not watching `proxy-status`.** A `STALE` EDS entry means that proxy is routing to an endpoint list that no longer reflects reality. It is the single most useful mesh signal and it is not on most dashboards.
- **Setting a mesh timeout shorter than the application's.** The mesh cuts the request and returns `504 UT`; the developer sees a timeout that their own instrumentation did not produce and starts debugging the backend. For long-running inference, the mesh timeout must exceed the longest legitimate generation.
- **Believing waypoint L7 policy is a security boundary against everything.** Istio documents that out-of-mesh workloads send directly to the destination and bypass the waypoint, as do sidecars and gateways in current releases. Waypoint policy is defence in depth on top of L3/L4 NetworkPolicy (lesson 05), not a replacement for it.
- **Running an application container as UID 1337.** The sidecar's iptables rules exempt UID 1337 so the proxy's own traffic is not redirected into itself. Any other process running as 1337 silently bypasses the mesh.
- **Putting a mesh anywhere near the RDMA data path.** There is no socket and no packet in the kernel for a proxy to intercept, a proxy hop is ~100× an RDMA half-round-trip, and a collective is a barrier so every rank pays it. Exclude the training namespaces explicitly rather than assuming the mesh will not find them.

## Self-check

**1. Quantify the mesh tax for a fleet doing 30,000 mesh-internal rps across 600 pods on 40 nodes, in sidecar and in ambient mode. Which do you pick and why?**

Every request traverses two proxies, so the proxies collectively handle 60,000 proxy-rps. Using Istio 1.24's published figures (1 KB payload, 2 worker threads): sidecar at **0.20 vCPU / 1000 rps** gives `60 × 0.20 = 12.0 vCPU` and `600 pods × 60 MB = 36 GB`. Ambient L4 with ztunnel at **0.06 vCPU / 1000 rps** and 12 MB per proxy gives `60 × 0.06 = 3.6 vCPU` and `40 nodes × 12 MB = 0.48 GB`. Adding waypoints for the services that genuinely need L7 — say 15 services carrying 40 % of traffic — adds `12 × 0.25 = 3.0 vCPU` and `15 × 2 replicas × 60 MB = 1.8 GB`, for **6.6 vCPU and 2.28 GB total**. Pick ambient: 45 % of the CPU and 6 % of the memory, and — the operational argument that matters more — the ~170 services that need only mTLS and telemetry never get an L7 proxy on their path and therefore cannot be broken by an L7 misconfiguration. The caveat to state: a waypoint costs *more* per proxy than a sidecar (0.25 vs 0.20 vCPU), so if the L7 fraction grows to most services the advantage collapses. Re-run the arithmetic when the L7 fraction changes. And budget the third row nobody budgets: Istio's standard metrics can add ~10⁵–10⁶ active Prometheus series depending on how connected the mesh is.

**2. Why is the sidecar/ambient latency figure everyone quotes wrong, and what is the current one?**

The commonly quoted "~2.7 ms" is Istio **1.10**'s published figure: two proxies added **2.65 ms at p90 and 2.91 ms at p99** over baseline, dropping to 1.7 / 2.69 with `jitter` enabled. By **1.13** the same benchmark reported 1.7 / 2.7 ms, and by **1.20** it reported **0.228 ms p90 and 0.298 ms p99** — roughly a 10× improvement. (Istio 1.24 publishes latency as charts rather than text, so no numeric figure is quoted for it here.) Two methodology points make the number usable: the benchmark is `http/1.1` with a 1 KB payload, mTLS enabled, 2 proxy workers, measured with `fortio` — Istio states explicitly that different hardware gives different values, so the defensible move is to run `istio/tools/perf/benchmark` yourself. And a subtlety Istio documents: Envoy collects telemetry *after* the response is sent, so telemetry cost does not appear in that request's service time but does delay the *next* request — mesh overhead therefore shows up as queue wait in the tail, which is why p99 degrades faster than p50 as load rises.

**3. What does an Istio install give you in the way of circuit breaking, out of the box?**

Nothing, in practice. From `ConnectionPoolSettings` in Istio's own API proto: `http2MaxRequests` defaults to **2³²−1**, `http1MaxPendingRequests` to 2³²−1, `maxRetries` to **2³²−1**, and `maxRequestsPerConnection` to **0** meaning unlimited (up to 2²⁹); the connection idle timeout defaults to 1 hour. So a default install has unlimited concurrency to any destination, an unlimited pending queue, and unlimited concurrent retries — it provides the *capability* to circuit-break and none of the *behaviour*. You get behaviour only when you write a `DestinationRule`, and the diagnostic that it is working is the `UO` (upstream overflow) response flag appearing in the access log. Size `http2MaxRequests` from the backend's real concurrency — for a vLLM replica with `max_num_seqs = 64`, allowing several times that in flight just builds queue at the GPU and inflates p99 without adding throughput. And set `maxRequestsPerConnection`, because with the default an HTTP/2 connection is never recycled and an endpoint-list change therefore does not rebalance existing traffic.

**4. A request fails with `503 UO` in the sidecar access log. What happened, and what did *not* happen?**

`UO` is Envoy's "upstream overflow" response flag: a **circuit-breaker limit was hit** — `http2MaxRequests`, `http1MaxPendingRequests`, `maxConnections`, or `maxRetries` — and the proxy rejected the request rather than forwarding it. What did *not* happen: no network failure (`UF` would say that), no absence of healthy backends (`UH`), no timeout (`UT`), no missing route (`NR`), and no retry exhaustion (`URX`). The application never saw the request. This is usually the mesh working as configured, and the correct response is to check whether the limit matches the backend's real concurrency rather than to raise it reflexively — for a GPU backend, shedding at the proxy is strictly better than queueing at the GPU, because the proxy can shed cheaply and the GPU cannot. If `UO` is appearing without anyone having set limits, check whether a default `DestinationRule` was inherited from a mesh-wide config.

**5. Why can a mesh never sit on the RDMA data path? Give the mechanism, not the slogan.**

Four reasons, each independently sufficient. **(a) There is nothing to intercept.** RDMA bypasses the kernel network stack: the application posts work requests directly to the NIC's queues in user space, so there is no `connect(2)` for a cgroup eBPF hook to rewrite and no packet traversing netfilter for an iptables `REDIRECT` to catch. Both interception mechanisms a mesh uses have no attachment point. **(b) A proxy reintroduces exactly what RDMA deletes** — copies into and out of user space, per-packet CPU, and the kernel scheduler on the data path. **(c) The latency scales are three orders of magnitude apart.** An RDMA half-round-trip is ~1–2 µs; a proxy hop is hundreds of microseconds at best. Because a collective is a **barrier**, every rank waits for the slowest, so the penalty is not amortised — it multiplies the step time. **(d) The threat model is different**: the RDMA fabric is a physically separate network reachable only by pods that were granted a VF or DRA claim, so its isolation comes from device allocation and fabric design rather than from an L7 proxy. The operational instruction that follows: mesh the inference frontend, and *explicitly* exclude the training namespaces and RDMA interfaces with a label rather than relying on a node-level dataplane not noticing them.

**6. `istioctl proxy-status` shows several proxies with `EDS: STALE`. Why does that matter more than a CPU alert, and what causes it?**

Because a `STALE` EDS entry means that proxy is load-balancing against an **endpoint list that no longer reflects reality** — it may be sending requests to pods that have been deleted, and not sending them to pods that have been created. A mesh whose data plane is minutes behind its control plane is worse than no mesh: it routes confidently to the wrong places. Causes, in rough order of likelihood: xDS push volume exceeding what Istiod can compute and deliver (a large Deployment rolling generates thousands of endpoint updates, each fanning out to every proxy that is not scoped away from it); missing `Sidecar` scoping, which makes "affected proxies" mean *all* proxies for every change; Istiod under-replicated or CPU-starved (its CPU scales with deploy rate, config-change rate, and connected-proxy count — Istio documents exactly those three factors); or control/data-plane version skew beyond the supported window, where Istiod emits configuration the proxy silently ignores. The metrics to pair with it are `pilot_xds_pushes` and `pilot_proxy_convergence_time`; the first fix is almost always scoping.

**7. When is "don't add a mesh" the right answer, and what do you propose instead?**

Five cases. **Small and homogeneous** — fewer than roughly 20 services, one or two languages, one team: the mesh solves an *organisational* problem (enforcing consistent networking policy across a boundary you cannot otherwise enforce across) that you do not have, and a shared client library plus SPIFFE/SPIRE delivers most of the value at a fraction of the operational cost. **Tight latency budget** — at ~0.9 ms of added p99 against a 10 ms SLO you are spending 9 % of the budget on infrastructure; propose ambient L4 only (mTLS and L4 telemetry, no L7 hop) or nothing. **No operational owner** — a mesh has a control plane, a CA, an upgrade cadence, and its own on-call surface; without an owner it will be six minor versions behind when a CVE lands. **Non-HTTP workloads** — the L7 features are the differentiator, and on opaque TCP ambient L4 delivers the same mTLS far more cheaply than sidecars. **The RDMA data path** — not a judgement call at all, for the mechanism reasons in question 5. In each of the first four cases the honest alternative is a shared client library plus NetworkPolicy (lesson 05) plus a workload-identity system, and the honest framing is that you are choosing to solve six problems N times rather than take on one large permanent dependency.

## Connections & what's next
This lesson sits directly on lesson 05's dataplane: the mesh's load balancing replaces kube-proxy's for meshed traffic, and its L7 authorisation is defence in depth on top of lesson 05's L3/L4 NetworkPolicy. It reuses lesson 03's mechanisms one layer up — outlier detection, panic thresholds (as `minHealthPercent`), slow start (as `warmupDurationSecs`), and the retry-amplification arithmetic, now with a retry budget as the bound. Lesson 04's cost lens reappears as the metrics-cardinality bill, which is usually the largest line in a mesh budget.

Next: **[07-gpu-and-rdma-networking.md](07-gpu-and-rdma-networking.md)** — the path that must never carry any of this, and the platform work required to provision and operate it.

## References & further reading

**Primary sources — read directly for this lesson**

1. `content/en/docs/ops/deployment/performance-and-scalability/index.md`, `istio/istio.io` **master** — the Istio 1.24 figures in §4: load-test mesh of **1000 services and 2000 pods at 70,000 mesh-wide rps**; at 1000 rps with 1 KB payloads and 2 worker threads, a **sidecar consumes ~0.20 vCPU and 60 MB**, a **waypoint ~0.25 vCPU and 60 MB**, and a **ztunnel ~0.06 vCPU and 12 MB**; the benchmark methodology (5 Equinix M3 Large bare-metal machines on the CNCF Community Infrastructure Lab, Flannel CNI, `http/1.1`, 1 KB payload, 500–1500 rps, 4 client connections, 2 proxy workers, mTLS, measured with `fortio`); the three factors Istiod CPU scales with; and the note that telemetry is collected after the response is sent, so its cost appears as queue wait in tail latency. **The 1.24 p90/p99 latency values are published as PNG charts and were not readable in this pass; they are deliberately not quoted.**
2. The same file on **`release-1.20`** — "the two proxies add about **0.228 ms and 0.298 ms** to the 90th and 99th percentile latency, respectively, over the baseline data plane latency," at `http/1.1`, 1 kB payload, 1000 rps, 2–64 client connections, 2 proxy workers, mTLS; and "a proxy consumes about 0.5 vCPU per 1000 requests per second," with ~50 MB of memory in a large namespace with namespace isolation enabled.
3. The same file on **`release-1.13`** and **`release-1.10`** — the historical figures that anchor §4's table: 1.13's "**1.7 ms and 2.7 ms**" p90/p99, and 1.10's "**2.65 ms and 2.91 ms** … after enabling `jitter`, those numbers reduced to 1.7 ms and 2.69 ms," together with the 1.10-era summary of "0.35 vCPU and 40 MB memory per 1000 requests per second." **Correction to the previous version of this lesson and to common practice:** the ~2.7 ms figure in circulation is 1.10-era and is roughly an order of magnitude too high for a current release.
4. `content/en/docs/ambient/architecture/data-plane/index.md`, `istio/istio.io` master — the ambient datapath in §3: the three workload categories and their labels (`istio.io/dataplane-mode=ambient`, `istio.io/use-waypoint`); transparent redirection to the node-local ztunnel; the HBONE upgrade when the destination is mesh-capable; the note that the tunnel is logically **pod-to-pod** with ztunnel encapsulating from inside each pod's network namespace; ztunnel holding a distinct certificate per node-local service account, authenticating as itself while requesting another identity, with the CA enforcing node-locality via the ServiceAccount JWT ("critical to ensure that a compromised node does not compromise the entire mesh"); and the documented gaps that out-of-mesh workloads, sidecars, and gateways bypass waypoints.
5. `content/en/docs/ambient/architecture/hbone/index.md`, `istio/istio.io` master — HBONE as HTTP/2 + HTTP CONNECT + mTLS on TCP port **15008**, one tunnel per (source identity, destination identity) pair with application connections multiplexed as streams, and the property that the application stream is proxied unaltered so no Istio-specific headers are injected.
6. `content/en/docs/ops/deployment/application-requirements/index.md`, `istio/istio.io` master — the complete port table reproduced in §2: 15000 Envoy admin, 15001 outbound, 15002 failure detection, 15004 debug, 15006 inbound, 15008 HBONE, 15020 merged Prometheus, 15021 health checks, 15053 DNS capture, 15090 Envoy telemetry; plus the Istiod ports 15010 (plaintext xDS/CA), 15012 (TLS xDS/CA), 15014 (monitoring), 15017 (webhook).
7. `tools/istio-iptables/pkg/constants/constants.go`, `istio/istio` master — `DefaultProxyUID = "1337"` and `OutboundMark = "1338"`, the basis for the UID-exemption rule in §2 and the pitfall about application containers running as 1337.
8. `networking/v1alpha3/destination_rule.proto`, `istio/api` master — the `ConnectionPoolSettings` defaults in §9: `http2MaxRequests` **2³²−1**, `maxRetries` **2³²−1**, `maxRequestsPerConnection` **0** (unlimited, up to 2²⁹) with the note that setting it to 1 disables keep-alive, `idleTimeout` **1 hour** if unset, and the TCP keepalive and `maxConnectionDuration` fields.
9. `networking/v1alpha3/virtual_service.proto`, `istio/api` master — `HTTPRetry`: "the maximum possible number of requests made will be **1 + `attempts`**," the automatic retry interval "determined automatically (25 ms+)," the interaction with route `timeout` and `perTryTimeout` that can reduce the actual retry count, and the pointer to Envoy's `x-envoy-retry-on` policy list.
10. `api/envoy/config/cluster/v3/cluster.proto` and `outlier_detection.proto`, `envoyproxy/envoy` main — read in lesson 03 and reused here for the outlier-detection and load-balancing defaults that Istio's `DestinationRule` surfaces (`consecutive_5xx = 5`, `interval = 10 s`, `base_ejection_time = 30 s`, `max_ejection_percent = 10 %`, `LEAST_REQUEST` with `choice_count = 2`), and for `RetryBudget` as the bound recommended in §8.

**Sources named but not fetched in this pass — do not treat the wording as verified**

11. `istio.io` rendered documentation and `envoyproxy.io` documentation are **blocked by this environment's egress policy**. All Istio material above was read from the `istio/istio.io`, `istio/api`, and `istio/istio` source repositories on GitHub, which is the same content pre-render; all Envoy material was read from the `envoyproxy/envoy` API protos. Two things in this lesson come from neither and are stated as engineering practice rather than as sourced fact: the specific `iptables` rule text in §2 (which is illustrative of the shape Istio's init produces, not a verbatim dump), and the Prometheus memory-per-series rule of thumb in §5 — measure both on your own cluster.
