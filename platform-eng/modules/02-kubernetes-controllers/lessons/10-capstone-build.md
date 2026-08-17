---
lesson: "02.10"
title: "Capstone build — gpu-cost-operator v0.1 under envtest"
module: "02"
concept: "Capstone build — gpu-cost-operator v0.1 under envtest"
status: not-started
est_time: "30h"
prev: "09-scheduler-and-gpu-scheduling.md"
next: null
artifacts: []
sources: 14
---

# 02.10 · Capstone build — gpu-cost-operator v0.1 under envtest

> **Concept.** Assemble CRDs, controller, finalizer, validating webhook, and least-privilege RBAC into `gpu-cost-operator` v0.1, and prove it with envtest: table-driven reconcile tests that assert idempotency and level-triggering, plus a webhook test — then demo and defend it.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Lesson 09 kept the scheduling layer at design-doc altitude on purpose. Everything else in this module — component internals (01), API machinery (02), the reconcile model (03), informers/caches/workqueues (04), CRD design (05), [controller-runtime mechanics + finalizers/GC (06)](06-controller-runtime-deep.md), [kubebuilder/RBAC (07)](07-kubebuilder-and-rbac.md), and admission webhooks (08) — has been built as a separate, testable piece. This lesson is where those pieces stop being separate.

It is also the one lesson in the module that is **executable end to end**. Everything below is a real command in a real order, producing real files, ending in a green `make test` against a real API server. You wire CRDs, reconciler, finalizer, webhook, and RBAC into one binary and prove — with tests, not a demo click-through — that it has the properties a production controller needs: idempotency, level-triggering, and least privilege. Passing this lesson's acceptance criteria *is* passing the [module checkpoint](../checkpoint.md).

## Why this matters

Everything in this module has been a component. This lesson is the **integration**, and integration is where the interesting bugs live. An RBAC rule that only fails once the webhook and the controller both run under one ServiceAccount. A cert path that is right in `config/` and wrong in the test suite. A status write that works in isolation and hot-loops the workqueue once a second controller starts watching the same object. None of those are visible in a component test; all of them are visible the first time you assemble the binary.

The bar at CoreWeave/NVIDIA is not "it reconciled in my kind cluster." It is: **can you demo it, and can you defend the test strategy?** *"How do you know your reconcile is idempotent?"* and *"what does envtest not cover, and how do you compensate?"* are the questions that separate someone who followed a tutorial from someone who owns operators at scale. A green `make test` that runs a real apiserver, asserts idempotency and level-triggering, and includes a webhook accept/reject test *is* the differentiator — and it is a differentiator precisely because most candidates have only ever written the happy-path test the scaffold gave them.

There is also a money argument specific to this operator. `WorkloadCost` carries the only durable record of what a workload spent. If the finalizer's cleanup is not idempotent, you double-charge; if it is not ordered correctly, you lose the number entirely. Those are financial defects, and the tests in this lesson are what stop them shipping.

## What's new here (calibration)

As an operator you *trust* controllers to converge and *observe* the result with `kubectl`. As the author you must **prove convergence yourself, without a cluster**, in CI, deterministically. Already know / skip vs genuinely new:

- **Already know, skip:** Go table-driven test structure, `t.Run` subtests, `go test` mechanics — you have written idiomatic Go tests for years (Module 01).
- **Already know, skip:** what "green CI" means operationally and why you gate merges on it.
- **New here:** the **whole-project assembly order** — which command writes which file, what `PROJECT` looks like after each, and how `main.go` grows one block per API and per webhook.
- **New here:** treating idempotency and level-triggering as *properties asserted in code*, with a concrete assertion technique for each (`resourceVersion` stability and semantic status equality for the first; seeded-drift rows for the second).
- **New here:** the precise envtest fidelity boundary — a real apiserver and etcd, therefore real admission, real CEL, real status subresources, real optimistic concurrency; **no** kubelet, scheduler, controller-manager, or garbage collector, therefore no running Pods and no cascade deletion.
- **New here:** the **typed** webhook API in controller-runtime v0.24 — `ctrl.NewWebhookManagedBy(mgr, &v1alpha1.WorkloadCost{}).WithValidator(...)`, with validators whose methods take your concrete type rather than `runtime.Object`. If you learned webhooks from an older tutorial, the signatures have changed.
- **New here:** CI cost/time tradeoffs. envtest starts real binaries — seconds per suite, not milliseconds — so "fake client for the bulk, envtest where API semantics matter" is a cost decision as much as a fidelity one.

The shift is from "did the pod come up?" to three engineering commitments: **idempotency** (reconciling the same observed state twice produces no additional change), **level-triggering** (the controller converges from any current state, including transitions it never witnessed), and **least privilege** (the shipped RBAC is exactly the verbs the code uses, generated from markers and verified against a live API server).

## Core concepts

**Versions this lesson's commands and code apply to.** Everything below was written against **kubebuilder v4.12** (`layout: go.kubebuilder.io/v4`, `PROJECT` schema `version: "3"`), **controller-tools v0.21.0**, **controller-runtime v0.24.1**, **k8s.io/api v0.36.0** (Kubernetes 1.36), **kustomize v5.8.1**, **Ginkgo v2.28 / Gomega v1.39**, and **Go 1.26**. These are the versions the module's lessons 06 and 07 pinned, and the scaffolded `Makefile` derives `ENVTEST_VERSION` and `ENVTEST_K8S_VERSION` from `go.mod`, so keeping `go.mod` correct keeps the toolchain correct. Where an API changed recently — the typed webhook builder, `omitzero` JSON tags, `applyconfiguration` generation — the lesson says so.

### 1. What v0.1 is, as one picture

Before any command, hold the finished shape. Six things must exist and be wired to each other through the manager:

```
  gpu-cost-operator v0.1 — component wiring
  ═════════════════════════════════════════════════════════════════════════

                    ┌──────────────────── kube-apiserver ────────────────────┐
                    │                                                        │
   kubectl apply ──▶│  admission chain                    etcd               │
   workloadcost.yaml│  ┌──────────────────────┐            ▲                 │
                    │  │ 1. authn / authz     │            │                 │
                    │  │ 2. mutating webhooks │            │                 │
                    │  │ 3. OBJECT SCHEMA     │◀── CRD OpenAPI + CEL         │
                    │  │    (structural+CEL)  │    config/crd/bases/*.yaml   │
                    │  │ 4. VALIDATING WEBHOOK│──┐         │                 │
                    │  │ 5. persist ──────────┼──┼─────────┘                 │
                    │  └──────────────────────┘  │                           │
                    └────────▲───────────────────┼───────────────────────────┘
                             │ watch/list/get    │ HTTPS :9443
                             │ create/update     │ AdmissionReview
                             │ patch (status)    │
   ┌─────────────────────────┼───────────────────┼──────────────────────────┐
   │  manager binary (cmd/main.go)               │                          │
   │  ┌──────────────────────┴───────────────┐   │                          │
   │  │ Manager                              │   │                          │
   │  │  · shared informer Cache (one watch  │   │                          │
   │  │    per GVK, shared by all reconcilers│   │                          │
   │  │  · delegating Client (cache reads,   │   │                          │
   │  │    live writes) + APIReader (live)   │   │                          │
   │  │  · leader-election Lease             │   │                          │
   │  │      gpu-cost-system/                │   │                          │
   │  │      da1d9c86.example.com            │   │                          │
   │  │  · metrics :8443, /healthz /readyz   │   │                          │
   │  └───┬──────────────┬──────────────┬────┘   │                          │
   │      │              │              │        │                          │
   │  ┌───▼──────────┐ ┌─▼───────────┐ ┌▼────────┴──────────────────────┐   │
   │  │ WorkloadCost │ │ Budget      │ │ webhook server                 │   │
   │  │ Reconciler   │ │ Reconciler  │ │  /validate-cost-example-com-   │   │
   │  │              │ │             │ │    v1alpha1-workloadcost       │   │
   │  │ For(WC)      │ │ For(Budget) │ │  WorkloadCostValidator         │   │
   │  │ Owns(CM)     │ │ Owns(WC)    │ │   · ValidateCreate             │   │
   │  │ finalizer:   │ │ Watches(    │ │   · ValidateUpdate             │   │
   │  │  cost.       │ │  GPUCost    │ │  rejects a WC whose projected  │   │
   │  │  example.com/│ │  Policy)    │ │  spend breaches an active      │   │
   │  │  record-     │ │ index:      │ │  Budget with enforcement=block │   │
   │  │  final-spend │ │  spec.      │ └────────────────────────────────┘   │
   │  └──────┬───────┘ │  policyRef  │                                      │
   │         │         └─────────────┘                                      │
   │         │ SetControllerReference → ownerReferences                     │
   │         ▼                                                              │
   │   ConfigMap "<name>-report"  (owned child; cascade-deleted with parent) │
   └────────────────────────────────────────────────────────────────────────┘

   Identity: ServiceAccount gpu-cost-system/gpu-cost-operator-controller-manager
             bound to ClusterRole manager-role  ← generated from
             +kubebuilder:rbac markers, zero wildcards (lesson 07)

   API surface (group cost.example.com/v1alpha1):
     GPUCostPolicy  cluster-scoped   price table, $/GPU-hour by GPU class
     WorkloadCost   namespaced       one per attributed workload, /status
     Budget         namespaced       limit + window + enforcement, /status
```

Three wiring facts in that picture are the ones people get wrong at integration time:

- **The webhook and the controller are the same binary and the same ServiceAccount.** A missing RBAC verb used only by the webhook's `List` call shows up as a rejected `create` in production and as nothing at all in a controller-only test.
- **`Owns()` and `SetControllerReference()` are independent** (lesson 06 §6). `Owns()` gives you a watch; the owner reference gives you cascade deletion. You need both, and envtest gives you *neither* half of the cascade, because it runs no garbage collector.
- **The CRD's CEL rules run before the webhook.** Anything you can express in the schema should be expressed there: it is cheaper, it is enforced even if your webhook pod is down, and it cannot be bypassed by a `failurePolicy: Ignore`.

### 2. Scaffolding: the exact commands, in order

Three commands create the project; two more per API; one per webhook. Run them in this order.

```console
$ mkdir gpu-cost-operator && cd gpu-cost-operator
$ go mod init github.com/you/gpu-cost-operator
$ kubebuilder init --domain example.com --repo github.com/you/gpu-cost-operator \
      --project-name gpu-cost-operator
INFO Writing kustomize manifests for you to edit...
INFO Writing scaffold for you to edit...
INFO Get controller runtime:
$ go get sigs.k8s.io/controller-runtime@v0.24.1
INFO Update dependencies:
$ go mod tidy
Next: define a resource with:
$ kubebuilder create api
```

Then one `create api` per kind. `--resource` scaffolds the types, `--controller` scaffolds the reconciler; omit `--controller` for a type you do not reconcile directly.

```console
$ kubebuilder create api --group cost --version v1alpha1 --kind GPUCostPolicy \
      --resource --controller=false --namespaced=false
$ kubebuilder create api --group cost --version v1alpha1 --kind WorkloadCost \
      --resource --controller
$ kubebuilder create api --group cost --version v1alpha1 --kind Budget \
      --resource --controller
INFO Writing kustomize manifests for you to edit...
INFO Writing scaffold for you to edit...
INFO api/v1alpha1/budget_types.go
INFO internal/controller/budget_controller.go
INFO Update dependencies:
$ go mod tidy
INFO Running make:
$ make generate
```

Note `--namespaced=false` on `GPUCostPolicy`: it is a cluster-scoped price table, which is why lesson 06's reconciler fetched it with `client.ObjectKey{Name: wc.Spec.PolicyRef}` and no namespace. That single flag is what writes `namespaced: false` into `PROJECT` and `scope: Cluster` into the generated CRD.

Then the webhook. `--programmatic-validation` scaffolds a validating webhook (as opposed to `--defaulting` for a mutating one):

```console
$ kubebuilder create webhook --group cost --version v1alpha1 --kind WorkloadCost \
      --programmatic-validation
INFO Writing kustomize manifests for you to edit...
INFO Writing scaffold for you to edit...
INFO internal/webhook/v1alpha1/workloadcost_webhook.go
INFO internal/webhook/v1alpha1/workloadcost_webhook_test.go
INFO internal/webhook/v1alpha1/webhook_suite_test.go
```

