---
lesson: "02.7"
title: "Kubebuilder project ownership and least-privilege RBAC"
module: "02"
concept: "Kubebuilder project ownership and least-privilege RBAC"
status: not-started
est_time: "16h"
prev: "06-controller-runtime-deep.md"
next: "08-admission-webhooks.md"
artifacts: []
sources: 12
---

# 02.7 · Kubebuilder project ownership and least-privilege RBAC

> **Concept.** Own the kubebuilder scaffold end-to-end: the marker→codegen pipeline that turns Go comments into CRDs, RBAC, and webhook manifests, and the least-privilege ClusterRole that survives a security review.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Lesson 06 gave you the controller-runtime machinery itself: the `Manager`, the cached `Client` versus the uncached `APIReader`, owner references and cascading garbage collection, and the finalizer dance that lets your reconciler do cleanup before an object disappears. Every one of those operations is an HTTP request to the API server, and every one of them passes through an authorization decision before it touches storage. A `Get` that your code compiles fine and your unit tests pass happily can return `403 Forbidden` in production because nobody wrote three words in a comment.

This lesson is where you stop treating the generated `ClusterRole` as scaffold boilerplate and start treating it as a first-class, reviewable artifact — produced by the same marker-comment mechanism that generates your CRDs and, in lesson 08, your webhook configuration. Get the discipline right here and admission webhooks in the next lesson are a small extension of a pipeline you already trust rather than a new one to learn cold.

## Why this matters

The RBAC your operator ships with is the blast radius it hands an attacker who pops the controller pod. At GPU scale that pod already touches Nodes, Pods, and probably Secrets (registry credentials, cost-API tokens). A reviewer who sees `verbs: ["*"]` on `secrets` cluster-wide will — correctly — block the release, and "the scaffold generated it" is not an answer a senior engineer gives. You must be able to point at the exact line in `Reconcile` that needs each grant.

Kubebuilder makes this tractable because RBAC is *generated from markers next to the code that uses the permission*, so the manifest and the intent live together and stay in sync through `make manifests`. Owning that pipeline — not just running `make` — is what separates "I used kubebuilder" from "I own this operator." The Tesla cryptojacking incident (below) is the concrete reason this is not academic: an over-privileged service account, once an attacker has any foothold at all, *is* the incident. Everything after the initial exposure was governed by what the compromised identity was already allowed to do.

## What's new here (calibration)

You already know, as a CKA:

- What a `ServiceAccount`, `Role`, `ClusterRole`, `RoleBinding`, and `ClusterRoleBinding` are; that RBAC is purely additive with no deny rules; that verbs are `get`/`list`/`watch`/`create`/`update`/`patch`/`delete`/`deletecollection`.
- How to read `kubectl auth can-i --as=system:serviceaccount:ns:sa`.
- That a Deployment's pod runs as its `serviceAccountName` and a projected, audience-bound token is mounted into the container.

What is internal to the extending layer and new here:

- The kubebuilder **project layout** and the `PROJECT` file as the machine-readable inventory that `create api` / `create webhook` mutate and that plugins reconcile against.
- The **marker → controller-gen → manifest** pipeline in full: how a `+kubebuilder:rbac` comment becomes a rule in `config/rbac/role.yaml`, how the same binary with different flags produces deepcopy code, CRD OpenAPI schemas, webhook configurations, and (since controller-tools v0.21) apply-configuration types.
- The **RBAC authorizer's actual matching algorithm** — rule visiting order, the combined `resource/subresource` string, and why the `Forbidden` message tells you exactly which rule is missing.
- Deriving a **least-privilege** ClusterRole from the verbs your reconciler actually calls, and defending it grant-by-grant.

Versions this lesson is written against: **kubebuilder v4.12** (`layout: go.kubebuilder.io/v4`, `PROJECT` schema `version: "3"`), **controller-tools v0.21.0**, **controller-runtime v0.24.1**, **Kubernetes 1.36**. Where behaviour is version-specific it is called out; the marker syntax and the RBAC object model have been stable for years, but the exact scaffolded file list moves between kubebuilder minor releases.

## Core concepts

### The problem before the mechanism

Permissions drift. That is the whole problem this machinery exists to solve.

The naive way to give a controller access is to hand-write a `ClusterRole` YAML and apply it. That artifact is correct exactly once — on the day you wrote it. Six months later someone adds a `r.List(ctx, &nodeList)` call to handle a new GPU SKU, the code compiles, the unit tests pass against a fake client that enforces nothing, CI is green, and the operator ships. The failure surfaces in production as a reconcile that logs `nodes is forbidden` every ten seconds and a `WorkloadCost` whose status never becomes `Ready`. Nobody linked the code change to the permission change because nothing in the repository connected them.

The opposite failure is worse and more common: someone hit that `Forbidden` once, widened the role to `verbs: ["*"]` on `apiGroups: ["*"]` to make the error go away, and never narrowed it again. Now the controller's ServiceAccount can delete every object in the cluster including Secrets it has never read, and the role has stopped describing anything at all about the code.

**The fix is to make the manifest a pure projection of the source.** If the permission is declared in a comment adjacent to the call that needs it, adding the call and adding the grant become the same diff, and regenerating is mechanical. That is what `controller-gen` is: a compiler from Go comments to Kubernetes manifests. Everything below is a consequence of taking that idea seriously.

### 1. What the scaffolding commands actually write

Three commands produce the entire project. Knowing which files each one touches is what lets you recover when a scaffold goes wrong.

```
kubebuilder init --domain example.com --repo github.com/you/gpu-cost-operator
kubebuilder create api --group cost --version v1alpha1 --kind Budget --resource --controller
kubebuilder create webhook --group cost --version v1alpha1 --kind Budget --programmatic-validation
```

| Command | Writes | Mutates `PROJECT` |
|---|---|---|
| `init` | `go.mod`, `Makefile`, `Dockerfile`, `cmd/main.go`, `config/{default,manager,rbac,prometheus,network-policy}/`, `hack/boilerplate.go.txt`, `.golangci.yml`, `test/e2e/` | creates it: `domain`, `repo`, `projectName`, `layout`, `version` |
| `create api` | `api/<version>/<kind>_types.go`, `internal/controller/<kind>_controller.go` + `_test.go`, `config/crd/bases/` entry, `config/samples/`, `config/rbac/<kind>_{admin,editor,viewer}_role.yaml` | appends a `resources[]` entry with `group`/`version`/`kind`/`api.crdVersion`/`controller: true` |
| `create webhook` | `internal/webhook/<version>/<kind>_webhook.go` + `_test.go`, `config/webhook/` wiring, `config/certmanager/` | adds `webhooks: {validation: true, webhookVersion: v1}` to that resource entry |

The `PROJECT` file is not decoration. It is the inventory every subsequent command and every kubebuilder plugin reads. `create webhook` looks up the group/version for a kind there; the `autoupdate` and `deploy-image` plugins read it; `kubebuilder edit --plugins=...` writes to it. Hand-editing scaffolded Go is normal and expected. Hand-editing `PROJECT` to claim something exists that does not will bite you on the next `create`.

A real `PROJECT` from the kubebuilder testdata project, trimmed to the shapes you will actually produce (kubebuilder v4.12 / schema `version: "3"`):

```yaml
# PROJECT
# Code generated by tool. DO NOT EDIT.
cliVersion: v4.12.0
domain: example.com
layout:
- go.kubebuilder.io/v4
projectName: gpu-cost-operator
repo: github.com/you/gpu-cost-operator
resources:
- api:
    crdVersion: v1
    namespaced: true
  controller: true
  domain: example.com
  group: cost
  kind: Budget
  path: github.com/you/gpu-cost-operator/api/v1alpha1
  version: v1alpha1
  webhooks:
    validation: true
    webhookVersion: v1
- api:
    crdVersion: v1
    namespaced: true
  controller: true
  domain: example.com
  group: cost
  kind: WorkloadCost
  path: github.com/you/gpu-cost-operator/api/v1alpha1
  version: v1alpha1
# A webhook on a *core* type you do not own — note core: true and no api: block.
- core: true
  group: apps
  kind: Deployment
  path: k8s.io/api/apps/v1
  version: v1
  webhooks:
    validation: true
    webhookVersion: v1
version: "3"
```

Two entries there are worth reading twice. `core: true` marks a resource whose Go types come from `k8s.io/api` rather than from your `api/` directory — that is how you scaffold a webhook against `apps/v1 Deployment`, which is exactly what lesson 08's Budget-enforcement webhook needs. And `namespaced: true` under `api:` is what makes the generated CRD `scope: Namespaced`; flip it and `create api` scaffolds a cluster-scoped kind instead.

