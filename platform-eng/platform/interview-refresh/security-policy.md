---
area: "Security & policy"
kind: refresh
status: not-refreshed      # not-refreshed | refreshed
---

# 🔁 Security & policy — interview refresh

> RBAC, admission control (OPA/Kyverno), secrets, supply chain, runtime security.
>
> You know this. Goal here is fast recall + crisp interview framing, not study.

Cluster security questions are really one question asked five ways: **what stands between
an authenticated request and the object it wants to touch, and where would you insert a
control?** Answer with the request pipeline, then hang RBAC, admission, secrets and
runtime isolation off the right stage of it. The GPU-specific twist — that device
isolation is weaker than CPU isolation and only MIG gives a hardware boundary — is what
separates this from a generic Kubernetes security answer.

Version and field-level claims below were verified on 2026-08-18 against
`kubernetes/website` (`rbac.md`, `pod-security-standards.md`, `pod-security-admission.md`,
`validating-admission-policy.md`, `encrypt-data.md`, `configure-service-account.md`),
`kubernetes/kubernetes` and `k8s.io/apiserver` feature-gate tables, `kubernetes/api`
(`core/v1/types.go`), and `NVIDIA/k8s-device-plugin` (`README.md`). `kubernetes.io`,
`kyverno.io` and vendor sites are egress-blocked here and were **not** read; the repo
content behind them was.

## Talking points to have ready

### 1. The request pipeline — the map everything else hangs on

```
  kubectl / SDK / controller
        │  TLS
        ▼
  ┌───────────────┐
  │ Authentication│  client certs · bearer tokens (SA JWT, OIDC) · webhook
  └───────┬───────┘  → produces a username + groups, or 401
          ▼
  ┌───────────────┐
  │ Authorization │  Node · RBAC · ABAC · Webhook, evaluated in order;
  └───────┬───────┘  ANY authorizer that ALLOWS ends it → 403 if none do
          ▼
  ┌────────────────────────────────────────────────────────────┐
  │ Admission — MUTATING first                                 │
  │   MutatingAdmissionWebhook · MutatingAdmissionPolicy (CEL) │
  │   sidecar injection · defaulting · image digest pinning    │
  └───────┬────────────────────────────────────────────────────┘
          ▼   object schema validation
  ┌────────────────────────────────────────────────────────────┐
  │ Admission — VALIDATING                                     │
  │   PodSecurity (built-in) · ValidatingAdmissionPolicy (CEL) │
  │   ValidatingAdmissionWebhook (Kyverno / Gatekeeper)        │
  │   ResourceQuota                                            │
  └───────┬────────────────────────────────────────────────────┘
          ▼
        etcd  ──(encryption at rest: KMS v2 / aescbc / identity)──▶ disk
```

Two consequences to say out loud. **Mutating runs before validating**, which is why an
image-digest-pinning mutation is verified by the validating stage in the same request —
and why a mutating webhook that adds a privileged sidecar can defeat a naive policy that
only reads the submitted object. And **authorization is additive, RBAC has no deny rule**:
you cannot subtract a permission with another Role, only avoid granting it. Denial as a
first-class concept lives in admission, not RBAC.

### 2. RBAC done right, and the escalation paths

The model: `Role`/`ClusterRole` list `apiGroups × resources × verbs` (plus
`resourceNames` for named objects and non-resource URLs); `RoleBinding`/
`ClusterRoleBinding` attach them to users, groups or ServiceAccounts. A `ClusterRole`
bound with a `RoleBinding` applies **only inside that namespace** — the most useful trick
in the model, because you define the permission set once and bind it per team namespace.

**The built-in guardrails** (verified from the RBAC docs) are the answer to "how do you
stop an admin from writing themselves a bigger role":

- **Privilege escalation prevention:** you may only create or update a Role/ClusterRole if
  you already hold every permission it contains, at the same scope — *or* you have the
  `escalate` verb on `roles`/`clusterroles` in `rbac.authorization.k8s.io`.
- **Binding restriction:** you may only create a binding to a role whose permissions you
  already hold — *or* you have the `bind` verb on that specific role.

