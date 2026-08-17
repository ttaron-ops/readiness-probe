---
lesson: "02.1"
title: "Kubernetes component internals — how each component works"
module: "02"
concept: "Kubernetes component internals"
status: not-started
est_time: "18h"
prev: null
next: "02-api-machinery.md"
artifacts: []
sources: 21
---

# 02.1 · Kubernetes component internals — how each component works

> **Concept.** The internal machinery of each control-plane and node component — request flow, caches, control loops, consensus — the layer beneath what a CKA operator already knows.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Module 01 built `gpu-cost-exporter` — a Go binary that reads GPU metrics and talks to *a* Kubernetes API, treated mostly as a black box you call. This module turns that black box into a machine you can extend: a controller that watches, reconciles, and writes back. Before you can build that controller, you need to know what it's actually a client *of* — what happens inside the apiserver when it writes, what the scheduler does before your pod lands on a node, and why a component you'll depend on (leader election, the watch cache, PLEG) behaves the way it does under load. This lesson is that foundation: no new Kubernetes concepts you haven't used, just the internals of the ones you already operate.

## Why this matters

Your capstone operator is a client of these internals, not a bystander to them. It will hold a `watch` against the apiserver's **watch cache** and must survive reconnects without dropping GPU-node events — so `resourceVersion` semantics are load-bearing, not trivia. It will run HA behind **leader election** for the exact reason kube-controller-manager does. And to reason about GPU placement cost you have to know how the **scheduler** actually binds a pod — device-plugin integer counts vs DRA claims change what "a GPU node is full" even means.

It's also where "operates Kubernetes" and "builds on Kubernetes" interviews diverge hardest. A CKA holder can explain *that* a node went `NotReady`; a Senior/Staff platform engineer building controllers for a GPU fleet is expected to explain *why* — PLEG stall vs missed lease renewal vs a genuinely dead kubelet are three different root causes with three different fixes, and conflating them wastes an incident's worth of time on a fleet where GPU-hours are burning idle the whole time you're debugging. NVIDIA and CoreWeave job descriptions for these roles explicitly ask for "extending Kubernetes components" and owning "the control plane" — this lesson is the first layer of that competence.

## What's new here (calibration)

You have a CKA and run 40+ clusters, so this lesson does **not** re-teach: what a pod/node/Service is, `kubectl` usage, YAML authoring, Helm/Kustomize, CNI/CSI/Ingress *usage*, or cluster bring-up. What it adds instead:

- **Internal request/data flow inside each component** — the apiserver's filter chain and admission pipeline order, the scheduler's Filter/Score/Reserve/Bind cycle, the kubelet's syncLoop and PLEG — not "what the component does" but "how it does it and what can make it slow or wrong."
- **The one structural fact that unifies all of it** — every component is a client of the apiserver, and the apiserver is the only client of etcd — which is the mental model your controller inherits.
- **Failure-mode causality** — PLEG stall vs lease expiry vs watch-cache staleness as distinguishable, diagnosable conditions, not synonyms for "something's wrong."
- **GPU-specific placement machinery** — device-plugin integer counts vs DRA's claim/attribute model, which changes what your cost operator can even observe about GPU allocation.

Version note: unless stated otherwise, defaults and flag values below are from the Kubernetes 1.3x line (verified against the master branch of `kubernetes/kubernetes` and the current kubernetes.io reference docs as of August 2026; v1.36 is the current stable release). Where a behaviour changed at a specific release, the release is named. Command transcripts are **representative** — formatted the way the real tool formats them, with realistic values — not literal captures from a specific cluster, except where a number is attributed to a named source.

## Core concepts

### The one structural fact

**Every component is a client of the apiserver, and the apiserver is the only client of etcd.** No component watches etcd directly, shares memory with another component, or sends another component a message. All coordination happens by writing objects through the apiserver and observing them through watches. Internalise this and everything else in Kubernetes is a variation on list-watch-reconcile.

This is not an accident of implementation; it is the reason the extension model works at all. Because there is exactly one write path into storage, there is exactly one place that enforces authentication, authorisation, admission, defaulting, schema validation, versioned conversion, and optimistic concurrency. Your operator gets all of that for free the moment it speaks HTTP to the apiserver, and it is subject to all of it in exactly the same way kube-controller-manager is. There is no privileged internal API for built-in controllers.

```
                        ┌────────────────────────────────┐
                        │   etcd cluster (Raft, MVCC)    │
                        │   3 or 5 members, one leader   │
                        └──────────────▲─────────────────┘
                                       │  gRPC — the ONLY client
                                       │
              ┌────────────────────────┴───────────────────────────┐
              │              kube-apiserver (stateless)            │
              │  filter chain → admission → registry → storage     │
              │  ┌──────────────────────────────────────────────┐  │
              │  │ watch cache: 1 etcd watch per resource type,  │  │
              │  │ N thousand client watches served from RAM     │  │
              │  └──────────────────────────────────────────────┘  │
              └──▲────────▲──────────▲─────────▲─────────▲─────────┘
   REST + watch  │        │          │         │         │
        ┌────────┘        │          │         │         └──────────┐
        │                 │          │         │                    │
   kubectl / you   kube-scheduler  kube-        cloud-controller   kubelet
                   (writes         controller-  manager            (per node)
                    Binding)       manager      (LB/route/node)      │ CRI gRPC
                                   (~30 ctrls,                       ▼
                                    leader-elected)          containerd / CRI-O
                                                                     │
                                                                     ▼
                                                              shim → runc → [ns+cgroups]

   kube-proxy (per node) ── watches Service + EndpointSlice ──▶ programs netfilter/IPVS
   CoreDNS    (deployment) ── watches Service + EndpointSlice ──▶ answers cluster DNS
```

Read the diagram as a claim you can falsify: pick any arrow that is *not* drawn — scheduler to kubelet, kubelet to etcd, kube-proxy to CoreDNS — and you will not find it in the source. The scheduler does not tell the kubelet anything. It writes a `Binding` subresource, the apiserver sets `pod.spec.nodeName`, and the kubelet finds out because it holds a watch on pods filtered by `spec.nodeName=<its own name>`.

### kube-apiserver: the write path, stage by stage

The apiserver is a stateless HTTP server. Every request passes through a chain of `http.Handler` decorators (built by `DefaultBuildHandlerChain` in `k8s.io/apiserver/pkg/server/config.go`) and then into a per-resource REST handler. Handlers wrap outermost-last in the code, so the reading order in the source is inside-out; the execution order for an incoming request is the reverse.

Here is the whole path for `kubectl apply -f pod.yaml`, with what fails at each stage and what the failure looks like from the client:

```
kubectl apply -f pod.yaml
   │  (client-side: read kubeconfig, TLS handshake, POST /api/v1/namespaces/ns/pods)
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1  PANIC RECOVERY / AUDIT INIT / REQUEST INFO                               │
│    Parses the URL into {verb, group, version, resource, subresource, ns,    │
│    name}. Everything downstream keys off this RequestInfo.                  │
│    fails → 404 (path not a known resource)                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│ 2  AUTHENTICATION                                                           │
│    Tries each authenticator in order: client cert (x509 CN=user, O=groups), │
│    bearer token (SA JWT, OIDC id_token), webhook, anonymous.                │
│    fails → 401 Unauthorized                                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│ 3  TIMEOUT / DEADLINE / CORS / WARNING RECORDER / IMPERSONATION             │
│    --request-timeout default 1m0s; long-running (watch, exec) exempted.     │
│    fails → 504 (timeout), 403 (impersonation not allowed)                   │
├─────────────────────────────────────────────────────────────────────────────┤
│ 4  API PRIORITY AND FAIRNESS (APF)                                          │
│    FlowSchema matches the request → PriorityLevelConfiguration → queue or   │
│    seat. Total seats = --max-requests-inflight (400) +                      │
│    --max-mutating-requests-inflight (200) = 600.                            │
│    fails → 429 Too Many Requests (+ Retry-After)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│ 5  AUTHORIZATION                                                            │
│    Authorizer chain, usually Node → RBAC → (Webhook). First ALLOW wins.     │
│    fails → 403 Forbidden ("User \"x\" cannot create resource \"pods\"…")     │
├─────────────────────────────────────────────────────────────────────────────┤
│ 6  DECODE + FIELD VALIDATION                                                │
│    Body (JSON/YAML/protobuf) → internal Go object via the scheme's codec.   │
│    fieldValidation=Strict|Warn|Ignore; kubectl defaults to Strict.          │
│    fails → 400 Bad Request (unknown field, bad type)                        │
├─────────────────────────────────────────────────────────────────────────────┤
│ 7  MUTATING ADMISSION                                                       │
│    In-process plugins (ServiceAccount, DefaultStorageClass, LimitRanger,    │
│    Priority, RuntimeClass …) then MutatingAdmissionWebhook (serial, by      │
│    reinvocation policy), then MutatingAdmissionPolicy (CEL, in-process).    │
│    THIS IS THE ONLY PLACE THE OBJECT CAN BE REWRITTEN.                      │
│    fails → 400/403/500 depending on plugin; webhook down + failurePolicy:   │
│            Fail → 500 "failed calling webhook"                              │
├─────────────────────────────────────────────────────────────────────────────┤
│ 8  DEFAULTING + SCHEMA/STRATEGY VALIDATION  (rest.BeforeCreate)             │
│    Apply API defaults, then the type's Strategy.Validate: required fields,  │
│    value ranges, immutability, name/label syntax. For CRs: the structural   │
│    OpenAPI schema + CEL x-kubernetes-validations.                           │
│    fails → 422 Unprocessable Entity (Invalid)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│ 9  VALIDATING ADMISSION                                                     │
│    Passed *into* the storage call as a validation callback. In-process      │
│    plugins (ResourceQuota, PodSecurity, NamespaceLifecycle …), then         │
│    ValidatingAdmissionPolicy (CEL), then ValidatingAdmissionWebhooks —      │
│    webhooks called IN PARALLEL, any reject kills the request.               │
│    NOTHING HERE MAY MUTATE. fails → 403 Forbidden (admission denied)        │
├─────────────────────────────────────────────────────────────────────────────┤
│ 10 FIELD MANAGEMENT (managedFields) + SERIALIZE                             │
│    Server-side apply bookkeeping; encode to the storage version.            │
│    --storage-media-type default application/vnd.kubernetes.protobuf         │
│    (CRs are always stored as JSON).                                         │
│    fails → 409 Conflict (SSA field-ownership conflict, no force)            │
├─────────────────────────────────────────────────────────────────────────────┤
│ 11 etcd TRANSACTION                                                         │
│    Txn: compare ModRevision(key) == expectedRV, then Put, else Get.         │
│    Create uses ModRevision == 0 ("key must not exist").                     │
│    Committed only after Raft quorum fsyncs the WAL.                         │
│    fails → 409 Conflict (RV mismatch), 500 (etcd unavailable/NOSPACE),      │
│            504 (quorum lost)                                                │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ etcd revision N assigned → becomes the
                               │ object's metadata.resourceVersion
                               ▼
                    WATCH CACHE fan-out (in the apiserver)
                    one MODIFIED/ADDED event copied to every
                    interested watcher: scheduler, kubelet(s),
                    controller-manager, your operator, kubectl -w
```