### 2. The project layout, annotated

This is the structural map. Every arrow in the diagram is "generated from"; every file marked `(gen)` is overwritten by a make target and must never be hand-edited.

```
gpu-cost-operator/
│
├── PROJECT ─────────────────── inventory: domain, repo, layout, every api & webhook
│                               read by:  kubebuilder create *, plugins
│
├── cmd/
│   └── main.go ─────────────── HAND-EDITED. The only file you touch on every new
│                               api/webhook. Registers: scheme, each reconciler's
│                               SetupWithManager, each webhook's Setup...WithManager,
│                               leader election, metrics server, /healthz + /readyz
│
├── api/v1alpha1/
│   ├── budget_types.go ─────── HAND-EDITED. BudgetSpec/BudgetStatus structs.
│   │        │                  Carries: +kubebuilder:object:root=true
│   │        │                           +kubebuilder:subresource:status
│   │        │                           +kubebuilder:validation:* on fields
│   │        │                           +kubebuilder:printcolumn:*
│   │        ├──[object]───────▶ zz_generated.deepcopy.go        (gen · make generate)
│   │        ├──[crd]──────────▶ config/crd/bases/*.yaml          (gen · make manifests)
│   │        └──[applyconfig]──▶ api/v1alpha1/applyconfiguration/ (gen · make manifests)
│   └── groupversion_info.go ── SchemeBuilder + AddToScheme for this GV
│
├── internal/
│   ├── controller/
│   │   └── budget_controller.go  HAND-EDITED. Reconcile + SetupWithManager.
│   │            │                Carries: +kubebuilder:rbac:*
│   │            └──[rbac]──────▶ config/rbac/role.yaml           (gen · make manifests)
│   └── webhook/v1alpha1/
│       └── budget_webhook.go     HAND-EDITED. CustomValidator / CustomDefaulter.
│                │                Carries: +kubebuilder:webhook:*
│                └──[webhook]───▶ config/webhook/manifests.yaml   (gen · make manifests)
│
├── config/                       kustomize bases — the deployable surface
│   ├── crd/bases/               (gen) one file per CRD
│   ├── crd/patches/              hand-written: conversion-webhook patches, cainjection
│   ├── rbac/
│   │   ├── role.yaml            (gen) manager-role: ONE ClusterRole, all rbac markers
│   │   ├── role_binding.yaml     hand-written: binds manager-role → the SA
│   │   ├── service_account.yaml  the controller's identity
│   │   ├── leader_election_role.yaml (+ _binding)  namespaced Role: leases, cm, events
│   │   ├── metrics_auth_role.yaml (+ _binding)     tokenreviews/subjectaccessreviews
│   │   ├── metrics_reader_role.yaml                nonResourceURLs: /metrics
│   │   └── budget_{admin,editor,viewer}_role.yaml  for HUMANS, not the controller
│   ├── webhook/manifests.yaml   (gen) Validating/MutatingWebhookConfiguration
│   ├── manager/manager.yaml      the Deployment
│   ├── certmanager/              Issuer + Certificate for the webhook serving cert
│   └── default/kustomization.yaml  the top-level overlay that composes all of it
│
├── Makefile ─────────────────── pins CONTROLLER_TOOLS_VERSION, KUSTOMIZE_VERSION,
│                                ENVTEST_K8S_VERSION; installs tools into ./bin
└── test/e2e/ ────────────────── kind-based end-to-end suite (build tag `e2e`)
```

The hard rule the layout encodes: **`api/` holds types plus generated deepcopy, `internal/controller/` holds reconcile logic, `internal/webhook/` holds admission logic, and `cmd/main.go` wires all three into a `manager.Manager`.** Reading `main.go` top to bottom is the fastest possible audit of what an unfamiliar operator actually does — it names every reconciled type, every webhook, and whether leader election is on.

### 3. controller-gen: one binary, five generators

Two make targets, one binary, different flags. These are the literal recipes from a kubebuilder v4.12 scaffold:

```make
.PHONY: manifests
manifests: controller-gen
	"$(CONTROLLER_GEN)" rbac:roleName=manager-role crd webhook \
	  applyconfiguration:headerFile="hack/boilerplate.go.txt" \
	  paths="./..." output:crd:artifacts:config=config/crd/bases

.PHONY: generate
generate: controller-gen
	"$(CONTROLLER_GEN)" object:headerFile="hack/boilerplate.go.txt",year=$(YEAR) paths="./..."

CONTROLLER_TOOLS_VERSION ?= v0.21.0
KUSTOMIZE_VERSION ?= v5.8.1
```

Read the `manifests` line as four generators invoked in one pass over the same parsed package set:

| Generator flag | Reads markers | Emits | Where |
|---|---|---|---|
| `object` | `+kubebuilder:object:root=true`, `+kubebuilder:object:generate=true` | `DeepCopyInto`/`DeepCopy`/`DeepCopyObject` | `api/<v>/zz_generated.deepcopy.go` |
| `crd` | `+kubebuilder:validation:*`, `+kubebuilder:default`, `+kubebuilder:subresource:*`, `+kubebuilder:printcolumn:*`, `+kubebuilder:resource:*` | `CustomResourceDefinition` with an OpenAPI v3 schema per version | `config/crd/bases/` |
| `rbac:roleName=manager-role` | `+kubebuilder:rbac:*` | one `ClusterRole` named `manager-role`, plus one `Role` per distinct `namespace=` value | `config/rbac/role.yaml` |
| `webhook` | `+kubebuilder:webhook:*` | `ValidatingWebhookConfiguration` and/or `MutatingWebhookConfiguration` | `config/webhook/manifests.yaml` |
| `applyconfiguration` | type structure (no marker) | typed server-side-apply builders | `api/<v>/applyconfiguration/` |

`paths="./..."` is the crucial argument: controller-gen parses **every** Go package in the module, not just `internal/controller`. That is why an RBAC marker technically works anywhere — convention puts it above `Reconcile` because that is where the reader looks for it, not because the tool requires it.

The whole pipeline, end to end:

```
  YOU EDIT                    make manifests / make generate                 CLUSTER
  ────────                    ─────────────────────────────                  ───────

  budget_types.go
   +kubebuilder:object:root ──┐
   +kubebuilder:validation:  ─┤
   +kubebuilder:subresource: ─┤
                              │
  budget_controller.go        │      ┌──────────────────┐
   +kubebuilder:rbac:  ───────┼─────▶│  controller-gen  │
                              │      │  v0.21.0 (pinned │
  budget_webhook.go           │      │   in Makefile,   │
   +kubebuilder:webhook: ─────┘      │   installed to   │
                                     │     ./bin)       │
                                     └────────┬─────────┘
                                              │ parses ./... with go/packages
                                              │ (compiler ignores the comments;
                                              │  controller-gen is the only reader)
                    ┌─────────────┬───────────┼───────────┬──────────────┐
                    ▼             ▼           ▼           ▼              ▼
        zz_generated      config/crd/    config/rbac/  config/webhook/  applyconfiguration/
        .deepcopy.go       bases/*.yaml   role.yaml     manifests.yaml
             │                  │             │              │
             │ compiled in      └─────────────┴──────────────┴──▶ kustomize build config/default
             │                                                            │
             ▼                                                            ▼
        manager binary  ────────── deployed as ────────────────▶  kubectl apply -f -
                                                                          │
                                                    ┌─────────────────────┼──────────────────┐
                                                    ▼                     ▼                  ▼
                                            CustomResourceDefinition  ClusterRole    ValidatingWebhook-
                                                                      manager-role    Configuration
                                                                          │
                                                              ClusterRoleBinding
                                                                          │
                                                                          ▼
                                                        ServiceAccount gpu-cost-operator-
                                                          controller-manager (in gpu-cost-system)
                                                                          │
                                                                          ▼
                                                        the token projected into the manager pod
```

Two consequences follow, and both are gates you should enforce in CI rather than in review comments:

**Never hand-edit a `(gen)` file.** `config/rbac/role.yaml` is overwritten on the next `make manifests`. If you need a rule, you need a marker.

**Pin the generator.** `CONTROLLER_TOOLS_VERSION ?= v0.21.0` and the `bin/` install path exist so every developer and CI runs the same binary. A teammate with a globally installed `controller-gen v0.19` will produce a `role.yaml` that differs in field ordering and in generated CRD schema details, and the resulting churn makes "the manifest matches the markers" unverifiable. Make CI run `make manifests generate` and fail on a dirty working tree — that converts a review question into a build failure.