So `escalate` and `bind` are the two verbs that quietly convert "can edit RBAC" into "can
become cluster-admin". The other indirect paths to name:

| Path | Mechanism |
|---|---|
| `create pods` in a namespace | You choose the ServiceAccount, mount any Secret in that namespace, set `hostPath`/`hostPID`/privileged if policy allows — pod creation is close to namespace-admin |
| `create pods/exec`, `pods/attach`, `pods/portforward` | Code execution inside an existing workload, with that workload's identity |
| `impersonate` | Act as any user or group, including `system:masters` if not constrained |
| `get`/`list` on `secrets` | Reads every credential in scope, including SA tokens where legacy secrets still exist |
| `create` on `serviceaccounts/token` | Mint a token for another ServiceAccount |
| Editing a workload controller (Deployment, DaemonSet) | Same as pod creation, one hop later, and DaemonSets land on every node |
| `patch`/`update` on nodes, or CSR approval rights | Node identity and kubelet trust; the Node authorizer scopes a kubelet to its own node's objects, and that boundary is worth protecting |

Practical hygiene: no wildcard verbs or resources in anything a human can bind; prefer
namespace-scoped bindings; audit with `kubectl auth can-i --list --as=system:serviceaccount:ns:sa`
and review `ClusterRoleBinding`s to `cluster-admin` as a standing control; give controllers
their own ServiceAccount with exactly the verbs their reconcile loop calls.

### 3. Workload identity — short-lived, audience-bound, node-bound

The modern token is a **projected ServiceAccount token**, not a Secret. Mechanics
(verified in `kubernetes/api` `core/v1/types.go` and the ServiceAccount docs):

```yaml
volumes:
  - name: vault-token
    projected:
      sources:
        - serviceAccountToken:
            path: vault-token
            audience: vault          # who may accept this token
            expirationSeconds: 7200  # default 3600, minimum 600
```

- The kubelet requests it via the TokenRequest API and **rotates when the token passes 80%
  of its TTL or is older than 24 hours**; the app must re-read the file.
- The JWT carries `sub=system:serviceaccount:<ns>:<sa>`, an explicit `aud`, and
  `kubernetes.io` claims binding it to the Pod and Node — deleting the pod or the
  ServiceAccount invalidates it. Legacy Secret-based tokens never expire and are not bound
  to anything, which is why auto-generated SA Secrets went away.
- `ServiceAccountTokenNodeBinding` reached **GA in 1.33**, so tokens can be bound to a Node
  object as well — a stolen token stops working when the node goes away.
- Receiving services must *check the audience*. A token minted for `vault` should be
  rejected by anything that is not Vault; audience checking is what stops token replay
  across services.

### 4. Admission control is the policy chokepoint

Three enforcement engines, and you should know when to use which:

| | Pod Security Admission | ValidatingAdmissionPolicy / MutatingAdmissionPolicy (CEL) | Kyverno / OPA Gatekeeper (webhooks) |
|---|---|---|---|
| Where it runs | In-process, built in | **In-process**, in the apiserver | Out-of-process webhook |
| Language | Fixed profiles | CEL expressions | Kyverno YAML / Rego |
| Scope | Pod security fields only | Any resource, any field, with parameters and bindings | Any resource, plus mutation, generation, image verification, external data |
| Availability risk | None | None | A down webhook with `failurePolicy: Fail` breaks the cluster; with `Ignore` it silently stops enforcing |
| Status | Stable since **1.25** | ValidatingAdmissionPolicy stable **1.30**; MutatingAdmissionPolicy **GA in 1.36** (alpha 1.32, beta 1.34) | Mature ecosystems |

**Pod Security Standards** are three profiles: `privileged` (no restrictions), `baseline`
(blocks known escalations — no privileged containers, no host namespaces, no hostPath, no
hostPorts beyond the allowed set, restricted capabilities, controlled seccomp/AppArmor/
SELinux/sysctls), and `restricted` (baseline plus hardening: limited volume types, no
privilege escalation, `runAsNonRoot`, `seccompProfile.type` must be `RuntimeDefault` or
`Localhost`, and containers must **drop ALL capabilities** with only `NET_BIND_SERVICE`
addable back, per the v1.22+ rules in the standard).

