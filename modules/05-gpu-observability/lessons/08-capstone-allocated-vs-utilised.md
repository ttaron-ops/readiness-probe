---
lesson: "05.8"
title: "Capstone — allocated vs utilised (your dashboard is lying)"
module: "05"
concept: "Capstone — allocated vs utilised (your dashboard is lying)"
status: not-started
est_time: "10h"
prev: "07-profiling-escalation.md"
next: null
artifacts: []
sources: 6
---
# 05.8 · Capstone — allocated vs utilised (your dashboard is lying)

> **Concept.** Ship the per-namespace **allocated-vs-utilised GPU-hours** dashboard, render the gap in **dollars**, and prove the `GPU_UTIL=100% / SM_ACTIVE≈0` lie from your own cluster. This is the flagship public artifact.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Where this fits

This capstone pulls together every lesson in the module into one shipped artifact. L1–L2 gave you the honest metric (`SM_ACTIVE` vs the `GPU_UTIL` lie) and the reason it's off by default. L3 gave you the exporter config to turn it on safely at fleet scale. L4 gave you the per-namespace attribution join — and its hard limit under time-slicing. L5 gave you the health/XID layer that keeps the fleet trustworthy in the first place. L6 gave you the inference-specific SLO metrics that feed the same dashboard for serving namespaces. L7 gave you the profiling round-trip that turns a finding into a fix. Nothing here is a new mechanism — it's the join, the dollar translation, and the packaging that turns seven lessons' worth of correct metrics into the one artifact a hiring panel, a CFO, and your own cost tooling can all consume.

## Why this matters

This is the artifact your career pivots on. Everyone applying for the same GPU-platform role can stand up dcgm-exporter and point Grafana at it. Almost none of them can walk into the room and say: *"Your GPU utilisation dashboard is lying to you, here's the proof from a real cluster, here's what the honest number is, and here's the dollar figure the gap represents."* That sentence is simultaneously your interview opener, the thesis of a blog post that will get shared, the input to module 11 (cost economics), and the core query of the `gpu-cost-operator`. One artifact, four payoffs.

It also isn't a course-invented thesis — it's a recognized, monetizable industry problem. Companies whose entire product is this exact allocated-vs-used gap (ScaleOps) and observability vendors who built first-class GPU products around the same taxonomy (Datadog) independently validate the framing. `nvidia-smi` utilisation — surfaced as `DCGM_FI_DEV_GPU_UTIL` — reads 100% whenever *at least one kernel was resident in the sample window*, regardless of whether that kernel used 1% or 100% of the SMs. Every default GPU dashboard shows this field. Every capacity-planning and chargeback decision built on it is built on sand. Industry material repeatedly puts *average* fleet GPU utilisation in the 10–25% range while the dashboards show it "busy" — a **2026, order-of-magnitude industry figure**, not a precise universal constant, and you should present it that way. The gap between what you *allocated* (and are paying for) and what you *utilised* (SM-active work that actually happened) is the single largest line of waste in a GPU org, and no vendor bill or default dashboard exposes it. You are going to build the dashboard that does, and put a dollar sign on it.

## What's new here (calibration)

You're certified on generic FinOps framing and you run Prometheus/PromQL/Grafana at 40+ clusters — this lesson doesn't re-teach either. It also doesn't re-teach the hardware util-lie *concept* (that was module 03) or the pod-resources→UUID join (module 04 — you *consume* it here, you don't rebuild it). What's genuinely new:

- **The allocated-vs-utilised join as a single artifact**, not two separate graphs — putting both on the same namespace/time axis so the gap is visually undeniable.
- **The dollar translation layer** (`gap_gpu_hours × hourly_rate`) that turns a platform-engineering graph into the exact number module 11 and a CFO conversation consume.
- **Packaging discipline** — an interview screenshot, a reusable query pack, and a blog draft are three different deliverables with three different bars, and this lesson is about hitting all three from one underlying dataset.

## Core concepts

### The util lie, precisely

