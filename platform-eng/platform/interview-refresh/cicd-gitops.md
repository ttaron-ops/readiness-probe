---
area: "CI/CD & GitOps"
kind: refresh
status: not-refreshed      # not-refreshed | refreshed
---

# 🔁 CI/CD & GitOps — interview refresh

> Pipeline design, ArgoCD/Flux, progressive delivery, supply-chain security.
>
> You know this. Goal here is fast recall + crisp interview framing, not study.

The whole area reduces to two mechanisms and one trust chain. The mechanisms are
**render-then-diff-then-apply on a loop** (GitOps) and **shift traffic while measuring,
abort on the measurement** (progressive delivery). The trust chain is **who signed this
artifact, what proves it came from that source, and where is that checked before it
runs**. If you can draw those three on a whiteboard with real field names and real
timings, you are done here.

Version-specific claims below were checked against upstream sources on GitHub —
`argoproj/argo-cd` (`docs/`, `cmd/argocd-application-controller`), `argoproj/argo-rollouts`
(`docs/features/`), `fluxcd/flagger` (`docs/gitbook/usage/how-it-works.md`),
`sigstore/cosign` (`README.md`, `doc/`), `slsa-framework/slsa` (`spec/levels.md` on
`releases/v1.1`), and `kyverno/policies`, read 2026-08-18. The corresponding vendor doc
sites (argo-cd.readthedocs.io, fluxcd.io, kyverno.io, slsa.dev) are egress-blocked from
this environment and were **not** relied on.

## Talking points to have ready

### 1. CI, CD and GitOps, cleanly separated

- **CI** builds and tests, and its output is an immutable, addressable artifact —
  an image **digest**, not a tag. Tags move; digests don't. Everything downstream should
  reference `@sha256:…`.
- **CD** promotes that same artifact through environments. The promotion unit is the
  digest plus config. If your pipeline rebuilds per environment, you are not promoting an
  artifact, you are hoping two builds are identical.
- **GitOps** is the delivery *mechanism*: a controller inside the cluster continuously
  reconciles live state toward a declarative desired state stored in Git (or an OCI
  artifact). Deploy = merge. Rollback = revert. Drift = corrected automatically.

The security argument, said precisely: with push-based CD, an external system holds
cluster-admin credentials and its compromise is a cluster compromise. With pull-based
GitOps, the credential is a read-only repo token held by an in-cluster controller,
outbound-only, and the audit trail is the Git history plus the controller's events. That
is a real reduction in attack surface, not a slogan.

### 2. The reconciliation loop, with real numbers

```
 developer                 git repo                    cluster
 ─────────                 ────────                    ───────
  merge PR ──▶ commit ──┐
                        │  ①  webhook (instant)          ┌───────────────────┐
                        └──────────────────────────────▶ │  repo-server      │
                        │                                │  git clone/fetch  │
                        │  ②  poll: resync every 120s    │  helm template /  │
                        │      + jitter up to 60s        │  kustomize build  │
                        └──────────────────────────────▶ └────────┬──────────┘
                                                                  │ rendered manifests
                                                                  ▼  (cached in Redis)
     ┌───────────────────────────────────────────────────────────────────────┐
     │                    application-controller                             │
     │   desired (rendered)  ⨯  live (informer cache, watch-driven)          │
     │   ── normalise ──▶ diff ──▶ Synced | OutOfSync                        │
     │                                │                                      │
     │                  auto-sync? ───┴── selfHeal? ──▶ apply by wave        │
     │                                                  (wave −1, 0, 1 …;     │
     │                                                   2 s between waves)   │
     └───────────────────────────────┬───────────────────────────────────────┘
                                     │ kubectl-equivalent apply (SSA or client-side)
                                     ▼
                              live objects ──▶ health assessment ──▶ Healthy/Degraded
                                     ▲                                    │
                                     └──── someone kubectl-edits ─────────┘
                                            → OutOfSync → selfHeal reverts it
```

