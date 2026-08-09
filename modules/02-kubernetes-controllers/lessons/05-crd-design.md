---
lesson: "02.5"
title: "CRD and API design"
module: "02"
concept: "CRD and API design"
status: not-started
est_time: "10h"
artifacts: []
---

# 02.5 · CRD and API design

> **Concept.** A CRD is a versioned API contract, not a config blob — schema, validation, subresources, and versioning decide whether your operator ages well.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Why this matters

Your operator's CRDs are a public API the moment someone `kubectl apply`s them. Every field is a promise you have to keep across upgrades. Get the spec/status split wrong and you fight the API server on every reconcile. Skip validation and garbage lands in etcd, then your controller panics at 3am reconciling a `WorkloadCost` with a negative rate. Ship a v1alpha1 with no versioning plan and your first breaking change means telling 40 clusters to hand-edit CRs.

For a GPU cost operator this is sharper than usual: money is involved. A `Budget` that accepts a negative limit, or a `GPUCostPolicy` with a typo'd currency, produces silently wrong spend numbers that people make purchasing decisions on. The API layer is your first and cheapest line of defense — validation rejected at admission never reaches your reconcile loop, never corrupts status, never needs a postmortem. Senior work here is designing the schema so that the invalid states you'd otherwise handle defensively in Go are simply unrepresentable.

## From operating to extending

**CKA already knows:** what a CRD is, `kubectl apply -f crd.yaml`, that CRs show up in `kubectl get`, that operators define their own types. You've consumed CRDs from cert-manager, Prometheus, Argo. You know `kubectl explain` and `--v=8` to see the API traffic.

**Internal to learn now:** how the `CustomResourceDefinition` object itself is structured — `spec.versions[]`, `schema.openAPIV3Schema`, `subresources`, `additionalPrinterColumns`, `conversion`. How the API server uses that schema as a structural-schema gate (pruning unknown fields, enforcing types) *before* your controller sees anything. How kubebuilder markers compile down to that OpenAPI schema. Where CEL (`x-kubernetes-validations`) runs and what it can express that OpenAPI can't. Why `/status` is a *subresource* and how that changes the write path and `metadata.generation`. When multiple versions force a conversion webhook versus when defaulting suffices. And server-side apply field ownership, because your controller and users will both write to the same objects.

## Core notes

**Structural schema is mandatory and it prunes.** Every CRD served needs a structural OpenAPI v3 schema. The API server *prunes* any field not in the schema before persisting — so an undeclared field silently vanishes, it does not error. This is why you never rely on "extra" annotations sneaking through spec. Declare everything.

**Spec vs status is a design decision, not a formality.** `spec` is desired state, written by users/clients. `status` is observed state, written only by your controller. The split is what makes the `/status` subresource meaningful:

```go
// api/v1alpha1/workloadcost_types.go
type WorkloadCostSpec struct {
    // WorkloadRef identifies the workload this cost is computed for.
    // +kubebuilder:validation:Required
    WorkloadRef corev1.ObjectReference `json:"workloadRef"`
    // PolicyRef selects the GPUCostPolicy pricing model to apply.
    // +kubebuilder:validation:Required
    PolicyRef string `json:"policyRef"`
}

type WorkloadCostStatus struct {
    // ObservedGPUHours is the accumulated GPU-hours the controller has metered.
    // +optional
    ObservedGPUHours resource.Quantity `json:"observedGPUHours,omitempty"`
    // ComputedSpend is observedGPUHours * policy rate, in the policy currency.
    // +optional
    ComputedSpend string `json:"computedSpend,omitempty"`
    // ObservedGeneration is the .metadata.generation the status reflects.
    // +optional
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`
    // Conditions follow the standard metav1.Condition contract.
    // +listType=map
    // +listMapKey=type
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}
```

**Kubebuilder validation markers → OpenAPI.** These compile into the CRD schema and are enforced at admission by the API server, zero controller code:

```go
// +kubebuilder:validation:Enum=USD;EUR;GBP
Currency string `json:"currency"`

// +kubebuilder:validation:Minimum=0
// +kubebuilder:validation:Maximum=100000
RatePerGPUHour resource.Quantity `json:"ratePerGPUHour"`

