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
sources: 17
---

# 02.5 · CRD and API design

> **Concept.** A CRD is a versioned API contract, not a config blob — schema, validation, subresources, and versioning decide whether your operator ages well.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Lesson 04 gave you the machine every controller runs on: reflector → FIFO → informer → lister → workqueue → worker. That machine assumes something already exists to watch — a type registered in the API server with a schema, versions, and a write contract. This lesson is where that type comes from. Before you can reconcile a `WorkloadCost`, something has to define what a `WorkloadCost` *is*: which fields are legal, which are computed, what happens when the shape changes next year. Get this lesson right and lesson 06's controller-runtime machinery has a clean, honest object to reconcile against. Get it wrong and every reconcile is compensating in Go for a schema that should have rejected the bad input at the door.

## Why this matters

Your operator's CRDs are a public API the moment someone `kubectl apply`s them. Every field is a promise you have to keep across upgrades. Get the spec/status split wrong and you fight the API server on every reconcile. Skip validation and garbage lands in etcd, then your controller panics at 3am reconciling a `WorkloadCost` with a negative rate. Ship a v1alpha1 with no versioning plan and your first breaking change means telling 40 clusters to hand-edit CRs.

For a GPU cost operator this is sharper than usual: money is involved. A `Budget` that accepts a negative limit, or a `GPUCostPolicy` with a typo'd currency, produces silently wrong spend numbers that people make purchasing decisions on. The API layer is your first and cheapest line of defense — validation rejected at admission never reaches your reconcile loop, never corrupts status, never needs a postmortem. This is also directly the job: NVIDIA and CoreWeave postings for controller/operator roles both name CRD design as core work, not an implementation detail delegated to someone else. Senior work here is designing the schema so that the invalid states you'd otherwise handle defensively in Go are simply unrepresentable.

## What's new here (calibration)

**CKA already knows:** what a CRD is, `kubectl apply -f crd.yaml`, that CRs show up in `kubectl get`, that operators define their own types. You've consumed CRDs from cert-manager, Prometheus, Argo. You know `kubectl explain` and `--v=8` to see the API traffic.

**Internal to learn now:** how the `CustomResourceDefinition` object itself is structured — `spec.versions[]`, `schema.openAPIV3Schema`, `subresources`, `additionalPrinterColumns`, `selectableFields`, `conversion`. How the API server uses that schema as a structural-schema gate (pruning unknown fields, enforcing types) *before* your controller sees anything. How kubebuilder markers compile down to that OpenAPI schema. Where CEL (`x-kubernetes-validations`) runs, what it can express that OpenAPI can't, and — the part everyone skips — how its cost budget is actually computed. Why `/status` is a *subresource* and how that changes the write path and `metadata.generation`. When multiple versions force a conversion webhook versus when defaulting suffices. And server-side apply field ownership, because your controller and users will both write to the same objects.

## Core concepts

**Versions this lesson's API claims apply to.** Everything here was verified against **Kubernetes 1.36** — `k8s.io/apiextensions-apiserver` v0.36.3 for the CRD types, admission strategies, and CEL cost limits; `k8s.io/apiserver` v0.36.3 for the CEL runtime constants; **controller-tools v0.21.0** (`controller-gen`) for the marker → schema mapping; and the kubernetes.io CRD, CEL and API-conventions references. Every error transcript below was produced by running the real apiextensions CEL validator, not transcribed from memory. Feature-gate graduation versions come from `pkg/features/kube_features.go` in the same tree.

### 1. What a CRD actually is at the API server

A `CustomResourceDefinition` is an ordinary API object in group `apiextensions.k8s.io/v1`. When you `kubectl apply` one, four things happen, in order, and knowing them tells you why a fresh CRD sometimes isn't usable for a second or two:

1. **The CRD object is validated and persisted.** This is where a bad CEL rule or an unstructural schema gets rejected — at *CRD create/update* time, not at CR-create time. That distinction causes real confusion: a cost-budget violation fails your `kubectl apply -f config/crd/bases/`, not your `kubectl apply -f sample.yaml`.
2. **The `NamesAccepted` condition is set** by the naming controller, which checks the plural/singular/kind/listKind/shortNames for collisions with existing resources in that group.
3. **The `Established` condition is set**, and only then does the apiserver install the REST endpoints `/apis/cost.example.com/v1alpha1/namespaces/{ns}/budgets` and, if declared, `.../budgets/{name}/status`.
4. **Discovery and the OpenAPI document are republished**, which is what lets `kubectl` resolve `budget` → `budgets.cost.example.com` and what lets a client's RESTMapper turn a GVK into a GVR (lesson 02).

Two operational consequences fall out. First, a controller that starts in the same second as its CRD may fail its initial LIST with `no matches for kind` — that's step 3/4 not having landed yet, and it is why `kubectl wait --for condition=established` exists. Second, `status.storedVersions` on the CRD is written by the apiserver, not by you; it is the list of every version that objects have *ever been persisted under*, and §8 shows why it governs when you may drop a version.

Here is the whole object, with everything that hangs off it:

```
 CustomResourceDefinition  (apiextensions.k8s.io/v1)
 └─ spec
    ├─ group: cost.example.com
    ├─ scope: Namespaced | Cluster
    ├─ names: {plural, singular, kind, listKind, shortNames[], categories[]}
    ├─ conversion
    │   ├─ strategy: None                 ← trivial passthrough; apiVersion rewrite only
    │   └─ strategy: Webhook
    │       └─ webhook
    │           ├─ clientConfig: {service{name,namespace,path,port} | url, caBundle}
    │           └─ conversionReviewVersions: [v1]
    └─ versions[]                          ← 1..N served versions, EXACTLY ONE storage:true
       ├─ name: v1alpha1
       ├─ served: true                     ← is there a REST endpoint for it?
       ├─ storage: true                    ← is this the shape etcd holds?
       ├─ deprecated / deprecationWarning  ← emits a Warning: header to clients
       ├─ subresources
       │   ├─ status: {}                   ← installs .../{name}/status
       │   └─ scale: {specReplicasPath, statusReplicasPath, labelSelectorPath}
       ├─ additionalPrinterColumns[]       ← {name,type,format,description,priority,jsonPath}
       ├─ selectableFields[]               ← {jsonPath} → usable in --field-selector (GA 1.32)
       └─ schema.openAPIV3Schema           ← THE STRUCTURAL SCHEMA
          │
          ├─ type/properties/required/... ────────▶ shape + single-field validation
          ├─ default: <value>             ────────▶ defaulting (see §5)
          ├─ x-kubernetes-preserve-unknown-fields ▶ opt OUT of pruning here
          ├─ x-kubernetes-int-or-string           ▶ accept int or string (Quantity, IntOrString)
          ├─ x-kubernetes-embedded-resource       ▶ this subtree is a whole k8s object
          ├─ x-kubernetes-list-type: map|set|atomic
          ├─ x-kubernetes-list-map-keys: [type]   ▶ merge key for SSA + strategic merge
          ├─ x-kubernetes-map-type: granular|atomic
          └─ x-kubernetes-validations[]   ────────▶ CEL (see §7)
             └─ {rule, message, messageExpression, reason, fieldPath, optionalOldSelf}

 status  (written by the apiserver, read-only to you)
 ├─ conditions[]: NamesAccepted, Established, Terminating, KubernetesAPIApprovalPolicyConformant
 ├─ acceptedNames
 └─ storedVersions[]  ← every version objects have EVER been written under
```

Everything below explains one branch of that tree.

### 2. The structural schema, and why pruning is silent

Every `apiextensions.k8s.io/v1` CRD **must** carry a schema, and that schema must be *structural*. The definition is three conditions, and they are worth knowing verbatim because "my CRD won't apply" almost always means one of them:

1. **Everything has a type.** Each object node, each field, and each array item specifies a non-empty `type` — except nodes marked `x-kubernetes-int-or-string: true` or `x-kubernetes-preserve-unknown-fields: true`.
2. **Logical junctors don't introduce new shape.** For each field or array item mentioned inside `allOf` / `anyOf` / `oneOf` / `not`, the schema *also* specifies that field or item outside those junctors. In other words, junctors may only constrain shape that is already declared.
3. **Junctors stay pure.** `description`, `type`, `default`, `nullable` and the `x-kubernetes-*` extensions may not appear inside `allOf`/`anyOf`/`oneOf`/`not` (with a documented exception for the int-or-string pattern).

Rule 1 is the one that decides your Go types: this is exactly why `controller-gen` renders `map[string]string` as `type: object, additionalProperties: {type: string}` and refuses to generate anything for `interface{}` unless you add `+kubebuilder:pruning:PreserveUnknownFields`.

**Pruning is the enforcement mechanism, and it does not error.** Before persisting, the apiserver walks the incoming object against the structural schema and **deletes every field the schema does not declare**. It does not reject; it silently drops. Apply this:

```yaml
apiVersion: cost.example.com/v1alpha1
kind: Budget
metadata: {name: team-ml}
spec:
  hardLimit: "5000"
  hardLimitt: "9999"      # typo — one extra 't'
```

and you get a `Budget` with `hardLimit: "5000"` and no error, no warning, no `hardLimitt`. The user believes they set 9,999. This is the single most common "why is my CRD ignoring my config" bug, and the mitigation is `kubectl apply --validate=strict` / `kubectl apply --server-side` with `fieldValidation=Strict`, which turns unknown fields into a `400` instead of a silent drop. Design accordingly: **never rely on an undeclared field getting through, and declare everything you intend to read.**