Enforcement is per-namespace labels, three modes:

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.36   # pin, or "latest"
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**The gotcha to have ready:** `audit` and `warn` are applied to *workload resources*
(Deployments, Jobs), but **`enforce` is not — it applies only to the resulting Pod
objects**. A Deployment with a non-compliant pod template is accepted; the ReplicaSet then
fails to create pods, and the failure appears as events on the ReplicaSet, not as a
rejected `kubectl apply`. That is why you set `warn` alongside `enforce`: the developer
gets told at apply time. Pinning `enforce-version` matters too — `latest` means a cluster
upgrade can tighten your policy underneath you.

**ValidatingAdmissionPolicy** covers most of what people used to spin up a policy engine
for, with no webhook to keep alive:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "gpu-limits.example.com"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    - expression: >-
        object.spec.containers.all(c,
          !has(c.resources.limits) || !('nvidia.com/gpu' in c.resources.limits) ||
          int(c.resources.limits['nvidia.com/gpu']) <= 8)
      message: "a single container may not request more than 8 GPUs"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "gpu-limits-binding.example.com"
spec:
  policyName: "gpu-limits.example.com"
  validationActions: [Deny]        # Deny · Warn · Audit (Deny+Warn is disallowed)
  matchResources:
    namespaceSelector:
      matchLabels: {tenant: "true"}
```

Reach for **Kyverno** when you need mutation, resource generation, image-signature
verification, or policies that consult data outside the request; Kyverno's YAML-native
rules are faster to write and review for the common cases. Reach for **OPA/Gatekeeper and
Rego** when the logic is genuinely complex — set arithmetic, cross-object joins,
policies you also want to run outside Kubernetes (Rego over Terraform plans is the same
language, which is a real advantage for a platform team standardising on one policy
engine). Reach for **CEL policies** first when the rule is expressible on the object
itself, because there is no availability risk and no extra deployment.

**Say the failure mode:** `failurePolicy: Fail` on a webhook means your policy engine is
now a control-plane dependency — if it is down or unreachable, every matching create is
rejected, up to and including the pods that would restore it. `failurePolicy: Ignore`
avoids that and silently disables enforcement exactly when something is going wrong.
Mitigations: scope `matchConditions`/`namespaceSelector` narrowly (never match
`kube-system`), run the webhook HA with a PDB, set short timeouts, and monitor for
"webhook unavailable" as a security event, not just an availability one.

### 5. Secrets

- **Kubernetes Secrets are base64, not encryption.** Confidentiality at rest comes from
  the apiserver's `EncryptionConfiguration`: an ordered provider list per resource, where
  the **first provider encrypts** and all listed providers can decrypt. `identity` means
  plaintext — an `identity` first entry disables encryption for that resource, and a
  common mistake is leaving it there after a migration. Providers include `aescbc`,
  `aesgcm`, `secretbox` and `kms`. KMS v2 is the current path; the `KMSv1` feature gate
  was deprecated in 1.28 and defaults to false from 1.29.
- **Encryption is not retroactive.** After enabling it you must rewrite every existing
  object — the standard move is `kubectl get secrets --all-namespaces -o json | kubectl
  replace -f -`, which re-persists each Secret through the new provider.
- **Prefer not storing the secret at all.** External Secrets Operator or the Secrets Store
  CSI driver keep the source of truth in Vault/cloud secret manager; better still, use
  dynamic short-lived credentials (a Vault database role, or cloud STS via workload
  identity) so the credential's lifetime is measured in minutes.
- **Never** in env vars that get logged, never in images, never in Git — and if it lands
  in Git, rotation is the fix, not a force-push.

### 6. Supply chain (the short version — the CI/CD refresh has the long one)

Sign artifacts with cosign (keyless: OIDC → Fulcio short-lived cert → Rekor transparency
log), generate SBOMs, emit SLSA provenance (Build L3 = hardened, isolated builds with
signing material unreachable from build steps), and **verify at admission** with a policy
that also rewrites the tag to the verified digest. Pin CI actions by commit SHA, scope CI
tokens per job, federate to cloud roles with OIDC instead of static keys, and scan images
(Trivy/Grype) both in CI and continuously in the registry — a passing scan on Tuesday says
nothing about Thursday's CVE disclosure.

### 7. Runtime security and the isolation ladder

```
  weakest ────────────────────────────────────────────────────▶ strongest
  namespace + RBAC + NetworkPolicy + PSS-restricted
        │  shared kernel, shared node, shared devices
        ▼
  dedicated node pool (taints/tolerations, no co-tenancy)
        │  still a shared kernel per tenant, but no cross-tenant node
        ▼
  user namespaces (GA in 1.36) — container root ≠ host root
        │  large class of kernel escapes becomes non-fatal
        ▼
  sandboxed runtime via RuntimeClass (gVisor, Kata Containers)
        │  syscall interposition or a per-pod VM; real CPU/latency cost
        ▼
  separate cluster / separate account
