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
sources: 9
---

# 02.10 · Capstone build — gpu-cost-operator v0.1 under envtest

> **Concept.** Assemble CRDs, controller, finalizer, validating webhook, and least-privilege RBAC into `gpu-cost-operator` v0.1, and prove it with envtest: table-driven reconcile tests that assert idempotency and level-triggering, plus a webhook test — then demo and defend it.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Lesson 09 kept the scheduling layer at design-doc altitude on purpose. Everything else in this module — component internals (01), API machinery (02), the reconcile model (03), informers/caches/workqueues (04), CRD design (05), controller-runtime mechanics + finalizers/GC (06), kubebuilder/RBAC (07), and admission webhooks (08) — has been built as a separate, testable piece. This lesson is where those pieces stop being separate. You wire CRDs, reconciler, finalizer, webhook, and RBAC into one operator and prove — with tests, not a demo click-through — that it has the properties a production controller needs: idempotency, level-triggering, and least privilege. This is also the module's **exit gate**: passing this lesson's acceptance criteria *is* passing the [module checkpoint](../checkpoint.md).

## Why this matters

Everything in this module has been a component: the reconcile loop, the CRD, the finalizer, the webhook, RBAC. This lesson is the **integration** — wiring them into one operator that a panel would accept as v0.1, and backing it with tests that prove the properties that *actually* distinguish a controller that survives production from a demo that passes once.

The bar at CoreWeave/NVIDIA is not "it reconciled in my kind cluster." It is: **can you demo it, and can you defend the test strategy?** "How do you know your reconcile is idempotent?" and "what does envtest *not* cover, and how do you compensate?" are the questions that separate someone who followed the kubebuilder tutorial from someone who owns operators at scale. A green `make test` that runs a real apiserver, asserts idempotency and level-triggering, and includes a webhook test *is* the differentiator. This lesson is the checklist plus the testing depth to get there — and it's also literally the debug prompt the checkpoint asks cold: *"your controller stopped reconciling"* is a question you can only answer with confidence once you've built and tested the whole assembled system, not a component in isolation.

## What's new here (calibration)

As an operator you *trust* controllers to converge and *observe* the result with `kubectl`. As the author you must **prove convergence yourself, without a cluster**, in CI, deterministically. That demands understanding what a test harness gives you and — more importantly — what it withholds. Already know / skip vs genuinely new:

- **Already know, skip:** Go table-driven test structure, `t.Run` subtests, `go test` mechanics — you've written idiomatic Go tests for years (Module 01).
- **Already know, skip:** what "green CI" means operationally and why you gate merges on it.
- **New here:** treating idempotency and level-triggering as *properties to assert in code*, not just design intentions — the senior-level upgrade over a typical tutorial's happy-path-only test.
- **New here:** the precise envtest fidelity boundary — what a real apiserver-and-etcd harness gives you for free (admission, RBAC, status subresources, optimistic concurrency) versus what no test harness below a real cluster can ever give you (scheduling, kubelet, GC controller, real GPUs).
- **New here:** CI cost/time tradeoffs at fleet scale — envtest spins up real binaries (seconds per run, not milliseconds), so an all-envtest test matrix is a real CI-latency and cost concern across 40 clusters' worth of pipelines, not just a fidelity question.

The shift is from "did the pod come up?" to three engineering commitments:

- **Idempotency** — reconciling the same observed state twice produces no additional change. Not aspirational; asserted.
- **Level-triggering** — the controller acts on *current observed state*, not on a stream of edge events. It must converge from *any* starting state, including one it never saw the transition into.
- **Least privilege** — the RBAC you ship is exactly the verbs the controller uses, generated from markers and verified, not `cluster-admin`.

## Core concepts

### 1. What v0.1 must contain (integration checklist)

Wire these together; each was built earlier in the module, now they compose:

- **CRD(s)** — your `GPUCostPolicy`, `WorkloadCost`, and `Budget` (each with a status subresource). Types + deepcopy generated (`make generate`), CRD manifests generated (`make manifests`), CEL/OpenAPI validation on the spec.
- **Controller / reconciler** — the `Reconcile` loop: fetch object, handle not-found (deleted) cleanly, compute desired state, apply, update `.status` with conditions, return `ctrl.Result`. Owner references / `SetControllerReference` on anything it creates so GC is automatic.
- **Finalizer** — a named finalizer added on first reconcile; on deletion (`DeletionTimestamp != nil`) run external cleanup (record final spend), then **remove the finalizer** so the API server can delete the object. Cleanup must be idempotent (deletion reconciles repeatedly).
- **Validating webhook** — reject a workload whose projected cost would breach an active `Budget` set to `block`, plus any bad spec the CRD's structural schema can't express. Wired through the manager's webhook server with cert paths envtest can provide.
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

**CI cost/time tradeoff.** envtest is real binaries, not an in-memory fake: each suite run costs seconds, not milliseconds, to bring the apiserver and etcd up. Multiply that by a large test matrix and it becomes a genuine CI-latency and pipeline-cost line item — this learner will own CI economics across a fleet of clusters, not just correctness, so "fake client for the bulk of tests, envtest only where API semantics actually matter" (§3) is as much a cost decision as a fidelity one.

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

- **Reject path** — `Create`/`Update` a workload that would breach an active `Budget` (or a `GPUCostPolicy` that violates a rule your webhook enforces). Assert the call returns an **error** (denied by admission) and the object was not persisted.
- **Accept path** — a valid object is created successfully.

Because envtest runs a real apiserver, the admission request genuinely reaches your webhook over TLS with envtest-managed certs — this exercises the wiring (cert paths, port, failure policy), not just the handler function in isolation.

## Perspectives

**Test-design perspective.** Table-driven tests plus the fake/envtest split is a general Go testing pattern; this lesson's specific application — idempotency and level-triggering as *properties to assert*, not just "does it not crash" — is the senior-level upgrade over a typical tutorial's happy-path-only test.

**Operator/production perspective.** The explicit list of what envtest does **not** cover (no kubelet, no scheduler, no GC, no real Pods running) is a map of exactly where you still need a real (kind/staging) cluster before calling something production-ready. Drawing that boundary correctly is itself part of the deliverable, not a caveat to skim past.

**Interview/defensibility perspective.** "How do you know your reconcile is idempotent" and "what does envtest not cover, and how do you compensate" are named above as *the* differentiating interview questions. Treat the Practice section's deliverable as literally rehearsing the answer with working code, not just checking a box — you should be able to point at the specific test that proves each claim.

**Systems-integration perspective.** This lesson is where every other lesson's piece (CRD schema, reconcile idempotency, finalizer, RBAC, webhook) has to compose correctly *together* for the first time. Bugs that only appear at integration — an RBAC gap only visible once the webhook and controller both run, a cert-path mismatch only visible under envtest's real TLS — are the entire point of this lesson, and the reason it can't be skipped or done piecemeal.

## Real-world use cases

- **SuperOrbital, "Testing Production Kubernetes Controllers"** — <https://superorbital.io/blog/testing-production-controllers/> — a Kubernetes training/consultancy firm's practitioner writeup recommending the same envtest+table-driven testing shape this lesson uses; corroborates the approach as industry-standard rather than idiosyncratic.
- **InfraCloud, "Testing Kubernetes Operators using EnvTest"** — <https://www.infracloud.io/blogs/testing-kubernetes-operator-envtest/> — an independent DevOps/Kubernetes consultancy describing envtest's "no kubelet, no controller-manager" fidelity boundary in the same terms as this lesson — a second, independent corroboration of where the line sits.
- **Marc Nuri (Red Hat, `fabric8` maintainer), "How to Test Kubernetes Applications in Go with EnvTest"** — <https://blog.marcnuri.com/go-testing-kubernetes-applications-envtest> — a named, verifiable individual practitioner's walkthrough of envtest setup and suite structure, useful as a second concrete reference alongside the Kubebuilder Book.

