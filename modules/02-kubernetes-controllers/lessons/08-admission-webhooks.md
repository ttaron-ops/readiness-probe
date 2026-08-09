---
lesson: "02.8"
title: "Validating admission webhooks and operational risk"
module: "02"
concept: "Validating admission webhooks and operational risk"
status: not-started
est_time: "10h"
artifacts: []
---

# 02.8 · Validating admission webhooks and operational risk

> **Concept.** Build a validating admission webhook that enforces GPU Budgets at the API boundary — and reason about the failure modes (self-lockout, cluster wedge) that make webhooks the most operationally dangerous extension point in Kubernetes.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Why this matters

A controller reacts *after* an object exists; a validating webhook decides whether it may exist *at all*. That's exactly what Budget enforcement needs — you want to reject an over-budget GPU workload before it schedules and starts billing, not reconcile it away after the fact. But an admission webhook sits synchronously in the write path of the entire API server: every create/update to matching resources blocks on your pod answering in time. Set `failurePolicy: Fail` and let your webhook pod crashloop, and you can wedge the cluster — including the very Deployment that would restart your webhook. This is a real, recurring big-tech outage class. The extension itself is a hundred lines of Go; the seniority is in scoping it so it can't fire on the wrong namespaces and choosing a failure policy you can defend when it's 3am and admission is rejecting everything.

## From operating to extending

You already know, as a CKA:

- That the API server runs a chain of built-in admission controllers (e.g. `ResourceQuota`, `PodSecurity`) before persisting an object to etcd.
- That authentication and authorization (RBAC) run *before* admission.
- How to `kubectl apply` and read the rejection message a webhook returns.
- That `ResourceQuota` already caps aggregate resource requests per namespace — useful context, because a Budget webhook is doing *cost-aware, cross-object* quota that the built-in can't express (it prices GPUs against Node labels and an active Budget CR).

Internal to the extending layer and new here:

- The **AdmissionReview** request/response contract and where dynamic webhooks plug into the chain.
- **Mutating vs validating** phases and their strict ordering.
- Scoping with `matchConditions` (CEL), `objectSelector`, `namespaceSelector`, and `rules`.
- **`failurePolicy`**, `sideEffects`, `timeoutSeconds`, and TLS/cert lifecycle for the webhook server.
- **ValidatingAdmissionPolicy** (in-tree CEL) as the webhook-less modern alternative.

## Core notes

**The admission chain and the AdmissionReview flow.** After authn + authz, the API server runs, in order: **mutating** admission (built-in, then `MutatingWebhookConfiguration`s), then **schema validation + defaulting**, then **validating** admission (built-in, then `ValidatingWebhookConfiguration`s), then persist to etcd. For each matching webhook the API server POSTs an `AdmissionReview` (`admission.k8s.io/v1`) whose `request` carries `uid`, `operation`, `object`, `oldObject`, `userInfo`, and `dryRun`. Your webhook replies with an `AdmissionReview` whose `response` echoes the `uid` and sets `allowed: true|false` (plus an optional `status.message` and `warnings`). Mutating webhooks additionally return a base64 JSONPatch. **The `uid` echo is mandatory** — a mismatched or missing uid is treated as a failure.

One subtlety among mutating webhooks: after the mutating phase completes, if any webhook changed the object, the API server may **re-run** mutating webhooks marked `reinvocationPolicy: IfNeeded` so a webhook can react to a later webhook's edit. Validating webhooks never reinvoke — they run once, at the end. This is why order-dependent logic belongs in validation (deterministic, sees the final object), not mutation (may run more than once, order among peers is unspecified).

**Validating vs mutating — and why validating can't mutate.** A mutating webhook returns a patch and runs *before* schema validation, so its changes are themselves validated afterward. A validating webhook runs *after* all mutation and defaulting, on the final object, and returns only allow/deny — the API server ignores any patch it sends. That ordering is the reason validation is trustworthy: it sees exactly what will be persisted, after every mutator (including other clusters' policy engines) has had its say. If validation could mutate, a later validating webhook could no longer trust that what it inspected is what gets stored. Corollary: **never** put enforcement logic that *depends on* a value a mutator sets in the mutator itself expecting validation to fix it up — mutators can run in any order among themselves.

**controller-runtime webhook (current API).** Kubebuilder scaffolds `internal/webhook/v1alpha1/` and registers a `CustomValidator`. The interface returns warnings + error; a non-nil error denies admission:

```go
// +kubebuilder:webhook:path=/validate-cost-example-com-v1alpha1-budget,mutating=false,failurePolicy=fail,sideEffects=None,groups=cost.example.com,resources=budgets,verbs=create;update,versions=v1alpha1,name=vbudget.kb.io,admissionReviewVersions=v1

type BudgetValidator struct{ client client.Client }

func (v *BudgetValidator) ValidateCreate(ctx context.Context, obj runtime.Object) (admission.Warnings, error) {
    // return apierrors.NewForbidden(...) or a plain error to reject
}
func (v *BudgetValidator) ValidateUpdate(ctx context.Context, oldObj, newObj runtime.Object) (admission.Warnings, error) { /* ... */ }
func (v *BudgetValidator) ValidateDelete(ctx context.Context, obj runtime.Object) (admission.Warnings, error) { return nil, nil }
```

Note the enforcement target: to gate *workloads* (Pods/Deployments) against Budgets, the webhook matches those resources — a validator that inspects arbitrary objects is registered with `builder.WebhookManagedBy(mgr).For(&corev1.Pod{}).WithValidator(v)` or via a raw `admission.Handler`. The `//+kubebuilder:webhook` marker is what `make manifests` turns into `config/webhook/manifests.yaml` (the `ValidatingWebhookConfiguration`), exactly like the RBAC markers in 02.7.

**Scoping — the safety controls.** A `ValidatingWebhookConfiguration` webhook entry is filtered by, cheapest-to-most-expensive:

- `rules`: `apiGroups`/`apiVersions`/`resources`/`operations` + `scope`. The coarse gate — match only what you enforce.
- `namespaceSelector`: label match on the object's namespace. **This is how you exclude control-plane namespaces.** Exclude `kube-system` so a broken webhook can't block the components that would recover it.
- `objectSelector`: label match on the object itself — opt workloads in/out.
- `matchConditions`: CEL expressions (GA 1.30) evaluated *after* the selectors, e.g. `request.userInfo.username != 'system:serviceaccount:kube-system:...'` to skip system actors, or `object.spec.requestsGPU`. CEL runs in-process in the API server — cheaper and safer than a network round-trip.

Order matters: `rules` → `namespaceSelector`/`objectSelector` → `matchConditions`. Anything filtered out never reaches your pod, so tightening these both reduces load and shrinks blast radius.

**`failurePolicy` — the wedge trap.** When the API server can't get a valid response (pod down, timeout, TLS error):

- `failurePolicy: Ignore` — the request is **admitted** as if the webhook approved it. Fails open: your Budget policy silently stops enforcing, but the cluster keeps working.
- `failurePolicy: Fail` — the request is **rejected**. Fails closed: policy holds, but if your webhook matches broadly and your pod is down, you block all matching writes. If it also matches the namespace your webhook Deployment lives in (or its dependencies), you can't redeploy the fix — **self-lockout / cluster wedge.**