### 4. The marker reference you actually need

Markers are ordinary Go line comments beginning `+kubebuilder:` (a leading space after `//` is optional and both `//+kubebuilder:` and `// +kubebuilder:` parse). They are attached to whatever declaration immediately follows, or — for `rbac` — to the package. The Go compiler ignores them entirely.

**Type-level markers** (on the `type Budget struct` declaration):

| Marker | Effect |
|---|---|
| `+kubebuilder:object:root=true` | Makes the type a `runtime.Object`: generates `DeepCopyObject()`. Required on the top-level kind and its `List` type. |
| `+kubebuilder:subresource:status` | Adds the `/status` subresource to the CRD. Changes the write path (`Status().Update()`) **and** adds `budgets/status` as a separate RBAC target. |
| `+kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.replicas` | Adds `/scale`; enables `kubectl scale` and HPA against your CRD. |
| `+kubebuilder:resource:scope=Cluster,shortName=bg,categories=cost` | Cluster- vs namespace-scoped, `kubectl get bg`, `kubectl get cost`. |
| `+kubebuilder:storageversion` | Marks which version etcd stores when you have several. |
| `+kubebuilder:printcolumn:name="Limit",type=string,JSONPath=".spec.limitUSD"` | Adds a column to `kubectl get budgets`. |

**Field-level markers** (on struct fields, feeding the OpenAPI v3 schema):

| Marker | Schema effect |
|---|---|
| `+kubebuilder:validation:Minimum=0` / `Maximum=` | `minimum` / `maximum` |
| `+kubebuilder:validation:MinLength=1` / `MaxLength=` | `minLength` / `maxLength` |
| `+kubebuilder:validation:Enum=USD;EUR` | `enum: [USD, EUR]` |
| `+kubebuilder:validation:Pattern=` | `pattern` (RE2 regex) |
| `+kubebuilder:default=100` | `default:` — applied by the API server on write |
| `+optional` / `+kubebuilder:validation:Optional` | omits the field from `required` |
| `+kubebuilder:validation:XValidation:rule="self >= 0",message="..."` | a CEL rule in `x-kubernetes-validations` |
| `+listType=map` + `+listMapKey=name` | server-side-apply merge semantics for a list |

**RBAC markers** (package-scoped; conventionally above `Reconcile`):

```go
// +kubebuilder:rbac:groups=cost.example.com,resources=budgets,verbs=get;list;watch;update;patch
```

**Webhook markers** (on the validator/defaulter type, covered in depth in lesson 08):

```go
// +kubebuilder:webhook:path=/validate-cost-example-com-v1alpha1-budget,mutating=false,failurePolicy=fail,sideEffects=None,groups=cost.example.com,resources=budgets,verbs=create;update,versions=v1alpha1,name=vbudget-v1alpha1.kb.io,admissionReviewVersions=v1
```

The point of putting all four families in one table is that they are **one mechanism**. Once you can read an RBAC marker as "arguments that become fields of a `PolicyRule`", you can read a webhook marker as "arguments that become fields of a `ValidatingWebhook`", and the third family stops being new material.

### 5. RBAC markers, argument by argument

The marker is defined in controller-tools as `markers.MakeDefinition("kubebuilder:rbac", markers.DescribesPackage, Rule{})`, and the `Rule` struct's fields are exactly the marker's arguments. Full mapping:

| Marker argument | `PolicyRule` field | Notes |
|---|---|---|
| `groups=` | `apiGroups` | `""` (empty string) is the core group: pods, nodes, secrets, configmaps, services, events. Semicolon-separated for multiples. |
| `resources=` | `resources` | Plural, lowercase, as the API server names them. Subresources use a slash: `budgets/status`, `pods/log`, `deployments/scale`. |
| `verbs=` | `verbs` | Lowercase, semicolon-separated. **Enumerate; never `*`.** |
| `resourceNames=` | `resourceNames` | Restricts the rule to named objects. Silently ineffective for four verbs — see §7. |
| `urls=` | `nonResourceURLs` | Only valid in a ClusterRole. `*` here is a *suffix glob*, so `/healthz/*` works. |
| `namespace=` | — | Not a rule field. If set, controller-gen emits the rule into a namespaced **Role** in that namespace instead of into the ClusterRole. |
| `roleName=` | — | Not a rule field. Overrides the target role's name for this marker (added in controller-tools v0.21.0); useful when one module produces several roles. |

Because the marker `DescribesPackage`, controller-gen collects *every* RBAC marker in the module and then normalizes and merges them before emitting. The merge key is the tuple **(groups, resources, resourceNames, urls)** — each field deduplicated and sorted. Two markers that share that key have their verb sets unioned into one rule. That has a practical consequence you will notice the first time you read a generated `role.yaml`:

```go
// +kubebuilder:rbac:groups="",resources=pods,verbs=get;list;watch
// ... elsewhere in the module ...
// +kubebuilder:rbac:groups="",resources=pods,verbs=delete
```

produces **one** rule with `verbs: [delete, get, list, watch]`, not two rules. So a stray `delete` marker in a helper package you forgot about silently widens the role that the reviewer is reading. The audit therefore has to be `grep -rn 'kubebuilder:rbac' .` across the whole module, not a read of one controller file.

The emitted result is a single ClusterRole:

```yaml
# config/rbac/role.yaml  (generated — DO NOT EDIT)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: manager-role
rules:
- apiGroups: ["cost.example.com"]
  resources: ["budgets"]
  verbs: ["get", "list", "watch", "patch", "update"]
- apiGroups: ["cost.example.com"]
  resources: ["budgets/status"]
  verbs: ["get", "patch", "update"]
- apiGroups: ["cost.example.com"]
  resources: ["budgets/finalizers"]
  verbs: ["update"]
- apiGroups: [""]
  resources: ["nodes", "pods"]
  verbs: ["get", "list", "watch"]
```

Note the last rule: `nodes` and `pods` merged because they shared groups/verbs after normalization. That merge is cosmetic, not semantic — a `PolicyRule` is a cross product of its lists, so `{nodes, pods} × {get, list, watch}` is exactly the six permissions you asked for.

### 6. What the API server does with that ClusterRole

You cannot reason about least privilege without knowing the matching algorithm. Here it is, as the RBAC authorizer in `plugin/pkg/auth/authorizer/rbac` implements it.

Every request is first converted to a set of **authorizer attributes**: user (and groups), verb, and either a resource tuple (API group, resource, subresource, namespace, name) or, for non-resource paths, a path string. The mapping from HTTP method to verb is fixed: `GET` on a collection → `list`, `GET` on an item → `get`, `GET` with `?watch=true` → `watch`, `POST` → `create`, `PUT` → `update`, `PATCH` → `patch`, `DELETE` on an item → `delete`, `DELETE` on a collection → `deletecollection`.

The authorizer then visits every rule that applies to that user, in a fixed order: **ClusterRoleBindings first, then RoleBindings in the request's namespace.** For each rule it asks five questions, and a rule allows the request only if all five say yes:

1. **VerbMatches** — is the request's verb in `rule.verbs`, or is `*` present?
2. **APIGroupMatches** — is the request's API group in `rule.apiGroups`, or is `*` present?
3. **ResourceMatches** — the request's resource and subresource are joined into a single string, `combinedResource = resource + "/" + subresource` when a subresource is present, and that string is compared against `rule.resources`. This is the mechanism behind "subresources are separate grants".
4. **ResourceNameMatches** — if `rule.resourceNames` is empty, everything matches; otherwise the request's object name must be in the list.
5. For non-resource requests, **NonResourceURLMatches** replaces 2–4: the request path must match an entry in `rule.nonResourceURLs`, with `*` acting as a suffix glob.

The first rule that says yes short-circuits and the decision is `DecisionAllow` with a reason string naming the binding (`RBAC: allowed by ClusterRoleBinding "manager-rolebinding" of ClusterRole "manager-role" to ServiceAccount "…"`). If no rule matches, RBAC returns **`DecisionNoOpinion`** — not "deny". That distinction matters: RBAC is one authorizer in a chain, and returning "no opinion" lets a later authorizer (node authorizer, webhook authorizer) still allow the request. The request is denied only when the *whole chain* has no opinion, which is why people describe RBAC as additive with no deny rules.

Here is that path drawn end to end, with a real failure traced backwards:

```
 THE REQUEST                                   THE AUTHORIZATION DECISION
 ───────────                                   ──────────────────────────

 manager pod
  │ serviceAccountName: gpu-cost-operator-controller-manager
  │ projected token at /var/run/secrets/kubernetes.io/serviceaccount/token
  │
  │ r.Status().Update(ctx, &budget)
  │   → PUT /apis/cost.example.com/v1alpha1/namespaces/team-a/budgets/q3/status
  ▼
 ┌──────────────────────────────────────────────────────────────────────────┐
 │ kube-apiserver                                                           │
 │                                                                          │
 │  1. AUTHN     token → user "system:serviceaccount:gpu-cost-system:       │
 │                            gpu-cost-operator-controller-manager"         │
 │                       groups [system:serviceaccounts, …:gpu-cost-system, │
 │                               system:authenticated]                      │
 │                                                                          │
 │  2. ATTRS     verb=update  group=cost.example.com  resource=budgets      │
 │               subresource=status  namespace=team-a  name=q3              │
 │               combinedResource = "budgets/status"   ◀── the joined string │
 │                                                                          │
 │  3. AUTHZ (RBAC)                                                         │
 │        visit ClusterRoleBindings for this user ──▶ manager-rolebinding   │
 │                                                     └▶ ClusterRole       │
 │                                                        manager-role      │
 │             rule[0] groups=[cost.example.com]                            │
 │                     resources=[budgets]                                  │
 │                     verbs=[get list watch patch update]                  │
 │                     ├ VerbMatches("update")           ✓                  │
 │                     ├ APIGroupMatches("cost…")        ✓                  │
 │                     └ ResourceMatches("budgets/status" vs ["budgets"]) ✗  │
 │                                                          └── NO MATCH    │
 │             rule[1] resources=[nodes pods] …           ✗                  │
 │        visit RoleBindings in namespace team-a ──▶ (none)                 │
 │                                                                          │
 │        → DecisionNoOpinion  → chain exhausted → 403                      │
 └──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
  budgets.cost.example.com "q3" is forbidden: User "system:serviceaccount:
  gpu-cost-system:gpu-cost-operator-controller-manager" cannot update resource
  "budgets/status" in API group "cost.example.com" in the namespace "team-a"
                                    │
                                    │  read it backwards:
                                    ▼
     verb ──── "update"        ─┐
     resource ─ "budgets/status" ├──▶ the missing marker is exactly:
     group ──── "cost.example.com"│    +kubebuilder:rbac:groups=cost.example.com,
     scope ──── namespaced        ┘      resources=budgets/status,verbs=get;update;patch
```

**Learn to read the `Forbidden` message as a marker.** The API server builds it from the same attributes the authorizer matched on, in a fixed format. There are exactly three shapes:

- `User %q cannot %s resource %q in API group %q in the namespace %q` — namespaced resource request.
- `User %q cannot %s resource %q in API group %q at the cluster scope` — cluster-scoped, or a cluster-wide `list`.
- `User %q cannot %s path %q` — non-resource request (`/metrics`, `/healthz`).

The quoted resource string is the *combined* `resource/subresource`. So `cannot update resource "budgets/status"` is telling you literally which `resources=` value your marker needs. There is no guesswork in this class of bug once you know the format — which is the point.

### 7. Subresources, and the bug that passes CI

Three subresources matter for an operator, and each is a separate RBAC target because of the `combinedResource` join above:

| Subresource | Who writes it | Marker you need |
|---|---|---|
| `budgets/status` | `r.Status().Update()` / `r.Status().Patch()` — once the CRD has `+kubebuilder:subresource:status` | `resources=budgets/status,verbs=get;update;patch` |
| `budgets/finalizers` | the API server's *ownership* check when you set a `metadata.finalizers` entry or a blocking owner reference on someone else's object | `resources=budgets/finalizers,verbs=update` |
| `budgets/scale` | `kubectl scale`, HPA | `resources=budgets/scale,verbs=get;update;patch` |

The `/finalizers` grant is the one people cannot explain in interviews. It is not needed to add a finalizer to *your own* object — that is an ordinary `update` on the parent resource. It is needed when the API server runs its **owner-reference blocking check**: setting `ownerReferences[].blockOwnerDeletion: true` on object B pointing at object A requires the writer to hold `update` on `A`'s `finalizers` subresource, because `blockOwnerDeletion` effectively installs a deletion barrier on A. `controllerutil.SetControllerReference` sets `blockOwnerDeletion: true` by default, so any controller that creates owned children needs `<parent>/finalizers` `update`. Kubebuilder scaffolds the marker for you precisely because everyone hits this.

**The bug that passes CI.** `envtest` starts a real API server, but the kubebuilder scaffold starts it with authorization effectively wide open — your test client is an admin. Nothing in `make test` exercises the ClusterRole. So a missing `/status` marker compiles, passes `go vet`, passes every envtest assertion (status updates succeed), passes review if the reviewer only reads the Go, and fails the first time the operator runs in a real cluster under its own ServiceAccount. This is the single highest-frequency "works in CI, Forbidden in prod" bug in operator development, and the reason the Practice section below asks you to verify with `kubectl auth can-i --list` against the deployed SA rather than trusting a green test suite.

### 8. `resourceNames` and its four blind spots

`resourceNames` is the sharpest scoping tool RBAC gives you, and the easiest to use wrongly, because when it does not apply it is **silently ignored rather than rejected**. It cannot restrict:

- **`create`** — at authorization time the object does not exist yet and its name is not known (it may not even be set: `generateName` assigns it later).
- **`deletecollection`** — by definition it targets an unnamed set.
- **`list`** and **`watch`** — these are collection requests. The API server only knows a name if the client sends a `metadata.name` field selector, and even then, a rule with `resourceNames` will only authorize the request if the client explicitly passes that selector.

So a marker like:

```go
// WRONG: reads as "may only list the one lease" — actually grants nothing extra
//        and the list will 403 unless the client passes a name field selector.
// +kubebuilder:rbac:groups=coordination.k8s.io,resources=leases,resourceNames=gpu-cost-operator-lock,verbs=get;list;watch;update;patch
```

is not the tightening it looks like. The correct shape splits the rule:

```go
// Leader election: create + list/watch cannot be name-restricted, so keep them
// unrestricted but namespace-scoped; pin the mutating verbs to the one lock object.
// +kubebuilder:rbac:namespace=gpu-cost-system,groups=coordination.k8s.io,resources=leases,verbs=create;list;watch
// +kubebuilder:rbac:namespace=gpu-cost-system,groups=coordination.k8s.io,resources=leases,resourceNames=gpu-cost-operator-lock,verbs=get;update;patch;delete
```

That is a real, reviewable reduction: after it, a compromised controller can read and overwrite exactly one Lease by name, and can enumerate Leases only within `gpu-cost-system`.

### 9. `nonResourceURLs` and the metrics surface

Non-resource URLs are the API server's HTTP paths that are not backed by objects: `/healthz`, `/readyz`, `/livez`, `/version`, `/openapi/v2`, and `/metrics`. They can only appear in a `ClusterRole` (there is no namespace to scope them to), and `*` there is a **suffix glob**, so `/healthz/*` matches `/healthz/etcd` but `/health*z` is not a pattern.

For an operator this shows up in exactly one place: the manager's own `/metrics` endpoint. In current kubebuilder the metrics server is protected by delegated authentication and authorization — the manager takes the scraper's bearer token, calls `TokenReview` to authenticate it and `SubjectAccessReview` to authorize it, and only then serves the metrics. That produces **two** distinct roles in `config/rbac/`, for two different identities:

```yaml
# config/rbac/metrics_auth_role.yaml — bound to the CONTROLLER's SA.
# Lets the manager ask the API server "is this scraper allowed?"
kind: ClusterRole
metadata: {name: metrics-auth-role}
rules:
- apiGroups: ["authentication.k8s.io"]
  resources: ["tokenreviews"]
  verbs: ["create"]
- apiGroups: ["authorization.k8s.io"]
  resources: ["subjectaccessreviews"]
  verbs: ["create"]
---
# config/rbac/metrics_reader_role.yaml — bound to PROMETHEUS's SA, by you.
# Nothing binds this by default; that is deliberate.
kind: ClusterRole
metadata: {name: metrics-reader}
rules:
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
```

The marker form of the second one, if you want to generate it rather than keep it as static YAML:

```go
// +kubebuilder:rbac:urls=/metrics,verbs=get
```

Keep these mentally separate from `manager-role`. `metrics-auth-role` is a permission the *controller* needs so it can delegate; `metrics-reader` is a permission *Prometheus* needs. The classic symptom of getting it wrong is metrics that return `401` or `403` to the scraper forever while the operator itself looks perfectly healthy — nothing in the controller's logs mentions it, because the failure is on the scraper's side of a `SubjectAccessReview`.

