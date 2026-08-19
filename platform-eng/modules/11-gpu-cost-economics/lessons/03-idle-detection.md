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
sources: 15
---

# Idle GPU detection and the cost of false positives

> Module: [💰 11 — GPU cost and unit economics](../README.md) · Deliverable: [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md)

## Where this fits

Lesson 02 closed the books on a fleet slice and produced one number you cannot ignore: **gap B**, the allocated-but-not-utilised GPU-hours, in dollars. On the worked fleet it was $846/day of tenant-side waste against $329/day of platform-side unallocated capacity. The lesson deliberately stopped there. It told you the gap exists, how large it is, and that it factorises into *hours held* `h` and *mean SM-active fraction while held* `ā`. It did not tell you which parts of that gap you may act on.

This lesson is the act-on-it lesson, and its central claim is uncomfortable: **gap B is not the same as reclaimable idle, and the difference is most of it.** A GPU that dips to `SM_ACTIVE = 0` for 40 ms between decode steps contributes to gap B exactly the same way as a GPU held by a notebook whose owner went on holiday. Both are dollars. Only one is a thing you can do something about, and mistaking the first for the second is how a cost programme causes an outage.

So this lesson does four things. It splits gap B into states that differ by *duration and recurrence structure*, not by depth. It gives you the actual hardware signals — DCGM field IDs, their exact semantics, and what each one is blind to — so you can build the classifier from evidence rather than folklore. It builds the classifier as a **state machine with dwell times and hysteresis**, because a threshold on its own is not a classifier. And it prices being wrong, in both directions, with arithmetic you can re-run on your own fleet.

Lesson 04 takes the *other* bucket — gap A, capacity nobody holds — and shows that free is not the same as usable. Lesson 05 divides whatever ledger survives this lesson by an application counter. Lesson 08 turns the reclaim policy into something a tenant is actually told about.

## Why this matters

The money is real and it is the largest single controllable line most GPU fleets have. At a rate `r` per GPU-hour, a fleet of `G` GPUs running at mean utilisation `ā` carries `G × 8,760 × (1 − ā) × r` of annual non-productive spend. Put the module's numbers in: 512 H100 at `r = $2.99` (the August 2026 `EffectiveCost` snapshot from lesson 02, inside an observed on-demand band of roughly $2–$7) and `ā = 0.40` gives `512 × 8,760 × 0.60 × 2.99 ≈ $8.05M/year`. Substitute your own `G`, `ā` and `r`; the shape does not change. Some meaningful fraction of that is genuinely recoverable, and finding it is the highest-leverage work a GPU platform engineer does.

The danger is equally real and it is asymmetric in a way most FinOps intuition gets wrong. In the CPU world a false-positive idle call is cheap: you flag a VM, someone checks, it was in use, you apologise, you move on. On a GPU serving fleet a false positive can cost far more than the idle it targeted, because the expensive state is not the process — it is what lives in **HBM**: tens of gigabytes of model weights, a populated KV cache, captured CUDA graphs, autotuned kernel selections. Evicting a quiet serving replica destroys all of it and takes the replica out of rotation for minutes while the rest of the fleet absorbs its traffic at higher occupancy — which is exactly when queueing delay blows up.

The worked example below prices one such mistake precisely: reclaiming a quiet inference replica for a 20-minute dwell **saves $1.00 and spends 2.8% of a monthly error budget**. That ratio, derived rather than asserted, is the entire reason this lesson exists.

And there is a third failure that is quieter than both: **detecting idle correctly and reporting it as one number.** "$21,178 of idle this week" is a true sentence that names no owner, no mechanism and no action, so nothing happens, and the next quarter you say it again with a bigger number. States with owners, or don't publish.

## What's new here (calibration)

- **Already yours (skip):** the pod-resources → DCGM join (module 04); `SM_ACTIVE` versus `GPU_UTIL` and why the latter lies (module 05); the `sum_over_time(x[W]) × Δ/3600` integral and the `avg_over_time` inflation bug (module 05, capstone); the four sharing regimes and their attribution accuracy (lesson 01); the two ledgers and the three nested gaps (lesson 02).
- **New angle 1 — idle is a *temporal* classification, not a threshold.** The archetypes separate by the *duration distribution of their quiescent gaps*, spanning six orders of magnitude from 40 µs to 40 hours. A rule that names only a depth (`SM_ACTIVE < 0.05`) has thrown away the only axis that discriminates.
- **New angle 2 — the state machine, with dwell, hysteresis and re-arm.** Including the two error paths drawn explicitly, and the rule for setting the dwell from a *measured* quantile of the workload's own gap distribution rather than a number someone liked.
- **New angle 3 — the signal layer done properly.** Real DCGM field IDs and their verbatim semantics, which ones ship enabled in `dcgm-exporter`'s default counters and which are commented out, what `FB_USED` tells you that no activity metric can, and why board power is a *coarse* cross-check (the sampling caveat is measured and published).
- **New angle 4 — the expected-value model, derived and then solved for the break-even.** Not "weigh the costs" but: for batch the classifier may be wrong 93% of the time and still pay; for serving it must be wrong less than about 0.2% of the time. Two orders of magnitude apart, from one inequality.
- **New angle 5 — reclaim is not the only remediation, and often not the best one.** Sleep-mode offload (vLLM ships `vllm:engine_sleep_state` with two levels), scale-to-zero on request signals, MPS/time-slice packing, TTL cullers, descheduler consolidation, preemption — each mapped to the state it actually fixes.
- **New angle 6 — what the tools mean by "idle" is a different quantity.** OpenCost's `__idle__` allocation is `asset cost − allocated cost`, clamped at zero, per node or per cluster. That is lesson 02's **gap A**, not gap B. Anyone comparing your idle number to OpenCost's is comparing two different measurements, and knowing that from the source is a cheap way to win an argument.

## Core concepts

### 1. Gap B is not one thing: the state taxonomy

Lesson 02 gave you three nested gaps. Gap B — allocated minus utilised — is the tenant-facing one, and it is a sum over wildly different phenomena. Split it by two questions that have mechanical answers: **is there a CUDA context on the device?** and **how long has the quiescence lasted?**

```
  DECOMPOSING ALLOCATED GPU-HOURS.  Every box is GPU-hours; the ⊞ boxes
  are the ones a reclaim policy may touch.

  ALLOCATED (② from lesson 02) = Σ share(p) × hours_held(p)
  │
  ├─ S1  WORKING            SM_ACTIVE above the workload's floor.
  │                         Not waste. Efficiency lives here (gap C).
  │
  ├─ S2  MICRO-IDLE         Quiescent for LESS than the dwell time.
  │      (not actionable)   Inter-request gaps, inter-step gaps, kernel
  │                         launch gaps, all-reduce waits, page faults.
  │                         REAL DOLLARS, ZERO RECLAIM SURFACE.
  │                         Fixed by batching/pipelining, never eviction.
  │
  ├─ S3 ⊞ HELD, NO CONTEXT  Device bound to a pod; no CUDA context on it.
  │      "cold hold"        FB_USED at the device floor. Crashed trainer
  │                         still holding its slot, notebook with a dead
  │                         kernel, stuck init, sidecar-only pod that
  │                         requested a GPU it never opened.
  │                         NOTHING WARM TO LOSE. Cheapest to act on.
  │
  ├─ S4 ⊞ HELD, CONTEXT,    Context present, weights resident, SMs quiet
  │      QUIESCENT ≥ dwell   for longer than the dwell time.
  │      "warm hold"        ── THIS IS THE DANGEROUS ONE ──
  │                         Identical signal from:
  │                           · a serving replica between requests
  │                           · a trainer inside a checkpoint barrier
  │                           · a job that finished and never exited
  │                           · a notebook whose owner went home
  │                         Disambiguated by CLASS + GUARDS, not by
  │                         a better utilisation threshold.
  │
  └─ S5 ⊞ HELD, CONTEXT,    Context present, quiescent, AND the pod's own
         APP-SIGNAL IDLE     application says it has no work: zero
         "attested idle"     in-flight requests, empty queue, run
                             completed. The only state where a machine
                             can be confident.

  gap B (lesson 02)  =  S2 + S3 + S4 + S5
  reclaimable idle   =  S3 + S5 + (the part of S4 a policy dares touch)

  ⇒ THE HEADLINE: reclaimable idle ⊂ gap B, usually a MINORITY of it.
    Publishing gap B as "recoverable waste" overstates by the S2 term,
    which on a busy serving fleet is the largest single component.
```

Three consequences fall out immediately.

**S2 has no reclaim surface at all.** A GPU serving decode requests at batch 32 with 5 ms gaps between forward passes spends real time at `SM_ACTIVE ≈ 0`. Integrated over a week that is a large number of GPU-hours and a large number of dollars, and there is no eviction that recovers any of it. The lever is batching, continuous batching, prefill/decode disaggregation, chunked prefill — throughput engineering. Putting S2 in a "reclaim this" report is the fastest way to get the report ignored by the people who could actually move it.

**S3 is nearly free to act on and is chronically under-detected**, because the default metric set has no field that says "a CUDA context exists." You have to infer it, and §2 shows how.

**S4 cannot be resolved by a better utilisation metric.** This is worth stating as strongly as the time-slicing impossibility result in lesson 01: the device-level activity counters of a warm serving replica between requests and of an abandoned-but-warm job are *the same signal*. No threshold, no additional profiling field, no smarter statistic separates them, because the hardware state genuinely is the same. What separates them is information the GPU does not have — what the workload is for, and whether anything is asking it for work. That information has to come from a label and from the application.

### 2. The signal layer: what the hardware actually exposes

Every idle rule is built on DCGM fields, so know exactly what each one is. These are the current field IDs and the semantics from the DCGM field header itself (`dcgmlib/dcgm_fields.h`). One thing to note first, because it will bite you on a recent install: **DCGM 4.x renamed the profiling fields**, and the names everybody writes are now compatibility aliases:

```c
/* dcgm_fields.h — current names, then the legacy aliases at the bottom */
#define DCGM_FI_PROF_GR_ENGINE_UTIL_RATIO   1001
#define DCGM_FI_PROF_SM_UTIL_RATIO          1002
#define DCGM_FI_PROF_SM_OCCUPANCY_RATIO     1003
#define DCGM_FI_PROF_TENSOR_UTIL_RATIO      1004
#define DCGM_FI_PROF_DRAM_UTIL_RATIO        1005
...
#define DCGM_FI_PROF_GR_ENGINE_ACTIVE    DCGM_FI_PROF_GR_ENGINE_UTIL_RATIO   // 1001
#define DCGM_FI_PROF_SM_ACTIVE           DCGM_FI_PROF_SM_UTIL_RATIO          // 1002
#define DCGM_FI_PROF_PIPE_TENSOR_ACTIVE  DCGM_FI_PROF_TENSOR_UTIL_RATIO      // 1004
#define DCGM_FI_DEV_POWER_USAGE          DCGM_FI_DEV_BOARD_POWER_WATTS       //  155
#define DCGM_FI_DEV_GPU_UTIL             DCGM_FI_DEV_GPU_UTIL_RATIO          //  203
```

The Prometheus metric names emitted by `dcgm-exporter` still use the legacy spellings, so your queries do not change — but if you go reading the header or the API looking for `DCGM_FI_PROF_SM_ACTIVE` and cannot find it, that is why.

Now the fields themselves, with what each one is actually measuring and what it is blind to:

| Field (metric name) | ID | Semantics (from the field header) | Range | What it is blind to | Role in an idle rule |
|---|---|---|---|---|---|
| `DCGM_FI_DEV_GPU_UTIL` | 203 | "GPU Utilization" — the NVML duty-cycle sample: the fraction of the sample period during which **any** kernel was resident | 0–100 (%) | *how much* of the chip that kernel used. One warp on one SM reads the same as a full-device kernel | **None. Never use it.** It reads ~100 on a decode-bound server whose SMs are 4% occupied |
| `DCGM_FI_PROF_SM_ACTIVE` | 1002 | "The ratio of cycles an SM has at least 1 warp assigned (computed from the number of cycles and elapsed cycles)", averaged over SMs | 0.0–1.0 | whether the resident warps did useful arithmetic — warps stalled on memory still count as active | **Primary activity signal.** Its floor is what "quiet" means |
| `DCGM_FI_PROF_SM_OCCUPANCY` | 1003 | "The ratio of number of warps resident on an SM (number of resident as a ratio of the theoretical maximum number of warps per elapsed cycle)" | 0.0–1.0 | same blindness to usefulness; much lower absolute values, so a naive threshold ported from `SM_ACTIVE` misfires | Optional depth signal; do **not** substitute for `SM_ACTIVE` |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | 1004 | "The ratio of cycles the any tensor pipe is active (off the peak sustained elapsed cycles)" | 0.0–1.0 | is legitimately ~0 for healthy memory-bound decode, fp32 work, embeddings, data movement | Gate against "busy but trivial" — **never a standalone idle signal** |
| `DCGM_FI_PROF_DRAM_ACTIVE` | 1005 | "The ratio of cycles the device memory interface is active sending or receiving data" | 0.0–1.0 | nothing about compute | Excellent companion: decode-bound serving shows low `SM_ACTIVE` with **high** `DRAM_ACTIVE`. A truly idle GPU is low on both |
| `DCGM_FI_DEV_FB_USED` | 252 | "Used Frame Buffer in MB" | MiB | *why* memory is held | **The context detector.** Distinguishes S3 (cold hold) from S4 (warm hold) |
| `DCGM_FI_DEV_FB_FREE` / `FB_RESERVED` | 251 / 253 | free / reserved framebuffer in MiB | MiB | — | Establishes the per-SKU floor so `FB_USED` has a baseline |
| `DCGM_FI_DEV_POWER_USAGE` | 155 | "Power usage for the device in Watts" | W | see the sampling caveat below | Coarse, tamper-resistant cross-check only |
| `DCGM_FI_DEV_SM_CLOCK` | 100 | "SM clock for the device" | MHz | — | Secondary: idle devices drop to their idle clock domain |

**Which of these you get for free matters more than it should.** `dcgm-exporter`'s shipped `etc/default-counters.csv` decides what a default install exposes, and the current file makes a specific and awkward set of choices:

| Field | In `default-counters.csv`? |
|---|---|
| `DCGM_FI_DEV_GPU_UTIL` | **enabled** |
| `DCGM_FI_PROF_SM_ACTIVE` | **commented out** |
| `DCGM_FI_PROF_SM_OCCUPANCY` | commented out |
| `DCGM_FI_PROF_GR_ENGINE_ACTIVE` | enabled |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | enabled |
| `DCGM_FI_PROF_DRAM_ACTIVE` | enabled |
| `DCGM_FI_DEV_POWER_USAGE` | enabled |
| `DCGM_FI_DEV_FB_USED` / `FB_FREE` / `FB_RESERVED` | enabled |
| `DCGM_FI_DEV_SM_CLOCK` | enabled |

So on a stock install the *wrong* utilisation field is present and the *right* one is not. That is not carelessness on anyone's part — the profiling fields carry a collection cost — but it means "our dashboard shows the fleet at 85% utilised" is the default outcome, and a bad idle rule is the default rule. Two operational riders, both from module 05: a custom counters CSV **replaces** the default set rather than extending it, so when you enable `DCGM_FI_PROF_SM_ACTIVE` you must re-list everything else you still want; and the exporter's `--collect-interval` defaults to **30000 ms**, which is the `Δ` in every integral below.

**On `GR_ENGINE_ACTIVE` versus `SM_ACTIVE`.** `GR_ENGINE_ACTIVE` (1001) is "active if a graphics/compute context is bound and the graphics pipe or compute pipe is busy" — closer to a device-busy flag than to an occupancy measure, and it is what OpenCost's utilisation query reads. It is a fine liveness signal and a poor waste signal, for the same reason as `GPU_UTIL`: it does not scale with how much of the chip is in use. Use `SM_ACTIVE` for the ledger and for the idle rule; `GR_ENGINE_ACTIVE` is acceptable only as a cross-check where `SM_ACTIVE` is unavailable (older SKUs, MIG configurations where profiling is restricted).

**On power as a cross-check — and its measured limitation.** Board power is attractive because it is hard to fake and survives exporter gaps: an idle datacentre GPU sits far below its TDP (H100 SXM is a 700 W part and idles well under 100 W; measure your own floor rather than trusting a number from a blog). But NVML/`nvidia-smi` power reporting is not a continuous integral. The published measurement study *Part-time Power Measurements: nvidia-smi's Lack of Attention* (arXiv 2312.02741) reports that on A100 and H100 only about **25% of runtime is sampled** for power — during the other 75% the GPU can be drawing very different power that the reading never sees — and that following its recommended practices reduces energy-measurement error by up to 35%. So power belongs in an idle rule as a corroborating term with a wide margin, never as the discriminating term. The right way to use it is: *if power is high, something is definitely happening* (a strong veto on "idle"), while *low power is weak evidence of idleness*.

**On what no DCGM field gives you: process presence.** There is no default-enabled field that says "a CUDA context exists on this device." DCGM has `DCGM_FI_DEV_PROCESS_ACCOUNTING_STATS` (205), documented as only supported when the host engine runs as root unless accounting is enabled ahead of time (`nvidia-smi -am 1`), and it is not in the default counter set. The practical substitutes, in order of preference:

1. **`FB_USED` against a measured floor.** A device with no compute context sits at its baseline (a small reserved amount); a device holding a 70 GB model sits at 70 GB plus cache. Threshold on `FB_USED > floor + margin` to detect "a context with real state." This is the cheapest reliable S3/S4 discriminator and it uses a default-enabled field.
2. **`nvidia-smi --query-compute-apps=pid,used_gpu_memory --format=csv`** or the NVML equivalent, run by a node agent. Exact, needs host PID visibility to map back to pods.
3. **`dcgm-exporter --kubernetes-virtual-gpus`** (default `false`), which brings NVML per-process data into play for the shared-device case — but note from lesson 01 that only the duty-cycle and framebuffer fields get a per-pod split this way; every `DCGM_FI_PROF_*` field stays device-level.

Establish your floors once, on your own hardware, and write them into the rule as named constants:

```console
$ nvidia-smi --query-gpu=index,name,power.draw,clocks.sm,memory.used \
             --format=csv,noheader,nounits
0, NVIDIA H100 80GB HBM3, 71.32, 345, 3
1, NVIDIA H100 80GB HBM3, 698.44, 1980, 74213
                          ▲       ▲     ▲
                          │       │     └─ GPU 0: 3 MiB → no compute context (S3)
                          │       │        GPU 1: 74 GiB → weights resident (S4)
                          │       └─ idle clock domain vs boost clock
                          └─ ~10% of TDP vs at the power limit

# Representative transcript from an H100 SXM node; your floors will differ by
# SKU, driver, persistence mode and ECC state. MEASURE THEM.
```

### 3. Why a threshold is not a classifier: the time-domain view

Every archetype that people argue about produces the *same depth* of quiescence. `SM_ACTIVE` goes to approximately zero for a decode gap, a checkpoint barrier and an abandoned notebook alike. What differs by six orders of magnitude is **how long the quiescence lasts and how it recurs**.

```
  QUIESCENT-GAP DURATION — LOG SCALE.  ▓ = where each archetype's gaps live.
  The classifier's whole job is to place a dwell line on this axis.

   10µs   100µs    1ms    10ms   100ms    1s     10s    1min   10min   1h    10h
    │       │       │       │       │      │       │      │      │      │      │
 kernel-launch / sync gaps
    ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
                     all-reduce & pipeline-bubble waits
                     ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
                          decode inter-step gaps (batched serving)
                          ▓▓▓▓▓▓▓▓▓▓▓▓▓▓
                                 dataloader stalls (host-bound input)
                                 ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
                                         inter-REQUEST gaps, low QPS
                                         ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
                                                 checkpoint barrier
                                                 ▓▓▓▓▓▓▓▓▓▓▓▓▓▓
                                                   eval / validation pass
                                                   ▓▓▓▓▓▓▓▓▓▓▓▓▓▓
                                                     traffic trough (diurnal)
                                                     ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
                                                       human think-time
                                                       ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
                                                          finished-but-held job
                                                          ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
                                                            abandoned notebook
                                                            ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
    │                                             │                │
    └─ SCRAPE FLOOR: Δ = 30 s. Everything left of │ is INVISIBLE   │
       here is aliased into a single sample and   │ to any dwell   │
       shows up only as a reduced SM_ACTIVE mean. │ test — it is   │
       IT IS S2 AND IT IS NOT RECLAIMABLE.        │ state S2.      │
                                                  └─ candidate dwell region:
                                                     20 min … 4 h, per class

  THE POINT: depth does not separate these. DURATION does.
  A rule that says only "SM_ACTIVE < 0.05" has collapsed this whole axis.
```

Two structural facts follow, and they set the rest of the design.

**Your sampling interval is a hard floor on what you can classify.** At `Δ = 30 s`, everything with a gap structure shorter than a minute is invisible as a *gap* — it appears only as a depressed mean. You cannot reclaim what you cannot resolve, and you should not try: those GPU-hours are state S2 and their fix is throughput engineering.

**The dwell threshold must be derived from the workload, not chosen.** The rule that survives review is:

```
  t_dwell(class)  ≥  k × q99( quiescent-gap duration | class )      k ≈ 2 … 3

  where q99 is the 99th percentile of the class's OWN observed
  gap-length distribution, measured over at least one full weekly cycle
  so diurnal troughs and weekend patterns are included.
```

You can measure `q99` directly rather than guessing it. The gap length is the run length of consecutive quiet samples, and Prometheus can approximate the tail of that distribution cheaply by asking, for a family of candidate windows `W`, what fraction of the time the *maximum* over the trailing `W` was below the quiet threshold:

```promql
# Fraction of time this class had NO activity at all for a full 20 minutes.
# Sweep W over 5m, 10m, 20m, 30m, 45m, 1h, 2h, 4h and read off where the
# curve collapses toward zero — that knee is q99 for the class.
avg by (workload_class) (
  avg_over_time(
    ( max_over_time(gpu:sm_active:ratio[20m]) < bool 0.02 )[7d:30s]
  )
)
```

Run that sweep per class before you set a single threshold. On a serving fleet the curve typically stays materially above zero out to 30–45 minutes because of overnight troughs, then collapses; on synchronous training it collapses just past the longest checkpoint barrier; on notebooks it barely collapses at all, which is precisely why notebooks are the reliable win. **Setting the dwell without doing this sweep is how you get a rule that pages during the nightly traffic minimum.**

### 4. The classifier as a state machine

With a depth threshold `θ`, a dwell `t_d`, a re-arm threshold `θ_r > θ` and a cooldown, the classifier becomes a state machine with explicit error paths. This is the shape to implement, and the shape to draw on a whiteboard when someone asks how your reclaim works.

