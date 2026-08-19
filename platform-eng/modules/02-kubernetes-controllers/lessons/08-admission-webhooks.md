---
lesson: "02.8"
title: "Validating admission webhooks and operational risk"
module: "02"
concept: "Validating admission webhooks and operational risk"
status: not-started
est_time: "20h"
prev: "07-kubebuilder-and-rbac.md"
next: "09-scheduler-and-gpu-scheduling.md"
artifacts: []
sources: 12
---

# 02.8 · Validating admission webhooks and operational risk

> **Concept.** Build a validating admission webhook that enforces GPU Budgets at the API boundary — and reason about the failure modes (self-lockout, cluster wedge) that make webhooks the most operationally dangerous extension point in Kubernetes.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Lesson 07 gave you the marker → controller-gen → manifest discipline through the lens of RBAC: a reviewable, regenerated-not-hand-edited artifact that is a tight projection of what your controller actually does. This lesson applies the same pipeline (`+kubebuilder:webhook` markers → `config/webhook/manifests.yaml`) to a component with a fundamentally different risk profile.

RBAC failures are about *excess access*. Webhook failures are about *availability of the entire API server's write path*. Everything you have built so far — CRDs (05), the reconcile loop (06), least-privilege RBAC (07) — reacts to objects that already exist. A validating webhook is the first thing in this module that can say "no" *before* an object is ever persisted, which is exactly what Budget enforcement needs: reject an over-budget GPU workload before it schedules and starts billing, rather than reconcile it away after four H100s have been running for twenty minutes.

## Why this matters

A controller reacts after an object exists; a validating webhook decides whether it may exist at all. The price of that power is that your pod is now synchronously in the path of every matching write in the cluster. Every `kubectl apply`, every controller-driven update, every rollout blocks on your HTTP handler answering within its timeout.

Set `failurePolicy: Fail`, let your webhook pod crashloop, and you can wedge the cluster — including the very Deployment that would restart the webhook. Worse, and less well known: a webhook that is merely *slow* can degrade a cluster even under `failurePolicy: Ignore`, because latency you add to admission is latency added to the API server's finite concurrency budget, and because critical control-plane writes (leader-election renewals among them) have their own deadlines that do not care about your failure policy. Both of those are documented upstream bugs with real clusters attached, quoted below.

The extension itself is a hundred lines of Go. The seniority is in scoping it so it cannot fire on the wrong namespaces, budgeting its latency, and choosing a failure policy you can defend at 3am when admission is rejecting everything.

## What's new here (calibration)

You already know, as a CKA:

- That the API server runs a chain of built-in admission controllers (`ResourceQuota`, `PodSecurity`, `LimitRanger`, `ServiceAccount`, …) before persisting an object to etcd.
- That authentication and authorization run *before* admission.
- How to `kubectl apply` and read the rejection message a webhook returns.
- That `ResourceQuota` already caps aggregate resource requests per namespace — useful context, because a Budget webhook is doing *cost-aware, cross-object* quota that the built-in cannot express: it prices GPUs against Node labels and an active `Budget` CR, in dollars, over a rolling window.

Internal to the extending layer and new here:

- The **`AdmissionReview` request/response contract** on the wire, including the base64 JSONPatch a mutating webhook returns, and where dynamic webhooks sit relative to schema validation and etcd.
- The **complete `ValidatingWebhookConfiguration` and `MutatingWebhookConfiguration` field set** — `failurePolicy`, `sideEffects`, `timeoutSeconds`, `matchPolicy`, `objectSelector`, `namespaceSelector`, `matchConditions`, `reinvocationPolicy`, `admissionReviewVersions` — and what each one costs you if you get it wrong.
- **How a webhook actually takes a cluster down**, in three distinct mechanisms, with the recovery procedure for each.
- **The latency arithmetic**: what milliseconds on your handler cost in API-server concurrency at fleet write rates.
- **ValidatingAdmissionPolicy** (GA 1.30) and **MutatingAdmissionPolicy** (GA 1.36) as the in-process, CEL-based alternatives that remove the entire availability surface — and precisely where they cannot replace a webhook.

Versions: **Kubernetes 1.36**, `admissionregistration.k8s.io/v1`, `admission.k8s.io/v1`, **controller-runtime v0.24.1**, **kubebuilder v4.12**.

## Core concepts

### 1. The problem: why a controller is not enough

Suppose a tenant applies a Deployment requesting 32 H100s and their team's `Budget` has $400 of headroom. A controller-only design notices this on the next reconcile — which is fast, but not instantaneous — and by then the Pods have been created, scheduled, pulled, and started. On a GPU cloud you are billed from the moment the instance-hours are consumed. Reconciling the Deployment down to zero replicas afterwards is a *refund request*, not a control.

The built-in `ResourceQuota` admission controller *is* synchronous and *does* reject over-quota Pods at admission time. It cannot help here for three structural reasons:

1. It counts resources, not money. `requests.nvidia.com/gpu: 32` is expressible; "32 H100-hours at $5.12/hr against a $400 monthly limit" is not.
2. Its scope is a single namespace's aggregate usage. A `Budget` may span a label-selected set of namespaces, or a cost centre.
3. It has no notion of a *window*. Quota is instantaneous occupancy; a budget is spend integrated over a month.

So you need something that runs synchronously in the write path, can read arbitrary cluster state and your own CRs, and can compute in your own units. That is a validating admission webhook. **Accepting a synchronous dependency in the API server's write path is the price, and everything difficult about this lesson follows from that one decision.**

### 2. The admission chain, drawn exactly

Here is the full write path for `POST /apis/apps/v1/namespaces/team-a/deployments`, with the webhook phases in their real positions relative to schema validation and etcd. The plugins named are those enabled by default in Kubernetes 1.36.

```
  client: kubectl apply -f deploy.yaml
    │
    ▼
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │ kube-apiserver                                                               │
 │                                                                              │
 │  ① AUTHENTICATION      token/cert → user + groups                            │
 │  ② AUTHORIZATION       RBAC (lesson 07) → allow / no-opinion                 │
 │  ③ DESERIALIZE         wire bytes → internal object; unknown fields pruned    │
 │                        against the OpenAPI schema                            │
 │                                                                              │
 │ ═══════════════ MUTATING ADMISSION PHASE (sequential) ═══════════════        │
 │  ④ built-in mutating plugins, in a FIXED order set by the apiserver:         │
 │       NamespaceLifecycle → LimitRanger → ServiceAccount →                    │
 │       DefaultStorageClass → DefaultTolerationSeconds → RuntimeClass →        │
 │       DefaultIngressClass → Priority → …                                     │
 │  ⑤ MutatingAdmissionPolicy   (CEL, in-process, GA in 1.36)                   │
 │  ⑥ MutatingAdmissionWebhook  → calls YOUR mutating webhooks, ONE AT A TIME,  │
 │       ┌─────────────────────────────────────────────────────────┐            │
 │       │ for each matching webhook, in lexical order by name:     │            │
 │       │   POST AdmissionReview → wait ≤ timeoutSeconds           │            │
 │       │   apply returned base64 JSONPatch to the object          │            │
 │       └─────────────────────────────────────────────────────────┘            │
 │       then: REINVOCATION pass — any webhook with                             │
 │       reinvocationPolicy: IfNeeded is called AGAIN if a later                │
 │       webhook changed the object. (Exactly one extra pass.)                  │
 │                                                                              │
 │ ═══════════════ SCHEMA VALIDATION + DEFAULTING ═══════════════               │
 │  ⑦ structural schema validation, OpenAPI/CEL (x-kubernetes-validations),     │
 │     defaulting, immutability checks.                                          │
 │     ◀── EVERYTHING A MUTATOR WROTE IS VALIDATED HERE. That is why mutation    │
 │         must come first: a mutator cannot smuggle an invalid value in.        │
 │                                                                              │
 │ ═══════════════ VALIDATING ADMISSION PHASE (parallel) ═══════════════        │
 │  ⑧ built-in validating plugins: PodSecurity, ResourceQuota,                  │
 │       CertificateApproval, CertificateSigning, …                             │
 │  ⑨ ValidatingAdmissionPolicy  (CEL, in-process, GA in 1.30)                  │
 │  ⑩ ValidatingAdmissionWebhook → calls YOUR validating webhooks               │
 │       ┌─────────────────────────────────────────────────────────┐            │
 │       │ ALL matching webhooks dispatched CONCURRENTLY            │            │
 │       │ (one goroutine each, joined on a WaitGroup)              │            │
 │       │ any allowed:false → request rejected, others' results    │            │
 │       │ discarded. No reinvocation, ever.                        │            │
 │       └─────────────────────────────────────────────────────────┘            │
 │                                                                              │
 │  ⑪ PERSIST to etcd  ── the object that ⑩ inspected is byte-for-byte what      │
 │                        gets stored. That guarantee is the entire reason for   │
 │                        the ⑥→⑦→⑩ ordering.                                    │
 └──────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
                       201 Created  /  403 Forbidden  /  500 Internal error


  THE failurePolicy FORK — taken whenever the API server cannot get a *valid
  response*: connection refused, TLS failure, timeout, malformed body, uid mismatch.

                    call to your webhook
                            │
              ┌─────────────┴──────────────┐
              ▼                            ▼
      valid response received      NO valid response
              │                            │
       ┌──────┴──────┐          ┌──────────┴──────────┐
       ▼             ▼          ▼                     ▼
  allowed:true  allowed:false  failurePolicy:      failurePolicy:
       │             │          Ignore                Fail
       │             │            │                     │
   continue      REJECT       ADMIT the request      REJECT the request
   the chain     (403)        (policy silently       ("failed calling
                              stops enforcing;        webhook … : ...";
                              counted in              counted in
                              apiserver_admission_    apiserver_admission_
                              webhook_fail_open_      webhook_rejection_
                              count)                  count{error_type=...})

  Note the asymmetry that trips people up: a webhook that answers `allowed: false`
  is a SUCCESSFUL call. failurePolicy governs only the "could not get an answer"
  branch. A deny is a deny under both policies.
```

Three properties of that diagram are load-bearing and worth stating on their own:

**Mutating webhooks run sequentially; validating webhooks run in parallel.** This is not a documentation detail, it is why mutating webhooks are the more dangerous kind for latency: five mutating webhooks at 20 ms each cost 100 ms serially, while five validating webhooks at 20 ms each cost ~20 ms. The API server's validating dispatcher launches one goroutine per relevant webhook and joins them on a `sync.WaitGroup`; the mutating dispatcher walks the list in order because each webhook must see the previous one's patch applied.

**A validating webhook's response cannot mutate.** The API server ignores any `patch` field on a validating response. This is deliberate: because validation runs last, on the post-mutation, post-schema-validation object, every validating webhook can trust that what it inspected is exactly what will be stored. If validation could mutate, that guarantee would evaporate for every *other* validating webhook in the chain, and the mutation would bypass schema validation entirely.

**Reinvocation applies only to mutation, and only once.** `reinvocationPolicy: IfNeeded` gets your mutating webhook one extra pass if a later webhook modified the object. It is not a fixpoint loop — two webhooks that keep undoing each other will not converge, they will simply produce whichever result the second pass leaves behind. This is why the official guidance is that mutating webhooks must be *idempotent*: "inject a sidecar if one is not already present" is safe; "append a container" is not.

### 3. The `AdmissionReview` wire contract

controller-runtime builds and parses this envelope for you, but you cannot debug a webhook from `kubectl -v=8` output or API-server logs without knowing its shape.

**Request.** The API server POSTs this to your `clientConfig` path, `Content-Type: application/json`:

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "kind":     {"group": "apps", "version": "v1", "kind": "Deployment"},
    "resource": {"group": "apps", "version": "v1", "resource": "deployments"},
    "requestKind":     {"group": "apps", "version": "v1", "kind": "Deployment"},
    "requestResource": {"group": "apps", "version": "v1", "resource": "deployments"},
    "name": "trainer",
    "namespace": "team-a",
    "operation": "CREATE",
    "userInfo": {
      "username": "system:serviceaccount:team-a:ci-deployer",
      "uid": "014fbff9a07c",
      "groups": ["system:serviceaccounts", "system:serviceaccounts:team-a", "system:authenticated"]
    },
    "object":    { "apiVersion": "apps/v1", "kind": "Deployment", "spec": {"replicas": 8, "...": "..."} },
    "oldObject": null,
    "options":   {"apiVersion": "meta.k8s.io/v1", "kind": "CreateOptions"},
    "dryRun": false
  }
}
```

Field-by-field, the ones that matter:

| Field | Why you care |
|---|---|
| `uid` | **You must echo it verbatim in the response.** A missing or mismatched `uid` is treated as a failed call, so `failurePolicy` fires. This is the single most common cause of "my webhook returns 200 but everything is rejected." |
| `operation` | `CREATE`, `UPDATE`, `DELETE`, or `CONNECT`. On `DELETE`, `object` is null and `oldObject` holds the object being deleted. On `CREATE`, `oldObject` is null. |
| `kind` / `resource` vs `requestKind` / `requestResource` | These differ when `matchPolicy: Equivalent` converted the request. `requestResource` is what the client actually sent; `resource` is the version your webhook registered for. If you care about the original, read `requestResource`. |
| `userInfo` | The authenticated identity. This is how you exempt system actors — but see §6 on why `matchConditions` is the better place to do that. |
| `subResource` | Present when the request targets a subresource (`scale`, `status`). **A `rules` entry for `deployments` does not match `deployments/scale`.** A budget webhook that ignores this can be bypassed by `kubectl scale`. |
| `dryRun` | `true` for `kubectl apply --dry-run=server`. You only receive dry-run requests at all if `sideEffects` is `None` or `NoneOnDryRun`. |
| `options` | The `CreateOptions`/`UpdateOptions`/`DeleteOptions` the client sent, including `fieldManager` — occasionally useful to tell a human's `kubectl` from a controller's server-side apply. |

**Response, deny with a message and warnings** — this is the shape your Budget webhook returns:

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "allowed": false,
    "status": {
      "code": 403,
      "message": "projected 512 GPU-h ($2621.44) exceeds active Budget \"team-a-q3\" remaining $400.00"
    },
    "warnings": ["reduce replicas to 1 or request a budget increase via GPUCostPolicy"]
  }
}
```

`status.message` is what the user sees on their terminal, wrapped by the API server as `admission webhook "vworkload.kb.io" denied the request: <message>`. Make it name the object, the number, and the remedy — it is the entire UX of your policy. `warnings` are surfaced by `kubectl` even on an *allowed* request; each warning over 256 characters may be truncated, and the API server caps all warnings from all sources at 4096 characters total.

**Response, mutating with a JSONPatch.** A mutating webhook returns the patch base64-encoded in the `patch` field, with `patchType: "JSONPatch"` (the only supported value in `admission.k8s.io/v1`):

```json
// the patch, before encoding:
[{"op": "add", "path": "/spec/replicas", "value": 3}]
```

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "allowed": true,
    "patchType": "JSONPatch",
    "patch": "W3sib3AiOiAiYWRkIiwgInBhdGgiOiAiL3NwZWMvcmVwbGljYXMiLCAidmFsdWUiOiAzfV0="
  }
}
```

Two mechanical traps in JSONPatch that cost people an afternoon each. First, JSON Pointer escaping: `/` inside a key becomes `~1` and `~` becomes `~0`, so patching the annotation `cost.example.com/budget` means `"path": "/metadata/annotations/cost.example.com~1budget"`. Second, `add` on a path whose *parent* does not exist fails the whole patch — if `metadata.annotations` is absent you must first `add` `/metadata/annotations` with value `{}`. controller-runtime's `admission.Patched()` computes the patch by diffing your mutated object against the original, which sidesteps both, and is why you should almost never hand-write JSONPatch in a Go webhook.

### 4. A complete `ValidatingWebhookConfiguration`, every field annotated

This is the configuration `gpu-cost-operator` ships. Nothing here is optional decoration; each field is a decision.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: gpu-cost-operator-validating-webhook-configuration
  annotations:
    # cert-manager's cainjector watches this annotation and writes the CA into
    # every webhooks[].clientConfig.caBundle below. Format: <namespace>/<certificate-name>.
    # (Current kubebuilder scaffolds a kustomize `replacements` block instead, which
    #  does the same job at build time rather than at runtime. Either works; pick one.)
    cert-manager.io/inject-ca-from: gpu-cost-system/gpu-cost-operator-serving-cert
webhooks:
- name: vworkload.kb.io          # REQUIRED. Must be a DNS-subdomain-ish name with at
                                 # least three dot-separated segments. It is the key in
                                 # every metric and every error message, so make it
                                 # greppable. Sorting is lexical, which is how the
                                 # apiserver orders MUTATING webhooks — irrelevant here.

  admissionReviewVersions: ["v1"] # REQUIRED, no default. Ordered preference list; the
                                 # apiserver sends the first version it also supports.
                                 # If it supports none of yours, the call is a failure
                                 # and failurePolicy fires. "v1beta1" was removed in
                                 # 1.22 — list only ["v1"] on any modern cluster.

  sideEffects: None              # REQUIRED, no default. Legal values in v1: None,
                                 # NoneOnDryRun. (Some/Unknown existed in v1beta1 and
                                 # are rejected now.) `None` asserts the webhook makes
                                 # no out-of-band state changes, which is what allows
                                 # the apiserver to send it dry-run requests at all.
                                 # A Budget check is a pure read → None.

  failurePolicy: Fail            # DEFAULT IS Fail. See §7 — this is the decision that
                                 # can wedge a cluster, and it must be paired with the
                                 # selectors below or it will.

  timeoutSeconds: 5              # Default 10. Legal range 1–30. The apiserver wraps the
                                 # call in context.WithTimeout(ctx, timeoutSeconds) —
                                 # but the parent is the *request's* context, so the
                                 # effective budget is min(timeoutSeconds, time left on
                                 # the request). Long chains exhaust the request budget
                                 # before any single webhook hits its own timeout.

  matchPolicy: Equivalent        # DEFAULT is Equivalent. Keep it. `Exact` matches only
                                 # the literal group/version/resource in `rules`, so a
                                 # client posting through any other served version of
                                 # the same resource bypasses the webhook entirely — a
                                 # silent hole in a security-relevant policy.

  rules:                         # The coarse gate, evaluated first and most cheaply.
  - apiGroups:   ["apps"]
    apiVersions: ["v1"]
    resources:   ["deployments", "statefulsets"]
                                 # Patterns: "*" = all resources but NOT subresources;
                                 # "*/*" = resources and subresources; "pods/*" = all
                                 # pod subresources; "*/status" = all status subresources.
                                 # Listing "deployments" does NOT cover
                                 # "deployments/scale" — enumerate it if you must gate it.
    operations:  ["CREATE", "UPDATE"]
                                 # CREATE | UPDATE | DELETE | CONNECT | "*".
                                 # No DELETE here: deleting a workload only frees budget.
    scope:       "Namespaced"    # Cluster | Namespaced | "*". Default "*".

  namespaceSelector:             # THE SELF-LOCKOUT GUARD. Matches labels on the
    matchExpressions:            # *namespace* of the object. Every namespace carries
    - key: kubernetes.io/metadata.name    # kubernetes.io/metadata.name automatically
      operator: NotIn                     # (set by the NamespaceLifecycle/label
      values:                             #  controller since 1.21), so you can select
      - kube-system                       #  by name without labelling anything.
      - kube-node-lease
      - gpu-cost-system          # ← the webhook's OWN namespace. Omitting this line is
                                 #   the single most common cause of a wedged cluster.
                                 # NOTE: has no effect on cluster-scoped resources other
                                 # than Namespace objects themselves.

  objectSelector:                # Matches labels on the OBJECT. Use for opt-in only:
    matchExpressions:            # a user can dodge an opt-out selector by simply not
    - key: cost.example.com/exempt        # setting the label, so never use objectSelector
      operator: DoesNotExist              # as a security boundary. Here it is an
                                          # explicitly-granted exemption, which is fine.

  matchConditions:               # CEL, GA in 1.30. Evaluated IN-PROCESS, after the
  - name: exclude-system-actors  # selectors, before any network call. Max 64 per webhook.
    expression: '!(request.userInfo.username.startsWith("system:serviceaccount:kube-system:"))'
  - name: only-gpu-workloads     # Available vars: object, oldObject, request, authorizer.
    expression: >-
      has(object.spec.template.spec.containers) &&
      object.spec.template.spec.containers.exists(c,
        has(c.resources.limits) && "nvidia.com/gpu" in c.resources.limits)
                                 # ALL conditions must be true for the webhook to be
                                 # called. Any one false → webhook skipped silently.
                                 # A CEL *runtime error* is treated per failurePolicy.

  clientConfig:
    service:
      namespace: gpu-cost-system
      name: gpu-cost-operator-webhook-service
      path: /validate--v1-deployment   # must begin with /validate- or /mutate-
      port: 443                        # the SERVICE port; it targets containerPort 9443
    caBundle: <PEM, base64>            # written by cainjector or kustomize replacements
    # Alternatively `url: https://…` for a webhook outside the cluster. Mutually
    # exclusive with `service`. Avoid it: it bypasses Service-based load balancing and
    # ties admission to external DNS and routing.
