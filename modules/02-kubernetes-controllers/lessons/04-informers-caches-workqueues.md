---
lesson: "02.4"
title: "Informers, caches, and workqueues"
module: "02"
concept: "Informers, caches, and workqueues"
status: not-started
est_time: "22h"
prev: "03-reconciliation-model.md"
next: "05-crd-design.md"
artifacts: []
sources: 10
---

# 02.4 · Informers, caches, and workqueues

> **Concept.** The reflector → Delta FIFO → shared informer → indexer/lister → rate-limited workqueue → worker machine that every controller runs on — and its failure modes.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Lesson 03 gave you a contract you had to take on faith: reconcile receives a *key*, not an event; the workqueue is allowed to coalesce, drop, reorder, and replay; a cache read can be stale. This lesson opens that box. Every one of those behaviors is the observable effect of five concrete, nameable components — reflector, Delta FIFO, indexer, lister, workqueue — wired together in a specific way. Once you can name the box each behavior comes from, "my controller is stuck" stops being a mystery and becomes a short diagnostic checklist. This is also the last purely-mechanical lesson before the module turns to what you build *on top of* this machine: CRD schemas (05), controller-runtime's finalizer/GC layer (06), and RBAC (07) all assume you already understand how a watch becomes a reconcile.

## Why this matters

This is the single highest-ROI internal you will learn in this module. Every controller you have ever run — kube-controller-manager, your CNI, your GPU device plugin's operator — is this exact machine repeated. Two things fall out of understanding it:

1. **Interviews.** "Walk me through what an informer does" and "your controller stopped reconciling — how do you debug it" are standard senior-platform probes. The bad answer is "it watches the API." The good answer names the reflector, the Delta FIFO, the thread-safe indexer, the lister, and the rate-limited workqueue, and can explain where a stall hides (cache not synced, workqueue backing off, a wedged worker holding `Done()`).
2. **Correctness.** Nearly every subtle operator bug — reconciling on stale data, hot-looping under an event storm, "it works but lags 30s" — is a misunderstanding of cache reads vs live reads, resync semantics, or the workqueue's backoff. You cannot write a *correct* GPU cost operator without this; a cost number computed from a stale cache is a wrong invoice.

## What's new here (calibration)

As a CKA you have a true but lossy mental model: *"the controller watches resources and acts on changes."* That sentence hides the entire engineering problem, and we won't re-litigate kubectl-level watch usage (`kubectl get --watch`, YAML basics) or CNI/CSI watch behavior — you've run those for years. A naive watcher would: hammer the API server with a `GET` per reconcile; lose events during a disconnect; melt the apiserver when 5,000 pods churn at once; and reprocess the same object ten times concurrently.

The client-go machinery exists to solve exactly those four problems:

| Operating view | Extending view (what's actually there) |
|---|---|
| "watches resources" | a **reflector** doing one LIST then a long-lived WATCH, tracking `resourceVersion`, relisting on `410 Gone` |
| "keeps a copy" | a **Delta FIFO** feeding a **thread-safe indexer**; you read it through a **lister**, never the API |
| "acts on changes" | handlers enqueue **keys** into a **rate-limited workqueue**; **workers** pull keys and reconcile |
| "reruns periodically" | **resync** re-delivers cached objects as synthetic updates — it is *not* a re-LIST |

Learn the named parts and the whole thing stops being magic.

## Core concepts

The whole pipeline, one type at a time:

```
                        (only component that talks to the apiserver)
 apiserver  ──LIST──▶  ┌───────────┐
    │                  │ Reflector │  RV tracking, relist on 410 Gone
    └──WATCH stream──▶ └─────┬─────┘
                             │ pushes deltas (Added/Updated/Deleted/Sync)
                             ▼
                       ┌───────────┐   per-key ordered deltas, deduped
                       │ DeltaFIFO │
                       └─────┬─────┘
                             │ Pop()  ── consumed by ──▶
                             ▼
                  ┌────────────────────┐  updates store, then fans out
                  │ SharedIndexInformer│
                  └───┬────────────┬───┘
             updates │            │ fires handlers
                     ▼            ▼
              ┌──────────┐   ┌──────────────┐  AddFunc/UpdateFunc/DeleteFunc
              │ Indexer  │   │ EventHandlers│  ── enqueue KEY, not object ──┐
              │ (cache)  │   └──────────────┘                               │
              └────┬─────┘                                                  ▼
        Lister ────┘  cache reads                          ┌─────────────────────────┐
        (podLister.Get) ◀──── worker reads current state   │ RateLimiting Workqueue  │
                                    │                       │ dedup + backoff + bucket│
                                    └───────────────────────┴────────────┬────────────┘
                                          Get() / Done / Forget /         │
                                          AddRateLimited          ◀───────┘
                                    N worker goroutines → your Reconcile
```

Everything below names one box in that diagram.

### The reflector: LIST then WATCH

A `Reflector` (`k8s.io/client-go/tools/cache`) is the only component that talks to the API server. Its `ListAndWatch` loop:

1. **LIST** all objects of one type. The list response carries a `resourceVersion` (RV) — a server-side logical clock for that resource collection.
2. Dump every listed object into the store as `Sync`/`Replace` deltas (the initial cache fill).
3. **WATCH** from that RV: `GET .../pods?watch=true&resourceVersion=<RV>`. The server streams `ADDED/MODIFIED/DELETED` events, each bumping the RV the reflector remembers.
4. On any watch error or the connection dropping, reconnect the watch from the last-seen RV.

**The `410 Gone` case is the one to memorize.** The apiserver only retains recent history in etcd's watch cache. If your watch falls far enough behind (a long disconnect, a slow consumer, an etcd compaction), the RV you resume from is no longer available and the server returns `410 Gone / "too old resource version"`. The reflector's answer is a **relist**: throw away the position, LIST fresh, get a new RV, resume watching. This is normal and healthy — but a relist re-delivers *everything*, which is why a flapping watch shows up as periodic reconcile spikes.

**Thundering herd on informer-factory start.** When a controller-runtime manager (or a raw `SharedInformerFactory`) starts against a cluster with, say, 50,000 Pods, the initial LIST for every watched type is a large, synchronous, memory-heavy operation — and if several controllers in the same process (or several controller *pods* restarting together, e.g. after a node drain or a rolling deploy) all race to fill their caches at once, that's a real burst of apiserver load concentrated at exactly the moment the cluster is already busy. This is not a hypothetical: OpenAI's large-cluster scaling posts describe apiserver/etcd load patterns driven by exactly this kind of synchronized LIST behavior at high node/pod counts, one of the reasons they run multiple apiserver replicas specifically to isolate blast radius (see References). The practical takeaway for your operator: stagger or rate-limit simultaneous controller restarts where you can, and don't be surprised if "everything reconciles slowly for the first 30 seconds after a fleet-wide rollout" is just every cache refilling at once.

### Delta FIFO: ordered, deduped, per-object

The reflector doesn't write the store directly; it writes a `DeltaFIFO`. This is a queue keyed by object (namespace/name) whose value is the *ordered list of deltas* seen for that object (`Added`, `Updated`, `Deleted`, `Replaced`, `Sync`). Two properties matter:

- **Per-key compression:** if an object changes five times before a consumer drains it, the consumer pops one entry containing all five deltas in order — you never lose the fact that it changed, and you process it once.
- **Deletes are never lost:** a delete becomes a `Deleted` delta (sometimes wrapped as `DeletedFinalStateUnknown` when the object vanished during a missed window), so the informer can fire your delete handler even if the object is already gone from the API.

A `Pop()` consumer (the informer) drains the FIFO, applies each delta to the indexer, and fires handlers.

### Shared informer + indexer + lister

The `SharedIndexInformer` owns the DeltaFIFO consumer plus a **thread-safe store** — the `Indexer` (`ThreadSafeStore` underneath). "Shared" means one informer per (type, namespace-scope) is fanned out to many event handlers, so ten controllers watching Pods cost *one* watch, not ten.

- **Indexer** is the cache: a concurrent map of `key → runtime.Object`, plus optional secondary indexes (e.g. index Pods by node name) you register with `AddIndexers`. Reads are lock-protected and cheap.
- **Lister** is the typed, read-only façade over the indexer: `podLister.Pods(ns).Get(name)` and `.List(selector)`. **A lister read never hits the API server** — it returns a pointer into the cache. Treat the returned object as read-only shared memory; `DeepCopy()` before mutating.

**Secondary indexes are the cache's superpower.** By default the indexer only maps `namespace/name → object`. Register an index func to answer reverse questions in O(1) without listing everything:

```go
informer.AddIndexers(cache.Indexers{
    "byNode": func(obj interface{}) ([]string, error) {
        return []string{obj.(*v1.Pod).Spec.NodeName}, nil
    },
})
// later: every Pod on a GPU node, straight from cache, no API call:
pods, _ := informer.GetIndexer().ByIndex("byNode", "gpu-node-7")
```

For a GPU cost operator this is exactly how you'd fan from "a node's price changed" to "all Pods on it" cheaply. controller-runtime exposes the same thing via `mgr.GetFieldIndexer().IndexField(...)`, which then makes `client.MatchingFields{...}` list queries index-backed.

**Quantifying why the index matters.** Without an index, "all Pods on node X" means listing the *entire* cached Pod set and filtering client-side — an O(N) scan of the whole cache on every call. At 5,000 cached Pods, that's roughly 5,000 comparisons per reconcile just to answer one question, repeated every time any of those Pods triggers a reconcile. With a registered `byNode` index, `ByIndex("byNode", nodeName)` is a map lookup — O(1) in the number of distinct index values, returning only the matching subset directly. The index costs you a small amount of memory (one extra map, updated incrementally as objects are added/updated/deleted) in exchange for turning a linear scan into a constant-time lookup on every reconcile that needs it. At small scale the difference is invisible; at fleet scale, with many controllers each doing their own filtered reads on every reconcile, an unindexed scan is the kind of CPU cost that shows up as "the operator pod's CPU request seems oddly high for what it does."

You build these through a `SharedInformerFactory`:

```go
factory := informers.NewSharedInformerFactory(clientset, 30*time.Second) // 30s = default resync
podInformer := factory.Core().V1().Pods()
lister := podInformer.Lister()          // cache reads
informer := podInformer.Informer()      // AddEventHandler lives here
factory.Start(ctx.Done())               // launches reflectors
cache.WaitForCacheSync(ctx.Done(), podInformer.Informer().HasSynced) // block until initial LIST done
```

`WaitForCacheSync` is load-bearing: reconciling before the cache is populated means every lister read is a spurious `NotFound`. This is the classic "my controller does nothing on startup then works after a restart" bug. `HasSynced` flips true once the **initial LIST** has been fully delivered into the indexer (not once the object is "up to date" forever — it never re-blocks after that). controller-runtime calls `WaitForCacheSync` for you before it starts any worker; if it returns false (context cancelled, RBAC denies the LIST/WATCH — a very common cause), the manager fails to start rather than reconciling blind. When debugging a dead controller, "can this SA even LIST/WATCH the type?" is the first check: no watch permission → cache never syncs → zero reconciles, often with only a quiet log line.

Note `AddEventHandler` now returns a `ResourceEventHandlerRegistration` and an error; you can later `RemoveEventHandler` it. Each handler also has its **own** `HasSynced` that only trips after the handler has seen the initial batch — relevant if you register a handler after the informer already started.

### Resync ≠ relist

The factory's resync period (`30s` above) is the most misunderstood knob. **Resync does not re-hit the API server.** At each interval the informer walks its *existing indexer contents* and re-delivers every object to your handlers as a synthetic **Update** (`old == new`). No LIST, no WATCH traffic — it's a local replay.

Why it exists: to paper over *your own* dropped work. If a reconcile failed in a way that didn't re-enqueue (a bug, a panic swallowed, a condition you couldn't converge yet), resync gives you a periodic "here's everything again, try once more" sweep. It is a safety net, not a data-freshness mechanism. A `resyncPeriod` of `0` disables it. Most controller-runtime controllers set a modest SyncPeriod (default ~10h) precisely because a good reconciler is level-based and re-enqueues its own retries — you don't want a tight resync flooding the workqueue with redundant work.