`DCGM_FI_DEV_GPU_UTIL` (the DCGM surfacing of `nvidia-smi`'s "GPU-Util") is defined as the *percent of time in the sample window during which one or more kernels was executing.* It is a **duty-cycle of occupancy, not of throughput.** A single tiny kernel looping keeps it pinned at 100. That's why:

```promql
# THE UTIL-LIE DETECTOR: dashboard says busy, hardware says idle
DCGM_FI_DEV_GPU_UTIL > 90
  and on (gpu, UUID, instance)
DCGM_FI_PROF_SM_ACTIVE < 0.2
```

`SM_ACTIVE` is the fraction of cycles an SM had ≥1 warp resident — a real occupancy fraction, 0–1. When `GPU_UTIL` is >90 *and* `SM_ACTIVE` is <0.2, the dashboard is reporting a GPU as fully busy that is doing almost nothing. **Capturing this on your own cluster is the exhibit** — a single time-series panel with both lines, `GPU_UTIL` pinned near 100 and `SM_ACTIVE` crawling along the floor. That screenshot *is* "your dashboard is lying to you."

> Note the join keys. dcgm-exporter labels every series with `gpu`, `UUID`, `Hostname`/`instance`, and (with `DCGM_EXPORTER_KUBERNETES=true`) `exported_namespace`, `exported_pod`, `exported_container`. All PromQL below joins on the GPU identity labels and groups by `exported_namespace`.

### The allocated-vs-utilised table

| | Allocated GPU-hours | Utilised GPU-hours |
|---|---|---|
| **Source** | pod-resources count (04) — an allocation record | `SM_ACTIVE` integrated over time (L1–L3) |
| **Measures** | what you *reserved and are billed for* | work the SMs *actually did* |
| **At 0% use** | still counts (you pay) | zero |
| **The lie field** | — | `DCGM_FI_DEV_GPU_UTIL` inflates this if you use it here |
| **Finance meaning** | the invoice | the value delivered |

**Allocated − Utilised = waste.** Rendered in dollars, that subtraction is the entire artifact.

### The honesty caveat the dashboard must carry: the time-slicing hole

L4 established that under time-slicing, DCGM's `SM_ACTIVE` is a **device-level** counter, not a per-pod one — multiple pods can share one physical GPU while the metric stays attached to the device, not the tenant. ScaleOps, a vendor whose product is exactly this gap, states the same limitation plainly: with time-slicing, "multiple pods may share one physical GPU while many DCGM metrics remain device-level, so duplicated labels should not be interpreted as exact per-pod usage." That means your namespace-ranking panel is only as honest as its allocation mode:

- **MIG-partitioned or dedicated GPUs** — per-namespace `SM_ACTIVE` attribution is clean; rank freely.
- **Time-sliced or MPS-shared GPUs** — per-pod attribution was never in the series. The dashboard's correct move is an explicit **`unattributable_gpu_hours`** bucket for those GPUs, not a silent credit to whichever pod's label happened to be exported last. Shipping the ranking without this caveat quietly corrupts the chargeback number the moment a viewer trusts it at face value.

Carry this caveat into the blog post, not just the code — it's the difference between "the dashboard exposes a lie" and "the dashboard tells a second, quieter lie of its own."

### The PromQL query pack

This pack is the reusable core — it drops verbatim into the `gpu-cost-operator` and module 11.

**1. Allocated-but-idle GPUs (the headline).** A GPU that has a pod-resources allocation but whose SM has been essentially dead for 15 minutes:

```promql
# GPUs allocated to a pod yet idle for 15m — pure waste, still billing
count by (exported_namespace) (
    avg_over_time(DCGM_FI_PROF_SM_ACTIVE[15m]) < 0.05
  and on (gpu, UUID, Hostname)
    (DCGM_FI_DEV_FB_USED >= 0)            # series exists ⇒ GPU is present/attributed
)
```

(In the operator you AND against the real allocation set from pod-resources rather than "series exists"; on a pure-Prometheus setup, presence of an `exported_pod` label is the allocation proxy: `... and on(...) count by (...) (DCGM_FI_PROF_SM_ACTIVE{exported_pod!=""})`.)

**2. The util-lie detector** — the exhibit query, above.

**3. Per-namespace wasted GPU-hours = allocated − SM-active.** Integrate both over a window. With a 1 Hz-ish scrape, SM-active GPU-hours over a range is the mean `SM_ACTIVE` × the window in hours × GPU count:

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
) * 2.50    # $/GPU-hour rate card — parameterise per instance type; treat as a dated snapshot
```

**5. The divergence (the reshuffle) — the single most persuasive panel.** Rank namespaces by *allocated* GPUs, then by *SM-active* GPU-hours; the order changes. The team holding the most GPUs is often *not* the team doing the most work:

```promql
# Ranking A — who HOLDS the most GPUs
topk(10, count by (exported_namespace) (DCGM_FI_PROF_SM_ACTIVE{exported_pod!=""}))

