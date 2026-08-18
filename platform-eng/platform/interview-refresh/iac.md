---
area: "Infrastructure as Code"
kind: refresh
status: not-refreshed      # not-refreshed | refreshed
---

# 🔁 Infrastructure as Code — interview refresh

> Terraform/OpenTofu module design, state at scale, testing, drift, policy-as-code.
>
> You know this. Goal here is fast recall + crisp interview framing, not study.

Interviewers at this level do not ask whether you can write HCL. They ask what happens
when forty engineers share one tool whose whole job is to remember what it built. Every
hard question in this area — state splitting, drift, refactors, blast radius, policy
gates — is downstream of one mechanism: **the state file is a cache of reality, and
everything Terraform does is a three-way diff between config, state, and the provider's
answer.** Get that sentence out early and the rest of the interview is you filling in
detail.

Version facts below were checked against the `hashicorp/terraform` and
`opentofu/opentofu` CHANGELOGs and LICENSE files on GitHub (Terraform `main` is
`1.17.0-dev`, OpenTofu `main` is `1.13.0` unreleased, read 2026-08-18). The vendor doc
sites are unreachable from this environment and were **not** relied on — where a
behaviour is not in a changelog, it is flagged as such.

## Talking points to have ready

### 1. The plan/apply cycle, stated precisely

Say this before anything else, because six other answers hang off it.

```
              ┌──────────────┐
   HCL ──────▶│   config     │ desired
              └──────┬───────┘
                     │
   state ────▶┌──────▼───────┐   ReadResource(prior)   ┌──────────┐
  (prior)     │  refresh /   │◀───────────────────────▶│ provider │──▶ cloud API
              │  read phase  │   real attributes now   └──────────┘
              └──────┬───────┘
                     │  three-way diff: config ⨯ prior state ⨯ refreshed reality
              ┌──────▼───────┐
              │  plan graph  │  actions: no-op / create / update /
              │  (DAG)       │           delete / delete+create (replace)
              └──────┬───────┘
                     │  terraform plan -out=tfplan   (binary; contains variable
                     │                                values → treat as a secret)
              ┌──────▼───────┐
              │    apply     │  walks DAG, default -parallelism=10
              └──────┬───────┘
                     │  writes new state after each resource completes,
                     │  bumps serial, releases lock
                     ▼
              ┌──────────────┐
              │ remote state │  ← lock held for the whole apply
              └──────────────┘
```

The load-bearing consequences:

- **State is a cache, not the truth.** Refresh is what reconciles it. `-refresh=false`
  makes plans fast and lies to you; that trade is fine in a PR pipeline that also runs a
  scheduled refresh job, and dangerous as a permanent default.
- **The plan file is a secret.** It embeds variable values and resource attributes
  (passwords, keys) as of plan time. Artifact stores holding plans need the same ACL as
  state.
- **`terraform plan -detailed-exitcode`** returns `0` = no changes, `1` = error,
  `2` = changes present. That is the entire API for a drift-detection cron job; you do
  not need to parse output.
- **Apply is not transactional.** It is a graph walk that persists state incrementally.
  A crash mid-apply leaves state describing what actually got built — which is why
  "apply failed" is normally recoverable and "someone Ctrl-C'd and deleted the state" is
  not.

### 2. Module design and composition

The interview answer is a shape, not a rule list:

```
  live/                          ← thin, no logic, one directory per blast-radius unit
    prod/us-east-1/gpu-cluster/  ← backend config + module calls + values
    prod/us-east-1/network/
    stage/us-east-1/gpu-cluster/
  modules/                       ← versioned, single-responsibility, published
    gpu-node-pool/               ← one concern each; composed by the live layer
    vpc/
    irsa-role/
```

- **Single responsibility, thin root.** A module owns one concern and exposes a small
  typed interface. The root/live layer wires modules together and holds values; if you
  find `for` expressions and conditional logic in the root, the logic belongs in a
  module.
- **Pin versions, and know what the operator means.** `~> 5.0` is `>= 5.0.0, < 6.0.0`;
  `~> 5.31.0` is `>= 5.31.0, < 5.32.0`. The pessimistic operator releases the
  rightmost component only. Use the tighter form for providers that break minor-to-minor.