The resulting tree — this is the real kubebuilder v4.12 layout, with `(gen)` marking files a make target overwrites:

```
gpu-cost-operator/
├── PROJECT                                   inventory (see lesson 07 §1)
├── Makefile                                  pins CONTROLLER_TOOLS_VERSION=v0.21.0,
│                                             KUSTOMIZE_VERSION=v5.8.1; derives
│                                             ENVTEST_VERSION + ENVTEST_K8S_VERSION
│                                             from go.mod
├── Dockerfile
├── go.mod / go.sum
├── cmd/
│   └── main.go                               HAND-EDITED: scheme, manager, one block
│                                             per reconciler + one per webhook
├── api/v1alpha1/
│   ├── groupversion_info.go                  SchemeBuilder / AddToScheme
│   ├── gpucostpolicy_types.go                HAND-EDITED
│   ├── workloadcost_types.go                 HAND-EDITED
│   ├── budget_types.go                       HAND-EDITED
│   ├── zz_generated.deepcopy.go              (gen · make generate)
│   └── applyconfiguration/                   (gen · make manifests)
├── internal/
│   ├── controller/
│   │   ├── workloadcost_controller.go        HAND-EDITED + rbac markers
│   │   ├── workloadcost_controller_test.go   HAND-EDITED  ← the real work
│   │   ├── budget_controller.go
│   │   ├── budget_controller_test.go
│   │   └── suite_test.go                     HAND-EDITED: envtest bootstrap
│   ├── webhook/v1alpha1/
│   │   ├── workloadcost_webhook.go           HAND-EDITED + webhook markers
│   │   ├── workloadcost_webhook_test.go
│   │   └── webhook_suite_test.go             HAND-EDITED: envtest + webhook server
│   └── cost/                                 reused CostSource + aggregation
│                                             from Module 01's gpu-cost-exporter
├── config/
│   ├── crd/bases/cost.example.com_*.yaml     (gen · make manifests)
│   ├── rbac/role.yaml                        (gen) ClusterRole manager-role
│   ├── rbac/role_binding.yaml, service_account.yaml, leader_election_role.yaml
│   ├── webhook/manifests.yaml                (gen) ValidatingWebhookConfiguration
│   ├── webhook/service.yaml, config/certmanager/
│   ├── manager/manager.yaml                  the Deployment
│   ├── samples/                              example CRs
│   └── default/kustomization.yaml            top-level overlay
├── test/e2e/                                 kind-based suite (build tag e2e)
└── bin/                                      pinned tools + envtest binaries (gitignored)
```

**Never hand-edit a `(gen)` file.** If you need a rule in `role.yaml`, you need a marker; if you need a field in a CRD, you need a Go field with markers. The generated tree is a pure projection of the source tree, and CI should enforce that by running `make manifests generate` and failing on a dirty working tree.

### 3. The API types, complete

`GPUCostPolicy` — the cluster-scoped price table.

```go
// api/v1alpha1/gpucostpolicy_types.go
package v1alpha1

import (
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// GPUClassPrice prices one GPU class, e.g. "h100-sxm" at 2.50 USD per GPU-hour.
type GPUClassPrice struct {
	// class matches the value of the node label cost.example.com/gpu-class.
	// +required
	// +kubebuilder:validation:MinLength=1
	// +kubebuilder:validation:MaxLength=63
	Class string `json:"class"`

	// usdPerGPUHour is the on-demand price. Kubernetes API conventions forbid
	// float fields (they round-trip badly across languages and encodings), so
	// money is a resource.Quantity: an exact decimal with a string wire form.
	// +required
	// +kubebuilder:validation:XValidation:rule="isQuantity(self) && quantity(self).sign() >= 0",message="usdPerGPUHour must be a non-negative quantity"
	USDPerGPUHour resource.Quantity `json:"usdPerGPUHour"`
}

type GPUCostPolicySpec struct {
	// currency is fixed for v0.1; the enum exists so adding EUR later is a
	// schema change rather than an API break.
	// +required
	// +kubebuilder:validation:Enum=USD
	// +kubebuilder:default=USD
	Currency string `json:"currency"`

	// prices must name each class at most once — enforced as a set by CEL,
	// which the structural schema alone cannot express.
	// +required
	// +listType=map
	// +listMapKey=class
	// +kubebuilder:validation:MinItems=1
	// +kubebuilder:validation:MaxItems=64
	Prices []GPUClassPrice `json:"prices"`
}

type GPUCostPolicyStatus struct {
	// +optional
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`

	// +listType=map
	// +listMapKey=type
	// +optional
	Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:resource:scope=Cluster,shortName=gcp
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Currency",type=string,JSONPath=`.spec.currency`
// +kubebuilder:printcolumn:name="Classes",type=integer,JSONPath=`.spec.prices.length()`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`

type GPUCostPolicy struct {
	metav1.TypeMeta `json:",inline"`
	// +optional
	metav1.ObjectMeta `json:"metadata,omitzero"`
	// +required
	Spec GPUCostPolicySpec `json:"spec"`
	// +optional
	Status GPUCostPolicyStatus `json:"status,omitzero"`
}

// +kubebuilder:object:root=true
type GPUCostPolicyList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitzero"`
	Items           []GPUCostPolicy `json:"items"`
}

func init() { SchemeBuilder.Register(&GPUCostPolicy{}, &GPUCostPolicyList{}) }
```

Four markers there are load-bearing and worth naming:

- `+kubebuilder:object:root=true` is what makes controller-gen emit `DeepCopyObject()` for the type, which is what makes it a `runtime.Object`. Forget it and the type will not compile into a scheme.
- `+kubebuilder:subresource:status` splits `/status` into its own endpoint (lesson 05 §3), which is why `Status().Update()` and `Update()` hit different URLs and clobber different halves of the object.
- `+listType=map` + `+listMapKey=class` tells the apiserver this array is a **set keyed by `class`**, which enables server-side apply to merge per-entry instead of replacing the whole list, and makes duplicate classes a schema violation.
- `omitzero` on `ObjectMeta`/`Status` is the current (Go 1.24+) scaffold convention, replacing `omitempty` on structs — `omitempty` never actually omitted a non-pointer struct, so the old scaffolds emitted `"status":{}` on every object.

`WorkloadCost` — one per attributed workload. The status fields here are exactly the ones [lesson 06](06-controller-runtime-deep.md)'s reconciler writes.

```go
// api/v1alpha1/workloadcost_types.go
type WorkloadRef struct {
	// +required
	// +kubebuilder:validation:Enum=Deployment;StatefulSet;Job
	Kind string `json:"kind"`
	// +required
	// +kubebuilder:validation:MinLength=1
	Name string `json:"name"`
}

// +kubebuilder:validation:XValidation:rule="self.policyRef == oldSelf.policyRef",message="spec.policyRef is immutable"
type WorkloadCostSpec struct {
	// policyRef names a cluster-scoped GPUCostPolicy. Immutable: re-pricing a
	// workload mid-life would make its accumulated spend meaningless.
	// +required
	// +kubebuilder:validation:MinLength=1
	PolicyRef string `json:"policyRef"`

	// +required
	WorkloadRef WorkloadRef `json:"workloadRef"`
}

type WorkloadCostStatus struct {
	// observedGPUHours is cumulative GPU-hours attributed to this workload.
	// +optional
	ObservedGPUHours resource.Quantity `json:"observedGPUHours,omitzero"`

	// computedSpend is the string form of observedGPUHours × price.
	// +optional
	ComputedSpend string `json:"computedSpend,omitempty"`

	// +optional
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`

	// +listType=map
	// +listMapKey=type
	// +optional
	Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:resource:shortName=wc
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Policy",type=string,JSONPath=`.spec.policyRef`
// +kubebuilder:printcolumn:name="GPU-Hours",type=string,JSONPath=`.status.observedGPUHours`
// +kubebuilder:printcolumn:name="Spend",type=string,JSONPath=`.status.computedSpend`
// +kubebuilder:printcolumn:name="Ready",type=string,JSONPath=`.status.conditions[?(@.type=="Ready")].status`
type WorkloadCost struct { /* ... as above ... */ }
```

`Budget` — the enforcement object the webhook consults.

```go
// api/v1alpha1/budget_types.go
type BudgetSpec struct {
	// +required
	// +kubebuilder:validation:XValidation:rule="isQuantity(self) && quantity(self).sign() >= 0",message="limitUSD must be a non-negative quantity"
	LimitUSD resource.Quantity `json:"limitUSD"`

	// +required
	// +kubebuilder:validation:Enum=Daily;Weekly;Monthly
	Window string `json:"window"`

	// enforcement: report = status only; warn = admission warning; block = deny.
	// +required
	// +kubebuilder:validation:Enum=report;warn;block
	// +kubebuilder:default=report
	Enforcement string `json:"enforcement"`

	// selector picks the WorkloadCosts this budget governs. Empty = all in ns.
	// +optional
	Selector *metav1.LabelSelector `json:"selector,omitempty"`
}

type BudgetStatus struct {
	// +optional
	SpendUSD string `json:"spendUSD,omitempty"`
	// +optional
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`
	// +listType=map
	// +listMapKey=type
	// +optional
	Conditions []metav1.Condition `json:"conditions,omitempty"`  // Ready, BudgetExceeded
}
```

The CEL rules are worth reading as a *layering* decision, not just syntax. `isQuantity(self) && quantity(self).sign() >= 0` uses the Kubernetes CEL quantity library (available since Kubernetes 1.29) to reject a negative budget **in the apiserver, before persistence, with no webhook involved**. The immutability rule `self.policyRef == oldSelf.policyRef` uses the transition-rule form (`oldSelf` is only bound on update). Both survive your operator being completely down. Push everything you can down to this layer; the webhook is for the rules that need to read *other objects*, which CEL cannot do.

### 4. Generation: markers in, manifests out

```console
$ make manifests generate
bin/controller-gen rbac:roleName=manager-role crd webhook \
  applyconfiguration:headerFile="hack/boilerplate.go.txt" \
  paths="./..." output:crd:artifacts:config=config/crd/bases
bin/controller-gen object:headerFile="hack/boilerplate.go.txt",year=2026 paths="./..."

$ git status --short
 M config/crd/bases/cost.example.com_budgets.yaml
 M config/crd/bases/cost.example.com_gpucostpolicies.yaml
 M config/crd/bases/cost.example.com_workloadcosts.yaml
 M config/rbac/role.yaml
 M config/webhook/manifests.yaml
 M api/v1alpha1/zz_generated.deepcopy.go
```

Read a slice of the generated CRD to confirm the markers landed where you expect:

```console
$ yq '.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.limitUSD' \
     config/crd/bases/cost.example.com_budgets.yaml
anyOf:
  - type: integer
  - type: string
pattern: ^(\+|-)?(([0-9]+(\.[0-9]*)?)|(\.[0-9]+))(([KMGTPE]i)|[numkMGTPE]|([eE](\+|-)?(([0-9]+(\.[0-9]*)?)|(\.[0-9]+))))?$
x-kubernetes-int-or-string: true
x-kubernetes-validations:
  - message: limitUSD must be a non-negative quantity
    rule: isQuantity(self) && quantity(self).sign() >= 0
```

That `anyOf`/`pattern`/`x-kubernetes-int-or-string` block is what `resource.Quantity` compiles to — controller-gen knows the type and emits the canonical quantity schema. `x-kubernetes-validations` is your CEL rule, now enforced by the apiserver.

And confirm the CRD scope came through:

```console
$ yq '.spec.scope' config/crd/bases/cost.example.com_gpucostpolicies.yaml
Cluster
$ yq '.spec.scope' config/crd/bases/cost.example.com_workloadcosts.yaml
Namespaced
```

### 5. The reconciler, assembled

This is [lesson 06](06-controller-runtime-deep.md)'s `WorkloadCostReconciler` with the pieces the capstone adds: the RBAC markers from [lesson 07](07-kubebuilder-and-rbac.md), the struct fields `main.go` must fill, and the owned-child write. Nothing here contradicts lesson 06 — it is the same code with its surroundings.

```go
// internal/controller/workloadcost_controller.go
package controller

import (
	"context"
	"fmt"
	"time"

	corev1 "k8s.io/api/core/v1"
	apierrors "k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/api/equality"
	"k8s.io/apimachinery/pkg/api/meta"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/client-go/tools/record"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/builder"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/predicate"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"

	costv1alpha1 "github.com/you/gpu-cost-operator/api/v1alpha1"
)

