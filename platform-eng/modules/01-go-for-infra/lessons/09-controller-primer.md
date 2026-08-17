---
lesson: "01.9"
title: "Controller primer — CRD, reconcile, envtest"
module: "01"
concept: "Controller primer"
status: not-started
est_time: "18h"
prev: "08-reading-unfamiliar-go.md"
next: null
artifacts: []
sources: 15
---

# 01.9 · Controller primer — CRD, reconcile, envtest

> **Concept.** Build a minimal but real controller: a CRD plus a reconcile loop that drives status toward spec, tested with envtest. The seed of the GPU cost/efficiency operator.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Where this fits

Lesson 08 gave you the skill of dropping into an unfamiliar codebase and finding the load-bearing code fast — and it spent that skill tracing exactly the path this lesson depends on: manager → shared cache → informer → `handler.EventHandler` → priority queue → worker goroutine → `Reconcile`. This lesson is where that trace stops being a reading exercise and becomes something you write. Everything from lessons 1–8 converges here into one artifact: a CRD, a `Reconciler`, and an envtest suite that proves it converges. Interfaces (3) give you the `CostSource` seam; errors (2) give you conflict classification; concurrency and `context` (4) are the manager's worker pool and its shutdown path; testing (5) is envtest; stdlib fluency (7) is the metrics endpoint the manager already serves for you. This is the module's flagship, final lesson. Passing it is passing the Go gate.

## Why this matters

Read the JDs you're targeting. CoreWeave, NVIDIA, Cisco-class GPU shops all say the same thing in different words: *"build and extend Kubernetes controllers, operators, and CRDs in Go."* That sentence is the job. Not "know what a CRD is" — you have a CKA, you know. The job is writing the Go machinery that watches cluster state and reconciles it, in a way that survives a busy API server, partial failures, and 40 clusters' worth of edge cases.

The cost of not knowing this cold is specific and checkable in an interview, and every item below is a real diagnosis you'd make on a real system:

- A candidate who reaches for `Result{Requeue: true}` signals stale knowledge — it has been deprecated since controller-runtime v0.20, and the maintainer's commit message says why in one sentence.
- A candidate who can't explain why `Status().Update` exists instead of `Update` hasn't shipped a controller, because the first time two writers touch one object the difference is the whole ballgame.
- A candidate who treats a conflict error on a status write as a bug rather than routine traffic will design retry logic that pages people at 3am for normal optimistic-concurrency contention.
- A candidate who assumes a `Get` immediately after an `Update` returns the value they just wrote will ship a controller that flaps, because controller-runtime's own package documentation says the split client makes no such promise.

None of this is exotic. It's the daily texture of the job description above.

## What's new here (calibration)

Per the [module README's skip-list](../README.md#calibrated-to-your-background--what-we-skip), this lesson never re-teaches: programming 101, pointers-as-concept, IDE/hello-world tours, OOP-in-Go, LeetCode/DSA grind, web-framework/CRUD tutorials, reflection/cgo/runtime internals, or cover-to-cover reading of the Kubernetes API. You already know what a CRD, a Deployment, and RBAC are operationally (CKA-level). What's genuinely new here, built on that base:

- The **mechanics** of how a watch event becomes a `Reconcile(ctx, req)` call — informer, cache, priority workqueue, worker pool — with the real defaults, not just "the controller watches things."
- The **reader/writer split** inside `client.Client`: reads come from an in-memory cache, writes go straight to the API server, and the gap between them is a real, documented staleness window you must design around.
- The **discipline** of idempotent, level-triggered design: why it's a hard constraint, not a style preference, and exactly which mechanism (`meta.SetStatusCondition`'s change detection) makes a repeated reconcile byte-identical.
- The **ownership boundary** between `.spec` and `.status` as an enforced API mechanism (the status subresource), not a convention you have to remember to respect — including what the API server actually does differently on the `/status` endpoint.
- The **operational gaps envtest doesn't cover** — and a correction: envtest *does* run the API server with RBAC enabled; what it doesn't do is give your test client a restricted identity, and it doesn't run the controller-manager at all, so garbage collection and namespace deletion silently don't happen.
- **Scale evidence**: what actually happens to this machinery at 100,000+ nodes, with real numbers from a hyperscaler, not hand-waving about "it should be fine."

Versions throughout: **kubebuilder v4.6**, **controller-runtime v0.24.1**, **controller-tools v0.21.0**, **Kubernetes 1.36**, **Go 1.24+**. Every default and signature below was read from that source.

## Core concepts

### Level-triggered, and what it costs you to get wrong

#### The problem

Imagine writing a controller edge-triggered — reacting to events. A user edits a `CostBudget`'s monthly budget from \$5,000 to \$8,000. Your handler receives "budget changed from 5000 to 8000" and adjusts an internal accumulator by the delta. Now enumerate what can go wrong:

- **Two edits coalesce.** The user edits twice in 100 ms. The workqueue dedupes by object key, so you get *one* wakeup for *two* edits. Your delta is wrong by one edit's worth, permanently.
- **A resync fires with no edit.** Every `SyncPeriod` (10 hours by default) the cache replays every object as a synthetic update. You apply a delta for a change that didn't happen.
- **The process restarts.** On startup the informer does a full LIST, and every object arrives as a Create event. Your accumulator starts from zero and re-applies everything.
- **An event is dropped.** A watch connection breaks; the informer re-lists and resyncs. Deltas between the last-seen and current state are never delivered as individual events, only as the current state.

Every one of those is normal, expected operation of the machinery — not a bug you can fix. So the framework refuses to hand you the delta at all.

#### The mechanism, made concrete

You traced this in Lesson 08, and it is three lines of `pkg/handler/enqueue.go`:

```go
item := reconcile.Request{NamespacedName: types.NamespacedName{
	Name:      evt.Object.GetName(),
	Namespace: evt.Object.GetNamespace(),
}}
```

The object is discarded. `reconcile.Request` has exactly one field — an embedded `types.NamespacedName` — and there is nowhere to put an event payload even if the framework wanted to give you one. **Level-triggered is not a design philosophy you must remember to honour; it is a data structure with no room for the alternative.**

controller-runtime's own package documentation states the contract directly:

> "Reconciliation is level-based, meaning action isn't driven off changes in individual Events, but instead is driven by actual cluster state read from the apiserver or a local cache. For example if responding to a Pod Delete Event, the Request won't contain that a Pod was deleted, instead the reconcile function observes this when reading the cluster state and seeing the Pod as missing."

So `Reconcile` is, mechanically, **a pure function over cluster state**: read the object, observe the world, compute the target, write it, return. It has no memory between calls. Every controller bug of the "it works until it doesn't" family is a violation of that — code that remembers something instead of re-deriving it.

#### Idempotency — and what it does *not* mean

Because reconcile runs an unknown number of times for the same state, every reconcile must be safe to repeat. This is **not** the same claim as "no side effects." A reconcile that creates, updates, or deletes objects is completely normal and idempotent *if* running it N times on unchanged inputs produces the same end state.

| Pattern | Idempotent? | Why |
|---|---|---|
| `Get` child; if `IsNotFound`, `Create` it | Yes | Second run finds it, does nothing. |
| `Create` child, treat `IsAlreadyExists` as success | Yes | Same end state, one fewer round trip on the happy path. |
| `Create` child, return the error unconditionally | No | Second run returns `AlreadyExists` forever; the object never reaches a stable state and the item backs off to the 1000 s cap. |
| `status.count++` | **No** | Increments on every replay. This is the archetypal bug. |
| `status.count = len(list)` | Yes | Recomputed from current inputs. |
| `append(status.conditions, cond)` | **No** | Grows without bound. Use `meta.SetStatusCondition`. |
| `Apply` a server-side-apply configuration | Yes | SSA is declarative by construction; `client.Client.Apply` exists in controller-runtime v0.24+ for exactly this. |

The controller-runtime FAQ states the rule at maintainer level, in answer to "How do I have different logic in my reconciler for different types of events?": you should not — *"Reconcile functions should be idempotent, and should always reconcile state by reading all the state it needs, then writing updates."* That single sentence is this lesson's technical spine.

### The machinery, with its real defaults

You never poll the API server. The manager wires this for you:

```
  API SERVER                       YOUR OPERATOR PROCESS
  ──────────                       ─────────────────────

                    ┌──────────────────────────────────────────────────────┐
                    │  manager.Manager                                     │
                    │    · owns ONE shared cache for the whole process     │
                    │    · owns the client, the scheme, the RESTMapper     │
                    │    · runs leader election (lease 15s / renew 10s     │
                    │      / retry 2s), /healthz, /readyz, /metrics        │
                    │    · graceful shutdown: 30s default                  │
                    └──────────────────────────────────────────────────────┘
                              │
   ┌──── LIST + WATCH ────────┤   ONE informer per Kind per process,
   │     (one per Kind,       │   NOT one per controller
   │      not per controller) ▼
   │                    ┌─────────────────────────────────────────┐
   │                    │  pkg/cache → client-go SharedIndexInformer│
   │◀───────────────────│    Reflector → DeltaFIFO → Indexer/Store │
   │                    │    resync (SyncPeriod) default 10h ±10%  │
   │                    │    WaitForCacheSync gates EVERYTHING     │
   │                    │    CacheSyncTimeout default 2m           │
   │                    └─────────────────────────────────────────┘
   │                              │ add / update / delete
   │                              ▼
   │                    ┌─────────────────────────────────────────┐
   │                    │  predicates (optional filter)            │
   │                    │  handler.EventHandler                    │
   │                    │    For()  → EnqueueRequestForObject      │
   │                    │    Owns() → EnqueueRequestForOwner       │
   │                    │             (+ OnlyControllerOwner)      │
   │                    │    ── emits Request{Namespace,Name} ──   │
   │                    │    initial LIST      → Priority = -100   │
   │                    │    old.RV == new.RV  → Priority = -100   │
   │                    └─────────────────────────────────────────┘
   │                              │
   │                              ▼
   │                    ┌─────────────────────────────────────────┐
   │                    │  priorityqueue (default since v0.21)     │
   │                    │    dedupe by key                         │
   │                    │    per-ITEM exponential backoff          │
   │                    │      5ms → ×2 → cap 1000s (~16.7 min)    │
   │                    └─────────────────────────────────────────┘
   │                              │  GetWithPriority()
   │                              ▼
   │                    ┌─────────────────────────────────────────┐
   │                    │  N workers, N = MaxConcurrentReconciles  │
   │                    │    DEFAULT 1                             │
   │                    │    same key is NEVER reconciled          │
   │                    │    concurrently, at any N                │
   │                    └─────────────────────────────────────────┘
   │                              │
   │                              ▼
   │                    ┌─────────────────────────────────────────┐
   │  cached reads ◀────│  YOUR Reconcile(ctx, req)                │
   │  (no API hit)      │    r.Get / r.List  → CACHE               │
   │                    │    r.Update / r.Status().Update → API ───┼───┐
   │                    └─────────────────────────────────────────┘   │
   │                              │                                    │
   │                              ├─ error, not Terminal ──▶ requeue, rate-limited
   │                              ├─ error IS TerminalError ▶ DROP (metric only)
   │                              ├─ Result{RequeueAfter: d} ▶ Forget, requeue after d
   │                              └─ Result{}, nil ─────────▶ Forget (backoff reset)
   │                                                                    │
   └────────────────────────── writes go straight here ◀────────────────┘
                               (the cache is NOT updated synchronously)
```