```
  IDLE CLASSIFIER — one device-or-instance, one holder, per workload class.
  θ  = quiet depth       (e.g. max_over_time(SM_ACTIVE) < 0.02)
  θ_r= re-arm depth      (θ_r > θ, e.g. 0.10 — hysteresis band)
  t_q= quiet confirm     (2–3 scrape intervals; kills single-sample noise)
  t_d= dwell             (from §3's q99 sweep, per class)

        ┌──────────────────────────────────────────────────────────┐
        │                                                          │
        ▼                                                          │
  ┌───────────┐  activity < θ         ┌───────────┐                │
  │  WORKING  │─────for t_q──────────▶│   QUIET   │                │
  │           │◀──── activity ≥ θ_r ──│           │                │
  └───────────┘      (immediate)      └─────┬─────┘                │
        ▲                                   │ still < θ            │
        │                                   │ for t_d              │
        │                                   ▼                      │
        │                            ┌──────────────┐              │
        │                            │ IDLE_CANDIDATE│─────────────┘
        │                            └───┬───────┬──┘  activity ≥ θ_r
        │                                │       │      ⇒ RE-ARM,
        │        ┌───────────────────────┘       │        cooldown C
        │        │ GUARDS PASS?                  │        before this
        │        ▼                               │        holder may be
        │  ┌───────────────┐   guards fail       │        a candidate
        │  │  RECLAIMABLE  │   (any one)         │        again
        │  └───────┬───────┘◀────────────────────┘
        │          │
        │          │ act (per class):
        │          │   S3 cold hold  → evict
        │          │   batch         → preempt / checkpoint-and-requeue
        │          │   dev           → cull (TTL)
        │          │   serving       → NEVER evict; sleep / scale-to-zero
        │          ▼
        │   ┌─────────────┐
        └───│  RECLAIMED  │
            └─────────────┘

  ── GUARDS (ALL must pass before RECLAIMABLE) ─────────────────────────
  G1 class enabled for reclaim               (label: workload_class)
  G2 no in-flight application work           (app metric, §6)
  G3 device healthy & not draining           (no XID storm, not cordoned)
  G4 no checkpoint/host I/O burst in progress (write-bytes or a hook)
  G5 min-runtime satisfied since pod start   (anti-thrash floor)
  G6 fleet-wide eviction budget not exhausted (blast-radius cap)
  G7 PodDisruptionBudget would still be met

  ── THE TWO ERROR PATHS, NAMED ───────────────────────────────────────
  FALSE POSITIVE ▸ WORKING→QUIET→IDLE_CANDIDATE→RECLAIMABLE→RECLAIMED
                   on a holder that was about to be asked for work.
                   COST: warm state destroyed + capacity removed during
                   the restart window. Priced in §7.
                   MITIGATED BY: larger t_d, guard G2, class gating.

  FALSE NEGATIVE ▸ holder sits in QUIET or IDLE_CANDIDATE forever, or
                   a periodic twitch keeps re-arming it (a 30-second
                   health-check kernel is enough).
                   COST: r per GPU-hour, indefinitely, silently.
                   MITIGATED BY: smaller t_d, and by measuring on the
                   MAX not the MEAN so a twitch does not hide the gap.
                   ⚠ THE TWITCH CASE IS WHY max_over_time IS DANGEROUS
                     AS THE ONLY TEST — see §5.
```

Three design choices in that diagram carry real weight.

**Hysteresis is not optional.** With one threshold, a workload hovering near `θ` flaps between states every scrape, and every flap either fires an alert or, worse, triggers an action. Setting the re-arm depth `θ_r` well above `θ` (a factor of 3–5 is a reasonable start) means it takes real activity to exit the candidate state, and once out, the cooldown `C` prevents an immediate re-entry. Without both, an eviction policy oscillates and you will discover it during an incident.

**The transition out is immediate; the transition in is delayed.** Asymmetry is deliberate: the cost of being slow to *notice* idle is `r` per GPU-hour, small and linear; the cost of being slow to notice *activity* is a wrongful eviction, large and discontinuous. Always prefer the error whose cost is linear.

**Guards are ANDed and are mostly not GPU metrics.** G2 (application attests it has no work), G5 (min-runtime) and G6 (eviction budget) come from outside DCGM entirely. That is the honest reflection of §1: the hardware cannot distinguish S4 from S5, so the distinguishing information must be supplied.

### 5. Strategies, and exactly what each one misclassifies

Every fleet has picked one of these, usually without writing down which. Here is the full menu with the failure modes attached — the two right-hand columns are the ones to read.

| # | Strategy | Signal(s) | What it misclassifies as IDLE (false positive) | What it misses (false negative) |
|---|---|---|---|---|
| 1 | **`GPU_UTIL` threshold** | `DCGM_FI_DEV_GPU_UTIL < X%` | almost nothing — it is too permissive to fire | **nearly everything.** Any resident kernel pins it high: a decode-bound server at 4% SM occupancy, a health-check loop, a spin-wait. The default dashboards' rule, and it finds nothing |
| 2 | **`SM_ACTIVE` mean over window** | `avg_over_time(SM_ACTIVE[W]) < θ` | trainers whose window happened to straddle a checkpoint; serving replicas in a diurnal trough; any workload with a bursty duty cycle whose mean sits below θ while it is doing real work | a job that twitches to 1.0 for one sample every window (mean stays low → it correctly fires) — but conversely a job at a steady 0.06 never fires despite doing nothing useful |
| 3 | **`SM_ACTIVE` max over window** (dwell) | `max_over_time(SM_ACTIVE[W]) < θ` | very little — this is the strict test | **the twitching zombie.** A crashed process whose watchdog runs one tiny kernel a minute keeps `max` above θ forever. Also misses partial idleness (an 8-GPU job where 7 ranks stall) |
| 4 | **Quantile over window** | `quantile_over_time(0.95, SM_ACTIVE[W]) < θ` | a genuinely busy workload that is quiet >5% of the window | tuneable middle ground; the honest default when zombie-twitching is common |
| 5 | **Tensor-pipe activity** | `PIPE_TENSOR_ACTIVE < θ` | **healthy memory-bound decode, fp32 workloads, embedding lookups, data movement** — all legitimately near zero. Firing on this evicts your inference fleet | genuinely idle GPUs are also near zero, so it "works" — which is what makes it seductive and wrong |
| 6 | **Board power** | `POWER_USAGE < W` | workloads running during the 75% of wall-clock the sensor does not sample (see §2); low-power-but-active phases | high idle floors on some SKUs make the separation narrow; clamped/throttled devices |
| 7 | **Framebuffer occupancy** | `FB_USED < floor` | **nothing running but memory pinned** is scored busy — the exact abandoned-notebook case | this is an S3/S4 *discriminator*, not an activity signal. Using it alone means a pod that allocates a big tensor and sleeps forever is never idle |
| 8 | **Process presence** | no compute processes on the device | a trainer between two `torch.distributed` phases that has torn down and rebuilt its context (rare but real) | any warm hold — the process is right there |
| 9 | **App-level attestation** | `vllm:num_requests_running == 0 and vllm:num_requests_waiting == 0`, or a job-completion signal | almost nothing when the app is honest | requires the app to emit it; silent when the metric endpoint dies (a dead exporter looks like zero work — **fail closed, treat absent as busy**) |
| 10 | **Last-activity annotation** (Kubeflow/JupyterHub cullers) | notebook server's last kernel activity | a kernel that is `busy` on a pure-CPU cell keeps the GPU alive; **the culler does not look at the GPU at all** | a kernel idle for the timeout is culled even while a background CUDA stream is still running — and conversely an idle kernel holding a 40 GB model is only culled after the timeout |
| 11 | **Composite (this lesson)** | dwell on `SM_ACTIVE` **and** `DRAM_ACTIVE` low, `FB_USED` to classify S3/S4, power as veto, class label + app attestation as guards | designed to misclassify only when the app lies or the label is wrong | costs you the S2 bucket entirely, by design |

Row 10 deserves expansion, because notebook cullers are the most-deployed "GPU idle" mechanism in the world and almost nobody knows what they measure. The Kubeflow notebook controller's culling loop (`controllers/culling_controller.go`) polls the notebook server's `/api/kernels`; if **any** kernel reports an execution state other than `idle` it stamps the last-activity annotation with *now*, otherwise it takes the most recent `last_activity` across kernels, and scales the Notebook to zero once `now − last_activity > CULL_IDLE_TIME`. The defaults, from the source:

```go
const DEFAULT_CULL_IDLE_TIME        = "1440"   // minutes — ONE FULL DAY
const DEFAULT_IDLENESS_CHECK_PERIOD = "1"      // minutes
const DEFAULT_ENABLE_CULLING        = "false"  // OFF unless you turn it on
```

Read those three lines as a cost statement: **out of the box nothing is culled at all, and when culling is switched on an abandoned GPU notebook is held for a full day first** — `24 × $2.99 = $71.76` per abandoned session on an H100, after the owner walked away. JupyterHub's `jupyterhub-idle-culler` is more aggressive (`--timeout` defaults to **600 s**) but its README is explicit that the timeout must exceed the single-user websocket ping interval (30 s) plus `JupyterHub.last_activity_interval` (5 min), because the activity data is not updated frequently — a published instance of §3's rule that the dwell must exceed the staleness of the signal you are dwelling on. Neither mechanism looks at the GPU: **both are kernel-activity cullers wearing a GPU-cost hat**, and ANDing them with `SM_ACTIVE` and `FB_USED` is a cheap, large improvement.

### 6. The queries, correct on the integral

Everything here builds on module 05's normalising recording rule `gpu:sm_active:ratio` (which gives every series a stable `ns` label and routes label-less devices to `__unallocated__`) and lesson 02's `gpu:allocated:indicator`. `Δ = 30 s` is the exporter's default collect interval and the rule-group interval; **change one and change both**.

```yaml
# queries/idle-states.yaml
#
# INTEGRATION CONSTANT: 30 / 3600. It is the `interval:` of the group.
# Every GPU-hour figure below is  sum_over_time(indicator[W]) × Δ/3600
# — NEVER avg_over_time(x[W]) × W_hours, which extrapolates a workload's
# mean over time when its series did not exist and inflates by
# window/time-present (module 05).
groups:

- name: gpu-idle-signals
  interval: 30s
  rules:

  # ── Per-GPU quiet test, on the MAX not the mean: "nothing at all
  #    happened for a full dwell window". Strict, and the right primitive
  #    for S4/S5. See §5 row 3 for its blind spot (the twitching zombie).
  - record: gpu:quiet_20m:bool
    expr: |
      ( max_over_time(gpu:sm_active:ratio[20m])            < bool 0.02 )
      * ( max_over_time(gpu:dram_active:ratio[20m])        < bool 0.05 )

  # ── The tolerant variant: quiet for 95% of the window. Use this where
  #    watchdog kernels or health checks twitch the max. Tune the
  #    quantile, not the threshold.
  - record: gpu:quiet_q95_20m:bool
    expr: quantile_over_time(0.95, gpu:sm_active:ratio[20m]) < bool 0.02

  # ── CONTEXT DETECTOR: is there real state in HBM?  FB floor is
  #    per-SKU and MEASURED (§2). 2048 MiB is a deliberately generous
  #    margin above an empty H100's few-MiB baseline.
  - record: gpu:has_resident_state:bool
    expr: DCGM_FI_DEV_FB_USED > bool 2048

  # ── POWER VETO: high power means something IS happening, whatever the
  #    activity counters say. Low power is only weak evidence of idle
  #    (25% duty-cycle sampling on A100/H100 — §2), so it is used one
  #    way only.
  - record: gpu:power_veto:bool
    expr: max_over_time(DCGM_FI_DEV_POWER_USAGE[20m]) > bool 200

- name: gpu-idle-states
  interval: 30s
  rules:

  # ── S3: HELD, NO CONTEXT — cold hold. Allocated, quiet, nothing in HBM.
  - record: gpu:state_cold_hold:bool
    expr: |
        gpu:allocated:indicator
      * gpu:quiet_20m:bool
      * (1 - gpu:has_resident_state:bool)
      * (1 - gpu:power_veto:bool)

  # ── S4: HELD, CONTEXT, QUIESCENT — warm hold. THE DANGEROUS STATE.
  #    Note it is NOT reclaimable on its own; it needs a class and G2.
  - record: gpu:state_warm_hold:bool
    expr: |
        gpu:allocated:indicator
      * gpu:quiet_20m:bool
      * gpu:has_resident_state:bool
      * (1 - gpu:power_veto:bool)

  # ── S5: ATTESTED IDLE — warm hold AND the application says it has no
  #    work. The app join is on the pod, so a serving replica with any
  #    in-flight or queued request drops straight out of this state.
  #    FAIL CLOSED: absent app metrics ⇒ not attested ⇒ not S5.
  - record: gpu:state_attested_idle:bool
    expr: |
        gpu:state_warm_hold:bool
      * on (namespace, pod) group_left()
        ( (vllm:num_requests_running == bool 0)
        * (vllm:num_requests_waiting == bool 0) )

- name: gpu-idle-hours-and-money
  interval: 5m
  rules:

  # ── GPU-HOURS PER STATE. Correct integral, per namespace and class.
  - record: ns:idle_gpu_hours_cold_hold:1d
    expr: sum by (ns, workload_class) (sum_over_time(gpu:state_cold_hold:bool[1d]))    * 30 / 3600
  - record: ns:idle_gpu_hours_warm_hold:1d
    expr: sum by (ns, workload_class) (sum_over_time(gpu:state_warm_hold:bool[1d]))    * 30 / 3600
  - record: ns:idle_gpu_hours_attested:1d
    expr: sum by (ns, workload_class) (sum_over_time(gpu:state_attested_idle:bool[1d])) * 30 / 3600

  # ── S2, BY SUBTRACTION. gap B minus everything that dwelled.
  #    This is the number people mistake for reclaimable idle.
  - record: ns:idle_gpu_hours_micro:1d
    expr: |
        ns:gpu_hours_gap_b:1d
      - ( ns:idle_gpu_hours_cold_hold:1d
        + ns:idle_gpu_hours_warm_hold:1d )

  # ── MONEY. Per-model rate, joined the same way as lesson 02 so a
  #    heterogeneous fleet costs correctly. Basis + date live in the
  #    gpu:hourly_rate_usd metric's HELP text, never inline.
  - record: ns:idle_cost_cold_hold:1d
    expr: |
      sum by (ns, workload_class) (
        sum_over_time(
          ( gpu:state_cold_hold:bool
            * on (modelName) group_left() gpu:hourly_rate_usd )[1d:30s]
        )
      ) * 30 / 3600
```

