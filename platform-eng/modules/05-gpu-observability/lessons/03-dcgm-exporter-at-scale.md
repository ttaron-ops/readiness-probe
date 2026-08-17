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
sources: 15
---
# 05.3 · dcgm-exporter at scale — auditing the default counter set

> **Concept.** The default dcgm-exporter counter set ships the honest utilization metric *commented out*, and a custom CSV *replaces* rather than extends it — so what you monitor at 500+ GPUs is a config decision you must make deliberately, cardinality included.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Where this fits

05.1 told you which fields carry truth and why field 203 does not. 05.2 explained the
machinery underneath: the host engine, field groups, watches, `updateFreq`, metric groups
and the three ways PROF fields vanish. Both were about *capability* — what DCGM can tell
you and at what cost.

This lesson is about *policy*: which of those capabilities the software you actually deploy
turns on by default. And the answer is uncomfortable. **dcgm-exporter's shipped counter set
enables `DCGM_FI_PROF_GR_ENGINE_ACTIVE` — a presence duty cycle — and comments out
`DCGM_FI_PROF_SM_ACTIVE` and `DCGM_FI_PROF_SM_OCCUPANCY`, the two fields 05.1 told you to
build the whole cost argument on.** The GPU Operator points at a slightly different file
with the same enabled set, so installing via the Operator does not save you. The official
NVIDIA Grafana dashboard has no panel for them, because there is no series to panel.

Get this lesson right and 05.4 has real `SM_ACTIVE` series to aggregate into namespaces and
dollars. Get it wrong and the capstone dashboard renders honestly-labelled emptiness.

Everything below is checked against **dcgm-exporter 4.8.3** (built on **DCGM 4.6.0**, image
`nvcr.io/nvidia/k8s/dcgm-exporter:4.6.0-4.8.3-distroless`) and **GPU Operator** as of the
same window, reading `etc/default-counters.csv`, `etc/dcp-metrics-included.csv`,
`deployment/values.yaml`, `internal/pkg/rendermetrics/render_metrics.go`,
`internal/pkg/transformation/`, `grafana/dcgm-exporter-dashboard.json`, and the Operator's
`assets/state-dcgm-exporter/0800_daemonset.yaml` and `deployments/gpu-operator/values.yaml`.
**The counter CSV drifts between releases — re-run the audit in §2 against the image tag you
actually deploy.**

## Why this matters

GPU telemetry is a named senior competency: NVIDIA hires *Senior Platform Telemetry
Engineer* and *SRE — Observability & Telemetry Platform* for it, and CoreWeave's product
pitch as a neocloud is "visibility into complex AI workloads at massive scale". Datadog
built a 2025 product whose taxonomy is this module's field list. The interview question is
not "do you know what dcgm-exporter is" — it is **"does a custom counters CSV extend or
replace the default set?"** and **"why is `SM_ACTIVE` missing from the default dashboard?"**,
both of which are on this module's checkpoint because they separate people who have deployed
GPU monitoring from people who have *audited* it.

The concrete cost has two halves and they pull in opposite directions.

**Half one: the metric that finds waste is off.** A platform team can run dcgm-exporter for
a year, pass every "do we have GPU dashboards" audit, present a green utilisation panel to
finance, and be structurally unable to answer "is this fleet actually busy". At the mid-2026
H100 on-demand band (~$2.50–3.00/GPU-hour, a dated directional snapshot), a 500-GPU fleet is
$11–13M/year, and the gap this metric would have exposed is most of it.

**Half two: turning it on carelessly costs you a Prometheus.** The pod-label enrichment that
makes the metric *useful* per team defaults, once enabled, to **including every pod label**.
On a fleet with per-run `job-id` labels that turns a bounded 13,000-series problem into an
unbounded millions-series problem, and the incident is called "Prometheus OOMed after we
improved observability".

Doing both correctly — turning on the honest field *and* bounding the labels, in the same
change — is a five-line diff that most teams get wrong in one direction or the other.

## What's new here (calibration)

You know Prometheus cardinality mechanics cold; you have blown up a TSDB with a bad
`label_replace`. Nothing here re-teaches series math from first principles. What is new is
GPU-specific and version-specific:

- **The counter set is a file read at process start**, not a scrape-time relabel.
  `metric_relabel_configs` cannot resurrect a field DCGM never collected — if the line is
  commented, there is no series to keep or drop. This is a different layer from every other
  exporter you operate.
- **The custom file replaces the default wholesale.** There is no union. The YAML config
  added in 4.8.x makes this explicit — `metrics.file` and `metrics.fields` are mutually
  exclusive, and either one *is* the counter set.
