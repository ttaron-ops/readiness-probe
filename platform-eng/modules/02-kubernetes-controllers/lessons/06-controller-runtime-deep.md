---
lesson: "02.6"
title: "controller-runtime in depth"
module: "02"
concept: "controller-runtime in depth"
status: not-started
est_time: "26h"
prev: "05-crd-design.md"
next: "07-kubebuilder-and-rbac.md"
artifacts: []
sources: 16
---

# 02.6 · controller-runtime in depth

> **Concept.** The Manager, cache, client, owner references, and finalizers are the machinery that turns a `Reconcile` function into a correct, restart-safe controller — wire them wrong and you get stale reads, stuck deletes, and orphaned children.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Lesson 05 gave your three CRDs a schema, a `/status` subresource, and a versioning plan. This lesson is where that schema becomes a running binary: the Manager that shares one cache and one client across every controller, the Builder that decides what wakes a reconcile, the owner references that let the cluster's own garbage collector do cascade deletion for free, and the finalizer that gives your `WorkloadCost` one guaranteed last chance to record spend before it disappears. Everything here is downstream of a fact lesson 04 already established — the cache is eventually consistent, reads and writes are different paths — applied now to the specific write contract lesson 05's `/status` subresource created.

## Why this matters

You already know the reconciliation model in the abstract. This lesson is where it becomes a running binary that survives leader failover, doesn't hammer the API server, cleans up after itself, and never leaves a namespace with a half-deleted object that blocks teardown across your 40 clusters. controller-runtime hands you all of this, but only if you use the right primitive at the right moment: read from the cache except when you can't, write status through the status subresource, wire owner references so the garbage collector does cascade deletion for free, and run a finalizer so a `WorkloadCost` records its final spend before it disappears — money data you can't recover after the object is gone.

The failure modes here are the ones that generate incidents: a controller that calls `Update()` and wipes status, a finalizer that never returns and wedges a namespace in `Terminating` forever, a child object that outlives its parent because the owner reference was missing, a replica that loses its lease mid-reconcile and writes anyway. Every one of these is a small API detail with a large blast radius. This is also exactly the depth CoreWeave's and NVIDIA's controller/operator postings expect — "builds the control plane" and "extend Kubernetes components" both cash out, day to day, as get-this-lesson-right-or-page-someone. Senior work is knowing which detail governs which failure.

## What's new here (calibration)

**CKA already knows:** that controllers watch and reconcile, that `ownerReferences` exist (you've seen `kubectl get` show a pod owned by a ReplicaSet owned by a Deployment), that `kubectl delete` sometimes cascades, that finalizers can wedge a namespace in `Terminating` (you've force-removed one under duress), and leader election on control-plane components.

**Internal to learn now:** how the **Manager** assembles a shared informer cache, typed clients, leader election, and the metrics/health servers — and in exactly what order it starts them, because that order is load-bearing. The difference between the cached **Client** and the live **APIReader**, and when a cache read will lie to you. The **Builder** (`For`/`Owns`/`Watches`) and **predicates** that decide what actually triggers a reconcile, and what `Owns()` really wires up underneath. Why `Status().Update()` and `Update()` are different write paths and what each destroys if misused. How `SetControllerReference` writes an owner reference and how the API server's garbage collector — itself just another controller — uses it. The full **finalizer** dance, including the apiserver rules that make certain orderings impossible. And `MaxConcurrentReconciles`, `ReconciliationTimeout`, and honoring `ctx` — the concurrency and cancellation you learned in the Go module, applied here.

## Core concepts

**Versions this lesson's API claims apply to.** Every default, field name, and code path below was read out of **controller-runtime v0.24.1** — the release aligned to **Kubernetes 1.36** — plus `k8s.io/apimachinery` v0.36.3 for the metadata constants and the kubernetes.io garbage-collection, owners-and-dependents, and finalizers references. The kube-controller-manager flag defaults come from the Kubernetes 1.36 command-line reference. Where controller-runtime changed behaviour recently (the priority queue in v0.23, `ReconciliationTimeout`), the lesson says so.

### 1. The Manager: one binary, one cache, one client

`manager.Manager` is the thing that owns everything shared. One per binary. It builds:

- a **`cluster.Cluster`** — the shared cache (a set of `SharedIndexInformer`s from lesson 04), the split client, the scheme, the RESTMapper, and the event recorder;
- **leader election**, if enabled;
- the **metrics server**, the **health/readiness probe server**, and optionally **pprof**;
- the **webhook server** (lessons 05's conversion webhook and 08's admission webhooks);
- a registry of **runnables** — your controllers, plus anything else you `mgr.Add()`.

Register ten controllers and they all share one cache and one client, which is why a well-built operator with ten controllers holds roughly *one watch per type*, not ten.

```go
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    Scheme: scheme,

    // ---- leader election ----
    LeaderElection:   true,
    LeaderElectionID: "gpu-cost-operator.cost.example.com",  // the Lease object's name
    // LeaderElectionNamespace defaults to the pod's namespace (in-cluster).
    // LeaderElectionResourceLock defaults to "leases" since v0.12.
    LeaderElectionReleaseOnCancel: true,   // see §2 — only safe if the binary exits

    // ---- servers ----
    Metrics:                metricsserver.Options{BindAddress: ":8080"},
    HealthProbeBindAddress: ":8081",
    PprofBindAddress:       ":8082",       // omit in production unless you need it

    // ---- cache scoping: this is your memory bill (lesson 04 §4) ----
    Cache: cache.Options{
        DefaultNamespaces: map[string]cache.Config{"gpu-cost-system": {}},
        ByObject: map[client.Object]cache.ByObject{
            &corev1.Pod{}: {
                Label:     labels.SelectorFromSet(labels.Set{"gpu": "true"}),
                Transform: stripPodNoise,   // drop managedFields, annotations, etc.
            },
        },
    },

    // ---- shutdown ----
    GracefulShutdownTimeout: ptr.To(30 * time.Second),   // this IS the default
})
if err != nil { return err }

if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil { return err }
if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil { return err }
```

The defaults you should be able to recite, all from `pkg/manager/internal.go` and `pkg/controller/controller.go`:

| Option | Default | Why it matters |
|---|---|---|
| `LeaseDuration` | **15 s** | how long a lease is valid without renewal |
| `RenewDeadline` | **10 s** | leader gives up if it can't renew within this |
| `RetryPeriod` | **2 s** | how often clients retry acquire/renew |
| `LeaderElectionResourceLock` | `"leases"` | was `configmapsleases` before v0.12 |
| `GracefulShutdownTimeout` | **30 s** | how long `Start` waits for runnables to return |
| `ReadinessEndpointName` | `/readyz` | |
| `LivenessEndpointName` | `/healthz` | |
| `Cache.SyncPeriod` | **10 h** (±10% jitter) | the resync backstop; not a freshness knob |
| `Controller.MaxConcurrentReconciles` | **1** | parallel reconciles *across different objects* |
| `Controller.CacheSyncTimeout` | **2 min** | manager fails to start if caches don't sync |
| `Controller.RecoverPanic` | **true** | a panicking reconcile no longer kills the process |
| `Controller.UsePriorityQueue` | **true** (since v0.23) | resync/initial-list events get `LowPriority = -100` |
| `Controller.ReconciliationTimeout` | **none** | *set this* — see §8 |

**The start order is not arbitrary, and two of the steps carry warnings in the source.** `controllerManager.Start` does exactly this:

```
  mgr.Start(ctx)
  │
  ├─ 1. HTTPServers        metrics :8080, health :8081, pprof :8082
  │     ⚠  MUST start before any cache, because health probes must answer
  │        while the cache fills — otherwise a conversion webhook that is
  │        gated on readiness never becomes ready, the cache never syncs,
  │        and you deadlock.
  │
  ├─ 2. Webhooks           conversion + admission servers
  │     ⚠  MUST start before any cache. If a watched CRD has a conversion
  │        webhook served by THIS binary, the initial LIST needs the webhook
  │        to already be answering. Start the cache first and you have a
  │        circular wait: cache waits on webhook, webhook waits on cache.
  │
  ├─ 3. Caches             start every informer, then BLOCK on
  │                        WaitForCacheSync (CacheSyncTimeout = 2 min)
  │
  ├─ 4. Others             non-leader-elected runnables (mgr.Add of something
  │                        that returns NeedLeaderElection() == false)
  │
  ├─ 5. Warmup             runnables with EnableWarmup: true — sources start
  │                        and fill their queues while still a FOLLOWER, so a
  │                        promoted replica reconciles immediately instead of
  │                        cold-starting
  │
  └─ 6. LeaderElection     acquire the Lease, THEN start leader-elected
                           runnables (i.e. your controllers) and close
                           mgr.Elected()
```

Two practical readings. First, **metrics and health are served whether or not you are the leader** — deliberately, so Prometheus scraping a follower gets a response rather than a connection refused. Second, **your controllers do not run at all until the lease is held**, which is why a follower replica shows zero `controller_runtime_reconcile_total` and that is correct, not broken.

### 2. Leader election, and what happens when you lose the lease mid-reconcile

Leader election is a `coordination.k8s.io/v1` **Lease** object named by `LeaderElectionID` in `LeaderElectionNamespace`. Every candidate races to write its identity into `spec.holderIdentity` with a `renewTime`; whoever succeeds is leader and must keep renewing.

The three numbers interact as follows:

```
  ── leader replica (holder) ─────────────────────────────────────────────────
  t=0     acquire: Lease.spec.holderIdentity = "pod-a", renewTime = t0
  every 2s (RetryPeriod)  → renew: bump renewTime
  ...
  t=T     apiserver becomes unreachable
  t=T+2   renew fails, retry
  t=T+4   renew fails, retry
  t=T+6   renew fails, retry
  t=T+8   renew fails, retry
  t=T+10  RenewDeadline (10 s) exceeded  ─▶ OnStoppedLeading fires

  ── follower replica (candidate) ────────────────────────────────────────────
  every 2s → read Lease. Is now() − renewTime > LeaseDuration (15 s)?
  t=T+15..T+17  → yes ⇒ acquire, become leader

  ── the overlap window ──────────────────────────────────────────────────────
  the old leader stops at T+10; the new leader starts at T+15..17.
  Guard band ≈ 5–7 s, which is the whole safety argument: LeaseDuration (15)
  MUST exceed RenewDeadline (10), and RenewDeadline must exceed a few
  RetryPeriods (2), or two replicas can believe they are leader at once.
```