Numbers an interviewer can check, from the Argo CD source rather than folklore:

| Setting | Default | Where |
|---|---|---|
| Application resync period | **120 s** (`defaultAppResyncPeriod = 120`) | `cmd/argocd-application-controller`, env `ARGOCD_RECONCILIATION_TIMEOUT`, ConfigMap key `timeout.reconciliation` |
| Resync jitter | **60 s** (`defaultAppResyncPeriodJitter = 60`) | same; spreads refresh load so 5,000 apps don't stampede the repo-server |
| Hard resync | **0 = disabled** | `--app-hard-resync` / `ARGOCD_HARD_RECONCILIATION_TIMEOUT` (bypasses the manifest cache) |
| Delay between sync waves | **2 s** | `ARGOCD_SYNC_WAVE_DELAY` — exists so other controllers can react before health is assessed |
| Sync wave of an unannotated resource | **0** (negatives allowed) | `argocd.argoproj.io/sync-wave` |

If your recall says "180 seconds", that is the older default — current upstream is 120 s
with jitter, and being right about *that* is a cheap credibility win.

Sync semantics worth stating because they explain half of all "why didn't it deploy"
tickets:

- Auto-sync fires **only when the app is OutOfSync**, and **only once per unique
  (commit SHA, parameters)** pair — a second sync of the same SHA is skipped unless
  `selfHeal: true`.
- **Pruning is off by default.** Deleting a manifest from Git does not delete the object
  until `prune: true`. And with prune on, `allowEmpty: false` (the default) refuses to
  prune everything when a render accidentally produces zero resources — that safety
  catch has saved more clusters than any policy engine.
- Failed syncs retry with exponential backoff you configure explicitly:

```yaml
spec:
  syncPolicy:
    automated:
      prune: true          # delete objects removed from Git
      selfHeal: true       # revert manual cluster edits
      allowEmpty: false    # refuse to prune down to zero resources
    retry:
      limit: 5             # -1 for unlimited
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

**Ordering.** Two mechanisms, often confused. *Phases/hooks* (`argocd.argoproj.io/hook:
PreSync | Sync | PostSync | SyncFail | PreDelete | PostDelete`) are coarse: PreSync
completes before any manifest is applied, PostSync runs after everything is applied *and
healthy*. *Waves* order resources inside a phase. During pruning the wave order reverses —
higher waves are deleted first — which is why a CRD in wave −1 and its CRs in wave 0
delete in the right order without you thinking about it.

### 3. Argo CD vs Flux — the comparison to have memorised

Both are CNCF-graduated and both implement the same loop; the differences are structural.

| | Argo CD | Flux v2 |
|---|---|---|
| Unit of config | `Application` / `ApplicationSet` CRs, usually one control plane per fleet | A set of controllers: source, kustomize, helm, notification, image-reflector, image-automation |
| Sources | Git, Helm repo, OCI | `GitRepository`, `OCIRepository`, `HelmRepository`, `HelmChart`, `Bucket` |
| Reconcile cadence | Global resync default (120 s + jitter), webhook-accelerated | **Per-object `interval`**, set on each CR — no single global default to remember |
| UI | First-class web UI, RBAC, SSO — the reason platform teams pick it | CLI/CRD-first; UI via third parties |
| Helm | Renders with `helm template` and applies the output — no Tiller, no in-cluster release by default | `HelmRelease` driven by helm-controller, real Helm releases with storage |
| Progressive delivery | Argo Rollouts (separate controller, new `Rollout` kind) | Flagger (separate controller, wraps existing Deployments) |
| Image updates | Argo CD Image Updater (separate, optional) | Built in: image-reflector + image-automation write the new digest back to Git |
| Multi-tenancy | App projects, cluster-scoped controller | Namespace-scoped reconciliation, tenants own their own `Kustomization`s |

The interview line: **Argo CD centralises** (one control plane, one UI, one RBAC model,
fleet visible in one place) and **Flux decentralises** (controllers reconcile per-object
with per-object intervals, tenants own their pipelines, easier to embed in a
product). Neither is meaningfully more or less "GitOps".

**Helm vs Kustomize** for overlays, since it always follows: Helm gives you
parameterisation, packaging and distribution — right when the artifact is *shipped to
other people* or when a vendor ships you a chart. Kustomize gives you patching over a
literal base with no template language — right when the manifests are *yours* and the
per-environment delta is small. The failure mode of "both badly" is a Helm chart whose
`values.yaml` has grown 400 keys because every field became configurable, then a
Kustomize overlay patching the rendered output because the chart still didn't expose the
one field you needed. Pick the layer where the variation actually lives.

### 4. Progressive delivery: what actually gates the promotion

Canary/blue-green/rolling is the easy half. The senior signal is naming **the metric,
the threshold, the sample window, and the abort rule** — and knowing what a rollback
costs.

**Argo Rollouts** replaces the `Deployment` with a `Rollout` kind and drives steps:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  replicas: 10
  strategy:
    canary:
      maxSurge: "25%"
      maxUnavailable: 0
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 2          # don't start measuring until 40% weight
      steps:
        - setWeight: 20
        - pause: {duration: 10m}
        - setWeight: 40
        - pause: {duration: 10m}
        - setWeight: 60
        - pause: {duration: 10m}
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 5m                        # sample every 5 minutes
      successCondition: result[0] >= 0.95 # Prometheus returns a vector
      failureLimit: 3                     # 3 bad samples ⇒ analysis Failed
      provider:
        prometheus:
          address: http://prometheus.example.com:9090
          query: |
            sum(rate(http_requests_total{status!~"5..",job="guestbook"}[5m]))
            /
            sum(rate(http_requests_total{job="guestbook"}[5m]))
```