// +kubebuilder:validation:Pattern=`^[0-9]+(h|d)$`
Window string `json:"window"`
```

Markers cover single-field constraints: enum, min/max, min/maxLength, pattern, required, format. They cannot see other fields.

**CEL (`x-kubernetes-validations`) does cross-field and transitions.** This is the leap OpenAPI can't make. CEL rules are evaluated at admission (GA since 1.29) with `self` bound to the current object/field and, in transition rules, `oldSelf` bound to the prior value:

```go
// On the type (validates across fields of Budget spec):
// +kubebuilder:validation:XValidation:rule="self.limit >= 0",message="budget limit must be non-negative"
// +kubebuilder:validation:XValidation:rule="self.hardCap >= self.softCap",message="hardCap must be >= softCap"
// Immutability via a transition rule (self compared to oldSelf):
// +kubebuilder:validation:XValidation:rule="self == oldSelf",message="scope is immutable"
```

What CEL validates that OpenAPI cannot: relationships between fields (`hardCap >= softCap`), conditional requirements (`has(self.window) || self.scope == 'cluster'`), immutability after creation (transition rules), list uniqueness by a key, and quantity comparisons. Each rule carries an estimated cost; the API server enforces a per-rule and per-CRD cost budget and rejects the CRD at *create* time if a rule could be too expensive on large inputs — so avoid unbounded loops over large lists. `messageExpression` lets the message itself be a CEL string, and `fieldPath` points the error at a subfield.

**Subresources change the write path.** Declaring `/status` (kubebuilder does this with `// +kubebuilder:subresource:status`) means:
- Writes to the *main* resource ignore changes to `.status` — a client PUT to the object cannot mutate status. Only `Status().Update()`/`Status().Patch()` (a write to `.../status`) touches it.
- Writes to `/status` ignore changes to `.spec`.
- `.metadata.generation` increments **only when spec changes**, never on a status write. This is the whole point of the `observedGeneration` pattern: compare `status.observedGeneration` to `metadata.generation` to know if you've reconciled the latest spec. Without the subresource, every status write would bump generation and the signal would be useless.

`/scale` is the other standard subresource — it exposes `spec.replicas`/`status.replicas` so `kubectl scale` and the HPA can target your CRD. The GPU cost CRDs don't scale, so you won't declare it, but know it exists.

**Printer columns make `kubectl get` useful.** JSONPath into your object:

```go
// +kubebuilder:printcolumn:name="Spend",type=string,JSONPath=`.status.computedSpend`
// +kubebuilder:printcolumn:name="GPU-Hours",type=string,JSONPath=`.status.observedGPUHours`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`
```

**Conditions: use the standard contract.** Use `[]metav1.Condition` with `listType=map`/`listMapKey=type`, and `meta.SetStatusCondition`/`FindStatusCondition` to manage them. Pick a small set of orthogonal `type`s (e.g. `Ready`, `BudgetExceeded`, `PolicyResolved`) with `reason` in PascalCase. Conditions are for humans and other controllers to read; don't overload them as your control state.

**Defaulting.** Simple constants use `// +kubebuilder:default=USD`, written into the schema and applied by the API server on admission. Dynamic or conditional defaults (default `window` based on `scope`) need a mutating admission (defaulting) webhook — the `Defaulter` interface in controller-runtime. Prefer schema defaults; reach for a webhook only when the default depends on other fields or external state.

**Multi-version CRDs and the storage version.** A CRD can serve several versions (`v1alpha1`, `v1beta1`), but exactly **one** has `storage: true` — that's the version etcd persists. All other served versions are converted to/from the storage version on the fly. Two conversion strategies:

- `conversion.strategy: None` — versions must be structurally identical (only additive, field-compatible changes). The API server does a trivial passthrough. Adding an optional field with a default fits here.
- `conversion.strategy: Webhook` — you run a conversion webhook that translates objects between versions. **Required** when a field is renamed, restructured, split, or given different semantics — anything a passthrough can't reconcile. The webhook must convert both directions and round-trip losslessly through the storage version, or clients on the old version see corruption.

Rule of thumb: **new optional field with a default → new version + `None` (or even same version).** Renamed/reshaped field, changed units, merged fields → **conversion webhook.** For a first-cut operator you ship a single `v1alpha1` as storage version and defer webhooks; but design fields now so your likely evolution stays additive.