```

Supporting controls to name: **NetworkPolicy default-deny per namespace** (NetworkPolicy
is additive with no deny rules, so isolation comes from selecting all pods and permitting
nothing, then adding allows):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: default-deny-all, namespace: tenant-a}
spec:
  podSelector: {}                 # every pod in the namespace
  policyTypes: [Ingress, Egress]  # with no rules below = deny both directions
```

…plus `seccompProfile: RuntimeDefault` (required by PSS restricted), AppArmor/SELinux
profiles, read-only root filesystems, and **runtime detection** — Falco or an eBPF-based
agent watching syscalls for the behaviours that indicate compromise (a shell spawned in a
container, an unexpected outbound connection, writes to `/etc`), because policy prevents
known-bad configuration while detection catches unknown-bad behaviour.

### 8. The GPU twist — isolation is weaker, and the reason is hardware

This is the part that makes the answer specific to a GPU platform:

| Sharing mode | Boundary | What a hostile or buggy co-tenant can do |
|---|---|---|
| **Exclusive device** (`nvidia.com/gpu: 1`) | Whole device to one container | Nothing to a neighbour on that device — there is none |
| **Time-slicing** | None. The plugin advertises `replicas ×` devices; CUDA interleaves contexts | Consume all framebuffer, starve others of compute, and **crash the device — all tenants sharing it fail together** (NVIDIA device-plugin README: "each workload has access to the GPU memory and runs in the same fault-domain as all the others") |
| **MPS** | Space partitioning with enforced memory and compute limits, via a control daemon | Limits are enforced, but it is still one device and one fault domain; not supported with MIG |
| **MIG** | **Hardware** partition — dedicated SMs, dedicated framebuffer slice, dedicated paths | The strongest boundary the GPU offers; the practical answer when tenants do not trust each other |

Other GPU-specific security surface worth naming:

- **Device nodes are the escape surface.** Containers get `/dev/nvidia*` handed in by the
  container runtime hook. A pod allowed `hostPath` onto `/dev`, or `privileged: true`,
  bypasses the device-plugin allocation entirely and can touch every GPU on the node —
  which is one more reason PSS `restricted` (no hostPath, no privileged, drop ALL
  capabilities) matters more here than on a CPU fleet.
- **Keep the container toolkit patched.** The GPU stack adds a privileged component
  (container runtime hook plus kernel driver) to every node; vulnerabilities in that layer
  have been published historically and are node-escape-class when they occur. Track and
  patch it on the same cadence as the kernel; do not cite a CVE number you have not
  verified.
- **Drivers are kernel modules.** The blast radius of a GPU driver bug is the node, not
  the container. That argues for uniform, pinned driver versions rolled per node pool, and
  for keeping the GPU operator's privileged DaemonSets under the same review as any other
  cluster-privileged component.
- **DRA changes the authorization surface.** With DRA, "which device do I get" becomes an
  API object (`ResourceClaim` against a `DeviceClass`), so it is RBAC-controllable in a
  way an integer extended resource never was. Note `DRAAdminAccess` — the gate that allows
  privileged/administrative access to devices — reached GA in 1.36 and should be granted
  to operations tooling only.