- **Nesting depth is a cost, not a virtue.** Each level of module nesting: multiplies the
  variable pass-through you have to maintain, widens the plan noise (`module.a.module.b.
  module.c.aws_instance.this[0]` addresses), and couples upgrade timing — you cannot ship
  a fix in the leaf without releasing every wrapper above it. Two levels (live → module →
  small shared submodule) is where most shops land.
- **Lock the providers, not just the modules.** `.terraform.lock.hcl` records provider
  versions and checksums. Two hash forms appear: `h1:` (the modern zip hash) and `zh:`
  (registry-published hashes). CI on Linux plus laptops on macOS will fight over the file
  unless you populate every platform:

  ```console
  $ terraform providers lock \
      -platform=linux_amd64 -platform=darwin_arm64 -platform=linux_arm64
  ```

  Commit the result. (OpenTofu 1.12 changed its registry to serve both `zh:` and `h1:`
  hashes specifically to reduce this cross-platform friction — CHANGELOG, v1.12.0 upgrade
  notes.)

### 3. State at scale — splitting, locking, recovery

**What is actually in the file.** JSON with a `version` (4 for the whole 1.x line), the
`terraform_version` that last wrote it, a monotonically increasing `serial`, a random
`lineage` UUID identifying the state's ancestry, and a `resources` array holding, per
instance: the resource address, the provider that manages it, the full attribute set as
last read, and its dependency list. Three of those are the ones interviewers poke at:

| Field | Why it exists | Failure it causes |
|---|---|---|
| `serial` | Optimistic-concurrency counter; a push with a lower serial is rejected | "state serial is out of date" after a manual edit or a restored backup |
| `lineage` | Identifies *which* state this is, across renames and copies | Pushing a state with a different lineage requires `-force`; two stacks that accidentally share a lineage will happily overwrite each other |
| `dependencies` | Ordering for destroy, after the config is gone | `terraform destroy` in the wrong order when a resource was hand-edited out of state |

