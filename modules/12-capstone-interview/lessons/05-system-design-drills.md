---
lesson: 05
title: "GPU system-design drills (P1–P6)"
module: 12
concept: "GPU system-design reps"
status: not-started
est_time: "6 hrs"
prev: "04-flagship-blog-demo.md"
next: "06-debugging-drills.md"
artifacts: ["gpu-system-design-drill-log", "p2-answer-skeleton"]
sources: 11
---

# GPU system-design drills (P1–P6)

## Where this fits
Lesson 04 got your flagship project public and demo-ready — the artifact that proves you *built* something real. That proof answers "have you done this?" but not "can you design this live, cold, under a clock, for a prompt you've never seen?" That's a different muscle: synthesizing everything from modules 01–11 into a fluent, unprompted, 25-minute spoken design under interview pressure. This lesson drills that muscle directly, against the six canonical GPU-infra design prompts.

## Why this matters
You already design distributed systems in your sleep. What a GPU-fleet operator's design loop actually tests is whether you carry the *hardware overlay* into the room: that a GPU is a shared, partitionable, failure-prone, absurdly expensive accelerator whose "utilization" number lies, and whose interconnect is a first-class scheduling constraint. Generic answers (LB, shard, cache, queue) score you as a strong web engineer, not a platform engineer who has run silicon.

The differentiator at Senior/Staff is a behavior, not knowledge: you **volunteer scale, cost, failure, and SLO before the interviewer asks.** Weak candidates wait to be prompted ("have you thought about failure?"). Strong candidates open with "At ~1k GPUs, at ~$2–3/GPU-hr, here's the SLO I'm protecting and the failure modes that dominate." These six prompts are the canonical GPU-infra design surface — the same six questions recur, in near-identical shape, across CoreWeave, Lambda, Together AI, Modal, Nebius, and NVIDIA loops. Drill them until the volunteering is reflex, because at this level the interviewer is scoring your *instincts*, not your ability to eventually arrive at the right answer with prompting.

## What's new here (calibration)
Nothing here is a new *system-design skill*. The new material is a set of six domain-specific rubrics ("probe-axis checklists") plus a scoring discipline. Each prompt below gives you:
- the **axes** an interviewer probes (the checklist you self-score against),
- the **2–3 tradeoffs you must name out loud** to sound senior,
- the **arming modules** — the artifacts you already built that supply the concrete detail.

The meta-skill: turn every axis into a volunteered sentence. Don't wait for "how would tenants be isolated?" — say "For isolation I've got three levers — passthrough, MIG, time-slicing — and I'd pick MIG here because..." Calibrate your depth to seniority: a Senior candidate needs to *land* each axis; a Staff candidate needs to additionally defend the tradeoff against a skeptical follow-up ("why not just always use MIG?") without stalling.

## Core concepts

**P1 — Design a multi-tenant GPU platform.** *Arms: 06, 02, 04, 02b.*
Probe-axis checklist:
- **Isolation model** — the spine tradeoff. Passthrough (whole GPU to one tenant) = strongest isolation, worst utilization. **MIG** (hardware partitions, e.g. up to 7 slices on A100/H100) = better utilization, but you lose some cross-slice performance and there's residual interference. **Time-slicing** (CUDA context swap) = cheapest, highest density, *no* isolation and no memory protection. Newer dynamic-slicing work (Kubernetes + NVIDIA MIG operators) lets you resize MIG partitions without a node reboot — worth naming as the "where this is heading" beat.
- **Hard vs soft multi-tenancy** — hostile tenants (chargeback, external, potentially adversarial) need hard isolation (MIG/passthrough + separate namespaces/quotas); cooperative internal teams can run soft (time-slicing + quotas).
- **Noisy-neighbor** — the GPU-specific ones: memory-bandwidth contention (HBM is shared even across MIG in some paths), PCIe/NVLink interconnect contention, shared memory-controller and L2.
- **QoS/priority + preemption** — can a high-pri job preempt a low-pri one? At what granularity (pod, gang)?
- **Fair-share quotas** — per-team GPU-hour budgets, borrowing/reclaim.
- **Security boundary** — the shared memory controller and fabric are the real trust boundary; time-slicing shares address space, so it is not a security boundary at all — say this explicitly if a prompt implies external/adversarial tenants.
- **Pooled vs isolated tenancy at the cluster level** — a level above per-GPU isolation: do you carve dedicated node pools per tenant (simpler billing, worse utilization) or run one pooled cluster with scheduler-enforced quotas (better utilization, harder blast-radius containment)? Name this as a second, cluster-scoped instance of the same isolation-vs-utilization dial.

