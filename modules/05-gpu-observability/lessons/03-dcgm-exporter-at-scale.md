---
lesson: "05.3"
title: "dcgm-exporter at scale — auditing the default counter set"
module: "05"
concept: "dcgm-exporter at scale — auditing the default counter set"
status: not-started
est_time: "4h"
artifacts: []
---
# 05.3 · dcgm-exporter at scale — auditing the default counter set
> **Concept.** The default dcgm-exporter counter set ships the honest utilization metric *commented out*, and a custom CSV *replaces* rather than extends it — so what you monitor at 500+ GPUs is a config decision you must make deliberately, cardinality included.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Why this matters

You already know (05.2) that `DCGM_FI_DEV_GPU_UTIL` is a liar: it reports "one or more
SMs were scheduled in the sample window," so a single active warp on one SM reads the
same 100% as a saturated card. The honest replacement is `DCGM_FI_PROF_SM_ACTIVE`
(fraction of SMs with at least one active warp, averaged over all SMs and the window)
and `DCGM_FI_PROF_SM_OCCUPANCY` (average resident-warps / max-warps). Here is the
operational trap that ends most "we have GPU dashboards" stories: **both of those are
commented out in the default counter file dcgm-exporter ships.** Every stock NVIDIA /
GPU Operator Grafana dashboard is built on `GPU_UTIL`. So the fleet-wide utilization
number your FinOps review quotes, and the one your capacity model divides cost by, is
the misleading metric — not because someone chose it, but because the truthful one was
never scraped. Nobody edited a config to lie; the default *is* the lie.

At 40+ clusters this is not a one-line fix. Turning on the prof metrics and pod-label
enrichment multiplies your active series per GPU, and dcgm-exporter's custom-CSV
mechanism *replaces* the default set rather than adding to it — so a careless "just add
SM_ACTIVE" silently drops XID errors and framebuffer from every dashboard. This lesson
is the audit: what ships on, what ships off, what a custom CSV does to the rest, and
what enrichment does to your Prometheus TSDB. The output feeds the deliverable directly.

## What's new vs 04 and your obs background

Module 04 built the *attribution spine*: your pod-resources exporter that maps device
UUID → pod → namespace, the same join dcgm-exporter does internally. That told you *how*
a GPU metric gets a `pod`/`namespace` label. This lesson is upstream of that join: it is
about *which* GPU metrics exist to be labelled at all, and what they cost to store.

You are fluent in Prometheus cardinality math already — you have blown up a TSDB with a
bad `label_replace` before. Nothing here re-teaches series cardinality. What is new is
**GPU-specific**:

- The counter set is a **CSV file the exporter reads at startup**, not scrape-time
  relabeling. You cannot `metric_relabel_configs` a metric into existence — if the field
  is not in the CSV, DCGM never collects it and there is no series to keep or drop.
- DCGM **profiling (`PROF_*`) fields are multiplexed on the hardware.** They are read
  from the same performance-monitoring units, and DCGM time-shares groups of them. So
  "just enable everything" is not free even before Prometheus sees it — it changes what
  DCGM samples per interval. This is a hardware constraint your CPU-exporter instincts
  don't carry.
- Enrichment cardinality is **per physical GPU**, and pod labels are attacker-of-your-
  TSDB grade: one `job-id` label on a Kubeflow fleet is unbounded.

## Core notes

### The counter file, exactly

dcgm-exporter reads a CSV (`-f/--collectors`, default `/etc/dcgm-exporter/default-counters.csv`)
whose grammar is three columns:

```
# DCGM FIELD, Prometheus metric type, help message
DCGM_FI_DEV_GPU_UTIL,        gauge, GPU utilization (in %).
DCGM_FI_DEV_FB_USED,         gauge, Framebuffer memory used (in MiB).
DCGM_FI_DEV_XID_ERRORS,      gauge, Value of the last XID error encountered.
DCGM_FI_PROF_GR_ENGINE_ACTIVE,   gauge, Ratio of time the graphics engine is active.
DCGM_FI_PROF_PIPE_TENSOR_ACTIVE, gauge, Ratio of cycles the tensor (HMMA) pipe is active.
DCGM_FI_PROF_DRAM_ACTIVE,        gauge, Ratio of cycles the device memory interface is active.
# DCGM_FI_PROF_SM_ACTIVE,        gauge, Ratio of cycles an SM has at least 1 warp assigned.
# DCGM_FI_PROF_SM_OCCUPANCY,     gauge, Ratio of resident warps to the maximum on an SM.
```

