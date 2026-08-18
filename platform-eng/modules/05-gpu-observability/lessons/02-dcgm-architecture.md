---
lesson: "05.2"
title: "DCGM architecture and the cost of profiling"
module: "05"
concept: "DCGM architecture and the cost of profiling"
status: not-started
est_time: "6h"
prev: "01-lie-and-truth.md"
next: "03-dcgm-exporter-at-scale.md"
artifacts: []
sources: 7
---

# 05.2 · DCGM architecture and the cost of profiling

> **Concept.** DCGM's PROF metrics are a separate, hardware-counter-backed sampled subsystem with real costs — multiplexing, superuser privileges, and a bounded averaging window — and choosing scrape intervals against those costs is the fleet-scale judgment the interview probes.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Where this fits

Lesson 05.1 told you exactly which fields to trust — `SM_ACTIVE`, `SM_OCCUPANCY`,
`PIPE_TENSOR_ACTIVE`, `DRAM_ACTIVE` — and which one (`GPU_UTIL`) to keep off every alert.
It did not explain *how* DCGM produces those trustworthy fields, or why they sometimes
silently go missing. This lesson closes that gap: the profiling subsystem that backs the
PROF fields is architecturally distinct from the NVML polling that backs `GPU_UTIL`, it
runs against a **finite hardware resource**, and it requires privileges NVML doesn't. If
you deploy the fields from 05.1 without understanding this lesson, you will eventually
watch `TENSOR_ACTIVE` flicker to `0.0` on a healthy, busy GPU and misdiagnose it as an
idle workload — the opposite of the lie this module exists to catch. What this unlocks:
lesson 05.3's cardinality math and lesson 05.5's health-field wiring both assume you
understand field groups, watches, and update frequency as first-class fleet
configuration, not scrape-interval trivia.

## Why this matters

A senior platform interview at NVIDIA, CoreWeave, or Datadog will not stop at "read
SM_ACTIVE instead of GPU_UTIL." It will follow with: "you want SM_ACTIVE, TENSOR_ACTIVE,
DRAM_ACTIVE, and PCIe/NVLink counters at 1-second resolution on 5,000 GPUs — what
breaks?" A candidate who says "nothing, just scrape faster" has never operated DCGM at
scale. The correct answer is this lesson: the GPU has a finite number of hardware
counter-collection slots, which fields co-reside is architecture-dependent, DCGM does not
error when you over-subscribe — it silently multiplexes and returns zeros — and
profiling metrics require privileges the container may not have. A live, real-world
version of exactly this question is sitting in NVIDIA's own DCGM issue tracker: a user
running dcgm-exporter and a custom health-check container as two separate embedded
engines on the same Kubernetes node, asking how to consolidate onto one shared
`nv-hostengine` — the standalone-vs-embedded contention question this lesson teaches is
not a textbook abstraction, it is a recurring, unresolved-until-you-know-this support
question in production Kubernetes fleets.

## What's new here (calibration)

- You already deployed dcgm-exporter and wrote Prometheus scrape configs in module 04,
  and you understand scrape interval as an operational knob generically. **Not
  re-taught.**
- New here: DCGM's *internal* architecture — `nv-hostengine` standalone vs embedded mode
  and the exact connection URI formats, fields vs field *groups* vs watches, and
  specifically the **profiling module** as a distinct, privileged, sampled subsystem with
  two real costs to design around — **counter multiplexing** (requesting more concurrent
  PROF fields than the hardware can co-collect) and the **bounded averaging window** of
  the activity counters. This is where "I know DCGM" becomes "I know how DCGM's sampling
  actually costs and lies at fleet scale."

## Core concepts

### Hostengine: standalone vs embedded

DCGM's engine is **`nv-hostengine`** — the process that owns the field cache, talks to
the driver/NVML and the profiling module, and answers field queries. It runs in one of
two modes:

- **Standalone (daemon) mode.** `nv-hostengine` runs as its own long-lived process on the
  node. Clients — `dcgmi`, your own bindings, or an exporter — connect over the DCGM API
  using a documented **remote hostengine URI**: `tcp://<HOST>:<PORT>` (default port
  `5555`), `unix:///<SOCKET_PATH>`, or `vsock://<CID>:<PORT>`. One engine per node serves
  many clients; this is what you run when multiple consumers (an exporter *and* ad-hoc
  `dcgmi` diagnostics) need the same GPUs, and it centralises the profiling module so two
  consumers don't fight over hardware counters.
