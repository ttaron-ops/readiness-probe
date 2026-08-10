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
sources: 7
---

# 02.2 · API machinery

> **Concept.** The type system, addressing, and serialization layer under every client, informer, and CRD — GVK vs GVR, the RESTMapper, the scheme and codecs, typed vs unstructured access, and watch/resourceVersion semantics.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Lesson 01 established the structural fact everything else in this module builds on: every component is a client of the apiserver, and the apiserver is the only client of etcd. That lesson treated "a client of the apiserver" as a black box — some code somewhere calls a REST endpoint and gets an object back. This lesson opens that box. Every client you'll ever write — a typed clientset call, a dynamic-client scan, an informer, a webhook handler — is `apimachinery` types wearing a different hat: the same GVK/GVR addressing, the same scheme-driven codecs, the same watch/`resourceVersion` protocol. Once you know this layer, you can read the source of client-go, controller-runtime, Helm, or `kubectl` itself and recognize the same pattern every time, instead of learning each tool's API surface from scratch.

## Why this matters

This is the driest lesson in the module and the one everything else stands on. Informers, clients, CRDs, admission — all of it is `apimachinery` types wearing different hats, and you cannot debug any of them from the outside if you don't know how an object is addressed, typed, and serialized underneath. It's also a reliable interview filter: "what's the difference between a GVK and a GVR, and who resolves one to the other?" separates people who *use* controllers from people who *build* them — it's a checkpoint probe for this module for exactly that reason. Your capstone operator registers three CRDs into a scheme and reads Nodes/Pods through both typed and dynamic clients — you'll touch every concept here on day one of the operator, and get it wrong (a stale RESTMapper, a `410 Gone` treated as an alert-worthy error, a mutated cache object) in ways that look like "impossible" bugs if you don't know this layer.

## What's new here (calibration)

A CKA holder already thinks in `kubectl get pods -n foo`, knows `apiVersion: apps/v1`, `kind: Deployment`, has run `kubectl api-resources` and seen `verbs`, `shortnames`, `namespaced`. That is the *operator's* view: the API as a REST surface you address with a CLI. This lesson does **not** re-teach that surface. What it adds is the machinery underneath the CLI:

- The `apiVersion`/`kind` you type is a **GVK** (a *type*); the plural lowercase `deployments` in the URL path is a **GVR** (a *resource*); a **RESTMapper** converts between them, and it's discovery-driven, not hardcoded.
- Every object your code holds is a `runtime.Object` decoded by a **codec** against a **scheme** — the registry mapping Go structs to GVKs.
- `kubectl get -w` is a **watch** stream keyed by **resourceVersion**, and the `410 Gone` you've maybe seen in logs is a first-class protocol event, not a random error.
- Your CRD isn't special — installing it just adds rows to discovery so the RESTMapper can resolve its GVK↔GVR like any built-in — and that discovery is cached, with a real refresh lag you'll hit in practice.

## Core concepts

**GVK vs GVR.** A **GroupVersionKind** names a *type*: `{Group: "apps", Version: "v1", Kind: "Deployment"}` — the identity stamped in an object's `apiVersion`+`kind`. A **GroupVersionResource** names a *REST resource* you can hit: `{Group: "apps", Version: "v1", Resource: "deployments"}` — the plural, lowercase path segment the API server serves. They are not the same and not mechanically derivable from each other (`Endpoints`→`endpoints`, not `endpointses`; `Ingress`→`ingresses`). The mapping is data, discovered at runtime.

**RESTMapper** is what resolves between them. It's populated from the API server's *discovery* document (`/apis`, `/api/v1`), which lists every resource, its GVK, whether it's namespaced, and its verbs. Given a GVK you call `RESTMapping(gvk.GroupKind(), gvk.Version)` and get back a `RESTMapping` carrying the GVR (`.Resource`) and scope (namespaced vs cluster). Dynamic and discovery-based clients *must* do this; a typed client has the mapping compiled in.

```go
gvk := schema.GroupVersionKind{Group: "apps", Version: "v1", Kind: "Deployment"}
mapping, _ := restMapper.RESTMapping(gvk.GroupKind(), gvk.Version)
// mapping.Resource == {apps, v1, deployments}; mapping.Scope.Name() == "namespace"
```

**Discovery caching and the refresh lag.** `kubectl` and controller-runtime's `DeferredDiscoveryRESTMapper` cache the discovery document for a TTL (controller-runtime defaults to refreshing on a cache-miss, roughly every ~10 minutes otherwise) rather than re-fetching it on every call — discovery is expensive to redo per-request. This is *why* a freshly-installed CRD can 404 for a few seconds to minutes after `kubectl apply`: the RESTMapper's cached view of discovery hasn't learned the new GVR yet, and a dynamic client resolving that CRD's GVK will fail until the cache refreshes or you force one. This is not a hypothetical edge case — DRA's own `resource.k8s.io/v1` types went through exactly this discovery-then-resolve dance as they moved through beta to GA, and any operator that installs its own CRDs and immediately tries to use them (a common install-then-reconcile pattern) needs to either tolerate the lag or force a discovery refresh.

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