And the alert — note what it does *not* do:

```yaml
- name: gpu-idle-alerts
  rules:
  # Fires ONLY for classes where the action is safe. A serving replica in
  # state S4 is a capacity-planning finding, not a reclaim candidate, and
  # must never reach a page that suggests eviction.
  - alert: GPUReclaimableIdle
    expr: |
        ( gpu:state_cold_hold:bool == 1 )
      or
        ( gpu:state_warm_hold:bool == 1
          and on (namespace, pod) group_left(workload_class)
              gpu_pod_workload_class{workload_class=~"batch|training|dev"} == 1 )
    for: 10m                       # a further guard on top of the dwell
    labels:
      severity: info               # NEVER page on a cost signal
    annotations:
      summary: >-
        Reclaimable idle GPU {{ $labels.gpu }} on {{ $labels.Hostname }}
        held by {{ $labels.namespace }}/{{ $labels.pod }}
        (class {{ $labels.workload_class }})
      action: >-
        cold hold → evict; batch → checkpoint-and-preempt;
        dev → cull. Serving is excluded from this alert by construction.
```

**Three query-level traps worth naming.** First, `avg_over_time` in a *threshold* is legitimate — it is only wrong when you multiply it by a window length to get hours. Keep the two uses mentally separate: thresholds may use any statistic; **integrals must be `sum_over_time(...) × Δ/3600`.** Second, scrape gaps bias idle *downward*: a missing sample is not a quiet sample, so `max_over_time` over a gap-riddled window can pass the quiet test on almost no data. Always pair the state rules with a completeness check — `count_over_time(gpu:sm_active:ratio[20m]) >= 38` for a 20-minute window at 30 s (allowing two missed scrapes out of 40). Third, under time-slicing the device metric is fanned out to every holder (lesson 01), so the *same* quiet device produces N "idle" pod-series. Deduplicate on the measurement identity before summing GPU-hours, or your idle number is inflated by exactly the holder count.

### 7. Pricing the mistake: the expected-value model, solved

The decision is not "is this GPU idle" but "does acting beat not acting." Write it as an expectation and then solve it, because the solved form is the thing worth remembering.

Let `p` be the probability the classifier is wrong (the holder was productive). Reclaiming yields:

```
  E[gain]  = (1 − p) × H_recovered × r
  E[loss]  =      p  × ( C_restart + C_capacity + C_lostwork )

  ACT IFF   (1 − p) · H_recovered · r  >  p · (C_restart + C_capacity + C_lostwork)

  Solve for the break-even error rate:

              H_recovered · r
  p*  =  ─────────────────────────────────────────
          H_recovered · r + C_restart + C_capacity + C_lostwork

  ACT IFF p < p*.   p* is the classifier accuracy your policy REQUIRES.
```

Now put mechanism behind each cost term instead of leaving them as symbols.

**`C_restart` — rebuilding what was in HBM.** This one is almost always smaller than people expect, and saying so out loud is what makes the rest of the argument credible.

```
  T_restart = T_weights + T_init + T_warm + T_ready

  T_weights = model_bytes / load_bandwidth
      70B params @ FP8  = 70 GB
        · local NVMe   at ~3.0 GB/s  →  23.3 s
        · object store at ~0.8 GB/s  →  87.5 s
  T_init    = CUDA context + engine build + graph capture / compile
              → 60–180 s for a large LLM engine; call it 120 s
  T_warm    = first-token / cache priming                    → 10 s
  T_ready   = readiness probe + load-balancer re-entry        → 15 s

  T_restart(local NVMe)   ≈  23 + 120 + 10 + 15  =  168 s
  T_restart(object store) ≈  88 + 120 + 10 + 15  =  233 s

  C_restart = T_restart × GPUs_held × r
            = (168/3600) × 1 × $2.99  =  $0.14        ← per replica

  SUBSTITUTE YOUR OWN: model_bytes, load_bandwidth and the compile time
  are all measurable in one experiment. Every figure above is a stated
  assumption, not a constant.
```

**$0.14. That is the whole direct GPU cost of a wrongful eviction of a serving replica.** If `C_restart` were the only loss term, you would reclaim everything all the time. The reason not to lives in the next term.

**`C_capacity` — what removing a replica does to latency.** A serving tier sized for peak runs at some utilisation `ρ` of its capacity. Take one of `N` replicas out for `T_restart` and the survivors' utilisation rises to `ρ' = ρ · N/(N−1)`. Queueing delay does not rise proportionally; it rises with `1/(1−ρ)`. For any queue whose waiting time scales as `ρ/(1−ρ)` (the M/M/1 form, and the right qualitative shape for a request queue in front of a batching engine), the multiplier is:

```
  wait multiplier  =  [ρ'/(1−ρ')] ÷ [ρ/(1−ρ)]

  N = 8 replicas, one removed ⇒ ρ' = ρ × 8/7 = 1.143 ρ

    ρ = 0.50 → ρ' = 0.571 → wait ×  1.50
    ρ = 0.70 → ρ' = 0.800 → wait ×  1.71
    ρ = 0.80 → ρ' = 0.914 → wait ×  2.66
    ρ = 0.85 → ρ' = 0.971 → wait ×  5.95
    ρ = 0.88 → ρ' = 1.006 → OVERLOADED — queue grows without bound

  ⇒ The SAME action is nearly free at ρ = 0.5 and catastrophic at ρ = 0.85.
    ANY idle-reclaim policy for serving that does not read current
    utilisation is a policy that behaves differently at 3am and at peak
    WITHOUT KNOWING IT.
```

Convert that to money through the error budget rather than inventing a dollar figure:

```
  SLO: 99.9% of requests under the TTFT target, 30-day window.
  Traffic: 1,200 req/s sustained.

  monthly requests   = 1,200 × 2,592,000 s        = 3.11e9
  monthly budget     = 0.1% of that               = 3.11e6 bad requests

  One false-positive reclaim at ρ = 0.80:
    degraded window  = T_restart = 168 s ≈ 2.8 min
    share of requests breaching during it (wait ×2.66) ≈ 30 %
    breached         = 168 s × 1,200 /s × 0.30       = 60,480 requests

    share of the MONTHLY budget consumed by ONE mistake
                     = 60,480 / 3.11e6              =  1.9 %

  And what the mistake was chasing:
    idle "recovered" = 1 GPU × 20 min dwell         = 0.333 GPU-h
                     = 0.333 × $2.99                =  $1.00

  ⇒ $1.00 of savings for 1.9 % of a month's error budget.
    Fifty-three such reclaims exhaust the budget and return $53.
```

**`C_lostwork` — recomputation, for checkpointing workloads only.** For a training job with checkpoint interval `I`, a uniformly-timed preemption loses on average `I/2` of progress across every GPU in the gang:

```
  C_lostwork = (I/2) × GPUs_in_gang × r
             = (0.5 h) × 8 × $2.99          = $11.96      (I = 1 h)
             = (0.25 h) × 8 × $2.99         = $5.98       (I = 30 min)
  plus requeue: T_queue × GPUs × r, which on a busy cluster can exceed
  the lost work — measure your own queue-wait quantiles (module 06).
```

Now solve `p*` for the two archetypes and watch them diverge by two orders of magnitude.

```
  ── BATCH / TRAINING ──────────────────────────────────────────────────
  Suspect: an 8-GPU job, quiet for 45 min, holding a 6 h reservation.
    H_recovered = 6 h × 8 GPUs                    = 48 GPU-h
    gain if right = 48 × $2.99                    = $143.52
    C_restart   = 10 min × 8 × $2.99              = $ 3.99
    C_lostwork  = 30 min × 8 × $2.99 (I = 1 h)    = $11.96
    C_capacity  = 0  (batch has no latency SLO)   = $ 0
    ───────────────────────────────────────────────────────
    p* = 143.52 / (143.52 + 15.95)                = 0.900

  ⇒ THE CLASSIFIER MAY BE WRONG 90 % OF THE TIME AND RECLAIM STILL PAYS.
    For batch, aggressive is correct. Checkpointing is what buys this:
    halve I and p* rises further.

  ── LATENCY SERVING ───────────────────────────────────────────────────
  Suspect: 1 replica of 8, quiet for a 20 min dwell, ρ = 0.80.
    H_recovered = 0.333 GPU-h → gain if right     = $  1.00
    C_restart                                     = $  0.14
    C_capacity  = 1.9 % of a monthly error budget.
                  Price the budget at B dollars/month (your credits,
                  your revenue-at-risk, your on-call cost — pick one and
                  write it down). At B = $25,000:  0.019 × 25,000
                                                  = $475.00
    ───────────────────────────────────────────────────────
    p* = 1.00 / (1.00 + 0.14 + 475.00)            = 0.0021

  ⇒ THE CLASSIFIER MUST BE WRONG LESS THAN ~1 TIME IN 470.
    No utilisation threshold achieves that on S4, because S4's signal is
    IDENTICAL for productive and abandoned replicas (§1).
    Therefore: DO NOT RECLAIM SERVING ON ACTIVITY SIGNALS. Require S5
    (application attestation), or use a non-destructive remediation (§8).

  ── THE RATIO IS THE LESSON ───────────────────────────────────────────
    p*(batch) / p*(serving)  ≈  0.900 / 0.0021  ≈  430×
    Same fleet, same metric, same threshold, same dwell —
    and a required accuracy that differs by more than two orders
    of magnitude. THAT is why the class label is load-bearing and a
    single fleet-wide idle rule cannot be correct.