Avoid the wedge: (1) `namespaceSelector` excluding `kube-system` **and** the webhook's own namespace; (2) `timeoutSeconds` small (e.g. 5–10) so a hung pod fails fast rather than stalling every write; (3) run the webhook with ≥2 replicas across nodes so a single failure doesn't take admission down; (4) keep `rules` narrow. A defensible default for a *policy* webhook is `Fail` (enforcement you can't silently lose) **with** those guards; `Ignore` is right when availability outranks the policy (e.g. best-effort labeling).

**`sideEffects`, `timeoutSeconds`, dry-run.** `sideEffects` declares whether the webhook mutates external state: `None` (pure function — required if you want `dryRun` requests to reach it), `NoneOnDryRun` (has side effects but honors `request.dryRun` by skipping them), or `Some`/`Unknown` (legacy; API server won't send dry-run). A validating Budget check is a pure read — declare `None`. `timeoutSeconds` (1–30, default 10) bounds how long the API server waits before applying `failurePolicy`.

**Certs.** The API server calls the webhook over HTTPS and verifies the serving cert against the `caBundle` in the webhook configuration. The serving cert's SAN must cover the Service DNS name the API server dials — `<service>.<namespace>.svc`. Two supported paths:

- **cert-manager**: a `Certificate` resource issues the serving cert into a Secret mounted by the webhook pod, and cert-manager's **CA injector** patches the matching `caBundle` into the `ValidatingWebhookConfiguration` via the `cert-manager.io/inject-ca-from` annotation. Kubebuilder wires this through `config/certmanager/` plus the commented `[WEBHOOK]`/`[CERTMANAGER]` kustomize patches in `config/default/kustomization.yaml` — you enable them to turn webhooks on. cert-manager also handles **rotation** automatically.
- **controller-runtime webhook server**: reads `tls.crt`/`tls.key` from `/tmp/k8s-webhook-server/serving-certs` (override with `webhook.Options{CertDir: ...}`); you supply and rotate the cert (and inject the caBundle) yourself.

Cert expiry is a classic silent outage: an expired or SAN-mismatched cert = every call fails TLS = `failurePolicy` fires cluster-wide, and under `Fail` that is an instant wedge. Alert on cert age well before expiry; prefer cert-manager so rotation isn't a human's calendar reminder.

**Debugging admission.** When `kubectl apply` hangs or returns `Internal error occurred: failed calling webhook`, the fault tree is short: is the webhook pod Ready? does its Service endpoint resolve? is the `caBundle` current and does the serving cert's SAN match the Service DNS (`<svc>.<ns>.svc`)? did a `matchConditions` CEL expression error (a runtime CEL error is treated per `failurePolicy`)? Inspect the live config with `kubectl get validatingwebhookconfiguration <name> -o yaml` and watch the API server's admission metrics (`apiserver_admission_webhook_rejection_count`, `..._admission_webhook_admission_duration_seconds`). A webhook that's slow but not failing still taxes every matching write — the duration histogram is your early-warning signal before `timeoutSeconds` starts tripping.

**ValidatingAdmissionPolicy (the modern alternative).** GA in Kubernetes 1.30, `admissionregistration.k8s.io/v1`. You write the rule as **CEL** in a `ValidatingAdmissionPolicy` and bind it to resources with a `ValidatingAdmissionPolicyBinding` — no webhook pod, no cert, no network hop, evaluated in-process by the API server. It removes the entire availability/wedge/cert failure surface. Use it when your check is expressible over the request object (and, via `paramKind`, a referenced config object) in CEL. Limits: CEL only (no arbitrary Go, no calling external cost APIs), no cross-object *live* lookups beyond a single bound param resource, no mutation. So a Budget check that needs to **list all Pods in a namespace and sum live GPU usage** still needs a webhook; a check like "reject if `object.spec.replicas * gpusPerPod > param.spec.limit`" using a bound Budget param is a perfect VAP.

## Worked example

Budget enforcement on Deployments, failing closed but wedge-proofed. First the validator logic:

```go
func (v *WorkloadValidator) ValidateCreate(ctx context.Context, obj runtime.Object) (admission.Warnings, error) {
    dep := obj.(*appsv1.Deployment)
    gpus := gpuRequestPerPod(dep) * int64(ptr.Deref(dep.Spec.Replicas, 1)) // sum requested GPUs
    if gpus == 0 {
        return nil, nil // not a GPU workload — allow
    }
    var budgets costv1alpha1.BudgetList
    if err := v.client.List(ctx, &budgets, client.InNamespace(dep.Namespace)); err != nil {
        return nil, fmt.Errorf("budget lookup failed: %w", err) // error => denies under Fail
    }
    for _, b := range budgets.Items {
        if !b.Status.Active { continue }
        projected := b.Status.CommittedGPUHours + estimateGPUHours(dep, gpus)
        if projected > b.Spec.LimitGPUHours {
            return admission.Warnings{"reduce replicas or request a budget increase"},
                apierrors.NewForbidden(schema.GroupResource{Group: "apps", Resource: "deployments"},
                    dep.Name, fmt.Errorf("projected %d GPU-h exceeds active Budget %q limit %d",
                        projected, b.Name, b.Spec.LimitGPUHours))
        }
    }
    return nil, nil
}
```

Then the configuration (generated from the `//+kubebuilder:webhook` marker + a namespaceSelector patch), showing the deliberate choices:

```yaml
webhooks:
- name: vworkload.kb.io
  admissionReviewVersions: ["v1"]
  sideEffects: None            # pure read of Budgets
  failurePolicy: Fail          # DELIBERATE: never silently stop enforcing spend
  timeoutSeconds: 5            # fail fast — don't stall the write path
  rules:
  - apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
    operations: ["CREATE", "UPDATE"]
  namespaceSelector:
    matchExpressions:
    - key: kubernetes.io/metadata.name
      operator: NotIn
      values: ["kube-system", "gpu-cost-system"]   # exclude control plane AND our own ns => no self-lockout
```

The `failurePolicy: Fail` decision is documented in-repo: *"Budget enforcement fails closed because silently admitting over-budget GPU workloads is a real cost incident; the wedge is bounded by excluding kube-system and the webhook's own namespace, a 5s timeout, and 2 replicas, so a webhook outage rejects only new tenant GPU workloads, never control-plane or webhook-recovery writes."* That paragraph is the interview answer.

## Practice

Implement the **Budget-enforcement validating webhook** in `gpu-cost-operator`.

1. `kubebuilder create webhook --group apps --version v1 --kind Deployment --programmatic-validation` (external type) or scaffold a raw `admission.Handler`; implement `ValidateCreate`/`ValidateUpdate` to reject Deployments whose projected GPU cost would exceed an **active** Budget in the same namespace.
2. In the `//+kubebuilder:webhook` marker set `failurePolicy`, `sideEffects=None`, `timeoutSeconds`, narrow `rules`; `make manifests`. Patch a `namespaceSelector` excluding `kube-system` and `gpu-cost-system`.
3. **Document the `failurePolicy` tradeoff** in `WEBHOOK-RISK.md` — which policy you chose, the wedge you're accepting/avoiding, and the guards.
4. Write a **webhook test** using `envtest` (or the controller-runtime `admission` decoder directly): assert a breaching Deployment is denied with the expected message and a within-budget one is allowed.

**Acceptance:**
- A budget-breaching Deployment is rejected with a clear `status.message` naming the Budget and the projected vs. limit.
- **Scaling the webhook Deployment to 0 demonstrates the chosen `failurePolicy`**: under `Fail`, applies to matched namespaces are rejected with a webhook-unavailable error; under `Ignore` they'd be admitted. Show the observed behavior matches the documented choice.
- Creating a workload in `kube-system` (or `gpu-cost-system`) is unaffected while the webhook is down — proving the exclusion.
- The webhook test passes in `make test`.

## Self-check

**(a) `failurePolicy: Fail` when your webhook pod is down — what happens, and how do you avoid self-lockout?**
**Answer:** The API server can't get a valid AdmissionReview response, so under `Fail` it **rejects every matching write** with a webhook-unavailable error. If the webhook's `rules`/selectors also match the namespace hosting the webhook (or its dependencies), you can't reschedule or redeploy the webhook itself — a self-inflicted cluster wedge. Avoid it by: `namespaceSelector` excluding `kube-system` **and** the webhook's own namespace; a small `timeoutSeconds` so hung calls fail fast; ≥2 replicas anti-affined across nodes so one failure doesn't down admission; narrow `rules` so the webhook can only affect the resources it must; and, where availability outranks policy, `Ignore` instead. Also monitor the serving cert — an expired cert triggers the same failure path cluster-wide.

**(b) Validating vs mutating ordering, and why can't a validating webhook change the object?**
**Answer:** The API server runs mutating admission first (built-ins, then mutating webhooks), then schema validation + defaulting, then validating admission, then persists. A validating webhook returns only `allowed` (any patch it sends is ignored) — by design, because it runs *last*, on the final post-mutation object, so it can trust that what it inspects is exactly what gets stored. If validation could mutate, that guarantee would break for any validating webhook downstream, and validation could re-introduce values that were never schema-validated. Mutation belongs in the earlier phase precisely so its output is re-validated before persistence.

**(c) When would ValidatingAdmissionPolicy (CEL) replace a webhook, and what are its limits?**
**Answer:** Use a `ValidatingAdmissionPolicy` (GA 1.30) when the rule is expressible in CEL over the request object plus at most one bound `paramKind` resource — e.g. reject a Deployment when `replicas * gpusPerPod > param.spec.limit`. It runs in-process in the API server, so it eliminates the webhook pod, the TLS cert, the network hop, and thus the entire availability/wedge/cert-expiry failure surface, and it can't self-lock the cluster. Limits: CEL only (no arbitrary Go, no calls to an external cost API), no live cross-object queries beyond the single bound param (so a check that must `list` all Pods and sum current GPU usage still needs a webhook), and it cannot mutate. Rule of thumb: static/relational checks → VAP; checks needing external data or fleet-wide aggregation → webhook.

## Resources

1. **Kubebuilder Book — Webhook overview** — scaffolding, `CustomValidator`, marker → manifest, cert-manager wiring: https://book.kubebuilder.io/reference/webhook-overview.html
2. **kubernetes.io — Dynamic Admission Control** — AdmissionReview contract, `failurePolicy`, `matchConditions`, selectors, `sideEffects`: https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/ (and the admission-controllers reference at `.../admission-controllers/`).
3. **kubernetes.io — Validating Admission Policy** — CEL policy + binding, `paramKind`, when to prefer it over a webhook: https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/