**Server-side apply and field ownership.** SSA (`kubectl apply --server-side`, or controller-runtime `client.Apply` patches with a `FieldOwner`) tracks *which manager owns which field* in `metadata.managedFields`. Two managers editing disjoint fields coexist; two claiming the same field conflict (resolvable with force). This matters because your controller writes `status` while users write `spec`, and if you ever apply into `spec` (e.g. defaulting a field) you must set a stable `FieldOwner` so you don't fight the user's own `kubectl apply` on the next round. Reconcile writes should be scoped to the fields you own.

## Worked example

Design the `Budget` CRD end to end, showing the marker → schema mapping.

```go
// api/v1alpha1/budget_types.go
package v1alpha1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/api/resource"
)

// +kubebuilder:validation:XValidation:rule="self.hardLimit >= self.softLimit",message="hardLimit must be >= softLimit"
type BudgetSpec struct {
    // HardLimit is the enforced ceiling for spend in the window.
    // +kubebuilder:validation:XValidation:rule="self >= 0",message="hardLimit must be non-negative"
    HardLimit resource.Quantity `json:"hardLimit"`

    // SoftLimit triggers a warning condition before the hard cap.
    // +kubebuilder:validation:XValidation:rule="self >= 0",message="softLimit must be non-negative"
    SoftLimit resource.Quantity `json:"softLimit"`

    // Currency of the limits; must match the referenced policy's currency.
    // +kubebuilder:validation:Enum=USD;EUR;GBP
    // +kubebuilder:default=USD
    Currency string `json:"currency"`

    // Window is a rolling accounting period, e.g. "24h" or "30d".
    // +kubebuilder:validation:Pattern=`^[0-9]+(h|d)$`
    Window string `json:"window"`

    // Scope selects which workloads count against this budget.
    // Immutable after creation so historical spend stays attributable.
    // +kubebuilder:validation:XValidation:rule="self == oldSelf",message="scope is immutable"
    Scope metav1.LabelSelector `json:"scope"`
}

type BudgetStatus struct {
    // +optional
    CurrentSpend string `json:"currentSpend,omitempty"`
    // +optional
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`
    // +listType=map
    // +listMapKey=type
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Limit",type=string,JSONPath=`.spec.hardLimit`
// +kubebuilder:printcolumn:name="Spend",type=string,JSONPath=`.status.currentSpend`
// +kubebuilder:printcolumn:name="Window",type=string,JSONPath=`.spec.window`
type Budget struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec   BudgetSpec   `json:"spec,omitempty"`
    Status BudgetStatus `json:"status,omitempty"`
}
```

`make manifests` runs `controller-gen` and emits `config/crd/bases/cost.example.com_budgets.yaml` where each marker has become schema:

```yaml
# excerpt of the generated CRD
spec:
  hardLimit:
    x-kubernetes-int-or-string: true
    x-kubernetes-validations:
    - rule: self >= 0
      message: hardLimit must be non-negative
  currency:
    type: string
    enum: [USD, EUR, GBP]
    default: USD
x-kubernetes-validations:
- rule: self.hardLimit >= self.softLimit
  message: hardLimit must be >= softLimit