```

Two honest riders. First, `p` is not knowable exactly, but it is *estimable*: shadow-run the classifier for a fortnight, label the candidates by hand or by asking the owner, and you get an empirical `p` per class. Publishing "our idle classifier's measured false-positive rate on batch is 12%" is what turns this from an argument into a policy. Second, the direction of the inequality is robust to large errors in `B`: at `B = $2,500` instead of $25,000, `p*(serving)` is still only 0.021 — 1 in 48. The conclusion does not depend on the precision of the number.

### 8. Remediation: the ladder, and the option that is not eviction

Match the action to the state. Reclaim-by-eviction is one row of six, and for the most expensive state it is the wrong row.

| State | Action | Mechanism | Risk | Typical latency to recover |
|---|---|---|---|---|
| **S2 micro-idle** | none — throughput work | larger batches, continuous batching, chunked prefill, prefill/decode disaggregation, fixing the dataloader | none | weeks (engineering) |
| **S3 cold hold** | evict | descheduler `PodLifeTime` on `states: [Succeeded]` / crash-loop conditions; a controller that deletes pods holding a device with no context past a TTL | very low — nothing warm exists | minutes |
| **S4 warm hold, batch** | checkpoint-aware preemption | priority/preemption (module 06); Kueue/Volcano/KAI reclaim; require a checkpoint hook | bounded by `I/2` lost work | minutes |
| **S4 warm hold, dev** | cull | JupyterHub `jupyterhub-idle-culler --timeout`; Kubeflow `ENABLE_CULLING=true` with a **much** lower `CULL_IDLE_TIME` than the 1440-minute default, joined to `SM_ACTIVE` and `FB_USED` | low, but tell users first and give a grace warning | minutes |
| **S4 warm hold, serving** | **do not evict** — offload or scale | engine sleep mode; scale-to-zero on request signals; consolidate replicas | see §7 | seconds to low minutes |
| **S5 attested idle** | any of the above, safely | the app has said it has no work | lowest of the warm states | minutes |
| **Gap A (unallocated)** | consolidate / scale down | descheduler `LowNodeUtilization`, bin-packing, cluster autoscaler | scheduling churn | lesson 04 |

**The option most fleets have not adopted: sleep instead of kill.** Modern inference engines can release the expensive resource without losing the cheap one. vLLM exposes this as a first-class, *observable* engine state — the current metrics code defines:

```
  vllm:engine_sleep_state{sleep_state="awake"|"weights_offloaded"|"discard_all"}
    "Engine sleep state; awake = 0 means engine is sleeping;
     awake = 1 means engine is awake;
     weights_offloaded = 1 means sleep level 1;
     discard_all = 1 means sleep level 2."
```

Two levels, two different trade-offs: level 1 offloads weights to host memory (fast to wake, host RAM is consumed), level 2 discards them entirely (frees the most, slowest to wake). Either way the *pod keeps existing* — its identity, its service endpoint, its warm process — while the HBM it was sitting on becomes available. That converts an S4 GPU from a binary "evict or pay" into a graded decision, and it changes the expected-value arithmetic completely: `C_capacity` collapses because wake-up is far cheaper than a full cold start, and there is no scheduling round trip.

The other three levers worth naming precisely:

- **Pack, don't reclaim.** Several replicas each at 5% occupancy are not four idle GPUs; they are one GPU's worth of work spread across four. MPS (with `CUDA_MPS_ACTIVE_THREAD_PERCENTAGE` bounding each client's SM share, and therefore bounding the attribution error — lesson 01) or time-slicing consolidates them with no eviction risk at all. The cost is attribution fidelity: you have traded an exactly-attributable device for an estimated one, and lesson 01's exposure fraction `E` is where that shows up.
- **Descheduler, driven by real utilisation.** `LowNodeUtilization` takes `thresholds`/`targetThresholds` as percentages and — importantly for us — **supports extended resources such as `nvidia.com/gpu`** if you list them explicitly, and can take actual utilisation from Prometheus via `metricsUtilization.source: Prometheus` with a `query` returning one value per node in `[0, 1]`. That is exactly the shape of an `SM_ACTIVE`-derived node score, which means the descheduler can be driven by the honest metric rather than by requests. Two limits to know: on the metrics path **at most one pod is evicted per over-utilised node per run**, and `evictionLimits.node` caps the blast radius further. Both are features, not bugs — use them as guard G6.
- **Anti-thrash floors.** NVIDIA's KAI Scheduler ships a *min-guaranteed-runtime* concept — a period during which the scheduler must not preempt or reclaim a running workload even if it is preemptible — alongside workload consolidation. That is guard G5 implemented by a scheduler instead of by you. Whatever the mechanism, some floor is mandatory: without it, a job can be evicted seconds after it starts, having paid its full startup cost and produced nothing, and the fleet does more restarting than computing.

### 9. What the tools call "idle" — and why it is a different number

You will be asked "why doesn't this match OpenCost?" The answer is in the source, and it is not a discrepancy — it is a different definition.

OpenCost computes idle in `computeIdleAllocations` (`pkg/costmodel/costmodel.go`) as the difference between the **asset** totals and the **allocation** totals, per key, with the key being the node or the cluster depending on `idleByNode`:

```go
cpuIdleCost := assetTotal.TotalCPUCost() - allocTotal.TotalCPUCost()
gpuIdleCost := assetTotal.TotalGPUCost() - allocTotal.TotalGPUCost()
ramIdleCost := assetTotal.TotalRAMCost() - allocTotal.TotalRAMCost()
// ...negative values are logged and clamped to zero
if gpuIdleCost < 0 { /* warn */ gpuIdleCost = 0 }
```

and inserts one synthetic allocation named `<key>/__idle__`. Map that onto lesson 02's areas: `assetTotal` is area ①, `allocTotal` is area ②, so **OpenCost's GPU idle is exactly gap A — paid-for-but-unallocated capacity.** It contains none of gap B. It cannot, because the allocated cost it subtracts is `GPUHours × CostPerGPUHr` where `GPUHours = <GPU request> × hours` — a request-based quantity that never consults utilisation. The same code path *does* populate `GPUUsageAverage` (from a DCGM engine-activity query) and `IsGPUShared`, and uses neither in the cost.

Two further details are worth carrying into a design conversation. **The clamp is a diagnostic**: the negative case is logged (`NEGATIVE_IDLE` in the debug path) with its causes enumerated in the comments — missing pricing data, misconfigured custom pricing, billing adjustments, or allocation exceeding asset cost through timing and rounding. Seeing it means your rate card and your allocation window disagree; chase it rather than suppress it. And **sharing idle back onto tenants is cost-weighted**: with `ShareIdle = "__weighted__"`, `computeIdleCoeffs` (in `core/pkg/opencost/allocation.go`) builds each allocation's coefficient as its `GPUTotalCost()` over the per-idle-key total and distributes idle in that proportion — precisely the loading-multiplier mechanism lesson 05 uses for fully-loaded unit costs, done by cost share rather than by usage.

**So the correct sentence in a review is:** *"OpenCost's idle number is our gap A, computed per node and clamped at zero; our gap B — the allocated-but-quiet hours — has no OpenCost equivalent, because its GPU cost is request × hours and never multiplies by utilisation."* Checkable, specific, and the same finding lesson 09 develops at length.

### 10. Guardrails: shipping this without causing an incident

Six constraints, each named with the failure it prevents. **Dry-run for two weeks** before acting on anything — it is also how you obtain the measured `p` that §7 requires, rather than discovering it through outages. **Alert at `severity: info`, never page** — a cost signal competing with reliability signals for on-call attention ends up muted permanently. **Gate on class in the *action*, not only the detection**, so a well-meaning operator cannot run the reclaim script over the full candidate list. **Cap the blast radius**: `evictionLimits`, a max-evictions-per-hour, and a circuit breaker that halts everything when candidate volume or cluster error rate spikes, so a correlated misclassification cannot become a fleet-wide eviction. **Fail closed on missing signals** — absent app metrics, absent DCGM samples and failed completeness checks all mean *not idle*, because the most dangerous correlated failure available to you is a monitoring outage read as fleet-wide idleness. And **give a grace notice for anything a human owns**, with a one-click extension: the political failure ends the programme even when the arithmetic was right.

## Perspectives

**Platform / on-call.** To the cost dashboard a false positive is a rounding error; to the pager it is an incident with a postmortem that names your automation as the cause. The single design rule that keeps these two from fighting is that **cost automation must never take a destructive action that reliability automation would not sanction** — which in practice means the reclaimer's class gating, min-runtime and eviction budget are owned jointly with whoever owns the error budget. If they cannot describe your reclaim policy in one sentence, it is not deployable.

**Workload owner.** From inside a namespace, `Running` means "working." There is no native signal telling a tenant that their pod held a device for eleven hours and computed for forty minutes. That asymmetry means an idle finding always arrives as news, and often as an accusation. Pair every idle number with the state name, the mechanism, and one concrete first action ("your job holds the GPU for 40 minutes after the last step — add a completion hook") and it reads as help; publish the dollar figure alone and it reads as blame and the dashboard stops being opened.

**Hardware / signal.** Every candidate signal is individually blind: `SM_ACTIVE` counts a single trivial warp as activity; `PIPE_TENSOR_ACTIVE` is legitimately zero for healthy memory-bound decode; power is sampled part-time on exactly the SKUs you care about; framebuffer occupancy says storage, not work. The composite rule ANDs several signals **because each closes a different blind spot**, and it treats power asymmetrically — as a veto on "idle", never as evidence for it — because the measurement's error structure is asymmetric.

**Economics.** The states have different dollar fates, and that is the whole reason for the taxonomy. S3 converts to savings at near-zero risk. S5 converts at low risk. S4-batch converts with a bounded, computable risk (`I/2` of recomputation). S4-serving does **not** convert under an eviction lever at any sensible price; it converts under a different lever entirely (sleep, scale-to-zero, packing) with a different risk profile. And S2 does not convert at all under any reclaim lever — it converts only through throughput engineering. A report that sums all five and calls it "recoverable" is not merely imprecise; it directs money and attention at the bucket where the lever does not exist.

**Auditor.** The allocated ledger is a scheduler fact and needs no defence. An idle *classification* is a model, and models get challenged. So carry the model's parameters with the number: the depth threshold, the dwell, the class, the guard set, the measured completeness of the underlying series, and — the one that ends most challenges — the shadow-mode false-positive rate. "3,140 idle GPU-hours" invites "is that right?". "3,140 idle GPU-hours in state S3 at θ=0.02/t_d=20m, 99.4% sample completeness, measured FPR 4%" invites a conversation about the 4%.

## Real-world use cases

- **NVIDIA `dcgm-exporter` — `etc/default-counters.csv` (fetched directly from the repository).** **What it shows:** the shipped counter set enables `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_PROF_GR_ENGINE_ACTIVE`, `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`, `DCGM_FI_PROF_DRAM_ACTIVE`, power, framebuffer and clocks — and leaves `DCGM_FI_PROF_SM_ACTIVE` and `DCGM_FI_PROF_SM_OCCUPANCY` commented out. **Why it matters here:** the default install gives you the duty-cycle field that cannot detect idle and withholds the occupancy field that can, so "we already monitor GPU utilisation" is usually a statement about the wrong number. The `--collect-interval` default of 30000 ms in `pkg/cmd/app.go` is the `Δ` in every integral in §6.

- **Kubeflow notebook controller — `culling_controller.go` (fetched directly).** **What it shows:** culling is driven by polling the notebook server's `/api/kernels`; any kernel not in `execution_state: idle` refreshes the last-activity annotation, otherwise the newest kernel `last_activity` is used, and the Notebook is scaled to zero once `CULL_IDLE_TIME` elapses. Defaults from the source: `DEFAULT_ENABLE_CULLING = "false"`, `DEFAULT_CULL_IDLE_TIME = "1440"` minutes, `DEFAULT_IDLENESS_CHECK_PERIOD = "1"` minute. **Why it matters here:** the most widely deployed GPU-notebook idle mechanism in existence is off by default, has a one-day timeout when on, and never reads a GPU metric — it misclassifies a CPU-busy kernel as GPU-active and a thinking user as idle. Joining it to `SM_ACTIVE` and `FB_USED` is the cheapest improvement in this lesson.

- **OpenCost — `computeIdleAllocations` in `pkg/costmodel/costmodel.go` and `computeIdleCoeffs` in `core/pkg/opencost/allocation.go` (both fetched directly).** **What it shows:** idle is `asset total − allocation total` per node or cluster, clamped at zero with a `NEGATIVE_IDLE` diagnostic path; idle is redistributed to tenants in proportion to each allocation's `GPUTotalCost()` when `ShareIdle` is weighted. **Why it matters here:** the leading OSS tool's "idle" is lesson 02's gap A, not the allocated-but-quiet gap B this lesson detects. The two numbers are both correct and are not comparable, and knowing which is which from source settles the argument in one sentence.

- **vLLM — `vllm/v1/metrics/loggers.py` (fetched directly).** **What it shows:** production engines expose exactly the attestation signals state S5 needs — `vllm:num_requests_running`, `vllm:num_requests_waiting` (with a `num_requests_waiting_by_reason` breakdown for `capacity` versus `deferred`), `vllm:kv_cache_usage_perc`, `vllm:num_preemptions` — **and** a first-class sleep state, `vllm:engine_sleep_state` with `weights_offloaded` (level 1) and `discard_all` (level 2). **Why it matters here:** it removes both excuses. The application *can* tell you it has no work, and there *is* a non-destructive remediation between "hold a warm GPU" and "kill the pod."

- **Kubernetes descheduler and NVIDIA KAI Scheduler (both READMEs fetched directly).** **What they show:** the descheduler's `LowNodeUtilization` accepts extended resources such as `nvidia.com/gpu` in `thresholds`/`targetThresholds` (ignored unless explicitly listed), can source real utilisation from Prometheus via `metricsUtilization.source: Prometheus` with a per-node query returning values in `[0, 1]`, evicts at most one pod per over-utilised node on that path, and caps blast radius with `evictionLimits.node`; `PodLifeTime` evicts on age, pod states, conditions and exit codes — the natural mechanism for state S3. KAI's feature list names bin-packing versus spread, workload consolidation, and **min-guaranteed-runtime**, a window in which a workload must not be preempted or reclaimed even if preemptible. **Why it matters here:** the reclaim machinery already exists, already accepts an honest GPU signal, and already implements guard G5 in a scheduler. What has been missing is a defensible definition of "idle" to feed it.

- **"Part-time Power Measurements: nvidia-smi's Lack of Attention" (arXiv 2312.02741).** **What it shows:** on A100 and H100 only about 25% of runtime is sampled for power, so readings can miss substantial excursions; the authors profile the sensor's behaviour across 70+ GPUs spanning every architecture since Fermi and report that their recommended practices cut energy-measurement error by up to 35%, with an estimated extra $1M of annual electricity cost attributable to measurement error at a 10,000-GPU scale. **Why it matters here:** it is the reason power appears in this lesson's rule as a one-directional veto rather than as a discriminator.

## Worked example

**Setup.** One fleet, one week, all figures reproducible from the query pack in §6.

```
  FLEET      64 × H100 80GB, exclusively allocated (regime 1 — so every
             utilisation figure here is EXACT; no lesson-01 error band).
  WINDOW     7 days = 168 h
  RATE       r = $2.99 / physical-GPU-hour
             BASIS: FOCUS EffectiveCost ÷ physical GPU-hours.
             SNAPSHOT: 2026-08. Observed on-demand H100 SXM band across
             provider classes at that date: roughly $2–$7. EVERYTHING
             BELOW IS LINEAR IN r — substitute yours.
  SAMPLING   Δ = 30 s ⇒ 20,160 samples per series per week.
             Completeness measured at 99.4% (≥98% floor: pass).
  CLASSIFIER θ = 0.02 on max_over_time(SM_ACTIVE), t_d = 20 min for
             serving/batch, 4 h for dev; guards G1–G7 as in §4.