- **PROF fields cost something before Prometheus sees a byte** (05.2's metric groups), so
  "just enable everything" is not free the way it is for a CPU exporter.
- **The exact label set the exporter emits**, from the renderer source, including which
  labels are renderer-owned, which come from the pod-resources join, and how pod labels are
  sanitised and de-conflicted.
- **The GPU Operator uses a different counters file from the standalone chart**, with the
  same enabled set, a different ServiceMonitor interval, and `privileged: true` instead of
  `SYS_ADMIN` — three deltas worth knowing before you debug someone else's cluster.

## Core concepts

### 1. Where the counter set comes from, exactly

dcgm-exporter reads a CSV at startup. The path comes from, in precedence order:

| Source | Default | Notes |
|---|---|---|
| `-f` / `--collectors` flag | `/etc/dcgm-exporter/default-counters.csv` | `appconfig.DefaultCollectorsFile` |
| `DCGM_EXPORTER_COLLECTORS` env | same | what the GPU Operator sets |
| `-m` / `--configmap-data` | `none` | `<NAMESPACE>:<NAME>` ConfigMap |
| `--config-file` YAML `metrics.file` | — | added 4.8.x |
| `--config-file` YAML `metrics.fields` | — | inline field list, mutually exclusive with `file` |

The grammar is three comma-separated columns and nothing else:

```
# DCGM FIELD, Prometheus metric type, help message
DCGM_FI_DEV_GPU_UTIL,      gauge, GPU utilization (in %).
```

- **Column 1** — the DCGM field symbol. The exporter resolves it to a numeric ID through the
  DCGM field table, so the legacy spellings from 05.1 §7 (`DCGM_FI_PROF_SM_ACTIVE` rather
  than `DCGM_FI_PROF_SM_UTIL_RATIO`) are what you write.
- **Column 2** — the Prometheus type: `gauge`, `counter`, or **`label`**. `label` is the one
  people miss: a `label`-typed row does *not* produce a series. It attaches the field's value
  as a label on every other metric from that entity. `DCGM_FI_DRIVER_VERSION, label, Driver
  Version` is the only such row enabled by default.
- **Column 3** — the help string, verbatim into `# HELP`. It is **not** authoritative: the
  shipped help for `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` says "tensor (HMMA) pipe" when the field
  is *any* tensor pipe (05.1 §6). Do not learn semantics from column 3.
- A leading `#` is a comment. **A commented field is not collected, is not watched, does not
  exist.**
- The README's own warning: *"Always make sure your entries have 2 commas."*

There is one more rule that only shows up in the counter list: any field whose name starts
with `DCGM_FI_PROF_` is treated by the exporter as a profiling metric
(`Counter.IsProfilingMetric()` is a literal prefix check), which is what puts it under the
stale-watch repair logic from 05.2 §6(c). Prefix, not field-ID range.

### 2. The audit: the real default file, line by line

Here is the shipped `etc/default-counters.csv` from dcgm-exporter 4.8.3, complete. This is
the file baked into the image at `/etc/dcgm-exporter/default-counters.csv`. Read it; the
point of this lesson is in the commented lines.

```
# Format
# If line starts with a '#' it is considered a comment
# DCGM FIELD, Prometheus metric type, help message

# Clocks
DCGM_FI_DEV_SM_CLOCK,  gauge, SM clock frequency (in MHz).
DCGM_FI_DEV_MEM_CLOCK, gauge, Memory clock frequency (in MHz).

# Temperature
DCGM_FI_DEV_MEMORY_TEMP, gauge, Memory temperature (in C).
DCGM_FI_DEV_GPU_TEMP,    gauge, GPU temperature (in C).

# Power
DCGM_FI_DEV_POWER_USAGE,              gauge, Power draw (in W).
DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION, counter, Total energy consumption since boot (in mJ).

# PCIE
# DCGM_FI_PROF_PCIE_TX_BYTES,  counter, Total number of bytes transmitted through PCIe TX via NVML.
# DCGM_FI_PROF_PCIE_RX_BYTES,  counter, Total number of bytes received through PCIe RX via NVML.
DCGM_FI_DEV_PCIE_REPLAY_COUNTER, counter, Total number of PCIe retries.

# Utilization (the sample period varies depending on the product)
DCGM_FI_DEV_GPU_UTIL,      gauge, GPU utilization (in %).
DCGM_FI_DEV_MEM_COPY_UTIL, gauge, Memory utilization (in %).
DCGM_FI_DEV_ENC_UTIL,      gauge, Encoder utilization (in %).
DCGM_FI_DEV_DEC_UTIL ,     gauge, Decoder utilization (in %).

# Errors and violations
DCGM_FI_DEV_XID_ERRORS,              gauge,   Value of the last XID error encountered.
# DCGM_FI_DEV_POWER_VIOLATION,       counter, Throttling duration due to power constraints (in ns).
# DCGM_FI_DEV_THERMAL_VIOLATION,     counter, Throttling duration due to thermal constraints (in ns).
# DCGM_FI_DEV_SYNC_BOOST_VIOLATION,  counter, Throttling duration due to sync-boost constraints (in ns).
# DCGM_FI_DEV_BOARD_LIMIT_VIOLATION, counter, Throttling duration due to board limit constraints (in ns).
# DCGM_FI_DEV_LOW_UTIL_VIOLATION,    counter, Throttling duration due to low utilization (in ns).
# DCGM_FI_DEV_RELIABILITY_VIOLATION, counter, Throttling duration due to reliability constraints (in ns).

# DCGM Exporter fields

# DCGM_EXP_CLOCK_EVENTS_COUNT, gauge,   Clock events observed during the configured time window.
# DCGM_EXP_CLOCK_EVENTS_TOTAL, counter, Total clock events observed since exporter start (edge-counted).
# DCGM_EXP_XID_ERRORS_COUNT,   gauge,   XID errors observed during the configured time window.
# DCGM_EXP_XID_ERRORS_TOTAL,   counter, Total XID errors observed since exporter start.
# DCGM_EXP_GPU_HEALTH_STATUS,  gauge,   Current DCGM-reported GPU health status.
# DCGM_EXP_P2P_STATUS,         gauge,   Current NVLink P2P status per peer link.

# Memory usage
DCGM_FI_DEV_FB_FREE, gauge, Framebuffer memory free (in MiB).
DCGM_FI_DEV_FB_USED, gauge, Framebuffer memory used (in MiB).
DCGM_FI_DEV_FB_RESERVED, gauge, Framebuffer memory reserved (in MiB).

# ECC
# DCGM_FI_DEV_ECC_SBE_VOL_TOTAL, counter, Total number of single-bit volatile ECC errors.
# DCGM_FI_DEV_ECC_DBE_VOL_TOTAL, counter, Total number of double-bit volatile ECC errors.
# DCGM_FI_DEV_ECC_SBE_AGG_TOTAL, counter, Total number of single-bit persistent ECC errors.
# DCGM_FI_DEV_ECC_DBE_AGG_TOTAL, counter, Total number of double-bit persistent ECC errors.

# Retired pages
# DCGM_FI_DEV_RETIRED_SBE,     counter, Total number of retired pages due to single-bit errors.
# DCGM_FI_DEV_RETIRED_DBE,     counter, Total number of retired pages due to double-bit errors.
# DCGM_FI_DEV_RETIRED_PENDING, counter, Total number of pages pending retirement.

# NVLink
# DCGM_FI_DEV_NVLINK_CRC_FLIT_ERROR_COUNT_TOTAL, counter, Total number of NVLink flow-control CRC errors.
# DCGM_FI_DEV_NVLINK_CRC_DATA_ERROR_COUNT_TOTAL, counter, Total number of NVLink data CRC errors.
# DCGM_FI_DEV_NVLINK_REPLAY_ERROR_COUNT_TOTAL,   counter, Total number of NVLink retries.
# DCGM_FI_DEV_NVLINK_RECOVERY_ERROR_COUNT_TOTAL, counter, Total number of NVLink recovery errors.
DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL,            gauge,   Total number of NVLink bandwidth counters for all lanes.
# DCGM_FI_DEV_NVLINK_BANDWIDTH_L0,               counter, The number of bytes of active NVLink rx or tx data including both header and payload.

# VGPU License status
DCGM_FI_DEV_VGPU_LICENSE_STATUS, gauge, vGPU License status

# Remapped rows
DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS, counter, Number of remapped rows for uncorrectable errors
DCGM_FI_DEV_CORRECTABLE_REMAPPED_ROWS,   counter, Number of remapped rows for correctable errors
DCGM_FI_DEV_ROW_REMAP_FAILURE,           gauge,   Whether remapping of rows has failed

# Static configuration information. These appear as labels on the other metrics
DCGM_FI_DRIVER_VERSION,        label, Driver Version
# DCGM_FI_NVML_VERSION,          label, NVML Version
# DCGM_FI_DEV_BRAND,             label, Device Brand
# DCGM_FI_DEV_SERIAL,            label, Device Serial Number
# DCGM_FI_DEV_OEM_INFOROM_VER,   label, OEM inforom version
# DCGM_FI_DEV_ECC_INFOROM_VER,   label, ECC inforom version
# DCGM_FI_DEV_POWER_INFOROM_VER, label, Power management object inforom version
# DCGM_FI_DEV_INFOROM_IMAGE_VER, label, Inforom image version
# DCGM_FI_DEV_VBIOS_VERSION,     label, VBIOS version of the device

# Datacenter Profiling (DCP) metrics
# NOTE: supported on Nvidia datacenter Volta GPUs and newer
DCGM_FI_PROF_GR_ENGINE_ACTIVE,   gauge, Ratio of time the graphics engine is active.
# DCGM_FI_PROF_SM_ACTIVE,          gauge, The ratio of cycles an SM has at least 1 warp assigned.
# DCGM_FI_PROF_SM_OCCUPANCY,       gauge, The ratio of number of warps resident on an SM.
DCGM_FI_PROF_PIPE_TENSOR_ACTIVE, gauge, Ratio of cycles the tensor (HMMA) pipe is active.
DCGM_FI_PROF_DRAM_ACTIVE,        gauge, Ratio of cycles the device memory interface is active sending or receiving data.
# DCGM_FI_PROF_PIPE_FP64_ACTIVE,   gauge, Ratio of cycles the fp64 pipes are active.
# DCGM_FI_PROF_PIPE_FP32_ACTIVE,   gauge, Ratio of cycles the fp32 pipes are active.
# DCGM_FI_PROF_PIPE_FP16_ACTIVE,   gauge, Ratio of cycles the fp16 pipes are active.
DCGM_FI_PROF_PCIE_TX_BYTES,      gauge, The rate of data transmitted over the PCIe bus - including both protocol headers and data payloads - in bytes per second.
DCGM_FI_PROF_PCIE_RX_BYTES,      gauge, The rate of data received over the PCIe bus - including both protocol headers and data payloads - in bytes per second.
# DCGM_FI_PROF_SM_CYCLES_ELAPSED_TOTAL, counter, Total elapsed SM cycles.
# DCGM_FI_PROF_SM_CYCLES_ACTIVE_TOTAL,  counter, Total SM cycles with active warps.
# DCGM_FI_PROF_MMA_CYCLES_ACTIVE_TOTAL, counter, Total MMA tensor cycles active.
# DCGM_FI_PROF_DMMA_CYCLES_ACTIVE_TOTAL, counter, Total DMMA tensor cycles active.
# DCGM_FI_PROF_HMMA_CYCLES_ACTIVE_TOTAL, counter, Total HMMA tensor cycles active.
# DCGM_FI_PROF_IMMA_CYCLES_ACTIVE_TOTAL, counter, Total IMMA tensor cycles active.
# DCGM_FI_PROF_DFMA_CYCLES_ACTIVE_TOTAL, counter, Total DFMA tensor cycles active.
# DCGM_FI_PROF_PCIE_TX_BYTES_TOTAL,     counter, Total PCIe transmitted bytes.
# DCGM_FI_PROF_PCIE_RX_BYTES_TOTAL,     counter, Total PCIe received bytes.
# DCGM_FI_PROF_INT_CYCLES_ACTIVE_TOTAL, counter, Total integer pipe cycles active.
# DCGM_FI_PROF_FP64_CYCLES_ACTIVE_TOTAL, counter, Total FP64 pipe cycles active.
# DCGM_FI_PROF_FP32_CYCLES_ACTIVE_TOTAL, counter, Total FP32 pipe cycles active.
# DCGM_FI_PROF_FP16_CYCLES_ACTIVE_TOTAL, counter, Total FP16 pipe cycles active.
```

**The audit result: 26 uncommented rows, of which 1 is `label`-typed, leaving 25 series per
GPU. 52 field rows are commented out.** Here is the enabled set, classified by what it can
actually answer:

| # | Field | Category | Answers |
|---|---|---|---|
| 1 | `DCGM_FI_DEV_SM_CLOCK` | clocks | throttling evidence |
| 2 | `DCGM_FI_DEV_MEM_CLOCK` | clocks | throttling evidence |
| 3 | `DCGM_FI_DEV_MEMORY_TEMP` | thermal | is it hot |
| 4 | `DCGM_FI_DEV_GPU_TEMP` | thermal | is it hot |
| 5 | `DCGM_FI_DEV_POWER_USAGE` | power | watts now |
| 6 | `DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION` | power | joules since boot |
| 7 | `DCGM_FI_DEV_PCIE_REPLAY_COUNTER` | health | link errors |
| 8 | **`DCGM_FI_DEV_GPU_UTIL`** | **presence** | **the lie (05.1)** |
| 9 | `DCGM_FI_DEV_MEM_COPY_UTIL` | presence | the memory-side lie |
| 10 | `DCGM_FI_DEV_ENC_UTIL` | media | NVENC duty cycle |
| 11 | `DCGM_FI_DEV_DEC_UTIL` | media | NVDEC duty cycle |
| 12 | `DCGM_FI_DEV_XID_ERRORS` | health | last XID (05.5) |
| 13 | `DCGM_FI_DEV_FB_FREE` | memory | headroom |
| 14 | `DCGM_FI_DEV_FB_USED` | memory | **the reclaim gate** |
| 15 | `DCGM_FI_DEV_FB_RESERVED` | memory | accounting |
| 16 | `DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL` | fabric | aggregate NVLink |
| 17 | `DCGM_FI_DEV_VGPU_LICENSE_STATUS` | vGPU | licensing |
| 18 | `DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS` | health | memory degradation |
| 19 | `DCGM_FI_DEV_CORRECTABLE_REMAPPED_ROWS` | health | memory degradation |
| 20 | `DCGM_FI_DEV_ROW_REMAP_FAILURE` | health | memory degradation |
| — | `DCGM_FI_DRIVER_VERSION` | **label** | *not a series* |
| 21 | **`DCGM_FI_PROF_GR_ENGINE_ACTIVE`** | **presence** | **a second lie (05.1 §6)** |
| 22 | `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | **efficiency** | are tensor pipes firing |
| 23 | `DCGM_FI_PROF_DRAM_ACTIVE` | **efficiency** | is HBM the wall |
| 24 | `DCGM_FI_PROF_PCIE_TX_BYTES` | fabric | host→device rate |
| 25 | `DCGM_FI_PROF_PCIE_RX_BYTES` | fabric | device→host rate |

And the two that are not there:

| Field | ID | Why it matters | Default |
|---|---|---|---|
| **`DCGM_FI_PROF_SM_ACTIVE`** | 1002 | The only *breadth* metric. Mean over all SMs of cycles with ≥1 warp assigned. The reclaim signal. The numerator of the capstone's dollar figure. | **Commented** |
| **`DCGM_FI_PROF_SM_OCCUPANCY`** | 1003 | The only *depth* metric. Resident warps ÷ architectural max. The kernel-tuning signal. | **Commented** |

**Sit with the asymmetry, because it is the point of the lesson.** The default set gives you
*two* presence metrics (`GPU_UTIL`, `GR_ENGINE_ACTIVE`), *two* efficiency ratios
(`PIPE_TENSOR_ACTIVE`, `DRAM_ACTIVE`), and *zero* breadth metrics. You can tell that
something is scheduled and that the tensor pipes are quiet — but you cannot tell how much of
the die is lit, which is the question "should we reclaim this GPU" reduces to. The
`GR_ENGINE_ACTIVE`-enabled / `SM_ACTIVE`-commented pairing is the exact inversion of what
05.1's alerting policy needs.

Two secondary observations from the same file that matter later:

- **`DCGM_FI_PROF_PCIE_TX_BYTES` appears twice.** Once near the top, commented, typed
  `counter`, with a help string saying "via NVML"; once at the bottom, **enabled**, typed
  `gauge`, described as a rate in bytes/second. The bottom row wins. If you see PCIe
  throughput behaving like a gauge when you expected a monotonic counter, this is why.
- **The `DCGM_EXP_*` block is entirely commented.** Those are exporter-computed metrics, not
  DCGM fields: `DCGM_EXP_XID_ERRORS_COUNT` (XIDs in the last window, default 5 min via
  `--xid-count-window-size`), `DCGM_EXP_XID_ERRORS_TOTAL` (a cumulative counter suitable for
  `increase()`), the clock-event equivalents, `DCGM_EXP_GPU_HEALTH_STATUS`, and
  `DCGM_EXP_P2P_STATUS`. Lesson 05.5 turns these on; note here that the *only* enabled XID
  metric is `DCGM_FI_DEV_XID_ERRORS`, which is a gauge holding the **last** XID code and is
  therefore useless with `rate()`/`increase()`.

### 3. The four-layer failure, drawn

The reason "we have GPU dashboards" and "we can see GPU waste" are different claims is that
the default answer is wrong at four independent layers, and each layer's default hides the
one above it.

```
  WHY THERE IS NO SM_ACTIVE PANEL — the causal chain, top to bottom
 ══════════════════════════════════════════════════════════════════════════════════

  LAYER 4 · GRAFANA
  ┌──────────────────────────────────────────────────────────────────────────┐
  │ NVIDIA's official dashboard (grafana.com #12239, shipped in the repo as  │
  │ grafana/dcgm-exporter-dashboard.json) — EVERY panel it contains:         │
  │                                                                          │
  │   GPU Temperature ............ DCGM_FI_DEV_GPU_TEMP                      │
  │   GPU Avg. Temp .............. DCGM_FI_DEV_GPU_TEMP                      │
  │   GPU Power Usage ............ DCGM_FI_DEV_POWER_USAGE                   │
  │   GPU Power Total ............ DCGM_FI_DEV_POWER_USAGE                   │
  │   GPU SM Clocks .............. DCGM_FI_DEV_SM_CLOCK                      │
  │   GPU Utilization ............ DCGM_FI_DEV_GPU_UTIL          ◀── THE LIE │
  │   GPU Framebuffer Mem Used ... DCGM_FI_DEV_FB_USED                       │
  │   Tensor Core Utilization .... DCGM_FI_PROF_PIPE_TENSOR_ACTIVE           │
  │                                                                          │
  │   SM_ACTIVE: absent.   SM_OCCUPANCY: absent.   DRAM_ACTIVE: absent.      │
  └───────────────────────────────────┬──────────────────────────────────────┘
                                      │ "why no panel?"
                                      ▼
  LAYER 3 · PROMETHEUS
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  DCGM_FI_PROF_SM_ACTIVE            → no data                             │
  │  count({__name__=~"DCGM_FI_.*"})   → 25 distinct names, none of them it  │
  │                                                                          │
  │  metric_relabel_configs CANNOT help. Relabeling operates on series that  │
  │  arrived in the scrape response. This one never arrived.                 │
  └───────────────────────────────────┬──────────────────────────────────────┘
                                      │ "why no series?"
                                      ▼
  LAYER 2 · DCGM-EXPORTER
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  /etc/dcgm-exporter/default-counters.csv, line 91:                       │
  │                                                                          │
  │     # DCGM_FI_PROF_SM_ACTIVE,  gauge, The ratio of cycles an SM has ...  │
  │     ▲                                                                    │
  │     └── one character                                                    │
  │                                                                          │
  │  The field is not in the counter list → not in the field group →         │
  │  no watch is created → the engine never samples it.                      │
  └───────────────────────────────────┬──────────────────────────────────────┘
                                      │ "why is it commented?"
                                      ▼
  LAYER 1 · DCGM ENGINE / SILICON
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  PROF fields need the profiling path (05.2): Volta+, elevated privileges,│
  │  and on pre-GPM parts a counter-bank budget shared with the tensor       │
  │  fields. A conservative default that "works everywhere" is a default     │
  │  with the cheap, universally-available fields on and the costed ones off.│
  └──────────────────────────────────────────────────────────────────────────┘

 ══════════════════════════════════════════════════════════════════════════════════
  EVERY LAYER DEFAULTS TO THE MISLEADING ANSWER, AND EACH ONE EXPLAINS THE NEXT.
  The fix is at layer 2 and nowhere else. No amount of Grafana or PromQL work
  can recover a metric the collector was never told to collect.
 ══════════════════════════════════════════════════════════════════════════════════