Four things in that path are worth stopping on.

**Mutating runs before validating, and only mutating may write.** That ordering is not stylistic. If validation ran first, a mutating webhook could inject something the validators already approved. The two-phase split guarantees that whatever the validators see is exactly what will be stored. It also means a validating webhook can never "fix" an object — if you want defaulting behaviour, you need a mutating webhook or a CEL mutating policy, and you inherit the ordering hazards that come with it (reinvocation, webhook-vs-webhook races). Lesson 08 lives here.

**Schema validation sits *between* the two admission phases.** In the source, `rest.BeforeCreate` (defaulting + `Strategy.Validate`) runs inside the registry `Create` call, and validating admission is passed in as the `createValidation` callback that the store invokes just before writing. Practically: a mutating webhook that produces a structurally invalid object gets a `422`, not a webhook error — which is why "my webhook broke and the error blames my CRD schema" is a real and confusing failure mode.

**Validating webhooks are called in parallel; mutating webhooks are not.** The docs state it plainly: matching validating webhooks are called in parallel and any rejection fails the request. Mutating webhooks must be serial because each one's output is the next one's input. Consequence for latency budgeting: N validating webhooks cost `max(RTT)`; N mutating webhooks cost `sum(RTT)`.

**The default in-process admission plugin set** (enabled unless you override `--enable-admission-plugins` / `--disable-admission-plugins`) is: `CertificateApproval`, `CertificateSigning`, `CertificateSubjectRestriction`, `DefaultIngressClass`, `DefaultStorageClass`, `DefaultTolerationSeconds`, `LimitRanger`, `MutatingAdmissionWebhook`, `NamespaceLifecycle`, `PersistentVolumeClaimResize`, `PodSecurity`, `Priority`, `ResourceQuota`, `RuntimeClass`, `ServiceAccount`, `StorageObjectInUseProtection`, `TaintNodesByCondition`, `ValidatingAdmissionPolicy`, `ValidatingAdmissionWebhook`. Note that `ResourceQuota` and `PodSecurity` are in the *validating* phase — that is why a quota rejection reads as a `403`, not a `422`.

`ValidatingAdmissionPolicy` (CEL, in-process, no webhook pod) reached GA in v1.30. `MutatingAdmissionPolicy` is the mutating counterpart, beta in v1.34 behind the `MutatingAdmissionPolicy` gate plus `--runtime-config=admissionregistration.k8s.io/v1beta1=true`. The reason to care: an in-process CEL policy cannot be "down," so it does not create the clusterwide write outage that a `failurePolicy: Fail` webhook with a dead backend does.

### How the apiserver actually writes to etcd

The storage layer (`k8s.io/apiserver/pkg/storage/etcd3`) keys objects as `/registry/<resource>/<namespace>/<name>` for core types and `/registry/<group>/<resource>/<namespace>/<name>` for grouped ones — for example `/registry/pods/default/trainer-0` or `/registry/deployments/prod/api`. Cluster-scoped objects drop the namespace segment.

Writes are compare-and-swap, not blind puts. `GuaranteedUpdate` reads the current object, lets the caller mutate a copy, then issues an etcd transaction that only commits if the key's `ModRevision` still equals the revision that was read. In current code that is wrapped as `OptimisticPut(ctx, key, newData, origState.rev, …)`; a `Create` calls the same helper with revision `0`, which means "only succeed if this key does not exist." If the transaction's compare fails, the store logs *"GuaranteedUpdate of %s failed because of a conflict, going to retry"*, re-reads, and re-applies — but if the mutation came from a client-supplied `resourceVersion` (a `PUT`/`Update`), the conflict is surfaced to the client instead:

```
Error from server (Conflict): Operation cannot be fulfilled on workloadcosts.finops.example.com
"gpu-train-77": the object has been modified; please apply your changes to the latest version and try again
```

That is the whole of optimistic concurrency in Kubernetes: **`metadata.resourceVersion` is the etcd `ModRevision` of the key, and an update carrying a stale one is rejected.** Lesson 02 takes this apart in detail; what matters here is that it is enforced by an etcd transaction, in the storage layer, not by anything in your controller.

Two consequences you will hit:

- `resourceVersion` values come from etcd's **global** revision counter, not a per-object counter. A Pod created after a busy Deployment rollout can have an RV thousands larger than its neighbour with no relationship between the two. Since v1.35 the API guarantees resource versions of the *same resource type* are orderable as decimal integers, which makes "is this newer?" answerable within a type — but never across types, and never across clusters.
- Object size is bounded by etcd's request limit: `--max-request-bytes` defaults to **1.5 MiB**. That is the real reason a giant ConfigMap or a CRD with a huge status fails with `etcdserver: request is too large`.

### The watch cache

**The problem.** A 5,000-node cluster has 5,000 kubelets watching pods, plus every controller in kube-controller-manager, plus kube-proxy and CoreDNS on every node, plus your operator, plus every `kubectl get -w` anyone has open. If each of those opened its own etcd watch, etcd would spend all its time fanning out identical events. etcd is a consensus store tuned for durable small writes, not a pub/sub broker.

**The mechanism.** For each resource type the apiserver runs one `Cacher` (`k8s.io/apiserver/pkg/storage/cacher`). Each Cacher holds one etcd watch, decodes each event once, and keeps two things: a **store** (the current object set, indexed) and a **ring buffer of recent events** ordered by resourceVersion. Client watches are served from that ring buffer and store — never from etcd.

```
              etcd                                kube-apiserver
   ┌────────────────────────┐        ┌──────────────────────────────────────────┐
   │ revision 900 … 1042    │  ONE   │  Cacher("pods")                          │
   │  watch from rev 900 ───┼────────▶  decode once, then:                      │
   └────────────────────────┘ stream │                                          │
                                     │  store:  {ns/name → *v1.Pod}  (current)  │
                                     │                                          │
                                     │  ring buffer (watchCache) — capacity     │
                                     │  grows/shrinks between                   │
                                     │  defaultLowerBoundCapacity = 100 and     │
                                     │  defaultUpperBoundCapacity = 100*1024,   │
                                     │  sized to hold eventFreshDuration = 75s  │
                                     │  of history                              │
                                     │                                          │
                                     │  [rv1001][rv1007][rv1012] … [rv1042]     │
                                     │     ▲oldest                  ▲newest     │
                                     └─────┬──────────────┬─────────────────────┘
                                           │              │
      client A: WATCH ?resourceVersion=1012│              │ client B: WATCH ?resourceVersion=930
      → replay 1012..1042 from ring,       │              │ → 930 < oldest-1
        then stream live  ✔                │              │ → 410 Gone:
                                           │              │   "too old resource version: 930 (1000)"
                                           │              │ → informer must LIST again
                                           ▼              ▼
                                    ADDED/MODIFIED/DELETED/BOOKMARK events
```

The exact error text is worth memorising because you will see it in logs: `watch_cache.go` returns

```go
if resourceVersion < oldest-1 {
    return nil, errors.NewResourceExpired(
        fmt.Sprintf("too old resource version: %d (%d)", resourceVersion, oldest-1))
}
```

which surfaces to the client as HTTP `410 Gone`. The first number is what you asked for; the second is the oldest revision the cache can still replay. **A `410` is a protocol event, not an error** — the contract is: throw away your resume point, do a fresh LIST to get current state and a current RV, and restart the watch. Informers do this automatically. Alert on watches that fail to *recover*, never on the 410 itself.

Numbers, from the source and the reference docs:

| Thing | Value | Where it comes from |
|---|---|---|
| Watch cache enabled | `--watch-cache` default `true` | kube-apiserver flags |
| Per-resource overrides | `--watch-cache-sizes` (e.g. `pods#1000,nodes#500`) | kube-apiserver flags |
| Ring buffer capacity | dynamic, `100` … `102400` events | `defaultLowerBoundCapacity` / `defaultUpperBoundCapacity`, `watch_cache.go` |
| History window target | `75s` (`eventFreshDuration`) | `watch_cache.go` |
| Bookmark cadence | ~`1m` (`defaultBookmarkFrequency`) | `cacher.go` |
| Wait-for-fresh timeout | `3s` (`blockTimeout`) | `watch_cache.go` |
| etcd history retained | ~5 min by default | apiserver `--etcd-compaction-interval` default `5m0s` |

**Bookmarks** exist because of a nasty interaction: if you watch a *quiet* resource, your last-seen RV stays frozen while the cluster's global revision marches on and etcd compacts past it. Reconnect and you get a `410` even though nothing you care about changed. A `BOOKMARK` event carries no object change, only a fresh `metadata.resourceVersion`, so idle watchers keep their resume point current. Request them with `allowWatchBookmarks=true`; client-go informers do by default.

**Consistent reads are no longer synonymous with "hit etcd."** This is the piece most older material gets wrong. Historically, a LIST with no `resourceVersion` forced a quorum read from etcd, and `resourceVersion=0` meant "serve from cache, possibly stale." Since the `ConsistentListFromCache` work went beta and on-by-default in **v1.31**, a consistent LIST can be served from the watch cache: the apiserver asks etcd for its current revision, then uses etcd's **watch progress notifications** (`WatchProgressRequest`, requested roughly every 100 ms) to wait until the cache has caught up to that revision, and serves from RAM once it has. The KEP reports the cache became fresh within 110 ms in 99.9% of cases, with 2–10× lower apiserver CPU and 20–50× lower LIST latency in their benchmarks. v1.34 extended this with `ListFromCacheSnapshot` (beta, on by default), which snapshots the cache's B-tree so paginated and RV-pinned LISTs can also be served from cache.

The practical upshot for your operator has not changed, though: **a LIST is expensive regardless of where it is served from**, because the server still has to serialise every matching object. `resourceVersion=0` remains the cheapest read (any version, straight from the store). Prefer watches over polling, and prefer a cached read over a LIST in a reconcile loop.

The current `?resourceVersion` semantics for GET/LIST, from the API reference:

| `resourceVersionMatch` | Paging | RV unset | RV = "0" | RV = other |
|---|---|---|---|---|
| unset | none | Most Recent | Any | Not older than |
| unset | `limit=n` | Most Recent | Any | Exact |
| unset | `limit`+`continue` | Continuation | Continuation | Invalid (400) |
| `Exact` | none | Invalid | Invalid | Exact |
| `NotOlderThan` | none | Invalid | Any | Not older than |

And for WATCH: RV unset → "get state and start at most recent" (consistent read, then synthetic ADDED events, then live); RV = `0` → "get state and start at any" (possibly stale initial state); RV = a value → "start at exact," no initial state, stream only what came after.

**Streaming lists** (`WatchList`, promoted in v1.32 and on by default in the 1.34+ line) collapse LIST+WATCH into one request: `?watch=1&sendInitialEvents=true&allowWatchBookmarks=true&resourceVersionMatch=NotOlderThan`. The server streams the initial state as synthetic `ADDED` events and then a `BOOKMARK` annotated `"k8s.io/initial-events-end": "true"` to mark the boundary. This removes the giant one-shot LIST response that used to spike apiserver heap when hundreds of informers restarted at once.

**Pagination.** Large LISTs are chunked with `?limit=500`, and the response carries a `continue` token. The collection's `resourceVersion` stays constant across pages, so you get a consistent snapshot. Continue tokens expire after roughly 5 minutes; using an expired one returns `410 Gone` and you restart the listing.

### API Priority and Fairness

APF replaced the old global in-flight caps. Requests are matched by a **FlowSchema** to a **PriorityLevelConfiguration**; each priority level gets a share of the server's total concurrency, and within a level, requests are distributed by shuffle-sharded fair queuing keyed on a flow distinguisher (usually the username or namespace).

- Total seats = `--max-requests-inflight` (400) + `--max-mutating-requests-inflight` (200) = **600** on a default apiserver.
- A level's nominal seats = `total × its nominalConcurrencyShares ÷ sum of all shares`.
- Default levels: `exempt` (system:masters, unlimited, never queued), `node-high` (node status/lease updates), `system` (kubelet traffic), `leader-election` (lease renewals — deliberately isolated so a busy cluster cannot cause a controller-manager to lose its lease), `workload-high` (built-in controllers), `workload-low` (service-account/pod traffic), `global-default` (interactive users), `catch-all` (a deliberately tiny share).
- LIST requests are charged **more than one seat**, estimated from the expected object count (the `SizeBasedListCostEstimate` gate, beta and on by default in v1.34, refines this by response size). A watch occupies a seat only until its initial burst of events has been delivered.
- Over the limit with `type: Reject` → HTTP **429** with a `Retry-After`. Metrics: `apiserver_flowcontrol_rejected_requests_total`, `apiserver_flowcontrol_current_inqueue_requests`, `apiserver_flowcontrol_request_wait_duration_seconds`.

Why it matters for your operator: leader-election lease renewals live in their own priority level *because* a controller that hot-loops LISTs used to be able to starve them and cause leadership flapping across the whole control plane. When your operator starts getting 429s, respect `Retry-After` — client-go's rate limiter and the workqueue's backoff already do — and then go look at whether you are LISTing where you should be watching.

### The aggregation layer vs CRDs

Two different extension mechanisms that people conflate:

| | CustomResourceDefinition | APIService (aggregation) |
|---|---|---|
| Who serves it | the main apiserver, from a generic registry | a separate **extension apiserver** process; the main apiserver proxies |
| Storage | etcd, as JSON, under `/registry/<group>/<plural>/…` | whatever that extension server wants — often nothing persistent |
| Schema | structural OpenAPI v3 + CEL validations you write | whatever the extension server implements |
| Cost to add | one YAML object | run and operate another HTTPS service |
| Typical use | your `GPUCostPolicy`, `WorkloadCost`, `Budget` | `metrics.k8s.io` (metrics-server), `custom.metrics.k8s.io` |

Your capstone uses CRDs. It may *consume* `metrics.k8s.io` (that is where `kubectl top` gets its numbers), which is an aggregated API and therefore has no etcd backing at all — the values are computed on demand from kubelet scrapes and are not historical.

### etcd

**Raft and quorum.** etcd members elect a leader; a write is committed only once a majority has persisted it to the Raft write-ahead log. The arithmetic is the whole reason for odd member counts:

| Members | Quorum | Failures tolerated | Notes |
|---|---|---|---|
| 1 | 1 | 0 | dev only |
| 3 | 2 | 1 | the standard |
| 4 | 3 | 1 | strictly worse than 3 — same tolerance, more latency |
| 5 | 3 | 2 | large/critical clusters |
| 7 | 4 | 3 | write latency starts to hurt |

Every additional member adds a replica the leader must reach before committing, so write latency is bounded below by the *slowest member in the quorum*. That is why cross-region etcd is a bad idea and why an even count is pure downside.

**MVCC and revisions.** etcd never overwrites in place. Every mutating transaction increments a cluster-global **revision**, and the old value stays readable at its old revision until compaction removes it. That single counter is:

- the source of `metadata.resourceVersion`,
- what makes `watch from revision N` possible at all,
- and, when it is compacted away, the ultimate origin of `410 Gone`.

**Compaction vs defragmentation** — different operations, constantly confused:

- **Compaction** discards MVCC history older than a chosen revision. It frees *logical* space inside the DB file and makes old revisions unreadable. The apiserver drives this itself: `--etcd-compaction-interval` defaults to `5m0s`. etcd can also self-compact with `--auto-compaction-mode=periodic --auto-compaction-retention=1h` (or `revision` mode, e.g. keep the last 1000 revisions, checked every 5 minutes). etcd's own docs suggest `10h` as a general-purpose default when etcd does its own compaction — but under Kubernetes the apiserver's 5-minute interval is what you actually get, which is why the practical watch history window is "about five minutes."
- **Defragmentation** (`etcdctl defrag`) returns the freed pages to the filesystem, shrinking the actual file. It **blocks reads and writes on the member it runs against** while it rebuilds. Run it one member at a time, never against all members at once, and never against the leader during a busy period.

**Limits.**

| Limit | Default | Flag / note |
|---|---|---|
| Max request size | 1.5 MiB | `--max-request-bytes` |
| DB size quota | 2 GiB | `--quota-backend-bytes`; exceeding it raises a `NOSPACE` alarm and etcd goes **read-only** |
| Suggested max DB size | 8 GiB | etcd warns at startup if you configure more |
| Clearing NOSPACE | compact → defrag → `etcdctl alarm disarm` | in that order |

A cluster that hits `NOSPACE` looks, from the outside, like a total control-plane outage: every write fails, nothing schedules, nodes eventually go `NotReady` because status updates fail. The fix is three commands, and knowing them cold is worth an hour of downtime.

**Disk latency is apiserver latency.** Raft commit requires an `fsync` to the WAL. The metrics that matter are `etcd_disk_wal_fsync_duration_seconds` and `etcd_disk_backend_commit_duration_seconds` — watch the p99. This is the single most reliably reproduced etcd production problem, and OpenAI's 2,500-node write-up documents it with numbers: on network-attached Azure P30 SSDs rated at 5,000 IOPS, etcd could use only about 10% of the available IOPS because it issues sequential, latency-bound writes and each one cost ~2 ms. Moving the etcd data directory to a locally-attached SSD dropped write latency to **200 µs** and made the cluster healthy. Their later 7,500-node post reports running **5 apiservers and 5 etcd members**, moving Kubernetes **Events into a separate etcd cluster** (Events are high-churn and were the main source of DB growth), and observing up to **70 GB of heap per apiserver** at that scale. Their high-level alert signal was the rate of HTTP 429 and 5xx from the apiservers.

### kube-scheduler

The scheduler watches for pods with an empty `spec.nodeName` and assigns each one a node by creating a `Binding`. Internally it is a plugin framework driving a queue.

```
   ┌────────────────────── scheduling queue ─────────────────────────────┐
   │                                                                     │
   │   activeQ            backoffQ              unschedulableQ           │
   │   (heap, ordered     (failed recently,     (no node fit last time;  │
   │    by priority       waiting out           waits for a cluster      │
   │    then timestamp)   exponential backoff)  event that could help)   │
   │       ▲   │               │                        │                │
   │       │   │  pop          │ backoff expires        │ QueueingHint    │
   │       │   └───────────┐   └────────────────────────┘  says "this     │
   │       └───────────────┴──────────────────────────────  event could   │
   │                                                        unblock you"  │
   └──────────────────────────┬──────────────────────────────────────────┘
                              │ one pod
   ┌──────────────── SCHEDULING CYCLE — serial, one pod at a time ───────────────┐
   │ PreEnqueue → (QueueSort) → PreFilter → Filter → [PostFilter: preemption]    │
   │            → PreScore → Score → NormalizeScore → Reserve → Permit           │
   │                                                                            │
   │ Filter runs against a SNAPSHOT of the scheduler cache taken at cycle start, │
   │ so the whole cycle sees one consistent view of nodes and pods.              │
   │ Reserve "assumes" the pod onto the node IN CACHE before the bind lands —    │
   │ that is what stops the next pod double-booking the same GPU.                │
   └──────────────────────────┬─────────────────────────────────────────────────┘
                              │ node chosen, resources assumed
   ┌──────────── BINDING CYCLE — may run concurrently with the next pod ─────────┐
   │ WaitOnPermit → PreBind → Bind (POST /binding) → PostBind                    │
   │ any failure here → Unreserve (all Reserve plugins, in reverse) → requeue    │
   └────────────────────────────────────────────────────────────────────────────┘
```

**Serial scheduling, concurrent binding** is the design's core tradeoff, and the docs state it directly: scheduling cycles run serially, binding cycles may run concurrently. Serial scheduling is what makes the assume-in-cache trick correct; concurrent binding is what stops a slow volume attach from blocking the whole cluster's scheduling throughput.

Extension points, in execution order:

| Point | Cycle | What a plugin does there |
|---|---|---|
| `PreEnqueue` | queue | gate a pod out of activeQ entirely (e.g. Kueue holding a job until its quota is granted) |
| `QueueSort` | queue | order activeQ; exactly one plugin may be enabled (`PrioritySort` by default) |
| `PreFilter` | sched | precompute per-pod state; can abort the cycle |
| `Filter` | sched | "can this pod run on this node?" — the old *predicates* |
| `PostFilter` | sched | only if no node passed Filter; `DefaultPreemption` lives here |
| `PreScore` | sched | precompute shared state for scoring |
| `Score` | sched | rank surviving nodes — the old *priorities* |
| `NormalizeScore` | sched | rescale a plugin's raw scores into 0..100 |
| `Reserve` | sched | claim resources in the scheduler cache (and `Unreserve` to roll back) |
| `Permit` | sched | approve / deny / **wait** — how gang scheduling holds pods |
| `PreBind` | bind | side effects that must succeed before binding (volume provisioning) |
| `Bind` | bind | write the `Binding`; first plugin to handle it wins (`DefaultBinder`) |
| `PostBind` | bind | informational cleanup |

