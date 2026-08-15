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
sources: 8
---

# 02.6 · controller-runtime in depth

> **Concept.** The Manager, cache, client, owner references, and finalizers are the machinery that turns a `Reconcile` function into a correct, restart-safe controller — wire them wrong and you get stale reads, stuck deletes, and orphaned children.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Lesson 05 gave your three CRDs a schema, a `/status` subresource, and a versioning plan. This lesson is where that schema becomes a running binary: the Manager that shares one cache and one client across every controller, the Builder that decides what wakes a reconcile, the owner references that let the cluster's own garbage collector do cascade deletion for free, and the finalizer that gives your `WorkloadCost` one guaranteed last chance to record spend before it disappears. Everything here is downstream of a fact lesson 04 already established — the cache is eventually consistent, reads and writes are different paths — applied now to the specific write contract lesson 05's `/status` subresource created.

## Why this matters

You already know the reconciliation model in the abstract. This lesson is where it becomes a running binary that survives leader failover, doesn't hammer the API server, cleans up after itself, and never leaves a namespace with a half-deleted object that blocks teardown across your 40 clusters. controller-runtime hands you all of this, but only if you use the right primitive at the right moment: read from the cache except when you can't, write status through the status subresource, wire owner references so the garbage collector does cascade deletion for free, and run a finalizer so a `WorkloadCost` records its final spend before it disappears — money data you can't recover after the object is gone.

The failure modes here are the ones that generate incidents: a controller that calls `Update()` and wipes status, a finalizer that never returns and wedges a namespace in `Terminating` forever, a child object that outlives its parent because the owner reference was missing. Every one of these is a small API detail with a large blast radius. This is also exactly the depth CoreWeave's and NVIDIA's controller/operator postings expect — "builds the control plane" and "extend Kubernetes components" both cash out, day to day, as get-this-lesson-right-or-page-someone. Senior work is knowing which detail governs which failure.

## What's new here (calibration)

**CKA already knows:** that controllers watch and reconcile, that `ownerReferences` exist (you've seen `kubectl get` show a pod owned by a ReplicaSet owned by a Deployment), that `kubectl delete` sometimes cascades, that finalizers can wedge a namespace in `Terminating` (you've force-removed one under duress), and leader election on control-plane components.

**Internal to learn now:** how the **Manager** assembles a shared informer cache, typed clients, leader election, and the metrics/health servers so all your controllers share one connection and one cache. The difference between the cached **Client** and the live **APIReader**, and when a cache read will lie to you. The **Builder** (`For`/`Owns`/`Watches`) and **predicates** that decide what actually triggers a reconcile. Why `Status().Update()` and `Update()` are different write paths and what each destroys if misused. How `SetControllerReference` writes an owner reference and how the API server's garbage collector uses it. The full **finalizer** dance and how `foreground`/`background`/`orphan` deletion policies change ordering. And `MaxConcurrentReconciles` plus honoring `ctx` — the concurrency and cancellation you learned in the Go module, applied here.

## Core concepts

**The Manager owns the shared machinery.** One `manager.Manager` per binary. It builds the shared cache (informers), the clients, leader election, and the metrics + health-probe servers. Every controller you register runs against the same cache and client — that's why a well-built operator with ten controllers still holds roughly one watch per type, not ten.

```go
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    Scheme:                 scheme,
    LeaderElection:         true,
    LeaderElectionID:       "gpu-cost-operator.cost.example.com",
    HealthProbeBindAddress: ":8081",
    Metrics:                metricsserver.Options{BindAddress: ":8080"},
})
_ = mgr.AddHealthzCheck("healthz", healthz.Ping)
_ = mgr.AddReadyzCheck("readyz", healthz.Ping)
```

Leader election means you can run multiple replicas for availability but only the leader reconciles — no double-writes. Health/readiness probes let the pod be restarted or held out of rotation. These aren't optional polish for a production operator; they're the difference between "runs on my kind cluster" and "runs on 40 real ones."