const finalizer = "cost.example.com/record-final-spend"

type WorkloadCostReconciler struct {
	client.Client
	Scheme    *runtime.Scheme
	Recorder  record.EventRecorder
	APIReader client.Reader // live, uncached — for conflict-retry reads
	Meter     GPUMeter      // injected so tests can supply a deterministic stub
}

// GPUMeter returns cumulative GPU-hours for a workload. Production reads Pods +
// Node labels; tests inject a fixture. Injecting this is what makes reconcile
// results deterministic under envtest.
type GPUMeter interface {
	GPUHours(ctx context.Context, wc *costv1alpha1.WorkloadCost) (resource.Quantity, error)
}

// --- WorkloadCost lifecycle --------------------------------------------------
// +kubebuilder:rbac:groups=cost.example.com,resources=workloadcosts,verbs=get;list;watch;update;patch
// +kubebuilder:rbac:groups=cost.example.com,resources=workloadcosts/status,verbs=get;update;patch
// SetControllerReference sets blockOwnerDeletion on children, which the apiserver
// authorizes against the PARENT's finalizers subresource.
// +kubebuilder:rbac:groups=cost.example.com,resources=workloadcosts/finalizers,verbs=update
// --- Price table (cluster-scoped, read-only) ---------------------------------
// +kubebuilder:rbac:groups=cost.example.com,resources=gpucostpolicies,verbs=get;list;watch
// --- Owned child: the rendered cost report -----------------------------------
// +kubebuilder:rbac:groups="",resources=configmaps,verbs=get;list;watch;create;update;patch
// --- Fleet observation for metering (read-only, cluster-wide) ----------------
// +kubebuilder:rbac:groups="",resources=pods,verbs=get;list;watch
// +kubebuilder:rbac:groups="",resources=nodes,verbs=get;list;watch
// --- Events ------------------------------------------------------------------
// +kubebuilder:rbac:groups="",resources=events,verbs=create;patch

func (r *WorkloadCostReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := ctrl.LoggerFrom(ctx)

	var wc costv1alpha1.WorkloadCost
	if err := r.Get(ctx, req.NamespacedName, &wc); err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	// ---- deletion path: record final spend, then release ----
	if !wc.DeletionTimestamp.IsZero() {
		if !controllerutil.ContainsFinalizer(&wc, finalizer) {
			return ctrl.Result{}, nil
		}
		final, err := r.Meter.GPUHours(ctx, &wc)
		if err != nil {
			log.Error(err, "metering final usage; object stays Terminating")
			return ctrl.Result{}, err
		}
		spend, err := r.price(ctx, &wc, final)
		if err != nil {
			return ctrl.Result{}, err
		}
		wc.Status.ObservedGPUHours = final
		wc.Status.ComputedSpend = spend.String()
		meta.SetStatusCondition(&wc.Status.Conditions, metav1.Condition{
			Type: "Finalized", Status: metav1.ConditionTrue,
			Reason: "FinalSpendRecorded", Message: "final spend recorded before deletion",
		})
		if err := r.Status().Update(ctx, &wc); err != nil { // money first
			return ctrl.Result{}, err
		}
		r.Recorder.Eventf(&wc, corev1.EventTypeNormal, "FinalSpendRecorded",
			"recorded %s before deletion", wc.Status.ComputedSpend)
		controllerutil.RemoveFinalizer(&wc, finalizer)
		return ctrl.Result{}, r.Update(ctx, &wc) // release second
	}

	// ---- normal path ----
	if controllerutil.AddFinalizer(&wc, finalizer) {
		if err := r.Update(ctx, &wc); err != nil {
			return ctrl.Result{}, err
		}
		return ctrl.Result{Requeue: true}, nil // re-read after the metadata write
	}

	var policy costv1alpha1.GPUCostPolicy
	if err := r.Get(ctx, client.ObjectKey{Name: wc.Spec.PolicyRef}, &policy); err != nil {
		if apierrors.IsNotFound(err) {
			meta.SetStatusCondition(&wc.Status.Conditions, metav1.Condition{
				Type: "Ready", Status: metav1.ConditionFalse,
				Reason: "PolicyNotFound", ObservedGeneration: wc.Generation,
				Message: fmt.Sprintf("spec.policyRef %q does not exist", wc.Spec.PolicyRef),
			})
			_ = r.Status().Update(ctx, &wc)
			return ctrl.Result{}, reconcile.TerminalError(err) // permanent: do not retry
		}
		return ctrl.Result{}, err // transient: retry with backoff
	}

	hours, err := r.Meter.GPUHours(ctx, &wc)
	if err != nil {
		return ctrl.Result{}, err
	}
	spend, err := r.price(ctx, &wc, hours)
	if err != nil {
		return ctrl.Result{}, err
	}

	// The owned child: a rendered report ConfigMap, cascade-deleted with the parent.
	cm := &corev1.ConfigMap{ObjectMeta: metav1.ObjectMeta{
		Name: wc.Name + "-report", Namespace: wc.Namespace,
	}}
	if _, err := controllerutil.CreateOrUpdate(ctx, r.Client, cm, func() error {
		cm.Data = map[string]string{
			"gpuHours": hours.String(),
			"spendUSD": spend.String(),
			"policy":   wc.Spec.PolicyRef,
		}
		return controllerutil.SetControllerReference(&wc, cm, r.Scheme)
	}); err != nil {
		return ctrl.Result{}, err
	}

	// THE IDEMPOTENCY GUARD: compute the whole next status, then write only if it
	// differs semantically. Without this, every reconcile is a write, every write
	// is a watch event, and the controller hot-loops on itself.
	next := wc.DeepCopy()
	next.Status.ObservedGPUHours = hours
	next.Status.ComputedSpend = spend.String()
	next.Status.ObservedGeneration = wc.Generation
	meta.SetStatusCondition(&next.Status.Conditions, metav1.Condition{
		Type: "Ready", Status: metav1.ConditionTrue,
		Reason: "SpendComputed", ObservedGeneration: wc.Generation,
		Message: "spend reconciled from policy " + wc.Spec.PolicyRef,
	})
	if !equality.Semantic.DeepEqual(wc.Status, next.Status) {
		if err := r.Status().Patch(ctx, next, client.MergeFrom(&wc)); err != nil {
			return ctrl.Result{}, err
		}
	}
	return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil
}

func (r *WorkloadCostReconciler) price(ctx context.Context,
	wc *costv1alpha1.WorkloadCost, hours resource.Quantity) (resource.Quantity, error) {
	// (elided: look up the workload's GPU class from Node labels, find the
	// matching GPUClassPrice, multiply. Deterministic and pure.)
	return hours, nil
}

func (r *WorkloadCostReconciler) SetupWithManager(mgr ctrl.Manager) error {
	r.Recorder = mgr.GetEventRecorderFor("workloadcost-controller")
	r.APIReader = mgr.GetAPIReader()
	return ctrl.NewControllerManagedBy(mgr).
		Named("workloadcost").
		For(&costv1alpha1.WorkloadCost{}, builder.WithPredicates(
			predicate.Or(
				predicate.GenerationChangedPredicate{},
				predicate.LabelChangedPredicate{},
			))).
		Owns(&corev1.ConfigMap{}).
		WithOptions(controller.Options{
			MaxConcurrentReconciles: 4,
			ReconciliationTimeout:   2 * time.Minute,
		}).
		Complete(r)
}
```

**The one integration-only design decision in that file is `Meter GPUMeter`.** In production it lists Pods and reads Node labels; in a test it returns a fixed quantity. Without the seam, every reconcile test would have to seed a realistic Pod/Node fleet into envtest just to get a deterministic number out, and "did the status converge" would be entangled with "did I model the fleet correctly." Injecting the metering source is what makes the idempotency assertion in §8 a one-line comparison rather than a fixture-management project.

### 6. The validating webhook, with the current typed API

The webhook enforces the one rule CEL structurally cannot: *reject a `WorkloadCost` whose projected spend would breach an active `Budget` in the same namespace with `enforcement: block`* — because that requires reading other objects.

Note the API shape. In controller-runtime v0.24 the builder takes the object as its second argument and the validator's methods take **your concrete type**, not `runtime.Object`:

```go
// internal/webhook/v1alpha1/workloadcost_webhook.go
package v1alpha1

import (
	"context"
	"fmt"

	apierrors "k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/api/resource"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/apimachinery/pkg/util/validation/field"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/webhook/admission"

	costv1alpha1 "github.com/you/gpu-cost-operator/api/v1alpha1"
)

var wclog = logf.Log.WithName("workloadcost-webhook")

func SetupWorkloadCostWebhookWithManager(mgr ctrl.Manager) error {
	return ctrl.NewWebhookManagedBy(mgr, &costv1alpha1.WorkloadCost{}).
		WithValidator(&WorkloadCostValidator{Client: mgr.GetClient()}).
		Complete()
}

// failurePolicy=Ignore is a DELIBERATE choice, justified in §7.
// sideEffects=None asserts the handler mutates nothing outside the request,
// which is what lets the apiserver skip it during a dry-run.
// +kubebuilder:webhook:path=/validate-cost-example-com-v1alpha1-workloadcost,mutating=false,failurePolicy=ignore,sideEffects=None,groups=cost.example.com,resources=workloadcosts,verbs=create;update,versions=v1alpha1,name=vworkloadcost-v1alpha1.kb.io,admissionReviewVersions=v1

// The webhook LISTS Budgets, so it needs its own grant. This marker is why the
// controller and webhook must share one audited ClusterRole (lesson 07).
// +kubebuilder:rbac:groups=cost.example.com,resources=budgets,verbs=get;list;watch

type WorkloadCostValidator struct{ client.Client }

func (v *WorkloadCostValidator) ValidateCreate(ctx context.Context,
	wc *costv1alpha1.WorkloadCost) (admission.Warnings, error) {
	return v.check(ctx, wc)
}

func (v *WorkloadCostValidator) ValidateUpdate(ctx context.Context,
	_, wc *costv1alpha1.WorkloadCost) (admission.Warnings, error) {
	return v.check(ctx, wc)
}

func (v *WorkloadCostValidator) ValidateDelete(_ context.Context,
	_ *costv1alpha1.WorkloadCost) (admission.Warnings, error) {
	return nil, nil // deletion is never blocked: never trap a user's cleanup
}

func (v *WorkloadCostValidator) check(ctx context.Context,
	wc *costv1alpha1.WorkloadCost) (admission.Warnings, error) {
	var budgets costv1alpha1.BudgetList
	if err := v.List(ctx, &budgets, client.InNamespace(wc.Namespace)); err != nil {
		// Fail OPEN inside the handler as well as at the failurePolicy: a cache
		// blip must not become an outage for every workload creation.
		wclog.Error(err, "listing budgets; admitting", "namespace", wc.Namespace)
		return admission.Warnings{"budget check skipped: could not list Budgets"}, nil
	}

	var warnings admission.Warnings
	for i := range budgets.Items {
		b := &budgets.Items[i]
		if b.Spec.Enforcement == "report" {
			continue
		}
		projected, err := v.projectedSpend(ctx, wc, b)
		if err != nil {
			return warnings, nil // same fail-open reasoning
		}
		if projected.Cmp(b.Spec.LimitUSD) <= 0 {
			continue
		}
		msg := fmt.Sprintf("projected spend %s exceeds Budget %q limit %s (%s window)",
			projected.String(), b.Name, b.Spec.LimitUSD.String(), b.Spec.Window)
		if b.Spec.Enforcement == "warn" {
			warnings = append(warnings, msg)
			continue
		}
		// enforcement == "block": deny with a STRUCTURED error so kubectl prints
		// the offending field path, not just a sentence.
		return warnings, apierrors.NewInvalid(
			schema.GroupKind{Group: "cost.example.com", Kind: "WorkloadCost"},
			wc.Name,
			field.ErrorList{field.Forbidden(
				field.NewPath("spec", "workloadRef"), msg,
			)},
		)
	}
	return warnings, nil
}

