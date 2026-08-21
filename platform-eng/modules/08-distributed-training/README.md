---
id: "08"
title: "Distributed training infrastructure"
notion: "https://app.notion.com/p/3b33abaeb82381619b81cc630fdf948c"
phase: "Phase 4 · Months 12–16 (deferrable stretch)"
effort: "~58 hrs ≈ 5 weeks @ 10–12 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["06"]
unlocks: []
started: null
completed: null
---

# 🧮 08 — Distributed training infrastructure

> **Goal.** Support training workloads competently — you don't need to be an ML
> engineer; you need to understand what they run and **why it fails**. The module's
> real subject is **failure-handling as the dominant term in effective throughput**.

- **Notion page:** https://app.notion.com/p/3b33abaeb82381619b81cc630fdf948c
- **Phase:** Phase 4 · **deferrable stretch** (do it if targeting a training-heavy shop) · **Est. effort:** ~58 hrs ≈ 5 weeks
- **Deliverable:** [Survive-a-failure lab](practice/survive-a-failure/) — kill a worker,
  recover from checkpoint, and **price the failure overhead**.

## Why this module, and to what bar

The **Llama-3 fleet story is the spine**: 16,384 H100s, **~1 interruption every 3 hours**
over 54 days, **~78% hardware-caused**, yet **>90% effective training time** — achieved
only through fast checkpoint/restart, NCCL hang diagnosis, and automated node draining.

- **NVIDIA** — *Sr ML Platform Engineer*: "deep understanding of distributed training (data/model parallelism, **NCCL**)… operating ML platforms at scale… operational reliability." (Asks for *understanding*, not model design — the platform-vs-ML-eng line.)
- **CoreWeave** — *Sr SWE II, Applied Training*: "Kubernetes-native research cluster platform… checkpoint-triggered evaluation, cross-cluster scheduling," Slurm/Ray. (**SUNK** runs >100,000 GPUs, jobs >32,000 GPUs each.)
- **Anthropic** — *RE, ML Infrastructure (Pre-training)*: "high-throughput, **fault-tolerant** system design."
- **Interview probes:** *"job hangs at 100% GPU, no error — debug it"* (NCCL/straggler) · *"walk the NCCL debugging sequence"* · *"optimal checkpoint frequency given a failure rate"* (Young/Daly) · *"a slow data loader idles the GPU — find it."*

## Calibrated to your background — what we skip

You did 02b (NVLink/NVSwitch/PCIe topology, GPUDirect, rail alignment), 08's NCCL sits on
that; 06 (gang scheduling, topology-aware placement); 05 (XID/DCGM); 01b (kernel net). So
we **reference, not re-teach**: *"06 places the gang; 08 keeps it alive; 05's XID is the
signal that triggers a restart."* And we **skip ML-eng entirely** — no model code,
architectures, optimizer math, or kernel authoring.

## Lessons

Anchored on **NCCL debugging** (L2); the throughput lever is **checkpointing** (L4); ends in the cost capstone.

| # | Lesson | Hrs | Reliability/cost decision |
|---|--------|-----|---------------------------|
| 01 | [Parallelism strategies: network and memory footprint](lessons/01-parallelism-strategies.md) | 7 | which collective, over which link, memory saved |
| 02 | [**NCCL collectives: topology, transport, and the silent hang**](lessons/02-nccl-collectives.md) (anchor) | 9 | minutes-to-detect a hang vs hours of gang burn |
| 03 | [Communication as the bottleneck](lessons/03-communication-bottleneck.md) | 7 | is the run comms-bound? MFU as report card |
| 04 | [**Checkpointing** (the throughput lever)](lessons/04-checkpointing.md) | 9 | interval sets effective time + $ wasted per failure |
| 05 | [Failure & elasticity](lessons/05-failure-and-elasticity.md) | 7 | auto-drain-and-restart vs babysitting |
| 06 | [Job orchestration (PyTorchJob / Trainer v2)](lessons/06-job-orchestration.md) | 6 | how the gang is expressed + re-rendezvous |
| 07 | [Data pipeline (starvation)](lessons/07-data-pipeline.md) | 6 | a $40/hr GPU idling on JPEG decode is a cost bug |
| 08 | [**Training economics** (capstone → 11)](lessons/08-training-economics.md) | 7 | cost per successful run incl. failure overhead |

Total ≈ **58 hrs ≈ 5 weeks** (lean; merge L5+L6 if time-pressed). Spine = L2 + L4 + L8.

## Resource spine

- **Llama 3 paper §3.3** — the outstanding real-world fleet reliability account (the anchor).
- **NCCL env-var + troubleshooting docs** (v2.30.x) — `NCCL_DEBUG`, `SUBSYS`, `ALGO`, RAS.
- **PyTorch torchrun fault-tolerance tutorial** — the hands-on core; **torchft** as the frontier.
- **ZeRO abstract + FSDP2 docs** (memory model, skim); **Megatron README** (MFU yardstick).
- **Kubeflow Trainer** (PyTorchJob → **TrainJob v2** direction); **CoreWeave SUNK** for the target-company pattern.

> ⚠️ Kubeflow v1 Training Operator / PyTorchJob is unifying into **Trainer v2's `TrainJob`**
> API — teach PyTorchJob as the deployed reality, name TrainJob v2 as the direction. FSDP2
> has superseded FSDP1.

## Deliverable & checkpoint

- Build the **[Survive-a-failure lab](practice/survive-a-failure/)** on 2 rented GPUs: run a
  DDP job, add checkpoint/resume, **kill a worker with and without checkpointing**, capture
  one `NCCL_DEBUG=INFO`, and compute **cost-per-successful-run including failure overhead**
  (a Module 11 input).
- The [**checkpoint**](checkpoint.md) is the gate — debug a hang, compute the optimal
  checkpoint interval, demonstrate killed-worker recovery, and price the failure overhead.

## How to work this module

1. Deep-read Llama-3 §3.3 first — it's the framing for everything.
2. Concentrate GPU rental into the checkpoint/failure weeks; ~$20–40 total on 2 GPUs.
3. Answer the [checkpoint](checkpoint.md) from memory; flip `status` and update Notion when
   the lab + cost-per-run figure exist.