When the analysis fails, the controller **aborts**: canary weight goes back to 0 and the
Rollout is marked `Degraded`. Without traffic management, `setWeight` is approximated by
replica counts (10 replicas, `setWeight: 10` ⇒ 1 canary pod / 9 stable); with a traffic
router (Istio, ALB, NGINX, Gateway API providers) the weight is real and the canary's
scale can be decoupled with `setCanaryScale`. `pause: {}` with no duration pauses
indefinitely until `kubectl argo rollouts promote <rollout>` — that is your manual gate.

**Flagger** takes the opposite shape: you keep your `Deployment` and add a `Canary`
object that manages copies of it. Field defaults worth knowing:

| Field | Meaning | Default |
|---|---|---|
| `analysis.interval` | How often the analysis loop runs | **60 s** |
| `analysis.threshold` | Max failed metric checks before rollback | — (set it; this is your abort rule) |
| `analysis.maxWeight` | Highest traffic % the canary reaches before promotion | — |
| `analysis.stepWeight` | Increment per successful interval | — |
| `primaryReadyThreshold` / `canaryReadyThreshold` | % of pods that must be ready before/while rolling | **100** |
| built-in metrics | `request-success-rate`, `request-duration` with `thresholdRange` | — |

A concrete canary schedule to quote: `interval: 1m`, `stepWeight: 5`, `maxWeight: 50`,
`threshold: 10` — traffic climbs 5% per minute to 50% over ten minutes, and any ten
failed checks abort and roll back. Webhooks let you attach conformance and load tests to
each interval, and manual gates.

**Which SLI gates a promotion?** The user-visible one, computed over the canary's traffic
only: request success rate and a latency quantile (p95/p99) are the default pair; add a
domain correctness metric where you have one. Do not gate on CPU, replica count, or
"pods are Running" — those are causes, not symptoms, and they will happily be green while
users get 500s. Sizing matters too: at 5% of traffic and 100 req/s you get 5 req/s into
the canary, so a 1% error rate is one error every 20 seconds and needs minutes of window
to be distinguishable from noise. State that arithmetic; it is the difference between a
canary and a coin flip.