Tradeoffs to name: (1) isolation ↔ utilization (the master dial, at both the single-GPU and cluster-pooling level); (2) density ↔ blast radius; (3) hard boundaries ↔ operational simplicity.

**P2 — Design GPU cost attribution / showback / chargeback.** *Arms: 01, 02, 03, 04, 06, 11.* **This is your home-field prompt — volunteer it if given a choice, and open by naming the attribution formula and its flaws before being asked.**
Probe-axis checklist:
- **Telemetry source** — DCGM exporter as a DaemonSet, scraped by Prometheus at `/metrics`, joined to pod/namespace labels for per-pod attribution.
- **Attribution formula** — `team_cost = (team_util / total_util) × gpu_hour_cost`, and you must immediately name its **flaws**: utilization ≠ value (a low-util inference job can be revenue-critical); idle-but-reserved capacity (a team holds a GPU at 5% — do you bill reservation or usage?); the formula rewards busy-looping.
- **Showback vs chargeback** — showback = visibility, no budget enforcement; chargeback = real internal billing, needs defensible numbers and a dispute process.
- **Shared-endpoint splitting** — a multi-tenant inference endpoint serving many teams: split by request count, token count, or GPU-seconds?
- **Virtual tagging** — synthetic cost-allocation tags when infra labels are missing/wrong.
- **FOCUS normalization** — map cloud + on-prem into one FinOps schema (the FinOps FOCUS spec) so cost is comparable across capex and cloud.