The opt-out is `x-kubernetes-preserve-unknown-fields: true` (kubebuilder marker `+kubebuilder:pruning:PreserveUnknownFields`) at a specific node — pruning stays on everywhere else. Use it for genuinely opaque blobs (a vendor's passthrough config), and know the price: CEL cannot see inside a preserved-unknown subtree, and neither can server-side apply's field ownership.

### 3. Spec, status, and what the `/status` subresource actually changes

`spec` is desired state, written by users and clients. `status` is observed state, written only by your controller. The split is what makes the `/status` subresource meaningful, and the subresource's behaviour is not folklore — it is twenty lines of Go in `apiextensions-apiserver/pkg/registry/customresource`.

**On create** (`customResourceStrategy.PrepareForCreate`), when the status subresource is enabled:

```go
delete(customResource, "status")   // create can never set status
accessor.SetGeneration(1)          // generation always starts at 1
```

**On a main-resource update** (`customResourceStrategy.PrepareForUpdate`):

```go
if a.status != nil {                       // /status subresource enabled
    switch {
    case oldHasStatus: new["status"] = old["status"]  // your status payload is DISCARDED
    case newHasStatus: delete(new, "status")
    }
}
// generation increments iff anything OUTSIDE metadata changed
if !apiequality.Semantic.DeepEqual(copyNonMetadata(new), copyNonMetadata(old)) {
    newAccessor.SetGeneration(oldAccessor.GetGeneration() + 1)
}
```

**On a `/status` update** (`statusStrategy.PrepareForUpdate`): the *entire old object is deep-copied over the new one*, then only `status` (and `managedFields`) is written back. So a `/status` write physically cannot change `spec`, `metadata.labels`, `metadata.finalizers`, or anything else.

Four consequences, all of which you will be asked about:

- **A plain `Update()` carrying computed status silently drops it.** No error. Your `computedSpend` is simply not persisted. This is lesson 06's canonical bug.
- **`metadata.generation` increments only when non-metadata content changes** — in practice, when `spec` changes. Status writes never bump it, because status is restored from the old object before the comparison runs.
- **Therefore `status.observedGeneration` is a trustworthy "have I reconciled the current spec?" signal.** `observedGeneration == metadata.generation` means up to date; `<` means a spec change is pending. Without the subresource, every status write would bump generation and the signal would be noise.
- **Metadata edits (labels, annotations, finalizers) do *not* bump generation.** Adding a finalizer via `Update()` is generation-neutral, which is why lesson 06's finalizer dance doesn't perturb `observedGeneration`.

`/scale` is the other standard subresource. It maps three JSONPaths — `specReplicasPath`, `statusReplicasPath`, and optionally `labelSelectorPath` — so `kubectl scale` and the HPA can target your CRD. The GPU cost CRDs don't scale, so you won't declare it, but know it exists and know that `labelSelectorPath` is what the HPA needs to find the pods.

**Conditions: use the standard contract.** `metav1.Condition` has `type`, `status` (`True`/`False`/`Unknown`), `observedGeneration`, `lastTransitionTime`, `reason` (CamelCase, non-empty, required), `message`. The Kubernetes API conventions are specific about the list markers — `+listType=map`, `+listMapKey=type`, `+patchStrategy=merge`, `+patchMergeKey=type` — because that is what makes the list behave as a map keyed by `type` for server-side apply and strategic merge patch. Use `meta.SetStatusCondition` / `meta.FindStatusCondition` rather than hand-editing the slice; `SetStatusCondition` is the thing that correctly leaves `lastTransitionTime` alone when only the message changed.

Convention notes worth internalising, straight from the API conventions doc: condition types are **adjectives or past-tense verbs describing observed state** (`Ready`, `Succeeded`, `MemoryExhausted`), never present-tense transitions (`Deploying`); polarity is per-condition, so you cannot generically summarise a condition list; controllers should set their conditions **on first visit even with `status: Unknown`**, so other components know the controller exists and is working; and the absence of a known condition should be read as `Unknown`. Pick a small set of orthogonal types — for the cost operator, `Ready`, `BudgetExceeded`, `PolicyResolved` — and do not use conditions as your control state.

*(New in 1.36: the CRD object itself now tracks `status.observedGeneration` and per-condition `observedGeneration`, under the `CRDObservedGenerationTracking` gate — KEP-4192, beta and on by default. Same pattern, applied one level up.)*

### 4. Printer columns and selectable fields

```yaml
additionalPrinterColumns:
- name: Spend
  type: string
  jsonPath: .status.currentSpend
  priority: 0          # 0 = always shown; >0 = only with -o wide
- name: Age
  type: date
  jsonPath: .metadata.creationTimestamp
```

Types are the OpenAPI set: `integer`, `number`, `string`, `boolean`, `date`. Priority `0` shows in the default `kubectl get`; anything higher is hidden until `-o wide`. Column rendering happens **server-side** — the apiserver evaluates the JSONPath and returns a `Table` — so a wrong path yields an empty cell, never an error.

**Selectable fields** (`selectableFields[].jsonPath`, `CustomResourceFieldSelectors`: alpha 1.30 → beta 1.31 → **GA and locked on in 1.32**) let clients do `kubectl get budgets --field-selector spec.currency=USD`, and let a controller-runtime cache filter server-side rather than pulling every object. That matters for the memory arithmetic in lesson 04: a field selector shrinks `N`, and `N × 5 KiB` is your cache. Only string-typed leaf fields under `spec`/`status` are eligible.

### 5. Defaulting

A `default:` in the schema is applied by the apiserver at three moments, and the third one surprises people:

1. **On the incoming request**, after decoding and **after mutating admission webhooks** — so a webhook's output is itself defaulted.
2. **On read from etcd**, so objects written before a default existed appear to have it.
3. **On the result of a conversion**, using the target version's schema.

The critical caveat, quoted in substance from the CRD docs: **defaults applied when reading from etcd are not written back to etcd.** The stored object keeps the field absent. That means you cannot use "the default is present" as evidence the user accepted it, and it means changing a default changes the observed value of every existing object on the next read. Two more rules: defaults on non-leaf nodes must themselves validate against the schema and are pruned like anything else; and a `null` supplied for a non-nullable field is **pruned first, then defaulted**, so `currency: null` yields `currency: USD`, not an error.

Schema defaults are static values only. Anything conditional — "default `window` to `24h` unless `scope` is cluster-wide" — needs a mutating admission (defaulting) webhook, which is lesson 08. **Prefer the schema default;** it costs nothing, cannot be down, and shows up in `kubectl explain`.

### 6. Validation layer one — OpenAPI markers

These are single-field constraints. `controller-gen` compiles the marker into the OpenAPI schema and the apiserver enforces it at admission with zero controller code. The complete generator-supported set (controller-tools v0.21.0, `pkg/crd/markers/validation.go`), all prefixed `+kubebuilder:validation:`:

| Category | Markers | Schema key |
|---|---|---|
| Numeric | `Maximum`, `Minimum`, `ExclusiveMaximum`, `ExclusiveMinimum`, `MultipleOf` | `maximum`, `minimum`, … |
| String | `MaxLength`, `MinLength`, `Pattern` | `maxLength`, `minLength`, `pattern` |
| Array | `MaxItems`, `MinItems`, `UniqueItems` | `maxItems`, `minItems`, `uniqueItems` |
| Object | `MinProperties`, `MaxProperties` | `minProperties`, `maxProperties` |
| General | `Enum`, `Format`, `Type`, `Required`, `Optional`, `Schemaless` | `enum`, `format`, `type`, `required` |
| Extensions | `XPreserveUnknownFields`, `XEmbeddedResource`, `XIntOrString`, `XValidation` | the `x-kubernetes-*` keys |
| Type-level only | `ExactlyOneOf`, `AtMostOneOf`, `AtLeastOneOf`, `+k8s:enum` | generated CEL / junctors |
| Non-`validation:` prefix | `+kubebuilder:default=`, `+kubebuilder:example=`, `+optional`, `+nullable`, `+kubebuilder:title=`, `+kubebuilder:pruning:PreserveUnknownFields` | `default`, `example`, … |

Two mechanics worth knowing. Any of these markers can be applied to a **type** as well as a field, so you can define `type Currency string` once with its enum and reuse it. And prefixing with `items:` — `+kubebuilder:validation:items:MaxLength=63` — applies the constraint to a slice's *elements* rather than the slice.

```go
// +kubebuilder:validation:Enum=USD;EUR;GBP
// +kubebuilder:default=USD
Currency string `json:"currency"`

// +kubebuilder:validation:Pattern=`^[0-9]+(h|d)$`
// +kubebuilder:validation:MaxLength=8
Window string `json:"window"`

// +kubebuilder:validation:MaxItems=25
// +kubebuilder:validation:items:MaxLength=63
Tags []string `json:"tags,omitempty"`
```

**Markers cannot see other fields.** Any rule relating two fields, or comparing to a previous value, must be CEL.

### 7. Validation layer two — CEL (`x-kubernetes-validations`)

CEL validation rules went alpha in 1.23, beta in 1.25, and **GA in 1.29**. A rule is evaluated at admission with `self` bound to the schema node the rule is attached to:

| Rule attached to | `self` is | Example |
|---|---|---|
| the root of the object | the whole CR | `self.spec.hardLimit != ''` |
| an object node | that object | `self.hardLimit >= self.softLimit` (on `spec`) |
| a scalar | the scalar value | `self.startsWith('gpu-')` |
| an array | the list | `self.all(x, x.weight > 0)` |
| a map (`additionalProperties`) | the map | `'a100' in self` |

`ValidationRule` has six fields, all generated from the `+kubebuilder:validation:XValidation:` marker: `rule`, `message`, `messageExpression`, `reason`, `fieldPath`, `optionalOldSelf`.

- **`message`** is a static string; **`messageExpression`** is a CEL expression returning a string, so you can interpolate: `"hardLimit " + string(self.hardLimit) + " is below softLimit"`. If both are present, `messageExpression` wins; if it errors or returns empty, it falls back to `message`.
- **`reason`** sets the HTTP status reason: `FieldValueInvalid` (default), `FieldValueForbidden`, `FieldValueRequired`, `FieldValueDuplicate`.
- **`fieldPath`** is a relative JSON path that points the error at a subfield — so a cross-field rule declared on `spec` can blame `.hardLimit` instead of the whole object. Very much worth setting; it is the difference between a usable error and "spec: Invalid value".
- **`optionalOldSelf`** turns `oldSelf` into a CEL optional so a transition rule also fires on **create** (`oldSelf.hasValue()`, `oldSelf.value()`).

**A rule that mentions `oldSelf` is implicitly a transition rule** and, by default, runs only on UPDATE and is skipped when no old value exists. The canonical patterns, from the Kubernetes docs:

| Intent | Rule |
|---|---|
| Immutability | `self.foo == oldSelf.foo` |
| Prevent removal once set | `!has(oldSelf.field) \|\| has(self.field)` |
| Append-only set | `self.all(element, element in oldSelf)` |
| Constrain the next value | `oldSelf != 'X' \|\| self in ['A', 'B']` |
| Monotonic counter | `self >= oldSelf` |

**Validation ratcheting** (`CRDValidationRatcheting`: alpha 1.28 → beta 1.30 → **GA and locked on in 1.33**) changes the update semantics in a way you should design around: on UPDATE/PATCH, validation errors on fields whose value **did not change** are ignored. So tightening a constraint no longer bricks every existing object — a user can still patch an unrelated field on a CR that violates your new rule. This is what makes shipping stricter validation to 40 clusters survivable.

#### The type-mapping trap: `resource.Quantity` is not a number

This is the single most common CEL bug in cost/quota CRDs, and it is worth walking with real output. `controller-gen` renders `resource.Quantity` (from `k8s.io/apimachinery/pkg/api/resource`) as:

```yaml
x-kubernetes-int-or-string: true
anyOf: [{type: integer}, {type: string}]
pattern: "^(\\+|-)?(([0-9]+(\\.[0-9]*)?)|(\\.[0-9]+))(([KMGTPE]i)|[numkMGTPE]|([eE](\\+|-)?(([0-9]+(\\.[0-9]*)?)|(\\.[0-9]+))))?$"
```

An int-or-string node is typed `dyn` in CEL. So `rule: "self >= 0"` **compiles fine and then behaves differently depending on what the user typed.** Running the real apiextensions CEL validator (`k8s.io/apiextensions-apiserver/pkg/test`, v0.36.3) against a schema with exactly that rule:

```
=== value is an integer: {"naive": 5}
    ACCEPTED                          ← the rule ran and passed

=== value is a string: {"naive": "500"}
    REJECTED: spec.naive: Invalid value: "": 'no such overload': call arguments
              did not match a supported operator, function or macro signature
              for rule: naive rule on an int-or-string field
```

So `hardLimit: 5` is validated and `hardLimit: "5"` is rejected with an incomprehensible internal error — and `hardLimit: "5000m"`, which is what a Quantity user will actually write, is rejected too. **`+kubebuilder:validation:Minimum=0` is equally useless here**, because `minimum` in OpenAPI applies to numeric types and the node's declared type is int-or-string.

The correct form uses the **Kubernetes quantity CEL library** (added in Kubernetes 1.28). Two functions and a gotcha:

- `isQuantity(string) bool` and `quantity(string) Quantity` — parse.
- Member functions on the parsed value: `compareTo`, `isGreaterThan`, `isLessThan`, `asInteger`, `isInteger`, `asApproximateFloat`, `add`, `sub`.
- **`sign` is a *global* function, not a member.** The docs table lists it among "member functions of a Quantity," but the implementation registers it as `cel.Overload("quantity_sign", …)`, not `cel.MemberOverload`. Write `sign(quantity(self))`; `quantity(self).sign()` fails to compile with `found no matching overload for 'sign' applied to 'kubernetes.Quantity.()'`. That is a real, reproducible discrepancy between the docs and the code, and it will cost you twenty minutes if nobody tells you.

Rewritten and re-run against the real validator:

```yaml
hardLimit:
  x-kubernetes-int-or-string: true
  anyOf: [{type: integer}, {type: string}]
  pattern: "^(\\+|-)?(([0-9]+(\\.[0-9]*)?)|(\\.[0-9]+))(([KMGTPE]i)|[numkMGTPE]|…)?$"
  x-kubernetes-validations:
  - rule: "sign(quantity(self)) >= 0"
    message: "hardLimit must be non-negative"
# and on the parent `spec` object:
x-kubernetes-validations:
- rule: "quantity(self.hardLimit).compareTo(quantity(self.softLimit)) >= 0"
  message: "hardLimit must be >= softLimit"
  fieldPath: ".hardLimit"
```

```
=== {"hardLimit":"5000","softLimit":"4000","window":"24h"}
    ACCEPTED
=== {"hardLimit":"100","softLimit":"200","window":"24h"}
    REJECTED: spec.hardLimit: Invalid value: hardLimit must be >= softLimit
=== {"hardLimit":"-5","softLimit":"-10","window":"24h"}
    REJECTED: spec.hardLimit: Invalid value: "-5": hardLimit must be non-negative
    REJECTED: spec.softLimit: Invalid value: "-10": softLimit must be non-negative
=== {"hardLimit":"1500m","softLimit":"1","window":"24h"}
    ACCEPTED                       ← 1500m = 1.5, correctly ≥ 1
=== {"hardLimit":"500m","softLimit":"1","window":"24h"}
    REJECTED: spec.hardLimit: Invalid value: hardLimit must be >= softLimit
                                   ← 500m = 0.5 < 1. A LEXICAL string compare
                                     would have accepted this ("5" > "1").
```

Those last two cases are why "just compare the strings" is not an option: SI suffixes make lexical order and numeric order disagree. Note also the `fieldPath: ".hardLimit"` doing its job — the cross-field error is reported against `spec.hardLimit`, not against `spec`.

**The other CEL libraries you have**, with the Kubernetes version each arrived in: list (`indexOf`, `min`, `max`, `sum`, `isSorted`), regex (`find`, `findAll`), URL (`isURL`, `url()`, `getScheme`, `getHost`…), IP and CIDR, authorizer (`authorizer.group(...).resource(...).check(...)` — admission policies only), **format** (1.31: `format.dns1123Label()`, `format.qualifiedName()`, `format.uuid()`, and friends, so you stop hand-writing DNS regexes), **quantity** (1.28), and **semver**.

#### The cost budget, explained concretely

CEL is non-Turing-complete precisely so its worst-case runtime can be bounded. Kubernetes bounds it twice — once when you write the CRD, once when a CR is validated — using a hardware-independent, deterministic **cost unit**.

Cost-unit basics from the CEL reference: a comparison such as `<` costs **1**; a list literal has a fixed base cost of **40**; native functions are priced by time complexity, e.g. a regex `match`/`find` is estimated at `length(regex) × length(input)`, reflecting RE2's worst case.

The four limits, with their source constants:

| Limit | Constant | Value | When it applies | Roughly |
|---|---|---:|---|---|
| Per-expression **estimated** cost | `validation.StaticEstimatedCostLimit` | **10,000,000** | CRD create/update | one rule's worst case |
| Whole-schema **estimated** cost | `validation.StaticEstimatedCRDCostLimit` | **100,000,000** | CRD create/update | all rules, one CRD |
| Per-call **runtime** budget | `celconfig.PerCallLimit` | **1,000,000** | each CR validation | ≈ 0.1 s of CPU per expression |
| Per-validation **runtime** budget | `celconfig.RuntimeCELCostBudget` | **10,000,000** | each CR validation | ≈ 1 s of CPU for all rules together |

The estimator "always assumes the worst case" for anything iterable when no bound is given, so **the fix for a rejected rule is almost never to simplify the rule — it is to add `maxItems`, `maxLength`, `maxProperties`.**

Here is the whole thing measured. Take the docs' canonical expensive rule, `self.all(x, x.contains('gpu'))` on an array of strings, and feed CRDs to the real validator (`apiextensions/validation.ValidateCustomResourceDefinition`, v0.36.3) with no bounds:

```
CRD REJECTED: spec.validation.openAPIV3Schema.properties[spec].properties[tags]
  .x-kubernetes-validations[0].rule: Forbidden: estimated rule cost exceeds budget
  by factor of more than 100x (try simplifying the rule, or adding maxItems,
  maxProperties, and maxLength where arrays, maps, and strings are declared)
CRD REJECTED: …rule: Forbidden: contributed to estimated rule cost total exceeding
  cost limit for entire OpenAPIv3 schema
CRD REJECTED: spec.validation.openAPIV3Schema: Forbidden: x-kubernetes-validations
  estimated rule cost total for entire OpenAPIv3 schema exceeds budget by factor of
  more than 100x (…)
```

Three errors from one bad rule: the per-expression limit, the contribution to the total, and the total itself. ("More than 100x" is only how the message formats large overshoots — `getCostErrorMessage` prints an exact factor below 100×.)

Now add bounds and binary-search the largest `maxItems` that is still accepted, for several `maxLength` values:

| `maxLength` on items | Largest accepted `maxItems` | Implied cost per item (10,000,000 ÷ maxItems) |
|---:|---:|---:|
| 16 | 909,090 | 11 |
| 63 | 333,333 | 30 |
| 253 | 94,339 | 106 |
| 1,024 | 24,154 | 414 |

Fit a line through those and you get a rule of thumb you can apply without a test harness:

```
  cost_per_item(contains, L) ≈ 0.4 · L + 4.6        (L = maxLength of the item)

  a rule of the form  self.all(x, x.contains(...))  is accepted while
      maxItems × (0.4 · maxLength + 4.6)  ≤  10,000,000

  worked: tags with maxLength 63 → 30 units/item → budget allows 333,000 items.
          You will declare maxItems: 25. Cost ≈ 750. You are 4 orders of
          magnitude under the limit and will never think about this again.
```

**That last line is the real lesson.** Realistic bounds put you nowhere near the ceiling; the budget only bites when a bound is *missing*, or when lists nest. Nesting is the killer, and the mechanism is multiplication: `self.all(...)` on a list whose items are themselves lists runs the inner traversal once per outer element, so the cost is the product of the two bounds. The docs give the minimal example — `self.all(x, x == 5)` on an unbounded `[]int` is accepted; the same rule on an unbounded `[][]int` is rejected. **Avoid nested lists and maps wherever a validation rule has to walk them.**

Finally, the two runtime budgets are separate from the estimate: even a rule that passed estimation is halted mid-evaluation if it exceeds `PerCallLimit` (1,000,000 ≈ 0.1 s) or if the CR's rules collectively exceed `RuntimeCELCostBudget` (10,000,000 ≈ 1 s). Those exist because a *real* object may be shaped differently from the estimator's worst case, in either direction.

### Where each validation actually runs

Same request, six gates. This is the diagram to have in your head when someone asks "why didn't my rule fire":

```
  kubectl apply -f budget.yaml
        │
        ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │ 1. AUTHN / AUTHZ                    RBAC on budgets (and separately   │
  │                                     on budgets/status — lesson 07)    │
  ├───────────────────────────────────────────────────────────────────────┤
  │ 2. DECODE + STRUCTURAL PRUNING      undeclared fields DELETED silently│
  │                                     (unless fieldValidation=Strict →  │
  │                                      400, or Warn → Warning: header)  │
  ├───────────────────────────────────────────────────────────────────────┤
  │ 3. MUTATING ADMISSION WEBHOOKS      lesson 08. Runs BEFORE defaulting │
  │                                     of its own output.                │
  ├───────────────────────────────────────────────────────────────────────┤
  │ 4. DEFAULTING                       schema `default:` values applied  │
  ├───────────────────────────────────────────────────────────────────────┤
  │ 5. OPENAPI VALIDATION               type, required, enum, pattern,    │
  │                                     min/max, maxLength, maxItems      │
  ├───────────────────────────────────────────────────────────────────────┤
  │ 6. CEL x-kubernetes-validations     self / oldSelf. Budget: 1e6 per   │
  │                                     expression, 1e7 for the object.   │
  │                                     Ratcheting: unchanged fields with │
  │                                     errors are IGNORED on update.     │
  ├───────────────────────────────────────────────────────────────────────┤
  │ 7. VALIDATING ADMISSION WEBHOOKS / ValidatingAdmissionPolicy          │
  │    (lesson 08 — the escape hatch for anything CEL-in-CRD can't see,   │
  │     e.g. "does the referenced GPUCostPolicy exist?")                  │
  └───────────────────────────────────────────────────────────────────────┘
        │
        ▼  convert to STORAGE version (None passthrough, or webhook — §8)
        ▼  serialize (protobuf for built-ins, JSON for CRs)
        ▼  etcd write   ← ≤ 1.5 MiB per object (etcd --max-request-bytes default)
        │                 apiserver body limit is 3 MiB (MaxRequestBodyBytes)
        ▼
   generation++ iff non-metadata content changed   → your controller wakes (lesson 04)
```

Two readings you should be able to give from this diagram. **"My CEL rule didn't run"** → the field was pruned at gate 2 before CEL ever saw it, or the object never reached gate 6 because gate 5 rejected it first. **"My defaulting webhook's value isn't validated"** → it is: gate 3 precedes gates 4–6, deliberately.

### 8. Versions, the storage version, and conversion

A CRD can serve several versions simultaneously; **exactly one** has `storage: true`. Every write is converted to the storage version before it hits etcd; every read is converted from the storage version to whatever version the client asked for. Conversion is required in more situations than people expect — the docs enumerate them:

- a CR is **requested** in a version different from the stored version;
- a **watch** is opened on one version but the changed object is stored in another;
- a **PUT** arrives in a version different from the storage version.

Two strategies:

- **`strategy: None`** — the apiserver rewrites `apiVersion` and changes nothing else. Legal only when the versions are structurally compatible: adding an optional field, adding a default, tightening validation that existing objects already satisfy.
- **`strategy: Webhook`** — the apiserver POSTs a `ConversionReview` containing **a list of objects** (batched to minimise calls) to your webhook, which returns `convertedObjects` in the same order. Constraints the docs are explicit about: the service must be exposed on **port 443**; the scheme must be **https**; a conversion webhook **must not mutate anything in `metadata` other than `labels` and `annotations`** — attempted changes to `name`, `uid`, `namespace` fail the request outright and all other metadata changes are silently ignored.

**The decision rule:** can the migration be expressed as "new optional field + default"? If yes, `None`. If converting requires *reading old fields to compute new ones*, you need a webhook.

**`status.storedVersions` is the thing that makes version removal hard.** The apiserver appends to it every time it persists an object under a version. You may not stop serving a version until every stored object has been rewritten under a newer one and the old version has been removed from `storedVersions`. The supported procedure:

1. Add the new version, `served: true`, `storage: false`. Deploy the conversion webhook first if one is needed.
2. Ship an operator binary whose scheme understands **both** versions, everywhere, before touching storage. Rolling upgrades across 40 clusters mean old and new binaries coexist.
3. Flip `storage: true` to the new version. From this moment, **new and updated objects are written in the new shape**; untouched objects still sit in etcd in the old shape.
4. Migrate the stored objects — either the [kube-storage-version-migrator](https://github.com/kubernetes-sigs/kube-storage-version-migrator), or the poor-man's version, a no-op read-modify-write of every object, which forces a rewrite at the current storage version.
5. Remove the old version from `status.storedVersions`.
6. Only now set the old version `served: false`, and later remove it entirely.

**Storage-version rollback is a database migration, not a deploy.** Once objects exist in the new shape, rolling the operator back to a build that only understands the old shape can fail to decode them. Plan a version bump with the same rollback discipline you would give a schema migration: staged, reversible up to step 3, and irreversible after step 4.

### 9. A CR is still an etcd value

Two hard ceilings, both worth quoting with provenance:

- **kube-apiserver `MaxRequestBodyBytes` defaults to 3 MiB** (`3 * 1024 * 1024`, `k8s.io/apiserver/pkg/server/config.go`), applied to create/update/patch/delete bodies.
- **etcd's `--max-request-bytes` defaults to 1.5 MiB** (`DefaultMaxRequestBytes = 1.5 * 1024 * 1024`, etcd `server/embed/config.go`). This is the effective per-object limit, and it is the smaller of the two.

The realistic way to hit it is a status-heavy CRD with an unbounded array. A `WorkloadCost` that appends one entry per reconcile to `status.history[]`, reconciling every 5 minutes, accumulates 288 entries a day; at ~200 bytes an entry that is ~57 KB/day, ~1.7 MB in a month — and **every status write after that fails**, with an error that names etcd, not your schema. Worse, each write of the whole object also costs bandwidth and etcd write amplification proportional to the *whole* object, not the delta.

Cap it in the type: `+kubebuilder:validation:MaxItems=20` plus ring-buffer semantics in the reconciler, or push detail to a metrics/warehouse backend and keep only a rolling summary in status. The API conventions doc says the same thing in general terms: status that is large or rapidly changing, or proportional in size to a collection of other resources, belongs in a separate object, "to ensure that GETs and watch remain reasonably efficient."

### 10. Server-side apply and field ownership

Server-side apply records, in `metadata.managedFields`, **which manager owns which field path**. Two managers editing disjoint fields coexist; two claiming the same field produce a conflict, which the second must either resolve or `force`.

This matters because your controller and your users write to the same objects. Rules:

- Always set a stable `client.FieldOwner("gpu-cost-operator")` on apply patches. The default derived owner changes and will fight the user's `kubectl apply`.
- **Apply only the fields you own.** If your controller applies a whole object it did not fully author, it claims ownership of every field in the payload — and the next `kubectl apply` from the user conflicts. Send a minimal apply configuration.
- Status writes go through `Status().Patch(ctx, obj, client.Apply, client.FieldOwner(...))`, which owns only status paths.
- `x-kubernetes-list-type` decides merge behaviour for lists: `atomic` means the whole list has one owner (a replacement); `map` (with `x-kubernetes-list-map-keys`) and `set` merge per element. Getting this wrong on `status.conditions` — the reason the conventions insist on `listType=map`/`listMapKey=type` — means two controllers writing different conditions clobber each other.

## Perspectives

**API-design perspective.** A CRD schema is a promise to every future caller, including future-you. "Invalid states unrepresentable" — enums plus CEL plus immutability rules — is cheaper than defensive Go re-checking the same invariants in every reconcile, and cheaper still than the incident where a bad object slipped through and corrupted status before anyone noticed. The measured version of this argument: a CEL rule with realistic bounds costs ~750 cost units, roughly 0.0075% of the per-object runtime budget, and it runs once at admission instead of once per reconcile forever.

**Operator/fleet perspective.** Across 40 clusters, a CRD schema change is a rollout problem, not a `kubectl apply`. Old and new operator binaries must both tolerate whichever API version is live during a rolling upgrade, and `status.storedVersions` is the ratchet that tells you when it is safe to drop the old one. Validation ratcheting (GA 1.33) is what makes *tightening* validation survivable mid-rollout — without it, a stricter rule would block every unrelated patch to every pre-existing violating object.

**Security/FinOps perspective.** For a cost/budget CRD, validation *is* your first fraud- and error-prevention control. A `Budget` with `hardLimit: -1` rejected at admission never becomes a wrong invoice or a bypassed enforcement decision downstream. And the quantity trap in §7 is exactly the kind of bug that fails *open*: a naive `self >= 0` rule accepts `hardLimit: 5` and blows up on `hardLimit: "5"`, which means whichever form your tests happened to use decided whether you shipped validation at all.

**Extensibility-ecosystem perspective.** CRDs are how the whole ecosystem — cert-manager, the Prometheus Operator, KubeVirt, Kueue, even DRA's own `resource.k8s.io` types — extends the API without forking Kubernetes. Your three CRDs use exactly the mechanism those production systems use; there is no separate, lesser path for "your" API versus "real" ones. That cuts both ways: you also inherit their operational hazards, including the conversion-webhook failure mode in the next section.

## Real-world use cases

- **kubernetes/kubernetes#110720 — "CRD Conversion webhook down results in controller-manager GC failure."** A real production report where an unreachable conversion webhook for a CRD (`milvus.io/v1alpha1, Kind=Milvus`) caused the garbage collector's dependency-graph builder to fail cache sync. The log signature is exact: `Failed to watch *v1.PartialObjectMetadata: failed to list *v1.PartialObjectMetadata: conversion webhook for milvus.io/v1alpha1, Kind=Milvus failed`, followed by `unable to sync caches for garbage collector` and `timed out waiting for dependency graph builder sync`. **Cascading deletion for every type stopped**, not just for that CRD, because the GC builds one graph over all types — and recovery came only when the webhook was restored. *What it shows:* a conversion webhook is a cluster-wide availability dependency, not a per-CRD one. This is the direct bridge into lesson 06's GC material.
- **Gardener, "Fix Problematic Conversion Webhooks."** SAP's multi-cluster management project ships operational guidance for exactly this failure, because a broken network path to a conversion webhook in a managed shoot cluster eventually stops kube-controller-manager's garbage collection. *What it shows:* the failure is common enough at scale that a major platform documents a runbook for it.
- **The CEL cost-budget rejection is a real deployment failure, not a theoretical one.** The three-error transcript in §7 is what a `make manifests && kubectl apply -f config/crd/bases/` prints when someone adds a rule over an unbounded list. It fails the *CRD apply*, in a maintenance window, with a message about "cost" that mentions nothing about the field the reader was thinking about. *What it shows:* the cost budget is enforced at CRD-write time, which is the least convenient time to learn about it.
- **Cloudflare, "Leveraging Kubernetes virtual machines with KubeVirt."** KubeVirt's `VirtualMachine` / `VirtualMachineInstance` CRDs in real Cloudflare production infrastructure — a large-scale CRD-as-API-extension case study to contrast against the cost operator's three CRDs, including the owner-reference relationship lesson 06 picks up.
- **kubernetes.io blog, "Enforce CRD Immutability with CEL Transition Rules."** The project's own walkthrough of the `self == oldSelf` pattern this lesson's `Budget.spec.scope` uses.

## Worked example

Design the `Budget` CRD end to end: Go types with markers, the generated CRD YAML with **every field annotated**, and the behaviour proven against the real validator.

### The Go types

```go
// api/v1alpha1/budget_types.go
package v1alpha1

import (
    "k8s.io/apimachinery/pkg/api/resource"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// Cross-field rules live on the struct, because `self` is then the whole spec.
// fieldPath makes the error blame the field a human would expect.
// +kubebuilder:validation:XValidation:rule="quantity(self.hardLimit).compareTo(quantity(self.softLimit)) >= 0",message="hardLimit must be >= softLimit",fieldPath=".hardLimit"
// +kubebuilder:validation:XValidation:rule="!has(self.window) || self.scope.matchLabels.size() > 0 || self.scope.matchExpressions.size() > 0",message="a windowed budget must select something"
type BudgetSpec struct {
    // HardLimit is the enforced ceiling for spend in the window.
    // NOTE: resource.Quantity is int-or-string in the schema, so `self >= 0`
    // would NOT work here. sign() is a GLOBAL CEL function, not a member.
    // +kubebuilder:validation:XValidation:rule="sign(quantity(self)) >= 0",message="hardLimit must be non-negative"
    HardLimit resource.Quantity `json:"hardLimit"`

    // SoftLimit triggers a warning condition before the hard cap.
    // +kubebuilder:validation:XValidation:rule="sign(quantity(self)) >= 0",message="softLimit must be non-negative"
    SoftLimit resource.Quantity `json:"softLimit"`

    // Currency of the limits; must match the referenced policy's currency.
    // Enum + default are pure OpenAPI: no CEL cost, shown by kubectl explain.
    // +kubebuilder:validation:Enum=USD;EUR;GBP
    // +kubebuilder:default=USD
    Currency string `json:"currency"`

    // Window is a rolling accounting period, e.g. "24h" or "30d".
    // MaxLength is not decoration: it bounds any future CEL rule over this field.
    // +kubebuilder:validation:Pattern=`^[0-9]+(h|d)$`
    // +kubebuilder:validation:MaxLength=8
    Window string `json:"window"`

    // Scope selects which workloads count against this budget.
    // Immutable after creation so historical spend stays attributable.
    // +kubebuilder:validation:XValidation:rule="self == oldSelf",message="scope is immutable"
    Scope metav1.LabelSelector `json:"scope"`

    // Tags are free-form. Bounded so a future CEL rule over them stays cheap:
    // 25 × (0.4×63 + 4.6) ≈ 750 cost units, vs a 1e7 per-rule limit.
    // +kubebuilder:validation:MaxItems=25
    // +kubebuilder:validation:items:MaxLength=63
    // +optional
    Tags []string `json:"tags,omitempty"`
}

type BudgetStatus struct {
    // +optional
    CurrentSpend string `json:"currentSpend,omitempty"`

    // ObservedGeneration is the .metadata.generation this status reflects.
    // Trustworthy ONLY because /status is a subresource (see Core concepts §3).
    // +optional
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`

    // Bounded on purpose: an unbounded status array is how you hit etcd's
    // 1.5 MiB per-object limit months into production.
    // +kubebuilder:validation:MaxItems=20
    // +optional
    History []SpendSample `json:"history,omitempty"`

    // listType=map + listMapKey=type is what makes this behave as a map keyed
    // by condition type for server-side apply and strategic merge patch.
    // +listType=map
    // +listMapKey=type
    // +patchStrategy=merge
    // +patchMergeKey=type
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type"`
}

type SpendSample struct {
    At    metav1.Time `json:"at"`
    Spend string      `json:"spend"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:storageversion
// +kubebuilder:resource:shortName=bud;buds,categories=cost
// +kubebuilder:printcolumn:name="Limit",type=string,JSONPath=`.spec.hardLimit`
// +kubebuilder:printcolumn:name="Spend",type=string,JSONPath=`.status.currentSpend`
// +kubebuilder:printcolumn:name="Window",type=string,JSONPath=`.spec.window`
// +kubebuilder:printcolumn:name="Ready",type=string,JSONPath=`.status.conditions[?(@.type=="Ready")].status`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`
// +kubebuilder:selectablefield:JSONPath=`.spec.currency`
type Budget struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec   BudgetSpec   `json:"spec,omitempty"`
    Status BudgetStatus `json:"status,omitempty"`
}
```

### The generated CRD, annotated line by line

`make manifests` runs `controller-gen` and emits `config/crd/bases/cost.example.com_budgets.yaml`. This is that file, complete, with a conversion stanza added for the multi-version future in §8:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # MUST be exactly <plural>.<group>. The apiserver rejects anything else.
  name: budgets.cost.example.com
  annotations:
    # controller-gen stamps the version it was built from; useful in bug reports.
    controller-gen.kubebuilder.io/version: v0.21.0
    # Injected by cert-manager's ca-injector; fills spec.conversion.webhook
    # .clientConfig.caBundle at runtime so you never commit a CA to git.
    cert-manager.io/inject-ca-from: gpu-cost-system/serving-cert
spec:
  group: cost.example.com                 # the API group; owns the URL path
  names:
    kind: Budget                          # Go type name; used in TypeMeta
    listKind: BudgetList                  # the List wrapper kind
    plural: budgets                       # the RESOURCE name — the URL segment
    singular: budget                      # what `kubectl get budget` accepts
    shortNames: [bud, buds]               # `kubectl get bud`
    categories: [cost]                    # `kubectl get cost` returns all of them
  scope: Namespaced                       # Cluster is the alternative; immutable

  # ---- multi-version conversion -------------------------------------------
  conversion:
    # None = trivial passthrough (apiVersion rewrite only). Webhook = call out.
    strategy: Webhook
    webhook:
      # The versions of ConversionReview this webhook understands. The apiserver
      # picks the first one it also supports. [v1] is correct for anything modern.
      conversionReviewVersions: ["v1"]
      clientConfig:
        # In-cluster service reference. MUST be reachable on port 443.
        service:
          namespace: gpu-cost-system
          name: gpu-cost-operator-webhook-service
          path: /convert                  # controller-runtime's default path
          port: 443
        # caBundle: PEM CA that signed the webhook's serving cert. Left empty
        # here because cert-manager's ca-injector writes it via the annotation
        # above. If you manage certs yourself, this field is mandatory.
        caBundle: ""

  versions:
  - name: v1alpha1
    served: true                          # a REST endpoint exists for this version
    storage: true                         # EXACTLY ONE version may set this
    # deprecated: true                    # would emit a Warning: header to clients
    # deprecationWarning: "cost.example.com/v1alpha1 Budget is deprecated; use v1beta1"

    # ---- subresources ---------------------------------------------------
    subresources:
      # Installs .../budgets/{name}/status. Consequences (Core concepts §3):
      #  - main-resource writes can no longer set .status
      #  - /status writes can no longer set .spec or metadata
      #  - metadata.generation increments ONLY on non-metadata change
      status: {}
      # scale: {specReplicasPath: .spec.replicas, statusReplicasPath: .status.replicas}
      #   would install .../scale so `kubectl scale` and the HPA can target this.
      #   A Budget does not scale, so it is omitted.

    # ---- what `kubectl get budgets` prints -------------------------------
    # Evaluated server-side; a wrong jsonPath yields an empty cell, not an error.
    additionalPrinterColumns:
    - name: Limit
      type: string
      jsonPath: .spec.hardLimit
    - name: Spend
      type: string
      jsonPath: .status.currentSpend
    - name: Window
      type: string
      jsonPath: .spec.window
    - name: Ready
      type: string
      # JSONPath filter syntax works here: pick the condition whose type is Ready.
      jsonPath: .status.conditions[?(@.type=="Ready")].status
    - name: Age
      type: date                          # rendered as a relative age by kubectl
      jsonPath: .metadata.creationTimestamp
    # - name: Currency
    #   type: string
    #   priority: 1                       # >0 → hidden unless `-o wide`
    #   jsonPath: .spec.currency

    # ---- server-side field selectors (GA in 1.32) -------------------------
    # Enables: kubectl get budgets --field-selector spec.currency=USD
    # and lets a controller-runtime cache filter server-side, shrinking N.
    selectableFields:
    - jsonPath: .spec.currency

    # ---- the structural schema -------------------------------------------
    schema:
      openAPIV3Schema:
        type: object                      # rule 1 of structural: everything typed
        description: Budget caps GPU spend for a selected set of workloads.
        properties:
          apiVersion: {type: string}      # controller-gen always emits these four
          kind:       {type: string}
          metadata:   {type: object}      # deliberately opaque: metadata is not
                                          # validated by the CRD schema
          spec:
            type: object
            required: [hardLimit, softLimit, window, scope]
            # Rules on the OBJECT node: `self` is the whole spec, so cross-field
            # comparisons are possible here and nowhere else.
            x-kubernetes-validations:
            - rule: "quantity(self.hardLimit).compareTo(quantity(self.softLimit)) >= 0"
              message: "hardLimit must be >= softLimit"
              # Blame the field a human would expect instead of the whole object.
              fieldPath: ".hardLimit"
              # reason: FieldValueInvalid   # default; also Forbidden/Required/Duplicate
            properties:

              hardLimit:
                description: Enforced spend ceiling for the window.
                # This trio is exactly what controller-gen emits for
                # resource.Quantity. The node is int-or-string → CEL type `dyn`,
                # which is why `self >= 0` is WRONG here (see Core concepts §7).
                x-kubernetes-int-or-string: true
                anyOf:
                - type: integer
                - type: string
                pattern: "^(\\+|-)?(([0-9]+(\\.[0-9]*)?)|(\\.[0-9]+))(([KMGTPE]i)|[numkMGTPE]|([eE](\\+|-)?(([0-9]+(\\.[0-9]*)?)|(\\.[0-9]+))))?$"
                x-kubernetes-validations:
                # sign() is a GLOBAL function (cel.Overload), not a member of
                # Quantity. `quantity(self).sign()` fails to COMPILE.
                - rule: "sign(quantity(self)) >= 0"
                  message: "hardLimit must be non-negative"

              softLimit:
                description: Spend level at which a warning condition is raised.
                x-kubernetes-int-or-string: true
                anyOf:
                - type: integer
                - type: string
                pattern: "^(\\+|-)?(([0-9]+(\\.[0-9]*)?)|(\\.[0-9]+))(([KMGTPE]i)|[numkMGTPE]|([eE](\\+|-)?(([0-9]+(\\.[0-9]*)?)|(\\.[0-9]+))))?$"
                x-kubernetes-validations:
                - rule: "sign(quantity(self)) >= 0"
                  message: "softLimit must be non-negative"

              currency:
                type: string
                # Pure OpenAPI: zero CEL cost, and `kubectl explain` prints it.
                enum: [USD, EUR, GBP]
                # Applied on request, on read from etcd, and after conversion —
                # but NOT written back to etcd (Core concepts §5).
                default: USD

              window:
                type: string
                pattern: "^[0-9]+(h|d)$"
                # Bounds every future CEL rule over this field. Cheap insurance.
                maxLength: 8

              scope:
                # metav1.LabelSelector expands in full; abbreviated here.
                type: object
                properties:
                  matchLabels:
                    type: object
                    additionalProperties: {type: string}
                  matchExpressions:
                    type: array
                    items:
                      type: object
                      required: [key, operator]
                      properties:
                        key:      {type: string}
                        operator: {type: string}
                        values:
                          type: array
                          items: {type: string}
                          x-kubernetes-list-type: atomic
                    x-kubernetes-list-type: atomic
                x-kubernetes-map-type: atomic
                x-kubernetes-validations:
                # A TRANSITION rule: mentioning oldSelf makes it update-only, and
                # it is skipped on create. Set optionalOldSelf: true to run it on
                # create too (oldSelf becomes a CEL optional).
                - rule: "self == oldSelf"
                  message: "scope is immutable"

              tags:
                type: array
                # maxItems + items.maxLength are the ONLY reason a future
                # `self.all(t, t.contains('gpu'))` rule fits the cost budget:
                # 25 × (0.4×63 + 4.6) ≈ 750 units against a 10,000,000 limit.
                maxItems: 25
                items:
                  type: string
                  maxLength: 63
                x-kubernetes-list-type: set   # dedup + order-insensitive equality

          status:
            type: object
            properties:
              currentSpend: {type: string}
              observedGeneration:
                type: integer
                format: int64
              history:
                type: array
                # Bounded so a long-lived Budget cannot grow past etcd's
                # 1.5 MiB per-object ceiling one reconcile at a time.
                maxItems: 20
                items:
                  type: object
                  required: [at, spend]
                  properties:
                    at:    {type: string, format: date-time}
                    spend: {type: string}
                x-kubernetes-list-type: atomic
              conditions:
                type: array
                # map + listMapKey=type: this list is semantically a map keyed by
                # condition type. Without it, two controllers writing different
                # conditions clobber each other under server-side apply.
                x-kubernetes-list-type: map
                x-kubernetes-list-map-keys: [type]
                items:
                  type: object
                  required: [type, status, lastTransitionTime, reason, message]
                  properties:
                    type:
                      type: string
                      maxLength: 316
                      pattern: "^([a-z0-9]([-a-z0-9]*[a-z0-9])?(\\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*/)?(([A-Za-z0-9][-A-Za-z0-9_.]*)?[A-Za-z0-9])$"
                    status:
                      type: string
                      enum: ["True", "False", "Unknown"]
                    observedGeneration:
                      type: integer
                      format: int64
                      minimum: 0
                    lastTransitionTime: {type: string, format: date-time}
                    reason:
                      type: string
                      maxLength: 1024
                      minLength: 1
                      pattern: "^[A-Za-z]([A-Za-z0-9_,:]*[A-Za-z0-9_])?$"
                    message:
                      type: string
                      maxLength: 32768
```

### Proving the behaviour

You do not need a cluster to test CEL. `k8s.io/apiextensions-apiserver/pkg/test` loads a CRD file and hands you the real validator, so this belongs in `make test` alongside envtest:

```go
func TestBudgetCEL(t *testing.T) {
    v, err := test.VersionValidatorFromFile(t,
        "../../config/crd/bases/cost.example.com_budgets.yaml", "v1alpha1")
    require.NoError(t, err)

    ok := func(spec map[string]any) {
        require.Empty(t, v(map[string]any{"spec": spec}, nil))
    }
    rejects := func(spec map[string]any, substr string) {
        errs := v(map[string]any{"spec": spec}, nil)
        require.NotEmpty(t, errs)
        require.Contains(t, errs.ToAggregate().Error(), substr)
    }

    ok(map[string]any{"hardLimit": "5000", "softLimit": "4000", "window": "24h"})
    rejects(map[string]any{"hardLimit": "100", "softLimit": "200", "window": "24h"},
        "hardLimit must be >= softLimit")
    rejects(map[string]any{"hardLimit": "-5", "softLimit": "-10", "window": "24h"},
        "must be non-negative")
}
```

Actual output from running that validator against the schema above (apiextensions-apiserver v0.36.3):

```
=== {"hardLimit":"5000","softLimit":"4000","window":"24h"}
    ACCEPTED
=== {"hardLimit":"100","softLimit":"200","window":"24h"}
    REJECTED: spec.hardLimit: Invalid value: hardLimit must be >= softLimit
=== {"hardLimit":"-5","softLimit":"-10","window":"24h"}
    REJECTED: spec.hardLimit: Invalid value: "-5": hardLimit must be non-negative
    REJECTED: spec.softLimit: Invalid value: "-10": softLimit must be non-negative
=== {"hardLimit":"1500m","softLimit":"1","window":"24h"}
    ACCEPTED                    ← 1500m = 1.5 ≥ 1
=== {"hardLimit":"500m","softLimit":"1","window":"24h"}
    REJECTED: spec.hardLimit: Invalid value: hardLimit must be >= softLimit
                                ← 500m = 0.5 < 1; a string compare says otherwise
```

None of that is Go you wrote and tested. It is the schema doing its job, and the test above proves the schema does it.

### The conversion decision, worked

Three hypothetical `v1alpha1 → v1beta1` changes:

| Change | Passthrough OK? | Why |
|---|---|---|
| (a) Add optional `notifyChannel` with a default | **Yes — `None`** | Old objects are valid under the new schema unmodified; the new field defaults on read. |
| (b) Rename `hardLimit` → `hardCapUSD` | **No — webhook** | Passthrough leaves old objects with no `hardCapUSD` and a `hardLimit` the new schema prunes. Silent data loss. |
| (c) Split `window: "24h"` into `windowValue: 24` + `windowUnit: "h"` | **No — webhook** | Converting requires parsing the old string; converting back requires reassembling it. |
| (d) Change `hardLimit`'s meaning from USD to cents, same name and type | **No — and no webhook saves you** | Passthrough validates *shape*, not *semantics*. This is a rename in disguise; give the field a new name so a webhook can convert it. |

Case (c), with controller-runtime's `conversion.Convertible` interface — the hub-and-spoke form kubebuilder generates, where the storage version is the hub:

```go
// api/v1alpha1/budget_conversion.go — v1alpha1 is a SPOKE; v1beta1 is the HUB.
func (src *Budget) ConvertTo(dstRaw conversion.Hub) error {
    dst := dstRaw.(*v1beta1.Budget)
    dst.ObjectMeta = src.ObjectMeta
    value, unit, err := parseWindow(src.Spec.Window) // "24h" -> (24, "h")
    if err != nil {
        // Returning an error here fails the API request that triggered the
        // conversion — including LIST and WATCH. Never fail on data you
        // already accepted; validate it at admission instead.
        return fmt.Errorf("converting spec.window %q: %w", src.Spec.Window, err)
    }
    dst.Spec.WindowValue, dst.Spec.WindowUnit = value, unit
    dst.Spec.HardLimit, dst.Spec.SoftLimit = src.Spec.HardLimit, src.Spec.SoftLimit
    dst.Spec.Currency, dst.Spec.Scope = src.Spec.Currency, src.Spec.Scope
    dst.Status = v1beta1.BudgetStatus(src.Status)
    return nil
}

func (dst *Budget) ConvertFrom(srcRaw conversion.Hub) error {
    src := srcRaw.(*v1beta1.Budget)
    dst.ObjectMeta = src.ObjectMeta
    dst.Spec.Window = fmt.Sprintf("%d%s", src.Spec.WindowValue, src.Spec.WindowUnit)
    // ... copy remaining fields ...
    return nil
}
```

Both directions must round-trip losslessly through the storage version. The standard test is a property test: generate a random `v1alpha1`, convert to `v1beta1` and back, and require equality. If a field cannot survive the round trip, you must either keep it in the storage version or accept documented data loss — and "accept" means writing it in the deprecation notes, not discovering it in an incident. Remember the metadata rule from §8: your converter may touch `labels` and `annotations` and nothing else in `metadata`.

## Practice

Advance `gpu-cost-operator`: design and scaffold all three v1alpha1 CRDs.

1. Scaffold the group and types: `kubebuilder create api --group cost --version v1alpha1 --kind GPUCostPolicy` (repeat for `WorkloadCost`, `Budget`), generating both resource and controller.
2. **`GPUCostPolicy`** — a price model. `spec.rates[]` mapping GPU class (`a100`, `h100`, …) to price-per-GPU-hour as a `resource.Quantity`, plus `spec.currency` (enum, defaulted). CEL: every rate non-negative **using `sign(quantity(...))`, not `self >= 0`** — verify by feeding both an integer and a quoted string. `/status` subresource with a `Ready` condition and `observedGeneration`. Bound `rates` with `maxItems`.
3. **`WorkloadCost`** — status-heavy. `spec.workloadRef` + `spec.policyRef`. Status carries `observedGPUHours`, `computedSpend`, `observedGeneration`, conditions, and a **bounded** `history[]` (`maxItems`). `/status` subresource, printer columns for spend and GPU-hours, and a `selectableFields` entry on whatever you will filter by.
4. **`Budget`** — as in the worked example: hard/soft limits, currency enum with default, window pattern, immutable scope selector, `hardLimit >= softLimit` cross-field CEL with `fieldPath`, non-negative limits.
5. Run `make manifests generate` and `kubectl apply -f config/crd/bases/`. Read the generated YAML for one type end to end and be able to point at the schema node each marker produced.
6. **Write the CEL unit test** (`k8s.io/apiextensions-apiserver/pkg/test`) as shown in the worked example, so validation is covered by `make test` without a cluster.
7. **Deliberately blow the cost budget once.** Add `x-kubernetes-validations` with `self.all(t, t.contains('gpu'))` on an *unbounded* `tags` array, run `kubectl apply -f config/crd/bases/`, and record the exact three-part error. Then add `maxItems`/`maxLength` and watch it pass. You want to have seen this message before you see it in a maintenance window.
8. **Prove pruning.** Apply a CR with a typo'd field name, `kubectl get -o yaml` it, and confirm the field is simply gone. Then repeat with `--validate=strict` and record the difference.

**Acceptance:**

- `kubectl apply` of a valid instance of each kind **succeeds**.
- `kubectl apply` of a `Budget` with `hardLimit: -1` **fails** with your CEL message (proves admission-time rejection, not controller-time), for **both** `-1` and `"-1"`.
- `kubectl apply` of a `Budget` with `softLimit > hardLimit` **fails** on the object-level rule, and the error names `spec.hardLimit` (proves `fieldPath`).
- `kubectl explain budget.spec` shows your fields and constraints; `kubectl get budget` shows your printer columns; `kubectl get budgets --field-selector spec.currency=USD` works.
- Editing a `Budget`'s `scope` after creation is rejected as immutable.
- A committed note recording the pruning experiment and the cost-budget error text.

This directly produces the checkpoint's item 5 ("CRDs reject invalid specs via CEL … at `kubectl apply`") and lays the schema every later lesson's controller, RBAC, and webhook work builds on.

## Common pitfalls

1. **Writing `self >= 0` or `+kubebuilder:validation:Minimum=0` on a `resource.Quantity`.** *Symptom:* validation appears to work in tests and then rejects real input with `'no such overload'`, or silently does nothing. *Mechanism:* `controller-gen` renders `Quantity` as `x-kubernetes-int-or-string`, which CEL types as `dyn` and OpenAPI treats as neither numeric nor string. Integers happen to work; every string form fails. Use `sign(quantity(self)) >= 0` and `quantity(a).compareTo(quantity(b))` — and remember `sign` is a global function, not a member.
2. **Believing OpenAPI markers can express cross-field constraints.** *Symptom:* someone adds `Minimum` to `softLimit` and thinks the ordering is enforced. *Mechanism:* an OpenAPI keyword sees one schema node in isolation; it has no access to sibling fields or to the prior value. Anything relating two fields (`hardLimit >= softLimit`) or referencing history (`self == oldSelf`) must be CEL, declared on the parent node so `self` is the object.
3. **Relying on an undeclared field surviving.** *Symptom:* "the operator ignores my config"; the field is absent from `kubectl get -o yaml`. *Mechanism:* structural-schema pruning deletes it before persistence, without an error. Declare everything; use `--validate=strict` or server-side apply with `fieldValidation=Strict` in CI so typos fail loudly.
4. **Forgetting that the `/status` RBAC grant is separate.** *Symptom:* CI is green, production logs `Forbidden` on every status write. *Mechanism:* `budgets` and `budgets/status` are distinct resources for RBAC, and envtest does not enforce RBAC by default. Lesson 07 fixes this; expect it.
5. **Shipping an "additive-looking" version bump that is secretly a semantic change.** *Symptom:* every existing object's meaning changes and nothing errors. *Mechanism:* passthrough conversion validates shape, not semantics — changing `hardLimit` from USD to cents keeps the same name and type, so nothing objects. Rename the field so a converter can do the arithmetic.
6. **A CEL rule with an unbounded loop that passes review and fails the CRD apply.** *Symptom:* `make manifests && kubectl apply -f config/crd/bases/` fails with three "cost" errors and no mention of what you were thinking about. *Mechanism:* the estimator assumes the worst case for anything iterable without `maxItems`/`maxLength`/`maxProperties`, and the per-rule limit is 10,000,000 units. Add bounds; avoid nested lists under `self.all(...)`, where costs multiply.
7. **An unbounded status array.** *Symptom:* months in, one long-lived object's status writes start failing and only that object is affected. *Mechanism:* each write persists the whole object, and etcd's default `--max-request-bytes` is 1.5 MiB. Cap with `maxItems` and ring-buffer semantics, or move detail out of status entirely.
8. **Applying a whole object with server-side apply from your controller.** *Symptom:* users' `kubectl apply` starts reporting conflicts on fields they own. *Mechanism:* an apply claims ownership of every field in the payload. Send a minimal apply configuration under a stable `FieldOwner`, and write status through `Status().Patch(..., client.Apply, ...)`.
9. **Treating a conversion webhook as a per-CRD concern.** *Symptom:* the webhook goes down and cascading deletion stops across the cluster. *Mechanism:* kubernetes/kubernetes#110720 — the garbage collector builds one dependency graph over *all* types, so a single type it cannot list wedges the whole graph builder's cache sync. Lesson 06 goes into this.

## Self-check

**(a) Why is `/status` a subresource, and what does it change about updates and `observedGeneration`?**

**Answer:** Declaring `subresources.status: {}` installs a second endpoint, `.../budgets/{name}/status`, and splits the write path in the apiserver's registry strategies. On a **main-resource** update, `customResourceStrategy.PrepareForUpdate` copies the *old* `status` over whatever you sent, so a plain `Update()` carrying computed status silently drops it. On a **`/status`** update, `statusStrategy.PrepareForUpdate` deep-copies the entire old object over the new one and writes back only `status` (plus `managedFields`), so a status write physically cannot change `spec` or `metadata`. Two payoffs. First, RBAC and code cleanly separate desired state (users write `spec`) from observed state (only the controller writes `status`, via `Status().Update()`/`Status().Patch()`). Second, generation: the strategy increments `metadata.generation` **only when something outside `metadata` differs** — and since status is restored from the old object before that comparison, status writes never bump it, while spec changes always do. That is what makes `status.observedGeneration == metadata.generation` a trustworthy "I have reconciled the current spec" signal. Without the subresource, every status write would bump generation and the signal would be pure noise. Bonus detail: metadata edits (labels, annotations, finalizers) do not bump generation either, which is why lesson 06's finalizer `Update()` is generation-neutral.

**(b) When do you need a conversion webhook versus a new version with defaulting?**

**Answer:** A new version with `conversion.strategy: None` is sufficient when the change is **additive and passthrough-compatible**: adding an optional field, giving it a schema default, or tightening validation that existing objects already satisfy (and validation ratcheting, GA in 1.33, makes that last one safe by ignoring errors on unchanged fields). The apiserver then only rewrites `apiVersion`. You need `strategy: Webhook` when converting requires **reading old fields to compute new ones**: a rename, a restructure, a split or merge, a unit change. The webhook must handle both directions and round-trip losslessly through the single storage version, or clients pinned to the other version see corruption. Constraints to know: the webhook receives a *batch* of objects in one `ConversionReview` and must return them in order; the service must be exposed on port 443 over https; and it must not mutate anything in `metadata` other than `labels` and `annotations` — changes to `name`, `uid` or `namespace` fail the request. The heuristic: *can I express this migration as "new optional field + default"?* Yes → no webhook. No → webhook. And beware the case neither handles: changing a field's **meaning** while keeping its name and type. Passthrough validates shape, not semantics, so that silently reinterprets every existing object; rename the field instead.

**(c) What can CEL validate that plain OpenAPI cannot — give an example — and what stops a CEL rule from being expensive?**

**Answer:** OpenAPI keywords constrain one schema node in isolation: `enum`, `minimum`, `maxLength`, `pattern`, `required`. CEL (`x-kubernetes-validations`) binds `self` to the node the rule is attached to — so a rule on the `spec` **object** sees every field — and binds `oldSelf` to the previous value, making it a transition rule. That buys **cross-field** constraints (`quantity(self.hardLimit).compareTo(quantity(self.softLimit)) >= 0`), **immutability** (`self == oldSelf`, which OpenAPI has no concept of), **conditional requirements** (`has(self.window) || self.scope == 'cluster'`), list uniqueness by key, and quantity- and semver-aware comparison via the Kubernetes CEL libraries. Cost is bounded four ways, all with concrete numbers. At **CRD write time** the apiserver estimates worst-case cost and rejects the CRD if any single expression exceeds `StaticEstimatedCostLimit = 10,000,000` or the whole schema exceeds `StaticEstimatedCRDCostLimit = 100,000,000`. At **CR validation time** each expression is halted if it exceeds `PerCallLimit = 1,000,000` cost units (≈ 0.1 s of CPU) and all rules together are halted at `RuntimeCELCostBudget = 10,000,000` (≈ 1 s). The estimator assumes the worst case for anything iterable, so the fix for a rejection is to add `maxItems`/`maxLength`/`maxProperties`, not to simplify the rule. Measured on 1.36: `self.all(x, x.contains('gpu'))` costs roughly `0.4 × maxLength + 4.6` units per item, so with `maxItems: 25` and `maxLength: 63` the rule costs about 750 units — four orders of magnitude under the limit. Nested lists multiply, which is why an unbounded `[]int` with `self.all(x, x == 5)` is accepted but the same rule over `[][]int` is not.

**(d) You need to rename `Budget.spec.limit` to `Budget.spec.hardLimitUSD` across 40 clusters. Walk the exact steps.**

**Answer:** A rename cannot be passthrough-converted, so this needs a conversion webhook plus a staged storage migration. (1) Add `v1beta1` with `hardLimitUSD` replacing `limit`, `served: true`, `storage: false`; keep `v1alpha1` as the storage version so rollback stays safe. (2) Implement and deploy the conversion webhook — `ConvertTo`/`ConvertFrom` copying `limit ↔ hardLimitUSD` — exposed on port 443 with a valid `caBundle` (cert-manager's `ca-injector` is the usual way), and confirm it handles batched `ConversionReview` requests. Test the round trip as a property test before it ever runs in a cluster. (3) Roll out an operator binary whose scheme understands **both** versions to every cluster, before touching storage, because rolling upgrades mean old and new binaries coexist. (4) Flip `storage: true` to `v1beta1`. New and updated objects are now persisted in the new shape; untouched ones are not, and `status.storedVersions` now contains both. **This is the point of no easy return**: an operator rolled back to a build that only knows `v1alpha1` may fail to decode the new stored shape. (5) Migrate stored objects — kube-storage-version-migrator, or a no-op read-modify-write of every object — then remove `v1alpha1` from `status.storedVersions`. (6) Set `v1alpha1` `deprecated: true` with a `deprecationWarning` so clients get a `Warning:` header, and only after the fleet has drained set `served: false`, then remove the version entirely on a separate rollout. Throughout, users applying `v1alpha1` YAML keep working unmodified because the webhook translates transparently — and if the webhook is ever unreachable, remember #110720: it does not just break admission, it can stall cluster-wide garbage collection.

## Connections & what's next

CRD design is the schema half of the contract; lesson 04's informer/workqueue machinery is the mechanism that watches whatever this lesson defines, and lesson 06's controller-runtime machinery is what reads and writes it — `Status().Update()` behaves the way it does *only because* of the `/status` subresource declared here, and `metadata.generation` is a usable signal *only because* of the registry strategy in §3. The bounded status arrays here are the same discipline as lesson 04's cache-memory arithmetic, one layer up: `N × S` is a budget whether the objects live in etcd or in your informer's heap. The RBAC lesson (07) needs the separate `/status` and `/finalizers` subresource grants this lesson's schema implies. The admission-webhooks lesson (08) picks up exactly where CEL stops — anything requiring a lookup of *another* object, or more compute than the cost budget allows, has to be a validating webhook or a `ValidatingAdmissionPolicy`. Next: **lesson 06** takes these three CRDs and wires the controller-runtime Manager, client, owner references, and finalizers that turn them into a running, restart-safe operator — and shows how a conversion-webhook outage from §8 silently stalls the garbage collector.

## References & further reading

**Primary sources**

- kubernetes.io — Extend the Kubernetes API with CustomResourceDefinitions — https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/ — the structural-schema rules quoted in §2, pruning, defaulting (including "defaults applied when reading data from etcd are not automatically written back"), printer columns, subresources, and the CEL resource-budget worked examples this lesson measures.
- kubernetes.io — CustomResourceDefinition versioning — https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/ — served vs storage versions, when conversion is invoked, the port-443/https requirements, permissible metadata mutations, `status.storedVersions`, and the six-step migration procedure in §8.
- kubernetes.io — Common Expression Language in Kubernetes — https://kubernetes.io/docs/reference/using-api/cel/ — the cost-unit model (comparison = 1, list literal = 40, regex ≈ `len(regex)×len(input)`), runtime cost budget vs estimated cost limits, and the library reference. *Correction noted:* this page lists `sign()` among a Quantity's **member** functions; the implementation registers it as a global `cel.Overload`, so `quantity(x).sign()` does not compile — use `sign(quantity(x))`.
- `k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1` godoc — https://pkg.go.dev/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1 — the Go structs behind every YAML key in the worked example: `CustomResourceDefinitionVersion`, `CustomResourceValidation`, `CustomResourceSubresources`, `CustomResourceConversion`, `SelectableField`, and `ValidationRule` (all six fields).
- `k8s.io/apiextensions-apiserver/pkg/registry/customresource` (v0.36.3) — `strategy.go` and `status_strategy.go` are ~40 lines that fully define §3: `PrepareForCreate` deleting status and setting generation 1, `PrepareForUpdate` restoring old status and the `copyNonMetadata` generation rule, and `statusStrategy` deep-copying the old object.
- `k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/validation` (v0.36.3) — `StaticEstimatedCostLimit = 10000000`, `StaticEstimatedCRDCostLimit = 100000000`, and `getCostErrorMessage`, which is where the "exceeds budget by factor of more than 100x" wording comes from.
- `k8s.io/apiserver/pkg/apis/cel/config.go` (v0.36.3) — `PerCallLimit = 1000000` ("roughly 0.1 second for each expression validation call"), `RuntimeCELCostBudget = 10000000` ("roughly 1 second"), `MaxRequestSizeBytes = 3 MiB`.
- `k8s.io/apiextensions-apiserver/pkg/features/kube_features.go` (v0.36.3) — the exact graduation table: `CRDValidationRatcheting` alpha 1.28 → beta 1.30 → GA-locked 1.33; `CustomResourceFieldSelectors` alpha 1.30 → beta 1.31 → GA-locked 1.32; `CRDObservedGenerationTracking` beta 1.35 → default-on 1.36.
- controller-tools v0.21.0 — `pkg/crd/markers/validation.go`, `pkg/crd/markers/crd.go`, `pkg/crd/known_types.go` — the authoritative marker list in §6, the `XValidation` marker's six fields, and the `resource.Quantity → x-kubernetes-int-or-string + anyOf + pattern` mapping that causes the trap in §7. This replaces the Kubebuilder Book's marker page as ground truth when the two disagree.
- Kubernetes API conventions — https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md — the spec/status contract, the `metav1.Condition` schema and its list markers, condition naming and polarity rules, `observedGeneration`, and the guidance to keep large or fast-changing status in separate objects.
- Server-Side Apply — https://kubernetes.io/docs/reference/using-api/server-side-apply/ — `managedFields`, field ownership, conflicts and `force`, and how `x-kubernetes-list-type` decides merge behaviour.

**Real-world engineering sources — with what each shows**

- kubernetes/kubernetes#110720 — https://github.com/kubernetes/kubernetes/issues/110720 — a down conversion webhook stalling kube-controller-manager's garbage collector cluster-wide; the exact `unable to sync caches for garbage collector` / `timed out waiting for dependency graph builder sync` signature.
- Gardener, "Fix Problematic Conversion Webhooks" — https://gardener.cloud/docs/guides/administer-shoots/conversion-webhook/ — production runbook from a large multi-cluster platform for the same failure.
- Google Open Source Blog, "Kubernetes CRD Validation Using CEL" — https://opensource.googleblog.com/2023/11/kubernetes-crd-validation-using-cel.html — the official introduction to CEL validation rules for CRDs.
- kubernetes.io blog, "Enforce CRD Immutability with CEL Transition Rules" — https://kubernetes.io/blog/2022/09/29/enforce-immutability-using-cel/ — the `self == oldSelf` pattern used by `Budget.spec.scope`.
- Cloudflare Blog, "Leveraging Kubernetes virtual machines at Cloudflare with KubeVirt" — https://blog.cloudflare.com/leveraging-kubernetes-virtual-machines-with-kubevirt/ — KubeVirt's CRDs as a large-scale production API-extension case study.

**Deeper dives**

- kube-storage-version-migrator — https://github.com/kubernetes-sigs/kube-storage-version-migrator — the supported tool for step 5 of the version-migration procedure.
