---
area: "Cloud & multi-cloud architecture"
kind: refresh
status: not-refreshed      # not-refreshed | refreshed
---

# 🔁 Cloud & multi-cloud architecture — interview refresh

> AWS depth plus GCP/Azure breadth; landing zones, identity, GPU offerings per cloud.
>
> You know this. Goal here is fast recall + crisp interview framing, not study.

Three sentences carry this whole area. **Accounts (projects, subscriptions) are the
blast-radius, quota and billing boundary — use them liberally.** **Identity, not the
network, is the perimeter, and the mechanism is a short-lived token exchanged for cloud
credentials.** **Data has gravity, and the gravity is priced per gigabyte in each
direction.** Everything else is detail hung off those.

Sourcing note: the mechanism-level facts below (projected ServiceAccount tokens, IRSA,
Azure workload identity) were verified against upstream source on GitHub —
`kubernetes/website` (`configure-service-account.md`), `kubernetes/api`
(`core/v1/types.go`), `aws/amazon-eks-pod-identity-webhook` (`README.md`), and
`Azure/azure-workload-identity` (`pkg/webhook/consts.go`), read 2026-08-18. AWS, GCP and
Azure documentation and pricing sites are egress-blocked from this environment and were
**not** relied on; per-GB prices and per-SKU specs are therefore given as a snapshot or as
"go check", never as invented precision.

## Talking points to have ready

### 1. Landing zone and organisation design

```
  Organisation (AWS Orgs / GCP org+folders / Azure management groups)
   │
   ├── Security OU
   │     ├── log-archive account      ← immutable, write-only from everywhere
   │     └── security-tooling account ← detection, scanning, break-glass audit
   ├── Infrastructure OU
   │     ├── network account          ← the hub: TGW/Cloud Router/vWAN, egress, DNS
   │     └── shared-services account  ← registries, CI runners, artifact stores
   ├── Workloads OU
   │     ├── prod-<team>-<region>     ← one account per team × environment
   │     └── nonprod-<team>
   └── Sandbox OU                     ← permissive, low budget, auto-nuked

  Guardrails apply downward:  SCPs / Org Policies / Azure Policy
  Identity comes sideways:    IdP ──SAML/OIDC──▶ SSO ──▶ permission sets ──▶ roles
  Telemetry flows inward:     every account ──▶ log-archive (append-only)
```

The reasoning to say out loud:

- **An account is the only hard boundary you get for free.** It bounds IAM (a policy
  cannot reference a principal in another account without an explicit trust), it bounds
  most service quotas (which is why a noisy team cannot eat your API rate limits), and it
  is the natural billing dimension. Namespaces and tags are conventions; account
  boundaries are enforced by the provider.
- **Guardrails are subtractive, not additive.** An AWS Service Control Policy never
  grants anything — effective permissions are the intersection of the SCP and the
  identity-based policy. This is the point people get wrong in interviews: attaching an
  SCP that "allows S3" grants nothing; it merely stops denying it. Use SCPs for
  absolutes — deny disabling CloudTrail, deny regions you do not operate in, deny
  deleting the log-archive bucket — and IAM policies for what people can do. GCP
  organisation policies and Azure Policy play the same role with different mechanics
  (constraint-based, and audit/deny/modify effects respectively).
- **Landing zone = the account factory plus the five things every account gets on day
  zero**: federated identity, centralised logging destination, network attachment,
  guardrail policies, and a budget with an alarm. If a team can create an account without
  those five, you don't have a landing zone, you have a spreadsheet.
- **Regions are a compliance and latency decision; AZs are a failure-domain decision.**
  Multi-region is expensive and mostly bought for data residency or genuine
  region-loss survival. Multi-AZ is the default for anything stateful — and it has a bill
  attached (§3).

### 2. Identity is the real perimeter — the mechanism, not the slogan

Every "how does a pod get cloud credentials without a static key" answer is the same
five-step token exchange. Draw it once and adapt the labels per cloud:

```
  ① Pod starts with serviceAccountName: trainer
        │
  ② kubelet requests a projected SA token from the API server
        │    audience: sts.amazonaws.com (or api://AzureADTokenExchange, …)
        │    expirationSeconds: default 3600, min 600, rotated at 80% TTL or 24 h
        ▼
     JWT signed by the cluster's SA issuer key
        claims: iss=<cluster OIDC issuer>  sub=system:serviceaccount:ml/trainer
                aud=<audience>  exp=…  kubernetes.io.pod/node bindings
        │
  ③ mounted at a file path; the SDK reads the file
        │
  ④ SDK calls the cloud STS: "here is a JWT, give me credentials for this role"
        │      AWS: sts:AssumeRoleWithWebIdentity
        ▼
  ⑤ Cloud validates the JWT against the cluster's *public* OIDC discovery document
        (<issuer>/.well-known/openid-configuration → JWKS), checks aud and sub
        against the role's trust policy, and returns short-lived credentials
        │
        ▼
     Pod calls S3/GCS/Blob with credentials that expire in ~1 hour
```

The key insight, and the sentence that wins the question: **the cluster becomes an OIDC
identity provider, and the cloud trusts it as a federated IdP.** Kubernetes serves the
provider configuration at `{service-account-issuer}/.well-known/openid-configuration`
(issuer discovery has been stable since v1.21), so the cloud can verify the token's
signature without ever talking to your API server. There is no shared secret anywhere in
the flow.

The Kubernetes-side numbers, from `kubernetes/api` `core/v1/types.go`
(`ServiceAccountTokenProjection`): `expirationSeconds` **defaults to 1 hour and must be at
least 10 minutes**; the kubelet proactively rotates when the token passes **80% of its
TTL or is older than 24 hours**; the application is responsible for re-reading the file
(reloading every few minutes is the usual approach). Tokens are bound to the pod and the
ServiceAccount — delete either and the token stops working, which is what makes them
better than the old, long-lived Secret-based tokens.

Per cloud, verified from each project's own source:

| | EKS (IRSA) | GKE (Workload Identity Federation) | AKS (Azure Workload Identity) |
|---|---|---|---|
| Cluster's role | OIDC provider registered in IAM | Cluster is bound to the project's fixed workload identity pool `PROJECT_ID.svc.id.goog` | Cluster OIDC issuer registered as a federated credential on the app/managed identity |
| Binding is declared | On the **role's trust policy**: `sts:AssumeRoleWithWebIdentity` with a condition on `<issuer>:sub == system:serviceaccount:<ns>:<sa>` | On the IAM allow-policy of the target resource/service account, referencing the KSA principal | As a **federated identity credential** on the Entra app: issuer + subject `system:serviceaccount:<ns>:<sa>` |
| Kubernetes side | SA annotation `eks.amazonaws.com/role-arn`; optional `eks.amazonaws.com/audience` (default `sts.amazonaws.com`) and `eks.amazonaws.com/token-expiration` (default **86400**) | KSA referenced directly by the IAM principal; pod reaches the **GKE metadata server** rather than reading env vars | SA labelled `azure.workload.identity/use: "true"`, annotated `azure.workload.identity/client-id` (+ optional `tenant-id`) |
| Injection | Mutating webhook adds `AWS_ROLE_ARN`, `AWS_WEB_IDENTITY_TOKEN_FILE`, `AWS_REGION`, and the projected-token volume | Handled by the metadata server; no per-pod env injection | Mutating webhook adds `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_FEDERATED_TOKEN_FILE`, `AZURE_AUTHORITY_HOST`; default audience `api://AzureADTokenExchange`; token expiry min 3600 / max 86400, default 3600 |

*(EKS also offers **Pod Identity**, an agent-based alternative that removes per-cluster
OIDC provider setup and uses an association API instead of role-trust-policy edits — know
it exists and that the trade is "less IAM plumbing, more EKS-specific". GKE's exact IAM
principal string is not reproduced here because the GCP docs are unreachable from this
environment; check the console rather than trusting a remembered format.)*