Default-enabled plugins include `NodeResourcesFit`, `NodeAffinity`, `TaintToleration`, `PodTopologySpread`, `InterPodAffinity`, `NodeUnschedulable`, `NodePorts`, `VolumeBinding` and the volume-limit plugins at Filter; `ImageLocality`, `NodeResourcesBalancedAllocation`, `TaintToleration`, `NodeAffinity`, `PodTopologySpread`, `InterPodAffinity`, `VolumeBinding` at Score. The exact set shifts between releases — dump `KubeSchedulerConfiguration` from your own cluster rather than trusting a list in a doc.

The FinOps-relevant knob is `NodeResourcesFit`'s `scoringStrategy`: **`LeastAllocated`** (the default) spreads pods across nodes; **`MostAllocated`** bin-packs them. On a GPU fleet with an autoscaler, `LeastAllocated` will happily leave you with eight nodes each holding one GPU pod — eight instances billed, seven GPUs idle. `MostAllocated` (or `RequestedToCapacityRatio` with a GPU-weighted shape) is what lets nodes drain to empty and scale down. This is a one-line config change with a five-figure monthly cost delta on a real GPU cluster.

**The queue is where throughput lives.** A pod that failed to schedule sits in `unschedulableQ`. Retrying it blindly on every cluster change wastes cycles proportional to `pending_pods × cluster_event_rate`. **QueueingHints** let each plugin declare which cluster events could plausibly make *its* rejection stale — a node added, a PVC bound, a `ResourceSlice` updated — so a GPU pod pending on capacity wakes precisely when a GPU is freed instead of on every unrelated pod update. For your cost operator, scheduler queue latency (`scheduler_pending_pods`, `scheduler_pod_scheduling_duration_seconds`) is a direct input to "how long did this job wait before it started burning GPU-hours."

**Preemption.** If Filter finds no feasible node, `PostFilter` runs `DefaultPreemption`: it looks for a node where deleting some lower-priority pods would let the pending pod fit, prefers victim sets that respect PodDisruptionBudgets, deletes the victims, and records `status.nominatedNodeName` on the pending pod so the freed space is not immediately stolen by another pod. A pod with `preemptionPolicy: Never` will not preempt others (it can still be preempted).

**GPU placement: device plugin vs DRA.** With the classic **device-plugin** model, a GPU is an opaque extended resource. The node's plugin advertises `nvidia.com/gpu: 8` in `status.capacity` and `NodeResourcesFit` treats it as an integer count. The scheduler cannot see GPU model, memory size, MIG partitioning, NVLink topology, or time-slicing — all of which get smuggled into node labels plus `nodeAffinity`, and none of which the scheduler can reason about. Fractional and shared GPUs are simply invisible.

**Dynamic Resource Allocation** replaces the count with a claim model. The core `resource.k8s.io` APIs graduated to **GA in v1.34**:

```
  DeviceClass                (cluster-scoped, admin-authored: "nvidia-h100",
      ▲                       names a driver + base selectors/config)
      │ referenced by
  ResourceClaim  ◀────────── ResourceClaimTemplate  ◀── pod.spec.resourceClaims
      │  (one per workload or per pod; carries the ALLOCATION RESULT once bound)
      │
      │  scheduler's DRA plugin allocates from ▼
  ResourceSlice              (published by the device driver on each node:
                              the real devices and their ATTRIBUTES —
                              model, memory, MIG profile, shareable capacity)
```

The DRA plugin runs at PreFilter/Filter/Reserve, matches the claim's device selectors against the attributes in `ResourceSlice`s, allocates *specific devices*, and records the allocation inside the `ResourceClaim`'s status. Kubelet then hands that allocation to the DRA driver, which prepares the device for the pod.

For a GPU-cost operator this is the pivotal change: **which pod holds which physical device is finally first-class API data** instead of a label-and-guess exercise. On a DRA cluster you attribute cost by reading `ResourceClaim.status.allocation` and joining to `ResourceSlice` attributes; on a device-plugin cluster you fall back to `nvidia.com/gpu` request counts plus node labels and accept the loss of fidelity. Design for both. (Lesson 09 goes deep on DRA and Kueue; here you only need the scheduler-internals shape.)

### kube-controller-manager

One binary hosting roughly thirty controllers: deployment, replicaset, statefulset, daemonset, job, cronjob, node-lifecycle, endpoints, endpointslice, serviceaccount, namespace, garbage-collector, PV/PVC binder, resourcequota, certificate-signing, ttl, and more. It is the reference implementation of the pattern your operator copies, so read it as a template rather than as infrastructure.

**Shared informers.** Every controller needs Pods; several need Nodes. Rather than each opening its own watch, they share informers from a single `SharedInformerFactory`:

```
  apiserver ──list+watch──▶ Reflector ──▶ DeltaFIFO ──pop──┬─▶ Indexer / Store   (thread-safe local cache)
                                                           │
                                                           └─▶ event handlers ──▶ workqueue (KEYS only)

  workqueue ──▶ worker goroutine ──▶ Get("ns/name") from Indexer ──▶ reconcile ──▶ writes back via apiserver
```

Three design decisions inside that picture, each of which you inherit:

1. **The queue holds keys (`namespace/name`), not objects.** So the worker always re-reads current cache state, and ten enqueues of the same key collapse into one unit of work. The queue's `dirty`/`processing` sets guarantee an item is never handed to two workers at once: `Add` on an item that is currently processing marks it dirty but does not enqueue; `Done` re-pushes it only if it went dirty while you held it.
2. **The queue is rate-limited.** client-go's `DefaultTypedControllerRateLimiter` is a `MaxOf` combination of a per-item exponential limiter (`baseDelay 5 ms`, `maxDelay 1000 s`, delay = `baseDelay × 2^failures`) and a global token bucket (`10 qps`, burst `100`). A failing reconcile backs off per key; a storm of distinct keys is capped globally.
3. **It is level-triggered.** The handler re-reads current state and drives toward the desired state. A missed, duplicated, or reordered event is harmless because nothing depends on having seen a specific event. Lesson 03 is entirely about why that is the only correct model, and lesson 04 opens the informer machinery.

**Leader election.** In HA all replicas start, but only one reconciles. They contend for a `coordination.k8s.io/v1` `Lease` in `kube-system` (`kube-controller-manager`, `kube-scheduler`). The defaults define the whole safety argument:

| Flag | Default | Meaning |
|---|---|---|
| `--leader-elect` | `true` | run the election at all |
| `--leader-elect-lease-duration` | `15s` | how long a non-leader waits after the last observed renewal before trying to take over |
| `--leader-elect-renew-deadline` | `10s` | the leader must successfully renew within this or it **stops leading itself** |
| `--leader-elect-retry-period` | `2s` | how often candidates poll/renew |

The invariant is `renewDeadline < leaseDuration`: the incumbent gives up leadership at 10 s of failure, and a challenger only takes over at 15 s, leaving a 5 s guard band for clock skew and API latency. That is what prevents split-brain — two controller-managers both scaling the same Deployment, or both deleting the same pod. Note also that a leader can lose leadership without crashing: a long GC pause or an apiserver 429 storm that blocks lease renewal for 10 s is enough. This is exactly why `leader-election` has its own APF priority level.

Your operator needs the identical mechanism. controller-runtime gives it to you with `LeaderElection: true` and `LeaderElectionID`, and it gates every reconciler on holding the lease.

**Garbage collector.** The GC controller builds a graph from `metadata.ownerReferences` and deletes objects whose owners are all gone. `ownerReferences` entries carry `apiVersion`, `kind`, `name`, `uid`, and optionally `controller: true` (at most one per object) and `blockOwnerDeletion: true`. Cross-namespace owner references are disallowed by design; an invalid one produces an event with reason `OwnerRefInvalidNamespace` and the dependent is never collected. Deleting an owner offers three propagation policies: **Background** (default — owner deleted immediately, dependents cleaned up asynchronously), **Foreground** (a `foregroundDeletion` finalizer is added to the owner and it stays visible in a deleting state until dependents with `blockOwnerDeletion: true` are gone), and **Orphan** (an `orphan` finalizer; dependents survive with the owner reference stripped). Lesson 06 builds on this; lesson 03 covers the finalizer mechanics.

### cloud-controller-manager

Cloud-specific controllers used to live inside KCM, which forced every cloud SDK into the core binary. CCM split them into an out-of-tree binary implementing `cloudprovider.Interface`:

- **node controller** — initialises new Node objects with provider labels (`topology.kubernetes.io/region`, `topology.kubernetes.io/zone`, `node.kubernetes.io/instance-type`) and deletes Node objects whose backing instance is gone. Those labels are exactly the join keys your cost model needs: instance type → price, zone → transfer cost.
- **route controller** — programs VPC routes for pod CIDRs where the CNI does not.
- **service controller** — reconciles `type: LoadBalancer` Services into cloud load balancers.

Same informer + workqueue + leader-election pattern; the side effects are cloud API calls instead of Kubernetes writes. Modern clusters run kubelet and KCM with `--cloud-provider=external`, deferring to CCM. The split is a real security boundary, not a refactor: cloud IAM credentials live in CCM only, so an over-permissioned or compromised kube-controller-manager does not imply compromised cloud IAM.

### kubelet

The node agent. Its heart is `syncLoop` — a single `select` over several channels, each of which can wake a pod's sync.

```
  syncLoop select {

    configCh ◀── desired pod set. Three sources, merged by the PodConfig mux:
    │             • apiserver watch, field-selected spec.nodeName == this node
    │             • static pod files in --pod-manifest-path (/etc/kubernetes/manifests)
    │             • HTTP endpoint (rare)
    │
    plegCh   ◀── PLEG: "a container in pod X started / died"
    │             generic PLEG relists ALL containers from the CRI every 1s
    │             and diffs against its last snapshot
    │
    syncCh   ◀── periodic full resync of every pod (syncFrequency, default 1m)
    │
    probeCh  ◀── liveness / readiness / startup probe results
    │
    housekeeping ◀── every 2s: clean up orphaned pods, volumes, cgroups
  }
        │
        ▼
   for each affected pod:
        canAdmitPod()  ─── local admission: does it actually fit? OS/arch ok?
        │                  no → pod status Rejected/OutOfcpu (NOT rescheduled by kubelet)
        ▼
        computePodActions()  ─── diff desired containers vs actual container statuses
        │                        → {kill these, create sandbox?, start these}
        ▼
        syncPod() ─── CRI calls: RunPodSandbox → (CNI) → PullImage → CreateContainer → StartContainer
        │
        ▼
        status manager ──▶ PATCH pod /status on the apiserver
```