```

Is the layer-1 explanation charitable or an excuse? Both, and the distinction matters for
how you argue it. `SM_ACTIVE` and `SM_OCCUPANCY` are in the same metric group as
`PIPE_TENSOR_ACTIVE` on GPM hardware, so on any Hopper-or-newer fleet enabling them is
genuinely free — the counters are already in the snapshot. On V100-class silicon they may
land in a different subgroup and cost a multiplexing pass. **The default is calibrated for
the oldest supported hardware, and it is 2026.** Audit it, don't inherit it.

### 4. Replace, not extend

The second trap, and a checkpoint question. **When you supply your own counters file,
dcgm-exporter uses that file as the entire counter set. It does not union it with the
default.**

The consequence is severe and quiet: a well-intentioned ConfigMap containing only

```
DCGM_FI_PROF_SM_ACTIVE, gauge, The ratio of cycles an SM has at least 1 warp assigned.
```

turns on the honest metric and **silently stops collecting `DCGM_FI_DEV_XID_ERRORS`,
`DCGM_FI_DEV_FB_USED`, temperature, power, and everything else**. Your reclaim rule starts
working and your hardware-fault alerting stops, in the same deploy, with no error anywhere.

The 4.8.x YAML config makes the semantics explicit rather than implicit:

```yaml
version: 1
metrics:
  file: /etc/dcgm-exporter/default-counters.csv   # EITHER a file …
  # fields:                                        # … OR an inline list.
  #   - name: DCGM_FI_DEV_GPU_TEMP                 # Never both — the README:
  #     prometheusType: gauge                      # "YAML metric sources are
  #     help: GPU temperature (in C).              #  mutually exclusive."
collection:
  interval: 30s
```

**The correct procedure, as a runbook:**

```bash
# 1. Extract the default from the exact image you deploy — not from GitHub main,
#    because the file drifts between releases.
kubectl -n gpu-operator exec ds/nvidia-dcgm-exporter -c nvidia-dcgm-exporter -- \
    cat /etc/dcgm-exporter/default-counters.csv > dcgm-metrics.csv

# 2. Uncomment exactly the two lines you need. Nothing else.
sed -i 's|^# \(DCGM_FI_PROF_SM_ACTIVE,\)|\1|;   s|^# \(DCGM_FI_PROF_SM_OCCUPANCY,\)|\1|' \
    dcgm-metrics.csv

# 3. Prove you changed two lines and only two lines.
kubectl -n gpu-operator exec ds/nvidia-dcgm-exporter -c nvidia-dcgm-exporter -- \
    cat /etc/dcgm-exporter/default-counters.csv | diff - dcgm-metrics.csv
# expected: exactly two hunks, each removing "# " from the front of one line.

# 4. Ship it as a ConfigMap whose key is EXACTLY dcgm-metrics.csv (Operator) or
#    whatever your chart mounts (standalone).
kubectl -n gpu-operator create configmap custom-dcgm-metrics \
    --from-file=dcgm-metrics.csv
```

Wiring it up, for each install path:

**GPU Operator (ClusterPolicy / Helm values).** The Operator's own `values.yaml` documents
the contract: *"Use `name` to either point to an existing ConfigMap or to create a new one…
When pointing to an existing ConfigMap, the ConfigMap must exist in the same namespace as
the release. The metrics are expected to be listed under a key called `dcgm-metrics.csv`."*

```yaml
dcgmExporter:
  enabled: true
  version: 4.6.0-4.8.3-distroless
  config:
    name: custom-dcgm-metrics      # existing ConfigMap, key: dcgm-metrics.csv
    # create: true                 # or have the chart build one from `data:`
    # data: |-
    #   <the full CSV inline>
