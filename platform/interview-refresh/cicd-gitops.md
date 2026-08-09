---
area: "CI/CD & GitOps"
kind: refresh
status: not-refreshed      # not-refreshed | refreshed
---

# 🔁 CI/CD & GitOps — interview refresh

> Pipeline design, ArgoCD/Flux, progressive delivery, supply-chain security.
>
> You know this. Goal here is fast recall + crisp interview framing, not study.

## Talking points to have ready

- **CI vs CD vs GitOps, cleanly separated.** CI = build/test/produce a signed artifact; CD =
  promote that artifact through envs; **GitOps = a controller (Argo/Flux) reconciles cluster
  state to a Git desired-state**, so deploy = merge, rollback = revert. The pull-based reconcile
  is the point: no CI system holding cluster creds, drift auto-corrected.
- **Progressive delivery.** Canary / blue-green / rolling; **Argo Rollouts or Flagger** driving
  it off real SLIs (latency/error rate) with automated abort. Say the metric that gates promotion
  and the automatic rollback trigger — that's the senior signal.
- **Supply-chain security (the 2025 table stakes).** Sign artifacts with **Sigstore/cosign**,
  generate **SBOMs** (Syft), enforce **provenance/SLSA** levels, verify signatures at admission
  (Kyverno/`cosign verify`). Pin actions by SHA; least-privilege CI tokens (OIDC federation, no
  long-lived cloud keys).
- **App-of-apps / multi-cluster.** Argo ApplicationSets to fan a config across clusters/regions;
  environment promotion by directory or overlay (Kustomize/Helm), not by mutating a shared branch.
- **The GPU/ML angle.** Deploying an inference service is a *model-plus-image* rollout: canary a
  new model/weights on real traffic with latency+quality gates; GitOps the DCGM/exporter/operator
  stack; the "artifact" may be a multi-GB model, so registry/caching and rollout speed matter.

## Self-quiz

- Why is pull-based GitOps more secure than a CI job with kubeconfig? *(no external system holds
  cluster creds; reconcile + drift correction; audit = git history.)*
- What SLI gates a canary promotion, and what triggers the automatic rollback?
- How do you verify an image's provenance at deploy time — the exact chain from build to admission?
- Helm vs Kustomize for environment overlays — when each, and why not both badly?

## Refresh only if

- **Sigstore/SLSA specifics** if you haven't wired cosign+Kyverno verification yourself — the
  supply-chain bar rose fast and it's a common 2025-26 interview probe.
- **Argo vs Flux** current state (Argo CD + Rollouts vs Flux + Flagger, and the CNCF graduation/
  governance shifts) if your mental model is a few years old.