A leading `#` is a comment — the field is *not collected*. Audit the shipped file at the
canonical URL (Resources). What ships **enabled** and relevant to you:

| Field | Metric | What it tells you honestly |
|---|---|---|
| `DCGM_FI_DEV_GPU_UTIL` | `DCGM_FI_DEV_GPU_UTIL` | The misleading "any SM busy" number (05.2). Enabled. |
| `DCGM_FI_DEV_FB_USED` | framebuffer MiB used | Memory pressure / OOM proximity. Enabled. |
| `DCGM_FI_DEV_XID_ERRORS` | last XID code | Hardware/driver faults (05.5). Enabled. |
| `DCGM_FI_PROF_GR_ENGINE_ACTIVE` | graphics/compute engine active ratio | Better than GPU_UTIL, coarser than SM_ACTIVE. Enabled. |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | tensor-pipe active ratio | Are the tensor cores actually doing matmuls. Enabled. |
| `DCGM_FI_PROF_DRAM_ACTIVE` | memory-interface active ratio | Memory-bandwidth bound vs compute bound. Enabled. |

What ships **commented out** and is the whole point of the deliverable:

| Field | Why it matters | Default |
|---|---|---|
| `DCGM_FI_PROF_SM_ACTIVE` | Fraction of SMs with ≥1 warp, averaged over *all* SMs — the honest "how much of the chip is lit." | **Commented** |
| `DCGM_FI_PROF_SM_OCCUPANCY` | Resident warps / max warps — how *densely* the lit SMs are packed. | **Commented** |

So `GR_ENGINE_ACTIVE` is enabled but `SM_ACTIVE` is not. That is subtle: `GR_ENGINE_ACTIVE`
is "was the graphics/compute engine active at all," closer in spirit to `GPU_UTIL` than
to a real occupancy measure. The metric that answers "are we wasting this $30k card" —
`SM_ACTIVE` — is the one you have to turn on yourself.

### Why "missing from the dashboard" and "missing from the metric store" are the same bug

There is no Grafana panel for `DCGM_FI_PROF_SM_ACTIVE` in the stock dashboards because
the series does not exist in Prometheus, because the field is commented in the CSV,
because that is the default file baked into the image. The failure is four layers deep
and every layer defaults to the misleading answer. This is why "we have GPU dashboards"
is not the same claim as "we can see GPU waste."

### Custom CSV: replace, not extend

This is the second trap. When you supply your own counters file — via the GPU Operator
(`dcgmExporter.config` ConfigMap) or the standalone Helm chart — dcgm-exporter uses
**that file as the entire counter set.** It does **not** union with the default. Enable
`SM_ACTIVE` in a file that contains only `SM_ACTIVE` and you have just stopped collecting
`XID_ERRORS`, `FB_USED`, and everything else. The correct move is: **start from the
shipped default, uncomment the two honest fields, keep everything else**, and ship the
whole file.

Standalone Helm:
```yaml
# values.yaml — the CSV becomes the FULL counter set, so paste the default + your edits
dcgmExporter:  # (chart key is `dcgm-exporter`; shown flattened)
  configmap: "" # or point to your own ConfigMap
extraEnv: []
# The chart mounts your CSV at /etc/dcgm-exporter/. Provide the COMPLETE file:
customConfig: |-
  DCGM_FI_DEV_GPU_UTIL,            gauge, GPU utilization (in %).
  DCGM_FI_DEV_FB_USED,            gauge, Framebuffer memory used (in MiB).
  DCGM_FI_DEV_FB_FREE,            gauge, Framebuffer memory free (in MiB).
  DCGM_FI_DEV_XID_ERRORS,         gauge, Value of the last XID error encountered.
  DCGM_FI_PROF_GR_ENGINE_ACTIVE,   gauge, Graphics/compute engine active ratio.
  DCGM_FI_PROF_PIPE_TENSOR_ACTIVE, gauge, Tensor (HMMA) pipe active ratio.
  DCGM_FI_PROF_DRAM_ACTIVE,        gauge, Device-memory interface active ratio.
  DCGM_FI_PROF_SM_ACTIVE,          gauge, Ratio of cycles an SM has >=1 warp assigned.   # UNCOMMENTED
  DCGM_FI_PROF_SM_OCCUPANCY,       gauge, Resident warps / max warps.                    # UNCOMMENTED
```