```

**Standalone dcgm-exporter chart.**

```yaml
# values.yaml
image:
  tag: 4.6.0-4.8.3-distroless
arguments: ["-f", "/etc/dcgm-exporter/dcgm-metrics.csv"]
# …plus a volume/volumeMount for your ConfigMap, or use configmap: <name>
```

**Then verify it took, from the outside:**

```promql
# Must all be non-empty. If XID_ERRORS is empty you replaced instead of copied.
count(DCGM_FI_PROF_SM_ACTIVE)
count(DCGM_FI_PROF_SM_OCCUPANCY)
count(DCGM_FI_DEV_XID_ERRORS)
count(DCGM_FI_DEV_FB_USED)

# Distinct metric names now exported. Should be 27 with the two additions.
count(count by (__name__) ({__name__=~"DCGM_FI_.*"}))
```

**And put the diff in CI.** A DCGM or Operator bump changes the shipped default; if your
custom file is a frozen copy of an old default, you silently lose whatever fields upstream
added. The CI job is: pull the default out of the new image, diff against your file ignoring
your two intentional uncomment hunks, and fail the build on any other difference. That turns
a silent metric loss into a reviewable pull request.

### 5. Two install paths that are not the same

If you inherit a cluster, know which one you are looking at. The differences are real and
each has bitten someone.

| | Standalone dcgm-exporter chart | GPU Operator (`dcgmExporter`) |
|---|---|---|
| Counters file | `/etc/dcgm-exporter/default-counters.csv` (flag default) | **`/etc/dcgm-exporter/dcp-metrics-included.csv`** via `DCGM_EXPORTER_COLLECTORS` |
| Enabled set | 26 rows (25 series + 1 label) | **the same 26** — the two files differ only in comments and whitespace |
| Kubernetes join | off unless `--kubernetes` | **`DCGM_EXPORTER_KUBERNETES=true`** set in the DaemonSet |
| Privileges | `capabilities: add: ["SYS_ADMIN"], drop: ["ALL"]`, `runAsUser: 0` | **`privileged: true`** on the container |
| ServiceMonitor interval | `30s`, `scrapeTimeout: 25s` | **`15s`**, `scrapeTimeout: 10s` |
| `honorLabels` | `false` | `false` |
| `hostNetwork` / `hostPID` | not set | `false` / `false` |
| Pod-resources mount | via chart values | `/var/lib/kubelet/pod-resources` hostPath, always |
| Pod labels | `kubernetes.enablePodLabels: false` | `dcgmExporter.enablePodLabels: false` |
| Allowlist | `kubernetes.podLabelAllowlistRegex: []` | `dcgmExporter.podLabelAllowlistRegex:` (commented out ⇒ empty) |
| RBAC for pod labels | chart creates ClusterRole when enabled | ClusterRole `nvidia-dcgm-exporter-read-pods` (`pods: get,list,watch`) |
| Ordering | none | init container blocks on `/run/nvidia/validations/toolkit-ready` (module 04) |

**The one to notice: the Operator scrapes every 15 s while the exporter's collect interval
defaults to 30,000 ms.** By 05.2 §7 that is a 2:1 oversample — every second Prometheus
sample is a duplicate of the previous one, because DCGM has not produced a new interval yet.
It is not harmful, but it doubles your PROF sample rate for zero information, and it means
that on an Operator-installed cluster the "resolution" of your GPU dashboards is 30 s no
matter what the scrape config says. Align them: either set
`DCGM_EXPORTER_INTERVAL=15000` via `dcgmExporter.env`, or set
`dcgmExporter.serviceMonitor.interval: 30s`.

### 6. The label set, exactly

Cardinality arithmetic needs the label list, and guessing it is how the arithmetic goes
wrong. From `internal/pkg/rendermetrics/render_metrics.go`, for `FE_GPU` entities, in
emission order:

| Label | Source | Cardinality per GPU | Notes |
|---|---|---|---|
| `gpu` | renderer | 1 | DCGM GPU index within the node |
| `UUID` | renderer | 1 | `GPU-<uuid>` — the join key for 05.4 |
| `pci_bus_id` | renderer | 1 | |
| `device` | renderer | 1 | `nvidia0`, … |
| `modelName` | renderer | 1 | `NVIDIA H100 80GB HBM3` |
| `GPU_I_PROFILE` | renderer | 1 | **only when MIG**; e.g. `1g.10gb` |
| `GPU_I_ID` | renderer | 1 | **only when MIG**; GPU-instance ID |
| `hostname` | renderer | 1 | suppressed by `--no-hostname` |
| `DCGM_FI_DRIVER_VERSION` | `label`-typed CSV row | 1 | fleet-wide low cardinality |
| `pod` | pod-resources | ~1 at a time, **unbounded over time** | ← the cardinality risk |
| `namespace` | pod-resources | small | |
| `container` | pod-resources | small | |
| `pod_uid` | pod-resources | **unbounded** | only with `--kubernetes-enable-pod-uid` |
| `vgpu` | device-ID suffix | small | only under GPU sharing |
| `hpc_job` | job-mapping dir | varies | Slurm integration |
| `dra_claim_name`, `dra_claim_namespace`, `dra_driver_name`, `dra_pool_name`, `dra_device_name`, `dra_mig_profile`, `dra_mig_device_uuid` | DRA | varies | only with `KUBERNETES_ENABLE_DRA` |
| *arbitrary pod labels* | pod informer | **unbounded** | only with `enablePodLabels` |

Three mechanics you need before doing the math:

**(a) Renderer-owned labels are 1:1 with the GPU.** `gpu`, `UUID`, `pci_bus_id`, `device`,
`modelName`, `hostname`, driver version — these add label *width* (bytes per series), not
series *count*. They are free from a cardinality standpoint and not free from a memory
standpoint, which matters in §7.

**(b) Pod labels are copied in as bare, sanitised label names — not prefixed.**
`SanitizeLabelName()` replaces invalid characters with `_`, prepends `_` if the name starts
with a digit, and collapses a leading `__` (reserved by Prometheus). So
`app.kubernetes.io/name` becomes `app_kubernetes_io_name`. The `pod_label_` prefix is applied
**only on collision** with a renderer-reserved label or an existing attribute, with
`_conflict1`, `_conflict2` … appended if even the prefixed form collides. Practical
consequence: **a pod label named `gpu` or `device` will not clobber the renderer's — it
becomes `pod_label_gpu` — but you cannot predict from the label name alone what it will be
called in Prometheus.** Anchor your allowlist regexes on the Kubernetes label name, and
expect the sanitised form in queries.

**(c) `honor_labels: false` is the default everywhere**, so Prometheus renames the
exporter's `pod`/`namespace`/`container` to `exported_pod`/`exported_namespace`/
`exported_container` whenever its own service-discovery labels collide. Both the standalone
chart and the Operator ship `honorLabels: false`. Which spelling your queries use is a
property of your scrape config, not of the exporter. Lesson 05.4 builds on this.

### 7. Cardinality: N × M × K, worked

Now the arithmetic, carried through with units, so you can re-run it with your own numbers.

**The model.**

```
  active_series  =  G  ×  M  ×  K

     G = GPUs (or MIG instances) exporting
     M = enabled, non-label counters per GPU        (25 in the default set)
     K = distinct label-value COMBINATIONS observed per GPU
         over the retention window  ← the only term that is not fixed
```

`K` is where everything goes wrong, and the reason is a Prometheus fact rather than a GPU
fact: **a series exists from the moment a sample lands until it is compacted out at the end
of retention, not just while it is being reported.** A GPU that hosts 40 short training pods
today has 40 distinct `pod` label values attached to its 25 counters for as long as your
retention window says, even though at most one is live at a time. Cardinality is an integral
over the window, not an instantaneous count.

**Fleet: 65 nodes × 8 GPUs = 520 GPUs. Retention 15 d. Scrape 30 s.**

*Posture 0 — device labels only (`--kubernetes=false`).*

```
  K = 1                     (one immutable label combination per GPU)
  series = 520 × 25 × 1                                        =  13,000
```

*Posture 1 — Kubernetes join on, pod labels off.*

The pod-resources join adds `pod`/`namespace`/`container`. `K` becomes the number of
distinct pods that ever held that GPU inside the window. A long-lived inference fleet might
be 2–3; a training fleet cycling jobs might be 30.

```
  inference-shaped, K = 3
  series = 520 × 25 × 3                                        =  39,000

  training-shaped,  K = 30
  series = 520 × 25 × 30                                       = 390,000
```

*Posture 2 — `enablePodLabels: true`, allowlist left at its default.*

**This is not a hypothetical misconfiguration.** `podLabelAllowlistRegex` defaults to `[]`
in both the standalone chart and the Operator, and the exporter's `newLabelFilterCache()`
sets `enabled: len(patterns) > 0` — an empty list means **filtering is off and every pod
label is included**. There is a second, nastier path to the same place: if every supplied
pattern fails to compile, the exporter logs *"No valid regex patterns for pod label
filtering, all labels will be included"* and **disables filtering** rather than failing
closed. A typo in your regex is indistinguishable in effect from having no regex.

Kubeflow, Argo Workflows, Ray, Volcano and Kueue all stamp per-run labels. Say each pod
carries `job-id` (unique per run), `pod-template-hash`, and `controller-revision-hash`. Now
`K` is the number of distinct *label-tuple* values, not the number of pods — but since
`job-id` is unique per pod, it is at least the pod count, and the other labels do not
multiply it further because they co-vary. Take the training-shaped 30 pods/GPU/day over
15 days:

```
  K ≈ 30 pods/GPU/day × 15 days = 450 distinct combinations per GPU
  series = 520 × 25 × 450                                      = 5,850,000
