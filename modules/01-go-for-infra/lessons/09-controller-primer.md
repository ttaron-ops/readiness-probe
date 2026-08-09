---
lesson: "01.9"
title: "Controller primer — CRD, reconcile, envtest"
module: "01"
concept: "Controller primer"
status: not-started
est_time: "16h"
artifacts: []
---

# 01.9 · Controller primer — CRD, reconcile, envtest

> **Concept.** Build a minimal but real controller: a CRD plus a reconcile loop that drives status toward spec, tested with envtest. The seed of the GPU cost/efficiency operator.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Why this matters

Read the JDs you're targeting. CoreWeave, NVIDIA, Cisco-class GPU shops all say the same thing in different words: *"build and extend Kubernetes controllers, operators, and CRDs in Go."* That sentence is the job. Not "know what a CRD is" — you have a CKA, you know. The job is writing the Go machinery that watches cluster state and reconciles it, in a way that survives a busy API server, partial failures, and 40 clusters' worth of edge cases.

Every prior lesson feeds this one. Types and interfaces (2, 3) become your API structs and the `Reconciler`. Errors (2) become requeue-vs-fail decisions. Concurrency + `context` (4) is literally the reconcile signature and the manager's worker pool. Testing (5) becomes envtest. Modules and stdlib (6, 7) are the dependency graph you'll pin. This lesson is where they converge into the thing you'll actually ship: a controller that computes GPU cost/efficiency and writes it back as status. Module 01 ends one commit into your flagship operator.

## The controller mental model

**Level-triggered, not edge-triggered.** An edge-triggered system reacts to *events* ("a pod was created"). A level-triggered system reacts to *state* ("here is the pod, is the world consistent with it?"). Controllers are level-triggered. `Reconcile` is handed an object's name and namespace — a `Request`, not an event — and its job is: *look at the current world, look at desired state, make them agree.* It is never told what changed, and it must not care. If you write code that assumes "this fired because the user edited the spec," you have a bug the day two edits coalesce into one reconcile, or a resync fires with no edit at all.

