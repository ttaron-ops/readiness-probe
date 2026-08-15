---
lesson: "A03.9"
title: "GPU and ML observability at fleet scale"
module: "A-03"
concept: "fleet GPU observability"
status: not-started
est_time: "6 hrs"
prev: "08-profiling-and-ebpf.md"
next: "10-telemetry-lakehouse.md"
artifacts: ["fleet GPU + NCCL signal plan", "goodput-regression burn-rate alert", "straggler-detection query"]
sources: 10
---

# A03.9 · GPU and ML observability at fleet scale

> **Concept.** Take single-node GPU truth (SM occupancy, MFU/goodput, XID) and make it survive thousands of GPUs: bounded cardinality, sharded scrape, per-tenant multitenancy, comms/straggler visibility, and alerts that page on burned GPU-hours instead of a moved gauge.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

This is the capstone lesson — lesson 9 of 9 — and it doesn't introduce a new signal type so much as it forces every discipline from L1–L8 to survive contact with a real fleet. Cardinality budgets (L1/L3) meet DCGM's highest-cardinality label set anywhere in the stack. PromQL correctness (L2) meets gauges that get `rate()`'d by habit. Mimir sharding and multitenancy (L3) meet a workload that can blow a tenant's series budget in one bad rollout. Collector-side enrichment (L4) meets the pod-resources join that turns "GPU 3 is hot" into "team-b's job owns GPU 3." Exemplars (L5), log discipline (L6), burn-rate alerting (L7), and off-CPU/eBPF profiling (L8) all show up again here, applied to the one workload where getting them wrong costs thousands of dollars an hour, not a missed dashboard refresh.

## Why this matters

A single-node GPU dashboard that is *correct* still tells you almost nothing about a 2,000-GPU training run, because a synchronous job runs at the speed of its slowest rank and that rank is invisible in a fleet average. At staff level the failure mode is not "we don't have GPU metrics" — it's "we have DCGM on every node, Prometheus fell over from `gpu_uuid` cardinality, nobody can attribute a hot GPU to a team, and the run lost 18% goodput to one fail-slow NVLink for six hours before anyone paged." This lesson is the fleet-wide synthesis of the whole module: it reuses cardinality discipline (L1/L3), gauge/quantile correctness on DCGM series (L2), Mimir sharding and multitenancy (L3), collector-side enrichment (L4), exemplars into slow steps (L5), NCCL log discipline (L6), goodput burn-rate alerting (L7), and off-CPU/eBPF straggler profiling (L8) — applied at once to GPUs.

## What's new here (calibration)

