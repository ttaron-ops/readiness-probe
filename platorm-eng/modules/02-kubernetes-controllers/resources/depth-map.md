# Depth map — Module 02 · Kubernetes controllers

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **The densest match in the repo.** The `kubernetes/` track is 46 chapters at 2,300–3,400 lines
> each, explicitly written at "staff level". Every lesson in this module has a chapter that goes
> substantially deeper, and the practice ladder in `k8s-learn/` is the best hands-on companion the
> course has for the capstone build.

| Lesson | Go deeper in | Why |
|---|---|---|
| 01 Component internals | [`kubernetes/03-kubernetes-architecture-overview`](https://github.com/harut8/system-design/blob/main/kubernetes/03-kubernetes-architecture-overview.md) | the whole control plane in one map, before the per-component dives |
| 01 Component internals | [`kubernetes/04-etcd-internals`](https://github.com/harut8/system-design/blob/main/kubernetes/04-etcd-internals.md) | 2,700 lines: MVCC, revisions, compaction, watch, and **why etcd is fsync-bound** |
| 01 Component internals | [`kubernetes/05-kube-apiserver-internals`](https://github.com/harut8/system-design/blob/main/kubernetes/05-kube-apiserver-internals.md) | the request path, the watch cache, priority & fairness — "the only door to etcd" |
| 02 API machinery | [`kubernetes/24-api-aggregation-and-extension-apiservers`](https://github.com/harut8/system-design/blob/main/kubernetes/24-api-aggregation-and-extension-apiservers.md) | discovery, aggregation layer, and when an extension apiserver beats a CRD |
| 03 Reconciliation model | [`kubernetes/08-controller-pattern-and-client-go`](https://github.com/harut8/system-design/blob/main/kubernetes/08-controller-pattern-and-client-go.md) | 3,000 lines on level-triggered reconciliation, edge-vs-level, and idempotence |
| 04 Informers, caches, workqueues | [`kubernetes/08-controller-pattern-and-client-go`](https://github.com/harut8/system-design/blob/main/kubernetes/08-controller-pattern-and-client-go.md) | the reflector → DeltaFIFO → indexer → workqueue chain, resync semantics, rate limiters |
| 05 CRD design | [`kubernetes/23-crds-operators-and-controller-runtime`](https://github.com/harut8/system-design/blob/main/kubernetes/23-crds-operators-and-controller-runtime.md) | 3,100 lines: schema design, versioning/conversion webhooks, subresources, status conventions |
| 06 controller-runtime deep | [`kubernetes/23-crds-operators-and-controller-runtime`](https://github.com/harut8/system-design/blob/main/kubernetes/23-crds-operators-and-controller-runtime.md) | manager, caches, owner references, predicates, leader election |
| 07 Kubebuilder & RBAC | [`kubernetes/07-authentication-authorization`](https://github.com/harut8/system-design/blob/main/kubernetes/07-authentication-authorization.md) | authn chain, RBAC evaluation, ServiceAccount tokens and projected volumes |
| 08 Admission webhooks | [`kubernetes/06-admission-control-deep-dive`](https://github.com/harut8/system-design/blob/main/kubernetes/06-admission-control-deep-dive.md) | mutating vs validating order, CEL policies, and **why webhooks are the #1 source of cluster outages** — failure policy is the lesson |
| 09 Scheduler & GPU scheduling | [`kubernetes/09-kube-scheduler-internals`](https://github.com/harut8/system-design/blob/main/kubernetes/09-kube-scheduler-internals.md) | the scheduling cycle, the framework's extension points, preemption |
| 10 Capstone build | [`kubernetes/36-garbage-collection-and-object-lifecycle`](https://github.com/harut8/system-design/blob/main/kubernetes/36-garbage-collection-and-object-lifecycle.md) | owner refs, finalizers, orphan/foreground/background deletion — where operators leak resources |

## Practice worth stealing

The `k8s-learn/` task ladders are the strongest practice material in the source repo:

- [`controller-tasks`](https://github.com/harut8/system-design/blob/main/k8s-learn/controller-tasks.md) — write one by hand with client-go, before frameworks
- [`operator-tasks`](https://github.com/harut8/system-design/blob/main/k8s-learn/operator-tasks.md) — the real thing with controller-runtime
- [`api-machinery-tasks`](https://github.com/harut8/system-design/blob/main/k8s-learn/api-machinery-tasks.md) — the layer every tool sits on

They feed directly into the [`gpu-cost-operator`](../practice/gpu-cost-operator/) deliverable, and
they run fine against the
[fake GPU fleet](../../04-gpu-on-kubernetes/practice/fake-gpu-fleet/README.md).

## The stretch chapter

[`kubernetes/38-building-a-kubernetes-from-scratch`](https://github.com/harut8/system-design/blob/main/kubernetes/38-building-a-kubernetes-from-scratch.md)
— a `minik8s` capstone: build an apiserver, scheduler and kubelet from nothing. Out of scope for
this course's timeline, but if a control-plane role is the target, nothing else on your résumé
would say "I understand this" as loudly.
