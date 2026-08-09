---
area: "Cloud & multi-cloud architecture"
kind: refresh
status: not-refreshed      # not-refreshed | refreshed
---

# 🔁 Cloud & multi-cloud architecture — interview refresh

> AWS depth plus GCP/Azure breadth; landing zones, identity, GPU offerings per cloud.
>
> You know this. Goal here is fast recall + crisp interview framing, not study.

## Talking points to have ready

- **Landing zone / org design.** Multi-account (AWS Orgs / GCP folders / Azure mgmt groups) by
  blast radius and billing boundary; centralized identity, logging, and guardrails (SCPs/Org
  policies); a network hub (Transit Gateway / Cloud Router). The senior line: *accounts are the
  security and cost boundary — use them liberally.*
- **Identity is the real perimeter.** Federated SSO, **workload identity / OIDC** so workloads get
  short-lived creds instead of static keys; IRSA (EKS) / Workload Identity (GKE) / managed
  identities (AKS). Cross-account role assumption with least privilege.
- **Cost & data gravity.** **Egress and cross-AZ/region traffic is the silent cost**; data gravity
  dictates where compute lives. Commitments (RIs/Savings Plans/CUDs) for baseline, on-demand/spot
  for peak — the same shape as the GPU commitment math.
- **Multi-cloud, honestly.** Most "multi-cloud" is *portability insurance + best-of-breed*, not
  active-active — the abstraction tax is real (lowest-common-denominator services, doubled
  operational surface). Be able to argue *when it's worth it* (GPU capacity/price arbitrage across
  neoclouds is a legitimate driver) and when it's résumé-driven.
- **GPU offerings per cloud (the differentiator here).** Hyperscaler GPU instances +
  Capacity Blocks/reservations vs **neoclouds** (CoreWeave/Lambda/Nebius) at lower $/GPU-hr but
  thinner SLAs and IB-included-or-not caveats. Know the fabric question (is InfiniBand/rail-
  optimized networking part of the offer?) — ties to the GPU cost + networking modules.

## Self-quiz

- What's the actual cost of a cross-AZ vs cross-region hop, and how does it change your data-tier
  placement?
- Workload identity vs static access keys — the exact mechanism that gives a pod short-lived creds.
- When is multi-cloud worth the abstraction tax, and when is it a trap?
- Why might a GPU workload pick a neocloud over a hyperscaler — and what do you give up?

## Refresh only if

- **Per-cloud GPU capacity mechanics** (AWS Capacity Blocks, GCP DWS/reservations, the neocloud
  contract model) if your last GPU-procurement mental model predates the 2024-26 capacity crunch.
- **The specific managed-K8s GPU integrations** (EKS/GKE/AKS device-plugin + driver stories) if
  you've only run one of the three.