```yaml
# The EKS shape, end to end
apiVersion: v1
kind: ServiceAccount
metadata:
  name: trainer
  namespace: ml
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::111122223333:role/checkpoint-writer
```
```json
// The trust policy on checkpoint-writer — this is where the binding actually lives
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::111122223333:oidc-provider/oidc.us-east-1.eks.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": { "StringEquals": {
      "oidc.us-east-1.eks.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud": "sts.amazonaws.com",
      "oidc.us-east-1.eks.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:ml:trainer"
    }}
  }]
}
```

**The failure mode to name:** a trust policy that conditions only on the issuer, or uses
`StringLike` with `system:serviceaccount:*:*`. Then *any* ServiceAccount in the cluster
can assume the role, and your "least privilege per workload" is actually "cluster-wide
credential". Always pin `:sub` to the exact namespace and name, and pin `:aud`.

For humans, the equivalent is federated SSO with permission sets mapped to groups, plus
cross-account role assumption with an external ID for third parties. Long-lived access
keys should exist only where a legacy system truly cannot federate — and then they belong
in a secrets manager with rotation, and they show up on a dashboard of shame.

### 3. Data gravity and the cost of a hop

The cost model most engineers carry is wrong in a specific way: they think of egress as
"traffic leaving the cloud". The expensive traffic is usually **internal**.

- **Cross-AZ traffic is metered per GB in each direction on AWS** — the sender's account
  and the receiver's account are both billed, which is why the effective rate looks like
  double the quoted per-direction figure. Replicated databases, Kafka with
  `min.insync.replicas` across AZs, and chatty service meshes generate this continuously
  and invisibly.
- **Load-balancer defaults differ and it matters:** ALB has cross-zone load balancing
  **enabled** by default; NLB has it **disabled** by default, per target group. Migrating
  between them without checking silently changes both your bill and your load
  distribution.
- **Internet egress list price on AWS starts around $0.09/GB** in the first tier, tapering
  at volume, with a monthly free allowance (2026 snapshot from this course's GPU cost
  module — re-verify before quoting in a contract). Several GPU neoclouds — Lambda,
  Nebius, Crusoe, SF Compute among them — have offered **free egress**, while CoreWeave,
  RunPod, Vast.ai and all three hyperscalers meter it. That difference alone can decide
  where a data-heavy inference service lives.

Do the arithmetic parametrically so it survives price changes:

```
  monthly_cross_AZ_$ = bytes_GB × rate_$/GB × directions_billed
  monthly_egress_$   = bytes_GB × egress_rate_$/GB

  Worked, replication-heavy service:
    40 TB/month = 40,960 GB replicated cross-AZ, billed both ways
      → 40,960 × rate × 2         (at a $0.01/GB-per-direction list rate: ≈ $819/mo)
    Same 40 TB served to the internet at ~$0.09/GB
      → 40,960 × 0.09             ≈ $3,686/mo
    Same 40 TB inside one AZ
      → $0
```

The architectural conclusions follow directly: colocate chatty replicas; use
topology-aware routing (`internalTrafficPolicy: Local`, topology hints) so a request is
served in the zone it landed in; keep the object store in the same region as the compute
that reads it; and **measure per-hop bytes (flow logs, cost allocation tags) before
redesigning anything**, because the meter that fired is often not the one you assumed.

For training specifically, the cost is not only dollars. A collective that spans AZs puts
the inter-AZ RTT (~0.5–1 ms per hop) on the critical path of *every step*, thousands of
times per run. You pay twice: on the transfer bill and again in extended GPU-hours.
That is why GPU training clusters are single-AZ, in one placement group / compact
placement domain, and why "spread across three AZs for resilience" is a
request-serving instinct that is actively wrong for a synchronous training job.

### 4. Commitments, quota and capacity

Generic FinOps says: cover the baseline with commitments (Reserved Instances, Savings
Plans, CUDs, Azure Reservations), burst on-demand, and use spot for interruptible work.
That is knob 1 — **price**.