- **Embedded mode.** The DCGM engine is loaded **in-process** as a shared library
  (`libdcgm`) — there is no separate daemon; the consuming process *is* the host engine.
  Lower overhead and no socket, but the engine's lifetime is the process's lifetime and
  only that process can use it.

**dcgm-exporter uses embedded mode by default.** That matters operationally: because the
exporter embeds the engine and holds the profiling module, running `dcgmi` — or a second
embedded consumer, such as a custom health-check sidecar — *also* asking for PROF fields
on the same GPUs can contend for the same hardware counters. This is exactly the failure
NVIDIA's own DCGM issue tracker documents: a user running dcgm-exporter and a custom
health-check container as two separate embedded engines on one node, correctly diagnosing
it as wrong and asking how to consolidate onto a single standalone `nv-hostengine` that
both clients connect to over `tcp://localhost:5555`. The fix generalizes: point
dcgm-exporter at a *remote* standalone `nv-hostengine` (`-r <URI>`) so the exporter and
other clients share one engine — the standard pattern whenever you need both the exporter
and node-level diagnostics without counter contention.

### Fields and field groups

Everything DCGM exposes is a **field** with a stable integer ID (`DCGM_FI_*`, e.g. 203,
1002, 252 — lesson 05.1). You don't query fields one call at a time at scale; you create
a **field group** — a named set of field IDs — and a **watch** on it with an
`updateFreq` (microseconds), a `maxKeepAge`, and a `maxKeepSamples`. The engine then
samples those fields into its cache on that cadence, and readers pull from the cache.
dcgm-exporter's config (the field CSV, e.g. `default-counters.csv`) is exactly this: the
list of field IDs it watches and the update frequency it watches them at. So "what do I
scrape" decomposes into two knobs: the **field group** (which IDs) and the **update
frequency** (how often the engine samples), which are *distinct from* your Prometheus
scrape interval (how often you pull the already-sampled cache).

### The profiling subsystem is a different, privileged, costed thing

Fields **1001–1012** (the `DCGM_FI_PROF_*` family: GR_ENGINE_ACTIVE, SM_ACTIVE,
SM_OCCUPANCY, PIPE_TENSOR_ACTIVE, DRAM_ACTIVE, the FP64/32/16 pipes, PCIe TX/RX, NVLink
TX/RX) do **not** come from NVML polling like `GPU_UTIL` does (lesson 05.1). They come
from the **profiling module**, which programs the GPU's **hardware performance
counters** and samples them. This is a fundamentally more expensive, more constrained,
and more *privileged* path than the NVML device fields, with three properties you must
design around.

#### 0) It requires elevated privileges NVML fields don't

Collecting profiling metrics requires `nv-hostengine` to run with **superuser
privileges**. This is not a theoretical gotcha — it is a documented, recurring
production failure: NVIDIA's own `dcgm-exporter` issue tracker records a real deployment
where a container on an A100 node with MIG enabled failed to start with a DCGM
initialization error, and the accompanying warning was explicit — *"dcgm-exporter doesn't
have sufficient privileges to expose profiling metrics. To get profiling metrics with
dcgm-exporter, use `--cap-add SYS_ADMIN`."* In Kubernetes terms: your dcgm-exporter
DaemonSet needs the right `securityContext` (typically `SYS_ADMIN` or a privileged pod)
or every PROF field from lesson 05.1 will simply come back empty — a silent, not a loud,
failure. Check this *before* debugging multiplexing; a missing capability and an
over-subscribed field group produce similar-looking gaps in your data.

#### 1) Multiplexing — the hardware can't collect every counter at once

The GPU has a **finite number of counter-collection resources**, and many PROF metrics
map onto overlapping hardware counters. Which fields can be collected *together* is
**architecture-dependent** — the clearest documented example: SM Activity and SM
Occupancy cannot be collected together with Tensor Utilization on a **V100**, but the
same three metrics *can* be collected together on a **T4**. There is no universal "safe
set" — you discover it per GPU model, not from a rule of thumb.