```

**From 39,000 to 5.85 million by flipping one boolean.** And note what you bought: the
ability to group by `app`, which you could have had for `K × 1`.

*Posture 3 — bounded allowlist.*

```yaml
kubernetes:
  enablePodLabels: true
  podLabelAllowlistRegex:
    - "^app$"
    - "^app\\.kubernetes\\.io/(name|instance|component)$"
    - "^(tier|environment|team)$"
```

(The first two patterns are lifted from the chart's own commented examples; the Operator's
`values.yaml` suggests `"^app$"` and `"^kueue\\.x-k8s\\.io/.*$"`.) These labels take one
value per *service*, not per run, so `K` collapses back to roughly the pod-churn number:

```
  K ≈ 30
  series = 520 × 25 × 30                                       = 390,000
```

You kept team attribution and dropped the unbounded churn. Summary:

| Posture | `K` | Series | vs. baseline |
|---|---|---|---|
| 0 · device labels only | 1 | 13,000 | 1× |
| 1 · K8s join, inference-shaped | 3 | 39,000 | 3× |
| 1 · K8s join, training-shaped | 30 | 390,000 | 30× |
| 2 · **pod labels, chart default allowlist** | 450 | **5,850,000** | **450×** |
| 3 · pod labels + anchored allowlist | 30 | 390,000 | 30× |
| 3 + `SM_ACTIVE`/`SM_OCCUPANCY` (M = 27) | 30 | 421,200 | 32× |

**The cost of the fix you actually want** — enabling the two honest fields — is
`520 × 2 × 30 = 31,200` extra series, an 8% increase on posture 3. **The cost of the
mistake next to it is 450×.** Say that sentence in the interview; it is the whole lesson in
one line.

**Turning series into memory and disk.** Series count is not the resource; it is a proxy for
two of them.

*Memory.* Prometheus holds an index entry plus a head chunk per active series. Published
estimates vary with label width and query load, roughly **3–8 KB per active series** in
resident memory; treat it as a band, measure your own with
`prometheus_tsdb_head_series` against `process_resident_memory_bytes`, and remember Go heap
and mmapped chunks put actual RSS well above the naive product.

```
  posture 3 (421,200 active)  ×  3 KB  ≈  1.3 GB
  posture 3 (421,200 active)  ×  8 KB  ≈  3.4 GB       ← plan for the top of the band
  posture 2 (5,850,000)       ×  3 KB  ≈  17.6 GB
  posture 2 (5,850,000)       ×  8 KB  ≈  46.8 GB      ← this is the OOM
```

Note the label-width term hiding in that constant. dcgm-exporter's renderer stamps eight
labels before any Kubernetes enrichment, and `modelName` alone is ~24 characters. A DCGM
series is *wide*, so it sits at the upper end of any per-series estimate.

*Disk.* Prometheus' own storage docs give the formula and the constant:
`needed_disk_space = retention_time_seconds × ingested_samples_per_second × bytes_per_sample`,
with *"Prometheus stores an average of only 1-2 bytes per sample"*.

```
  posture 3, 30 s scrape:
     421,200 series / 30 s            = 14,040 samples/s
     14,040 × (15 × 86,400) s × 2 B   ≈ 36.4 GB

  posture 2, 30 s scrape (only the ACTIVE fraction is being scraped —
  the stale 5.85M live in old blocks, so this understates block size):
     ~39,000 active / 30 s            =  1,300 samples/s
     …but the index for 5.85M series is carried in every block it touched.
```

**The asymmetry is the lesson: high cardinality is cheap on disk and expensive in RAM**,
because samples compress to 1–2 bytes while every distinct series costs a fixed index and
head-chunk allocation. That is why "Prometheus OOMed" and not "Prometheus filled the disk".

**Guardrails worth setting regardless.** Prometheus scrape configs support per-scrape limits
that fail the scrape rather than ingesting an explosion, all defaulting to `0` (unlimited):

```yaml
scrape_configs:
  - job_name: dcgm-exporter
    scrape_interval: 30s
    scrape_timeout: 25s
    honor_labels: false
    # 520 GPUs / 65 nodes = 8 GPUs/node × 27 counters = 216 samples/target.
    # 500 gives headroom for MIG instances and DCGM_EXP_* without hiding a blowup.
    sample_limit: 500
    # 8 renderer labels + driver version + pod/namespace/container + 3 allowlisted
    # pod labels = ~15. 25 catches a runaway allowlist at the door.
    label_limit: 25
    label_value_length_limit: 128
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: nvidia-dcgm-exporter
        action: keep
```

A tripped `sample_limit` fails the whole scrape and shows up as `up == 0` — loud, immediate,
and infinitely better than discovering the problem when the OOM killer does.

## Perspectives

**Developer / config.** The counter set is a deploy-time file, not a query-time relabel.
Every instinct from application metrics — "just add a relabel rule", "drop it at scrape
time", "we can recompute it later" — fails here, because the data was never produced. The
mental model to import is closer to a kernel `perf` event list than to a Prometheus
exporter: you are choosing what the hardware records, and the choice is made before the
process starts.

**Operator / fleet.** Cardinality is per-physical-GPU and multiplies by label-value churn
over the retention window, not by instantaneous GPU count. The two operational habits that
matter: (1) enable pod labels and the allowlist in the *same* change, never in separate PRs,
because the chart's default the moment you flip the boolean is "all labels"; (2) keep a CI
diff between your custom CSV and the default shipped in the image tag you deploy, so a
version bump surfaces as a reviewable change rather than a silent metric loss.

**Hardware / protocol.** Because PROF fields share hardware counter resources on pre-GPM
silicon (05.2 §5), "just enable everything" is not purely a Prometheus-side decision — it
changes what DCGM samples per interval before a single byte is exported. The CPU-exporter
instinct that cardinality is entirely a storage problem does not carry over. On Hopper and
newer it *does* mostly carry over, since all the ratio fields share one metric group, which
is a good reason to stop treating the conservative default as a law of nature.

**Economics.** The default costs money twice. It costs you the waste you cannot see, because
`SM_ACTIVE` is off — and on a 500-GPU fleet at 2026 prices that is a seven-figure blind
spot. Then, when you go to fix it, the adjacent default costs you the metrics bill, because
the enrichment that makes the metric useful per team defaults to unbounded. The competent
version of this change is five lines and costs about 8% more series. The incompetent version
is one line and costs 450×. **The difference between them is entirely knowledge, which is
why it is a good interview question.**

## Real-world use cases

- **`NVIDIA/dcgm-exporter` — `etc/default-counters.csv` at 4.8.3.** Reproduced in full in
  §2. **What it shows:** `DCGM_FI_PROF_SM_ACTIVE` and `DCGM_FI_PROF_SM_OCCUPANCY` are
  commented out *today*, on the current release, while `DCGM_FI_PROF_GR_ENGINE_ACTIVE` — a
  presence duty cycle — is enabled. This is not a stale claim about an old version; verify it
  yourself with one `curl` against the raw file, and again inside your own running pod,
  because the file drifts.

- **`NVIDIA/dcgm-exporter` — `grafana/dcgm-exporter-dashboard.json`** (published as
  grafana.com dashboard 12239, and linked from the exporter's own README as *"the official
  NVIDIA DCGM-Exporter dashboard"*). Grepping the shipped JSON for `DCGM_FI_*` yields exactly
  six distinct metrics across eight panels: `GPU_TEMP` (×6 references), `POWER_USAGE` (×2),
  `SM_CLOCK`, `GPU_UTIL`, `FB_USED`, `PIPE_TENSOR_ACTIVE`. **What it shows:** the most widely
  installed GPU dashboard in the world leads with the presence metric and has no breadth
  panel — because the series does not exist. The four-layer chain in §3 is not a thought
  experiment; it is this file, that CSV, and one `#`.

- **The GPU Operator's DaemonSet asset (`assets/state-dcgm-exporter/0800_daemonset.yaml`).**
  It sets `DCGM_EXPORTER_COLLECTORS=/etc/dcgm-exporter/dcp-metrics-included.csv`,
  `DCGM_EXPORTER_KUBERNETES=true`, `DCGM_EXPORTER_LISTEN=:9400`, mounts
  `/var/lib/kubelet/pod-resources`, runs the container `privileged: true`, and gates startup
  on the `toolkit-ready` barrier file from module 04. **What it shows:** installing "the
  same" exporter through the Operator gives you a *different counters file path*, a
  *different privilege model*, and a *different ServiceMonitor interval* (15 s vs 30 s). If
  you inherit a cluster, check which path is in play before you debug anything. And the
  punchline: the two counter files have identical enabled sets, so the Operator does not
  rescue you from the audit.

- **The chart defaults as a cardinality time bomb.** `deployment/values.yaml` ships
  `kubernetes.enablePodLabels: false` alongside `kubernetes.podLabelAllowlistRegex: []`, with
  the comment *"Empty list means all labels are included (default behavior)"*, and the
  Operator's `values.yaml` ships the allowlist commented out entirely. The exporter's
  `newLabelFilterCache()` implements this literally: `enabled: len(patterns) > 0`.
  **What it shows:** the ~450× blowup in §7 posture 2 is the chart's own documented
  behaviour, not a contrived misconfiguration. It also shows a second failure path worth
  knowing: patterns that fail to compile disable filtering entirely with a warning, so a
  regex typo fails *open*.

- **Grafana Labs' cardinality-management guidance** — the general Prometheus-at-scale
  pattern of exporters ported with noisy label sets exploding a TSDB, and the standard
  remedies (allowlists at the source, `metric_relabel_configs` for what does arrive,
  per-scrape limits). **What it shows:** the GPU case is an instance of a well-understood
  class, not a novel problem — which is useful because it means the mitigations are boring
  and proven, and the only GPU-specific part is that half of them (relabeling) cannot help
  you at the collector layer.

