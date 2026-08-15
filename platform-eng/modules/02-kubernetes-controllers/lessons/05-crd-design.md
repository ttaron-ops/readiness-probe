---
lesson: "02.5"
title: "CRD and API design"
module: "02"
concept: "CRD and API design"
status: not-started
est_time: "20h"
prev: "04-informers-caches-workqueues.md"
next: "06-controller-runtime-deep.md"
artifacts: []
sources: 10
---

# 02.5 · CRD and API design

> **Concept.** A CRD is a versioned API contract, not a config blob — schema, validation, subresources, and versioning decide whether your operator ages well.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Lesson 04 gave you the machine every controller runs on: reflector → Delta FIFO → informer → lister → workqueue → worker. That machine assumes something already exists to watch — a type registered in the API server with a schema, versions, and a write contract. This lesson is where that type comes from. Before you can reconcile a `WorkloadCost`, something has to define what a `WorkloadCost` *is*: which fields are legal, which are computed, what happens when the shape changes next year. Get this lesson right and lesson 06's controller-runtime machinery has a clean, honest object to reconcile against. Get it wrong and every reconcile is compensating in Go for a schema that should have rejected the bad input at the door.

## Why this matters

Your operator's CRDs are a public API the moment someone `kubectl apply`s them. Every field is a promise you have to keep across upgrades. Get the spec/status split wrong and you fight the API server on every reconcile. Skip validation and garbage lands in etcd, then your controller panics at 3am reconciling a `WorkloadCost` with a negative rate. Ship a v1alpha1 with no versioning plan and your first breaking change means telling 40 clusters to hand-edit CRs.

For a GPU cost operator this is sharper than usual: money is involved. A `Budget` that accepts a negative limit, or a `GPUCostPolicy` with a typo'd currency, produces silently wrong spend numbers that people make purchasing decisions on. The API layer is your first and cheapest line of defense — validation rejected at admission never reaches your reconcile loop, never corrupts status, never needs a postmortem. This is also directly the job: NVIDIA and CoreWeave postings for controller/operator roles both name CRD design as core work, not an implementation detail delegated to someone else. Senior work here is designing the schema so that the invalid states you'd otherwise handle defensively in Go are simply unrepresentable.

## What's new here (calibration)

**CKA already knows:** what a CRD is, `kubectl apply -f crd.yaml`, that CRs show up in `kubectl get`, that operators define their own types. You've consumed CRDs from cert-manager, Prometheus, Argo. You know `kubectl explain` and `--v=8` to see the API traffic.

**Internal to learn now:** how the `CustomResourceDefinition` object itself is structured — `spec.versions[]`, `schema.openAPIV3Schema`, `subresources`, `additionalPrinterColumns`, `conversion`. How the API server uses that schema as a structural-schema gate (pruning unknown fields, enforcing types) *before* your controller sees anything. How kubebuilder markers compile down to that OpenAPI schema. Where CEL (`x-kubernetes-validations`) runs and what it can express that OpenAPI can't. Why `/status` is a *subresource* and how that changes the write path and `metadata.generation`. When multiple versions force a conversion webhook versus when defaulting suffices. And server-side apply field ownership, because your controller and users will both write to the same objects.

## Core concepts

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

**A CRD instance is still an etcd value — it has a size limit.** The API server enforces a default per-object size limit (~1.5MiB) regardless of type, because it's still one etcd value under the Raft commit path. This is easy to forget on a status-heavy CRD: an unbounded `status.history[]` that appends one entry per reconcile is a realistic way for a long-lived `WorkloadCost` to hit the limit months into production, at which point every further status write starts failing. Cap history length in the type itself (ring buffer semantics, or push detail to a metrics backend and keep only a summary in status) rather than discovering the limit in an incident.

**Storage-version rollback is an operational trap, not just a design nicety.** Once objects have actually been *written* under a new storage version, rolling the operator back to a build that only understands the old storage version can fail to read them — the old binary's scheme doesn't know how to decode the new stored shape. On a fleet doing staged rollouts across 40 clusters, this means "roll back the operator" is not always a safe undo button once a version bump has landed and persisted objects; treat a storage-version change with the same rollback planning you'd give a database migration, not a routine deploy.

**Server-side apply and field ownership.** SSA (`kubectl apply --server-side`, or controller-runtime `client.Apply` patches with a `FieldOwner`) tracks *which manager owns which field* in `metadata.managedFields`. Two managers editing disjoint fields coexist; two claiming the same field conflict (resolvable with force). This matters because your controller writes `status` while users write `spec`, and if you ever apply into `spec` (e.g. defaulting a field) you must set a stable `FieldOwner` so you don't fight the user's own `kubectl apply` on the next round. Reconcile writes should be scoped to the fields you own.

## Perspectives

