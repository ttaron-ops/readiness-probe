---
lesson: "02.2"
title: "API machinery"
module: "02"
concept: "API machinery"
status: not-started
est_time: "6h"
artifacts: []
---

# 02.2 · API machinery

> **Concept.** The type system, addressing, and serialization layer under every client, informer, and CRD — GVK vs GVR, the RESTMapper, the scheme and codecs, typed vs unstructured access, and watch/resourceVersion semantics.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Why this matters

This is the driest lesson in the module and the one everything else stands on. Informers, clients, CRDs, admission — all of it is `apimachinery` types wearing different hats, and you cannot debug any of them from the outside if you don't know how an object is addressed, typed, and serialized underneath. It's also a reliable interview filter: "what's the difference between a GVK and a GVR, and who resolves one to the other?" separates people who *use* controllers from people who *build* them. Your capstone registers three CRDs into a scheme and reads Nodes/Pods through both typed and dynamic clients — you'll touch every concept here on day one of the operator.

## From operating to extending

A CKA holder already thinks in `kubectl get pods -n foo`, knows `apiVersion: apps/v1`, `kind: Deployment`, has run `kubectl api-resources` and seen `verbs`, `shortnames`, `namespaced`. That is the *operator's* view: the API as a REST surface you address with a CLI. What you learn now is the machinery underneath that CLI:

- The `apiVersion`/`kind` you type is a **GVK** (a *type*); the plural lowercase `deployments` in the URL path is a **GVR** (a *resource*); a **RESTMapper** converts between them, and it's discovery-driven, not hardcoded.
- Every object your code holds is a `runtime.Object` decoded by a **codec** against a **scheme** — the registry mapping Go structs to GVKs.
- `kubectl get -w` is a **watch** stream keyed by **resourceVersion**, and the `410 Gone` you've maybe seen in logs is a first-class protocol event, not a random error.
- Your CRD isn't special — installing it just adds rows to discovery so the RESTMapper can resolve its GVK↔GVR like any built-in.

## Core notes

**GVK vs GVR.** A **GroupVersionKind** names a *type*: `{Group: "apps", Version: "v1", Kind: "Deployment"}` — the identity stamped in an object's `apiVersion`+`kind`. A **GroupVersionResource** names a *REST resource* you can hit: `{Group: "apps", Version: "v1", Resource: "deployments"}` — the plural, lowercase path segment the API server serves. They are not the same and not mechanically derivable from each other (`Endpoints`→`endpoints`, not `endpointses`; `Ingress`→`ingresses`). The mapping is data, discovered at runtime.

**RESTMapper** is what resolves between them. It's populated from the API server's *discovery* document (`/apis`, `/api/v1`), which lists every resource, its GVK, whether it's namespaced, and its verbs. Given a GVK you call `RESTMapping(gvk.GroupKind(), gvk.Version)` and get back a `RESTMapping` carrying the GVR (`.Resource`) and scope (namespaced vs cluster). Dynamic and discovery-based clients *must* do this; a typed client has the mapping compiled in. The RESTMapper is also why a freshly-installed CRD can 404 for a few seconds — discovery is cached, and the mapper needs a refresh to learn the new GVR.

```go
gvk := schema.GroupVersionKind{Group: "apps", Version: "v1", Kind: "Deployment"}
mapping, _ := restMapper.RESTMapping(gvk.GroupKind(), gvk.Version)
// mapping.Resource == {apps, v1, deployments}; mapping.Scope.Name() == "namespace"
```

**The scheme** (`runtime.Scheme`) is the in-process registry that ties Go types to GVKs. `scheme.AddKnownTypes(gv, &Deployment{}, ...)` registers `*Deployment` ↔ `apps/v1, Kind=Deployment`. It also stores conversions between API versions and the `DeepCopy` functions. Every client-go clientset registers its types into a scheme at init; a controller-runtime manager carries one scheme you must register your CRD types into (`AddToScheme`). The scheme answers two questions the codecs need: "given this Go object, what GVK do I stamp?" and "given this GVK on the wire, what Go struct do I allocate?"

**Codecs and serialization.** Bytes on the wire (JSON, YAML, or protobuf) become `runtime.Object`s via a **codec** built from the scheme. Decode reads the `apiVersion`+`kind`, asks the scheme for the matching type, allocates it, and unmarshals. Encode does the reverse, stamping the GVK back on. The key subtlety: a decoded typed object often has an *empty* `TypeMeta` (the GVK lives implicitly in its Go type), which is why `obj.GetObjectKind().GroupVersionKind()` can come back blank for a typed object but is always populated on an `Unstructured`. protobuf is used for built-in types on the internal wire for speed; CRDs are always JSON.