**PLEG in detail, because it is the failure mode you will actually hit.** The kubelet must notice a container dying without anyone telling it. The *generic* PLEG solves this by polling: every relist period (1 s) it calls the CRI's `ListPodSandbox` + `ListContainers`, diffs the result against its previous snapshot, and emits `ContainerStarted` / `ContainerDied` / `ContainerRemoved` events onto `plegCh`. It also records the wall-clock time of the last successful relist.

The kubelet's `Healthy()` check compares "time since the last completed relist" against a threshold (3 minutes by default). Exceed it and the check fails with:

```
pleg was last seen active 3m21.4s ago; threshold is 3m0s
```

which propagates into the Node's `Ready` condition:

```
$ kubectl describe node gpu-a3-07
Conditions:
  Type     Status  LastHeartbeatTime  Reason              Message
  Ready    False   ...                KubeletNotReady     PLEG is not healthy: pleg was last
                                                          seen active 3m21.4s ago; threshold is 3m0s
```

**The node goes `NotReady` while the kubelet process is perfectly alive.** That is the whole point of distinguishing this failure. Relist is a synchronous call into the container runtime; if the runtime is wedged, slow, or saturated, relist blocks and PLEG "stalls." Common real causes on a GPU fleet:

1. The container runtime is overloaded or deadlocked — a multi-GB CUDA image pull saturating the containerd content store and its client goroutines, or a `runc` operation hung on a stuck device.
2. Very high container churn — a job scheduler cycling training pods rapidly means every relist has to enumerate and diff a large, constantly changing set.
3. Disk I/O saturation on the image filesystem, which makes every CRI call slow.

Distinguish them with `kubelet_pleg_relist_duration_seconds` (relist is running but slow → runtime/CRI problem) versus `kubelet_pleg_relist_interval_seconds` (relist is not running at all → the kubelet's own goroutine is stuck, which is rare and a different fix). And confirm the runtime independently: `crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps` should return promptly; if it hangs, you have your answer without touching the kubelet.

**Evented PLEG** (KEP-3386) replaces polling with a CRI event stream (`GetContainerEvents`) so the kubelet learns about state changes immediately and burns far less CPU on high-pod-count nodes. It has had a bumpy graduation — beta but **off by default** because it requires runtime support, then reverted to alpha in the 1.30 timeframe after a static-pod startup bug, with fixes still landing in the 1.3x line. Treat it as "know it exists, do not assume it is on."

**The CRI boundary.** The kubelet never runs a container itself. It speaks the **Container Runtime Interface** — a gRPC contract in `k8s.io/cri-api` — over a Unix socket set by `--container-runtime-endpoint` (commonly `unix:///run/containerd/containerd.sock`). Two services:

- `RuntimeService`: `Version`, `RunPodSandbox`, `StopPodSandbox`, `RemovePodSandbox`, `CreateContainer`, `StartContainer`, `StopContainer`, `ContainerStatus`, `ListContainers`, `ExecSync`/`Exec`/`Attach`/`PortForward`, `UpdateContainerResources`.
- `ImageService`: `PullImage`, `ListImages`, `ImageStatus`, `RemoveImage`, `ImageFsInfo`.

`--runtime-request-timeout` defaults to `2m` for everything except long-running calls (pull, logs, exec, attach). `crictl` speaks the same protocol, which is why it is the right debugging tool when you suspect the layer below the kubelet.

**Pod admission on the node.** The scheduler's decision is a proposal, not a command. Before syncing, the kubelet runs local admit handlers: does the pod actually fit in current allocatable, is the OS/arch right, are the requested sysctls allowed, is the topology satisfiable. A rejected pod gets a terminal status (`Rejected`, or the classic `OutOfcpu` / `OutOfmemory`) — and the kubelet does *not* reschedule it; something else has to notice. This exists because the kubelet has ground truth the scheduler's snapshot may lag by a beat.

**QoS, cgroups, and eviction.** The kubelet writes the cgroup hierarchy and assigns each pod a QoS class: `Guaranteed` (every container has requests == limits for both CPU and memory), `Burstable` (at least one request set, not all equal to limits), `BestEffort` (nothing set). `--kube-reserved`, `--system-reserved`, and `--enforce-node-allocatable` carve reservations out of node capacity, and `allocatable = capacity − kube-reserved − system-reserved − eviction-hard`. Under pressure the eviction manager ranks pods by QoS class and by usage-over-request and evicts. Default hard eviction thresholds: `memory.available < 100Mi`, `nodefs.available < 10%`, `nodefs.inodesFree < 5%`, `imagefs.available < 15%`. Default `maxPods` is `110`.

**Volume manager** runs its own reconcile loop (desired mounted volumes vs actually mounted volumes) and calls CSI to attach/mount. It is deliberately independent of `syncPod`, so a stuck mount blocks only that pod rather than freezing the node's whole sync loop.

**Static pods** are how the control plane bootstraps. The kubelet reads manifests from `--pod-manifest-path` (default `/etc/kubernetes/manifests`) with no apiserver involvement — which is how `kube-apiserver`, `etcd`, `kube-scheduler`, and `kube-controller-manager` can run as pods on a cluster whose apiserver is one of them. The kubelet creates a read-only **mirror pod** in the apiserver so they appear in `kubectl get pods -n kube-system`, but the file on disk is the source of truth: `kubectl delete` on a mirror pod does nothing durable.

**Where usage metrics come from — capstone-critical.** The kubelet embeds cAdvisor to read per-container cgroup counters and exposes three endpoints on its authenticated port (10250):

| Endpoint | Format | Contents | Consumer |
|---|---|---|---|
| `/metrics/cadvisor` | Prometheus text | full cAdvisor set: `container_cpu_usage_seconds_total`, `container_memory_working_set_bytes`, per-container, high cardinality | your Prometheus |
| `/metrics/resource` | Prometheus text | trimmed set designed for metrics-server | metrics-server |
| `/stats/summary` | JSON | Summary API: node + pod + container CPU/memory/fs, plus volume stats | metrics-server, some agents |
| `/metrics` | Prometheus text | the kubelet's *own* internals — `kubelet_pleg_*`, `kubelet_runtime_operations_*`, `kubelet_pod_start_duration_seconds` | your Prometheus |

**GPU utilisation is not in any of them.** cgroups do not measure SM occupancy. GPU utilisation comes from NVIDIA's DCGM exporter as a separate scrape, and joining it to pod identity (via the `pod`/`namespace`/`container` labels DCGM-exporter attaches from the device-plugin's pod-resources API) is a seam your operator has to build and defend. That join is where cost attribution actually succeeds or fails.

**Two heartbeat channels, and why they were split.** This is a favourite interview question because getting it wrong means misreading every node dashboard you own.

| Channel | Object | Cadence | Size | Who consumes it |
|---|---|---|---|---|
| Lease | `Lease` in `kube-node-lease` | renewed ~every 10 s | tens of bytes — just `renewTime` | node-lifecycle-controller in KCM |
| NodeStatus | `Node.status` | on change, else every `nodeStatusReportFrequency` = **5m** | multi-KB: capacity, allocatable, conditions, addresses, images | everything else |

The kubelet's `nodeStatusUpdateFrequency` defaults to `10s` — that is how often it *computes* status and renews the lease; `nodeLeaseDurationSeconds` defaults to `40`, and the kubelet renews at a fraction of that. On the controller side, `--node-monitor-period` is `5s` (how often KCM checks) and `--node-monitor-grace-period` is `50s` (how long a node may be silent before it is marked unhealthy). Do the arithmetic: 50 s of grace against a 10 s renewal interval gives you five missed renewals before a node is condemned — deliberate slack for a GC pause or a brief apiserver hiccup.

They were split because at a few thousand nodes, rewriting a multi-KB Node status object every 10 seconds *just to say "I'm alive"* was generating enough write traffic and etcd revisions to hurt the whole control plane. **A node whose lease is renewing normally while its `status` timestamps look "stale" is healthy, not broken.**

### kube-proxy

kube-proxy turns Service objects into node-local datapath rules. It watches Services and EndpointSlices and programs the kernel; it is never in the data path itself.

| Mode | Data structure | Update cost | Lookup cost | Notes |
|---|---|---|---|---|
| `iptables` | chains of DNAT rules, backend chosen by `statistic --probability` | O(n) — full `iptables-restore` of a large ruleset | O(n) chain traversal | still the **default** |
| `ipvs` | kernel IPVS hash table + a small iptables set | O(1) per Service | O(1) | real LB algorithms (`rr`, `lc`, `sh`) |
| `nftables` | nftables verdict maps | O(1)-ish incremental | O(1) map lookup | **GA in v1.33**; fixes the iptables scaling wall but is *not* the default — opt in with `--proxy-mode nftables` |
| eBPF (Cilium etc.) | eBPF maps at socket/tc/XDP | incremental | O(1) | replaces kube-proxy entirely; where AI-native clusters trend |

`--iptables-sync-period` defaults to `30s` with `--iptables-min-sync-period` `1s`, so rule programming is batched rather than per-event.

**EndpointSlices, not Endpoints.** The datapath is driven by `discovery.k8s.io/v1` EndpointSlices, each holding at most `--max-endpoints-per-slice` (default 100) endpoints. The old single `Endpoints` object meant a 5,000-pod Service rewrote one enormous object on every pod readiness flip — an O(n) write for an O(1) change, multiplied by every watcher. Sharding turns that into a small write to one slice. Readiness (from kubelet probes) is what flips an endpoint's `conditions.ready`, which is the whole chain from "probe failed" to "traffic stops."

**conntrack.** iptables and IPVS DNAT both depend on the kernel connection-tracking table to keep a flow pinned to the backend it was first translated to. kube-proxy sizes it from `--conntrack-max-per-core` (default 32768) with a floor of `--conntrack-min` (default 131072); established TCP entries idle out after `--conntrack-tcp-timeout-established` (24h) and CLOSE_WAIT after 1h. Two classic failures: the table fills (`nf_conntrack: table full, dropping packet` in dmesg, connections silently dropped), and stale **UDP** entries survive after a backend dies, blackholing DNS until the entry expires. eBPF datapaths largely sidestep both by not relying on conntrack for service translation.

### CoreDNS

CoreDNS is itself a controller. Its `kubernetes` plugin runs informers over Services and EndpointSlices, builds an in-memory map, and answers from it — so a newly created Service is resolvable within one watch round-trip, with no zone file and no reload.

