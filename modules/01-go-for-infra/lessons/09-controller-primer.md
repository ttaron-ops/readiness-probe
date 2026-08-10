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
sources: 12
---

# 01.9 · Controller primer — CRD, reconcile, envtest

> **Concept.** Build a minimal but real controller: a CRD plus a reconcile loop that drives status toward spec, tested with envtest. The seed of the GPU cost/efficiency operator.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Where this fits

Lesson 08 (Reading unfamiliar Go) gave you the skill of dropping into an unfamiliar codebase — controller-runtime, client-go, a vendored operator — and finding the load-bearing code fast: entry points, interfaces, the call graph from event to handler. This lesson is where that skill gets a specific, permanent target: the reconcile loop is the single most important piece of unfamiliar Go you will read for the rest of this program, because every later module (02's controller deep-dive, 04's GPU-on-Kubernetes device plugins and scheduler extensions, the capstone operator) is built on this shape. Everything from lessons 1–8 — types and interfaces (2, 3), errors (2), concurrency and `context` (4), testing (5), modules and stdlib (6, 7) — converges here into one artifact: a CRD, a `Reconciler`, and an envtest suite that proves it converges. This is the module's flagship, final lesson. Passing it is passing the Go gate.

## Why this matters

Read the JDs you're targeting. CoreWeave, NVIDIA, Cisco-class GPU shops all say the same thing in different words: *"build and extend Kubernetes controllers, operators, and CRDs in Go."* That sentence is the job. Not "know what a CRD is" — you have a CKA, you know. The job is writing the Go machinery that watches cluster state and reconciles it, in a way that survives a busy API server, partial failures, and 40 clusters' worth of edge cases.

