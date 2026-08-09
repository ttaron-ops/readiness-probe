---
lesson: "A03.9"
title: "GPU and ML observability at fleet scale"
module: "A-03"
concept: "fleet GPU observability"
status: not-started
est_time: "4 hrs"
artifacts: ["fleet GPU signal plan + goodput alert + straggler query"]
---

# A03.9 · GPU and ML observability at fleet scale

> **Concept.** Take single-node GPU truth (SM occupancy, MFU/goodput, XID) and make it survive thousands of GPUs: bounded cardinality, sharded scrape, per-tenant multitenancy, comms/straggler visibility, and alerts that page on burned GPU-hours instead of a moved gauge.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Why this matters
A single-node GPU dashboard that is *correct* still tells you almost nothing about a 2,000-GPU training run, because a synchronous job runs at the speed of its slowest rank and that rank is invisible in a fleet average. At staff level the failure mode is not "we don't have GPU metrics" — it's "we have DCGM on every node, Prometheus fell over from `gpu_uuid` cardinality, nobody can attribute a hot GPU to a team, and the run lost 18% goodput to one fail-slow NVLink for six hours before anyone paged." This lesson is the fleet-wide synthesis of the whole module: it reuses cardinality discipline (L1/L3), gauge/quantile correctness on DCGM series (L2), Mimir sharding and multitenancy (L3), collector-side enrichment (L4), exemplars into slow steps (L5), NCCL log discipline (L6), goodput burn-rate alerting (L7), and off-CPU/eBPF straggler profiling (L8) — applied at once to GPUs.

## Core notes
**Skip (module 05 covered it):** what DCGM is, GPU_UTIL-lies-vs-SM-occupancy, MFU/goodput defs, XID meaning, single-node GPU dashboards. This lesson assumes all of that and only scales it.

**Collection topology.** `dcgm-exporter` runs as a DaemonSet — one exporter per node, scraping the local DCGM host engine over the GPU. A Prometheus `ServiceMonitor` / `PodMonitor` (or native service discovery) auto-discovers new nodes as the fleet grows, so onboarding a rack is zero-touch. Turn on the exporter's **Kubernetes device-plugin mapping** so every GPU metric carries `pod`, `namespace`, `container` derived from the NVIDIA device-plugin allocation. That correlation is what converts "GPU 3 on node-47 is at 95 °C" into "team-b's `llama-serve` pod owns GPU 3" — attribution (L4) is the difference between a metric and an on-call action. Reference dashboards: Grafana IDs **12239** and **15117**.

**Cardinality is the central scaling problem.** DCGM series carry `gpu_uuid` (unbounded across a fleet and across RMA churn), `mig_instance`, plus injected `pod` (churny — every rollout mints new series) and any `job_id`. Multiply those and a few dozen base metrics explode into millions of active series; Prometheus OOMs. Staff answer, mirroring L1/L3: keep only **bounded** labels on the time series — `node`, `gpu` (local index 0–7), `tenant`, `model_class` — and drop the unbounded ones (`gpu_uuid`, `pod` hash suffixes, `container_id`) with `metric_relabel_configs` at scrape time. Per-run identity does not belong on metrics; it belongs in **exemplars/traces** (L5) that hang off the metric and link to the specific slow step. Then **hashmod-shard** the scrape across N Prometheus/agent replicas and **remote-write to Mimir** with per-tenant series limits so one team's cardinality blowout can't take down everyone's GPU visibility.

**Comms observability — what single-node dashboards structurally cannot see.** In distributed training, goodput dies in the *collectives* (all-reduce, all-gather, reduce-scatter), not in the kernels a node-local SM_ACTIVE gauge watches. A node can show 100% SM occupancy while the whole job is stalled waiting on one slow link. Emerging tooling — **NCCL Inspector / NIXT-style exporters** (Prometheus-shaped NCCL collective metrics, validated to ~2048 H100s) — injects CUDA events into NCCL calls to time each communication group and surface fail-slow ranks. Fleet signals to collect: per-collective duration outliers, and USE-method saturation on the **interconnect** itself — NVLink, PCIe, and IB/RoCE bandwidth vs line rate. The interconnect is a resource; apply Utilization/Saturation/Errors to it exactly as you would to a NIC.