```

### 5. A complete `MutatingWebhookConfiguration`

Same field set plus one. `gpu-cost-operator` v0.1 does **not** ship a mutating webhook (the deliverable's scope guardrails say validating only), but you must be able to read and reason about one — the cluster you deploy into will have several.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: cost-labeller.example.com
webhooks:
- name: mcostlabel.kb.io
  admissionReviewVersions: ["v1"]
  sideEffects: NoneOnDryRun     # This webhook writes a row to an external ledger when it
                                # stamps a cost-centre label. It suppresses that write
                                # when request.dryRun is true — which is exactly what
                                # NoneOnDryRun asserts, and what lets `kubectl apply
                                # --dry-run=server` still reach it.
  failurePolicy: Ignore         # Best-effort labelling. Losing a label is a reporting
                                # gap; blocking every Pod create is an outage. The
                                # availability/correctness trade lands the other way
                                # here than it does for the Budget validator.
  timeoutSeconds: 3             # Mutating webhooks run SEQUENTIALLY, so this number is
                                # additive with every other mutator in the cluster.
                                # Budget it accordingly: 3 s is generous for a pure
                                # in-process label stamp.
  matchPolicy: Equivalent
  reinvocationPolicy: IfNeeded  # MUTATING ONLY. Never (default) = called at most once.
                                # IfNeeded = called a second time if a later mutating
                                # webhook changed the object. Exactly one extra pass,
                                # not a fixpoint loop. Requires the webhook to be
                                # idempotent: "set the label if absent", never "append".
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
    operations: ["CREATE"]
    scope: "Namespaced"
  namespaceSelector:
    matchExpressions:
    - key: kubernetes.io/metadata.name
      operator: NotIn
      values: ["kube-system", "kube-node-lease", "gpu-cost-system"]
  clientConfig:
    service: {namespace: gpu-cost-system, name: gpu-cost-operator-webhook-service, path: /mutate--v1-pod, port: 443}
    caBundle: <PEM, base64>
```

The rule to carry away: **anything a mutator sets must be independently validated, and anything a validator depends on must not be something a mutator might set.** Mutators run in an order you do not control among themselves, and may run twice. Enforcement logic that assumes "the labeller already ran" is a race waiting for a cluster with one more webhook in it.

### 6. The matching funnel, and why it is a cost control

Requests are filtered in a fixed order, and each stage is cheaper than the next. Everything filtered out never touches your pod.

```
  ALL write requests to the apiserver
  ─────────────────────────────────── say 4,200 req/s cluster-wide (mixed verbs)
            │
            │  ① rules:  apiGroups × apiVersions × resources × operations × scope
            │     in-process, string compare. Also applies matchPolicy: Equivalent
            │     (converting an equivalent version into the one you registered for).
            ▼
  apps/v1 deployments+statefulsets, CREATE|UPDATE ──────── ~35 req/s
            │
            │  ② namespaceSelector — label lookup on the Namespace object from the
            │     apiserver's own cache. Cheap. Excludes kube-system, kube-node-lease,
            │     gpu-cost-system.
            ▼
  tenant namespaces only ────────────────────────────────── ~31 req/s
            │
            │  ③ objectSelector — label match on the object already in memory. Free.
            ▼
  not explicitly exempted ───────────────────────────────── ~30 req/s
            │
            │  ④ matchConditions — CEL, compiled once, evaluated in-process against
            │     object/oldObject/request/authorizer. Costs microseconds. Metered by
            │     apiserver_admission_match_condition_evaluation_seconds.
            ▼
  actually requests a GPU, not a kube-system actor ───────── ~2 req/s
            │
            │  ⑤ ***NETWORK CALL*** — TLS handshake (amortised over a kept-alive
            │     connection), JSON serialise, POST, your handler, JSON parse.
            ▼
  your webhook pod: ~2 req/s, p50 6 ms, p99 40 ms
```

The funnel above turns a 35 req/s webhook into a 2 req/s webhook — a 17× reduction in blast radius, load, and exposure, for four lines of YAML and one CEL expression. **`matchConditions` is the highest-leverage field in the whole configuration** and the one most often left out, because it arrived late (beta 1.28, GA 1.30) and most tutorials predate it.

One subtlety: a `matchConditions` expression that *errors at runtime* (say, `object.spec.template.spec.containers` on an object where `spec.template` is absent) is not treated as "false" — it is treated as a webhook failure, so `failurePolicy` fires. Guard every field access with `has()` as the example above does, and check `apiserver_admission_match_condition_evaluation_errors_total` after any change.

### 7. How a webhook takes down a cluster — three mechanisms

This is the part of the lesson that is actually about seniority. There are three distinct ways to break a cluster with a webhook, they have different symptoms, and only the first one is what people mean when they say "failurePolicy".

#### Mechanism A — hard wedge: `Fail` + a broad rule + a dead backend

```
 T+0    You apply a webhook: rules {"*"/"*"/"*", CREATE+UPDATE}, failurePolicy: Fail,
        no namespaceSelector. It works. Tests pass. You go home.

 T+3d   The node hosting the single webhook replica is drained for a kernel patch.
        │
        ▼
 T+3d   The scheduler tries to create a replacement Pod for the webhook Deployment.
 +0s    That Pod CREATE is itself a matching write.
        │
        ▼        apiserver → webhook Service → no ready endpoints → connection refused
        │        → no valid response → failurePolicy: Fail → REJECT
        ▼
        "Internal error occurred: failed calling webhook "v.example.com": failed to
         call webhook: Post "https://webhook-svc.ns.svc:443/validate?timeout=10s":
         dial tcp 10.96.4.17:443: connect: connection refused"
        │
        ▼
 T+3d   The webhook cannot be recreated, because recreating it requires admission,
 +5s    which requires the webhook. ReplicaSet controller retries with backoff, forever.
        │
        │  AND, because the rule matches everything:
        ├── no Deployment in any namespace can be created or updated
        ├── no ConfigMap, Secret, Service can be created or updated
        ├── kube-controller-manager's leader-election Lease renewal (an UPDATE) fails
        │     → controller-manager loses leadership → ReplicaSet/Deployment/GC
        │       controllers stop acting entirely
        └── a control-plane upgrade that must write to kube-system fails half-done
        │
        ▼
 RECOVERY — the only reliable exit, and it requires an identity holding delete on
 admissionregistration.k8s.io (which is why cluster-admin break-glass exists):

   $ kubectl get validatingwebhookconfiguration
   $ kubectl delete validatingwebhookconfiguration v.example.com     # ← unwedges
   … fix the configuration …
   $ kubectl apply -f fixed-webhook.yaml

 Deleting the *configuration* is the fix, not deleting the pod. As long as the
 configuration object exists, the apiserver will keep trying to call it.