- **Data isolation follows the job, not the pod.** Shared checkpoint stores and dataset
  mounts are where multi-tenant GPU platforms actually leak: per-tenant prefixes, per-tenant
  workload identity (§3), and bucket policies that reference the tenant's role — not one
  shared node role that every job inherits.

## Self-quiz

**1. Which RBAC verbs let a user escalate to cluster-admin indirectly?**
`escalate` (create a Role containing permissions you do not have) and `bind` (bind a role
whose permissions you do not have) — those two exist precisely to bypass the built-in
escalation prevention, and grant them to nobody routine. Then the indirect set:
`create pods` (choose any ServiceAccount in the namespace, mount any Secret, request host
namespaces if policy allows), `pods/exec` and `pods/attach` (run code as an existing
workload), `impersonate` (become another user or group), `get`/`list` on `secrets`,
`create` on `serviceaccounts/token`, and write access to workload controllers — a
DaemonSet lands a pod on every node. Anything that can edit `ValidatingWebhookConfiguration`
or `MutatingWebhookConfiguration` is also effectively cluster-admin, because it can
disable or subvert enforcement.

**2. Kyverno vs OPA/Gatekeeper — when do you reach for Rego?**
Kyverno for the common Kubernetes cases: it is YAML-native, so policies read like the
resources they govern, and it does mutation, generation and cosign image verification out
of the box. Rego/Gatekeeper when the logic is genuinely complex (set operations, joins
across objects via the cached inventory), when you need the same policy language outside
Kubernetes — Rego over Terraform plan JSON in CI is the same engine — or when your org has
already standardised on OPA. And increasingly the right first question is "does this need
a webhook at all?": ValidatingAdmissionPolicy (stable 1.30) and MutatingAdmissionPolicy
(GA 1.36) run CEL in-process, so there is no webhook availability risk and no extra
deployment to keep alive.

**3. How does a pod get a short-lived, audience-scoped token instead of a static SA
secret?**
Through a projected volume of type `serviceAccountToken` with an `audience` and
`expirationSeconds`. The kubelet calls the TokenRequest API on the pod's behalf and writes
a JWT to the mount path; the token defaults to 1 hour (minimum 10 minutes), carries
`sub=system:serviceaccount:<ns>:<sa>` plus pod/node binding claims, and the kubelet
proactively rotates it at 80% of its TTL or after 24 hours. The application re-reads the
file. Deleting the pod or the ServiceAccount invalidates the token, unlike a legacy Secret
token which never expires. The receiving service must validate the `aud` claim, otherwise
a token intended for one service can be replayed at another. `ServiceAccountTokenNodeBinding`
(GA 1.33) extends this to binding tokens to the Node object as well.

**4. Why is multi-tenant isolation weaker on a shared GPU than on shared CPU, and what
restores it?**
Because the kernel does not mediate GPU access the way it mediates CPU and memory. CPU
shares are enforced by the scheduler and cgroups, memory by the MMU and the OOM killer,
and a misbehaving process is contained. On a GPU shared by time-slicing, CUDA contexts
simply interleave: there is no framebuffer quota, no compute guarantee, and — the sharp
edge — one workload's fault takes down the device for every workload sharing it, because
they are in the same fault domain. What restores isolation, in ascending order: MPS
(space partitioning with enforced memory and compute limits, still one fault domain);
**MIG**, which is a hardware partition with dedicated SMs and framebuffer and is the only
real boundary; and, above that, giving each tenant whole devices or whole nodes. The
policy consequence: `restricted` PSS to keep tenants off `/dev` and off hostPath, RBAC
around DRA claims and admin access, and per-tenant identity on the storage the jobs read
and write.