```

### Step 1 — the envelope, from lesson 02's model

```
  PAID (area ①)      = 64 × 168               = 10,752 GPU-h
                     × $2.99                  = $32,148.48

  ALLOCATED (area ②) = 52 GPUs mean × 168 h   =  8,736 GPU-h  → $26,120.64
  GAP A (① − ②)      = 12 GPUs × 168 h        =  2,016 GPU-h  →  $6,027.84
                       (unallocated, cordoned, shape-stranded — LESSON 04)

  UTILISED (area ③)  = Σ sum_over_time(sm_active) × 30/3600
                                              =  3,669 GPU-h  → $10,970.31
  ā  = 3,669 / 8,736                          = 0.420
  GAP B (② − ③)      = 8,736 − 3,669          =  5,067 GPU-h  → $15,150.33
```

At this point most reports stop and print **"$15,150 of idle GPU time this week."** That sentence is the mistake this lesson exists to prevent. Keep going.

### Step 2 — split gap B by state

Run the state rules; sum with the correct integral; subtract to get S2.

```
  STATE            GPU-h/wk    $ / wk       share of gap B
  ─────────────────────────────────────────────────────────────
  S2 micro-idle      2,400    $ 7,176.00       47.4 %
     (below the 20-minute dwell — inter-request and inter-step gaps)
  S3 cold hold         700    $ 2,093.00       13.8 %
     (allocated, quiet, FB_USED at floor — no CUDA context)
  S4 warm hold       1,967    $ 5,881.33       38.8 %
     ├─ class=serving 1,120   $ 3,348.80
     ├─ class=batch     520   $ 1,554.80
     └─ class=dev       327   $   977.73
  ─────────────────────────────────────────────────────────────
  TOTAL gap B        5,067    $15,150.33      100.0 %

  of which S5 (attested idle: warm hold AND the app reports zero
  running and zero waiting requests for the whole dwell):
     class=serving       210   $   627.90   ← a SUBSET of the 1,120
```

**Read the first row before anything else.** Nearly half the "idle" money is state S2 — quiescence shorter than the dwell, distributed across the ordinary operation of healthy workloads. There is no eviction, TTL, descheduler or preemption that recovers a cent of it. Reporting it as recoverable inflates the promise by roughly a factor of two, and the promise is the thing you will be held to.

### Step 3 — apply the decision rule per state

Use §7's `p*` with this fleet's numbers.

```
  S3 COLD HOLD — 700 GPU-h, $2,093
    C_restart = 0 (nothing warm), C_lostwork = 0, C_capacity = 0
    ⇒ p* → 1.0.  Act unconditionally.
    Mechanism: PodLifeTime on Succeeded/crash-looping pods + a TTL
    controller for devices with no context.
    RECOVERABLE: $2,093 / wk, essentially in full.

  S4 BATCH — 520 GPU-h, $1,555
    Suspects are 8-GPU gangs holding 6 h reservations, I = 1 h.
    From §7: p* = 0.900.  Measured shadow-mode FPR on this class = 0.12.
    0.12 < 0.900 ⇒ ACT, with a wide margin.
    Expected value on the whole bucket:
       gain  = (1 − 0.12) × $1,554.80          = $1,368.22
       loss  = 0.12 × (C_restart + C_lostwork) per candidate;
               15 candidates × $15.95 × 0.12   = $   28.71
       NET                                      ≈ $1,339 / wk
    RECOVERABLE: ~$1,339 / wk.

  S4 DEV — 327 GPU-h, $978
    4 h dwell, notebooks, grace notice sent, one-click extension.
    C_restart ≈ a container restart; C_capacity = 0; C_lostwork = the
    user's unsaved in-memory state — REAL, and the reason for the
    notice rather than a silent kill.
    Also fix the config: CULL_IDLE_TIME 1440 → 120 minutes, gated on
    SM_ACTIVE and FB_USED so a CPU-busy kernel no longer counts as
    GPU activity.
    RECOVERABLE: ~$900 / wk, and rising as the shorter TTL bites.

  S4 SERVING — 1,120 GPU-h, $3,349
    From §7: p* = 0.0021 at ρ = 0.80 with the error budget priced at
    B = $25,000/month. No activity-based classifier reaches that.
    ⇒ EVICTION IS OFF THE TABLE for the 910 GPU-h that are warm-hold-
      only. Two other levers apply:
      · S5 SUBSET (210 GPU-h, $628): the app itself attests zero
        running and zero waiting requests for the full dwell. Put the
        engine to SLEEP LEVEL 1 rather than evicting: HBM is released,
        the pod and its endpoint survive, wake-up is a fraction of
        T_restart. Recover most of the $628 at a fraction of the risk.
      · THE REST (910 GPU-h, $2,721): this is a SIZING finding, not a
        waste finding. Route it to the autoscaler (scale on
        num_requests_waiting / queue depth, not on SM_ACTIVE) and to
        MPS packing for the small replicas. Expect a partial recovery
        over weeks, not a reclaim this week.
    RECOVERABLE NOW: ~$550 / wk (the S5 subset, conservatively).

  S2 MICRO-IDLE — 2,400 GPU-h, $7,176
    RECOVERABLE BY RECLAIM: $0. By throughput engineering: unknown
    until measured, and owned by the tenants' ML engineers.
```

### Step 4 — the report a platform lead can actually send

```
  GPU IDLE, WEEK 34.  Fleet 64 × H100.  Rate $2.99/GPU-h
  (EffectiveCost basis, 2026-08 snapshot). Sample completeness 99.4 %.
  Classifier: max SM_ACTIVE < 0.02 over 20 min (4 h for dev);
  measured false-positive rate 12 % on batch, shadow-mode, 2 weeks.

    paid                 10,752 GPU-h   $32,148   100 %
    ├─ utilised           3,669 GPU-h   $10,970    34 %
    ├─ gap A unallocated  2,016 GPU-h   $ 6,028    19 %  → platform (L04)
    └─ gap B              5,067 GPU-h   $15,150    47 %
        ├─ S2 micro-idle  2,400 GPU-h   $ 7,176    22 %  → NOT reclaimable
        ├─ S3 cold hold     700 GPU-h   $ 2,093     7 %  → evict now
        ├─ S4 batch         520 GPU-h   $ 1,555     5 %  → preempt
        ├─ S4 dev           327 GPU-h   $   978     3 %  → cull (TTL 2 h)
        └─ S4 serving     1,120 GPU-h   $ 3,349    10 %  → DO NOT EVICT
            └─ of which S5  210 GPU-h   $   628          → sleep level 1

  ACTION THIS WEEK      ~$4,882 / wk  ≈ $254k / yr
    (S3 $2,093 + S4-batch $1,339 + S4-dev $900 + S5 sleep $550)
  NOT WASTE, DO NOT PROMISE      $7,176 (S2) + $2,721 (serving sizing)
  PLATFORM'S OWN NUMBER, SAME PAGE                     $6,028 (gap A)

  One line for the exec: of $15,150/wk of allocated-but-quiet GPU time,
  $4,882 is safely recoverable now; $7,176 is inherent to how the
  workloads run and needs throughput work, not reclaim; and the
  platform's own unallocated capacity is $6,028, which is on us.