**`runtime.Object` and `metav1.Object`.** Two interfaces you'll type-assert constantly. `runtime.Object` (`GetObjectKind()`, `DeepCopyObject()`) is "anything the scheme/codecs handle" — it carries *type* identity. `metav1.Object` (`GetName()`, `GetNamespace()`, `GetLabels()`, `GetResourceVersion()`, ...) is "anything with standard metadata" — it carries *instance* identity. `metav1.ObjectMeta` embedded in your struct satisfies the latter; `metav1.TypeMeta` plus a generated `DeepCopyObject` satisfies the former. `meta.Accessor(obj)` extracts the `metav1.Object` view from any object generically — that's how framework code reads a name/namespace without knowing the concrete type.

**Typed vs unstructured clients.**

- **Typed** (client-go clientset, or controller-runtime's `client.Client` with registered types): you get `*appsv1.Deployment` with compile-time fields. Safe, autocompleted, refactorable. Requires the type registered in the scheme and known at build time — the natural default for your own CRDs and built-ins.
- **Unstructured / dynamic** (`dynamic.Interface`, `*unstructured.Unstructured`): the object is a `map[string]interface{}`; you address fields by path (`unstructured.NestedInt64(obj.Object, "status", "capacity", "nvidia.com/gpu")`). You call by **GVR**, resolved via the RESTMapper. No compile-time knowledge of the type required — essential for generic tooling that must handle CRDs it's never seen: garbage collectors, `kubectl`, policy engines, a cost operator that inspects arbitrary workloads.

Rule of thumb: typed for the handful of kinds you own and compile against; unstructured/dynamic when the *set of kinds is open* or discovered at runtime.

**Watch + resourceVersion.** Every object carries a `resourceVersion` (RV) — an opaque, per-object token backed by etcd's mod-revision. It is **not** a number to do arithmetic on; only equality and "the server understands it" are meaningful. A `WATCH` is `LIST` (returns a collection RV) then a long-lived stream of `ADDED/MODIFIED/DELETED` events, each carrying the object's new RV. You resume a watch by passing the last RV you saw. RV also powers optimistic concurrency: an `Update` sends the RV you read; if the server's is newer, you get a `409 Conflict` and must re-read.

**`410 Gone`** is the watch-specific failure: the API server keeps only a bounded history window (etcd compaction, ~5 min default watch-cache horizon). If you resume from an RV older than that window, the server can't replay the gap and returns `410 Gone` as a watch event/status. The contract: **on 410, discard your resume point, do a fresh LIST to get a current RV and full state, and restart the watch from there** ("list-then-watch relist"). Informers do this automatically — a 410 is a normal, expected relist, not an error to alert on.

**How CRDs slot in.** A `CustomResourceDefinition` is itself an API object; applying it makes the API server *serve a new GVR* and add it to discovery. From that moment the RESTMapper can resolve `gpucostpolicies` ↔ `GPUCostPolicy`, dynamic clients can address it by GVR, and — once you register your generated Go types into a scheme — typed clients work too. CRDs get no privileged path: same GVK/GVR, same scheme/codec, same watch/RV semantics as `Pod`. That uniformity is the whole point of the machinery.

## Worked example

Trace reading a Node's GPU capacity two ways.

**Typed path** (`nvidia.com/gpu` off `corev1.Node`):

```go
node, _ := clientset.CoreV1().Nodes().Get(ctx, "gpu-node-1", metav1.GetOptions{})
// Compiled mapping: CoreV1().Nodes() already knows GVR {"", v1, nodes}, cluster-scoped.
qty := node.Status.Capacity["nvidia.com/gpu"] // resource.Quantity, typed field access
gpus, _ := qty.AsInt64()
```

No RESTMapper call happens — the clientset has `nodes` compiled in. The returned `*corev1.Node` has (typically) empty `TypeMeta`; its GVK is implied by the Go type via the scheme.

**Dynamic path** (same data, no compiled type):

```go
gvr := schema.GroupVersionResource{Group: "", Version: "v1", Resource: "nodes"}
u, _ := dyn.Resource(gvr).Get(ctx, "gpu-node-1", metav1.GetOptions{})
// u is *unstructured.Unstructured; u.Object is map[string]interface{}, GVK explicitly set.
gpus, found, _ := unstructured.NestedString(u.Object,
    "status", "capacity", "nvidia.com/gpu") // "8", as a string — no Quantity type here
```

Here you supply the **GVR** directly (or resolve it from the GVK `{"",v1,Node}` via `restMapper.RESTMapping`). The `Unstructured` has its GVK explicitly populated in the map, and capacity is a bare string — you own the parsing. Same bytes came off the API server; the codec decoded them into a struct in path one and a generic map in path two. Both report `8`. The typed path gives you `resource.Quantity`, compile-time safety, and refactor support; the dynamic path handles kinds you don't compile against and is what a generic cost-scanner needs.

## Practice

Add a small diagnostic binary to the operator repo that reads Node GPU capacity through **both** clients and reports what the machinery resolved.

1. In `../practice/gpu-cost-operator/`, add `cmd/apimachinery-probe/main.go`. Build a `*rest.Config` from the default kubeconfig loading rules (`clientcmd.NewNonInteractiveDeferredLoadingClientConfigLoader` / `config.GetConfig()`).
2. **Typed:** `kubernetes.NewForConfig` → list Nodes → print each node's `status.capacity["nvidia.com/gpu"]` (zero-value when absent).
3. **Dynamic:** `dynamic.NewForConfig` with the GVR `{"", v1, nodes}` → list the same Nodes → pull `nvidia.com/gpu` out of the unstructured map with `unstructured.NestedString`.
4. Build a RESTMapper from discovery (`restmapper.NewDeferredDiscoveryRESTMapper` over a `discovery` client) and print the `RESTMapping` for GVK `{"", v1, Node}` — show the resolved GVR and scope.
5. Write a 4–6 line comment block: when you'd reach for each client, and one concrete capstone case for the dynamic one (scanning arbitrary workload kinds for GPU requests).

**Acceptance:**
- Both clients print the *same* GPU count per node (allowing string-vs-int formatting).
- Program prints the resolved GVR (`nodes`) and scope (`root`/cluster) from the RESTMapper.
- The comment block names a real capstone use for the dynamic client.
- Runs against any cluster (kind/minikube fine; a fake `nvidia.com/gpu` capacity via `kubectl` on a node works if you have no real GPUs).

## Self-check

**(a) GVK vs GVR — who resolves one to the other and how?**
**Answer:** A **GVK** (`apps/v1`, `Deployment`) names a Go *type*; a **GVR** (`apps/v1`, `deployments`) names the REST *resource path*. The **RESTMapper** resolves between them, driven by the API server's **discovery** document (which lists every resource, its kind, scope, and verbs). Given a GVK you call `RESTMapping(gvk.GroupKind(), version)` to get the GVR and namespaced/cluster scope. Typed clients have the mapping compiled in; dynamic/generic clients must consult the RESTMapper — which is also why a just-installed CRD needs a discovery refresh before its GVR resolves.

**(b) Typed vs unstructured client — when each, and what does the scheme do?**
**Answer:** Use a **typed** client for the fixed set of kinds you compile against (built-ins, your own CRDs) — you get `*corev1.Node` with real fields and compile-time safety. Use an **unstructured/dynamic** client when the kinds are open or discovered at runtime (generic tooling, scanning arbitrary workloads) — objects are `map[string]interface{}` addressed by GVR. The **scheme** is the registry mapping Go structs ↔ GVKs (plus conversions and DeepCopy); codecs use it to decode wire bytes into the right struct and to stamp the GVK back on encode. Typed access needs the type registered in the scheme; unstructured access does not.

**(c) What does resourceVersion guarantee on a watch, and what is a 410 Gone?**
**Answer:** `resourceVersion` is an opaque token (etcd mod-revision) that marks a point in the change stream — you resume a watch by passing the last RV you saw, and every event carries the object's new RV. It's not an arithmetic number; only "resume from here" and optimistic-concurrency equality (stale RV on `Update` → `409 Conflict`) are meaningful. **410 Gone** means the RV you're resuming from is older than the server's bounded history window (etcd compaction / watch-cache horizon, ~5 min default): the gap can't be replayed. The required response is to drop the resume point, do a fresh `LIST` for current state and a new RV, and restart the watch — exactly what informers do automatically on relist.

## Resources

- **"Kubernetes Programming with Go" — API machinery chapters** (Philippe Martin, https://www.amazon.com/dp/1484290259). *What-for:* the single best from-first-principles treatment of schemes, codecs, GVK/GVR, RESTMapper, and unstructured access — the exact layer this lesson covers. **Deep-read** the apimachinery/serialization chapters; do the examples. *Why:* controller-runtime hides all of this, and you can only debug what you understand underneath.
- **Kubernetes API Concepts** (https://kubernetes.io/docs/reference/using-api/api-concepts/). *What-for:* the authoritative spec for watch semantics, resourceVersion, list-then-watch, and the 410 relist contract. **Deep-read** the "Efficient detection of changes" and resourceVersion sections; **skim** the rest. *Why:* it's the source of truth for the watch behavior your informers depend on.
- **client-go** (https://github.com/kubernetes/client-go). *What-for:* the real implementations — `kubernetes/`, `dynamic/`, `discovery/`, `restmapper/`, `tools/clientcmd`. **Skim** `dynamic/` and `restmapper/` to see GVR resolution in code (godoc mirror: https://pkg.go.dev/k8s.io/apimachinery). *Why:* your practice binary imports these packages directly, and reading them turns the abstractions concrete.