**Alert on goodput regression, not utilization.** Ties directly to L7. The page-worthy event is "the fleet is burning money," expressed as a **burn-rate** alert on `achieved_tokens_per_sec / expected` (or step-time regression), where the error budget is **wasted GPU-hours**. A gauge moving is not an incident; a synchronous job silently running at 0.7× expected throughput across 512 GPUs is thousands of dollars an hour on fire. Multi-window burn-rate (fast + slow) keeps it from flapping on transient dips.

**Training-run observability.** Beyond infra metrics, capture the run's own health: loss and grad-norm curves (divergence / NaN detection), and **straggler detection** via per-rank step-time outliers — the one slow GPU or throttled link that gates the whole synchronous step. Distinguish **fail-stop** (a rank dies — caught cheaply by liveness/health checks) from **fail-slow** (a rank degrades but keeps running — only caught by goodput/step-time *variance*, never by a health check). Roll up **XID errors** across the fleet and correlate the rollup with job failures and goodput dips, so a spike in XID 79 (GPU fell off the bus) or ECC/thermal events maps to the runs it actually hurt.

**Operating across thousands of GPUs.** The mechanics: scrape sharding + Mimir multitenancy for scale; **recording rules** for fleet rollups so dashboards never fan out over millions of raw series (`fleet:gpu_sm_active:avg`, `fleet:sm_active:avg_by_tenant`, per-tenant goodput); continuous **eBPF profiling** (L8) for host-side and kernel regressions that starve the GPU (a slow dataloader is a CPU-side straggler); and event correlation joining XID/thermal/ECC with goodput. The governing law: a synchronous job runs at the speed of its slowest rank, so **tail and outlier detection matter more than averages** — an average SM_ACTIVE of 90% hides the p99 rank sitting at 40% that is setting everyone's step time.

## Worked example
A four-part fleet plan, worked end to end.

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
Any rank whose step time exceeds 1.3× the job median is a candidate straggler — pivot from that `rank` label into the NCCL exemplar and the eBPF off-CPU profile for that host to see whether it's a slow link (comms) or a starved dataloader (host CPU).

## Practice
Build the **fleet GPU signal plan**: a table of every GPU/comms signal, its bounded label set, where per-run identity goes (exemplar vs metric), the scrape-shard/remote-write layout to Mimir, plus the three artifacts above (relabel config, goodput burn-rate alert, straggler recording rule/query). See [fleet observability design](../practice/fleet-observability/README.md).

## Self-check
- Why does dropping `gpu_uuid` and `pod` from DCGM series not lose you the ability to debug a specific bad GPU? **Answer:** Because bounded labels (`node`, `gpu`, `tenant`) still identify the physical GPU for time-series alerting and rollups, while the high-cardinality per-run identity moves to exemplars/traces that hang off the metric — you click from the metric into the exemplar to reach the exact run/UUID, instead of paying for millions of active series to keep it inline. It's the L1/L3 cardinality trade applied to GPUs.
- A 512-GPU synchronous run shows fleet-average SM_ACTIVE at 91% but throughput is 30% below expected. What is happening and what signal exposes it? **Answer:** Fail-slow: one or a few ranks are degraded (a slow NVLink/IB link or a starved dataloader), and because the step is synchronous every fast rank blocks on the slow one — the average stays high while goodput collapses. The average hides it; a per-rank step-time p99/outlier query and NCCL per-collective duration outliers expose the straggler, which is why tail detection beats averages here.
- Why alert on `achieved_tokens_per_sec / expected` burn-rate instead of on GPU utilization or SM_ACTIVE dropping? **Answer:** Because utilization can be high (or low for benign reasons) while the thing that actually costs money — goodput — regresses, and vice versa. A burn-rate alert whose error budget is wasted GPU-hours pages only when the fleet is genuinely burning money, aligns the signal with business cost, and multi-window smoothing stops it flapping on checkpoints/warmup — the L7 SLO discipline applied to training economics.

## References
- NVIDIA dcgm-exporter (DaemonSet, Kubernetes mapping, ServiceMonitor): https://github.com/NVIDIA/dcgm-exporter
- NVIDIA DCGM documentation (fields, profiling metrics): https://docs.nvidia.com/datacenter/dcgm/latest/
- NVIDIA NCCL documentation (collectives, environment, debugging): https://docs.nvidia.com/deeplearning/nccl/
- Grafana NVIDIA DCGM Exporter dashboard (ID 12239): https://grafana.com/grafana/dashboards/12239-nvidia-dcgm-exporter-dashboard/
- Grafana Mimir multitenancy and per-tenant limits: https://grafana.com/docs/mimir/latest/manage/