**Client (cached) vs APIReader (live).** `mgr.GetClient()` reads through the cache — fast, but *eventually consistent*. Right after you `Create` an object, a cache read may not see it yet (the informer hasn't received the watch event). For the 99% case this is fine and correct. When you need a guaranteed-fresh read — a read-modify-write where staleness causes a wrong decision, or reading a resource you deliberately excluded from the cache — use `mgr.GetAPIReader()`, which bypasses the cache and hits the API server directly. Writes (`Create`/`Update`/`Delete`/`Status().Update`) always go straight to the API server; only reads are cached.

**`Status().Update()` vs `Update()` — different subresources, different consequences.** With the `/status` subresource enabled (lesson 05):
- `client.Update(ctx, obj)` writes to the **main** resource. It persists `spec` and `metadata`, and the API server **ignores `.status` in the payload**. It bumps `metadata.generation` if spec changed.
- `client.Status().Update(ctx, obj)` writes to the **`/status`** subresource. It persists `.status` and **ignores `.spec`**. It does not bump generation.

The classic bug: you compute status, populate `obj.Status`, and call `Update()`. The status write is silently dropped (main resource ignores status), and if you'd also mutated spec you may have written stale spec back over a user's edit. **Always** write computed observed-state through `Status().Update()` (or `Status().Patch()`), and use plain `Update()`/`Patch()` only for spec, finalizers, and labels. Prefer patch over update under concurrency to avoid `Update()`'s optimistic-lock conflicts on a stale `resourceVersion`.

```go
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

**Builder: `For`, `Owns`, `Watches`, predicates.** The Builder declares what wakes your reconciler:
- `For(&v1alpha1.WorkloadCost{})` — the primary type this controller reconciles.
- `Owns(&v1alpha1.WorkloadCost{})` — watch a child type; when a child changes, enqueue its **owner** (via the owner reference). This is how a parent reconciler reacts to its children.
- `Watches(&source, handler)` — watch an arbitrary type and map its events to reconcile requests of your type (e.g. a `GPUCostPolicy` change should re-reconcile every `WorkloadCost` referencing it).
- **Predicates** filter events *before* they enqueue. `predicate.GenerationChangedPredicate{}` skips status-only updates (generation unchanged) so you don't reconcile your own status writes in a loop — a common source of hot loops.

```go
return ctrl.NewControllerManagedBy(mgr).
    For(&v1alpha1.WorkloadCost{}).
    Owns(&v1alpha1.WorkloadCost{}). // if a parent owns child WorkloadCosts
    WithOptions(controller.Options{MaxConcurrentReconciles: 4}).
    WithEventFilter(predicate.GenerationChangedPredicate{}).
    Complete(r)
```

**Owner references and `SetControllerReference`.** An owner reference on a child says "this object belongs to that parent." `controllerutil.SetControllerReference(owner, child, scheme)` stamps it (setting `controller: true` and `blockOwnerDeletion`). Two payoffs: `Owns()` uses it to enqueue the owner on child changes, and the **garbage collector** uses it to delete children when the parent goes away. Owner and child must be in the same namespace (or the owner cluster-scoped); cross-namespace owner refs are invalid and the GC will treat the child as orphaned.

```go
if err := controllerutil.SetControllerReference(&parent, child, r.Scheme); err != nil {
    return ctrl.Result{}, err
}
if err := r.Create(ctx, child); err != nil {
    return ctrl.Result{}, client.IgnoreAlreadyExists(err)
}
```

**The garbage collector is itself just another controller.** It's tempting to think of "GC" as apiserver magic, but kube-controller-manager's garbage collector builds an in-memory DAG from `ownerReferences` across every watched type by running the *exact same* reflector → informer → workqueue machine lesson 04 taught. It watches every resource type in the cluster, maintains the owner→children graph, and when a parent's `deletionTimestamp` clears (or the parent is gone), it enqueues the children for deletion. There is no separate mechanism — cascading delete is a controller you didn't write, reconciling a graph instead of a single object. This is the same through-line as lesson 01's "no shared memory, only watch": the GC has no privileged access to your objects beyond what any other watcher has.

**Garbage collection: foreground / background / orphan.** When you delete a parent, the `deletionPropagation` policy decides child handling:
- **Background** (default for most `kubectl delete`): parent is deleted immediately; the GC deletes children asynchronously afterward.
- **Foreground**: parent enters deletion, gets a `foregroundDeletion` finalizer, and is **not removed until all children with `blockOwnerDeletion: true` are gone first** — children deleted before parent. Use when the parent must outlive its children during teardown.
- **Orphan**: children's owner references are stripped and they survive the parent.

You rarely set this in controller code — you rely on owner refs so the *cluster's* GC does the cascade — but you must understand it to explain why a delete did or didn't cascade, and to pick the propagation policy on your own `Delete` calls.

**A conversion-webhook outage silently stalls GC for that type.** This is a sharp, citable failure mode tying lesson 05 directly to this one: kubernetes/kubernetes#110720 documents that when a CRD's conversion webhook is unreachable, the GC's cache sync for that type fails, and cascading deletion for objects of that type **silently stops** — no loud error, just children that stop getting cleaned up. If you ever add a second version with a conversion webhook to one of your CRDs, a webhook outage doesn't just break admission of new writes; it breaks garbage collection of everything already owned under that type.

**Finalizers: the pre-delete hook.** A finalizer is a string in `metadata.finalizers`. Its presence means the API server will **not** actually delete the object on `kubectl delete`; instead it sets `metadata.deletionTimestamp` and leaves the object visible in `Terminating`. Your reconciler then runs cleanup and removes the finalizer; only when the list is empty does the object vanish. The dance:

```go
const finalizer = "cost.example.com/record-final-spend"

func (r *WorkloadCostReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var wc v1alpha1.WorkloadCost
    if err := r.Get(ctx, req.NamespacedName, &wc); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    if wc.DeletionTimestamp.IsZero() {
        // Not being deleted: ensure our finalizer is present.
        if controllerutil.AddFinalizer(&wc, finalizer) {
            if err := r.Update(ctx, &wc); err != nil { // finalizer lives in metadata -> plain Update
                return ctrl.Result{}, err
            }
        }
        // ... normal reconcile: compute spend, Status().Update ...
        return ctrl.Result{}, nil
    }

    // Being deleted: run cleanup, then drop the finalizer.
    if controllerutil.ContainsFinalizer(&wc, finalizer) {
        if err := r.recordFinalSpend(ctx, &wc); err != nil {
            return ctrl.Result{}, err // retry; object stays in Terminating until this succeeds
        }
        controllerutil.RemoveFinalizer(&wc, finalizer)
        if err := r.Update(ctx, &wc); err != nil {
            return ctrl.Result{}, err
        }
    }
    return ctrl.Result{}, nil
}
```

Key properties: add the finalizer via **plain `Update()`** (it's in metadata, not status). Cleanup must be **idempotent** — you may reconcile the terminating object multiple times before the finalizer is removed. And cleanup that fails should return an error so it retries, but must eventually succeed or the object is stuck. Never do unbounded blocking work in cleanup.

**The stuck-object failure and how to avoid it.** If your controller is down, crashing, or the cleanup always errors, the finalizer is never removed and the object sits in `Terminating` indefinitely — and if it's a namespace-scoped object blocking `foregroundDeletion`, it can wedge namespace teardown. Avoidance: keep cleanup idempotent and bounded, alert on objects stuck in `Terminating`, and treat a "can never succeed" cleanup (e.g. depends on a deleted external system) as a bug to design around — don't rely on operators hand-editing `metadata.finalizers` as the recovery path, though that (`kubectl patch ... -p '{"metadata":{"finalizers":[]}}'`) is the break-glass.

**Concurrency and context.** `MaxConcurrentReconciles: N` runs N reconciles in parallel *across different objects* (the workqueue guarantees a single object is never reconciled concurrently with itself). Bump it for throughput on large clusters; keep reconciles independent and idempotent so parallelism is safe. Honor `ctx` — pass it to every client call and any external I/O — so that on leader loss or shutdown, in-flight reconciles cancel promptly instead of writing after you've lost the lease.

## Perspectives

**Developer perspective.** `Status().Update()` vs `Update()` looks like a one-line API choice, but it has outsized consequences — treat it as the canonical "know your write path" lesson for anyone building on any subresource-based API, not just this operator. The same discipline applies to `/scale`, `/finalizers`, and any custom subresource you add later.

**Operator/fleet perspective.** A stuck finalizer isn't a code bug in isolation — it's an operational incident. A namespace wedged in `Terminating` blocks every subsequent operation in that namespace, and it doesn't stay contained to one cluster if the same controller bug is running fleet-wide. The break-glass (`kubectl patch` to empty `finalizers`) is a last resort that skips cleanup entirely; know its cost before reaching for it under pressure.

**Distributed-systems perspective.** Leader election, finalizers, and owner-ref GC are three independent mechanisms all solving the same underlying problem — safe coordination without shared memory — via the same substrate: objects in etcd, watched through the apiserver. Recognizing that they're three instances of one pattern (not three unrelated features to memorize) is what lets you reason about a fourth mechanism you haven't seen yet.

**Money/FinOps perspective (module-specific).** The finalizer recording final spend before a `WorkloadCost` disappears is the *only* chance to capture that number — there is no "recompute it later" once the object and its underlying usage data are gone. Design review should ask "what happens to unrecorded spend if this finalizer's cleanup can never succeed" as a first-class question, not an afterthought discovered in an incident review.

## Real-world use cases

- **kubernetes/kubernetes #110720** — "CRD Conversion webhook down results in controller-manager GC failure." https://github.com/kubernetes/kubernetes/issues/110720 — primary-source-grade evidence, from a k/k issue rather than a blog, that a conversion-webhook outage silently stalls the garbage collector's cascade deletion for that type.
- **Gardener docs, "Fix Problematic Conversion Webhooks"** — https://gardener.cloud/docs/guides/administer-shoots/conversion-webhook/ — real production operational guidance from SAP's large-scale multi-cluster management project, describing how a broken VPN/network path to a conversion webhook causes kube-controller-manager to eventually stop garbage collection.
- **Cloudflare Blog, "Leveraging Kubernetes virtual machines at Cloudflare with KubeVirt"** — https://blog.cloudflare.com/leveraging-kubernetes-virtual-machines-with-kubevirt/ — owner-referenced `VirtualMachineInstance` → `VirtualMachine` cascade deletion running in real Cloudflare production infrastructure, a concrete instance of this lesson's owner-ref/GC material at scale.

## Worked example

Wire the `WorkloadCost` controller: reconcile observed GPU usage into status, own a child `WorkloadCost` per workload with an owner reference, and record final spend via a finalizer on delete.

```go
func (r *WorkloadCostReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)

    var wc v1alpha1.WorkloadCost
    if err := r.Get(ctx, req.NamespacedName, &wc); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // ---- deletion path: finalizer records final spend ----
    if !wc.DeletionTimestamp.IsZero() {
        if controllerutil.ContainsFinalizer(&wc, finalizer) {
            final, err := r.meterGPUHours(ctx, &wc) // read final usage
            if err != nil {
                return ctrl.Result{}, err
            }
            wc.Status.ComputedSpend = r.price(ctx, &wc, final).String()
            meta.SetStatusCondition(&wc.Status.Conditions, metav1.Condition{
                Type: "Finalized", Status: metav1.ConditionTrue,
                Reason: "FinalSpendRecorded", Message: "final spend recorded before deletion",
            })
            if err := r.Status().Update(ctx, &wc); err != nil { // persist spend...
                return ctrl.Result{}, err
            }
            controllerutil.RemoveFinalizer(&wc, finalizer)
            if err := r.Update(ctx, &wc); err != nil { // ...then release the object
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{}, nil
    }

    // ---- normal path: ensure finalizer, reconcile status ----
    if controllerutil.AddFinalizer(&wc, finalizer) {
        if err := r.Update(ctx, &wc); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{Requeue: true}, nil // re-read after metadata write
    }

    hours, err := r.meterGPUHours(ctx, &wc)
    if err != nil {
        return ctrl.Result{}, err
    }
    wc.Status.ObservedGPUHours = hours
    wc.Status.ComputedSpend = r.price(ctx, &wc, hours).String()
    wc.Status.ObservedGeneration = wc.Generation
    meta.SetStatusCondition(&wc.Status.Conditions, metav1.Condition{
        Type: "Ready", Status: metav1.ConditionTrue,
        Reason: "SpendComputed", ObservedGeneration: wc.Generation,
    })
    if err := r.Status().Update(ctx, &wc); err != nil {
        return ctrl.Result{}, err
    }
    return ctrl.Result{RequeueAfter: time.Minute}, nil
}

func (r *WorkloadCostReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&v1alpha1.WorkloadCost{}).
        WithOptions(controller.Options{MaxConcurrentReconciles: 4}).
        Complete(r)
}
```

Note the deletion path writes status **first** (record the spend) and removes the finalizer **second** (release the object) — order matters: if you dropped the finalizer first, the object could be GC'd before the spend write lands. And normal-path status goes through `Status().Update()` while the finalizer goes through `Update()`.

For the owned-child case (a parent kind creating one `WorkloadCost` per workload), stamp the owner ref and add `Owns(&v1alpha1.WorkloadCost{})` to the parent's builder so child status changes re-enqueue the parent, and so deleting the parent cascades to its `WorkloadCost` children via the GC.

**What if the finalizer's cleanup depends on a system that's gone?** Suppose the pricing API `r.price()` calls to compute final spend is unreachable at delete time — a real risk, since delete can happen long after the workload finished and infrastructure has moved on. Walk it through: `recordFinalSpend` returns an error, `Reconcile` returns that error, the object stays in `Terminating`, and the workqueue's exponential backoff climbs toward its cap while retrying. Two honest mitigations, and you should be able to defend the choice between them: (1) **bounded retries with an alert** — after N attempts or a time budget, emit a `FinalizationFailed` condition (which the object won't show to `kubectl get` once it's mid-delete, so this has to go to an event and a metric, not just status) and page a human, keeping the object `Terminating` until someone intervenes; or (2) **a time-boxed fallback** — after a bounded window, record `computedSpend: "unknown"` with a `Reason: PricingUnavailable` condition and let the finalizer clear anyway, trading a guaranteed-accurate invoice for a bounded teardown SLA. For a cost operator, (1) is usually right for high-value workloads and (2) for low-value ones — the design review question from the Perspectives section above is exactly what decides which.

## Practice

Extend `gpu-cost-operator`:

1. Implement `WorkloadCostReconciler.Reconcile` to meter observed GPU usage (stub the metering with a deterministic function for now) and write `observedGPUHours` + `computedSpend` into status via `Status().Update()`, setting `observedGeneration` and a `Ready` condition.
2. Have the reconciler **create and own** a child `WorkloadCost` per discovered workload using `controllerutil.SetControllerReference` before `Create`, and add `Owns(&v1alpha1.WorkloadCost{})` in `SetupWithManager`.
3. Add the `cost.example.com/record-final-spend` finalizer: added on the normal path, and on deletion it computes and persists final spend to status before removing itself.
4. Set `MaxConcurrentReconciles: 4` and thread `ctx` through every client call.

**Acceptance:**
- Deleting a parent object cascades: its child `WorkloadCost`(s) are garbage-collected (verify with `kubectl get workloadcost` before/after and by inspecting `metadata.ownerReferences` on a child).
- Deleting a `WorkloadCost` directly leaves it briefly in `Terminating`, the finalizer runs and writes a final `computedSpend` + `Finalized` condition to status, **then** the object disappears (verify the status was persisted, e.g. via an audit log or a `kubectl get -o yaml` caught mid-terminate, or an event the reconciler emits).
- Status is written exclusively through `Status().Update()`; a `git grep` shows no `r.Update` call carrying computed status fields.
- The controller does not hot-loop (add `predicate.GenerationChangedPredicate{}` or gate status writes on change) — reconcile count per object is bounded, not continuous.

This is checkpoint items 2 and 3 directly: reconcile producing correct, owner-referenced status, and a lifecycle where deletion cascades via GC and the finalizer records final spend first.

## Common pitfalls

1. **Adding the finalizer and doing "real work" in the same reconcile before re-fetching.** This risks a race between the metadata write and whatever you compute next against a now-stale in-memory copy. The worked example above does this correctly — `Requeue: true` immediately after the finalizer's metadata write, forcing a fresh `Get` next pass. Copy that pattern.
2. **Assuming `Owns()` alone is enough without also stamping `SetControllerReference`.** `Owns()` wires the *watch* (child changes enqueue the parent); the owner reference is what the *garbage collector* needs. Skip the stamp and you get children that watch correctly but don't cascade-delete on parent deletion — a subtle gap that only shows up when someone actually deletes the parent.
3. **Treating `foreground` deletion as strictly "safer."** It can also mean a parent that *looks* deleted (has a `deletionTimestamp`, shows as `Terminating`) sits around indefinitely if a child's own finalizer never clears — silently changing your teardown SLA from "the delete API call returns" to "every child's cleanup succeeds first."

## Self-check

**(a) What breaks if you call `Update()` instead of `Status().Update()`?**

**Answer:** With the `/status` subresource enabled, `client.Update()` writes the **main** resource and the API server **ignores the `.status`** portion of the payload — so your computed spend, GPU-hours, and conditions are silently dropped and never persisted. Worse, `Update()` also writes `.spec` and `.metadata` from your in-memory object, so if you'd mutated or hold a stale spec you can clobber a user's edit and bump `metadata.generation` needlessly. The status subresource exists precisely so observed state has its own write path; computed status must go through `Status().Update()` (or `Status().Patch()`), and `Update()` is reserved for spec, finalizers, and labels.

**(b) What happens on delete if a finalizer never returns, and how do you avoid a stuck object?**

**Answer:** With a finalizer present, `kubectl delete` doesn't remove the object — the API server sets `metadata.deletionTimestamp` and the object stays visible in `Terminating`. The object is only actually deleted once every finalizer is removed. If your cleanup never succeeds (controller down, crash-looping, or cleanup that always errors), the finalizer is never removed and the object is **stuck in `Terminating` forever** — and if it blocks `foregroundDeletion` it can wedge namespace teardown. Avoid it by making cleanup **idempotent and bounded** (safe to run repeatedly, no unbounded blocking), returning errors so it retries but ensuring it can actually succeed, honoring `ctx` so shutdown cancels cleanly, and alerting on objects stuck in `Terminating`. Break-glass is `kubectl patch` to empty `metadata.finalizers`, but that skips cleanup and is a last resort, not a design.

**(c) How do owner references drive GC, and what's foreground vs background deletion?**

**Answer:** An owner reference on a child names its parent; `SetControllerReference` stamps it with `controller: true` and `blockOwnerDeletion: true`. The cluster's garbage collector watches these: when the parent is deleted, it deletes children that reference it — cascade deletion for free, no controller code. **Background** deletion (the common default) removes the parent immediately and the GC deletes children asynchronously afterward. **Foreground** deletion adds a `foregroundDeletion` finalizer to the parent so the parent is **not removed until all children with `blockOwnerDeletion: true` are deleted first** — children go before parent. (Orphan strips the owner refs and lets children survive.) Owner and child must share a namespace, or the owner be cluster-scoped; a cross-namespace owner ref is invalid and the child is treated as orphaned.

**(d) A CRD's conversion webhook has been down for an hour. What happens to garbage collection of that type's owned children, and how would you detect it from cluster symptoms alone?**

**Answer:** Per kubernetes/kubernetes#110720, the garbage collector's cache sync for that type fails while the conversion webhook is unreachable, and cascading deletion for objects of that type **silently stops** — parents that get deleted no longer trigger their children's removal, with no loud error surfaced anywhere obvious. From cluster symptoms alone, you'd notice: deleted parents whose owner-referenced children are still `kubectl get`-able well past when GC should have swept them; the GC controller's own health/sync metrics (`controller_runtime_reconcile_errors_total`-equivalent on kube-controller-manager, or its logs) showing repeated failures for that resource's informer sync; and, separately, the conversion webhook's own availability metrics or a spike in admission latency/errors for that CRD's other version. The fix is the same root cause as an admission outage — restore the webhook's reachability (certs, service, network path) — but the GC-stall symptom can appear before anyone notices admission is affected, since existing objects don't need the webhook to keep serving reads.

## Connections & what's next

This lesson is where lesson 05's CRD schemas and lesson 04's cache/workqueue machinery meet: the Manager's shared cache is the same reflector/informer pipeline, `Status().Update()` behaves the way it does because of the `/status` subresource lesson 05 declared, and a conversion-webhook outage (05's versioning story) is what silently stalls the GC described here. The finalizer discipline here is also the first real distributed-coordination pattern in the module, alongside leader election from lesson 01 — both are "safe coordination without shared memory," the same idea in two different guises. Next: **lesson 07** takes the RBAC surface this controller actually needs — every `Get`/`List`/`Create`/`Status().Update` call site you just wrote — and turns it into a minimal, justified `ClusterRole` via the same marker-driven pipeline that generated your CRDs.

## References & further reading

**Primary sources**
- controller-runtime godoc — client package — https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client — ground truth for the `Client`/`StatusClient`/`APIReader` interfaces and `Status().Update`/`Patch`; deep-read especially the cache-vs-direct-read documentation.
- controller-runtime godoc — manager package — https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager — the Manager options (leader election, cache, metrics/health) you set in `main.go`.
- Kubebuilder Book — using finalizers — https://book.kubebuilder.io/reference/using-finalizers.html — the canonical finalizer dance (add on create, cleanup on `deletionTimestamp`, remove finalizer), matching the `AddFinalizer`/`ContainsFinalizer`/`RemoveFinalizer` helpers you call verbatim.
- kubernetes.io — Garbage collection — https://kubernetes.io/docs/concepts/architecture/garbage-collection/ — how owner references drive cascading deletion and the foreground/background/orphan propagation policies from the API-server side.

**Real-world engineering blogs**
- kubernetes/kubernetes issue #110720 — https://github.com/kubernetes/kubernetes/issues/110720 — what it shows: a conversion-webhook outage silently stalling GC's cache sync for a CRD type, in production.
- Gardener docs, "Fix Problematic Conversion Webhooks" — https://gardener.cloud/docs/guides/administer-shoots/conversion-webhook/ — what it shows: real operational guidance from a large-scale multi-cluster project on diagnosing a GC stall caused by a wedged webhook.
- Cloudflare Blog, "Leveraging Kubernetes virtual machines at Cloudflare with KubeVirt" — https://blog.cloudflare.com/leveraging-kubernetes-virtual-machines-with-kubevirt/ — what it shows: owner-ref cascade deletion between real CRDs (`VirtualMachineInstance`/`VirtualMachine`) running in production.

**Deeper dives**
- kubernetes/sample-controller — https://github.com/kubernetes/sample-controller — the primary hand-written reference for reading a complete, real reconcile loop with owner-ref handling end to end, outside the kubebuilder scaffolding.