Tradeoffs: (1) utilization-based vs reservation-based billing; (2) accuracy ↔ explainability (teams dispute what they can't understand); (3) showback (adoption-friendly) vs chargeback (behavior-changing).

**P3 — Design a training-job scheduler.** *Arms: 06, 02b, 09.*
Probe-axis checklist:
- **Gang / all-or-nothing scheduling** — the headline. Vanilla K8s schedules pods independently, so a 4-rank job can land 3 ranks running and the 4th Pending → the job **silently hangs at the collective while you bill for 3 idle GPUs.** Gang scheduling makes placement all-or-nothing. NVIDIA's open-sourced KAI-Scheduler is a real, citable example that implements gang scheduling plus hierarchical queuing out of the box.
- **Quota borrowing / reclaim** — idle team A capacity lent to team B, reclaimed when A returns. Kueue implements this via **cohorts**: a group of ClusterQueues that share and borrow quota from each other, with reclaim when the lending queue needs its capacity back — know this term, it's the precise mechanism name.
- **Preemption** — must evict the *whole gang*, not one pod, or you just recreate the partial-placement hang.
- **Fair-share with priority decay** — Slurm multifactor / aging so long-waiting jobs rise.
- **Topology-aware placement** — co-locate ranks on the same NVLink domain / network rail; a job split across rails runs at inter-rail bandwidth.
- **Scheduler choice** — Kueue (quota/borrowing on top of K8s) vs Volcano (batch/gang) vs NVIDIA KAI vs Slurm (HPC-native) vs **Dynamic Resource Allocation (DRA)**, the K8s-native structured-parameter allocation path that reached GA in Kubernetes 1.34 — cite this to show you're current, not reciting a 2022 mental model.

Tradeoffs: (1) gang scheduling ↔ fragmentation/lower utilization; (2) topology-aware packing ↔ scheduling latency; (3) K8s-native (Kueue/DRA) ↔ HPC-native (Slurm) operational models.

**P4 — Design a model-serving / inference platform.** *Arms: 07, 05.* Trap: hand-waving the LLM layer. Interviewers at inference shops probe exactly the part generic candidates skip.
Probe-axis checklist:
- **Prefill vs decode** — compute-bound prompt processing vs memory-bandwidth-bound token-by-token generation; they have different bottlenecks and ideally different batching/hardware.
- **KV-cache management** — PagedAttention (the vLLM/SOSP 2023 mechanism: fixed-size blocks, near-zero fragmentation), eviction/swap under memory pressure. **KV cache is the real capacity limit, not FLOPs** — say this sentence explicitly; it's the single most-checked line on this prompt's rubric.
- **Continuous / iteration-level batching** — Orca → vLLM: add/remove requests every decode step instead of static batches; the single biggest throughput lever.
- **SLOs** — TTFT (time-to-first-token, prefill latency) and TBT/ITL (inter-token latency, decode smoothness). Name both.
- **Multi-model GPU sharing** — the "serve 100 models on shared capacity" prompt: how do you pack many models with cold-start and memory limits (MIG, time-slicing, model swapping)?
- **Autoscaling incl. scale-to-zero / cold start** — cold-start cost (weight load from storage) vs idle-GPU cost; the scale-to-zero tradeoff.
- **Quantization** — FP8/INT8/INT4 as a cost/throughput lever with a quality tradeoff.

Tradeoffs: (1) latency (TTFT/TBT) ↔ throughput (batch size); (2) scale-to-zero cost savings ↔ cold-start latency; (3) quantization cost win ↔ accuracy loss.

**P5 — Design cluster health detection & automated remediation.** *Arms: 08, 04.*
Probe-axis checklist:
- **Failure taxonomy** — immediate (Xid errors, fallen-off-bus, ECC), degraded (thermal throttle, slow node), intermittent (flaky NVLink, transient ECC). Different detection + response for each.
- **Lemon / grey-node detection** — proactive lemon-node removal dropped large-job (512+ GPU) failure rate from **14% → 4%**, per Meta's own study of two production clusters (RSC-1 at 16K A100s, RSC-2 at 8K GPUs) over eleven months, with roughly 40 faulty nodes identified at >85% detection accuracy. A lemon node passes health checks but repeatedly kills big jobs.
- **Fleet-scale diagnostics** — nightly DCGM diagnostics across the fleet; you can't `dcgmi diag -r 3` a node mid-job, so you schedule it in maintenance windows.
- **Straggler detection** — one slow rank throttles the whole collective; detect via per-rank step-time outliers. This is now productized in the industry (see Real-world use cases below) — knowing the manual method still matters because it's what you'd build or extend.
- **Tiered triage / remediation** — soft (reset/GPU-reset) → medium (cordon+drain+reschedule) → hard (remove from scheduling, RMA).
- **Cordon/drain + reschedule** — must integrate with the gang scheduler so the whole job reschedules cleanly.
- Trap: **"passes idle, fails under load."** Idle health checks are necessary but insufficient; you need load-based diagnostics.

Tradeoffs: (1) detection sensitivity ↔ false-positive node churn; (2) automated remediation ↔ human-in-the-loop for expensive RMA decisions; (3) nightly diag coverage ↔ capacity taken offline.

**P6 — Design the fleet's observability.** *Arms: 05, 01, 03.*
Probe-axis checklist:
- **Metric hierarchy** — utilization → SM occupancy → MFU/goodput → job throughput. You climb the ladder; each rung is closer to business value.
- **Why utilization lies** — `nvidia-smi` util = "a kernel was resident," not useful FLOPs; 100% util can be a memory-bound or spin-waiting kernel.
- **Pipeline** — DCGM-exporter → Prometheus → Grafana, per-pod/per-namespace labels.
- **Per-tenant cardinality** — labeling by tenant×gpu×model explodes Prometheus cardinality; a naive "add a label per dimension" answer reads as junior. You must explicitly plan for it — recording rules, cardinality budgets, or a separate long-term-storage tier for high-cardinality series.
- **Alert on goodput regression, not utilization** — util going up is not good news; goodput/MFU dropping is the real alert.
- **NCCL comms vs compute tracing** — split time-in-compute from time-in-collectives to locate network vs kernel bottlenecks.

Tradeoffs: (1) metric fidelity (MFU needs model-level instrumentation) ↔ collection cost/cardinality; (2) alerting on leading indicators ↔ noise; (3) push vs pull at fleet scale.

**Timed-drill protocol.** Run each prompt as a **35-minute rep**: 3 min clarify + volunteer scale/cost/SLO/failure, 25 min design out loud (record yourself), 7 min self-score against that prompt's checklist. Score each axis 0/1/2 (missed / mentioned / defended-with-tradeoff). Target: every axis ≥1 and at least three axes at 2, plus the four "volunteered before asked" dimensions hit in the first 3 minutes. Log scores in `gpu-system-design-drill-log`; re-drill any prompt scoring <70% until two clean reps.

## Perspectives

**The interviewer's rubric view.** When an interviewer at a GPU-fleet operator listens to your answer, they are silently running a checklist much like the probe-axis lists above — not judging whether you reach the "correct" architecture (there usually isn't one), but whether you *touch* each axis and defend at least a few with a real tradeoff. "Volunteering" isn't a soft-skills nicety; it is literally what separates a 2 from a 1 on their scoresheet, because a mentioned-but-undefended axis (you say "MIG" but never say why not passthrough) scores lower than a defended one. Reciting the checklist from memory in the right order reads as rehearsed; hitting the same points *because the design genuinely needs them, in whatever order the conversation goes*, reads as reasoning.

