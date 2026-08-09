---
lesson: 05
title: "GPU system-design drills (P1–P6)"
module: 12
concept: "GPU system-design reps"
status: not-started
est_time: "5 hrs"
artifacts: ["gpu-system-design-drill-log", "p2-answer-skeleton"]
---

# GPU system-design drills (P1–P6)

## Why this matters
You already design distributed systems in your sleep. What a GPU-fleet operator's design loop actually tests is whether you carry the *hardware overlay* into the room: that a GPU is a shared, partitionable, failure-prone, absurdly expensive accelerator whose "utilization" number lies, and whose interconnect is a first-class scheduling constraint. Generic answers (LB, shard, cache, queue) score you as a strong web engineer, not a platform engineer who has run silicon.

The differentiator at Senior/Staff is a behavior, not knowledge: you **volunteer scale, cost, failure, and SLO before the interviewer asks.** Weak candidates wait to be prompted ("have you thought about failure?"). Strong candidates open with "At ~1k GPUs, at ~$2–3/GPU-hr, here's the SLO I'm protecting and the failure modes that dominate." These six prompts are the canonical GPU-infra design surface. Drill them until the volunteering is reflex.

## What's new here
Nothing here is a new *system-design skill*. The new material is a set of six domain-specific rubrics ("probe-axis checklists") plus a scoring discipline. Each prompt below gives you:
- the **axes** an interviewer probes (the checklist you self-score against),
- the **2–3 tradeoffs you must name out loud** to sound senior,
- the **arming modules** — the artifacts you already built that supply the concrete detail.

The meta-skill: turn every axis into a volunteered sentence. Don't wait for "how would tenants be isolated?" — say "For isolation I've got three levers — passthrough, MIG, time-slicing — and I'd pick MIG here because..."

## Core notes

**P1 — Design a multi-tenant GPU platform.** *Arms: 06, 02, 04, 02b.*
Probe-axis checklist:
- **Isolation model** — the spine tradeoff. Passthrough (whole GPU to one tenant) = strongest isolation, worst utilization. **MIG** (hardware partitions, e.g. 7 slices on A100/H100) = better utilization, but you lose some cross-slice performance and there's residual interference. **Time-slicing** (CUDA context swap) = cheapest, highest density, *no* isolation and no memory protection.
- **Hard vs soft multi-tenancy** — hostile tenants (chargeback, external) need hard isolation (MIG/passthrough + separate namespaces/quotas); cooperative internal teams can run soft (time-slicing + quotas).
- **Noisy-neighbor** — the GPU-specific ones: memory-bandwidth contention (HBM is shared even across MIG in some paths), PCIe/NVLink interconnect contention, shared memory-controller and L2.
- **QoS/priority + preemption** — can a high-pri job preempt a low-pri one? At what granularity (pod, gang)?
- **Fair-share quotas** — per-team GPU-hour budgets, borrowing/reclaim.
- **Security boundary** — the shared memory controller and fabric are the real trust boundary; time-slicing shares address space, so it is not a security boundary at all.

Tradeoffs to name: (1) isolation ↔ utilization (the master dial); (2) density ↔ blast radius; (3) hard boundaries ↔ operational simplicity.

**P2 — Design GPU cost attribution / showback / chargeback.** *Arms: 01, 02, 03, 04, 06, 11.* **This is your home-field prompt — volunteer it if given a choice, and open by naming the attribution formula and its flaws before being asked.**
Probe-axis checklist:
- **Telemetry source** — DCGM exporter as a DaemonSet, scraped by Prometheus at `/metrics`, joined to pod/namespace labels for per-pod attribution.
- **Attribution formula** — `team_cost = (team_util / total_util) × gpu_hour_cost`, and you must immediately name its **flaws**: utilization ≠ value (a low-util inference job can be revenue-critical); idle-but-reserved capacity (a team holds a GPU at 5% — do you bill reservation or usage?); the formula rewards busy-looping.
- **Showback vs chargeback** — showback = visibility, no budget enforcement; chargeback = real internal billing, needs defensible numbers and a dispute process.
- **Shared-endpoint splitting** — a multi-tenant inference endpoint serving many teams: split by request count, token count, or GPU-seconds?
- **Virtual tagging** — synthetic cost-allocation tags when infra labels are missing/wrong.
- **FOCUS normalization** — map cloud + on-prem into one FinOps schema so cost is comparable across capex and cloud.