Contrast the three "reruns":
- **Watch event** — real change, from the apiserver.
- **Resync** — local replay of cache, no change, safety net.
- **Relist** (410) — reflector lost its place, full LIST refreshes the cache with real data.

### Handlers enqueue keys, not objects

Your `ResourceEventHandlerFuncs` (`AddFunc/UpdateFunc/DeleteFunc`) should do almost nothing: derive a **key** and enqueue it. Classic form is a string key via `cache.MetaNamespaceKeyFunc` (`"namespace/name"`); modern sample-controller uses `cache.ObjectToName` yielding a `cache.ObjectName{Namespace, Name}`. Either way you enqueue a **lightweight reference, never the object itself**:

```go
func (c *Controller) enqueue(obj interface{}) {
    ref, err := cache.ObjectToName(obj)   // {ns, name}, not the pod
    if err != nil { utilruntime.HandleError(err); return }
    c.workqueue.Add(ref)
}
```

Why keys and not objects? Three reasons, all central, and this is the deepest idea in the whole module — the one that makes the previous lesson's level-triggered contract *actually true* at the plumbing layer, not just a design intention:

1. **Dedup.** The workqueue collapses duplicate keys (see below). Ten changes to one object → one queue entry. Enqueue the object and you defeat that.
2. **Freshness.** When a worker finally pops the key, it fetches the *current* state from the lister. Enqueuing the object would freeze a stale snapshot at enqueue time.
3. **Level-triggered semantics.** You reconcile toward "what should be true for this name," not "replay this specific event." Missing an intermediate event is fine because you always read current state.

### The rate-limited workqueue