GPU procurement has a second knob that generic cloud does not: **availability**. When
frontier accelerators are allocation-constrained, the alternative to a commitment is not
"pay more", it is "do not run the workload". AWS productised exactly this with **EC2
Capacity Blocks for ML**: you book a future window (1–14 days, or multiples of 7 up to
182 days, starting up to 8 weeks ahead, up to 64 instances per block and 256 across
blocks), pay **in full up front**, and get no discount for it — it is a hotel
reservation, not a coupon. *(Figures are as recorded in this course's GPU cost module,
which took them from AWS documentation via search extracts because `docs.aws.amazon.com`
is egress-blocked here; re-verify before using them commercially.)*

The other capacity mechanic to have ready is **quota**. Accelerator quotas are per
account, per region, often per instance family, and increases take human time. A design
review that ends "and we'll scale to 512 GPUs in us-east-1" without a quota check has not
finished. Treat quota as lead-time-bearing inventory: track current limit vs current use
per family per region, request ahead of the plan, and encode the check in CI (see the IaC
refresh's use of `check` blocks).

### 5. Multi-cloud, honestly

Sort the claim into one of four postures before answering — most interview "multi-cloud"
questions are really asking which one you mean:

| Posture | What it means | Real cost | When it's right |
|---|---|---|---|
| **Portability insurance** | Avoid designs that cannot be moved; run on one cloud today | Low — mostly discipline | Almost always defensible |
| **Best-of-breed** | Workload A here, workload B there, deliberately | Medium — two operational surfaces, egress between them | When a specific service is genuinely differentiated |
| **Capacity/price arbitrage** | Follow GPU supply and $/GPU-hr across hyperscalers and neoclouds | Medium-high — data movement, image/weights distribution, per-provider integration | **The legitimate GPU driver** — supply is the constraint, not preference |
| **Active-active** | Same workload live on two clouds, failover between them | Very high — lowest-common-denominator services, doubled runbooks, cross-cloud data consistency, egress on every sync | Rarely; usually regulatory or a contractual demand |

**The abstraction tax, itemised** (this is what makes the answer sound lived-in rather
than theoretical): identity models do not map (IAM roles vs GCP service accounts vs Entra
principals — the trust semantics differ, not just the syntax); network primitives do not
map (security groups vs firewall rules vs NSGs, and VPC peering/transit models diverge);
quota systems are per-cloud and per-region; managed services with the same name behave
differently under failure; storage consistency and durability semantics differ; and every
byte you move between them is billed at internet-egress rates unless your provider has
waived egress. Kubernetes portably abstracts *the workload API*, not the platform beneath
it.

For a GPU platform, the honest position: **arbitrage is a real driver, active-active is
usually not.** What makes arbitrage tractable is that the portable surface is small and
well-defined — container images, a scheduler API, checkpoint objects in object storage,
and a fabric that supports RDMA collectives. What makes it painful is data: moving a
multi-terabyte dataset or a checkpoint stream across providers costs both money and hours,
which is why the pattern is "run whole jobs where the capacity is", not "spread one job
across clouds".

### 6. GPU offerings per cloud — what actually differentiates them

Do not try to memorise SKU tables; they change every quarter and an interviewer who works
there will know them better than you. Memorise the **questions**, which do not change:

1. **What is the accelerator and how many per node?** (8 per node is the near-universal
   dense-node shape for training-class parts; AWS's `p5.48xlarge` is 8×H100 and
   `p6-b200.48xlarge` is 8×B200, to anchor the naming convention.)
2. **What is the scale-out fabric, and is it in the price?** InfiniBand, RoCE, or a
   provider-specific fabric (AWS EFA, GCP's GPUDirect-over-TCP/RDMA variants, Azure's
   InfiniBand-equipped ND series). Ask: per-node aggregate bandwidth, whether the fabric
   is rail-optimised, and whether you are guaranteed to land in one fabric domain.
3. **How is placement controlled?** Cluster placement groups, compact placement policies,
   capacity blocks/reservations tied to a specific fabric block. A 64-node job scattered
   across the datacentre is a slow job regardless of the GPU.
4. **What is the driver/device-plugin story on the managed Kubernetes?** Who installs the
   driver (node image vs GPU operator), how the device plugin is delivered, whether MIG
   and time-slicing are supported and configurable, and how node upgrades interact with
   driver versions. This is the most common source of "same YAML, different behaviour"
   between EKS, GKE and AKS.
5. **What does the contract actually promise?** Hyperscalers: published SLAs, mature
   support, high $/GPU-hr. Neoclouds: lower $/GPU-hr, sometimes free egress, but thinner
   SLAs, variable support depth, and take-or-pay minimums. "Is InfiniBand included?" and
   "what is the remedy when a node is a lemon?" are the two questions that separate a
   procurement conversation from a price comparison.
6. **What is the storage path for checkpoints and datasets?** A parallel filesystem, a
   managed one, or object storage plus a cache — and at what throughput per node. On a
   large run, checkpoint write bandwidth is a first-class capacity requirement, not a
   detail.

The GPU-platform-specific tell in an interview is that you cost a GPU offer as
**$/useful-GPU-hour**, not $/GPU-hour: price ÷ (availability × goodput). A $2.50/GPU-hr
node with a mediocre fabric and 12% job-failure overhead can be more expensive than a
$4.90/GPU-hr node that finishes the run.

## Self-quiz

**1. What does a cross-AZ hop actually cost, and how does it change data-tier placement?**
Two costs. Money: cross-AZ transfer is metered per GB, and on AWS it is billed on both the
sending and receiving side, so a replicated data tier pays continuously and in both
directions — at replication volumes that is a four-figure monthly line for a mid-sized
service. Latency: roughly 0.5–1 ms per hop, which is irrelevant for a single request and
decisive for anything synchronous and chatty. Placement conclusion: keep the chatty
replication inside an AZ and buy resilience deliberately (quorum across three AZs only
where the failure budget justifies it, computed in the same dollars as the transfer cost);
serve reads locally with topology-aware routing; check LB cross-zone defaults (ALB on,
NLB off) because they silently move traffic across the boundary. For synchronous training
jobs, AZ-spreading is not a resilience win at all — it puts inter-AZ latency on every
collective.

**2. Workload identity vs static access keys — the exact mechanism.**
The kubelet mounts a projected ServiceAccount token: a JWT signed by the cluster's SA
issuer key, with `sub=system:serviceaccount:<ns>:<sa>`, an explicit `aud`, an expiry
(default 1 h, minimum 10 min), and bindings to the pod and node. The cluster publishes an
OIDC discovery document and JWKS, so the cloud can validate the signature offline. The SDK
reads the token file and calls the cloud's STS — on AWS
`sts:AssumeRoleWithWebIdentity` — and the role's trust policy checks the issuer, the
`aud`, and the exact `sub` before returning credentials valid for about an hour. The
kubelet rotates the token at 80% of TTL or 24 hours; the app just re-reads the file. No
static secret exists at any point, revocation is deleting the ServiceAccount or the pod,
and the audit trail is the STS call. The failure mode: a trust policy that wildcards the
subject, which converts per-workload identity back into a cluster-wide credential.

**3. When is multi-cloud worth the abstraction tax, and when is it a trap?**
Worth it when a *specific, quantified* constraint requires it: GPU capacity you cannot get
from one provider, a regulatory demand for provider diversity, a customer contract, or one
genuinely differentiated managed service. It is a trap when the driver is "avoid lock-in"
in the abstract — the usual outcome is that you pay the tax (lowest-common-denominator
services, two identity models, two network models, doubled runbooks, egress on every
cross-cloud byte) and still cannot fail over, because the data layer was never actually
portable. The mature position: **portability discipline always, active-active almost
never, and arbitrage when supply economics demand it.** For GPU platforms the third case
is real, and the reason it works is that the portable surface — images, scheduler API,
checkpoints in object storage — is small.

**4. Why might a GPU workload pick a neocloud over a hyperscaler, and what do you give up?**
You pick it for supply and price: capacity available on a shorter horizon, materially
lower $/GPU-hr, InfiniBand often included rather than an add-on, and in several cases no
egress metering. You give up SLA depth and remedies, breadth of managed services (you will
run more of the platform yourself — identity, networking, storage, observability),
integration with an existing landing zone and its compliance evidence, and often support
response times. There is usually a take-or-pay minimum, so the commitment risk moves onto
you. The decision rule is to compare **$/useful-GPU-hour including the self-run platform
burden**, not the headline rate — and to check the two contract questions: is the fabric
included and specified, and what happens when nodes are unhealthy.

**5. A team asks for 512 H100s in one region next month. What do you check before saying
yes?**
Quota (current limit vs use for that family in that region, and the lead time to raise
it); capacity mechanism (on-demand won't hold 64 nodes — is this a reservation, a capacity
block, or a committed contract, and what is the booking horizon?); placement (one fabric
domain / placement group, or the job runs slow); network fabric (per-node bandwidth,
rail-optimised or not); storage (checkpoint write throughput at that scale, and dataset
locality — data in another region means both an egress bill and a slow first epoch);
account/blast-radius boundary (does this get its own account or share quota with
someone?); and the exit (what happens to the reservation when the project ends). Then the
economics: commitment shape, and the $/useful-GPU-hour number that includes expected
interruption overhead.

## Refresh only if

- **Per-cloud GPU capacity mechanics**, if your procurement model predates the 2024–2026
  capacity crunch: prepaid, non-discounted booking products (AWS Capacity Blocks and
  peers), reservation/DWS-style scheduled capacity on GCP, and neocloud take-or-pay
  contracts. The framing that matters is availability as a distinct benefit from discount —
  the FOCUS billing spec models them as different categories for exactly this reason.
- **The managed-Kubernetes GPU integrations** across EKS/GKE/AKS if you have only run one:
  who installs the driver, how the device plugin arrives, MIG and time-slicing support,
  and node-upgrade interaction with driver versions.
- **EKS Pod Identity** if your IRSA mental model is the only one you have — it removes the
  per-cluster OIDC provider and role-trust-policy editing in favour of an association API,
  and it is increasingly the default answer on new EKS clusters.

## Recall card

Cover the right column and say each value out loud; if one is fuzzy, reread the section
in brackets.

| Fact | Value |
|---|---|
| SCP semantics | Subtractive only; effective = SCP ∩ identity policy [§1] |
| Account boundary buys | IAM isolation + separate quotas + billing dimension [§1] |
| Projected SA token | default TTL **1 h**, min **10 min**, rotate at **80% TTL or 24 h** [§2] |
| Token claims that matter | `iss` (cluster issuer), `sub=system:serviceaccount:ns:sa`, `aud` [§2] |
| OIDC discovery path | `<issuer>/.well-known/openid-configuration` (stable since k8s 1.21) [§2] |
| IRSA wiring | SA annotation `eks.amazonaws.com/role-arn`; `AWS_ROLE_ARN` + `AWS_WEB_IDENTITY_TOKEN_FILE` injected; default token expiry 86400 s [§2] |
| Azure WI wiring | label `azure.workload.identity/use`, env `AZURE_FEDERATED_TOKEN_FILE`, audience `api://AzureADTokenExchange` [§2] |
| Cross-AZ billing | per GB, **each direction** on AWS [§3] |
| LB cross-zone defaults | ALB **on**, NLB **off** (per target group) [§3] |
| Internet egress list price | ≈ $0.09/GB first tier (2026 snapshot, re-verify) [§3] |
| Capacity Blocks shape | 1–14 days or multiples of 7 up to 182; ≤8 weeks ahead; ≤64/block, ≤256 total; prepaid [§4] |
| Multi-cloud postures | insurance · best-of-breed · arbitrage · active-active [§5] |
| GPU offer comparison unit | **$/useful-GPU-hour** = price ÷ (availability × goodput) [§6] |
