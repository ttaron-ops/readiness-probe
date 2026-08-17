---
lesson: "02.2"
title: "API machinery"
module: "02"
concept: "API machinery"
status: not-started
est_time: "10h"
prev: "01-component-internals.md"
next: "03-reconciliation-model.md"
artifacts: []
sources: 14
---

# 02.2 · API machinery

> **Concept.** The type system, addressing, and serialization layer under every client, informer, and CRD — GVK vs GVR, the RESTMapper, the scheme and codecs, typed vs unstructured access, and watch/resourceVersion semantics.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Lesson 01 established the structural fact everything else in this module builds on: every component is a client of the apiserver, and the apiserver is the only client of etcd. That lesson treated "a client of the apiserver" as a black box — some code somewhere calls a REST endpoint and gets an object back. This lesson opens that box. Every client you will ever write — a typed clientset call, a dynamic-client scan, an informer, a webhook handler — is `apimachinery` types wearing a different hat: the same GVK/GVR addressing, the same scheme-driven codecs, the same watch/`resourceVersion` protocol. Once you know this layer, you can read the source of client-go, controller-runtime, Helm, or `kubectl` itself and recognise the same pattern every time, instead of learning each tool's API surface from scratch.

## Why this matters

This is the driest lesson in the module and the one everything else stands on. Informers, clients, CRDs, admission — all of it is `apimachinery` types wearing different hats, and you cannot debug any of them from the outside if you do not know how an object is addressed, typed, and serialised underneath. It is also a reliable interview filter: "what's the difference between a GVK and a GVR, and who resolves one to the other?" separates people who *use* controllers from people who *build* them — it is a checkpoint probe for this module for exactly that reason.

Your capstone operator registers three CRDs into a scheme and reads Nodes and Pods through both typed and dynamic clients. You will touch every concept here on day one of the operator, and get it wrong — a stale RESTMapper, a `410 Gone` treated as an alert-worthy error, a mutated cache object, a `kubectl patch --type merge` that silently deletes a container — in ways that look like "impossible" bugs if you do not know this layer. The three-way patch comparison in this lesson is not academic: choosing the wrong patch type is one of the most common ways an operator destroys production state, and it does it quietly.

## What's new here (calibration)

A CKA holder already thinks in `kubectl get pods -n foo`, knows `apiVersion: apps/v1`, `kind: Deployment`, has run `kubectl api-resources` and seen `verbs`, `shortnames`, `namespaced`. That is the *operator's* view: the API as a REST surface you address with a CLI. This lesson does **not** re-teach that surface. What it adds is the machinery underneath the CLI:

- The `apiVersion`/`kind` you type is a **GVK** (a *type*); the plural lowercase `deployments` in the URL path is a **GVR** (a *resource*); a **RESTMapper** converts between them, and it is discovery-driven, not hardcoded.
- Every object your code holds is a `runtime.Object` decoded by a **codec** against a **scheme** — the registry mapping Go structs to GVKs.
- `metadata` is not boilerplate. `resourceVersion`, `generation`, `uid`, `finalizers`, `ownerReferences`, and `managedFields` are each a distinct protocol with distinct writers, and your controller reads or writes most of them.
- `kubectl get -w` is a **watch** stream keyed by **resourceVersion**, and the `410 Gone` you have maybe seen in logs is a first-class protocol event, not a random error.
- There are three patch formats plus server-side apply, they produce **different results from the same patch bytes**, and knowing which is which is load-bearing.
- Your CRD is not special — installing it just adds rows to discovery so the RESTMapper can resolve its GVK↔GVR like any built-in — and that discovery is cached, with a real refresh path you will hit in practice.

Version note: field semantics below are from the current API conventions and the `k8s.io/apimachinery` types on master (verified August 2026; Kubernetes v1.36 is current stable). Command transcripts are **representative** — real in format and in the values a real cluster would produce — rather than literal captures, except where attributed.

## Core concepts

### An object has four separate identities

Before any of the machinery makes sense, separate four things that beginners collapse into one:

| Identity | Example | Lives where | Answers |
|---|---|---|---|
| **Type** (GVK) | `apps/v1, Kind=Deployment` | `apiVersion` + `kind` fields; the scheme | *What kind of thing is this?* |
| **Resource** (GVR) | `apps/v1, Resource=deployments` | the URL path; discovery | *What endpoint serves it?* |
| **Instance** | `finops/gpu-cost-exporter` | `metadata.namespace` + `metadata.name` | *Which one?* |
| **Incarnation** | `uid: 6f7c…` | `metadata.uid` | *Which one, across delete-and-recreate?* |

Everything in this lesson is about the machinery that maps between these. The scheme maps Go struct ↔ GVK. The RESTMapper maps GVK ↔ GVR. The codec maps GVK + bytes ↔ Go struct. `metadata` carries instance and incarnation identity plus the protocol state (versioning, ownership, deletion) that controllers coordinate through.

```
   YOUR GO CODE                      apimachinery                        THE WIRE
   ────────────                      ────────────                        ────────

   &appsv1.Deployment{…}
        │                       ┌──────────────────────┐
        │  "what GVK is this?"  │   runtime.Scheme     │
        ├──────────────────────▶│  Go type ↔ GVK       │
        │                       │  + conversions       │
        │                       │  + DeepCopy funcs    │
        │                       └──────────┬───────────┘
        │                                  │ apps/v1, Kind=Deployment
        │                                  ▼
        │                       ┌──────────────────────┐
        │  "which URL?"         │    RESTMapper        │      GET /apis / discovery
        ├──────────────────────▶│  GVK ↔ GVR + scope   │◀────────────────────────────
        │                       └──────────┬───────────┘      (APIGroupDiscoveryList)
        │                                  │ apps/v1, Resource=deployments,
        │                                  │ scope=namespace
        │                                  ▼
        │                       ┌──────────────────────┐
        │  "make it bytes"      │   Codec (serializer  │   PUT /apis/apps/v1/namespaces/
        └──────────────────────▶│   from the scheme)   │──▶ finops/deployments/gpu-cost-exporter
                                │  JSON / YAML / proto │    Content-Type: application/json
                                └──────────┬───────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │  kube-apiserver │──▶ (lesson 01's pipeline) ──▶ etcd
                                  └─────────────────┘

   Decode is the same picture run backwards: bytes → read apiVersion+kind →
   scheme.New(gvk) allocates the Go struct → unmarshal into it.
```

### GVK vs GVR

A **GroupVersionKind** names a *type*: `{Group: "apps", Version: "v1", Kind: "Deployment"}`. It is what is stamped into an object's `apiVersion` and `kind` fields, and it is what a Go struct is registered as in a scheme.

A **GroupVersionResource** names a *REST resource*: `{Group: "apps", Version: "v1", Resource: "deployments"}`. It is the path segment the apiserver serves, and it is what a dynamic client addresses.

They are not the same thing, and — this is the part people get wrong — the mapping between them is **data returned by the server**, not a rule you can compute. There *is* a guess function in apimachinery, and its name tells you how much to trust it:

```go
// k8s.io/apimachinery/pkg/api/meta.UnsafeGuessKindToResource
var unpluralizedSuffixes = []string{"endpoints"}

func UnsafeGuessKindToResource(kind schema.GroupVersionKind) (plural, singular schema.GroupVersionResource) {
    singularName := strings.ToLower(kind.Kind)
    singular = kind.GroupVersion().WithResource(singularName)
    for _, skip := range unpluralizedSuffixes {
        if strings.HasSuffix(singularName, skip) {
            return singular, singular          // Endpoints → "endpoints", not "endpointses"
        }
    }
    switch string(singularName[len(singularName)-1]) {
    case "s":
        return kind.GroupVersion().WithResource(singularName + "es"), singular   // Ingress → ingresses
    case "y":
        return kind.GroupVersion().WithResource(strings.TrimSuffix(singularName, "y") + "ies"), singular
                                                                                 // NetworkPolicy → networkpolicies
    }
    return kind.GroupVersion().WithResource(singularName + "s"), singular         // Pod → pods
}
```

The `unpluralizedSuffixes` allowlist exists precisely because the rule is wrong, and the source comment is blunt about *why* it exists at all: some code is lazy and does not use the RESTMapper the way it should. More decisively, a CRD author picks the plural by hand in `spec.names.plural` — nothing forces `GPUCostPolicy` to become `gpucostpolicies` rather than `costpolicies`. **The only authority on GVK↔GVR is discovery.**

#### What discovery actually returns

Ask the server directly. This is the raw discovery document for the `apps/v1` group-version:

```
$ kubectl get --raw /apis/apps/v1 | jq '{groupVersion, resources: [.resources[] | select(.name|startswith("deployments"))]}'
{
  "groupVersion": "apps/v1",
  "resources": [
    {
      "name": "deployments",
      "singularName": "deployment",
      "namespaced": true,
      "kind": "Deployment",
      "verbs": ["create","delete","deletecollection","get","list","patch","update","watch"],
      "shortNames": ["deploy"],
      "categories": ["all"],
      "storageVersionHash": "8aSe+NMegvE="
    },
    {
      "name": "deployments/scale",
      "singularName": "",
      "namespaced": true,
      "group": "autoscaling",
      "version": "v1",
      "kind": "Scale",
      "verbs": ["get","patch","update"]
    },
    {
      "name": "deployments/status",
      "singularName": "",
      "namespaced": true,
      "kind": "Deployment",
      "verbs": ["get","patch","update"]
    }
  ]
}
```

Read that carefully, because six separate facts are encoded in it:

| Field | Value here | What it tells the machinery |
|---|---|---|
| `name` | `deployments` | the GVR's `Resource` — the URL segment |
| `singularName` | `deployment` | what `kubectl get deployment` resolves through |
| `namespaced` | `true` | `RESTScopeNamespace` — the URL needs `/namespaces/<ns>/` |
| `kind` | `Deployment` | the GVK's `Kind` — this is the GVK↔GVR link |
| `verbs` | includes `watch` | whether an informer is even possible for this resource |
| `shortNames` | `["deploy"]` | `kubectl get deploy` |
| `categories` | `["all"]` | `kubectl get all` |
| `storageVersionHash` | opaque | equality-only; changes when the storage version changes |

Subresources appear as separate rows with a `/`: `deployments/status` has only `get`/`patch`/`update` — **no `create`, no `delete`, no `watch`**. That is why `Status().Update()` exists as a distinct call and why you cannot watch a status subresource. And `deployments/scale` has a *different group and version* (`autoscaling/v1`) and a different `kind` (`Scale`) than its parent — which is exactly how the HorizontalPodAutoscaler can scale a Deployment, a StatefulSet, and your CRD through one uniform interface.

The core group (`""`) lives under `/api/v1` rather than `/apis/…`, which is a historical wart you will trip over when constructing URLs by hand:

```
$ kubectl get --raw /api/v1 | jq '.resources[] | select(.name=="nodes" or .name=="pods")'
{
  "name": "nodes",
  "singularName": "node",
  "namespaced": false,                       <-- cluster-scoped: no /namespaces/ segment
  "kind": "Node",
  "verbs": ["create","delete","deletecollection","get","list","patch","update","watch"],
  "shortNames": ["no"]
}
{
  "name": "pods",
  "singularName": "pod",
  "namespaced": true,
  "kind": "Pod",
  "verbs": ["create","delete","deletecollection","get","list","patch","update","watch"],
  "shortNames": ["po"],
  "categories": ["all"]
}
```

**Aggregated discovery** is the modern form. Instead of one request per group-version (which on a cluster with 40 CRD groups is 40+ round trips at every client start), the apiserver serves the whole thing from `/apis` when you ask for it with a specific Accept header:

```
$ kubectl get --raw /apis \
    -H 'Accept: application/json;v=v2;g=apidiscovery.k8s.io;as=APIGroupDiscoveryList' \
  | jq '.items[] | select(.metadata.name=="finops.example.com")'
{
  "metadata": {"name": "finops.example.com"},
  "versions": [
    {
      "version": "v1alpha1",
      "resources": [
        {
          "resource": "gpucostpolicies",
          "responseKind": {"group": "finops.example.com", "version": "v1alpha1", "kind": "GPUCostPolicy"},
          "scope": "Namespaced",
          "singularResource": "gpucostpolicy",
          "verbs": ["delete","deletecollection","get","list","patch","create","update","watch"],
          "shortNames": ["gcp"],
          "subresources": [
            {"subresource": "status",
             "responseKind": {"group":"finops.example.com","version":"v1alpha1","kind":"GPUCostPolicy"},
             "verbs": ["get","patch","update"]}
          ]
        }
      ],
      "freshness": "Current"
    }
  ]
}
```

Two requests (`/api` and `/apis`), once, for the entire cluster's API surface. The `freshness` field matters when aggregated APIServices are involved: `Stale` means the main apiserver could not reach that extension server and is serving a cached view.

#### The RESTMapper

A `meta.RESTMapper` is the object that answers "given this GVK, what GVR and scope?" and the reverse. The core call:

```go
import (
    "k8s.io/apimachinery/pkg/api/meta"
    "k8s.io/apimachinery/pkg/runtime/schema"
)

gvk := schema.GroupVersionKind{Group: "apps", Version: "v1", Kind: "Deployment"}

mapping, err := restMapper.RESTMapping(gvk.GroupKind(), gvk.Version)
if err != nil {
    // meta.IsNoMatchError(err) == true means "discovery has never heard of this kind"
    return err
}

mapping.Resource        // schema.GroupVersionResource{Group:"apps", Version:"v1", Resource:"deployments"}
mapping.GroupVersionKind// schema.GroupVersionKind{Group:"apps", Version:"v1", Kind:"Deployment"}
mapping.Scope.Name()    // meta.RESTScopeNameNamespace ("namespace") | meta.RESTScopeNameRoot ("root")
```

Internally a `DefaultRESTMapper` is a set of maps — `kindToPluralResource`, `pluralToSingular`, `singularToPlural`, `resourceToKind`, `kindToScope` — populated from discovery. Three variants you will actually encounter:

| Implementation | Populated from | Refresh behaviour | Used by |
|---|---|---|---|
| `DefaultRESTMapper` | explicit `Add()` calls | never; static | tests, `scheme`-driven code |
| `restmapper.NewDeferredDiscoveryRESTMapper` | a cached discovery client | `Reset()` on demand; disk cache with a TTL | `kubectl` |
| `apiutil.NewDynamicRESTMapper` | a live discovery client | **reloads that group automatically on a `NoMatchError`** | controller-runtime's `Client` and `Manager` |

controller-runtime's dynamic mapper is worth knowing precisely, because it is the one your operator uses. When a lookup misses, it calls `addKnownGroupAndReload()`, which re-fetches `/apis/<group>/<version>` for the specific missing group, rebuilds the mapper, and retries. It also clears the cached entry on a `NotFound`. The practical consequence: **an operator that installs its own CRDs and then immediately uses them will usually recover on its own**, because the first miss triggers a targeted reload. What it will *not* do is recover instantly if the apiserver has not yet finished serving the new resource — CRD establishment is asynchronous, signalled by the CRD's `Established` condition:

```
$ kubectl apply -f config/crd/bases/finops.example.com_gpucostpolicies.yaml
customresourcedefinition.apiextensions.k8s.io/gpucostpolicies.finops.example.com created

$ kubectl get gpucostpolicies
error: the server doesn't have a resource type "gpucostpolicies"       # <-- ~1s later

$ kubectl get crd gpucostpolicies.finops.example.com \
    -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"\n"}{end}'
NamesAccepted=True
Established=True

$ kubectl get gpucostpolicies
No resources found in default namespace.                                # <-- now it resolves
```

The correct install-then-use pattern is to wait on `Established=True` (and, for `kubectl`, to let its cached discovery expire or run `kubectl api-resources` to force a refresh), not to sleep and hope.

### The scheme

`runtime.Scheme` is the in-process registry that ties Go types to GVKs. It answers exactly two questions the codecs need — "given this Go object, what GVK do I stamp?" and "given this GVK on the wire, what Go struct do I allocate?" — plus it holds conversion functions between API versions and the generated `DeepCopy` implementations.

```go
// Typical generated CRD registration (zz_generated / groupversion_info.go in kubebuilder)
var (
    GroupVersion  = schema.GroupVersion{Group: "finops.example.com", Version: "v1alpha1"}
    SchemeBuilder = &scheme.Builder{GroupVersion: GroupVersion}
    AddToScheme   = SchemeBuilder.Register(
        &GPUCostPolicy{}, &GPUCostPolicyList{},
        &WorkloadCost{},  &WorkloadCostList{},
        &Budget{},        &BudgetList{},
    ).AddToScheme
)

// main.go — the manager's scheme must know every type the client will touch
scheme := runtime.NewScheme()
utilruntime.Must(clientgoscheme.AddToScheme(scheme))   // built-ins: Pod, Node, Deployment, …
utilruntime.Must(finopsv1alpha1.AddToScheme(scheme))   // your CRDs
```

**The single most common scheme bug**: forgetting `AddToScheme` for a type you read, and getting

```
no kind is registered for the type v1alpha1.WorkloadCost in scheme "pkg/runtime/scheme.go:100"
```

That error is the scheme saying "you handed me a Go struct and I have no GVK to stamp on it." It has nothing to do with RBAC, the CRD being installed, or the cluster at all — it is purely in-process.

The scheme is also where **version conversion** lives. Built-in types have an internal "hub" version (`__internal`); the apiserver decodes `apps/v1beta2` into the hub, then encodes to whatever the client asked for. For CRDs the equivalent is a conversion webhook plus a `storageVersion: true` marker on one version. Lesson 05 builds that; here the point is only that conversion is a scheme concern, not a client concern.

### Codecs and serialisation

Bytes become `runtime.Object`s through a **codec** built from the scheme. Decode reads `apiVersion` + `kind`, asks the scheme for the matching Go type, allocates it, and unmarshals. Encode does the reverse, stamping the GVK back on.

Three serialisers exist: JSON, YAML, and protobuf. Which one is used where:

| Path | Format | Why |
|---|---|---|
| `kubectl` ↔ apiserver | JSON (YAML accepted on input) | human-debuggable; `-v=8` shows it |
| client-go typed clientset ↔ apiserver | **protobuf** by default (`application/vnd.kubernetes.protobuf`) | ~3–10× smaller and faster to decode for built-ins |
| any client ↔ apiserver for a **CRD** | JSON, always | CRs have no generated protobuf marshallers |
| apiserver ↔ etcd for built-ins | protobuf (`--storage-media-type` default) | storage size and decode cost |
| apiserver ↔ etcd for CRs | JSON | same reason |

The subtlety that bites everyone: **a decoded typed object usually has an empty `TypeMeta`.**

```go
node, _ := clientset.CoreV1().Nodes().Get(ctx, "gpu-a3-07", metav1.GetOptions{})

node.TypeMeta                                // metav1.TypeMeta{} — EMPTY
node.GetObjectKind().GroupVersionKind()      // schema.GroupVersionKind{} — EMPTY, zero value
node.Kind                                    // "" — not "Node"

// The GVK is implied by the Go type. To recover it, ask the scheme:
gvks, _, err := scheme.ObjectKinds(node)     // [{"" "v1" "Node"}]
```

