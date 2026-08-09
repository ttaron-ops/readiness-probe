---
lesson: "05.8"
title: "Capstone — allocated vs utilised (your dashboard is lying)"
module: "05"
concept: "Capstone — allocated vs utilised (your dashboard is lying)"
status: not-started
est_time: "8h"
artifacts: []
---
# 05.8 · Capstone — allocated vs utilised (your dashboard is lying)
> **Concept.** Ship the per-namespace **allocated-vs-utilised GPU-hours** dashboard, render the gap in **dollars**, and prove the `GPU_UTIL=100% / SM_ACTIVE≈0` lie from your own cluster. This is the flagship public artifact.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Why this matters

This is the artifact your career pivots on. Everyone applying for the same GPU-platform role can
stand up dcgm-exporter and point Grafana at it. Almost none of them can walk into the room and
say: *"Your GPU utilisation dashboard is lying to you, here's the proof from a real cluster, here's
what the honest number is, and here's the dollar figure the gap represents."* That sentence is
simultaneously your interview opener, the thesis of a blog post that will get shared, the input to
module 11 (cost economics), and the core query of the `gpu-cost-operator`. One artifact, four
payoffs.

The reason it lands is that it attacks a **universally believed false number**. `nvidia-smi`
utilisation — surfaced as `DCGM_FI_DEV_GPU_UTIL` — reads 100% whenever *at least one kernel was
resident in the sample window*, regardless of whether that kernel used 1% or 100% of the SMs. Every
default GPU dashboard shows this field. Every capacity-planning and chargeback decision built on it
is built on sand. Industry surveys repeatedly put *average* fleet GPU utilisation around **10–25%**
while the dashboards show it "busy." The gap between what you *allocated* (and are paying for) and
what you *utilised* (SM-active work that actually happened) is the single largest line of waste in a
GPU org, and no vendor bill or default dashboard exposes it. You are going to build the dashboard
that does, and put a dollar sign on it.

## What's new here

Every earlier lesson produced a piece; this capstone **joins them into one shipped thing** and adds
the two moves that make it public-artifact-grade:

1. **The allocated-vs-utilised join.** From module 04 you have `allocated` — GPU count per namespace
   from the kubelet **pod-resources** API (an allocation exists whether or not the GPU does any
   work). From L3/L6 you have `utilised` — **SM_ACTIVE-weighted busy GPU-hours**, the honest signal.
   New here: putting them on the *same axis, per namespace,* so the gap is undeniable.
2. **The gap in dollars.** `gap_gpu_hours × hourly_rate`. This is the translation layer that turns a
   platform-engineering graph into a CFO conversation — and it's what module 11 consumes.

The conceptual core is that **allocated and utilised are different physical quantities**, and the
whole field routinely conflates them:

| | Allocated GPU-hours | Utilised GPU-hours |
|---|---|---|
| **Source** | pod-resources count (04) — an allocation record | `SM_ACTIVE` integrated over time (L3/L6) |
| **Measures** | what you *reserved and are billed for* | work the SMs *actually did* |
| **At 0% use** | still counts (you pay) | zero |
| **The lie field** | — | `DCGM_FI_DEV_GPU_UTIL` inflates this if you use it here |
| **Finance meaning** | the invoice | the value delivered |

**Allocated − Utilised = waste.** Rendered in dollars, that subtraction is the entire artifact.

## Core notes

### The util lie, precisely