- `A`/`AAAA` for `<svc>.<ns>.svc.cluster.local` → ClusterIP. Headless Services (`clusterIP: None`) return one record per ready endpoint straight from the slices.
- `SRV` for named ports (`_http._tcp.<svc>.<ns>.svc.cluster.local`), `PTR` for reverse.
- Configuration is the **Corefile**, a plugin chain: `errors`, `health`, `ready`, `kubernetes`, `prometheus`, `forward . /etc/resolv.conf`, `cache 30`, `loop`, `reload`, `loadbalance`.

**The `ndots:5` amplification, with real numbers.** A pod's `/etc/resolv.conf` carries `options ndots:5` and a search list (`<ns>.svc.cluster.local`, `svc.cluster.local`, `cluster.local`, plus the node's domains). Any name with fewer than 5 dots is tried against every search suffix *before* being tried as written. `api.stripe.com` has two dots, so it walks the whole search list first. Zalando's January 2019 postmortem quantifies it: **10 DNS queries per external hostname lookup**, because each suffix is tried for both A and AAAA.

That incident is the canonical worked example of this whole section. On 7 January 2019 a non-critical downstream service began timing out; the aggregation layer returned 404s; clients retried; the retry storm multiplied requests, and because the Node.js app did no DNS caching and used short-lived connections, each retry became a fresh resolution — amplified 10× by `ndots:5`. CoreDNS pods hit their **100Mi memory limit**, were OOM-killed, and crash-looped. With DNS gone the entire cluster's service discovery failed, and — critically — the monitoring stack failed too, delaying detection. Recovery came from manually raising the limit to **2000Mi**. Total customer-facing outage: over an hour of errors on all Zalando Fashion Store product and outfit pages. Action items: node-local DNS caching, external (out-of-cluster) monitoring, and simplified escalation.

The mitigations that fall out of that: node-local DNS cache (removes both the CoreDNS round trip and the conntrack entry per query), fully-qualified external names with a trailing dot (`api.stripe.com.` skips the search list entirely), a tuned `ndots` in `dnsConfig` for pods that mostly resolve external names, and — obviously — a CoreDNS memory limit sized against measured QPS with an autoscaler behind it.

### Container runtime via CRI

```
 kubelet ──CRI gRPC over unix socket──▶ containerd
                                          ├─ ImageService  ─▶ content store + snapshotter
                                          │                   (overlayfs default; stargz/nydus
                                          │                    for lazy pull)
                                          └─ RuntimeService ─▶ containerd-shim-runc-v2  (one per pod)
                                                                    │
                                                                    ▼
                                                                  runc
                                                                    │
                                            ┌───────────────────────┴────────────────────┐
                                            │ namespaces (pid/net/ipc/mnt/uts)           │
                                            │ cgroups v2 (cpu.max, memory.max, pids.max) │
                                            │ device cgroup: /dev/nvidia0, /dev/nvidiactl│
                                            │ bind mounts: CUDA driver libs (NVIDIA hook)│
                                            └────────────────────────────────────────────┘
```

Ordering matters and explains a symptom you have certainly seen:

1. `RunPodSandbox` creates the **pause** container, which owns the pod's network/IPC/UTS namespaces and does nothing but reap zombies.
2. The runtime invokes **CNI**, which assigns the pod IP *into that sandbox's netns*.
3. Only then are app containers created and joined to the sandbox's namespaces.

So a CNI failure leaves you with a live sandbox, no pod IP, and no app containers: `ContainerCreating` forever, with `FailedCreatePodSandBox` events. The pod is not "stuck pulling"; it never got to step 3.

The **shim** (`containerd-shim-runc-v2`, one process per pod) is what decouples container lifetime from daemon lifetime: it is the parent of the container processes and holds their I/O, so `systemctl restart containerd` does not kill running pods. For GPU pods, the NVIDIA container runtime inserts a hook at OCI-spec creation time that adds `/dev/nvidia*` device nodes to the device cgroup and bind-mounts the driver user-space libraries — that step is the literal boundary between "the scheduler decided" and "CUDA can see a GPU."

**Image pull dominates GPU pod startup.** A CUDA base image plus a framework is routinely 5–15 GB. Lazy-pulling snapshotters (stargz, nydus) let a container start before the whole image is local by fetching chunks on demand, which is the difference between a 4-minute and a 20-second time-to-running on a cold node. On a fleet where a node costs dollars per GPU-hour, that difference *is* the optimisation.

## Perspectives

**Developer/extender perspective.** Every component you will touch while building the operator — an informer, a webhook, a scheduler plugin — is itself just another apiserver client. There is no back door, no shared memory, no privileged internal API. Writing your operator means internalising "no shared state, only watch," the same constraint kube-controller-manager lives under. The upside is that everything you learn about debugging kube-controller-manager transfers directly to debugging your own code.

**Operator/SRE perspective.** The two heartbeat channels, and metrics like `workqueue_depth`, `etcd_disk_wal_fsync_duration_seconds`, `apiserver_flowcontrol_rejected_requests_total`, and `kubelet_pleg_relist_duration_seconds`, are what you actually page on. A CKA already knows `kubectl get nodes`; what is new is knowing *which* metric explains *why* a node flapped — a PLEG stall, a missed lease renewal, and a genuinely dead kubelet produce the same `NotReady` on the surface and need three different fixes.

**Hardware/kernel perspective.** CRI → shim → runc → cgroups/namespaces → `/dev/nvidia*` device-cgroup injection is the literal boundary between "the control plane decided" and "the GPU is usable inside the container." PLEG's 1-second poll of the CRI socket is a userspace polling loop over what is ultimately kernel process and cgroup state; etcd's commit latency is an `fsync` on a physical device. The abstraction stack from "Kubernetes object" to "electrons on a GPU" runs straight through both.

**Economics/FinOps perspective.** Every second between "GPU node Ready" and "workload running" is idle GPU-hours at whatever the instance costs. Watch-cache staleness, PLEG stalls, slow etcd disk, and a `LeastAllocated` scoring strategy all show up as schedule latency or as under-packed nodes — which are cost metrics on a GPU fleet, not just reliability ones. This is the thread tying module 01's cost exporter to module 02's internals: the internals in this lesson are where the money leaks when they misbehave.

## Real-world use cases

- **OpenAI, "Scaling Kubernetes to 2,500 nodes."** The etcd disk-latency lesson with numbers: network-attached Azure P30 SSDs rated 5,000 IOPS, of which etcd could use ~10% because its writes are sequential and latency-bound at ~2 ms each; moving the data directory to a locally-attached SSD cut write latency to **200 µs**. They subsequently hit etcd's storage quota and had to raise `--quota-backend-bytes`. Shows why "the control plane feels slow" so often resolves to one disk.
- **OpenAI, "Scaling Kubernetes to 7,500 nodes."** The later post: **5 apiservers and 5 etcd members** on dedicated nodes, Kubernetes **Events split into their own etcd cluster** (Events are the highest-churn object type and were driving DB growth), up to **70 GB apiserver heap** at that scale, and alerting on the *rate of 429 and 5xx from the apiservers* as the top-level control-plane health signal. Every one of those is a direct consequence of the "apiserver is the only etcd client, and it serves everything from the watch cache" architecture in this lesson.
- **Zalando, "Total DNS outage in Kubernetes cluster" (7 Jan 2019).** CoreDNS pods with a **100Mi** memory limit OOM-killed under a retry storm that was amplified **10×** per external lookup by `ndots:5` and an application that did no DNS caching. Over an hour of customer-facing errors; monitoring failed along with DNS, delaying detection; fixed in the moment by raising the limit to **2000Mi**, and structurally by node-local DNS caching and out-of-cluster monitoring. The concrete version of "CoreDNS is a controller with a memory budget."
- **Uber, "Migrating Uber's Compute Platform to Kubernetes."** A large-scale migration built on custom job controllers reading CRDs and custom scheduling, rather than on stock workload types — the "extending, not just operating" anchor for this whole module, and a preview of the reconcile shape lesson 03 formalises.

## Worked example

Trace `kubectl apply -f gpu-pod.yaml` to a running container with a latency budget at each hop, then do the arithmetic that tells you where optimisation is worth anything.

```yaml
# gpu-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: trainer-0
  namespace: ml
spec:
  restartPolicy: Never
  containers:
  - name: train
    image: nvcr.io/nvidia/pytorch:24.10-py3      # ~9 GB compressed
    command: ["python", "train.py"]
    resources:
      limits:
        nvidia.com/gpu: 8                        # device-plugin extended resource
        cpu: "64"
        memory: 400Gi
      requests:
        nvidia.com/gpu: 8
        cpu: "64"
        memory: 400Gi                            # requests == limits → QoS Guaranteed
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

| Hop | Internal work | Latency (healthy cluster) | Dominated by |
|---|---|---|---|
| 1. kubectl → apiserver | TLS, authN (client cert), APF seat, authZ (RBAC), decode + strict field validation, mutating admission (ServiceAccount, Priority, LimitRanger, any webhooks), defaulting + validation, validating admission (ResourceQuota, PodSecurity, any `ValidatingAdmissionPolicy`), managedFields | **1–5 ms** in-process; **+ RTT per webhook** (serial for mutating, parallel for validating) | webhook count and their latency |
| 2. apiserver → etcd | protobuf encode, `Txn(ModRevision==0) → Put`, Raft replicate, quorum `fsync` | **5–20 ms** on local NVMe; **100 ms+** on a network-attached volume | WAL `fsync` on the slowest quorum member |
| 3. etcd → watch cache → scheduler | one etcd event decoded once, fanned out to every watcher; pod enters activeQ; snapshot; PreFilter/Filter/PostFilter/PreScore/Score/Reserve; `POST /binding` | **1–2 ms** fan-out + **1–10 ms** cycle for a few hundred nodes | node count × enabled Filter/Score plugins |
| 4. apiserver → kubelet | kubelet's field-selected pod watch delivers on `configCh`; local admission (`canAdmitPod`); `computePodActions` | **1–2 ms** delivery + sub-ms admission | watch fan-out |
| 5. kubelet → CRI → containerd | `RunPodSandbox` (pause + CNI assigns pod IP + NVIDIA hook injects `/dev/nvidia*` into the device cgroup) → `PullImage` → `CreateContainer` → `StartContainer` | **seconds to minutes** — a 9 GB image on a 1 GB/s link is ~9 s at line rate and routinely 60–240 s in practice | **image pull**, then decompression and snapshotter layer assembly |
| 6. status back up | PLEG relist (≤1 s) sees `ContainerStarted` → wakes syncLoop → status manager PATCHes `/status` → new etcd revision → fan-out to anything watching (EndpointSlice controller, kube-proxy, CoreDNS if it backs a Service) | **≤1 s** PLEG poll + probe cadence | the 1 s relist period |

**The arithmetic.** Steps 1–4 — the entire control-plane path from `kubectl` to the kubelet having the pod in hand — total well under **100 ms** on a healthy cluster. Step 5 is seconds to minutes. So:

```
  control-plane path (steps 1–4)      ≈   0.05 s
  image pull + start (step 5, cold)   ≈  90–240 s
  ─────────────────────────────────────────────────
  time-to-running (cold node)         ≈  90–240 s
  control-plane share of that         ≈  0.02 % – 0.06 %