`workqueue.TypedRateLimitingInterface[T]` (recently made generic; the old `RateLimitingInterface` is a deprecated alias). It layers three behaviors:

- **Dedup + fairness (base queue):** an item present in the queue is stored once; a `dirty` set plus a `processing` set guarantee a key that's re-added *while a worker holds it* is re-queued exactly once *after* `Done()`, never processed by two workers at once. "Stingy": at-most-once concurrent processing per key.
- **Delaying:** `AddAfter(key, d)` holds a key for `d` before it becomes gettable.
- **Rate limiting:** `AddRateLimited(key)` asks a rate limiter `When(key)` for a delay, then `AddAfter`s it. The default is `DefaultTypedControllerRateLimiter[T]()` = `MaxOf(exponential-per-item, token-bucket-overall)`:
  - **Per-item exponential backoff:** `baseDelay × 2^(numRequeues)`, capped at `maxDelay` (defaults 5ms → 1000s). A key that keeps failing backs off further each time.
  - **Overall token bucket:** a global limiter (default 10 qps, burst 100) so *no matter how many distinct keys* are failing, the whole controller can't spin faster than the bucket allows.

The three methods that form the worker contract:

```go
func (c *Controller) processNextItem(ctx context.Context) bool {
    key, shutdown := c.workqueue.Get()
    if shutdown { return false }
    defer c.workqueue.Done(key)            // ALWAYS: releases the key for future re-adds

    if err := c.reconcile(ctx, key); err != nil {
        c.workqueue.AddRateLimited(key)    // failure: requeue with backoff
        return true
    }
    c.workqueue.Forget(key)                // success: reset this key's backoff counter to zero
    return true
}
```

- **`Done(key)`** — "I'm finished holding this key." Must be called (defer it) or the key is stuck in `processing` forever and never reprocesses. A wedged worker that never returns to call `Done` is a top cause of "controller stopped reconciling."
- **`AddRateLimited(key)`** — requeue after the backoff `When()` computes. Each call increments the key's requeue count → longer next delay.
- **`Forget(key)`** — clears the rate limiter's memory for that key so its *next* failure starts backoff from `baseDelay` again. **`Forget` does not remove the key from the queue** (that's `Done`); it only zeroes the backoff. Forgetting on success is what lets a normally-healthy object fail once, recover, and not carry a permanent penalty.

### Debug signals: where a stall actually hides

"Your controller stopped reconciling" resolves to one of a small set, and each has a signal:

- **Cache never synced** — RBAC on the watched type, or apiserver unreachable. Signal: `WaitForCacheSync` false / manager won't start; zero reconciles ever.
- **Workqueue depth climbing, no progress** — a wedged worker holding a key and never calling `Done` (a deadlock or infinite loop in `Reconcile`). Signal: `workqueue_depth` high and flat, `workqueue_unfinished_work_seconds` growing. This is why you never do blocking I/O without a context deadline in a reconciler.
- **Everything backing off** — a dependency (e.g. your pricing API) is down, so every reconcile errors and `AddRateLimited` pushes delays toward the 1000s cap. Signal: `workqueue_retries_total` spiking, `workqueue_longest_running_processor_seconds` fine, reconciles happening but slowly. Recovery is automatic once `Forget`-on-success resumes.
- **Watch silently wedged** — reflector connection half-open; no events arrive but no error either. Resync (or a controller-runtime SyncPeriod) is your backstop here, and a relist on the next 410 self-heals it.

client-go and controller-runtime both export these `workqueue_*` and `controller_runtime_reconcile_*` Prometheus metrics for free — wire them up in the capstone; they turn "it feels stuck" into a diagnosis.

### Why the workqueue decouples event rate from reconcile rate

This is the payoff. Events arrive at the *apiserver's* rate — a rollout can fire thousands of Pod updates in seconds. Reconciles happen at the *workqueue's* rate — bounded by worker count and the token bucket. The queue is the shock absorber: a storm of events for one object collapses to one key; a storm across many objects is drained at ≤ bucket qps; failing objects back off exponentially instead of hot-looping. Your `Reconcile` never sees the storm — it sees a deduped, rate-limited trickle of keys. Remove the workqueue and a CrashLooping dependency would have your controller pegging a CPU and DoSing the apiserver.

### Cache vs live reads, and staleness

The lister/indexer is *eventually* consistent. The window between "apiserver commits a write" and "your reflector's watch delivers it and the indexer updates" is real — usually milliseconds, but seconds under load or during a relist. Consequences:

- A lister read can return an object **older** than what a fresh `GET` would return, or return `NotFound` for something just created, or still return something just deleted.
- **Read-after-write of your own writes is not guaranteed from cache.** You `Update()` a Foo via the client, then immediately re-read from the lister and see the *old* version, because your own write hasn't looped back through the watch yet.
- For most reconcile logic this is fine — you're level-triggered and will get the update event momentarily. It bites when you make a decision that must reflect the absolute latest state (e.g. reading a `resourceVersion` for an optimistic-concurrency update, or a "did my finalizer land" check). For those, read **live**.

### How controller-runtime wraps all of this

controller-runtime hides the plumbing but it's the same machine:

- The **Manager** owns one shared **cache** (`sigs.k8s.io/controller-runtime/pkg/cache`) = a set of `SharedIndexInformer`s, started lazily per type when a controller `Watches`/`Owns` it.
- `mgr.GetClient()` returns a **split client**: **reads go to the cache** (the informer/indexer), **writes go straight to the API server**. So `r.Get(ctx, key, &obj)` is a *cache* read by default — same staleness rules as a lister.
- `Reconcile(ctx, req)` receives a `reconcile.Request{NamespacedName}` — that's the **key** popped off controller-runtime's internal rate-limiting workqueue. Same dedup, same exponential backoff. Returning `err` or `Result{Requeue: true}` → `AddRateLimited`; returning `Result{RequeueAfter: d}` → `AddAfter`; a clean `Result{}` → `Forget`.
- To bypass the cache and force a live read, use `mgr.GetAPIReader()` — an **`APIReader`** that talks directly to the apiserver, no cache. Use it sparingly (it defeats the whole point of the cache and adds apiserver load), only where you provably need the freshest state.

So everything in "Core concepts" is present in your kubebuilder controller — the manager just constructed the reflector, DeltaFIFO, informer, indexer, and workqueue for you.

## Perspectives

**Developer perspective.** The reflector/DeltaFIFO/indexer/workqueue pipeline is the single mental model that explains *every* client-go-based tool you'll ever debug — your CNI's controller, your CSI driver's, kube-controller-manager's, your own. Once it's internalized, reading an unfamiliar controller's source is a matter of finding where it wires up `AddEventHandler` and what it does inside `Reconcile`, not relearning a new architecture each time.

**Operator perspective.** `workqueue_depth`, `workqueue_unfinished_work_seconds`, `workqueue_retries_total` are metrics a CKA has scraped for years without necessarily knowing which specific stall pattern each one diagnoses; this lesson turns dashboards into causal reasoning — a high, flat `workqueue_depth` means something different from a climbing `workqueue_retries_total`, and now you know which.

**Systems/queueing-theory perspective.** The workqueue is a literal shock absorber: dedup collapses N events to 1 key, exponential backoff plus a global token bucket decouples event-arrival rate from reconcile-processing rate. This is the same shape as any bounded-queue backpressure system — worth naming as a transferable pattern you'll recognize in message brokers, rate-limited APIs, and load shedders elsewhere in your career.

**Economics perspective (module-specific).** A controller hot-looping because a worker never calls `Done()` isn't just a reliability bug on a GPU fleet; it can pin a CPU core on every controller replica across 40 clusters simultaneously, and if the reconcile loop does a paid external API call per iteration (e.g. a billing API), it can generate real API-cost overrun — a stuck workqueue on a cost operator can literally cost money in two different ways at once (wasted compute, and wasted paid API calls).

## Real-world use cases

- **"Scaling Kubernetes with Assurance at Pinterest"** — Pinterest Engineering. https://medium.com/pinterest-engineering/scaling-kubernetes-with-assurance-at-pinterest-a23f821168da — discusses watch-cache sizing and event retention for informer/watch reliability during connection churn at real production scale, directly on-topic for the reflector/relist material above.
- **"Kubernetes Informers are so easy... to misuse!"** — Render.com Engineering. https://render.com/blog/kubernetes-informers — a real infrastructure company's engineering blog covering common informer-misuse patterns in production controllers, a practitioner-level complement to the pitfalls below.
- **"Scaling Kubernetes to 7,500 nodes"** — OpenAI. https://openai.com/index/scaling-kubernetes-to-7500-nodes/ — explicitly discusses watch/list load on the apiserver from many controllers' informers at high node/pod counts, and the apiserver/etcd sharding decisions made to isolate that load — the production-scale version of this lesson's "thundering herd on informer-factory start" paragraph. (Dated snapshot: describes OpenAI's cluster architecture as of the 2023 post; cite as an engineering case study, not current headcount/scale.)

## Worked example

Trace a single Pod update — a GPU node's Pod gets a new `nvidia.com/gpu` annotation — from the API to your `Reconcile`, naming the client-go types:

1. **Write lands.** Something `PATCH`es the Pod. The apiserver commits it to etcd; the Pod's `resourceVersion` bumps.
2. **Reflector sees it.** Your process's `cache.Reflector` for Pods has an open WATCH from the last RV. The apiserver streams a `MODIFIED` event carrying the new Pod. (Had the watch been broken past etcd's history, the reflector would instead get `410 Gone` and do a full **relist** — same object arrives, via LIST.)
3. **Into the DeltaFIFO.** The reflector pushes an `Updated` delta for key `ns/pod` onto the `cache.DeltaFIFO`. If the Pod had changed twice since the last drain, both deltas sit under that one key, in order.
4. **Informer drains.** The `SharedIndexInformer`'s `Pop` loop takes the deltas, calls `indexer.Update(pod)` on the `cache.Indexer` (thread-safe store) so the cache now holds the new Pod, then fans the event out to registered handlers.
5. **Handler enqueues a key.** Your `UpdateFunc` runs. It does *not* inspect the Pod deeply — it computes `cache.ObjectToName(pod)` (or the owning GPUCostBudget's key via `handler.EnqueueRequestForOwner`) and calls `workqueue.Add(name)` on the `workqueue.TypedRateLimitingInterface`. Duplicate keys collapse here.
6. **Worker pops.** One of N worker goroutines calls `workqueue.Get()`, receives the key, and `defer workqueue.Done(key)`.
7. **Reconcile reads current state from cache.** The worker calls `podLister.Pods(ns).Get(name)` (raw client-go) or `r.Get(ctx, req.NamespacedName, &pod)` (controller-runtime) — a **cache read** off the indexer, returning the *now-current* Pod, not a stale snapshot from step 5. It computes cost, updates the GPUCostBudget status via a **direct API write**.
8. **Settle the queue.** Success → `workqueue.Forget(key)` (reset backoff) then `Done` fires via defer. Failure → `workqueue.AddRateLimited(key)` (exponential backoff) then `Done`. If the write hit a conflict and you need the freshest Pod to retry, re-read via the **`APIReader`**, not the cache.

**Raw client-go (sample-controller) vs controller-runtime:** the sample-controller writes steps 1–8 by hand — you construct the informer factory, the workqueue, the worker loop, and call `Get/Done/Forget/AddRateLimited` yourself (see `controller.go`). controller-runtime does steps 2–6 and 8's queue mechanics for you; you write only step 7's `Reconcile` body and declare the watches. Same types under the hood — `pkg/cache` builds the `SharedIndexInformer`s, `pkg/internal/controller` runs the workqueue loop.

**Secondary-index worked calc.** Suppose the same reconcile also needs "every other Pod on this node" to compute node-level utilization. Without an index: `podLister.List(labels.Everything())` returns all cached Pods — at 5,000 Pods, that's a slice of 5,000 objects the reconcile then filters client-side by `spec.nodeName == "gpu-node-7"`, roughly 5,000 comparisons per call. With `AddIndexers({"byNode": ...})` registered once at startup: `informer.GetIndexer().ByIndex("byNode", "gpu-node-7")` returns only the matching Pods directly, a map lookup plus a slice append for the matches — no scan of the other 4,990+ objects. The index trades a small amount of memory (one map, maintained incrementally) for turning an O(N) filter into an O(1) lookup on every reconcile that needs "Pods on this node" — worth registering the moment more than one reconcile path needs the same reverse lookup.

## Practice

Instrument your `gpu-cost-operator` (or the module-01 controller if the operator isn't scaffolded yet) and record what you observe. Commit a short findings note (`docs/informer-observations.md` in the practice dir).

1. **Cache-hit vs live read.** In `Reconcile`, do the normal cached `r.Get(...)`, then also fetch the same object via `mgr.GetAPIReader().Get(...)`. Log both `resourceVersion`s side by side. Do a rapid self-write (update status, then immediately reconcile) and catch a case where the cached RV lags the live RV. Record the divergence.
2. **Short resync.** Set the manager's `SyncPeriod` (or a raw informer factory's resync) to `10s`. Log a distinctive line at the top of `Reconcile`. Confirm you see periodic reconciles for *every* object with **no corresponding apiserver watch event** (check with `kubectl get --watch` in another pane, or apiserver audit) — proving resync is a local replay, not a re-LIST.
3. **Force a stale-cache scenario.** Either: (a) add an artificial delay/pause and `kubectl edit` the object during it, then observe your cached read returning the pre-edit version while `APIReader` shows the new one; or (b) delete an object and log whether the lister still returns it for a beat. Note the staleness window you measured.
4. **Register and measure a field index.** Add a `byNode` field indexer via `mgr.GetFieldIndexer()`, then log and compare wall-clock time for a `client.MatchingFields{"spec.nodeName": x}` indexed list vs an unindexed `List()` + client-side filter, at whatever Pod count your test cluster has. Even a small cluster should show the indexed path doing measurably less work.
5. **Workqueue under stress (optional but recommended).** Make `Reconcile` return an error for one specific object and watch the requeue timestamps in the logs — confirm the delays grow (5ms, 10ms, 20ms…) i.e. exponential backoff, and that `Forget` (returning nil) resets it.

**Acceptance:** a committed `informer-observations.md` describing the observed cache-vs-live RV divergence, the resync replay behavior, a measured staleness window, the indexed-vs-unindexed lookup comparison, and (if done) the backoff progression — plus the instrumentation code committed alongside.

## Common pitfalls

1. **Believing a short resync period improves data freshness.** It doesn't touch the API server at all; it's a local safety-net replay. Setting it too low just adds redundant reconcile load without making any data fresher.
2. **Mutating a lister-returned object in place instead of `DeepCopy()`-ing first.** This corrupts shared cache state other goroutines are reading concurrently — a genuinely hard-to-reproduce bug class because it depends on goroutine timing.
3. **Enqueuing the object itself instead of its key.** This defeats dedup and processes a stale snapshot instead of current state at pop-time — silently reintroducing the edge-triggered bugs lesson 03 warned about, at the plumbing layer.
4. **Assuming a `Get` right after your own `Create`/`Update` will reflect it.** Cache reads lag your own writes; use `APIReader` when you truly need read-after-write consistency, not routine reconcile logic.
5. **Not calling `WaitForCacheSync` (or ignoring its `false` return).** Reconciling before the cache is populated looks like "the controller does nothing on startup" — a confusing symptom for a simple ordering bug.

## Self-check

- **When can a controller-runtime client read return stale data, and how do you force a live read?**
  **Answer:** `mgr.GetClient()` is a split client whose **reads are served from the manager's shared cache** (the `SharedIndexInformer`'s indexer), which is eventually consistent — it lags the apiserver by the watch-delivery window. So `r.Get`/`r.List` can return an object older than a fresh `GET`, return `NotFound` for something just created, still return something just deleted, or fail to reflect **your own write** that hasn't looped back through the watch yet. To force a live read, use `mgr.GetAPIReader()` — an `APIReader` that goes straight to the apiserver, bypassing the cache. Use it only where you truly need the freshest state (optimistic-concurrency retries, "did this land" checks); routine reconcile logic should stay on the cache since it's level-triggered and will get the update event momentarily.