## Worked example

**Fleet.** 65 nodes × 8 GPUs = 520 GPUs. Mixed: 40 nodes of A100-SXM4 running training jobs
through Kubeflow, 25 nodes of H100-SXM5 running vLLM inference. Prometheus retention 15 d.
Installed via GPU Operator. You have been asked to "add real utilisation to the GPU
dashboard" and you have one change window.

**Step 1 — establish the baseline, from Prometheus, before touching anything.**

```promql
# How many distinct DCGM metric names exist?
count(count by (__name__) ({__name__=~"DCGM_FI_.*"}))
  → 25

# Total DCGM series right now.
count({__name__=~"DCGM_FI_.*"})
  → 13,046

# The two we care about.
count(DCGM_FI_PROF_SM_ACTIVE)        → (no data)
count(DCGM_FI_PROF_SM_OCCUPANCY)     → (no data)

# What IS there from the profiling family?
count by (__name__) ({__name__=~"DCGM_FI_PROF_.*"})
  → DCGM_FI_PROF_GR_ENGINE_ACTIVE     520
    DCGM_FI_PROF_PIPE_TENSOR_ACTIVE   520
    DCGM_FI_PROF_DRAM_ACTIVE          520
    DCGM_FI_PROF_PCIE_TX_BYTES        520
    DCGM_FI_PROF_PCIE_RX_BYTES        520
```

13,046 ≈ 520 × 25 plus a handful of extras — the arithmetic checks out, which also confirms
`K = 1` today (no pod labels, and the pod/namespace labels are only present on GPUs
currently allocated).

**Step 2 — confirm the cause at the collector, not by inference.**

```console
$ kubectl -n gpu-operator exec ds/nvidia-dcgm-exporter -c nvidia-dcgm-exporter -- \
      printenv DCGM_EXPORTER_COLLECTORS
/etc/dcgm-exporter/dcp-metrics-included.csv

$ kubectl -n gpu-operator exec ds/nvidia-dcgm-exporter -c nvidia-dcgm-exporter -- \
      grep -n 'SM_ACTIVE\|SM_OCCUPANCY\|GR_ENGINE' /etc/dcgm-exporter/dcp-metrics-included.csv
72:DCGM_FI_PROF_GR_ENGINE_ACTIVE,   gauge, Ratio of time the graphics engine is active.
73:# DCGM_FI_PROF_SM_ACTIVE,          gauge, The ratio of cycles an SM has at least 1 warp assigned.
74:# DCGM_FI_PROF_SM_OCCUPANCY,       gauge, The ratio of number of warps resident on an SM.
```

There it is, in your own cluster, with line numbers. **Screenshot this** — it is a stronger
exhibit for the blog post than any diagram, because it is one `#` standing between a fleet
and its own cost data.

**Step 3 — check the hardware can afford it (05.2 §5).**

```console
# H100 node
$ dcgmi profile -l -i 0 | awk 'NR>4 {print $2}' | sort -u
A.0
B.0
C.0
# → 1002 and 1003 are in A.0 with 1004 and 1005. Already in the GPM snapshot. Free.

# A100 node
$ dcgmi profile -l -i 0 | awk 'NR>4 {print $2}' | sort -u
A.0
A.1
B.0
C.0
D.0
# → letter A has two subgroups. Adding 1002/1003 may add a multiplexing pass.
#   At a 30 s watch interval each group still gets ~15 s of observation, which
#   is plenty. Proceed, and validate the values on A100s specifically.
```

**Step 4 — build the CSV correctly.**

```console
$ kubectl -n gpu-operator exec ds/nvidia-dcgm-exporter -c nvidia-dcgm-exporter -- \
      cat /etc/dcgm-exporter/dcp-metrics-included.csv > dcgm-metrics.csv

$ sed -i 's|^# \(DCGM_FI_PROF_SM_ACTIVE,\)|\1|; s|^# \(DCGM_FI_PROF_SM_OCCUPANCY,\)|\1|' \
      dcgm-metrics.csv

$ kubectl -n gpu-operator exec ds/nvidia-dcgm-exporter -c nvidia-dcgm-exporter -- \
      cat /etc/dcgm-exporter/dcp-metrics-included.csv | diff - dcgm-metrics.csv
73c73
< # DCGM_FI_PROF_SM_ACTIVE,          gauge, The ratio of cycles an SM has at least 1 warp assigned.
---
> DCGM_FI_PROF_SM_ACTIVE,          gauge, The ratio of cycles an SM has at least 1 warp assigned.
74c74
< # DCGM_FI_PROF_SM_OCCUPANCY,       gauge, The ratio of number of warps resident on an SM.
---
> DCGM_FI_PROF_SM_OCCUPANCY,       gauge, The ratio of number of warps resident on an SM.
```

Two hunks. Nothing else changed. **That diff is the proof you extended rather than
replaced**, and it belongs in the pull request description.

```console
$ kubectl -n gpu-operator create configmap custom-dcgm-metrics --from-file=dcgm-metrics.csv
```

```yaml
# ClusterPolicy / Helm values
dcgmExporter:
  config:
    name: custom-dcgm-metrics       # key must be dcgm-metrics.csv
  enablePodLabels: true             # ── these two lines ──
  podLabelAllowlistRegex:           # ── go in TOGETHER ──
    - "^app$"
    - "^app\\.kubernetes\\.io/(name|instance|component)$"
    - "^(team|environment)$"
  serviceMonitor:
    interval: 30s                   # align with the 30 s collect interval
    scrapeTimeout: 25s
```

**Step 5 — verify, including the negative.**

```promql
count(DCGM_FI_PROF_SM_ACTIVE)      → 520     ✓  new
count(DCGM_FI_PROF_SM_OCCUPANCY)   → 520     ✓  new
count(DCGM_FI_DEV_XID_ERRORS)      → 520     ✓  NOT lost — you copied, not replaced
count(DCGM_FI_DEV_FB_USED)         → 520     ✓  NOT lost
count(count by (__name__) ({__name__=~"DCGM_FI_.*"}))
                                   → 27      ✓  25 + 2
```

And check that the allowlist landed rather than failing open:

```console
$ kubectl -n gpu-operator logs ds/nvidia-dcgm-exporter -c nvidia-dcgm-exporter \
      | grep -i 'pod label'
Compiled pod label allowlist pattern  pattern=^app$
Compiled pod label allowlist pattern  pattern=^app\.kubernetes\.io/(name|instance|component)$
Compiled pod label allowlist pattern  pattern=^(team|environment)$
Pod label filtering enabled  patterns=3 originalPatterns=3 cacheSize=…
```

If you instead see *"No valid regex patterns for pod label filtering, all labels will be
included"*, your regexes did not compile and **you are in posture 2 right now**. Roll back.

**Step 6 — measure the cardinality delta, don't estimate it.**

```promql
# Immediately after rollout
count({__name__=~"DCGM_FI_.*"})                    → 14,092    (+8%)
# 24 hours later, after a day of Kubeflow churn
count({__name__=~"DCGM_FI_.*"})                    → 42,300
# 15 days later, at steady state on the retention window
prometheus_tsdb_head_series                        → …
count({__name__=~"DCGM_FI_.*"})                    → ~420,000
```

The +8% is the two new counters. The growth to ~420k over the window is `K` filling in — the
pod-churn term from §7 — and it is the number that matters for capacity, not the day-one
figure. **Set a recording rule for `count({__name__=~"DCGM_FI_.*"})` before the change**, so
this curve is observed rather than reconstructed.

**Step 7 — the counterfactual, for the write-up.** Re-run posture 2 for ten minutes on one
node with the allowlist removed, measure the per-GPU combination count, and extrapolate:

```promql
# distinct label combinations currently attached to one GPU's SM_ACTIVE
count(count by (pod, namespace, container) (DCGM_FI_PROF_SM_ACTIVE{UUID="GPU-8f2c…"}))
```

With the allowlist: small and stable. Without it, on a Kubeflow node, each new run adds a
combination and the count only goes up. That is the graph for the blog post: **one boolean,
two curves, three orders of magnitude.**

**The numbers to write down for the deliverable:** series-per-GPU × fleet size for each of
the four postures, the counters list and allowlist regex that produced each, the measured
day-1 and day-15 series counts, and the `diff` proving you extended rather than replaced.

## Practice

Feeds ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md).
Steps 1–5 work fine against the
[fake GPU fleet lab](../../04-gpu-on-kubernetes/practice/fake-gpu-fleet/README.md); step 3's
`dcgmi profile -l` needs real hardware.

1. **Baseline.** Record `count(count by (__name__) ({__name__=~"DCGM_FI_.*"}))` and
   `count({__name__=~"DCGM_FI_.*"})`. Add a recording rule for both so you have a *curve*,
   not two points, by the time you finish.

2. **Audit in situ.** `kubectl exec` into a running exporter pod, print
   `DCGM_EXPORTER_COLLECTORS`, and `cat` that file. Diff it against the canonical upstream
   file for your image tag. Confirm — in *your* deployment, with line numbers —
   that `SM_ACTIVE` and `SM_OCCUPANCY` are commented and `GR_ENGINE_ACTIVE` is not.
   **Screenshot the grep.**

3. **Check the hardware budget.** `dcgmi profile -l` on one node of each GPU model. Record
   whether 1002/1003 share a metric group with 1004/1005 (free) or land in an exclusive
   subgroup (a multiplexing pass). Note the answer per model.

4. **Custom CSV, done properly.** Copy the full default out of the running pod, uncomment
   exactly two lines, `diff` to prove it, ship it as a ConfigMap, and verify with the five
   PromQL checks from step 5 of the worked example — including the *negative* checks that
   `XID_ERRORS` and `FB_USED` survived.