**What controller-runtime does when the lease is lost is unusually blunt, and you must know it.** In `initLeaderElector`, `OnStoppedLeading` does three things:

```go
OnStoppedLeading: func() {
    if cm.onStoppedLeading != nil { cm.onStoppedLeading() }
    // Skip graceful shutdown: we lost the lock without intending to.
    cm.gracefulShutdownTimeout = time.Duration(0)
    cm.errChan <- errors.New("leader election lost")
},
```

That error propagates out of `mgr.Start(ctx)`, and the kubebuilder-generated `main.go` does `os.Exit(1)` on it. So: **losing the lease kills the process, immediately, with the graceful-shutdown grace period forced to zero.** That is the correct behaviour — a replica that has lost the lease must not keep writing, and the fastest way to guarantee that is to stop existing — but it has direct consequences for how you write `Reconcile`:

- **An in-flight reconcile is cancelled, not completed.** The manager's internal context is cancelled and the process ends. If your reconcile was between "I charged the customer's account" and "I recorded that in status," that gap is now permanent.
- **Therefore reconciles must be idempotent and re-derivable.** Never treat a reconcile as a transaction that must complete. The whole level-triggered model exists so the next leader can pick up from observed state.
- **Therefore honour `ctx` on every call.** Pass it to every client call and every external I/O. A reconcile that ignores `ctx` and blocks on a 60-second HTTP timeout is a reconcile that is still writing 20 seconds after another replica became leader.
- **Therefore avoid unguarded write-after-read gaps.** Use optimistic concurrency (`Update` with the `resourceVersion` you read) so a stale writer from a dead leader loses with a `409 Conflict` rather than clobbering.

`LeaderElectionReleaseOnCancel: true` makes voluntary shutdown fast: on a clean `SIGTERM` the leader actively clears the Lease so a successor takes over in ~2 s instead of waiting out the 15-second `LeaseDuration`. The source's warning is precise: this "requires the binary to immediately end when the Manager is stopped, otherwise this setting is unsafe" — because a released lease plus a still-running process is exactly the split-brain the guard band was protecting you from.

**`EnableWarmup` is the newer answer to slow failover.** By default a follower starts no sources at all, so on promotion it must LIST every watched type before the first reconcile — at 20,000 Pods that is tens of seconds (lesson 04 §1). With `EnableWarmup: true` the controller's sources start while the replica is still a follower and fill its queue; on promotion it drains an already-warm queue. You trade follower memory and a duplicate watch for a much shorter recovery.

For symmetry: kube-controller-manager uses the *same* three defaults — `--leader-elect-lease-duration 15s`, `--leader-elect-renew-deadline 10s`, `--leader-elect-retry-period 2s`. Your operator is not doing anything exotic; it is running the control plane's own recipe.

### 3. Client (cached) versus APIReader (live), and how to scope the cache

`mgr.GetClient()` is a **split client**:

- **Reads** (`Get`, `List`) are served from the manager's shared cache. The cache lazily starts an informer the first time a type is read, unless you set `ReaderFailOnMissingInformer: true`, in which case it returns `ErrResourceNotCached` instead — worth enabling, because it turns "we accidentally cached every Secret in the cluster" into a loud error.
- **Writes** (`Create`, `Update`, `Patch`, `Delete`, `DeleteAllOf`, and everything under `Status()` and `SubResource()`) go **straight to the apiserver**. There is no write-through cache; the cache only learns about your write when the watch event comes back.

`mgr.GetAPIReader()` is the uncached reader. Use it where staleness changes the answer, and nowhere else (lesson 04 §9 lists the four cases: conflict retries, "did this land" checks, uniqueness/quota decisions, and reading a resource you deliberately excluded from the cache).

**Scoping the cache is the single highest-leverage tuning knob in the whole operator**, because lesson 04's arithmetic — one realistic Pod costs ~5.2 KiB resident, ~3× its JSON size — applies directly:

```go
Cache: cache.Options{
    // 1. Shrink N by namespace.
    DefaultNamespaces: map[string]cache.Config{"gpu-cost-system": {}},

    ByObject: map[client.Object]cache.ByObject{
        &corev1.Pod{}: {
            // 2. Shrink N by label — a server-side selector on the LIST/WATCH.
            Label: labels.SelectorFromSet(labels.Set{"workload-type": "gpu"}),
            // 3. Shrink N by field — needs a selectableField on a CRD (lesson 05 §4),
            //    or a built-in indexed field like spec.nodeName for Pods.
            Field: fields.OneTermEqualSelector("spec.nodeName", myNode),
            // 4. Shrink S by transform — runs before the object enters the cache.
            Transform: func(o any) (any, error) {
                p, ok := o.(*corev1.Pod)
                if !ok { return o, nil }
                p.ManagedFields = nil                        // often 1–2 KiB alone
                delete(p.Annotations,
                    "kubectl.kubernetes.io/last-applied-configuration")
                p.Status.ContainerStatuses = nil             // if you never read it
                return p, nil
            },
        },
    },
},
Client: client.Options{
    Cache: &client.CacheOptions{
        // 5. Never cache Secrets: a cluster-wide Secret informer is both a
        //    memory problem and a blast-radius problem.
        DisableFor: []client.Object{&corev1.Secret{}},
    },
},
```

Sizing it, with lesson 04's measured constants:

```
  unscoped:  20,000 Pods × 5.2 KiB               ≈  99 MiB live
             ×2 for GOGC=100 headroom            ≈ 198 MiB RSS
  label-scoped to GPU pods (say 2,000):
             2,000 × 5.2 KiB                     ≈  10 MiB live
  + transform stripping managedFields/annotations (measure yours; 40–60% is
    typical for objects that have been kubectl-applied):
             2,000 × ~2.4 KiB                    ≈   5 MiB live
  → a 10× to 20× reduction from two options and a five-line function.
```

The catch with `Transform`: whatever you strip is *gone from every cache read in the process*. If another controller in the same binary needs `ContainerStatuses`, you have just broken it, silently and only in production. Put the transform behind a comment naming which fields the whole binary is allowed to depend on.

### 4. `Status().Update()` versus `Update()` — different subresources, different casualties

Lesson 05 §3 established the apiserver-side mechanism; here is what it means at the call site. With `/status` declared:

- `client.Update(ctx, obj)` → `PUT /apis/.../budgets/{name}`. The registry strategy **restores `status` from the stored object** before persisting, so your computed status is discarded with no error. It persists `spec` and `metadata`, and bumps `metadata.generation` if non-metadata content changed.
- `client.Status().Update(ctx, obj)` → `PUT /apis/.../budgets/{name}/status`. The strategy **deep-copies the stored object over yours** and writes back only `status` (plus `managedFields`), so `spec` and `metadata` in your payload are ignored. Generation is untouched.

```go
// CORRECT — computed observed state through the status subresource.
wc.Status.ObservedGPUHours = hours
wc.Status.ComputedSpend = spend.String()
wc.Status.ObservedGeneration = wc.Generation
meta.SetStatusCondition(&wc.Status.Conditions, metav1.Condition{
    Type: "Ready", Status: metav1.ConditionTrue,
    Reason: "SpendComputed", Message: "spend reconciled",
    ObservedGeneration: wc.Generation,
})
if err := r.Status().Update(ctx, &wc); err != nil {
    return ctrl.Result{}, err
}
```

Three refinements that separate a working controller from a well-behaved one:

**Prefer patch to update under concurrency.** `Update` is a full-object PUT carrying the `resourceVersion` you read; any competing write between your read and your write gives you a `409 Conflict`. Under `MaxConcurrentReconciles: 4` and a chatty cluster, that is common. Two options:

```go
// (a) Merge-from patch: sends only the diff, computed against the object you read.
patch := client.MergeFrom(wc.DeepCopy())
wc.Status.ComputedSpend = spend.String()
if err := r.Status().Patch(ctx, &wc, patch); err != nil { ... }

// (b) Optimistic-lock patch: same, but FAILS on conflict instead of merging.
//     Use when a lost update would be wrong (e.g. accumulating a total).
patch := client.MergeFromWithOptions(wc.DeepCopy(), client.MergeFromWithOptimisticLock{})
```

Option (a) is not universally safer — a merge patch that says "set computedSpend to X" will happily overwrite a concurrent write. Choose (b) whenever the new value depends on the old one.

**Retry conflicts against a *live* read, not a cached one.** This is the read-after-write failure mode in its most common clothing:

```go
err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
    var fresh v1alpha1.WorkloadCost
    // r.Get would return the SAME stale resourceVersion that just conflicted.
    if err := r.APIReader.Get(ctx, req.NamespacedName, &fresh); err != nil {
        return err
    }
    fresh.Status.ComputedSpend = spend.String()
    return r.Status().Update(ctx, &fresh)
})
```

Re-reading from the cache in a conflict-retry loop is a genuine infinite loop: the cache holds the version you started from, so the retry conflicts identically, `retry.DefaultRetry` exhausts its five steps, and you return an error that requeues with backoff. The symptom is a controller that "works but takes 20 minutes to update status," and the cause is one letter of difference at the call site.

**Server-side apply for status, if you have co-writers.** `r.Status().Patch(ctx, obj, client.Apply, client.FieldOwner("gpu-cost-operator"))` claims ownership of exactly the status paths you send. Combined with lesson 05's `listType=map` on `conditions`, that lets two controllers own different conditions on the same object without clobbering.

### 5. The Builder: what actually wakes your reconciler