**The cost/finance view (P2).** Picture a CFO or FinOps lead reading your attribution formula, not an engineer. They don't care that `team_util / total_util × gpu_hour_cost` is elegant — they care whether it survives a dispute. A team that reserved a GPU and used 5% of it will contest a usage-based bill; a team running a low-utilization but revenue-critical inference endpoint will contest being billed as if idle time were waste. The formula's flaws aren't academic footnotes, they're the exact objections a real finance stakeholder raises in the first review meeting, which is why volunteering them unprompted is the tell of someone who has actually sat in that meeting.

**The fleet-operator's 3am view (P3/P5).** Gang-scheduling failures and lemon nodes don't show up as design diagrams at 3am, they show up as a page: a 512-GPU job stuck at 60% progress for six hours, or a training run that dies every time it lands on rack 14. The on-call engineer doesn't care about the elegance of your quota-borrowing cohort hierarchy; they care whether your design gives them a fast, cheap signal (per-rank step-time outliers, a repeat-offender node log) that turns "mysteriously stuck job" into "cordon node X, reschedule." Designing P3/P5 with that on-call reader in mind — not just the steady-state throughput reader — is what separates a scheduler design that merely works from one that's operable.

**The ML-researcher's view (P4).** The person actually running inference workloads on your platform thinks in prefill/decode, KV cache, and tokens/sec — not "GPU allocation." If your P4 answer stays entirely at the platform layer (autoscaling, load balancing, health checks) and never descends into why KV cache — not FLOPs — is the real capacity ceiling, you've designed a platform an ML researcher would immediately distrust, because it optimizes the wrong resource. Naming KV cache as the binding constraint is what proves you'd actually be a good infra partner to the researchers, not just a generic ops layer bolted underneath them.

## Real-world use cases

- **Together AI — multi-tenant GPU cluster design for AI-native teams.** https://www.together.ai/blog/multi-tenant-gpu-cluster-design-for-ai-native-teams — shows how a real GPU-cloud provider frames pooled-vs-isolated tenancy tradeoffs at cluster scale, directly informing P1's cluster-level isolation-vs-utilization dial.
- **NVIDIA — open-sourcing the Run:ai scheduler (KAI-Scheduler).** https://developer.nvidia.com/blog/nvidia-open-sources-runai-scheduler-to-foster-community-collaboration/ — a real, production-derived OSS scheduler implementing gang scheduling and hierarchical queuing, giving P3 a concrete named system beyond Kueue/Volcano/Slurm.
- **CNCF — understanding Dynamic Resource Allocation in Kubernetes.** https://www.cncf.io/blog/2026/07/01/understanding-dynamic-resource-allocation-in-kubernetes/ — covers DRA's path to GA in Kubernetes 1.34, the emerging K8s-native mechanism for structured GPU/accelerator allocation that P3 should name as the direction the ecosystem is moving.
- **AWS — GPU cost attribution in Amazon EKS.** https://aws.amazon.com/blogs/mt/gpu-cost-attribution-in-amazon-eks-using-amazon-managed-service-for-prometheus-amazon-managed-grafana-and-opentelemetry/ — a concrete, production reference architecture (DCGM + Prometheus + Grafana + OpenTelemetry) for exactly the P2/P6 telemetry pipeline described above.