The cost of not knowing this cold is specific and checkable in an interview: a candidate who says `Result{Requeue: true}` signals stale knowledge (it's deprecated); a candidate who can't explain why `Status().Update` exists instead of `Update` hasn't actually shipped a controller; a candidate who treats a conflict error on a status write as a bug rather than routine traffic will design retry logic that pages people at 3am for normal contention. None of this is exotic — it's the daily texture of the job description above.

## What's new here (calibration)

Per the [module README's skip-list](../README.md#calibrated-to-your-background--what-we-skip), this lesson never re-teaches: programming 101, pointers-as-concept, IDE/hello-world tours, OOP-in-Go, LeetCode/DSA grind, web-framework/CRUD tutorials, reflection/cgo/runtime internals, or cover-to-cover reading of the Kubernetes API. You already know what a CRD, a Deployment, and RBAC are operationally (CKA-level). What's genuinely new here, built on that base:

- The **mechanics** of how a watch event becomes a `Reconcile(ctx, req)` call — informer, cache, workqueue — not just "the controller watches things."
- The **discipline** of idempotent, level-triggered design: why it's a hard constraint, not a style preference, and what breaks when you violate it.
- The **ownership boundary** between `.spec` and `.status` as an enforced API mechanism (the status subresource), not a convention you have to remember to respect.
- The **operational gaps** envtest doesn't cover (RBAC enforcement, real API-server load) that bite in production but never in CI.
- **Scale evidence**: what actually happens to this machinery at 100,000+ nodes, with real numbers from a hyperscaler, not hand-waving about "it should be fine."

## Core concepts

### The controller mental model

**Level-triggered, not edge-triggered.** An edge-triggered system reacts to *events* ("a pod was created"). A level-triggered system reacts to *state* ("here is the pod, is the world consistent with it?"). Controllers are level-triggered. `Reconcile` is handed an object's name and namespace — a `Request`, not an event — and its job is: *look at the current world, look at desired state, make them agree.* It is never told what changed, and it must not care. If you write code that assumes "this fired because the user edited the spec," you have a bug the day two edits coalesce into one reconcile, or a resync fires with no edit at all.

Mechanically, this reduces to something simpler than it sounds: **Reconcile is a pure function over cluster state.** Get the object, observe the world, compute the target, write it, return. No memory of its own between calls — everything it needs to decide comes from a fresh read. "Idempotent, level-triggered" is really one discipline stated twice: don't let your reconcile function remember anything; recompute from inputs every time.

**Desired (spec) vs observed (status).** The user owns `.spec` — the intent. The controller owns `.status` — what it observed and computed. Reconcile reads spec, does work, writes status. It never writes its own spec (that's the user's field), and the user never writes status (that's yours). Keeping this boundary clean is what makes the loop composable. This isn't just convention: the [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) codify spec/status as the canonical shape for every built-in and custom resource, and the `/status` subresource (below) turns the convention into an enforced write boundary.

**Idempotency — and what it does *not* mean.** Because reconcile runs an unknown number of times for the same state — resyncs, requeues, restarts, duplicate watch events all replay it — every reconcile must be safe to repeat. This is not the same claim as "no side effects." A reconcile that creates, updates, or deletes objects is completely normal and idempotent *if* running it N times on unchanged inputs produces the same end state. "Create the thing if absent, otherwise leave it" (checking first, or handling `AlreadyExists` as a no-op) is idempotent. "Always `Create`, ignore the error" is not — it either panics on the second run or leaves you with an unhandled error path, depending on how you wrote it. The [controller-runtime FAQ](https://github.com/kubernetes-sigs/controller-runtime/blob/main/FAQ.md) states this as a maintainer-level rule: *"Reconcile functions should be idempotent, and should always reconcile state by reading all the state it needs, then writing updates."* That single sentence is the entire lesson's technical spine.

**The path from cluster to your function.** You never poll the API server. controller-runtime wires this for you:

```
API server  --watch-->  Informer  -->  Cache (shared in-memory store)
                                          |
                                     (indexed reads)
                                          |
   change event --> Workqueue (dedup + rate-limit) --> Reconcile(ctx, req)
```

An **informer** does one `LIST` then a long-lived `WATCH`, streaming deltas into a **cache**. Your `client.Get` reads from that cache, not the network — so a thousand reconciles cost near-zero API calls. Events are deduplicated by object key in the **workqueue**: ten rapid edits to one object collapse into (often) one reconcile, and failures requeue with exponential backoff. This is why you design for level-triggered convergence — the queue is allowed to drop, coalesce, and reorder edges; only the current level is guaranteed.

This isn't a diagram drawn for pedagogy — it's load-bearing at real scale. Google's writeup on [building a 130,000-node GKE cluster](https://cloud.google.com/blog/products/containers-kubernetes/how-we-built-a-130000-node-gke-cluster) (2026, dated snapshot) reports a control plane sustaining 1,000 pod creates/schedules per second and storing 1M+ objects, made possible by two specific API-machinery features: **KEP-2340 Consistent Reads from Cache** and **KEP-4988 Snapshottable API Server Cache** — plus a Spanner-backed store handling 13,000 QPS for lease updates. The diagram above is the same cache-first design, at a scale where "just read from the API server" would fall over.

### Scaffolding (kubebuilder v4.x, Go 1.24+, controller-runtime v0.21+)

```bash
kubebuilder init --domain example.com --repo github.com/you/gpu-operator
kubebuilder create api --group cost --version v1alpha1 --kind CostBudget
# answer y to "Create Resource" and "Create Controller"
```

`init` lays down `go.mod`, `main.go` (the manager), `Makefile`, `config/` (kustomize manifests), and the Dockerfile. `create api` adds `api/v1alpha1/costbudget_types.go` (your Go structs) and `internal/controller/costbudget_controller.go` (your `Reconcile`). `make manifests` regenerates CRD YAML and RBAC from the markers; `make generate` regenerates `DeepCopy` methods. Run both after touching types or markers.

### The API types + markers

Markers are `+kubebuilder:...` comments that drive code/CRD/RBAC generation.

```go
// +kubebuilder:object:root=true          // this is a top-level API object (gets runtime.Object)
// +kubebuilder:subresource:status        // /status is a separate subresource endpoint
// +kubebuilder:printcolumn:name="Budget",type=string,JSONPath=`.spec.monthlyBudgetUSD`
// +kubebuilder:printcolumn:name="Observed",type=string,JSONPath=`.status.observedCostUSD`
type CostBudget struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec   CostBudgetSpec   `json:"spec,omitempty"`
    Status CostBudgetStatus `json:"status,omitempty"`
}
```

`object:root=true` makes the type a real `runtime.Object`; `subresource:status` splits `/status` off so a status write can't clobber a concurrent spec edit and vice versa. Printcolumns shape `kubectl get`.

### The Reconcile signature and Result semantics

```go
func (r *CostBudgetReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error)
```

`req` carries only `NamespacedName`. Return values steer the workqueue:

- `return ctrl.Result{}, nil` — done; requeue only when a watch fires again.
- `return ctrl.Result{}, err` — failed; requeue with **exponential backoff** (this is your retry).
- `return ctrl.Result{RequeueAfter: 5*time.Minute}, nil` — succeeded, but check again after a fixed delay (polling an external cost API, say).

Note: the bare `Requeue bool` field is **deprecated** in recent controller-runtime — return an `error` to retry, or set `RequeueAfter` to poll. Don't reach for `Requeue: true`; an interviewer noticing it in your code is a real, checkable signal of stale knowledge.

This exact shape — async work-queue driven sync loops, return errors immediately rather than blocking-retrying inline, sleep-and-recheck for long external operations rather than blocking the reconcile — is not folklore, it's documented field guidance. Google Cloud's [best practices for building Kubernetes Operators](https://cloud.google.com/blog/products/containers-kubernetes/best-practices-for-building-kubernetes-operators-and-stateful-apps) states it almost line for line, and adds one more structural recommendation worth internalizing before your interview: **compartmentalize responsibilities across multiple single-purpose controllers** rather than writing one controller that does everything. Keep that in your back pocket for system-design questions — "would you split this into two controllers?" is a real thing staff-level interviewers ask.

**What's underneath `RequeueAfter` and the error-triggered retry:** the workqueue isn't a naive FIFO with a sleep. controller-runtime's default rate limiter combines a token-bucket limiter with a per-item exponential backoff, so a single flapping object gets throttled without starving every other object in the queue. If you want the internals — `DefaultControllerRateLimiter`, how per-item backoff state is tracked and reset — Daniel Mangum's [Rate Limiting in controller-runtime and client-go](https://danielmangum.com/posts/controller-runtime-client-go-rate-limiting/) unpacks exactly this, written by a SIG contributor and frequently cited for it.

### The client: Get / Update / Status().Update

`client.Client` reads from the cache and writes to the API server.

```go
var cb costv1alpha1.CostBudget
if err := r.Get(ctx, req.NamespacedName, &cb); err != nil {
    return ctrl.Result{}, client.IgnoreNotFound(err) // object deleted between enqueue and now — normal
}
// ... compute ...
if err := r.Status().Update(ctx, &cb); err != nil {   // writes /status ONLY
    return ctrl.Result{}, err
}
```

`r.Update` writes spec+metadata; `r.Status().Update` writes the status subresource. Use the latter for anything you compute — it won't fight a user's spec edit, and it's the half of the object you own. A shared struct that gets passed to the wrong call (`Update` instead of `Status().Update`) is an easy, silent bug: it compiles, it "works" in a quiet cluster, and it breaks the ownership boundary the subresource split exists to enforce — it can fight a user's concurrent spec edit and nobody notices until two controllers or a human are touching the object at once.

Both `Update` calls can return a conflict (`apierrors.IsConflict`) if the cached copy is stale. At any real concurrency — more than one controller touching overlapping objects, or your own controller running with `MaxConcurrentReconciles > 1` — conflicts are **routine, expected traffic**, not a corner case. The correct response is to return the error and let backoff replay you with a fresh read; that's a designed-in feature of the workqueue, not a workaround. Treating a conflict as a hard failure (alerting on it, retry-storming) produces spurious pages for completely normal contention. The [controller-runtime FAQ](https://github.com/kubernetes-sigs/controller-runtime/blob/main/FAQ.md) gives the two sanctioned patterns for cache staleness more generally: optimistic locking with deterministic object names (so a re-create after conflict is itself idempotent), or track-and-requeue.

### Owner references, garbage collection, and finalizers

If your controller *creates* child objects (a Deployment, a ConfigMap), set an owner reference via `controllerutil.SetControllerReference(&cb, &child, r.Scheme)`. This isn't just bookkeeping metadata — Kubernetes runs a dedicated, built-in **garbage collector controller** that watches for objects whose owners are deleted and cascades deletion via `ownerReferences`. You get a real operational choice here: **foreground deletion** (`blockOwnerDeletion` semantics) blocks the parent's deletion until all children are actually gone, useful when you need a guarantee that nothing orphaned survives; **background deletion** (the default) deletes the parent immediately and cleans up children asynchronously, which is faster but means a brief window where the parent is gone and children aren't yet. Which one you want is a real design decision, not a default to leave unexamined, the moment your controller manages anything with an external side effect (a cloud disk, a load balancer).

Owner references also drive re-enqueue: edits to an owned child re-enqueue the parent (that's what `Owns()` in the builder watches, below).

**Finalizers** are the inverse mechanism: a string in `metadata.finalizers` that blocks deletion until you clear it, giving you a hook to clean up *external* resources (a cloud budget, a billing record) before the object vanishes. You add the finalizer on first reconcile, and when `DeletionTimestamp` is set you do the cleanup then remove the finalizer. Our `CostBudget` owns nothing external, so it needs neither owner references nor finalizers — but you'll reach for both the moment the operator provisions real resources.

### Manager + Builder wiring

```go
func (r *CostBudgetReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&costv1alpha1.CostBudget{}).      // primary: reconcile these
        Owns(&corev1.ConfigMap{}).            // secondary: re-enqueue owner on child change
        Named("costbudget").
        WithOptions(controller.Options{MaxConcurrentReconciles: 4}).
        Complete(r)
}
```

`For` sets the primary resource whose changes drive reconcile. `Owns` watches owned children and maps their events back to the owner. `MaxConcurrentReconciles` is the worker-pool width — the manager runs N `Reconcile` calls in parallel across *distinct* keys (the workqueue guarantees a single key is never reconciled concurrently, so you get parallelism without per-object races). This is lesson 4's concurrency reappearing: the manager is the goroutine pool, the `ctx` you're handed is cancelled on shutdown, and you thread it into every client call so a `SIGTERM` unwinds in-flight work cleanly.

### RBAC markers

These generate the ClusterRole in `config/rbac/`:

```go
// +kubebuilder:rbac:groups=cost.example.com,resources=costbudgets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=cost.example.com,resources=costbudgets/status,verbs=get;update;patch
// +kubebuilder:rbac:groups="",resources=namespaces;pods,verbs=get;list;watch
```

The `/status` line is separate because it's a separate subresource — forget it and your `Status().Update` gets a 403 in-cluster. **envtest does not enforce RBAC.** Every envtest run passes with a missing status RBAC verb, because envtest boots a real API server but your test client authenticates as an unrestricted user by default. This is the single most dangerous "works in tests, fails in prod" gap in this whole workflow: the loop can be green in CI and 403 on day one in a real cluster. Check the generated ClusterRole by hand against every verb your reconciler actually calls; don't trust the test suite to catch this for you.

### ObservedGeneration — a reusable staleness idiom

The worked example below sets `Status.ObservedGeneration = cb.Generation`. This pattern generalizes well beyond this one CRD: `.metadata.generation` increments on every spec change (only spec — status writes don't bump it), and comparing `.status.observedGeneration` to `.metadata.generation` lets *any* consumer — a `kubectl` printcolumn, a dashboard, another controller — detect "has status caught up with the latest spec edit?" without understanding your reconcile internals at all. It's how Kubernetes' own built-in controllers (Deployments, for instance) surface staleness, and it's worth reaching for by default any time your status has meaningful lag behind spec. Kubernetes' 2026 staleness-mitigation work ([Kubernetes v1.36 blog post](https://kubernetes.io/blog/2026/04/28/kubernetes-v1-36-staleness-mitigation-for-controllers/)) extends exactly this family of idiom with more formal observability for controller staleness — worth a read once this pattern feels natural.

### What's next beyond Reconcile: admission webhooks

One boundary worth naming even though it's out of scope for this lesson: **Reconcile assumes the object it's handed is already schema-valid.** The CRD's OpenAPI schema (generated from your `+kubebuilder:validation` markers) catches type errors and simple constraints (`Minimum=0`, `Required`). It cannot express cross-field invariants — "the namespace this budget references must actually exist," "field A and field B can't both be set." That validation belongs in a **validating (or mutating) admission webhook**, a layer that runs *before* your object is even persisted, not inside Reconcile. You won't build one in this lesson — it's a module-02-and-beyond topic — but knowing where the boundary sits (schema → webhook → reconcile) is itself staff-level signal.

## Perspectives

**Developer view.** The Get→observe→compute→write-status→return shape is, mechanically, a pure function over cluster state. The entire discipline of "idempotent, level-triggered" reduces to one operational rule: don't let your reconcile function have memory of its own; recompute from inputs every time. Once you see it this way, most controller bugs are just violations of referential transparency — code that remembers something from a previous call instead of re-deriving it.

**Operator/scale view.** The informer/cache/workqueue machinery exists so that N reconciles cost ~0 marginal API-server load — not a theoretical claim, load-tested at real scale. Google's own 130,000-node GKE cluster work shows the concrete mechanisms (consistent reads from cache, a snapshottable API server cache) that make a control plane survive that many watchers without falling over. The pattern you're implementing at CRD scale is the same pattern that survives at fleet scale; you're not learning a toy version of production, you're learning the actual shape.

**Reliability/failure-mode view.** Optimistic-concurrency conflicts on `.status` writes are not corner cases at fleet scale — they're routine, expected traffic the moment more than one controller (or the same controller under `MaxConcurrentReconciles > 1`) can touch overlapping objects. The correct response — return the error, let backoff replay with a fresh read — is a designed-in feature of the workqueue, not a workaround you bolt on. Treat every conflict-handling code path as normal-case, not exceptional-case, and your alerting will stay quiet when nothing is actually wrong.

**Economics/organizational view.** Google Cloud's operator best-practices guidance states, as official field guidance, exactly the two structural decisions this lesson's worked example makes correctly: asynchronous work queues over blocking polls, and multiple single-purpose controllers over one do-everything controller. That's a strong "hyperscaler independently confirms the pattern" signal for a learner about to be interviewed on exactly this design space — you're not guessing at good architecture, you're replicating what the biggest operators of this software have already converged on.

## Real-world use cases

- **Google Cloud — [Best practices for building Kubernetes Operators and stateful apps](https://cloud.google.com/blog/products/containers-kubernetes/best-practices-for-building-kubernetes-operators-and-stateful-apps)** — official field guidance: operators should use asynchronous sync loops with work queues (not blocking polling), a reconcile that hits an error should return it immediately and let the queue schedule a resync (not block/retry inline), long-running-operation watchers should sleep-and-recheck rather than block, and responsibilities should be compartmentalized across multiple controllers. Maps almost line-for-line onto this lesson's Reconcile-signature/Result-semantics section — what it shows: the pattern you're building here is the pattern the platform vendor itself recommends.
- **Google Cloud — [How we built a 130,000-node GKE cluster](https://cloud.google.com/blog/products/containers-kubernetes/how-we-built-a-130000-node-gke-cluster)** (2026, dated snapshot) — concrete numbers: control plane sustained 1,000 pod creates/schedules per second, stored 1M+ objects, relied on two specific scalability features (KEP-2340 Consistent Reads from Cache, KEP-4988 Snapshottable API Server Cache) plus a Spanner-backed store handling 13,000 QPS for lease updates — what it shows: real evidence that "the cache is what makes N reconciles cost ~0 extra API load" isn't a simplification, it's the literal mechanism a hyperscaler leans on at extreme scale.
- **kubernetes-sigs/controller-runtime — [FAQ.md](https://github.com/kubernetes-sigs/controller-runtime/blob/main/FAQ.md)** — maintainer-stated idempotency rule ("Reconcile functions should be idempotent, and should always reconcile state by reading all the state it needs, then writing updates") plus cache-staleness handling guidance (optimistic locking with deterministic names, or track-and-requeue) — what it shows: this is a primary-source-grade citation, straight from the people who wrote the library, for the exact idempotency claim this lesson's core concepts section is built on.

## Worked example

A tiny `CostBudget` CRD: spec says "watch namespaces matching this label, this is the monthly budget"; status reports the computed observed cost and an `Exceeded`/`WithinBudget` condition. The cost computation is a placeholder here — in your flagship it's the `CostSource` aggregation from the exporter.

**`api/v1alpha1/costbudget_types.go`:**

```go
package v1alpha1

import metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

type CostBudgetSpec struct {
    // Namespaces carrying this label are included in the budget.
    // +kubebuilder:validation:Required
    NamespaceSelector map[string]string `json:"namespaceSelector"`

    // +kubebuilder:validation:Minimum=0
    MonthlyBudgetUSD int64 `json:"monthlyBudgetUSD"`
}

type CostBudgetStatus struct {
    ObservedCostUSD int64 `json:"observedCostUSD,omitempty"`
    // Generation this status reflects — lets you detect stale status.
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`
    // +listType=map
    // +listMapKey=type
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Budget",type=integer,JSONPath=`.spec.monthlyBudgetUSD`
// +kubebuilder:printcolumn:name="Observed",type=integer,JSONPath=`.status.observedCostUSD`
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

**`internal/controller/costbudget_controller.go`:**

```go
package controller

import (
    "context"

    corev1 "k8s.io/api/core/v1"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/api/meta"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/labels"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"

    costv1alpha1 "github.com/you/gpu-operator/api/v1alpha1"
)

// CostSource is the exporter's aggregation core, injected as a dependency.
type CostSource interface {
    CostForNamespaces(ctx context.Context, ns []string) (usd int64, err error)
}

type CostBudgetReconciler struct {
    client.Client
    Scheme *runtime.Scheme
    Cost   CostSource
}

// +kubebuilder:rbac:groups=cost.example.com,resources=costbudgets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=cost.example.com,resources=costbudgets/status,verbs=get;update;patch
// +kubebuilder:rbac:groups="",resources=namespaces,verbs=get;list;watch

func (r *CostBudgetReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)

    var cb costv1alpha1.CostBudget
    if err := r.Get(ctx, req.NamespacedName, &cb); err != nil {
        // Deleted between enqueue and reconcile — nothing to do, don't requeue.
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 1. Observe: list namespaces matching the spec selector (cache read, no API hit).
    var nsList corev1.NamespaceList
    sel := labels.SelectorFromSet(cb.Spec.NamespaceSelector)
    if err := r.List(ctx, &nsList, &client.ListOptions{LabelSelector: sel}); err != nil {
        return ctrl.Result{}, err // transient — backoff requeue
    }
    names := make([]string, 0, len(nsList.Items))
    for i := range nsList.Items {
        names = append(names, nsList.Items[i].Name)
    }

    // 2. Compute: business logic from the exporter. Idempotent — pure function of inputs.
    observed, err := r.Cost.CostForNamespaces(ctx, names)
    if err != nil {
        return ctrl.Result{}, err
    }

    // 3. Write status only. Converge to target — safe to repeat.
    cb.Status.ObservedCostUSD = observed
    cb.Status.ObservedGeneration = cb.Generation
    cond := metav1.Condition{
        Type:               "WithinBudget",
        Status:             metav1.ConditionTrue,
        Reason:             "UnderBudget",
        Message:            "observed cost within monthly budget",
        ObservedGeneration: cb.Generation,
    }
    if observed > cb.Spec.MonthlyBudgetUSD {
        cond.Status = metav1.ConditionFalse
        cond.Reason = "OverBudget"
        cond.Message = "observed cost exceeds monthly budget"
    }
    meta.SetStatusCondition(&cb.Status.Conditions, cond)

    if err := r.Status().Update(ctx, &cb); err != nil {
        if apierrors.IsConflict(err) {
            // Cache was stale; a fresh reconcile will replay with the new version.
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }
    log.Info("reconciled", "observedUSD", observed, "budgetUSD", cb.Spec.MonthlyBudgetUSD)
    return ctrl.Result{}, nil
}

func (r *CostBudgetReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&costv1alpha1.CostBudget{}).
        Named("costbudget").
        Complete(r)
}
```

Note the shape: **Get → observe → compute → write status → return.** No edge assumptions, no accumulated state, one owned field written. Run it a hundred times on the same cluster state and the object lands in the same place — that's the whole discipline. And notice what makes it idempotent isn't the *absence* of writes — it writes status every single time — it's that the write is a full recomputation from current inputs, not an increment or an append.

## Practice

Scaffold the operator for real and prove the loop with envtest. This feeds directly into [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md), the module's deliverable.

1. `kubebuilder init` + `kubebuilder create api --group cost --version v1alpha1 --kind CostBudget` in `../practice/gpu-cost-exporter/operator/` (or a fresh module — your call; keep it in the deliverable tree).
2. Fill `costbudget_types.go` and the reconciler as above. Inject your exporter's aggregation as the `CostSource` — the reconcile's business logic *is* the exporter core; don't re-implement it.
3. `make manifests generate` — commit the generated CRD + RBAC + DeepCopy.
4. Write an **envtest** suite (`suite_test.go` is scaffolded). envtest boots a real `kube-apiserver` + `etcd` (no kubelet, no scheduler) via `setup-envtest`, so your CRD applies and status subresources behave exactly as in-cluster. Test: create namespaces with the selector label, create a `CostBudget`, then `Eventually(...)` assert `.status.observedCostUSD` and the condition. Reconcile a second time (re-trigger or wait a resync) and assert the status is **unchanged** — that's your idempotency proof.
5. By hand — because envtest won't do this for you — read the generated `config/rbac/role.yaml` against every client call your reconciler makes, and confirm the `costbudgets/status` verb list is present. This is the check that catches the pitfall below before it reaches a real cluster.

**Acceptance:**
- `make test` (envtest) is green.
- Reconcile is idempotent: running it twice on identical cluster state produces byte-identical status.
- The status subresource updates (assert via `.Status().Update` path — a plain `Update` must *not* be what moves status).
- RBAC role reviewed by hand for the `/status` verb — not just assumed correct because tests pass.

## Common pitfalls

1. **Writing to `.spec` from inside Reconcile** — even by accident, via a shared struct that gets `Update`-called instead of `Status().Update`-called. This breaks the ownership boundary the subresource split exists to enforce, and can silently fight a user's concurrent spec edit. Correction: keep a hard mental (and, ideally, code-reviewed) rule — Reconcile calls `Status().Update`, never plain `Update`, unless it's deliberately reconciling a child object it owns.
2. **Treating `Result{Requeue: true}` as still-idiomatic** — it's deprecated in recent controller-runtime. Correction: return an `error` to get backoff-driven retry, or set `RequeueAfter` for fixed-delay polling. An interviewer noticing `Requeue: true` in new code is a real, checkable signal of stale knowledge.
3. **Assuming envtest enforces RBAC** — it does not. A missing `/status` RBAC verb passes every envtest run and then 403s in a real cluster on day one. Correction: manually diff the generated ClusterRole against every client call your reconciler makes; don't let a green test suite stand in for this check.
4. **Not handling `apierrors.IsConflict` on a `Status().Update`** — at any real concurrency, conflicts are routine, not exceptional. Correction: treat conflict as a signal to let the queue's backoff replay you with a fresh read (return `nil` or a benign retry), not as a hard failure worth alerting on.
5. **Confusing "idempotent" with "no side effects."** A reconcile can create/update/delete objects — idempotency means running it N times on unchanged inputs produces the same end state. "Create the thing if absent, otherwise leave it" is idempotent; "always `Create`" (ignoring `AlreadyExists`) is not, unless that error is specifically handled as a no-op. Correction: audit every mutating call in your reconcile for what happens on the *second* run with nothing changed.

## Self-check

- **Why must `Reconcile` be idempotent and level-triggered, not edge-triggered?**
  **Answer:** Because the workqueue makes no promise about *when* or *how many times* your function runs for a given object. Edits coalesce, resyncs fire with no change, failures requeue, the process restarts and replays everything from a fresh LIST. You're never told what changed — only which object to check. So you must compute desired state from current inputs and converge to it every time, safely repeatable. Any logic that assumes "this ran because X happened" or accumulates side effects breaks the first time two events collapse into one reconcile.

- **Difference between spec and status ownership, and why status is a subresource?**
  **Answer:** `.spec` is user intent (the user writes it, the controller only reads it); `.status` is controller-observed state (the controller writes it, the user only reads it). Making `/status` a *subresource* gives it a separate write endpoint and its own resourceVersion semantics: a `Status().Update` can't accidentally overwrite a concurrent spec edit, and a user editing spec can't clobber your computed status. It also lets RBAC grant status-write independently, and keeps spec-generation bumps from being triggered by status churn.

- **How does the controller's cache/informer avoid hammering the API server?**
  **Answer:** controller-runtime runs a shared **informer** per watched type: one initial `LIST`, then a single long-lived `WATCH` that streams deltas into an in-memory **cache**. Every `client.Get`/`List` in your reconcile reads from that cache, not the network — so N reconciles cost ~0 extra API calls. The **workqueue** deduplicates by object key (a burst of edits to one object collapses to one reconcile) and rate-limits retries with backoff. Net: the API server sees a couple of watch connections regardless of how hot your reconcile loop runs. This is the same design that lets Google's GKE control plane sustain 1,000 pod schedules/sec at 130,000 nodes without every controller separately polling.

- **Why doesn't a green envtest suite prove your controller is production-ready?**
  **Answer:** envtest boots a real `kube-apiserver` and `etcd`, so object validation, defaulting, and status-subresource semantics all behave exactly as in-cluster — but it does not enforce RBAC the way a real cluster with your generated ClusterRole bound to a real ServiceAccount does. A missing `costbudgets/status` verb, or a missing `namespaces` read verb, will pass every envtest assertion and then 403 the moment the controller runs under its actual (restricted) in-cluster identity. envtest proves reconcile *logic*; it does not prove the *permission* your controller will actually have.

- **What's the difference between foreground and background deletion for an owned child object, and when would you choose one over the other?**
  **Answer:** Both rely on the built-in garbage collector controller cascading deletion via `ownerReferences`. Background deletion (the default) deletes the parent immediately and lets the GC clean up children asynchronously — fast, but there's a window where the parent is gone and children linger. Foreground deletion (`blockOwnerDeletion`) blocks the parent's actual deletion until all children are confirmed gone — slower, but guarantees no orphaned children survive the parent, which matters when a child holds an external resource (a cloud disk, an IP allocation) you need cleaned up deterministically before considering the parent gone.

## Connections & what's next

This is the **last lesson in Module 01** — the Go gate closes here. Everything upstream (interfaces in lesson 3, concurrency and `context` in lesson 4, testing in lesson 5, stdlib fluency in lesson 7, reading unfamiliar Go in lesson 8) was building toward being able to write and trust the reconcile loop above. From here:

- Take the [**checkpoint**](../checkpoint.md) — the module's completion gate. Items 1–5 are provable from the exporter core you built across earlier lessons; items 6–7 (the controller proof and the depth probes, including the three controller questions this lesson's self-check mirrors) are provable directly from this lesson's practice work.
- Finish (or keep building on) [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md) — the CRD and reconciler here are meant to sit directly on top of the exporter's `CostSource` aggregation, not duplicate it.
- This module **unlocks two others**: [**02 — Kubernetes internals and controllers**](../../02-kubernetes-controllers/README.md) goes deep on everything this lesson only introduced — informer internals, custom schedulers, webhook admission chains, multi-controller architectures — and [**04 — GPU on Kubernetes**](../../04-gpu-on-kubernetes/README.md) is where this exact reconcile shape gets pointed at GPU-specific resources (device plugins, node feature discovery, GPU-aware scheduling). Both assume you can write and reason about a reconcile loop without re-explanation — that assumption is what this lesson exists to earn.

## References & further reading

**Primary sources**
- [kubernetes-sigs/controller-runtime — FAQ.md](https://github.com/kubernetes-sigs/controller-runtime/blob/main/FAQ.md) — read for the maintainer-stated idempotency rule and the two sanctioned patterns for handling cache staleness (optimistic locking with deterministic names, or track-and-requeue).
- [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) — read for the canonical spec/status/subresource/object-reference conventions underpinning this lesson's ownership-boundary section.
- [kubernetes/enhancements — sig-api-machinery KEPs](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery) — read for the primary source of the API-machinery evolution referenced in this lesson (e.g. the consistent-reads-from-cache and snapshottable-cache KEPs cited in the 130k-node post) — the "where do I go to see WHY" answer.
- [controller-runtime godoc](https://pkg.go.dev/sigs.k8s.io/controller-runtime) — the authoritative API surface: `Manager`, `Builder`, `Reconciler`, `Result`, `client.Client`. Skim now for the surface (esp. `pkg/reconcile` and `pkg/builder`); deep-dive when a signature surprises you.

**Real-world engineering blogs**
- [Google Cloud — Best practices for building Kubernetes Operators and stateful apps](https://cloud.google.com/blog/products/containers-kubernetes/best-practices-for-building-kubernetes-operators-and-stateful-apps) — what it shows: official guidance for async work queues, error-return-not-block, sleep-and-recheck, and multi-controller compartmentalization — nearly a direct spec for this lesson's Result-semantics section.
- [Google Cloud — How we built a 130,000-node GKE cluster](https://cloud.google.com/blog/products/containers-kubernetes/how-we-built-a-130000-node-gke-cluster) (2026, dated snapshot) — what it shows: concrete scale numbers (1,000 schedules/sec, 1M+ objects, 13,000 QPS lease updates) and the two specific KEPs that make cache-first reconciliation hold up at extreme scale.
- [Kubernetes blog — Kubernetes v1.36: Staleness Mitigation and Observability for Controllers](https://kubernetes.io/blog/2026/04/28/kubernetes-v1-36-staleness-mitigation-for-controllers/) — what it shows: how the `observedGeneration` staleness idiom this lesson teaches is being formalized further upstream, with better observability for when controllers fall behind.

**Deeper dives**
- Daniel Mangum — [Rate Limiting in controller-runtime and client-go](https://danielmangum.com/posts/controller-runtime-client-go-rate-limiting/) — unpacks the token-bucket + exponential per-item backoff (`DefaultControllerRateLimiter`) underneath every `RequeueAfter`/error-return decision in this lesson.
- **Kubebuilder Book — CronJob tutorial** (https://book.kubebuilder.io/cronjob-tutorial/cronjob-tutorial.html) — the canonical end-to-end walkthrough: types, markers, reconcile, owner refs, envtest. Do it hands-on — type every step, run `make manifests`/`make test`. The single highest-leverage hour for turning "I read about controllers" into "I've built one."
- **"Kubernetes Programming with Go"** (https://www.oreilly.com/library/view/kubernetes-programming-with/9781484290262/) — the client-go / apimachinery layer *under* controller-runtime: schemes, codecs, informers, work queues from first principles. Deep-read the informer + workqueue chapters — controller-runtime hides this, and you'll debug it the day the cache does something surprising.
- **"Programming Kubernetes"** (https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/) — the conceptual companion: controllers, informers, optimistic concurrency, API machinery explained. Skim the controllers + custom-resources chapters — it makes the level-triggered/cache model click at a level the tutorials assume.
- **sample-controller** (https://github.com/kubernetes/sample-controller) — the same loop written in raw client-go, no controller-runtime. Skim once — seeing the informer/workqueue/`SetupWithManager` machinery unhidden shows exactly what controller-runtime does for you, invaluable when you read the source (lesson 8's skill) of a controller in the wild.