```go
func (r *BudgetReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // Index registration must happen BEFORE the controller starts, or
    // client.MatchingFields returns an error at reconcile time.
    if err := mgr.GetFieldIndexer().IndexField(context.Background(),
        &v1alpha1.WorkloadCost{}, "spec.policyRef",
        func(o client.Object) []string {
            return []string{o.(*v1alpha1.WorkloadCost).Spec.PolicyRef}
        }); err != nil {
        return err
    }

    return ctrl.NewControllerManagedBy(mgr).
        Named("budget").
        For(&v1alpha1.Budget{},
            builder.WithPredicates(predicate.GenerationChangedPredicate{})).
        Owns(&v1alpha1.WorkloadCost{}).
        Watches(&v1alpha1.GPUCostPolicy{},
            handler.EnqueueRequestsFromMapFunc(r.budgetsForPolicy),
            builder.WithPredicates(predicate.ResourceVersionChangedPredicate{})).
        WithOptions(controller.Options{
            MaxConcurrentReconciles: 4,
            ReconciliationTimeout:   2 * time.Minute,
        }).
        Complete(r)
}
```

Here is the full path an event takes, and where each knob sits on it:

```
  apiserver watch event on some type T
        │
        ▼
  informer for T in the manager's shared cache   (lesson 04)
        │  OnAdd / OnUpdate / OnDelete / OnGeneric
        ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │ PREDICATES — run per source, BEFORE anything is enqueued.            │
  │   builder.WithPredicates(...)  → this watch only                     │
  │   WithEventFilter(...)         → EVERY watch on this controller,     │
  │                                  including Owns and Watches. Rarely  │
  │                                  what you want.                      │
  │                                                                      │
  │   GenerationChangedPredicate   Update passes iff generation changed. │
  │                                Skips your own status writes → the    │
  │                                standard fix for a status hot-loop.   │
  │                                ⚠ ALSO skips label/annotation edits,  │
  │                                  since metadata never bumps          │
  │                                  generation (lesson 05 §3).          │
  │   ResourceVersionChangedPredicate  Update passes iff RV changed.     │
  │                                Lets resync events through.           │
  │   LabelChangedPredicate / AnnotationChangedPredicate                 │
  │   And(...) / Or(...) / Not(...)  compose them                        │
  └───────────────────────────────┬──────────────────────────────────────┘
                                  │ survivors
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                         ▼
  For(&Budget{})            Owns(&WorkloadCost{})     Watches(&GPUCostPolicy{},
        │                         │                          EnqueueRequestsFromMapFunc)
  EnqueueRequestForObject   EnqueueRequestForOwner           │
        │                    + OnlyControllerOwner()    your MapFunc runs:
        │                      (unless you pass         it may LIST the cache
        │                       builder.MatchEveryOwner) to fan 1 event → N reqs
        │                         │                          │
  Request{ns,name}          reads child's                Request per result
   of the object            metadata.ownerReferences,
   itself                   emits Request{childNS,
                            ownerName}
        └─────────────────────────┼─────────────────────────┘
                                  ▼
              priority queue (default since v0.23)
              • dedup on Request  • LowPriority = -100 for initial-list
                                    and resync events
                                  ▼
              MaxConcurrentReconciles workers → Reconcile(ctx, req)
```

Points that are easy to get wrong:

- **`Owns()` matches only the *controller* owner reference by default.** The builder appends `handler.OnlyControllerOwner()` unless you pass `builder.MatchEveryOwner`. So a child with a plain (non-controller) owner reference to your parent will **not** wake the parent. `SetControllerReference` sets `controller: true`; `SetOwnerReference` does not.
- **`Owns()` requires `For()`.** The builder errors with `Owns() can only be used together with For()` because it needs to know which type the owner reference should resolve to.
- **A `MapFunc` runs on the informer's dispatch path**, so it must be fast and must read only from the cache. This is where a missing index turns into the O(N) scan of lesson 04 §4: fanning "a `GPUCostPolicy` changed" out to "every `WorkloadCost` referencing it" without an index on `spec.policyRef` is a full-cache scan **on every policy event**.

```go
func (r *BudgetReconciler) budgetsForPolicy(ctx context.Context, o client.Object) []reconcile.Request {
    var list v1alpha1.WorkloadCostList
    // Index-backed because of the IndexField call in SetupWithManager.
    if err := r.List(ctx, &list,
        client.MatchingFields{"spec.policyRef": o.GetName()}); err != nil {
        log.FromContext(ctx).Error(err, "mapping policy to workloads")
        return nil
    }
    reqs := make([]reconcile.Request, 0, len(list.Items))
    for i := range list.Items {
        reqs = append(reqs, reconcile.Request{
            NamespacedName: client.ObjectKeyFromObject(&list.Items[i]),
        })
    }
    return reqs
}
```

- **`WithEventFilter` applies globally to the controller**, including `Owns` and `Watches` sources. Putting `GenerationChangedPredicate` there means child status changes stop waking the parent — usually the exact opposite of what you wanted from `Owns()`. Prefer `builder.WithPredicates` on the specific source.
- **`GenerationChangedPredicate` has a real blind spot.** Generation only tracks non-metadata changes, so a label or annotation edit produces an Update with an unchanged generation and is filtered out. If your reconcile reads labels — and a cost operator reading a `cost-center` label does — combine it: `predicate.Or(predicate.GenerationChangedPredicate{}, predicate.LabelChangedPredicate{})`.

### 6. Owner references and the garbage collector

`controllerutil.SetControllerReference(owner, child, scheme)` writes exactly this onto the child:

```go
ref := metav1.OwnerReference{
    APIVersion:         gvk.GroupVersion().String(),
    Kind:               gvk.Kind,
    Name:               owner.GetName(),
    UID:                owner.GetUID(),      // ← the identity that actually matters
    BlockOwnerDeletion: ptr.To(true),
    Controller:         ptr.To(true),
}
```

and it refuses in two cases, from `validateOwner`:

- a **cluster-scoped child with a namespaced owner** → `"cluster-scoped resource must not have a namespace-scoped owner"`;
- a **cross-namespace** owner → `"cross-namespace owner references are disallowed"`.

It also returns `AlreadyOwnedError` if the child already has a *different* controller reference, because an object may have many owners but at most one `controller: true`.

**The `UID` is the point.** An owner reference is not "the object with this name"; it is "the object with this UID." Delete a parent and recreate it with the same name, and every old child is now owned by a UID that no longer exists — so the garbage collector deletes them. That is correct and it surprises people who think of owner references as name-based foreign keys.

**The garbage collector is just another controller running lesson 04's machine.** It is not apiserver magic. Inside kube-controller-manager, the `garbagecollector` runs a `GraphBuilder` that:

1. asks **discovery** for every resource in the cluster that supports `list` and `watch`;
2. starts a **metadata-only informer** per resource (`PartialObjectMetadata`, so it caches just `metadata` — including `ownerReferences` — not whole objects);
3. maintains an in-memory **DAG of owner → dependents** across all of them;
4. when a node's owner disappears or is marked for deletion, enqueues the dependents for deletion, with `--concurrent-gc-syncs` workers (**default 20**).

Three consequences follow directly from "it is one graph over all types":

- **A single type it cannot list wedges the whole graph builder.** This is kubernetes/kubernetes#110720 in one sentence, and §"Real-world use cases" gives the log lines.
- **Discovery changes are picked up on a resync, not instantly.** A brand-new CRD's objects are not GC-eligible until the graph builder has re-synced monitors.
- **The GC needs RBAC to list/watch every type**, which is why it runs with broad permissions and why an aggregated API or CRD that 500s on LIST is a cluster-wide problem.

**Deletion propagation** decides what happens to dependents. The API constants are `metav1.DeletePropagationBackground`, `Foreground`, `Orphan`; the finalizer strings the apiserver adds are `metav1.FinalizerDeleteDependents = "foregroundDeletion"` and `metav1.FinalizerOrphanDependents = "orphan"`.

| Policy | Parent | Children | The finalizer added | Use when |
|---|---|---|---|---|
| **Background** (default for most deletes) | removed **immediately** | deleted asynchronously by the GC afterwards | none | you want the DELETE call to return fast |
| **Foreground** | enters `Terminating` with `deletionTimestamp` set; **not removed until dependents with `blockOwnerDeletion: true` are gone** | deleted **first** | `foregroundDeletion` on the parent | the parent must outlive its children during teardown |
| **Orphan** | removed after owner refs are stripped | **survive**, owner references removed | `orphan` on the parent | handing children to another owner |

Two sharp edges on foreground deletion, both from the GC docs:

- **Only dependents that are *in the GC's cache* and have `blockOwnerDeletion: true` block the parent.** Objects of a type the GC cannot list, or created concurrently with the parent's deletion, do not block — so "foreground" is a best-effort ordering, not a transactional one.
- **`blockOwnerDeletion` is permission-controlled.** The `OwnerReferencesPermissionEnforcement` admission plugin (enabled on many managed clusters) requires that whoever sets `blockOwnerDeletion: true` also has `delete` permission on the *owner*. Without it your `Create` of the child fails with a `Forbidden` that names the owner, not the child — a genuinely confusing error, and one lesson 07's RBAC work has to account for.

Finally, the cross-namespace rule from the apiserver side, which is stricter than most people expect: a namespaced dependent's namespaced owner **must** be in the same namespace, and if it is not, the owner reference is *treated as absent* — meaning the dependent is deleted once all its owners are verified absent. A cluster-scoped dependent naming a namespaced owner is treated as having an unresolvable reference and is **never** garbage-collected. Both cases emit a warning Event you can find with:

```console
$ kubectl get events -A --field-selector=reason=OwnerRefInvalidNamespace
NAMESPACE   LAST SEEN   TYPE      REASON                     OBJECT
team-ml     2m14s       Warning   OwnerRefInvalidNamespace   workloadcost/train-42
```

### 7. Finalizers: the apiserver rules first, then the dance

A finalizer is a string in `metadata.finalizers`. The apiserver's behaviour, from the Kubernetes finalizers reference:

- On `DELETE` of an object with a non-empty `finalizers` list, the apiserver **sets `metadata.deletionTimestamp`**, does **not** remove the object, and returns **HTTP `202 Accepted`**.
- The object stays visible and shows as `Terminating` until the list is empty; when it empties, the object is deleted automatically.
- **Once `deletionTimestamp` is set you may remove finalizers but you may not add one**, and you may not modify `deletionTimestamp`. This is enforced, and it is the rule most people do not know.
- The object cannot be resurrected. Clearing the timestamp is not possible; you must delete and recreate.
- **Custom finalizer names must be qualified** — `cost.example.com/record-final-spend`, not `record-final-spend`. The apiserver rejects unqualified custom finalizer names.

That "cannot add after deletionTimestamp" rule is why the finalizer must be added on the **first** reconcile of a live object, before anyone could plausibly delete it. If you defer adding it until "the object is actually doing something," you have a race in which a fast delete wins and your cleanup never runs.

The dance, written the way it should be written:

```go
const finalizer = "cost.example.com/record-final-spend"  // MUST be qualified

func (r *WorkloadCostReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var wc v1alpha1.WorkloadCost
    if err := r.Get(ctx, req.NamespacedName, &wc); err != nil {
        // NotFound is the normal terminal state: object gone, nothing to do.
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    if !wc.DeletionTimestamp.IsZero() {
        // ---- deletion path ----
        if !controllerutil.ContainsFinalizer(&wc, finalizer) {
            return ctrl.Result{}, nil        // someone else's finalizer; not ours
        }
        final, err := r.meterGPUHours(ctx, &wc)
        if err != nil {
            // Object stays Terminating and the workqueue backs off. See the
            // math below for how long that actually is.
            return ctrl.Result{}, err
        }
        wc.Status.ComputedSpend = r.price(ctx, &wc, final).String()
        meta.SetStatusCondition(&wc.Status.Conditions, metav1.Condition{
            Type: "Finalized", Status: metav1.ConditionTrue,
            Reason: "FinalSpendRecorded", Message: "final spend recorded before deletion",
        })
        // ORDER MATTERS: persist the money first...
        if err := r.Status().Update(ctx, &wc); err != nil {
            return ctrl.Result{}, err
        }
        // ...then release the object. Reverse these and the object can be
        // deleted before the status write lands, losing the number forever.
        controllerutil.RemoveFinalizer(&wc, finalizer)
        if err := r.Update(ctx, &wc); err != nil {   // finalizers live in metadata
            return ctrl.Result{}, err
        }
        return ctrl.Result{}, nil
    }

    // ---- normal path: ensure the finalizer FIRST, on the first ever reconcile ----
    if controllerutil.AddFinalizer(&wc, finalizer) {
        if err := r.Update(ctx, &wc); err != nil {
            return ctrl.Result{}, err
        }
        // Requeue rather than continue: our in-memory copy's resourceVersion is
        // now stale, and any later Update in this same pass would 409.
        return ctrl.Result{Requeue: true}, nil
    }

    // ... normal reconcile ...
    return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil
}
```

Four properties to hold onto:

1. **Add the finalizer with plain `Update()`** — it is in `metadata`, not `status`. This does not bump `generation` (lesson 05 §3), so it does not perturb `observedGeneration`.
2. **Cleanup must be idempotent.** The terminating object may be reconciled many times before the finalizer clears; `recordFinalSpend` running twice must not double-charge.
3. **Cleanup must be bounded.** Honour `ctx`, set timeouts on external calls, and never block indefinitely.
4. **Write status before removing the finalizer.** The finalizer is the *only* thing keeping the object alive.

**controller-runtime ships a finalizer registry most people never find.** `pkg/finalizer` gives you a `Finalizers` map from key → `Finalizer`, and one `Finalize` call handles both directions:

```go
fins := finalizer.NewFinalizers()
_ = fins.Register("cost.example.com/record-final-spend", spendRecorder{client: r.Client})

// inside Reconcile, after the Get:
res, err := fins.Finalize(ctx, &wc)
if res.Updated {
    if uerr := r.Update(ctx, &wc); uerr != nil { return ctrl.Result{}, uerr }
}
if res.StatusUpdated {
    if uerr := r.Status().Update(ctx, &wc); uerr != nil { return ctrl.Result{}, uerr }
}
if err != nil { return ctrl.Result{}, err }
```

`Finalize` adds every registered finalizer when `deletionTimestamp` is zero, and runs each registered `Finalize` and removes it when it is set — reporting via `Result{Updated, StatusUpdated}` which write you owe. Note that a *failing* finalizer still reports `Updated`/`StatusUpdated` from its own result, precisely so it can set a condition explaining the failure. Worth using when you have more than one finalizer on a type; the hand-rolled version above stops composing at two.

**The stuck-object failure, with the actual mechanism.** If cleanup always errors — controller down, crash-looping, a dependency permanently gone — the finalizer is never removed and the object sits in `Terminating` forever. Two escalations:

- If the object was the *parent* of a foreground delete, the parent's own `foregroundDeletion` finalizer never clears either.
- If the object lives in a namespace being deleted, the namespace controller cannot finish. It reports which finalizers are blocking, and the diagnosis is one command:

```console
$ kubectl get namespace team-ml -o jsonpath='{.status.conditions}' | jq -r '.[].type'
NamespaceDeletionDiscoveryFailure
NamespaceContentRemaining
NamespaceFinalizersRemaining
```

Those five condition types — `NamespaceDeletionDiscoveryFailure`, `NamespaceDeletionContentFailure`, `NamespaceDeletionGroupVersionParsingFailure`, `NamespaceContentRemaining`, `NamespaceFinalizersRemaining` — are the supported way to diagnose a namespace stuck in `Terminating`, and they name the resource and finalizer at fault. Reach for those *before* the break-glass, which is:

```console
$ kubectl patch workloadcost/train-42 -n team-ml --type=merge \
    -p '{"metadata":{"finalizers":[]}}'
```

The Kubernetes docs are unusually direct that this is a last resort: "avoid manually removing finalizers… forcefully removing them can lead to issues in your cluster. This should only be done when the purpose of the finalizer is understood and is accomplished in another way." For a cost operator, "accomplished in another way" means you recorded the spend by hand first.

### 8. Concurrency, context, and timeouts

`MaxConcurrentReconciles: N` runs N reconciles in parallel **across different objects**; the workqueue's `processing` set (lesson 04 §7) guarantees one object is never reconciled concurrently with itself. Raise it for throughput, and keep reconciles independent and idempotent so parallelism is safe. The ceiling that actually binds is usually your client's rate limit, not N — `rest.Config` defaults to 5 QPS / 10 burst unless you raise it, so eight workers all making API calls will queue behind the client-side limiter, not the apiserver.

`ReconciliationTimeout` (added in recent controller-runtime; **no timeout by default**) wraps each `Reconcile` call's context with a deadline. **Set it.** Without it, one reconcile blocked on a hung HTTP call holds its workqueue item forever, `Done` is never called (lesson 04 §7), and that object stops reconciling for the life of the process while `workqueue_longest_running_processor_seconds` climbs monotonically. When the guardrail fires, the context is cancelled with `errReconciliationTimeout` and `controller_runtime_reconcile_timeouts_total` increments — a metric that should be zero and should alert if it isn't.

`RecoverPanic` defaults to **true**: a panicking reconcile is recovered, counted in `controller_runtime_reconcile_panics_total`, and requeued rather than killing the process. Good default, but treat a non-zero panic counter as a page — the object that panicked will keep panicking on every retry.

And the discipline that ties them together: **pass `ctx` to every client call and every external I/O.** On leader loss (§2) or shutdown, the manager cancels the internal context; a reconcile that ignores it keeps writing after the process has conceptually stopped being the leader.

### 9. The five failure modes, with mechanism