### 5. Supply chain: the chain from commit to admission

```
 commit ──▶ CI runner (GitHub Actions, ephemeral)
              │  permissions: id-token: write   → OIDC token, no long-lived cloud keys
              │  actions pinned by commit SHA, not tag
              ▼
           build ──▶ image (digest sha256:…)
              │
              ├─▶ syft ──▶ SBOM (SPDX/CycloneDX) ──┐
              │                                     │  attach as attestation
              ├─▶ provenance (in-toto/SLSA) ────────┤
              │                                     ▼
              └─▶ cosign sign (keyless) ──▶ Fulcio issues a short-lived cert
                        │                    bound to the OIDC identity
                        │                    (workflow ref @ ref/heads/main)
                        ▼
                  Rekor transparency log  ← signature + cert recorded (index returned)
                        │
                  signature stored in the registry next to the image
                        │
                        ▼
   deploy ──▶ admission webhook (Kyverno / Gatekeeper+external data / policy-controller)
                        │  verify signature against expected identity + issuer
                        │  mutateDigest: rewrite tag → digest so what you verified is
                        │                what runs
                        ▼
                   pod admitted / rejected
```

**Keyless signing, mechanically.** `cosign sign` (no `--key`) obtains an OIDC identity
token — in CI that is the workflow's own token, issuer
`https://token.actions.githubusercontent.com`, subject the workflow ref — exchanges it at
the **Fulcio** CA for a short-lived code-signing certificate, signs the image digest, and
records signature + certificate in the **Rekor** transparency log (cosign prints the
`tlog entry created with index: …`). The signature is pushed to the registry alongside
the image. Cosign uses ECDSA-P256 with SHA-256 for both keyless and key-based signing
(cosign README). Verification pins *who*, not *what key*:

```console
$ cosign verify ghcr.io/acme/inference@sha256:9f4e… \
    --certificate-identity \
      "https://github.com/acme/inference/.github/workflows/release.yml@refs/heads/main" \
    --certificate-oidc-issuer "https://token.actions.githubusercontent.com"
```

`--certificate-identity-regexp` / `--certificate-oidc-issuer-regexp` exist for fleets of
repos; `--offline` plus a `--trusted-root` covers air-gapped verification, since the
default flow periodically fetches Sigstore's trust root over TUF. The reason keyless wins
in CI: there is no signing key to store, rotate, or leak — the private key exists for the
lifetime of one build, and the transparency log makes any misuse discoverable.

**SLSA build levels** (v1.1 spec, `spec/levels.md`, Build track):

| Level | Requirement | Threat it addresses |
|---|---|---|
| Build L0 | none | — |
| Build L1 | Provenance exists: the platform generates provenance saying what built the artifact from what inputs; may be unsigned | Mistakes, undocumented builds |
| Build L2 | Signed provenance from a **hosted** build platform | Tampering **after** the build |
| Build L3 | Hardened platform: runs cannot influence one another, and the provenance signing key is not reachable from user-defined build steps | Tampering **during** the build |

Two things people get wrong: L3 is about *build-platform isolation and unforgeable
provenance*, not about how many scanners you run; and there is no released Build L4 —
it is an active workstream in the SLSA repo. Say "L3 is the practical ceiling today" and
you sound current rather than aspirational.

**Enforcement at admission**, the part that makes the rest real:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-inference-images
spec:
  validationFailureAction: Enforce     # Audit first, then flip to Enforce
  background: false
  rules:
    - name: verify-signature
      match:
        any:
          - resources:
              kinds: [Pod]
      verifyImages:
        - imageReferences:
            - "ghcr.io/acme/*"
          mutateDigest: true           # pin tag → digest at admission
          attestors:
            - entries:
                - keys:
                    publicKeys: |
                      -----BEGIN PUBLIC KEY-----
                      …
                      -----END PUBLIC KEY-----