- Module 05's GPU-observability lesson already covers **what DCGM is**, the **util-lie** (`GPU_UTIL` vs the PROF metrics, exact field IDs), single-node **MFU/goodput** definitions, and per-node **XID** meaning. That's the artifact you built there — this lesson assumes it and never re-derives it.
- New here: the **fleet-scale synthesis** — what breaks when you go from one node's DCGM feed to thousands, how comms (NCCL) observability sees what node-local dashboards structurally can't, and how to detect a **straggler** (a fail-slow rank) rather than just a **failed** one.
- New here: framing GPU observability as a **reliability-engineering problem at scale**, not a monitoring problem — at thousands of GPUs, hardware failure is a statistical certainty within any given training run, so the job is minimizing detection-to-recovery time, not chasing an anomaly.
- New here: the honest state of fleet-scale comms-observability tooling (NCCL Inspector/NIXT, Meta's GCM) — this is actively being built out in public right now, not a solved, textbook-stable domain, and knowing that is itself an interview-credible answer.

## Core concepts

### Collection topology at fleet scale

`dcgm-exporter` runs as a DaemonSet — one exporter per node, scraping the local DCGM host engine over the GPU. A Prometheus `ServiceMonitor`/`PodMonitor` (or native service discovery) auto-discovers new nodes as the fleet grows, so onboarding a rack is zero-touch. Turn on the exporter's **Kubernetes device-plugin mapping** so every GPU metric carries `pod`, `namespace`, `container` derived from the NVIDIA device-plugin allocation — that correlation is what converts "GPU 3 on node-47 is at 95 °C" into "team-b's `llama-serve` pod owns GPU 3." Attribution (L4) is the difference between a metric and an on-call action. Reference dashboards: Grafana IDs **12239** and **15117**.

Kubernetes-plus-device-plugin is not the only attribution pattern worth knowing. Meta's open-sourced **GPU Cluster Monitoring (GCM)** correlates DCGM/NVML hardware telemetry with **Slurm job metadata** at fleet scale (hundreds of thousands of GPUs internally), enriching OTel data with per-job attribution the same way the device-plugin join does — but anchored to a Slurm Job ID instead of a K8s pod/namespace. Large research and HPC-style training fleets frequently run Slurm, not Kubernetes, so the transferable lesson isn't "join to K8s" — it's "join hardware telemetry to whatever the scheduler treats as the unit of job identity."

### Cardinality is the central scaling problem

DCGM series carry `gpu_uuid` (unbounded across a fleet and across RMA churn), `mig_instance`, plus injected `pod` (churny — every rollout mints new series) and any `job_id`. Multiply those and a few dozen base metrics explode into millions of active series; Prometheus OOMs. Staff answer, mirroring L1/L3: keep only **bounded** labels on the time series — `node`, `gpu` (local index 0–7), `tenant`, `model_class` — and drop the unbounded ones (`gpu_uuid`, `pod` hash suffixes, `container_id`) with `metric_relabel_configs` at scrape time. Per-run identity does not belong on metrics; it belongs in **exemplars/traces** (L5) that hang off the metric and link to the specific slow step. Then **hashmod-shard** the scrape across N Prometheus/agent replicas and **remote-write to Mimir** with per-tenant series limits so one team's cardinality blowout can't take down everyone's GPU visibility.

### Reliability framing: failure is a statistical certainty, not an anomaly

At single-node scale, a GPU dying is an incident. At fleet scale, it's a Tuesday. Meta's Llama 3 paper documents this precisely: on a 16,384-H100 cluster over 54 days, there were **466 job interruptions**, of which **419 were unexpected** — and the majority of those were caused by hardware, with a reported breakdown of roughly **GPU (incl. NVLink) ~30%**, **HBM3 memory ~17%**, **GPU SRAM ~4.5%**, **GPU system processor ~4%**, and **network switch/cable ~8.4%** of interruptions. That's roughly one interruption every ~3 hours, continuously, for the whole run. Despite that failure rate, Meta reports **>90% effective training time** — because the observability and automation caught almost all of it: only **3 manual interventions** across the entire 54 days, the rest handled by automated detection and recovery.

Two conclusions follow, and they reset what "GPU observability" is for at this scale:

- **The job is detection-to-recovery time, not failure prevention.** You cannot engineer away hardware failure at 16k+ GPUs; you can only shrink the window between "a GPU/HBM/NIC failed" and "the job resumed on healthy hardware." That window is entirely a function of observability quality — how fast the fleet rollup surfaces the failure and routes the restart.
- **Fail-stop is the easy half of this problem; fail-slow is the hard half.** Automated tooling handling all but 3 interruptions over 54 days implies **fail-stop** detection — a rank dies outright, a health check or liveness probe catches it — is largely a solved problem at this scale. **Fail-slow** — a rank degrades but keeps running — is the open problem the rest of this lesson's tooling (straggler detection, NCCL comms observability) exists to close.

This also resets how to treat XID/hardware-error events operationally: at roughly one interruption every three hours on a 16k-GPU cluster, XID and thermal/ECC events are a **routine, continuous stream**, not a rare-event category. Alerting on individual XID occurrences doesn't scale; you need a fleet-wide **rollup** correlated against job/goodput impact (below), the same rollup-not-per-event discipline L7's burn-rate alerting teaches for any high-frequency signal.

### Comms observability — what single-node dashboards structurally cannot see

In distributed training, goodput dies in the *collectives* (all-reduce, all-gather, reduce-scatter), not in the kernels a node-local `SM_ACTIVE` gauge watches. A node can show 100% SM occupancy while the whole job is stalled waiting on one slow link. This is a distributed-systems problem with nothing GPU-specific about its shape — it's the same "find the straggler in a synchronous, barrier-bound system" problem as a slow reducer in a MapReduce shuffle or a lagging replica in consensus, just instrumented differently.

Emerging tooling targets exactly this: **NCCL Inspector**, backed by NVIDIA's **NIXT** exporter, injects CUDA events into NCCL calls to time each communication group and surface fail-slow ranks, exporting Prometheus-shaped NCCL collective metrics with reported overhead **under 2%**. Treat that overhead figure the way you'd treat DCGM's own profiling-sampler cost from module 05 — instrumentation is never free, and under-2% is the number to validate against your own workload before trusting it blind. **Validated scale matters too: NIXT's own published validation tops out around ~2,048 GPUs** — meaningfully below this module's 4,000–10,000-GPU design target. Deploying it unmodified at fleet scale is an engineering risk to de-risk with a pilot, not an assumed-solved dependency.

Fleet signals to collect regardless of which exporter you pick: per-collective duration outliers, and USE-method saturation on the **interconnect** itself — NVLink, PCIe, and IB/RoCE bandwidth vs line rate. The interconnect is a resource; apply Utilization/Saturation/Errors to it exactly as you would to a NIC.

### Alert on goodput regression, not utilization

Ties directly to L7. The page-worthy event is "the fleet is burning money," expressed as a **burn-rate** alert on `achieved_tokens_per_sec / expected` (or step-time regression), where the error budget is **wasted GPU-hours**. A gauge moving is not an incident; a synchronous job silently running at 0.7× expected throughput across 512 GPUs is thousands of dollars an hour on fire. Multi-window burn-rate (fast + slow) keeps it from flapping on transient dips.

Don't conflate this with MFU. **MFU measures compute efficiency relative to peak hardware FLOPs** — it's sensitive to model architecture and kernel choice even on fully healthy hardware. **Goodput measures actual throughput against expected, across every loss source** — restarts, stragglers, scheduling gaps, queueing. A run can post excellent MFU while its goodput is terrible, if it simply isn't *running* often enough. Track both; alert on goodput, because goodput is the one that maps directly to wasted spend.

### Training-run observability and straggler detection

Beyond infra metrics, capture the run's own health: loss and grad-norm curves (divergence/NaN detection), and **straggler detection** via per-rank step-time outliers — the one slow GPU or throttled link that gates the whole synchronous step. Recall the fail-stop/fail-slow split from above: fail-stop is caught cheaply by liveness/health checks; fail-slow is caught *only* by goodput/step-time **variance**, never by a health check, because the rank is still alive and still responding — it's just slow. Roll up **XID errors** across the fleet and correlate the rollup with job failures and goodput dips, so a spike in XID 79 (GPU fell off the bus) or ECC/thermal events maps to the runs it actually hurt.

### Operating across thousands of GPUs

The mechanics: scrape sharding + Mimir multitenancy for scale; **recording rules** for fleet rollups so dashboards never fan out over millions of raw series (`fleet:gpu_sm_active:avg`, `fleet:sm_active:avg_by_tenant`, per-tenant goodput); continuous **eBPF profiling** (L8) for host-side and kernel regressions that starve the GPU (a slow dataloader is a CPU-side straggler); and event correlation joining XID/thermal/ECC with goodput. The governing law: a synchronous job runs at the speed of its slowest rank, so **tail and outlier detection matter more than averages** — an average SM_ACTIVE of 90% hides the p99 rank sitting at 40% that is setting everyone's step time.

## Perspectives

**Hardware-reliability.** Meta's Llama 3 data reframes "GPU observability" as fundamentally a reliability-engineering problem at fleet scale, not a monitoring one. Hardware failures (GPU + HBM3) caused the majority of unplanned interruptions on a 16,384-H100 cluster over 54 days. Observability's job in this frame is minimizing detection-to-recovery time for failures that are a statistical certainty at that scale — not preventing an anomaly that, at this size, isn't really an anomaly.

**Distributed-systems.** NCCL collective observability solves a problem with nothing GPU-specific about its shape: finding the straggler in a synchronous, barrier-bound system is the same challenge as a slow reducer in a distributed shuffle or a lagging replica in a consensus protocol. NIXT/NCCL Inspector just happens to instrument it via CUDA event injection instead of RPC timing — the pattern transfers even if you never touch NCCL directly.

**Economics/FinOps.** CoreWeave's public benchmarking — goodput up to 96%, MFU exceeding 50% against a claimed industry baseline of 35–45% — turns observability quality into a direct customer-facing SLA and sales argument in the neocloud market. Goodput isn't an internal engineering nicety here; it's a competitive differentiator a provider puts in a blog post. Read those numbers as **directional marketing, not an audited benchmark** — methodology, model size, and measurement window aren't published, so they're not something to cite as a capacity-planning input for your own fleet.

**Tooling-maturity.** NIXT/NCCL Inspector (2025–2026 vintage, validated only to ~2,048 GPUs) and Meta's GCM (open-sourced 2026) are both very recent. Say so plainly to an interview panel: fleet-scale GPU/NCCL observability tooling is still actively being built out in public, not a solved, textbook-stable domain. That's a legitimate, staff-credible thing to know and say — pretending otherwise is the tell of someone who hasn't actually operated at this scale.

## Real-world use cases

- **Meta, "The Llama 3 Herd of Models"** — https://arxiv.org/pdf/2407.21783 — the primary, heavily-citable source for concrete fleet-scale failure statistics: 466 interruptions (419 unexpected) over 54 days on 16,384 H100s, the hardware-failure breakdown by component, and >90% effective training time achieved via automated recovery (only 3 manual interventions). What it shows: at fleet scale, reliability engineering *is* GPU observability.
- **Meta, "GPU Cluster Monitoring" (GCM), open-sourced** — https://github.com/facebookresearch/gcm — a real, current (2026) open-source tool correlating DCGM/NVML telemetry with Slurm job metadata at fleet scale, enriching OTel data with per-job attribution. What it shows: the attribution *pattern* (join hardware telemetry to scheduler-native job identity) is the transferable lesson — GCM anchors to Slurm Job IDs, not K8s pods, a useful counter-example to a K8s-only mental model.
- **NVIDIA, "Enhancing Communication Observability of AI Workloads with NCCL Inspector"** — https://forums.developer.nvidia.com/t/enhancing-communication-observability-of-ai-workloads-with-nccl-inspector/354225 (associated paper, NIXT: https://arxiv.org/abs/2608.01449) — NVIDIA's own material on the NCCL Inspector plugin/NIXT exporter, validated to ~2,048 GPUs at under 2% overhead. What it shows: comms-level fail-slow detection is possible and cheap to instrument — but flag its recency (a 2026 preprint) and that its published validation sits below this module's 4,000–10,000-GPU design target.
- **CoreWeave, "CoreWeave Drives 20% Higher GPU Cluster Performance"** — https://www.coreweave.com/blog/coreweave-leads-the-charge-in-ai-infrastructure-efficiency-with-up-to-20-higher-gpu-cluster-performance-than-alternative-solutions — reports goodput up to 96% and MFU exceeding 50% (vs a claimed 35–45% industry baseline), attributed to cluster validation, health monitoring, and proactive node replacement. What it shows: goodput as a customer-facing competitive metric — read as vendor marketing, methodology unaudited, directional only.

## Worked example

Context first, with real numbers: at Meta's reported failure rate on a 16,384-GPU cluster (~one interruption every ~3 hours), a 512-GPU job running continuously will see hardware-driven interruptions on a similar order of frequency, scaled by cluster share — this is the backdrop that makes "did we catch it fast" the only question that matters. A four-part fleet plan, worked end to end.

**1. Drop unbounded cardinality at scrape (`metric_relabel_configs`):**
```yaml
metric_relabel_configs:
  - source_labels: [__name__, gpu_uuid]
    regex: "DCGM_.*;.*"
    target_label: gpu_uuid
    replacement: ""          # strip the unbounded UUID off every DCGM series
  - regex: "gpu_uuid|UUID|container_id|pod_hash"
    action: labeldrop        # keep node, gpu, tenant, model_class only
```

**2. Fleet rollup recording rule (dashboards read this, not raw series):**
```yaml
groups:
  - name: gpu-fleet
    rules:
      - record: fleet:sm_active:avg_by_tenant
        expr: avg by (tenant) (DCGM_FI_PROF_SM_ACTIVE)
      - record: fleet:sm_active:p99_by_tenant   # the tail is the story
        expr: quantile by (tenant) (0.99, DCGM_FI_PROF_SM_ACTIVE)
      - record: fleet:xid_errors:rate5m_by_tenant   # rollup, not per-event alerting
        expr: sum by (tenant, xid) (rate(DCGM_FI_DEV_XID_ERRORS[5m]))
```

**3. Goodput-regression burn-rate alert (pages on wasted GPU-hours, not a gauge):**
```promql
(
  sum by (job) (rate(training_tokens_total[10m]))
  / on(job) group_left expected_tokens_per_sec
) < 0.8
```
Fire only when sustained across a fast (10m) and slow (1h) window so a checkpoint pause doesn't page.

**4. Straggler query (find the rank gating a synchronous step):**
```promql
step_time_seconds
  > 1.3 * on(job) group_left() quantile by (job) (0.5, step_time_seconds)
```
Any rank whose step time exceeds 1.3× the job median is a candidate straggler — pivot from that `rank` label into the NCCL exemplar (per-collective duration for that rank) and the eBPF off-CPU profile for that host to distinguish a slow link (comms) from a starved dataloader (host CPU) — exactly the L8 off-CPU workflow, applied to the rank the straggler query just flagged.

## Practice

Feeds the module deliverable directly: **[fleet observability design](../practice/fleet-observability/README.md)**. This lesson is the source of the design doc's **DCGM + NCCL signal plan** section and the alert set's **GPU goodput-regression SLO + straggler alert**. Build the **fleet GPU signal plan**: a table of every GPU/comms signal, its bounded label set, where per-run identity goes (exemplar vs metric), and the scrape-shard/remote-write layout to Mimir — then produce the three artifacts worked above (relabel config, goodput burn-rate alert, straggler recording rule/query) as deployable YAML, plus a one-paragraph note on where NCCL Inspector/NIXT would sit in the topology and what you'd validate before trusting it past 2,048 GPUs.

## Common pitfalls

- **"High fleet-average SM_ACTIVE means the fleet is healthy."** A synchronous job's delivered throughput is gated by its slowest rank — high average is fully compatible with a straggler quietly burning goodput. Always pair an average with a p99/tail query.
- **"XID errors are rare enough that per-XID alerting is sufficient."** At Meta's reported failure rate — roughly one interruption every ~3 hours on a 16,384-GPU cluster — XID/hardware-failure events are a routine, continuous operational stream at fleet scale, not a rare-event category. You need fleet rollup and correlation-with-goodput-dips machinery, not per-event pages.
- **"NCCL Inspector/NIXT is production-mature — deploy it as-is."** The primary source itself states validation only to ~2,048 GPUs. Deploying at 4,000–10,000-GPU scale is explicitly beyond published validation — pilot it first, don't assume it as a solved dependency.
- **"Goodput and MFU are the same metric."** MFU measures FLOPs efficiency relative to peak hardware FLOPs — a compute-efficiency metric sensitive to model/kernel choice even on healthy hardware. Goodput measures actual-vs-expected throughput accounting for *all* loss sources, including restarts, stragglers, and scheduling gaps. A run can have excellent MFU while running rarely — track both, alert on goodput.
- **"Vendor-reported goodput/MFU numbers are apples-to-apples comparable across providers."** Methodology, model size/architecture, and measurement window are usually unstated in marketing material (see CoreWeave's blog above). Treat such numbers as directional, never as a benchmark for your own capacity planning.

## Self-check

- Why does dropping `gpu_uuid` and `pod` from DCGM series not lose you the ability to debug a specific bad GPU? **Answer:** Because bounded labels (`node`, `gpu`, `tenant`) still identify the physical GPU for time-series alerting and rollups, while the high-cardinality per-run identity moves to exemplars/traces that hang off the metric — you click from the metric into the exemplar to reach the exact run/UUID, instead of paying for millions of active series to keep it inline. It's the L1/L3 cardinality trade applied to GPUs.
- A 512-GPU synchronous run shows fleet-average SM_ACTIVE at 91% but throughput is 30% below expected. What is happening and what signal exposes it? **Answer:** Fail-slow: one or a few ranks are degraded (a slow NVLink/IB link or a starved dataloader), and because the step is synchronous every fast rank blocks on the slow one — the average stays high while goodput collapses. The average hides it; a per-rank step-time p99/outlier query and NCCL per-collective duration outliers expose the straggler, which is why tail detection beats averages here.
- Why alert on `achieved_tokens_per_sec / expected` burn-rate instead of on GPU utilization or SM_ACTIVE dropping? **Answer:** Because utilization can be high (or low for benign reasons) while the thing that actually costs money — goodput — regresses, and vice versa. A burn-rate alert whose error budget is wasted GPU-hours pages only when the fleet is genuinely burning money, aligns the signal with business cost, and multi-window smoothing stops it flapping on checkpoints/warmup — the L7 SLO discipline applied to training economics.
- Meta reports only 3 manual interventions across 54 days despite 419 unexpected interruptions on a 16,384-GPU cluster. What does that imply about fail-stop vs fail-slow detection maturity, and why does that matter for where you invest observability effort? **Answer:** It implies fail-stop detection (a rank dies outright, caught by liveness/health checks and automated restart) is largely solved at this scale — automation handled nearly all of it. Fail-slow detection (a rank degrades but keeps responding) is the open problem, because it's invisible to health checks and only shows up as goodput/step-time variance. That's why straggler detection and NCCL comms observability — not more health-check coverage — are where fleet-scale investment should go.
- Why is Meta's GCM anchoring attribution to Slurm Job IDs instead of Kubernetes pod/namespace labels a useful counter-example, not a gap in your design? **Answer:** Because the transferable lesson is the *pattern* — join hardware telemetry to whatever the scheduler treats as the unit of job identity — not the specific K8s device-plugin mechanism. Large research/HPC-style training fleets commonly run Slurm rather than Kubernetes; a design that only knows how to attribute via K8s labels breaks the moment it meets a Slurm-scheduled cluster.

## Connections & what's next

This closes the module's through-line on the **hot path**: **cardinality is the master constraint, and delivered work (goodput) is the master SLI** — every lesson from L1 (cardinality as the signal-fit/cost matrix) through L8 (off-CPU profiling as the fleet-wide substrate) was building toward exactly this application. Turn this lesson's DCGM+NCCL signal plan and goodput-regression SLO/straggler alert directly into the **[fleet observability design](../practice/fleet-observability/README.md)**, then work the **[checkpoint](../checkpoint.md)** — items 4 and 5 (sizing a fleet metrics system, alerting on goodput) are this lesson's material stated as pass criteria.

One thread is deliberately left hanging. To keep this fleet inside its cardinality budget you dropped `pod`, `workload_id` and `gpu_uuid` from your metric labels — which are exactly the dimensions a *cost* question needs. **[Lesson 10 — the telemetry lakehouse](10-telemetry-lakehouse.md)** is the other half of that trade: a second path off the same DCGM source, tuned for accounting rather than alerting, that answers the questions this one structurally cannot.

Two sibling modules pick up where this one stops: **[modules/05-gpu-observability](../../../modules/05-gpu-observability/README.md)** is the single-node prerequisite this lesson explicitly builds on (DCGM internals, the util-lie, per-node MFU/goodput/XID) — go there first if any of that felt unfamiliar rather than merely "referenced." **[modules/11-gpu-cost-economics](../../../modules/11-gpu-cost-economics/README.md)** is the downstream consumer — every dollar figure in allocated-vs-utilised cost attribution sits on top of the goodput and attribution signals this lesson defines at fleet scale.

## References & further reading

**Primary sources**
- Meta, "The Llama 3 Herd of Models" (infrastructure/reliability section) — https://arxiv.org/pdf/2407.21783
- Meta, GPU Cluster Monitoring (GCM), open-source repo — https://github.com/facebookresearch/gcm
- NVIDIA, NIXT paper (NCCL Inspector) — https://arxiv.org/abs/2608.01449 — *very recent 2026 preprint; treat validation claims (≈2,048-GPU scale, <2% overhead) as early-stage, pilot before trusting at fleet scale.*
- NVIDIA DCGM documentation (fields, profiling metrics) — https://docs.nvidia.com/datacenter/dcgm/latest/
- NVIDIA NCCL documentation (collectives, environment, debugging) — https://docs.nvidia.com/deeplearning/nccl/

**Real-world engineering blogs**
- NVIDIA developer forum, "Enhancing Communication Observability of AI Workloads with NCCL Inspector" — https://forums.developer.nvidia.com/t/enhancing-communication-observability-of-ai-workloads-with-nccl-inspector/354225
- CoreWeave, "CoreWeave Drives 20% Higher GPU Cluster Performance" — https://www.coreweave.com/blog/coreweave-leads-the-charge-in-ai-infrastructure-efficiency-with-up-to-20-higher-gpu-cluster-performance-than-alternative-solutions — *vendor marketing; goodput/MFU figures are directional, not an audited benchmark.*

**Deeper dives**
- NVIDIA dcgm-exporter (DaemonSet, Kubernetes mapping, ServiceMonitor) — https://github.com/NVIDIA/dcgm-exporter
- Grafana NVIDIA DCGM Exporter dashboard (ID 12239) — https://grafana.com/grafana/dashboards/12239-nvidia-dcgm-exporter-dashboard/ (also see dashboard ID 15117 for an alternate fleet layout)
- Grafana Mimir multitenancy and per-tenant limits — https://grafana.com/docs/mimir/latest/manage/