```

Now price it. An 8×H100 instance at an on-demand rate of roughly $30–$40/hour (varies hard by provider, region, and commitment — check your own bill, do not trust this number) burns about **$0.0083–$0.011 per second**. A 180-second cold start is **$1.50–$2.00 of idle GPU** per pod launch. On a cluster cycling 500 training pods a day that is **$750–$1,000/day**, or roughly **$275k–$365k a year**, spent on pulling images.

Shave 20 ms off the control-plane path and you save about $0.0002 per launch. Cut cold-start image pull from 180 s to 20 s with pre-pulled images or a lazy-pull snapshotter and you save **$1.30–$1.80 per launch** — four orders of magnitude more. **This is the number that should drive your operator's first efficiency recommendation**, and it is only visible if you understand which hop in the trace is actually slow.

The corollary matters too: control-plane latency is not worth optimising for cost, but it is worth optimising for *correctness*, because a slow etcd or a saturated APF level shows up as failed lease renewals, leadership flapping, and watch-cache staleness — none of which are cost problems until they become outages.

## Practice

Use a multi-node `kind` cluster (multi-node so leases and scheduling are non-trivial). Deliverables: a short **"internals observed"** note plus the Node-watch snippet — this snippet is the seed of your operator's cost signal, because it must react to nodes joining and leaving, which is where GPU capacity and spend enter and exit. This feeds the [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md) deliverable directly.

1. **Node-watch client-go program.** Write a tiny informer that logs Node add/update/delete and prints capacity — specifically any `nvidia.com/gpu`. `kind` has no real GPUs; fake it by patching a node's `status.capacity`, or use `kwok`.

   ```go
   package main

   import (
       "log"
       "time"

       v1 "k8s.io/api/core/v1"
       "k8s.io/client-go/informers"
       "k8s.io/client-go/kubernetes"
       "k8s.io/client-go/tools/cache"
       "k8s.io/client-go/tools/clientcmd"
   )

   func main() {
       cfg, err := clientcmd.BuildConfigFromFlags("", clientcmd.RecommendedHomeFile)
       if err != nil {
           log.Fatalf("kubeconfig: %v", err)
       }
       clientset, err := kubernetes.NewForConfig(cfg)
       if err != nil {
           log.Fatalf("clientset: %v", err)
       }

       // 30s resync = a local replay of every cached object, NOT a re-LIST.
       factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)
       nodeInformer := factory.Core().V1().Nodes().Informer()

       _, err = nodeInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
           AddFunc: func(obj interface{}) {
               n := obj.(*v1.Node)
               gpu := n.Status.Capacity["nvidia.com/gpu"]
               log.Printf("ADD node=%s rv=%s gpu=%s instance=%s",
                   n.Name, n.ResourceVersion, gpu.String(),
                   n.Labels["node.kubernetes.io/instance-type"])
           },
           UpdateFunc: func(oldObj, newObj interface{}) {
               o, n := oldObj.(*v1.Node), newObj.(*v1.Node)
               if o.ResourceVersion == n.ResourceVersion {
                   log.Printf("RESYNC node=%s rv=%s (no change)", n.Name, n.ResourceVersion)
                   return
               }
               log.Printf("UPD node=%s rv=%s→%s", n.Name, o.ResourceVersion, n.ResourceVersion)
           },
           DeleteFunc: func(obj interface{}) {
               if tomb, ok := obj.(cache.DeletedFinalStateUnknown); ok {
                   log.Printf("DEL (tombstone) key=%s", tomb.Key)
                   return
               }
               log.Printf("DEL node=%s", obj.(*v1.Node).Name)
           },
       })
       if err != nil {
           log.Fatalf("handler: %v", err)
       }

       stop := make(chan struct{})
       factory.Start(stop)
       if !cache.WaitForCacheSync(stop, nodeInformer.HasSynced) {
           log.Fatal("cache never synced — check RBAC list/watch on nodes")
       }
       log.Println("cache synced")
       <-stop
   }
   ```

   Two things to observe and write down: the `resourceVersion` on the initial synthetic `ADD` events versus subsequent watch events, and the `RESYNC` lines where old and new RV are identical — that is the resync replaying the local cache with nothing having changed, and it is the reason your reconcile must be idempotent (lesson 03).

2. **Component `/metrics`.** Pull each component's metrics and find one signal per component:

   ```
   kubectl get --raw /metrics | grep -E '^apiserver_(request_duration|flowcontrol_rejected)' | head
   kubectl get --raw /metrics | grep etcd_request_duration_seconds_bucket | head
   kubectl get --raw /api/v1/nodes/<node>/proxy/metrics | grep kubelet_pleg_relist_duration | head
   ```

   For the scheduler and controller-manager in `kind`, port-forward their static pods (`kubectl -n kube-system port-forward pod/kube-scheduler-<node> 10259:10259`) and scrape `https://localhost:10259/metrics` with your token. Find `scheduler_pending_pods`, `scheduler_scheduling_attempt_duration_seconds`, `workqueue_depth`, `workqueue_adds_total`, `leader_election_master_status`. Record which component each came from and what a rising value would mean.

3. **Scheduler framework config.** `docker exec` into the kind control-plane node, read `/etc/kubernetes/manifests/kube-scheduler.yaml` and any `KubeSchedulerConfiguration` it references. Identify the enabled Score plugins and whether `NodeResourcesFit` uses `LeastAllocated` or `MostAllocated`. Write one sentence on what that means for GPU node packing and therefore for spend.

4. **Leader-election and node leases.** 

   ```
   kubectl get lease -n kube-system
   kubectl get lease kube-controller-manager -n kube-system -o yaml
   kubectl get lease -n kube-node-lease
   kubectl get lease -n kube-node-lease <node> -o yaml --watch
   ```

   Watch `renewTime` advance about every 10 s on the node lease. Then `kubectl get node <node> -o yaml` twice a minute apart and confirm `status` is *not* changing at the same rate. Record both cadences and the `holderIdentity` on the KCM lease.

5. **Watch-cache and `410 Gone` (stretch).** With the Node watcher running, create and delete node objects (or scale a `kwok` node group) and observe how the informer behaves. Then compare a cached read against a consistent one and note the difference in latency:

   ```
   time kubectl get --raw '/api/v1/nodes?resourceVersion=0&limit=500'   # any version, from cache
   time kubectl get --raw '/api/v1/nodes?limit=500'                      # most recent
   ```

**Acceptance:** `internals-observed.md` (~15 lines) covering: the RV seen on initial sync vs on watch events and vs on resync; one metric per component with its meaning; the scheduler Score config and its packing implication; the KCM/scheduler/node lease names and their renew cadences; the two heartbeat cadences you measured — plus the committed Node-watch snippet.

## Common pitfalls

1. **"The scheduler talks to the kubelet."** It never does. It `POST`s a `Binding` subresource; the apiserver sets `spec.nodeName`; the kubelet learns about it through its own field-selected watch. Symptom of the misconception: debugging a `Pending` pod by looking at kubelet logs, when the pod has no `nodeName` and no kubelet has ever heard of it. Check `kubectl describe pod` events for `FailedScheduling` first — if that is what you see, the kubelet is not involved at all.

2. **Doing arithmetic on `resourceVersion`.** Diffing it, sorting it across resource types, or comparing it across clusters. It is etcd's global revision, shared by every object in the cluster; since v1.35 it is guaranteed orderable *within a resource type*, and that is the only ordering you may rely on. Only equality and "resume from here" are portable. Symptom: a controller that "skips" objects because it decided RV 900 must be older than RV 1200 on a different kind.

3. **Assuming NodeStatus is the heartbeat.** The Lease is. NodeStatus is throttled to change-only or every 5 minutes. A dashboard showing a 4-minute-old `lastHeartbeatTime` on a Node condition with a lease renewing every 10 s is a healthy node and a misleading dashboard. Mechanism: they were deliberately split so the cheap signal could be frequent and the expensive one rare.

4. **Reading `NotReady` as "the kubelet crashed."** A PLEG stall keeps the kubelet process alive and healthy while the node reports `NotReady` with `PLEG is not healthy` — because the check measures time-since-last-relist, and relist is a synchronous call into a wedged container runtime. Restarting the kubelet does nothing; you need `crictl` against the runtime. Distinguish with `kubelet_pleg_relist_duration_seconds` (slow) vs `kubelet_pleg_relist_interval_seconds` (not running).

5. **Treating a `410 Gone` / "too old resource version" as an incident.** It is the documented recovery path for a resume point that fell out of the ~5-minute history window, and informers handle it by relisting. Alert on watches that *stay* broken, or on relist rate as a load signal — never on the 410 itself. Alerting on it burns on-call attention on the protocol working correctly.

6. **Believing the CCM split-out was a refactor.** It is a credential boundary: cloud IAM lives in cloud-controller-manager, not kube-controller-manager. Scoping IAM as if KCM needed cloud permissions grants a much larger blast radius than the architecture requires.

7. **Running `etcdctl defrag` against the whole cluster at once.** Defrag blocks reads and writes on the member it runs against. All members at once is a full control-plane outage for the duration. Roll it one member at a time, and compact first — defrag only returns space that compaction already freed.

## Self-check

- **Walk `kubectl apply` to a running container end to end, naming what makes a GPU pod fail at each stage.**
  **Answer:** (1) *apiserver front half* — authN failure → `401`; RBAC denial → `403`; APF saturation → `429`; unknown field under strict validation → `400`. (2) *Admission* — a mutating webhook that is down with `failurePolicy: Fail` → `500 failed calling webhook`, and the pod is never created; a `ResourceQuota` or `PodSecurity` rejection → `403`; a schema/validation failure (including a CEL rule on a CRD) → `422`. (3) *etcd* — quorum loss means the transaction never commits, so the write hangs then fails `504`/`500`; a `NOSPACE` alarm makes etcd read-only and every write fails. (4) *Scheduler down or no feasible node* — the pod sits `Pending` with `FailedScheduling` events and no `spec.nodeName`; nothing on any node is involved. (5) *Kubelet dead or partitioned* — the pod has a `nodeName` but never syncs; the node eventually goes `NotReady` via lease expiry (~50 s grace) and the node-lifecycle controller taints it. (6) *Wedged container runtime* — `syncPod` blocks in `RunPodSandbox`/`CreateContainer`, PLEG relist stalls, and after 3 minutes the node goes `NotReady` with `PLEG is not healthy` while the kubelet itself is fine. (7) *CNI failure* — the sandbox exists but has no IP, so app containers never start and the pod sits in `ContainerCreating` with `FailedCreatePodSandBox`. (8) *GPU-specific* — the device plugin not running means `nvidia.com/gpu` never appears in `status.capacity`, so the pod is `Pending` on `Insufficient nvidia.com/gpu` even though the hardware is physically present.