**Locking, per backend.** S3 gained native locking in Terraform 1.10 (dual-writing to
DynamoDB when both were configured) and it went GA in 1.11 as `use_lockfile = true`,
with the DynamoDB arguments deprecated in the same release (Terraform CHANGELOG 1.10.0,
1.11.0). OpenTofu shipped the equivalent in 1.10 ("the `s3` backend can now implement
locking without DynamoDB"). If your mental model is still "S3 for state, DynamoDB table
with a `LockID` string key for locks", update it — the DynamoDB table is legacy and worth
one sentence in the interview.

```hcl
terraform {
  backend "s3" {
    bucket       = "acme-tfstate-prod"
    key          = "us-east-1/gpu-cluster/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true          # S3-native lock object, GA in 1.11 / tofu 1.10
    encrypt      = true
    kms_key_id   = "arn:aws:kms:us-east-1:111122223333:key/…"
  }
}
```

**The splitting rule.** One state per **blast-radius and lifecycle boundary** — a thing
that is created, destroyed, and reviewed as a unit, owned by one team, and changing at
one cadence. Concretely: network/landing-zone (slow, high blast radius), per-environment
per-region platform (medium), per-service (fast). Not one state per resource (you lose
the dependency graph and drown in `terraform_remote_state`), not one state for the org
(every plan refreshes everything and every apply blocks everyone).

Do the arithmetic in the interview — it lands better than the principle:

> A 1,200-resource state refreshes roughly one provider API call per resource instance,
> ten in flight by default, at maybe 150–300 ms per call. That is on the order of 20–40
> seconds of refresh before the plan even starts, per invocation, and every engineer
> touching that state pays it and queues behind the lock. Split it and the same change is
> a 3-second plan on a state nobody else is holding. (Illustrative arithmetic — the
> per-call latency is provider- and API-dependent.)

**Cross-stack reads, in order of preference.** (1) A real data source that queries the
provider by tag or name — no coupling to another state's internals. (2) A published
value in a well-known store (SSM Parameter Store, Secrets Manager, a registry). (3)
`terraform_remote_state` — works, but it grants the consumer read access to the whole
producing state, including its secrets, and turns the producer's outputs into a public
API you can no longer rename. Say (3) is your last choice and why.

**The recovery story.** Have this sequence ready, because "tell me about a time state
went bad" is a standard question:

```console
# 0. Always: snapshot before touching anything.
$ terraform state pull > backup-$(date +%s).tfstate

# 1. Stale lock (the CI runner died mid-apply). ID comes from the error message.
$ terraform force-unlock 4d1e8f2a-...

# 2. Resource exists in cloud but not in state (someone clicked, or an apply half-failed).
$ terraform import aws_instance.gpu_node i-0abc123          # legacy CLI path
#   or, config-driven and plannable, since 1.5:
#   import { to = aws_instance.gpu_node, id = "i-0abc123" }
#   $ terraform plan -generate-config-out=generated.tf      # writes the HCL for you

# 3. Resource in state but gone/adopted elsewhere — forget it without destroying.
$ terraform state rm aws_instance.gpu_node

# 4. Surgical apply. A scalpel, not a habit — it skips the rest of the graph and the
#    next full plan will show whatever you deferred.
$ terraform apply -target=module.gpu_pool.aws_eks_node_group.this

# 5. Restore a corrupted state: edit the pulled JSON, bump serial, push.
$ terraform state push backup-1723900000.tfstate            # -force if lineage differs
```

`-target` is the answer interviewers listen for. The correct framing: it is an incident
tool, it produces a plan Terraform itself warns is not a complete plan, and if it appears
in a runbook that runs weekly, the state is split wrong.

### 4. Refactoring without churn — `count`, `for_each`, `moved`, `removed`

`count` addresses instances by **position** (`aws_instance.node[0]`), `for_each` by
**key** (`aws_instance.node["a100-us-east-1a"]`). Deleting the middle element of a
`count` list renumbers every element after it, and Terraform reads a renumber as
destroy-and-recreate. On a GPU fleet that is not a cosmetic problem: the plan says
`4 to destroy` for machines you cannot re-source this quarter.

```hcl
# Positional — removing "b" replans node[1] and node[2].
variable "pools" {
  type    = list(string)
  default = ["a", "b", "c"]
}
resource "aws_eks_node_group" "np" { count = length(var.pools) ... }

# Keyed — removing "b" touches exactly aws_eks_node_group.np["b"].
variable "pools" {
  type = map(object({ sku = string, size = number }))
}
resource "aws_eks_node_group" "np" { for_each = var.pools ... }
```

Use `count` only for the zero-or-one case, and note that OpenTofu 1.11 added an explicit
`lifecycle { enabled = … }` meta-argument for exactly that pattern (OpenTofu CHANGELOG
1.11.0); Terraform has no equivalent in its changelogs through 1.16.

Refactors are declarative now — this is the part people miss:

| Block | Since | What it does |
|---|---|---|
| `moved` | TF 1.1 | Records that an address changed, so Terraform re-keys state instead of destroy/create. Ship it in the module, not as an operator's `terraform state mv`. |
| `import` | TF 1.5 / tofu 1.7 (with `for_each`) | Adoption becomes a plannable, reviewable diff instead of an out-of-band CLI action |
| `removed` | TF 1.7 / tofu 1.7 | Drops an object from state *or* destroys it, declaratively, after the resource block is gone |

### 5. Drift, and what "GitOps for infra" actually buys

Drift is any divergence between the last applied plan and reality: console edits, an
autoscaler adjusting a count, a controller mutating a tag, a provider changing a default
between versions. Three practical moves:

1. **Detect on a schedule, not on demand.** A cron pipeline running `terraform plan
   -detailed-exitcode` per stack; exit 2 opens a ticket with the plan attached. Alert on
   *un-reconciled* drift older than N days, not on every diff.
2. **`-refresh-only` for the "reality changed and it's fine" case.** It updates state to
   match the world without proposing config changes, so you can accept an out-of-band
   change deliberately rather than having the next apply revert it.
3. **Stop fighting the controllers.** Anything a controller legitimately owns —
   cluster-autoscaler node counts, HPA-driven sizes, tags injected by a policy engine —
   gets `lifecycle { ignore_changes = [...] }`, or you and the controller will trade
   applies forever.

**The GitOps-for-infra layer** (Atlantis, HCP Terraform, Spacelift, Env0) exists for one
reason worth stating: it makes the plan an artifact that a human approves and the machine
then applies **unchanged**. Local `terraform apply` from a laptop means the thing
reviewed and the thing executed are different objects, and cloud credentials live on
laptops. The chain to describe: PR → `plan -out=tfplan` in CI → policy engine evaluates
`terraform show -json tfplan` → human approves the stored plan → runner applies that
exact plan file with a short-lived OIDC-federated role → state updated → PR merged.

```
   PR opened ──▶ init ──▶ validate ──▶ fmt -check ──▶ tflint
                                                       │
                            ┌──────────────────────────┘
                            ▼
                     plan -out=tfplan ──▶ show -json ──▶ ┌────────────────┐
                            │                            │ conftest / OPA │
                            │                            │  Sentinel      │
                            │                            └────┬───────────┘
                            │                        deny ◀───┘   allow
                            │                         │             │
                            │                    comment on PR   human approve
                            │                                        │
                            └────────────────── apply tfplan ◀───────┘
                                                (same artifact, OIDC role, no keys)
```

### 6. Testing and policy-as-code

Know the ladder and what each rung catches:

| Rung | Tool | Catches | Cost |
|---|---|---|---|
| Syntax/schema | `terraform validate`, `fmt -check` | Typos, wrong types, missing required args | milliseconds |
| Lint/opinion | `tflint` (+ provider rulesets) | Deprecated args, invalid instance types, naming | seconds |
| Static security | `checkov`, `trivy config` (tfsec's rules now ship inside Trivy) | Public buckets, unencrypted volumes, wide SGs | seconds |
| Plan policy | `conftest`/OPA, Sentinel | Anything expressible over the *actual planned change* | seconds |
| Unit/contract | `terraform test` (`.tftest.hcl`) | Module logic, computed names, conditional branches | seconds–minutes |
| Integration | Terratest, `terraform test` with real applies | Does it actually come up | minutes–hours |

**Policy over the plan JSON** is the rung people describe worst. `terraform show -json
tfplan` yields `resource_changes[]`, each with `.address`, `.type`, `.change.actions`
(`["no-op"]`, `["create"]`, `["update"]`, `["delete"]`, or `["delete","create"]` for a
replace) and `.change.after`. That is the input document:

```rego
package terraform.gpu

import rego.v1

# Deny creating a GPU instance without a cost-attribution tag.
deny contains msg if {
    rc := input.resource_changes[_]
    rc.type == "aws_instance"
    "create" in rc.change.actions
    startswith(rc.change.after.instance_type, "p5.")
    not rc.change.after.tags.CostCenter
    msg := sprintf("%s: p5-class instance without CostCenter tag", [rc.address])
}

# Deny *replacing* reserved GPU capacity — supply you cannot re-acquire.
deny contains msg if {
    rc := input.resource_changes[_]
    rc.type == "aws_ec2_capacity_reservation"
    rc.change.actions == ["delete", "create"]
    msg := sprintf("%s: plan replaces a capacity reservation", [rc.address])
}
```

The second rule is the one that shows GPU-platform judgement: the generic policy sets
care about tags and encryption; the GPU-fleet policy set cares about **actions that
destroy scarce capacity**.

**`terraform test`** went GA in 1.6 (`.tftest.hcl` files, `run` blocks that plan or apply
against the module under test and assert on the result). 1.7 added mocking —
`mock_provider`, `override_resource`, `override_data`, `override_module` — so a module's
logic can be unit-tested with no cloud account at all. 1.11 made `-junit-xml` GA for CI
reporting. (Terraform CHANGELOG 1.6.0, 1.7.0, 1.11.0.)

```hcl
# tests/naming.tftest.hcl — unit test, no credentials, no infrastructure
mock_provider "aws" {}

run "node_group_name_is_sku_qualified" {
  command = plan
  variables {
    cluster_name = "gpu-prod"
    sku          = "p5.48xlarge"
  }
  assert {
    condition     = aws_eks_node_group.this.node_group_name == "gpu-prod-p5-48xlarge"
    error_message = "node group name must encode the GPU SKU for scheduler targeting"
  }
}
```

**What policy-as-code catches that review misses**: review sees the diff an author chose
to show; policy sees every resource change in the plan, including the ones produced
indirectly by a module three levels down, and it applies at 3 a.m. to the emergency
change nobody reviewed. Plan-time policy and admission-time policy are complements, not
alternatives — plan-time is cheap and pre-merge but only sees what Terraform manages;
admission-time (Kyverno/Gatekeeper, see the security refresh) sees everything that
reaches the cluster, including what a human `kubectl apply`s at 3 a.m.

### 7. Secrets, and the plaintext-state problem

State stores every attribute the provider returned, including generated passwords and
private keys. `sensitive = true` only redacts CLI output. The mitigations, in the order
you should say them:

1. **Don't put it in state.** Terraform 1.10 added **ephemeral resources and ephemeral
   values** (read fresh in each phase, never written to plan or state); 1.11 added
   **write-only attributes**, so a provider can accept a password without persisting it.
   OpenTofu shipped its ephemeral-values equivalent in 1.11. (Both CHANGELOGs.)
2. **Encrypt the state itself.** OpenTofu 1.7 introduced client-side state encryption:
   AES-GCM with pluggable key providers — PBKDF2 passphrase, AWS KMS, GCP KMS, OpenBao —
   configurable and, since 1.8, `enforced` so an unencrypted write fails. This is the
   single most-cited functional divergence between the two tools; Terraform's changelogs
   through 1.16 contain no state-encryption feature, so on Terraform you encrypt the
   *bucket* (SSE-KMS) and treat state as a secret at rest.
3. **Restrict who can read it.** Bucket policy + KMS key policy, separate from who can
   apply.

### 8. Terraform vs OpenTofu — the one-line POV, with substance

HashiCorp relicensed Terraform under the Business Source License 1.1 in August 2023. The
`LICENSE` file in `hashicorp/terraform` today names the Licensed Work as "Terraform
Version 1.6.0 or later", now under IBM as licensor following IBM's acquisition of
HashiCorp, with a **Change Date four years after publication and a Change License of MPL
2.0** — that is, each version becomes MPL four years after it ships. The Additional Use
Grant permits production use but not offering a competitive hosted/embedded product.
OpenTofu is the fork of the last MPL-2.0 tree (1.5.x), governed under the Linux
Foundation, and `opentofu/opentofu`'s LICENSE is MPL 2.0.

The features have since diverged in both directions — worth knowing which way round:

| Capability | Terraform | OpenTofu |
|---|---|---|
| State encryption (client-side, KMS/passphrase key providers) | — | 1.7 |
| `-exclude` planning option (inverse of `-target`) | — | 1.9 |
| `for_each` on `provider` blocks (multi-region without alias sprawl) | — | 1.9 |
| Variables/locals in `module` `source`/`version` | 1.15 | 1.8 |
| Module packages + provider mirrors from OCI registries | — | 1.10 |
| Ephemeral values / write-only attributes | 1.10 / 1.11 | 1.11 |
| `enabled` meta-argument (zero-or-one without `count`) | — | 1.11 |
| Test mocking (`mock_provider`, `override_*`) | 1.7 | 1.8 |
| List resources + `terraform query` (query existing infra, generate config) | 1.14 | — |
| Provider-defined actions (`before_destroy`/`after_destroy` triggers) | 1.14 (+1.16 `on_failure`) | — |
| Stacks CLI (multi-deployment orchestration) | 1.13 | — |

*(Compiled from the two CHANGELOGs; "—" means "not present in the changelogs read", not
"impossible". The vendor docs sites are blocked from this environment.)*

The honest interview answer: for a platform team the decision is rarely technical
day-one — it is about licence exposure if you resell, whether you need state encryption
without a KMS-encrypted bucket, and whether you depend on HCP Terraform's managed
control plane. Both are drop-in for ordinary usage; state files are compatible in the
direction you'd expect, and the migration cost is mostly CI and provider-registry
plumbing.

### 9. The GPU-fleet angle — what makes this different from generic IaC

IaC for a GPU platform is **capacity-and-quota-shaped**, and the differences are concrete:

- **Replacement is not free, because supply is not elastic.** In generic cloud IaC,
  `create_before_destroy` and rolling replacement are safe defaults: the new instance
  always exists. For frontier accelerators the fallback often does not exist — destroy a
  reserved block and you may not get it back this quarter. Practical encodings:
  `lifecycle { prevent_destroy = true }` on reservations, an OPA rule that denies any
  plan containing `["delete","create"]` on capacity resources, and a review rule that any
  plan touching GPU node pools needs a named approver. (The commitment-instrument details
  — prepaid Capacity Blocks, CUDs, take-or-pay neocloud contracts — live in the GPU cost
  module; the IaC point is that those objects are *fragile state*, not cattle.)
- **Quota is a precondition, not a resource.** A plan can be perfectly valid and still
  fail at apply with a quota error, after having created half the graph. Terraform 1.5's
  `check` blocks with scoped data sources are the tidy encoding: assert available
  accelerator quota ≥ what the plan wants, continuously, without blocking the lifecycle.
- **Draw the boundary at the driver.** Terraform owns the machine shape: node pools per
  GPU SKU, instance type, placement group / topology hints, the labels and taints that
  make the pool targetable (`nvidia.com/gpu.present=true`, a `gpu=true:NoSchedule` taint).
  GitOps owns what runs *on* it: GPU operator, driver DaemonSet, DCGM exporter, device
  plugin config. Teams that let Terraform install the GPU operator via the Helm provider
  end up with cluster-scoped Helm releases in a state file nobody wants to touch, and
  with Terraform and Argo fighting over the same objects.
- **Node counts belong to the autoscaler.** `ignore_changes = [scaling_config[0].
  desired_size]` (or the provider's equivalent) or every plan proposes to undo the
  autoscaler.
- **On bare metal, Terraform's role shrinks.** When you own the "cloud API" — Ironic,
  Cluster API, Metal³ — the declarative reconciliation loop is a controller's, not
  Terraform's. Terraform keeps the things around it: DNS records, DHCP/IPAM ranges,
  switch and BMC configuration, the bootstrap bucket. Say where you draw that line and
  why; it's a strong signal for a GPU-platform role.
- **Multi-tenant cost attribution starts in IaC.** Mandatory tag/label policy at plan
  time is what makes the FinOps view possible later. It is much cheaper to deny an
  untagged GPU node group than to reconstruct ownership from a $400k invoice.

## Self-quiz

Answer out loud before reading on; each answer is what a good response contains.

**1. When do you split Terraform state, and what is the boundary rule?**
Split on **blast radius + lifecycle + ownership**: a unit that is planned, approved, and
destroyed together, owned by one team, changing at one cadence. Typical layering:
org/landing zone → per-env-per-region platform → per-service. Not per-resource (you lose
the dependency graph and multiply cross-stack reads), not org-wide (every plan pays a
full refresh and every apply serialises behind one lock). The tell that you split wrong:
`-target` appears in normal runbooks, or a routine service change requires a plan over
the VPC.

**2. `count` vs `for_each` — why does `for_each` avoid destroy-and-recreate churn?**
Because instance identity in state is the address. `count` addresses by ordinal
(`res.name[2]`); removing an earlier element shifts every later element's index, and
Terraform sees `res.name[2]` now describing a different object — so it plans a
replacement. `for_each` addresses by map key (`res.name["a100-1a"]`), which is stable
under insertion and deletion, so removing one entry touches exactly that entry. Use
`count` only for zero-or-one toggles. On GPU pools this is the difference between a
one-line change and a plan that destroys machines you cannot re-source.

**3. How do you roll a provider or module major version across 40 stacks without a big
bang?**
Version constraints per stack, not a global pin, so stacks can move independently. Land
the new module version in the registry; bump one low-risk non-prod stack; read the plan
for unexpected replacements (major provider bumps commonly change defaults, which shows
up as in-place updates or forced replacement); then bump by ring — dev → stage → one prod
cell → the rest — with the plan diffed at each ring. `terraform init -upgrade` plus
`terraform providers lock -platform=…` to refresh `.terraform.lock.hcl` for every
platform CI and laptops use. Keep a `moved`-block-only release available for address
changes so consumers get re-keying rather than replacement. Expect the long tail to be
stacks with hand-edited state, and budget for them explicitly.

**4. What does policy-as-code catch that code review misses, and where do you enforce it?**
Review examines a diff; policy examines the **planned change graph**, including
resources created indirectly by nested modules and changes made outside review hours.
Enforce at two points: **plan time** (`conftest`/OPA or Sentinel over `terraform show
-json`) to block a merge with a full explanation of which resource violated what, and
**admission time** in the cluster for anything Terraform doesn't manage. Plan-time alone
misses `kubectl apply`; admission-time alone tells you at 3 a.m. what you could have
known at PR time.

**5. Someone applied from a laptop, the process died, and the state is locked and stale.
Walk the recovery.**
Get the lock ID from the error and confirm no apply is genuinely in flight (check the
cloud provider's recent API activity, not just the person's memory), then
`terraform force-unlock <ID>`. Snapshot with `terraform state pull > backup.tfstate`
before anything else. Run `terraform plan -refresh-only` to see what reality contains
that state does not. Adopt orphans with `import` blocks (`plan -generate-config-out` to
draft the HCL), drop ghosts with `state rm`. Only then run a normal plan, and expect it
to be boring. Follow-up work is the actual fix: remote state with locking, apply only
from CI with OIDC-federated short-lived credentials, and no human IAM principal with
apply rights on prod.

**6. Your plan says it will replace a reserved GPU capacity block. What do you do?**
Stop — replacement of scarce capacity is a supply decision, not a config decision. Find
the forcing attribute in the plan output (`# forces replacement`), and check whether it
can be avoided: `ignore_changes`, a provider upgrade that changed a default, or a
`moved` block if it is really an address change. If the change is genuinely required,
sequence it as create-then-migrate with the old reservation held until the new one is
confirmed live, never destroy-then-create. Then encode the lesson: `prevent_destroy` on
the resource and an OPA rule denying `["delete","create"]` on capacity types, so the next
person hits the guardrail instead of the incident.

## Refresh only if

- **OpenTofu vs Terraform (post-BSL).** If your POV predates the fork's maturity, note
  the divergence table above: OpenTofu leads on state encryption (1.7), `-exclude` (1.9),
  provider `for_each` (1.9) and OCI-registry distribution (1.10); Terraform leads on
  query/list resources (1.14), actions (1.14) and Stacks (1.13). Have one line on licence
  exposure (BUSL, four-year change date to MPL, no competing hosted offering) and one on
  migration cost.
- **Stacks / multi-deployment orchestration.** If your model is still `terraform
  workspace`, the orchestration layer above the state file has moved: Terraform exposed a
  `terraform stacks` CLI in 1.13, and Spacelift/Env0 solve the same problem — many
  deployments of one component set, with dependencies and ordering between them, without
  a hand-rolled wrapper repo of `for` loops in Makefiles.
- **The 2024–2026 language additions** if you last read the changelog at 1.5: ephemeral
  values and write-only attributes (1.10/1.11) for secrets, `removed` blocks (1.7),
  test mocking (1.7), S3 native locking (1.10, GA 1.11), variables in module sources
  (1.15), and `terraform query` (1.14).

## Recall card

Say these from memory; if one is fuzzy, reread the section in brackets.

| Fact | Value |
|---|---|
| Default apply parallelism | 10 [§1] |
| `plan -detailed-exitcode` | 0 no changes · 1 error · 2 changes [§1] |
| Plan-file contents | full proposed change + variable values → secret [§1] |
| `~> 5.0` vs `~> 5.31.0` | `<6.0.0` vs `<5.32.0` [§2] |
| State format version (1.x) | 4; `serial` + `lineage` guard concurrent/foreign writes [§3] |
| S3 native locking | TF 1.10, GA 1.11 `use_lockfile`; DynamoDB args deprecated; tofu 1.10 [§3] |
| State-splitting rule | blast radius × lifecycle × ownership [§3] |
| `moved` / `import` / `removed` blocks | TF 1.1 / 1.5 / 1.7 [§4] |
| Plan-JSON policy input | `resource_changes[].change.actions` = create/update/delete/`["delete","create"]` [§6] |
| `terraform test` | GA 1.6; mocks 1.7; `-junit-xml` GA 1.11 [§6] |
| OpenTofu state encryption | 1.7, AES-GCM, key providers: PBKDF2 / AWS KMS / GCP KMS / OpenBao [§7] |
| Terraform licence | BUSL 1.1 from 1.6.0; change date +4 years → MPL 2.0; licensor IBM [§8] |
| GPU-specific guardrail | deny `["delete","create"]` on capacity reservations; `prevent_destroy` [§9] |
