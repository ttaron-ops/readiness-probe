---
lesson: "02.7"
title: "Kubebuilder project ownership and least-privilege RBAC"
module: "02"
concept: "Kubebuilder project ownership and least-privilege RBAC"
status: not-started
est_time: "8h"
artifacts: []
---

# 02.7 · Kubebuilder project ownership and least-privilege RBAC

> **Concept.** Own the kubebuilder scaffold end-to-end: the marker→codegen pipeline that turns Go comments into CRDs, RBAC, and webhook manifests, and the least-privilege ClusterRole that survives a security review.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Why this matters

The RBAC your operator ships with is the blast radius it hands an attacker who pops the controller pod. At GPU scale that pod already touches Nodes, Pods, and probably Secrets (registry creds, cost-API tokens). A reviewer who sees `verbs: ["*"]` on `secrets` cluster-wide will — correctly — block the release, and "the scaffold generated it" is not an answer a senior engineer gives. You must be able to point at each grant and say the reconcile line that needs it. Kubebuilder makes this tractable because RBAC is *generated from markers next to the code that uses the permission*, so the manifest and the intent live together and stay in sync through `make manifests`. Owning that pipeline — not just running `make` — is what separates "I used kubebuilder" from "I own this operator."

## From operating to extending

You already know, as a CKA:

- What a `ServiceAccount`, `Role`, `ClusterRole`, `RoleBinding`, `ClusterRoleBinding` are; that RBAC is purely additive and deny-by-default; that verbs are `get/list/watch/create/update/patch/delete/deletecollection`.
- How to read `kubectl auth can-i --as=system:serviceaccount:ns:sa`.
- That a Deployment's pod runs as its `serviceAccountName` and the token is projected in.

What is internal to the extending layer and new here:

- The kubebuilder **project layout** and the `PROJECT` file as the source of truth that `create api`/`create webhook` mutate.
- The **marker → controller-gen → manifest** pipeline: how a `//+kubebuilder:rbac` comment becomes a rule in `config/rbac/role.yaml`, and how `deepcopy`, CRD, and webhook YAML are generated the same way.
- Deriving a **least-privilege** ClusterRole from the verbs your reconciler *actually calls*, and defending it grant-by-grant.

## Core notes

**Layout.** `kubebuilder init --domain example.com --repo github.com/you/gpu-cost-operator` then `kubebuilder create api --group cost --version v1alpha1 --kind Budget` gives you:

```
PROJECT                      # machine-readable inventory: domain, repo, every api/webhook
cmd/main.go                  # manager wiring: scheme, reconcilers, webhooks, health/metrics
api/v1alpha1/                # Budget types + zz_generated.deepcopy.go (generated)
internal/controller/         # budget_controller.go — Reconcile lives here
internal/webhook/v1alpha1/   # (added by `create webhook`) validators/defaulters
config/                      # kustomize bases: crd/, rbac/, webhook/, manager/, default/
Makefile                     # manifests, generate, docker-build, deploy targets
```

The `PROJECT` file is not decoration — `kubebuilder create webhook --kind Budget --programmatic-validation` reads it to know the group/version and to record that the kind now has a webhook. Hand-editing scaffolds is fine; hand-editing `PROJECT` to lie about what exists will bite you on the next `create`. Its `resources` list is the inventory `kubebuilder` and plugins reconcile against:

```yaml
# PROJECT (excerpt)
domain: example.com
repo: github.com/you/gpu-cost-operator
resources:
- api: {crdVersion: v1, namespaced: true}
  group: cost
  kind: Budget
  version: v1alpha1
  controller: true
  webhooks: {validation: true, webhookVersion: v1}
```

The layout follows a hard rule: **`api/` holds types + generated deepcopy; `internal/controller/` holds reconcile logic; `cmd/main.go` wires them into a `manager.Manager`.** `main.go` is where you register each reconciler (`SetupWithManager`), each webhook, the scheme, leader election, and the health/metrics servers — it is the one file you edit by hand on every new api/webhook, and reading it top-to-bottom is the fastest way to audit what an operator actually does.

**The codegen pipeline.** Two make targets, two tools, both `controller-gen`:

- `make generate` → runs `controller-gen object` → writes `zz_generated.deepcopy.go` (the `DeepCopyObject()` etc. that make your type a `runtime.Object`). Driven by `//+kubebuilder:object:root=true` on the type.
- `make manifests` → runs `controller-gen crd rbac:roleName=manager-role webhook` → writes `config/crd/bases/*.yaml`, `config/rbac/role.yaml`, `config/webhook/manifests.yaml`. Driven by `//+kubebuilder:validation:*`, `//+kubebuilder:rbac:*`, and `//+kubebuilder:webhook:*` markers respectively.