**API-design perspective.** A CRD schema is a promise to every future caller, including future-you. "Invalid states unrepresentable" — CEL plus enums plus immutability rules — is cheaper than defensive Go code re-checking the same invariants in every reconcile, and it's cheaper still than the incident where a bad object slipped through and corrupted status before anyone noticed.

**Operator/fleet perspective.** Across 40 clusters, a CRD schema change is a rollout problem, not a single `kubectl apply`. Old and new operator binaries must both tolerate whichever API version is live during a rolling upgrade — this is the practical reason versioning discipline matters, not an admission-time nicety you can defer.

**Security/FinOps perspective.** For a cost/budget CRD specifically, validation *is* your first fraud- and error-prevention control. A `Budget.spec.hardLimit: -1` rejected at admission never becomes a wrong invoice or a bypassed budget enforcement further down the pipeline — the cheapest bug is the one that never reaches your reconcile loop.

**Extensibility-ecosystem perspective.** CRDs are how the entire Kubernetes ecosystem — cert-manager, the Prometheus Operator, KubeVirt, Kueue, even DRA's own `resource.k8s.io` types — extends the API without forking Kubernetes itself. Your three CRDs are doing exactly what those widely-run production systems do; there's no separate, lesser mechanism for "your" API versus "real" ones.

## Real-world use cases

- **"Kubernetes CRD Validation Using CEL"** — Google Open Source Blog. https://opensource.googleblog.com/2023/11/kubernetes-crd-validation-using-cel.html — official Google engineering explanation of CEL validation rules for CRDs; the direct production source for this lesson's CEL section.
- **"Enforce CRD Immutability with CEL Transition Rules"** — official kubernetes.io blog. https://kubernetes.io/blog/2022/09/29/enforce-immutability-using-cel/ — walks the exact `self == oldSelf` immutability pattern this lesson's `Budget.spec.Scope` example uses, from the project itself.
- **"Leveraging Kubernetes virtual machines at Cloudflare with KubeVirt"** — Cloudflare Blog. https://blog.cloudflare.com/leveraging-kubernetes-virtual-machines-with-kubevirt/ — KubeVirt's `VirtualMachine`/`VirtualMachineInstance` CRDs running in real Cloudflare production infrastructure; a large-scale CRD-as-API-extension case study to contrast against the cost operator's own three CRDs.
- **kubernetes/kubernetes #110720** — "CRD Conversion webhook down results in controller-manager GC failure." https://github.com/kubernetes/kubernetes/issues/110720 — real production bug report showing a conversion-webhook outage stalling the garbage collector's cache sync; ties CRD versioning directly into lesson 06's GC material.

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

**Conversion-webhook decision table.** Given three hypothetical `v1alpha1 → v1beta1` changes to `Budget`, classify each as passthrough-safe or webhook-required, using the rule of thumb above:

| Change | Passthrough OK? | Why |
|---|---|---|
| (a) Add optional field `notifyChannel` with a default | Yes — `None` | Old objects are valid under the new schema unmodified; the new field just defaults on read. |
| (b) Rename `hardLimit` → `hardCapUSD` | No — webhook required | The field name changed; a passthrough leaves old objects with no `hardCapUSD` and a dangling `hardLimit` the new schema doesn't know. |
| (c) Split `window: "24h"` into `windowValue: 24` + `windowUnit: "h"` | No — webhook required | This is a genuine reshape: converting requires parsing the old string and writing two new fields, and converting back requires reassembling the string. |

For case (c), the conversion webhook's `Convert` function has to run in both directions:

```go
func (c *BudgetConverter) ConvertTo(src *v1alpha1.Budget) (*v1beta1.Budget, error) {
    dst := &v1beta1.Budget{ObjectMeta: src.ObjectMeta}
    value, unit, err := parseWindow(src.Spec.Window) // "24h" -> (24, "h")
    if err != nil {
        return nil, err
    }
    dst.Spec.WindowValue, dst.Spec.WindowUnit = value, unit
    // ... copy remaining fields ...
    return dst, nil
}

func (c *BudgetConverter) ConvertFrom(src *v1beta1.Budget) (*v1alpha1.Budget, error) {
    dst := &v1alpha1.Budget{ObjectMeta: src.ObjectMeta}
    dst.Spec.Window = fmt.Sprintf("%d%s", src.Spec.WindowValue, src.Spec.WindowUnit) // (24, "h") -> "24h"
    // ... copy remaining fields ...
    return dst, nil
}
```

Both directions must round-trip losslessly through whichever version is `storage: true`, or a client pinned to the other version eventually sees corrupted data.

## Practice

Advance `gpu-cost-operator`: design and scaffold all three v1alpha1 CRDs.

