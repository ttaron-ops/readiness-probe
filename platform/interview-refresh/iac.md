---
area: "Infrastructure as Code"
kind: refresh
status: not-refreshed      # not-refreshed | refreshed
---

# 🔁 Infrastructure as Code — interview refresh

> Terraform/OpenTofu module design, state at scale, testing, drift, policy-as-code.
>
> You know this. Goal here is fast recall + crisp interview framing, not study.

## Talking points to have ready

- **Module design & composition.** Small, versioned, single-responsibility modules; a thin
  root/env layer wiring them; no logic in the root. Registry-published + pinned versions so
  environments don't drift by accident. Say why you *avoid* deeply nested modules (blast radius,
  plan noise, upgrade pain).
- **State at scale.** Remote state with locking (S3+DynamoDB / GCS / TF Cloud); **one state per
  blast-radius boundary** (per-env, per-service) — not one mega-state. `terraform_remote_state`
  or data sources for cross-stack reads. The failure story: a lost/locked/ corrupted state and
  how you recover (`state pull`/`push`, `import`, `-target` as a scalpel not a habit).
- **Drift & reconciliation.** Detect with scheduled `plan`; treat drift as a signal, not a
  fix-in-console. GitOps-for-infra (Atlantis/TF Cloud/Spacelift) so every change is a reviewed PR
  with a stored plan.
- **Testing & safety.** `validate` → `plan` review → policy gate → apply. Policy-as-code with
  **OPA/Conftest or Sentinel** (deny public buckets, enforce tags/regions). `terraform test`,
  Terratest, or `checkov`/`tfsec` in CI. Ephemeral environments for PRs.
- **The GPU-fleet angle.** IaC for GPU capacity is **capacity-and-quota-shaped**: reserved-
  instance/committed-use provisioning, quota requests, node pools by GPU SKU, and the
  bare-metal/Cluster-API path where the "cloud API" is yours (ties to the bare-metal module).

## Self-quiz

- When do you split Terraform state, and what's the boundary rule? *(blast radius / lifecycle /
  ownership — not "one per resource" and not "one per org".)*
- `count` vs `for_each` — why does `for_each` avoid the destroy-and-recreate churn? *(stable keys
  vs positional index.)*
- How do you roll out a provider or module major-version bump across 40 stacks without a big bang?
- What does policy-as-code catch that code review misses, and where do you enforce it (plan-time
  vs admission-time)?

## Refresh only if

- **OpenTofu vs Terraform (post-BSL).** The 2023 license change and the OpenTofu fork — where the
  ecosystem landed, state encryption in OpenTofu, and why some shops migrated. Have a one-line POV.
- **Stacks/Stacks-like orchestration** (TF Stacks, Spacelift, env0) if you last ran plain
  workspaces — the multi-env orchestration layer has moved.