When the set of PROF fields you ask the engine to watch **exceeds what can be gathered in
a single pass**, DCGM does **not** error — it **automatically multiplexes**, statistically
sampling the requested metrics and rotating the groupings internally, transparently to
the caller. The consequences:

- Each metric is now only sampled a *fraction* of the time, so each individual field's
  effective resolution drops even though your `updateFreq` says 1s.
- During intervals where a given field isn't the one currently being collected, **DCGM
  reports it as `0.0`** (or a stale/held value) rather than a real reading. On a
  dashboard this looks like a metric that keeps dropping to zero — the classic "why is
  TENSOR_ACTIVE flickering to 0" symptom is over-subscribed profiling, not an idle GPU
  (and not a missing capability, once you've ruled that out).

The practical rule: **keep the concurrently-watched PROF set small** (the four from 05.1
— SM_ACTIVE, SM_OCCUPANCY, TENSOR_ACTIVE, DRAM_ACTIVE — plus a couple of link counters is
a reasonable co-resident set on most architectures) and, if you need more, accept a
*longer* effective interval or split them across watches. `dcgmi profile -l` lists what
the GPU supports and the metric *groups* it organises them into — metrics in *different*
groups are the ones prone to multiplexing when watched together, and this is a
per-generation fact you look up, not memorize once.

Also: the profiling module can be **paused and resumed** (`dcgmi profile --pause` /
`--resume`). Pausing releases the counters so an external profiler (Nsight
Compute/Systems, module 05.7, or a job that programs counters itself) can run without
fighting DCGM — and while paused, PROF fields return no fresh data. Any external tool
that grabs the counters can likewise make DCGM's PROF fields go blank; the two cannot own
the hardware counters simultaneously.

#### 2) The bounded averaging window of the activity counters

The activity ratios (SM_ACTIVE, TENSOR_ACTIVE, DRAM_ACTIVE, etc.) are **averages of
counter deltas over the sampling interval**, and the engine maintains them over a
bounded window — on the order of **≤30 seconds** for the tensor/activity counters. Two
failure modes follow:

- **Scrape too *slowly* (Prometheus interval ≫ the window):** you under-sample. If the
  engine's usable averaging window is ~30s and you scrape every 60–120s, you are reading
  a value that only reflects the last ~30s of a 2-minute gap — you miss bursts, and a
  bursty training step looks quiet. The **maximum useful scrape interval is set by that
  ~30s window: don't scrape PROF activity fields slower than ~30s** or you're sampling a
  window shorter than your gap.
- **Scrape too *fast* (sub-second `updateFreq`):** you over-cost. Programming and reading
  hardware counters is not free; pushing `updateFreq` below ~1s multiplies profiling
  overhead across the fleet (and worsens multiplexing) for resolution the ~30s-averaged
  counters can't even express meaningfully. ~1s is the sane floor.

**The fleet judgment:** a PROF `updateFreq` around **1s** at the engine, and a
Prometheus **scrape interval in the ~10–30s band** for the PROF fields, keeps you inside
both bounds — fast enough to sit under the averaging window, slow enough to keep counter
overhead and cardinality sane across thousands of GPUs. Cheap NVML fields (util, memory,
power, temp, clocks — 100s/200s) can be scraped faster without profiling cost if you want.

## Perspectives

**Developer.** A developer running `dcgmi dmon` locally on a devbox sees clean, unbroken
numbers because they're the only client contending for counters and the only field group
watching them. Multiplexing, permission gaps, and pause/resume conflicts are invisible
until a second consumer — an exporter, a second `dcgmi` session, an external profiler —
shows up. It is a textbook "works on my node" trap, and it's why the fleet story in this
lesson looks nothing like a single-GPU debugging session.

**Operator.** The engine mode (standalone vs embedded) and each field group's
`updateFreq` are fleet-wide configuration decisions with real, non-obvious failure modes.
Silent multiplexing and permission gaps don't show up as Prometheus scrape errors — they
show up as a dashboard that "looks fine" except for a metric that mysteriously flickers
to zero, or a PROF panel that's simply empty. Diagnosing that requires knowing this
architecture, not staring harder at PromQL.

**Hardware.** The GPU has a genuinely finite number of hardware performance-counter
collection slots. PROF metrics are grouped by which counters they physically share, and
metrics in different groups round-robin when over-subscribed — a property of the
silicon (and it varies by generation, per the V100-vs-T4 example), not a software
limitation DCGM could simply "fix" with more threads or a bigger buffer.

**Economics / reliability.** Sub-second profiling `updateFreq` multiplies overhead across
a fleet for resolution the ~30s-averaged counters cannot even express — over-sampling
PROF fields fleet-wide is a real, avoidable cost line, distinct from (and much smaller
than) the compute cost of the workloads themselves, but non-zero at thousands of GPUs.
Getting the `updateFreq`/scrape-interval band right once, fleet-wide, is cheaper than
re-deriving it per team that asks "can we get GPU metrics faster."

## Real-world use cases

- **NVIDIA/DCGM GitHub — Issue #287, "How to correctly use the nv-hostengine on
  Kubernetes"** — https://github.com/NVIDIA/DCGM/issues/287. A real user runs
  dcgm-exporter and a custom health-check container as two separate embedded engines on
  one node, recognizes it as wrong, and asks whether to consolidate onto a shared
  standalone `nv-hostengine` reachable over `tcp://localhost:5555` or a Unix socket.
  **What it shows:** the standalone-vs-embedded contention question this lesson teaches
  is a live, recurring production support question, not a textbook abstraction.
- **NVIDIA/dcgm-exporter GitHub — Issue #34, "Error starting nv-hostengine: DCGM
  initialization error"** — https://github.com/NVIDIA/dcgm-exporter/issues/34. On an
  A100 node with MIG enabled, dcgm-exporter fails to start and explicitly warns:
  "dcgm-exporter doesn't have sufficient privileges to expose profiling metrics. To get
  profiling metrics with dcgm-exporter, use `--cap-add SYS_ADMIN`." **What it shows:**
  the superuser-privilege requirement for profiling metrics is a real, documented,
  first-encounter production failure — not a footnote.
- **NVIDIA/dcgm-exporter GitHub — README, "Remote Hostengine URI Formats"** —
  https://github.com/NVIDIA/dcgm-exporter — documents the `-r` flag accepting
  `tcp://<HOST>:<PORT>`, `unix:///<SOCKET_PATH>`, and `vsock://<CID>:<PORT>` for
  connecting the exporter to a remote standalone hostengine. **What it shows:** the exact,
  current syntax for the standalone-mode pattern this lesson recommends — copy this,
  don't guess the flag.
- **DigitalOcean — "How to Enable GPU Metrics on GPU Droplets with DCGM"** —
  https://docs.digitalocean.com/products/droplets/how-to/gpu/enable-metrics/. A major
  GPU cloud's own doc on standing up `nv-hostengine` at the VM/fleet level for customers.
  **What it shows:** a real cloud provider operationalizing exactly the hostengine
  deployment decision this lesson covers, at fleet scale, for external customers — not
  just an internal NVIDIA reference.

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
values, roughly in rotation. First you check the container's capabilities — `SYS_ADMIN`
is present, so this isn't the privilege failure from Issue #34. That rules in
multiplexing: you asked for more PROF fields than the A100 can co-collect in one pass, so
DCGM is round-robining the pipe/activity counters and reporting 0.0 for whichever group
isn't currently sampled. You confirm architecture-dependence separately: the same
12-field request on a T4 node in the same fleet shows far less flickering because more of
its metrics co-reside in fewer hardware groups than the A100's — a concrete instance of
the documented V100-style behavior (some architectures group tensor/occupancy metrics
more restrictively than others). Reducing the A100's watched set to the four load-bearing
fields (1002, 1003, 1004, 1005) makes the zeros disappear and every field read
continuously. **Conclusion for the fleet:** watch the minimal PROF set per GPU, at 1s
engine `updateFreq`, scraped by Prometheus every ~15s; add the extra pipe/link counters
only on a diagnostic profile you enable on demand per node, not fleet-wide — and verify
per architecture with `dcgmi profile -l` rather than assuming last quarter's GPU
generation behaves the same way.

## Practice

On a rented GPU with DCGM installed, feeding the module's
["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)
deliverable's "why our scrape config is what it is" appendix:

1. **Discover.** `dcgmi discovery -l` to enumerate GPUs; `dcgmi profile -l` to list the
   PROF metrics the GPU supports and the metric *groups* it organises them into (note
   which metrics live in different groups — those are your multiplexing candidates).
2. **Check privileges.** Confirm `nv-hostengine`/your dcgm-exporter container has
   `SYS_ADMIN` (or equivalent privileged `securityContext` in Kubernetes) *before*
   debugging anything else — a missing capability and multiplexing produce similar
   symptoms, and ruling this out first saves time.
3. **Monitor.** `dcgmi dmon -e 1002,1004,1005` to stream SM_ACTIVE / TENSOR_ACTIVE /
   DRAM_ACTIVE while a workload runs.
4. **Force multiplexing.** Deliberately request too many PROF fields at once at 1s — e.g.
   `dcgmi dmon -e 1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1011,1012 -d 1000` —
   and observe fields **flickering to 0.0** as DCGM round-robins. Note which fields zero
   out together (same/different group).
5. **Pause/resume.** `dcgmi profile --pause`, confirm PROF fields stop updating
   (blank/held), run something that needs the counters, then `dcgmi profile --resume` and
   confirm they return.
6. **Standalone consolidation (if you have two consumers).** Stand up `nv-hostengine` in
   standalone mode, point both dcgm-exporter (`-r tcp://localhost:5555`) and a `dcgmi`
   session at it, and confirm you no longer see contention symptoms when both query PROF
   fields concurrently.

**Acceptance:** a written note capturing (i) the observed multiplexing — which fields
returned 0.0 when over-subscribed and roughly the rotation — (ii) confirmation the
privilege requirement was checked and satisfied, and (iii) your chosen **fleet PROF
scrape interval and engine `updateFreq`, with the reason** (the ~30s averaging-window
upper bound and the ~1s cost/multiplexing lower bound). This note is the "why our scrape
config is what it is" appendix of the deliverable.

## Common pitfalls

1. **Assuming DCGM errors out when over-subscribed.** It silently multiplexes and
   returns 0.0/stale values instead, which looks like real telemetry — always suspect
   multiplexing before assuming the workload went idle.
2. **Running dcgm-exporter (embedded) *and* ad-hoc `dcgmi profile` diagnostics on the
   same node without a shared standalone hostengine.** The two contend for the same
   hardware counters — exactly the scenario in the real DCGM issue #287 above.
3. **Debugging flickering PROF fields as a multiplexing problem when it's actually a
   privilege problem, or vice versa.** Check `SYS_ADMIN`/superuser access first (a real,
   documented dcgm-exporter failure mode) before assuming an over-subscribed field group.
4. **Forgetting the ~30s bounded averaging window and scraping PROF fields slower than
   that.** Under-samples bursty training steps into invisibility.
5. **Assuming a "safe" co-resident PROF field set is universal across GPU generations.**
   It is architecture-dependent (V100 vs T4 is the documented example) — verify with
   `dcgmi profile -l` per fleet, don't carry a rule of thumb across hardware refreshes.

## Self-check

- Why can't you sample every `PROF_*` field at 1s on one GPU? **Answer:** The PROF fields
  are backed by a *finite* set of GPU hardware performance counters, and many metrics map
  onto overlapping counters. Requesting more concurrent PROF fields than the hardware can
  collect in one pass forces DCGM to time-multiplex (round-robin) the groups; DCGM
  doesn't error — it returns `0.0` for whichever field isn't currently being sampled and
  drops each field's effective resolution. So "all fields at 1s" gives you flickering
  zeros, not more data. Keep the co-resident set small and verify it per GPU model.
- Standalone vs embedded — which does dcgm-exporter use by default, and when would you
  override it? **Answer:** dcgm-exporter runs the DCGM engine **embedded** (in-process,
  via `libdcgm`) by default — no separate `nv-hostengine` daemon, and it owns the
  profiling module itself. Override to a **remote standalone `nv-hostengine`** (`-r
  tcp://<host>:<port>`, or a unix/vsock URI) when the exporter and another client (e.g.
  `dcgmi` diagnostics, a health-check sidecar) need to share one engine and not contend
  over hardware counters — the exact fix for the scenario in DCGM issue #287.
- What privilege does `nv-hostengine` need to collect profiling metrics, and why does
  that matter operationally? **Answer:** Superuser privileges — in a container, typically
  the `SYS_ADMIN` capability (or a privileged `securityContext` in Kubernetes). It
  matters because the failure is silent-ish but documented: dcgm-exporter starts, NVML
  fields work, but PROF fields simply never populate, with an explicit warning in the
  logs if you look — a real, first-encounter production failure recorded in
  dcgm-exporter's own issue tracker, easy to mistake for multiplexing if you don't check
  capabilities first.
- What's the maximum useful scrape interval given the tensor counter's ≤30s window?
  **Answer:** ~30 seconds. The activity/tensor counters are averaged over a bounded
  window on the order of 30s; scraping slower than that means your Prometheus gap
  exceeds the window the value represents, so you under-sample and miss bursts.
  Practically: engine `updateFreq` ~1s, Prometheus scrape in the ~10–30s band — never
  slower than ~30s for PROF activity fields.
- Why does multiplexing depend on GPU architecture? **Answer:** Because which hardware
  counters different PROF metrics map onto is a property of the specific GPU generation's
  performance-monitoring silicon, not a DCGM software choice — documented example: SM
  Activity/SM Occupancy cannot be collected together with Tensor Utilization on a V100,
  but the same three metrics can be on a T4. `dcgmi profile -l` tells you the answer for
  your specific hardware; there is no cross-generation rule of thumb.

## Connections & what's next

This lesson explains the collection mechanics behind every PROF field lesson 05.1 told
you to trust — without it, "page on SM_ACTIVE" is a policy you can't actually operate at
fleet scale. It sets up lesson 05.3 directly: dcgm-exporter's field CSV and cardinality
math both build on the field-group/watch model introduced here, and the "why does
`SM_ACTIVE` ship commented out by default" question in 05.3 is partly a consequence of
this lesson's cost story (someone chose defaults conservatively because profiling isn't
free). It also underwrites lesson 05.7's profiling escalation ladder — Nsight
Compute/Systems and DCGM's profiling module compete for the same finite hardware
counters, which is why `dcgmi profile --pause` exists. Next: lesson 05.3 takes the field
groups this lesson taught you to configure and audits what dcgm-exporter actually ships
watching by default at real fleet scale.

## References & further reading

**Primary sources**
- DCGM profiling API / feature overview — https://docs.nvidia.com/datacenter/dcgm/latest/dcgm-api/dcgm-api-profiling.html — read for: the authoritative description of the profiling module, the supported PROF metrics, counter constraints, and multiplexing behaviour.
- DCGM User Guide — feature overview & `dcgmi` — https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html — read for: the hostengine (standalone vs embedded) architecture, field groups/watches/`updateFreq`, the documented V100-vs-T4 multiplexing example, and the `dcgmi discovery`/`dmon`/`profile` commands used in practice.

**Real-world engineering blogs**
- NVIDIA/DCGM GitHub — Issue #287 — https://github.com/NVIDIA/DCGM/issues/287 — what it shows: the standalone-vs-embedded contention question as a live production support issue.
- NVIDIA/dcgm-exporter GitHub — Issue #34 — https://github.com/NVIDIA/dcgm-exporter/issues/34 — what it shows: a real, documented profiling-privilege failure (`--cap-add SYS_ADMIN`) on an A100/MIG node.
- NVIDIA/dcgm-exporter GitHub — README — https://github.com/NVIDIA/dcgm-exporter — what it shows: the exact `-r` remote-hostengine URI syntax (`tcp://`, `unix://`, `vsock://`).
- DigitalOcean — "How to Enable GPU Metrics on GPU Droplets with DCGM" — https://docs.digitalocean.com/products/droplets/how-to/gpu/enable-metrics/ — what it shows: a major GPU cloud operationalizing `nv-hostengine` at fleet scale for customers.

**Deeper dives**
- `dcgmi` command reference (within the DCGM User Guide above) — go deeper on: `dcgmi profile -l`, `dcgmi dmon`, `dcgmi profile --pause/--resume` flags used throughout this lesson's practice section.