func (v *WorkloadCostValidator) projectedSpend(ctx context.Context,
	wc *costv1alpha1.WorkloadCost, b *costv1alpha1.Budget) (resource.Quantity, error) {
	// (elided: sum status.computedSpend across the Budget's selected
	// WorkloadCosts, plus this one's projection.)
	return resource.Quantity{}, nil
}
```

Three things to read carefully:

- **`admission.Warnings` is a first-class return.** A non-empty warnings slice with a `nil` error *admits* the object and prints the strings to the client with a `Warning:` prefix. That is what `enforcement: warn` is for, and it is the reason the enum has three values rather than a boolean.
- **Return `apierrors.NewInvalid` with a `field.ErrorList`, not `fmt.Errorf`.** The apiserver renders the field path, so the user sees `spec.workloadRef: Forbidden: projected spend 812 exceeds Budget "team-ml" limit 500` instead of a bare sentence. Structured errors are the difference between a webhook people can debug and one they file tickets about.
- **`ValidateDelete` returns nil unconditionally.** A validating webhook that can block deletion is a webhook that can trap an object — combine it with a finalizer and you have manufactured a resource nobody can remove.

### 7. `failurePolicy`, and how not to lock yourself out

The generated `ValidatingWebhookConfiguration` needs three fields chosen deliberately. Patch them in `config/webhook/` (the `namespaceSelector` is not expressible as a marker in the scaffold and is added by kustomize patch):

```yaml
# config/webhook/manifests.yaml  (generated) + a kustomize patch for the selector
webhooks:
- name: vworkloadcost-v1alpha1.kb.io
  failurePolicy: Ignore            # ← chosen, not defaulted
  sideEffects: None
  timeoutSeconds: 5                # default is 10; keep admission latency bounded
  admissionReviewVersions: ["v1"]
  namespaceSelector:
    matchExpressions:
      - key: kubernetes.io/metadata.name
        operator: NotIn
        values: ["kube-system", "kube-node-lease", "gpu-cost-system"]
  rules:
  - apiGroups: ["cost.example.com"]
    apiVersions: ["v1alpha1"]
    operations: ["CREATE", "UPDATE"]
    resources: ["workloadcosts"]
    scope: Namespaced
```

| Choice | `failurePolicy: Fail` | `failurePolicy: Ignore` |
|---|---|---|
| Webhook pod down | **every** create/update of the resource is rejected | requests are admitted unvalidated |
| Blast radius | availability incident scoped to the resource — and, if the selector is wrong, to the operator's own namespace, which is a **self-lockout**: you cannot deploy the fix | a window of unenforced budgets |
| Right when | the rule is a security or correctness invariant (no privileged pods, no untrusted images) | the rule is a **policy** whose violation is recoverable after the fact |
| v0.1 choice | — | **yes** — an over-budget workload can be reconciled and reported after admission; a cluster that cannot create workloads cannot be fixed by the thing that is broken |

The `namespaceSelector` excluding `gpu-cost-system` is the anti-lockout mechanism and is mandatory regardless of policy: the webhook must never be able to block the creation of objects in its own namespace. `kubernetes.io/metadata.name` is a label the apiserver sets automatically on every namespace, so the selector needs no extra labelling step.

**Even with `Ignore`, `timeoutSeconds` matters.** Every `CREATE` on a matching resource waits for your handler. A handler that lists a large `BudgetList` on every admission adds that latency to every workload creation in the cluster; that is why the validator reads through the manager's **cache** (`mgr.GetClient()`), not a live client.

### 8. main.go: the wiring you own

`cmd/main.go` is the only hand-edited file that grows with every new API and webhook. The scaffold gives you flag parsing, TLS options, and metrics; you add one block per reconciler and one per webhook:

```go
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
	Scheme:                 scheme,
	Metrics:                metricsServerOptions,
	WebhookServer:          webhookServer,
	HealthProbeBindAddress: probeAddr,
	LeaderElection:         enableLeaderElection,
	LeaderElectionID:       "da1d9c86.example.com",
})
if err != nil {
	setupLog.Error(err, "Failed to start manager")
	os.Exit(1)
}

if err := (&controller.WorkloadCostReconciler{
	Client: mgr.GetClient(),
	Scheme: mgr.GetScheme(),
	Meter:  cost.NewPodNodeMeter(mgr.GetClient()),
}).SetupWithManager(mgr); err != nil {
	setupLog.Error(err, "Failed to create controller", "controller", "workloadcost")
	os.Exit(1)
}
if err := (&controller.BudgetReconciler{
	Client: mgr.GetClient(),
	Scheme: mgr.GetScheme(),
}).SetupWithManager(mgr); err != nil {
	setupLog.Error(err, "Failed to create controller", "controller", "budget")
	os.Exit(1)
}
// ENABLE_WEBHOOKS=false lets you run the manager locally against a cluster
// without a serving cert. This is the scaffold's convention — keep it.
if os.Getenv("ENABLE_WEBHOOKS") != "false" {
	if err := webhookv1alpha1.SetupWorkloadCostWebhookWithManager(mgr); err != nil {
		setupLog.Error(err, "Failed to create webhook", "webhook", "WorkloadCost")
		os.Exit(1)
	}
}

if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil { /* ... */ }
if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil { /* ... */ }

setupLog.Info("starting manager")
if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
	setupLog.Error(err, "Problem running manager")
	os.Exit(1)
}
```

Reading `main.go` top to bottom is the fastest audit of an unfamiliar operator: it names every reconciled type, every webhook, and whether leader election is on.

### 9. envtest: what it is, precisely

**envtest** starts a **real `kube-apiserver` and a real `etcd`** as local processes — no Docker, no kubelet, no cluster. Your code talks to a genuine API server over a genuine `*rest.Config`. That means the following are *real*, not simulated:

| Behaviour | Real under envtest? | Why it matters here |
|---|---|---|
| CRD installation + structural schema | yes | your generated CRDs are actually validated |
| CEL (`x-kubernetes-validations`) | yes | the negative-budget rejection is genuinely tested |
| Validating/mutating webhooks over TLS | yes (with `WebhookInstallOptions`) | cert paths, port, `failurePolicy`, `namespaceSelector` all exercised |
| `/status` subresource semantics | yes | `Update()` vs `Status().Update()` behave differently, as in production |
| Optimistic concurrency (`resourceVersion`, 409) | yes | conflict-retry code paths are testable |
| Watches, informers, the shared cache | yes | your controller's event wiring is exercised |
| Finalizers: `deletionTimestamp`, `202 Accepted`, `Terminating` | yes | the apiserver rules from lesson 06 §7 apply |
| RBAC objects exist and can be created | yes | but see the caveat below |
| **Pods running** | **no** | no kubelet: Pods stay `Pending` forever |
| **Scheduling** | **no** | no scheduler: `spec.nodeName` is never set |
| **Deployments → ReplicaSets → Pods** | **no** | no kube-controller-manager |
| **Owner-reference garbage collection** | **no** | no GC controller: children outlive deleted parents |
| **Namespace deletion completing** | **no** | the namespace controller is part of KCM |
| Real Nodes, GPUs, DRA drivers, CNI | **no** | — |

Two of those deserve emphasis because they change how you write tests:

**Garbage collection does not happen.** A test that deletes a `WorkloadCost` and asserts its report ConfigMap disappears will *fail*, and the failure is not your bug. What you *can* assert under envtest is the thing you actually control: that the child carries a correct `ownerReferences` entry with `controller: true` and `blockOwnerDeletion: true`. Cascade behaviour itself belongs in the kind-based e2e suite.

**RBAC exists but your test client is not restricted by it.** `testEnv.Start()` hands you an admin-equivalent config. If you want to prove your `manager-role` is sufficient, you must build a second client from an impersonating config (`cfg.Impersonate = rest.ImpersonationConfig{UserName: "system:serviceaccount:gpu-cost-system:gpu-cost-operator-controller-manager"}`) after creating the ServiceAccount, ClusterRole and binding. That is worth doing once; it is the only way an RBAC gap fails in CI rather than in production.

**Provisioning the binaries.** `setup-envtest` (from `sigs.k8s.io/controller-runtime/tools/setup-envtest`) downloads a matching `kube-apiserver`, `etcd` and `kubectl`:

```console
$ make setup-envtest
Setting up envtest binaries for Kubernetes version 1.36...
/home/you/gpu-cost-operator/bin/k8s/1.36.0-linux-amd64

$ ls bin/k8s/1.36.0-linux-amd64/
etcd  kube-apiserver  kubectl
```

The scaffolded `Makefile` derives both versions from `go.mod` rather than hardcoding them, which is what stops CI and your laptop from silently testing against different API versions:

```make
ENVTEST_VERSION     ?= $(shell ... gomodver sigs.k8s.io/controller-runtime ...)
ENVTEST_K8S_VERSION ?= $(shell ... gomodver k8s.io/api ... | sed -E 's/^v?[0-9]+\.([0-9]+).*/1.\1/')

.PHONY: test
test: manifests generate fmt vet setup-envtest
	KUBEBUILDER_ASSETS="$(shell "$(ENVTEST)" use $(ENVTEST_K8S_VERSION) --bin-dir "$(LOCALBIN)" -p path)" \
	  go test $$(go list ./... | grep -v /e2e) -coverprofile cover.out
```

Note that `make test` runs `manifests generate` first. That is not decoration: it means a test run against stale CRDs is impossible, and it is why the "CI fails on a dirty tree after generation" rule is enforceable.

### 10. The controller test suite

`internal/controller/suite_test.go` — the envtest bootstrap. This is the scaffold with the manager actually started, because idempotency tests call `Reconcile` directly while level-triggering tests want the cache warm:

```go
// internal/controller/suite_test.go
package controller

import (
	"context"
	"os"
	"path/filepath"
	"testing"

	. "github.com/onsi/ginkgo/v2"
	. "github.com/onsi/gomega"

	"k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/rest"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/envtest"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"

	costv1alpha1 "github.com/you/gpu-cost-operator/api/v1alpha1"
)

var (
	ctx       context.Context
	cancel    context.CancelFunc
	testEnv   *envtest.Environment
	cfg       *rest.Config
	k8sClient client.Client
)

func TestControllers(t *testing.T) {
	RegisterFailHandler(Fail)
	RunSpecs(t, "Controller Suite")
}

var _ = BeforeSuite(func() {
	logf.SetLogger(zap.New(zap.WriteTo(GinkgoWriter), zap.UseDevMode(true)))
	ctx, cancel = context.WithCancel(context.TODO())

	Expect(costv1alpha1.AddToScheme(scheme.Scheme)).To(Succeed())

	By("bootstrapping test environment")
	testEnv = &envtest.Environment{
		CRDDirectoryPaths:     []string{filepath.Join("..", "..", "config", "crd", "bases")},
		ErrorIfCRDPathMissing: true,
	}
	// Lets `go test ./internal/controller/...` work from an IDE without
	// KUBEBUILDER_ASSETS being exported by the Makefile.
	if d := firstEnvTestBinaryDir(); d != "" {
		testEnv.BinaryAssetsDirectory = d
	}

	var err error
	cfg, err = testEnv.Start()
	Expect(err).NotTo(HaveOccurred())
	Expect(cfg).NotTo(BeNil())

	k8sClient, err = client.New(cfg, client.Options{Scheme: scheme.Scheme})
	Expect(err).NotTo(HaveOccurred())
})

var _ = AfterSuite(func() {
	By("tearing down the test environment")
	cancel()
	Expect(testEnv.Stop()).To(Succeed())
})

func firstEnvTestBinaryDir() string {
	base := filepath.Join("..", "..", "bin", "k8s")
	entries, err := os.ReadDir(base)
	if err != nil {
		return ""
	}
	for _, e := range entries {
		if e.IsDir() {
			return filepath.Join(base, e.Name())
		}
	}
	return ""
}
```

Now the tests that matter. **Table-driven rows, one fresh namespace per row for isolation** — namespaces are cheap under envtest, and using one per row avoids the classic "test 3 passes alone, fails in the suite" problem:

```go
// internal/controller/workloadcost_controller_test.go
type fixedMeter struct{ q resource.Quantity }

func (m fixedMeter) GPUHours(context.Context, *costv1alpha1.WorkloadCost) (resource.Quantity, error) {
	return m.q, nil
}