`DCGM_FI_DEV_GPU_UTIL` (the DCGM surfacing of `nvidia-smi`'s "GPU-Util") is defined as the *percent
of time in the sample window during which one or more kernels was executing.* It is a **duty-cycle
of occupancy, not of throughput.** A single tiny kernel looping keeps it pinned at 100. That's why:

```promql
# THE UTIL-LIE DETECTOR: dashboard says busy, hardware says idle
DCGM_FI_DEV_GPU_UTIL > 90
  and on (gpu, UUID, instance)
DCGM_FI_PROF_SM_ACTIVE < 0.2
```

`SM_ACTIVE` is the fraction of cycles an SM had ≥1 warp resident — a real occupancy fraction, 0–1.
When `GPU_UTIL` is >90 *and* `SM_ACTIVE` is <0.2, the dashboard is reporting a GPU as fully busy that
is doing almost nothing. **Capturing this on your own cluster is the exhibit** — a single time-series
panel with both lines, `GPU_UTIL` pinned near 100 and `SM_ACTIVE` crawling along the floor. That
screenshot *is* "your dashboard is lying to you."

> Note the join keys. dcgm-exporter labels every series with `gpu`, `UUID`, `Hostname`/`instance`,
> and (with `DCGM_EXPORTER_KUBERNETES=true`) `exported_namespace`, `exported_pod`,
> `exported_container`. All PromQL below joins on the GPU identity labels and groups by
> `exported_namespace`.

### The PromQL query pack

This pack is the reusable core — it drops verbatim into the `gpu-cost-operator` and module 11.

**1. Allocated-but-idle GPUs (the headline).** A GPU that has a pod-resources allocation but whose
SM has been essentially dead for 15 minutes:

```promql
# GPUs allocated to a pod yet idle for 15m — pure waste, still billing
count by (exported_namespace) (
    avg_over_time(DCGM_FI_PROF_SM_ACTIVE[15m]) < 0.05
  and on (gpu, UUID, Hostname)
    (DCGM_FI_DEV_FB_USED >= 0)            # series exists ⇒ GPU is present/attributed
)
```

(In the operator you AND against the real allocation set from pod-resources rather than "series
exists"; on a pure-Prometheus setup, presence of an `exported_pod` label is the allocation proxy:
`... and on(...) count by (...) (DCGM_FI_PROF_SM_ACTIVE{exported_pod!=""})`.)

**2. The util-lie detector** — the exhibit query, above.

**3. Per-namespace wasted GPU-hours = allocated − SM-active.** Integrate both over a window. With a
1 Hz-ish scrape, SM-active GPU-hours over a range is the mean `SM_ACTIVE` × the window in hours ×
GPU count:

```promql
# Allocated GPU-hours per namespace over the last day (each allocated GPU = 24 GPU-hours)
count by (exported_namespace) (DCGM_FI_PROF_SM_ACTIVE{exported_pod!=""}) * 24

# Utilised (SM-active) GPU-hours per namespace over the same day
sum by (exported_namespace) (avg_over_time(DCGM_FI_PROF_SM_ACTIVE{exported_pod!=""}[24h])) * 24

# WASTED GPU-hours per namespace = allocated − utilised
(count by (exported_namespace) (DCGM_FI_PROF_SM_ACTIVE{exported_pod!=""}) * 24)
  -
(sum by (exported_namespace) (avg_over_time(DCGM_FI_PROF_SM_ACTIVE{exported_pod!=""}[24h])) * 24)
```

**4. The dollar gap** — the CFO panel. Multiply wasted GPU-hours by the rate-card hourly rate:

```promql
# $ wasted per namespace per day  ($HOURLY_RATE recorded as a rule or injected constant)
(
  (count by (exported_namespace) (DCGM_FI_PROF_SM_ACTIVE{exported_pod!=""}) * 24)
    -
  (sum by (exported_namespace) (avg_over_time(DCGM_FI_PROF_SM_ACTIVE{exported_pod!=""}[24h])) * 24)
) * 2.50    # $/GPU-hour rate card — parameterise per instance type
```

**5. The divergence (the reshuffle) — the single most persuasive panel.** Rank namespaces by
*allocated* GPUs, then by *SM-active* GPU-hours; the order changes. The team holding the most GPUs is
often *not* the team doing the most work:

```promql
# Ranking A — who HOLDS the most GPUs
topk(10, count by (exported_namespace) (DCGM_FI_PROF_SM_ACTIVE{exported_pod!=""}))

# Ranking B — who actually USES the most GPU
topk(10, sum by (exported_namespace) (avg_over_time(DCGM_FI_PROF_SM_ACTIVE{exported_pod!=""}[24h])))
```

Show the two rankings side by side; the rows that jump (high allocation, low SM-active) are your
reclaim targets. Prefer `DCGM_FI_PROF_GR_ENGINE_ACTIVE` (graphics/compute engine active fraction)
over `SM_ACTIVE` if you want a slightly broader "engine busy" definition; use `PIPE_TENSOR_ACTIVE`
if the story is specifically "holding GPUs but not doing tensor math."

### The dollar framing — the part finance repeats

The numbers are the whole point of publishing:

- Fleet utilisation averages roughly **10–25%** in practice; call it **~15%**, so **~85% of paid
  GPU-hours are idle.**
- On **500 H100s at $2–3/GPU-hour**, a fleet running 24×7 is **$8.76M–$13.1M/yr** of allocation.
  At 15% utilisation the *idle* portion alone is on the order of **$7.5M–$11M/yr.**
- You don't promise "fix it all." You promise a **realistic 10-point utilisation improvement**
  (15% → 25%), which on those 500 H100s is roughly **$0.9M–$1.3M/yr recovered** — a number a CFO can
  act on, sourced from a graph you built, not a vendor pitch.

Two framings, one graph: engineers see "85% idle SMs," finance sees "$1M/yr on the table." The
allocated-vs-utilised dollar gap is the object that speaks both languages.

## Worked example — reading one cluster's day

A 3-node, 24×A100 cluster, one day, rate card $2.50/GPU-hour. dcgm-exporter with the k8s labels on;
the query pack loaded.

- **Allocated (pod-resources):** `team-research` holds 12 GPUs, `team-serving` holds 8, `team-batch`
  holds 4. All 24 allocated ⇒ **576 allocated GPU-hours/day**, **$1,440/day** billed.
- **Utilised (SM-active over 24h):** mean `SM_ACTIVE` — research **0.62**, serving **0.11**, batch
  **0.44**. Utilised GPU-hours = `research 12×24×0.62 = 178.6`, `serving 8×24×0.11 = 21.1`,
  `batch 4×24×0.44 = 42.2` ⇒ **241.9 utilised GPU-hours**, **42% fleet** (already better than
  typical — a healthy cluster).
- **Wasted = allocated − utilised:** research 109.4, **serving 170.9**, batch 53.8 GPU-hours.
  In dollars: research $273, **serving $427**, batch $135 — **$835/day, ~$305k/yr** on this small
  cluster.
- **The reshuffle:** by *allocation*, research (12) > serving (8) > batch (4). By *SM-active
  GPU-hours*, research (178.6) > batch (42.2) > **serving (21.1)** — serving drops from 2nd to last.
  **`team-serving` is the reclaim target:** 8 GPUs held, 11% SM-active, $427/day idle. The util-lie
  detector confirms it — serving's GPUs show `GPU_UTIL` in the 80s (an inference server always has a
  kernel resident) while `SM_ACTIVE` sits at 0.11. That's the exhibit and the recommendation in one:
  right-size serving (MIG slices, batching, or fewer replicas) and recover ~$150k/yr from one
  namespace.