**Server-Side Apply (SSA) and `managedFields`.** SSA is fundamentally a scheme/apimachinery-layer concept, not just a `kubectl apply` flag: every write made via SSA (`PATCH` with `Content-Type: application/apply-patch+yaml`, or an equivalent typed call) is tracked in `metadata.managedFields` as a set of field paths owned by a named **manager** (`FieldOwner`). When two different managers write to the *same* field, the apiserver detects the conflict at the API-machinery layer — before your controller's `Reconcile` ever runs — and either rejects the write (unless `force: true`) or resolves it by ownership transfer. This matters directly for a controller that patches only *some* fields of an object other things also write to (like a Node's labels, or a shared status object): SSA lets you own exactly the fields you write and coexist with the kubelet or other controllers writing different fields on the same object, with the conflict detection happening in apimachinery, not in your reconcile logic. (CRD-authoring implications of SSA — like which fields to mark as server-side-apply-friendly — are covered in lesson 05; this lesson only needs the mechanism.)

## Perspectives

**Developer perspective.** You almost always want typed clients for your own CRDs — compile-time safety and refactor support are worth the coupling. Dynamic/unstructured is a tool for *generic* tooling (garbage collection, `kubectl`, policy engines, a generic cost-scanner) that must handle kinds it's never compiled against. Reaching for dynamic clients for your own known types is usually a sign you're solving the wrong problem with the wrong tool.

**Operator perspective.** `kubectl api-resources` and `kubectl explain` are RESTMapper and scheme output you've used for years without seeing the machinery behind them. A `410 Gone` showing up in apiserver or controller logs is the watch protocol working exactly as designed — not an error to alert on, and treating it as one wastes on-call attention on expected behavior.

**Framework-author perspective.** Every framework that talks to Kubernetes — client-go, controller-runtime, Helm, ArgoCD, `kubectl` itself — re-derives the same GVK↔GVR resolution and the same typed-vs-dynamic tradeoff. Recognizing the pattern once means you can read any of their source trees quickly, because they're all built on the same small set of apimachinery primitives.

**Failure-mode perspective.** A stale RESTMapper cache after a CRD install or upgrade is a real, reproducible "why does my controller 404 on a CRD I just applied" bug class. Knowing the fix — force a discovery refresh, or tolerate/retry through the TTL window — turns a confusing intermittent failure into an understood, five-minute diagnosis.

## Real-world use cases

