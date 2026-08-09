---
lesson: "05.2"
title: "DCGM architecture and the cost of profiling"
module: "05"
concept: "DCGM architecture and the cost of profiling"
status: not-started
est_time: "4h"
artifacts: []
---

# 05.2 · DCGM architecture and the cost of profiling

> **Concept.** DCGM's PROF metrics are a separate, hardware-counter-backed sampled subsystem with real costs — multiplexing and a bounded averaging window — and choosing scrape intervals against those costs is the fleet-scale judgment the interview probes.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Why this matters

Lesson 05.1 said "read the PROF fields, not GPU_UTIL." This lesson is why that is not
free. The profiling metrics are hardware performance counters, and on real fleets you
cannot naively request all of them at 1-second resolution on every GPU — you hit
multiplexing (zeros) and sampling-cost limits. A senior platform interview at
NVIDIA/CoreWeave/Datadog will ask "you want SM_ACTIVE, TENSOR_ACTIVE, DRAM_ACTIVE, and
PCIe/NVLink counters at 1s on 5,000 GPUs — what breaks?" The correct answer is this
lesson: which fields co-reside, what multiplexing does to your data, and the scrape
interval the counter windows actually justify.

## What's new for you

Module **04** had you *deploy* dcgm-exporter and do the pod-resources join; you already
run Prometheus scrape configs and understand scrape intervals as an operational knob.
**That is not re-taught.** What's new: DCGM's *internal* architecture — `nv-hostengine`
standalone vs embedded mode, fields vs field *groups*, and specifically the **profiling
module** as a distinct sampled subsystem with two costs you must reason about —
**counter multiplexing** (requesting more concurrent PROF fields than the hardware can
co-collect) and the **bounded averaging window** of the tensor-style counters. This is
where "I know DCGM" becomes "I know how DCGM's sampling actually costs and lies at fleet
scale."

## Core notes

### Hostengine: standalone vs embedded

DCGM's engine is **`nv-hostengine`** — the process that owns the field cache, talks to
the driver/NVML and the profiling module, and answers field queries. It runs in one of
two modes:

- **Standalone (daemon) mode.** `nv-hostengine` runs as its own long-lived process/service
  on the node, listening on a socket (default TCP `:5555`, or a unix socket). Clients —
  `dcgmi`, your own bindings, or an exporter — connect over the DCGM API. One engine per
  node serves many clients; this is what you run when multiple consumers (an exporter
  *and* ad-hoc `dcgmi`) need the same GPUs, and it centralises the profiling module so two
  consumers don't fight over counters.
- **Embedded mode.** The DCGM engine is loaded **in-process** as a shared library — there
  is no separate daemon; the consuming process *is* the host engine. Lower overhead and no
  socket, but the engine's lifetime is the process's lifetime and only that process can
  use it.

**dcgm-exporter uses embedded mode by default** — it loads DCGM in-process (via the Go
bindings / `libdcgm`) and owns its own host engine. That matters operationally: because
the exporter embeds the engine and holds the profiling module, running `dcgmi` *also*
asking for PROF fields on the same GPUs can contend. You can instead point dcgm-exporter
at a *remote* standalone `nv-hostengine` (`-r <host:port>`) so the exporter and other
clients share one engine — the standard pattern when you need both the exporter and
node-level diagnostics without counter contention.

### Fields and field groups

Everything DCGM exposes is a **field** with a stable integer ID (`DCGM_FI_*`, e.g. 203,
1002, 252 — lesson 05.1). You don't query fields one call at a time at scale; you create
a **field group** — a named set of field IDs — and a **watch** on it with an
`updateFreq` (microseconds), a `maxKeepAge`, and a `maxKeepSamples`. The engine then
samples those fields into its cache on that cadence, and readers pull from the cache.
dcgm-exporter's config (the `-f`/CSV field file, historically
`dcp-metrics-included.csv`) is exactly this: the list of field IDs it watches and the
update frequency it watches them at. So "what do I scrape" decomposes into two knobs:
the **field group** (which IDs) and the **update frequency** (how often the engine
samples), which are *distinct from* your Prometheus scrape interval (how often you pull
the already-sampled cache).

### The profiling subsystem is a different, costed thing