5. **Prove the trap.** Deliberately deploy a one-line CSV containing only `SM_ACTIVE`.
   Confirm `DCGM_FI_DEV_XID_ERRORS` disappears from Prometheus. Then roll back. **Do this
   once**; the memory of watching your hardware-fault alerting evaporate is worth more than
   reading about it.

6. **Cardinality experiment.** Snapshot the series count. Enable `enablePodLabels` with the
   default (empty) allowlist. Run a workload that churns pods with unique labels — a `for`
   loop creating short-lived Jobs with a `job-id` label works. Measure the series delta after
   one churn cycle and after an hour. Then add an anchored allowlist, confirm the *"Pod label
   filtering enabled"* log line, and re-measure.

7. **Fail it open on purpose.** Set `podLabelAllowlistRegex` to a deliberately broken pattern
   (`"^app[$"`). Confirm the exporter logs *"No valid regex patterns… all labels will be
   included"* and that filtering is silently off. This is the failure mode most likely to
   bite you in a real change window.

8. **Guardrails.** Add `sample_limit` and `label_limit` to the dcgm-exporter scrape config,
   sized from your own per-target sample count (`count by (instance) ({__name__=~"DCGM_FI_.*"})`).
   Prove the limit works by lowering it below the real value and watching `up` go to 0.

9. **Wire the CI diff.** A job that pulls the default CSV out of the pinned image, diffs it
   against your committed file ignoring your intentional uncomment hunks, and fails on
   anything else.

**Acceptance:** (a) a committed custom counters CSV that enables `SM_ACTIVE` and
`SM_OCCUPANCY` *and* retains the full default set, with the `diff` in the PR proving
replace-safety; (b) a cardinality note stating series-per-GPU × fleet size for all four
postures, with the allowlist regex you chose, the measured day-1 and day-15 series counts,
and the memory estimate band; (c) the in-pod grep screenshot showing the commented lines in
your own cluster. All three drop straight into the deliverable — (c) in particular is the
image that makes the blog post's argument concrete rather than theoretical.

## Common pitfalls

1. **Believing `metric_relabel_configs` can resurrect a field DCGM never collected.**
   **Symptom:** hours spent on relabel rules for a metric that has no series.
   **Mechanism:** relabeling operates on samples present in the scrape response; a commented
   CSV line means the field is not in the field group, no watch exists, the engine never
   sampled it, and the exporter never rendered it. Fix at the collector or not at all.

2. **Shipping a custom CSV containing only the new field.** **Symptom:** `SM_ACTIVE` appears
   and, days later, someone notices XID alerting has been silent since the deploy.
   **Mechanism:** the supplied file *is* the counter set; there is no union. The YAML config
   states it outright — `metrics.file` and `metrics.fields` are mutually exclusive metric
   sources. **Fix:** always start from the full default extracted from the deployed image,
   and put the diff in CI.

3. **Turning on `enablePodLabels` without `podLabelAllowlistRegex` in the same change.**
   **Symptom:** Prometheus RSS climbs for a week and then the pod OOMs.
   **Mechanism:** `newLabelFilterCache()` sets `enabled: len(patterns) > 0`; an empty list
   means no filtering, and both the chart and the Operator ship it empty. This is a
   documented default, not a mistake you have to make.

4. **Assuming a compiled allowlist means a working allowlist.** **Symptom:** you set the
   regexes, cardinality explodes anyway. **Mechanism:** patterns that fail to compile are
   skipped with a warning, and if *all* of them fail the exporter disables filtering with
   *"No valid regex patterns for pod label filtering, all labels will be included"* — it
   fails **open**. **Fix:** grep the startup logs for *"Pod label filtering enabled"* and the
   pattern count, every time.

5. **Predicting Prometheus label names from Kubernetes label names.** **Symptom:** your
   `group by (app.kubernetes.io/name)` does not parse, or matches nothing.
   **Mechanism:** `SanitizeLabelName()` maps invalid characters to `_`, so it arrives as
   `app_kubernetes_io_name`; and a pod label colliding with a renderer-owned label
   (`gpu`, `device`, `modelName`, `UUID`, `hostname`, `pci_bus_id`) is renamed with a
   `pod_label_` prefix, plus `_conflictN` if that also collides.

6. **Assuming the GPU Operator install is the standalone install.** **Symptom:** you edit the
   wrong file path, or you cannot find the `SYS_ADMIN` capability you expected.
   **Mechanism:** the Operator sets `DCGM_EXPORTER_COLLECTORS` to `dcp-metrics-included.csv`,
   runs the container `privileged: true` rather than with a capability list, and scrapes at
   15 s against a 30 s collect interval. Different path, different privilege model, different
   interval — same enabled counter set.

7. **Reading `DCGM_FI_DEV_XID_ERRORS` as a counter.** **Symptom:** `increase()` over it
   returns nonsense. **Mechanism:** it is a gauge holding the *last* XID code observed, not a
   count. The cumulative counter is `DCGM_EXP_XID_ERRORS_TOTAL`, which is commented out in
   the default set. Lesson 05.5 owns this; note it here because it is the same audit.

8. **Trusting the CSV's help strings as semantics.** **Symptom:** you tell an interviewer
   `PIPE_TENSOR_ACTIVE` is HMMA-only. **Mechanism:** column 3 is free text written years ago;
   the field maps to `NVML_GPM_METRIC_ANY_TENSOR_UTIL` and the HMMA-only field is 1014
   (05.1 §6). Semantics come from `dcgm_fields.h`, not from the exporter's CSV.

9. **Assuming `label`-typed rows produce series.** **Symptom:** your counter arithmetic is
   off by one. **Mechanism:** `DCGM_FI_DRIVER_VERSION, label, …` attaches a label to other
   metrics; it emits nothing itself. 26 enabled rows, 25 series per GPU.

## Self-check

