# ⚙️ Checkpoint — 02 · Kubernetes internals and controllers

The **completion gate**. Answer the probes from memory, and prove the rest with the
[`gpu-cost-operator` v0.1](practice/gpu-cost-operator/) deliverable running green
under envtest.

## Pass criteria (prove with the operator)

- [ ] **1 · Tests green.** `make test` (envtest) passes; reconcile tests prove
      **idempotency** (two reconciles → identical status) and **level-triggering**
      (converges regardless of event order).
- [ ] **2 · Reconcile works.** Applying a `GPUCostPolicy` + workload produces a
      `WorkloadCost` with correct computed `.status` and a `Ready` condition,
      owner-referenced to its parent.
- [ ] **3 · Lifecycle.** Deleting a parent cascades (GC); the finalizer records final
      spend before the object is removed.
- [ ] **4 · Webhook.** A workload that would breach an active `Budget` is **rejected by
      the validating webhook**; disabling the webhook pod demonstrates the
      `failurePolicy` behavior you chose — and can explain.
- [ ] **5 · Validation.** CRDs reject invalid specs via CEL (e.g. negative budget) at
      `kubectl apply`.
- [ ] **6 · RBAC.** The generated `ClusterRole`/`Role` has **no wildcard verbs**; you
      can justify every grant.

## Depth probes (answer cold, unaided)

**Component internals**
- [ ] Walk `kubectl apply -f pod.yaml` to a running container, naming each component and what it does internally (apiserver pipeline → etcd → scheduler → kubelet syncLoop → CRI).
- [ ] Why does leader election exist in kube-controller-manager, and why will your operator need it in HA?
- [ ] What does the apiserver watch cache do, and what does `resourceVersion` guarantee across a watch reconnect?

**API machinery & reconcile**
- [ ] GVK vs GVR — who resolves one to the other?
- [ ] Why must reconcile never branch on "which event fired"? Give a concrete bug.
- [ ] The three requeue paths and how backoff differs; where does `Queue.Forget()` belong?

**Informers**
- [ ] Explain the reflector → Delta FIFO → informer → workqueue → worker path.
- [ ] When can a controller-runtime client read return stale data, and how do you force a live read?
- [ ] How do the workqueue's dedup + rate limiter keep a controller stable under an event storm?

**CRDs, controller-runtime, webhooks, RBAC**
- [ ] Why is `/status` a subresource, and what changes about updates + `observedGeneration`?
- [ ] What breaks if you call `Update()` instead of `Status().Update()`?
- [ ] What happens on delete if a finalizer never returns, and how do you avoid a stuck object?
- [ ] `failurePolicy: Fail` when your webhook pod is down — what happens, and how do you avoid self-lockout?

**GPU scheduling (differentiator)**
- [ ] DRA-by-attribute vs device-plugin integer counts — what does DRA enable for cost/MIG?
- [ ] What does Kueue's gang / all-or-nothing scheduling solve that the default scheduler can't?

**Debug prompt**
- [ ] "Your controller stopped reconciling." Give your diagnosis order (backoff/`Forget`, stale cache, missing watch/owner-ref, RBAC).

## Answers / notes

_Record answers as you close each lesson; link the operator code proving items 1–6._