- **What does the resync period actually re-trigger, and why isn't it a re-LIST?**
  **Answer:** Resync walks the informer's **existing indexer contents** and re-delivers every cached object to your event handlers as a **synthetic Update** (`old == new`). It generates **zero apiserver traffic** — no LIST, no WATCH — it's a purely local replay of the cache. It isn't a re-LIST because its purpose isn't refreshing data (the watch already keeps the cache current); it's a **safety net** that periodically re-enqueues everything so a reconcile that silently failed to converge or re-enqueue itself gets another attempt. A re-LIST (which *does* hit the apiserver) only happens on reflector startup or on `410 Gone`, when the reflector has actually lost its watch position.

- **How does the workqueue's dedup + rate limiter keep a controller stable under an event storm, and where does `Forget()` fit?**
  **Answer:** **Dedup** collapses the storm: because handlers enqueue *keys*, N events for one object become one queue entry, and a key already being processed is re-queued exactly once (after `Done`), never processed concurrently. **Rate limiting** bounds throughput: `AddRateLimited` applies per-item exponential backoff (`base × 2^requeues`, capped) so a persistently failing object retries ever more slowly, plus a global token bucket (default 10 qps / burst 100) so the whole controller can't exceed a fixed reconcile rate no matter how many distinct keys are failing. Together they decouple the reconcile rate from the event rate — `Reconcile` sees a deduped, rate-limited trickle instead of the storm. **`Forget(key)`** is called on **success**: it clears that key's backoff counter in the rate limiter (resetting its next-failure delay to `baseDelay`) so a once-flaky object that recovers doesn't carry a permanent penalty. Note `Forget` only zeroes backoff state — `Done` is what releases the key from processing; you call both, and forgetting without success would defeat the backoff.

- **Why is enqueuing a *key* rather than the *object* central to level-triggered correctness, not just an optimization?**
  **Answer:** It's not just cheaper — it's what makes level-triggering *true* at runtime. If you enqueued the object, you'd freeze a snapshot at enqueue time and the worker would reconcile that stale snapshot instead of current state, which reintroduces edge-triggered behavior through the back door (you'd effectively be reacting to "the object as of when I saw the event," not "the object as of now"). Enqueuing a key forces the worker to re-fetch from the lister/cache at pop-time, guaranteeing it always reconciles against the *current* state regardless of how many changes happened in between or how long the key sat in the queue. It also enables dedup — the workqueue can only collapse duplicate *keys*, not duplicate objects, since two enqueues of "the same object" at different points in time aren't equal as values.

