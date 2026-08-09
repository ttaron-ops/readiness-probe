---
lesson: "02.10"
title: "Capstone build — gpu-cost-operator v0.1 under envtest"
module: "02"
concept: "Capstone build — gpu-cost-operator v0.1 under envtest"
status: not-started
est_time: "16h"
artifacts: []
---

# 02.10 · Capstone build — gpu-cost-operator v0.1 under envtest

> **Concept.** Assemble CRDs, controller, finalizer, validating webhook, and least-privilege RBAC into `gpu-cost-operator` v0.1, and prove it with envtest: table-driven reconcile tests that assert idempotency and level-triggering, plus a webhook test — then demo and defend it.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Why this matters

Everything in this module has been a component: the reconcile loop, the CRD, the finalizer, the webhook, RBAC. This lesson is the **integration** — wiring them into one operator that a panel would accept as v0.1, and backing it with tests that prove the properties that *actually* distinguish a controller that survives production from a demo that passes once.

The bar at CoreWeave/NVIDIA is not "it reconciled in my kind cluster." It is: **can you demo it, and can you defend the test strategy?** "How do you know your reconcile is idempotent?" and "what does envtest *not* cover, and how do you compensate?" are the questions that separate someone who followed the kubebuilder tutorial from someone who owns operators at scale. A green `make test` that runs a real apiserver, asserts idempotency and level-triggering, and includes a webhook test *is* the differentiator. This lesson is the checklist plus the testing depth to get there.

## From operating to extending

As an operator you *trust* controllers to converge and *observe* the result with `kubectl`. As the author you must **prove convergence yourself, without a cluster**, in CI, deterministically. That demands understanding what a test harness gives you and — more importantly — what it withholds.

The shift is from "did the pod come up?" to three engineering commitments:

- **Idempotency** — reconciling the same observed state twice produces no additional change. Not aspirational; asserted.
- **Level-triggering** — the controller acts on *current observed state*, not on a stream of edge events. It must converge from *any* starting state, including one it never saw the transition into.
- **Least privilege** — the RBAC you ship is exactly the verbs the controller uses, generated from markers and verified, not `cluster-admin`.

## Core notes

### 1. What v0.1 must contain (integration checklist)

Wire these together; each was built earlier in the module, now they compose:

- **CRD(s)** — your `GPUCostPolicy` (and any status subresource). Types + deepcopy generated (`make generate`), CRD manifests generated (`make manifests`), OpenAPI validation on the spec.
- **Controller / reconciler** — the `Reconcile` loop: fetch object, handle not-found (deleted) cleanly, compute desired state, apply, update `.status` with conditions, return `ctrl.Result`. Owner references / `SetControllerReference` on anything it creates so GC is automatic.
- **Finalizer** — a named finalizer added on first reconcile; on deletion (`DeletionTimestamp != nil`) run external cleanup, then **remove the finalizer** so the API server can delete the object. Cleanup must be idempotent (deletion reconciles repeatedly).
- **Validating webhook** — reject bad `GPUCostPolicy` specs the CRD's structural schema can't express (cross-field rules, semantic constraints). Wired through the manager's webhook server with cert paths envtest can provide.
- **Least-privilege RBAC** — `+kubebuilder:rbac` markers generate a Role/ClusterRole with exactly the groups/resources/verbs used. No wildcards you don't need; watch/list/get + the specific writes.
- **main.go / manager wiring** — scheme registration, manager options (leader election, metrics, health/readiness probes), controller `SetupWithManager`, webhook `SetupWebhookWithManager`.

Acceptance for the lesson = the **module checkpoint** ([`../checkpoint.md`](../checkpoint.md)); the full spec is the deliverable README ([`../practice/gpu-cost-operator/README.md`](../practice/gpu-cost-operator/README.md)).

### 2. envtest — what it provides and what it does not

**envtest** (from controller-runtime) starts a **real kube-apiserver and a real etcd** as local binaries — no Docker, no full cluster. Your controller talks to a genuine API server: CRDs install, admission webhooks fire, RBAC objects exist, watches and the informer cache behave for real, optimistic concurrency (resourceVersion conflicts) is real. This is dramatically higher fidelity than a fake client for anything touching API semantics.

**What it does NOT provide: there is no kubelet, no scheduler, no controller-manager, no CNI, no container runtime.** Concretely:

- Pods **never run**. Create a Pod and it stays Pending forever — nothing schedules or starts it. You cannot assert a container came up, a probe passed, or logs appeared.
- No built-in controllers act. A Deployment does **not** produce ReplicaSets or Pods; a Job does not create Pods; owner-reference **garbage collection does not happen** (no GC controller) unless you delete manually or enable it.
- No real nodes, no real GPUs, no DRA drivers, no scheduling decisions.

**Implication for testing:** envtest verifies *your controller's* behavior against the API — reconcile writes, status updates, finalizer add/remove, webhook accept/reject, RBAC-shaped clients. It does **not** verify anything downstream that depends on other components doing their jobs. To test "the workload actually ran on a GPU node" you need an e2e cluster (kind/real). Know this boundary cold — self-check (a).