var _ = Describe("WorkloadCost controller", func() {

	newReconciler := func(hours string) *WorkloadCostReconciler {
		return &WorkloadCostReconciler{
			Client:   k8sClient,
			Scheme:   k8sClient.Scheme(),
			Recorder: record.NewFakeRecorder(16),
			Meter:    fixedMeter{q: resource.MustParse(hours)},
		}
	}

	// seed creates an isolated namespace, a cluster-scoped policy, and a
	// WorkloadCost, and returns their keys.
	seed := func(name string) (types.NamespacedName, func()) { /* ... */ }

	Context("convergence", func() {
		It("computes status, adds the finalizer, and owns a report ConfigMap", func() {
			key, cleanup := seed("converge")
			defer cleanup()
			r := newReconciler("12")

			// First pass adds the finalizer and requeues (metadata write).
			res, err := r.Reconcile(ctx, reconcile.Request{NamespacedName: key})
			Expect(err).NotTo(HaveOccurred())
			Expect(res.Requeue).To(BeTrue())

			// Second pass does the work.
			_, err = r.Reconcile(ctx, reconcile.Request{NamespacedName: key})
			Expect(err).NotTo(HaveOccurred())

			var wc costv1alpha1.WorkloadCost
			Expect(k8sClient.Get(ctx, key, &wc)).To(Succeed())
			Expect(wc.Finalizers).To(ContainElement("cost.example.com/record-final-spend"))
			Expect(wc.Status.ObservedGPUHours.String()).To(Equal("12"))
			Expect(wc.Status.ObservedGeneration).To(Equal(wc.Generation))
			Expect(meta.IsStatusConditionTrue(wc.Status.Conditions, "Ready")).To(BeTrue())

			var cm corev1.ConfigMap
			Expect(k8sClient.Get(ctx, types.NamespacedName{
				Name: key.Name + "-report", Namespace: key.Namespace}, &cm)).To(Succeed())

			// GC does not run under envtest, so assert the owner reference itself.
			Expect(cm.OwnerReferences).To(HaveLen(1))
			Expect(*cm.OwnerReferences[0].Controller).To(BeTrue())
			Expect(*cm.OwnerReferences[0].BlockOwnerDeletion).To(BeTrue())
			Expect(cm.OwnerReferences[0].Kind).To(Equal("WorkloadCost"))
		})
	})
})
```

### 11. Asserting idempotency — the assertion that actually proves it

Idempotency is *"reconciling the same observed state again changes nothing observable."* The weak version of this test — "the second call returned no error" — passes for a reconciler that rewrites identical status every pass, which is exactly the bug you are trying to catch. That reconciler bumps `resourceVersion` on every write, each write is a watch event, each event re-enqueues the object, and the controller spins at whatever rate the workqueue's rate limiter allows, burning apiserver capacity forever.

Under a **real** apiserver, `resourceVersion` is the right instrument: a no-op does no write, and no write means no new `resourceVersion`.

```go
It("is idempotent: a second reconcile changes nothing observable", func() {
	key, cleanup := seed("idem")
	defer cleanup()
	r := newReconciler("12")

	// Drive to steady state (finalizer pass + work pass).
	_, err := r.Reconcile(ctx, reconcile.Request{NamespacedName: key})
	Expect(err).NotTo(HaveOccurred())
	_, err = r.Reconcile(ctx, reconcile.Request{NamespacedName: key})
	Expect(err).NotTo(HaveOccurred())

	var before costv1alpha1.WorkloadCost
	Expect(k8sClient.Get(ctx, key, &before)).To(Succeed())
	var cmBefore corev1.ConfigMap
	cmKey := types.NamespacedName{Name: key.Name + "-report", Namespace: key.Namespace}
	Expect(k8sClient.Get(ctx, cmKey, &cmBefore)).To(Succeed())

	// The assertion.
	res, err := r.Reconcile(ctx, reconcile.Request{NamespacedName: key})
	Expect(err).NotTo(HaveOccurred())
	Expect(res.Requeue).To(BeFalse())                       // no spurious requeue
	Expect(res.RequeueAfter).To(Equal(5 * time.Minute))      // the steady resync only

	var after costv1alpha1.WorkloadCost
	Expect(k8sClient.Get(ctx, key, &after)).To(Succeed())
	Expect(after.ResourceVersion).To(Equal(before.ResourceVersion),
		"a no-op reconcile must not write the object")
	Expect(equality.Semantic.DeepEqual(before.Status, after.Status)).To(BeTrue())

	var cmAfter corev1.ConfigMap
	Expect(k8sClient.Get(ctx, cmKey, &cmAfter)).To(Succeed())
	Expect(cmAfter.ResourceVersion).To(Equal(cmBefore.ResourceVersion),
		"CreateOrUpdate must not rewrite an unchanged child")

	// And no duplicate children were created.
	var cms corev1.ConfigMapList
	Expect(k8sClient.List(ctx, &cms, client.InNamespace(key.Namespace))).To(Succeed())
	Expect(cms.Items).To(HaveLen(1))
})
```

Four assertions, each catching a different real bug: **`ResourceVersion` equality** catches the unconditional status write; **semantic status equality** catches a status that changes in a way `resourceVersion` alone would not explain (e.g. a condition's `lastTransitionTime` churning); **the child's `resourceVersion`** catches a `CreateOrUpdate` whose mutate function is not a pure function of desired state; **the list length** catches a reconciler that generates a fresh child name each pass.

The `lastTransitionTime` case is worth a sentence on its own, because it is the most common way this test fails for a *correct-looking* reconciler. `meta.SetStatusCondition` only updates `lastTransitionTime` when the condition's `Status` actually changes — that is precisely why you use it instead of appending to `Status.Conditions` by hand. Hand-rolled condition code that stamps `metav1.Now()` every pass will fail this assertion, and it should.

### 12. Asserting level-triggering — seed the level, never the transition

Level-triggered means the controller converges from **whatever state it finds**, not from a stream of events it happened to witness. The test technique follows directly: seed a state the controller never saw come into being, reconcile **once**, and assert convergence. No row may depend on a previous row's transition.

```go
DescribeTable("level-triggering: converges from any observed state",
	func(mutate func(key types.NamespacedName), verify func(key types.NamespacedName)) {
		key, cleanup := seed("level")
		defer cleanup()
		r := newReconciler("12")

		// Reach steady state, then break it in a way the controller never observed.
		_, _ = r.Reconcile(ctx, reconcile.Request{NamespacedName: key})
		_, _ = r.Reconcile(ctx, reconcile.Request{NamespacedName: key})
		mutate(key)

		// EXACTLY ONE reconcile is allowed to fix it.
		_, err := r.Reconcile(ctx, reconcile.Request{NamespacedName: key})
		Expect(err).NotTo(HaveOccurred())
		verify(key)
	},

	Entry("drifted status is corrected",
		func(key types.NamespacedName) {
			var wc costv1alpha1.WorkloadCost
			Expect(k8sClient.Get(ctx, key, &wc)).To(Succeed())
			wc.Status.ComputedSpend = "999999"          // a human "fixed" it
			wc.Status.ObservedGPUHours = resource.MustParse("1")
			Expect(k8sClient.Status().Update(ctx, &wc)).To(Succeed())
		},
		func(key types.NamespacedName) {
			var wc costv1alpha1.WorkloadCost
			Expect(k8sClient.Get(ctx, key, &wc)).To(Succeed())
			Expect(wc.Status.ObservedGPUHours.String()).To(Equal("12"))
		}),

	Entry("deleted child is recreated",
		func(key types.NamespacedName) {
			cm := &corev1.ConfigMap{ObjectMeta: metav1.ObjectMeta{
				Name: key.Name + "-report", Namespace: key.Namespace}}
			Expect(k8sClient.Delete(ctx, cm)).To(Succeed())
		},
		func(key types.NamespacedName) {
			var cm corev1.ConfigMap
			Expect(k8sClient.Get(ctx, types.NamespacedName{
				Name: key.Name + "-report", Namespace: key.Namespace}, &cm)).To(Succeed())
			Expect(cm.Data["gpuHours"]).To(Equal("12"))
		}),

	Entry("drifted child content is corrected",
		func(key types.NamespacedName) {
			var cm corev1.ConfigMap
			k := types.NamespacedName{Name: key.Name + "-report", Namespace: key.Namespace}
			Expect(k8sClient.Get(ctx, k, &cm)).To(Succeed())
			cm.Data["spendUSD"] = "0"
			Expect(k8sClient.Update(ctx, &cm)).To(Succeed())
		},
		func(key types.NamespacedName) { /* assert spendUSD restored */ }),

	Entry("missing finalizer is re-added",
		func(key types.NamespacedName) {
			var wc costv1alpha1.WorkloadCost
			Expect(k8sClient.Get(ctx, key, &wc)).To(Succeed())
			wc.Finalizers = nil
			Expect(k8sClient.Update(ctx, &wc)).To(Succeed())
		},
		func(key types.NamespacedName) {
			var wc costv1alpha1.WorkloadCost
			Expect(k8sClient.Get(ctx, key, &wc)).To(Succeed())
			Expect(wc.Finalizers).To(ContainElement(finalizer))
		}),

	Entry("missing policy sets Ready=False and does not retry",
		func(key types.NamespacedName) { /* delete the GPUCostPolicy */ },
		func(key types.NamespacedName) {
			var wc costv1alpha1.WorkloadCost
			Expect(k8sClient.Get(ctx, key, &wc)).To(Succeed())
			c := meta.FindStatusCondition(wc.Status.Conditions, "Ready")
			Expect(c.Status).To(Equal(metav1.ConditionFalse))
			Expect(c.Reason).To(Equal("PolicyNotFound"))
		}),
)
```

The last row is the one an interviewer will probe: a *terminal* error (the referenced policy does not exist) must not be retried with backoff forever. `reconcile.TerminalError` tells controller-runtime to stop requeueing and record the failure, and the test asserts the user-visible consequence — a `Ready=False` condition naming the missing policy — rather than an internal flag.

**Also test the deletion path, and test it twice.** The terminating object is reconciled repeatedly until the finalizer clears, so the cleanup must be idempotent:

```go
It("records final spend on delete and is safe to reconcile twice", func() {
	key, cleanup := seed("delete")
	defer cleanup()
	r := newReconciler("12")
	_, _ = r.Reconcile(ctx, reconcile.Request{NamespacedName: key})
	_, _ = r.Reconcile(ctx, reconcile.Request{NamespacedName: key})

	var wc costv1alpha1.WorkloadCost
	Expect(k8sClient.Get(ctx, key, &wc)).To(Succeed())
	Expect(k8sClient.Delete(ctx, &wc)).To(Succeed())

	// The apiserver set deletionTimestamp; the object still exists (finalizer).
	Expect(k8sClient.Get(ctx, key, &wc)).To(Succeed())
	Expect(wc.DeletionTimestamp).NotTo(BeNil())

	_, err := r.Reconcile(ctx, reconcile.Request{NamespacedName: key})
	Expect(err).NotTo(HaveOccurred())

	// Finalizer removed ⇒ apiserver completes the delete.
	Eventually(func() bool {
		return apierrors.IsNotFound(k8sClient.Get(ctx, key, &costv1alpha1.WorkloadCost{}))
	}).Should(BeTrue())

	// Reconciling a now-absent object is a clean no-op, not an error.
	_, err = r.Reconcile(ctx, reconcile.Request{NamespacedName: key})
	Expect(err).NotTo(HaveOccurred())
})
```

### 13. The webhook test, and why it can only be envtest

The fake client runs **no admission at all**, so it cannot test a webhook — not the handler wiring, not the certs, not the `failurePolicy`. Under envtest, `WebhookInstallOptions` reads your generated `config/webhook/manifests.yaml`, generates a serving cert, patches the `caBundle` into the configuration, and installs it into the test apiserver. Your webhook is then reached over real TLS by real admission requests.

```go
// internal/webhook/v1alpha1/webhook_suite_test.go  (the parts that matter)
testEnv = &envtest.Environment{
	CRDDirectoryPaths:     []string{filepath.Join("..", "..", "..", "config", "crd", "bases")},
	ErrorIfCRDPathMissing: false,
	WebhookInstallOptions: envtest.WebhookInstallOptions{
		Paths: []string{filepath.Join("..", "..", "..", "config", "webhook")},
	},
}
cfg, err := testEnv.Start()
Expect(err).NotTo(HaveOccurred())