- **[ymmt2005, "An example of using dynamic client of k8s.io/client-go"](https://ymmt2005.hatenablog.com/entry/2020/04/14/An_example_of_using_dynamic_client_of_k8s.io/client-go) — a practicing Kubernetes/Cybozu engineer.** Walks RESTMapper → GVR resolution → dynamic client → decoding YAML into `Unstructured` in real working code — essentially this lesson's worked example, written by someone building it for production, not a tutorial mill.
- **[Tim Ebert (Gardener project, SAP), "Kubernetes Controllers at Scale: Clients, Caches, Conflicts, Patches Explained"](https://medium.com/@timebertt/kubernetes-controllers-at-scale-clients-caches-conflicts-patches-explained-aa0f7a8b4332).** A practitioner from Gardener — a real, large production multi-cluster-management system — writing about client/cache/patch conflicts at scale; grounds the SSA and typed-client material above in what actually happens when many controllers touch the same objects at fleet scale.
- **[Uber Engineering, "Migrating Uber's Compute Platform to Kubernetes: A Technical Journey"](https://www.uber.com/blog/migrating-ubers-compute-platform-to-kubernetes-a-technical-journey/).** Reused from lesson 01 with a different lens here: Uber's job controller reads a job CRD object, resolves its identity, and reconciles it — a real production description of the GVK-addressed, scheme-decoded object your own controller will work with.

## Worked example

Trace reading a Node's GPU capacity two ways, then extend it to show Server-Side Apply field ownership.

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

**Server-Side Apply, two managers, one object.** Now add a third actor writing to the *same* Node: the kubelet writes `status.capacity` (it owns that field), and your cost operator patches a `gpu-cost.io/cost-score` label via SSA:

```go
patch := &corev1.Node{
    TypeMeta:   metav1.TypeMeta{APIVersion: "v1", Kind: "Node"},
    ObjectMeta: metav1.ObjectMeta{Name: "gpu-node-1", Labels: map[string]string{"gpu-cost.io/cost-score": "0.83"}},
}
err := c.Patch(ctx, patch, client.Apply,
    client.FieldOwner("gpu-cost-operator"), client.ForceOwnership)
```

Inspect `kubectl get node gpu-node-1 -o yaml` afterward and look at `metadata.managedFields`: you'll see two entries, one for manager `kubelet` owning `f:status.capacity` and related fields, and one for manager `gpu-cost-operator` owning only `f:metadata.labels.gpu-cost.io/cost-score`. Because the two managers touch disjoint fields, there's no conflict — this is the concrete proof that SSA's per-field, per-manager tracking happens at the apimachinery layer, independent of either writer's reconcile logic.

## Practice

Add a small diagnostic binary to the operator repo that reads Node GPU capacity through **both** clients and reports what the machinery resolved. This feeds the [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md) deliverable's typed-vs-unstructured GPU-capacity read.

1. In `../practice/gpu-cost-operator/`, add `cmd/apimachinery-probe/main.go`. Build a `*rest.Config` from the default kubeconfig loading rules (`clientcmd.NewNonInteractiveDeferredLoadingClientConfigLoader` / `config.GetConfig()`).
2. **Typed:** `kubernetes.NewForConfig` → list Nodes → print each node's `status.capacity["nvidia.com/gpu"]` (zero-value when absent).
3. **Dynamic:** `dynamic.NewForConfig` with the GVR `{"", v1, nodes}` → list the same Nodes → pull `nvidia.com/gpu` out of the unstructured map with `unstructured.NestedString`.
4. Build a RESTMapper from discovery (`restmapper.NewDeferredDiscoveryRESTMapper` over a `discovery` client) and print the `RESTMapping` for GVK `{"", v1, Node}` — show the resolved GVR and scope.
5. **SSA field ownership check.** Patch a test label onto one node via SSA with `FieldOwner: "apimachinery-probe"`, then print that node's `metadata.managedFields` and confirm your manager owns only the label field you wrote.
6. Write a 4–6 line comment block: when you'd reach for each client, and one concrete capstone case for the dynamic one (scanning arbitrary workload kinds for GPU requests).

**Acceptance:**
- Both clients print the *same* GPU count per node (allowing string-vs-int formatting).
- Program prints the resolved GVR (`nodes`) and scope (`root`/cluster) from the RESTMapper.
- Program prints `managedFields` showing the SSA write's field ownership scoped to only the field you patched.
- The comment block names a real capstone use for the dynamic client.
- Runs against any cluster (kind/minikube fine; a fake `nvidia.com/gpu` capacity via `kubectl` on a node works if you have no real GPUs).

## Common pitfalls

1. **Assuming a CRD's plural resource name is mechanically derivable from its Kind.** It isn't — `Endpoints`→`endpoints`, `Ingress`→`ingresses`; it's data returned by discovery, not a pluralization rule you can hardcode.
2. **Reading `obj.GetObjectKind().GroupVersionKind()` on a typed object and getting an empty GVK, then assuming something is broken.** It's expected — `TypeMeta` is usually blank on typed objects fetched via a typed client; the scheme supplies the GVK implicitly from the Go type.
3. **Treating a `410 Gone` as an application error to alert on**, instead of the expected relist signal it is. Alert on watches that fail to *recover* after relist, not on the 410 itself.
4. **Forgetting that installing a CRD doesn't make it immediately resolvable everywhere.** Discovery caches lag, and a dynamic client (or a freshly-started controller) hitting the CRD seconds after `kubectl apply` can 404 until the RESTMapper refreshes.
5. **Mutating an object returned from a lister/cache read in place.** It's shared memory read by other goroutines concurrently — an apimachinery-adjacent footgun that shows up as "impossible" cross-goroutine bugs that only reproduce under concurrent load. Always `DeepCopy()` before mutating.

## Self-check

- GVK vs GVR — who resolves one to the other and how? **Answer:** A **GVK** (`apps/v1`, `Deployment`) names a Go *type*; a **GVR** (`apps/v1`, `deployments`) names the REST *resource path*. The **RESTMapper** resolves between them, driven by the API server's **discovery** document (which lists every resource, its kind, scope, and verbs). Given a GVK you call `RESTMapping(gvk.GroupKind(), version)` to get the GVR and namespaced/cluster scope. Typed clients have the mapping compiled in; dynamic/generic clients must consult the RESTMapper — which is also why a just-installed CRD needs a discovery refresh before its GVR resolves.
- Typed vs unstructured client — when each, and what does the scheme do? **Answer:** Use a **typed** client for the fixed set of kinds you compile against (built-ins, your own CRDs) — you get `*corev1.Node` with real fields and compile-time safety. Use an **unstructured/dynamic** client when the kinds are open or discovered at runtime (generic tooling, scanning arbitrary workloads) — objects are `map[string]interface{}` addressed by GVR. The **scheme** is the registry mapping Go structs ↔ GVKs (plus conversions and DeepCopy); codecs use it to decode wire bytes into the right struct and to stamp the GVK back on encode. Typed access needs the type registered in the scheme; unstructured access does not.
- What does resourceVersion guarantee on a watch, and what is a 410 Gone? **Answer:** `resourceVersion` is an opaque token (etcd mod-revision) that marks a point in the change stream — you resume a watch by passing the last RV you saw, and every event carries the object's new RV. It's not an arithmetic number; only "resume from here" and optimistic-concurrency equality (stale RV on `Update` → `409 Conflict`) are meaningful. **410 Gone** means the RV you're resuming from is older than the server's bounded history window (etcd compaction / watch-cache horizon, ~5 min default): the gap can't be replayed. The required response is to drop the resume point, do a fresh `LIST` for current state and a new RV, and restart the watch — exactly what informers do automatically on relist.
- Why does `kubectl apply` of a fresh CRD sometimes 404 for a few seconds afterward? **Answer:** Installing a CRD adds a new GVR to the apiserver's discovery document, but clients (including `kubectl` and controller-runtime's `DeferredDiscoveryRESTMapper`) cache discovery for a TTL rather than re-fetching it per request. Until that cache refreshes — either on its own schedule or forced by a cache-miss retry — the RESTMapper can't resolve the new CRD's GVK↔GVR, so a request against it 404s even though the CRD object itself was created successfully.
- When would you reach for the dynamic client over a typed client in the capstone operator specifically? **Answer:** When scanning arbitrary workload kinds you don't compile against — e.g. a generic "which objects request GPUs" scan across Deployments, StatefulSets, Jobs, and any CRD-based workload type a cluster happens to run, where the set of kinds isn't known at build time. For the operator's own CRDs (`GPUCostPolicy`, `WorkloadCost`, `Budget`) and known built-ins (`Node`, `Pod`), the typed client is the right default.

## Connections & what's next

This lesson is the layer every subsequent lesson in the module quietly depends on: lesson 04's informers are Reflectors decoding watch events through exactly this scheme/codec machinery; lesson 05's CRD design leans on the scheme registration and SSA field-ownership model covered here; lesson 06's `Status().Update()` vs `Update()` distinction is a scheme/subresource-addressing question underneath the controller-runtime API. The `410 Gone` and RESTMapper-refresh failure modes from this lesson resurface as concrete debugging scenarios in the module's checkpoint.

Next: **[02.3 · Reconciliation model](03-reconciliation-model.md)** takes the objects this lesson taught you to address, type, and watch, and asks what your controller actually *does* with them on every event — the level-triggered, idempotent reconcile loop that is the heart of the whole module.

## References & further reading

**Primary sources**
- [Kubernetes API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/) — the authoritative spec for watch semantics, resourceVersion, list-then-watch, and the 410 relist contract; read the "Efficient detection of changes" and resourceVersion sections deeply.
- [Server-Side Apply docs](https://kubernetes.io/docs/reference/using-api/server-side-apply/) — read for the `managedFields`/`FieldOwner`/conflict-resolution contract behind the SSA worked example above.
- [client-go](https://github.com/kubernetes/client-go) — read `dynamic/`, `discovery/`, `restmapper/`, `tools/clientcmd` to see GVR resolution in real code.
- [`pkg.go.dev/k8s.io/apimachinery`](https://pkg.go.dev/k8s.io/apimachinery) — godoc for the scheme, codec, and unstructured types this lesson is built on.

**Real-world engineering blogs**
- ymmt2005, ["An example of using dynamic client of k8s.io/client-go"](https://ymmt2005.hatenablog.com/entry/2020/04/14/An_example_of_using_dynamic_client_of_k8s.io/client-go) — RESTMapper → GVR → dynamic client, from a real practicing engineer.
- Tim Ebert (Gardener), ["Kubernetes Controllers at Scale: Clients, Caches, Conflicts, Patches Explained"](https://medium.com/@timebertt/kubernetes-controllers-at-scale-clients-caches-conflicts-patches-explained-aa0f7a8b4332) — client/cache/SSA-conflict behavior at real production scale.
- Uber Engineering, ["Migrating Uber's Compute Platform to Kubernetes"](https://www.uber.com/blog/migrating-ubers-compute-platform-to-kubernetes-a-technical-journey/) — a real job controller reading and reconciling CRD-addressed objects.

**Deeper dives**
- "Kubernetes Programming with Go" (Philippe Martin) — the single best from-first-principles treatment of schemes, codecs, GVK/GVR, RESTMapper, and unstructured access; the module's spine text for this layer — deep-read the apimachinery/serialization chapters and do the examples.