```

Now the acceptance behavior: a `Budget` with `hardLimit: -5` is rejected by the API server with `hardLimit must be non-negative` — before your controller ever runs. A `Budget` with `hardLimit: 100, softLimit: 200` is rejected by the type-level rule. A later `kubectl edit` changing `scope` is rejected by the transition rule. None of this is Go you wrote or tested; it's the schema doing its job.

## Practice

Advance `gpu-cost-operator`: design and scaffold all three v1alpha1 CRDs.

1. Scaffold the group and types (kubebuilder): `kubebuilder create api --group cost --version v1alpha1 --kind GPUCostPolicy` (repeat for `WorkloadCost`, `Budget`), generating both resource and controller.
2. **`GPUCostPolicy`** — a price model. `spec.rates[]` mapping GPU class (`a100`, `h100`, ...) to `$-per-GPU-hour` (`resource.Quantity`), plus `spec.currency` (enum). CEL: every rate `>= 0`; currency in the enum. `/status` subresource with a `Ready` condition and `observedGeneration`.
3. **`WorkloadCost`** — status-heavy. `spec.workloadRef` + `spec.policyRef`. Status carries `observedGPUHours`, `computedSpend`, `observedGeneration`, and conditions. `/status` subresource. Printer columns for spend and GPU-hours.
4. **`Budget`** — as in the worked example: hard/soft limits, currency enum, window pattern, immutable scope selector, `hardLimit >= softLimit` cross-field CEL, non-negative limits.
5. Run `make manifests generate` and `kubectl apply -f config/crd/bases/`.

**Acceptance:**
- `kubectl apply` of a valid instance of each kind **succeeds**.
- `kubectl apply` of a `Budget` with `hardLimit: -1` **fails** with the CEL message (proves admission-time rejection, not controller-time).
- `kubectl apply` of a `Budget` with `softLimit > hardLimit` **fails** on the type-level rule.
- `kubectl explain budget.spec` shows your fields and constraints; `kubectl get budget` shows your printer columns.
- Editing a `Budget`'s `scope` after creation is rejected as immutable.

## Self-check

**(a) Why is `/status` a subresource, and what does it change about updates and `observedGeneration`?**

**Answer:** Making status a subresource splits the write path: writes to the main object can't change `.status`, and writes to `/status` can't change `.spec`. Two consequences. First, RBAC and code separate desired state (users/clients write spec) from observed state (only the controller, via `Status().Update()`/`Status().Patch()`, writes status) — a client can't accidentally clobber controller-computed fields. Second, `.metadata.generation` increments **only on spec changes**, not on status writes. That makes `status.observedGeneration` a reliable "have I reconciled the current spec?" signal: when `observedGeneration == metadata.generation` you're up to date. Without the subresource, every status write would bump generation and destroy that signal, and you'd risk write conflicts between spec and status updates racing on the same object.

**(b) When do you need a conversion webhook versus a new version with defaulting?**

**Answer:** A new version with defaulting (conversion strategy `None`) is enough when the change is *additive and passthrough-compatible*: adding an optional field, giving it a schema default, tightening validation that old objects already satisfy. The API server converts between versions by trivial passthrough because the shapes are compatible. You need a **conversion webhook** (strategy `Webhook`) when a field is renamed, restructured, split/merged, or changes units or semantics — anything where converting an old object to the new shape (and back to the storage version) requires actual logic, not a passthrough. The webhook must convert in both directions and round-trip losslessly through the single storage version; otherwise clients pinned to the old version see corrupted data. Heuristic: can I express the migration purely as "new optional field + default"? If yes, no webhook. If it requires reading old fields to compute new ones, webhook.

**(c) What can CEL validate that plain OpenAPI cannot — give an example?**

**Answer:** OpenAPI markers constrain one field in isolation (enum, min/max, pattern, required). CEL (`x-kubernetes-validations`) sees the whole object via `self` and the prior value via `oldSelf`, so it can express **cross-field** and **transition** constraints. Example from `Budget`: `self.hardLimit >= self.softLimit` — a relationship between two fields that no single-field OpenAPI constraint can capture. Another: `self == oldSelf` on `scope` enforces **immutability after creation**, which OpenAPI has no concept of. CEL can also do conditional requirements (`has(self.window) || self.scope == 'cluster'`) and list uniqueness by key. It runs at admission with a cost budget, so it stays cheap and rejects bad objects before they reach etcd or your controller.

## Resources

1. **Kubebuilder Book — CRD validation & webhooks** — https://book.kubebuilder.io/reference/markers/crd-validation.html — the authoritative list of `+kubebuilder:validation` markers and how they map to schema, plus the multi-version/conversion and webhook chapters. **Deep read** the validation markers page while writing your types; **skim** the conversion webhook chapter for the mental model. Why: this is exactly the marker → CRD compilation your `make manifests` performs.
2. **kubernetes.io — CRD reference & CEL validation** — https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/ and https://kubernetes.io/docs/reference/using-api/cel/ — the API-server view: structural schemas, subresources, versions/storage version, and the CEL variables (`self`, `oldSelf`), cost budget, `messageExpression`, and transition rules. **Deep read** the CEL page (it's the highest-leverage validation tool you have); **skim** the CRD task page for subresource/versioning specifics. Why: markers are sugar over this; when a rule is rejected at CRD-create time for cost, this is where the rules live.
3. **apiextensions-apiserver types godoc** — https://pkg.go.dev/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1 — the Go structs behind a `CustomResourceDefinition` (`CustomResourceDefinitionVersion`, `CustomResourceValidation`, `CustomResourceSubresources`, `ValidationRule`). **Skim** once to see the ground truth your YAML deserializes into. Why: demystifies what "storage version" and "x-kubernetes-validations" actually are as first-class API objects.