wio := &testEnv.WebhookInstallOptions
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
	Scheme: scheme.Scheme,
	WebhookServer: webhook.NewServer(webhook.Options{
		Host:    wio.LocalServingHost,
		Port:    wio.LocalServingPort,
		CertDir: wio.LocalServingCertDir,
	}),
	LeaderElection: false,
	Metrics:        metricsserver.Options{BindAddress: "0"},
})
Expect(err).NotTo(HaveOccurred())
Expect(SetupWorkloadCostWebhookWithManager(mgr)).To(Succeed())

go func() {
	defer GinkgoRecover()
	Expect(mgr.Start(ctx)).To(Succeed())
}()

// Do not skip this. Without it the first spec races the server's TLS listener
// and fails with "connection refused" perhaps one run in five.
dialer := &net.Dialer{Timeout: time.Second}
addr := fmt.Sprintf("%s:%d", wio.LocalServingHost, wio.LocalServingPort)
Eventually(func() error {
	conn, err := tls.DialWithDialer(dialer, "tcp", addr, &tls.Config{InsecureSkipVerify: true})
	if err != nil {
		return err
	}
	return conn.Close()
}).Should(Succeed())
```

The tests themselves drive the **real client**, so the assertion is on the API call's outcome, not on a handler's return value:

```go
It("rejects a WorkloadCost that would breach a blocking Budget", func() {
	Expect(k8sClient.Create(ctx, blockingBudget("team-ml", "100"))).To(Succeed())

	err := k8sClient.Create(ctx, overBudgetWorkloadCost("team-ml", "train-42"))
	Expect(err).To(HaveOccurred())
	Expect(apierrors.IsInvalid(err)).To(BeTrue())
	Expect(err.Error()).To(ContainSubstring("exceeds Budget"))

	// And it was genuinely not persisted.
	Expect(apierrors.IsNotFound(k8sClient.Get(ctx, types.NamespacedName{
		Namespace: "team-ml", Name: "train-42"}, &costv1alpha1.WorkloadCost{}))).To(BeTrue())
})

It("admits with a warning when enforcement is warn", func() {
	// admission.Warnings are surfaced through the client's WarningHandler;
	// register a collecting handler on the rest.Config to assert on them.
	Expect(k8sClient.Create(ctx, overBudgetWorkloadCost("team-warn", "train-43"))).To(Succeed())
	Expect(collected).To(ContainElement(ContainSubstring("exceeds Budget")))
})

It("admits a compliant WorkloadCost", func() {
	Expect(k8sClient.Create(ctx, cheapWorkloadCost("team-ml", "train-44"))).To(Succeed())
})