**Setup:** binaries are provisioned by **`setup-envtest`** (`sigs.k8s.io/controller-runtime/tools/setup-envtest`): `setup-envtest use <k8s-version>` downloads matching apiserver/etcd/kubectl and prints a path you export as `KUBEBUILDER_ASSETS`. Kubebuilder's `make test` runs this for you. Pin the Kubernetes version explicitly so CI and laptops match. In `TestMain`/suite setup you configure `envtest.Environment{CRDDirectoryPaths: [...], WebhookInstallOptions: ...}`, call `Start()`, build a client from the returned `*rest.Config`, and `Stop()` at the end.

### 3. envtest vs fake client — when each

- **fake client** (`sigs.k8s.io/controller-runtime/pkg/client/fake`) — an in-memory object tracker, no server. Fast, zero setup, great for **unit-testing pure reconcile logic**: given these objects in the store, does one `Reconcile` call produce the right writes? Tradeoffs: it *approximates* API behavior and historically diverged on the sharp edges — status subresource handling, field validation, defaulting, admission/webhooks (it runs none), and some resourceVersion/conflict semantics. **Webhooks never fire under the fake client**, so it cannot test your validating webhook at all.
- **envtest** — real apiserver, higher fidelity, slower (seconds to start). Use it when the *interaction with the API* is what's under test: webhook admission, CRD structural validation, status subresource, optimistic-concurrency conflicts, finalizer + deletion flow, RBAC.

Rule of thumb: **fake client for fast, focused reconcile-logic unit tests; envtest for anything that exercises real API semantics or admission.** Your suite should have both. Self-check (c).

### 4. Table-driven reconcile tests

Standard Go table-driven form fits reconcilers well. Each row: a name, a set of seed objects, the request, and the expected post-conditions (object state, status conditions, error). Loop, subtest per row (`t.Run`), fresh client per row for isolation.

Skeleton (illustrative):

```go
cases := []struct {
    name    string
    objects []client.Object // seed state
    req     ctrl.Request
    verify  func(t *testing.T, c client.Client)
    wantErr bool
}{ /* rows */ }

for _, tc := range cases {
    t.Run(tc.name, func(t *testing.T) {
        r := &GPUCostPolicyReconciler{Client: c, Scheme: scheme}
        _, err := r.Reconcile(ctx, tc.req)
        // assert err vs tc.wantErr, then tc.verify(t, c)
    })
}
```

### 5. Asserting idempotency

Idempotency is proven by **reconciling twice and showing the second call is a no-op.** Two complementary techniques:

1. **State stability** — call `Reconcile`, snapshot the object's `resourceVersion` (and the fields you manage). Call `Reconcile` again on the same state. Assert `resourceVersion` is **unchanged** — a no-op write bumps nothing. (Under the fake client be aware of its resourceVersion behavior; under envtest it's real. Comparing the managed fields directly is the most robust form.)
2. **No spurious requeue / no error** — the second call returns `Result{}` (or the same steady requeue) and `nil`.

The trap idempotency catches: a reconcile that *always* writes (e.g. unconditionally `Update`s status, or re-adds something already present) will churn resourceVersions forever and hot-loop the work queue. The assertion "second reconcile changed nothing observable" is the guard. Self-check (b).

### 6. Asserting level-triggering

Level-triggered means the controller converges from **whatever the current state is**, not from a sequence of events. Test it by **seeding a partially-converged or drifted state the controller never "saw happen"** and asserting one `Reconcile` brings it to desired:

- Seed the object with status/child already partly set (as if a previous run half-finished, or a human edited it) → reconcile → asserts it converges, doesn't duplicate, doesn't error.
- Seed with a child object *missing* that should exist → reconcile → it's created.
- Seed with a child that *shouldn't* exist / drifted field → reconcile → corrected.

The key property: no row depends on a prior transition. Each is a standalone "given this level, converge." That's the difference from an edge-triggered design that would only act if it witnessed the change.

### 7. Testing the validating webhook (envtest only)

The webhook cannot be tested with the fake client. Under envtest, set `WebhookInstallOptions` on the `Environment`, register your webhook with the manager, wait for the webhook server to be ready, then drive it through a **real client**:

- **Reject path** — `Create`/`Update` a `GPUCostPolicy` that violates a rule your webhook enforces (a constraint the CRD schema can't). Assert the call returns an **error** (denied by admission) and the object was not persisted.
- **Accept path** — a valid object is created successfully.

Because envtest runs a real apiserver, the admission request genuinely reaches your webhook over TLS with envtest-managed certs — this exercises the wiring (cert paths, path/port, failure policy), not just the handler function in isolation.

## Worked example

A tight loop proving idempotency **and** level-triggering for `GPUCostPolicy` in one envtest-backed test:

1. **Suite setup** (`TestMain` or Ginkgo `BeforeSuite`): `setup-envtest use 1.34.x`, export `KUBEBUILDER_ASSETS`; start `envtest.Environment{CRDDirectoryPaths, WebhookInstallOptions}`; build the manager, register reconciler + webhook, start it; build a client.
2. **Create** a valid `GPUCostPolicy`. Reconcile (or let the manager reconcile). Assert: finalizer present, `.status` conditions set, any managed child created.
3. **Idempotency:** read the object, record `resourceVersion` and managed fields. Trigger a second `Reconcile` on the unchanged state. Assert managed fields identical and no new child created — the reconcile did nothing observable.
4. **Level-trigger — drift:** directly `Update` a managed field to a wrong value (simulating drift the controller never witnessed). Reconcile. Assert it's corrected back to desired.
5. **Level-trigger — missing child:** delete a child object the policy requires. Reconcile. Assert it's recreated (converges from the "missing" level, no error).
6. **Webhook:** `Create` an invalid `GPUCostPolicy`; assert the API call is **denied**. `Create` a valid one; assert success.
7. **Finalizer/deletion:** `Delete` the object. Reconcile. Assert cleanup ran, finalizer removed, object gone. Reconcile again on the now-absent object → clean no-op (deletion path is idempotent too).
8. **Teardown:** `Environment.Stop()`.

That single flow demonstrates every property the checkpoint asks for, all against a real API server, no live cluster.

## Practice

Build the **full module deliverable** — the complete spec lives at [`../practice/gpu-cost-operator/README.md`](../practice/gpu-cost-operator/README.md).

Wire it end to end:

- CRD(s) + generated deepcopy + manifests for `GPUCostPolicy`.
- Controller with a converging, idempotent, level-triggered `Reconcile`.
- **Finalizer** with idempotent external cleanup on deletion.
- **Validating webhook** enforcing at least one cross-field/semantic rule the schema can't.
- **Least-privilege RBAC** from `+kubebuilder:rbac` markers — exactly the verbs used.
- **Tests:** table-driven reconcile tests; **idempotency proven** (second reconcile is a no-op); **level-triggering proven** (converges from drifted/missing states); **a webhook test** (accept + reject) under envtest.

**Acceptance = the module checkpoint** ([`../checkpoint.md`](../checkpoint.md)): `make test` (envtest) is **green**, with idempotency, level-triggering, and webhook admission all demonstrated by passing tests, and you can demo the operator and defend the test strategy.

## Self-check

**(a) What does envtest provide and what does it NOT provide — and given no kubelet, what can't you test?**

**Answer:** envtest provides a **real kube-apiserver and etcd** (via `setup-envtest`), so CRD install, admission webhooks, RBAC, status subresource, watches/informers, and optimistic-concurrency conflicts all behave for real. It provides **no kubelet, no scheduler, no controller-manager, no runtime** — so **Pods never run** (they stay Pending), built-in controllers don't act (a Deployment yields no ReplicaSets/Pods, Jobs create no Pods), and **owner-reference GC doesn't happen**. You therefore **cannot** test anything downstream of other components: that a container actually started, a probe passed, a workload landed on a GPU node, or that GC cleaned up children. Those need an e2e cluster (kind/real). envtest tests *your controller against the API*, not the rest of the system.

**(b) How do you assert reconcile idempotency in a test?**

**Answer:** Reconcile once to reach steady state, then reconcile **again on the same observed state and assert nothing changes.** Concretely: snapshot the object's managed fields (and `resourceVersion`) after the first reconcile; call `Reconcile` a second time; assert the managed fields are byte-for-byte identical (and, under a real apiserver, `resourceVersion` is unchanged — a no-op does no write) and that no additional child objects were created and no error/spurious requeue was returned. A reconcile that always writes would bump resourceVersion on the second call and hot-loop the queue; the "second reconcile is a no-op" assertion is the guard.

**(c) Fake client vs envtest — when do you use each?**

**Answer:** **Fake client** (in-memory, no server) for fast, focused **unit tests of reconcile logic** — given a seeded object store, does one `Reconcile` produce the right writes? It runs **no admission/webhooks** and only approximates API semantics (status subresource, validation, conflicts), so it can't test those. **envtest** (real apiserver + etcd) for anything exercising **real API semantics or admission**: validating-webhook accept/reject, CRD structural validation, status subresource behavior, resourceVersion conflicts, the finalizer/deletion flow, RBAC-shaped clients. A good suite uses both — fake client for the bulk of fast logic tests, envtest for the interactions that only a real API server can exercise.

## Resources

- Kubebuilder Book — testing with envtest: <https://book.kubebuilder.io/reference/envtest.html>
- controller-runtime `envtest` package godoc (`Environment`, `WebhookInstallOptions`): <https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/envtest>
- `setup-envtest` — provisioning apiserver/etcd binaries, `KUBEBUILDER_ASSETS`: <https://github.com/kubernetes-sigs/controller-runtime/tree/main/tools/setup-envtest>
- **Version-sensitive:** pin the envtest Kubernetes version (`setup-envtest use <version>`) so CI and local runs match; behavior of the fake client's status/validation handling has changed across controller-runtime releases — check the version in your `go.mod`.
- fake client godoc (unit-testing reconcile logic): <https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client/fake>
