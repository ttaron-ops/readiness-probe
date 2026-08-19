---
id: "06"
title: "Scheduling, queueing and capacity"
notion: "https://app.notion.com/p/3b33abaeb82381d9b1baea460826f00e"
phase: "Phase 3 · Months 8–12"
effort: "~75 hrs ≈ 6.5–7 weeks @ 10–12 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["02"]
unlocks: []
started: null
completed: null
---

# 🗓️ 06 — Scheduling, queueing and capacity

> **Goal.** Make a fixed, scarce GPU fleet serve many teams fairly and efficiently.
> This is **where platform engineering meets FinOps — your strongest natural fit**, and
> every scheduling decision here is a cost/fairness decision.

- **Notion page:** https://app.notion.com/p/3b33abaeb82381d9b1baea460826f00e
- **Phase:** Phase 3 · requires 02 · **Est. effort:** ~75 hrs ≈ 6.5–7 weeks
- **Deliverable:** [Kueue setup + per-queue showback](practice/kueue-showback/) — all
  runnable on a **GPU-less kind cluster** with fake resources.

## Why this module, and to what bar

For your target companies this is *mandatory core*, and named in the JDs:

- **Anthropic** — *Sr Staff+ SWE, Kubernetes Platform*: "building or operating cluster schedulers or batch systems (**Kueue, Volcano, Slurm**)… **gang scheduling; topology-aware placement**; NCCL."
- **CoreWeave** — *Principal/Staff Eng, Cluster Orchestration*: "a technical authority on **scheduling, quota enforcement, fairness, pre-emption, and multi-tenant GPU isolation**" across "Kubernetes, Slurm, SUNK, **Kueue**." (CoreWeave runs Kueue in production.)
- **NVIDIA** open-sourced the Run:ai scheduler as **KAI Scheduler** (gang, fractional GPU, consolidation, topology).
- **Interview probes:** *"why does the default scheduler deadlock distributed jobs?"* · *"design queues for 3 research teams + 1 prod service on a fixed 128-GPU fleet"* · *"explain Kueue ClusterQueue/cohort/borrowing/preemption"* · *"compute effective capacity of a fragmented fleet"* · *"reserved vs on-demand vs spot for accelerators."*

## Calibrated to your background — what we skip

You did 02 (scheduler framework), 02b (NVLink domains/Topology Manager), 04 (GPU
quotas), so we **reference, not re-teach** those. New here: gang/coscheduling, **Kueue
as a system**, Volcano/KAI, fragmentation math, checkpoint-survivable preemption, and
commitment strategy — where your **FinOps** edge lets you out-answer pure SWEs.

## Lessons

Anchored on **Kueue** (L3–L4, the deepest interview surface).

| # | Lesson | Hrs | Cost/fairness decision |
|---|--------|-----|------------------------|
| 01 | [Why the default scheduler fails](lessons/01-why-default-scheduler-fails.md) | 6 | held-but-idle GPUs = pure burn |
| 02 | [Gang / co-scheduling](lessons/02-gang-scheduling.md) | 7 | all-or-nothing admission fixes the deadlock |
| 03 | [**Kueue I — the queueing model**](lessons/03-kueue-queueing-model.md) (anchor) | 10 | quota = the unit you do showback against |
| 04 | [**Kueue II — cohorts, borrowing, preemption, fair-sharing**](lessons/04-kueue-cohorts-borrowing-preemption.md) | 13 | borrow to sweat a fixed fleet without losing floors |
| 05 | [Alternatives — Volcano & KAI](lessons/05-alternatives-volcano-kai.md) | 9 | which scheduler for which fleet shape |
| 06 | [Topology-aware placement](lessons/06-topology-aware-placement.md) | 9 | topology-blind gang = 2–5× slower all-reduce |
| 07 | [Fragmentation & effective capacity](lessons/07-fragmentation-effective-capacity.md) | 9 | the "$X of hidden capacity" story |
| 08 | [Priority/preemption + capacity economics](lessons/08-priority-preemption-capacity-economics.md) | 12 | commitment laddering; why you can't autoscale GPUs |

Total ≈ **75 hrs ≈ 6.5–7 weeks** (+ ~12 hrs capstone). Spine = L3–L4 + the capstone showback.

## Resource spine

- **Kueue docs** (Concepts: ClusterQueue / Cohort / Preemption / Fair Sharing) + **Kueue
  source** (a clean Go controller — read as a module-01/02 exercise).
- **scheduler-plugins coscheduling** docs for gang; **Volcano** docs + NVIDIA's Volcano
  fragmentation blog; **NVIDIA KAI Scheduler** repo.
- **Kueue TAS** for topology; **SemiAnalysis** rental-capacity for the commitment thesis.

> ⚠️ GPU $/hr figures are snapshots — the durable lesson is the committed-vs-spot
> divergence. Kueue TAS is beta-default (v0.14); the TAS API's storage version is now
> **v1beta2** (`v1beta1` is still served but deprecated) — verify feature gates and API
> version against your installed release.

## Deliverable & checkpoint

- Build **[Kueue setup + per-queue showback](practice/kueue-showback/)** on kind with fake
  resources: 2+ ClusterQueues in a cohort with borrowing + demonstrated preemption, a gang
  deadlock-then-fix, a **per-ClusterQueue showback report** (extends `gpu-cost-operator`),
  and a **128-GPU queue-design + fragmentation + commitment-ladder doc**.
- The [**checkpoint**](checkpoint.md) is the gate — construct the deadlock and fix, explain
  Kueue cold, compute effective capacity, design the 128-GPU queues, and build the commitment mix.

## How to work this module

1. Front-load L3–L4 (Kueue is the anchor and the bulk of interview value).
2. Everything runs on kind with fake `nvidia.com/gpu` resources — no real GPUs needed.
3. Answer the [checkpoint](checkpoint.md) from memory; flip `status` and update Notion when
   the Kueue setup + showback report + 128-GPU doc are done.