Markers are ordinary Go comments (`// +kubebuilder:...`, leading space optional) that controller-gen parses; the compiler ignores them. This is the whole trick: **the manifest is a projection of the source, so it is reproducible and reviewable in the same PR as the code that needs it.** Never hand-edit `config/rbac/role.yaml` — it is overwritten. Edit the marker.

The same mechanism drives the other generated artifacts, so once you understand RBAC markers you understand all of them:

| Marker | Placed on | `controller-gen` generator | Output |
|---|---|---|---|
| `+kubebuilder:object:root=true` | the CRD Go type | `object` | `DeepCopyObject()` in `zz_generated.deepcopy.go` |
| `+kubebuilder:validation:Minimum=0`, `+optional`, `+kubebuilder:default=` | struct fields | `crd` | OpenAPI schema in the CRD YAML |
| `+kubebuilder:rbac:...` | anywhere (convention: `Reconcile`) | `rbac` | rules in `config/rbac/role.yaml` |
| `+kubebuilder:webhook:...` | the validator/defaulter | `webhook` | `config/webhook/manifests.yaml` |

`make manifests` = `generate` (deepcopy) + the crd/rbac/webhook generators; both targets just invoke `controller-gen` with different flags declared in the `Makefile`. If a teammate's `role.yaml` doesn't match their markers, they forgot to re-run it — CI should run `make manifests generate` and fail on a dirty tree.

**RBAC markers.** The marker lives above `Reconcile` (convention, not requirement — controller-gen scans the whole package):

```go
// +kubebuilder:rbac:groups=cost.example.com,resources=budgets,verbs=get;list;watch;update;patch
// +kubebuilder:rbac:groups=cost.example.com,resources=budgets/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=cost.example.com,resources=budgets/finalizers,verbs=update
// +kubebuilder:rbac:groups="",resources=pods,verbs=get;list;watch
func (r *BudgetReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
```

Marker arguments map one-to-one onto a PolicyRule:

| Marker arg | Rule field | Notes |
|---|---|---|
| `groups` | `apiGroups` | `""` = core group (pods, secrets, nodes, configmaps). |
| `resources` | `resources` | subresources as `budgets/status`, `pods/log`. |
| `verbs` | `verbs` | semicolon-separated; **enumerate, never `*`**. |
| `resourceNames` | `resourceNames` | restrict to named objects (e.g. one leader-election Lease). Cannot be used with `list`/`watch`/`create`/`deletecollection`. |
| `namespace` | — | if set, controller-gen emits a namespaced **Role** in that namespace instead of a ClusterRole rule. |
| `urls` | `nonResourceURLs` | for `/metrics`, `/healthz` style non-resource paths. |

`controller-gen rbac:roleName=manager-role` aggregates every marker in the module into one `ClusterRole` (`config/rbac/role.yaml`), which `config/rbac/role_binding.yaml` binds to the controller's `ServiceAccount`. That is the wiring: marker → controller-gen → `role.yaml` → kustomize → applied ClusterRole → bound to the pod's SA.

The three `budgets`, `budgets/status`, `budgets/finalizers` markers reflect a real split you must get right: the main resource verbs let you read and patch spec/metadata; the `/status` subresource is a *separate* RBAC target (because the CRD enables the status subresource, `Status().Update()` writes go through it and need their own grant); `/finalizers` needs `update` so the controller can add/remove finalizers without holding broad `update` on the parent. Miss the `/status` grant and status updates fail at runtime with a Forbidden that is invisible until you hit it.

**Why wildcard verbs fail review.** `verbs: ["*"]` grants `delete` and `deletecollection` you never call, so a compromised controller can wipe every object of that resource; it also silently expands as new verbs are added to the API. `resources: ["*"]` or `apiGroups: ["*"]` is worse — it grants access to CRDs installed *after* your operator, including future secrets-bearing types. Least privilege means the rule is a tight upper bound on observed behavior: if `Reconcile` only reads Pods, the verbs are exactly `get;list;watch` — no `update`. Reviewers grep for `"*"` first; a clean role passes on sight.