1. Scaffold the group and types (kubebuilder): `kubebuilder create api --group cost --version v1alpha1 --kind GPUCostPolicy` (repeat for `WorkloadCost`, `Budget`), generating both resource and controller.
2. **`GPUCostPolicy`** — a price model. `spec.rates[]` mapping GPU class (`a100`, `h100`, ...) to `$-per-GPU-hour` (`resource.Quantity`), plus `spec.currency` (enum). CEL: every rate `>= 0`; currency in the enum. `/status` subresource with a `Ready` condition and `observedGeneration`.
3. **`WorkloadCost`** — status-heavy. `spec.workloadRef` + `spec.policyRef`. Status carries `observedGPUHours`, `computedSpend`, `observedGeneration`, and conditions. `/status` subresource. Printer columns for spend and GPU-hours. Cap any status history array explicitly (see the etcd size-limit note in Core concepts) rather than leaving it unbounded.
4. **`Budget`** — as in the worked example: hard/soft limits, currency enum, window pattern, immutable scope selector, `hardLimit >= softLimit` cross-field CEL, non-negative limits.
5. Run `make manifests generate` and `kubectl apply -f config/crd/bases/`.

**Acceptance:**
- `kubectl apply` of a valid instance of each kind **succeeds**.
- `kubectl apply` of a `Budget` with `hardLimit: -1` **fails** with the CEL message (proves admission-time rejection, not controller-time).
- `kubectl apply` of a `Budget` with `softLimit > hardLimit` **fails** on the type-level rule.
- `kubectl explain budget.spec` shows your fields and constraints; `kubectl get budget` shows your printer columns.
- Editing a `Budget`'s `scope` after creation is rejected as immutable.

This directly produces the checkpoint's item 5 ("CRDs reject invalid specs via CEL … at `kubectl apply`") and lays the schema every later lesson's controller, RBAC, and webhook work builds on.

## Common pitfalls

1. **Believing OpenAPI `required`/`enum` markers can express cross-field constraints.** They can't — they see one field in isolation. Any rule that compares two fields (`hardLimit >= softLimit`) or references the prior value (`self == oldSelf`) has to be CEL.
2. **Forgetting the `/status` RBAC grant is separate from the main-resource grant.** This connects forward to lesson 07: status writes fail at runtime with a `Forbidden` that passes CI, because envtest doesn't enforce RBAC, and only surfaces once the operator hits a real cluster.
3. **Shipping an "additive-looking" version bump that's secretly a semantic change** — e.g. changing units from MB to GiB while keeping the same field name. Passthrough conversion validates shape, not semantics, so this silently corrupts every existing object's meaning without a single validation error firing.
4. **Writing a CEL rule with an unbounded loop over a list that's fine in testing but blows the per-rule cost budget on a real large list in production** — at which point it's the *CRD update itself* that gets rejected, not a CR, which is a confusing failure to debug for the first time in a maintenance window.

## Self-check

**(a) Why is `/status` a subresource, and what does it change about updates and `observedGeneration`?**

**Answer:** Making status a subresource splits the write path: writes to the main object can't change `.status`, and writes to `/status` can't change `.spec`. Two consequences. First, RBAC and code separate desired state (users/clients write spec) from observed state (only the controller, via `Status().Update()`/`Status().Patch()`, writes status) — a client can't accidentally clobber controller-computed fields. Second, `.metadata.generation` increments **only on spec changes**, not on status writes. That makes `status.observedGeneration` a reliable "have I reconciled the current spec?" signal: when `observedGeneration == metadata.generation` you're up to date. Without the subresource, every status write would bump generation and destroy that signal, and you'd risk write conflicts between spec and status updates racing on the same object.

**(b) When do you need a conversion webhook versus a new version with defaulting?**

**Answer:** A new version with defaulting (conversion strategy `None`) is enough when the change is *additive and passthrough-compatible*: adding an optional field, giving it a schema default, tightening validation that old objects already satisfy. The API server converts between versions by trivial passthrough because the shapes are compatible. You need a **conversion webhook** (strategy `Webhook`) when a field is renamed, restructured, split/merged, or changes units or semantics — anything where converting an old object to the new shape (and back to the storage version) requires actual logic, not a passthrough. The webhook must convert in both directions and round-trip losslessly through the single storage version; otherwise clients pinned to the old version see corrupted data. Heuristic: can I express the migration purely as "new optional field + default"? If yes, no webhook. If it requires reading old fields to compute new ones, webhook.

**(c) What can CEL validate that plain OpenAPI cannot — give an example?**

