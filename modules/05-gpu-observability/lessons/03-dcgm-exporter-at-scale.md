---
lesson: "05.3"
title: "dcgm-exporter at scale — auditing the default counter set"
module: "05"
concept: "dcgm-exporter at scale — auditing the default counter set"
status: not-started
est_time: "6h"
prev: "02-dcgm-architecture.md"
next: "04-attribution.md"
artifacts: []
sources: 7
---
# 05.3 · dcgm-exporter at scale — auditing the default counter set

> **Concept.** The default dcgm-exporter counter set ships the honest utilization metric *commented out*, and a custom CSV *replaces* rather than extends it — so what you monitor at 500+ GPUs is a config decision you must make deliberately, cardinality included.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Where this fits

05.2 established *why* profiling fields are expensive: they come off a finite set of hardware performance-counter slots, DCGM multiplexes over-subscribed groups silently, and `nv-hostengine` needs superuser privileges to read any of it. That was the physics. This lesson is the config layer sitting directly on top of that physics: dcgm-exporter — the DaemonSet that actually turns DCGM fields into Prometheus series — ships a specific, opinionated *choice* of which fields to collect, and that choice is not the one you'd expect. Get this lesson right and 05.4 (attribution) has real `SM_ACTIVE` series to aggregate into namespaces; get it wrong and the capstone dashboard has nothing honest to show.

## Why this matters

GPU telemetry is a named senior competency now, not a monitoring sub-bullet — NVIDIA hires *Senior Platform Telemetry Engineer* and *SRE — Observability & Telemetry Platform* roles explicitly for this, and CoreWeave's whole product pitch as a neocloud is "visibility into complex AI workloads at massive scale." Datadog's 2025 GPU-monitoring product organizes its entire metric taxonomy around device-level (`gpu.sm_active`), process-level (`gpu.process.sm_active`), and Hopper-only pipeline metrics (`gpu.tensor_active`, `gpu.sm_occupancy`) — confirming that the SM_ACTIVE-vs-GPU_UTIL distinction this module teaches is exactly what a paid observability vendor decided was worth building a product around.

The concrete cost of getting this wrong: every stock GPU Grafana dashboard — the NVIDIA reference dashboard, the GPU Operator's bundled dashboard, most community dashboards on grafana.com — is built on fields that ship *enabled by default*, and `SM_ACTIVE`/`SM_OCCUPANCY` are not among them. A platform team can run dcgm-exporter in production for a year, pass every "do we have GPU dashboards" audit, and still be structurally unable to answer "is this fleet actually busy," because the honest field was never collected. That gap is precisely the multi-hundred-thousand-dollar/year blind spot the module's economics thesis depends on — and it starts here, at the collector config, not at the query layer.

## What's new here (calibration)

You already know Prometheus cardinality mechanics cold — you've blown up a TSDB with a bad `label_replace` before, and nothing here re-teaches series math from first principles. What's new is GPU-specific:

- The counter set is a **CSV file the exporter reads at process start**, not a scrape-time relabel. `metric_relabel_configs` cannot resurrect a field DCGM never collected — if the line isn't in the file, there is no series to keep or drop.
- **PROF fields are physically multiplexed** (05.2's hardware-counter constraint) — "just enable everything" changes what DCGM samples per interval *before* Prometheus ever sees a byte, unlike a pure software toggle.
- Enrichment cardinality on GPUs is **per-physical-device and label-driven** in a way that mirrors, but is structurally different from, generic Kubernetes cardinality blowups — the pod-labels-on-a-DaemonSet pattern is specific to how dcgm-exporter joins to pod-resources (04), not a generic `kube-state-metrics` problem.

## Core concepts

### The counter file, exactly

dcgm-exporter reads a CSV (`-f/--collectors`, default `/etc/dcgm-exporter/default-counters.csv`) whose grammar is three columns:

```
# DCGM FIELD, Prometheus metric type, help message
DCGM_FI_DEV_GPU_UTIL,        gauge, GPU utilization (in %).
DCGM_FI_DEV_FB_USED,         gauge, Framebuffer memory used (in MiB).
DCGM_FI_DEV_XID_ERRORS,      gauge, Value of the last XID error encountered.
DCGM_FI_PROF_GR_ENGINE_ACTIVE,   gauge, Ratio of time the graphics engine is active.
DCGM_FI_PROF_PIPE_TENSOR_ACTIVE, gauge, Ratio of cycles the tensor (HMMA) pipe is active.
DCGM_FI_PROF_DRAM_ACTIVE,        gauge, Ratio of cycles the device memory interface is active.
DCGM_FI_PROF_PCIE_TX_BYTES,      gauge, PCIe transmit throughput (bytes/sec).
DCGM_FI_PROF_PCIE_RX_BYTES,      gauge, PCIe receive throughput (bytes/sec).
# DCGM_FI_PROF_SM_ACTIVE,        gauge, Ratio of cycles an SM has at least 1 warp assigned.
# DCGM_FI_PROF_SM_OCCUPANCY,     gauge, Ratio of resident warps to the maximum on an SM.
```

A leading `#` is a comment — the field is *not collected*. This is a live read of the current upstream `main` file (fetched directly for this lesson, not paraphrased — see References). What ships **enabled**:

| Field | What it tells you honestly |
|---|---|
| `DCGM_FI_DEV_GPU_UTIL` | The misleading "any SM busy" number (05.1/05.2). Enabled. |
| `DCGM_FI_DEV_FB_USED` | Memory pressure / OOM proximity. Enabled. |
| `DCGM_FI_DEV_XID_ERRORS` | Hardware/driver faults (05.5). Enabled. |
| `DCGM_FI_PROF_GR_ENGINE_ACTIVE` | Graphics/compute engine active ratio — better than `GPU_UTIL`, coarser than `SM_ACTIVE`. Enabled. |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | Are the tensor cores actually doing matmuls. Enabled. |
| `DCGM_FI_PROF_DRAM_ACTIVE` | Memory-bandwidth bound vs compute bound. Enabled. |
| `DCGM_FI_PROF_PCIE_TX/RX_BYTES` | Host↔device transfer rate — enabled in current releases; useful for spotting dataloader/host-transfer-bound jobs (05.7 territory). |

What ships **commented out** and is the whole point of the deliverable:

| Field | Why it matters | Default |
|---|---|---|
| `DCGM_FI_PROF_SM_ACTIVE` | Fraction of SMs with ≥1 warp, averaged over *all* SMs — the honest "how much of the chip is lit." | **Commented** |
| `DCGM_FI_PROF_SM_OCCUPANCY` | Resident warps / max warps — how *densely* the lit SMs are packed. | **Commented** |

`GR_ENGINE_ACTIVE` is enabled but `SM_ACTIVE` is not — that asymmetry is subtle and worth internalizing. `GR_ENGINE_ACTIVE` answers "was the graphics/compute engine active at all," which is closer in spirit to `GPU_UTIL` than to a real occupancy measure. The metric that actually answers "are we wasting this card" — `SM_ACTIVE` — is the one you must turn on yourself.

### Why "missing from the dashboard" and "missing from the metric store" are the same bug

There is no Grafana panel for `DCGM_FI_PROF_SM_ACTIVE` in the stock dashboards because the series does not exist in Prometheus, because the field is commented in the CSV, because that is the default file baked into the image. The failure is four layers deep and every layer defaults to the misleading answer. "We have GPU dashboards" is not the same claim as "we can see GPU waste" — and no amount of Grafana panel-building fixes a collector-level gap.

### Custom CSV: replace, not extend

The second trap. When you supply your own counters file — via the GPU Operator (`dcgmExporter.config` ConfigMap) or the standalone Helm chart — dcgm-exporter uses **that file as the entire counter set.** It does **not** union with the default. Enable `SM_ACTIVE` in a file that contains only `SM_ACTIVE` and you have just stopped collecting `XID_ERRORS`, `FB_USED`, and everything else. The correct move: **start from the shipped default, uncomment the two honest fields, keep everything else**, and ship the whole file.

Standalone Helm:
```yaml
# values.yaml — the CSV becomes the FULL counter set, so paste the default + your edits
dcgmExporter:
  configmap: "" # or point to your own ConfigMap
customConfig: |-
  DCGM_FI_DEV_GPU_UTIL,            gauge, GPU utilization (in %).
  DCGM_FI_DEV_FB_USED,             gauge, Framebuffer memory used (in MiB).
  DCGM_FI_DEV_XID_ERRORS,          gauge, Value of the last XID error encountered.
  DCGM_FI_PROF_GR_ENGINE_ACTIVE,   gauge, Graphics/compute engine active ratio.
  DCGM_FI_PROF_PIPE_TENSOR_ACTIVE, gauge, Tensor (HMMA) pipe active ratio.
  DCGM_FI_PROF_DRAM_ACTIVE,        gauge, Device-memory interface active ratio.
  DCGM_FI_PROF_SM_ACTIVE,          gauge, Ratio of cycles an SM has >=1 warp assigned.   # UNCOMMENTED
  DCGM_FI_PROF_SM_OCCUPANCY,       gauge, Resident warps / max warps.                    # UNCOMMENTED
```

GPU Operator (ClusterPolicy) points dcgm-exporter at a ConfigMap:
```yaml
spec:
  dcgmExporter:
    config:
      name: custom-dcgm-metrics   # a ConfigMap with key dcgm-metrics.csv = the FULL file
```

Ship the *whole* default file with two lines uncommented, and diff it against upstream in CI — a driver/operator bump that changes the default should surface as a reviewable diff, not a silent metric loss.

### Cardinality at 500+ GPUs

Baseline: **series per GPU = number of enabled counters** (each field is one gauge, GPU UUID is one label value). With the ~8 enabled fields above you are at roughly 8 series per GPU before enrichment. dcgm-exporter also stamps identity labels — `gpu`, `UUID`, `device`, `modelName`, `Hostname`, `DCGM_FI_DRIVER_VERSION` — but those are **1:1 with the GPU**, adding label *width*, not series *count*.

Enrichment is where it explodes. The Kubernetes GPU Operator's own `deployment/values.yaml` (fetched directly for this lesson — see References) confirms the exact chart defaults:

- **`enablePodLabels`** defaults to **`false`** — sets `DCGM_EXPORTER_KUBERNETES_ENABLE_POD_LABELS=true` when flipped. Requires the exporter's ServiceAccount to have pod get/list/watch permission (the operator ships the ClusterRole `nvidia-dcgm-exporter-read-pods`).
- **`podLabelAllowlistRegex`** defaults to **`[]`** — an *empty list*, which means **every pod label is included** the moment `enablePodLabels` is flipped on. This is not a hypothetical misconfiguration; it is the chart's own documented default behavior. The chart itself ships commented example regexes worth using verbatim: `"^app$"`, `"^app\.kubernetes\.io/.*"`, `"^(tier|environment|version)$"`.

Series multiply by the churn of label values over the retention window, not just instantaneous GPU count:

```
active_series ≈ enabled_counters
              × GPUs
              × (distinct label-value combinations seen per GPU over retention)
```

A GPU fleet is instantaneously ~1 pod per GPU, so at any single scrape it looks bounded — but Prometheus counts a series as *alive* until it goes stale (~5 min) and *retained* for the whole block. A GPU that cycles 200 short training pods a day, each carrying a unique `job-id` / `pod-template-hash` label, generates 200 × counters new series per GPU per day. Multiply by 500 GPUs and 15-day retention and you have added millions of series whose only purpose was a label nobody queries. This exact failure mode — legacy exporters ported with noisy label sets exploding Prometheus TSDBs — is well-documented as a general Prometheus-at-scale pattern, not a GPU-specific edge case (Grafana Labs' cardinality-management guidance, cited below, generalizes it).

### Constraining enrichment

Enable pod labels and the allowlist **in the same change**, never in separate PRs — because the chart's own default the moment you flip `enablePodLabels` is "all labels," so a one-commit gap between the two changes is a live cardinality time bomb, not a theoretical risk:

```yaml
dcgmExporter:
  enablePodLabels: true
  podLabelAllowlistRegex: '^app\.kubernetes\.io/(name|instance|component)$'
```

That admits three bounded team/service dimensions and drops `pod-template-hash`, `job-id`, `controller-revision-hash`, and every operator-injected UUID label.

## Perspectives

**Developer/config perspective.** The counter set is a build-time/deploy-time CSV, not a query-time relabel. A developer used to "just add a relabel rule" for a missing dashboard field will hit a wall here — you cannot fix a missing metric with PromQL, only by changing the exporter's config and redeploying the DaemonSet.

**Operator/fleet perspective.** Cardinality is per-physical-GPU and multiplies by label-value churn over the retention window, not just instantaneous GPU count. "Prometheus OOM after we improved observability" is a genuinely common, well-documented incident category — you are usually the one who caused it by turning on the *right* metric the *wrong* way.

**Hardware/protocol perspective.** Because PROF fields are multiplexed on shared hardware performance-counter slots (05.2), "just enable everything" costs something *before* Prometheus ever sees a byte. The CPU-exporter instinct — cardinality is purely a Prometheus-side problem — doesn't fully carry over to GPU telemetry.

**Economics perspective.** The trap costs money twice: once because `SM_ACTIVE` — the metric that finds waste — ships off by default, so the waste stays invisible; and again because turning it plus careless pod-label enrichment on can 100–1000× your active series and blow your metrics-storage budget, spending money to buy the visibility that should have been nearly free.

## Real-world use cases

- **`NVIDIA/dcgm-exporter` — `etc/default-counters.csv`** — https://raw.githubusercontent.com/NVIDIA/dcgm-exporter/main/etc/default-counters.csv — fetched directly for this lesson. **What it shows:** a live, current confirmation that `SM_ACTIVE`/`SM_OCCUPANCY` really are commented out in the shipped default right now, on the current `main` branch — this isn't a stale claim from an old release.
- **`NVIDIA/dcgm-exporter` — `deployment/values.yaml`** — https://github.com/NVIDIA/dcgm-exporter/blob/main/deployment/values.yaml — fetched directly for this lesson. **What it shows:** confirms the chart ships cardinality-unsafe by default (`enablePodLabels: false`, but `podLabelAllowlistRegex: []` = all labels once flipped on) and supplies ready-made allowlist regex examples straight from the source.
- **Grafana Labs — "How to manage high cardinality metrics in Prometheus and Kubernetes"** — https://grafana.com/blog/how-to-manage-high-cardinality-metrics-in-prometheus-and-kubernetes/ (canonical URL; blocked by this sandbox's egress proxy on fetch, cited per SPEC's fallback rule). **What it shows:** the cardinality-explosion failure mode generalizes well beyond GPUs — validates that the module's cardinality math isn't a GPU-specific edge case but a known Prometheus-at-scale pattern, from the vendor whose community dashboards this module also references.

## Worked example

Fleet: 520 GPUs (65 nodes × 8), Prometheus retention 15 days, scrape 30s.

**Step 1 — baseline counters.** Ship the audited CSV: 8 enabled fields (`GPU_UTIL`, `FB_USED`, `XID_ERRORS`, `GR_ENGINE_ACTIVE`, `PIPE_TENSOR_ACTIVE`, `DRAM_ACTIVE`, `PCIE_TX_BYTES`, `PCIE_RX_BYTES`). Series from counters = `8 × 520 = 4,160`. The identity labels ride along for free. Trivial.

**Step 2 — Kubernetes attribution on, pod labels off.** Adds `pod`/`namespace`/`container` from the 04 pod-resources join. Still ~1 owner per GPU at a time; instantaneous series stay ~4,160, but over 15 days GPUs that churn pods accumulate stale series. On a training fleet averaging ~30 distinct pods/GPU over the window: `8 × 520 × 30 ≈ 124,800` retained series. Manageable.

**Step 3 — `enablePodLabels: true`, allowlist left at the chart default.** This is not a hypothetical misstep — `podLabelAllowlistRegex` defaults to `[]` in the upstream chart, confirmed above, so simply flipping `enablePodLabels` on with no further change *is* "no allowlist." Someone's Kubeflow pods carry `job-id` (unique per run) and `pod-template-hash`. Distinct label combos per GPU over 15 days jump to ~250. Series ≈ `8 × 520 × 250 = 1,040,000`. You just added ~900k series for labels nobody dashboards on — the classic "Prometheus OOM after we improved observability" incident, and the chart's own shipped default walked you into it.

**Step 4 — add the allowlist.** `^app\.kubernetes\.io/(name|instance|component)$` (one of the chart's own documented example patterns). Distinct combos per GPU collapse back toward the number of real services (~30). Series ≈ `8 × 520 × 30 ≈ 125k`. You kept team attribution and dropped the unbounded churn.

**The number to write down for the deliverable:** *series-per-GPU × fleet size*, stated for each of the four postures above, with the counter list and allowlist regex that produced it.

## Practice

Feeds the module deliverable, ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md):

1. **Deploy.** Via the GPU Operator, confirm dcgm-exporter is running as a DaemonSet and scraped. Capture `count({__name__=~"DCGM_FI_.*"})` and `count(count by (__name__)({__name__=~"DCGM_FI_.*"}))` (distinct metric names).
2. **Audit the default.** Pull the shipped `default-counters.csv` from the running pod (`kubectl exec ... cat /etc/dcgm-exporter/default-counters.csv`) and diff it against the canonical file. Confirm `SM_ACTIVE`/`SM_OCCUPANCY` are commented in *your* deployed copy. Verify there is **no `DCGM_FI_PROF_SM_ACTIVE` series** in Prometheus.
3. **Custom CSV.** Copy the full default, uncomment the two honest fields, ship it as the dcgm-exporter config. Re-check: `SM_ACTIVE` now present; **`XID_ERRORS`/`FB_USED` still present** (proves you extended by copying, not replaced-and-dropped). Save the diff.
4. **Enrichment blow-up.** Snapshot `count({__name__=~"DCGM_FI_PROF_SM_ACTIVE.*"})`. Turn on `enablePodLabels` with the chart's default (empty) allowlist. After one churn cycle, re-count active series and record the delta. Then add `podLabelAllowlistRegex` and re-measure.
5. **Fleet math.** Compute series-per-GPU for each posture and multiply by your real fleet size. Write the estimate note.

**Acceptance:** (a) a committed custom counters CSV that enables `SM_ACTIVE` and `SM_OCCUPANCY` *and* retains the full default field set (diff proves replace-safety); (b) a cardinality-estimate note stating series-per-GPU × fleet size for the four enrichment postures, with the allowlist regex you chose and why. Both drop into the deliverable.

## Common pitfalls

1. **Believing `metric_relabel_configs` can resurrect a field DCGM never collected.** The field must be un-commented in the exporter's own CSV; Prometheus-side relabeling only touches series that already exist. No amount of scrape-time cleverness fixes a collector-time omission.
2. **Shipping a custom CSV with just the new field added, believing it merges with the default.** It does not — dcgm-exporter's config *replaces* the full set. Always start from the complete default file.
3. **Turning on `enablePodLabels` without setting `podLabelAllowlistRegex` in the same change.** The chart's own default is "all labels" (`[]`), so this is a one-line, chart-default time bomb, not a hypothetical misconfiguration — the worked example above shows the exact ~900k-series blast radius.
4. **Assuming "just enable everything" is free because it's "just config."** PROF fields share finite hardware counter slots (05.2); over-enabling changes what DCGM samples per interval before Prometheus ever sees a byte.

## Self-check

- Why is `SM_ACTIVE` missing from the default Grafana GPU dashboard? **Answer:** Because the metric doesn't exist to be graphed. `DCGM_FI_PROF_SM_ACTIVE` ships *commented out* in dcgm-exporter's `default-counters.csv`, so DCGM never collects it, so no series reaches Prometheus, so the stock dashboards (built on the enabled `DCGM_FI_DEV_GPU_UTIL`) have nothing to panel. It's a collector default, not a dashboard omission — fix it in the CSV, not in Grafana.
- Does a custom counters CSV extend or replace the default set? **Answer:** Replace. The file you supply becomes the *entire* counter set; dcgm-exporter does not union it with the shipped default. A custom file containing only `SM_ACTIVE` silently stops collecting `XID_ERRORS`, `FB_USED`, and every other default field. Always start from the full default file, uncomment your additions, and diff against upstream in CI.
- What is the actual, confirmed default of `podLabelAllowlistRegex` in the upstream GPU Operator chart, and why does that make Step 3 of the worked example not a hypothetical? **Answer:** `[]` — an empty list, which the chart documents as meaning all pod labels are included once `enablePodLabels` is true. So flipping that one flag with no further change already is "no allowlist"; the ~900k-series blowup in the worked example is the chart's literal shipped behavior, not a contrived misconfiguration.
- Name one risky pod label to enrich on across 500 GPUs and why. **Answer:** `job-id` (or `pod-template-hash`, `controller-revision-hash`, any per-run UUID label). It is high-cardinality and unbounded — a fresh value per training run — so with `enablePodLabels` on it multiplies retained series by the number of distinct runs per GPU over the retention window, turning ~8 series/GPU into hundreds and OOMing Prometheus. Exclude it with an anchored `podLabelAllowlistRegex`.

## Connections & what's next

This lesson is the hinge between 05.2's hardware-level explanation of *why* profiling is expensive and 05.4's aggregation of the truthful metrics into per-namespace GPU-hours — without `SM_ACTIVE` actually landing in Prometheus here, there is nothing for 05.4 to join against or for the capstone dashboard to render. The pod-label enrichment discipline you build here also reuses the exact pod-resources join from Module 04 — you are not re-deriving that join, only deciding what rides along with it. Next: [05.4 — Attribution](04-attribution.md) takes these now-truthful, now-labelled series and turns them into the "who holds vs who uses" query that makes the whole deliverable land.

## References & further reading

**Primary sources**
- `NVIDIA/dcgm-exporter` — `etc/default-counters.csv` — https://raw.githubusercontent.com/NVIDIA/dcgm-exporter/main/etc/default-counters.csv — read for the exact, current set of enabled vs commented fields; fetched directly for this lesson.
- `NVIDIA/dcgm-exporter` — repository root — https://github.com/NVIDIA/dcgm-exporter — read for the exporter's source, CLI flags, and release notes.
- `NVIDIA/dcgm-exporter` — `deployment/values.yaml` — https://github.com/NVIDIA/dcgm-exporter/blob/main/deployment/values.yaml — read for the exact `enablePodLabels`/`podLabelAllowlistRegex` keys, defaults, and documented example regexes; fetched directly for this lesson.
- DeepWiki — dcgm-exporter custom metrics — https://deepwiki.com/NVIDIA/dcgm-exporter/4.3-custom-metrics — read for the replace-not-extend CSV semantics in narrative form.

**Real-world engineering blogs**
- Grafana Labs — "How to manage high cardinality metrics in Prometheus and Kubernetes" — https://grafana.com/blog/how-to-manage-high-cardinality-metrics-in-prometheus-and-kubernetes/ — what it shows: the cardinality-explosion pattern this lesson teaches generalizes across the whole Prometheus-on-Kubernetes ecosystem, not just GPU exporters.
- Datadog — GPU Monitoring Reference Architecture — https://www.datadoghq.com/architecture/gpu-monitoring/ — what it shows: a paid observability vendor's production metric taxonomy is organized around the exact enabled/commented distinction this lesson audits (device-level `sm_active`, process-level `process.sm_active`).

**Deeper dives**
- Grafana Labs — "NVIDIA DCGM Exporter Dashboard" (community dashboard) — https://grafana.com/grafana/dashboards/12239-nvidia-dcgm-exporter-dashboard/ — the widely-installed default dashboard built on the pre-audit counter set; install it and see the gap yourself.