GPU Operator (ClusterPolicy) points dcgm-exporter at a ConfigMap:
```yaml
# clusterpolicy patch
spec:
  dcgmExporter:
    config:
      name: custom-dcgm-metrics   # a ConfigMap with key dcgm-metrics.csv = the FULL file
```

Ship the *whole* default file with two lines uncommented. Diff it against upstream in CI
so a driver/operator bump that changes the default surfaces as a reviewable diff, not a
silent metric loss.

### Cardinality at 500+ GPUs

Baseline: **series per GPU = number of enabled counters** (each field is one gauge, and
the GPU UUID is one label value). With the ~half-dozen fields above you are at roughly
6–10 series per GPU before enrichment. dcgm-exporter also stamps identity labels on
every series — `gpu`, `UUID`, `device`, `modelName`, `Hostname`, `DCGM_FI_DRIVER_VERSION`
— but those are **1:1 with the GPU**, so they add label *width*, not series *count*.

Enrichment is where it explodes. Turning on Kubernetes attribution adds `pod`,
`namespace`, `container` (from your 04 pod-resources join) — still bounded, still ~1:1
with the GPU at any instant. But **pod-label enrichment** (`enablePodLabels`) copies
*arbitrary pod labels* onto every GPU series. Now series multiply by the churn of those
label values over the retention window:

```
active_series ≈ enabled_counters
              × GPUs
              × (distinct label-value combinations seen per GPU over retention)
```

A whole-GPU fleet is instantaneously 1 pod per GPU, so at any scrape it looks bounded —
but Prometheus counts a series as *alive* until it goes stale (~5 min) and *retained*
for the block. A GPU that cycles 200 short training pods a day, each with a unique
`job-id` / `pod-template-hash` label, generates 200 × counters new series per GPU per
day. Multiply by 500 GPUs and a 15-day retention and you have added millions of series
whose only purpose was a label you never query.

### Constraining enrichment

Two GPU-Operator / chart knobs:

- **`enablePodLabels: true`** → sets `DCGM_EXPORTER_KUBERNETES_ENABLE_POD_LABELS=true`.
  Off by default. On its own it copies *every* pod label. Requires the exporter's
  ServiceAccount to have pod get/list/watch (the operator ships the ClusterRole
  `nvidia-dcgm-exporter-read-pods`).
- **`podLabelAllowlistRegex`** → an allowlist regex; only pod labels whose *key* matches
  are emitted as metric labels. This is your cardinality valve. Anchor it and keep it to
  stable, low-cardinality keys:

  ```yaml
  dcgmExporter:
    enablePodLabels: true
    podLabelAllowlistRegex: '^app\.kubernetes\.io/(name|instance|component)$'
  ```

  That admits three bounded team/service dimensions and drops `pod-template-hash`,
  `job-id`, `controller-revision-hash`, and every operator-injected UUID label.

Rule: enrichment defaults to *all labels* the moment you enable it; the allowlist is not
optional at fleet scale. Enable pod labels and the allowlist in the same change, never
in separate PRs.

## Worked example

Fleet: 520 GPUs (65 nodes × 8), Prometheus retention 15 days, scrape 30s.

**Step 1 — baseline counters.** Ship the audited CSV: 8 enabled fields (the six above
plus `FB_FREE`, `MEM_COPY_UTIL`). Series from counters = `8 × 520 = 4,160`. The identity
labels ride along for free. Trivial.

**Step 2 — Kubernetes attribution on, pod labels off.** Adds `pod`/`namespace`/
`container` from pod-resources. Still ~1 owner per GPU at a time; instantaneous series
stay ~4,160, but over 15 days GPUs that churn pods accumulate stale series. On a
training fleet averaging ~30 distinct pods/GPU over the window: `8 × 520 × 30 ≈ 124,800`
retained series. Manageable.

**Step 3 — `enablePodLabels: true`, no allowlist.** Someone's Kubeflow pods carry
`job-id` (unique per run) and `pod-template-hash`. Distinct label combos per GPU over 15
days jump to ~250. Series ≈ `8 × 520 × 250 = 1,040,000`. You just added ~900k series for
labels nobody dashboards on. On a multi-tenant fleet with a dozen such labels this is the
classic "Prometheus OOM after we improved observability" incident.