It("rejects a negative Budget in the apiserver, with no webhook involved", func() {
	b := blockingBudget("team-ml", "-1")
	err := k8sClient.Create(ctx, b)
	Expect(err).To(HaveOccurred())
	Expect(err.Error()).To(ContainSubstring("limitUSD must be a non-negative quantity"))
})
```

That last spec is the CEL test, and it belongs here rather than in the controller suite: it proves the *schema* rejects the object, which is a stronger guarantee than the webhook rejecting it, because it holds even when your operator is not running.

### 14. Fake client versus envtest, decided per test

| | fake client | envtest |
|---|---|---|
| What it is | in-memory object tracker (`pkg/client/fake`) | real `kube-apiserver` + `etcd` processes |
| Startup | microseconds | ~2–5 s per suite |
| Admission / webhooks | **none — ever** | real, over TLS |
| CEL + structural validation | not applied | applied |
| `/status` subresource | approximated (needs `WithStatusSubresource`) | real |
| `resourceVersion` / 409 conflicts | approximated | real |
| Finalizers, `deletionTimestamp` | approximated | real |
| Field indexes | must be registered explicitly | real |
| Use it for | pure reconcile-logic branches, error paths, arithmetic | anything touching API semantics |

The rule: **fake client for the bulk of fast logic tests; envtest for anything exercising real API semantics or admission.** That is a CI-cost decision as much as a fidelity one — a fleet-scale test matrix in which every suite starts an apiserver is minutes of pipeline latency per PR, multiplied by every repository and every branch.

## Perspectives

**Test-design view.** Table-driven tests plus the fake/envtest split is standard Go practice; the senior-level upgrade is treating idempotency and level-triggering as *properties asserted in code*. Concretely: the idempotency assertion compares `resourceVersion` before and after a repeat reconcile, and the level-triggering table has one row per broken *level* with no row depending on a previous transition. Both are mechanical to write once you see them, and both catch bugs that a happy-path test structurally cannot.

**Production/operations view.** The list of what envtest does not cover is a map of exactly where you still need a real cluster. Drawing that boundary correctly is part of the deliverable, not a caveat to skim: it tells you what the kind-based `test/e2e/` suite is *for* (cascade deletion, a Pod actually starting, the device plugin injecting `/dev/nvidia*`, the webhook Service resolving) and stops you from either over-building e2e or over-trusting unit tests.

**Interview/defensibility view.** "How do you know your reconcile is idempotent" and "what does envtest not cover" are the two differentiating questions for this material, and both want a specific artifact as the answer, not a definition. Rehearse pointing at a file and a line: the `ResourceVersion` equality assertion, the `DescribeTable` of drifted levels, the owner-reference assertion that stands in for the GC you cannot run. The third probe — "your controller stopped reconciling" — is answerable only once you have assembled the whole system, which is why it is the checkpoint's closing question.

**Money/FinOps view.** `WorkloadCost` is the only durable record of what a workload spent; once the object is gone the number is unrecoverable. That makes two things in this build financial controls rather than style choices: the ordering in the deletion path (status write, *then* finalizer removal), and the idempotency of the metering call (a terminating object is reconciled repeatedly, and a non-idempotent recorder double-charges). Both have a test above. Design review should ask "what happens to unrecorded spend if this finalizer can never succeed?" and the answer should already be in the code.

## Real-world use cases

- **The `kubebuilder` scaffold as the industry baseline.** The generated `Makefile` derives `ENVTEST_K8S_VERSION` from `k8s.io/api` in `go.mod` and `ENVTEST_VERSION` from `sigs.k8s.io/controller-runtime`, rather than hardcoding either. That change exists because pinning them by hand produced a well-known class of drift: a project bumps `controller-runtime` in `go.mod`, `setup-envtest` keeps downloading the old apiserver, and a behaviour change lands in production that CI never simulated. What it shows: version derivation is a correctness feature, not tidiness, and it is why §9's rule is "keep `go.mod` right" rather than "pin the Makefile."
- **The typed webhook migration in controller-runtime.** `ctrl.NewWebhookManagedBy(mgr, obj)` with validators whose methods take a concrete type replaced the older pattern where `admission.CustomValidator` received `runtime.Object` and every handler began with an unchecked type assertion. The old shape moved a whole class of "wrong object type registered on this path" bugs from compile time to admission time — that is, into production. What it shows: when you find a tutorial whose webhook code does not compile against v0.24, the tutorial is not stale in a cosmetic way; the API changed to remove a real failure mode.
- **SuperOrbital, "Testing Production Kubernetes Controllers"** — <https://superorbital.io/blog/testing-production-controllers/> — a Kubernetes consultancy's practitioner writeup recommending the same envtest + table-driven shape used here. What it shows: this is the industry-standard structure, not an idiosyncratic one, which matters when you are defending it in an interview.
- **InfraCloud, "Testing Kubernetes Operators using EnvTest"** — <https://www.infracloud.io/blogs/testing-kubernetes-operator-envtest/> — an independent consultancy describing the "no kubelet, no controller-manager" fidelity boundary in the same terms. What it shows: a second, independent corroboration of exactly where the line sits — useful because the boundary is the part people most often state wrongly.

## Worked example

The full build, executed in order, from empty directory to green suite. Transcripts are representative of a v4.12 scaffold on Linux; version strings and timings will differ on your machine.

```
  build → test → deploy, as a sequence
  ═════════════════════════════════════════════════════════════════════════

  1  kubebuilder init          ──▶ PROJECT, Makefile, cmd/main.go, config/
  2  kubebuilder create api ×3 ──▶ api/v1alpha1/*_types.go
     (GPUCostPolicy cluster-      internal/controller/*_controller.go
      scoped; WC + Budget         PROJECT gains 3 resources[] entries
      namespaced)
  3  kubebuilder create webhook──▶ internal/webhook/v1alpha1/*_webhook.go
     --programmatic-validation     PROJECT gains webhooks:{validation:true}
                                   │
  4  EDIT types + markers          │  ← the only creative step
     EDIT reconcilers + rbac       │
     EDIT webhook + failurePolicy  │
                                   ▼
  5  make manifests generate ──▶ controller-gen v0.21.0 parses ./...
        │                          ├─ crd ──────────▶ config/crd/bases/*.yaml
        │                          ├─ rbac ─────────▶ config/rbac/role.yaml
        │                          ├─ webhook ──────▶ config/webhook/manifests.yaml
        │                          ├─ object ───────▶ zz_generated.deepcopy.go
        │                          └─ applyconfig ──▶ api/v1alpha1/applyconfiguration/
        │
        ├─ CI GATE: git diff --exit-code   (generated tree must match source)
        ▼
  6  make setup-envtest ─────▶ bin/k8s/1.36.0-linux-amd64/{etcd,kube-apiserver,kubectl}
        │
        ▼
  7  make test ──────────────▶ manifests → generate → fmt → vet → go test
        │                       │
        │                       ├─ controller suite   ┌──────────────────────┐
        │                       │   envtest.Start() ─▶│ kube-apiserver +etcd │
        │                       │   install CRDs      │   (real, local)      │
        │                       │   Reconcile() ×N    │  no kubelet          │
        │                       │   assert RV stable  │  no scheduler        │
        │                       │   assert drift fix  │  no GC controller    │
        │                       │                     └──────────────────────┘
        │                       └─ webhook suite
        │                           envtest + WebhookInstallOptions
        │                           → cert generated, caBundle injected
        │                           → mgr.Start() serves TLS :random
        │                           → Create() denied / admitted for real
        ▼
  8  make docker-build docker-push IMG=...
        ▼
  9  make deploy IMG=...  ──▶ kustomize build config/default | kubectl apply -f -
        │                      CRDs, ClusterRole, binding, SA, Deployment,
        │                      Service, ValidatingWebhookConfiguration, Certificate
        ▼
 10  kubectl auth can-i --list --as=system:serviceaccount:gpu-cost-system:...
        │   ← the RBAC review artifact (lesson 07)
        ▼
 11  make test-e2e  ──▶ kind cluster: the things envtest structurally cannot do
                        (Pods actually running, GC cascade, Service resolution)
```

**Steps 1–4** are §2–§7 above. Pick up at generation.

**Step 5 — generate, and gate on it.**

```console
$ make manifests generate
$ git diff --exit-code || { echo "generated tree is stale"; exit 1; }

$ grep -c '"\*"' config/rbac/role.yaml
0

$ head -24 config/rbac/role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: manager-role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create", "get", "list", "patch", "update", "watch"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch"]
- apiGroups: [""]
  resources: ["nodes", "pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["cost.example.com"]
  resources: ["budgets", "gpucostpolicies", "workloadcosts"]
  verbs: ["get", "list", "watch"]
...
```

Note the merge: `budgets`, `gpucostpolicies` and `workloadcosts` collapsed into one rule because their `get;list;watch` verb sets and their (groups, resources, resourceNames, urls) tuple matched after normalization — the merge behaviour from lesson 07 §5. The wider verb sets on `workloadcosts` appear in a separate rule further down.

**Step 6–7 — provision binaries and run the suite.**

```console
$ make setup-envtest
Setting up envtest binaries for Kubernetes version 1.36...
/home/you/gpu-cost-operator/bin/k8s/1.36.0-linux-amd64

$ make test
bin/controller-gen rbac:roleName=manager-role crd webhook applyconfiguration:... paths="./..."
bin/controller-gen object:headerFile="hack/boilerplate.go.txt",year=2026 paths="./..."
go fmt ./...
go vet ./...
KUBEBUILDER_ASSETS="/home/you/gpu-cost-operator/bin/k8s/1.36.0-linux-amd64" \
  go test $(go list ./... | grep -v /e2e) -coverprofile cover.out

Running Suite: Controller Suite - /home/you/gpu-cost-operator/internal/controller
================================================================================
Random Seed: 1755412233

Will run 9 of 9 specs
STEP: bootstrapping test environment @ 08/17/26 11:04:02.118
•••••••••

Ran 9 of 9 Specs in 6.402 seconds
SUCCESS! -- 9 Passed | 0 Failed | 0 Pending | 0 Skipped

Running Suite: Webhook Suite - /home/you/gpu-cost-operator/internal/webhook/v1alpha1
====================================================================================
Will run 4 of 4 specs
STEP: bootstrapping test environment @ 08/17/26 11:04:09.551
••••

Ran 4 of 4 Specs in 5.118 seconds
SUCCESS! -- 4 Passed | 0 Failed | 0 Pending | 0 Skipped

ok  github.com/you/gpu-cost-operator/api/v1alpha1              0.019s  coverage: 12.4%
ok  github.com/you/gpu-cost-operator/internal/controller       7.114s  coverage: 78.9%
ok  github.com/you/gpu-cost-operator/internal/webhook/v1alpha1 5.842s  coverage: 71.3%
```

Read the timings, because they are the CI-cost argument in one line: **13.0 s of the 13.4 s total is envtest process startup and API round-trips.** The pure-logic tests in `api/v1alpha1` finish in 19 ms. That ratio is why §14's split exists.

**Seeing a failure is more instructive than seeing a pass.** Break idempotency deliberately — replace the guarded status write with an unconditional one:

```go
// deliberately wrong
if err := r.Status().Update(ctx, next); err != nil { return ctrl.Result{}, err }
```

```console
$ make test
------------------------------
• [FAILED] [0.284 seconds]
WorkloadCost controller convergence [It] is idempotent: a second reconcile
changes nothing observable

  Expected
      <string>: "1247"
  to equal
      <string>: "1243"
  a no-op reconcile must not write the object
  In [It] at: internal/controller/workloadcost_controller_test.go:141

Ran 9 of 9 Specs in 6.517 seconds
FAIL! -- 8 Passed | 1 Failed | 0 Pending | 0 Skipped
```

Four `resourceVersion` units of difference: the status write plus the watch-triggered churn behind it. In a running cluster that is not a test failure, it is a controller that reconciles the same object forever. **This is the single most valuable test in the suite, and it is four lines long.**

**Step 8–10 — deploy and audit the identity.**

```console
$ make docker-build docker-push IMG=ghcr.io/you/gpu-cost-operator:v0.1.0
$ make deploy IMG=ghcr.io/you/gpu-cost-operator:v0.1.0
namespace/gpu-cost-system created
customresourcedefinition.apiextensions.k8s.io/budgets.cost.example.com created
customresourcedefinition.apiextensions.k8s.io/gpucostpolicies.cost.example.com created
customresourcedefinition.apiextensions.k8s.io/workloadcosts.cost.example.com created
serviceaccount/gpu-cost-operator-controller-manager created
clusterrole.rbac.authorization.k8s.io/gpu-cost-operator-manager-role created
clusterrolebinding.rbac.authorization.k8s.io/gpu-cost-operator-manager-rolebinding created
service/gpu-cost-operator-webhook-service created
deployment.apps/gpu-cost-operator-controller-manager created
validatingwebhookconfiguration.admissionregistration.k8s.io/gpu-cost-operator-validating-webhook-configuration created

$ kubectl auth can-i --list \
    --as=system:serviceaccount:gpu-cost-system:gpu-cost-operator-controller-manager
Resources                                 Non-Resource URLs   Resource Names   Verbs
configmaps                                []                  []               [create get list patch update watch]
events                                    []                  []               [create patch]
nodes                                     []                  []               [get list watch]
pods                                      []                  []               [get list watch]
budgets.cost.example.com                  []                  []               [get list watch]
gpucostpolicies.cost.example.com          []                  []               [get list watch]
workloadcosts.cost.example.com            []                  []               [get list watch update patch]
workloadcosts.cost.example.com/finalizers []                  []               [update]
workloadcosts.cost.example.com/status     []                  []               [get update patch]
```

**No `secrets`. No `delete` on anything. No wildcards.** That output pasted into a PR description is the review artifact for checkpoint item 6.

**The demo, four `kubectl` commands.**

```console
$ kubectl apply -f config/samples/cost_v1alpha1_gpucostpolicy.yaml
gpucostpolicy.cost.example.com/default created

$ kubectl -n team-ml apply -f config/samples/cost_v1alpha1_workloadcost.yaml
workloadcost.cost.example.com/train-42 created

$ kubectl -n team-ml get workloadcost
NAME       POLICY    GPU-HOURS   SPEND    READY   AGE
train-42   default   12          30.00    True    8s

$ kubectl -n team-ml apply -f - <<'EOF'
apiVersion: cost.example.com/v1alpha1
kind: Budget
metadata: {name: team-ml, namespace: team-ml}
spec: {limitUSD: "-1", window: Daily, enforcement: block}
EOF
The Budget "team-ml" is invalid: spec.limitUSD: Invalid value: "-1":
  limitUSD must be a non-negative quantity
```

That last rejection came from the **CRD's CEL rule**, in the apiserver, with the operator Pod deliberately scaled to zero — which is the point of pushing validation down to the schema.

**What this flow structurally cannot catch.** Three real bugs, each needing a kind or staging cluster: (1) a GPU device plugin failing to inject `/dev/nvidia*` — envtest has no kubelet, so no container ever starts; (2) a NetworkPolicy blocking the apiserver's egress to the webhook Service — envtest calls the webhook directly on localhost with no CNI or Service in the path, so a Service/port/selector mistake is invisible; (3) the report ConfigMap surviving its parent's deletion — envtest runs no garbage collector, so cascade deletion is never exercised at all. Keep that list: it is the concrete, memorable answer to "what can't your test suite prove."

## Practice

Build the **full module deliverable** — the complete spec is at [`../practice/gpu-cost-operator/README.md`](../practice/gpu-cost-operator/README.md). Work it in this order; each step is verifiable before the next.

1. **Scaffold.** Run the `init` / `create api` ×3 / `create webhook` sequence from §2 verbatim. Commit the untouched scaffold as its own commit — it is the baseline that makes every later diff readable, and it is how you prove which files you own versus which the tool generated.
2. **Types.** Write the three `*_types.go` files with markers: `/status` subresources, `printcolumn`s, `listType=map` on conditions, `scope=Cluster` on `GPUCostPolicy`, and at least **two CEL rules** — a non-negative `limitUSD` and one immutability rule using `oldSelf`. Run `make manifests` and read the emitted `x-kubernetes-validations` block to confirm the rule landed.
3. **Reconcilers.** Implement `WorkloadCostReconciler` and `BudgetReconciler` with the finalizer, the owned report ConfigMap via `CreateOrUpdate` + `SetControllerReference`, `observedGeneration`, and the semantic-equality guard around the status write. Inject the metering source behind an interface so tests can supply a fixture.
4. **RBAC.** Replace every scaffolded marker with per-resource markers carrying only the verbs your code uses, including `workloadcosts/finalizers` (needed by `SetControllerReference`) and the webhook's `budgets` read. Regenerate; assert `grep -c '"\*"' config/rbac/role.yaml` returns `0`.
5. **Webhook.** Implement the budget-breach validator with the typed `ValidateCreate`/`ValidateUpdate` signatures. Choose a `failurePolicy` and write the reasoning down in `docs/webhook-failure-policy.md`. Add the `namespaceSelector` excluding `kube-system` and your own namespace.
6. **envtest suites.** Write `internal/controller/suite_test.go` and `internal/webhook/v1alpha1/webhook_suite_test.go`, including the TLS-readiness `Eventually` loop. Verify `make setup-envtest` puts binaries in `bin/k8s/<version>-<os>-<arch>/`.
7. **The three proofs.** (a) An **idempotency** spec asserting `resourceVersion` stability, semantic status equality, unchanged child `resourceVersion`, and no duplicate children. (b) A **level-triggering** `DescribeTable` with at least four rows: drifted status, deleted child, drifted child content, missing finalizer. (c) A **webhook** pair: one create denied with `apierrors.IsInvalid`, one create admitted — plus a CEL spec proving the schema rejects a negative budget with the operator not running.
8. **Break it on purpose.** Make the status write unconditional and confirm the idempotency spec fails with a `resourceVersion` mismatch. Revert. Paste the failure output into your lab log — it is the evidence that the assertion has teeth.
9. **Deploy and audit.** `make deploy` to a kind cluster, then capture `kubectl auth can-i --list` for the operator's ServiceAccount and paste it into `RBAC-JUSTIFICATION.md` with one line per grant naming the call site that needs it.
10. **Name the gaps.** Write three bullets: one bug class per envtest limitation (no kubelet, no CNI/Service, no GC), each with the cheapest real-cluster test that would catch it.

**Acceptance = the module checkpoint** ([`../checkpoint.md`](../checkpoint.md)): `make manifests generate` is reproducible (clean `git status` on re-run), `make test` is green with idempotency, level-triggering, and webhook admission all demonstrated by named passing specs, the generated RBAC has zero wildcard verbs and a justification line per grant, and you can demo the operator and defend the test strategy — including naming what the suite structurally cannot prove.

## Common pitfalls

1. **Treating a green `make test` as "production ready."** It proves your controller's contract with the API and nothing about the rest of the system. Say the boundary out loud in a demo before someone asks — "this suite proves reconcile semantics, admission and schema validation; cascade deletion, Pod startup and Service routing are covered by the kind suite" — because being asked and not having an answer reads much worse than volunteering it.
2. **Asserting idempotency as "no error returned."** A reconcile that unconditionally rewrites identical status passes that assertion and hot-loops forever in production. Assert `resourceVersion` equality across a repeat reconcile, plus semantic status equality, plus an unchanged child. The mechanism: every write is a watch event, every event re-enqueues the object, and the loop is bounded only by the workqueue's rate limiter.
3. **Testing cascade deletion under envtest.** There is no garbage collector, so deleting a parent leaves children behind and your assertion fails for a reason that is not your bug. Assert the `ownerReferences` entry — `Controller: true`, `BlockOwnerDeletion: true`, correct `Kind` and `UID` — and put the cascade assertion in the kind-based e2e suite.
4. **Skipping the webhook test because "the fake client is faster."** The fake client runs no admission at all, so skipping envtest here leaves the parts most likely to break in a rollout — cert paths, port, `failurePolicy`, `namespaceSelector`, the AdmissionReview round trip — with **zero** coverage.
5. **Omitting the TLS-readiness wait in the webhook suite.** `mgr.Start()` returns before the webhook server's listener is accepting, so the first spec races it and fails with `connection refused` intermittently. The `Eventually(tls.DialWithDialer)` loop is not boilerplate to trim; it is the fix for a flake that will otherwise cost you an afternoon.
6. **Letting the envtest Kubernetes version drift from `go.mod`.** CI and a laptop testing against different apiserver versions is a hard-to-diagnose flake source. The v4.12 scaffold derives `ENVTEST_K8S_VERSION` from `k8s.io/api` and `ENVTEST_VERSION` from `sigs.k8s.io/controller-runtime`; do not replace that with a hardcoded literal that will rot.
7. **Adding the finalizer and doing real work in the same reconcile pass.** After `r.Update()` writes metadata, your in-memory object's `resourceVersion` is stale and any later `Update` in the same pass returns `409`. Requeue after the metadata write and let the next pass do the work (lesson 06 §7).
8. **`failurePolicy: Fail` without a `namespaceSelector` excluding your own namespace.** If the webhook governs a resource your own deployment creates and the webhook pod is down, you cannot deploy the fix. Exclude `kube-system` and the operator's namespace unconditionally, and choose `Fail` only for invariants where admitting an unvalidated object is worse than an outage.
9. **Hand-editing a generated file.** `config/rbac/role.yaml`, `config/crd/bases/*.yaml`, `config/webhook/manifests.yaml` and `zz_generated.deepcopy.go` are overwritten by the next `make manifests generate`. If you need a rule, you need a marker. Enforce it with a CI step that regenerates and fails on a dirty tree.
10. **Sharing one namespace across test rows.** Objects leak between specs and you get order-dependent failures that pass in isolation. Create a fresh namespace per row; under envtest they cost microseconds and, since the namespace controller is absent, you do not even have to wait for deletion to complete.

## Self-check

**Q1. What does envtest provide and what does it not — and given no kubelet, what can you not test?**
**Answer:** envtest starts a **real `kube-apiserver` and `etcd`** as local processes (provisioned by `setup-envtest`, path exported as `KUBEBUILDER_ASSETS`). Real, therefore: CRD installation and structural schema validation, CEL `x-kubernetes-validations`, validating and mutating webhooks over TLS via `WebhookInstallOptions`, the `/status` subresource distinction, optimistic concurrency and 409 conflicts, watches and informer caches, and the apiserver's finalizer rules (`deletionTimestamp`, `202 Accepted`, `Terminating`). Absent: **no kubelet, no scheduler, no kube-controller-manager, no CNI, no container runtime.** Therefore Pods never run (they stay `Pending` with no `nodeName`), a Deployment produces no ReplicaSets or Pods, **owner-reference garbage collection never happens**, and namespace deletion never completes. You cannot test that a container started, a probe passed, a workload landed on a GPU node, or that a child was cascade-deleted. What you *can* test in place of the cascade is that the child carries a correct `ownerReferences` entry with `controller: true` and `blockOwnerDeletion: true`. Also note the test client is admin-equivalent: RBAC objects exist but do not restrict you unless you build an impersonating client.

**Q2. How do you assert reconcile idempotency, concretely?**
**Answer:** Drive to steady state, then reconcile **again on the same observed state and assert nothing changed**. Four assertions, each catching a different bug: (1) the object's **`resourceVersion` is unchanged** — under a real apiserver a no-op does no write, so an unconditional `Status().Update()` shows up immediately; (2) **`equality.Semantic.DeepEqual(before.Status, after.Status)`** — catches churn a `resourceVersion` check alone would not explain, most often a hand-rolled condition stamping `lastTransitionTime` every pass (use `meta.SetStatusCondition`, which only moves it when `Status` changes); (3) the **owned child's `resourceVersion` is unchanged** — catches a `CreateOrUpdate` mutate function that is not a pure function of desired state; (4) **no duplicate children** — catches a reconciler generating a fresh child name each pass. Also assert the return: `Result{RequeueAfter: <steady value>}` and `nil`, with no `Requeue: true`. The bug this catches is not cosmetic: every write is a watch event, every event re-enqueues the object, and the controller spins forever at the rate limiter's ceiling.

**Q3. How do you assert level-triggering, and why is the test shaped that way?**
**Answer:** Seed a state the controller **never witnessed coming into being**, reconcile exactly once, and assert convergence. Rows: status drifted to a wrong value by a human edit; the owned child deleted; the child's content edited; the finalizer stripped; the referenced `GPUCostPolicy` deleted (expect `Ready=False` with reason `PolicyNotFound`, and no infinite retry because the reconciler returns `reconcile.TerminalError`). The shape matters because **no row may depend on a prior transition** — that is exactly the property that distinguishes a level-triggered controller from an edge-triggered one, which would only act if it saw the change happen. An edge-triggered design passes a sequential test and fails after a controller restart, a missed watch event, or a cache resync, because none of those replay the transition.

**Q4. Fake client vs envtest — when each, and why is it a cost decision?**
**Answer:** **Fake client** (`pkg/client/fake`, in-memory tracker, no server) for fast, focused unit tests of reconcile *logic*: given a seeded object store, does one `Reconcile` take the right branch and produce the right writes? It runs **no admission and no webhooks ever**, only approximates the `/status` subresource (you must opt in with `WithStatusSubresource`), does not apply structural or CEL validation, and approximates `resourceVersion` and conflict semantics. **envtest** (real apiserver + etcd) for anything exercising real API semantics: webhook accept/reject, CEL and structural validation, status subresource behaviour, 409 conflicts, the finalizer/deletion flow, field indexes. The cost dimension: an envtest suite spends seconds on process startup and API round-trips versus microseconds for in-memory tests — in the worked example, 13.0 s of a 13.4 s run was envtest. Across a fleet-scale CI matrix that is minutes of pipeline latency per PR, so "fake client for the bulk, envtest where API semantics actually matter" is a budget decision, not only a fidelity one.

**Q5. Name one bug class your envtest suite structurally cannot catch, and the cheapest real-cluster test that would.**
**Answer:** A GPU device plugin failing to inject `/dev/nvidia*` into a container is structurally uncatchable: envtest runs no kubelet and no container runtime, so there is no container for the device to be injected into. Cheapest catch: a `kind` cluster (or a small staging cluster) running the actual device-plugin DaemonSet plus one Pod requesting a GPU resource, asserting `nvidia-smi` succeeds inside the running container — a minutes-long smoke test, not a full e2e suite. Two more in the same class: a webhook `Service` with the wrong port or selector (envtest calls the handler directly on localhost, with no Service and no CNI in the path), and an owned child surviving its parent's deletion (envtest runs no garbage collector, so the cascade is never exercised).

**Q6. `failurePolicy: Fail` and your webhook pod is down — what happens, and how do you avoid self-lockout?**
**Answer:** Every `CREATE`/`UPDATE` matching the webhook's rules is **rejected** by the apiserver, because it cannot obtain the required admission decision. If the webhook's rules match a resource your own deployment needs to create, or if its `namespaceSelector` does not exclude the operator's namespace, you cannot deploy the fix — the failure is self-sustaining. Defences, in order: (1) always set a `namespaceSelector` excluding `kube-system`, `kube-node-lease` and the operator's own namespace, using the automatic `kubernetes.io/metadata.name` label; (2) scope `rules` as narrowly as possible — specific `apiGroups`, `resources`, `operations`, and `scope: Namespaced`; (3) keep `timeoutSeconds` low (the default is 10 s; 5 s is plenty for a cache-backed handler) so a slow handler degrades rather than stalls; (4) choose `Ignore` whenever a violation is recoverable after admission, which for a **budget policy** it is — you can reconcile and report an over-budget workload, but you cannot fix a cluster whose workload creation is broken. Reserve `Fail` for security or correctness invariants where admitting an unvalidated object is genuinely worse than the outage.

**Q7. "Your controller stopped reconciling." What is your diagnosis order?**
**Answer:** (1) **Backoff / `Forget`** — is the key climbing the exponential-backoff ladder from a persistent error, so it is retrying slowly rather than stopped? Check `workqueue_retries_total` and the controller's error logs (lesson 03). (2) **Stale or unstarted cache** — did `WaitForCacheSync` return true at startup, and is the informer's watch still alive, or did it stop relisting silently (lesson 04)? (3) **Missing watch or owner reference** — does the controller actually `For`/`Owns`/`Watches` the type that changed, and are your predicates filtering the event out? `GenerationChangedPredicate` in particular drops label and annotation edits, because metadata never bumps `generation` (lesson 06 §5). (4) **A hung reconcile holding its workqueue item** — with no `ReconciliationTimeout`, one blocked HTTP call means `Done` is never called and that object stops reconciling for the life of the process; `workqueue_longest_running_processor_seconds` climbing monotonically is the signature. (5) **RBAC** — a `list`/`watch`/`get` grant removed or never generated produces a `Forbidden` that envtest does not catch, because the envtest client is admin-equivalent (lesson 07). Working rate-limiting → cache → watch wiring → stuck item → permissions before touching business logic is the fast path, and it is the order the module taught them in.

## Connections & what's next

This lesson closes the arc the whole module has been building: component internals (01) explained *what* watches and reconciles; API machinery (02) gave you the typed and dynamic client tools; the reconciliation model (03) established idempotency and level-triggering as design law; informers, caches and workqueues (04) explained *why* those properties are non-negotiable given how the queue behaves; CRD design (05) shaped the API surface and the `/status` split; [controller-runtime and finalizers/GC (06)](06-controller-runtime-deep.md) wired the manager, the builder, owner references and the finalizer dance; [kubebuilder and RBAC (07)](07-kubebuilder-and-rbac.md) turned markers into a defensible permission set; admission webhooks (08) added a synchronous gate at the API boundary; and the scheduler/DRA/Kueue lesson (09) mapped where placement decisions live around all of it. Here, all of it becomes one binary, proven by tests you can point at.

This also **closes Module 02**. With `gpu-cost-operator` v0.1 green under envtest and the checkpoint's depth probes answerable cold, you are equipped for the two modules this one **unlocks**: **[Module 04 — GPU on Kubernetes](../../04-gpu-on-kubernetes/README.md)**, where the operator's `GPUMeter` stops being a stub and starts doing real per-Pod GPU attribution against live nodes instead of a static price table, and **[Module 06 — Scheduling, queueing and capacity](../../06-scheduling-capacity/README.md)**, where lesson 09's Kueue literacy becomes a hands-on showback deliverable with real quota, fairness and preemption design. The operator you just built is the artifact both modules extend — keep it running and keep it green.

## References & further reading

**Primary sources**

- [Kubebuilder Book — Quick Start and the tutorial chapters](https://book.kubebuilder.io/quick-start) — the canonical `init` / `create api` / `create webhook` sequence and what each command scaffolds, matching §2.
- [Kubebuilder Book — testing with envtest](https://book.kubebuilder.io/reference/envtest.html) — the canonical `envtest.Environment` setup, `WebhookInstallOptions`, and the fidelity caveats reproduced in §9.
- [Kubebuilder Book — project configuration (`PROJECT`)](https://book.kubebuilder.io/reference/project-config) — every field of the inventory file, including `core: true` and `namespaced: false`.
- [Kubebuilder Book — CRD validation markers](https://book.kubebuilder.io/reference/markers/crd-validation.html) — the marker vocabulary used in §3, including `XValidation` and its `rule`/`message`/`reason`/`fieldPath` arguments.
- [`kubernetes-sigs/kubebuilder` — `testdata/project-v4`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/testdata/project-v4) — the reference scaffold this lesson's file layout, `Makefile` recipes, `suite_test.go`, `webhook_suite_test.go` and typed webhook code were read from. When a tutorial and this directory disagree, this directory is right.
- [controller-runtime `envtest` godoc](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/envtest) — `Environment`, `CRDDirectoryPaths`, `ErrorIfCRDPathMissing`, `BinaryAssetsDirectory`, `WebhookInstallOptions` (`LocalServingHost`/`Port`/`CertDir`), and `ControlPlaneStartTimeout`.
- [`setup-envtest`](https://github.com/kubernetes-sigs/controller-runtime/tree/main/tools/setup-envtest) — provisioning the apiserver/etcd/kubectl binaries, `--bin-dir`, `-p path`, and `KUBEBUILDER_ASSETS`; the source of the `bin/k8s/<ver>-<os>-<arch>` layout in §9.
- [controller-runtime fake client godoc](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client/fake) — what it approximates and what it does not, including `WithStatusSubresource`; the basis for §14's comparison table.
- [Kubernetes — Validation rules (CEL)](https://kubernetes.io/docs/reference/using-api/cel/) — the CEL feature/version matrix, including the **quantity library (Kubernetes 1.29+)** whose `isQuantity()`/`quantity().sign()` this lesson uses for the non-negative-budget rule, and transition rules with `oldSelf`.
- [Kubernetes — Dynamic admission control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) — `failurePolicy`, `sideEffects`, `timeoutSeconds` (default 10 s), `namespaceSelector`, `matchPolicy`, and the ordering of schema validation relative to webhooks.
- [`kubernetes-sigs/controller-tools`](https://github.com/kubernetes-sigs/controller-tools) — the generator behind `make manifests`. `controller-gen crd -w` and `controller-gen rbac -w` print every marker option straight from the source.

**Real-world engineering blogs**

- [SuperOrbital — "Testing Production Kubernetes Controllers"](https://superorbital.io/blog/testing-production-controllers/) — what it shows: envtest plus table-driven testing as industry-standard practice rather than an idiosyncratic choice.
- [InfraCloud — "Testing Kubernetes Operators using EnvTest"](https://www.infracloud.io/blogs/testing-kubernetes-operator-envtest/) — what it shows: an independent statement of the "no kubelet, no controller-manager" fidelity boundary, in the same terms as §9.
- [Marc Nuri — "How to Test Kubernetes Applications in Go with EnvTest"](https://blog.marcnuri.com/go-testing-kubernetes-applications-envtest) — what it shows: a named practitioner's concrete suite-structure walkthrough, useful as a second reading alongside the Kubebuilder Book.

**Deeper dives**

- [`../checkpoint.md`](../checkpoint.md) — the module's completion gate; every depth probe this lesson's self-check maps to.
- [`../practice/gpu-cost-operator/README.md`](../practice/gpu-cost-operator/README.md) — the full deliverable spec: CRD table, scope guardrails, suggested layout, and the acceptance criteria this lesson builds toward.
