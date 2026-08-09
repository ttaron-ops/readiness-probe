---
id: "11"
title: "GPU cost and unit economics"
notion: "https://app.notion.com/p/3b33abaeb8238190b01ce1117b80d2b1"
phase: "Phase 3 · Months 8–12 (ongoing, the signature module)"
effort: "~34 hrs ≈ 3–4 weeks @ 10–12 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["04", "05", "07", "08", "10"]
unlocks: ["12"]
started: null
completed: null
---

# 💰 11 — GPU cost and unit economics

> **Goal.** Your single biggest differentiator. Anyone can run a GPU; almost nobody can say
> *what a GPU-hour bought* — attributed to a tenant, split into allocated vs utilised, and
> turned into $/token or $/run. This module makes that your signature, grounded in the
> **source code of the tools that get it wrong** and the **industry cost standard (FOCUS)**.

- **Notion page:** https://app.notion.com/p/3b33abaeb8238190b01ce1117b80d2b1
- **Phase:** Phase 3 · **the signature module** · **Est. effort:** ~34 hrs ≈ 3–4 weeks
- **Prerequisites:** `04` `05` `07` `08` `10` · **Unlocks:** `12`
- **Deliverable:** [GPU cost synthesis](practice/gpu-cost-synthesis/) — the finished
  `gpu-cost-operator` + "where every tool fails" writeup + FOCUS-aligned schema.

## Why this module, and to what bar

Cost ownership is the platform-engineer skill that GPU-fleet operators pay most for, because
the money is enormous and the tooling is immature:

- **Neoclouds & AI platforms** (CoreWeave, Lambda, Nebius, Together, Modal) live and die on
  **$/GPU-hour** and utilisation — a platform engineer who can attribute and defend unit cost
  is directly load-bearing on gross margin.
- **Big-tech ML-infra teams** run internal **showback/chargeback** on shared accelerator pools;
  "who owns this GPU-hour and was it useful?" is an unsolved-in-practice question.
- The leading OSS tool (**OpenCost/Kubecost**) computes GPU cost as **request-based whole-GPU
  allocation** — blind to MIG fractions, time-slicing, and actual utilisation. Being able to
  name that gap *from its source* is a rare, high-signal credential.
- **Interview probes:** *"bill 4 pods time-slicing one A100"* (correct answer: you can't
  accurately — here are the approximations) · *"design a cost schema for a shared GPU fleet"*
  (→ FOCUS + a utilisation ledger) · *"OpenCost says team A spent $X — is it right?"*

## Calibrated to your background — what we skip

You already built the **per-pod attribution mechanism** (04), the **SM_ACTIVE-vs-GPU_UTIL
util-lie** (05), **cost-per-token** (07), **cost-per-run + MFU** (08), and the
**capex-vs-cloud model** (10); you're FinOps-certified on rates/commitments/tagging. So this
module does **not** re-teach any of that — it's the **synthesis**: attribution *theory* (the
four sharing regimes and what's provably unattributable), where the *tools fail* (OpenCost
source), and the *industry schema* (FOCUS 1.x + the ledger it lacks).

## Lessons

Anchored on **attribution theory** (L1) and closing on the **FOCUS schema** (L10 capstone).

| # | Lesson | Hrs | Decision it settles |
|---|--------|-----|---------------------|
| 01 | [**Attribution: four sharing regimes**](lessons/01-attribution-models.md) (anchor) | 4 | who owns a GPU-hour under exclusive/MIG/time-slice/DRA |
| 02 | [Allocated vs utilised: the two ledgers](lessons/02-allocated-vs-utilised.md) | 2 | which ledger the bill vs the waste lives on |
| 03 | [Idle detection & false-positive cost](lessons/03-idle-detection.md) | 3 | when a low-SM GPU is safe to reclaim |
| 04 | [Fragmentation: unschedulable GPUs](lessons/04-fragmentation-cost.md) | 3 | the cost of free-but-unusable capacity |
| 05 | [Unit economics: $ ↔ app counters](lessons/05-unit-economics.md) (synthesis) | 4 | $/GPU-hr → $/token, $/run, $/experiment |
| 06 | [Commitment & procurement strategy](lessons/06-commitment-strategy.md) | 3 | commit / spot / own the baseline vs peak |
| 07 | [Neocloud vs hyperscaler price gap](lessons/07-neocloud-vs-hyperscaler.md) | 3 | decompose & normalize the 3–6× gap |
| 08 | [Chargeback, showback & queue-wait billing](lessons/08-chargeback-showback.md) | 3 | charge allocation, report utilisation |
| 09 | [**Where tooling fails: OpenCost source**](lessons/09-existing-tooling-limits.md) | 4 | name the exact GPU gap, from the code |
| 10 | [**FOCUS 1.x & a GPU cost schema**](lessons/10-focus-spec.md) (capstone) | 4 | the deliverable's schema |

Total ≈ **34 hrs ≈ 3–4 weeks**. **Non-skippable spine:** L1 (regimes), L5 (unit economics),
L9 (OpenCost gap), L10 (FOCUS schema).

## Resource spine

- **DCGM field docs** — SM_ACTIVE / PIPE_TENSOR_ACTIVE / power, the utilisation basis.
- **NVIDIA MIG + time-slicing + DRA docs/KEPs** — the four sharing regimes' mechanics.
- **OpenCost source** (`pkg/costmodel/costmodel.go`, `allocation.go`, `pkg/cloud/`) — read the gap.
- **FOCUS spec** (focus.finops.org, 1.x) — the industry cost/usage schema + split cost.
- **Neocloud vs hyperscaler pricing pages + 2026 GPU TCO posts** — inputs (flag $ as snapshots).

> ⚠️ **Time-sliced attribution is provably unsolvable from driver/DCGM signals alone** (L1) —
> the hardware reports one number per device, not per CUDA context. The honest answer is an
> approximation with error bars, and knowing that is the interview differentiator.

## Deliverable & checkpoint

- Build the **[GPU cost synthesis](practice/gpu-cost-synthesis/)**: (A) the finished
  `gpu-cost-operator` emitting **both ledgers** + a unit cost, degrading honestly on
  time-slicing; (B) the source-grounded **"where every tool fails"** writeup; (C) the
  **FOCUS-aligned schema** with a utilisation extension.
- The [**checkpoint**](checkpoint.md) is the gate — attribute across all four regimes, defend
  the time-slicing limit, name the OpenCost gap from source, and present the FOCUS schema.

## How to work this module

1. The operator runs against DCGM on **any** GPU (even one) or replayed/published metrics — no
   big fleet needed; the writeup and schema are pure analysis.
2. Flag every $/rate as a dated 2026 snapshot; the durable content is the **method**.
3. Answer the [checkpoint](checkpoint.md) from memory; flip `status` and update Notion when all
   three synthesis artifacts exist. This is the portfolio centerpiece you lead with in `12`.