## Worked example
**P2 answer skeleton (your strongest) — a full spoken structure:**

1. **Volunteer the frame (first 90 seconds):** "Assume ~1,000 H100s across a few teams, ~$2–3/GPU-hr, so ~$40–60M/yr of spend — cost visibility is the whole point. My SLO is *attribution accuracy and explainability*: numbers a team lead won't dispute. Dominant failure mode is mis-attribution eroding trust in the whole system."
2. **Telemetry:** DCGM exporter DaemonSet → Prometheus scrape of `/metrics`, joined to pod/namespace/team labels. Emit per-pod GPU-seconds and utilization.
3. **Formula + immediate flaw disclosure:** `team_cost = (team_util / total_util) × gpu_hour_cost`. "But I'd flag three flaws unprompted: util ≠ value, idle-but-reserved capacity, and the formula rewards busy-loops. So I'd bill *reserved* capacity on reservation and *shared* capacity on usage — a hybrid."
4. **Shared endpoints:** for a multi-tenant inference endpoint, split by GPU-seconds attributed via request-level tokens, not raw request count.
5. **Showback → chargeback path:** launch as showback (dashboards, no enforcement) to build trust, add a dispute process, then flip to chargeback once numbers hold up for a quarter.
6. **Normalization:** map on-prem + cloud into the FinOps FOCUS spec so capex and cloud GPU-hours are comparable; virtual tags backfill missing labels.
7. **Volunteer scale/failure close:** cardinality risk (tenant×gpu×model) in Prometheus, and the reconciliation job that catches unlabeled "orphan" spend.

Notice: scale, cost, SLO, and failure all appear in step 1 — before any interviewer prompt.

## Practice
- Run one 35-minute timed rep on **each** of P1–P6, recorded, self-scored against the checklist. Two full passes over the week.
- Re-drill your two lowest-scoring prompts to two clean reps (every axis ≥1, three axes at 2).
- Write your **P2 skeleton** into a one-page card you can deliver in 8 minutes cold. Do the same for P4 (your trap prompt — force yourself through prefill/decode + KV cache + continuous batching so you never hand-wave the LLM layer).
- For P3, practice stating the Kueue cohort/borrowing mechanism by name (not just "quota borrowing") — precision on the term is a cheap, high-signal win.
- Feed all six polished skeletons into the [GPU platform capstone](../practice/gpu-platform-capstone/README.md) as your design-round appendix.

## Common pitfalls
- **Reciting the isolation-mechanism list instead of positioning it per-prompt.** Naming MIG/time-slicing/passthrough as a memorized fact-dump, with no connection to the specific tenancy scenario in front of you, reads as rehearsed. The fix: always land on *which one you'd pick here and why*, not just that you know all three exist.
- **Explaining gang scheduling mechanically but never volunteering its cost consequence.** It's common to correctly describe why K8s pods land partially and hang — and then stop, without saying the sentence that actually matters to a platform-cost-conscious interviewer: that you're billing for idle GPUs while the job hangs. State the dollar consequence, not just the mechanism.
- **Defining KV cache correctly but never naming it as *the* capacity constraint.** Many candidates can describe PagedAttention accurately yet still answer capacity-planning questions in FLOPs. Say explicitly that KV cache — not FLOPs — is usually what runs out first in serving.
- **Answering P6 with "add more dashboards."** Proposing broader per-tenant/per-model/per-GPU labeling without acknowledging the Prometheus cardinality cost of that cross-product is a common trap; a senior answer names the cardinality budget problem and a mitigation (recording rules, sampling, a separate high-cardinality store) in the same breath as proposing the labels.
- **Treating the six prompts as independent silos.** In a real loop, a P1 answer that never touches cost (P2) or a P3 answer that never touches observability (P6) misses the credit for synthesis; the strongest candidates cross-reference their own answers ("as I said in the isolation discussion...").