## Worked example

A tight loop proving idempotency **and** level-triggering for `GPUCostPolicy`/`WorkloadCost`/`Budget` in one envtest-backed test:

1. **Suite setup** (`TestMain` or Ginkgo `BeforeSuite`): `setup-envtest use 1.34.x`, export `KUBEBUILDER_ASSETS`; start `envtest.Environment{CRDDirectoryPaths, WebhookInstallOptions}`; build the manager, register reconciler + webhook, start it; build a client.
2. **Create** a valid `GPUCostPolicy` and workload. Reconcile (or let the manager reconcile). Assert: finalizer present, `.status` conditions set (`Ready`), a `WorkloadCost` child created and owner-referenced.
3. **Idempotency:** read the object, record `resourceVersion` and managed fields. Trigger a second `Reconcile` on the unchanged state. Assert managed fields identical and no new child created — the reconcile did nothing observable.
4. **Level-trigger — drift:** directly `Update` a managed field to a wrong value (simulating drift the controller never witnessed). Reconcile. Assert it's corrected back to desired.
5. **Level-trigger — missing child:** delete the `WorkloadCost` child. Reconcile. Assert it's recreated (converges from the "missing" level, no error).
6. **Webhook:** `Create` a workload that would breach an active `Budget` set to `block`; assert the API call is **denied**. `Create` a compliant one; assert success.
7. **Finalizer/deletion:** `Delete` the parent object. Reconcile. Assert cleanup ran (final spend recorded), finalizer removed, object gone. Reconcile again on the now-absent object → clean no-op (deletion path is idempotent too).
8. **Teardown:** `Environment.Stop()`.

That single flow demonstrates every property the checkpoint asks for, all against a real API server, no live cluster.

**What this flow would NOT have caught.** Three real bugs, each requiring a kind/staging cluster instead: (1) a GPU device-plugin failing to actually inject `/dev/nvidia*` into a container — envtest has no kubelet, so no container ever starts; (2) a NetworkPolicy silently blocking the webhook's egress to an external pricing API — envtest runs no CNI, so network policy is never enforced; (3) an actual OOM kill under real container memory limits — envtest never runs a container, so a resource limit that's wrong in practice looks fine in every envtest run. Keep this list as the concrete answer to "what can't your test suite catch" — it's more memorable than the abstract "no kubelet" statement alone.

## Practice

Build the **full module deliverable** — the complete spec lives at [`../practice/gpu-cost-operator/README.md`](../practice/gpu-cost-operator/README.md).

Wire it end to end:

- CRD(s) + generated deepcopy + manifests for `GPUCostPolicy`, `WorkloadCost`, `Budget`.
- Controller with a converging, idempotent, level-triggered `Reconcile` for each.
- **Finalizer** with idempotent external cleanup (final spend recorded) on deletion.
- **Validating webhook** rejecting a workload that would breach an active `Budget` in `block` mode, plus at least one cross-field/semantic rule the schema can't express.
- **Least-privilege RBAC** from `+kubebuilder:rbac` markers — exactly the verbs used, zero wildcards.
- **Tests:** table-driven reconcile tests; **idempotency proven** (second reconcile is a no-op); **level-triggering proven** (converges from drifted/missing states); **a webhook test** (accept + reject) under envtest.

**Acceptance = the module checkpoint** ([`../checkpoint.md`](../checkpoint.md)): `make test` (envtest) is **green**, with idempotency, level-triggering, and webhook admission all demonstrated by passing tests, RBAC has zero wildcard verbs, and you can demo the operator and defend the test strategy — including naming what the test suite structurally cannot prove.

## Common pitfalls