- **At 5,000 Pods in cache, contrast the cost of a plain `List()` with a label filter vs a registered field-index lookup for "all Pods on node X."**
  **Answer:** A plain `List()` (with or without a label selector applied client-side against `spec.nodeName`) means the lister returns the full cached set — approximately 5,000 objects — and your code then filters them one by one, roughly 5,000 comparisons per call, repeated on every reconcile that needs the answer. A registered field index (`AddIndexers`/`mgr.GetFieldIndexer().IndexField` on `spec.nodeName`) maintains a secondary map from node name to the set of matching keys, updated incrementally as Pods are added/updated/deleted in the cache; `ByIndex("byNode", "X")` or `client.MatchingFields{"spec.nodeName": "X"}` then does a single map lookup and returns only the matching Pods directly — no scan of the other ~4,995 objects. The tradeoff is a small constant memory overhead for the index versus a linear-time cost paid on every unindexed lookup; for any reconcile pattern that does this lookup repeatedly (which a per-node cost aggregation does), the index is a clear win.

## Connections & what's next

This lesson is the machinery that makes lesson 03's reconciliation contract real: the reflector/DeltaFIFO/indexer/workqueue pipeline is *why* reconcile receives a key instead of an event, *why* a cache read can be stale, and *why* backoff behaves the way it does. It's also the layer every subsequent lesson quietly assumes: lesson 06's owner-reference garbage collector is, under the hood, just another controller running this exact reflector→informer→workqueue loop watching every type in the cluster; lesson 08's admission webhooks sit in a different path entirely (synchronous, in the write path) precisely because informers are inherently asynchronous and can't gate an in-flight write. Next, **lesson 05, CRD design**, turns to the shape of the objects flowing through this pipeline — how a well-designed schema (validation, subresources, versioning) makes the reconcile loop you just learned to trust actually safe to build on.

## References & further reading

**Primary sources**
- client-go `tools/cache` godoc — https://pkg.go.dev/k8s.io/client-go/tools/cache — the authoritative reference for `Reflector`, `DeltaFIFO`, `SharedIndexInformer`, `Indexer`, and the key funcs; the exact resync/relist/410 semantics covered above.
- client-go `informers` godoc — https://pkg.go.dev/k8s.io/client-go/informers — the shared-informer-factory API.
- client-go `util/workqueue` godoc — https://pkg.go.dev/k8s.io/client-go/util/workqueue — `TypedRateLimitingInterface`, `DefaultTypedControllerRateLimiter`, and the exponential/token-bucket rate limiters; the workqueue contract (`Get/Done/Forget/AddRateLimited`) you must be able to recite.
- controller-runtime `pkg/cache` godoc — https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache — confirms the manager's cache is wrapped `SharedIndexInformer`s and locates `APIReader`/`GetAPIReader` semantics.
- kubernetes/sample-controller — https://github.com/kubernetes/sample-controller — the canonical hand-written version of everything controller-runtime hides: factory setup, `WaitForCacheSync`, the worker loop, `enqueue`, and `Get/Done/Forget/AddRateLimited`. Read `controller.go` in full once.

**Real-world engineering blogs**
- Pinterest Engineering, "Scaling Kubernetes with Assurance at Pinterest" — https://medium.com/pinterest-engineering/scaling-kubernetes-with-assurance-at-pinterest-a23f821168da — watch-cache sizing and event retention in a real production fleet.
- Render.com Engineering, "Kubernetes Informers are so easy... to misuse!" — https://render.com/blog/kubernetes-informers — real-world informer-misuse patterns from a production infrastructure company.
- OpenAI, "Scaling Kubernetes to 7,500 nodes" — https://openai.com/index/scaling-kubernetes-to-7500-nodes/ — apiserver/etcd load from many informers' watch/list traffic at very high node/pod counts (2023 snapshot).

**Deeper dives**
- leftasexercise.com, "Understanding Kubernetes controllers part I – queues and the core controller loop" — https://leftasexercise.com/2019/07/08/understanding-kubernetes-controllers-part-i-queues-and-the-core-controller-loop/ — an individual engineer's deep technical walkthrough of the workqueue.
- leftasexercise.com, "Understanding Kubernetes controllers part III – informers" — https://leftasexercise.com/2019/07/15/understanding-kubernetes-controllers-part-iii-informers/ — the companion piece on informers specifically.
