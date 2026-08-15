---
area: "Security & policy"
kind: refresh
status: not-refreshed      # not-refreshed | refreshed
---

# 🔁 Security & policy — interview refresh

> RBAC, admission control (OPA/Kyverno), secrets, supply chain, runtime security.
>
> You know this. Goal here is fast recall + crisp interview framing, not study.

## Talking points to have ready

- **AuthN vs AuthZ, and K8s RBAC done right.** Roles/ClusterRoles bound narrowly; no wildcard
  verbs; **ServiceAccounts as workload identity** with projected, short-lived, audience-bound
  tokens (not static secrets). The escalation paths to name: `escalate`/`bind`, `create pods`
  (→ mount any secret), and node/kubelet trust.
- **Admission control as the policy chokepoint.** Validating + mutating webhooks; **Kyverno vs
  OPA/Gatekeeper** (Kyverno = YAML-native, easier; OPA = Rego, more expressive). Enforce Pod
  Security Standards (restricted), require signed images, block `hostPath`/privileged/`hostNetwork`,
  mandate resource limits and non-root. This is where supply-chain verification lands too.
- **Secrets.** External secret stores (Vault / cloud SM) via CSI driver or External Secrets
  Operator; **encryption-at-rest for etcd** (KMS provider); short-lived dynamic secrets over
  long-lived. Never secrets in env/images/git.
- **Supply chain (2025 table stakes).** Sign+verify (cosign/Sigstore), SBOMs, **SLSA provenance**,
  admission-time signature verification. Pin dependencies; scan (Trivy/Grype).
- **Runtime & isolation.** **Falco/eBPF** runtime detection; network policy default-deny;
  seccomp/AppArmor; the multi-tenant isolation ladder (namespace → node → **sandboxed runtime
  (gVisor/Kata)** → separate cluster). The GPU twist: **GPU device isolation is weaker than CPU**
  — MIG gives a hardware boundary, time-slicing gives *none*, and a shared GPU's memory/fabric is
  a real tenancy concern (ties to the cost-attribution + multi-tenant design work).

## Self-quiz

- Which RBAC verbs let a user escalate to cluster-admin indirectly? *(`escalate`, `bind`,
  pod-create → secret mount, impersonate.)*
- Kyverno vs OPA/Gatekeeper — when do you reach for Rego?
- How does a pod get a short-lived, audience-scoped token instead of a static SA secret?
- Why is multi-tenant isolation *weaker* on a shared GPU than on shared CPU, and what restores it?

## Refresh only if

- **Pod Security Admission** (the PSP replacement) if your mental model is still PodSecurityPolicy.
- **Sigstore/SLSA enforcement mechanics** if you haven't wired admission-time verification — it's a
  frequent 2025-26 probe and overlaps the CI/CD refresh.