The defaults you must be able to state, all read from controller-runtime v0.24.1:

| Setting | Default | Where | Consequence of the default |
|---|---|---|---|
| `MaxConcurrentReconciles` | **1** | `pkg/controller/controller.go` | One slow reconcile blocks every other object of that Kind. Raise it deliberately. |
| Cache `SyncPeriod` | **10 hours**, ±10% jitter | `pkg/cache/cache.go` | Your safety net against a missed event is ten hours wide. For anything time-sensitive, return `RequeueAfter` instead of shortening this. |
| `CacheSyncTimeout` | **2 minutes** | `pkg/controller/controller.go` | A controller whose informer can't sync (RBAC, missing CRD) fails to start after two minutes. |
| Rate limiter (priority queue) | per-item exponential, **5 ms → 1000 s** | `pkg/controller/controller.go:254` | A permanently failing object retries ~every 16.7 minutes at steady state, without starving others. |
| Rate limiter (legacy queue) | above, **plus** a 10 QPS / burst-100 token bucket | `client-go` `default_rate_limiters.go` | The global bucket is why the *legacy* queue could throttle unrelated objects. The priority queue drops it. |
| Leader election lease / renew / retry | **15 s / 10 s / 2 s** | `pkg/manager/internal.go` | Worst-case failover is roughly the lease duration after a leader dies. |
| Graceful shutdown period | **30 s** | `pkg/manager/internal.go` | On SIGTERM the manager cancels `ctx` and waits up to 30 s for in-flight reconciles. Your `ctx` handling determines whether that's clean. |
| Health / ready endpoints | **`/healthz` / `/readyz`** | `pkg/manager/internal.go` | Only served if `HealthProbeBindAddress` is set. |

This isn't a diagram drawn for pedagogy — it's load-bearing at real scale. Google's writeup on [building a 130,000-node GKE cluster](https://cloud.google.com/blog/products/containers-kubernetes/how-we-built-a-130000-node-gke-cluster) (Nov 2025 — a dated snapshot; it was an experimental cluster at double the then-supported 65,000-node limit) reports scaling to 130,000 Pods in about 3 minutes 40 seconds at a sustained ~1,000 Pods/second, with two specific API-machinery features doing the heavy lifting: **KEP-2340 Consistent Reads from Cache** (serve strongly-consistent reads from the API server's memory) and **KEP-4988 Snapshottable API Server Cache** (serve LIST for a previous state from the watch cache). Plus a Spanner-backed store handling ~13,000 QPS for lease updates alone. The diagram above is the same cache-first design, at a scale where "just read from the API server" falls over.

### The reader client and the writer client

This is the part most tutorials skip, and it is the source of the most confusing class of controller bug.

#### What `mgr.GetClient()` actually is

It is a **split client**. Reads and writes take different paths:

| Operation | Path | Latency | Consistency |
|---|---|---|---|
| `Get`, `List` | The manager's in-memory cache (unless the type is in `Cache.DisableFor`, or is `Unstructured` with `Cache.Unstructured` false) | microseconds, no network | **Eventually** consistent. Reflects whatever the watch stream has delivered so far. |
| `Create`, `Update`, `Patch`, `Delete`, `Apply`, `DeleteAllOf` | Straight to the API server over HTTPS | a network round trip | Linearizable at the API server. |
| `Status().Update`, `Status().Patch` | Straight to the API server, `/status` subresource | a network round trip | Same. |
| `mgr.GetAPIReader().Get/List` | Straight to the API server, **bypassing the cache** | a network round trip | Strongly consistent read. |

You can see the branch in `pkg/client/client.go`:

```go
func (c *client) Get(ctx context.Context, key ObjectKey, obj Object, opts ...GetOption) error {
	if isUncached, err := c.shouldBypassCache(obj); err != nil {
		return err
	} else if !isUncached {
		return c.cache.Get(ctx, key, obj, opts...)   // ← the normal path
	}
	// ... live lookup ...
}
```

#### The staleness window

Writes do **not** invalidate the cache. The only thing that updates the cache is the watch stream, which means the sequence after a write is:

```
  t=0ms    r.Status().Update(ctx, &cb)
             │
             ├── HTTPS POST /apis/cost.example.com/v1alpha1/.../status
             │
  t≈2ms      ├── API server persists it, resourceVersion 4711 → 4712
             │        │
             │        └── etcd watch → API server watch cache
             │                 │
             │                 └── WATCH event on the wire ────┐
             │                                                 │
  t≈2ms      └── Update() RETURNS nil. Your code continues.     │
                   ▲                                            │
                   │  ◀══ THE STALENESS WINDOW ══▶              │
                   │   Anything you r.Get() in here             │
                   │   still returns resourceVersion 4711.      │
                   │                                            │
  t≈3–15ms   Reflector receives the event ◀─────────────────────┘
             DeltaFIFO → Indexer/Store updated to 4712
             informer fires → handler → Request enqueued
                   │
  t≈15ms+    Your Reconcile runs again, now sees 4712
```

The window is normally single-digit milliseconds, but it is unbounded in principle: under API-server load, a slow watch, or a re-established watch connection after a network blip, it can be seconds. controller-runtime's package documentation says so in as many words:

> "The default split client does not promise to invalidate the cache during writes (nor does it promise sequential create/get coherence), and code should not assume a get immediately following a create/update will return the updated resource."

#### What goes wrong, and what to do

The bug this produces is a **flapping controller**. Reconcile writes status, returns success, is immediately re-enqueued by its own write's watch event, reads a stale object, recomputes the same status, writes it again... or worse, sees a stale `.status` and concludes it hasn't done the work yet, so it does the work again. A controller that creates an external resource on that path creates two.

Four responses, in order of preference:

1. **Don't read back what you just wrote.** The object you passed to `Update` is mutated in place with the server's response, including the new `resourceVersion`. Use that in-memory value for the rest of the reconcile; you already have the truth.
2. **Make the operation idempotent by name.** The controller-runtime FAQ's first sanctioned pattern for cache staleness: create objects with **deterministic names** derived from the parent, so a duplicate create fails with `AlreadyExists` and is a harmless no-op. This is why you should almost never use `metadata.generateName` in a controller.
3. **Track and requeue.** The FAQ's second pattern, for cases where deterministic names aren't possible: record what you did, and if you don't observe it within a bounded time, requeue and re-check. The FAQ's framing is the one to internalise: *"write your controller with the assumption that information will eventually be correct, but may be slightly out of date."*
4. **Use `mgr.GetAPIReader()`** for a strongly consistent read. This is the escape hatch, and it is deliberately awkward to reach — every use is an uncached round trip to the API server, so a hot reconcile loop using it re-creates the load the cache exists to prevent. Reserve it for one-shot startup checks and leader-election-adjacent logic.