Fields **1001–1012** (the `DCGM_FI_PROF_*` family: GR_ENGINE_ACTIVE, SM_ACTIVE,
SM_OCCUPANCY, PIPE_TENSOR_ACTIVE, DRAM_ACTIVE, the FP64/32/16 pipes, PCIe TX/RX, NVLink
TX/RX) do **not** come from NVML polling. They come from the **profiling module**, which
programs the GPU's **hardware performance counters** and samples them. This is a
fundamentally more expensive and more constrained path than the NVML device fields
(100s/200s), and it has two properties you must design around.

#### 1) Multiplexing — the hardware can't collect every counter at once

The GPU has a **finite number of counter-collection resources**, and many PROF metrics
map onto overlapping hardware counters. When the set of PROF fields you ask the engine
to watch **exceeds what can be gathered in a single pass**, DCGM does **not** error — it
**time-multiplexes** (round-robins) the groups across successive sample intervals. The
consequences:

- Each metric is now only sampled a *fraction* of the time, so each individual field's
  effective resolution drops even though your `updateFreq` says 1s.
- During intervals where a given field is not the one currently being collected, **DCGM
  reports it as `0.0`** (or a stale/held value) rather than a real reading. On a dashboard
  this looks like a metric that keeps dropping to zero — the classic "why is TENSOR_ACTIVE
  flickering to 0" symptom is over-subscribed profiling, not an idle GPU.

The practical rule: **keep the concurrently-watched PROF set small** (the four from 05.1
— SM_ACTIVE, SM_OCCUPANCY, TENSOR_ACTIVE, DRAM_ACTIVE — plus a couple of link counters
is a reasonable co-resident set on most architectures) and, if you need more, accept a
*longer* effective interval or split them across watches. Which fields can co-reside is
architecture-dependent; `dcgmi profile -l` lists what the GPU supports and the metric
*groups* it organises them into, and metrics in *different* groups are the ones prone to
multiplexing when watched together.

Also: the profiling module can be **paused and resumed** (`dcgmi profile --pause` /
`--resume`). Pausing releases the counters so an external profiler (Nsight
Compute/Systems, or a job that programs counters itself) can run without fighting DCGM —
and while paused, PROF fields return no fresh data. Any external tool that grabs the
counters can likewise make DCGM's PROF fields go blank; the two cannot own the hardware
counters simultaneously.

#### 2) The bounded averaging window of the activity counters

The activity ratios (SM_ACTIVE, TENSOR_ACTIVE, DRAM_ACTIVE, etc.) are **averages of
counter deltas over the sampling interval**, and the engine maintains them over a
bounded window — on the order of **≤30 seconds** for the tensor/activity counters. Two
failure modes follow:

- **Scrape too *slowly* (Prometheus interval ≫ the window):** you under-sample. If the
  engine's usable averaging window is ~30s and you scrape every 60–120s, you are reading a
  value that only reflects the last ~30s of a 2-minute gap — you miss bursts, and a bursty
  training step looks quiet. So the **maximum useful scrape interval is set by that ~30s
  window: don't scrape PROF activity fields slower than ~30s** or you're sampling a window
  shorter than your gap.
- **Scrape too *fast* (sub-second `updateFreq`):** you over-cost. Programming and reading
  hardware counters is not free; pushing `updateFreq` below ~1s multiplies profiling
  overhead across the fleet (and worsens multiplexing) for resolution the ~30s-averaged
  counters can't even express meaningfully. ~1s is the sane floor.

**The fleet judgment:** a PROF `updateFreq` around **1s** at the engine, and a
Prometheus **scrape interval in the ~10–30s band** for the PROF fields, keeps you inside
both bounds — fast enough to sit under the averaging window, slow enough to keep counter
overhead and cardinality sane across thousands of GPUs. Cheap NVML fields (util, memory,
power, temp, clocks — 100s/200s) can be scraped faster without profiling cost if you
want.

## Worked example

You configure dcgm-exporter's field CSV to watch, at `updateFreq` 1s on a single A100:

```
1001 GR_ENGINE_ACTIVE
1002 SM_ACTIVE
1003 SM_OCCUPANCY
1004 PIPE_TENSOR_ACTIVE
1005 DRAM_ACTIVE
1006 PIPE_FP64_ACTIVE
1007 PIPE_FP32_ACTIVE
1008 PIPE_FP16_ACTIVE
1009 PCIE_TX_BYTES
1010 PCIE_RX_BYTES
1011 NVLINK_TX_BYTES
1012 NVLINK_RX_BYTES
```