**(a) Stale cache read after your own write.** *Symptom:* "my finalizer didn't get added," duplicate child objects, a status update that conflicts forever. *Mechanism:* writes go direct to the apiserver, reads come from the cache; your own write returns to you before the watch event carrying it does (lesson 04 §9). *Handling:* use the object returned by the write call rather than re-reading; requeue after a metadata write instead of continuing in the same pass (see §7's `Requeue: true`); and in a conflict-retry loop read through `mgr.GetAPIReader()`, never the cache.

**(b) Hot-looping on a permanent error.** *Symptom:* `controller_runtime_reconcile_errors_total` climbing linearly for one object, forever; logs full of the same message. *Mechanism:* returning an error requeues with per-item exponential backoff, but backoff **caps at 1000 s** — it never gives up. A spec the controller can never satisfy (a `policyRef` pointing at a kind that does not exist) retries about 20 times an hour, forever, across every replica restart. *Handling:* classify the error. Permanent, user-caused failures return `reconcile.TerminalError(err)`, which is **not requeued** and increments `controller_runtime_terminal_reconcile_errors_total`; then surface the reason on a status condition so the user can fix it and the resulting spec change wakes you again:

```go
policy := &v1alpha1.GPUCostPolicy{}
if err := r.Get(ctx, key, policy); err != nil {
    if apierrors.IsNotFound(err) {
        meta.SetStatusCondition(&wc.Status.Conditions, metav1.Condition{
            Type: "PolicyResolved", Status: metav1.ConditionFalse,
            Reason: "PolicyNotFound",
            Message: fmt.Sprintf("spec.policyRef %q does not exist", key.Name),
            ObservedGeneration: wc.Generation,
        })
        _ = r.Status().Update(ctx, &wc)
        // Do not retry: only a spec change or the policy appearing can fix this,
        // and both produce a watch event that re-enqueues us.
        return ctrl.Result{}, reconcile.TerminalError(err)
    }
    return ctrl.Result{}, err   // transient: DO retry with backoff
}
```

The other hot-loop is self-inflicted: reconcile writes status → status write produces an Update event → reconcile runs again. With the `/status` subresource the generation does not change, so `predicate.GenerationChangedPredicate{}` on the `For()` source breaks the cycle; the belt-and-braces version is to skip the write entirely when the computed status equals the stored status.

**(c) Unbounded requeue.** *Symptom:* `workqueue_depth` and `workqueue_adds_total` climbing without bound; CPU pinned; the apiserver seeing thousands of requests per second from one operator. *Mechanism:* a reconcile that unconditionally returns `ctrl.Result{Requeue: true}` on a path that is reached every time, or a `MapFunc` that fans one event out to every object of a type which then each fan out again. Note that `Requeue: true` and returning an error both go through `AddRateLimited`, so the per-item limiter throttles a *single* key — but N distinct keys each re-adding themselves is not throttled per item at all. *Handling:* every requeue must be conditional on an observable "not yet converged" state; prefer `RequeueAfter: d` (which calls `Forget` first, so it is not a failure and does not accumulate backoff) for polling; and cap fan-out by indexing the reverse lookup instead of listing everything.

**(d) Missing index causing a full-cache scan.** *Symptom:* the operator's CPU request looks absurd; a CPU profile is dominated by GC; latency scales with cluster size rather than with the number of objects you actually touched. *Mechanism:* an unindexed filtered `List` copies the entire cached set into a fresh slice and filters client-side. Measured at 20,000 Pods (lesson 04 §4): **709 µs and 329 KB per call**, versus **2.3 µs and 896 B** for an index-backed `ByIndex`. In a `MapFunc` this runs on every event of the watched type. *Handling:* register the index in `SetupWithManager` **before** `Complete(r)`; `client.MatchingFields` on an unregistered field returns an error rather than silently scanning, which is the good failure — do not "fix" it by falling back to a manual filter.

**(e) Leader-election lease loss mid-reconcile.** *Symptom:* the operator pod restarts with `leader election lost` and a non-zero exit, in the middle of otherwise normal operation; occasionally a partially-applied change. *Mechanism:* §2 — `OnStoppedLeading` forces `gracefulShutdownTimeout = 0` and pushes an error that ends `Start`, so the process exits without draining. The usual trigger is not a crash but apiserver latency: four failed renews inside the 10-second `RenewDeadline` is enough. *Handling:* make every reconcile idempotent and re-derivable from observed state; honour `ctx` so in-flight work stops promptly; use optimistic concurrency so a straggling write from the old leader loses with a `409`; consider `EnableWarmup: true` so the successor is not cold; and alert on the restart rather than ignoring it, because repeated lease loss is an apiserver-health signal, not an operator bug.

## Perspectives

**Developer perspective.** `Status().Update()` versus `Update()` looks like a one-line API choice with outsized consequences — treat it as the canonical "know your write path" lesson for anything built on a subresource-based API, not just this operator. The same discipline applies to `/scale`, `/finalizers`, and any custom subresource you add later. The generalisation worth carrying: whenever an API splits one object across two endpoints, the split exists to give two writers non-overlapping authority, and using the wrong endpoint doesn't error — it silently drops your half.

**Operator/fleet perspective.** A stuck finalizer is an operational incident, not an isolated code bug. A namespace wedged in `Terminating` blocks every subsequent operation in that namespace, and it does not stay contained to one cluster if the same controller bug is running fleet-wide. Learn the diagnosis path — namespace `status.conditions` → `NamespaceFinalizersRemaining` → the named finalizer → the controller that owns it — before you learn the break-glass, because the break-glass skips the cleanup you wrote the finalizer to guarantee.

**Distributed-systems perspective.** Leader election, finalizers, and owner-ref GC are three independent mechanisms solving the same underlying problem — safe coordination without shared memory — on the same substrate: objects in etcd, watched through the apiserver. The lease is a mutex with a timeout; the finalizer is a reference count you decrement; the owner reference is a parent pointer that a separate collector traces. Recognising them as three instances of one pattern is what lets you reason about a fourth mechanism you have not seen yet, and it is why the guard-band arithmetic in §2 (`LeaseDuration > RenewDeadline > k × RetryPeriod`) will look familiar the next time you configure any lease-based system.

**Money/FinOps perspective (module-specific).** The finalizer recording final spend before a `WorkloadCost` disappears is the *only* chance to capture that number; there is no "recompute it later" once the object and its underlying usage data are gone. That makes the ordering in §7 — status write, then finalizer removal — a financial control, not a stylistic preference. Design review should ask "what happens to unrecorded spend if this finalizer's cleanup can never succeed?" as a first-class question, and the honest answers are the two in the Worked example below: page a human and hold the object, or record `unknown` and release it. Pick per workload value; do not discover the choice during an incident.

## Real-world use cases

- **kubernetes/kubernetes#110720 — "CRD Conversion webhook down results in controller-manager GC failure."** A production report in which an unreachable conversion webhook for the `milvus.io/v1alpha1` `Milvus` CRD stopped garbage collection. The log signature is exact and worth memorising: `Failed to watch *v1.PartialObjectMetadata: failed to list *v1.PartialObjectMetadata: conversion webhook for milvus.io/v1alpha1, Kind=Milvus failed`, then `unable to sync caches for garbage collector`, then `timed out waiting for dependency graph builder sync`. Note the type in that message — `PartialObjectMetadata` — which is §6's metadata-only informer, and the direct evidence that the GC builds *one* graph over all types. Cascading deletion recovered only when the webhook was restored. *What it shows:* a conversion webhook is a cluster-wide availability dependency; adding a second CRD version is an availability decision, not just a schema decision.
- **Gardener, "Fix Problematic Conversion Webhooks."** SAP's large-scale multi-cluster management project ships an operational runbook for exactly that failure, because a broken network path from a shoot's control plane to a conversion webhook eventually stops kube-controller-manager's garbage collection. *What it shows:* the failure is common enough at scale that a major platform documents it as routine.
- **`kubernetes.io/pv-protection`, the finalizer everyone has already met.** A `PersistentVolume` in use by a Pod carries this finalizer; deleting it puts it in `Terminating` and it stays there until the Pod stops using it. *What it shows:* the exact mechanism your `record-final-spend` finalizer uses, running in every cluster you have ever operated — including the "it looks stuck but it is working as designed" reading you need before reaching for `kubectl patch`.
- **Cloudflare, "Leveraging Kubernetes virtual machines with KubeVirt."** KubeVirt's `VirtualMachine` → `VirtualMachineInstance` owner-referenced cascade running in Cloudflare production. *What it shows:* the owner-ref/GC pattern of §6 at real scale, with the same UID-identity semantics.

## Worked example

Wire the `WorkloadCost` controller end to end: reconcile observed GPU usage into status, own children with owner references, record final spend via a finalizer, and reason about what happens when the finalizer's dependency is gone.

### The reconciler

```go
const finalizer = "cost.example.com/record-final-spend"

func (r *WorkloadCostReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)     // already carries controller name + reconcileID

    var wc v1alpha1.WorkloadCost
    if err := r.Get(ctx, req.NamespacedName, &wc); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // ---- deletion path: the finalizer records final spend ----
    if !wc.DeletionTimestamp.IsZero() {
        if !controllerutil.ContainsFinalizer(&wc, finalizer) {
            return ctrl.Result{}, nil
        }
        final, err := r.meterGPUHours(ctx, &wc)
        if err != nil {
            log.Error(err, "metering final usage; object stays Terminating")
            return ctrl.Result{}, err
        }
        wc.Status.ComputedSpend = r.price(ctx, &wc, final).String()
        meta.SetStatusCondition(&wc.Status.Conditions, metav1.Condition{
            Type: "Finalized", Status: metav1.ConditionTrue,
            Reason: "FinalSpendRecorded", Message: "final spend recorded before deletion",
        })
        if err := r.Status().Update(ctx, &wc); err != nil {   // money first
            return ctrl.Result{}, err
        }
        // Emit an Event too: once the object is gone, its status is gone with it,
        // so the Event (and a metric) are the only durable record.
        r.Recorder.Eventf(&wc, corev1.EventTypeNormal, "FinalSpendRecorded",
            "recorded %s before deletion", wc.Status.ComputedSpend)
        controllerutil.RemoveFinalizer(&wc, finalizer)
        return ctrl.Result{}, r.Update(ctx, &wc)              // release second
    }

    // ---- normal path ----
    if controllerutil.AddFinalizer(&wc, finalizer) {
        if err := r.Update(ctx, &wc); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{Requeue: true}, nil   // re-read after the metadata write
    }

    policy := &v1alpha1.GPUCostPolicy{}
    if err := r.Get(ctx, client.ObjectKey{Name: wc.Spec.PolicyRef}, policy); err != nil {
        if apierrors.IsNotFound(err) {
            meta.SetStatusCondition(&wc.Status.Conditions, metav1.Condition{
                Type: "PolicyResolved", Status: metav1.ConditionFalse,
                Reason: "PolicyNotFound", ObservedGeneration: wc.Generation,
                Message: fmt.Sprintf("spec.policyRef %q does not exist", wc.Spec.PolicyRef),
            })
            _ = r.Status().Update(ctx, &wc)
            return ctrl.Result{}, reconcile.TerminalError(err)  // permanent: do NOT retry
        }
        return ctrl.Result{}, err                               // transient: DO retry
    }

    hours, err := r.meterGPUHours(ctx, &wc)
    if err != nil {
        return ctrl.Result{}, err
    }

    // Skip the write when nothing changed — belt-and-braces against a hot loop.
    next := wc.DeepCopy()
    next.Status.ObservedGPUHours = hours
    next.Status.ComputedSpend = r.price(ctx, &wc, hours).String()
    next.Status.ObservedGeneration = wc.Generation
    meta.SetStatusCondition(&next.Status.Conditions, metav1.Condition{
        Type: "Ready", Status: metav1.ConditionTrue,
        Reason: "SpendComputed", ObservedGeneration: wc.Generation,
    })
    if !equality.Semantic.DeepEqual(wc.Status, next.Status) {
        if err := r.Status().Patch(ctx, next, client.MergeFrom(&wc)); err != nil {
            return ctrl.Result{}, err
        }
    }
    return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil
}

func (r *WorkloadCostReconciler) SetupWithManager(mgr ctrl.Manager) error {
    r.Recorder = mgr.GetEventRecorderFor("workloadcost-controller")
    r.APIReader = mgr.GetAPIReader()
    return ctrl.NewControllerManagedBy(mgr).
        Named("workloadcost").
        For(&v1alpha1.WorkloadCost{}, builder.WithPredicates(
            predicate.Or(
                predicate.GenerationChangedPredicate{},
                predicate.LabelChangedPredicate{},   // cost-center labels matter to us
            ))).
        Owns(&corev1.ConfigMap{}).                   // rendered cost report per workload
        WithOptions(controller.Options{
            MaxConcurrentReconciles: 4,
            ReconciliationTimeout:   2 * time.Minute,
        }).
        Complete(r)
}
```

Read the ordering once more: the deletion path writes status **first** and removes the finalizer **second**. Reverse them and the object can be garbage-collected before the status write lands — and since the object is what carries the number, the number is gone. `Status().Update()` for computed state; plain `Update()` for the finalizer, which lives in metadata.

### Owning a child, correctly

```go
cm := &corev1.ConfigMap{
    ObjectMeta: metav1.ObjectMeta{
        Name:      wc.Name + "-report",
        Namespace: wc.Namespace,   // MUST match the owner's namespace
    },
}
op, err := controllerutil.CreateOrUpdate(ctx, r.Client, cm, func() error {
    cm.Data = map[string]string{"spend": wc.Status.ComputedSpend}
    // Inside the mutate fn, so it is applied on both create and update.
    return controllerutil.SetControllerReference(&wc, cm, r.Scheme)
})
if err != nil {
    return ctrl.Result{}, err
}
log.V(1).Info("reconciled report", "operation", op)   // created | updated | unchanged
```

`Owns(&corev1.ConfigMap{})` in the builder makes a change to that ConfigMap re-enqueue the `WorkloadCost`, because `EnqueueRequestForOwner` reads `metadata.ownerReferences` and emits a request for the controller owner. `SetControllerReference` is what makes cascade deletion work. **You need both, and they are independent** — `Owns()` alone gives you a watch with no cascade; the owner reference alone gives you a cascade with no watch.

### Verifying the cascade, with real output

```console
$ kubectl get workloadcost,configmap -n team-ml
NAME                                          SPEND     GPU-HOURS   AGE
workloadcost.cost.example.com/train-42        $184.20   6.14        41m

NAME                        DATA   AGE
configmap/train-42-report   1      41m

$ kubectl get configmap train-42-report -n team-ml -o jsonpath='{.metadata.ownerReferences}' | jq
[
  {
    "apiVersion": "cost.example.com/v1alpha1",
    "kind": "WorkloadCost",
    "name": "train-42",
    "uid": "2f9d1c04-6b3a-4f21-9f0e-7c5a1b2d3e4f",
    "controller": true,
    "blockOwnerDeletion": true
  }
]

$ kubectl delete workloadcost train-42 -n team-ml
workloadcost.cost.example.com "train-42" deleted

$ kubectl get configmap -n team-ml
No resources found in team-ml namespace.
```

You wrote no deletion code. The `uid` in that owner reference is the identity the GC traced.

*(Transcript is representative of what this controller produces; the UID and money values are illustrative.)*

### Worked math 1 — how long does a failing finalizer actually hold the object?

Suppose `meterGPUHours` calls a pricing API that is down. Each failure returns an error, which requeues through the per-item exponential limiter (`5 ms` base, `1000 s` cap, from lesson 04 §7):

| Attempt | Delay before it | Elapsed |
|---:|---:|---:|
| 1 | — | 0 |
| 5 | 40 ms | 0.08 s |
| 10 | 2.56 s | 5.1 s |
| 14 | 40.96 s | 82 s |
| 18 | 655.36 s | **21 m 51 s** |
| 19+ | 1000 s (capped) | +16 m 40 s each |

So: the object is `Terminating` for **22 minutes** before retries slow to once every ~17 minutes, and it retries about **20 times in the first hour** and ~3–4 times per hour thereafter — forever. Two consequences worth planning for:

```
  time-to-teardown if the dependency comes back after T minutes:
      T ≤ 22 min   → detected within ≈ (last interval), i.e. seconds to minutes
      T = 2 hours  → up to 16 m 40 s of extra delay after recovery
      T = never    → the object never deletes; if it was a foreground-delete
                     child, the parent never deletes either; if the namespace
                     is terminating, the namespace never finishes
```

That is why "return the error and let backoff handle it" is a *policy decision*, not a default. Two defensible alternatives, and you should be able to defend the choice:

1. **Bounded retries plus an alert.** Track attempts (a status condition on the terminating object is visible to `kubectl get -o yaml` right up until deletion, but not after — so also emit an Event and a metric). After N attempts or a wall-clock budget, set `Finalization=False, Reason=PricingUnavailable`, emit a Warning Event, increment a counter you alert on, and **keep** the finalizer. The object stays `Terminating` until a human intervenes. Correct for high-value workloads.
2. **Time-boxed fallback.** After a bounded window, record `computedSpend: "unknown"` with `Reason: PricingUnavailable`, emit the Event, and remove the finalizer anyway. You trade a guaranteed-accurate invoice for a bounded teardown SLA. Correct for low-value workloads, and for anything that could block a namespace.

Implementing (2) needs the elapsed time, and `deletionTimestamp` gives it to you for free:

```go
const finalizeBudget = 30 * time.Minute

if err := r.recordFinalSpend(ctx, &wc); err != nil {
    if time.Since(wc.DeletionTimestamp.Time) < finalizeBudget {
        return ctrl.Result{}, err                  // keep retrying with backoff
    }
    // Budget blown: record what we know and release the object.
    wc.Status.ComputedSpend = "unknown"
    meta.SetStatusCondition(&wc.Status.Conditions, metav1.Condition{
        Type: "Finalized", Status: metav1.ConditionFalse,
        Reason: "PricingUnavailable",
        Message: fmt.Sprintf("gave up after %s: %v", finalizeBudget, err),
    })
    _ = r.Status().Update(ctx, &wc)
    r.Recorder.Eventf(&wc, corev1.EventTypeWarning, "FinalSpendUnknown",
        "released after %s without pricing: %v", finalizeBudget, err)
    controllerutil.RemoveFinalizer(&wc, finalizer)
    return ctrl.Result{}, r.Update(ctx, &wc)
}
```

Note the backoff cap interacts with the budget: with a 30-minute budget and a cap reached at ~22 minutes, you get roughly **18–19 attempts** before giving up. If you want more attempts inside the same wall-clock budget, use `RequeueAfter` on this path instead of returning an error — `RequeueAfter` calls `Forget` first, so it does not accumulate backoff:

```go
return ctrl.Result{RequeueAfter: 60 * time.Second}, nil   // ~30 attempts in 30 min
```

### Worked math 2 — the cost of getting it wrong

Put numbers on the two failure modes that cost money rather than uptime, at your stated fleet scale of 40 clusters:

```
  Hot loop from a self-triggered status write (no GenerationChangedPredicate):
      each reconcile writes status → one Update event → one reconcile
      bounded only by MaxConcurrentReconciles and client QPS.
      With the kubebuilder default rest.Config (5 QPS, burst 10):
          ≈ 5 status writes/s/replica  ×  40 clusters  = 200 writes/s of pure churn
          each write is an etcd transaction and a watch fan-out to every
          informer on that type in the cluster.
      If Reconcile also makes ONE paid pricing-API call:
          5 calls/s × 86,400 s × 40 clusters ≈ 17.3 million calls/day
      That is the "a stuck controller costs money twice" scenario: wasted CPU
      and a metered third-party bill.

  Unindexed MapFunc fan-out (lesson 04 §4 constants, 20,000 cached objects):
      709 µs + 329 KB allocated per GPUCostPolicy event
      a policy reconciled every 5 min across 200 policies
          = 200 events / 300 s ≈ 0.67 events/s
          ≈ 0.05% of a core — invisible.
      the same code when someone bumps a shared policy during a rollout,
      firing 200 events in 2 s:
          200 × 329 KB = 66 MB allocated in 2 s, in the informer's dispatch
          goroutine, which is the goroutine that must not block (lesson 04 §3).
      With the index: 200 × 896 B = 179 KB. Same code path, 370× less garbage.
```

The second block is the argument for registering the index even when the steady-state numbers look harmless: the failure is not in steady state, it is in the burst.

## Practice

Extend `gpu-cost-operator`:

1. **Manager wiring.** Enable leader election with an explicit `LeaderElectionID`, `LeaderElectionReleaseOnCancel: true`, health and readiness probes, and the metrics server. Scope the cache: `DefaultNamespaces`, a `Label` selector for the Pods you actually care about, and a `Transform` that nils `ManagedFields`. Set `Client.Cache.DisableFor` for `&corev1.Secret{}`.
2. **Reconcile.** Implement `WorkloadCostReconciler.Reconcile` to meter observed GPU usage (a deterministic stub is fine) and write `observedGPUHours` + `computedSpend` into status via `Status().Patch(..., client.MergeFrom(...))`, setting `observedGeneration` and a `Ready` condition. Skip the write when the computed status is unchanged.
3. **Own a child.** Create and own a per-workload `ConfigMap` report using `controllerutil.CreateOrUpdate` with `SetControllerReference` **inside** the mutate function, and add `Owns(&corev1.ConfigMap{})` in `SetupWithManager`.
4. **Finalizer.** Add `cost.example.com/record-final-spend` on the normal path; on deletion, compute and persist final spend to status **and emit an Event**, then remove the finalizer. Implement the time-boxed fallback from the worked example, driven off `deletionTimestamp`.
5. **Terminal errors.** Make a missing `spec.policyRef` return `reconcile.TerminalError` with a `PolicyResolved=False` condition, and confirm the object is not requeued.
6. **Concurrency and timeouts.** Set `MaxConcurrentReconciles: 4` and `ReconciliationTimeout: 2 * time.Minute`, and thread `ctx` through every client call and every stub external call.

**Experiments to run and record** (`docs/controller-runtime-observations.md` in the practice dir):

- **Prove the `Update()` trap.** Temporarily write status through `r.Update()` instead of `r.Status().Update()`. Confirm the call returns **no error** and the status is **not persisted**. Then revert. Record the exact behaviour.
- **Prove generation semantics.** Watch `metadata.generation` across: a spec edit, a status write, a label edit, and adding a finalizer. Confirm only the spec edit increments it.
- **Prove the cascade.** Capture `kubectl get -o jsonpath='{.metadata.ownerReferences}'` on the child, delete the parent, and confirm the child disappears with no code of yours involved. Then delete the parent with `--cascade=orphan` and confirm the child survives with its owner references stripped.
- **Prove foreground ordering.** Delete with `--cascade=foreground` and capture the parent's `metadata.finalizers` showing `foregroundDeletion` while children still exist.
- **Prove the stuck object.** Make the finalizer's cleanup fail permanently. Capture the object sitting in `Terminating`, capture the requeue timestamps showing exponential backoff, and capture the namespace `status.conditions` if you delete the namespace while it is stuck. Then recover it *without* the break-glass by fixing the cleanup.
- **Prove lease loss.** Run two replicas, confirm only the leader reconciles (`controller_runtime_reconcile_total` is zero on the follower), then `kubectl delete lease <LeaderElectionID>` or kill the leader and measure the failover time against the 15 s / 10 s / 2 s arithmetic in §2.
- **Measure the cache.** Record the operator's RSS with and without the `Transform` and label selector, and derive the memory limit you would set.

**Acceptance:**

- Deleting a parent cascades: its child `ConfigMap`(s) are garbage-collected, verified before/after and by inspecting `metadata.ownerReferences`.
- Deleting a `WorkloadCost` directly leaves it briefly in `Terminating`, the finalizer runs and writes a final `computedSpend` + `Finalized` condition **and emits an Event**, then the object disappears. The Event is the proof, since the status is gone with the object.
- Status is written exclusively through `Status().Update()`/`Status().Patch()`; `git grep` shows no `r.Update` carrying computed status fields.
- The controller does not hot-loop: reconcile count per object is bounded, not continuous, with predicates in place.
- A missing `policyRef` produces a `PolicyResolved=False` condition and **no requeue**; `controller_runtime_terminal_reconcile_errors_total` increments.
- Failover between two replicas completes inside the window §2 predicts, and the observations note records the measured time.

This is checkpoint items 2 and 3 directly — reconcile producing correct, owner-referenced status, and a lifecycle where deletion cascades via GC and the finalizer records final spend first — plus the operational evidence for the checkpoint's debug prompt.

## Common pitfalls

1. **Calling `Update()` with computed status.** *Symptom:* status never appears; no error anywhere. *Mechanism:* with `/status` declared, the main-resource strategy restores `status` from the stored object before persisting (lesson 05 §3). Your payload's status is discarded. Worse, `Update()` also writes your possibly-stale `spec`, so you can clobber a user's edit.
2. **Adding the finalizer and doing real work in the same pass.** *Symptom:* intermittent `409 Conflict` on the next write in that reconcile. *Mechanism:* the `Update` that added the finalizer bumped `resourceVersion`; your in-memory copy is now stale. Return `ctrl.Result{Requeue: true}` immediately after the metadata write and do the work on the next pass, against a fresh read.
3. **Deferring the finalizer until "the object matters."** *Symptom:* cleanup silently never runs for short-lived objects. *Mechanism:* once `deletionTimestamp` is set, the apiserver **refuses to let you add a finalizer** — you may only remove them. Add it on the first reconcile of a live object or not at all.
4. **Unqualified finalizer names.** *Symptom:* the `Update` adding the finalizer fails with a validation error naming `metadata.finalizers`. *Mechanism:* the apiserver requires custom finalizers to be qualified (`domain/name`). Use `cost.example.com/record-final-spend`.
5. **Assuming `Owns()` alone is enough.** *Symptom:* child changes wake the parent correctly, but deleting the parent leaves orphans. *Mechanism:* `Owns()` wires the *watch* via `EnqueueRequestForOwner`; the *owner reference* is what the garbage collector traces. You need `SetControllerReference` too — and it must be a **controller** reference, because `Owns()` defaults to `OnlyControllerOwner()`.
6. **`WithEventFilter(predicate.GenerationChangedPredicate{})`.** *Symptom:* the parent stops reacting to child status changes. *Mechanism:* `WithEventFilter` applies to *every* source on the controller, including `Owns` and `Watches`, and children signal through status, which never bumps generation. Use `builder.WithPredicates` on the specific source instead.
7. **Retrying a `409 Conflict` against the cache.** *Symptom:* status updates take minutes; `workqueue_retries_total` climbs. *Mechanism:* the cache still holds the `resourceVersion` that just lost, so every retry conflicts identically until `RetryOnConflict` gives up. Re-read through `mgr.GetAPIReader()` inside the retry closure.
8. **Treating `foreground` deletion as strictly safer.** *Symptom:* a parent that looks deleted sits in `Terminating` indefinitely. *Mechanism:* the parent's `foregroundDeletion` finalizer does not clear until every blocking dependent is gone — and if a dependent has its own wedged finalizer, that is never. Foreground changes your teardown SLA from "the DELETE returns" to "every child's cleanup succeeds first." It is also only best-effort: only dependents *in the GC's cache* with `blockOwnerDeletion: true` actually block.
9. **Setting `blockOwnerDeletion: true` without `delete` on the owner.** *Symptom:* creating the child fails with a `Forbidden` that talks about the *owner*, not the child. *Mechanism:* the `OwnerReferencesPermissionEnforcement` admission plugin requires the writer to hold `delete` on the owner. Lesson 07's RBAC has to grant it, or use `controllerutil.WithBlockOwnerDeletion(false)`.
10. **Cross-namespace owner references.** *Symptom:* a child is deleted for no visible reason, or is never garbage-collected. *Mechanism:* `SetControllerReference` refuses outright, but a hand-written reference is accepted by the apiserver and then treated as *absent* by the GC — so the child is deleted once all owners are verified absent. Check with `kubectl get events -A --field-selector=reason=OwnerRefInvalidNamespace`.
11. **No `ReconciliationTimeout`.** *Symptom:* one object stops reconciling forever while everything else works; `workqueue_longest_running_processor_seconds` climbs monotonically. *Mechanism:* a reconcile blocked on an untimed call never returns, so `Done` is never called and that key stays in `processing` permanently (lesson 04 §7). Set the timeout and alert on `controller_runtime_reconcile_timeouts_total`.
12. **Reaching for `kubectl patch -p '{"metadata":{"finalizers":[]}}'` first.** *Symptom:* the object disappears and the cleanup it guaranteed never ran — for a cost operator, the spend is unrecoverable. *Mechanism:* emptying the list satisfies the apiserver's "delete when finalizers is empty" rule immediately. Diagnose first — namespace `status.conditions`, then the named finalizer, then the controller that owns it — and only break glass when you have done the cleanup another way.

## Self-check

**(a) What breaks if you call `Update()` instead of `Status().Update()`?**

**Answer:** With `/status` declared as a subresource, `client.Update()` targets the **main** resource, and the apiserver's `customResourceStrategy.PrepareForUpdate` **copies the stored `status` over whatever you sent** before persisting. Your computed spend, GPU-hours, and conditions are silently discarded — no error, no warning, nothing in the logs. That is the primary breakage. There is a second, worse one: `Update()` is a full-object PUT, so it also writes the `spec` and `metadata` from your in-memory copy. If that copy is stale, or if you mutated spec, you clobber the user's edit and bump `metadata.generation` needlessly, which in turn makes `observedGeneration` lie. The subresource exists precisely so observed state has its own write path with its own RBAC and its own generation semantics: computed status goes through `Status().Update()` or `Status().Patch()`, and plain `Update()`/`Patch()` is reserved for spec, finalizers, labels and annotations. Under concurrency prefer `Status().Patch(ctx, obj, client.MergeFrom(old))` over `Status().Update`, and use `client.MergeFromWithOptimisticLock{}` whenever the new value is computed from the old one, so a lost update becomes a `409` instead of silent corruption.

**(b) What happens on delete if a finalizer never returns, and how do you avoid a stuck object?**

**Answer:** With a finalizer present, `DELETE` does not remove the object. The apiserver sets `metadata.deletionTimestamp`, returns **HTTP 202 Accepted**, and leaves the object visible in `Terminating`; it is deleted automatically only when `metadata.finalizers` becomes empty. From that moment you may **remove** finalizers but you may not **add** one, and you cannot clear the timestamp — the object cannot be resurrected. If your cleanup always errors, the controller returns an error each pass, the workqueue's per-item exponential limiter backs off (5 ms doubling, capped at 1000 s, reaching the cap after 18 failures spanning ~22 minutes), and the object retries roughly 20 times in the first hour and ~3–4 times per hour after that — forever. Escalations: if it was a foreground-delete dependent, the parent's own `foregroundDeletion` finalizer never clears either; if the namespace is being deleted, the namespace controller reports `NamespaceFinalizersRemaining` / `NamespaceContentRemaining` and never finishes, blocking everything in that namespace. Avoid it by making cleanup **idempotent** (it will run repeatedly) and **bounded** (honour `ctx`, timeout every external call, set `ReconciliationTimeout`); by adding the finalizer on the very first reconcile so a fast delete cannot race you; by using a **qualified** finalizer name; and by deciding in advance between bounded-retries-plus-alert and a time-boxed fallback that records `unknown` and releases the object — computing the elapsed time from `deletionTimestamp`. Alert on objects in `Terminating` past a threshold. The break-glass, `kubectl patch` to an empty `finalizers` list, skips your cleanup entirely and is a last resort, not a design.

**(c) How do owner references drive GC, and what's foreground vs background vs orphan deletion?**

**Answer:** An owner reference on a dependent names its owner by `apiVersion`, `kind`, `name` and — crucially — **`uid`**, so it binds to a specific object instance, not a name. `controllerutil.SetControllerReference` stamps it with `controller: true` and `blockOwnerDeletion: true`, and refuses cross-namespace and cluster-scoped-child/namespaced-owner combinations. The garbage collector is **not apiserver magic**: it is a controller inside kube-controller-manager running lesson 04's exact machine — it discovers every listable/watchable resource, runs a **metadata-only informer** (`PartialObjectMetadata`) per resource, maintains one in-memory owner→dependents DAG across all of them, and deletes dependents whose owners are gone, with `--concurrent-gc-syncs` workers (default 20). Because it is *one* graph over *all* types, a single type it cannot list stalls the whole graph builder — which is exactly kubernetes/kubernetes#110720. **Background** (the common default) removes the owner immediately and the GC cleans up dependents asynchronously. **Foreground** puts the owner in `Terminating` with a `foregroundDeletion` finalizer and does not remove it until dependents with `blockOwnerDeletion: true` **that are in the GC's cache** are gone — dependents of unlistable types, or created concurrently with the deletion, do not block, so it is best-effort ordering rather than a transaction. **Orphan** adds an `orphan` finalizer, strips the owner references, and lets the dependents survive. One RBAC wrinkle: the `OwnerReferencesPermissionEnforcement` admission plugin requires `delete` on the *owner* to set `blockOwnerDeletion: true`, so a missing grant shows up as a `Forbidden` naming the owner while you were creating the child.

**(d) A CRD's conversion webhook has been down for an hour. What happens to garbage collection of that type's owned children, and how would you detect it from cluster symptoms alone?**

**Answer:** The garbage collector's graph builder starts a metadata-only informer for **every** listable resource, including that CRD. With the conversion webhook unreachable, the LIST fails, the informer never syncs, and `WaitForCacheSync` for the graph builder times out — so **the whole dependency graph builder fails to start**, and cascading deletion stops for *every* type, not only the affected CRD. That is the reported behaviour in kubernetes/kubernetes#110720, and the log signature is: `Failed to watch *v1.PartialObjectMetadata: failed to list *v1.PartialObjectMetadata: conversion webhook for <group>/<version>, Kind=<Kind> failed`, then `unable to sync caches for garbage collector`, then `timed out waiting for dependency graph builder sync`. From cluster symptoms alone you would notice: owner-referenced children that outlive parents deleted well past when the GC should have swept them, across multiple unrelated types; namespaces stuck in `Terminating` with `NamespaceContentRemaining`; kube-controller-manager logs showing repeated informer-sync failures naming `PartialObjectMetadata`; and separately, admission latency or errors for the affected CRD's other version. The nasty part is ordering: **reads of existing objects do not need the webhook**, so the CRD looks healthy under `kubectl get` while GC is already broken — the GC symptom often appears before anyone notices admission is affected. The fix is the same root cause as an admission outage — restore reachability: certificates (expiry is the usual culprit), the Service and its endpoints, the network path, and the `caBundle` — and the durable mitigation is to treat "add a second CRD version with a conversion webhook" as taking a cluster-wide availability dependency, with the webhook deployed HA and monitored accordingly.

## Connections & what's next

This lesson is where lesson 05's CRD schemas and lesson 04's cache/workqueue machinery meet. The Manager's shared cache *is* the reflector/informer pipeline of lesson 04, which is why cache scoping is a memory-arithmetic problem and why a missing field index shows up as a CPU bill. `Status().Update()` behaves the way it does *only* because of the `/status` subresource lesson 05 declared, and `observedGeneration` is trustworthy *only* because of the generation rule in the same registry strategy. The garbage collector is itself lesson 04's machine, run over every type at once — which is what turns lesson 05's conversion-webhook decision into a cluster-wide availability decision. And the finalizer discipline here is the module's first real distributed-coordination pattern alongside leader election: a lease is a mutex with a timeout, a finalizer is a reference count, an owner reference is a parent pointer — three instances of "safe coordination without shared memory," all on the same etcd-plus-watch substrate.

Next: **lesson 07** takes the RBAC surface this controller actually needs — every `Get`, `List`, `Watch`, `Create`, `Update`, `Status().Update`, and the `finalizers` and `/status` subresource grants, plus the `delete`-on-owner grant that `blockOwnerDeletion` demands — and turns it into a minimal, justified `ClusterRole` via the same marker-driven pipeline that generated your CRDs.

## References & further reading

**Primary sources**

- controller-runtime `pkg/manager` godoc — https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager — every `Options` field in §1. The defaults quoted here (`LeaseDuration 15s`, `RenewDeadline 10s`, `RetryPeriod 2s`, `GracefulShutdownTimeout 30s`, `/healthz`, `/readyz`) are constants in `internal.go`; the start ordering and the two "MUST start before any cache" warnings are in `Start`; the `OnStoppedLeading` behaviour (graceful shutdown forced to zero, error pushed to `errChan`) is in `initLeaderElector`. Audited at **v0.24.1**.
- controller-runtime `pkg/controller` godoc — https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/controller — `MaxConcurrentReconciles` (default 1), `CacheSyncTimeout` (2 min), `RecoverPanic` (true), `UsePriorityQueue` (true since v0.23), `EnableWarmup`, `ReconciliationTimeout`.
- controller-runtime `pkg/client` godoc — https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client — the split `Client`, `APIReader`, `StatusWriter`, `MergeFrom` / `MergeFromWithOptimisticLock`, `client.Apply` + `FieldOwner`, and `client.Options.Cache.DisableFor`.
- controller-runtime `pkg/cache` godoc — https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache — `ByObject` (`Namespaces`, `Label`, `Field`, `Transform`, `UnsafeDisableDeepCopy`, `EnableWatchBookmarks`, `SyncPeriod`), `DefaultNamespaces`, `ReaderFailOnMissingInformer`.
- controller-runtime `pkg/builder`, `pkg/handler`, `pkg/predicate` godoc — https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/builder — `For`/`Owns`/`Watches`, `builder.WithPredicates` vs `WithEventFilter`, `builder.MatchEveryOwner`, and the fact that `Owns()` appends `handler.OnlyControllerOwner()` by default. The predicate catalogue (`GenerationChangedPredicate`, `ResourceVersionChangedPredicate`, `LabelChangedPredicate`, `AnnotationChangedPredicate`, `And`/`Or`/`Not`) is in `pkg/predicate/predicate.go`.
- controller-runtime `pkg/controller/controllerutil` godoc — https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/controller/controllerutil — `SetControllerReference` (and its `validateOwner` rules), `SetOwnerReference`, `RemoveControllerReference`, `WithBlockOwnerDeletion`, `CreateOrUpdate`, `CreateOrPatch`, `AddFinalizer`/`RemoveFinalizer`/`ContainsFinalizer`.
- controller-runtime `pkg/finalizer` godoc — https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/finalizer — the `Finalizers` registry used in §7: `Register(key, Finalizer)` and one `Finalize(ctx, obj)` that both adds and runs finalizers, returning `Result{Updated, StatusUpdated}` telling you which write you owe.
- controller-runtime `pkg/reconcile` godoc — https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/reconcile — `Result` (note `Requeue` is deprecated in favour of `RequeueAfter`) and `TerminalError`. *Correction noted:* returning a non-nil error together with `RequeueAfter` silently drops the `RequeueAfter` and rate-limits instead; controller-runtime logs a warning.
- kubernetes.io — Garbage collection — https://kubernetes.io/docs/concepts/architecture/garbage-collection/ — foreground/background/orphan, and the caveat that only dependents *in the GC's cache* with `blockOwnerDeletion: true` block a foreground delete.
- kubernetes.io — Owners and dependents — https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/ — `blockOwnerDeletion` and the admission control on it, the cross-namespace rules, and the `OwnerRefInvalidNamespace` warning Event.
- kubernetes.io — Finalizers — https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/ — the `202 Accepted` response, the rule that finalizers cannot be *added* once `deletionTimestamp` is set, the qualified-name requirement, and the explicit warning against manually removing finalizers.
- Kubebuilder Book — using finalizers — https://book.kubebuilder.io/reference/using-finalizers.html — the canonical dance, matching the `AddFinalizer`/`ContainsFinalizer`/`RemoveFinalizer` helpers used above.
- kube-controller-manager reference — https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/ — `--concurrent-gc-syncs` (default **20**), `--enable-garbage-collector` (default **true**), and the control plane's own leader-election defaults (**15s / 10s / 2s**), which are the same three numbers controller-runtime uses.

**Real-world engineering sources — with what each shows**

- kubernetes/kubernetes#110720 — https://github.com/kubernetes/kubernetes/issues/110720 — a down conversion webhook stalling the GC's dependency-graph builder cluster-wide, with the `PartialObjectMetadata` / `unable to sync caches for garbage collector` / `timed out waiting for dependency graph builder sync` log signature.
- Gardener docs, "Fix Problematic Conversion Webhooks" — https://gardener.cloud/docs/guides/administer-shoots/conversion-webhook/ — a production runbook for diagnosing a GC stall caused by a wedged webhook in a large multi-cluster platform.
- Cloudflare Blog, "Leveraging Kubernetes virtual machines at Cloudflare with KubeVirt" — https://blog.cloudflare.com/leveraging-kubernetes-virtual-machines-with-kubevirt/ — owner-referenced cascade deletion between real CRDs (`VirtualMachineInstance` / `VirtualMachine`) in production.

**Deeper dives**

- kubernetes/sample-controller — https://github.com/kubernetes/sample-controller — the hand-written reference for a complete reconcile loop with owner-ref handling, outside the kubebuilder scaffolding. Useful for seeing what the Manager does for you.
- Kubernetes blog, "Using Finalizers to Control Deletion" — https://kubernetes.io/blog/2021/05/14/using-finalizers-to-control-deletion/ — a walkthrough of the deletion lifecycle from the project, including the interaction between finalizers and owner references.
