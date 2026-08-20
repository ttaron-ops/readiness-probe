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
sources: 15
---

# 02.4 · Informers, caches, and workqueues

> **Concept.** The reflector → Delta FIFO → shared informer → indexer/lister → rate-limited workqueue → worker machine that every controller runs on — and its failure modes.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)
>
> **Advanced Learning** — [Informers, Caches and Workqueues](../../../learning/04-informers-caches-workqueues.html): the six-stage pipeline end to end, and the queue/dirty/processing state grid behind one permanently deaf key. Optional visual companion; this lesson stays the source of truth.

## Where this fits

Lesson 03 gave you a contract you had to take on faith: reconcile receives a *key*, not an event; the workqueue is allowed to coalesce, drop, reorder, and replay; a cache read can be stale. This lesson opens that box. Every one of those behaviors is the observable effect of five concrete, nameable components — reflector, FIFO, indexer, lister, workqueue — wired together in a specific way. Once you can name the box each behavior comes from, "my controller is stuck" stops being a mystery and becomes a short diagnostic checklist. This is also the last purely-mechanical lesson before the module turns to what you build *on top of* this machine: CRD schemas (05), controller-runtime's finalizer/GC layer (06), and RBAC (07) all assume you already understand how a watch becomes a reconcile.

## Why this matters

This is the single highest-ROI internal you will learn in this module. Every controller you have ever run — kube-controller-manager, your CNI, your GPU device plugin's operator — is this exact machine repeated. Two things fall out of understanding it:

1. **Interviews.** "Walk me through what an informer does" and "your controller stopped reconciling — how do you debug it" are standard senior-platform probes. The bad answer is "it watches the API." The good answer names the reflector, the FIFO, the thread-safe indexer, the lister, and the rate-limited workqueue, and can explain where a stall hides (cache not synced, workqueue backing off, a wedged worker holding `Done()`).
2. **Correctness.** Nearly every subtle operator bug — reconciling on stale data, hot-looping under an event storm, "it works but lags 30s" — is a misunderstanding of cache reads vs live reads, resync semantics, or the workqueue's backoff. You cannot write a *correct* GPU cost operator without this; a cost number computed from a stale cache is a wrong invoice.

## What's new here (calibration)

As a CKA you have a true but lossy mental model: *"the controller watches resources and acts on changes."* That sentence hides the entire engineering problem, and we won't re-litigate kubectl-level watch usage (`kubectl get --watch`, YAML basics) or CNI/CSI watch behavior — you've run those for years. A naive watcher would: hammer the API server with a `GET` per reconcile; lose events during a disconnect; melt the apiserver when 5,000 pods churn at once; and reprocess the same object ten times concurrently.

The client-go machinery exists to solve exactly those four problems:

| Operating view | Extending view (what's actually there) |
|---|---|
| "watches resources" | a **reflector** doing one LIST then a long-lived WATCH, tracking `resourceVersion`, relisting on `410 Gone` |
| "keeps a copy" | a **FIFO** (DeltaFIFO, or RealFIFO in current releases) feeding a **thread-safe indexer**; you read it through a **lister**, never the API |
| "acts on changes" | handlers enqueue **keys** into a **rate-limited workqueue**; **workers** pull keys and reconcile |
| "reruns periodically" | **resync** re-delivers cached objects as synthetic updates — it is *not* a re-LIST |

Learn the named parts and the whole thing stops being magic.

## Core concepts

**Versions this lesson's API claims apply to.** Everything below was read out of **client-go v0.36.3** (the client library shipped with Kubernetes 1.36) and **controller-runtime v0.24.1** (the release aligned to Kubernetes 1.36), plus the Kubernetes 1.36 API-concepts reference. Where a behaviour changed recently — and one big one did, in 1.33 — the lesson says so and gives both the old and the new shape, because you will read older code and older blog posts that describe the old one.

### 0. The problem the machinery solves

Start from the naive controller, because every piece below is a fix for one of its failures.

```go
// The controller you would write on day one. Do not ship this.
for {
    pods, _ := clientset.CoreV1().Pods("").List(ctx, metav1.ListOptions{})
    for _, pod := range pods.Items {
        reconcile(&pod)
    }
    time.Sleep(10 * time.Second)
}
```

Four things are wrong, and each names a component:

1. **It re-LISTs the world every 10s.** At 20,000 Pods that is a full serialize-and-transfer of the whole collection, from every controller, forever. *Fix: LIST once, then WATCH — the **reflector**.*
2. **It has no memory across iterations,** so it can't tell a change from a no-change, and every consumer of "the current set of Pods" pays another LIST. *Fix: keep a local replica — the **indexer/store**, read through a **lister**.*
3. **A burst of 5,000 Pod updates produces 5,000 reconciles,** many of them for the same object, all racing. *Fix: enqueue **keys** into a deduplicating **workqueue**.*
4. **A permanently-failing object retries as fast as the loop spins,** pegging a CPU and hammering the apiserver. *Fix: **rate limiters** on the workqueue.*

The whole pipeline, one type at a time:

```
                      the ONLY component that talks to the apiserver
   kube-apiserver                     │
   ──────────────                     ▼
        │  LIST ?resourceVersion=0  ┌───────────────────────────────┐
        │  (served from watch cache)│  Reflector                    │
        ├──────────────────────────▶│  • ListAndWatch loop          │
        │                           │  • tracks lastSyncResourceVer │
        │  WATCH ?resourceVersion=N │  • timeout rand[5m,10m)       │
        │◀──────────────────────────│  • on 410 Gone: RV="" relist  │
        │  ADDED/MODIFIED/DELETED   │  • backoff 800ms→30s, jitter  │
        │  BOOKMARK                 └───────────────┬───────────────┘
        │                                           │ Add/Update/Delete/Replace
        │   ┌───────────────────────────────────────▼───────────────┐
        │   │  FIFO                                                 │
        │   │  ≤1.32 default: DeltaFIFO  items map[key]Deltas       │
        │   │                            queue []key  (per-key)     │
        │   │  ≥1.33 default: RealFIFO   items []Delta (flat, in    │
        │   │                            watch-stream order)        │
        │   └───────────────────────────────┬───────────────────────┘
        │                                   │ Pop(process)
        │   ┌───────────────────────────────▼───────────────────────┐
        │   │  sharedIndexInformer.handleDeltas / processDeltas     │
        │   │  1. write the store   2. THEN fire handlers           │
        │   └──────┬────────────────────────────────┬───────────────┘
        │          │ store write                    │ fan-out
        │          ▼                                ▼
        │   ┌──────────────┐              ┌────────────────────────┐
        │   │ Indexer      │              │ sharedProcessor        │
        │   │ threadSafeMap│              │  one processorListener │
        │   │  items map   │              │  per handler, each with│
        │   │  storeIndex  │              │  an UNBOUNDED pending  │
        │   └──────┬───────┘              │  notification buffer   │
        │          │                      └───────────┬────────────┘
        │   Lister │ cache reads                      │ OnAdd/OnUpdate/OnDelete
        │  (Get/List/ByIndex)                         │ derive KEY, enqueue
        │          │                                  ▼
        │          │                  ┌───────────────────────────────┐
        │          └─────────────────▶│  RateLimiting workqueue       │
        │            worker re-reads  │   queue []T · dirty · process │
        │            current state    │   + delaying heap (AddAfter)  │
        │                             │   + rate limiter (When)       │
        │                             └───────────────┬───────────────┘
        │                                             │ Get()
        │                       N worker goroutines ──┴──▶ your Reconcile
        │                                             │
        └────── writes go DIRECT to the apiserver ────┘
                (never through the cache)
```

Everything below names one box in that diagram.

### 1. The reflector: LIST then WATCH

A `Reflector` (`k8s.io/client-go/tools/cache`) is the only component that talks to the API server. It owns a `ListerWatcher` (a pair of `List` and `Watch` closures for one resource) and a `ReflectorStore` (in practice, the FIFO). Its `Run` method loops `ListAndWatch` forever under a backoff manager.

**The `ListAndWatch` algorithm, as it actually runs:**

1. **Choose a starting resourceVersion.** `relistResourceVersion()` returns:
   - `"0"` on the very first list — meaning "*Any* resource version; newest preferred but staleness tolerated." The apiserver **always serves this from its in-memory watch cache**, never from etcd. That is deliberate: it makes controller startup cheap for the control plane.
   - the last observed resourceVersion on a normal re-entry (`NotOlderThan` semantics — also served from the watch cache).
   - `""` after a `410 Gone` — meaning "*Most recent*", a consistent read. Historically that was a quorum read straight out of etcd; since Kubernetes 1.31 (with etcd ≥ 3.4.31 / ≥ 3.5.13) the apiserver can serve "most recent" from the watch cache too, using an etcd progress notification to prove the cache is current.
2. **LIST.** The reflector pages through the collection with `k8s.io/client-go/tools/pager`. There is a subtlety worth knowing: when `resourceVersion` is set to something other than `"0"`, the reflector deliberately sets `pager.PageSize = 0` — **pagination off** — because a paginated list *skips the watch cache* and goes to etcd. Turning paging off to force a watch-cache read is the opposite of the usual instinct, and the code comment says why: "to avoid thundering herd on etcd (e.g. on master upgrades)."
3. **Replace the store.** The full listed set is handed to `store.Replace(list, rv)`. This is not a merge — see §2, it also *synthesises deletions* for anything the store knew about that the list did not contain.
4. **WATCH from that RV.** The request carries a randomised `timeoutSeconds` drawn uniformly from `[minWatchTimeout, 2 × minWatchTimeout)` — default `[5 min, 10 min)` (`defaultMinWatchTimeout = 5 * time.Minute`). The randomisation is load-spreading: without it, every reflector in the fleet that started together would re-establish its watch at the same instant forever.
5. **Feed every event into the store** as an `Added` / `Updated` / `Deleted` delta, updating `lastSyncResourceVersion` from each event's object.
6. **On watch termination or error, go back to step 1** — with backoff.

**The backoff numbers, exactly.** The reflector's `Run` loop is wrapped in an exponential backoff with `Duration: 800ms`, `Cap: 30s`, `Factor: 2.0`, `Jitter: 1.0`, and a reset after `2 minutes` without a backoff. The client-go source states the intent in a comment: it "used to make the call every 1sec (1 QPS)"; these parameters "achieve ~98% traffic reduction when API server is not healthy," settling into a `[30, 60)`-second retry interval — about **0.22 QPS per reflector**. That number matters at fleet scale: if the apiserver falls over, 2,000 reflectors across your clusters retry at 0.22 QPS ≈ 440 requests/second in aggregate, not 2,000/s. The backoff is what makes an apiserver outage recoverable instead of self-sustaining.

**The `410 Gone` case is the one to memorise.** The apiserver only retains recent change history — for etcd-3-backed clusters the documented default is the **last 5 minutes**. If your watch falls behind that window (a long disconnect, a slow consumer, an etcd compaction), resuming from your old RV returns `410 Gone` / "too old resource version." The reflector's `isExpiredError` catches it, sets `isLastSyncResourceVersionUnavailable = true`, and the next `relistResourceVersion()` returns `""` — forcing a fresh, consistent LIST. This is normal and healthy, but a relist re-delivers *everything*, which is why a flapping watch shows up as periodic reconcile spikes.

**Bookmarks shrink the 410 window.** A `BOOKMARK` watch event carries only `.metadata.resourceVersion` and means "you have seen everything up to here." The reflector sets `allowWatchBookmarks=true`, so even a watch on a quiet resource keeps advancing its RV and stays inside the 5-minute history window. Without bookmarks, a Pod-less namespace's watch would sit at a stale RV and 410 on every reconnect.

**Streaming lists (`WatchList`) change the shape of step 2.** Since client-go 1.30 the reflector can skip the separate LIST entirely: `GET ...?watch=1&sendInitialEvents=true&allowWatchBookmarks=true&resourceVersion=&resourceVersionMatch=NotOlderThan` streams the initial state as synthetic `ADDED` events followed by a `BOOKMARK`, then continues as a normal watch. The point is apiserver *memory*: a classic LIST materialises the entire collection in the apiserver's heap before writing it out; a streaming list emits objects one at a time. The client-go feature gate is `WatchListClient` — **beta and enabled by default since client-go 1.35** — with automatic fallback to a regular list if the server does not support it (`fallbackToList = true` in `ListAndWatchWithContext`).

Here is the same reflector as a timeline, with all three "rerun" paths drawn:

```
 t ──────────────────────────────────────────────────────────────────────▶

 START
  │ LIST rv="0"  (watch cache, ~34 MB for 20k pods)
  ├──────────────▶ store.Replace(all)     ── initialPopulationCount = N
  │                                          HasSynced flips true only after
  │                                          the Pop loop drains all N
  │ WATCH rv=10245  timeout=rand[5m,10m)
  ├──▶ MODIFIED  rv=10596 ─▶ Updated delta ─▶ store.Update ─▶ OnUpdate
  ├──▶ ADDED     rv=10601 ─▶ Added   delta ─▶ store.Add    ─▶ OnAdd
  ├──▶ BOOKMARK  rv=12746 ─▶ (advances lastSyncResourceVersion only)
  │
  │ ══ resync timer fires (local only, ZERO apiserver traffic) ═══════════
  ├──▶ store.Resync() ─▶ Sync delta per known key ─▶ OnUpdate(old==new)
  │
  │ ══ watch server-side timeout at ~7m ══════════════════════════════════
  ├──▶ watch closes cleanly ─▶ re-WATCH rv=12746          (cheap, no LIST)
  │
  │ ══ long disconnect: apiserver history window (5 min) exceeded ════════
  ├──▶ WATCH rv=12746 ─▶ 410 Gone "too old resource version"
  │    isLastSyncResourceVersionUnavailable = true
  ├──▶ RELIST rv=""  (consistent read)  ─▶ store.Replace(all)
  │        └── Replace() synthesises Deleted deltas (DeletedFinalStateUnknown)
  │            for every key the store knew about but the new list lacks
  │            → your DeleteFunc fires for objects deleted while you were away
  │
  │ ══ apiserver unreachable ═════════════════════════════════════════════
  └──▶ ListAndWatch returns err ─▶ backoff 0.8s,1.6s,3.2s…30s (jitter 1.0)
       ─▶ steady state ≈ [30s,60s) ≈ 0.22 QPS per reflector; resets after
          2 minutes of success
```

**Thundering herd on informer start, with numbers.** A GPU-fleet Pod — one training worker with the labels, annotations, tolerations, env vars and container statuses you actually see in production — serialises to about **1,730 bytes of JSON** or **1,074 bytes of protobuf** (measured on the `corev1.Pod` type from `k8s.io/api` v0.36.3). At 20,000 Pods that is **≈34.6 MB per full LIST in JSON, ≈21.5 MB in protobuf**, per watching process. If a rolling deploy restarts 200 controller replicas across your fleet inside the same minute, that is **≈6.9 GB** the apiservers must fetch-from-cache, serialize, and write — concentrated at the moment the cluster is already churning. OpenAI's 7,500-node writeup gives the production-scale version of this: they observe **up to 70 GB of heap per API server** at that cluster size, and run **5 apiservers and 5 etcd nodes** on their largest clusters specifically to spread and isolate that load. Their sharpest finding is the same shape — WATCHes on `Endpoints` for cluster-wide services (kubelet, node-exporter) fire on every node add/remove, and because each node is itself watching, the bandwidth is **O(N²)** in node count.

Practical takeaways for your operator: stagger controller restarts where you can; scope caches to the namespaces and label selectors you actually need (lesson 06); and do not be surprised when "everything reconciles slowly for the first 30 seconds after a fleet-wide rollout" turns out to be every cache refilling at once.

### 2. The FIFO between reflector and store — and the 1.33 change nobody told you about

The reflector does not write the indexer directly. It writes a queue, and the informer drains that queue. **Which queue depends on your client-go version, and the answer changed recently.**

#### DeltaFIFO (the classic, still what every blog post describes)

`DeltaFIFO` is keyed by object, with the value being an *ordered list of deltas* for that object:

```go
type DeltaFIFO struct {
    items map[string]Deltas  // key "ns/name" -> ordered deltas
    queue []string           // FIFO order of keys; no duplicates
    ...
}
type Delta struct { Type DeltaType; Object interface{} }
type Deltas []Delta          // oldest at [0], newest last
```

Delta types: `Added`, `Updated`, `Deleted`, `Replaced`, `ReplacedAll`, `Sync`, `SyncAll`, `Bookmark`.

Three properties that are widely misstated:

- **Per-key accumulation, *not* compression.** If a Pod changes five times before the consumer drains it, `items["ns/pod"]` holds five deltas and `Pop` returns all five in one `Deltas` value. They are not merged. The often-repeated "the DeltaFIFO dedupes events" is only true in one narrow case: `dedupDeltas` collapses the **last two deltas only, and only when both are deletions** (`isDeletionDup`), preferring the one carrying a real object over a `DeletedFinalStateUnknown` tombstone. That is the entire dedup logic. Read it and you will never overclaim again.
- **Deletes are never silently lost.** A missed delete during a disconnect is recovered by `Replace()`: for every key in the store (or in `knownObjects`) that is *not* in the new list, it enqueues `Deleted{DeletedFinalStateUnknown{key, lastKnownObject}}`. That is why your `DeleteFunc` must handle a `DeletedFinalStateUnknown` wrapper and not blindly type-assert to `*v1.Pod` — a classic panic in first-time controllers.
- **Resync is per-key and skips busy keys.** `Resync()` walks `knownObjects.ListKeys()` and calls `syncKeyLocked`, which enqueues a `Sync` delta **only if `len(f.items[key]) == 0`** — if that object already has pending deltas, the resync for it is dropped, because the real event carries fresher data.

`Pop` removes a key from `queue`, deletes it from `items`, and calls `process(deltas, isInInitialList)` **while holding the FIFO lock**. That lock is why a slow event handler stalls the whole informer, and why client-go emits a trace when the queue is deeper than 10 and one item takes over 100 ms, with the literal reason string `"slow event handlers blocking the queue"` (added in response to kubernetes/kubernetes#103789, where a reporter saw client-go informer memory reach **2.3 GB after 30 minutes** of pods being created and deleted every 10 seconds).

#### RealFIFO (what a current informer actually runs)

The `DeltaFIFO`'s per-key queue has a consequence people rarely think about: **it reorders events across objects.** A key already sitting in `queue` keeps its original queue position while newer deltas pile onto it, so object B's change at t=2 can be delivered before object A's change at t=3 even though A's *earlier* change at t=1 is what put A in the queue. Handlers that reason about relative ordering between objects get it wrong.

The `InOrderInformers` feature gate fixes that by swapping the queue for `RealFIFO`:

```go
// k8s.io/client-go/tools/cache/controller.go — newQueueFIFO
if clientgofeaturegate.FeatureGates().Enabled(clientgofeaturegate.InOrderInformers) {
    f := NewRealFIFOWithOptions(...)   // ← the default path today
} else {
    f := NewDeltaFIFOWithOptions(...)  // ← the legacy path
}
```

`RealFIFO` is a flat slice:

```go
type RealFIFO struct {
    items []Delta   // strictly watch-stream order; NO per-key grouping
    ...
}
```

Its doc comment states the contract: "every notification from the Reflector is passed in order to the Queue via `Pop`… it delivers multiple notifications per item instead of simply the most recent value." No per-key accumulation, no cross-object reordering, one delta per `Pop`.

**Timeline of the gate** (from `k8s.io/client-go/features/known_features.go`):

| Feature gate | client-go / k8s version | Default | Effect |
|---|---|---|---|
| `InOrderInformers` | 1.33 | Beta, **on** | shared informers use `RealFIFO` instead of `DeltaFIFO` |
| `InOrderInformers` | 1.36 | **GA, locked on** | `DeltaFIFO` is no longer reachable from a shared informer |
| `InOrderInformersBatchProcess` | 1.35 | Beta, on | `Pop` can hand the informer a batch of deltas in one transaction |
| `AtomicFIFO` | 1.36 | Beta, on | `Replace`/`Resync` emit a single `ReplacedAll` / `SyncAll` event instead of N |
| `UnlockWhileProcessingFIFO` | 1.36 | Beta, on | `Pop` may release the FIFO lock while the handler runs, so bursts are not dropped |
| `WatchListClient` | 1.30 beta / 1.35 default on | on | reflector uses streaming lists |

**What this means for you, concretely.** On any cluster running Kubernetes 1.33 or newer, do not tell an interviewer that "the DeltaFIFO coalesces the five updates into one." It does not, and on 1.36 the DeltaFIFO is not even in the path. The dedup you are relying on is the **workqueue's**, not the FIFO's — which is exactly why §6's "enqueue keys, not objects" matters so much. Say instead: *the FIFO preserves every notification in order; deduplication happens one layer later, in the workqueue, on keys.*

### 3. The Pop loop: store first, handlers second

Whichever FIFO is in use, the `sharedIndexInformer` supplies the `process` function. Its core is `processDeltas` (`tools/cache/controller.go`), and the ordering inside it is load-bearing:

```go
for _, d := range deltas {           // oldest to newest
    obj := d.Object
    switch d.Type {
    case Sync, Replaced, Added, Updated:
        if old, exists, err := clientState.Get(obj); err == nil && exists {
            clientState.Update(obj)          // 1. STORE FIRST
            handler.OnUpdate(old, obj)       // 2. then notify
        } else {
            clientState.Add(obj)
            handler.OnAdd(obj, isInInitialList)
        }
    case Deleted:
        clientState.Delete(obj)
        handler.OnDelete(obj)
    case Bookmark:
        clientState.Bookmark(info.ResourceVersion)
    }
}
```

**Read the order once more: the store is written before the handler fires.** That is the guarantee that makes "enqueue a key, then read the cache in the worker" correct. By the time your `UpdateFunc` runs, a lister read for that key already returns the *new* object. Without this ordering, every controller would race its own event.

Note also that `Added` and `Updated` are not trusted: the switch decides `OnAdd` vs `OnUpdate` by whether the object is *already in the store*, not by the delta type. A `Replaced` delta after a relist therefore surfaces as `OnUpdate` for objects you already had and `OnAdd` for ones you did not — which is the correct semantic and another reason not to write handlers that branch on "which event fired."

**Handler fan-out and the unbounded buffer.** "Shared" in `SharedIndexInformer` means one reflector and one store serve many handlers. Each registered handler gets a `processorListener` with two goroutines, `pop()` and `run()`, joined by an **unbounded growing ring buffer** (`pendingNotifications`). `pop()` never blocks the informer — it appends to the ring instead. That is a deliberate trade: **a slow handler cannot stall the informer, but it can grow memory without bound.** If your `UpdateFunc` does an HTTP call, you have built a memory leak whose rate is (event rate − handler rate) × object size. This is the mechanism behind #103789's 2.3 GB. Handlers must do essentially nothing but compute a key and `Add` it.

**`HasSynced` and `WaitForCacheSync`.** `Replace()` sets `initialPopulationCount = len(keys) + queuedDeletions`. Each `Pop` decrements it; when it hits zero the `synced` channel closes and `HasSynced()` returns true. So **`HasSynced` means "the initial LIST has been fully delivered *through* the handlers," not "the LIST returned."** `WaitForCacheSync` polls those functions every `syncedPollPeriod = 100ms` until all are true or the stop channel closes.

```go
factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)
podInformer := factory.Core().V1().Pods()
lister := podInformer.Lister()      // cache reads
inf := podInformer.Informer()       // AddEventHandler lives here
reg, err := inf.AddEventHandler(cache.ResourceEventHandlerFuncs{ /* ... */ })
if err != nil { return err }
factory.Start(ctx.Done())           // launches reflectors
if !cache.WaitForCacheSync(ctx.Done(), inf.HasSynced) {
    return fmt.Errorf("failed to sync pod cache")
}
defer inf.RemoveEventHandler(reg)
```

`WaitForCacheSync` is load-bearing: reconciling before the cache is populated means every lister read is a spurious `NotFound`. This is the classic "my controller does nothing on startup then works after a restart" bug. controller-runtime calls it for you before starting any worker; if it returns false — context cancelled, or **RBAC denies LIST/WATCH on the type**, which is by far the most common cause — the manager fails to start rather than reconciling blind. When debugging a dead controller, "can this ServiceAccount even LIST and WATCH this type?" is the first check: no watch permission → cache never syncs → zero reconciles, often with only one quiet log line.

`AddEventHandler` returns a `ResourceEventHandlerRegistration` and an error, and each registration has its **own** `HasSynced` that only trips once *that handler* has seen the initial batch — relevant if you register a handler after the informer already started, since `blockDeltas` is taken to replay the current store contents into it.

### 4. The indexer: what the cache actually is

The `Indexer` is a `threadSafeMap`:

```go
type threadSafeMap struct {
    lock  sync.RWMutex
    items map[string]interface{}   // "ns/name" -> runtime.Object
    index storeIndex
}
type storeIndex struct {
    indexers Indexers                       // name -> IndexFunc
    indices  map[string]map[string]sets.Set[string]  // indexName -> indexedValue -> keys
}
```

- **Reads are `RLock`-protected map lookups.** `Get` is O(1); `List` copies every value into a fresh slice, so it is O(N) in both time and allocation.
- **A lister read never hits the API server.** `podLister.Pods(ns).Get(name)` returns *the pointer stored in the cache*. Treat it as read-only shared memory: **`DeepCopy()` before mutating.** Mutating in place corrupts state that other goroutines are concurrently reading, and the bug is timing-dependent and brutal to reproduce. (client-go ships a `cacheMutationDetector` that periodically re-hashes cached objects and panics if they changed underneath it; it is off unless `KUBE_CACHE_MUTATION_DETECTOR=true` — turn it on in your tests.)
- **Secondary indexes are maintained incrementally.** On every `Add`/`Update`/`Delete`, `updateSingleIndex` recomputes the index values for the object, deletes the key from stale buckets and inserts it into new ones. It short-circuits the common case: if the index func returns exactly one value and it is unchanged, it does nothing at all.

```go
inf.AddIndexers(cache.Indexers{
    "byNode": func(obj interface{}) ([]string, error) {
        pod, ok := obj.(*v1.Pod)
        if !ok { return nil, fmt.Errorf("not a Pod: %T", obj) }
        return []string{pod.Spec.NodeName}, nil
    },
})
// every Pod on a GPU node, straight from cache, no API call:
pods, err := inf.GetIndexer().ByIndex("byNode", "gpu-node-0007")
```

controller-runtime exposes the same mechanism as `mgr.GetFieldIndexer().IndexField(ctx, &v1.Pod{}, "spec.nodeName", fn)`, which registers an indexer named `field:spec.nodeName` and makes `client.MatchingFields{"spec.nodeName": x}` index-backed.

#### Worked math 1 — what N objects of size S cost you in RAM

The naive estimate is "N × the JSON size." It is wrong by roughly 3×, because what the cache holds is a *decoded Go object graph*: every string is a header plus a backing array, every map has bucket overhead, every slice a header, and the pointers are 8 bytes each.

Measured directly (client-go v0.36.3, Go 1.26, `cache.NewIndexer`, 20,000 realistic GPU-training Pods, heap read via `runtime.ReadMemStats` after `runtime.GC()`):

| Quantity | Value |
|---|---|
| One Pod, JSON-serialised (`S_json`) | 1,730 B |
| One Pod, protobuf-serialised | 1,074 B |
| One Pod resident in the indexer (`S_heap`) | **5,191 B** |
| 20,000 Pods in the indexer, no secondary index | **99.0 MiB** |
| 20,000 Pods with a `byNode` secondary index | 99.8 MiB |
| Marginal cost of the `byNode` index | **41 B/object** (0.77 MiB total) |

The rule of thumb that falls out: **`heap ≈ 3 × wire-JSON`, and a single-valued secondary index costs about 40 bytes per object** — one set entry plus amortised map overhead. Now you can size a controller:

```
  cache_bytes ≈ N × S_heap × (1 + 0.008 per single-valued index)

  Pods only, 20k objects:            20,000 × 5.2 KiB   ≈  99 MiB
  Pods + Nodes (400 × ~9 KiB):                          ≈ 103 MiB
  Pods + Nodes + your 3 CRD kinds (5k × ~2 KiB):        ≈ 113 MiB
  ×2 headroom for GC (Go's default GOGC=100 lets the
  heap reach ~2× live before a collection):             ≈ 226 MiB

  → a memory limit of 256 MiB is tight; 384 MiB is honest.
```

That last step is the one people forget. A Go process with 113 MiB *live* will happily sit at 226 MiB RSS under `GOGC=100`, and an OOMKill at exactly the moment the cache finishes filling looks exactly like "the operator crashloops on startup for no reason."

Two levers cut `N × S_heap` directly, and lesson 06 shows where to set them: a **cache `TransformFunc`** that strips `managedFields`, annotations and container statuses before the object is stored (often 40–60% off `S_heap`), and **`ByObject` label/field selectors** that shrink `N`.

#### Worked math 2 — index lookup vs full-cache scan

"All Pods on node X," two ways, benchmarked on the same 20,000-object indexer (Go 1.26, Intel Xeon @ 2.10 GHz, 50 matching Pods):

| Approach | Time per call | Bytes allocated | Allocations |
|---|---|---|---|
| `idx.List()` + client-side `NodeName ==` filter | **708,757 ns** (709 µs) | 328,695 B | 7 |
| `idx.ByIndex("byNode", "gpu-node-0007")` | **2,338 ns** (2.3 µs) | 896 B | 1 |

That is a **303× time reduction and a 367× allocation reduction**, and the mechanism is visible in the numbers: `List()` must build a 20,000-entry `[]interface{}` (20,000 × 16 B ≈ 320 KB — that is the 328,695 B) and then compare 20,000 strings; `ByIndex` is one map lookup for the index, one for the bucket, and a 50-entry result slice.

Scale it to a fleet. Suppose a per-node cost aggregation runs this lookup once per reconcile, and you reconcile 20,000 objects every 10 minutes:

```
  unindexed: 20,000 × 709 µs   = 14.2 s of CPU per sweep
             14.2 s / 600 s    = 2.4% of one core, continuously, per replica
             × 40 clusters      ≈ 0.95 cores burned fleet-wide doing nothing
             plus 20,000 × 329 KB = 6.6 GB allocated per sweep → GC pressure

  indexed:   20,000 × 2.3 µs   = 0.047 s per sweep  (0.008% of a core)
             plus 20,000 × 896 B = 18 MB per sweep
```

The 6.6 GB of garbage per sweep is the part that bites first: it is not the CPU time in the filter, it is the allocation rate driving the garbage collector. **This is the concrete answer to "why does the operator pod's CPU request look so high for what it does."**

### 5. Resync ≠ relist ≠ watch event

The factory's resync period is the most misunderstood knob in client-go. **Resync does not touch the API server.** At each interval the informer calls `store.Resync()`, which walks the *existing store contents* and enqueues a `Sync` delta per key; `processDeltas` then delivers each one to your handlers as `OnUpdate(obj, obj)` — old and new are the same pointer.

| Mechanism | Triggered by | Apiserver traffic | Data freshness | Handler sees |
|---|---|---|---|---|
| **Watch event** | a real write in etcd | the watch stream (already open) | current | `OnAdd`/`OnUpdate`/`OnDelete` with real change |
| **Resync** | local timer (`resyncPeriod`) | **none** | unchanged — replay of what is already cached | `OnUpdate(old, new)` with `old == new` |
| **Relist** | `410 Gone`, or reflector (re)start | one full LIST | fully refreshed | `OnUpdate`/`OnAdd` per object, plus synthesised `OnDelete` for vanished keys |

Why resync exists at all: to paper over *your own* dropped work. If a reconcile failed in a way that did not re-enqueue — a bug, a swallowed panic, a condition you could not converge — resync gives you a periodic "here is everything again, try once more" sweep. It is a safety net, not a freshness mechanism. `resyncPeriod: 0` disables it; client-go silently clamps anything below `minimumResyncPeriod = 1 * time.Second` up to 1 s.

controller-runtime's equivalent is `Options.SyncPeriod`, **default `10 * time.Hour`** with 10% jitter between controllers, and its own doc is blunt about the knob: "Change this value only if you know what you are doing," and if what you actually want is to poll something unwatchable, **return `ctrl.Result{RequeueAfter: t}` instead**. A good reconciler is level-based and re-enqueues its own retries; a tight resync just floods the workqueue with redundant work. (In current controller-runtime the flood is partly defanged: resync-generated events carry `handler.LowPriority = -100` in the priority queue, so real changes jump ahead of them. See §8.)

### 6. Handlers enqueue keys, not objects

Your `ResourceEventHandlerFuncs` should do almost nothing: derive a **key** and enqueue it.

```go
func (c *Controller) enqueue(obj interface{}) {
    // ObjectToName returns cache.ObjectName{Namespace, Name} — the modern form.
    ref, err := cache.ObjectToName(obj)
    if err != nil {
        utilruntime.HandleError(err)
        return
    }
    c.workqueue.Add(ref)
}

func (c *Controller) enqueueOnDelete(obj interface{}) {
    // Handles the DeletedFinalStateUnknown tombstone that Replace() produces.
    key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
    if err != nil {
        utilruntime.HandleError(err)
        return
    }
    c.workqueue.Add(key)
}
```

Three key funcs you should be able to tell apart:

| Func | Returns | Use for |
|---|---|---|
| `cache.MetaNamespaceKeyFunc` | `"ns/name"` string (`"name"` if cluster-scoped) | classic string-keyed queues |
| `cache.ObjectToName` | `cache.ObjectName{Namespace, Name}` (a comparable struct) | typed generic queues — the current idiom |
| `cache.DeletionHandlingMetaNamespaceKeyFunc` | unwraps `DeletedFinalStateUnknown` first | **any delete handler** |

Why keys and not objects? Three reasons, and this is the deepest idea in the module — it is what makes lesson 03's level-triggered contract *true at the plumbing layer*, not merely a design intention:

1. **Dedup.** The workqueue collapses duplicate keys because it stores them in a `sets.Set[T]`. Ten changes to one object → one queue entry. Enqueue the object and every enqueue is a distinct value, so nothing collapses. Since §2 established that the FIFO does **not** coalesce, the workqueue is the *only* place dedup happens.
2. **Freshness.** When a worker finally pops the key, it re-reads current state from the lister. Enqueuing the object freezes a snapshot at enqueue time, which the worker then reconciles even though the world moved on.
3. **Level-triggered semantics.** You reconcile toward "what should be true for this name," not "replay this specific event." Missing an intermediate event is then harmless.

There is a fourth, practical reason: the workqueue's type parameter is `[T comparable]`. A `*v1.Pod` is comparable (pointer equality) but useless as a dedup key; a `v1.Pod` struct is not comparable at all and will not compile.

### 7. The workqueue, method by method

`k8s.io/client-go/util/workqueue` is three interfaces stacked. Here they are in full, straight from the source:

```go
// The base queue.
type TypedInterface[T comparable] interface {
    Add(item T)
    Len() int
    Get() (item T, shutdown bool)
    Done(item T)
    ShutDown()
    ShutDownWithDrain()
    ShuttingDown() bool
}

// Adds deferred insertion.
type TypedDelayingInterface[T comparable] interface {
    TypedInterface[T]
    AddAfter(item T, duration time.Duration)
}

// Adds rate-limited insertion. This is what controllers use.
type TypedRateLimitingInterface[T comparable] interface {
    TypedDelayingInterface[T]
    AddRateLimited(item T)  // AddAfter(item, rateLimiter.When(item))
    Forget(item T)          // clear the item's failure count. Does NOT dequeue.
    NumRequeues(item T) int
}

// The pluggable policy behind AddRateLimited.
type TypedRateLimiter[T comparable] interface {
    When(item T) time.Duration
    Forget(item T)
    NumRequeues(item T) int
}
```

#### The three sets, and what "stingy" means

The base queue is exactly three fields plus a condition variable:

```go
type Typed[T comparable] struct {
    queue      Queue[T]      // ordered work list; slice-backed FIFO by default
    dirty      sets.Set[T]   // everything that needs processing
    processing sets.Set[T]   // everything a worker currently holds
    cond       *sync.Cond
}
```

The invariant, quoted from the source: *"Every element of `queue` should be in the `dirty` set and not in the `processing` set."* Here is the whole state machine, including the one case everyone gets wrong — `Add` arriving while a worker holds the item:

```
                         Add(k)
                           │
             ┌─────────────┴──────────────┐
             │ k already in dirty?        │
             ├── yes ─▶ k in processing?  │
             │            ├─ no  ─▶ queue.Touch(k); RETURN  (pure no-op dedup)
             │            └─ yes ─▶ RETURN                  (already re-marked)
             └── no ──▶ dirty.Insert(k)
                            │
                            ├── k in processing? ── yes ─▶ RETURN
                            │     (do NOT push: worker still holds it. The
                            │      re-queue happens later, inside Done.)
                            └── no ─▶ queue.Push(k); cond.Signal()


  STATE TABLE for one key k
  ─────────────────────────────────────────────────────────────────────────
   queue   dirty   processing   meaning
   ─────   ─────   ──────────   ────────────────────────────────────────────
     ·       ·          ·       unknown / fully settled
     ✓       ✓          ·       waiting to be picked up
     ·       ·          ✓       a worker is reconciling it, no new events
     ·       ✓          ✓       worker is reconciling it AND it changed again
                                → Done() will push it back onto queue
  ─────────────────────────────────────────────────────────────────────────


  Get()                              Done(k)
   ├─ block while queue empty         ├─ processing.Delete(k)
   ├─ k = queue.Pop()                 ├─ dirty.Has(k)?
   ├─ processing.Insert(k)            │    ├─ yes ─▶ queue.Push(k); Signal()
   └─ dirty.Delete(k)                 │    └─ no  ─▶ (nothing; if processing is
                                      │              now empty, Signal so that
                                      │              ShutDownWithDrain wakes)
                                      └─ return

  ─────────────────────────────────────────────────────────────────────────
  TIMELINE: 1,000 events for one key during a 3-second reconcile
  ─────────────────────────────────────────────────────────────────────────
  t=0.00  Add(k)              queue=[k] dirty={k} processing={}
  t=0.01  Get() -> k          queue=[]  dirty={}  processing={k}   worker starts
  t=0.02  Add(k) ×1000        queue=[]  dirty={k} processing={k}   ← 1000 events
                                                                      collapse to
                                                                      ONE dirty bit
  t=3.00  Done(k)             queue=[k] dirty={k} processing={}    ← exactly one
                                                                      re-queue
  t=3.01  Get() -> k          worker reconciles CURRENT state, once.
  ─────────────────────────────────────────────────────────────────────────
  Net effect: 1,001 events → 2 reconciles. Never 2 concurrent reconciles of k.
```

Two guarantees fall out, and you should be able to state both cold:

- **A key is never processed by two workers at once.** `Get` moves it into `processing`, and `Add` refuses to push a key that is in `processing`. This is the "stingy" property the package comment names.
- **A change that arrives during processing is never lost and never duplicated.** It sets the dirty bit; `Done` converts that bit into exactly one re-queue.

#### `Done` is not optional — here is exactly what breaks

`Done(item)` is the only thing that removes the key from `processing`. Skip it — a `return` before the `defer`, a panic in a worker with no recover, a `Reconcile` that blocks forever on an HTTP call with no context deadline — and:

1. `k` stays in `processing` **forever**.
2. Every future `Add(k)` sees `dirty.Has(k)` false → inserts into `dirty` → sees `processing.Has(k)` true → **returns without pushing**. So the key is now permanently invisible to `Get`.
3. **That one object stops reconciling, permanently, for the life of the process.** Every other object keeps working, which is what makes this so confusing to diagnose: your controller is not "down," it is deaf to exactly one key.
4. `workqueue_unfinished_work_seconds` and `workqueue_longest_running_processor_seconds` for that queue climb linearly forever — those two metrics are the fingerprint.
5. `ShutDownWithDrain()` blocks indefinitely, because it waits for `processing.Len() == 0`. Your pod then hangs until `terminationGracePeriodSeconds` expires and it is SIGKILLed. The source says it outright: *"Workers must call `Done` on an item after processing it, otherwise `ShutDownWithDrain` will block indefinitely."*

Hence the shape of every correct worker, `defer` on line two:

```go
func (c *Controller) processNextItem(ctx context.Context) bool {
    key, shutdown := c.workqueue.Get()
    if shutdown {
        return false
    }
    defer c.workqueue.Done(key)        // ALWAYS. Releases the key for re-adds.

    if err := c.reconcile(ctx, key); err != nil {
        c.workqueue.AddRateLimited(key) // failure: requeue with backoff
        return true
    }
    c.workqueue.Forget(key)             // success: reset THIS key's failure count
    return true
}
```

`Forget` and `Done` are frequently confused. The source comment on `Forget` settles it: *"This only clears the `rateLimiter`, you still have to call `Done` on the queue."*

| Call | Touches | Effect | When |
|---|---|---|---|
| `Done(k)` | the three sets | releases `k` from `processing`; re-queues it if dirty | **always**, deferred |
| `Forget(k)` | the rate limiter's failure map | `delete(failures, k)` → next `When(k)` returns `baseDelay` again | on **success** only |
| `AddRateLimited(k)` | rate limiter + delaying heap | `AddAfter(k, When(k))`; increments `failures[k]` | on **failure** |

#### The delaying layer

`AddAfter(item, d)` does not sleep. It sends a `waitFor{data, readyAt}` down a channel into `waitingLoop`, which keeps a **min-heap ordered by `readyAt`** plus a map from item to heap entry. Re-adding an item already waiting does not duplicate it — `insert()` keeps the *earlier* `readyAt`. The loop sleeps on a timer for the head of the heap, with a `maxWait = 10 * time.Second` heartbeat ticker as insurance ("just insurance against weird things happening," per the source). When `readyAt` passes, the item is moved onto the base queue via the ordinary `Add`, so it re-enters the three-set dance and gets deduped normally.

#### The rate limiters, with real defaults

```go
func DefaultTypedControllerRateLimiter[T comparable]() TypedRateLimiter[T] {
    return NewTypedMaxOfRateLimiter(
        NewTypedItemExponentialFailureRateLimiter[T](5*time.Millisecond, 1000*time.Second),
        // 10 qps, 100 bucket size. Only for retry speed, and only overall (not per item).
        &TypedBucketRateLimiter[T]{Limiter: rate.NewLimiter(rate.Limit(10), 100)},
    )
}
```

| Limiter | Scope | Defaults | `When` returns |
|---|---|---|---|
| `TypedItemExponentialFailureRateLimiter` | **per item** | `baseDelay = 5 ms`, `maxDelay = 1000 s` | `min(5ms × 2^failures[item], 1000s)`, then `failures[item]++` |
| `TypedBucketRateLimiter` | **whole queue** | `qps = 10`, `burst = 100` | `limiter.Reserve().Delay()` — 0 while tokens remain |
| `TypedMaxOfRateLimiter` | composition | — | the **maximum** of its children's delays |
| `TypedItemFastSlowRateLimiter` | per item | user-supplied | `fastDelay` for the first *N* failures, then `slowDelay` |
| `TypedWithMaxWaitRateLimiter` | wrapper | user-supplied | caps any inner limiter's delay |

#### Worked math 3 — the backoff schedule, computed out

One object whose reconcile always fails (a pricing API returning 503). Delay for failure *n* (zero-indexed) is `5 ms × 2^n`, capped at 1000 s:

| Failure # | Formula | Delay | Cumulative elapsed |
|---:|---|---:|---:|
| 1 | 5 ms × 2⁰ | 5 ms | 0.005 s |
| 2 | 5 ms × 2¹ | 10 ms | 0.015 s |
| 3 | 5 ms × 2² | 20 ms | 0.035 s |
| 4 | 5 ms × 2³ | 40 ms | 0.075 s |
| 5 | 5 ms × 2⁴ | 80 ms | 0.155 s |
| 6 | 5 ms × 2⁵ | 160 ms | 0.315 s |
| 7 | 5 ms × 2⁶ | 320 ms | 0.635 s |
| 8 | 5 ms × 2⁷ | 640 ms | 1.275 s |
| 9 | 5 ms × 2⁸ | 1.28 s | 2.56 s |
| 10 | 5 ms × 2⁹ | 2.56 s | 5.12 s |
| 11 | 5 ms × 2¹⁰ | 5.12 s | 10.2 s |
| 12 | 5 ms × 2¹¹ | 10.24 s | 20.5 s |
| 13 | 5 ms × 2¹² | 20.48 s | 41.0 s |
| 14 | 5 ms × 2¹³ | 40.96 s | 81.9 s |
| 15 | 5 ms × 2¹⁴ | 81.92 s | 2 m 44 s |
| 16 | 5 ms × 2¹⁵ | 163.84 s | 5 m 28 s |
| 17 | 5 ms × 2¹⁶ | 327.68 s | 10 m 55 s |
| 18 | 5 ms × 2¹⁷ | 655.36 s | **21 m 51 s** |
| 19+ | capped | **1000 s** (16 m 40 s) | +16 m 40 s each |

Two derived facts worth carrying in your head:

```
  time to reach the cap = Σ 5ms·2ⁿ for n = 0..17
                        = 5 ms × (2¹⁸ − 1) = 5 ms × 262,143 = 1,310.7 s ≈ 21.8 min

  retries in the first hour of continuous failure
                        = 18 (to reach cap) + ⌊(3600 − 1311)/1000⌋ = 18 + 2 = 20

  compare: no backoff, bucket-limited only → 10 qps × 3600 s = 36,000 reconciles/h
```

**20 retries per hour versus 36,000.** That is the entire value of the per-item limiter, and it is also why a broken object recovers slowly: after 22 minutes of failure, the fix lands and it can still be 16 minutes before the next attempt. If that latency is unacceptable, do not lower `maxDelay` globally — return `ctrl.Result{RequeueAfter: 30 * time.Second}` for the specific "waiting on an external dependency" case, which uses the delaying path and **does not** touch the failure counter.

Now the global bucket. With `qps = 10, burst = 100`, suppose 5,000 distinct keys all fail at once (your pricing API is down cluster-wide). Each key's `When` returns `max(5 ms, bucketDelay)`. The bucket hands out the 100 burst tokens instantly, then one token per 100 ms; the 5,000th reservation is scheduled at `(5000 − 100)/10 = 490 s` ≈ 8 minutes out. So the *whole controller* drains its failure storm over ~8 minutes instead of ~25 ms — this is what "the workqueue is a shock absorber" means quantitatively.

**Important caveat, and a correction to most write-ups:** controller-runtime does **not** use `DefaultTypedControllerRateLimiter` any more. See §8.

#### The metrics, by their real names

Both client-go and controller-runtime register the same workqueue metric family (subsystem `workqueue`, from `k8s.io/component-base/metrics/prometheus/workqueue`):

| Metric | Type | Reads as |
|---|---|---|
| `workqueue_depth` | gauge | items waiting. High **and flat** ⇒ nothing is draining. |
| `workqueue_adds_total` | counter | enqueue rate — your event pressure. |
| `workqueue_queue_duration_seconds` | histogram | how long items wait before pickup. |
| `workqueue_work_duration_seconds` | histogram | how long `Reconcile` takes. |
| `workqueue_unfinished_work_seconds` | gauge | total in-flight time of items held right now. |
| `workqueue_longest_running_processor_seconds` | gauge | the oldest still-held item. **Climbing monotonically ⇒ a wedged worker / missing `Done`.** |
| `workqueue_retries_total` | counter | `AddRateLimited` calls. Spiking ⇒ a dependency is failing. |

controller-runtime adds `controller_runtime_reconcile_total{controller,result}`, `controller_runtime_reconcile_errors_total`, `controller_runtime_terminal_reconcile_errors_total`, `controller_runtime_reconcile_panics_total`, `controller_runtime_reconcile_time_seconds`, `controller_runtime_active_workers`, `controller_runtime_max_concurrent_reconciles`, and `controller_runtime_reconcile_timeouts_total`. Wire these into the capstone: they turn "it feels stuck" into a diagnosis.

### 8. How controller-runtime wraps all of this

controller-runtime hides the plumbing, but it is the same machine — with three deliberate differences you must know.

- **The Manager owns one shared cache.** `sigs.k8s.io/controller-runtime/pkg/cache` is a set of `SharedIndexInformer`s, started lazily per type when a controller `Watches`/`Owns`/`For`s it. Ten controllers watching Pods cost one reflector, one store, ten `processorListener`s.
- **`mgr.GetClient()` is a split client:** **reads served from the cache**, **writes sent straight to the apiserver.** So `r.Get(ctx, key, &obj)` has exactly the staleness properties of a lister read. `mgr.GetAPIReader()` is the uncached escape hatch.
- **The queue is a priority queue now.** Since controller-runtime v0.23 the default is `sigs.k8s.io/controller-runtime/pkg/controller/priorityqueue` (`ptr.Deref(options.UsePriorityQueue, true)` in `pkg/controller/controller.go`). It keeps the same `TypedRateLimitingInterface` contract and the same internal dedup, but orders ready items by priority in a B-tree instead of a flat slice. `handler.LowPriority = -100` is stamped on **events from the initial LIST and on resync events** (`evt.IsInInitialList`, or `ObjectOld.ResourceVersion == ObjectNew.ResourceVersion`), so a real change always jumps ahead of replayed noise. This directly answers controller-runtime issue #2374, which asked for exactly that: "give resync events a lower priority while assigning to other events a higher priority."
- **Therefore the default rate limiter is different.** With the priority queue on (the default), the limiter is **`NewTypedItemExponentialFailureRateLimiter(5*time.Millisecond, 1000*time.Second)` only — no global token bucket.** The `MaxOf(exponential, 10qps/100burst)` combination applies only when you explicitly set `UsePriorityQueue: false`. If you have read that "controller-runtime rate-limits the whole controller to 10 qps," that is now stale; the per-item exponential curve is doing all the work, and your global ceiling is `MaxConcurrentReconciles` plus your client's own QPS/burst.

**`Reconcile`'s return value maps onto queue operations exactly** (`pkg/internal/controller/controller.go`, `reconcileHandler`):

| You return | Queue operation | Failure counter |
|---|---|---|
| `ctrl.Result{}, nil` | `Forget(req)` | reset to 0 |
| `ctrl.Result{RequeueAfter: d}, nil` | `Forget(req)` **then** `AddWithOpts{After: d}` | reset to 0 |
| `ctrl.Result{Requeue: true}, nil` (deprecated) | `AddWithOpts{RateLimited: true}` | incremented |
| `ctrl.Result{}, err` | `AddWithOpts{RateLimited: true}` | incremented |
| `ctrl.Result{}, reconcile.TerminalError(err)` | **nothing — not requeued** | n/a; bumps `controller_runtime_terminal_reconcile_errors_total` |

Two traps in that table. First, **returning a non-nil error together with a `RequeueAfter` silently drops the `RequeueAfter`** — controller-runtime logs a warning and rate-limits instead. Second, `reconcile.TerminalError` is the *supported* answer to hot-looping on a permanent error (a malformed spec that no retry can fix): it stops the requeue entirely and surfaces on its own metric, instead of grinding toward the 1000-second cap forever. Other current `controller.Options` you should know: `MaxConcurrentReconciles` (default **1**), `CacheSyncTimeout` (default **2 minutes**), `RecoverPanic` (default **true**), and `ReconciliationTimeout` (default **none** — set it, so a hung client call cannot wedge a worker).

### 9. Cache staleness and read-after-write

The cache is *eventually* consistent. The window between "the apiserver commits a write" and "your reflector's watch delivers it and the indexer updates" is real: usually single-digit milliseconds, but seconds under load or during a relist, and unbounded if the watch is wedged. Consequences:

- A cache read can return an object **older** than a fresh `GET`.
- A cache read can return `NotFound` for something you just created.
- A cache read can still return something you just deleted.
- **Read-after-write of your own writes is not guaranteed.** You `Status().Update()` a `WorkloadCost`, immediately re-`Get` it from the cache, and see the old `resourceVersion` — because your write went straight to the apiserver while your read came from a cache that has not yet seen the resulting watch event.

For level-triggered reconcile logic this is fine: the update event is coming, and you will converge. It bites in four specific places, and each has a specific handling:

| Situation | Why the cache lies | What to do instead |
|---|---|---|
| Optimistic-concurrency retry after a `409 Conflict` | you need the *newest* `resourceVersion`, and the cache has the old one | re-read through `mgr.GetAPIReader()`, or use `retry.RetryOnConflict` with a fresh live get |
| "Did my finalizer land?" / "Did my `Create` succeed?" | the object may not be in the cache yet | check the returned object from the write call itself; it carries the persisted state |
| Uniqueness / quota decisions ("has anyone else already claimed this GPU?") | a stale list under-counts | do not decide from cache; enforce with a validating webhook (lesson 08) or an apiserver-side constraint |
| Reading your own status back to compute a delta | you may read pre-write status and double-count | keep the computed value in a local variable; never re-read your own write to confirm it |

And the general rule: **an `APIReader` read costs the apiserver a real request.** Use it where correctness demands it, not as a reflex — a controller that live-reads on every reconcile has thrown away the entire reason the cache exists.

## Perspectives

**Developer perspective.** The reflector → FIFO → indexer → workqueue pipeline is the single mental model that explains *every* client-go-based tool you will ever debug — your CNI's controller, your CSI driver's, kube-controller-manager's, your own. Once it is internalised, reading an unfamiliar controller's source is a matter of finding where it wires up `AddEventHandler` and what it does inside `Reconcile`, not relearning an architecture.

**Operator perspective.** `workqueue_depth`, `workqueue_unfinished_work_seconds`, `workqueue_longest_running_processor_seconds` and `workqueue_retries_total` are metrics a CKA has scraped for years without necessarily knowing which stall each one diagnoses. This lesson converts dashboards into causal reasoning: a high **flat** depth means a wedged worker; a climbing `longest_running_processor_seconds` means a missing `Done`; a spiking `retries_total` with healthy depth means a failing dependency and a self-healing controller.

**Systems / queueing-theory perspective.** The workqueue is a literal shock absorber, and §7's arithmetic quantifies it: 1,001 events collapse to 2 reconciles; a 5,000-key failure storm drains over ~8 minutes instead of 25 ms; a permanently broken object retries 20 times an hour instead of 36,000. This is the same shape as any bounded-queue backpressure system — worth naming as a transferable pattern you will meet again in message brokers, rate-limited APIs, and load shedders.

**Economics perspective (module-specific).** A controller that never calls `Done()` is not just a reliability bug on a GPU fleet: an unindexed cache scan burning 2.4% of a core per replica across 40 clusters is about one core of pure waste, and if the reconcile loop makes a *paid* external call per iteration — a billing API — a hot loop bills you twice, in wasted compute and in metered API calls. The 6.6 GB/sweep allocation figure from §4 is the one to bring to a design review; "it's only 700 microseconds" is not.

## Real-world use cases

- **OpenAI, "Scaling Kubernetes to 7,500 nodes."** At 7,500 nodes they observed **up to 70 GB of heap per API server**, ran **5 apiservers and 5 etcd nodes** on the largest clusters, and moved Kubernetes Events into a **separate etcd cluster** to isolate that write volume. Their sharpest finding is directly this lesson's material: WATCHes on `Endpoints` for services every node belongs to (kubelet, node-exporter) fire on every node add/remove, and because each node also watches, the response count and bandwidth are **O(N²)** in node count. They also alert on the rate of HTTP **429** and **5xx** on apiservers as the top-level "informers are hurting the control plane" signal. *What it shows:* the aggregate cost of many reflectors is a control-plane capacity problem, not a client-side detail. (2023 snapshot — cite as a case study, not current scale.)
- **kubernetes/kubernetes#103789, "StreamWatcher memory leak?"** A reporter creating and deleting pods every 10 seconds saw client-go informer memory reach **2.3 GB after 30 minutes**. The mechanism is §3's unbounded `pendingNotifications` ring: a handler slower than the event rate accumulates notifications without bound. The fix landed in PR #103917, and the tracing added alongside it is why client-go now logs `"slow event handlers blocking the queue"` when FIFO depth exceeds 10 and an item takes over 100 ms. *What it shows:* "do almost nothing in the handler" is a memory-safety rule, not a style preference.
- **kubernetes-sigs/controller-runtime#2374, "Priority Queue."** The request: "when there are many objects of the same type and at every resync period there is a storm of events being added to the queue," resync work competes with real changes. The resolution shipped as `pkg/controller/priorityqueue`, **on by default since v0.23**, stamping `LowPriority = -100` on initial-list and resync events. *What it shows:* the resync safety net has a real cost at scale, and the fix was scheduling, not disabling it.
- **Pinterest Engineering, "Scaling Kubernetes with Assurance at Pinterest."** Production experience with watch-cache sizing and event retention during connection churn — the operational counterpart to §1's 5-minute history window and `410 Gone` relist path.
- **Render.com Engineering, "Kubernetes Informers are so easy… to misuse!"** A practitioner catalogue of the misuse patterns in this lesson's Pitfalls — mutating cached objects, heavy handlers, and starting work before `WaitForCacheSync`.

## Worked example

Trace a single Pod update — a GPU node's Pod gets a new `nvidia.com/gpu` annotation — from the API to your `Reconcile`, naming the client-go types at every hop, and then price it.

**Step 1 — write lands.** Something `PATCH`es the Pod. The apiserver validates, runs admission, commits to etcd, and bumps `metadata.resourceVersion` from `128374651` to `128374982`. The watch cache is updated and the event is broadcast to every registered watcher.

**Step 2 — reflector receives it.** Your process's `cache.Reflector` for Pods has an open WATCH established at RV `128374651` with a randomised timeout somewhere in `[5 min, 10 min)`. The apiserver streams:

```
{"type":"MODIFIED","object":{"kind":"Pod","apiVersion":"v1",
  "metadata":{"name":"training-job-000042-worker-0","namespace":"team-014",
              "resourceVersion":"128374982",
              "annotations":{"nvidia.com/gpu":"h100-80gb"}}, ...}}
```

`watchHandler` calls `store.Update(pod)` and sets `lastSyncResourceVersion = "128374982"`.

*(Alternative path: had the watch been broken past the apiserver's 5-minute history, the reflector would have received `410 Gone`, set `isLastSyncResourceVersionUnavailable`, and done a full relist with `resourceVersion=""`. The same Pod would arrive, but as part of a `Replace()` of all ~20,000 Pods — 34.6 MB of JSON — and every other Pod would be re-delivered to handlers too.)*

**Step 3 — into the FIFO.** On Kubernetes 1.33+, `RealFIFO.addToItems_locked` appends one `Delta{Type: Updated, Object: pod}` to a flat, strictly-ordered `[]Delta`. On ≤1.32 it would instead have appended to `items["team-014/training-job-000042-worker-0"]`, joining any deltas already pending for that key. **Either way, nothing is coalesced away.**

**Step 4 — the Pop loop drains it.** `sharedIndexInformer.handleDeltas` → `processDeltas`. Type is `Updated`; `clientState.Get(obj)` finds the old Pod, so:

1. `indexer.Update(pod)` — the `threadSafeMap` swaps the pointer under `Lock()`, and `updateSingleIndex` re-evaluates every registered index. Your `byNode` index func returns `"gpu-node-0007"` both before and after, so the single-value fast path fires and the index is untouched.
2. `handler.OnUpdate(oldPod, newPod)` is queued to each `processorListener`.

**Step 5 — handler enqueues a key.** Your `UpdateFunc` runs on the listener's `run()` goroutine. It does **not** inspect the Pod deeply. It computes `cache.ObjectToName(pod)` → `cache.ObjectName{Namespace: "team-014", Name: "training-job-000042-worker-0"}` and calls `workqueue.Add(name)`. In controller-runtime this is `handler.EnqueueRequestForObject` (or `EnqueueRequestForOwner`, which reads `metadata.ownerReferences` and enqueues the *parent's* key instead).

The three-set dance: `dirty` does not have the key → insert; `processing` does not have it → `queue.Push`, `cond.Signal()`. **If a worker were already reconciling this key,** the push would be skipped and only the dirty bit set, guaranteeing exactly one follow-up reconcile after `Done`.

**Step 6 — a worker pops.** One of `MaxConcurrentReconciles` goroutines calls `Get()`, receives the key, and immediately `defer`s `Done(key)`.

**Step 7 — reconcile reads *current* state.** `r.Get(ctx, req.NamespacedName, &pod)` — a **cache read** off the indexer. Because §3's ordering guarantees the store was written before the handler fired, this returns at least the Pod from step 2, and possibly a *newer* one if the Pod changed again while the key sat in the queue. That is the point. The reconciler then needs every other Pod on the same node to compute node-level utilisation:

```go
var pods corev1.PodList
if err := r.List(ctx, &pods,
    client.MatchingFields{"spec.nodeName": pod.Spec.NodeName}); err != nil {
    return ctrl.Result{}, err
}
```

That is index-backed **only because** `SetupWithManager` registered the index first:

```go
if err := mgr.GetFieldIndexer().IndexField(ctx, &corev1.Pod{}, "spec.nodeName",
    func(o client.Object) []string {
        return []string{o.(*corev1.Pod).Spec.NodeName}
    }); err != nil {
    return err
}
```

**Cost of that one line, measured (§4):** 2.3 µs and 896 B with the index; 709 µs and 329 KB without. Over a 20,000-object sweep every 10 minutes that is 0.047 s vs 14.2 s of CPU and 18 MB vs 6.6 GB of allocation.

**Step 8 — write and settle the queue.**

```go
wc.Status.ObservedGPUHours = hours
wc.Status.ComputedSpend = spend.String()
if err := r.Status().Update(ctx, &wc); err != nil {   // straight to the apiserver
    return ctrl.Result{}, err                          // → AddRateLimited, failures++
}
return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil // → Forget, then AddAfter(5m)
```

If that `Status().Update` returns `409 Conflict` because a competing writer bumped `resourceVersion`, do **not** re-read from the cache to retry — the cache still holds the version you started from. Use `mgr.GetAPIReader()` or `retry.RetryOnConflict`.

**Step 9 — what a storm looks like.** Now suppose a rollout touches all 64 workers of this job 30 times each in 20 seconds: 1,920 events.

```
  events into handlers ............ 1,920   (RealFIFO delivers every one)
  Add() calls into the workqueue .. 1,920
  distinct keys ...................    64
  reconciles actually executed ....  ~64–128  (one per key, plus at most one
                                               follow-up per key whose dirty bit
                                               was set mid-reconcile)
  apiserver reads during those reconciles .. 0  (all cache)
  apiserver writes ......................... ≤128 status updates
```

A 30× event amplification collapses to a 1–2× reconcile amplification, and zero read amplification, purely from key-based dedup plus cache reads.

**Raw client-go vs controller-runtime.** kubernetes/sample-controller writes steps 3–8 by hand: you build the informer factory, the workqueue, the worker loop, and call `Get`/`Done`/`Forget`/`AddRateLimited` yourself. controller-runtime does steps 3–6 and step 8's queue mechanics for you; you write step 7's `Reconcile` body and declare the watches. Same types underneath — `pkg/cache` builds the `SharedIndexInformer`s, `pkg/internal/controller` runs the worker loop against a `priorityqueue`. Read `sample-controller/controller.go` end to end once; it is about 400 lines and it is the whole machine with nothing hidden.

## Practice

Instrument your `gpu-cost-operator` (or the module-01 controller if the operator isn't scaffolded yet) and record what you observe. Commit a short findings note (`docs/informer-observations.md` in the practice dir).

1. **Cache-hit vs live read.** In `Reconcile`, do the normal cached `r.Get(...)`, then also fetch the same object via `mgr.GetAPIReader().Get(...)`. Log both `resourceVersion`s side by side. Do a rapid self-write (status update, then immediately reconcile) and catch a case where the cached RV lags the live RV. Record the divergence and how long it persisted.
2. **Short resync.** Set the manager's `SyncPeriod` to `10s`. Log a distinctive line at the top of `Reconcile`. Confirm you see periodic reconciles for *every* object with **no corresponding apiserver watch event** (check with `kubectl get --watch` in another pane, or the apiserver audit log) — proving resync is a local replay, not a re-LIST. Note whether these reconciles arrive at lower priority than real changes.
3. **Force a stale-cache scenario.** Either (a) add an artificial delay and `kubectl edit` the object during it, then observe your cached read returning the pre-edit version while `APIReader` shows the new one; or (b) delete an object and log whether the lister still returns it for a beat. Measure and record the staleness window.
4. **Register and measure a field index.** Add a `spec.nodeName` index via `mgr.GetFieldIndexer().IndexField`, then time a `client.MatchingFields{...}` list against an unindexed `List()` + client-side filter at whatever object count your test cluster has. Report ns/op and B/op for both (a Go benchmark is the honest way; `go test -bench . -benchmem`). Compare the ratio you measure to this lesson's 303×/367× at 20,000 objects and explain the difference from your object count.
5. **Measure your cache's memory.** Expose `/debug/pprof` on the manager, fill the cache, then `go tool pprof -top http://localhost:8080/debug/pprof/heap`. Record bytes-per-object for your CRD types and compare against the ~3× wire-JSON rule of thumb. Compute the memory limit you would set.
6. **Workqueue under stress.** Make `Reconcile` return an error for one specific object and log the requeue timestamps. Confirm the delays grow 5 ms, 10 ms, 20 ms, 40 ms… and that returning `nil` (→ `Forget`) resets the sequence to 5 ms. Then return `reconcile.TerminalError(err)` instead and confirm the object is **not** requeued and `controller_runtime_terminal_reconcile_errors_total` increments.

**Acceptance:** a committed `informer-observations.md` describing the observed cache-vs-live RV divergence, the resync replay behavior, a measured staleness window, the indexed-vs-unindexed benchmark with ns/op and B/op, the measured bytes-per-object and the memory limit you derived from it, and the backoff progression including the `TerminalError` case — plus the instrumentation code committed alongside.

## Common pitfalls

1. **Believing a short resync period improves data freshness.** *Symptom:* someone sets `SyncPeriod: 30s` "so the data stays fresh," and the controller's CPU and workqueue depth climb with no accuracy improvement. *Mechanism:* `Resync()` iterates `knownObjects.ListKeys()` and enqueues `Sync` deltas from the **existing store**. There is no HTTP call anywhere in that path. It replays stale data more often; it cannot make it fresher. If you need freshness for something unwatchable, return `RequeueAfter`.
2. **Believing the FIFO coalesces events.** *Symptom:* a design that assumes "N updates arrive as one." *Mechanism:* `dedupDeltas` only collapses the last two deltas when **both are deletions**; and since Kubernetes 1.33 the informer uses `RealFIFO`, a flat in-order slice with no per-key grouping at all. All dedup is in the workqueue, keyed on the request key.
3. **Mutating a lister-returned object in place instead of `DeepCopy()`-ing first.** *Symptom:* impossible-to-reproduce wrong values, usually under load, often in a *different* controller than the one doing the mutating. *Mechanism:* the lister returns the pointer stored in the shared `threadSafeMap`. Ten handlers hold the same pointer. Writing through it is an unsynchronised write to memory other goroutines are reading. Set `KUBE_CACHE_MUTATION_DETECTOR=true` in CI to catch it.
4. **Enqueuing the object instead of its key.** *Symptom:* reconciles against state that is visibly older than what `kubectl get` shows, plus no dedup under load. *Mechanism:* the queue's dedup is `sets.Set[T]` membership; two enqueues of the same object at different times are different values, so nothing collapses, and the worker reconciles a frozen snapshot instead of re-reading current state — reintroducing edge-triggered behaviour through the back door.
5. **Doing real work inside an event handler.** *Symptom:* memory climbs steadily; `workqueue_adds_total` lags `workqueue_depth`; eventually OOMKill. *Mechanism:* `processorListener.pop()` never blocks the informer — it appends to an **unbounded** ring buffer. Handler slower than the event stream ⇒ unbounded growth. This is exactly kubernetes/kubernetes#103789 (2.3 GB in 30 minutes).
6. **Forgetting `defer queue.Done(key)`.** *Symptom:* one object stops reconciling forever while everything else works; `workqueue_longest_running_processor_seconds` climbs monotonically; shutdown hangs until SIGKILL. *Mechanism:* the key is stuck in `processing`, so every future `Add` sets the dirty bit and refuses to push, and `ShutDownWithDrain` waits on `processing.Len() == 0` forever.
7. **Assuming a `Get` right after your own `Create`/`Update` reflects it.** *Symptom:* "my finalizer didn't get added" / duplicate child objects created. *Mechanism:* writes bypass the cache; reads do not. Your own write returns to you before the watch event carrying it does. Use the object returned by the write call, or `mgr.GetAPIReader()`.
8. **Ignoring `WaitForCacheSync` returning false.** *Symptom:* "the controller does nothing on startup, then works after a restart." *Mechanism:* usually RBAC — the ServiceAccount can `get` but not `list`/`watch` the type, so the reflector's initial LIST fails, `HasSynced` never trips, and controller-runtime aborts after `CacheSyncTimeout` (2 minutes) with one log line.
9. **Missing index causing a full-cache scan.** *Symptom:* the operator's CPU request looks absurd for what it does; GC time dominates a CPU profile. *Mechanism:* an unindexed filtered `List` is O(N) in time **and allocates an N-entry slice every call** — 329 KB per call at 20,000 objects, 6.6 GB per full sweep. Register the index in `SetupWithManager` before you need it; `client.MatchingFields` on an unregistered field returns an error rather than silently scanning, which is the good failure.

## Self-check

- **When can a controller-runtime client read return stale data, and how do you force a live read?**
  **Answer:** `mgr.GetClient()` is a split client: **reads are served from the manager's shared cache** (the `SharedIndexInformer`'s `threadSafeMap` indexer), **writes go straight to the apiserver.** The cache is only as current as the last watch event the reflector delivered, so `r.Get`/`r.List` can return an object older than a fresh `GET`, return `NotFound` for something just created, still return something just deleted, or fail to reflect **your own write** — because the write went direct while the read came from a cache that has not yet received the resulting event. Typical lag is milliseconds; it stretches to seconds under load or during a relist and is unbounded if the watch is wedged. To force a live read use `mgr.GetAPIReader()`, which bypasses the cache entirely. Use it only where staleness changes the answer: optimistic-concurrency retries after a `409`, "did this land" checks, and uniqueness/quota decisions. Routine reconcile logic should stay on the cache, because it is level-triggered and the event is coming.

- **What does the resync period actually re-trigger, and why isn't it a re-LIST?**
  **Answer:** Resync calls `store.Resync()`, which walks the informer's **existing store contents** (`knownObjects.ListKeys()`) and enqueues a `Sync` delta per key; `processDeltas` then delivers each as `OnUpdate(obj, obj)` — old and new are the same object. It generates **zero apiserver traffic**: no LIST, no WATCH, no HTTP at all. It is not a re-LIST because its purpose is not refreshing data — the watch already keeps the cache current — but re-driving your reconciler in case a previous attempt silently failed to converge or to re-enqueue itself. In DeltaFIFO the resync for a key is *skipped* if that key already has pending deltas, since the real event carries fresher data. A re-LIST — which does hit the apiserver — happens only on reflector start or after `410 Gone`, when the reflector has genuinely lost its watch position. controller-runtime's `SyncPeriod` defaults to **10 hours** with 10% jitter, and resync-generated events are stamped `LowPriority = -100` so they queue behind real changes.

- **How does the workqueue's dedup + rate limiter keep a controller stable under an event storm, and where does `Forget()` fit?**
  **Answer:** **Dedup** comes from three sets — `queue`, `dirty`, `processing`. `Add(k)` returns immediately if `k` is already dirty; if `k` is in `processing` it sets the dirty bit but does **not** push, and `Done(k)` converts that bit into exactly one re-queue. So 1,000 events for one object during a 3-second reconcile become one extra reconcile, and a key is never processed by two workers concurrently. **Rate limiting** bounds retry speed: `AddRateLimited` calls `rateLimiter.When(k)` and `AddAfter`s the result. The per-item exponential limiter returns `min(5 ms × 2^failures, 1000 s)`, so a permanently failing object reaches the 1000-second cap after 18 failures spanning ~21.8 minutes and then retries **20 times an hour** instead of thousands. client-go's `DefaultTypedControllerRateLimiter` also composes a global token bucket (**10 qps, burst 100**) via `MaxOf`, so a 5,000-key failure storm drains over ~8 minutes; note that **controller-runtime's default priority queue uses the exponential limiter alone**, without the bucket. **`Forget(k)`** is called on **success**: it deletes `k` from the limiter's failure map so the next failure starts at 5 ms again. It does **not** dequeue anything — the source says so explicitly ("this only clears the rateLimiter, you still have to call `Done`"). You call both: `Done` always (deferred), `Forget` only on success.

- **Why is enqueuing a *key* rather than the *object* central to level-triggered correctness, not just an optimization?**
  **Answer:** It is what makes level-triggering true at runtime. Enqueue the object and you freeze a snapshot at enqueue time; the worker then reconciles "the object as of when I saw the event," which is edge-triggered behaviour wearing a level-triggered costume. Enqueue a key and the worker *must* re-fetch from the cache at pop time, so it always reconciles against current state regardless of how many changes happened while the key waited. It also enables dedup at all: the queue's `dirty`/`processing` sets are `sets.Set[T]` on a `comparable` type — two enqueues of "the same object" at different times are different values and cannot collapse, whereas two enqueues of `ObjectName{ns, name}` are the same value. This matters more than it used to: since Kubernetes 1.33 the informer's `RealFIFO` performs no coalescing whatsoever, so the workqueue is the *only* dedup in the entire pipeline.

- **At 20,000 Pods in cache, contrast the cost of a plain `List()` with a client-side filter against a registered field-index lookup for "all Pods on node X" — in time, allocation, and memory.**
  **Answer:** Measured on client-go v0.36.3 / Go 1.26 with 20,000 realistic Pods and 50 matches: `List()` + filter takes **708,757 ns and allocates 328,695 B per call** (the list copies all 20,000 entries into a fresh `[]interface{}` — 20,000 × 16 B ≈ 320 KB — then does 20,000 string comparisons). `ByIndex("byNode", …)` takes **2,338 ns and allocates 896 B**: two map lookups plus a 50-entry result slice. That is **303× faster and 367× less garbage**. Over a 20,000-object sweep every 10 minutes the unindexed path burns 14.2 s of CPU and 6.6 GB of allocation per sweep — 2.4% of a core continuously per replica, ~1 core fleet-wide across 40 clusters — versus 0.047 s and 18 MB indexed. The index's cost is tiny and measurable: **41 bytes per object** (0.77 MiB over 20,000 Pods), one `sets.Set[string]` entry plus amortised map overhead, maintained incrementally by `updateSingleIndex`, which short-circuits entirely when a single-valued index key is unchanged. For context on the base memory: one such Pod is 1,730 B as JSON but **5,191 B resident** in the indexer — roughly 3× — so 20,000 Pods is ~99 MiB live, and you should budget ~2× that for `GOGC=100` headroom.

## Connections & what's next

This lesson is the machinery that makes lesson 03's reconciliation contract real: the reflector/FIFO/indexer/workqueue pipeline is *why* reconcile receives a key instead of an event, *why* a cache read can be stale, and *why* backoff behaves the way it does. It is also the layer every subsequent lesson quietly assumes. Lesson 06's owner-reference garbage collector is, under the hood, just another controller running this exact reflector → informer → workqueue loop — watching every type in the cluster, which is precisely why a broken conversion webhook for one CRD can stall its cache sync and stop cascade deletion. Lesson 08's admission webhooks sit in a different path entirely — synchronous, in the write path — precisely because informers are inherently asynchronous and cannot gate an in-flight write, which is also why §9's uniqueness/quota decisions must not be made from cache. Next, **lesson 05, CRD design**, turns to the shape of the objects flowing through this pipeline: how a well-designed schema (structural validation, CEL, subresources, versioning) makes the reconcile loop you just learned to trust actually safe to build on.

## References & further reading

**Primary sources — read the code, it is short**

- client-go `tools/cache` godoc — https://pkg.go.dev/k8s.io/client-go/tools/cache — the authoritative reference for `Reflector`, `DeltaFIFO`, `RealFIFO`, `SharedIndexInformer`, `Indexer` and the key funcs. The version audited for this lesson is **v0.36.3**; `reflector.go` (backoff constants, `relistResourceVersion`, watch timeouts), `delta_fifo.go` (`dedupDeltas`, `Replace`, `Resync`), `the_real_fifo.go`, `controller.go` (`processDeltas`, `newQueueFIFO`), `shared_informer.go` (`processorListener`, `HasSynced`) and `thread_safe_store.go` (`storeIndex`, `updateSingleIndex`) are the six files that contain everything in §1–§4.
- client-go `util/workqueue` godoc — https://pkg.go.dev/k8s.io/client-go/util/workqueue — `TypedInterface` / `TypedDelayingInterface` / `TypedRateLimitingInterface`, `DefaultTypedControllerRateLimiter` (5 ms → 1000 s exponential, `MaxOf` a 10 qps / 100 burst bucket), and the three-set `Add`/`Get`/`Done` contract. `queue.go` is ~250 lines and worth reading line by line.
- client-go `features` package — https://pkg.go.dev/k8s.io/client-go/features — the source of the feature-gate table in §2 (`InOrderInformers` beta 1.33 → GA-locked 1.36, `AtomicFIFO`, `UnlockWhileProcessingFIFO`, `WatchListClient`). *Correction noted:* earlier versions of this lesson described the `DeltaFIFO` as the informer's queue unconditionally; on Kubernetes 1.33+ the default is `RealFIFO`, and on 1.36 the gate is locked on.
- kubernetes.io — API concepts, "Resource versions" and "Streaming lists" — https://kubernetes.io/docs/reference/using-api/api-concepts/ — the `Any` / `Most recent` / `Not older than` / `Exact` semantics table, the 5-minute etcd history window, `410 Gone`, watch bookmarks, and the `sendInitialEvents=true` streaming-list protocol.
- controller-runtime `pkg/cache` godoc — https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache — the manager's cache as wrapped `SharedIndexInformer`s, `SyncPeriod` (10 h default, 10% jitter), `ByObject` scoping, `ReaderFailOnMissingInformer`.
- controller-runtime `pkg/controller` + `pkg/controller/priorityqueue` godoc — https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/controller — `UsePriorityQueue` (default true since v0.23), `MaxConcurrentReconciles` (default 1), `CacheSyncTimeout` (2 min), `RecoverPanic` (true), `ReconciliationTimeout`, and the exponential-only default rate limiter. *Correction noted:* the "10 qps / 100 burst global bucket" is client-go's default, not controller-runtime's current one.
- controller-runtime `pkg/reconcile` godoc — https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/reconcile — `Result` (note `Requeue` is deprecated in favour of `RequeueAfter`) and `TerminalError`.
- kubernetes/sample-controller — https://github.com/kubernetes/sample-controller — the canonical hand-written version of everything controller-runtime hides. Read `controller.go` in full once.

**Real-world engineering blogs and issues — with what each actually shows**

- OpenAI, "Scaling Kubernetes to 7,500 nodes" — https://openai.com/index/scaling-kubernetes-to-7500-nodes/ — up to 70 GB apiserver heap at 7,500 nodes; 5 apiservers + 5 etcd nodes; Events split to a separate etcd; the O(N²) `Endpoints` WATCH problem; alerting on apiserver 429/5xx. (2023 snapshot.)
- kubernetes/kubernetes#103789 — https://github.com/kubernetes/kubernetes/issues/103789 — informer memory reaching 2.3 GB in 30 minutes from pods churning every 10 s; the origin of client-go's `"slow event handlers blocking the queue"` trace. Fixed by PR #103917.
- kubernetes-sigs/controller-runtime#2374 — https://github.com/kubernetes-sigs/controller-runtime/issues/2374 — the resync-storm problem that produced `pkg/controller/priorityqueue` and `handler.LowPriority`.
- Pinterest Engineering, "Scaling Kubernetes with Assurance at Pinterest" — https://medium.com/pinterest-engineering/scaling-kubernetes-with-assurance-at-pinterest-a23f821168da — watch-cache sizing and event retention in a real production fleet.
- Render.com Engineering, "Kubernetes Informers are so easy... to misuse!" — https://render.com/blog/kubernetes-informers — the production misuse catalogue behind this lesson's pitfalls.

**Deeper dives**

- leftasexercise.com, "Understanding Kubernetes controllers part I — queues and the core controller loop" — https://leftasexercise.com/2019/07/08/understanding-kubernetes-controllers-part-i-queues-and-the-core-controller-loop/ — a line-by-line walkthrough of the workqueue. Written pre-generics; the semantics are unchanged, the type names are the old non-`Typed` ones.
- leftasexercise.com, "Understanding Kubernetes controllers part III — informers" — https://leftasexercise.com/2019/07/15/understanding-kubernetes-controllers-part-iii-informers/ — the companion piece on informers. Predates `RealFIFO`; read §2 above alongside it.