1. **Treating a green `make test` (envtest) as "production ready."** It proves your controller's contract with the API, nothing about the rest of the system (scheduler, kubelet, real GPUs, real network policy). Say this explicitly in a demo before someone asks and catches you assuming otherwise.
2. **Asserting idempotency by checking "no error returned" instead of "no observable change."** A reconcile that always writes an identical-looking status can still churn `resourceVersion` and hot-loop the queue while every test assertion passes — the assertion has to compare the actual managed fields/resourceVersion, not just the absence of an error.
3. **Skipping webhook tests because "the fake client is faster."** The fake client cannot run webhooks at all; skipping envtest here means your validating webhook has *zero* test coverage of its actual wiring (cert paths, ports, failurePolicy) — the parts most likely to break in a real rollout.
4. **Not pinning the `setup-envtest` Kubernetes version.** CI and a laptop silently testing against different simulated API versions is a real, hard-to-diagnose flake source — pin it in the Makefile and check it into version control alongside `go.mod`.
5. **Adding the finalizer and doing "real work" in the same reconcile before re-fetching.** This is a race/staleness risk (Lesson 06's material resurfacing here at integration time): requeue after the metadata write and let the next reconcile do the work, rather than assuming the in-memory object is still current.

## Self-check

- What does envtest provide and what does it NOT provide — and given no kubelet, what can't you test? **Answer:** envtest provides a **real kube-apiserver and etcd** (via `setup-envtest`), so CRD install, admission webhooks, RBAC, status subresource, watches/informers, and optimistic-concurrency conflicts all behave for real. It provides **no kubelet, no scheduler, no controller-manager, no runtime** — so **Pods never run** (they stay Pending), built-in controllers don't act (a Deployment yields no ReplicaSets/Pods, Jobs create no Pods), and **owner-reference GC doesn't happen**. You therefore **cannot** test anything downstream of other components: that a container actually started, a probe passed, a workload landed on a GPU node, or that GC cleaned up children. Those need an e2e cluster (kind/real). envtest tests *your controller against the API*, not the rest of the system.
- How do you assert reconcile idempotency in a test? **Answer:** Reconcile once to reach steady state, then reconcile **again on the same observed state and assert nothing changes.** Concretely: snapshot the object's managed fields (and `resourceVersion`) after the first reconcile; call `Reconcile` a second time; assert the managed fields are byte-for-byte identical (and, under a real apiserver, `resourceVersion` is unchanged — a no-op does no write) and that no additional child objects were created and no error/spurious requeue was returned. A reconcile that always writes would bump resourceVersion on the second call and hot-loop the queue; the "second reconcile is a no-op" assertion is the guard.
- Fake client vs envtest — when do you use each? **Answer:** **Fake client** (in-memory, no server) for fast, focused **unit tests of reconcile logic** — given a seeded object store, does one `Reconcile` produce the right writes? It runs **no admission/webhooks** and only approximates API semantics (status subresource, validation, conflicts), so it can't test those. **envtest** (real apiserver + etcd) for anything exercising **real API semantics or admission**: validating-webhook accept/reject, CRD structural validation, status subresource behavior, resourceVersion conflicts, the finalizer/deletion flow, RBAC-shaped clients. A good suite uses both — fake client for the bulk of fast logic tests, envtest for the interactions that only a real API server can exercise.
- Name one bug class your envtest suite is structurally incapable of catching, and the cheapest real-cluster test that would catch it instead. **Answer:** A GPU device-plugin failing to inject `/dev/nvidia*` into a container is structurally uncatchable under envtest, because envtest never runs a kubelet or a container runtime — there is no container for the device to be injected into. The cheapest real-cluster catch is a `kind` cluster (or a small staging cluster) running the actual device-plugin DaemonSet plus one real Pod requesting a GPU resource, then asserting `nvidia-smi` (or an equivalent check) succeeds inside the running container — a minutes-long smoke test, not a full e2e suite.
- "Your controller stopped reconciling." Given everything assembled in this lesson, what's your diagnosis order? **Answer:** Check, in order: (1) **backoff/`Forget`** — is the object's key stuck climbing the exponential-backoff ladder toward the ~1000s cap from a persistent error, masking as "stopped" when it's actually retrying slowly (Lesson 03); (2) **stale cache** — did `WaitForCacheSync` actually return `true` at startup, and is the informer's watch connection still alive, or did it silently stop relisting (Lesson 04); (3) **missing watch or owner-ref** — does the controller actually `Owns()`/watch the type that changed, or is the change invisible to it entirely (Lesson 06); (4) **RBAC** — was a `list`/`watch`/`get` grant removed or never generated for a resource the reconciler needs, producing a Forbidden that envtest wouldn't have caught because envtest doesn't enforce RBAC (Lesson 07). Working through this order — rate limiting, then cache freshness, then watch wiring, then permissions — before touching business logic is the fast path to root cause, and it's exactly the order the module's lessons were taught in.

## Connections & what's next

This lesson closes the arc the whole module has been building: component internals (01) explained *what* watches and reconciles; API machinery (02) gave you the typed/dynamic client tools; the reconciliation model (03) gave you idempotency and level-triggering as design law; informers/caches/workqueues (04) explained *why* those properties are non-negotiable given how the queue behaves; CRD design (05) shaped the API surface; controller-runtime/finalizers/GC (06) wired lifecycle and cleanup; kubebuilder/RBAC (07) generated a defensible permission set; admission webhooks (08) added a synchronous gate at the API boundary; the scheduler/DRA/Kueue lesson (09) mapped where placement decisions live around all of it. Here, all of that becomes one operator, proven by tests you can defend.

This also **closes Module 02**. With `gpu-cost-operator` v0.1 green under envtest and the checkpoint's depth probes answerable cold, you're equipped for two modules this one directly **unlocks**: **[Module 04 — GPU on Kubernetes](../../04-gpu-on-kubernetes/README.md)**, where the operator starts doing real per-Pod GPU attribution against live GPU nodes instead of static price tables, and **[Module 06 — Scheduling, queueing and capacity](../../06-scheduling-capacity/README.md)**, where Lesson 09's Kueue literacy becomes a hands-on Kueue-showback deliverable with real quota, fairness, and preemption design. The operator you just built is the artifact both modules extend — keep it running and green.

## References & further reading

**Primary sources**
- Kubebuilder Book — testing with envtest — <https://book.kubebuilder.io/reference/envtest.html> — read for the canonical `Environment` setup pattern.
- controller-runtime `envtest` package godoc (`Environment`, `WebhookInstallOptions`) — <https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/envtest> — read for the exact API surface you'll call in `TestMain`/`BeforeSuite`.
- `setup-envtest` — provisioning apiserver/etcd binaries, `KUBEBUILDER_ASSETS` — <https://github.com/kubernetes-sigs/controller-runtime/tree/main/tools/setup-envtest> — read for version pinning and CI usage.
- fake client godoc (unit-testing reconcile logic) — <https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client/fake> — read for what it does and does not approximate.

**Real-world engineering blogs**
- SuperOrbital, "Testing Production Kubernetes Controllers" — <https://superorbital.io/blog/testing-production-controllers/> — envtest+table-driven testing as industry-standard practice.
- InfraCloud, "Testing Kubernetes Operators using EnvTest" — <https://www.infracloud.io/blogs/testing-kubernetes-operator-envtest/> — independent corroboration of the envtest fidelity boundary.
- Marc Nuri, "How to Test Kubernetes Applications in Go with EnvTest" — <https://blog.marcnuri.com/go-testing-kubernetes-applications-envtest> — a named practitioner's concrete walkthrough.

**Deeper dives**
- [`../checkpoint.md`](../checkpoint.md) — the module's completion gate; every depth probe this lesson's self-check maps to.
- [`../practice/gpu-cost-operator/README.md`](../practice/gpu-cost-operator/README.md) — the full deliverable spec, CRDs, scope guardrails, and acceptance criteria this lesson builds toward.