**The metrics/health surface.** The scaffold also exposes `/metrics` and `/healthz`. In current kubebuilder the metrics endpoint is served with authn/authz protection built into the manager, and scraping it requires RBAC on the non-resource URL — expressed with the `urls` marker, e.g. `//+kubebuilder:rbac:urls="/metrics",verbs=get`, which becomes a `nonResourceURLs` rule bound to the Prometheus ServiceAccount (not the controller's). This is the one place `nonResourceURLs` shows up in an operator, and it's a common miss: metrics silently 401 until the scraper's SA has the grant. Keep it separate from the manager-role; it's the scraper's permission, not the controller's.

**Two audiences, two ClusterRoles.** The `manager-role` above is for the *controller's* ServiceAccount. Separately, kubebuilder scaffolds `budget_editor_role.yaml` / `budget_viewer_role.yaml` and the aggregation ClusterRoles (`config/rbac/role_binding.yaml` siblings) carrying the `rbac.authorization.k8s.io/aggregate-to-admin: "true"` / `-to-edit` / `-to-view` labels. Those exist so that *humans* who already have the built-in `admin`/`edit`/`view` ClusterRoles automatically gain matching access to your new CRD — aggregation means the built-in role's rules are recomputed to include any labeled ClusterRole. Don't confuse them with the controller's role: the controller needs `list;watch` on Pods; a `view` user should not. Keep the two permission sets independent.

**Namespaced vs cluster scope.** Use `namespace=` (→ Role) when the operator only ever touches objects in its own namespace *and* is deployed once per namespace. Use a ClusterRole when the operator watches or acts across namespaces — which it must if it `list;watch`es Pods cluster-wide to sum GPU cost, or reads Nodes (a cluster-scoped resource that *cannot* live in a Role at all). A single cluster-scoped resource in your access set forces a ClusterRole. The refinement is per-resource: keep Nodes/Pods in the ClusterRole, but pin leader-election Leases to a Role in the controller's namespace with `resourceNames` on the specific lease.

**Reproducibility.** The acceptance bar "`make manifests` produces no diff" has one non-obvious dependency: the **`controller-gen` version**. Kubebuilder pins it in the `Makefile` (`CONTROLLER_TOOLS_VERSION`) and installs it to `bin/` so everyone runs the same generator; a teammate with a globally-installed different version can produce a churny diff. Treat the pinned version as part of the API contract, bump it deliberately, and run `make manifests generate` in CI failing on a dirty tree — that turns "did you regenerate?" from a review comment into a gate.

## Worked example

`gpu-cost-operator`'s reconciler: watches `Budget` CRs, lists `Pods` cluster-wide to sum requested GPUs against node cost, reads `Node` for GPU SKU/price labels, writes an alert `ConfigMap` in its own namespace, and does leader election via a `Lease`. Markers that express exactly that:

```go
// Budget CR lifecycle + status:
// +kubebuilder:rbac:groups=cost.example.com,resources=budgets,verbs=get;list;watch;update;patch
// +kubebuilder:rbac:groups=cost.example.com,resources=budgets/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=cost.example.com,resources=budgets/finalizers,verbs=update
// Sum GPU requests across the fleet (read-only):
// +kubebuilder:rbac:groups="",resources=pods,verbs=get;list;watch
// GPU SKU + price labels live on Nodes (cluster-scoped, read-only):
// +kubebuilder:rbac:groups="",resources=nodes,verbs=get;list;watch
// Publish breach alerts as a ConfigMap in the operator's namespace:
// +kubebuilder:rbac:groups="",namespace=gpu-cost-system,resources=configmaps,verbs=get;list;watch;create;update;patch
```

Leader election is handled by controller-runtime, which already emits its own `coordination.k8s.io/leases` + `events` markers into the scaffold — leave them; don't duplicate. If you want to tighten even those, the lease grant can be pinned to the single election lease with `resourceNames=<operator>-leader-election` on the `update;patch;delete` verbs (keep `create` and `get;list;watch` unpinned, since `create`/`list`/`watch` can't carry `resourceNames`) — a small but real reduction that reviewers notice. Run `make manifests`; `config/rbac/role.yaml` now contains exactly these six rules, no `secrets`, no `*`. Verify the effective access after `make deploy`:

```
$ kubectl auth can-i --list \
    --as=system:serviceaccount:gpu-cost-system:gpu-cost-operator-controller-manager
Resources                       Verbs
budgets.cost.example.com        [get list watch update patch]
budgets.cost.example.com/status [get update patch]
pods                            [get list watch]
nodes                           [get list watch]
...
# no secrets, no delete on pods/nodes, no wildcards
```