Kubernetes 1.36 (April 2026) began addressing this class of problem upstream. Under [KEP-5647](https://kubernetes.io/blog/2026/04/28/kubernetes-v1-36-staleness-mitigation-for-controllers/) (SIG API Machinery), `client-go`'s `Store` interface gained `LastStoreSyncResourceVersion()`, letting a controller introspect how far behind its own cache is; kube-controller-manager enables staleness mitigation by default for the DaemonSet, StatefulSet, ReplicaSet and Job controllers, gated per-controller by `StaleControllerConsistency` feature gates. New observability landed alongside: an alpha `stale_sync_skips_total` metric for syncs skipped due to a stale cache, and a `client-go` `store_resource_version` metric. Read that as confirmation that the staleness window is a first-class engineering problem, not an edge case — the Kubernetes project just shipped machinery for it.

### The `Reconcile` signature and `Result` semantics

```go
func (r *CostBudgetReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error)
```

`ctrl.Request` carries only `types.NamespacedName` — `{Namespace, Name}`. That's the entire input beyond the context.

`ctrl.Result` has three fields in controller-runtime v0.24.1:

| Field | Type | Meaning |
|---|---|---|
| `Requeue` | `bool` | **Deprecated.** Requeue via the rate limiter. |
| `RequeueAfter` | `time.Duration` | If > 0, requeue this key after the duration. Implies `Requeue`. |
| `Priority` | `*int` | Priority to use if the item is re-enqueued (including on error). If nil, the request's original priority is preserved. Only honoured on a priority queue — which is the default since v0.21. |

The return-value contract, documented on the `TypedReconciler` interface itself and implemented in `pkg/internal/controller/controller.go`'s `reconcileHandler`:

| You return | The controller does | Metric label |
|---|---|---|
| `ctrl.Result{}, nil` | `Queue.Forget(req)` — **resets the backoff counter**. No requeue; you wake again only on a watch event or resync. | `success` |
| `ctrl.Result{}, err` | Requeue with rate-limited exponential backoff. **The `Result` is ignored entirely.** | `error` |
| `ctrl.Result{RequeueAfter: d}, nil` | `Forget`, then requeue after exactly `d`. | `requeue_after` |
| `ctrl.Result{RequeueAfter: d}, err` | Requeue with backoff; `d` is **silently dropped**, and the framework logs a warning about it. | `error` |
| `ctrl.Result{}, reconcile.TerminalError(err)` | **No requeue at all.** Increments `TerminalReconcileErrors` and logs. | `error` |
| `ctrl.Result{Requeue: true}, nil` | Requeue via rate limiter. Deprecated; use one of the above. | `requeue` |

Three things follow that are worth memorising:

**`Forget` is not "drop."** It resets the item's per-item backoff counter to zero. If you return success and the object is immediately re-enqueued by a watch event, it starts at 5 ms again rather than continuing an escalating backoff. Missing this is how people convince themselves the backoff is broken.

**Returning an error and a `RequeueAfter` is a real, common mistake.** Someone hit it often enough that controller-runtime added a literal warning log for it: *"Warning: Reconciler returned both a result with either RequeueAfter or Requeue set and a non-nil error. RequeueAfter and Requeue will always be ignored if the error is non-nil."* If you see that in your logs, you have a control-flow bug.

**`TerminalError` is the "don't retry this" signal.** Wrap an error with `reconcile.TerminalError(err)` when retrying cannot possibly help — the spec is semantically invalid in a way schema validation didn't catch, a referenced external resource is permanently gone, a credential is structurally malformed. Without it, an unfixable object retries forever at the 1000 s cap, burning a queue slot and generating error metrics that mask real problems. This is the controller-shaped answer to Lesson 2's "retryable vs terminal" classification.

**Don't reach for `Requeue: true`.** It was deprecated in PR #3107 (commit `f15ff17`, merged 2025-02-24) with the maintainer's reasoning stated in the commit: *"There is no good reason to use this setting, either an error or `RequeueAfter` should be used instead. Deprecate it to avoid confusion."* An interviewer noticing it in new code is a real, checkable signal of stale knowledge.

The overall shape — asynchronous work-queue-driven sync loops, return errors immediately rather than blocking-retrying inline, sleep-and-recheck for long external operations rather than blocking the reconcile — is not folklore, it's documented field guidance. Google Cloud's [best practices for building Kubernetes Operators](https://cloud.google.com/blog/products/containers-kubernetes/best-practices-for-building-kubernetes-operators-and-stateful-apps) states it almost line for line, and adds one structural recommendation worth having in your pocket for a system-design interview: **compartmentalise responsibilities across multiple single-purpose controllers** rather than writing one controller that does everything. "Would you split this into two controllers?" is a real staff-level question.

### Spec, status, and what the subresource actually enforces

The user owns `.spec` — the intent. The controller owns `.status` — what it observed and computed. Reconcile reads spec, does work, writes status. It never writes its own spec, and the user never writes status. The [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) codify this as the canonical shape for every built-in and custom resource.

`+kubebuilder:subresource:status` turns the convention into an enforced mechanism. Concretely, at the API server:

| Without the subresource | With `subresource:status` |
|---|---|
| One endpoint: `PUT /apis/.../costbudgets/{name}` | Two: `.../costbudgets/{name}` and `.../costbudgets/{name}/status` |
| A `PUT` to the main endpoint writes spec **and** status together | A `PUT` to the main endpoint **ignores** any status in the body; a `PUT` to `/status` **ignores** any spec |
| One RBAC resource: `costbudgets` | Two: `costbudgets` and `costbudgets/status`, grantable independently |
| `metadata.generation` semantics are muddied | `metadata.generation` increments **only** on spec changes; status writes never bump it |

That last row is what makes the `observedGeneration` idiom work at all, and it's why it deserves its own section below.

The practical failure it prevents: without the split, your controller reads an object, computes status, and `Update`s it — sending the *whole object* including the spec it read. If a user edited the spec in the window between your read and your write, you have just reverted their edit. With the split, `Status().Update` physically cannot carry a spec change, so the two writers cannot fight. A shared struct sent to the wrong call (`Update` instead of `Status().Update`) compiles, works in a quiet cluster, and breaks exactly when concurrency appears.

### Conflicts are routine traffic

Both `Update` and `Status().Update` can return a conflict (`apierrors.IsConflict(err)` — HTTP 409) when the `resourceVersion` in your object no longer matches the server's. This is optimistic concurrency working as designed.

At any real concurrency — more than one controller touching overlapping objects, a human running `kubectl edit`, or your own controller with `MaxConcurrentReconciles > 1` across objects that share a dependency — conflicts are **routine, expected traffic**, not a corner case. The correct response is to let the queue replay you with a fresh read. Two ways to spell it:

```go
if err := r.Status().Update(ctx, &cb); err != nil {
    // Option A: return the error. Backoff requeue, fresh read, correct by construction.
    // Slightly noisy: it increments controller_runtime_reconcile_errors_total.
    return ctrl.Result{}, err
}
```

```go
if err := r.Status().Update(ctx, &cb); err != nil {
    if apierrors.IsConflict(err) {
        // Option B: treat as benign and requeue promptly. Keeps the error metric
        // meaningful, but note this is NOT "return nil" — returning a bare
        // ctrl.Result{}, nil relies on a watch event arriving to wake you, which
        // is normally true (your write generated one) but is not guaranteed.
        return ctrl.Result{RequeueAfter: time.Second}, nil
    }
    return ctrl.Result{}, err
}
```

Option A is the safe default; reach for B when conflict rate is high enough that the error metric stops being a useful alerting signal. What is *not* correct is treating a conflict as a hard failure worth paging on, or writing a manual retry loop inside `Reconcile` — the workqueue's backoff already is that retry loop, and an inner loop just holds a worker slot while doing it worse.

`controllerutil.CreateOrUpdate` and `CreateOrPatch` wrap the read-mutate-write cycle for child objects and return the conflict for you to propagate; `retry.RetryOnConflict` from `client-go` exists for non-controller code that has no queue behind it.

### Scaffolding: kubebuilder v4.6

```bash
# One-time: the CLI itself
go install sigs.k8s.io/kubebuilder/v4/cmd@latest

kubebuilder init --domain example.com --repo github.com/you/gpu-operator
kubebuilder create api --group cost --version v1alpha1 --kind CostBudget
# answer y to "Create Resource" and "Create Controller"
```

`init` lays down `go.mod`, `cmd/main.go` (the manager), `Makefile`, `config/` (kustomize manifests), `Dockerfile`, and a `test/` tree. `create api` adds `api/v1alpha1/costbudget_types.go` (your Go structs) and `internal/controller/costbudget_controller.go` (your `Reconcile`).

The Makefile targets that matter:

| Target | What it does |
|---|---|
| `make manifests` | Runs `controller-gen` over your `+kubebuilder:` markers to regenerate `config/crd/bases/*.yaml` and `config/rbac/role.yaml`. |
| `make generate` | Runs `controller-gen object` to regenerate `zz_generated.deepcopy.go` — the `DeepCopyObject` methods that make your types satisfy `runtime.Object`. |
| `make test` | Downloads envtest binaries via `setup-envtest` if needed, exports `KUBEBUILDER_ASSETS`, runs `go test ./...`. |
| `make install` / `make deploy` | Applies the CRDs / the full operator manifests to the current kube context. |

Run `make manifests generate` after touching types or markers, every time. The generated files are committed — reviewers read the CRD YAML and the ClusterRole, and a diff that shows a new RBAC verb appearing is a review signal.

kubebuilder v4.6 upgraded its scaffold from controller-runtime v0.23.3 to **v0.24.1** for Kubernetes 1.36 support, controller-tools to **v0.21.0**, and started pinning the ENVTEST version in generated projects (supported from controller-runtime v0.24.0). If your scaffold predates that, `make test` may silently pull a different Kubernetes version's control-plane binaries than the cluster you deploy to.

### Markers: the code-generation surface

Markers are `+kubebuilder:...` comments read by `controller-gen`. The ones that carry weight:

| Marker | Applies to | Effect |
|---|---|---|
| `+kubebuilder:object:root=true` | A struct | Generates `DeepCopyObject`, making it a real `runtime.Object`. Required on both the Kind and its List type. |
| `+kubebuilder:subresource:status` | The Kind | Adds `/status` to the CRD's `subresources`. See above for what that changes. |
| `+kubebuilder:printcolumn:name=…,type=…,JSONPath=…` | The Kind | Adds a column to `kubectl get`. `type` ∈ `integer`, `number`, `string`, `boolean`, `date`. |
| `+kubebuilder:resource:shortName=cb,scope=Namespaced` | The Kind | Short names for `kubectl`, and cluster- vs namespace-scope. |
| `+kubebuilder:validation:Required` / `Optional` | A field | Emits it into the CRD's `required` list or not. |
| `+kubebuilder:validation:Minimum=0` / `Maximum` / `MinLength` / `MaxLength` / `Pattern` | A field | OpenAPI v3 constraints enforced by the API server at admission. |
| `+kubebuilder:validation:Enum=A;B;C` | A field | Enumerated allowed values. |
| `+kubebuilder:default=30` | A field | Server-side defaulting when the field is absent. |
| `+kubebuilder:validation:XValidation:rule="…",message="…"` | A field or type | A CEL expression evaluated by the API server (CRD validation rules, GA since 1.29). This is how you express *some* cross-field invariants without a webhook. |
| `+listType=map` + `+listMapKey=type` | A slice field | Declares the list as an associative list keyed by `type`, so server-side apply merges by key instead of replacing wholesale. **Required on `[]metav1.Condition`.** |
| `+kubebuilder:rbac:groups=…,resources=…,verbs=…` | Anywhere in the package (conventionally above `Reconcile`) | Emits rules into `config/rbac/role.yaml`. |

The full RBAC marker grammar, from controller-tools v0.21.0's own documentation:

```
+kubebuilder:rbac:groups=<groups>,resources=<resources>,resourceNames=<names>,verbs=<verbs>,urls=<non-resource-urls>,namespace=<ns>
```

Semicolons separate multiple values within a field; the core API group is the empty string, written `groups=""`.

### RBAC, and the truth about envtest

```go
// +kubebuilder:rbac:groups=cost.example.com,resources=costbudgets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=cost.example.com,resources=costbudgets/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=cost.example.com,resources=costbudgets/finalizers,verbs=update
// +kubebuilder:rbac:groups="",resources=namespaces,verbs=get;list;watch
// +kubebuilder:rbac:groups="",resources=events,verbs=create;patch
```

The `/status` line is separate because it's a separate subresource — forget it and every `Status().Update` gets a 403 in-cluster. The `/finalizers` line is needed if you add a finalizer to an object you don't own. The `events` line is needed the moment you use an `EventRecorder`.

**Now the correction that matters.** The commonly repeated claim is "envtest does not enforce RBAC." That is imprecise in a way that will mislead you if you ever try to test RBAC. What is actually true, read from `pkg/internal/testing/controlplane`:

- envtest starts `kube-apiserver` with `--authorization-mode=RBAC`. The authorizer **is** running.
- envtest's `ControlPlane` provisions a default user via `AddUser(User{Name: "default", Groups: []string{"system:masters"}}, ...)`, and the `rest.Config` your test receives authenticates as that user. `system:masters` is hardwired in the API server's authorization chain to allow everything, before RBAC rules are consulted.
- envtest also passes `--disable-admission-plugins=ServiceAccount`, so no default ServiceAccount is created in new namespaces.

So: RBAC is enforced, and your test client is a superuser, so it never notices. **The practical consequence is unchanged — a missing `costbudgets/status` verb passes every envtest run and 403s on day one in a real cluster — but now you know the mechanism, and you know the fix.** You can create a restricted identity with `testEnv.AddUser(envtest.User{Name: "operator", Groups: []string{}}, nil)`, bind your generated ClusterRole to it, and build a client from *that* config. Running one test that way turns a manual review step into an automated one.

Either way, check the generated `config/rbac/role.yaml` by hand against every client call your reconciler makes. A green test suite is not evidence about permissions.

### `observedGeneration` and conditions

`.metadata.generation` increments on every spec change — and only on spec changes, which is exactly what the status subresource buys you. Comparing `.status.observedGeneration` to `.metadata.generation` lets **any** consumer — a `kubectl` printcolumn, a dashboard, another controller, a CI gate — answer "has status caught up with the latest spec edit?" without knowing anything about your reconcile internals. It's how Kubernetes' own built-in controllers surface staleness, and it costs one line.

`metav1.Condition` is the standard status-element type, and its fields are validated by the API server:

| Field | Constraint |
|---|---|
| `Type` | Required. CamelCase or `foo.example.com/CamelCase`. Max 316 chars. |
| `Status` | Required. Enum: `True`, `False`, `Unknown`. |
| `Reason` | **Required, non-empty.** Must match `^[A-Za-z]([A-Za-z0-9_,:]*[A-Za-z0-9_])?$`. Max 1024. |
| `Message` | Required (may be empty string). Max 32768. |
| `LastTransitionTime` | Required. |
| `ObservedGeneration` | Optional, `Minimum=0`. |

The `Reason` pattern is the one that bites: a reason of `"Over budget"` (with a space) or `""` is rejected by the API server with a validation error, from a `Status().Update` that looks fine in Go. Use CamelCase identifiers: `OverBudget`, `UnderBudget`, `UpstreamUnavailable`.

Always mutate conditions with `meta.SetStatusCondition(&cb.Status.Conditions, cond)` rather than appending, and understand *why* — it is the mechanism that makes your status write idempotent. Reading `k8s.io/apimachinery/pkg/api/meta/conditions.go`:

- If no condition of that `Type` exists, it appends, setting `LastTransitionTime` to now if unset.
- If one exists, it updates `Status`, `Reason`, `Message`, and `ObservedGeneration` **only where they differ**, and returns `changed bool`.
- Crucially, **`LastTransitionTime` is only rewritten when `Status` actually changes.** `Reason` or `Message` changing alone does not bump it.

That last rule is why a second reconcile on unchanged inputs produces a byte-identical status — which is precisely the idempotency property your envtest suite will assert. Append instead of `SetStatusCondition` and the list grows without bound; set `LastTransitionTime` yourself and every reconcile is a diff, so every reconcile is a write, so every write generates a watch event, so you have built an infinite loop.

You can also use `changed` to skip the write entirely:

```go
changed := meta.SetStatusCondition(&cb.Status.Conditions, cond)
if !changed && cb.Status.ObservedCostUSD == observed && cb.Status.ObservedGeneration == cb.Generation {
    return ctrl.Result{}, nil // nothing to write; don't generate a pointless watch event
}
```

At fleet scale that check is not a micro-optimisation: every avoided write is one fewer API-server request, one fewer etcd revision, and one fewer watch event fanned out to every informer watching that Kind across every controller in the cluster.

### Owner references, garbage collection, and finalizers

If your controller *creates* child objects, set an owner reference with `controllerutil.SetControllerReference(&parent, &child, r.Scheme)`. This does two things:

**It enables cascading deletion.** Kubernetes runs a built-in **garbage collector** in kube-controller-manager that watches for objects whose owners are gone and deletes them. You get an operational choice:

| Policy | Behaviour | Choose when |
|---|---|---|
| **Background** (default) | Parent is deleted immediately; the GC removes children asynchronously. | Speed matters and a brief orphan window is harmless. |
| **Foreground** (`blockOwnerDeletion: true` + `deletionPropagation=Foreground`) | Parent enters deletion but is not removed until all blocking children are gone. | A child holds an external resource — a cloud disk, an IP allocation, a load balancer — and you need deterministic cleanup ordering. |
| **Orphan** | Children survive the parent, owner reference removed. | Rare; handing objects off to another owner. |

**It drives re-enqueue.** `Owns(&corev1.ConfigMap{})` in the builder installs `handler.EnqueueRequestForOwner`, so an edit to a child enqueues the *parent's* key. Note the default: the builder passes `handler.OnlyControllerOwner()` unless you opt out with `builder.MatchEveryOwner`, so only the owner reference with `controller: true` triggers a re-enqueue. `SetControllerReference` sets that flag; `SetOwnerReference` does not. "My `Owns` doesn't fire" is almost always this.

**Finalizers** are the inverse mechanism: a string in `metadata.finalizers` that blocks the object's actual deletion until you remove it, giving you a hook to clean up *external* state (a cloud budget, a billing record, a DNS entry) before the object vanishes. The protocol:

```go
const finalizerName = "cost.example.com/budget-cleanup"

if cb.DeletionTimestamp.IsZero() {
    // Not being deleted: ensure our finalizer is present.
    if controllerutil.AddFinalizer(&cb, finalizerName) {
        return ctrl.Result{}, r.Update(ctx, &cb) // note: Update, not Status().Update —
                                                  // finalizers live in metadata
    }
} else {
    // Being deleted: do external cleanup, then release.
    if controllerutil.ContainsFinalizer(&cb, finalizerName) {
        if err := r.cleanupExternal(ctx, &cb); err != nil {
            return ctrl.Result{}, err // retry; the object stays in Terminating
        }
        controllerutil.RemoveFinalizer(&cb, finalizerName)
        return ctrl.Result{}, r.Update(ctx, &cb)
    }
    return ctrl.Result{}, nil
}
```

Two hazards. First, `cleanupExternal` must be idempotent — it will run again on any retry, and "delete the cloud disk" must tolerate the disk already being gone. Second, a finalizer whose cleanup can never succeed **wedges the object in `Terminating` forever**, and the only escape is a human patching the finalizer out. Every finalizer needs a defined failure path: either the cleanup is guaranteed-eventually-successful, or you give up after a bounded number of attempts and release the finalizer with a loud event. Our `CostBudget` owns nothing external, so it needs neither owner references nor finalizers — but you'll reach for both the moment the operator provisions real resources.

### envtest: what it is, and the four things it does not do

envtest boots **real `kube-apiserver` and `etcd` binaries** as local processes, wired to a temporary data directory, and hands your test a `*rest.Config`. It is not a fake and not a mock: CRD schema validation, defaulting, admission of the built-in plugins, status-subresource semantics, optimistic concurrency, and watch behaviour are all the genuine article. That makes it dramatically more trustworthy than `fake.Client`, which reimplements a subset of the API server's behaviour in Go and diverges in exactly the places that matter (subresources, field validation, resourceVersion semantics).

`setup-envtest` downloads the control-plane binaries; `KUBEBUILDER_ASSETS` points at them. The generated `Makefile` wires this for you.

What it does **not** run, and the consequences:

| Missing component | Consequence in your tests |
|---|---|
| **kubelet** | Pods can be created but never run, never get an IP, never report status. Anything that waits on a Pod being `Running` hangs. |
| **kube-scheduler** | Pods are created and stay `Pending` with no node assignment forever. |
| **kube-controller-manager** | **No garbage collection** — deleting a parent does *not* delete children with owner references. Assert on the owner reference, not on the child's disappearance. **No namespace controller** — a deleted namespace enters `Terminating` and never completes, so you cannot recreate a namespace of the same name within a suite. Use a unique namespace per test, or filter reconciles by namespace. |
| **A restricted identity** | The API server runs with `--authorization-mode=RBAC`, but your client is `system:masters`, which is allowed everything before RBAC is consulted. RBAC bugs are invisible unless you explicitly `AddUser` a restricted identity. |

The namespace one catches everyone: `Eventually(func() bool { return apierrors.IsNotFound(k8sClient.Get(ctx, nsKey, &ns)) })` will time out in envtest and pass in a real cluster, because there is no namespace controller to finish the deletion.

`envtest.Environment` also gives you `CRDDirectoryPaths` (point it at `config/crd/bases`), `ErrorIfCRDPathMissing: true` (turn a typo'd path into a loud failure rather than a silently CRD-less suite), and `ControlPlaneStartTimeout` (default 20 s, or `KUBEBUILDER_CONTROLPLANE_START_TIMEOUT`).

### What's next beyond Reconcile: admission webhooks

One boundary worth naming even though it's out of scope: **Reconcile assumes the object it's handed is already schema-valid.** The CRD's OpenAPI schema, generated from your `+kubebuilder:validation` markers, catches type errors and simple constraints. CEL rules (`+kubebuilder:validation:XValidation`) extend that to some cross-field invariants evaluated by the API server. What neither can do is anything requiring a *lookup*: "the namespace this budget references must actually exist," "this budget must not overlap another budget's selector." That validation belongs in a **validating admission webhook** — a layer that runs *before* the object is persisted, so an invalid object never enters the system rather than sitting there making your reconcile defensive.

You won't build one in this lesson — it's a module-02-and-beyond topic — but knowing where the boundary sits (schema → CEL → webhook → reconcile) is itself staff-level signal, as is knowing the operational cost: a webhook with `failurePolicy: Fail` that goes down blocks writes to its resources cluster-wide.

## Perspectives

**Developer view.** The Get→observe→compute→write-status→return shape is, mechanically, a pure function over cluster state. The entire discipline of "idempotent, level-triggered" reduces to one rule: don't let your reconcile function have memory of its own; recompute from inputs every time. Once you see it this way, most controller bugs are violations of referential transparency — code that remembers something from a previous call instead of re-deriving it. The corollary for testing is that a reconciler is unusually easy to test *if* you keep the seam: inject the expensive part behind an interface (`CostSource`), and every branch is reachable from a table test with no cluster at all.

**Operator/scale view.** The informer/cache/workqueue machinery exists so that N reconciles cost ~0 marginal API-server load. That's not a theoretical claim; it's load-tested at extreme scale. Google's 130,000-node GKE work shows the concrete mechanisms — consistent reads from cache (KEP-2340), a snapshottable API-server cache (KEP-4988) — that let a control plane sustain ~1,000 Pod creates/second with a million-plus objects. You're not learning a toy version of production; you're learning the actual shape. The operator's corollary: the manager already exposes `controller_runtime_reconcile_total{controller,result}`, `controller_runtime_reconcile_errors_total`, `controller_runtime_reconcile_time_seconds`, and `controller_runtime_active_workers` on its metrics endpoint. Alert on `rate(controller_runtime_reconcile_errors_total[5m])` and on `controller_runtime_active_workers == MaxConcurrentReconciles` sustained (a saturated worker pool). You get all of it for free.

**Reliability/failure-mode view.** Two things are routine that beginners treat as exceptional. Optimistic-concurrency conflicts on status writes are normal traffic the moment more than one writer can touch an object — the designed response is to let the queue's backoff replay you with a fresh read, not to build an inner retry loop. And cache staleness after a write is *guaranteed*, not occasional: the write goes to the API server, the cache is updated only by the watch stream, and the library documents that a read after a write may return the old value. Design for both as normal-case, and your alerting stays quiet when nothing is actually wrong.

**Economics/organizational view.** Google Cloud's operator best-practices guidance states, as official field guidance, exactly the two structural decisions this lesson's worked example makes: asynchronous work queues over blocking polls, and multiple single-purpose controllers over one do-everything controller. There's a cost dimension too, and it's the one Lesson 7 taught you to look for: every unnecessary status write is an API-server request, an etcd revision, and a watch event fanned out to every informer on that Kind in the cluster. A controller that writes status on every reconcile regardless of change, over 10,000 objects with a 10-hour resync, is 10,000 pointless writes twice a day plus whatever churn it induces in every other controller watching. The `changed` return from `SetStatusCondition` is how you avoid that, and it's a one-line diff.

## Real-world use cases

- **kubernetes-sigs/controller-runtime — [FAQ.md](https://github.com/kubernetes-sigs/controller-runtime/blob/main/FAQ.md)** — the maintainer-stated idempotency rule, in answer to "How do I have different logic in my reconciler for different types of events?": you shouldn't, because *"Reconcile functions should be idempotent, and should always reconcile state by reading all the state it needs, then writing updates"* — which is what lets a controller handle generic events, coalesced events, and startup replay uniformly. It also gives the two sanctioned patterns for cache staleness: **optimistic locking with deterministic object names** (so a re-create after a stale read fails harmlessly with `AlreadyExists`) and **track-and-requeue** for cases where deterministic names aren't possible, under the framing *"write your controller with the assumption that information will eventually be correct, but may be slightly out of date."* **What it shows:** a primary-source, maintainer-grade citation for the exact claims this lesson's core is built on.
- **Google Cloud — [How we built a 130,000-node GKE cluster](https://cloud.google.com/blog/products/containers-kubernetes/how-we-built-a-130000-node-gke-cluster)** (Nov 2025, dated snapshot) — an experimental cluster at double the then-supported 65,000-node limit: 130,000 Pods scheduled in ~3 min 40 s, sustained ~1,000 Pods/second, relying on KEP-2340 (Consistent Reads from Cache) and KEP-4988 (Snapshottable API Server Cache), with a Spanner-backed store replacing etcd and handling ~13,000 QPS for lease updates alone. **What it shows:** hard evidence that "the cache is what makes N reconciles cost ~0 extra API load" is the literal mechanism a hyperscaler leans on, not a simplification for teaching.
- **Kubernetes v1.36 — [Staleness Mitigation and Observability for Controllers](https://kubernetes.io/blog/2026/04/28/kubernetes-v1-36-staleness-mitigation-for-controllers/)** (Apr 2026, KEP-5647, SIG API Machinery) — `client-go`'s `Store` gained `LastStoreSyncResourceVersion()` so a controller can introspect how far behind its cache is; kube-controller-manager turns staleness mitigation on by default for DaemonSet, StatefulSet, ReplicaSet and Job controllers, with per-controller `StaleControllerConsistency` feature gates to disable; new alpha `stale_sync_skips_total` and `client-go` `store_resource_version` metrics. **What it shows:** the staleness window this lesson makes you design around is important enough that the Kubernetes project shipped first-class machinery and observability for it in a recent release.
- **Google Cloud — [Best practices for building Kubernetes Operators and stateful apps](https://cloud.google.com/blog/products/containers-kubernetes/best-practices-for-building-kubernetes-operators-and-stateful-apps)** — official field guidance: asynchronous sync loops with work queues (not blocking polling), a reconcile that hits an error should return it and let the queue schedule a resync (not block/retry inline), long-running-operation watchers should sleep-and-recheck, and responsibilities should be compartmentalised across multiple single-purpose controllers. **What it shows:** the pattern you're building is the pattern the platform vendor independently recommends — useful in a system-design interview where "would you split this?" is the follow-up.

## Worked example

A `CostBudget` CRD: spec says "watch namespaces matching this label selector, this is the monthly budget"; status reports the computed observed cost, an `ObservedGeneration`, and a `WithinBudget` condition. The cost computation is injected behind the `CostSource` interface — in your flagship it's the aggregation core from `gpu-cost-exporter`, not a reimplementation.

### `api/v1alpha1/costbudget_types.go`

```go
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// CostBudgetSpec is user intent. The controller only ever READS this.
type CostBudgetSpec struct {
	// NamespaceSelector selects the namespaces included in this budget.
	// A label selector (not a bare map) so it composes with matchExpressions.
	// +kubebuilder:validation:Required
	NamespaceSelector metav1.LabelSelector `json:"namespaceSelector"`

	// MonthlyBudgetUSD is the budget ceiling in whole US dollars.
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Minimum=0
	MonthlyBudgetUSD int64 `json:"monthlyBudgetUSD"`

	// CheckInterval is how often to re-evaluate against the cost source,
	// independent of watch events. Defaults to 5 minutes.
	// +kubebuilder:validation:Optional
	// +kubebuilder:default="5m"
	CheckInterval metav1.Duration `json:"checkInterval,omitempty"`
}

// CostBudgetStatus is controller-observed state. Users only ever READ this.
type CostBudgetStatus struct {
	// ObservedCostUSD is the cost computed at the last successful reconcile.
	// +kubebuilder:validation:Optional
	ObservedCostUSD int64 `json:"observedCostUSD,omitempty"`

	// ObservedGeneration is the .metadata.generation this status reflects.
	// Because status writes never bump .metadata.generation (that is what the
	// status subresource buys us), observedGeneration < generation means
	// "status has not caught up with the latest spec edit".
	// +kubebuilder:validation:Optional
	// +kubebuilder:validation:Minimum=0
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`

	// Conditions follows the standard Kubernetes condition convention.
	// listType=map + listMapKey=type make this an associative list, so
	// server-side apply merges by condition Type instead of replacing the
	// whole slice. This is REQUIRED on []metav1.Condition.
	// +listType=map
	// +listMapKey=type
	// +patchStrategy=merge
	// +patchMergeKey=type
	// +kubebuilder:validation:Optional
	Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:shortName=cb
// +kubebuilder:printcolumn:name="Budget",type=integer,JSONPath=`.spec.monthlyBudgetUSD`
// +kubebuilder:printcolumn:name="Observed",type=integer,JSONPath=`.status.observedCostUSD`
// +kubebuilder:printcolumn:name="Within",type=string,JSONPath=`.status.conditions[?(@.type=="WithinBudget")].status`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`
type CostBudget struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   CostBudgetSpec   `json:"spec,omitempty"`
	Status CostBudgetStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true
type CostBudgetList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []CostBudget `json:"items"`
}

func init() { SchemeBuilder.Register(&CostBudget{}, &CostBudgetList{}) }
```

Two lines to notice. The `Within` printcolumn uses a JSONPath filter to pull one condition's status out of the list — that's how you get `kubectl get cb` to show a health column without adding a redundant top-level status field. And `+kubebuilder:default="5m"` means the API server fills in the interval on create, so your reconciler never has to handle a zero value.

### `internal/controller/costbudget_controller.go`

```go
package controller

import (
	"context"
	"fmt"

	corev1 "k8s.io/api/core/v1"
	apierrors "k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/api/meta"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"

	costv1alpha1 "github.com/you/gpu-operator/api/v1alpha1"
)

// CostSource is the seam (Lesson 3: interfaces on the consumer). In production
// it is the gpu-cost-exporter aggregation core; in tests it is a hand-written
// fake. The reconciler never knows which.
type CostSource interface {
	CostForNamespaces(ctx context.Context, namespaces []string) (usd int64, err error)
}

// ErrBudgetMisconfigured marks a spec problem that retrying cannot fix.
var ErrBudgetMisconfigured = fmt.Errorf("costbudget is misconfigured")

type CostBudgetReconciler struct {
	client.Client                 // embedded: gives r.Get, r.List, r.Update, r.Status(), ...
	Scheme        *runtime.Scheme
	Cost          CostSource
}

// +kubebuilder:rbac:groups=cost.example.com,resources=costbudgets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=cost.example.com,resources=costbudgets/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=cost.example.com,resources=costbudgets/finalizers,verbs=update
// +kubebuilder:rbac:groups="",resources=namespaces,verbs=get;list;watch

func (r *CostBudgetReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := ctrl.LoggerFrom(ctx) // already carries controller name, object key, reconcileID

	// ── 1. READ. From the shared cache: no API-server request. ────────────────
	var cb costv1alpha1.CostBudget
	if err := r.Get(ctx, req.NamespacedName, &cb); err != nil {
		// NotFound = the object was deleted between enqueue and now. Normal:
		// the handler enqueued a KEY, and keys outlive objects. Don't requeue.
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	// ── 2. OBSERVE. List namespaces matching the selector — also a cache read. ─
	sel, err := metav1.LabelSelectorAsSelector(&cb.Spec.NamespaceSelector)
	if err != nil {
		// A malformed selector is a SPEC bug. Retrying cannot fix it, so mark it
		// terminal: the item is dropped from the queue instead of backing off
		// to the 1000s cap forever. Surface the reason in status first.
		condChanged := r.setCondition(&cb, metav1.ConditionFalse, "InvalidSelector", err.Error())
		if uerr := r.writeStatus(ctx, &cb, 0, condChanged); uerr != nil {
			return ctrl.Result{}, uerr
		}
		return ctrl.Result{}, reconcile.TerminalError(
			fmt.Errorf("%w: %v", ErrBudgetMisconfigured, err))
	}

	var nsList corev1.NamespaceList
	if err := r.List(ctx, &nsList, &client.ListOptions{LabelSelector: sel}); err != nil {
		return ctrl.Result{}, err // transient: backoff requeue
	}
	names := make([]string, 0, len(nsList.Items))
	for i := range nsList.Items {
		names = append(names, nsList.Items[i].Name)
	}

	// ── 3. COMPUTE. Pure function of the inputs above — no accumulated state. ─
	// ctx carries the manager's shutdown cancellation, so a SIGTERM unwinds an
	// in-flight upstream call instead of waiting it out (Lesson 4, Lesson 7).
	observed, err := r.Cost.CostForNamespaces(ctx, names)
	if err != nil {
		condChanged := r.setCondition(&cb, metav1.ConditionUnknown, "CostSourceUnavailable", err.Error())
		if uerr := r.writeStatus(ctx, &cb, cb.Status.ObservedCostUSD, condChanged); uerr != nil {
			return ctrl.Result{}, uerr
		}
		return ctrl.Result{}, err // transient: backoff requeue
	}

	// ── 4. WRITE STATUS ONLY. Full recomputation, never an increment. ─────────
	var condChanged bool
	if observed > cb.Spec.MonthlyBudgetUSD {
		condChanged = r.setCondition(&cb, metav1.ConditionFalse, "OverBudget",
			fmt.Sprintf("observed %d USD exceeds monthly budget %d USD",
				observed, cb.Spec.MonthlyBudgetUSD))
	} else {
		condChanged = r.setCondition(&cb, metav1.ConditionTrue, "UnderBudget",
			fmt.Sprintf("observed %d USD within monthly budget %d USD",
				observed, cb.Spec.MonthlyBudgetUSD))
	}
	if err := r.writeStatus(ctx, &cb, observed, condChanged); err != nil {
		return ctrl.Result{}, err
	}

	log.Info("reconciled",
		"namespaces", len(names), "observedUSD", observed,
		"budgetUSD", cb.Spec.MonthlyBudgetUSD, "generation", cb.Generation)

	// ── 5. Poll the external cost source on a schedule. NOT Requeue: true. ────
	return ctrl.Result{RequeueAfter: cb.Spec.CheckInterval.Duration}, nil
}

// setCondition mutates the condition list in place and reports whether anything
// actually changed. meta.SetStatusCondition rewrites LastTransitionTime ONLY when
// the condition's Status changes — that is the mechanism that makes a repeated
// reconcile produce a byte-identical status. Never append to Conditions directly.
func (r *CostBudgetReconciler) setCondition(
	cb *costv1alpha1.CostBudget, status metav1.ConditionStatus, reason, msg string,
) bool {
	return meta.SetStatusCondition(&cb.Status.Conditions, metav1.Condition{
		Type:               "WithinBudget",
		Status:             status,
		Reason:             reason, // MUST match ^[A-Za-z]([A-Za-z0-9_,:]*[A-Za-z0-9_])?$
		Message:            msg,
		ObservedGeneration: cb.Generation,
	})
}

// writeStatus writes the status subresource, skipping the request entirely when
// nothing changed. Every avoided write is one fewer etcd revision and one fewer
// watch event fanned out to every informer on this Kind in the cluster.
func (r *CostBudgetReconciler) writeStatus(
	ctx context.Context, cb *costv1alpha1.CostBudget, observed int64, condChanged bool,
) error {
	// If the condition was unchanged AND both scalar fields already match, the
	// object in hand is identical to what we would PUT. Skip the round trip.
	if !condChanged &&
		cb.Status.ObservedCostUSD == observed &&
		cb.Status.ObservedGeneration == cb.Generation {
		return nil
	}
	cb.Status.ObservedCostUSD = observed
	cb.Status.ObservedGeneration = cb.Generation

	// Status().Update writes ONLY /status. It physically cannot carry a spec
	// change, so it can never revert a user's concurrent spec edit.
	if err := r.Status().Update(ctx, cb); err != nil {
		if apierrors.IsConflict(err) {
			// Optimistic-concurrency conflict: routine traffic, not a failure.
			// Returning the error lets the workqueue replay us with a fresh
			// cache read. Do NOT retry in-line — the queue is the retry loop.
			return err
		}
		return fmt.Errorf("update status: %w", err)
	}
	return nil
}

func (r *CostBudgetReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&costv1alpha1.CostBudget{}).       // primary: wires source.Kind +
		                                        // handler.EnqueueRequestForObject
		Named("costbudget").                    // required if For() is absent; good hygiene always
		WithOptions(controller.Options{
			MaxConcurrentReconciles: 4,          // DEFAULT IS 1. The queue still
			                                     // guarantees one key is never
			                                     // reconciled concurrently.
		}).
		Complete(r)                             // → Build → doController + doWatch
}
```

Read the shape: **Get → observe → compute → write status → return.** No edge assumptions, no accumulated state, one owned field written. Run it a hundred times on the same cluster state and the object lands in the same place. And notice what makes it idempotent isn't the *absence* of writes — it writes status — it's that the write is a full recomputation from current inputs, never an increment or an append.

### `internal/controller/suite_test.go` — the envtest harness

```go
package controller

import (
	"context"
	"path/filepath"
	"testing"
	"time"

	. "github.com/onsi/ginkgo/v2"
	. "github.com/onsi/gomega"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/api/meta"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/kubernetes/scheme"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/envtest"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"
	metricsserver "sigs.k8s.io/controller-runtime/pkg/metrics/server"

	costv1alpha1 "github.com/you/gpu-operator/api/v1alpha1"
)

var (
	k8sClient client.Client
	testEnv   *envtest.Environment
	ctx       context.Context
	cancel    context.CancelFunc
)

// fakeCostSource: a hand-written fake, no mocking library (Lesson 5).
type fakeCostSource struct{ usd int64; err error }

func (f *fakeCostSource) CostForNamespaces(_ context.Context, _ []string) (int64, error) {
	return f.usd, f.err
}

var costSrc = &fakeCostSource{}

func TestControllers(t *testing.T) {
	RegisterFailHandler(Fail)
	RunSpecs(t, "CostBudget Controller Suite")
}

var _ = BeforeSuite(func() {
	logf.SetLogger(zap.New(zap.WriteTo(GinkgoWriter), zap.UseDevMode(true)))
	ctx, cancel = context.WithCancel(context.TODO())

	testEnv = &envtest.Environment{
		// Real kube-apiserver + etcd binaries; KUBEBUILDER_ASSETS points at them.
		CRDDirectoryPaths:     []string{filepath.Join("..", "..", "config", "crd", "bases")},
		ErrorIfCRDPathMissing: true, // a typo'd path becomes a loud failure, not a
		                              // silently CRD-less suite
	}
	cfg, err := testEnv.Start()
	Expect(err).NotTo(HaveOccurred())
	Expect(cfg).NotTo(BeNil())

	Expect(costv1alpha1.AddToScheme(scheme.Scheme)).To(Succeed())

	k8sClient, err = client.New(cfg, client.Options{Scheme: scheme.Scheme})
	Expect(err).NotTo(HaveOccurred())

	// Run the real manager, so the real cache/informer/workqueue path is under
	// test — not a hand-called Reconcile.
	mgr, err := ctrl.NewManager(cfg, ctrl.Options{
		Scheme:  scheme.Scheme,
		Metrics: metricsserver.Options{BindAddress: "0"}, // "0" disables; avoids
		                                                  // port clashes in CI
	})
	Expect(err).NotTo(HaveOccurred())

	Expect((&CostBudgetReconciler{
		Client: mgr.GetClient(),
		Scheme: mgr.GetScheme(),
		Cost:   costSrc,
	}).SetupWithManager(mgr)).To(Succeed())

	go func() {
		defer GinkgoRecover()
		Expect(mgr.Start(ctx)).To(Succeed())
	}()
})

var _ = AfterSuite(func() {
	cancel()
	Expect(testEnv.Stop()).To(Succeed())
})

var _ = Describe("CostBudget controller", func() {
	const ns = "default"

	It("drives status toward spec and is idempotent", func() {
		costSrc.usd = 1200
		key := types.NamespacedName{Name: "team-a-budget", Namespace: ns}

		Expect(k8sClient.Create(ctx, &costv1alpha1.CostBudget{
			ObjectMeta: metav1.ObjectMeta{Name: key.Name, Namespace: key.Namespace},
			Spec: costv1alpha1.CostBudgetSpec{
				NamespaceSelector: metav1.LabelSelector{
					MatchLabels: map[string]string{"cost-team": "team-a"},
				},
				MonthlyBudgetUSD: 5000,
				CheckInterval:    metav1.Duration{Duration: time.Hour}, // don't poll during the test
			},
		})).To(Succeed())

		// Eventually, because reconcile is asynchronous: the create returns as
		// soon as the API server persists it; the informer, queue and worker
		// still have to run.
		var got costv1alpha1.CostBudget
		Eventually(func(g Gomega) {
			g.Expect(k8sClient.Get(ctx, key, &got)).To(Succeed())
			g.Expect(got.Status.ObservedCostUSD).To(Equal(int64(1200)))
			g.Expect(got.Status.ObservedGeneration).To(Equal(got.Generation))
			cond := meta.FindStatusCondition(got.Status.Conditions, "WithinBudget")
			g.Expect(cond).NotTo(BeNil())
			g.Expect(cond.Status).To(Equal(metav1.ConditionTrue))
			g.Expect(cond.Reason).To(Equal("UnderBudget"))
		}, 20*time.Second, 200*time.Millisecond).Should(Succeed())

		// THE IDEMPOTENCY PROOF. Capture the status, force another reconcile by
		// touching a label (which bumps resourceVersion but NOT generation, since
		// generation only moves on spec changes), and assert the status is
		// byte-identical — INCLUDING LastTransitionTime, which meta.SetStatusCondition
		// only rewrites when the condition's Status actually changes.
		before := *got.Status.DeepCopy()

		Eventually(func() error {
			if err := k8sClient.Get(ctx, key, &got); err != nil {
				return err
			}
			if got.Labels == nil {
				got.Labels = map[string]string{}
			}
			got.Labels["poke"] = "1"
			return k8sClient.Update(ctx, &got) // may 409; Eventually retries
		}, 10*time.Second, 100*time.Millisecond).Should(Succeed())

		Consistently(func(g Gomega) {
			var after costv1alpha1.CostBudget
			g.Expect(k8sClient.Get(ctx, key, &after)).To(Succeed())
			g.Expect(after.Status).To(Equal(before))
		}, 3*time.Second, 300*time.Millisecond).Should(Succeed())
	})

	It("reports OverBudget when the cost source exceeds the budget", func() {
		costSrc.usd = 9000
		key := types.NamespacedName{Name: "team-b-budget", Namespace: ns}

		Expect(k8sClient.Create(ctx, &costv1alpha1.CostBudget{
			ObjectMeta: metav1.ObjectMeta{Name: key.Name, Namespace: key.Namespace},
			Spec: costv1alpha1.CostBudgetSpec{
				NamespaceSelector: metav1.LabelSelector{
					MatchLabels: map[string]string{"cost-team": "team-b"},
				},
				MonthlyBudgetUSD: 5000,
				CheckInterval:    metav1.Duration{Duration: time.Hour},
			},
		})).To(Succeed())

		Eventually(func(g Gomega) {
			var got costv1alpha1.CostBudget
			g.Expect(k8sClient.Get(ctx, key, &got)).To(Succeed())
			cond := meta.FindStatusCondition(got.Status.Conditions, "WithinBudget")
			g.Expect(cond).NotTo(BeNil())
			g.Expect(cond.Status).To(Equal(metav1.ConditionFalse))
			g.Expect(cond.Reason).To(Equal("OverBudget"))
		}, 20*time.Second, 200*time.Millisecond).Should(Succeed())
	})

	It("rejects a status write through the main endpoint", func() {
		// Proves the subresource split is real, not a convention: a plain Update
		// carrying a status change must NOT move status.
		key := types.NamespacedName{Name: "team-a-budget", Namespace: ns}
		var got costv1alpha1.CostBudget
		Expect(k8sClient.Get(ctx, key, &got)).To(Succeed())

		got.Status.ObservedCostUSD = 999999
		Expect(k8sClient.Update(ctx, &got)).To(Succeed()) // succeeds — and is ignored

		var after costv1alpha1.CostBudget
		Expect(k8sClient.Get(ctx, key, &after)).To(Succeed())
		Expect(after.Status.ObservedCostUSD).NotTo(Equal(int64(999999)))
	})

	// Deliberately NOT tested here, because envtest cannot: cascading deletion of
	// owned children (no kube-controller-manager => no garbage collector) and
	// namespace teardown (no namespace controller => Terminating forever).
})
```

**Reading the run.**

```
$ make test
...
KUBEBUILDER_ASSETS="/root/.local/share/kubebuilder-envtest/k8s/1.36.0-linux-amd64" go test ./... -coverprofile cover.out

Running Suite: CostBudget Controller Suite - /work/internal/controller
======================================================================
Random Seed: 1755424913

Will run 3 of 3 specs
1.755e+09  INFO  controller-runtime.metrics  Metrics server is starting to listen  {"addr": "0"}
1.755e+09  INFO  Starting EventSource  {"controller": "costbudget", "source": "kind source: *v1alpha1.CostBudget"}
1.755e+09  INFO  Starting Controller  {"controller": "costbudget"}
1.755e+09  INFO  Starting workers  {"controller": "costbudget", "worker count": 4}
1.755e+09  INFO  reconciled  {"controller":"costbudget","name":"team-a-budget","namespace":"default","reconcileID":"...","namespaces":0,"observedUSD":1200,"budgetUSD":5000,"generation":1}
•••
Ran 3 of 3 Specs in 9.412 seconds
SUCCESS! -- 3 Passed | 0 Failed | 0 Pending | 0 Skipped
```

(Representative transcript; timings and the reconcileID vary.) Line by line:

- `Starting EventSource ... "kind source: *v1alpha1.CostBudget"` is the `source.Kind` from Lesson 08's trace, asking the shared cache for an informer. If your CRD path were wrong, this line would be followed by the retry-loop error `"if kind is a CRD, it should be installed before calling Start"` every 10 seconds — which is why `ErrorIfCRDPathMissing: true` is worth setting.
- `Starting workers ... "worker count": 4` confirms `MaxConcurrentReconciles: 4` took effect. Without `WithOptions` this would read `1`.
- The `reconciled` line carries `reconcileID` — a UUID the framework attaches to the logger for the duration of one reconcile (`pkg/internal/controller/controller.go`). Grep by it to isolate one reconcile's logs out of a busy operator, and note there is exactly one per invocation: seeing the same object's key with many distinct reconcileIDs in a second is your flapping signature.
- `"namespaces": 0` because no namespace in the test carries the selector label — a deliberate detail: the reconciler handles the empty case without special-casing it, because `len(nil slice)` is 0 and the fake returns a constant.
- The whole suite is under ten seconds, most of which is the API server and etcd starting. That is the cost of testing against a real control plane, and it is worth every millisecond compared to a fake client that gets subresource semantics wrong.

## Practice

Scaffold the operator for real and prove the loop with envtest. This feeds directly into [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md), the module's deliverable.

1. `kubebuilder init` + `kubebuilder create api --group cost --version v1alpha1 --kind CostBudget` in `../practice/gpu-cost-exporter/operator/` (or a fresh module — your call; keep it in the deliverable tree). Record the kubebuilder, controller-runtime, and controller-tools versions in the README.
2. Fill `costbudget_types.go` and the reconciler as above. **Inject your exporter's aggregation as the `CostSource`** — the reconcile's business logic *is* the exporter core; do not re-implement it. If your exporter's aggregation doesn't currently sit behind an interface, refactor it so it does; that refactor is the point.
3. `make manifests generate` — commit the generated CRD, RBAC, and DeepCopy. Read the generated `config/crd/bases/*.yaml` once and find your `subresources.status: {}` and your printcolumns in it.
4. Write the **envtest** suite. Run the real manager (not a hand-called `Reconcile`), assert convergence with `Eventually`, and prove idempotency with the `Consistently` pattern above — including that `LastTransitionTime` does not move.
5. Add one test that proves the **subresource split**: a plain `Update` carrying a status change must not move status.
6. **Classify one error terminally.** Give the reconciler at least one path that returns `reconcile.TerminalError(...)` for an unfixable spec, and one that returns a plain error for a transient failure. Write a test for each and explain, in a comment, why each is which.
7. **By hand — because the test suite cannot do this for you** — read the generated `config/rbac/role.yaml` against every client call your reconciler makes, and confirm `costbudgets/status` is present with `get;update;patch`. Stretch: add a test that calls `testEnv.AddUser` with a restricted identity, binds the generated ClusterRole to it, and drives the reconciler with that client — turning the manual check into an automated one.

**Acceptance:**

- `make test` (envtest) is green, and the log shows `Starting workers` with the worker count you configured.
- Reconcile is idempotent: a second reconcile on identical cluster state produces a **byte-identical** `.status`, `LastTransitionTime` included.
- Status moves only through `.Status().Update` — proven by a test, not asserted in prose.
- One `TerminalError` path and one transient-error path exist, each tested, each explained.
- The RBAC role has been reviewed by hand against every client call — not assumed correct because tests pass.
- `go vet` and `golangci-lint` clean; `go test -race` clean.

## Common pitfalls

1. **Writing to `.spec` from inside Reconcile** — usually by accident, via a shared struct sent to `Update` instead of `Status().Update`. *Symptom:* a user's spec edit silently reverts, intermittently, only under concurrency. *Mechanism:* `Update` sends the whole object including the spec you read; if the user edited between your read and your write, you've overwritten them. `Status().Update` targets `/status`, which the API server will not let carry a spec change. *Correction:* Reconcile calls `Status().Update`, never plain `Update`, except deliberately on child objects it owns or on `metadata` (finalizers, labels).

2. **Treating `Result{Requeue: true}` as still-idiomatic.** *Symptom:* a deprecation warning from your linter and a raised eyebrow in an interview. *Mechanism:* deprecated in PR #3107 (merged 2025-02-24) because a rate-limiter-derived interval is meaningless for anything other than retry-on-error. *Correction:* return an error for backoff-driven retry, or `RequeueAfter` for a fixed poll interval.

3. **Returning both an error and a `RequeueAfter`.** *Symptom:* your polling interval is ignored, and a warning appears in the logs. *Mechanism:* `reconcileHandler`'s switch checks `err != nil` first and ignores the `Result` entirely; the framework logs *"RequeueAfter and Requeue will always be ignored if the error is non-nil."* *Correction:* pick one. If you want both "there was a problem" and "check again in five minutes," return `Result{RequeueAfter: 5*time.Minute}, nil` and record the problem in a condition.

4. **Assuming envtest catches RBAC problems.** *Symptom:* a green CI suite and a 403 on day one in a real cluster. *Mechanism:* the API server does run with `--authorization-mode=RBAC`, but envtest's default `rest.Config` authenticates as user `default` in group `system:masters`, which short-circuits authorization before RBAC rules are consulted. *Correction:* diff the generated ClusterRole against every client call by hand, or add a test using `testEnv.AddUser` with a restricted identity.

5. **Assuming envtest deletes owned children, or finishes deleting namespaces.** *Symptom:* an assertion that a child object disappears, or that a namespace goes away, times out — and passes in a real cluster. *Mechanism:* envtest runs no kube-controller-manager, so there is no garbage collector and no namespace controller. *Correction:* assert on the `ownerReference` being correct, not on the child's deletion. Use a unique namespace per test rather than deleting and recreating one.

6. **Not handling `apierrors.IsConflict` on a status write.** *Symptom:* noisy `controller_runtime_reconcile_errors_total` and pages for normal contention. *Mechanism:* optimistic concurrency returns 409 whenever your cached `resourceVersion` is behind, which at `MaxConcurrentReconciles > 1` or with any second writer is routine. *Correction:* return the error and let the queue's backoff replay you with a fresh read, or classify it as benign and `RequeueAfter` a short interval. Never write an in-line retry loop — the queue already is one, and an inner loop holds a worker slot while doing it worse.

7. **Reading back what you just wrote.** *Symptom:* a controller that flaps, or performs an external action twice. *Mechanism:* `Update` goes to the API server; the cache is updated only by the watch stream, which arrives milliseconds later — or seconds, under load. controller-runtime's docs say explicitly that a get after a create/update may return the old value. *Correction:* use the in-memory object, which `Update` mutated with the server's response; make creates idempotent with deterministic names; or use `mgr.GetAPIReader()` for the rare case that genuinely needs a strongly consistent read.

8. **Confusing "idempotent" with "no side effects."** *Symptom:* an audit reveals duplicated external resources, or a status counter that grows forever. *Mechanism:* idempotency means running N times on unchanged inputs produces the same end state — not that nothing is written. `status.count++` and `append(conditions, c)` fail it; `status.count = len(list)` and `meta.SetStatusCondition` pass it. *Correction:* audit every mutating call for what happens on the *second* run with nothing changed, and prove it with the `Consistently` test above.

## Self-check

- **Why must `Reconcile` be idempotent and level-triggered, not edge-triggered?**
  **Answer:** Because the workqueue makes no promise about *when* or *how many times* your function runs for a given object. Edits coalesce (the queue dedupes by key), resyncs fire with no change (`SyncPeriod`, 10 hours by default), failures requeue with backoff, and a process restart replays everything from a fresh LIST. You are never told what changed — and the framework structurally cannot tell you, because `handler.EnqueueRequestForObject` builds a `reconcile.Request` containing only `{Namespace, Name}` and discards the object. So you must read current state, compute desired state, and converge, safely repeatable. Any logic that assumes "this ran because X happened," or that accumulates (`count++`, `append`), breaks the first time two events collapse into one reconcile.

- **What is the difference between spec and status ownership, and why is status a subresource?**
  **Answer:** `.spec` is user intent — the user writes it, the controller only reads it. `.status` is controller-observed state — the controller writes it, the user only reads it. Making `/status` a *subresource* turns that convention into an enforced API mechanism with four concrete effects: it creates a second endpoint, so `Status().Update` physically cannot carry a spec change and can never revert a user's concurrent edit (and a plain `Update` cannot move status); it creates a second RBAC resource, `costbudgets/status`, grantable independently — which is why a missing `/status` verb is its own distinct 403; it gives status its own write path with its own optimistic-concurrency check; and critically it makes `.metadata.generation` increment on spec changes *only*, so status writes don't bump it. That last property is what makes the `observedGeneration` idiom work: `status.observedGeneration < metadata.generation` means "status has not caught up with the latest spec edit," readable by any consumer that knows nothing about your internals.

- **How does the controller's cache/informer avoid hammering the API server — and what does that cost you?**
  **Answer:** The manager runs one shared **informer per watched Kind per process** (not per controller): one initial LIST, then a single long-lived WATCH streaming deltas into an in-memory indexed store. Every `client.Get`/`List` in your reconcile reads from that store, not the network, so N reconciles cost ~0 extra API calls. The **workqueue** deduplicates by object key and rate-limits retries with per-item exponential backoff (5 ms doubling to a 1000 s cap). Net: the API server sees a couple of watch connections regardless of how hot your loop runs — the same design that lets Google's GKE control plane sustain ~1,000 Pod schedules/second at 130,000 nodes. **The cost is staleness.** Writes go straight to the API server and do *not* invalidate the cache; only the watch stream updates it. controller-runtime's own docs state that a get after a create/update may return the old resource, and that there is no sequential create/get coherence. So you design around it: don't read back what you just wrote, use deterministic names so re-creates are harmless, track-and-requeue, or reach for `mgr.GetAPIReader()` for a strongly consistent read when you genuinely need one.

- **Why doesn't a green envtest suite prove your controller is production-ready?**
  **Answer:** envtest boots real `kube-apiserver` and `etcd` binaries, so schema validation, defaulting, status-subresource semantics, and optimistic concurrency all behave exactly as in-cluster — that part is genuinely trustworthy. What's missing is everything else. It runs no **kube-controller-manager**, so there is no garbage collector (owned children are not cascade-deleted) and no namespace controller (a deleted namespace hangs in `Terminating` forever). It runs no **kubelet** and no **scheduler**, so Pods never run or get scheduled. And on authorization: the API server *does* run with `--authorization-mode=RBAC`, but the `rest.Config` you're handed authenticates as user `default` in group `system:masters`, which is allowed everything before RBAC rules are consulted — so a missing `costbudgets/status` verb passes every run and 403s on day one in a real cluster. envtest proves reconcile *logic*; it does not prove the *permissions* or the *lifecycle* your controller will actually have.

- **What are the four ways a `Reconcile` return value steers the workqueue?**
  **Answer:** `Result{}, nil` → `Queue.Forget(req)`, which resets the per-item backoff counter, and no requeue — you wake again only on a watch event or resync. `Result{}, err` → requeue with rate-limited exponential backoff (5 ms → 1000 s); **the `Result` is ignored entirely**, so a `RequeueAfter` returned alongside an error is silently dropped and the framework logs a warning about it. `Result{RequeueAfter: d}, nil` → `Forget`, then requeue after exactly `d` — the right way to poll an external system. And `reconcile.TerminalError(err)` → **no requeue at all**, just a metric and a log, for errors that retrying cannot fix (a semantically invalid spec, a permanently vanished external resource). There's also the deprecated `Result{Requeue: true}` — don't. Each branch increments `controller_runtime_reconcile_total` with a `result` label of `success` / `error` / `requeue_after` / `requeue`, which is how you answer "is it reconciling?" from a dashboard.

- **What's the difference between foreground and background deletion for an owned child, and when would you choose one?**
  **Answer:** Both rely on the built-in garbage collector cascading deletion via `ownerReferences`. **Background** (the default) deletes the parent immediately and cleans up children asynchronously — fast, but there's a window where the parent is gone and children linger. **Foreground** (`blockOwnerDeletion: true` plus `deletionPropagation=Foreground`) keeps the parent in a deleting state until all blocking children are confirmed gone — slower, but guarantees no orphans survive the parent, which matters when a child holds an external resource (a cloud disk, an IP allocation) you need cleaned up deterministically first. A related trap: `controllerutil.SetControllerReference` sets `controller: true`, and the builder's `Owns()` defaults to `handler.OnlyControllerOwner()`, so a child linked with `SetOwnerReference` (no controller flag) will not re-enqueue its parent. And neither policy does anything in envtest, because there is no garbage collector there.

## Connections & what's next

This is the **last lesson in Module 01** — the Go gate closes here. Everything upstream was building toward being able to write and trust the reconcile loop above: interfaces (Lesson 3) are the `CostSource` seam that makes it testable; error classification (Lesson 2) is `TerminalError` versus a plain return; concurrency and `context` (Lesson 4) are the worker pool and the `SIGTERM` path; testing (Lesson 5) is envtest and the hand-written fake; stdlib fluency (Lesson 7) is the metrics endpoint the manager serves and the timeouts on any HTTP call your `CostSource` makes; and reading unfamiliar Go (Lesson 8) is why you can explain every arrow in this lesson's first diagram from source rather than from belief.

From here:

- Take the [**checkpoint**](../checkpoint.md) — the module's completion gate. Items 1–5 are provable from the exporter core you built across earlier lessons; items 6–7 (the controller proof and the depth probes, including the three controller questions this lesson's self-check mirrors) are provable directly from this lesson's practice work.
- Finish (or keep building on) [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md) — the CRD and reconciler here are meant to sit directly on top of the exporter's `CostSource` aggregation, not duplicate it.
- This module **unlocks two others**: [**02 — Kubernetes internals and controllers**](../../02-kubernetes-controllers/README.md) goes deep on everything this lesson only introduced — informer internals, the Reflector/DeltaFIFO/Indexer chain, custom schedulers, webhook admission chains, multi-controller architectures — and [**04 — GPU on Kubernetes**](../../04-gpu-on-kubernetes/README.md) is where this exact reconcile shape gets pointed at GPU-specific resources (device plugins, node feature discovery, GPU-aware scheduling). Both assume you can write and reason about a reconcile loop without re-explanation — that assumption is what this lesson exists to earn.

## References & further reading

**Primary sources**

- [kubernetes-sigs/controller-runtime — FAQ.md](https://github.com/kubernetes-sigs/controller-runtime/blob/main/FAQ.md) — read for the maintainer-stated idempotency rule (*"Reconcile functions should be idempotent, and should always reconcile state by reading all the state it needs, then writing updates"*) and the two sanctioned patterns for cache staleness: optimistic locking with deterministic names, and track-and-requeue, under the framing *"write your controller with the assumption that information will eventually be correct, but may be slightly out of date."*
- [controller-runtime godoc](https://pkg.go.dev/sigs.k8s.io/controller-runtime) — the authoritative API surface: `Manager`, `Builder`, `Reconciler`, `Result` (including the `Priority` field and the deprecation on `Requeue`), and `client.Client`. The **root package doc comment** is the single best short description of the split client's staleness guarantee — it states in as many words that code must not assume a get after a create/update returns the updated resource. Read `pkg/reconcile`, `pkg/builder`, and `pkg/client` first. Defaults cited in this lesson come from `pkg/controller/controller.go` (MaxConcurrentReconciles 1, CacheSyncTimeout 2m, rate limiter 5 ms→1000 s), `pkg/cache/cache.go` (SyncPeriod 10h), and `pkg/manager/internal.go` (lease 15s/10s/2s, graceful shutdown 30s).
- [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) — read for the canonical spec/status/subresource/object-reference conventions, the condition contract, and `observedGeneration` semantics that this lesson's ownership-boundary and staleness sections are built on.
- [Kubebuilder Book — envtest reference](https://book.kubebuilder.io/reference/envtest) — the authoritative list of what envtest runs (etcd + kube-apiserver) and what it does not (no kubelet, no controller-manager — hence no garbage collection and no namespace deletion — no scheduler), plus `setup-envtest` usage and the `KUBEBUILDER_ASSETS` / `USE_EXISTING_CLUSTER` / `KUBEBUILDER_CONTROLPLANE_START_TIMEOUT` environment variables.
- [controller-tools — marker reference](https://book.kubebuilder.io/reference/markers) — the full `+kubebuilder:` grammar reproduced in this lesson's marker table, including the RBAC marker's `groups`/`resources`/`resourceNames`/`verbs`/`urls`/`namespace` fields and the `subresource:status`, `printcolumn`, `validation:*`, and `XValidation` markers. Checked against controller-tools v0.21.0.
- [kubernetes/enhancements — sig-api-machinery KEPs](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery) — the primary source for the API-machinery evolution cited here: KEP-2340 (consistent reads from cache), KEP-4988 (snapshottable API server cache), and KEP-5647 (controller staleness mitigation). The "where do I go to see *why*" answer.

**Real-world engineering blogs**

- [Google Cloud — Best practices for building Kubernetes Operators and stateful apps](https://cloud.google.com/blog/products/containers-kubernetes/best-practices-for-building-kubernetes-operators-and-stateful-apps) — what it shows: official guidance for async work queues, error-return-not-block, sleep-and-recheck for long operations, and multi-controller compartmentalisation — nearly a direct spec for this lesson's `Result`-semantics section.
- [Google Cloud — How we built a 130,000-node GKE cluster](https://cloud.google.com/blog/products/containers-kubernetes/how-we-built-a-130000-node-gke-cluster) (Nov 2025, dated snapshot) — what it shows: 130,000 Pods in ~3 min 40 s at a sustained ~1,000 Pods/second, an experimental cluster at double the then-supported 65,000-node limit, resting on KEP-2340 and KEP-4988 plus a Spanner-backed store handling ~13,000 QPS of lease updates. Real evidence that cache-first reconciliation is the literal mechanism at extreme scale.
- [Kubernetes blog — v1.36: Staleness Mitigation and Observability for Controllers](https://kubernetes.io/blog/2026/04/28/kubernetes-v1-36-staleness-mitigation-for-controllers/) (Apr 2026) — what it shows: KEP-5647 adds `LastStoreSyncResourceVersion()` to `client-go`'s `Store`, enables staleness mitigation by default in kube-controller-manager for DaemonSet/StatefulSet/ReplicaSet/Job (gated per-controller by `StaleControllerConsistency`), and ships alpha `stale_sync_skips_total` and `store_resource_version` metrics. Confirms the cache-staleness problem this lesson teaches is a first-class upstream concern.
- [controller-runtime PR #3107 — Deprecate `reconcile.Result.Requeue`](https://github.com/kubernetes-sigs/controller-runtime/pull/3107) — what it shows: the maintainer's own one-sentence justification (commit `f15ff17`, merged 2025-02-24) for a deprecation you will be asked about — *"There is no good reason to use this setting, either an error or `RequeueAfter` should be used instead."*

**Deeper dives**

- Daniel Mangum — [Rate Limiting in controller-runtime and client-go](https://danielmangum.com/posts/controller-runtime-client-go-rate-limiting/) — unpacks how the per-item exponential backoff and the token-bucket limiter compose, and how per-item backoff state is tracked and reset by `Forget`. Read it once and the 5 ms → 1000 s numbers stop being magic.
- [Kubebuilder Book — CronJob tutorial](https://book.kubebuilder.io/cronjob-tutorial/cronjob-tutorial.html) — the canonical end-to-end walkthrough: types, markers, reconcile, owner refs, envtest, webhooks. Do it hands-on — type every step, run `make manifests` and `make test`. The single highest-leverage hour for turning "I read about controllers" into "I've built one."
- [kubernetes/sample-controller](https://github.com/kubernetes/sample-controller) — the same loop in raw `client-go`, no controller-runtime: you write the informer registration, the workqueue, the worker loop, and the key-splitting yourself. Skim once after this lesson; seeing the machinery unhidden shows exactly what controller-runtime does for you, and it is invaluable when you read a controller in the wild (Lesson 8's skill).
- [Kubernetes Programming with Go](https://www.oreilly.com/library/view/kubernetes-programming-with/9781484290262/) — the `client-go`/apimachinery layer *under* controller-runtime: schemes, codecs, informers, work queues from first principles. Deep-read the informer and workqueue chapters — controller-runtime hides this, and you'll debug it the day the cache does something surprising.
- [Programming Kubernetes](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/) — the conceptual companion: controllers, informers, optimistic concurrency, API machinery explained. Skim the controllers and custom-resources chapters; it makes the level-triggered/cache model click at a level the tutorials assume.