This is deliberate: for a typed object the Go type *is* the GVK, so carrying it in a field would be redundant and could disagree with reality. The decoder clears it after using it. An `*unstructured.Unstructured`, by contrast, has nowhere to put the type except the map, so its GVK is always populated. Code that works for both must go through `scheme.ObjectKinds()` or `apiutil.GVKForObject()`, never through `obj.GetObjectKind()` directly.

### `runtime.Object` vs `metav1.Object`

Two interfaces you will type-assert constantly, and they mean different things:

```go
// runtime.Object — TYPE identity. "the scheme and codecs can handle this."
type Object interface {
    GetObjectKind() schema.ObjectKind
    DeepCopyObject() Object
}

// metav1.Object — INSTANCE identity. "this has standard metadata."
type Object interface {
    GetName() string;            SetName(string)
    GetNamespace() string;       SetNamespace(string)
    GetUID() types.UID;          SetUID(types.UID)
    GetResourceVersion() string; SetResourceVersion(string)
    GetGeneration() int64;       SetGeneration(int64)
    GetLabels() map[string]string
    GetAnnotations() map[string]string
    GetOwnerReferences() []OwnerReference
    GetFinalizers() []string
    GetDeletionTimestamp() *Time
    GetManagedFields() []ManagedFieldsEntry
    // …
}
```

Embedding `metav1.TypeMeta` plus a generated `DeepCopyObject` satisfies the first; embedding `metav1.ObjectMeta` satisfies the second. controller-runtime's `client.Object` is simply both at once:

```go
type Object interface {
    metav1.Object
    runtime.Object
}
```

Framework code that must read a name or namespace without knowing the concrete type uses `meta.Accessor(obj)`, which returns the `metav1.Object` view of anything that has an `ObjectMeta` — including an `Unstructured`.

### Object metadata: the field table you actually need

`metadata` is a protocol, not boilerplate. Here is every field that carries semantics, who writes it, and what goes wrong when you get it wrong.

| Field | Type | Set by | Mutable? | What it means / what breaks |
|---|---|---|---|---|
| `name` | string | user (or server via `generateName`) | **no** | Unique within namespace + resource. Immutable — "renaming" is delete + create, which changes `uid` and orphans owner references. |
| `generateName` | string | user | on create only | Server appends a random suffix and returns the real `name`. Use it for objects your controller creates in a loop; it eliminates a whole class of name-collision retries. |
| `namespace` | string | user | **no** | Empty for cluster-scoped kinds. |
| `uid` | UID | **server**, on create | never | RFC 4122 UUID. **This is the identity that survives name reuse.** A Pod named `trainer-0` deleted and recreated has a different `uid`; owner references and `ownerReferences[].uid` mismatches are how the GC controller avoids adopting the wrong object. |
| `resourceVersion` | string | **server**, on every write | server-only | Opaque; backed by etcd's `ModRevision`. Send it on `Update` to get optimistic concurrency; omit it to force an overwrite. **Never** do arithmetic on it. Orderable within a resource type since v1.35; never across types or clusters. |
| `generation` | int64 | **server** | server-only | Bumped when the *desired state* changes. For a CR with a `/status` subresource: incremented on any non-`metadata`, non-`status` change (the apiextensions strategy deep-compares the object with `metadata` and `status` removed). Also bumped when `deletionTimestamp` is first set. Status-only writes do **not** bump it — that is the whole point of the subresource. |
| `creationTimestamp` | Time | **server** | never | RFC 3339, UTC. Not guaranteed monotonic across separate operations, so do not use it to order events. |
| `deletionTimestamp` | *Time | **server**, on graceful delete | server-only, may only be shortened | Non-nil means "deletion requested." The object is **still fully readable and still reconciled**; it is removed only when `finalizers` empties. Your reconciler must branch on this. |
| `deletionGracePeriodSeconds` | *int64 | **server** | shorten only | Only set alongside `deletionTimestamp`. `--grace-period=0` sets it to 0 (still not a bypass of finalizers). |
| `labels` | map | user + controllers | yes | Indexed and selectable. Selector semantics; changing a label can move an object between controllers. |
| `annotations` | map | user + controllers | yes | Not queryable. Total object size still counts against etcd's 1.5 MiB request limit — this is how `last-applied-configuration` used to blow up large objects. |
| `ownerReferences` | []OwnerReference | usually the creating **controller** | yes | `{apiVersion, kind, name, uid, controller, blockOwnerDeletion}`. Drives garbage collection *and* controller-runtime's `Owns()` watch mapping. At most one entry may have `controller: true`. Cross-namespace references are invalid by design and silently prevent collection (event reason `OwnerRefInvalidNamespace`). |
| `finalizers` | []string | **controllers** | yes, but entries may only be *removed* once `deletionTimestamp` is set | Each entry blocks final removal until its owner deletes it. Order is explicitly **not** enforced — the API conventions say so, because ordering "introduces significant risk of stuck finalizers." |
| `managedFields` | []ManagedFieldsEntry | **server** | server-only | Server-side-apply bookkeeping: which manager owns which field paths. Read it to debug "who keeps overwriting this field." |
| `selfLink` | string | — | — | Legacy, no longer populated. Ignore it. |

Two of these deserve their own diagram, because together they form the state machine every well-behaved controller implements.

#### `generation` / `observedGeneration`: the "is the status current?" protocol

`generation` counts revisions of *intent*. `status.observedGeneration` records which revision of intent the controller has actually processed. Comparing them is the only reliable way — for a human or for a CI gate — to know whether a status you are reading describes the spec you just wrote or the previous one.

```
  TIME ──────────────────────────────────────────────────────────────────────▶

  user: kubectl apply (spec.rateUSDPerGPUHour: 3.20)
        │
        ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ metadata.generation      : 1                                 │  server bumps on
   │ status.observedGeneration: 0   (absent)                      │  spec change
   │ status.conditions[Ready] : (none)                            │
   └──────────────────────────────────────────────────────────────┘
        │  watch fires → Reconcile(key)
        ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ controller computes cost from CURRENT pods,                  │
   │ writes via Status().Update():                                │
   │   status.observedGPUCostUSD  = 12.00                         │
   │   status.observedGeneration  = 1     ← "I have seen gen 1"   │
   │   status.conditions[Ready]   = True, Reason=CostComputed,    │
   │                                observedGeneration: 1          │
   │ metadata.generation          : 1     ← UNCHANGED (subresource)│
   └──────────────────────────────────────────────────────────────┘
        │
        │  user edits spec again (rate → 4.10)
        ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ metadata.generation      : 2   ← bumped                      │
   │ status.observedGeneration: 1   ← STALE. This status describes│
   │                                  generation 1, not 2.        │
   │ ⇒ "spec has changed and I have not caught up yet"            │
   └──────────────────────────────────────────────────────────────┘
        │  reconcile
        ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ status.observedGeneration: 2   ← converged                   │
   │ generation == observedGeneration  ⇒ status is TRUSTWORTHY    │
   └──────────────────────────────────────────────────────────────┘

  FAILURE MODE: generation keeps climbing, observedGeneration frozen at N.
  → the controller is not reconciling this object at all
    (crashed, lost leader lease, RBAC gap, backoff maxed, missing watch).
  → `kubectl wait --for=condition=Ready` will hang forever, and worse, a
    naive check of conditions[Ready]==True will report SUCCESS using a
    status that describes an old spec.
```

That last line is the reason `observedGeneration` exists at all. Without it, "Ready: True" is ambiguous — ready *for which spec?* The API conventions spell out the same rule for conditions: a condition carries its own optional `observedGeneration`, and "if `.metadata.generation` is currently 12 but `.status.conditions[x].observedGeneration` is 9, the condition is out of date."

The standard condition shape (`metav1.Condition`), which your CRD should use verbatim rather than inventing:

```yaml
status:
  observedGeneration: 2
  conditions:
  - type: Ready                       # CamelCase, or foo.example.com/CamelCase   (required)
    status: "True"                    # "True" | "False" | "Unknown"              (required)
    observedGeneration: 2             # the metadata.generation this was based on (optional)
    lastTransitionTime: "2026-08-17T09:14:22Z"  # only changes when status flips   (required)
    reason: CostComputed              # programmatic, CamelCase                   (required)
    message: "Computed 12.00 USD from 4 pods across 2 nodes"                      (required)
```

Note `lastTransitionTime` changes only on a *transition*. Rewriting an identical condition must not touch it — if it does, every reconcile produces a write, every write produces a watch event, and you have built an infinite reconcile loop. `meta.SetStatusCondition()` from `k8s.io/apimachinery/pkg/api/meta` implements the correct semantics; use it instead of appending by hand.

### Typed vs unstructured clients