**5. Your admission webhook is down and pods are being rejected cluster-wide. What
happened and what is the fix?**
The webhook's `failurePolicy` is `Fail` and its `matchConditions`/`namespaceSelector` are
broad enough to match everything, so a control-plane request cannot complete without a
response from a workload that is itself down — including the workload that would fix it.
Immediate mitigation: delete or narrow the `ValidatingWebhookConfiguration` (a break-glass
role that can do exactly this, and only this, should exist and be audited), restore the
webhook, reapply. Structural fixes: exclude `kube-system` and the policy engine's own
namespace from the selector, run the engine HA across nodes with a PDB, set a short
`timeoutSeconds`, and alert on webhook error rate as a security signal — because with
`failurePolicy: Ignore` the same outage would have silently stopped enforcing instead.

**6. You enabled etcd encryption at rest. Are existing Secrets encrypted?**
No. Encryption applies on write, so pre-existing objects remain in their previous form
until they are rewritten — the standard step is
`kubectl get secrets --all-namespaces -o json | kubectl replace -f -`. Two related traps:
the provider list is ordered and the **first** provider encrypts while all listed ones can
decrypt, so leaving `identity` first means nothing is encrypted; and rolling *back*
requires the old provider still being present in the list, or you cannot read what you
wrote. Also be honest about the threat model this addresses: it protects etcd's disk and
backups, not an attacker who can read Secrets through the API, which is what RBAC is for.

## Refresh only if

- **Pod Security Admission**, if your model is still PodSecurityPolicy: PSA is stable
  since 1.25, is configured with namespace labels
  (`pod-security.kubernetes.io/{enforce,audit,warn}` plus `-version`), applies the three
  Pod Security Standards profiles, and — the detail that catches people — **`enforce` does
  not apply to workload resources, only to the resulting pods**, so pair it with `warn`.
- **In-process CEL policy**, if your only enforcement model is webhooks:
  ValidatingAdmissionPolicy has been stable since 1.30 and **MutatingAdmissionPolicy
  reached GA in 1.36**, which changes the default answer to "do I need Kyverno for this?"
  for a large class of simple rules.
- **Sigstore/SLSA enforcement mechanics**, if you have not wired admission-time
  verification — it overlaps the CI/CD refresh and is a frequent 2025–26 probe.
- **User namespaces**, if your isolation ladder skips from "node pools" to "gVisor":
  `UserNamespacesSupport` reached **GA in 1.36**, which makes container-root ≠ host-root a
  mainstream option rather than an experiment.

## Recall card

Cover the right column and say each value out loud; if one is fuzzy, reread the section
in brackets.

| Fact | Value |
|---|---|
| Request pipeline | authn → authz (Node/RBAC/webhook, additive) → mutating admission → validating admission → etcd [§1] |
| RBAC has no deny | Denial lives in admission, not RBAC [§1] |
| Escalation verbs | `escalate` (write any Role), `bind` (bind any Role) [§2] |
| Indirect escalation | create pods · pods/exec · impersonate · read secrets · edit webhooks [§2] |
| Projected token | default 1 h, min 10 min, rotate at 80% TTL or 24 h; audience-bound [§3] |
| Node-bound tokens | `ServiceAccountTokenNodeBinding` GA 1.33 [§3] |
| PSS profiles | privileged · baseline · restricted (drop ALL caps, only `NET_BIND_SERVICE` back; seccomp `RuntimeDefault`) [§4] |
| PSA labels | `pod-security.kubernetes.io/{enforce,audit,warn}` + `-version`; enforce skips workload resources [§4] |
| CEL policy status | ValidatingAdmissionPolicy stable 1.30 · MutatingAdmissionPolicy GA 1.36 [§4] |
| Webhook failure policy | `Fail` = policy engine is a control-plane dependency; `Ignore` = silent gap [§4] |
| Encryption at rest | first provider encrypts, all decrypt; `identity` = plaintext; KMSv1 off by default since 1.29 [§5] |
| Encryption is not retroactive | rewrite objects with `get -o json \| kubectl replace -f -` [§5] |
| NetworkPolicy default-deny | `podSelector: {}` + `policyTypes: [Ingress, Egress]`, no rules [§7] |
| User namespaces | GA 1.36 [§7] |
| GPU isolation ladder | time-slicing (none) → MPS (enforced limits, shared fault domain) → MIG (hardware) [§8] |