**Desired (spec) vs observed (status).** The user owns `.spec` — the intent. The controller owns `.status` — what it observed and computed. Reconcile reads spec, does work, writes status. It never writes its own spec (that's the user's field), and the user never writes status (that's yours). Keeping this boundary clean is what makes the loop composable.

**Idempotency.** Because reconcile runs an unknown number of times for the same state — resyncs, requeues, restarts, duplicate watch events all replay it — every reconcile must be safe to repeat. "Create the thing if absent, otherwise leave it" is idempotent. "Increment a counter" is not. Compute the target state from inputs and converge to it; don't accumulate side effects.

**The path from cluster to your function.** You never poll the API server. controller-runtime wires this for you:

```
API server  --watch-->  Informer  -->  Cache (shared in-memory store)
                                          |
                                     (indexed reads)
                                          |
   change event --> Workqueue (dedup + rate-limit) --> Reconcile(ctx, req)
```

An **informer** does one `LIST` then a long-lived `WATCH`, streaming deltas into a **cache**. Your `client.Get` reads from that cache, not the network — so a thousand reconciles cost near-zero API calls. Events are deduplicated by object key in the **workqueue**: ten rapid edits to one object collapse into (often) one reconcile, and failures requeue with exponential backoff. This is why you design for level-triggered convergence — the queue is allowed to drop, coalesce, and reorder edges; only the current level is guaranteed.

## Core notes

**Scaffolding (kubebuilder v4.x, Go 1.24+, controller-runtime v0.21+).**

```bash
kubebuilder init --domain example.com --repo github.com/you/gpu-operator
kubebuilder create api --group cost --version v1alpha1 --kind CostBudget
# answer y to "Create Resource" and "Create Controller"
```

`init` lays down `go.mod`, `main.go` (the manager), `Makefile`, `config/` (kustomize manifests), and the Dockerfile. `create api` adds `api/v1alpha1/costbudget_types.go` (your Go structs) and `internal/controller/costbudget_controller.go` (your `Reconcile`). `make manifests` regenerates CRD YAML and RBAC from the markers; `make generate` regenerates `DeepCopy` methods. Run both after touching types or markers.

**The API types + markers.** Markers are `+kubebuilder:...` comments that drive code/CRD/RBAC generation.

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

**The Reconcile signature and Result semantics.**

```go
func (r *CostBudgetReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error)
```

`req` carries only `NamespacedName`. Return values steer the workqueue:

- `return ctrl.Result{}, nil` — done; requeue only when a watch fires again.
- `return ctrl.Result{}, err` — failed; requeue with **exponential backoff** (this is your retry).
- `return ctrl.Result{RequeueAfter: 5*time.Minute}, nil` — succeeded, but check again after a fixed delay (polling an external cost API, say).

Note: the bare `Requeue bool` field is **deprecated** in recent controller-runtime — return an `error` to retry, or set `RequeueAfter` to poll. Don't reach for `Requeue: true`.

**The client: Get / Update / Status().Update.** `client.Client` reads from the cache and writes to the API server.

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

`r.Update` writes spec+metadata; `r.Status().Update` writes the status subresource. Use the latter for anything you compute — it won't fight a user's spec edit, and it's the half of the object you own. Both can return a conflict (`apierrors.IsConflict`) if the cached copy is stale; the standard response is to return the error and let the backoff replay you with a fresh read.

**Owner references + finalizers (brief).** If your controller *creates* child objects (a Deployment, a ConfigMap), set an owner reference via `controllerutil.SetControllerReference(&cb, &child, r.Scheme)` — Kubernetes then garbage-collects the child when the parent is deleted, and edits to the child re-enqueue the parent (that's what `Owns()` below watches). **Finalizers** are the inverse: a string in `metadata.finalizers` that blocks deletion until you clear it, giving you a hook to clean up *external* resources (a cloud budget, a billing record) before the object vanishes. You add the finalizer on first reconcile, and when `DeletionTimestamp` is set you do the cleanup then remove the finalizer. Our CostBudget owns nothing external, so it needs neither — but you'll reach for both the moment the operator provisions real resources.

**Manager + Builder wiring.**

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

**RBAC markers.** These generate the ClusterRole in `config/rbac/`:

```go
// +kubebuilder:rbac:groups=cost.example.com,resources=costbudgets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=cost.example.com,resources=costbudgets/status,verbs=get;update;patch
// +kubebuilder:rbac:groups="",resources=namespaces;pods,verbs=get;list;watch
```

The `/status` line is separate because it's a separate subresource — forget it and your `Status().Update` gets a 403 in-cluster (envtest won't enforce RBAC, so this bites in prod, not tests).

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

Note the shape: **Get → observe → compute → write status → return.** No edge assumptions, no accumulated state, one owned field written. Run it a hundred times on the same cluster state and the object lands in the same place — that's the whole discipline.

## Practice

Scaffold the operator for real and prove the loop with envtest.

1. `kubebuilder init` + `kubebuilder create api --group cost --version v1alpha1 --kind CostBudget` in `../practice/gpu-cost-exporter/operator/` (or a fresh module — your call; keep it in the deliverable tree).
2. Fill `costbudget_types.go` and the reconciler as above. Inject your exporter's aggregation as the `CostSource` — the reconcile's business logic *is* the exporter core; don't re-implement it.
3. `make manifests generate` — commit the generated CRD + RBAC + DeepCopy.
4. Write an **envtest** suite (`suite_test.go` is scaffolded). envtest boots a real `kube-apiserver` + `etcd` (no kubelet, no scheduler) via `setup-envtest`, so your CRD applies and status subresources behave exactly as in-cluster. Test: create namespaces with the selector label, create a `CostBudget`, then `Eventually(...)` assert `.status.observedCostUSD` and the condition. Reconcile a second time (re-trigger or wait a resync) and assert the status is **unchanged** — that's your idempotency proof.

**Acceptance:**
- `make test` (envtest) is green.
- Reconcile is idempotent: running it twice on identical cluster state produces byte-identical status.
- The status subresource updates (assert via `.Status().Update` path — a plain `Update` must *not* be what moves status).

## Self-check

**(a) Why must `Reconcile` be idempotent and level-triggered, not edge-triggered?**
**Answer:** Because the workqueue makes no promise about *when* or *how many times* your function runs for a given object. Edits coalesce, resyncs fire with no change, failures requeue, the process restarts and replays everything from a fresh LIST. You're never told what changed — only which object to check. So you must compute desired state from current inputs and converge to it every time, safely repeatable. Any logic that assumes "this ran because X happened" or accumulates side effects breaks the first time two events collapse into one reconcile.

**(b) Difference between spec and status ownership, and why status is a subresource?**
**Answer:** `.spec` is user intent (the user writes it, the controller only reads it); `.status` is controller-observed state (the controller writes it, the user only reads it). Making `/status` a *subresource* gives it a separate write endpoint and its own resourceVersion semantics: a `Status().Update` can't accidentally overwrite a concurrent spec edit, and a user editing spec can't clobber your computed status. It also lets RBAC grant status-write independently, and keeps spec-generation bumps from being triggered by status churn.

**(c) How does the controller's cache/informer avoid hammering the API server?**
**Answer:** controller-runtime runs a shared **informer** per watched type: one initial `LIST`, then a single long-lived `WATCH` that streams deltas into an in-memory **cache**. Every `client.Get`/`List` in your reconcile reads from that cache, not the network — so N reconciles cost ~0 extra API calls. The **workqueue** deduplicates by object key (a burst of edits to one object collapses to one reconcile) and rate-limits retries with backoff. Net: the API server sees a couple of watch connections regardless of how hot your reconcile loop runs.

## Resources

- **Kubebuilder Book — CronJob tutorial** (https://book.kubebuilder.io/cronjob-tutorial/cronjob-tutorial.html). *What-for:* the canonical end-to-end walkthrough — types, markers, reconcile, owner refs, envtest. **Do it hands-on**, don't just read: type every step, run `make manifests`/`make test`. This is the single highest-leverage hour for turning "I read about controllers" into "I've built one."
- **controller-runtime godoc** (https://pkg.go.dev/sigs.k8s.io/controller-runtime). *What-for:* the authoritative API — `Manager`, `Builder`, `Reconciler`, `Result`, `client.Client`. **Skim now** to see the surface (esp. `pkg/reconcile` and `pkg/builder`), **deep-dive** when a signature surprises you. Why: you'll live in these docs; know where `RequeueAfter` and `Options` are documented.
- **"Kubernetes Programming with Go"** (https://www.oreilly.com/library/view/kubernetes-programming-with/9781484290262/). *What-for:* the client-go / apimachinery layer *under* controller-runtime — schemes, codecs, informers, work queues from first principles. **Deep-read** the informer + workqueue chapters. Why: controller-runtime hides this, and you'll debug it the day the cache does something surprising.
- **"Programming Kubernetes"** (https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/). *What-for:* the conceptual companion — controllers, informers, optimistic concurrency, API machinery explained. **Skim** the controllers + custom-resources chapters. Why: it makes the level-triggered/cache model click at a level the tutorials assume.
- **sample-controller** (https://github.com/kubernetes/sample-controller). *What-for:* the same loop written in raw client-go, no controller-runtime. **Skim** once. Why: seeing the informer/workqueue/`SetupWithManager` machinery unhidden shows you exactly what controller-runtime does for you — invaluable when you read the source (lesson 8's skill) of a controller in the wild.