```

**Preventing it** is four things, and you need all four:

1. `namespaceSelector` excluding `kube-system`, `kube-node-lease`, **and the webhook's own namespace**. Also exclude `kube-public` and any namespace hosting your CNI, CSI, or cert-manager, because they are on the recovery path.
2. Narrow `rules`. `"*"/"*"/"*"` in a webhook is the same review failure as `["*"]` in a ClusterRole and for the same reason: it covers resources that do not exist yet.
3. ≥2 replicas with `topologySpreadConstraints` or pod anti-affinity across nodes, plus a `PodDisruptionBudget` with `minAvailable: 1` so a drain cannot take the last one.
4. Small `timeoutSeconds` (3–5 s for a validator that does a cached read), so failures are fast rather than stalling every write for 10 s each.

The official rollout order for a new webhook exists for exactly this reason: install the server first, register the configuration with `failurePolicy: Ignore` and a `namespaceSelector` scoped to a single test namespace, watch the metrics, then widen the selector, then — if the policy warrants it — flip to `Fail`.

#### Mechanism B — soft wedge: `Ignore` is not a safety net for *latency*

This is the one that surprises people, and it is documented upstream in [kubernetes/kubernetes#90217](https://github.com/kubernetes/kubernetes/issues/90217). A webhook matching all resources on CREATE and UPDATE, configured with `failurePolicy: Ignore`, degraded clusters anyway. The mechanism: `kube-controller-manager` renews its leader-election lease with a `PUT` carrying its own client-side timeout (`?timeout=10s` on the request URL). That write matched the webhook's rules, so it waited on the webhook's default 10 s timeout. `Ignore` meant the request was eventually *admitted* — but not before the controller-manager's own deadline expired and it concluded it had lost the lease. Leadership churned; the built-in controllers stopped and restarted repeatedly.

The general form: **`failurePolicy: Ignore` protects the correctness of the admission decision; it does not protect the latency of the request.** Any client with a deadline shorter than your webhook's timeout is broken by a slow webhook regardless of failure policy. The clients with deadlines you should care about are the leader-election renewals of every control-plane component, kubelet's node-status updates, and the CSR-signing path.

A closely related upstream report, [kubernetes/kubernetes#128162](https://github.com/kubernetes/kubernetes/issues/128162) (reported against v1.31.1, closed as not-planned), shows the compounding version: several webhooks each with `failurePolicy: Ignore` and a 10 s `timeoutSeconds`, all slow at once, exhaust the *enclosing request's* deadline. The API server's dispatcher checks `ctx.Done()` before each call and wraps each webhook in `context.WithTimeout(ctx, timeoutSeconds)` — with `ctx` being the request's context, so the effective per-webhook budget is `min(timeoutSeconds, time remaining on the request)`. Once that runs out you get `Timeout: request did not complete within requested timeout - context deadline exceeded`, which names no webhook at all because none of them technically failed. That error message with no webhook name in it is the fingerprint of this failure.

#### Mechanism C — the dead-connection stall after a rollout

[kubernetes/kubernetes#80313](https://github.com/kubernetes/kubernetes/issues/80313), still open. Reported on EKS at Kubernetes 1.12.6: when the node hosting a webhook pod was **shut down abruptly rather than drained**, the API server kept using a cached connection to the now-dead pod IP. TCP connections in that state are half-open — the peer is gone but no `RST` or `FIN` ever arrives — so every admission call sat until its timeout expired. The reporter measured roughly **15 minutes** of blocked pod creation before the stale connection was finally reaped, long after the webhook pod had been rescheduled and was serving happily elsewhere. The workaround was to `kubectl drain` the node instead of powering it off, which lets the endpoint be removed cleanly.

This is why "my webhook only times out right after I roll it" is a real mechanism rather than flakiness, and why your rollout story is part of your webhook design:

- Give the webhook container a real `readinessProbe` on its serving port so it leaves and enters `Endpoints` at the right moments.
- Set `terminationGracePeriodSeconds` high enough for in-flight admission calls to drain, and use a `preStop` sleep so the pod leaves `Endpoints` before it stops serving.
- `maxUnavailable: 0` / `maxSurge: 1` on the webhook Deployment, so a healthy backend always exists during the roll.
- Keep `timeoutSeconds` small: it bounds how long a stale connection can stall each write.

### 8. The latency arithmetic: what your webhook costs the cluster

Every admission call consumes an API-server *seat* for its whole duration, because the request handler is blocked waiting. With API Priority and Fairness enabled (default since 1.29 GA), the server's total concurrency limit is `--max-requests-inflight` + `--max-mutating-requests-inflight` = **400 + 200 = 600 seats** by default, shared out across priority levels.

Work the numbers for `gpu-cost-operator`'s validator. Assumptions stated so you can re-run them with your own:

```
  matched write rate, steady state        R_s   =  2 req/s
  matched write rate, fleet-wide rollout  R_b   = 90 req/s   (every team redeploys)
  webhook handler latency (cached List
    of Budgets + arithmetic), p50/p99     L     =  6 ms / 40 ms
  network + TLS + serialise overhead      O     =  2 ms      (keep-alive, same cluster)

  seats held = arrival rate × time held           (Little's law)

  steady state, p50:   2  × (6+2)  ms =    16 ms/s → 0.016 seats  (0.003 % of 600)
  steady state, p99:   2  × (40+2) ms =    84 ms/s → 0.084 seats
  rollout burst, p50: 90  × (6+2)  ms =   720 ms/s → 0.72  seats  (0.12 % of 600)
  rollout burst, p99: 90  × (40+2) ms = 3,780 ms/s → 3.78  seats  (0.63 % of 600)
```

Under 1% of the server's concurrency in the worst normal case. That is a healthy webhook, and the reason is entirely the matching funnel in §6 — 90 req/s is already the *filtered* rate.

Now break it. Suppose the webhook's Budget lookup stops hitting the informer cache and starts doing a live `List` against the API server, and that list takes 900 ms:

```
  rollout burst, degraded: 90 × (900+2) ms = 81,180 ms/s → 81.2 seats (13.5 % of 600)
```

13.5% of the entire cluster's request concurrency, consumed by requests that are doing nothing but waiting on your pod — and each of those calls is itself issuing a `List` back into the same API server, so the load is self-amplifying. Push it one step further and let the handler hang until timeout:

```
  rollout burst, hung:     90 × 5,000 ms = 450,000 ms/s → 450 seats (75 % of 600)
```

At that point the API server is effectively down for everyone, and **`failurePolicy: Ignore` does not save you** — the seats are consumed by the waiting, not by the outcome. This is Mechanism B expressed in arithmetic.

Two design rules fall out of that table:

1. **Never do an uncached read in an admission handler.** Use the controller-runtime manager's cached client (informer-backed) for anything you read. If you must read live, cap it with a context deadline well under `timeoutSeconds` and fail deterministically when it expires.
2. **Budget the handler in milliseconds, not seconds.** The official guidance is that webhooks should evaluate "typically in milliseconds." A p99 above ~50 ms on a high-frequency resource is a design smell; above ~200 ms it is a capacity plan.

### 9. `sideEffects`, dry-run, and idempotence

`sideEffects` is a *declaration*, not an enforcement mechanism — the API server takes you at your word and uses it to decide whether to send dry-run requests:

| Value | Meaning | Receives `dryRun: true` requests? |
|---|---|---|
| `None` | The webhook mutates no state outside the `AdmissionReview` response. | Yes |
| `NoneOnDryRun` | It does have side effects, but it inspects `request.dryRun` and skips them. | Yes |
| (`Some` / `Unknown`) | v1beta1 only; **rejected by `admissionregistration.k8s.io/v1`.** | n/a |

Declaring `None` while actually writing to a ledger means `kubectl apply --dry-run=server` silently performs a real write — an especially nasty bug because dry-run is the thing people reach for precisely when they want to be safe. A validating Budget check is a pure read of `Budget` CRs, so `None` is honest.

The related requirement on mutating webhooks is idempotence, which follows directly from reinvocation (§2): your webhook may be called twice on the same request, on an object it already modified. "Set `.spec.securityContext.runAsNonRoot = true`" is idempotent. "Inject a sidecar named `proxy` if no container named `proxy` exists" is idempotent. "Append a container" and "inject a sidecar named `proxy-<timestamp>`" are not — the second pass adds a duplicate.

### 10. TLS, certificates, and the silent outage

The API server dials your webhook over HTTPS and validates the presented certificate against the `caBundle` in the configuration. Two requirements, both of which fail closed:

- The serving certificate's **SAN must cover `<service>.<namespace>.svc`** — the DNS name the API server constructs from `clientConfig.service`. Not the pod IP, not the ClusterIP, not `<service>.<namespace>.svc.cluster.local` alone. Kubebuilder's scaffolded cert-manager `Certificate` lists both `<svc>.<ns>.svc` and `<svc>.<ns>.svc.cluster.local` in `dnsNames`.
- The `caBundle` must be the CA that signed it, base64-encoded PEM.

controller-runtime's webhook server defaults, which you need to match on both ends:

| Setting | Default |
|---|---|
| Port | `9443` (so the Service maps 443 → 9443) |
| Cert directory | `<os.TempDir()>/k8s-webhook-server/serving-certs` |
| Cert file name | `tls.crt` |
| Key file name | `tls.key` |
| Client CA (mTLS from apiserver) | unset — no client cert verification |

Two supported wiring paths:

**cert-manager** (what kubebuilder scaffolds). A `Certificate` in `config/certmanager/` issues into a Secret that the manager Deployment mounts at the cert directory. The CA gets into the `caBundle` by one of two routes: cert-manager's **cainjector**, which watches the annotation `cert-manager.io/inject-ca-from: <namespace>/<certificate-name>` on `ValidatingWebhookConfiguration`, `MutatingWebhookConfiguration`, `CustomResourceDefinition`, and `APIService` objects and writes the `caBundle` at runtime; or kustomize `replacements` in `config/default/kustomization.yaml`, which current kubebuilder uses to splice the value in at build time. cert-manager also handles **rotation**, which is the reason to prefer it.

**Self-managed.** You supply `tls.crt`/`tls.key` (via `webhook.Options{CertDir: ...}`) and patch the `caBundle` yourself. Every one of those steps is a thing that expires on a calendar you now own.

**Cert expiry is a textbook silent cluster outage.** The certificate expires at 03:00. Every admission call now fails the TLS handshake, which is a "no valid response" — so under `Fail`, every matching write in the cluster is rejected, instantly and simultaneously, with an error message about x509 rather than about your policy. Nothing changed in your code and nothing was deployed. Alert on certificate age (cert-manager exposes `certmanager_certificate_expiration_timestamp_seconds`) with at least a third of the lifetime as headroom.

### 11. Debugging admission, with the metrics that matter

When `kubectl apply` hangs or returns `failed calling webhook`, work this tree in order — it is sorted by how often each branch is the answer:

```
 SYMPTOM: "Internal error occurred: failed calling webhook X"
   │
   ├─ 1. Is the backend serving?
   │      kubectl get endpoints -n gpu-cost-system gpu-cost-operator-webhook-service
   │      → empty Endpoints ⇒ no ready pod. Check readinessProbe, then pod logs.
   │
   ├─ 2. Is it TLS?  error text contains "x509"
   │      → caBundle stale, or SAN missing <svc>.<ns>.svc, or the cert expired.
   │      kubectl get validatingwebhookconfiguration X -o jsonpath='{.webhooks[0].clientConfig.caBundle}' | base64 -d | openssl x509 -noout -dates -text
   │
   ├─ 3. Is it a timeout?  error text contains "context deadline exceeded"
   │      → if the message names YOUR webhook: your handler is slow (§8).
   │      → if the message names NO webhook: the *request* deadline expired across the
   │        whole chain (k/k#128162). Sum every matching webhook's timeoutSeconds.
   │      → if it only happens right after a rollout: dead connections (k/k#80313).
   │
   ├─ 4. Is it a CEL error in matchConditions?
   │      apiserver_admission_match_condition_evaluation_errors_total{name="vworkload.kb.io"}
   │      → a runtime error is a failure, not a "skip". Guard field access with has().
   │
   └─ 5. Is it actually a *denial* rather than a failure?
          The message reads: admission webhook "X" denied the request: <your message>
          → your logic ran and said no. Not an outage. Read status.message.
```

The API server exports these, and they are the difference between guessing and knowing (names taken from `k8s.io/apiserver/pkg/admission/metrics`):

| Metric | Type | Labels | What it tells you |
|---|---|---|---|
| `apiserver_admission_webhook_admission_duration_seconds` | histogram | `name`, `type`, `operation`, `rejected` | Your webhook's observed latency **from the API server's side**, including network. Buckets go to 25 s, so you can see timeouts. The single most useful signal. |
| `apiserver_admission_webhook_rejection_count` | counter | `name`, `type`, `operation`, `error_type`, `rejection_code` | Rejections split by cause. `error_type` distinguishes a policy denial from a call failure — watch this one, not just the total. |
| `apiserver_admission_webhook_fail_open_count` | counter | `name`, `type` | **How often `failurePolicy: Ignore` silently let something through.** If you chose `Ignore`, this counter is your policy's blind-spot meter. It must be alerted on, or "we have a policy" is not true. |
| `apiserver_admission_webhook_request_total` | counter | `name`, `type`, `operation`, `code`, `rejected` | Call volume and HTTP status. Divide into the duration histogram for throughput. |
| `apiserver_admission_step_admission_duration_seconds` | histogram | `type`, `operation`, `rejected` | Total time in the whole admission phase — the number that matters for cluster health, versus your webhook's share of it. |
| `apiserver_admission_match_condition_evaluation_seconds` | histogram | `name`, `kind`, `type`, `operation` | CEL evaluation cost. Buckets top out at 250 ms; if you are near that, the expression is too complex. |
| `apiserver_admission_match_condition_exclusions_total` | counter | `name`, `kind`, `type`, `operation` | How many calls your `matchConditions` *saved*. The number that justifies §6. |

`type` is `admit` or `validate`; `rejected` is `true`/`false`. A useful pair of alerts: p99 of `..._webhook_admission_duration_seconds{name="vworkload.kb.io"}` above 100 ms for 10 minutes, and any nonzero rate on `..._fail_open_count`.

### 12. The webhook-less alternatives: VAP and MAP

**`ValidatingAdmissionPolicy`** (GA in Kubernetes **1.30**, `admissionregistration.k8s.io/v1`) lets you express the rule as CEL and have the API server evaluate it in-process. No pod, no Service, no certificate, no network hop — and therefore none of §7's three failure mechanisms. It is two objects: the policy (the logic) and a binding (where it applies and what it does on failure).

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: gpu-replica-cap.cost.example.com
spec:
  failurePolicy: Fail
  paramKind:                       # at most ONE bound param resource
    apiVersion: cost.example.com/v1alpha1
    kind: Budget
  matchConstraints:
    resourceRules:
    - apiGroups: ["apps"]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["deployments"]
  variables:                       # named, lazily-evaluated sub-expressions
  - name: gpusPerPod
    expression: >-
      object.spec.template.spec.containers
        .map(c, int(c.resources.limits["nvidia.com/gpu"] ?? "0"))
        .sum()
  validations:
  - expression: "object.spec.replicas * variables.gpusPerPod <= params.spec.maxConcurrentGPUs"
    messageExpression: >-
      "requested " + string(object.spec.replicas * variables.gpusPerPod) +
      " GPUs exceeds Budget cap of " + string(params.spec.maxConcurrentGPUs)
    reason: Forbidden
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: gpu-replica-cap-binding
spec:
  policyName: gpu-replica-cap.cost.example.com
  validationActions: ["Deny"]      # Deny | Warn | Audit. Deny and Warn are mutually
                                   # exclusive (they would double-report the failure).
                                   # "Audit" alone is how you dry-run a policy in prod.
  paramRef:
    selector:
      matchLabels: {cost.example.com/scope: namespace-default}
    parameterNotFoundAction: Deny
  matchResources:
    namespaceSelector:
      matchExpressions:
      - {key: kubernetes.io/metadata.name, operator: NotIn, values: ["kube-system"]}
```

**`MutatingAdmissionPolicy`** is the mutating counterpart, **GA in Kubernetes 1.36** and enabled by default. It applies mutations expressed as CEL, in one of two forms: `ApplyConfiguration` (declare the desired shape, merged server-side-apply style) or `JSONPatch` (a CEL expression evaluating to a list of patch operations). It removes the sequential-dispatch latency problem of mutating webhooks along with everything else.

**Where policies cannot replace your webhook.** CEL in a policy runs inside the API server with a **cost budget** to prevent unbounded computation, and it can access only: `object`, `oldObject`, `request`, `params` (one bound resource), `namespaceObject`, and `authorizer`. It cannot make an outbound call and it cannot query a collection. So:

| Check | VAP/MAP? | Why |
|---|---|---|
| `replicas × gpusPerPod ≤ Budget.spec.maxConcurrentGPUs` | ✅ | Pure function of the object plus one bound param. |
| "reject if `spec.priceClass` is not in the enum" | ✅ | Better still: put it in the CRD's own `x-kubernetes-validations` (lesson 05). |
| "reject if projected spend + **current spend across all Pods in this namespace** > limit" | ❌ | Requires listing a collection. No CEL access to other objects. |
| "reject if the live cloud-billing API says the account is delinquent" | ❌ | No outbound calls, ever. |
| "reject unless the node pool this would land on has a price label under $X" | ❌ | Requires reading Node objects. |

`gpu-cost-operator`'s Budget enforcement needs the *third* row — it sums current GPU-hours across a namespace — so it needs a webhook. **But the correct architecture is both:** put the cheap, static, per-object checks in a `ValidatingAdmissionPolicy` where they cost nothing and cannot wedge anything, and reserve the webhook for the one check that genuinely needs fleet state. That split shrinks the webhook's `matchConditions` surface, which shrinks its call rate, which shrinks every number in §8.

### 13. Implementing it: kubebuilder markers and controller-runtime

The marker is the same pipeline as lesson 07's RBAC markers, with different arguments. Its full argument set, with controller-tools' defaults:

| Marker arg | Maps to | Default |
|---|---|---|
| `path=` | `clientConfig.service.path` | none — must start `/validate-` or `/mutate-` |
| `mutating=true\|false` | which configuration kind is emitted | required |
| `failurePolicy=fail\|ignore` | `failurePolicy` | `Fail` |
| `matchPolicy=Exact\|Equivalent` | `matchPolicy` | `Equivalent` |
| `sideEffects=None\|NoneOnDryRun` | `sideEffects` | required in v1 |
| `timeoutSeconds=` | `timeoutSeconds` (1–30) | `10` |
| `groups=`, `resources=`, `versions=`, `verbs=` | `rules[0].*` (`verbs` → `operations`) | none |
| `name=` | `webhooks[].name` | none — ≥3 dot-separated segments |
| `admissionReviewVersions=` | `admissionReviewVersions` | none, required |
| `reinvocationPolicy=Never\|IfNeeded` | mutating only | `Never` |
| `webhookVersions=` | the config API version | `v1` |
| `serviceName=` / `serviceNamespace=` / `servicePort=` | `clientConfig.service.*` | `webhook-service` / `system` / `443` |
| `url=` | `clientConfig.url` (external backend) | unset; mutually exclusive with `path` |

Note what the marker **cannot** express: `namespaceSelector`, `objectSelector`, and `matchConditions`. Those are added by a kustomize patch on top of the generated `config/webhook/manifests.yaml`. This is a real gap and the reason so many shipped webhooks lack the self-exclusion — the generator does not prompt you for it.

The Go side, on controller-runtime v0.24.1 (note the **generics**; the old non-generic `admission.CustomValidator` still exists as `Validator[runtime.Object]` but is deprecated):

```go
package v1

import (
    "context"
    "fmt"

    appsv1 "k8s.io/api/apps/v1"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/runtime/schema"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/webhook/admission"

    costv1alpha1 "github.com/you/gpu-cost-operator/api/v1alpha1"
)

// +kubebuilder:webhook:path=/validate-apps-v1-deployment,mutating=false,failurePolicy=fail,sideEffects=None,timeoutSeconds=5,groups=apps,resources=deployments,verbs=create;update,versions=v1,name=vworkload.kb.io,admissionReviewVersions=v1

type WorkloadValidator struct {
    Client client.Client // the MANAGER's cached client — see §8
}

// Compile-time proof we satisfy the generic interface for *appsv1.Deployment.
var _ admission.Validator[*appsv1.Deployment] = &WorkloadValidator{}

func SetupWorkloadWebhookWithManager(mgr ctrl.Manager) error {
    return ctrl.NewWebhookManagedBy(mgr, &appsv1.Deployment{}).
        WithValidator(&WorkloadValidator{Client: mgr.GetClient()}).
        Complete()
}

func (v *WorkloadValidator) ValidateCreate(ctx context.Context, dep *appsv1.Deployment) (admission.Warnings, error) {
    return v.check(ctx, dep)
}

func (v *WorkloadValidator) ValidateUpdate(ctx context.Context, _, dep *appsv1.Deployment) (admission.Warnings, error) {
    return v.check(ctx, dep)
}

func (v *WorkloadValidator) ValidateDelete(context.Context, *appsv1.Deployment) (admission.Warnings, error) {
    return nil, nil // deleting a workload only frees budget
}
```

Two contract details worth memorising. **A non-nil `error` return denies the request**; return `apierrors.NewForbidden(...)` or `apierrors.NewInvalid(...)` so the client sees a `403`/`422` with a structured `Status`, rather than a bare error that becomes a generic `500`. And **`admission.Warnings` are surfaced even when you allow** — they are the right channel for "this will breach your budget in 3 days," which is a nudge rather than a rejection.

## Perspectives

**API-design perspective.** The mutate → schema-validate → validate ordering is a trust boundary, not a pipeline convenience. It is what lets any consumer — including another validating webhook, and including the API server's own storage layer — reason "what validation saw is what gets persisted," with no exceptions. That is why a validating webhook's patch is *ignored* rather than *rejected*: the API server does not need to argue with you, it simply does not read the field.

**Operator / on-call perspective.** `failurePolicy` is a decision made once, at design time, by someone who will not be the person paged for it. The `WEBHOOK-RISK.md` the Practice section asks for is what converts a design decision into a runbook: which policy, what the wedge boundary is, which namespaces are excluded and why, and the exact `kubectl delete validatingwebhookconfiguration` incantation that unwedges the cluster. One paragraph, findable at 3am.

**Availability-versus-correctness perspective.** This is the clearest such trade in the module. `Fail` buys policy correctness with availability; `Ignore` buys the reverse — and, per §7's Mechanism B, `Ignore` buys *less* availability than people assume, because latency is not a failure. `ValidatingAdmissionPolicy` is the rare case where you escape the trade entirely, but only for CEL-expressible rules. Naming which of the three you chose, and why, is the whole interview answer.

**Adversarial perspective.** Read your own configuration as an attacker: what is the cheapest way to wedge this cluster? A `Fail` webhook with broad `rules` and no self-exclusion needs no exploit at all — one drained node does it. A webhook with `matchPolicy: Exact` can be bypassed by posting through a different served version. A webhook whose `objectSelector` is an *opt-out* label can be bypassed by anyone who can set labels, which is everyone who can create the object. Each of those is a review question, not a code question.

**Cost perspective (the one you own).** A webhook is the only place in this module where the operator's decision has a direct, immediate dollar consequence: rejecting a 32×H100 Deployment at admission saves the whole run's cost, while reconciling it away afterwards saves only the remainder. That asymmetry is the argument for `failurePolicy: Fail` on a *budget* webhook specifically, and it is worth stating in those terms rather than as a general preference for failing closed.

## Real-world use cases

- **kubernetes/kubernetes#80313 — "Admission webhooks affected by dead tcp connections"** ([issue](https://github.com/kubernetes/kubernetes/issues/80313), still open). **What happened:** on an EKS cluster at Kubernetes 1.12.6, a worker node hosting a mutating webhook pod was shut down abruptly rather than drained. The API server continued to use a cached connection to the departed pod's IP; because the TCP connection was half-open, no reset ever arrived and every admission call blocked until timeout. Pod creation was blocked for roughly **15 minutes** after the webhook had already been rescheduled and was serving normally elsewhere. **The fix in practice** was to `kubectl drain` rather than power off, so the endpoint is withdrawn cleanly. **What it shows:** "times out only right after a rollout" is a mechanism with a name, and your webhook's disruption story (readiness probes, `preStop`, `maxUnavailable: 0`, PDB) is part of its design, not an afterthought.
- **kubernetes/kubernetes#90217 — "Webhook with failurePolicy: Ignore that applies to controller manager leases breaks clusters"** ([issue](https://github.com/kubernetes/kubernetes/issues/90217)). **What happened:** a webhook matching all resources on `CREATE` and `UPDATE`, explicitly configured `failurePolicy: Ignore` so it "could not break anything," degraded clusters when its backend became slow. `kube-controller-manager`'s leader-election renewal is a `PUT` to `kube-system` carrying its own `?timeout=10s`; that write matched the webhook and waited on the webhook's default 10 s timeout. The request was eventually admitted — after the controller-manager had already concluded it lost leadership. **What it shows:** `Ignore` protects the admission *decision*, not the request's *latency*. Excluding `kube-system` is not paranoia; it is the specific fix for this specific class.
- **kubernetes/kubernetes#128162 — "Pod admission can fail due to webhooks + context deadline exceeded, even when all webhooks are set to failurePolicy = Ignore"** ([issue](https://github.com/kubernetes/kubernetes/issues/128162), reported on v1.31.1, closed as not-planned). **What happened:** multiple slow webhooks, each `Ignore` with its own timeout, cumulatively exhausted the enclosing request's deadline; admission failed with `Timeout: request did not complete within requested timeout - context deadline exceeded` and **no webhook named in the error**, because none had individually failed. **What it shows:** webhook timeouts compose additively against a shared request budget, and the error you get carries no attribution. The operational takeaway is to inventory every `ValidatingWebhookConfiguration` and `MutatingWebhookConfiguration` in a cluster and sum the `timeoutSeconds` that can apply to one resource — a number nobody tracks until it bites.

## Worked example

Budget enforcement on Deployments in `gpu-cost-operator`: failing closed, wedge-proofed, and measured.

**Step 1 — the handler.** Note the cached client, the early return for non-GPU workloads, and the structured error.

```go
func (v *WorkloadValidator) check(ctx context.Context, dep *appsv1.Deployment) (admission.Warnings, error) {
    gpusPerPod := gpuLimitPerPod(dep)          // sums nvidia.com/gpu across containers
    if gpusPerPod == 0 {
        return nil, nil                        // not a GPU workload — allow, no lookup
    }
    replicas := int64(1)
    if dep.Spec.Replicas != nil {
        replicas = int64(*dep.Spec.Replicas)
    }
    concurrentGPUs := gpusPerPod * replicas

    // Cached read. The manager's informer already holds every Budget; this is a map
    // lookup, not an API call. See §8 for why that distinction is load-bearing.
    var budgets costv1alpha1.BudgetList
    if err := v.Client.List(ctx, &budgets, client.InNamespace(dep.Namespace)); err != nil {
        // Under failurePolicy: Fail this denies. That is deliberate: we would rather
        // block one Deployment than admit unbounded spend on a degraded lookup.
        return nil, fmt.Errorf("budget lookup failed for namespace %s: %w", dep.Namespace, err)
    }

    for i := range budgets.Items {
        b := &budgets.Items[i]
        if !b.Status.Active || b.Spec.Enforcement != costv1alpha1.EnforcementBlock {
            continue                            // report/warn modes do not reject
        }
        projected := b.Status.SpendUSD + estimateSpendUSD(dep, concurrentGPUs, b.Spec.Window)
        if projected > b.Spec.LimitUSD {
            return admission.Warnings{
                    "reduce replicas or raise the limit on Budget " + b.Name,
                },
                apierrors.NewForbidden(
                    schema.GroupResource{Group: "apps", Resource: "deployments"},
                    dep.Name,
                    fmt.Errorf("projected spend $%.2f over window %s exceeds active Budget %q limit $%.2f (currently $%.2f, this workload adds %d concurrent GPUs)",
                        projected, b.Spec.Window, b.Name, b.Spec.LimitUSD, b.Status.SpendUSD, concurrentGPUs),
                )
        }
        if projected > 0.8*b.Spec.LimitUSD {
            return admission.Warnings{
                fmt.Sprintf("this workload puts Budget %q at %.0f%% of its $%.2f limit",
                    b.Name, 100*projected/b.Spec.LimitUSD, b.Spec.LimitUSD),
            }, nil                              // allowed, but the user is told
        }
    }
    return nil, nil
}
```

**Step 2 — generate the configuration, then patch in what the marker cannot express.**

```
$ make manifests
bin/controller-gen rbac:roleName=manager-role crd webhook … paths="./..."

$ cat config/webhook/manifests.yaml     # generated — rules/failurePolicy/timeout only
```

The `namespaceSelector` and `matchConditions` are added by a kustomize patch, because no marker emits them:

```yaml
# config/webhook/selector_patch.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: validating-webhook-configuration
webhooks:
- name: vworkload.kb.io
  namespaceSelector:
    matchExpressions:
    - key: kubernetes.io/metadata.name
      operator: NotIn
      values: ["kube-system", "kube-node-lease", "kube-public", "cert-manager", "gpu-cost-system"]
  matchConditions:
  - name: only-gpu-workloads
    expression: >-
      has(object.spec.template) &&
      object.spec.template.spec.containers.exists(c,
        has(c.resources) && has(c.resources.limits) && "nvidia.com/gpu" in c.resources.limits)
```

**Step 3 — the documented decision.** This paragraph goes in `WEBHOOK-RISK.md`, and it is verbatim the interview answer:

> Budget enforcement uses **`failurePolicy: Fail`** because silently admitting an over-budget GPU workload is an irreversible cost event — on a GPU cloud you are billed from the first instance-second, so a decision the webhook misses cannot be undone by a later reconcile, only refunded. The wedge is bounded by five things: (1) `rules` limited to `apps/v1 deployments,statefulsets` on `CREATE`/`UPDATE` only; (2) a `namespaceSelector` excluding `kube-system`, `kube-node-lease`, `kube-public`, `cert-manager`, and `gpu-cost-system` — the last being our own namespace, so we can always redeploy ourselves; (3) `matchConditions` narrowing to workloads that actually request `nvidia.com/gpu`, which drops matched traffic by roughly 15×; (4) `timeoutSeconds: 5` with an informer-cached lookup whose p99 is ~40 ms; (5) two replicas anti-affined across nodes behind a `PodDisruptionBudget` with `minAvailable: 1`. Worst case, a total webhook outage rejects new *tenant GPU Deployments* and nothing else — no control-plane writes, no non-GPU workloads, and not our own recovery path. Break-glass is `kubectl delete validatingwebhookconfiguration gpu-cost-operator-validating-webhook-configuration`, which restores admission immediately and disables enforcement until reapplied.

**Step 4 — prove it, including the failure behaviour.**

```
# 4a. Reject path — the whole point.
$ kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: {name: trainer, namespace: team-a}
spec:
  replicas: 8
  selector: {matchLabels: {app: trainer}}
  template:
    metadata: {labels: {app: trainer}}
    spec:
      containers:
      - name: train
        image: nvcr.io/nvidia/pytorch:24.10-py3
        resources: {limits: {nvidia.com/gpu: 4}}
EOF
Error from server (Forbidden): error when creating "STDIN": admission webhook
"vworkload.kb.io" denied the request: deployments.apps "trainer" is forbidden:
projected spend $2621.44 over window 720h exceeds active Budget "team-a-q3"
limit $400.00 (currently $118.20, this workload adds 32 concurrent GPUs)

# 4b. Warning path — allowed, but the user is told.
$ kubectl apply -f within-budget.yaml
Warning: this workload puts Budget "team-a-q3" at 87% of its $400.00 limit
deployment.apps/inference created

# 4c. failurePolicy demonstration — scale the webhook to zero.
$ kubectl scale deploy/gpu-cost-operator-controller-manager -n gpu-cost-system --replicas=0
$ kubectl apply -f within-budget.yaml
Error from server (InternalError): error when creating "within-budget.yaml":
Internal error occurred: failed calling webhook "vworkload.kb.io": failed to call
webhook: Post "https://gpu-cost-operator-webhook-service.gpu-cost-system.svc:443/
validate-apps-v1-deployment?timeout=5s": no endpoints available for service
"gpu-cost-operator-webhook-service"
        ↑ matches the documented Fail behaviour. Note it took ~0s, not 5s: "no
          endpoints" is an immediate failure, not a timeout.

# 4d. The exclusions hold while the webhook is down — this is the wedge boundary.
$ kubectl apply -f gpu-workload.yaml -n kube-system
deployment.apps/system-thing created           # namespaceSelector excluded it
$ kubectl apply -f cpu-only.yaml -n team-a
deployment.apps/cpu-thing created              # matchConditions excluded it
$ kubectl scale deploy/gpu-cost-operator-controller-manager -n gpu-cost-system --replicas=2
deployment.apps/gpu-cost-operator-controller-manager scaled
        ↑ AND we could still scale ourselves back up, because gpu-cost-system is excluded.
          That single line is the self-lockout test. Run it.
```

**Step 5 — the fault drill table**, one row per failure category, with the signal that distinguishes it:

| Fault | Under `Fail` | Under `Ignore` | First signal | Mitigation |
|---|---|---|---|---|
| Pod crash-loop / scaled to 0 | rejects all matched writes | admits all matched writes | `no endpoints available`; `..._fail_open_count` climbing under Ignore | ≥2 replicas anti-affined; PDB `minAvailable: 1`; narrow `rules` |
| Cert expired or SAN mismatch | rejects all matched writes | admits all matched writes | `x509:` in the error; `certmanager_certificate_expiration_timestamp_seconds` | cert-manager rotation; alert at ⅓ lifetime remaining |
| Dead connection after ungraceful node loss | intermittent rejects, ~minutes | intermittent silent admits | timeout spikes correlated with node/rollout events, not load | drain instead of power-off; `preStop` sleep; `maxUnavailable: 0`; small `timeoutSeconds` (k/k#80313) |
| Handler slow (uncached read) | rejects on timeout | **still degrades** — seats consumed while waiting | p99 of `..._webhook_admission_duration_seconds` rising; apiserver p99 rising with it | cached client only; deadline inside the handler; §8 arithmetic (k/k#90217) |
| Whole-chain deadline exhausted | reject, **webhook not named** | reject, **webhook not named** | `Timeout: request did not complete within requested timeout` | inventory and sum every applicable `timeoutSeconds` (k/k#128162) |
| `matchConditions` CEL runtime error | reject | admit | `..._match_condition_evaluation_errors_total` nonzero | guard with `has()`; test the expression against a nil-ish object |

**Step 6 — test it without a cluster, at two levels.**

*Unit*, for branch coverage: build the `*appsv1.Deployment` and `Budget` fixtures, hand the validator a `fake.NewClientBuilder().WithObjects(...).Build()` client, call `ValidateCreate` directly, assert on the returned error and warnings. Milliseconds per case; cover every branch here.

*Integration*, for wiring: under `envtest`, set `WebhookInstallOptions{Paths: []string{"config/webhook"}}`, register the webhook with the manager, wait for the serving port to accept connections, then `Create` a breaching Deployment through a real client. This is the only level that exercises the cert paths, the port, the generated `ValidatingWebhookConfiguration`, and the `admissionReviewVersions` negotiation — all the parts most likely to break in a real rollout, and none of which the fake client can touch, because **the fake client runs no admission at all**. One accept and one deny path is enough here; lesson 10 wires it up in full.

## Practice

Implement the **Budget-enforcement validating webhook** in `gpu-cost-operator`. Feeds the module deliverable's [checkpoint item 4](../checkpoint.md) directly.

1. **Scaffold.** `kubebuilder create webhook --group apps --version v1 --kind Deployment --programmatic-validation` (this registers a `core: true` external-type resource in `PROJECT`). Implement `ValidateCreate`/`ValidateUpdate` to reject Deployments whose projected spend would exceed an **active** `Budget` in the same namespace with `enforcement: block`. Use `mgr.GetClient()` — the cached client — and return `apierrors.NewForbidden` with a message naming the Budget, the projected figure, and the limit.
2. **Configure deliberately.** In the `+kubebuilder:webhook` marker set `failurePolicy`, `sideEffects=None`, `timeoutSeconds`, and narrow `rules`; run `make manifests`. Then add a kustomize patch supplying `namespaceSelector` (excluding `kube-system`, `kube-node-lease`, and **your own namespace**) and at least one `matchConditions` expression restricting the webhook to workloads that actually request `nvidia.com/gpu`.
3. **Document the risk.** Write `WEBHOOK-RISK.md` covering: the policy you chose and why; the five guards bounding the wedge; the exact break-glass command; and the two alerts you would page on (webhook p99 latency, and `fail_open_count` if you chose `Ignore`).
4. **Test at both levels.** Unit tests with the fake client for every branch (no GPU, no active budget, under limit, at warning threshold, over limit, lookup error). One envtest case each for accept and deny, driven through a real client so the wiring is exercised.
5. **Inventory the chain.** Run `kubectl get validatingwebhookconfiguration,mutatingwebhookconfiguration -o custom-columns=NAME:.metadata.name,HOOKS:.webhooks[*].name,TIMEOUT:.webhooks[*].timeoutSeconds,POLICY:.webhooks[*].failurePolicy` on a cluster and sum the timeouts that could apply to a single Deployment write. Record the number in `WEBHOOK-RISK.md` — that is your k/k#128162 exposure.

**Acceptance:**

- A budget-breaching Deployment is rejected with a `status.message` naming the Budget, the projected spend, and the limit.
- **Scaling the webhook Deployment to 0 demonstrates the chosen `failurePolicy`** — under `Fail`, matched applies are rejected with a webhook-unavailable error; the observed behaviour matches what `WEBHOOK-RISK.md` says it will be.
- **With the webhook down**, a workload in `kube-system` still applies, a non-GPU workload in a tenant namespace still applies, and you can still `kubectl scale` the webhook Deployment back up. That last one is the self-lockout test.
- CEL `matchConditions` measurably reduces call volume: `apiserver_admission_match_condition_exclusions_total` is nonzero.
- The webhook tests pass in `make test`.

## Common pitfalls

1. **Omitting the webhook's own namespace from `namespaceSelector`.** Symptom: a wedged cluster you cannot redeploy out of. Mechanism: the webhook's own replacement Pod, or its Deployment update, is itself a matching write that requires the dead webhook to admit it. This is distinct from — and as important as — excluding `kube-system`. Fix: exclude both, and test it by scaling to zero and then scaling back up.
2. **Believing `failurePolicy: Ignore` makes a webhook safe.** Symptom: cluster-wide latency and control-plane leadership churn with no rejections in the logs. Mechanism: `Ignore` governs the outcome, not the wait; API-server seats and client-side deadlines are consumed identically either way (k/k#90217). Fix: budget latency (§8), exclude control-plane namespaces regardless of policy, and alert on `apiserver_admission_webhook_admission_duration_seconds`.
3. **Choosing `matchPolicy: Exact` because it sounds more precise.** Symptom: the policy silently does not fire for some clients. Mechanism: `Exact` matches only the literal group/version/resource, so a request arriving through any other served version of the same resource bypasses the webhook. Fix: leave the default `Equivalent` for anything security- or cost-relevant.
4. **Forgetting subresources.** Symptom: `kubectl scale deploy/trainer --replicas=64` sails past a budget webhook that rejects the same change via `kubectl apply`. Mechanism: `rules.resources: ["deployments"]` does not match `deployments/scale`; the request carries `subResource: "scale"` and a different object kind (`autoscaling/v1 Scale`). Fix: enumerate `deployments/scale` explicitly, and handle the `Scale` shape in the handler.
5. **Doing an uncached read (or any network call) inside the handler.** Symptom: apiserver p99 rises across the whole cluster during rollouts. Mechanism: every millisecond in the handler is a held API-server seat multiplied by the matched request rate (§8). Fix: the manager's cached client only; if you truly need live data, precompute it in the reconciler and read the result from your own CR status.
6. **Treating `ValidatingAdmissionPolicy` as a drop-in for every webhook.** Symptom: an elegant CEL policy that cannot express the actual rule. Mechanism: policy CEL has no collection queries and no outbound calls — it sees `object`, `oldObject`, `request`, `namespaceObject`, `authorizer`, and one bound `params`. Fix: split the policy — static/relational checks in VAP, fleet-aggregating checks in the webhook, which shrinks the webhook's matched surface as a bonus.
7. **Declaring `sideEffects: None` on a webhook that writes somewhere.** Symptom: `kubectl apply --dry-run=server` performs a real side effect. Mechanism: the declaration is what makes the API server willing to send dry-run requests at all; it is trusted, not verified. Fix: `NoneOnDryRun` plus an actual `if req.DryRun { return }` branch.

## Self-check

- **`failurePolicy: Fail` and your webhook pod is down — what happens, and how do you avoid self-lockout?**
  **Answer:** The API server cannot obtain a valid `AdmissionReview` response (connection refused, no endpoints, TLS failure, timeout, or a `uid` mismatch all count), so under `Fail` it **rejects every matching write** with `Internal error occurred: failed calling webhook "<name>"`. If the webhook's `rules` and selectors also match writes in the namespace hosting the webhook — including the Pod the ReplicaSet controller is trying to create to replace it — you cannot recover by redeploying, because redeploying requires admission. That is the self-lockout. Avoid it with all five of: a `namespaceSelector` excluding `kube-system`, `kube-node-lease`, and the webhook's **own** namespace; narrow `rules` (never `"*"/"*"/"*"`); `matchConditions` narrowing further to only the objects your policy is about; a small `timeoutSeconds` (3–5 s) so failures are fast; and ≥2 replicas anti-affined across nodes behind a PDB with `minAvailable: 1`. Break-glass when it happens anyway: `kubectl delete validatingwebhookconfiguration <name>` — deleting the *configuration*, not the pod, because the API server keeps calling as long as the configuration object exists. And monitor certificate expiry, because an expired serving cert triggers this identical path cluster-wide with no deploy having occurred.

- **Validating versus mutating ordering, and why can a validating webhook not change the object?**
  **Answer:** The API server runs, in order: built-in mutating plugins, `MutatingAdmissionPolicy`, then mutating webhooks **sequentially** (with one reinvocation pass for any webhook declaring `reinvocationPolicy: IfNeeded`); then structural schema validation, defaulting, and CEL `x-kubernetes-validations`; then built-in validating plugins, `ValidatingAdmissionPolicy`, and validating webhooks **in parallel**; then persist to etcd. A validating webhook returns only `allowed` plus an optional `status` and `warnings` — any `patch` field it sends is ignored by the API server. That is deliberate: because validation runs last, on the fully-mutated, fully-schema-validated object, every validating webhook can trust that what it inspected is byte-for-byte what will be stored. If validation could mutate, that guarantee would break for every other validating webhook in the chain (they run concurrently and would see different objects), and the mutation itself would bypass schema validation, since that phase is already past. Mutation is placed before schema validation precisely so that everything a mutator writes is re-validated before persistence.

- **When does `ValidatingAdmissionPolicy` replace a webhook, and what are its limits?**
  **Answer:** Use a `ValidatingAdmissionPolicy` (GA in 1.30, `admissionregistration.k8s.io/v1`) whenever the rule is expressible in CEL over the request object plus at most one bound `paramKind` resource — for example `object.spec.replicas * gpusPerPod <= params.spec.maxConcurrentGPUs`. It evaluates in-process in the API server, so it eliminates the pod, the Service, the TLS certificate, and the network hop, and with them all three failure mechanisms: no wedge, no cert-expiry outage, no dead-connection stall. Its binding also gives you `validationActions: ["Audit"]`, which is a genuine dry-run of a policy in production that no webhook can match. `MutatingAdmissionPolicy` is the mutating equivalent, GA in 1.36, using `ApplyConfiguration` or CEL-generated `JSONPatch`. The limits are structural: policy CEL runs under a cost budget and can access only `object`, `oldObject`, `request`, `params`, `namespaceObject`, and `authorizer` — **no collection queries and no outbound calls**. So "reject if projected spend plus current spend summed across every Pod in this namespace exceeds the limit" still needs a webhook, as does anything consulting an external billing API. The right architecture is both: static per-object checks as a policy, fleet-aggregating checks in a webhook whose `matchConditions` are correspondingly narrow.

- **Your webhook times out intermittently only right after each rolling update, never at steady state. What is the mechanism, and what do you change?**
  **Answer:** A stale or half-open TCP connection from the API server to a webhook pod that no longer exists — documented upstream as kubernetes/kubernetes#80313, still open, originally reported on EKS at 1.12.6 where an abrupt node shutdown left pod creation blocked for roughly 15 minutes after the webhook had already been rescheduled. Because the peer vanished without sending `FIN` or `RST`, the API server's cached connection looks usable and every call sits until its timeout expires. Changes, in order of leverage: (1) always drain rather than power off nodes hosting webhooks, so endpoints are withdrawn cleanly; (2) give the webhook a real `readinessProbe` on the serving port plus a `preStop` sleep and a generous `terminationGracePeriodSeconds`, so the pod leaves `Endpoints` *before* it stops serving; (3) set `maxUnavailable: 0` / `maxSurge: 1` on the webhook Deployment so a healthy backend always exists during the roll, plus a PDB with `minAvailable: 1`; (4) keep `timeoutSeconds` small (3–5 s), which bounds how much each stale connection costs. Confirm the diagnosis with `apiserver_admission_webhook_admission_duration_seconds` — the tell is that the latency spike correlates with rollout and node events rather than with request volume.

- **Someone tells you `failurePolicy: Ignore` means the webhook "can't break anything." What is wrong with that?**
  **Answer:** `Ignore` governs the *outcome* of a failed call, not the *time spent waiting for it*. Three concrete consequences. First, latency: every admission call holds an API-server concurrency seat for its full duration, and with API Priority and Fairness the default total is `--max-requests-inflight` (400) plus `--max-mutating-requests-inflight` (200) = 600 seats. A 5-second hang at 90 matched writes per second consumes 450 seats — 75% of the cluster's entire request concurrency — regardless of failure policy. Second, client deadlines: kubernetes/kubernetes#90217 documents an `Ignore` webhook matching all resources that degraded clusters because `kube-controller-manager`'s leader-election renewal (`PUT` with its own `?timeout=10s`) waited on the webhook's 10-second timeout, lost the lease, and churned leadership — the request was eventually admitted, just far too late. Third, chain composition: kubernetes/kubernetes#128162 shows several `Ignore` webhooks cumulatively exhausting the *enclosing request's* deadline, producing `Timeout: request did not complete within requested timeout - context deadline exceeded` with no webhook named, because none of them individually failed. The correct statement is that `Ignore` protects the correctness of the admission decision at the cost of silently dropping enforcement — which is why `apiserver_admission_webhook_fail_open_count` must be alerted on — while the availability risk is governed by latency, scope, and exclusions, all of which you must handle under either policy.

## Connections & what's next

This lesson closes the marker → controller-gen → manifest arc that started with lesson 05's CRD schema markers and continued through lesson 07's RBAC markers: three generators, one pipeline, one discipline — the manifest is a projection of the code's intent, regenerated and reviewed, never hand-edited. It also closes the module's availability-versus-correctness thread. Lesson 06's finalizers asked "what if cleanup cannot complete"; lesson 07's RBAC asked "what is the blast radius of compromise"; this lesson asks "what is the blast radius of the safety mechanism itself failing." The same fail-open/fail-closed tension recurs at every extension point, and naming it here is what makes it transferable to the scheduler and DRA-driver failure modes you meet next.

One asymmetry to carry forward: the webhook's `matchConditions` funnel and lesson 07's `resourceNames` scoping are the same idea at two layers — narrow the surface before you narrow the permission. **Next**, lesson 09 moves from "may this object exist" to "where should it run": the scheduler's extension points in order, how `nvidia.com/gpu` reaches the scheduler as an indivisible extended resource through the device-plugin gRPC API, and the DRA and Kueue layers that sit around the scheduler for GPU workloads — the place where `gpu-cost-operator`'s cost signal finally has somewhere to plug into a *placement* decision rather than an admission one.

## References & further reading

**Primary sources**

- [kubernetes.io — Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) — the authoritative wire contract: full `ValidatingWebhookConfiguration`/`MutatingWebhookConfiguration` examples, the `AdmissionReview` request/response JSON, the base64 JSONPatch form, and every field default quoted in §4–§5.
- [kubernetes.io — Admission Webhook Good Practices](https://kubernetes.io/docs/concepts/cluster-administration/admission-webhooks-good-practices/) — the official checklist: idempotence, `matchPolicy: Equivalent`, what never to intercept (`kube-system`, node leases, `TokenReview`/`SubjectAccessReview`, mirror Pods), the self-hosting deadlock warning, the "milliseconds" latency bar, and the staged rollout order (`Ignore` + test namespace first).
- [kubernetes.io — Admission Controllers Reference](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) — the default-enabled plugin list and the two-phase (mutating then validating) execution model diagrammed in §2.
- [kubernetes.io — Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/) — VAP in full: `paramKind`/`paramRef`, `matchConstraints`, `variables`, `messageExpression`, `validationActions`, and the CEL cost budget that bounds what it can do.
- [kubernetes.io — MutatingAdmissionPolicy API reference](https://kubernetes.io/docs/reference/kubernetes-api/admissionregistration/mutating-admission-policy-v1/) — the mutating counterpart, GA in 1.36; `ApplyConfiguration` versus `JSONPatch` mutation forms.
- [Kubebuilder Book — Webhook overview](https://book.kubebuilder.io/reference/webhook-overview.html) and [webhook markers](https://book.kubebuilder.io/reference/markers/webhook.html) — the scaffolding commands, the generated file layout, and the marker argument set reproduced in §13.
- [`sigs.k8s.io/controller-runtime/pkg/webhook/admission`](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/webhook/admission) — the generic `Validator[T]`/`Defaulter[T]` interfaces (the non-generic `CustomValidator` is now a deprecated alias), `Warnings`, and the `Allowed`/`Denied`/`Patched`/`Errored` response constructors.
- [kubernetes/kubernetes — `apiserver/pkg/admission/metrics`](https://github.com/kubernetes/kubernetes/tree/master/staging/src/k8s.io/apiserver/pkg/admission/metrics) — where every metric name and label set in §11 is defined, including the histogram buckets.

**Real-world engineering writeups**

- [kubernetes/kubernetes#80313 — "Admission webhooks affected by dead tcp connections"](https://github.com/kubernetes/kubernetes/issues/80313) — what it shows: ungraceful node loss leaves the API server holding a half-open connection to a departed webhook pod; ~15 minutes of blocked pod creation on EKS 1.12.6, with `kubectl drain` as the practical workaround. Still open.
- [kubernetes/kubernetes#90217 — "Webhook with failurePolicy: Ignore that applies to controller manager leases breaks clusters"](https://github.com/kubernetes/kubernetes/issues/90217) — what it shows: `Ignore` does not protect latency; a slow broad-matching webhook cost `kube-controller-manager` its leader-election lease and churned cluster leadership.
- [kubernetes/kubernetes#128162 — "Pod admission can fail due to webhooks + context deadline exceeded, even when all webhooks are set to failurePolicy = Ignore"](https://github.com/kubernetes/kubernetes/issues/128162) — what it shows: per-webhook timeouts compose against the enclosing request's deadline, producing an error that names no webhook at all. Reported on v1.31.1; closed as not-planned, so the operational burden of inventorying timeouts is yours.

**Deeper dives**

- [cert-manager — `cainjector`](https://cert-manager.io/docs/concepts/ca-injector/) — the `cert-manager.io/inject-ca-from`, `inject-ca-from-secret`, and `inject-apiserver-ca` annotations and the four kinds it patches (`ValidatingWebhookConfiguration`, `MutatingWebhookConfiguration`, `CustomResourceDefinition`, `APIService`). Read it once and cert plumbing stops being mysterious.