```

(Structure taken from `kyverno/policies`. For keyless verification the attestor entry
pins the certificate subject and OIDC issuer instead of a public key — check the field
names against the CRD for your Kyverno version rather than trusting memory.)
`mutateDigest: true` is the load-bearing line: verifying a *tag* and then letting the
kubelet resolve that tag at pull time is a time-of-check/time-of-use hole.

**CI hygiene that gets asked about by name**: pin third-party actions to a commit SHA
(tags are mutable, and tag-hijack has been the vector in real Actions supply-chain
incidents); scope `permissions:` per job with `id-token: write` only where signing
happens; use OIDC federation to cloud roles instead of long-lived access keys; keep
secrets out of pull-request-triggered workflows from forks.

### 6. Multi-cluster and promotion

`ApplicationSet` generates `Application`s from a generator (list, cluster, Git
directory/file, matrix, merge, SCM/pull-request, cluster-decision-resource) so one
template fans out across clusters and regions. The pattern to describe:

```
  apps/                      ← Git directory generator walks this
    inference/
      base/                  ← kustomize base: Deployment, Service, HPA
      overlays/
        dev/kustomization.yaml     images: ghcr.io/acme/inference@sha256:aaa…
        stage/kustomization.yaml   images: ghcr.io/acme/inference@sha256:aaa…
        prod-us/kustomization.yaml images: ghcr.io/acme/inference@sha256:bbb…
        prod-eu/kustomization.yaml images: ghcr.io/acme/inference@sha256:bbb…