- **Typed** (client-go clientset, or controller-runtime's `client.Client` with registered types): you get `*appsv1.Deployment` with compile-time fields, `resource.Quantity` instead of strings, and refactor support. Requires the type registered in the scheme and known at build time. The natural default for your own CRDs and for built-ins.
- **Unstructured / dynamic** (`dynamic.Interface`, `*unstructured.Unstructured`): the object is a `map[string]interface{}` addressed by path. You call by **GVR**, resolved through the RESTMapper. No compile-time knowledge of the type required — essential for generic tooling that must handle kinds it has never seen: garbage collectors, `kubectl`, policy engines, or a cost operator that inspects arbitrary workload kinds for GPU requests.

The tradeoff in one table:

| | Typed | Unstructured / dynamic |
|---|---|---|
| Field access | `node.Status.Capacity["nvidia.com/gpu"]` | `unstructured.NestedString(u.Object, "status","capacity","nvidia.com/gpu")` |
| Types preserved | `resource.Quantity` | `string` — you parse |
| Compile-time safety | yes | none; typos are runtime `found=false` |
| Needs scheme registration | yes | no |
| Needs RESTMapper at runtime | no (compiled in) | yes |
| Wire format | protobuf for built-ins | JSON always |
| Handles unknown kinds | no | yes |
| GVK on the object | empty `TypeMeta` | always populated |

Rule of thumb: **typed for the handful of kinds you own and compile against; unstructured/dynamic when the set of kinds is open or discovered at runtime.** Reaching for a dynamic client to talk to your own CRD is usually a sign you skipped code generation, not a design decision.

One shared footgun: `unstructured.NestedString` and friends return `(value, found, err)`. Ignoring `found` gives you the zero value and no indication that the path was wrong — the single most common source of "my dynamic client sees nothing" bugs.

### Watch and `resourceVersion`, from the client's side

Lesson 01 covered the watch cache from the server's side. Here is the same protocol as your client experiences it.

```
   CLIENT                                              APISERVER
   ──────                                              ─────────

   GET /api/v1/nodes?limit=500
        ──────────────────────────────────────────────▶
        ◀──────────────────────────────────────────────
        200 {"kind":"NodeList",
             "metadata":{"resourceVersion":"884219",     ← COLLECTION rv: your resume point
                         "continue":"eyJ2IjoibWV0YS…"}, ← more pages exist
             "items":[ …500 nodes, each with its own metadata.resourceVersion… ]}

   GET /api/v1/nodes?limit=500&continue=eyJ2IjoibWV0YS…
        ──────────────────────────────────────────────▶   (collection rv stays 884219 —
        ◀──────────────────────────────────────────────    you get a consistent snapshot)

   GET /api/v1/nodes?watch=1&resourceVersion=884219&allowWatchBookmarks=true&timeoutSeconds=300
        ──────────────────────────────────────────────▶
        ◀── {"type":"ADDED",   "object":{…"resourceVersion":"884231"…}}
        ◀── {"type":"MODIFIED","object":{…"resourceVersion":"884247"…}}
        ◀── {"type":"BOOKMARK","object":{"kind":"Node","apiVersion":"v1",
                                          "metadata":{"resourceVersion":"884990"}}}
                                                     ↑ no object change; just "you are current
                                                       through 884990" so an idle watcher does
                                                       not fall out of the history window
        ◀── {"type":"DELETED", "object":{…"resourceVersion":"885002"…}}

   ── connection drops (LB idle timeout, apiserver rollout, network blip) ──

   GET /api/v1/nodes?watch=1&resourceVersion=885002&allowWatchBookmarks=true
        ──────────────────────────────────────────────▶
        ◀── {"type":"ERROR","object":{"kind":"Status","apiVersion":"v1",
              "code":410,"reason":"Expired","status":"Failure",
              "message":"too old resource version: 885002 (887410)"}}
                                                     ↑ asked for 885002; oldest replayable
                                                       is 887410. The gap cannot be replayed.

   ⇒ CONTRACT: discard the resume point. LIST again for current state and a fresh
     collection rv, then WATCH from that. This is a "relist", it is NORMAL, and
     informers do it for you. Do not alert on it.
```

Three details that matter in code:

1. **`timeoutSeconds`** — watches are not eternal. client-go sets a randomised timeout (`minWatchTimeout` 5 minutes by default, randomised up to 2×) so that thousands of informers do not all reconnect in the same second after an apiserver restart. Your watch *will* be closed cleanly and reopened; that is not an error either.
2. **`allowWatchBookmarks=true`** — informers set this by default. Without it, watching a quiet resource on a busy cluster is a reliable way to earn a `410` for no reason: the global revision marches past your frozen resume point and etcd compacts it away.
3. **`sendInitialEvents`** — the streaming-list form (`?watch=1&sendInitialEvents=true&allowWatchBookmarks=true&resourceVersionMatch=NotOlderThan`) merges LIST and WATCH into one request. The server emits synthetic `ADDED` events for existing objects and then a `BOOKMARK` annotated `"k8s.io/initial-events-end": "true"` to mark the boundary. The `WatchList` gate is on by default in the 1.34+ line; the payoff is not needing to buffer a whole LIST response in apiserver memory.

### Optimistic concurrency, with a real transcript

`resourceVersion` does double duty: it is a watch resume point *and* it is the optimistic lock. An `Update` (HTTP `PUT`) that carries a `resourceVersion` succeeds only if the server's current version is identical — enforced, as lesson 01 showed, by an etcd transaction comparing `ModRevision`.

Here is the conflict, end to end. Two terminals, one object.

```
# ── Terminal A ──────────────────────────────────────────────────────────────
$ kubectl get workloadcost gpu-train-77 -n finops -o yaml > wc.yaml
$ grep resourceVersion wc.yaml
  resourceVersion: "884219"

# (you edit wc.yaml: spec.samplingInterval 60s → 30s ... and take 20 seconds doing it)

# ── Terminal B, meanwhile ───────────────────────────────────────────────────
$ kubectl label workloadcost gpu-train-77 -n finops team=ml-infra
workloadcost.finops.example.com/gpu-train-77 labeled
$ kubectl get workloadcost gpu-train-77 -n finops -o jsonpath='{.metadata.resourceVersion}'
884613

# ── Terminal A, submitting the stale copy ───────────────────────────────────
$ kubectl replace -f wc.yaml
Error from server (Conflict): error when replacing "wc.yaml": Operation cannot be fulfilled on
workloadcosts.finops.example.com "gpu-train-77": the object has been modified; please apply your
changes to the latest version and try again
```

Two things to notice. First, the conflict fired even though the two writers touched **completely different fields** — a label and a spec field. `Update` is whole-object replacement, so RV equality is checked on the object, not on the fields you meant to change. Second, HTTP status is `409`, and `apierrors.IsConflict(err)` is the only correct way to detect it in Go — never string-match the message.

In a controller, conflicts on your *own* status writes are routine and expected: you read from a cache that lags, another actor (or your own previous reconcile) wrote in between. The idiomatic handling:

```go
import (
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/client-go/util/retry"
)

// Option 1 — let the workqueue retry. Preferred in controller-runtime:
// returning the error requeues the key with exponential backoff, and the
// next Reconcile re-reads fresh state. Idempotency (lesson 03) makes this safe.
if err := r.Status().Update(ctx, wc); err != nil {
    if apierrors.IsConflict(err) {
        return ctrl.Result{}, err          // requeue; do NOT log it as an error
    }
    return ctrl.Result{}, err
}

// Option 2 — retry in place, for a tight read-modify-write you do not want to
// pay a full reconcile for. retry.DefaultRetry = {Steps: 5, Duration: 10ms,
// Factor: 1.0, Jitter: 0.1} — five attempts, ~10ms apart, no exponential growth.
err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
    latest := &finopsv1alpha1.WorkloadCost{}
    if err := r.Get(ctx, key, latest); err != nil {   // MUST re-Get inside the closure
        return err
    }
    latest.Status.ObservedGPUCostUSD = computed
    latest.Status.ObservedGeneration = latest.Generation
    return r.Status().Update(ctx, latest)
})
```

The single most common mistake in option 2 is re-using the object fetched *outside* the closure: you then retry the identical stale RV five times and fail five times. The `Get` has to be inside.

client-go also ships `retry.DefaultBackoff` = `{Steps: 4, Duration: 10ms, Factor: 5.0, Jitter: 0.1}` — documented as the right choice when you are making an *unrelated* modification to an object under active management by one or more controllers, i.e. when you expect to lose the race repeatedly and want to back off hard rather than hammer.

**Escaping the lock deliberately.** Sending an empty `resourceVersion` on an `Update` means "I do not care what is there, overwrite it." That is occasionally correct (a recovery tool) and almost always wrong in a controller — it silently discards whatever the other writer did. If you find yourself wanting it, you want server-side apply instead.

### The write verbs, and the three-and-a-half patch formats

| Verb | HTTP | Content-Type | Semantics | RV check |
|---|---|---|---|---|
| Create | `POST` | json/yaml/protobuf | fails if the name exists (`409 AlreadyExists`) | n/a (etcd `ModRevision == 0`) |
| Update | `PUT` | json/yaml/protobuf | **whole-object replace** | yes, if RV present |
| Patch (JSON Patch) | `PATCH` | `application/json-patch+json` | RFC 6902 — an ordered op list (`add`/`remove`/`replace`/`test`) | no (use a `test` op) |
| Patch (JSON Merge) | `PATCH` | `application/merge-patch+json` | RFC 7386 — recursive object merge; **arrays replace wholesale**; `null` deletes a key | no |
| Patch (Strategic Merge) | `PATCH` | `application/strategic-merge-patch+json` | Kubernetes-specific; arrays merge per `patchStrategy`/`patchMergeKey` struct tags. **Built-in types only** | no |
| Apply (SSA) | `PATCH` | `application/apply-patch+yaml` | declarative; server computes the diff from your field set and tracks ownership in `managedFields` | no — conflicts are per-field |

Note `application/apply-patch+yaml` is the media type even when the body is JSON (all JSON is valid YAML). And note the row that says **built-in types only**: strategic merge patch is driven by Go struct tags in the Kubernetes source, which do not exist for custom resources. `kubectl patch` on a CR with the default `--type strategic` fails with:

```
Error from server: cannot apply strategic merge patch for finops.example.com/v1alpha1, Kind=WorkloadCost
locally: unable to find api field in struct Unstructured for the json field "spec"
```

Use `--type merge` or `--type json` on custom resources, or better, apply.

#### The same patch, three ways, three different results

Start from this Deployment. This is the base object for all three cases:

```yaml
# base.yaml  — applied with: kubectl apply -f base.yaml  (client-side, manager "kubectl-client-side-apply")
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-cost-exporter
  namespace: finops
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gpu-cost-exporter
  template:
    metadata:
      labels:
        app: gpu-cost-exporter
    spec:
      containers:
      - name: exporter
        image: ghcr.io/example/gpu-cost-exporter:v0.3.1
        ports:
        - name: metrics
          containerPort: 9102
        resources:
          requests: {cpu: 50m, memory: 64Mi}
```

And here is the patch — **identical bytes in all three cases**. The intent is "add a DCGM sidecar."

```yaml
# add-sidecar.yaml
spec:
  template:
    spec:
      containers:
      - name: dcgm-sidecar
        image: nvcr.io/nvidia/k8s/dcgm-exporter:3.3.9-3.6.1-ubuntu22.04
```

**Case 1 — strategic merge patch (kubectl's default).**

```
$ kubectl patch deployment gpu-cost-exporter -n finops --patch-file add-sidecar.yaml
deployment.apps/gpu-cost-exporter patched

$ kubectl get deploy gpu-cost-exporter -n finops \
    -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}'
dcgm-sidecar    nvcr.io/nvidia/k8s/dcgm-exporter:3.3.9-3.6.1-ubuntu22.04
exporter        ghcr.io/example/gpu-cost-exporter:v0.3.1
```

**Result: two containers.** Why: `PodSpec.Containers` carries the struct tags

```go
Containers []Container `json:"containers" patchStrategy:"merge" patchMergeKey:"name" …`
```

so the patch engine merges the two lists **keyed on `name`**. A patch entry whose `name` matches an existing entry is merged into it field-by-field; a new `name` is appended. Note the ordering changed — patched entries land first. Order is not part of the contract for `merge`-strategy lists, so never depend on container index.

The counter-example in the same object: `PodSpec.Tolerations` has *no* `patchStrategy` tag, so even under strategic merge it replaces wholesale. "Strategic merge merges lists" is false in general — it merges the lists the Go authors tagged, and replaces the rest.

**Case 2 — JSON merge patch (RFC 7386).**

```
$ kubectl patch deployment gpu-cost-exporter -n finops --type merge --patch-file add-sidecar.yaml
deployment.apps/gpu-cost-exporter patched

$ kubectl get deploy gpu-cost-exporter -n finops \
    -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}'
dcgm-sidecar    nvcr.io/nvidia/k8s/dcgm-exporter:3.3.9-3.6.1-ubuntu22.04
```

**Result: ONE container. You just deleted your exporter.** RFC 7386 has no concept of list identity: an array in the patch replaces the array in the target, entirely. The Deployment is still valid, the rollout proceeds, and your metrics vanish. Nothing warns you.

This is the single most dangerous line in this lesson. On a custom resource, `--type merge` is what you are *forced* to use (strategic merge is unavailable), which means **the patch type that is mandatory for CRs is also the one that silently truncates lists.** If your operator patches a list field on a CR, use JSON Patch with explicit index operations, or use apply.

**Case 3 — server-side apply, as a new field manager.**

```
$ kubectl apply --server-side --field-manager=sidecar-injector -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-cost-exporter
  namespace: finops
spec:
  template:
    spec:
      containers:
      - name: dcgm-sidecar
        image: nvcr.io/nvidia/k8s/dcgm-exporter:3.3.9-3.6.1-ubuntu22.04
EOF
deployment.apps/gpu-cost-exporter serverside-applied

$ kubectl get deploy gpu-cost-exporter -n finops \
    -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}'
dcgm-sidecar    nvcr.io/nvidia/k8s/dcgm-exporter:3.3.9-3.6.1-ubuntu22.04
exporter        ghcr.io/example/gpu-cost-exporter:v0.3.1
```

**Result: two containers — like case 1, but for a completely different reason.** SSA does not merge lists by struct tag; it merges by the OpenAPI extensions `x-kubernetes-list-type: map` and `x-kubernetes-list-map-keys: ["name"]`, and it tracks **who owns what**. The `exporter` entry survives not because the patch "merged" but because a *different manager* owns it and this apply did not claim it.

Prove it by reading `managedFields`:

```
$ kubectl get deploy gpu-cost-exporter -n finops -o json | jq '.metadata.managedFields'
[
  {
    "manager": "kubectl-client-side-apply",
    "operation": "Update",
    "apiVersion": "apps/v1",
    "time": "2026-08-17T09:02:11Z",
    "fieldsType": "FieldsV1",
    "fieldsV1": {
      "f:spec": {
        "f:replicas": {},
        "f:selector": {},
        "f:template": {
          "f:spec": {
            "f:containers": {
              "k:{\"name\":\"exporter\"}": {                <-- owned by the ORIGINAL applier
                ".": {}, "f:image": {}, "f:name": {},
                "f:ports": {"k:{\"containerPort\":9102,\"protocol\":\"TCP\"}": {".": {}, "f:name": {}}},
                "f:resources": {"f:requests": {"f:cpu": {}, "f:memory": {}}}
              }
            }
          }
        }
      }
    }
  },
  {
    "manager": "sidecar-injector",
    "operation": "Apply",                                    <-- Apply, not Update
    "apiVersion": "apps/v1",
    "time": "2026-08-17T09:41:03Z",
    "fieldsType": "FieldsV1",
    "fieldsV1": {
      "f:spec": {
        "f:template": {
          "f:spec": {
            "f:containers": {
              "k:{\"name\":\"dcgm-sidecar\"}": {             <-- owns ONLY its own entry
                ".": {}, "f:image": {}, "f:name": {}
              }
            }
          }
        }
      }
    }
  },
  {
    "manager": "kube-controller-manager",
    "operation": "Update",
    "subresource": "status",                                 <-- status writes are a SEPARATE entry
    "apiVersion": "apps/v1",
    "fieldsV1": {"f:status": {"f:availableReplicas": {}, "f:conditions": {}, "f:replicas": {}}}
  }
]
```

The `fieldsV1` grammar in three rules: `f:<name>` is an object field; `k:{...}` is a *list-map key* selecting one entry of an associative list; `.` (a bare dot) means "the entry itself exists," as distinct from owning its children. `v:` (not shown) selects a member of a `set`-type list by value.

**Case 3b — the same apply, but as the *original* manager.** This is the behaviour that surprises people:

```
$ kubectl apply --server-side --field-manager=kubectl -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: {name: gpu-cost-exporter, namespace: finops}
spec:
  template:
    spec:
      containers:
      - name: dcgm-sidecar
        image: nvcr.io/nvidia/k8s/dcgm-exporter:3.3.9-3.6.1-ubuntu22.04
EOF

# if "kubectl" also owned f:containers k:{"name":"exporter"}, that entry is now
# absent from its applied configuration ⇒ SSA REMOVES it.
```

**SSA prunes fields you used to own and no longer list.** That is the whole declarative contract: your applied configuration is the complete statement of what *you* manage, not a delta. A controller that switches from `Update` to `Apply` and forgets to include a field it previously wrote will delete it.

**Conflicts.** When two appliers claim the same field with different values:

```
$ kubectl apply --server-side --field-manager=deploy-tool -f scale-up.yaml
error: Apply failed with 1 conflict: conflict with "kube-controller-manager" using autoscaling/v2: .spec.replicas
Please review the fields above--they currently have other managers. Here
are the ways you can resolve this warning:
* If you intend to manage all of these fields, please re-run the apply
  command with the `--force-conflicts` flag.
* If you do not intend to manage all of the fields, please edit your
  manifest to remove references to the fields that should keep their
  current managers.
* You may co-own fields by updating your manifest to match the existing
  value; in this case, you'll become the manager if the other manager(s)
  stay out of the way.
```

Three resolutions, exactly as the message says: force (take ownership, removing the field from the other manager's entry), drop the field from your manifest, or set it to the same value and become a co-owner. In Go:

```go
patch := &appsv1.Deployment{
    TypeMeta:   metav1.TypeMeta{APIVersion: "apps/v1", Kind: "Deployment"},  // REQUIRED for SSA
    ObjectMeta: metav1.ObjectMeta{Name: "gpu-cost-exporter", Namespace: "finops"},
}
patch.Spec.Template.Spec.Containers = []corev1.Container{{
    Name:  "dcgm-sidecar",
    Image: "nvcr.io/nvidia/k8s/dcgm-exporter:3.3.9-3.6.1-ubuntu22.04",
}}

err := c.Patch(ctx, patch, client.Apply,
    client.FieldOwner("gpu-cost-operator"),   // REQUIRED: SSA rejects an empty manager
    client.ForceOwnership)                    // = --force-conflicts
```

`TypeMeta` must be set explicitly for an SSA patch, because the server needs the GVK from the body — it cannot infer it from a Go type it never sees. Forgetting it produces `Object 'Kind' is missing`.

**Why an operator should prefer SSA over read-modify-write.** With `Update` you must re-`Get` on every conflict and you overwrite fields other actors own. With `Apply` you declare only your fields, conflicts are reported per field rather than per object, and there is no RV round trip at all — which removes an entire category of hot-loop bugs where two controllers fight over one object and each `Update` triggers the other's watch. The cost is that you must be disciplined about your field set, because omission means deletion.

RBAC note: SSA needs both `patch` (to modify) and `create` (because apply creates the object if it is absent). A ClusterRole with only `patch` gives you a confusing `403` on first apply.

### How CRDs slot in

A `CustomResourceDefinition` is itself an API object. Applying one makes the apiserver serve a new GVR from its generic registry and adds it to discovery. From that moment:

- the RESTMapper can resolve `gpucostpolicies` ↔ `GPUCostPolicy`, once its cache refreshes;
- dynamic clients can address it by GVR immediately;
- typed clients work as soon as your generated Go types are registered in a scheme;
- watch, `resourceVersion`, optimistic concurrency, field validation, SSA, and `managedFields` all behave identically to `Pod`.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: gpucostpolicies.finops.example.com          # MUST be <plural>.<group>
spec:
  group: finops.example.com
  names:
    kind: GPUCostPolicy                              # the GVK's Kind
    listKind: GPUCostPolicyList
    plural: gpucostpolicies                          # the GVR's Resource — arbitrary, not derived
    singular: gpucostpolicy
    shortNames: [gcp]
  scope: Namespaced                                  # → meta.RESTScopeNamespace
  versions:
  - name: v1alpha1
    served: true                                     # appears in discovery
    storage: true                                    # exactly one version may be the storage version
    subresources:
      status: {}                                     # creates the /status GVR row
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [rateUSDPerGPUHour]
            properties:
              rateUSDPerGPUHour:
                type: string
                pattern: '^[0-9]+(\.[0-9]+)?$'
              samplingInterval:
                type: string
                default: "60s"
          status:
            type: object
            properties:
              observedGeneration: {type: integer, format: int64}
              conditions:
                type: array
                x-kubernetes-list-type: map          # ← this is what makes SSA merge conditions
                x-kubernetes-list-map-keys: [type]   #   by type instead of replacing the array
                items:
                  type: object
                  required: [type, status, lastTransitionTime, reason, message]
                  properties:
                    type: {type: string}
                    status: {type: string, enum: ["True","False","Unknown"]}
                    observedGeneration: {type: integer, format: int64}
                    lastTransitionTime: {type: string, format: date-time}
                    reason: {type: string}
                    message: {type: string}
```

Two lines there are pure API-machinery, and skipping them is a classic bug: without `x-kubernetes-list-type: map` and `x-kubernetes-list-map-keys: [type]`, the `conditions` array defaults to `atomic`, which means **any SSA writer replaces the whole conditions list** — so two controllers each maintaining their own condition will delete each other's on every write. Lesson 05 goes deep on CRD design; this is the piece that belongs here because it is a merge-semantics decision, not a schema decision.

`subresources: {status: {}}` is what creates the `gpucostpolicies/status` row in discovery — and therefore what makes `generation` stop incrementing on status writes, and what makes `Status().Update()` a distinct call. The whole `observedGeneration` protocol depends on that one line.

## Perspectives

**Developer perspective.** You almost always want typed clients for your own CRDs — compile-time safety and refactor support are worth the coupling, and code generation gives them to you for free. Dynamic and unstructured access is a tool for *generic* tooling that must handle kinds it has never compiled against. Reaching for a dynamic client for your own known types usually means you skipped `make generate`, not that you made an architectural choice.

**Operator perspective.** `kubectl api-resources`, `kubectl explain`, and `kubectl get -o wide` are RESTMapper, OpenAPI, and discovery output you have used for years without seeing the machinery. A `410 Gone` in a controller log is the watch protocol working exactly as designed. And `kubectl get <obj> -o json | jq .metadata.managedFields` is the fastest answer to "who keeps changing this field" — a question that otherwise turns into an afternoon of `kubectl get events` archaeology.

**Framework-author perspective.** Every framework that talks to Kubernetes — client-go, controller-runtime, Helm, Argo CD, `kubectl` — re-derives the same GVK↔GVR resolution, the same typed-vs-dynamic tradeoff, and the same patch-format decision. Recognising the pattern once means you can read any of their source trees quickly, because they are all a thin layer over the same small set of apimachinery primitives. It also means you can predict their bugs: anything doing generic object handling has a RESTMapper staleness story and a patch-semantics story, and those are the first two places to look.

**Failure-mode perspective.** Three bug classes in this lesson look like magic until you know the layer: (1) "my controller 404s on a CRD I just applied" is discovery cache plus asynchronous CRD establishment; (2) "my patch deleted a container" is RFC 7386 array replacement; (3) "two controllers keep overwriting each other's status conditions" is a missing `x-kubernetes-list-type: map`. Each takes minutes to diagnose with this model and hours without it.

## Real-world use cases

- **[Tim Ebert (Gardener / SAP), "Kubernetes Controllers at Scale: Clients, Caches, Conflicts, Patches Explained"](https://medium.com/@timebertt/kubernetes-controllers-at-scale-clients-caches-conflicts-patches-explained-aa0f7a8b4332).** A practitioner on Gardener — a production multi-cluster management system running thousands of clusters — working through exactly the client/cache/conflict/patch decisions in this lesson at fleet scale: why optimistic-lock conflicts become routine rather than exceptional when many controllers touch shared objects, and why the patch type you choose changes your conflict rate. This is the grounding for "conflicts are normal, design for them" rather than "conflicts are a bug."
- **[ymmt2005, "An example of using dynamic client of k8s.io/client-go"](https://ymmt2005.hatenablog.com/entry/2020/04/14/An_example_of_using_dynamic_client_of_k8s.io/client-go).** RESTMapper → GVR resolution → dynamic client → decoding YAML into `Unstructured`, in real working code from an engineer building it for production rather than for a tutorial. Effectively the worked example below, written independently.
- **The `kubectl` client-side-to-server-side apply migration.** The `kubectl.kubernetes.io/last-applied-configuration` annotation was client-side apply's way of remembering "what I applied last time" so it could compute deletions — a full copy of the object stored *inside* the object, which on large objects pushed against etcd's 1.5 MiB request limit and made every object roughly twice its own size. `managedFields` replaced a whole-object snapshot with a field-path set owned per manager. Ecosystem adoption has been slow and instructive: Helm only defaulted to SSA for new releases in Helm 4 (November 2025), because switching an existing manager to `Apply` changes deletion semantics for every field it used to own — the exact hazard in case 3b above.
- **[Uber Engineering, "Migrating Uber's Compute Platform to Kubernetes"](https://www.uber.com/blog/migrating-ubers-compute-platform-to-kubernetes-a-technical-journey/).** Reused from lesson 01 with a different lens: Uber's job controller reads a CRD-addressed object, resolves its identity, and reconciles it — a production description of the GVK-addressed, scheme-decoded object your own controller will work with.

## Worked example

Read a Node's GPU capacity three ways — typed, dynamic, and through an explicit RESTMapper — then write back with SSA and inspect the resulting field ownership. This is the full round trip of everything above.

```go
package main

import (
    "context"
    "fmt"
    "log"

    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
    "k8s.io/apimachinery/pkg/runtime/schema"
    "k8s.io/client-go/discovery"
    "k8s.io/client-go/discovery/cached/memory"
    "k8s.io/client-go/dynamic"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/restmapper"
    "sigs.k8s.io/controller-runtime/pkg/client/config"
)

func main() {
    ctx := context.Background()
    cfg, err := config.GetConfig() // KUBECONFIG, ~/.kube/config, or in-cluster
    if err != nil {
        log.Fatal(err)
    }
    const nodeName = "gpu-a3-07"

    // ── 1. TYPED ────────────────────────────────────────────────────────────
    cs, err := kubernetes.NewForConfig(cfg)
    if err != nil {
        log.Fatal(err)
    }
    node, err := cs.CoreV1().Nodes().Get(ctx, nodeName, metav1.GetOptions{})
    if err != nil {
        log.Fatal(err)
    }
    // No RESTMapper call happened: CoreV1().Nodes() has GVR {"",v1,nodes} and
    // cluster scope compiled in. The wire format was protobuf.
    q := node.Status.Capacity["nvidia.com/gpu"]      // resource.Quantity
    gpus, _ := q.AsInt64()
    fmt.Printf("typed:      gpus=%d  TypeMeta=%q (empty on purpose)\n",
        gpus, node.GetObjectKind().GroupVersionKind().String())

    // ── 2. RESTMAPPER: GVK → GVR ────────────────────────────────────────────
    dc, err := discovery.NewDiscoveryClientForConfig(cfg)
    if err != nil {
        log.Fatal(err)
    }
    mapper := restmapper.NewDeferredDiscoveryRESTMapper(memory.NewMemCacheClient(dc))
    gvk := schema.GroupVersionKind{Group: "", Version: "v1", Kind: "Node"}
    m, err := mapper.RESTMapping(gvk.GroupKind(), gvk.Version)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("restmapper: gvk=%s -> gvr=%s scope=%s\n",
        gvk, m.Resource, m.Scope.Name())

    // ── 3. DYNAMIC, using the resolved GVR ──────────────────────────────────
    dyn, err := dynamic.NewForConfig(cfg)
    if err != nil {
        log.Fatal(err)
    }
    u, err := dyn.Resource(m.Resource).Get(ctx, nodeName, metav1.GetOptions{})
    if err != nil {
        log.Fatal(err)
    }
    raw, found, err := unstructured.NestedString(u.Object, "status", "capacity", "nvidia.com/gpu")
    if err != nil || !found {
        log.Fatalf("nvidia.com/gpu not present on %s (found=%v err=%v)", nodeName, found, err)
    }
    fmt.Printf("dynamic:    gpus=%q  TypeMeta=%q (always populated)\n",
        raw, u.GetObjectKind().GroupVersionKind().String())

    // ── 4. SSA WRITE: claim exactly one label ───────────────────────────────
    apply := &unstructured.Unstructured{Object: map[string]interface{}{
        "apiVersion": "v1",
        "kind":       "Node",
        "metadata": map[string]interface{}{
            "name": nodeName,
            "labels": map[string]interface{}{
                "finops.example.com/cost-score": "0.83",
            },
        },
    }}
    out, err := dyn.Resource(m.Resource).Apply(ctx, nodeName, apply, metav1.ApplyOptions{
        FieldManager: "apimachinery-probe",   // required; empty manager is rejected
        Force:        false,                  // do NOT steal fields we do not own
    })
    if err != nil {
        log.Fatal(err)
    }
    for _, e := range out.GetManagedFields() {
        fmt.Printf("managedField: manager=%-28s op=%-6s sub=%-6s\n",
            e.Manager, e.Operation, e.Subresource)
    }
}
```

Representative output:

```
typed:      gpus=8  TypeMeta="/, Kind=" (empty on purpose)
restmapper: gvk=/v1, Kind=Node -> gvr=/v1, Resource=nodes scope=root
dynamic:    gpus="8"  TypeMeta="/v1, Kind=Node" (always populated)
managedField: manager=kubelet                      op=Update sub=
managedField: manager=kubelet                      op=Update sub=status
managedField: manager=kube-controller-manager      op=Update sub=
managedField: manager=apimachinery-probe           op=Apply  sub=
```

Read the output line by line:

- The typed and dynamic paths returned **the same bytes off the wire**, decoded differently: `resource.Quantity` in one case, the string `"8"` in the other. Same server, same object, different in-process representation. If a node had `nvidia.com/gpu: 8k` (nobody does, but the type allows it), the typed path would give you `8000` and the string path would give you `"8k"` — which is exactly why you want `Quantity` for anything you compute on.
- `scope=root` is the RESTMapper telling you Nodes are cluster-scoped, so the URL has no `/namespaces/` segment. A dynamic client that calls `.Namespace("default")` on a root-scoped GVR gets a confusing 404 rather than a type error.
- The kubelet appears **twice**: once for the main resource and once with `sub=status`. Subresource writes are tracked as separate managed-field entries even for the same manager name — that is what the `subresource` field on `ManagedFieldsEntry` is for, and it is why your controller's `Status().Update()` never conflicts with its own spec-level writes.
- `apimachinery-probe` owns `f:metadata.labels.finops.example.com/cost-score` and nothing else. Because it touched no field the kubelet owns, `Force: false` was sufficient — the disjoint-field case is the common one, and it is the reason SSA lets many controllers annotate the same Node without fighting.

To see the ownership set itself:

```
$ kubectl get node gpu-a3-07 -o json \
  | jq '.metadata.managedFields[] | select(.manager=="apimachinery-probe") | .fieldsV1'
{
  "f:metadata": {
    "f:labels": {
      "f:finops.example.com/cost-score": {}
    }
  }
}
```

## Practice

Add a diagnostic binary to the operator repo that exercises every mechanism in this lesson and prints what the machinery resolved. This feeds the [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md) deliverable's typed-vs-unstructured GPU-capacity read.

1. In `../practice/gpu-cost-operator/`, add `cmd/apimachinery-probe/main.go`. Build a `*rest.Config` with `sigs.k8s.io/controller-runtime/pkg/client/config.GetConfig()` so it works both in-cluster and from a kubeconfig.
2. **Typed read.** `kubernetes.NewForConfig` → list Nodes → print each node's `status.capacity["nvidia.com/gpu"]` as an int64 (zero when absent), plus `status.allocatable` for the same key. Note where they differ and why.
3. **Dynamic read.** `dynamic.NewForConfig` with GVR `{"", "v1", "nodes"}` → list the same Nodes → pull `nvidia.com/gpu` with `unstructured.NestedString`. **Handle `found == false` explicitly** and print a distinct message for "absent" vs "error."
4. **RESTMapper.** Build a `restmapper.NewDeferredDiscoveryRESTMapper` over a memory-cached discovery client. Print the `RESTMapping` for GVK `{"", "v1", "Node"}` (expect GVR `nodes`, scope `root`) *and* for your own `{"finops.example.com", "v1alpha1", "GPUCostPolicy"}`. Then delete the CRD, call `RESTMapping` again, and record the exact error — `meta.IsNoMatchError` should be true.
5. **Discovery dump.** Print the resource rows for one built-in group and for your CRD group, showing `name`, `kind`, `namespaced`, `verbs`. Confirm your CRD has a `/status` row and that it lacks `create`, `delete`, and `watch`.
6. **Patch comparison — the important one.** Create a throwaway ConfigMap with three keys. Then, from the probe binary or from `kubectl`, apply the *same* logical change (`add key `d``) four ways and record the resulting key set each time:
   - `--type strategic`
   - `--type merge`
   - `--type json` with an explicit `[{"op":"add","path":"/data/d","value":"4"}]`
   - `--server-side --field-manager=probe`
   Then repeat the exercise on a *list* field — use a Deployment's `spec.template.spec.containers` or your CR's `status.conditions` — and record which formats preserve existing entries. Write down the one-sentence rule you would give a teammate.
7. **Optimistic concurrency.** Read an object, hold the RV, mutate it from another terminal, then attempt `Update` with the stale RV. Capture the exact error and assert `apierrors.IsConflict(err)`. Then wrap the same write in `retry.RetryOnConflict(retry.DefaultRetry, …)` with the `Get` *inside* the closure and show it succeeding.
8. **SSA field ownership.** Patch a test label onto one node with `client.Apply` + `FieldOwner("apimachinery-probe")`, print `metadata.managedFields`, and confirm your manager owns only that label path. Then apply a second time with the label *removed* from your configuration and confirm the label is deleted — that is the pruning behaviour from case 3b.
9. Write a 6–8 line comment block: when you would reach for each client, when for each patch type, and one concrete capstone case for the dynamic client (scanning arbitrary workload kinds for GPU requests).

**Acceptance:**

- Both clients print the same GPU count per node (allowing string-vs-int formatting).
- The program prints the resolved GVR (`nodes`) and scope (`root`) from the RESTMapper, and a `NoMatchError` for a GVK that does not exist.
- The discovery dump shows your CRD's `/status` subresource row and its restricted verb set.
- A table in the artifact records the four patch types × two field shapes (scalar map, list) with the actual resulting state for each of the eight cells.
- A captured `409 Conflict` transcript, plus a passing `RetryOnConflict` version.
- `managedFields` output shows the SSA write's ownership scoped to exactly the field you patched, and a second run demonstrating pruning.
- Runs against any cluster (`kind`/minikube fine; fake `nvidia.com/gpu` capacity with `kubectl patch node <n> --subresource=status --type=merge -p '{"status":{"capacity":{"nvidia.com/gpu":"8"}}}'` if you have no real GPUs).

## Common pitfalls

1. **Hardcoding a plural resource name derived from the Kind.** `UnsafeGuessKindToResource` exists, is named "Unsafe," and carries a special-case list because the rule is wrong. CRD authors choose `spec.names.plural` freely. Symptom: a dynamic client that works for `Pod` and 404s for someone else's CRD. Fix: always resolve through the RESTMapper or discovery.

2. **Reading `obj.GetObjectKind().GroupVersionKind()` on a typed object, getting an empty GVK, and assuming something is broken.** Expected: the decoder clears `TypeMeta` because the Go type already carries the identity. Symptom: an `if gvk.Kind == "" { return err }` guard that rejects every healthy object. Fix: `scheme.ObjectKinds(obj)` or `apiutil.GVKForObject(obj, scheme)`.

3. **`kubectl patch --type merge` (or a controller's `client.MergeFrom`) on a list field.** RFC 7386 replaces arrays wholesale. Symptom: containers, tolerations, env vars, ports, or status conditions silently vanish and the object stays valid, so nothing errors. Mechanism: JSON merge patch has no concept of list identity. Fix: strategic merge for built-ins, JSON Patch with explicit indices, or SSA — and for CRs, remember strategic merge is *not available*, which is exactly why this bites hardest there.

4. **Treating `410 Gone` as an application error to alert on.** It is the documented relist signal for a resume point older than the history window. Alert on watches that fail to *recover*, or on relist rate as a load indicator — never on the 410.

5. **Assuming a CRD is usable the instant `kubectl apply` returns.** The apiserver establishes CRDs asynchronously; discovery caches lag; controller-runtime's dynamic mapper recovers on a miss but the resource genuinely may not be served yet. Fix: wait for the CRD's `Established=True` condition, and expect a retry on the first use.

6. **Mutating an object returned from a lister or a controller-runtime cached client.** That pointer is shared memory read concurrently by every other handler. Symptom: "impossible" cross-goroutine corruption that only reproduces under load and never in tests. Fix: `obj.DeepCopy()` before touching anything. (controller-runtime's default client copies on `Get` but not necessarily on `List` into a shared slice — copy anyway.)

7. **Switching a controller from `Update` to `Apply` without auditing its field set.** SSA prunes fields the manager previously owned and no longer lists. Symptom: an upgrade that quietly deletes annotations, ports, or an entire container. Fix: enumerate what the old code wrote, make sure the applied configuration includes all of it, and roll out behind a flag.

8. **Omitting `TypeMeta` on an SSA patch object in Go.** The server needs the GVK from the request body. Symptom: `Object 'Kind' is missing in ...`. Fix: set `APIVersion` and `Kind` explicitly; the typed client will not do it for you.

## Self-check

- **GVK vs GVR — who resolves one to the other, and how?**
  **Answer:** A **GVK** (`apps/v1`, `Deployment`) names a *type* — it is stamped in `apiVersion`+`kind` and registered against a Go struct in a `runtime.Scheme`. A **GVR** (`apps/v1`, `deployments`) names the *REST resource path* the apiserver serves. The **RESTMapper** resolves between them, populated from the apiserver's **discovery** document (`/api/v1`, `/apis/<group>/<version>`, or aggregated via `Accept: application/json;v=v2;g=apidiscovery.k8s.io;as=APIGroupDiscoveryList`), which lists each resource's `name`, `kind`, `namespaced`, `verbs`, `shortNames`, and subresource rows. Call `mapper.RESTMapping(gvk.GroupKind(), gvk.Version)` to get `mapping.Resource` (the GVR) and `mapping.Scope.Name()` (`namespace` or `root`). Typed clients skip all of this — the mapping is compiled into the clientset. Dynamic and generic clients must consult the mapper, which is why a freshly installed CRD needs discovery to refresh (controller-runtime's dynamic mapper reloads the specific group on a `NoMatchError`) and why `UnsafeGuessKindToResource` is not an acceptable substitute — CRDs declare their plural by hand.

- **Typed vs unstructured client — when each, and what does the scheme do?**
  **Answer:** **Typed** for the fixed set of kinds you compile against (built-ins and your own CRDs): `*corev1.Node` with real fields, `resource.Quantity` instead of strings, compile-time safety, protobuf on the wire for built-ins. **Unstructured/dynamic** when the set of kinds is open or discovered at runtime — generic tooling, policy engines, a cost scanner walking arbitrary workload kinds — where objects are `map[string]interface{}` addressed by GVR and always JSON on the wire. The **scheme** is the in-process registry mapping Go struct ↔ GVK, plus conversion functions between API versions and generated `DeepCopy`s; codecs use it to allocate the right struct on decode and to stamp the GVK on encode. Typed access requires registration (`AddToScheme`); the error for forgetting it is `no kind is registered for the type … in scheme`, which is purely local and has nothing to do with the cluster. Unstructured access needs no scheme but does need a RESTMapper.

- **What does `resourceVersion` guarantee on a watch, and what is a `410 Gone`?**
  **Answer:** It is an opaque token backed by etcd's `ModRevision` that marks a point in the change stream. You resume a watch by passing the last RV you saw, and every event carries the object's new RV. It is not arithmetic: only equality, "resume from here," and (since v1.35, within a single resource type) integer ordering are meaningful. It also implements optimistic concurrency — an `Update` carrying a stale RV gets `409 Conflict` with `Operation cannot be fulfilled on …: the object has been modified; please apply your changes to the latest version and try again`. **`410 Gone`** means the RV you are resuming from is older than the server's bounded history window (the watch cache ring buffer targets ~75 s of events and the apiserver compacts etcd every 5 minutes by default), so the gap cannot be replayed; the message is literally `too old resource version: <yours> (<oldest replayable>)`. The contract is: drop the resume point, LIST for fresh state and a fresh collection RV, restart the watch. Informers do it automatically. Bookmarks (`allowWatchBookmarks=true`) exist to keep idle watchers' resume points current so this happens less.

- **Same patch bytes, three formats — what are the three results and why?**
  **Answer:** Adding a container to a Deployment's `spec.template.spec.containers`: (1) **Strategic merge** (`--type strategic`, the kubectl default) merges the lists keyed on `name`, because `PodSpec.Containers` carries `patchStrategy:"merge" patchMergeKey:"name"` — result: both containers, order not guaranteed. It is available for built-in types only. (2) **JSON merge patch** (`--type merge`, RFC 7386) replaces the array wholesale — result: **only the new container; the existing one is deleted**, with no error, because RFC 7386 has no list identity. This is the mandatory format for custom resources, which is why it is most dangerous exactly where you are forced to use it. (3) **Server-side apply** (`--server-side`) merges using the OpenAPI `x-kubernetes-list-type: map` / `x-kubernetes-list-map-keys: [name]` extensions *and* tracks ownership in `managedFields` — result: both containers, because the existing one is owned by a different manager. But if the *same* manager applies a configuration that omits a field it previously owned, SSA **prunes** it. Conflicts between managers over one field produce `Apply failed with 1 conflict: conflict with "<manager>" using <apiVersion>: <path>`, resolvable by forcing, dropping the field, or matching the value to co-own.

- **Walk `generation` / `observedGeneration` and say what each failure looks like.**
  **Answer:** The server increments `metadata.generation` when desired state changes — for a CR with a `/status` subresource, any change outside `metadata` and `status` (the apiextensions strategy deep-compares with those stripped), plus once when `deletionTimestamp` is first set. Writes through `/status` do **not** bump it, which is the entire reason the subresource exists. The controller writes `status.observedGeneration = metadata.generation` after processing, and conditions may carry their own `observedGeneration`. `generation == observedGeneration` means the status describes the current spec and is trustworthy. `generation > observedGeneration` means the controller has not caught up — transiently normal, but if it stays frozen while `generation` climbs, the controller is not reconciling that object at all (crashed, lost its leader lease, missing RBAC so the cache never synced, backoff maxed at ~1000 s, or a missing watch on a dependency). The dangerous failure is silent: a CI gate that checks only `conditions[Ready].status == "True"` will report success against a status computed for an older spec. Always compare the generations too.

- **Why would an operator prefer server-side apply to read-modify-write `Update`?**
  **Answer:** `Update` is whole-object replacement under an optimistic lock, so you must `Get` first, you conflict with writers who touched fields you do not care about, and you overwrite fields other managers own. Apply declares only your fields; the server computes the diff, records ownership per field path in `managedFields`, and reports conflicts per field rather than per object — so two controllers writing disjoint fields on the same Node never fight, and there is no RV round trip to lose. It also removes a common hot-loop bug where two controllers each `Update` the whole object and each write wakes the other. The costs: your applied configuration must be complete for the fields you own (omission means deletion), you must set `TypeMeta` and a non-empty `FieldOwner`, and RBAC needs `create` in addition to `patch`.

## Connections & what's next

This lesson is the layer every subsequent lesson quietly depends on. Lesson 04's informers are reflectors decoding watch events through exactly this scheme/codec machinery, resuming on exactly these `resourceVersion` rules. Lesson 05's CRD design builds on scheme registration, the `x-kubernetes-list-type` merge markers, and the storage/served version split. Lesson 06's `Status().Update()` vs `Update()` distinction is the `/status` subresource row in discovery, and its finalizer work is the `metadata.finalizers` protocol from the field table. Lesson 08's admission webhooks receive an `AdmissionReview` carrying a raw object you must decode against a scheme. The `410 Gone`, RESTMapper-refresh, and patch-semantics failure modes here resurface as concrete debugging scenarios in the module checkpoint.

Next: **[02.3 · Reconciliation model](03-reconciliation-model.md)** takes the objects this lesson taught you to address, type, patch, and watch, and asks what your controller actually *does* with them on every event — the level-triggered, idempotent reconcile loop, and the deletion/finalizer lifecycle that is the other half of the contract.

## References & further reading

**Primary sources — specifications and conventions**
- [Kubernetes API concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/) — the authoritative spec for `resourceVersion` semantics, the GET/LIST/WATCH resource-version tables, bookmarks, streaming lists, pagination and continue-token expiry, the patch content-type table, dry run, and field validation levels.
- [API conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) (kubernetes/community) — the normative rules for spec/status separation, `generation` vs `observedGeneration`, the standard condition fields, and the optimistic-concurrency contract (`409` → re-GET → re-apply → retry). Read the "Objects / Metadata" and "Spec and Status" sections closely.
- [Server-Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/) — `managedFields` structure, conflict resolution (force / drop / co-own), field-ownership transfer, the `listType`/`listMapKey`/`mapType` merge markers, and the RBAC requirement for `create` alongside `patch`.
- [Update API objects in place using kubectl patch](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/) — the source of the three-way patch comparison: the containers merge example, the tolerations counter-example, `$retainKeys`, and the statement that strategic merge patch is not supported for custom resources.
- [Custom resource definitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) — CRD structure, subresources, and the `Established` condition.

**Primary sources — source code**
- [`k8s.io/apimachinery`](https://pkg.go.dev/k8s.io/apimachinery) — `pkg/apis/meta/v1/types.go` is where the `ObjectMeta`, `OwnerReference`, `ManagedFieldsEntry`, `APIResource`, and `APIResourceList` field documentation in this lesson comes from; `pkg/api/meta/restmapper.go` holds `DefaultRESTMapper` and `UnsafeGuessKindToResource`; `pkg/util/managedfields/internal/conflict.go` builds the "Apply failed with N conflicts" message.
- [`k8s.io/client-go`](https://github.com/kubernetes/client-go) — read `discovery/`, `restmapper/`, `dynamic/`, and `util/retry/util.go` (`DefaultRetry` = 5 steps × 10 ms, `DefaultBackoff` = 4 steps × 10 ms × factor 5) to see all of this in working code.
- [`sigs.k8s.io/controller-runtime`](https://github.com/kubernetes-sigs/controller-runtime) — `pkg/client/apiutil/restmapper.go` for the dynamic mapper that reloads a group on a `NoMatchError`, and `pkg/client` for `client.Apply`, `client.FieldOwner`, and `client.MergeFrom`.

**Real-world engineering**
- Tim Ebert (Gardener), ["Kubernetes Controllers at Scale: Clients, Caches, Conflicts, Patches Explained"](https://medium.com/@timebertt/kubernetes-controllers-at-scale-clients-caches-conflicts-patches-explained-aa0f7a8b4332) — client/cache/conflict/patch behaviour on a production fleet.
- ymmt2005, ["An example of using dynamic client of k8s.io/client-go"](https://ymmt2005.hatenablog.com/entry/2020/04/14/An_example_of_using_dynamic_client_of_k8s.io/client-go) — RESTMapper → GVR → dynamic client in working code.
- Kubernetes blog, ["Server Side Apply Is Great And You Should Be Using It"](https://kubernetes.io/blog/2022/10/20/advanced-server-side-apply/) — the ownership model explained by the people who built it, with the controller-migration hazards spelled out.
- Uber Engineering, ["Migrating Uber's Compute Platform to Kubernetes"](https://www.uber.com/blog/migrating-ubers-compute-platform-to-kubernetes-a-technical-journey/) — a production job controller reading and reconciling CRD-addressed objects.

**Deeper dives**
- "Kubernetes Programming with Go" (Philippe Martin, Apress 2023) — the best from-first-principles treatment of schemes, codecs, GVK/GVR, RESTMapper, and unstructured access; the module's spine text for this layer. Deep-read the apimachinery and serialization chapters and do the examples.
- [KEP-555, Server-Side Apply](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/555-server-side-apply/README.md) — the design rationale, including why `managedFields` replaced the `last-applied-configuration` annotation.