That paragraph — allocated vs utilised, the reshuffle, the dollar gap, one named reclaim target — is
the shape of the blog's central section, run on *your* cluster's real numbers.

## Practice — THE MODULE DELIVERABLE

Build it against [`../practice/gpu-dashboard-lie/README.md`](../practice/gpu-dashboard-lie/README.md);
acceptance is the [module checkpoint](../checkpoint.md).

1. **Dashboard.** A Grafana dashboard with, per namespace: allocated GPU-hours vs utilised
   (SM_ACTIVE-weighted) GPU-hours on one axis, a **wasted-GPU-hours** panel, and a **$-gap** panel
   (`gap × rate`). Include the divergence (two rankings side by side).
2. **The util-lie exhibit.** A panel from *your own cluster* showing `DCGM_FI_DEV_GPU_UTIL` pinned
   >90 while `DCGM_FI_PROF_SM_ACTIVE` sits <0.2, with the detector query firing. This is the money
   screenshot.
3. **The PromQL pack.** The five queries above, parameterised (namespace, window, rate), committed
   as a reusable file — the same queries the `gpu-cost-operator` will run.
4. **The blog post.** *"Your GPU utilisation dashboard is lying to you."* Thesis (allocated ≠
   utilised; `GPU_UTIL` is a duty-cycle lie) → the exhibit → the per-namespace gap on real numbers →
   the dollar framing → the reclaim recommendation. Fold in the L7 before/after round-trip as the
   "and here's how you *fix* an inefficient GPU once you've found it" section.