### 10. Role versus ClusterRole, decided per-resource

The decision is not "which one for the operator" — it is per-resource, and a well-scoped operator uses both.

```
                       is the resource cluster-scoped?
                       (Node, PersistentVolume, CRD, Namespace, ClusterRole…)
                                     │
                      ┌── yes ───────┴───────── no ──┐
                      ▼                              ▼
              MUST be a ClusterRole      does the controller need it in
              (a Role literally cannot   namespaces other than its own?
               express a cluster-scoped        │
               resource)             ┌── yes ──┴── no ──┐
                                     ▼                  ▼
                              ClusterRole          namespaced Role
                              (grants it in         (namespace=… marker;
                               EVERY namespace)      grants it in exactly one)
```

For `gpu-cost-operator`:

- `Node` — cluster-scoped, read-only. ClusterRole, forced.
- `Pod` — namespaced, but the controller sums GPU requests across the whole fleet. ClusterRole, `get;list;watch` only.
- `Budget`, `WorkloadCost`, `GPUCostPolicy` — namespaced CRs that tenants create anywhere. ClusterRole.
- `Lease` (leader election) — namespaced, and only ever in `gpu-cost-system`. `namespace=gpu-cost-system` → Role.
- `ConfigMap` (alert output) — namespaced, only in `gpu-cost-system`. `namespace=gpu-cost-system` → Role.
- `Event` (`create;patch`) — namespaced; the recorder emits events into the *object's* namespace, so this one has to stay in the ClusterRole.

The important asymmetry: **a ClusterRole bound with a `ClusterRoleBinding` grants the rule in every namespace, present and future.** So demoting `configmaps` from the ClusterRole to a namespaced Role is not cosmetic — it changes "can read and write every ConfigMap in the cluster, including `kube-system`'s `kube-root-ca.crt` and any config that carries credentials" into "can read and write ConfigMaps in one namespace."

### 11. Aggregation: the roles that are for humans

`create api` also scaffolds `budget_admin_role.yaml`, `budget_editor_role.yaml`, and `budget_viewer_role.yaml`. These have nothing to do with the controller. They carry aggregation labels:

```yaml
kind: ClusterRole
metadata:
  name: budget-editor-role
  labels:
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
- apiGroups: ["cost.example.com"]
  resources: ["budgets"]
  verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
- apiGroups: ["cost.example.com"]
  resources: ["budgets/status"]
  verbs: ["get"]
```

The built-in `admin`, `edit`, and `view` ClusterRoles each carry an `aggregationRule` with a `clusterRoleSelectors` list matching those labels. A controller in kube-controller-manager watches for labelled ClusterRoles and **recomputes the aggregate role's `rules` field automatically** — which is why you must never hand-edit `admin`'s rules; they will be overwritten. The net effect: a developer who already has `edit` in namespace `team-a` gains create/update on `Budget` the moment you install your CRD and these roles, with no additional binding.

Getting this wrong in the other direction is a real leak. If you copy `manager-role`'s rules into the `view` aggregation because "it's easier", every user with `view` anywhere now has `list;watch` on Pods cluster-wide. The controller's permission set and the human permission set answer different questions and must stay independent.

One more RBAC rule to know before you install any of this: **privilege escalation prevention.** You may only create or update a Role/ClusterRole whose rules you already hold yourself, at the same scope. Otherwise you need the `escalate` verb on `roles`/`clusterroles`. Similarly, creating a RoleBinding to a role requires that you hold that role's permissions, or hold the `bind` verb on it. This is why `make deploy` run by a CI identity with a narrow role fails with a confusing `attempt to grant extra privileges` message — the *installer*, not the operator, is the one lacking permission. The fix is to grant the installer identity the union of what it installs (or `escalate`), never to widen `manager-role`.

### 12. Why wildcards fail review — with the numbers

`verbs: ["*"]` is not a shorthand for "the verbs I use." It is a statement about the future:

- It grants `delete` and `deletecollection` that no code path calls today. A compromised controller pod can issue one `DELETE /api/v1/namespaces/team-a/pods` and evict every workload in a namespace.
- It expands automatically as the API adds verbs. You are pre-approving behaviour that does not exist yet.

`resources: ["*"]` and `apiGroups: ["*"]` are strictly worse, because they cover **CRDs installed after your operator**. Install cert-manager next quarter and your operator can now read every `Certificate` and, transitively, the Secrets they name. Install an external-secrets operator and your controller can read its `SecretStore` credentials. Nobody re-reviews an existing role when a new CRD lands.

Put a number on the specific case reviewers care about most — `list;watch` on `secrets` cluster-wide. Assumptions stated so you can re-run it: a mid-sized GPU cluster with 40 namespaces, roughly 60 Secrets per namespace (service-account tokens, registry pull secrets, TLS material, app config), and an average serialized Secret of about 4 KiB.

```
  objects in the cache : 40 ns × 60 secrets     = 2,400 Secret objects
  initial LIST payload : 2,400 × 4 KiB          ≈ 9.4 MiB  (one response, one shot,
                                                            decoded from etcd by the
                                                            apiserver, then held in
                                                            the informer's store)
  informer memory      : ~2–3× serialized for   ≈ 19–28 MiB resident in the pod
                         decoded Go objects
  blast radius         : every credential in the cluster is now in the RSS of one pod,
                         plaintext, readable with a single `cat /proc/<pid>/mem`-class
                         primitive or one `kubectl exec`
  across the fleet     : × 40 clusters          = the entire estate's credentials
                                                  behind one identity's token
```

That last line is the argument that actually lands. A single over-broad marker in a repo that ships to 40 clusters is not one mistake; it is the same mistake instantiated 40 times, and the 40 instances all get compromised together. The cheap version of this exposure is often unnecessary anyway: if the controller needs one specific Secret, use `get` with `resourceNames` on that name and read it with `APIReader` (an uncached, direct read) rather than caching the whole class.

If you genuinely must watch a class of Secrets, the mitigation is a **cache selector**, not just a narrower role — controller-runtime lets you restrict what the informer stores:

```go
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    Scheme: scheme,
    Cache: cache.Options{
        ByObject: map[client.Object]cache.ByObject{
            &corev1.Secret{}: {
                // Only these Secrets ever enter the cache. The RBAC rule still has to
                // permit list;watch, but the memory and exposure both collapse.
                Label:      labels.SelectorFromSet(labels.Set{"cost.example.com/managed": "true"}),
                Namespaces: map[string]cache.Config{"gpu-cost-system": {}},
            },
        },
    },
})
```

RBAC bounds what is *permitted*; the cache selector bounds what is *held*. A review that only checks one of the two misses half the exposure.

### 13. Verifying what you actually shipped

Three checks, in increasing order of trust:

```bash
# 1. What does the manifest say? (static, pre-deploy — the grep every reviewer runs first)
$ grep -n '"\*"' config/rbac/role.yaml
# (no output == pass)

# 2. What did the cluster actually resolve for that identity? (post-deploy, authoritative)
$ kubectl auth can-i --list \
    --as=system:serviceaccount:gpu-cost-system:gpu-cost-operator-controller-manager

# 3. One specific call site, answered yes/no
$ kubectl auth can-i update budgets/status \
    --as=system:serviceaccount:gpu-cost-system:gpu-cost-operator-controller-manager \
    -n team-a
yes
```

`kubectl auth can-i --list` is implemented as a `SelfSubjectRulesReview` (or, with `--as`, the impersonated equivalent), which asks the API server to enumerate the effective rules rather than re-deriving them from YAML. It is the only check that accounts for *all* bindings — including ones some other chart applied to the same ServiceAccount, which is exactly the failure mode a manifest grep cannot see.

## Perspectives

**Reviewer / security perspective.** The first thing a reviewer does is `grep '"\*"'`. The second is to open the diff and check that every new rule arrived with a code change that needs it. A marker-driven pipeline makes the second check mechanical: if `role.yaml` changed and no Go file changed, someone hand-edited a generated file. The `RBAC-JUSTIFICATION.md` this lesson's Practice asks for is the artifact that turns a 15-minute review into a 3-minute one — one line per rule, tied to a call site.

**Developer perspective.** Colocation is the whole design. RBAC markers sit next to `Reconcile` for the same reason validation markers sit next to the struct field: the two things that must agree are edited in the same window. Drift becomes impossible rather than merely discouraged, provided CI regenerates and fails on a dirty tree.