```

### Step 5 — sensitivity, so the argument survives a challenge to an input

Everything is linear in `r` and in the state split, so publish the derivatives rather than a single number:

```
  ACTIONABLE $/wk as a function of the two contested inputs:

              t_d = 10 min   t_d = 20 min   t_d = 45 min   t_d = 2 h
   r = $2.00     $4,180        $3,266         $2,510        $1,640
   r = $2.99     $6,249        $4,882         $3,752        $2,451
   r = $5.00     $10,450       $8,164         $6,275        $4,098

  READ IT AS: shortening the dwell moves S2 into S4 and inflates the
  actionable number WITHOUT ANY NEW SAVINGS BEING AVAILABLE — the extra
  candidates are healthy workloads between steps. The dwell is where an
  idle programme cheats, so state it, justify it from the q99 sweep in
  §3, and show this table alongside.
```

Do not promise the ceiling. A fleet cannot reach `ā = 1`; a serving tier sized for peak cannot run at peak; and the S2 term is a property of how the workloads compute, not of how the platform is managed.

## Practice

Feeds the module deliverable at [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md).

1. **Measure your own floors and your own gap distribution.** Capture `power.draw`, `clocks.sm` and `memory.used` on an idle device and on a loaded one; then run the §3 dwell sweep over 5m/10m/20m/30m/45m/1h/2h/4h for at least one week, per workload class.
   **Acceptance:** a per-class table of `q99(quiescent gap)` with the chosen `t_d = k × q99` and the `k` you used, plus the measured `FB_USED` floor written into the rule as a named constant.

2. **Ship the state rules, not a threshold.** Implement `gpu:state_cold_hold:bool`, `gpu:state_warm_hold:bool` and `gpu:state_attested_idle:bool` with the completeness guard, and emit GPU-hours per state with `sum_over_time(...) × Δ/3600`.
   **Acceptance:** `S2 + S3 + S4 = gap B` reconciles to within 1% for every namespace, and S2 is computed by subtraction rather than measured.

3. **Run it in shadow mode for two weeks and measure `p`.** Log candidates, confirm each by hand or by asking the owner, and publish the false-positive rate per class.
   **Acceptance:** a measured FPR per class, and the `p*` from §7 computed with your own `r`, `I`, `N`, `ρ` and error-budget price `B`, showing which classes clear their bar.

4. **Price one false positive end to end.** Pick one serving replica: measure `T_weights` (model bytes ÷ observed load bandwidth), `T_init`, `T_warm`, `T_ready`; compute `C_restart`; compute the wait multiplier at your current `ρ` for `N−1` replicas; convert the degraded window into a share of your monthly error budget.
   **Acceptance:** a single paragraph of the form "reclaiming this replica saves $X and spends Y% of a monthly error budget", with every input named.

5. **Wire one non-destructive remediation.** Either engine sleep level 1 driven by the S5 rule, or scale-to-zero on `num_requests_waiting`, or MPS packing for a set of low-occupancy replicas.
   **Acceptance:** a before/after on GPU-hours held for the same served traffic, plus the wake-up latency you measured.

6. **Add the deliverable section.** "Idle GPU-hours and dollars this week, by state" with the recording rules, the split table, one sentence per state naming owner *and* mechanism, the classifier parameters, and the sensitivity table from step 5 of the worked example.
   **Acceptance:** a reader can tell which dollars are recoverable this week, which need engineering, and which are the platform's own.

## Common pitfalls

1. **Reporting gap B as recoverable idle.** *Mechanism:* gap B integrates every quiescent moment including the sub-dwell gaps inherent to batched serving and step loops (state S2), which no reclaim lever touches. *Symptom:* a savings promise roughly double what any action can deliver, followed by a credibility problem when the fleet does not get cheaper. *Fix:* compute S2 by subtraction and label it "not reclaimable" in the same table.

2. **Using `DCGM_FI_DEV_GPU_UTIL` as the idle signal.** *Mechanism:* it is a duty-cycle sample — high if *any* kernel was resident, regardless of how much of the chip that kernel used — so a decode-bound server at 4% occupancy and a spin-waiting zombie both read as busy. It ships enabled while `SM_ACTIVE` ships commented out, so this is the default outcome rather than a mistake someone made. *Symptom:* an idle rule that finds almost nothing on a fleet you know is half empty.

3. **Thresholding on `PIPE_TENSOR_ACTIVE`.** *Mechanism:* tensor-pipe activity is legitimately near zero for memory-bound decode, fp32 work, embeddings and data movement. *Symptom:* the rule fires hardest on your healthy inference fleet, which is the exact population where a false positive is most expensive. Use it as a gate against "busy but trivial", never as the idle signal.

4. **One dwell time for the whole fleet.** *Mechanism:* the archetypes' quiescent gaps span six orders of magnitude; a dwell tuned for notebooks fires during every training checkpoint, and one tuned for training never catches a notebook. *Fix:* derive `t_d` per class from the measured `q99` and carry the class label into the *action*, not just the detection.

5. **`max_over_time` as the only test, against a twitching zombie.** *Mechanism:* a crashed process whose watchdog launches one trivial kernel a minute keeps the max above any threshold forever, so the strictest test is also the one most easily defeated. *Fix:* offer `quantile_over_time(0.95, ...)` as the tolerant variant and pick per class, knowing that the quantile buys detection at the price of a slightly higher false-positive rate.

6. **Reading a monitoring outage as fleet-wide idleness.** *Mechanism:* absent samples are not quiet samples, but `max_over_time` over an empty window returns nothing and naive rules treat "no data" as "no activity". A DCGM or Prometheus incident then marks every GPU idle simultaneously. *Fix:* a `count_over_time` completeness guard in every state rule and a circuit breaker that halts all action when candidate volume spikes — fail closed, always.

7. **Treating reclaim as free because the restart is cheap.** *Mechanism:* the direct GPU cost of a restart genuinely is tiny (§7 computes $0.14 for a 70B replica), which tempts people to conclude the action is low-risk. The real cost is the *capacity* removed during the window, and queueing delay rises as `1/(1−ρ)` — the same eviction is nearly free at `ρ = 0.5` and unbounded at `ρ = 0.88`. *Fix:* make current utilisation an input to the serving policy, or exclude serving from eviction entirely.

8. **Publishing an idle number without publishing the classifier.** *Mechanism:* an idle figure is a model output, not a measurement, and an unstated model invites "is that even right?" — to which the only answer is a long one. *Fix:* publish θ, `t_d`, the guard set, sample completeness and the measured false-positive rate alongside every number.

9. **Letting notebook cullers stand in for GPU idle detection.** *Mechanism:* both major cullers key on kernel/HTTP activity, not GPU activity — a CPU-busy kernel keeps a 40 GB model resident indefinitely, and a thinking user is culled while a background CUDA stream still runs. Kubeflow's culling is additionally **off by default** with a **1,440-minute** timeout when enabled. *Fix:* keep the culler but AND it with `SM_ACTIVE` and `FB_USED`, and shorten the timeout deliberately.

10. **Summing fanned-out device metrics under time-slicing, or evicting with no anti-thrash floor.** *Mechanism:* `SM_ACTIVE` is device-scoped and the exporter labels it with every holding pod (lesson 01), so one quiet device produces N "idle" pod-series and idle GPU-hours inflate by exactly the holder count — the diagnostic symptom is idle hours exceeding allocated hours, which is impossible. Separately, without a min-runtime a workload can be evicted moments after starting, having paid its full startup cost and produced nothing; if the dwell is shorter than the startup, that becomes a loop. *Fix:* deduplicate on the measurement identity before summing, and enforce guard G5 plus the `θ`/`θ_r` hysteresis and re-arm cooldown.

## Self-check

- **Why is "gap B" not the same as "reclaimable idle", and roughly how large is the difference?**
  **Answer:** Gap B is the integral of `1 − SM_ACTIVE` over held time, so it counts *every* quiescent moment, including gaps far shorter than any dwell you could act on — inter-request gaps in batched serving, inter-step gaps in a training loop, kernel-launch and collective-wait gaps. Those are state S2: real dollars with zero reclaim surface, addressable only by throughput engineering (batching, chunked prefill, fixing the dataloader). Reclaimable idle is the subset that dwells: S3 (held with no CUDA context), the actionable part of S4 (warm hold with the right class and guards), and S5 (warm hold with the application attesting no work). In the worked example S2 alone was 47% of gap B — nearly half the "idle" money is not recoverable by any reclaim lever, and promising it is how the programme loses credibility. Compute S2 by subtraction and label it explicitly.

- **Name the DCGM fields you would threshold on, why `GPU_UTIL` is the wrong one, and what each of the others is blind to.**
  **Answer:** Primary: `DCGM_FI_PROF_SM_ACTIVE` (1002) — "the ratio of cycles an SM has at least 1 warp assigned", averaged over SMs. Companion: `DCGM_FI_PROF_DRAM_ACTIVE` (1005), because decode-bound serving shows low SM activity with *high* memory-interface activity, and a genuinely idle GPU is low on both. Discriminator: `DCGM_FI_DEV_FB_USED` (252), which separates a cold hold (framebuffer at its floor, no context) from a warm hold (weights resident) — no default-enabled field states "a CUDA context exists", so framebuffer occupancy is the practical stand-in. Veto: `DCGM_FI_DEV_POWER_USAGE` (155), one-directional only. `DCGM_FI_DEV_GPU_UTIL` (203) is wrong because it is a duty-cycle sample: it reads high whenever any kernel was resident in the sample period regardless of how much of the chip that kernel used, so one warp on one SM looks identical to a full-device kernel — and it ships enabled in `dcgm-exporter`'s default counters while `SM_ACTIVE` ships commented out, which is why most fleets are running the wrong rule by default. Blind spots: `SM_ACTIVE` counts warps stalled on memory as active and cannot see usefulness; `PIPE_TENSOR_ACTIVE` is legitimately ~0 for healthy memory-bound work, so it must never be the idle signal; power is sampled roughly 25% of runtime on A100/H100, so low power is only weak evidence; `FB_USED` says storage, not work.

- **State the reclaim decision rule as an inequality, solve it for the break-even error rate, and give the two archetypes' answers.**
  **Answer:** Act iff `(1 − p) · H_recovered · r > p · (C_restart + C_capacity + C_lostwork)`, which solves to `p* = H_recovered·r / (H_recovered·r + C_restart + C_capacity + C_lostwork)`; act iff the classifier's measured error rate `p` is below `p*`. For batch — an 8-GPU gang holding a 6-hour reservation, `H_recovered = 48` GPU-h, `C_restart ≈ $4`, `C_lostwork = I/2 × 8 × r ≈ $12` at a one-hour checkpoint interval, `C_capacity = 0` — `p* ≈ 0.90`: **the classifier may be wrong nine times in ten and reclaim still pays.** For latency serving — one replica of eight, 20-minute dwell, `H_recovered = 0.333` GPU-h worth $1.00, `C_restart = $0.14`, and `C_capacity` equal to the ~1.9% of a monthly error budget consumed by a 168-second degradation at `ρ = 0.80`, priced at $475 if the budget is worth $25,000/month — `p* ≈ 0.002`: **the classifier must be wrong less than about one time in 470.** The ratio is roughly 430×, from the same metric and the same threshold, which is why the class label is load-bearing.

- **What makes a warm serving replica indistinguishable from an abandoned warm job, and what actually resolves it?**
  **Answer:** Nothing in the device's state differs. Both have a CUDA context, both hold weights (and possibly a KV cache) in HBM, both show `SM_ACTIVE` near zero and both may show low power and low tensor activity. The hardware state genuinely is the same, so no threshold, no additional profiling field and no smarter statistic separates them — this is a structural limit of the same family as lesson 01's time-slicing result. What resolves it is information the GPU does not have: (1) the workload class, carried as a label, which decides whether a destructive action is even permitted; and (2) an application attestation — for vLLM, `vllm:num_requests_running == 0 and vllm:num_requests_waiting == 0` sustained across the dwell — which moves the holder from state S4 to S5. The attestation must fail closed: an absent metric means busy, never idle, or a scrape outage becomes a fleet-wide eviction event.

- **How do you choose the dwell time, and what goes wrong at each extreme?**
  **Answer:** Measure, do not choose. For each class, sweep a candidate window `W` over 5m/10m/20m/30m/45m/1h/2h/4h and record the fraction of time `max_over_time(SM_ACTIVE[W])` stayed below the quiet threshold, over at least one full weekly cycle so diurnal troughs and weekends are represented; the knee where the curve collapses is `q99` of that class's quiescent-gap distribution, and set `t_d = k · q99` with `k ≈ 2–3`. Too short and you sweep healthy workloads into the candidate set — training checkpoint barriers, evaluation passes, overnight traffic troughs — which both inflates the reported "actionable" figure without creating any real savings and drives the false-positive rate straight through `p*` for serving. Too long and you pay `r` per GPU-hour for every abandoned holder for longer than necessary, and the loss is linear in the extra dwell: raising `t_d` from 20 to 60 minutes across 15 abandoned candidates costs `15 × 0.667 h × r`. Prefer the long side, because the false-negative cost is linear and bounded while the false-positive cost is discontinuous.

- **Why is engine sleep a better answer than eviction for a quiet serving replica, and what does it cost?**
  **Answer:** Because it separates the two resources that eviction bundles together. Eviction destroys the pod identity, the process, the loaded weights, the compiled graphs and the position in the load balancer, and it removes capacity for the full `T_restart` — the expensive part being `C_capacity`, since queueing delay rises as `1/(1−ρ)` and `ρ` rises to `ρ·N/(N−1)` for the survivors. Sleep releases only the HBM: vLLM exposes `vllm:engine_sleep_state` with level 1 (`weights_offloaded`, weights moved to host memory) and level 2 (`discard_all`), and in both cases the pod, the endpoint and the process survive, so wake-up is a fraction of a cold start and no scheduling round trip is involved. The costs are real but small and different in kind: level 1 consumes host RAM proportional to the model, level 2 must re-read the weights on wake, and either way the replica cannot serve until it wakes — so sleep still needs the S5 attestation and a fast wake path, not just a dwell.

- **Your idle number and OpenCost's do not match. Who is wrong?**
  **Answer:** Neither; they are different quantities. OpenCost's `computeIdleAllocations` builds one synthetic `<key>/__idle__` allocation per node (or per cluster, depending on `idleByNode`) as `assetTotal − allocTotal` per resource, with negative results logged and clamped to zero. In lesson 02's terms that is area ① minus area ② — **gap A**, paid-for-but-unallocated capacity. It contains none of gap B, and it structurally cannot: the allocated cost it subtracts is `GPUHours × CostPerGPUHr` where `GPUHours = <GPU request> × hours`, a request-based figure that never multiplies by utilisation, even though the same code path already collects `GPUUsageAverage` from a DCGM engine-activity query and an `IsGPUShared` flag and uses neither. So the correct reply is that OpenCost is reporting your gap A correctly, your number is gap B split into states, and the two should be published side by side rather than reconciled. If OpenCost is also *sharing* idle back onto tenants, note that `computeIdleCoeffs` distributes it in proportion to each allocation's GPU cost — a cost-weighted loading, which is exactly the mechanism lesson 05 uses for fully-loaded unit costs.

## Connections & what's next

Backward: this lesson consumes lesson 02's gap B wholesale and refuses to treat it as a single number; it consumes module 05's honest signal and correct integral for every GPU-hour figure here; and it inherits lesson 01's regimes, because "idle" on a MIG instance is a per-instance measured fact while "idle" on a time-sliced replica is a device-level fact fanned out to N holders — sum it naively and your idle hours inflate by the holder count.

Forward: lesson 04 takes gap A, the bucket this lesson deliberately hands off, and shows that free capacity and usable capacity are different numbers — the same free GPU count can be worth full price or nothing depending only on its shape. Lesson 05 divides a ledger by an application counter, and the idle states priced here are exactly what separates a *direct* unit cost from a *fully-loaded* one; the loading multiplier is built from these buckets. Lesson 06 needs the idle picture to size commitments, because committing to capacity that will sit in state S4 is the most expensive mistake in procurement. Lesson 08 turns the reclaim policy into a statement a tenant receives, where the "charge allocated, report utilised" rule of lesson 02 finally gets teeth: without a reclaim lever, a tenant can hold allocated-but-quiet capacity indefinitely and no bill ever changes their mind.

Next: **lesson 04** — why a fleet with plenty of free GPUs can still be unable to start a single job.

## References & further reading

**Primary sources — the signals**

1. **NVIDIA DCGM — `dcgmlib/dcgm_fields.h`** — https://github.com/NVIDIA/DCGM — fetched directly. Source of every field ID and semantic in §2: `DCGM_FI_PROF_SM_UTIL_RATIO` 1002 ("the ratio of cycles an SM has at least 1 warp assigned"), `SM_OCCUPANCY_RATIO` 1003, `TENSOR_UTIL_RATIO` 1004, `DRAM_UTIL_RATIO` 1005, `GR_ENGINE_UTIL_RATIO` 1001, `DEV_GPU_UTIL_RATIO` 203, `BOARD_POWER_WATTS` 155, `FB_USED` 252 / `FB_FREE` 251 / `FB_RESERVED` 253, `SM_CLOCK` 100, and `PROCESS_ACCOUNTING_STATS` 205 with its root/accounting-mode requirement. **Correction recorded:** DCGM 4.x renamed the profiling fields to `*_UTIL_RATIO`; the familiar `DCGM_FI_PROF_SM_ACTIVE` spellings are now compatibility aliases defined at the bottom of the same header. (docs.nvidia.com was unreachable from this environment; the header is the authoritative source anyway.)
2. **NVIDIA `dcgm-exporter` — `etc/default-counters.csv` and `pkg/cmd/app.go`** — https://github.com/NVIDIA/dcgm-exporter — fetched directly. Which fields ship enabled versus commented out (§2's table), the `--collect-interval` default of **30000 ms** that fixes `Δ`, and the `--kubernetes-virtual-gpus` / `--kubernetes-enable-dra` flags, both defaulting to `false`.
3. **Prometheus — query functions and recording rules** — https://prometheus.io/docs/prometheus/latest/querying/functions/ and https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/ — `max_over_time`, `quantile_over_time`, `sum_over_time`, `count_over_time`, the `< bool` comparison modifier, subquery syntax `[1d:30s]`, and rule-group `interval` semantics, which is where the integration constant belongs.
4. **"Part-time Power Measurements: nvidia-smi's Lack of Attention"** — arXiv 2312.02741 — the measured limits of NVML power reporting: roughly 25% of runtime sampled on A100/H100, a micro-benchmark suite run across 70+ GPUs spanning every architecture since Fermi, best practices cutting energy-measurement error by up to 35%, and an estimated $1M/year of extra electricity cost attributable to measurement error at 10,000-GPU scale. **Note:** arxiv.org was unreachable from this environment; the figures above are from the paper's abstract as surfaced in search, and the paper should be read directly before quoting it further.

**Primary sources — the mechanisms**

5. **vLLM — `vllm/v1/metrics/loggers.py`** — https://github.com/vllm-project/vllm — fetched directly. `vllm:num_requests_running`, `vllm:num_requests_waiting` (+ `_by_reason` with `capacity`/`deferred`), `vllm:kv_cache_usage_perc`, `vllm:num_preemptions`, `vllm:prompt_tokens`/`vllm:generation_tokens` (Counters, so exposed with the `_total` suffix), and `vllm:engine_sleep_state` with its documented levels — `weights_offloaded` = sleep level 1, `discard_all` = sleep level 2.
6. **Kubeflow notebook controller — `controllers/culling_controller.go`** — https://github.com/kubeflow/kubeflow — fetched directly. The `/api/kernels` polling mechanism, the "any busy kernel refreshes last-activity" rule, and the defaults `DEFAULT_ENABLE_CULLING = "false"`, `DEFAULT_CULL_IDLE_TIME = "1440"` minutes, `DEFAULT_IDLENESS_CHECK_PERIOD = "1"` minute.
7. **JupyterHub — `jupyterhub-idle-culler`** — https://github.com/jupyterhub/jupyterhub-idle-culler — fetched directly. `--timeout` default **600 s**, `--cull-every`, `--max-age`, and the documented constraint that the timeout must exceed the websocket ping interval (30 s) plus `JupyterHub.last_activity_interval` (5 min) because activity data is not updated frequently.
8. **Kubernetes SIG — descheduler** — https://github.com/kubernetes-sigs/descheduler — fetched directly. `LowNodeUtilization` (`thresholds`/`targetThresholds`, extended resources such as `nvidia.com/gpu` only when explicitly listed, `useDeviationThresholds`, `evictionLimits.node`, `metricsUtilization.source: Prometheus` with a per-node query returning values in `[0, 1]`, and the one-eviction-per-over-utilised-node limit on that path) and `PodLifeTime` (`maxPodLifeTimeSeconds`, `states`, `conditions` with `minTimeSinceLastTransitionSeconds`, `exitCodes`, `ownerKinds`).
9. **NVIDIA KAI Scheduler** — https://github.com/NVIDIA/KAI-Scheduler — fetched directly. Bin-packing versus spread, **workload consolidation**, **min-guaranteed-runtime** (the scheduler-side implementation of guard G5), GPU sharing, and queue-level reclaim strategies with DRF fairness.
10. **NVIDIA Multi-Process Service (MPS)** — https://docs.nvidia.com/deploy/mps/ — the "pack more instead of reclaim" lever, and `CUDA_MPS_ACTIVE_THREAD_PERCENTAGE` as the per-client SM cap that also bounds the attribution error under sharing (lesson 01).
11. **Kubernetes — Horizontal Pod Autoscaling** — https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/ — the correct lever for a quiet serving tier: scale on request-rate or queue-depth custom metrics, not on a GPU activity threshold.

**Reading the tools' source**

12. **OpenCost — `pkg/costmodel/costmodel.go`, `computeIdleAllocations`** — https://github.com/opencost/opencost — fetched directly. Idle as `assetTotal − allocTotal` per node or cluster, per resource, clamped at zero with the enumerated causes of a negative result, inserted as `<key>/__idle__`. This is lesson 02's gap A, not gap B.
13. **OpenCost — `core/pkg/opencost/allocation.go`** — fetched directly. `GPUCostIdle`, `IdleSuffix`, `IsIdle()`, the `ShareWeighted`/`ShareEven`/`ShareNone` constants, the eleven-step ordering comment in `AggregateBy`, and `computeIdleCoeffs`, which builds idle-sharing coefficients from each allocation's `GPUTotalCost()` over the per-idle-key total — a cost-weighted loading identical in shape to lesson 05's fully-loaded multiplier.
14. **OpenCost — `pkg/costmodel/allocation_helpers.go`** — fetched directly. `applyGPUsAllocated` (`GPUHours = <GPU request> × hours`, with the in-code note that GPU-hours reflect the full reserved allocation and that usage-based accounting would apply `GPUUsageAverage` separately) and `applyGPUUsageAvg` / `applyGPUUsageShared` / `applyGPUInfo`, which collect utilisation, the shared-device flag and the GPU UUID — and are unused in the cost.

**Real-world engineering**

15. **Anyscale — "GPU (In)efficiency in AI Workloads"** — https://www.anyscale.com/blog/gpu-in-efficiency-in-ai-workloads — a vendor teardown of the causes behind sustained low GPU utilisation in production: prefill and decode stalling each other in LLM serving, Python dataloaders starving training GPUs, CPU-heavy pipeline stages bottlenecking GPU-heavy ones. **What it shows:** the S2 bucket has named, recurring causes and is addressed by pipeline design, not by eviction. Treat any specific percentages as dated vendor snapshots; the durable content is the causal taxonomy.

---
Module backlink: [💰 11 — GPU cost and unit economics](../README.md)