**Step 4 — add the allowlist.** `^app\.kubernetes\.io/(name|instance|component)$`.
Distinct combos per GPU collapse back toward the number of real services (~30). Series ≈
`8 × 520 × 30 ≈ 125k`. You kept team attribution and dropped the unbounded churn.

**The number to write down for the deliverable:** *series-per-GPU × fleet size*, stated
for each of the three enrichment postures, with the counter list that produced it.

## Practice — feeds the deliverable

1. **Deploy.** Via the GPU Operator, confirm dcgm-exporter is running as a DaemonSet and
   scraped. Capture `count({__name__=~"DCGM_FI_.*"})` and
   `count(count by (__name__)({__name__=~"DCGM_FI_.*"}))` (distinct metric names).
2. **Audit the default.** Pull the shipped `default-counters.csv` from the running pod
   (`kubectl exec ... cat /etc/dcgm-exporter/default-counters.csv`) and diff it against
   the canonical file. Confirm `SM_ACTIVE` / `SM_OCCUPANCY` are commented in *your*
   deployed copy. Verify there is **no `DCGM_FI_PROF_SM_ACTIVE` series** in Prometheus.
3. **Custom CSV.** Copy the full default, uncomment the two honest fields, ship it as the
   dcgm-exporter config. Re-check: `SM_ACTIVE` now present; **XID_ERRORS/FB_USED still
   present** (proves you extended by copying, not replaced-and-dropped). Save the diff.
4. **Enrichment blow-up.** Snapshot `count({__name__=~"DCGM_FI_PROF_SM_ACTIVE.*"})`.
   Turn on `enablePodLabels` with no allowlist. After one churn cycle, re-count active
   series and record the delta. Then add `podLabelAllowlistRegex` and re-measure.
5. **Fleet math.** Compute series-per-GPU for each posture and multiply by your real
   fleet size. Write the estimate note.

**Acceptance:** (a) a committed custom counters CSV that enables `SM_ACTIVE` and
`SM_OCCUPANCY` *and* retains the full default field set (diff proves replace-safety);
(b) a cardinality-estimate note stating series-per-GPU × fleet size for the three
enrichment postures, with the allowlist regex you chose and why. Both drop into the
"Your GPU dashboard is lying to you" deliverable.

## Self-check

**(a) Why is `SM_ACTIVE` missing from the default Grafana GPU dashboard?**
**Answer:** Because the metric doesn't exist to be graphed. `DCGM_FI_PROF_SM_ACTIVE` is
shipped *commented out* in dcgm-exporter's `default-counters.csv`, so DCGM never collects
it, so no series reaches Prometheus, so the stock dashboards (built on the enabled
`DCGM_FI_DEV_GPU_UTIL`) have nothing to panel. It's not a dashboard omission — it's a
collector default. You fix it in the CSV, not in Grafana.

**(b) Does a custom counters CSV extend or replace the default set?**
**Answer:** Replace. The file you supply becomes the *entire* counter set; dcgm-exporter
does not union it with the shipped default. So a custom file containing only `SM_ACTIVE`
silently stops collecting `XID_ERRORS`, `FB_USED`, and every other default field. Always
start from the full default file and uncomment your additions, then diff against upstream
in CI.

**(c) Name one risky pod label to enrich on across 500 GPUs and why.**
**Answer:** `job-id` (or `pod-template-hash`, `controller-revision-hash`, any per-run
UUID label). It is high-cardinality and unbounded — a fresh value per training run — so
with `enablePodLabels` on it multiplies retained series by the number of distinct runs
per GPU over the retention window, turning ~6–10 series/GPU into hundreds and OOMing
Prometheus. Exclude it with an anchored `podLabelAllowlistRegex`.

## Resources

1. **dcgm-exporter `default-counters.csv`** (canonical) — audit it line by line; this is
   the file that decides what you see:
   https://raw.githubusercontent.com/NVIDIA/dcgm-exporter/main/etc/default-counters.csv
   and the repo https://github.com/NVIDIA/dcgm-exporter
2. **DeepWiki — dcgm-exporter custom metrics** — the replace-not-extend semantics and CSV
   grammar: https://deepwiki.com/NVIDIA/dcgm-exporter/4.3-custom-metrics
3. **dcgm-exporter `deployment/values.yaml`** — the exact `enablePodLabels` /
   `podLabelAllowlistRegex` keys and the env vars they set:
   https://github.com/NVIDIA/dcgm-exporter/blob/main/deployment/values.yaml