**Acceptance = the module checkpoint:** the dashboard renders allocated-vs-utilised per namespace
with a dollar gap; the util-lie exhibit is captured from a real cluster with the detector query; the
PromQL pack is committed and reusable; the blog post is drafted with your cluster's real gap and its
dollar figure. The PromQL pack and the dollar-gap definition must be importable into the
`gpu-cost-operator` unchanged — that's the module-11 handoff.

## Self-check

**(a)** Write, from memory, the PromQL for "GPUs allocated to a pod but idle for more than 15
minutes."

**Answer:**
```promql
count by (exported_namespace) (
    avg_over_time(DCGM_FI_PROF_SM_ACTIVE{exported_pod!=""}[15m]) < 0.05
)
```
The load-bearing parts: **`SM_ACTIVE`** (the honest occupancy fraction, *not* `DCGM_FI_DEV_GPU_UTIL`,
which would hide the idle GPUs), **`avg_over_time([15m])`** (sustained idle, not an instantaneous
dip), **`< 0.05`** (essentially dead SMs), and the **`{exported_pod!=""}`** filter (there *is* an
allocation — a pod holds it). In the operator you replace that label filter with an explicit AND
against the pod-resources allocation set.

**(b)** Explain the utilisation trap and the allocated-vs-utilised dollar gap to a CFO in two
minutes.

**Answer:** "Our GPU dashboard says the fleet is ~90% busy. That number is a lie — not maliciously,
it's how the metric is defined: it reads 'busy' if *any* work touched the GPU in the last second,
even 1% of the chip. The honest metric — the fraction of the GPU's compute cores actually working —
averages about 15%. So we *allocate and pay for* 100% of these GPUs, but only *use* about 15% of
them. On our 500 H100s at ~$2.50/hour, that's roughly $10M/year allocated and about $8.5M/year of it
sitting idle. I built a per-team dashboard that shows exactly which teams hold GPUs they aren't
using — the gap, in dollars, per namespace. If we recover even 10 points of utilisation by
right-sizing the worst offenders, that's about $1M/year back. It's not a new purchase or a vendor;
it's reclaiming what we already own." (Allocated = the invoice; utilised = the value; the gap = the
waste, in dollars, per team.)

**(c)** Which single metric is simultaneously the interview answer, the blog thesis, and the
module-11 input?

**Answer:** The **allocated-vs-utilised GPU-hours gap** — allocated GPU-hours (from pod-resources)
minus `SM_ACTIVE`-weighted utilised GPU-hours, rendered in dollars. It's the interview opener ("your
utilisation dashboard is lying — here's the honest gap"), the blog's thesis (`GPU_UTIL` is a
duty-cycle lie; the real number is the gap), and the exact quantity the module-11 cost work and the
`gpu-cost-operator` consume. `SM_ACTIVE` is the honest ingredient; the *gap in dollars* is the
artifact.

## Resources

1. **ScaleOps — GPU Cost Optimization.** The allocated-vs-used framing and the reclaim economics,
   from a vendor whose whole product is this gap. https://scaleops.com/blog/gpu-cost-optimization/
2. **rack2cloud — GPU Utilisation & Cloud Waste.** The "~15% utilised / 85% idle" fleet numbers and
   the dollar-of-waste framing for the blog's finance section.
   https://www.rack2cloud.com/gpu-utilization-cloud-waste/
3. **All of L1–L7 of this module** — the metrics that lie (L2) vs tell the truth (L3), dcgm-exporter
   labels (L4), attribution (L6), and the L7 profiling round-trip that becomes the blog's "how to
   fix it" section.