You run `dcgmi dmon -e 1002,1004,1005,1008` and watch a training job. Observation:
`SM_ACTIVE` holds ~0.85 steadily, but `PIPE_TENSOR_ACTIVE`, `PIPE_FP16_ACTIVE`, and
`DRAM_ACTIVE` **each intermittently read exactly 0.0** and then jump back to plausible
values, roughly in rotation. That is not the GPU idling — it is **multiplexing**: you
asked for more PROF fields than the A100 can co-collect in one pass, so DCGM is
round-robining the pipe/activity counters and reporting 0.0 for whichever group isn't
currently sampled. Reducing the watched set to the four load-bearing fields (1002, 1003,
1004, 1005) makes the zeros disappear and every field read continuously. **Conclusion
for the fleet:** watch the minimal PROF set per GPU, at 1s engine `updateFreq`, scraped
by Prometheus every ~15s; add the extra pipe/link counters only on a diagnostic profile
you enable on demand, not fleet-wide.

## Practice

On a rented GPU with DCGM installed:

1. **Discover.** `dcgmi discovery -l` to enumerate GPUs; `dcgmi profile -l` to list the
   PROF metrics the GPU supports and the metric *groups* it organises them into (note which
   metrics live in different groups — those are your multiplexing candidates).
2. **Monitor.** `dcgmi dmon -e 1002,1004,1005` to stream SM_ACTIVE / TENSOR_ACTIVE /
   DRAM_ACTIVE while a workload runs.
3. **Force multiplexing.** Deliberately request too many PROF fields at once at 1s — e.g.
   `dcgmi dmon -e 1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1011,1012 -d 1000` —
   and observe fields **flickering to 0.0** as DCGM round-robins. Note which fields zero
   out together (same/different group).
4. **Pause/resume.** `dcgmi profile --pause`, confirm PROF fields stop updating
   (blank/held), run something that needs the counters, then `dcgmi profile --resume` and
   confirm they return.

**Acceptance:** a written note capturing (i) the observed multiplexing — which fields
returned 0.0 when over-subscribed and roughly the rotation — and (ii) your chosen
**fleet PROF scrape interval and engine `updateFreq`, with the reason** (the ~30s
averaging-window upper bound and the ~1s cost/multiplexing lower bound). This note is
the "why our scrape config is what it is" appendix of the deliverable.

## Self-check

**(a) Why can't you sample every `PROF_*` field at 1s on one GPU?**
**Answer:** The PROF fields are backed by a *finite* set of GPU hardware performance
counters, and many metrics map onto overlapping counters. Requesting more concurrent
PROF fields than the hardware can collect in one pass forces DCGM to time-multiplex
(round-robin) the groups; DCGM doesn't error — it returns `0.0` for whichever field
isn't currently being sampled and drops each field's effective resolution. So "all
fields at 1s" gives you flickering zeros, not more data. Keep the co-resident set small.

**(b) What's the maximum useful scrape interval given the tensor counter's ≤30s
window?**
**Answer:** ~30 seconds. The activity/tensor counters are averaged over a bounded window
on the order of 30s; scraping slower than that means your Prometheus gap exceeds the
window the value represents, so you under-sample and miss bursts. Practically: engine
`updateFreq` ~1s, Prometheus scrape in the ~10–30s band — never slower than ~30s for
PROF activity fields.

**(c) Hostengine vs embedded — which does dcgm-exporter use?**
**Answer:** dcgm-exporter runs the DCGM engine **embedded** (in-process, via `libdcgm`)
by default — no separate `nv-hostengine` daemon, and it owns the profiling module
itself. You can override this to connect to a **remote standalone `nv-hostengine`** (`-r
host:port`) when you need the exporter and other clients (e.g. `dcgmi`) to share one
engine and not contend over the hardware counters.

## Resources

1. **DCGM profiling API / feature overview** —
   https://docs.nvidia.com/datacenter/dcgm/latest/dcgm-api/dcgm-api-profiling.html —
   *deep.* The authoritative description of the profiling module, the supported PROF
   metrics, counter constraints, and the multiplexing behaviour; the source to verify how
   many fields co-reside on a given architecture and how pause/resume interacts with
   external profilers.
2. **DCGM User Guide — feature overview & `dcgmi`** —
   https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html —
   *skim.* Grounds the hostengine (standalone vs embedded) architecture, field groups /
   watches / `updateFreq`, and the `dcgmi discovery`/`dmon`/`profile` commands you use in
   practice; skim for the architecture picture, reference for exact flags.