**Attacker perspective.** Walk the Tesla incident from the other side: exposed dashboard → unauthenticated access → a service account already present in the cluster with broad access → mining containers deployed with deliberately throttled CPU to stay under alerting thresholds → cloud credentials exfiltrated through that same access. Only the *first* step was a network-exposure failure. Every step after it was governed by what a compromised identity was permitted to do, which is a design-time decision you make in a comment.

**Fleet-scale perspective.** Least privilege compounds across 40 clusters. A wildcard grant is one careless regenerate away from being `cluster-admin`-equivalent everywhere simultaneously, and the roll-out mechanism you built to make deployment easy is the same mechanism that makes the mistake instantaneous. One reviewed, marker-generated role is what makes "roll this operator out to 40 clusters" a safe sentence.

**Operational perspective.** The `Forbidden` error is a *good* failure — loud, specific, and traceable to one missing marker. The bad failure is the one where someone widened the role to silence it. Treat every `Forbidden` in the operator's logs as a design question ("does this call belong here?") before treating it as a permissions question.

## Real-world use cases

- **Tesla Kubernetes cryptojacking (RedLock research, February 2018)** — [VentureBeat writeup](https://venturebeat.com/security/redlock-hackers-used-teslas-public-cloud-for-cryptocurrency-mining/), [Engadget writeup](https://www.engadget.com/2018-02-20-tesla-cryptojacking-report.html). **What happened:** researchers found a Kubernetes dashboard exposed to the internet with no authentication. Inside it, credentials for Tesla's AWS environment — including an S3 bucket containing telemetry — were reachable. The attackers deployed cryptomining containers configured to keep CPU usage low and to route mining traffic through a non-standard port behind CloudFlare, specifically to evade the IP/usage-based detection that most cloud security tooling relied on at the time. **What it shows:** the intrusion vector was network exposure, but the *impact* was entirely a function of what the identity behind the dashboard could already do. If the reachable service account had held enumerated verbs on enumerated resources, "deploy mining workloads" and "read cloud credentials" would both have been denied at the API server with no exploit required to stop them.
- **Datadog, "How to monitor Kubernetes audit logs"** — [datadoghq.com](https://www.datadoghq.com/blog/monitor-kubernetes-audit-logs/). **What it shows:** the detection-side complement to design-time least privilege. Audit events carry `user.username`, `verb`, `objectRef.resource`, and the authorization decision, so you can alert on the *use* of a grant rather than only on its existence — an unexpected `list` on `secrets` from a service account, or any verb from a service account outside its usual set. This is the safety net for the case where the least-privilege review missed something, and it is the reason the Practice section's justification file is worth writing: it defines "usual" precisely enough to alert on deviations from it.

## Worked example

`gpu-cost-operator`'s reconciler, in full. It watches `Budget` CRs, lists Pods cluster-wide to sum requested GPUs, reads Node labels for GPU SKU and price class, publishes breach alerts as a ConfigMap in its own namespace, emits Events, and holds a leader-election Lease.

**Step 1 — grep the reconciler, build the permission set.** This is a mechanical pass, one row per client call:

| Call site in `budget_controller.go` | HTTP verb | GVK + subresource | Scope |
|---|---|---|---|
| `r.Get(ctx, req.NamespacedName, &budget)` | `get` | `cost.example.com/v1alpha1 Budget` | namespaced |
| `mgr` watch established by `For(&Budget{})` | `list`, `watch` | `Budget` | all namespaces |
| `r.List(ctx, &podList)` | `list`, `watch` | `v1 Pod` | all namespaces |
| `r.Get(ctx, nodeKey, &node)` | `get` | `v1 Node` | cluster |
| `mgr` watch from `Owns(&WorkloadCost{})` | `list`, `watch` | `WorkloadCost` | all namespaces |
| `controllerutil.SetControllerReference(&budget, &wc, scheme)` | `update` | `Budget/finalizers` | namespaced |
| `r.Create(ctx, &workloadCost)` | `create` | `WorkloadCost` | namespaced |
| `r.Status().Update(ctx, &budget)` | `update` | `Budget/status` | namespaced |
| `controllerutil.AddFinalizer(&budget, …)` + `r.Update` | `update` | `Budget` | namespaced |
| `r.Create/r.Update` on the alert ConfigMap | `create`, `update`, `get`, `list`, `watch` | `v1 ConfigMap` | `gpu-cost-system` only |
| `r.Recorder.Event(...)` | `create`, `patch` | `v1 Event` | object's namespace |
| controller-runtime leader election | `create`, `list`, `watch`, `get`, `update` | `coordination.k8s.io/v1 Lease` | `gpu-cost-system` only |

**Step 2 — write the markers that express exactly that, and nothing more.**

```go
// --- Budget CR lifecycle -----------------------------------------------------
// Read the Budget and keep a watch on it; patch spec/metadata to add & remove the
// finalizer. No `delete`: the controller never deletes a Budget, users do.
// +kubebuilder:rbac:groups=cost.example.com,resources=budgets,verbs=get;list;watch;update;patch
// Status subresource is a SEPARATE target (combinedResource "budgets/status").
// +kubebuilder:rbac:groups=cost.example.com,resources=budgets/status,verbs=get;update;patch
// Needed because SetControllerReference sets blockOwnerDeletion=true on children,
// which the apiserver authorizes against the PARENT's finalizers subresource.
// +kubebuilder:rbac:groups=cost.example.com,resources=budgets/finalizers,verbs=update

// --- Owned children ----------------------------------------------------------
// The controller creates, updates and (on drift) deletes WorkloadCost objects it owns.
// +kubebuilder:rbac:groups=cost.example.com,resources=workloadcosts,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=cost.example.com,resources=workloadcosts/status,verbs=get;update;patch

// --- Fleet observation (read-only, cluster-wide) -----------------------------
// Sum GPU requests across every namespace to compare against Budget.spec.limitUSD.
// +kubebuilder:rbac:groups="",resources=pods,verbs=get;list;watch
// GPU SKU + price-class labels live on Nodes. Node is cluster-scoped: this rule
// CANNOT be expressed in a Role, which is why manager-role is a ClusterRole at all.
// +kubebuilder:rbac:groups="",resources=nodes,verbs=get;list;watch

// --- Alert output (namespace-scoped => emitted as a Role, not a ClusterRole) --
// +kubebuilder:rbac:namespace=gpu-cost-system,groups="",resources=configmaps,verbs=get;list;watch;create;update;patch

// --- Events ------------------------------------------------------------------
// The recorder writes into the *object's* namespace, so this must stay cluster-wide.
// +kubebuilder:rbac:groups="",resources=events,verbs=create;patch

// --- Leader election ---------------------------------------------------------
// create/list/watch cannot carry resourceNames; the mutating verbs can and do.
// +kubebuilder:rbac:namespace=gpu-cost-system,groups=coordination.k8s.io,resources=leases,verbs=create;list;watch
// +kubebuilder:rbac:namespace=gpu-cost-system,groups=coordination.k8s.io,resources=leases,resourceNames=gpu-cost-operator-lock,verbs=get;update;patch;delete
func (r *BudgetReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
```

**Step 3 — regenerate and read the output.**

```
$ make manifests
bin/controller-gen rbac:roleName=manager-role crd webhook \
  applyconfiguration:headerFile="hack/boilerplate.go.txt" \
  paths="./..." output:crd:artifacts:config=config/crd/bases

$ grep -c '"\*"' config/rbac/role.yaml
0

$ head -32 config/rbac/role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: manager-role
rules:
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch"]
- apiGroups: [""]
  resources: ["nodes", "pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["cost.example.com"]
  resources: ["budgets", "workloadcosts"]
  verbs: ["get", "list", "watch"]
...
```

Note what happened to the `namespace=gpu-cost-system` markers: they are **not** in `role.yaml`. controller-gen wrote them into a separate namespaced `Role` in the same file stream. Check with `grep -A3 'kind: Role$' config/rbac/role.yaml`. If you expected a ClusterRole rule and cannot find it, the `namespace=` argument is the usual reason.

**Step 4 — verify against the running cluster, not the YAML.**

```
$ make deploy IMG=ghcr.io/you/gpu-cost-operator:v0.1.0
$ kubectl auth can-i --list \
    --as=system:serviceaccount:gpu-cost-system:gpu-cost-operator-controller-manager

Resources                                   Non-Resource URLs   Resource Names           Verbs
selfsubjectreviews.authentication.k8s.io    []                  []                       [create]
selfsubjectaccessreviews.authorization…     []                  []                       [create]
selfsubjectrulesreviews.authorization…      []                  []                       [create]
events                                      []                  []                       [create patch]
nodes                                       []                  []                       [get list watch]
pods                                        []                  []                       [get list watch]
budgets.cost.example.com                    []                  []                       [get list watch update patch]
budgets.cost.example.com/finalizers         []                  []                       [update]
budgets.cost.example.com/status             []                  []                       [get update patch]
workloadcosts.cost.example.com              []                  []                       [get list watch create update patch delete]
workloadcosts.cost.example.com/status       []                  []                       [get update patch]
tokenreviews.authentication.k8s.io          []                  []                       [create]
subjectaccessreviews.authorization.k8s.io   []                  []                       [create]
                                            [/healthz]          []                       [get]
                                            [/readyz]           []                       [get]
```

Read it line by line. The three `selfsubject*` rows come from the built-in `system:basic-user` ClusterRole that every authenticated identity gets — they are not yours and cannot be removed. The two `*Review` rows are `metrics-auth-role`, doing delegated authz for the metrics endpoint. Everything else traces to one marker. **No `secrets`. No `delete` on `pods` or `nodes`. No wildcards.** That output, pasted into a PR description, is the review artifact.

**Step 5 — the contrast.** The naive role someone writes at 5pm on a Friday to make a `Forbidden` go away:

```yaml
# ANTI-PATTERN — never ship this
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: {name: manager-role}
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

Four lines. `kubectl auth can-i --list` against it returns every resource in the cluster with `[*]`, including `secrets`, `clusterrolebindings` (so the identity can grant itself anything permanently), `nodes` with `delete`, and every CRD installed from now until the cluster is decommissioned. Diff the two `can-i --list` outputs and paste both into the justification file; the delta *is* the argument for the pipeline.

**Step 6 — the justification file.** One line per rule, naming the reconcile behaviour:

```markdown
| Rule | Why | Call site |
|---|---|---|
| `budgets` get;list;watch;update;patch | read the CR, add/remove finalizer | `Reconcile` L41, L88 |
| `budgets/status` get;update;patch | write computed spend + Ready condition | `Reconcile` L142 |
| `budgets/finalizers` update | SetControllerReference sets blockOwnerDeletion on children | `Reconcile` L97 |
| `pods` get;list;watch | sum requested GPUs fleet-wide vs Budget.spec.limitUSD | `costFromPods` L23 |
| `nodes` get;list;watch | read `cost.example.com/gpu-class` label for the price table | `priceForNode` L14 |
| `configmaps` (ns-scoped) create;update;… | publish breach alerts in gpu-cost-system only | `publishAlert` L11 |
| `leases` (ns-scoped, name-pinned) get;update;patch;delete | leader election on one named lock | controller-runtime |
```

Each row survives the question "which line of code stops working if I delete this rule?" — which is the only question a reviewer actually needs answered.

## Practice

Advance `gpu-cost-operator`: **regenerate its RBAC down to the minimum verbs and resources it actually uses.** This feeds the module deliverable's [checkpoint item 6](../checkpoint.md) directly.

1. **Audit.** `grep -rn 'r\.\(Get\|List\|Create\|Update\|Patch\|Delete\)\|Status()\.' internal/ api/` and build the call-site table from the Worked example: one row per client call, with the verb, the GVK, the subresource, and the scope. Include the watches established by `For`/`Owns`/`Watches` in `SetupWithManager` — those are `list` and `watch` grants that no explicit call site shows.
2. **Replace every `+kubebuilder:rbac` marker** with per-resource markers carrying only the observed verbs. Split leader-election Leases into an unrestricted `create;list;watch` marker and a `resourceNames`-pinned mutating marker. Scope the alert ConfigMap with `namespace=gpu-cost-system`.
3. **Regenerate** with `make manifests` and commit `config/rbac/role.yaml`. Re-run it; confirm `git status` is clean (the reproducibility bar).
4. **Contrast.** Write `role-naive.yaml` containing the single `["*"]/["*"]/["*"]` ClusterRole, and `diff` it against the generated one. Deploy each in turn to a kind cluster and capture `kubectl auth can-i --list` for both.
5. **Justify.** In `RBAC-JUSTIFICATION.md`, one interview-ready line per grant naming the reconcile behaviour that requires it (`pods get;list;watch — sum GPU requests fleet-wide to compare with Budget.spec.limitUSD`). Include both `can-i --list` outputs.

**Acceptance:**

- No wildcard verbs, resources, or apiGroups anywhere in `config/rbac/role.yaml` (`grep -c '"\*"' config/rbac/role.yaml` returns `0`).
- Every rule has a one-line justification tied to actual controller code.
- `make manifests` is reproducible: re-running produces no git diff.
- `kubectl auth can-i --list --as=system:serviceaccount:gpu-cost-system:gpu-cost-operator-controller-manager` shows no access you cannot justify, and specifically shows **no `secrets`** and **no `delete` on `pods` or `nodes`**.
- The leader-election Lease grant is name-pinned for its mutating verbs.

## Common pitfalls

1. **Granting `list`/`watch` on Secrets cluster-wide "in case a future feature needs it."** Symptom: nothing, until the pod is compromised — then every credential in the cluster is in one process's memory. Mechanism: `list`/`watch` populates the informer store with the full decoded objects, so the grant is not "may read a Secret" but "holds all Secrets resident, continuously." Fix: no reconcile line reading a Secret today means no grant today; if you need one, `get` with `resourceNames` plus an `APIReader` read, and a `cache.Options.ByObject` selector if you truly must watch.
2. **Forgetting `/status` and `/finalizers` are separate targets.** Symptom: status updates fail with `cannot update resource "budgets/status"` only in a real cluster; every test passes. Mechanism: the authorizer joins resource and subresource into one string before matching, and the kubebuilder envtest scaffold does not run your ClusterRole at all. Fix: verify with `kubectl auth can-i update budgets/status --as=…` after deploy, not with `make test`.
3. **Using `resourceNames` with `create`, `list`, `watch`, or `deletecollection`.** Symptom: the role *looks* pinned in review but grants nothing extra, and the `list` 403s at runtime. Mechanism: the object name is unknown at authorization time for `create`, and collection verbs have no single name to match. Fix: split the marker — collection verbs unrestricted (and namespace-scoped where possible), item verbs name-pinned.
4. **Letting `controller-gen` versions drift between machines.** Symptom: `make manifests` produces a large diff on a colleague's laptop with no source change. Mechanism: generator output formatting and CRD schema emission change between minor versions. Fix: `CONTROLLER_TOOLS_VERSION` pinned in the Makefile, tools installed into `./bin`, and a CI step that runs `make manifests generate` and fails on a dirty tree.
5. **Assuming the ClusterRole is the whole story.** Symptom: `can-i --list` shows verbs you never granted. Mechanism: some other chart, or an operator-lifecycle tool, bound an additional role to the same ServiceAccount. Fix: `can-i --list` (a `SelfSubjectRulesReview`) is authoritative; a grep of your own YAML is not.
6. **Widening `manager-role` to fix an `attempt to grant extra privileges` error at install time.** Symptom: `make deploy` fails, someone adds rules to `manager-role`, and the error persists. Mechanism: that error is privilege-escalation prevention firing on the *installer's* identity, not the operator's — you cannot create a role granting permissions you do not hold. Fix: grant the installer identity the union of what it installs, or the `escalate` verb; leave `manager-role` alone.

## Self-check

- **Which `+kubebuilder:rbac` marker produces which rule, and how does `make manifests` wire it into a running cluster?**
  **Answer:** Each marker argument maps onto a `PolicyRule` field: `groups`→`apiGroups` (with `""` meaning the core group), `resources`→`resources` (subresources joined with a slash, e.g. `budgets/status`), `verbs`→`verbs`, `resourceNames`→`resourceNames`, `urls`→`nonResourceURLs`. Two arguments are not rule fields: `namespace=` moves the rule out of the ClusterRole into a namespaced `Role` in that namespace, and `roleName=` (controller-tools v0.21+) overrides the target role's name. The marker is declared `DescribesPackage`, so `controller-gen` scans every package under `paths="./..."`, normalizes each rule (dedupe + sort each list), and merges rules whose (groups, resources, resourceNames, urls) tuple is identical by unioning their verbs — which is why a stray marker in a helper package silently widens the role. `make manifests` runs `controller-gen rbac:roleName=manager-role crd webhook applyconfiguration … paths="./..."`, writing `config/rbac/role.yaml`. `config/rbac/role_binding.yaml` binds `manager-role` to the controller's ServiceAccount, kustomize composes it via `config/default`, and `make deploy` applies the result. The manifest is a pure projection of the markers: regenerate, never hand-edit.

- **Why is a controller holding `list;watch` on Secrets cluster-wide a red flag, and how do you scope it down?**
  **Answer:** `list;watch` is not "may read a Secret on demand" — it populates the controller's informer store with every Secret in the cluster, decoded and resident. On a 40-namespace cluster averaging 60 Secrets of ~4 KiB, that is roughly 2,400 objects, a ~9.4 MiB initial LIST, and on the order of 19–28 MiB of decoded objects held continuously; compromising the pod or its projected token leaks all of it, and across a 40-cluster fleet it leaks the whole estate. It is also the exact shape of the Tesla 2018 incident, where cloud credentials reachable by an over-privileged identity turned a dashboard exposure into a full compromise. Scope down in this order: (1) remove it if no reconcile line reads a Secret; (2) if one named Secret is needed, use `get` with `resourceNames` on that name and read it through the uncached `APIReader` — note `list`/`watch` cannot be name-restricted, so this only works if you avoid them; (3) confine to one namespace with `namespace=` so the rule lands in a `Role`; (4) if a class of Secrets must be watched, add both a label-selector RBAC posture *and* a controller-runtime `cache.Options.ByObject` selector, because RBAC bounds what is permitted while the cache selector bounds what is actually held in memory.

- **Namespaced Role versus ClusterRole for this operator — how do you decide?**
  **Answer:** Decide **per resource**, not per operator. Any cluster-scoped resource (`Node`, `PersistentVolume`, CRDs themselves) *must* live in a ClusterRole — a `Role` cannot express it, so one such resource in the access set forces a ClusterRole to exist. Beyond that: namespaced resources the controller touches across arbitrary namespaces (Pods for fleet-wide GPU summing, tenant-created `Budget` CRs, Events written into the object's own namespace) stay in the ClusterRole; namespaced resources the controller only ever touches in its own namespace (the leader-election Lease, the alert ConfigMap) get demoted to a `Role` with the `namespace=gpu-cost-system` marker argument. The asymmetry that makes the demotion worth doing: a ClusterRole bound with a ClusterRoleBinding grants the rule in *every* namespace including ones created later, so moving `configmaps` into a Role converts "can write any ConfigMap in the cluster" into "can write ConfigMaps in one namespace." `gpu-cost-operator` therefore ships both objects.

- **Why does `verbs: ["*"]` fail review even when today's code only reads and updates?**
  **Answer:** A wildcard is a claim about the future, not a description of the present. It grants `delete` and `deletecollection` that no code path calls, so a compromised controller can issue a single collection delete and wipe a namespace; and it expands automatically to cover verbs the API adds later, so you are pre-approving behaviour that does not exist yet. `resources: ["*"]` and `apiGroups: ["*"]` compound this by covering **CRDs installed after your operator** — install cert-manager or an external-secrets controller next quarter and your operator silently gains access to their credential-bearing types, with nobody re-reviewing the role. The point of the marker pipeline is that the role is a tight upper bound on *observed* behaviour, re-derived on every `make manifests`; a wildcard severs that link and the manifest stops describing the code at all. That is what the module checkpoint means by "no wildcard verbs, justify every grant."

- **Walk the Tesla 2018 incident and map each step to the control that would have stopped it.**
  **Answer:** (1) A Kubernetes dashboard was exposed to the internet — stopped by not exposing it publicly (network/exposure control, the entry point but not the blast radius). (2) It required no authentication — stopped by requiring authn before any API access. (3) The identity reachable through it held broad, effectively admin-equivalent access, and cloud credentials were reachable through it — stopped by exactly this lesson's discipline: enumerated verbs on enumerated resources, no `secrets` grant that no code path needs, generated from markers and verified with `kubectl auth can-i --list`. (4) Attackers deployed cryptomining workloads, deliberately throttling CPU and proxying mining traffic to evade usage-based detection — stopped outright if the compromised identity lacked `create` on `pods`/`deployments`, which most controllers do not need. (5) Cloud credentials were exfiltrated through that same access — stopped by the Secrets scoping in the second self-check answer. Ordered by cost to implement: do not expose the dashboard; require authn; least-privilege RBAC generated from markers; audit-log alerting on unexpected verb/resource pairs per service account (the Datadog use case) as the safety net when the first three fail. The lesson to state out loud in an interview: only step 1 was a networking failure; steps 3–5 were all decisions someone made — or failed to make — in a permissions manifest.

## Connections & what's next

RBAC markers are the second instance of the marker → controller-gen → manifest pattern that lesson 05 introduced for CRD schemas. Recognising it as *one* pipeline with several generators (`object`, `crd`, `rbac`, `webhook`, `applyconfiguration`) is what makes the next instance feel familiar rather than new. This lesson's `/status` and `/finalizers` grants are the RBAC-layer completion of lesson 06's `Status().Update()` and `SetControllerReference` material — code that compiles and passes envtest can still be `Forbidden` in production, and now you can read the error backwards to the exact missing marker.

**Next**, lesson 08 takes the same colocated-marker discipline to the highest-stakes extension point in Kubernetes: admission webhooks. There the generated manifest is not a permission set but a synchronous gate in the API server's write path, so "reviewed grant by grant" becomes "reasoned about failure mode by failure mode" — a webhook does not just need the right permissions, it needs a `failurePolicy` you can defend at 3am with the cluster wedged.

## References & further reading

**Primary sources**

- [Kubebuilder Book — RBAC markers](https://book.kubebuilder.io/reference/markers/rbac.html) — the canonical marker reference; the argument list here is generated from the `Rule` struct in controller-tools, so it is always in sync with the tool.
- [`sigs.k8s.io/controller-tools/pkg/rbac`](https://pkg.go.dev/sigs.k8s.io/controller-tools/pkg/rbac) — the generator's own godoc: the `Rule` struct whose fields *are* the marker arguments, and the normalization/merge behaviour described in §5.
- [kubernetes.io — Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) — the authoritative object model: verbs, `resourceNames` limits, aggregation labels, the default `admin`/`edit`/`view` roles, and privilege-escalation prevention (`escalate`/`bind`).
- [kubernetes.io — RBAC Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/) — the official least-privilege checklist this lesson operationalizes; particularly good on why `list` on Secrets is equivalent to `get` on all of them.
- [kubernetes/kubernetes — `plugin/pkg/auth/authorizer/rbac`](https://github.com/kubernetes/kubernetes/tree/master/plugin/pkg/auth/authorizer/rbac) — the authorizer itself: `RuleAllows`, the `resource + "/" + subresource` join, and the `DecisionNoOpinion` semantics described in §6.
- [kubernetes/kubernetes — `responsewriters/errors.go`](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/apiserver/pkg/endpoints/handlers/responsewriters/errors.go) — where the three `Forbidden` message formats are built; read it once and you can decode any 403 into a marker.
- [Kubebuilder Book — project configuration (`PROJECT`)](https://book.kubebuilder.io/reference/project-config) — every field of the inventory file, including `core: true` and `external: true` resource entries.
- [kubernetes-sigs/kubebuilder — `testdata/project-v4`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/testdata/project-v4) — a complete, always-current scaffold: the `Makefile` recipes, the full `config/rbac/` file list, and the generated `role.yaml`, all quoted in this lesson.

**Real-world engineering writeups**

- VentureBeat, ["RedLock: Hackers used Tesla's public cloud for cryptocurrency mining"](https://venturebeat.com/security/redlock-hackers-used-teslas-public-cloud-for-cryptocurrency-mining/) — what it shows: an exposed, unauthenticated dashboard plus a broadly-privileged identity turning into full cloud-credential compromise.
- Engadget, ["Intruders 'borrowed' Tesla's public cloud for cryptocurrency mining"](https://www.engadget.com/2018-02-20-tesla-cryptojacking-report.html) — what it shows: independent corroboration, with the detail on CPU throttling and traffic proxying used to evade usage-based detection.
- Datadog, ["How to monitor Kubernetes audit logs"](https://www.datadoghq.com/blog/monitor-kubernetes-audit-logs/) — what it shows: the detection-side complement — alerting on the *use* of a grant (unexpected verbs or resources per service account) when design-time review misses something.

**Deeper dives**

- [`kubernetes-sigs/controller-tools`](https://github.com/kubernetes-sigs/controller-tools) — the generator. `controller-gen rbac -w` prints every RBAC marker option straight from the source; `controller-gen crd -w` does the same for schema markers. Use it to settle argument-name arguments in seconds.