- **Why does the watch cache exist, and what is the cost/consistency tradeoff between `resourceVersion=0` and a consistent LIST today?**
  **Answer:** It exists so the apiserver holds **one** etcd watch per resource type and serves every client watch — thousands of kubelets, controllers, and `kubectl -w` sessions — from an in-memory store plus a ring buffer of recent events, instead of driving one etcd stream per watcher. The ring buffer is dynamically sized between 100 and 102,400 events, targeting 75 s of history; a resume point older than the oldest entry produces `410 Gone: too old resource version: N (M)`. On the read side, `resourceVersion=0` means "any version" and is the cheapest possible read — straight from the store, possibly stale. A consistent LIST (no RV) historically forced a quorum read from etcd; since `ConsistentListFromCache` went on by default in **v1.31** it is served from the cache too, using etcd watch-progress notifications to prove the cache has caught up to etcd's current revision (fresh within ~110 ms in 99.9% of cases per the KEP). So the *consistency* gap has largely closed, but the *cost* gap has not: a LIST still serialises every matching object, so a controller should watch rather than poll, and use cached reads on the hot path regardless of where the server sources them.

- **What is the practical difference between the node Lease and NodeStatus, and why were they split?**
  **Answer:** The Lease in `kube-node-lease` is a tens-of-bytes object whose `renewTime` the kubelet bumps roughly every 10 s (`nodeStatusUpdateFrequency: 10s`, `nodeLeaseDurationSeconds: 40`); it is the liveness signal the node-lifecycle controller watches, with `--node-monitor-period 5s` and `--node-monitor-grace-period 50s` giving about five missed renewals of slack before a node is condemned. NodeStatus is a multi-kilobyte object — capacity, allocatable, conditions, addresses, image list — written only on change or every `nodeStatusReportFrequency: 5m`. They were split because rewriting the full status object every 10 s per node generated enough apiserver write traffic and etcd revisions to degrade large clusters. Operationally: a node with a fresh lease and a 4-minute-old status heartbeat is healthy.

- **Given `PLEG is not healthy` in the logs, what are three concrete production causes and how do you tell them apart?**
  **Answer:** The check fails when time since the last completed relist exceeds 3 minutes (`pleg was last seen active 3m21s ago; threshold is 3m0s`), and relist is a synchronous CRI call, so the cause is almost always below the kubelet. (1) **Overloaded or wedged container runtime** — a huge image pull saturating containerd, or a `runc` op hung on a stuck device (common with GPU driver problems). Test: `crictl ps` — if it hangs, that is your answer; check containerd logs and CPU. (2) **Very high container churn** — a job scheduler cycling pods fast enough that each 1-second relist has to enumerate and diff a large changing set. Test: pod churn rate and pod count against `maxPods` (default 110). (3) **Disk I/O saturation on the image filesystem** making every CRI call slow. Test: node disk utilisation and `kubelet_runtime_operations_duration_seconds` by operation type. Discriminator metric: `kubelet_pleg_relist_duration_seconds` rising means relist runs but is slow (runtime problem); `kubelet_pleg_relist_interval_seconds` blowing up means relist is not running at all (the kubelet's own goroutine is stuck — rare, and a kubelet restart is then the right move).

- **Your operator starts getting `429 Too Many Requests` from the apiserver. What is happening and what do you change?**
  **Answer:** APF classified your requests via a FlowSchema into a PriorityLevelConfiguration whose seat budget is exhausted, and that level is configured to reject rather than queue further. Total server concurrency is `--max-requests-inflight` (400) + `--max-mutating-requests-inflight` (200) = 600 seats, divided by nominal shares across levels; LIST requests are charged multiple seats based on estimated response size. First, respect `Retry-After` — client-go and the workqueue backoff already do, so a 429 storm usually means you are generating far more requests than you think. Then find the source: `apiserver_flowcontrol_rejected_requests_total` by flow schema and priority level tells you which of your calls is being throttled. The fix is almost never "raise the limits" — it is to stop LISTing where you should be watching, add a cached client instead of live reads, and cap `MaxConcurrentReconciles`. If your operator genuinely needs a guaranteed share, give it its own FlowSchema and priority level rather than letting it compete in `global-default`.

## Connections & what's next

This lesson is the machinery your capstone operator sits on top of. Leader election here becomes controller-runtime's `LeaderElection: true` in lesson 06. The watch cache, `resourceVersion`, and `410 Gone` become the reflector/DeltaFIFO/informer pipeline in lesson 04. The admission pipeline's two-phase ordering is the whole subject of lesson 08, and the `ValidatingAdmissionPolicy` alternative to webhooks shows up again in lesson 05's CRD design. The scheduler's device-plugin-vs-DRA split is the foundation of lesson 09. The economics thread — schedule latency and image-pull time as idle-GPU cost — is the same thread module 01's `gpu-cost-exporter` started and module 11 finishes.

Next: **[02.2 · API machinery](02-api-machinery.md)** takes "everything is a client of the apiserver" and goes one layer deeper — how an object is *addressed* (GVK vs GVR), typed, serialised, versioned, and patched underneath every client and CRD you will build.

## References & further reading

**Primary sources — Kubernetes docs**
- [Kubernetes cluster architecture](https://kubernetes.io/docs/concepts/architecture/) — the canonical component map; read for the exact responsibility split, especially CCM vs KCM.
- [Kubernetes API concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/) — the authoritative spec for `resourceVersion` semantics, the GET/LIST/WATCH resource-version tables reproduced above, bookmarks, streaming lists (`sendInitialEvents`), pagination, and the 410 relist contract.
- [Admission controllers reference](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) — the default plugin list and the statement that mutating runs first and validating webhooks run in parallel.
- [API Priority and Fairness](https://kubernetes.io/docs/concepts/cluster-administration/flow-control/) — default priority levels, seat accounting, and the flow-control metrics.
- [Scheduling framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/) — the authoritative extension-point contract and the serial-scheduling / concurrent-binding statement.
- [Scheduler configuration](https://kubernetes.io/docs/reference/scheduling/config/) — default-enabled plugins per extension point and `NodeResourcesFit` scoring strategies (`LeastAllocated` is the default).
- [kube-apiserver flags](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/), [kube-controller-manager flags](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/), [kube-proxy flags](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/), [kubelet config API](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/) — every default quoted in this lesson traces to one of these four.
- [Garbage collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/) — owner references, `blockOwnerDeletion`, and the foreground/background/orphan propagation policies.
- [Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/) — the `resource.k8s.io` object model (DeviceClass / ResourceClaim / ResourceClaimTemplate / ResourceSlice), GA in v1.34.

**Primary sources — source code**
- [`kubernetes/kubernetes`](https://github.com/kubernetes/kubernetes) — `staging/src/k8s.io/apiserver/pkg/server/config.go` (`DefaultBuildHandlerChain`, the filter order in the write-path diagram); `staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go` (ring-buffer capacities, `eventFreshDuration`, and the exact `too old resource version` error); `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go` (the optimistic `Txn`); `pkg/scheduler` (framework and queue); `pkg/kubelet/kubelet.go` (`syncLoop`) and `pkg/kubelet/pleg/generic.go` (relist and the health check).
- [`kubernetes/client-go`](https://github.com/kubernetes/client-go) — `util/workqueue/queue.go` for the `dirty`/`processing` dedup guarantee and `util/workqueue/default_rate_limiters.go` for the 5 ms→1000 s exponential limiter plus the 10 qps / burst 100 bucket.
- [KEP-2340, Consistent reads from cache](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/2340-Consistent-reads-from-cache/README.md) — the watch-progress-notification mechanism and the measured latency/CPU improvements quoted above.

**etcd**
- [etcd maintenance guide](https://etcd.io/docs/latest/op-guide/maintenance/) — compaction modes and retention, defragmentation's blocking behaviour, and the NOSPACE alarm recovery sequence.
- [etcd system limits](https://etcd.io/docs/latest/dev-guide/limit/) — the 1.5 MiB request limit and the 2 GiB default / 8 GiB suggested-maximum storage quota.

**Real-world engineering**
- OpenAI, ["Scaling Kubernetes to 2,500 nodes"](https://openai.com/index/scaling-kubernetes-to-2500-nodes/) — the etcd disk-latency investigation: 5,000-IOPS network SSD at ~2 ms per write vs 200 µs on local SSD, and hitting the storage quota.
- OpenAI, ["Scaling Kubernetes to 7,500 nodes"](https://openai.com/index/scaling-kubernetes-to-7500-nodes/) — 5 apiservers / 5 etcd members, Events on a separate etcd cluster, ~70 GB apiserver heap, and alerting on 429/5xx rates. (2023 vintage; cite as an engineering case study, not current infrastructure.)
- Zalando, ["Total DNS outage in Kubernetes cluster" postmortem](https://github.com/zalando-incubator/kubernetes-on-aws/blob/dev/docs/postmortems/jan-2019-dns-outage.md) — 7 Jan 2019: CoreDNS at a 100Mi limit OOM-killed under a retry storm amplified 10× by `ndots:5`; raised to 2000Mi to recover.
- Uber Engineering, ["Migrating Uber's Compute Platform to Kubernetes"](https://www.uber.com/blog/migrating-ubers-compute-platform-to-kubernetes-a-technical-journey/) — custom job controllers and scheduling at scale.

**Deeper dives**
- Red Hat Developer, ["Pod Lifecycle Event Generator: Understanding the 'PLEG is not healthy' issue"](https://developers.redhat.com/blog/2019/11/13/pod-lifecycle-event-generator-understanding-the-pleg-is-not-healthy-issue-in-kubernetes) — a vendor engineering walkthrough of exactly the relist-stall failure mode above.
- ["what-happens-when-k8s"](https://github.com/jamiehannaford/what-happens-when-k8s) — a community line-by-line trace of `kubectl apply`, complementary to this lesson's worked example.
- "Programming Kubernetes" (Hausenblas & Schimanski, O'Reilly) — the API-machinery and controller chapters; the concepts hold up even though the sample code has aged.