Tradeoffs: (1) utilization-based vs reservation-based billing; (2) accuracy ↔ explainability (teams dispute what they can't understand); (3) showback (adoption-friendly) vs chargeback (behavior-changing).

**P3 — Design a training-job scheduler.** *Arms: 06, 02b, 09.*
Probe-axis checklist:
- **Gang / all-or-nothing scheduling** — the headline. Vanilla K8s schedules pods independently, so a 4-rank job can land 3 ranks running and the 4th Pending → the job **silently hangs at the collective while you bill for 3 idle GPUs.** Gang scheduling makes placement all-or-nothing.
- **Quota borrowing / reclaim** — idle team A capacity lent to team B, reclaimed when A returns (Kueue cohorts).
- **Preemption** — must evict the *whole gang*, not one pod, or you just recreate the partial-placement hang.
- **Fair-share with priority decay** — Slurm multifactor / aging so long-waiting jobs rise.
- **Topology-aware placement** — co-locate ranks on the same NVLink domain / network rail; a job split across rails runs at inter-rail bandwidth.
- **Scheduler choice** — Kueue (quota/borrowing on top of K8s) vs Volcano (batch/gang) vs NVIDIA KAI vs Slurm (HPC-native) vs DRA (dynamic resource allocation, the emerging K8s-native path).

Tradeoffs: (1) gang scheduling ↔ fragmentation/lower utilization; (2) topology-aware packing ↔ scheduling latency; (3) K8s-native (Kueue/DRA) ↔ HPC-native (Slurm) operational models.

**P4 — Design a model-serving / inference platform.** *Arms: 07, 05.* Trap: hand-waving the LLM layer. Interviewers at inference shops probe exactly the part generic candidates skip.
Probe-axis checklist:
- **Prefill vs decode** — compute-bound prompt processing vs memory-bandwidth-bound token-by-token generation; they have different bottlenecks and ideally different batching/hardware.
- **KV-cache management** — PagedAttention (fixed-size blocks, near-zero fragmentation), eviction/swap under memory pressure. KV cache is the real capacity limit, not FLOPs.
- **Continuous / iteration-level batching** — Orca → vLLM: add/remove requests every decode step instead of static batches; the single biggest throughput lever.
- **SLOs** — TTFT (time-to-first-token, prefill latency) and TBT/ITL (inter-token latency, decode smoothness). Name both.
- **Multi-model GPU sharing** — the "Together serves 100 models" prompt: how do you pack many models with cold-start and memory limits (MIG, time-slicing, model swapping)?
- **Autoscaling incl. scale-to-zero / cold start** — Modal-style: cold-start cost (weight load from storage) vs idle-GPU cost; the scale-to-zero tradeoff.
- **Quantization** — FP8/INT8/INT4 as a cost/throughput lever with a quality tradeoff.

Tradeoffs: (1) latency (TTFT/TBT) ↔ throughput (batch size); (2) scale-to-zero cost savings ↔ cold-start latency; (3) quantization cost win ↔ accuracy loss.

**P5 — Design cluster health detection & automated remediation.** *Arms: 08, 04.*
Probe-axis checklist:
- **Failure taxonomy** — immediate (Xid errors, fallen-off-bus, ECC), degraded (thermal throttle, slow node), intermittent (flaky NVLink, transient ECC). Different detection + response for each.
- **Lemon / grey-node detection** — Meta's number: proactive lemon-node removal dropped large-job failure rate from **14% → 4%.** A lemon node passes health checks but repeatedly kills big jobs.
- **Fleet-scale diagnostics** — nightly DCGM diagnostics across the fleet (Azure runs this at ~100k GPUs); you can't `dcgmi diag -r 3` a node mid-job, so you schedule it in maintenance windows.
- **Straggler detection** — one slow rank throttles the whole collective; detect via per-rank step-time outliers.
- **Tiered triage / remediation** — soft (reset/GPU-reset) → medium (cordon+drain+reschedule) → hard (remove from scheduling, RMA).
- **Cordon/drain + reschedule** — must integrate with the gang scheduler so the whole job reschedules cleanly.
- Trap: **"passes idle, fails under load."** Idle health checks are necessary but insufficient; you need load-based diagnostics.

Tradeoffs: (1) detection sensitivity ↔ false-positive node churn; (2) automated remediation ↔ human-in-the-loop for expensive RMA decisions; (3) nightly diag coverage ↔ capacity taken offline.

**P6 — Design the fleet's observability.** *Arms: 05, 01, 03.*
Probe-axis checklist:
- **Metric hierarchy** — utilization → SM occupancy → MFU/goodput → job throughput. You climb the ladder; each rung is closer to business value.
- **Why utilization lies** — `nvidia-smi` util = "a kernel was resident," not useful FLOPs; 100% util can be a memory-bound or spin-waiting kernel.
- **Pipeline** — DCGM-exporter → Prometheus → Grafana, per-pod/per-namespace labels.
- **Per-tenant cardinality** — labeling by tenant×gpu×model explodes Prometheus cardinality; you must plan for it.
- **Alert on goodput regression, not utilization** — util going up is not good news; goodput/MFU dropping is the real alert.
- **NCCL comms vs compute tracing** — split time-in-compute from time-in-collectives to locate network vs kernel bottlenecks.

Tradeoffs: (1) metric fidelity (MFU needs model-level instrumentation) ↔ collection cost/cardinality; (2) alerting on leading indicators ↔ noise; (3) push vs pull at fleet scale.

**Timed-drill protocol.** Run each prompt as a **35-minute rep**: 3 min clarify + volunteer scale/cost/SLO/failure, 25 min design out loud (record yourself), 7 min self-score against that prompt's checklist. Score each axis 0/1/2 (missed / mentioned / defended-with-tradeoff). Target: every axis ≥1 and at least three axes at 2, plus the four "volunteered before asked" dimensions hit in the first 3 minutes. Log scores in `gpu-system-design-drill-log`; re-drill any prompt scoring <70% until two clean reps.

## Worked example
**P2 answer skeleton (your strongest) — a full spoken structure:**

1. **Volunteer the frame (first 90 seconds):** "Assume ~1,000 H100s across a few teams, ~$2–3/GPU-hr, so ~$40–60M/yr of spend — cost visibility is the whole point. My SLO is *attribution accuracy and explainability*: numbers a team lead won't dispute. Dominant failure mode is mis-attribution eroding trust in the whole system."
2. **Telemetry:** DCGM exporter DaemonSet → Prometheus scrape of `/metrics`, joined to pod/namespace/team labels. Emit per-pod GPU-seconds and utilization.
3. **Formula + immediate flaw disclosure:** `team_cost = (team_util / total_util) × gpu_hour_cost`. "But I'd flag three flaws unprompted: util ≠ value, idle-but-reserved capacity, and the formula rewards busy-loops. So I'd bill *reserved* capacity on reservation and *shared* capacity on usage — a hybrid."
4. **Shared endpoints:** for a multi-tenant inference endpoint, split by GPU-seconds attributed via request-level tokens, not raw request count.
5. **Showback → chargeback path:** launch as showback (dashboards, no enforcement) to build trust, add a dispute process, then flip to chargeback once numbers hold up for a quarter.
6. **Normalization:** map on-prem + cloud into FOCUS so capex and cloud GPU-hours are comparable; virtual tags backfill missing labels.
7. **Volunteer scale/failure close:** cardinality risk (tenant×gpu×model) in Prometheus, and the reconciliation job that catches unlabeled "orphan" spend.

Notice: scale, cost, SLO, and failure all appear in step 1 — before any interviewer prompt.

## Practice
- Run one 35-minute timed rep on **each** of P1–P6, recorded, self-scored against the checklist. Two full passes over the week.
- Re-drill your two lowest-scoring prompts to two clean reps (every axis ≥1, three axes at 2).
- Write your **P2 skeleton** into a one-page card you can deliver in 8 minutes cold. Do the same for P4 (your trap prompt — force yourself through prefill/decode + KV cache + continuous batching so you never hand-wave the LLM layer).
- Feed all six polished skeletons into the [GPU platform capstone](../practice/gpu-platform-capstone/README.md) as your design-round appendix.

## Self-check
- In P1, what is the single master tradeoff dial and where do MIG, passthrough, and time-slicing sit on it? **Answer:** Isolation vs utilization. Passthrough = max isolation / min utilization; MIG = middle (hardware partitions, better utilization, some interference/perf loss); time-slicing = max density / zero isolation (shared address space, not a security boundary).
- In P3, why does vanilla Kubernetes cause a training job to silently waste money, and what fixes it? **Answer:** K8s schedules pods independently, so a gang of 4 ranks can get 3 running + 1 Pending; the collective hangs while you bill 3 idle GPUs. Gang / all-or-nothing scheduling (Kueue/Volcano/KAI) fixes it, and preemption must evict the whole gang. **Answer:** Gang scheduling.
- What is the "strong candidate tell" these drills train, and which four things must you volunteer before being asked? **Answer:** Volunteering unprompted; you must surface scale, cost, failure modes, and SLO in the first ~3 minutes rather than waiting for the interviewer to probe them.

## Resources
- Red Hat — multitenant GPU isolation (MIG / time-slicing / passthrough)
- Together — Multi-tenant GPU cluster design for AI-native teams (together.ai/blog/multi-tenant-gpu-cluster-design-for-ai-native-teams)
- CoreWeave — Kueue for GPU batch scheduling (blog)
- AWS — GPU cost attribution in Amazon EKS (aws.amazon.com/blogs/mt/gpu-cost-attribution-in-amazon-eks...)
- Kubernetes GPU scheduling: DRA / KAI / MIG (techplained.com/kubernetes-gpu-scheduling)
- Exponent — ML system design interview guide
- [🎓 12 — Capstone & interview preparation](../README.md)