**Answer:** OpenAPI markers constrain one field in isolation (enum, min/max, pattern, required). CEL (`x-kubernetes-validations`) sees the whole object via `self` and the prior value via `oldSelf`, so it can express **cross-field** and **transition** constraints. Example from `Budget`: `self.hardLimit >= self.softLimit` — a relationship between two fields that no single-field OpenAPI constraint can capture. Another: `self == oldSelf` on `scope` enforces **immutability after creation**, which OpenAPI has no concept of. CEL can also do conditional requirements (`has(self.window) || self.scope == 'cluster'`) and list uniqueness by key. It runs at admission with a cost budget, so it stays cheap and rejects bad objects before they reach etcd or your controller.

**(d) You need to rename `Budget.spec.limit` to `Budget.spec.hardLimitUSD`. Walk the exact steps.**

**Answer:** This is a renamed field, so passthrough conversion cannot handle it — it needs a conversion webhook. Steps: (1) add a new CRD version, e.g. `v1beta1`, with `hardLimitUSD` replacing `limit`; (2) pick exactly one version as `storage: true` for the migration window — typically keep `v1alpha1` as storage initially so a rollback stays safe, then flip storage to `v1beta1` once the new controller is proven; (3) write and deploy a conversion webhook whose `ConvertTo`/`ConvertFrom` copy `limit` ↔ `hardLimitUSD` in both directions, round-tripping losslessly through the storage version; (4) ship an operator binary whose scheme understands both versions before flipping any defaults, so in-flight rolling upgrades across the 40 clusters never see a version they can't decode; (5) communicate to cluster owners that `v1alpha1` is deprecated but still served — existing CRs and tooling using `limit` keep working unmodified because the webhook translates transparently; (6) only after every cluster's controller is on the new build and storage has been flipped do you deprecate/remove `v1alpha1` serving, on its own separate rollout.

## Connections & what's next

CRD design is the schema half of the contract; lesson 04's informer/workqueue machinery is the mechanism that watches whatever this lesson defines, and lesson 06's controller-runtime machinery is what actually reads and writes it — `Status().Update()` only behaves the way it does *because* of the `/status` subresource declared here. The RBAC lesson (07) needs the `/status` and `/finalizers` subresource grants this lesson's schema implies; the admission-webhooks lesson (08) builds directly on the CEL and conversion-webhook material here, since a validating webhook is CEL's escape hatch for anything the cost budget or the type system can't express. Next: **lesson 06** takes these three CRDs and wires the controller-runtime Manager, client, owner references, and finalizers that turn them into a running, restart-safe operator.

## References & further reading

**Primary sources**
- Kubebuilder Book — CRD validation markers — https://book.kubebuilder.io/reference/markers/crd-validation.html — the authoritative list of `+kubebuilder:validation` markers and how they map to schema; deep-read while writing your types.
- kubernetes.io — CustomResourceDefinitions — https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/ — the API-server view of structural schemas, subresources, versions, and the storage version.
- kubernetes.io — Common Expression Language (CEL) reference — https://kubernetes.io/docs/reference/using-api/cel/ — `self`/`oldSelf`, the cost budget, `messageExpression`, and transition rules; this is where a rejected-for-cost CRD's rules live.
- apiextensions-apiserver types godoc — https://pkg.go.dev/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1 — the Go structs behind a `CustomResourceDefinition` (`CustomResourceDefinitionVersion`, `CustomResourceValidation`, `CustomResourceSubresources`, `ValidationRule`); skim once to see what your YAML deserializes into.
- Server-Side Apply docs — https://kubernetes.io/docs/reference/using-api/server-side-apply/ — the `managedFields`/`FieldOwner` mechanics referenced in the field-ownership section above.

**Real-world engineering blogs**
- Google Open Source Blog, "Kubernetes CRD Validation Using CEL" — https://opensource.googleblog.com/2023/11/kubernetes-crd-validation-using-cel.html — what it shows: official walkthrough of CEL validation rules in production CRDs.
- kubernetes.io blog, "Enforce CRD Immutability with CEL Transition Rules" — https://kubernetes.io/blog/2022/09/29/enforce-immutability-using-cel/ — what it shows: the exact `self == oldSelf` pattern this lesson's `Budget.Scope` uses, from the project itself.
- Cloudflare Blog, "Leveraging Kubernetes virtual machines at Cloudflare with KubeVirt" — https://blog.cloudflare.com/leveraging-kubernetes-virtual-machines-with-kubevirt/ — what it shows: KubeVirt's CRDs as a real large-scale production CRD-as-API-extension case study.
- kubernetes/kubernetes issue #110720 — https://github.com/kubernetes/kubernetes/issues/110720 — what it shows: a real production bug where a down conversion webhook stalled the garbage collector, linking CRD versioning to lesson 06's GC material.

**Deeper dives**
- kubernetes.io blog, "CustomResourceDefinition Validation Rules Graduate to Beta" — https://kubernetes.io/blog/2022/09/23/crd-validation-rules-beta/ — historical/version context for when CEL validation rules became available and what changed as they matured to GA.