## Self-check
- In P1, what is the single master tradeoff dial and where do MIG, passthrough, and time-slicing sit on it? **Answer:** Isolation vs utilization. Passthrough = max isolation / min utilization; MIG = middle (hardware partitions, better utilization, some interference/perf loss); time-slicing = max density / zero isolation (shared address space, not a security boundary).
- In P3, why does vanilla Kubernetes cause a training job to silently waste money? **Answer:** K8s schedules pods independently, so a gang of 4 ranks can get 3 running + 1 Pending; the collective hangs while you bill for 3 idle GPUs.
- In P3, what fixes the silent-hang problem, and what is the precise name for Kueue's idle-capacity-sharing mechanism? **Answer:** Gang / all-or-nothing scheduling (Kueue/Volcano/KAI) fixes it, with preemption evicting the whole gang, not one pod; Kueue's mechanism for sharing idle quota across teams is called a **cohort** (a group of ClusterQueues that borrow from and reclaim quota with each other).
- In P4, what should you name as the real capacity constraint in LLM serving, and why does it matter more than FLOPs? **Answer:** KV cache. Memory to hold the growing key/value cache per active sequence, not compute, is typically what limits how many concurrent requests a serving system can hold — which is why PagedAttention-style block management is the throughput lever, not just a raw-FLOPs upgrade.
- What is the "strong candidate tell" these drills train, and which four things must you volunteer before being asked? **Answer:** Volunteering unprompted; you must surface scale, cost, failure modes, and SLO in the first ~3 minutes rather than waiting for the interviewer to probe them.

## Connections & what's next
P1–P6 pull directly on the concrete numbers and artifacts you built across modules 01–11 (module 06's isolation mechanisms, module 02b's scheduler internals, module 07's serving stack, module 08's failure taxonomy, module 04's flagship writeup) — this lesson is where all of that gets fused into fluent, spoken synthesis rather than staying as separate written artifacts. It also feeds forward into lesson 07 (narrating artifacts), where you'll practice presenting the concrete project evidence behind these designs. Next: [06 — Debugging drills](06-debugging-drills.md), which drills the incident/live-terminal round — a harder-to-fake companion skill to the whiteboard design round you just drilled here.

## References & further reading

**Primary sources**
- vLLM PagedAttention paper (SOSP 2023): https://dl.acm.org/doi/10.1145/3600006.3613165
- Kubernetes Kueue official docs — Cohort / quota borrowing: https://kueue.sigs.k8s.io/docs/concepts/cohort/
- FinOps FOCUS specification: https://focus.finops.org/

**Real-world engineering blogs**
- Together AI — Multi-tenant GPU cluster design for AI-native teams: https://www.together.ai/blog/multi-tenant-gpu-cluster-design-for-ai-native-teams
- Together AI — Inside multi-node training: how to scale model training across GPU clusters: https://www.together.ai/blog/multi-node-gpu-training
- CoreWeave — Kueue: a Kubernetes-native system for AI training workloads: https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads
- AWS — GPU cost attribution in Amazon EKS using Amazon Managed Service for Prometheus, Amazon Managed Grafana, and OpenTelemetry: https://aws.amazon.com/blogs/mt/gpu-cost-attribution-in-amazon-eks-using-amazon-managed-service-for-prometheus-amazon-managed-grafana-and-opentelemetry/
- Red Hat Developers — Dynamic GPU slicing on Red Hat OpenShift and NVIDIA MIG: https://developers.redhat.com/articles/2025/10/14/dynamic-gpu-slicing-red-hat-openshift-and-nvidia-mig
- Red Hat — Sharing and caring: how to make the most of your GPUs, part 2 (Multi-Instance GPU): https://www.redhat.com/en/blog/sharing-caring-how-make-most-your-gpus-part-2-multi-instance-gpu
- NVIDIA Developer Blog — NVIDIA open sources Run:ai scheduler (KAI-Scheduler): https://developer.nvidia.com/blog/nvidia-open-sources-runai-scheduler-to-foster-community-collaboration/
- CNCF Blog — Understanding Dynamic Resource Allocation in Kubernetes: https://www.cncf.io/blog/2026/07/01/understanding-dynamic-resource-allocation-in-kubernetes/

**Deeper dives**
- Exponent — ML system design interview guide
- [🎓 12 — Capstone & interview preparation](../README.md)