Contrast the naive grant a hurried engineer writes:

```yaml
# ANTI-PATTERN — do not ship
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

That one rule grants delete-everything, cluster-wide, including future CRDs and all Secrets. The reviewer's job is to reject it; your job is to have never written it.

## Practice

Advance `gpu-cost-operator`: **regenerate its RBAC to the minimum verbs/resources it actually uses.**

1. Audit `internal/controller/budget_controller.go` (and any helper packages): list every client call (`r.Get`, `r.List`, `r.Status().Update`, `r.Create`, ...) and the GVK it targets. That set *is* your permission set.
2. Replace all `//+kubebuilder:rbac` markers with per-resource markers carrying only the observed verbs. Pin the leader-election Lease with `resourceNames` if you can name it; scope the alert ConfigMap with `namespace=`.
3. `make manifests` and commit `config/rbac/role.yaml`.
4. Write a naive `role-naive.yaml` with a single `["*"],["*"],["*"]` ClusterRole and `diff` it against the generated one. In a `RBAC-JUSTIFICATION.md`, add **one interview-ready line per grant** naming the reconcile behavior that requires it (e.g. `pods get;list;watch — sum GPU requests across the fleet to compare with Budget.spec.limit`).

**Acceptance:**
- No wildcard verbs, resources, or apiGroups anywhere in `config/rbac/role.yaml`.
- Every rule has a one-line justification tied to actual controller code.
- `make manifests` is reproducible: re-running produces no git diff.
- `kubectl auth can-i --list --as=system:serviceaccount:gpu-cost-system:gpu-cost-operator-controller-manager` shows no access you can't justify.

## Self-check

**(a) Which `+kubebuilder:rbac` marker produces which rule, and how does `make manifests` wire it?**
**Answer:** Each marker maps arg-for-arg onto a PolicyRule: `groups`→`apiGroups` (`""`=core), `resources`→`resources` (incl. subresources like `budgets/status`), `verbs`→`verbs`, `resourceNames`→`resourceNames`, `urls`→`nonResourceURLs`; `namespace=` switches the output from a ClusterRole rule to a namespaced Role. `make manifests` invokes `controller-gen rbac:roleName=manager-role`, which scans every Go package for these comment markers, aggregates them into one ClusterRole written to `config/rbac/role.yaml`; kustomize (`config/rbac/`) then binds that role to the controller's ServiceAccount via `role_binding.yaml`. The manifest is a pure projection of the markers, so it is regenerated, never hand-edited.

**(b) Why is a controller listing Secrets cluster-wide a red flag, and how do you scope it down?**
**Answer:** `list;watch` on `secrets` cluster-wide streams the plaintext of every Secret in the cluster through the controller's cache, so compromising the pod (or its projected token) leaks all cluster credentials — the single highest-value RBAC grant. Scope down by: (1) removing it if the controller doesn't truly need Secrets; (2) if it needs one specific Secret, use `get` with `resourceNames` on that name (no `list`/`watch`, which can't be name-restricted) and read it directly rather than caching; (3) confine to the operator's namespace with `namespace=`; (4) if it must watch a class of Secrets, filter the informer with a label selector and, ideally, a dedicated Role in one namespace rather than a ClusterRole.

**(c) Namespaced Role vs ClusterRole for this operator — how decide?**
**Answer:** Decide by the scope of the objects the reconciler touches. If it only ever accesses namespaced objects in its own namespace and is deployed per-namespace, a Role suffices. But `gpu-cost-operator` `list;watch`es Pods across all namespaces and reads `Node` (a cluster-scoped resource that *cannot* appear in a Role), so it requires a ClusterRole — one cluster-scoped resource in the access set forces it. Best practice is mixed: keep the cross-namespace/cluster-scoped grants (Pods, Nodes) in the ClusterRole, but demote namespace-local, name-pinnable grants (leader-election Lease, alert ConfigMap) to a namespaced Role with `resourceNames`, minimizing what the ClusterRole exposes.

## Resources

1. **Kubebuilder Book — RBAC markers** — canonical marker syntax and controller-gen behavior: https://book.kubebuilder.io/reference/markers/rbac.html (see also `.../reference/markers.html` and the "Running and deploying" chapter).
2. **kubernetes.io — Using RBAC Authorization** — PolicyRule semantics, resourceNames limits, aggregation: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
3. **controller-gen** — the generator itself (`sigs.k8s.io/controller-tools`): `controller-gen rbac -w` prints every RBAC marker option; useful to confirm exact arg names.