- **Why is `SM_ACTIVE` missing from the default Grafana GPU dashboard?** *Answer:* because
  the series does not exist to be graphed. `DCGM_FI_PROF_SM_ACTIVE` ships **commented out** in
  dcgm-exporter's `default-counters.csv` (and in the `dcp-metrics-included.csv` the GPU
  Operator points at), so it is not in the field group, no DCGM watch is created, the engine
  never samples it, and no series reaches Prometheus. The official dashboard
  (grafana.com #12239, shipped as `grafana/dcgm-exporter-dashboard.json`) therefore has eight
  panels built on six metrics — `GPU_TEMP`, `POWER_USAGE`, `SM_CLOCK`, `GPU_UTIL`, `FB_USED`,
  `PIPE_TENSOR_ACTIVE` — and no breadth panel at all. It is a collector default, not a
  dashboard omission, and it is fixable only in the CSV.

- **Does a custom counters CSV extend or replace the default set?** *Answer:* **replace.** The
  file you supply becomes the entire counter set; there is no union with the shipped default.
  A file containing only `SM_ACTIVE` silently stops collecting `XID_ERRORS`, `FB_USED`,
  temperature, power and everything else. The 4.8.x YAML config makes it explicit —
  `metrics.file` and `metrics.fields` are mutually exclusive metric *sources*. The correct
  procedure is: extract the default from the exact image you deploy, uncomment only what you
  need, `diff` to prove exactly two hunks changed, ship the whole file, and keep that diff in
  CI so an upstream change surfaces as a review rather than a silent loss.

- **How many series per GPU does the default set produce, and why is it not 26?** *Answer:*
  25. The file has 26 uncommented field rows, but one of them —
  `DCGM_FI_DRIVER_VERSION, label, Driver Version` — is typed `label`, which attaches the
  value as a label on the other metrics rather than emitting a series of its own. So
  `count(count by (__name__) ({__name__=~"DCGM_FI_.*"}))` returns 25 on a stock install, and
  27 after you uncomment `SM_ACTIVE` and `SM_OCCUPANCY`.

- **What is the actual default of `podLabelAllowlistRegex`, and why does that make the
  cardinality blowup non-hypothetical?** *Answer:* `[]` — an empty list, in both the
  standalone chart (`kubernetes.podLabelAllowlistRegex: []`, documented as *"Empty list means
  all labels are included"*) and the GPU Operator (commented out entirely). The exporter
  implements it as `enabled: len(patterns) > 0`, so empty means filtering is off. Flipping
  `enablePodLabels: true` with no other change therefore *is* "no allowlist", and every pod
  label — including per-run `job-id` and `pod-template-hash` — becomes a series dimension.
  There is a second path to the same state: if all supplied patterns fail to compile, the
  exporter logs *"No valid regex patterns for pod label filtering, all labels will be
  included"* and disables filtering. It fails open.

- **Work the cardinality for 520 GPUs, 27 counters, 30 pods per GPU over a 15-day window,
  with and without an allowlist.** *Answer:* `series = GPUs × counters × distinct label
  combinations per GPU over retention`. With an anchored allowlist admitting only
  service-level labels, `K` ≈ the pod-churn number ≈ 30, so `520 × 27 × 30 ≈ 421,000` series.
  Without it, per-run labels make `K` ≈ 30 pods/day × 15 days = 450, so
  `520 × 27 × 450 ≈ 6.3M` series — about 15× more, or 450× the no-enrichment baseline. At a
  3–8 KB-per-active-series band that is roughly 1.3–3.4 GB versus 19–50 GB of Prometheus
  RSS. Disk is not the constraint: at 1–2 bytes per sample and a 30 s scrape, even the large
  case is tens of GB over 15 days. **High cardinality is cheap on disk and expensive in RAM.**

- **Name a risky pod label to enrich on across 500 GPUs, and one that is safe.** *Answer:*
  risky — `job-id`, `pod-template-hash`, `controller-revision-hash`, `batch.kubernetes.io/job-name`,
  or any operator-injected UUID: unbounded, one fresh value per run, multiplying retained
  series by the number of runs per GPU over the retention window. Safe — `app`,
  `app.kubernetes.io/name`, `app.kubernetes.io/instance`, `app.kubernetes.io/component`,
  `team`, `environment`: one value per service, so `K` stays at the pod-churn number. Bound
  them with anchored regexes (`^app$`, not `app`), and remember they arrive in Prometheus
  sanitised (`app_kubernetes_io_name`).

- **A colleague adds `metric_relabel_configs` to drop `pod_template_hash` and reports that
  cardinality did not improve. What happened?** *Answer:* dropping a label at scrape time
  does reduce the *stored* series going forward, but it does not stop the exporter from
  producing them, does not reduce the exporter's own work, and — critically — does not remove
  the series already in the head block and in every block within the retention window. It
  also merges previously-distinct series into one, which can produce duplicate-sample errors
  if two series collapse to the same label set at the same timestamp. The fix belongs at the
  source: `podLabelAllowlistRegex`, so the label is never emitted.

- **Which two defaults would you change on day one of inheriting a GPU cluster, and what do
  they cost?** *Answer:* (1) uncomment `DCGM_FI_PROF_SM_ACTIVE` and
  `DCGM_FI_PROF_SM_OCCUPANCY` in a full copy of the shipped counters file — cost: two extra
  series per GPU, about +8% on a fleet that already has pod enrichment, and possibly one
  multiplexing pass on pre-GPM silicon which a 30 s watch interval absorbs comfortably.
  (2) Set `podLabelAllowlistRegex` to anchored service-level patterns *before or in the same
  change as* `enablePodLabels: true` — cost: nothing; it prevents a 450× blowup. Secondary:
  align the ServiceMonitor interval with `--collect-interval` (the Operator's 15 s against a
  30 s collect interval is a pure 2× oversample), and add `sample_limit` / `label_limit` to
  the scrape config so a future explosion fails loudly instead of silently.

## Connections & what's next

This lesson is the hinge between 05.2's explanation of *why* profiling costs what it costs
and 05.4's aggregation of truthful metrics into per-namespace GPU-hours. Without `SM_ACTIVE`
actually landing in Prometheus here, 05.4 has nothing to join and the capstone dashboard has
nothing honest to render. The label inventory in §6 is 05.4's raw material: which labels
exist, where each comes from, and why `honor_labels: false` means you will be writing
`exported_namespace`. The cardinality discipline carries forward too — 05.4's recording rules
and the capstone's per-namespace panels are exactly the kind of query that turns a
high-cardinality mistake into a slow dashboard. Backwards, this lesson closes the loop on
05.1: the metric that lesson told you to page on is the one the default config does not
collect, and now you know it is one character and one CI job away. Next:
[05.4 — Attribution](04-attribution.md) takes these now-truthful, now-labelled series and
turns them into "who holds versus who uses", including the case where the join is
mathematically impossible.

## References & further reading

**Primary sources**

- `NVIDIA/dcgm-exporter` — `etc/default-counters.csv` (4.8.3) — https://github.com/NVIDIA/dcgm-exporter/blob/main/etc/default-counters.csv — the file reproduced in full in §2. Read for: which fields ship enabled, which are commented, and the double-listing of `DCGM_FI_PROF_PCIE_TX/RX_BYTES`. *Correction vs. the previous version of this lesson: the enabled set is 26 rows (25 series + 1 label), not 8 — it includes clocks, temperature, power, ECC-adjacent remapped-row counters, framebuffer and NVLink aggregate bandwidth, and the PCIe PROF fields ARE enabled (as gauges, at the bottom of the file).*
- `NVIDIA/dcgm-exporter` — `etc/dcp-metrics-included.csv` — https://github.com/NVIDIA/dcgm-exporter/blob/main/etc/dcp-metrics-included.csv — the file the GPU Operator actually points at. Read for: confirmation that its enabled set is identical to `default-counters.csv` (the two differ only in comment blocks and whitespace).
- `NVIDIA/dcgm-exporter` — `README.md` — https://github.com/NVIDIA/dcgm-exporter#changing-metrics — read for: the CSV grammar and the "always 2 commas" rule; the `-f`/`--collectors` flag; the `--config-file` YAML schema with `metrics.file` / `metrics.fields` declared mutually exclusive; `collection.watchGroups`; and the `DCGM_EXP_XID_ERRORS_TOTAL` / `DCGM_EXP_CLOCK_EVENTS_TOTAL` opt-in counters with their `_COUNT`-vs-`_TOTAL` semantics.
- `NVIDIA/dcgm-exporter` — `deployment/values.yaml` — https://github.com/NVIDIA/dcgm-exporter/blob/main/deployment/values.yaml — read for: `kubernetes.enablePodLabels: false`, `kubernetes.podLabelAllowlistRegex: []` with the *"Empty list means all labels are included"* comment and the three example regexes; the `securityContext` with `SYS_ADMIN`; `serviceMonitor.interval: 30s` / `scrapeTimeout: 25s` / `honorLabels: false`; and the pod resource requests and limits.
- `NVIDIA/dcgm-exporter` — `internal/pkg/rendermetrics/render_metrics.go` — https://github.com/NVIDIA/dcgm-exporter/blob/main/internal/pkg/rendermetrics/render_metrics.go — read for: the exact renderer-owned label set per entity group (`gpu`, `UUID`, `pci_bus_id`, `device`, `modelName`, `GPU_I_PROFILE`, `GPU_I_ID`, `hostname`) used in §6's table.
- `NVIDIA/dcgm-exporter` — `internal/pkg/transformation/kubernetes.go` and `internal/pkg/utils/utils.go` — https://github.com/NVIDIA/dcgm-exporter/tree/main/internal/pkg/transformation — read for: `newLabelFilterCache()` (`enabled: len(patterns) > 0`, and the fail-open behaviour when all patterns fail to compile), `copyPodLabels()` / `availablePodLabelName()` (the `pod_label_` prefix applied only on collision, with `_conflictN`), and `SanitizeLabelName()`.
- `NVIDIA/dcgm-exporter` — `internal/pkg/counters/types.go` — https://github.com/NVIDIA/dcgm-exporter/blob/main/internal/pkg/counters/types.go — read for: `IsProfilingMetric()` as a literal `DCGM_FI_PROF_` prefix check, and `LabelCounters()` — the mechanism behind `label`-typed rows not producing series.
- `NVIDIA/dcgm-exporter` — `grafana/dcgm-exporter-dashboard.json` — https://github.com/NVIDIA/dcgm-exporter/blob/main/grafana/dcgm-exporter-dashboard.json — read for: the eight panels and six metrics of the official dashboard (grafana.com #12239), and the absence of `SM_ACTIVE`, `SM_OCCUPANCY` and `DRAM_ACTIVE`.
- `NVIDIA/gpu-operator` — `assets/state-dcgm-exporter/0800_daemonset.yaml` and `0210_clusterrole.yaml` — https://github.com/NVIDIA/gpu-operator/tree/master/assets/state-dcgm-exporter — read for: `DCGM_EXPORTER_COLLECTORS=/etc/dcgm-exporter/dcp-metrics-included.csv`, `DCGM_EXPORTER_KUBERNETES=true`, `DCGM_EXPORTER_LISTEN=:9400`, `privileged: true`, the `/var/lib/kubelet/pod-resources` hostPath, the `toolkit-ready` init-container barrier, and the `nvidia-dcgm-exporter-read-pods` ClusterRole.
- `NVIDIA/gpu-operator` — `deployments/gpu-operator/values.yaml`, `dcgmExporter` block — https://github.com/NVIDIA/gpu-operator/blob/master/deployments/gpu-operator/values.yaml — read for: the `config.name` / `config.create` / `config.data` contract and the requirement that the ConfigMap key be `dcgm-metrics.csv`; `enablePodLabels: false`; `serviceMonitor.interval: 15s` / `scrapeTimeout: 10s` / `honorLabels: false`; and the pinned exporter image tag.
- Prometheus — configuration reference — https://prometheus.io/docs/prometheus/latest/configuration/configuration/ — read for: `honor_labels` (default `false`, and the `exported_<label>` renaming rule), `sample_limit`, `label_limit`, `label_name_length_limit`, `label_value_length_limit`, `target_limit` — all defaulting to `0` (unlimited) — and the `scrape_interval` / `scrape_timeout` defaults.
- Prometheus — storage — https://prometheus.io/docs/prometheus/latest/storage/ — read for: `needed_disk_space = retention_time_seconds × ingested_samples_per_second × bytes_per_sample`, the *"1-2 bytes per sample"* figure used in §7, and the 15 d default retention.

**Real-world engineering blogs**

- Grafana Labs — "How to manage high cardinality metrics in Prometheus and Kubernetes" — https://grafana.com/blog/how-to-manage-high-cardinality-metrics-in-prometheus-and-kubernetes/ — what it shows: the cardinality-explosion pattern generalises far beyond GPUs, and the standard remedies (allowlist at the source, relabel what arrives, per-scrape limits) are boring and proven. The GPU-specific twist is that relabeling cannot help at the collector layer.
- Datadog — GPU Monitoring Reference Architecture — https://www.datadoghq.com/architecture/gpu-monitoring/ — what it shows: a paid vendor's production taxonomy is organised around exactly the enabled/commented distinction this lesson audits — device-level `sm_active` alongside process-level `process.sm_active` — which is a useful sanity check that the field this default omits is the one a commercial product considers central.

**Deeper dives**

- Grafana Labs — "NVIDIA DCGM Exporter Dashboard" (community dashboard 12239) — https://grafana.com/grafana/dashboards/12239-nvidia-dcgm-exporter-dashboard/ — go deeper on: install it against your pre-audit fleet, then against your post-audit fleet, and note that nothing changes until you add the panel yourself. The dashboard is downstream of the collector, and that is the whole lesson.
</content>
