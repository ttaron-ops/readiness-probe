---
id: "10"
title: "Bare metal and cluster lifecycle"
notion: "https://app.notion.com/p/3b33abaeb823817fb539ecac9645878f"
phase: "Phase 4 · Months 12–16 (deferrable stretch)"
effort: "~69 hrs ≈ 6–7 weeks @ 10–12 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["02b"]
unlocks: []
started: null
completed: null
---

# 🖥️ 10 — Bare metal and cluster lifecycle

> **Goal.** Close your biggest structural gap: your Kubernetes experience is **managed**
> (EKS/GKE); GPU fleets frequently run on **bare metal where the control plane is yours**.
> This module makes it yours — and its capstone (capex-vs-cloud) is your FinOps edge.

- **Notion page:** https://app.notion.com/p/3b33abaeb823817fb539ecac9645878f
- **Phase:** Phase 4 · **deferrable stretch** · **Est. effort:** ~69 hrs ≈ 6–7 weeks (cheap VMs)
- **Deliverable:** [Capex-vs-cloud model + KTHW/etcd writeup](practice/capex-vs-cloud/) — two
  finishable-on-VMs artifacts.

## Why this module, and to what bar

The bare-metal / cluster-lifecycle layer is *core competency* at every neocloud:

- **NVIDIA** — *Sr SSE, Kubernetes Node Lifecycle, DGX Cloud*: "build and refine **CAPI providers** for NVIDIA Kubernetes Engine… scalable node provisioning" (Cluster API in Go against bare metal).
- **CoreWeave** — *Kubernetes Platforms*: "provisioning bare-metal and virtual clusters with **Cluster API**… automated tooling to provision control planes on bare-metal (Python/Go/operators)… **day-2 lifecycle**, fault-tolerant architectures." (CKS runs "directly on bare metal, without a hypervisor.")
- CoreWeave's thesis: *"faulty nodes can take up to a month before failure detection"* — quantifying why automated remediation matters.
- **Interview probes:** *"bootstrap a control plane by hand — name every cert"* · *"etcd quorum 3 vs 5; DB hits quota at noon, apiserver goes read-only — recover it"* · *"driver+firmware rollout across 200 nodes"* · *"at what utilisation does owning beat renting?"*

## Calibrated to your background — what we skip

You did 02 (control-plane anatomy, **etcd's role**), 04 (GPU Operator, driver rollout),
05 (XID/NPD concepts), and have on-prem/PXE/colocation fluency. So we **reference, not
re-teach** those and pivot from **role → operations** ("you know what etcd *is*; now you
own its disk, its quorum, and its 2am page"). No FinOps basics — you're certified.

## Lessons

Anchored on **etcd operations** (L2, the thing that pages you); capstone = **capex-vs-cloud** (your edge).

| # | Lesson | Hrs | Decision |
|---|--------|-----|----------|
| 01 | [Cluster provisioning: build one by hand](lessons/01-cluster-provisioning.md) | 10 | name every cert; KTHW then kubeadm-diff |
| 02 | [**etcd operations**](lessons/02-etcd-operations.md) (anchor) | 12 | RTO of the whole control plane |
| 03 | [Control-plane HA & upgrades](lessons/03-control-plane-ha.md) | 8 | stacked vs external etcd; VIP; skew order |
| 04 | [Declarative fleets: Cluster API + Talos](lessons/04-declarative-fleets-capi-talos.md) | 8 | hand-built → declarative fleet |
| 05 | [Node provisioning: PXE → image → firmware](lessons/05-node-provisioning-pxe.md) | 7 | netboot-to-ready pipeline |
| 06 | [Hardware health, remediation & RMA](lessons/06-hardware-health-remediation-rma.md) | 7 | close the NPD → drain → RMA loop |
| 07 | [Storage for AI](lessons/07-storage-for-ai.md) | 7 | keep expensive GPUs fed |
| 08 | [**Capex-vs-cloud economics** + LB](lessons/08-capex-vs-cloud-and-lb.md) (capstone) | 10 | build-vs-buy; the CFO conversation |

Total ≈ **69 hrs ≈ 6–7 weeks** (enriched depth adds ~2–3 weeks over the original estimate; cheap VMs throughout). **Non-deferrable core:** L1 (hand-built), L2 (etcd ops), L8 (capex model).

## Resource spine

- **Kubernetes The Hard Way** (updated 2025) + **kubeadm PKI/HA docs** — build it by hand.
- **k8s "Operating etcd" + etcd maintenance/recovery** — the runbook (note etcd 3.6 `etcdutl` restore).
- **Cluster API Book + Metal3 + Talos** — declarative/immutable fleet provisioning.
- **Node Problem Detector + Cloudflare/AKS remediation** — the health loop.
- **GPU TCO 2026 + neocloud economics** posts — capex-model inputs (flag $ as snapshots).

> ⚠️ etcd 3.6 (k8s 1.33+): snapshot save via `etcdctl`, but **restore moved to `etcdutl snapshot restore`**. Metal3 reached CNCF-incubating (Aug 2025).

## Deliverable & checkpoint

- Build the **[Capex-vs-cloud model + KTHW/etcd writeup](practice/capex-vs-cloud/)**: (A) the
  64-GPU crossover model with your own inputs (the blog nobody's written well), and (B) a
  publishable hand-built-control-plane + etcd break/restore runbook.
- The [**checkpoint**](checkpoint.md) is the gate — provision from scratch, recover etcd
  timed, design HA, close the health loop, and defend the economics.

## How to work this module

1. Everything runs on cheap VMs (KVM/multipass/small cloud VMs) — no real GPUs.
2. If time-boxed, the module reduces to KTHW + etcd break/restore + the economic model and
   still delivers both capstone artifacts.
3. Answer the [checkpoint](checkpoint.md) from memory; flip `status` and update Notion when both artifacts exist.