```

Promotion is **a commit that changes a digest in the next overlay** — reviewable,
revertable, and auditable. It is not "merge stage into prod" (branch-per-environment
turns promotion into a merge conflict and makes cherry-picking hotfixes a manual art) and
it is definitely not "rebuild with `--env=prod`". A promotion PR that changes one line
from `sha256:aaa` to `sha256:bbb` is the artifact of a well-designed pipeline.

### 7. The GPU / ML angle

This is where a generic CI/CD answer stops being enough.

- **The rollout unit is model + image, and the model dominates.** A 70B model in bf16 is
  ~140 GB of weights; even a modest 13B is ~26 GB. Worked arithmetic: pulling 140 GB at a
  sustained 1 GB/s is ~140 s *per node* if the bytes come from a local cache, and pulling
  it 64 times through a shared 10 Gb/s registry path (≈1.25 GB/s aggregate) is ~2 hours
  of serialised transfer. That single number decides your architecture: keep weights out
  of the container image, distribute them as an OCI artifact or from object storage with
  a node-local cache, pre-warm before you shift traffic, and make the canary's first step
  "weights present on the target nodes", not "5% of traffic".
- **Canary capacity is not free.** In a web service, a 10% canary is 10% more cheap pods.
  On a GPU fleet, the canary needs *GPUs you already own* — so a canary is usually carved
  out of the stable fleet, which means you are running the old version at reduced capacity
  during the analysis. Say that out loud, then say the mitigation: shorter analysis
  windows with tighter thresholds, or blue-green at the granularity of a whole node pool
  during a maintenance window.
- **Gate on quality, not only on latency and errors.** A new model version can be fast,
  200-OK, and wrong. The gating set for an inference rollout is: request success rate,
  p95/p99 latency (and time-to-first-token separately from total, since they have
  different bottlenecks), tokens/second throughput, and an offline-eval or
  shadow-comparison score. Rollouts' `AnalysisTemplate` takes any Prometheus query, so
  the quality metric is just another `successCondition`.
- **GitOps the accelerator stack too.** GPU operator, driver DaemonSet, device plugin,
  DCGM exporter, and the feature-discovery labels belong in Git with everything else — and
  they carry a version matrix (driver ↔ CUDA runtime ↔ operator ↔ kernel) that is the
  most common source of "the node came back without GPUs". Pin those versions in the same
  repo that pins the workload, and roll them per node pool with a canary pool first.
- **Reconciliation must not evict a training job.** `selfHeal` reverting a manual change
  is fine for a stateless service and catastrophic for a 500-GPU run that has been going
  for nine days. Practical guardrails: keep long-running jobs out of auto-synced
  Applications (or use `argocd.argoproj.io/compare-options: IgnoreExtraneous` and
  `ignoreDifferences` deliberately), scope PodDisruptionBudgets so a sync cannot drain a
  node mid-step, and treat "does this sync restart pods?" as a review question on every PR
  touching the GPU namespaces.

## Self-quiz

**1. Why is pull-based GitOps more secure than a CI job with a kubeconfig?**
Because the credential moves. In push CD, an internet-facing CI system holds a
cluster-admin kubeconfig for every cluster; compromising the CI system — or one of its
third-party actions — compromises the fleet. In pull GitOps, the cluster runs a
controller with a read-only repo credential and outbound-only connectivity; no external
system has cluster write access. Secondary benefits worth naming: continuous drift
correction (`selfHeal`), an audit trail that is the Git history plus controller events,
and the fact that the desired state is inspectable without cluster access. The honest
caveat: the controller itself is now a very privileged in-cluster component, and your
Git repo's branch protection is now a production access control — compromise the repo
and you have compromised the cluster, just more slowly and more visibly.

**2. What SLI gates a canary promotion, and what triggers the automatic rollback?**
User-visible symptom metrics measured on canary traffic: success rate (non-5xx / total)
and a latency quantile, plus a correctness metric where one exists. The rollback trigger
is an explicit failure budget in the analysis: in Argo Rollouts, `successCondition` plus
`failureLimit` (N bad samples at `interval` cadence ⇒ analysis Failed ⇒ abort, canary
weight to 0, Rollout `Degraded`); in Flagger, `threshold` failed checks at
`analysis.interval` (default 60 s) ⇒ traffic reset and rollback. Then the sizing point:
the window must be long enough that the canary's traffic share makes the threshold
statistically meaningful — 5% of 100 req/s is 5 req/s, which cannot detect a 1% error-rate
regression in 60 seconds.

**3. How do you verify an image's provenance at deploy time — the exact chain?**
CI runs with an OIDC identity (`permissions: id-token: write`), builds the image, and
signs the **digest** keylessly with cosign: the OIDC token is exchanged at Fulcio for a
short-lived certificate whose subject is the workflow identity, the signature and
certificate go into the Rekor transparency log, and the signature is stored in the
registry beside the image. Provenance (in-toto/SLSA) and an SBOM are attached as
attestations the same way. At admission, Kyverno's `verifyImages` rule matches the image
reference, verifies the signature against the expected `--certificate-identity` and
`--certificate-oidc-issuer` (or a public key), verifies any required attestation
predicates, and — critically — `mutateDigest: true` rewrites the tag to the verified
digest so the kubelet cannot pull something else. Anything unsigned or signed by an
unexpected identity is rejected.

**4. Helm vs Kustomize for environment overlays — when each, and why not both badly?**
Helm when the thing is *packaged and distributed* (a vendor chart, an internal chart other
teams consume) or when the variation is genuinely parametric. Kustomize when the manifests
are yours and each environment is a small patch over one base — no templating language,
diffs that read as diffs. Both badly = a chart with hundreds of values keys plus overlays
patching its rendered output, so a change requires understanding two indirection layers.
If you must combine, do it in one direction only: render the chart, then patch the render
(Argo CD and Flux both support this), and keep the chart's values surface small.

**5. Argo CD says `Synced` but the app is broken. Where do you look?**
`Synced` means live state matches the rendered desired state — it says nothing about
whether the workload works, that is the separate *health* assessment. Walk it: is the
Application `Healthy` or `Degraded`? Degraded means Argo's health check (built-in per
kind, or a custom Lua check) says the resource is unhappy — go to pod-level debugging. If
`Healthy` but users disagree, the desired state itself is wrong: check which commit the app
is synced to (`argocd app get`), whether the render used the parameters you expect, and
whether something outside the Application's scope is broken (a shared CRD, a webhook, a
secret in another namespace). Also check `ignoreDifferences` and `compare-options` — an
over-broad ignore rule makes real drift invisible and is a classic self-inflicted wound.

**6. A GPU inference service needs a new model version rolled out safely. Design it.**
Separate the two artifacts: the server image (small, versioned, signed) and the weights
(large, content-addressed, distributed as an OCI artifact or from object storage). Stage
weights onto target nodes and verify presence before any traffic moves — pulling ~140 GB
per node is minutes at best and hours through a shared registry, so it cannot happen
inside a traffic step. Then run a canary sized by GPUs you can borrow from the stable
fleet, gated on success rate, time-to-first-token, end-to-end p95 and an eval/shadow
quality score, with `failureLimit` set so a bad model aborts within minutes rather than
after the whole ramp. Everything — Rollout, AnalysisTemplate, model digest, GPU operator
version — lives in Git and is promoted by changing a digest in the next overlay. Roll back
by reverting that commit; keep the previous weights cached on the nodes so the rollback is
seconds, not another hour of pulls.

## Refresh only if

- **Sigstore/SLSA specifics**, if you have not wired cosign + admission verification
  yourself. The bar to be able to describe end to end: OIDC → Fulcio short-lived cert →
  Rekor log entry → registry-stored signature → `cosign verify --certificate-identity
  --certificate-oidc-issuer` → Kyverno `verifyImages` with `mutateDigest: true`. Plus the
  SLSA build-level table above and the fact that L3 (isolated builds, non-falsifiable
  provenance) is the current top of the released Build track.
- **Argo vs Flux current state**, if your mental model is a few years old: both are CNCF
  **graduated**; Argo CD's default reconciliation is 120 s + up to 60 s jitter (not 180 s);
  Flux's model is per-object `interval` across the source/kustomize/helm/notification/
  image-automation controllers, with OCIRepository as a first-class source; and progressive
  delivery lives in a separate controller in both stacks (Rollouts / Flagger).
- **Rollouts' plugin direction**, if you plan to name metric providers: the project has
  moved to a plugin architecture for metrics, traffic routers and steps, and states it is
  no longer accepting new metric integrations into the core controller.

## Recall card

Cover the right column and say each value out loud; if one is fuzzy, reread the section
in brackets.

| Fact | Value |
|---|---|
| Argo CD resync default | 120 s + up to 60 s jitter; hard resync 0 (off) [§2] |
| Sync wave default / delay | wave 0 · 2 s between waves (`ARGOCD_SYNC_WAVE_DELAY`) [§2] |
| Auto-sync fires when | OutOfSync only, once per (SHA, params) unless `selfHeal` [§2] |
| Prune / allowEmpty defaults | prune off; `allowEmpty: false` blocks pruning to zero [§2] |
| Hook phases | PreSync · Sync · PostSync · SyncFail · PreDelete · PostDelete [§2] |
| Flagger `analysis.interval` default | 60 s; `threshold` = failed checks before rollback [§4] |
| Rollouts abort semantics | `failureLimit` bad samples ⇒ weight 0, `Degraded` [§4] |
| Keyless signing chain | OIDC → Fulcio cert → Rekor log → signature in registry (ECDSA-P256/SHA-256) [§5] |
| Verify flags | `--certificate-identity`, `--certificate-oidc-issuer` (+`-regexp`) [§5] |
| SLSA build track | L1 provenance exists · L2 signed, hosted · L3 hardened/isolated; no released L4 [§5] |
| Admission must-have | `mutateDigest: true` — verify and run the same bytes [§5] |
| Promotion unit | an image **digest** moved between overlays, never a rebuild [§6] |
| GPU rollout constraint | weights (10s–100s of GB) pre-staged before any traffic step [§7] |