# Ranking B — who actually USES the most GPU
topk(10, sum by (exported_namespace) (avg_over_time(DCGM_FI_PROF_SM_ACTIVE{exported_pod!=""}[24h])))
```

Show the two rankings side by side; the rows that jump (high allocation, low SM-active) are your reclaim targets. Prefer `DCGM_FI_PROF_GR_ENGINE_ACTIVE` (graphics/compute engine active fraction) over `SM_ACTIVE` if you want a slightly broader "engine busy" definition; use `PIPE_TENSOR_ACTIVE` if the story is specifically "holding GPUs but not doing tensor math." **Tag every row sourced from a time-sliced pool as `unattributable`** per the caveat above, rather than ranking it as if the label were trustworthy.

### The dollar framing — the part finance repeats

The numbers are the whole point of publishing (all figures below are **2026, order-of-magnitude industry snapshots** — state them as such, not as precise constants):

- Fleet utilisation averages roughly **10–25%** in practice; call it **~15%**, so **~85% of paid GPU-hours are idle.**
- On **500 H100s at $2–3/GPU-hour** — a range consistent with published 2026 GPU-cloud rate cards, which run from roughly **$2.55/hr (reserved)** to **$7.49/hr (on-demand, newer accelerators)** — a fleet running 24×7 is **$8.76M–$13.1M/yr** of allocation. At 15% utilisation the *idle* portion alone is on the order of **$7.5M–$11M/yr.**
- You don't promise "fix it all." You promise a **realistic, partial recovery target.** Two independent anchors support this: a flat 10-point utilisation improvement (15% → 25%) on 500 H100s is roughly **$0.9M–$1.3M/yr recovered**; separately, teams that fix batching specifically have reported taking a *single serving fleet* from under 20% utilisation to over 70% — i.e. the ceiling on a *targeted* fix is far higher than the fleet-wide average improvement, but the fleet-wide number is the one you should promise, because not every namespace is a batching-fixable inference service.

Two framings, one graph: engineers see "85% idle SMs," finance sees "$1M/yr on the table." The allocated-vs-utilised dollar gap is the object that speaks both languages.

## Perspectives

**Engineer.** "85% idle SMs" is the sentence that lands with a peer — it's a hardware occupancy fact, provable from your own cluster's `SM_ACTIVE` series, with no framing required. It's also the sentence that gets you taken seriously in a systems interview, because it proves you know the difference between a device counter and a workload's real behavior.

**Finance/CFO.** "$X/year on the table" is the only sentence that gets budget approved. The module's dollar math exists to make this translation mechanical, not persuasive-by-vibes: allocated GPU-hours (the invoice) minus SM-active GPU-hours (the value delivered), times the rate card, per namespace, per day. No adjectives required — the number does the persuading.

**Platform-vendor.** ScaleOps and Datadog didn't invent this gap — they built products around it because it's real and recurring enough to sell. That a paid vendor's entire GPU-monitoring architecture (device-level, process-level, and Hopper-only advanced pipeline metrics, plus error metrics) is organized around exactly the `GPU_UTIL`-vs-`SM_ACTIVE` distinction this module teaches is independent commercial validation that you're not chasing a course-invented problem.

**Failure-mode.** The gap is not a one-time finding you fix and move past — it recurs continuously as teams scale allocation faster than usage. The worked example's reshuffle, where the biggest *holder* of GPUs isn't the biggest *user*, is a durable organizational pattern (teams over-request for headroom, inference services provision for peak and idle at the median), not a one-off bug you patch. The dashboard's job is to keep surfacing it as the org grows, not to be a single audit.

## Real-world use cases

- **ScaleOps — "GPU Cost Optimization"** — https://scaleops.com/blog/gpu-cost-optimization/. A vendor whose entire product is the allocated-vs-used gap. **What it shows:** independent commercial validation that the allocated-vs-utilised framing is a real, monetizable industry problem, not a course-invented thesis.
- **rack2cloud — "GPU Utilisation & Cloud Waste"** — https://www.rack2cloud.com/gpu-utilization-cloud-waste/. **What it shows:** the "~15% utilised / 85% idle" fleet-average framing that this lesson's dollar math is built on — cite as a directional, dated industry figure.
- **Datadog — GPU Monitoring Reference Architecture** — https://www.datadoghq.com/architecture/gpu-monitoring/. Confirmed live: Datadog's production metric taxonomy spans device-level (`gpu.sm_active`), process-level per-pod attribution (`gpu.process.sm_active`, used to "detect idle or zombie allocations"), Hopper-only advanced pipeline metrics (`gpu.tensor_active`, `gpu.fp16_active`, `gpu.sm_occupancy`), and error/reliability metrics (`gpu.errors.xid.total`, `gpu.remapped_rows.*`). **What it shows:** a paid observability product's entire GPU architecture mirrors this capstone's own dashboard design — the "how a paid product does it" reference the module points to.
- **NVIDIA/NVSentinel** — https://github.com/nvidia/nvsentinel. Confirmed live: an open-source, Kubernetes-native GPU fault-detection and remediation system, validated across Volta through Blackwell architectures, running the same DCGM-based health monitoring this module's L5 covers. **What it shows:** what running this module's full discipline (health automation plus honest utilisation telemetry) looks like at real production fleet scale — a natural closing reference tying L5's automation back into the capstone's fleet picture.

## Worked example

**Reading one cluster's day.** A 3-node, 24×A100 cluster, one day, rate card $2.50/GPU-hour. dcgm-exporter with the k8s labels on; the query pack loaded.

- **Allocated (pod-resources):** `team-research` holds 12 GPUs, `team-serving` holds 8, `team-batch` holds 4. All 24 allocated ⇒ **576 allocated GPU-hours/day**, **$1,440/day** billed.
- **Utilised (SM-active over 24h):** mean `SM_ACTIVE` — research **0.62**, serving **0.11**, batch **0.44**. Utilised GPU-hours = `research 12×24×0.62 = 178.6`, `serving 8×24×0.11 = 21.1`, `batch 4×24×0.44 = 42.2` ⇒ **241.9 utilised GPU-hours**, **42% fleet** (already better than typical — a healthy cluster).
- **Wasted = allocated − utilised:** research 109.4, **serving 170.9**, batch 53.8 GPU-hours. In dollars: research $273, **serving $427**, batch $135 — **$835/day, ~$305k/yr** on this small cluster.
- **The reshuffle:** by *allocation*, research (12) > serving (8) > batch (4). By *SM-active GPU-hours*, research (178.6) > batch (42.2) > **serving (21.1)** — serving drops from 2nd to last. **`team-serving` is the reclaim target:** 8 GPUs held, 11% SM-active, $427/day idle. The util-lie detector confirms it — serving's GPUs show `GPU_UTIL` in the 80s (an inference server always has a kernel resident) while `SM_ACTIVE` sits at 0.11.

**The remediation coda — turning an industry benchmark into a defensible projection (the CFO-test skill).** Don't promise 100% recovery; promise a credible partial move. If `team-serving`'s batching gets fixed (L6/L7's tools — continuous batching, right batch size, no fp32 stragglers) and its mean `SM_ACTIVE` rises from 0.11 to a still-conservative **0.40** — well short of the <20%→>70% ceiling some teams report for a targeted batching fix, and short of research's own 0.62 — utilised GPU-hours rise from 21.1 to `8×24×0.40 = 76.8`, and wasted drops from 170.9 to `192 − 76.8 = 115.2` GPU-hours: **$427/day → $288/day**, a **$139/day (~$51k/yr) recovery from one namespace, on a 24-GPU cluster.** That's the shape of the CFO-test answer: name the namespace, name the mechanism (batching, not more hardware), name a conservative target short of the cited ceiling, and show the dollar delta the reader can verify from the graph.

That two-part story — allocated vs utilised, the reshuffle, the dollar gap, one named reclaim target, one conservative remediation projection — is the shape of the blog's central section, run on *your* cluster's real numbers.

## Practice — THE MODULE DELIVERABLE

Build it against [`../practice/gpu-dashboard-lie/README.md`](../practice/gpu-dashboard-lie/README.md); acceptance is the [module checkpoint](../checkpoint.md).

1. **Dashboard.** A Grafana dashboard with, per namespace: allocated GPU-hours vs utilised (SM_ACTIVE-weighted) GPU-hours on one axis, a **wasted-GPU-hours** panel, and a **$-gap** panel (`gap × rate`). Include the divergence (two rankings side by side), and if any GPUs in your cluster are time-sliced or MPS-shared, tag their rows `unattributable` rather than ranking them at face value.
2. **The util-lie exhibit.** A panel from *your own cluster* showing `DCGM_FI_DEV_GPU_UTIL` pinned >90 while `DCGM_FI_PROF_SM_ACTIVE` sits <0.2, with the detector query firing. This is the money screenshot.
3. **The PromQL pack.** The five queries above, parameterised (namespace, window, rate), committed as a reusable file — the same queries the `gpu-cost-operator` will run.
4. **The blog post.** *"Your GPU utilisation dashboard is lying to you."* Thesis (allocated ≠ utilised; `GPU_UTIL` is a duty-cycle lie) → the exhibit → the per-namespace gap on real numbers → the dollar framing (explicitly dated/directional where it draws on industry figures) → the reclaim recommendation with a conservative, not maximal, recovery target. Fold in the L7 before/after round-trip as the "and here's how you *fix* an inefficient GPU once you've found it" section.

**Acceptance = the module checkpoint:** the dashboard renders allocated-vs-utilised per namespace with a dollar gap; the util-lie exhibit is captured from a real cluster with the detector query; the PromQL pack is committed and reusable; the blog post is drafted with your cluster's real gap, its dollar figure, and (if applicable) the time-slicing attribution caveat. The PromQL pack and the dollar-gap definition must be importable into the `gpu-cost-operator` unchanged — that's the module-11 handoff.

## Common pitfalls

1. **Quoting "~15% average utilization" as a precise, universally-sourced statistic.** It's an industry-consensus order of magnitude from 2026 material, not a measured constant for any specific fleet. Always caveat it as directional and dated, and use *your own cluster's* measured number as the headline figure.
2. **Presenting the dollar gap as money you will definitely recover, rather than money currently wasted with a credible, partial recovery target.** The worked example's move from 11% to a conservative 40% (not the cited 70% ceiling) is the right level of promise — oversell to "100% recovery" and the CFO conversation loses credibility the first time reality undershoots it.
3. **Forgetting to caveat the divergence query's time-sliced/unattributable bucket.** An unqualified per-namespace ranking under time-slicing silently misattributes waste to whichever pod's label happened to land last in the exporter's scrape — this quietly corrupts the exact chargeback number the artifact exists to make trustworthy.
4. **Building the wasted-GPU-hours query as a rough `count()`-based allocation proxy and forgetting to swap in the real pod-resources join when it's available.** The `{exported_pod!=""}` filter is a reasonable Prometheus-only stand-in, but the `gpu-cost-operator` and a rigorous chargeback number need the actual module-04 allocation join — ship the proxy for the blog, note explicitly that production use requires the real join.

## Self-check

- Write, from memory, the PromQL for "GPUs allocated to a pod but idle for more than 15 minutes." **Answer:**
  ```promql
  count by (exported_namespace) (
      avg_over_time(DCGM_FI_PROF_SM_ACTIVE{exported_pod!=""}[15m]) < 0.05
  )
  ```
  The load-bearing parts: **`SM_ACTIVE`** (the honest occupancy fraction, *not* `DCGM_FI_DEV_GPU_UTIL`, which would hide the idle GPUs), **`avg_over_time([15m])`** (sustained idle, not an instantaneous dip), **`< 0.05`** (essentially dead SMs), and the **`{exported_pod!=""}`** filter (there *is* an allocation — a pod holds it). In the operator you replace that label filter with an explicit AND against the pod-resources allocation set.
- Explain the utilisation trap and the allocated-vs-utilised dollar gap to a CFO in two minutes. **Answer:** "Our GPU dashboard says the fleet is ~90% busy. That number is a lie — not maliciously, it's how the metric is defined: it reads 'busy' if *any* work touched the GPU in the last second, even 1% of the chip. The honest metric — the fraction of the GPU's compute cores actually working — averages about 15% industry-wide, and I measured ours directly. So we *allocate and pay for* 100% of these GPUs, but only *use* a fraction of them. On our 500 H100s at ~$2.50/hour, that's roughly $10M/year allocated and several million a year of it sitting idle. I built a per-team dashboard that shows exactly which teams hold GPUs they aren't using — the gap, in dollars, per namespace. If we recover even 10 points of utilisation by right-sizing the worst offenders, that's roughly $1M/year back. It's not a new purchase or a vendor; it's reclaiming what we already own." (Allocated = the invoice; utilised = the value; the gap = the waste, in dollars, per team.)
- Which single metric is simultaneously the interview answer, the blog thesis, and the module-11 input? **Answer:** The **allocated-vs-utilised GPU-hours gap** — allocated GPU-hours (from pod-resources) minus `SM_ACTIVE`-weighted utilised GPU-hours, rendered in dollars. It's the interview opener, the blog's thesis, and the exact quantity the module-11 cost work and the `gpu-cost-operator` consume.
- Why should the "~15% fleet utilization" industry figure be presented as a dated, directional snapshot rather than a precise constant? **Answer:** Because it's aggregated across many organizations' 2026-era published material (vendor blogs, cost-optimization posts), not a controlled measurement of any one fleet — utilisation varies enormously by workload mix, and the number will drift as batching/scheduling practices improve industry-wide. Presenting it as a hard constant invites a CFO or a peer to challenge the one number that's least defensible; your own cluster's measured figure is the number to actually stand behind.
- How does the L4 time-slicing attribution hole change what the capstone's divergence dashboard can honestly claim about a namespace's SM-active GPU-hours? **Answer:** Under time-slicing, `SM_ACTIVE` is a device-level counter — multiple pods can share the physical GPU while the metric stays attached to the device, not cleanly to one tenant. So for any GPU in a time-sliced or MPS-shared pool, the dashboard cannot honestly rank that namespace's SM-active hours against a dedicated-GPU namespace's; it can only report an `unattributable_gpu_hours` bucket for that pool. Only MIG-partitioned or dedicated GPUs support a clean per-namespace ranking.

## Connections & what's next

This is the last lesson in the module — there's no next lesson, only the gate. Two places carry this artifact forward: the [module checkpoint](../checkpoint.md), which is where you prove — unaided — that you can state what `GPU_UTIL` measures, alert on the right metric, classify XIDs, define TTFT/TPOT, write the allocated-but-idle query from memory, pass the CFO test, and produce the artifact itself; and **module 11 (GPU cost economics)**, which takes this capstone's dollar-gap query and per-namespace waste numbers as its direct input dataset — everything you built here is the raw material module 11 turns into fleet-wide cost modeling and chargeback design.

## References & further reading

**Primary sources**
- NVIDIA GPU Operator — "Time-Slicing GPUs in Kubernetes" — the authoritative source on why time-sliced GPUs share one physical device and what that costs in attribution fidelity; read for the hardware basis of this lesson's `unattributable_gpu_hours` caveat. https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html

**Real-world engineering blogs**
- ScaleOps — "GPU Cost Optimization" — the allocated-vs-used framing and reclaim economics, from a vendor whose whole product is this gap. https://scaleops.com/blog/gpu-cost-optimization/
- rack2cloud — "GPU Utilisation & Cloud Waste" — the "~15% utilised / 85% idle" fleet numbers and dollar-of-waste framing used in this lesson's dollar section. https://www.rack2cloud.com/gpu-utilization-cloud-waste/
- Datadog — "GPU Monitoring Reference Architecture" — a paid observability product's device/process/pipeline/error metric taxonomy, confirmed live and matching this capstone's dashboard design. https://www.datadoghq.com/architecture/gpu-monitoring/
- NVIDIA/NVSentinel (GitHub) — open-source, Kubernetes-native GPU fault detection and remediation, validated Volta through Blackwell; confirmed live. https://github.com/nvidia/nvsentinel

**Deeper dives**
- ScaleOps — "GPU Sharing in Kubernetes: MIG vs MPS vs Time-Slicing" — the source of this lesson's direct quote on DCGM metrics staying device-level under time-slicing; go here for the full MIG/MPS/time-slicing comparison behind the attribution caveat. https://scaleops.com/blog/kubernetes-gpu-sharing/
