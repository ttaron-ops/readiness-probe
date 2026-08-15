---
lesson: 03
title: "Idle GPU detection and the cost of false positives"
module: 11
concept: "Workload-aware idle reclaim"
status: not-started
est_time: "4.5 hrs"
prev: "02-allocated-vs-utilised.md"
next: "04-fragmentation-cost.md"
artifacts: ["Prometheus idle recording rule + alert + weekly idle-$ report line, added to the gpu-cost synthesis deliverable"]
sources: 9
---

# Idle GPU detection and the cost of false positives

> Module: [💰 11 — GPU cost and unit economics](../README.md) · Deliverable: [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md)

## Where this fits

Lesson 02 gave you the two-ledger model: every GPU-hour lives on the **allocated** ledger (what you're billed for — request-based, whole-GPU or fractional) and the **utilised** ledger (what was actually computed), and the gap between the two is where waste hides. That lesson stopped at *naming* the gap. It didn't tell you when the gap is safe to close, who owns it, or what closing it costs if you're wrong.

This lesson supplies the missing piece: a precise taxonomy that splits "allocated but not fully utilised" into three states with three different owners and three different remediations, plus a decision rule for exactly *when* reclaiming that idle capacity saves more than it risks. By the end you can turn "the gap" from lesson 02 into a dollar figure split by state — some of it safe to act on immediately, some of it dangerous to touch, and none of it a single undifferentiated number that just looks alarming.

## Why this matters

"Idle GPU" is the single line item every GPU-fleet cost review opens with, and it is the one most infra teams get wrong in both directions. Under-detect and you burn $3–4/hr per H100 on capacity nobody is using — at 512 GPUs that is roughly $15M/yr of pure waste. Over-detect and reclaim aggressively, and you evict a warm inference replica mid-traffic, tank p99 latency, breach an SLO, and page the on-call. Coreweave, Lambda, and every internal platform team running a shared training/inference fleet lives on this exact tightrope; the ability to articulate *why the naive idle rule is dangerous* is a direct interview signal that you have run one of these fleets rather than read about them.

The trap is that "idle" is not one thing. A GPU with no pod bound, a GPU with a pod that never opened a CUDA context, and a GPU running a live process at 2% SM occupancy are three completely different cost problems with three different owners and three different remediations. Conflating them produces reports that are simultaneously alarming and useless.

The deeper trap is asymmetric cost. Idle detection is a *classifier*, and like any classifier it has false positives. In most FinOps domains a false positive is cheap — you flag a VM as idle, someone checks, it wasn't, you move on. In GPU serving a false-positive reclaim can cost more than the idle it was trying to recover, because warm state (KV cache, loaded weights, compiled kernels) lives in HBM and is expensive to rebuild. This lesson is about making idle detection *workload-aware* so the reclaim decision is an inequality you can defend, not a threshold someone picked in a hurry.

## What's new here (calibration)

- **Skip:** the attribution plumbing (pod-resources API + DCGM → per-pod GPU, mod 04/05) and *why GPU_UTIL lies* (mod 05). You already know `DCGM_FI_DEV_GPU_UTIL` is a "any-SM-touched-in-sample" occupancy flag, not real work. We build directly on that.
- **New:** a precise three-way idle **taxonomy** (unallocated / allocated-idle-reserved / allocated-idle-running) that maps to three different owners and remediations.
- **New:** the **cost-of-false-positive** inequality — treating reclaim as a decision with an expected-value test, not a threshold, because evicting a latency-bound inference pod destroys warm HBM state.
- **New:** turning the idle rule into a **Prometheus recording rule + alert + a dollar-denominated weekly report line**, which is what actually ships to the deliverable.

## Core concepts

### The idle taxonomy — three states, three owners

| State | Definition (signal) | Root cause | Owner | Remediation |
|---|---|---|---|---|
| **(a) Unallocated** | No pod bound to the GPU. Device advertised by device-plugin, `allocatable` but not in any pod's `resources`. | Scheduler can't place / fragmentation / over-provisioned pool | Platform / scheduler | Scale down node, consolidate, fix bin-packing (→ lesson 04) |
| **(b) Allocated-idle-reserved** | Pod bound, GPU held, **no CUDA process** ever started (or exited). No context on the device. | Crashed job holding its slot, stuck init, notebook with a dead kernel | Workload owner | TTL / descheduler evict; requires policy, not just detection |
| **(c) Allocated-idle-running** | Live CUDA process, but **SM_ACTIVE ≈ 0** for a sustained window. | Over-provisioned request, latency-serving between requests, checkpointing, data-stall (→ mod 01b PSI) | Ambiguous — *this is the dangerous one* | Workload-aware; **do not blindly reclaim** |

State (a) is pure stranded capacity: it is a scheduling/fragmentation problem, not a utilisation problem, and it is the subject of lesson 04. State (b) is cheap to detect and cheap to fix — no context on the device means nothing warm to lose, so reclaim is almost always safe (a notebook with a dead kernel is the canonical case). State (c) is where the money and the danger both live, because the same signal — near-zero SM activity on a live process — is produced by both "wasteful over-provisioned job" and "healthy warm inference replica waiting for the next request."

### The DCGM idle rule — signals and thresholds

Use profiling metrics, **not** `DCGM_FI_DEV_GPU_UTIL`. GPU_UTIL reports high whenever *any* SM was scheduled in the sample window, so a memory-bound or single-SM workload reads as busy (mod 05). The real occupancy signal is:

- **`DCGM_FI_PROF_SM_ACTIVE`** — fraction of time at least one warp was resident on an SM, averaged across SMs. This is the primary idle signal. Sustained near-zero = no real compute.
- **`DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`** — fraction of cycles the tensor pipes were active. Gates against "SMs busy but doing trivial work"; for LLM training/inference the tensor pipes carry the FLOPs.
- **`DCGM_FI_DEV_POWER_USAGE`** — board power (W). A truly idle H100 sits near its ~75–100W idle floor vs ~700W TDP; power is a coarse but tamper-resistant cross-check that survives metric-exporter gaps.

**Concrete rule (allocated-idle-running):** a GPU is *idle-candidate* when, over a **rolling 20-minute window**:

```
DCGM_FI_PROF_SM_ACTIVE        < 0.05   (5%)   for ≥ 90% of samples
AND DCGM_FI_PROF_PIPE_TENSOR_ACTIVE < 0.01
AND DCGM_FI_DEV_POWER_USAGE   < 150 W          (near idle floor for H100)
```

The window matters as much as the threshold. Too short (1–2 min) and you flag every inference replica in a traffic trough and every training job across a checkpoint barrier. 15–30 min filters transient troughs while still catching a genuinely-abandoned job within an acceptable waste budget. Tune the window per workload class (see below), not globally.

### The cost of false positives — the money section

A false-positive reclaim is: you classified a *productive-but-low-occupancy* GPU as idle and preempted it. The cost is not zero — it is the cost of rebuilding whatever warm state you evicted, plus any SLO damage.

Two archetypes generate low SM_ACTIVE while being perfectly healthy:

1. **Latency-bound LLM inference.** A serving replica holds model weights and a populated **KV cache in HBM** and spends most wall-clock time *waiting for requests*. SM_ACTIVE between requests is near zero — that is the design, not waste. Reclaim it and you evict warm weights + KV cache; the replacement replica must cold-start: pull/load weights (seconds to minutes for a large model), warm the KV cache, re-JIT/compile kernels, and re-enter the load balancer. During that window you have *fewer* replicas serving, so p99 spikes and you may breach the SLO — the reclaim *caused* the incident it was supposed to prevent.
2. **Training mid-checkpoint / data-stalled.** A training job writing a checkpoint to object storage, or stalled on the data loader (mod 01b PSI: starved loader → idle GPU), shows SM_ACTIVE ≈ 0 for minutes while being entirely healthy. Preempt it and you lose progress since the last checkpoint and pay the requeue.

**The decision inequality.** Reclaim only when the expected saving beats the expected cost of being wrong:

```
Reclaim if:   P(truly-idle) × (GPU_hours_recovered × $rate)
            >  P(false-positive) × (warmup_reload_cost + E[SLO_breach_cost])
```

Where `warmup_reload_cost` = (weights load + KV/cache rebuild + kernel recompile) × $rate, and `E[SLO_breach_cost]` is the expected penalty (credits, error budget burn, lost revenue) of running degraded during the cold-start window. For a batch/training job with frequent checkpoints, `warmup_reload_cost` is small and `E[SLO_breach_cost] ≈ 0`, so the inequality almost always favours reclaim. For a latency-serving replica, the right-hand side is large and the inequality almost never fires from SM_ACTIVE alone.

**Therefore idle detection must be workload-aware.** The classifier's *threshold and window are the same, but the reclaim action is gated on the pod's workload class*, carried as a label:

| Class (label) | Reclaim policy on idle-candidate |
|---|---|
| `workload=batch` / `training` | Reclaim after window; rely on checkpointing. Aggressive. |
| `workload=serving` / `latency` | **Never** reclaim on SM_ACTIVE alone. Scale on request-rate / queue-depth / QPS instead; a serving replica at 0 SM with 0 inflight requests is a *horizontal-autoscaler* decision, not an idle-reclaim decision. |
| `workload=dev` / `notebook` | Reclaim via TTL (idle-timeout), independent of SM — notebooks are the #1 idle sink. |

### Reclamation mechanisms and their safety

- **Descheduler** (`LowNodeUtilization`, custom plugin on the idle rule) — evicts pods off under-used nodes for the scheduler to re-pack. Respects PDBs; safe for batch, dangerous if it evicts serving without an HPA below min-replicas.
- **Preemption** (mod 06 priority/preemption) — a higher-priority job preempts the idle-candidate. Correct *if* the victim is checkpointing batch.
- **Consolidation / bin-packing** (mostAllocated, Karpenter-style consolidation) — drains near-empty nodes to scale the pool down; the primary lever against state (a) unallocated. (→ lesson 04.)
- **TTL on idle dev sessions** — Kubeflow/JupyterHub culler, an idle-timeout controller. The cheapest, highest-ROI reclaim because dev sessions are almost always safe to kill and are chronic idle sinks.
- **MPS / time-slicing — the *pack-more* alternative.** Instead of reclaiming a low-occupancy GPU, co-locate more work on it. MPS (spatial) or time-slicing (temporal) lets several low-SM pods share one physical GPU, converting "idle-running" into "packed." This raises utilisation *without* the eviction risk, and is often the right answer for a fleet of small latency-serving replicas that each sit at 5% SM — you don't reclaim them, you stack them. Anyscale's Replica Compaction (see Real-world use cases) is a production instance of exactly this "pack more, don't blindly reclaim" pattern, applied automatically to serving replicas.

### Turning the rule into observability + a dollar line

Recording rule (per-GPU boolean idle flag, workload-labelled via the pod-attribution join from mod 04/05):

```yaml
groups:
- name: gpu-idle
  rules:
  - record: gpu:idle_candidate:bool
    expr: |
      (avg_over_time(DCGM_FI_PROF_SM_ACTIVE[20m]) < 0.05)
      and (avg_over_time(DCGM_FI_PROF_PIPE_TENSOR_ACTIVE[20m]) < 0.01)
      and (avg_over_time(DCGM_FI_DEV_POWER_USAGE[20m]) < 150)
  - alert: GPUIdleReclaimable
    expr: gpu:idle_candidate:bool == 1 and on(pod) gpu_pod_workload_class{class=~"batch|dev"} == 1
    for: 10m
    labels: { severity: info }
    annotations:
      summary: "Reclaimable idle GPU {{ $labels.gpu }} ({{ $labels.pod }})"
```

Note the alert *only fires for reclaimable classes* — serving pods are excluded so the classifier never recommends a dangerous action. The weekly report line:

```
idle_gpu_hours_week = sum_over_time(gpu:idle_candidate:bool[7d]) / (samples_per_hour)
idle_dollars_week   = idle_gpu_hours_week × $rate_per_gpu_hour
```

Report it **split by taxonomy state** — "unallocated $X (fix: scale down), reserved-idle $Y (fix: TTL), running-idle-batch $Z (fix: reclaim), running-idle-serving $W (informational, not waste)" — so the number is actionable instead of alarming.

## Perspectives

**On-call/SRE perspective.** To a platform engineer looking at a cost dashboard, a false-positive reclaim is a line item — a few hundred dollars of avoidable warmup cost. To the on-call engineer holding the pager, it's an incident: p99 spikes, an SLO burns, and the postmortem traces back to an idle-reclaim job that "helpfully" evicted a warm serving replica during a traffic ramp. The taxonomy and the decision inequality exist specifically so the cost-saving automation and the reliability automation don't fight each other — the reclaim policy has to be legible to whoever owns the error budget, not just whoever owns the cloud bill.

**Workload-class perspective.** Batch/training, latency-serving, and dev/notebook workloads have structurally different idle economics, not just different tolerances. A batch job's idle cost is bounded (checkpoint-and-requeue) and its idle signal is unambiguous over a long enough window. A serving replica's "idle" is often the *entire point* — provisioned for peak, sitting quiet between requests by design — so the same SM_ACTIVE signal means something completely different. A notebook's idle is almost always pure waste with no recovery cost at all. One idle rule cannot serve all three; the class label is not a nice-to-have, it's the variable the whole policy pivots on.

**Hardware/signal perspective.** `SM_ACTIVE`, `PIPE_TENSOR_ACTIVE`, and board power each measure a different layer of "is this GPU doing something," and each is individually gameable or blind in a different way. SM_ACTIVE can be nonzero from a single warp doing trivial pointer-chasing work. Tensor-pipe activity can be zero on a perfectly healthy job that's memory-bound rather than compute-bound (e.g. decode-phase LLM inference). Power can lag or plateau at levels that don't cleanly separate "idle" from "light load" on every SKU. No single signal is a safe idle classifier alone; the rule in this lesson ANDs all three specifically because each closes a blind spot in the others.

**FinOps/dollar perspective.** The taxonomy isn't academic tidiness — each bucket has a different, quantifiable dollar fate. Unallocated capacity converts to savings via a scheduler/consolidation fix with near-zero risk. Reserved-idle converts via a TTL policy with near-zero risk. Running-idle-batch converts via preemption with small, bounded risk (a checkpoint replay). Running-idle-serving mostly *doesn't* convert to savings at all under an idle-reclaim lever — it converts via a completely different lever (HPA, MPS packing) with a different risk profile. A FinOps report that collapses all four into "$X idle" is not just imprecise, it's actively misleading about which dollar is safely recoverable this week.

## Real-world use cases

- **Anyscale — "Introducing Anyscale's New Replica Compaction"** — <https://www.anyscale.com/blog/new-feature-replica-compaction>. A production Ray Serve feature that migrates replicas onto fewer nodes as traffic and cluster size change, rather than treating low-occupancy replicas as candidates for blind reclaim. Anyscale reports a case where instance-seconds declined 3.7% even as traffic (measured in tokens) rose 11.2% over the same period, and states costs can drop 50%+ in less-scaled scenarios. It's a direct production instance of this lesson's "pack more instead of reclaim" alternative (MPS/time-slicing) applied automatically at the replica level.
- **Anyscale — "GPU (In)efficiency in AI Workloads"** — <https://www.anyscale.com/blog/gpu-in-efficiency-in-ai-workloads>. Anyscale's own analysis reports production AI systems commonly running under 50% sustained GPU utilisation even under active load, and traces the causes to exactly the archetypes in this lesson: prefill/decode phases stalling each other in LLM serving, Python dataloaders starving training GPUs, and CPU-heavy pipeline stages bottlenecking GPU-heavy ones. What it shows: "low SM activity" in production is routinely a pipeline-design symptom, not a simple idle-vs-busy binary — reinforcing why a taxonomy (not a single number) is the right frame.
- **CoreWeave — "Why Inference Latency and Availability Drift in Production"** — <https://www.coreweave.com/blog/why-inference-latency-and-availability-drift-in-production>. A named GPU-cloud vendor's engineering post on how inference latency and availability degrade gradually rather than failing loudly — driven in part by request-shape variance and infrastructure sized for average rather than tail load. What it shows: the same fragility that makes aggressive idle-reclaim dangerous for serving (thin headroom, tail-sensitive capacity) is a first-class production concern independent of cost tooling, which is exactly why this lesson insists reclaim on serving pods must route through capacity/autoscaling logic, not a bare SM_ACTIVE threshold.

## Worked example

A 64× H100 shared fleet at an effective $3.20/GPU-hr (on-demand-ish blended). A week's `gpu:idle_candidate:bool` and attribution join produce:

| State | Idle GPU-hrs / wk | $ / wk | Correct action |
|---|---|---|---|
| (a) Unallocated | 1,150 | $3,680 | Scale pool down / consolidate |
| (b) Reserved-idle (dead notebooks, crashed jobs) | 620 | $1,984 | TTL culler + descheduler |
| (c) Running-idle **batch** | 340 | $1,088 | Reclaim via preemption |
| (c) Running-idle **serving** | 890 | $2,848 | **NOT waste** — replicas between requests |
| **Total flagged** | 3,000 | $9,600 | — |

Naive reading: "$9,600/wk, ~28% of the fleet, idle — reclaim it all." Total fleet capacity is 64 × 168 = 10,752 GPU-hr/wk, so 3,000 flagged is 28%.

Workload-aware reading:
- The $2,848 of *serving* idle is **not reclaimable** by this classifier. Those replicas hold warm KV cache; killing them is a false positive. If they're genuinely over-provisioned the fix is the **HPA on QPS/queue-depth**, or **MPS packing** several onto fewer GPUs — not idle-reclaim. Suppose HPA + MPS packing recovers half: ~$1,400/wk, with p99 protected.
- The $3,680 unallocated + $1,984 reserved + $1,088 batch = **$6,752/wk is genuinely and safely recoverable** now: consolidate the empty capacity, TTL the dead notebooks, preempt the checkpointing-tolerant batch idle.

**Cost-of-false-positive check on the serving bucket.** One serving replica, weights + KV cold-start ≈ 90 s, during which the service runs one replica short and p99 crosses the SLO for ~2 min. Say an SLO-breach minute is valued at $40 (credits + error-budget). Reclaiming that replica to "save" 890/ (number of replicas) GPU-hrs is dominated by `E[SLO_breach_cost]` — the inequality says **don't**. The classifier that reported all $9,600 as "waste" would have driven exactly the wrong action on the largest bucket.

Net: the workload-aware report recovers ~$6,752 safely now and ~$1,400 more via autoscaling/packing, while explicitly protecting $2,848 of misclassified serving capacity — versus a naive rule that recommends destroying it.

## Practice

Feeds `../practice/gpu-cost-synthesis/README.md`.

1. **Write the recording rule + alert.** Author `gpu:idle_candidate:bool` with the 20-min SM_ACTIVE/tensor/power rule, joined to workload class via the pod-attribution metric from mod 04/05. Make the alert fire *only* for `batch|dev`.
2. **Classify a synthetic week.** Given (or synthesise) a metrics dump, bucket idle GPU-hours into the four states (a / b / c-batch / c-serving) and produce the `idle_dollars_week` line **split by state** at a stated $rate.
3. **Defend one reclaim decision with the inequality.** Pick one c-serving GPU and one c-batch GPU. Estimate `warmup_reload_cost` (weights load + KV rebuild + kernel recompile, in $) and `E[SLO_breach_cost]` for each, and show the inequality resolving to *reclaim* for batch and *don't* for serving.
4. **Deliverable line:** add a "Idle GPU-hours $ this week (by taxonomy state)" section to the synthesis README, with the recording rule, the split dollar table, and one sentence per state naming owner + remediation.

## Common pitfalls

1. **Using `DCGM_FI_DEV_GPU_UTIL` as the idle signal.** This is the single most common and most damaging mistake. GPU_UTIL reads "busy" whenever *any* SM was touched in the sample window, so memory-bound and single-SM workloads look fully utilised while doing almost no real work (the mod 05 utilisation lie) — and conversely it can mask genuinely idle GPUs that briefly spin. Use `SM_ACTIVE` gated on `PIPE_TENSOR_ACTIVE` and power instead.
2. **Applying one idle threshold/window identically across workload classes.** A serving replica waiting for the next request and a genuinely abandoned batch job look identical on SM_ACTIVE alone. Without the workload-class label, the classifier physically cannot tell them apart — the fix isn't a smarter threshold, it's routing the *action* (not just the detection) through the class.
3. **Reclaiming on a single short sample instead of a sustained window.** A 1–2 minute check misclassifies checkpoint barriers, traffic troughs, and brief data-loader stalls as idle. The 15–30 min sustained window exists specifically to filter these transients while still catching real abandonment.
4. **Treating reclaim as a free action.** For stateful workloads (warm KV cache, loaded weights, compiled kernels), reclaim has a real rebuild cost that can exceed the idle savings it was meant to capture. The decision inequality — weighing `P(truly-idle) × savings` against `P(false-positive) × (warmup + SLO cost)` — is the fix; a bare threshold has no way to account for this.
5. **Conflating "no pod scheduled" (unallocated) with "pod scheduled but idle" (allocated-idle) as one waste bucket.** They have different owners (scheduler/platform vs workload owner) and different remediations (consolidation vs TTL/preemption). Reporting them together produces a number nobody can act on because it's not clear whose job it is to fix.

## Self-check

- Why must idle detection use `DCGM_FI_PROF_SM_ACTIVE` (gated on tensor-active and power) rather than `DCGM_FI_DEV_GPU_UTIL`, and why a rolling 15–30 min window? **Answer:** GPU_UTIL is a coarse "any SM touched in the sample" flag that reads high for memory-bound or single-SM work (the mod 05 utilisation lie), so it under-reports idle. SM_ACTIVE measures actual warp residency; gating on `PIPE_TENSOR_ACTIVE` rejects trivial-but-nonzero SM work and low `POWER_USAGE` is a tamper-resistant cross-check. The 15–30 min window filters transient troughs — inference traffic dips and training checkpoint barriers — that a 1–2 min window would misflag, while still catching a genuinely abandoned job within an acceptable waste budget.
- Give the three-state idle taxonomy and why (c) allocated-idle-running is the dangerous one. **Answer:** (a) *unallocated* — no pod bound, pure stranded capacity, a scheduler/fragmentation problem; (b) *allocated-idle-reserved* — pod holds the GPU but never started a CUDA process (dead notebook, crashed job), safe to reclaim because no warm state exists; (c) *allocated-idle-running* — a live process at ~0 SM_ACTIVE. (c) is dangerous because the identical signal is produced by both a wasteful over-provisioned job *and* a healthy latency-serving replica sitting on a warm KV cache between requests, or a training job mid-checkpoint — so reclaiming on the signal alone risks destroying productive warm HBM state.
- State the reclaim decision inequality and what makes a false positive expensive for a serving pod. **Answer:** Reclaim only if `P(truly-idle) × (GPU_hours_recovered × $rate) > P(false-positive) × (warmup_reload_cost + E[SLO_breach_cost])`. For a serving replica the right side is large because a false-positive reclaim evicts warm weights and KV cache from HBM, forcing a cold start (weights load + cache rebuild + kernel recompile) during which the service runs short of replicas, spiking p99 and burning error budget — so the reclaim can cost more than the idle it targeted, and the correct lever is a QPS/queue-depth HPA or MPS packing, not idle-reclaim.
- Why is MPS/time-slicing framed as the "pack more" alternative to reclaim, and when should you reach for it instead of a reclaim policy? **Answer:** Reclaim assumes the right fix for a low-occupancy GPU is to free it up; packing assumes the right fix is to put more work on it. For a fleet of many small, low-SM replicas (e.g. lightweight serving models each at 5% SM), none of them is individually "idle enough" to safely evict, but co-locating several via MPS (spatial sharing) or time-slicing (temporal sharing) raises aggregate utilisation without any eviction risk. It's the right lever whenever the workload class makes reclaim dangerous (serving) but the per-GPU occupancy is genuinely low — Anyscale's Replica Compaction is a production example of exactly this choice made automatically.

## Connections & what's next

This lesson is the direct sequel to lesson 02's two-ledger model: it takes "the gap between allocated and utilised" and splits it into a taxonomy with owners, remediations, and a defensible reclaim decision instead of one alarming number. It also forward-connects to lesson 08 (chargeback/showback): idle reclaim is the enforcement lever that gives an allocation-based chargeback model teeth — without it, a tenant can hoard allocated-but-idle capacity indefinitely and the bill never reflects it.

Next, lesson 04 takes the *other* half of "free capacity" — GPUs that are unallocated, not idle — and shows why that's a structurally different problem: not a utilisation question but a scheduling/shape question, where the same free-GPU count can be worth $0 or full price depending only on how it's laid out across nodes.

## References & further reading

**Primary sources**
1. **DCGM profiling metrics reference** — <https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html#profiling> — authoritative definitions of `SM_ACTIVE`, `PIPE_TENSOR_ACTIVE`, and the device metrics used in the idle rule.
2. **NVIDIA MPS documentation** — <https://docs.nvidia.com/deploy/mps/index.html> — the pack-more alternative to reclaim for many low-occupancy co-located processes.
3. **Kubernetes descheduler (`LowNodeUtilization`, RemovePodsViolating…)** — <https://github.com/kubernetes-sigs/descheduler> — the reclaim/consolidation mechanism and its PDB/priority safety knobs.
4. **NVIDIA GPU time-slicing on Kubernetes** — <https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html> — temporal sharing as the alternative to eviction.
5. **Prometheus recording & alerting rules** — <https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/> — for authoring `gpu:idle_candidate:bool` and the dollar report line.

**Real-world engineering blogs**
6. **Anyscale — "Introducing Anyscale's New Replica Compaction"** — <https://www.anyscale.com/blog/new-feature-replica-compaction> — production feature that consolidates underused serving replicas instead of blindly reclaiming them; read for the "pack more, don't reclaim" pattern in production.
7. **Anyscale — "GPU (In)efficiency in AI Workloads"** — <https://www.anyscale.com/blog/gpu-in-efficiency-in-ai-workloads> — vendor analysis of why production AI workloads commonly run under 50% GPU utilisation; read for the concrete failure archetypes behind low SM_ACTIVE.
8. **CoreWeave — "Why Inference Latency and Availability Drift in Production"** — <https://www.coreweave.com/blog/why-inference-latency-and-availability-drift-in-production> — GPU-cloud vendor's take on latency/availability drift in inference; read for why serving capacity is more fragile than it looks, reinforcing the false-positive-reclaim danger.

**Deeper dives**
9. **Kubernetes Horizontal Pod Autoscaling** — <https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/> — for going further on the QPS/queue-depth-based autoscaling lever that replaces idle-reclaim for serving workloads.
