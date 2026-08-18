---
lesson: 04
title: "Flagship blog + demo: 'Your GPU dashboard is lying'"
module: 12
concept: "contrarian proof + narrated demo"
status: not-started
est_time: "5 hrs"
prev: "03-portfolio-writeup.md"
next: "05-system-design-drills.md"
artifacts: ["a published flagship blog post ('Your GPU dashboard is lying'), two secondary post outlines (cost-per-1M-tokens, survive-a-failure), and a scripted 5-minute narrated demo of the value chain"]
sources: 10
---

# Flagship blog + demo: "Your GPU dashboard is lying"

[🎓 12 — Capstone & interview preparation](../README.md)

## Where this fits

Lesson 03 established the *written* portfolio: a repo README with an inline platform diagram, an RFC-style design doc reasoning in Goals / Non-goals / Alternatives / Risks, and a brag-doc of quantified outcomes. That is evidence for a reader who has already clicked into your repo. What 04 adds is the **public contrarian proof and the narrated demo** — the two artifacts built for reach rather than depth, designed to pull in a stranger and hand them your framing in one sentence before they have read a line of your design doc. The flagship post and the demo go at the very top of the README from lesson 03, and they lead every application from here forward.

## Why this matters

A design doc proves you can reason. A flagship post proves you *know a thing most people in the room do not*.

"Your GPU dashboard is lying to you" is the strongest public proof in your portfolio because it is contrarian, quotable, and load-bearing on the exact insight that separates GPU-infra people from generic platform engineers: **the flagship GPU metric reports that a kernel was resident on the device, not that the device did useful work.** A generic SRE reads 99% utilisation and reports the fleet is busy. You read 99% utilisation alongside 16% SM-active and 1% tensor-pipe activity and report that 86% of the compute die is dark while the invoice runs at full rate. That gap is the whole thesis, and it is what a GPU-fleet hiring manager is scanning your writing for.

The reason a post beats a repo here is reach and framing. A repo requires the reader to reconstruct the insight; a post hands it to them in a sentence they can repeat to their own team. That repeatability is the mechanism: a hiring manager who reads it can say it in their next standup, and now your framing is in their head with your name attached. A repo is discovered only by someone already evaluating you; a post can be discovered by someone who has never heard of you and decides, on the strength of one argument, to go find your repo.

The title is contrarian on purpose — it makes a claim strong enough to be wrong, then earns it. Which means the entire load is carried by whether the reader can reproduce the measurement. A contrarian claim invites scrutiny by design; the thing that survives scrutiny is not cleverness, it is a fifteen-minute reproduction recipe.

This is also not a novel genre you are inventing, and that is a feature. Vendors and researchers have published versions of this argument — the term "Model FLOPs Utilization" exists precisely because the field needed a throughput number that a memory-bound kernel could not game by pinning a duty cycle at 100%. Writing your version well means executing a known-effective genre with your own fleet's numbers, your own derivation, and your own validation. **The differentiator available to you is not the claim; it is the rigour behind it** — the derivation from first principles, the synthetic ground-truth test, and the willingness to publish the mistake you made first.

## What's new here

- **You already know** how to write clearly and record a screen. **Skip** blogging and screencasting mechanics. The new thing is structuring an argument for a hostile, skimming reader, and narrating a value chain rather than a codebase.
- **You already know** the `SM_ACTIVE` vs `GPU_UTIL` distinction cold from module 05. **Skip** re-deriving it as a concept. Here it becomes the spine of a public argument that has to survive someone running your queries.
- **New**: the **substantial draft** — this lesson contains the post, in prose, section by section, built on module 05's verified findings. You edit it into your own voice and swap in your own measurements; you do not start from a blank file.
- **New**: the **objection map** — every beat of the post exists to close one specific objection a skeptical reader will raise, and knowing which is which tells you what may never be cut.
- **New**: the **reproduction kit** — the ten-line spin kernel, the `dcgmi dmon` invocation, and the detector query that let a reader verify the claim on any GPU in fifteen minutes.
- **New**: a **correction you must not inherit** — `SM_ACTIVE` and MFU are not the same number and must not be conflated. Earlier drafts of this material did conflate them, and a reader who knows the field will notice immediately.
- **New**: the **demo script**, minute by minute, with the narration written out and the payoff shot identified.

## Core concepts

### 1. What makes it the flagship

Three properties, all required, none sufficient alone:

- **Contrarian.** It contradicts the number everyone trusts. A claim strong enough to be wrong earns attention that a "how I built X" post never will.
- **Quotable.** The thesis compresses to one sentence a reader can repeat verbatim: *GPU utilisation measures whether a kernel was resident, not whether the silicon did work.*
- **Load-bearing.** It proves the specific competency a GPU-fleet operator screens for. A "how I built my exporter" post proves you can build. This proves you know something the reader's own team probably does not — which is what routes you to a loop.

A fourth property makes it survivable rather than merely attention-getting: **it is reproducible in fifteen minutes on any GPU**. That is what turns a provocative title into a citable post.

### 2. The claim you can actually defend — and one correction

Before writing a word, fix the three signals precisely, because the post's credibility collapses if you blur them. This is also where a widely repeated error creeps in, including into earlier drafts of this very lesson.

| Signal | Field | What it *actually* measures | What it hides |
|---|---|---|---|
| `GPU_UTIL` | `DCGM_FI_DEV_GPU_UTIL`, ID **203**, units **percent (integer)** | The percentage of a driver-chosen sample window (1 s to 1/6 s, product-dependent) during which **at least one kernel was resident**. An unmodified passthrough of NVML's `nvmlUtilization_t.gpu`. | Everything about intensity. It is a threshold at one: one kernel and ten thousand kernels evaluate the same predicate. |
| `SM_ACTIVE` | `DCGM_FI_PROF_SM_ACTIVE`, ID **1002**, units **ratio 0.0–1.0** | Summed active cycles over summed elapsed cycles, **averaged across all SMs**. A *breadth* measure: on a 132-SM H100, a kernel occupying 32 SMs continuously reads 32/132 = 0.24. | Depth and productivity. An SM with one stalled warp counts as active exactly as much as one with 64 running warps. |
| `PIPE_TENSOR_ACTIVE` | `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`, ID **1004**, ratio | Cycles during which **any** tensor pipe is active, against peak sustained elapsed cycles. The closest single field to "are we getting the FLOPs we are renting." | Which precision — that needs the sub-pipe fields 1013–1016. |
| **MFU** | derived, not a DCGM field | Achieved model FLOPs ÷ the hardware's peak FLOPs, over a window. A *workload-level* number computed from tokens or samples per second and the model's FLOPs per token — not read off a counter. | Nothing, but it requires knowing the model's arithmetic, which the platform layer usually does not. |

**The correction.** `SM_ACTIVE` is *not* MFU and must never be presented as one. `SM_ACTIVE` is an occupancy/breadth measure from a hardware counter; MFU is a throughput ratio derived from application-level work. A GPU running FP32 elementwise kernels can show `SM_ACTIVE = 0.9` with `PIPE_TENSOR_ACTIVE = 0.04` and an MFU near zero. If your post says "95% util, 31% MFU" while the number you measured was `SM_ACTIVE`, a reader who works in this field will spot it and stop reading. **Say which number you measured, in field-ID terms.**

**Why the platform layer usually publishes `SM_ACTIVE` rather than MFU.** MFU needs the model's FLOPs-per-token, which lives with the tenant, not with the fleet operator. `SM_ACTIVE` needs nothing but the counter. So the honest platform-side claim is the occupancy claim — *you allocated a GPU and almost nothing was scheduled on it* — and MFU belongs in the post as the calibration point rather than as your headline. That distinction is itself a paragraph worth writing, because it demonstrates that you know which layer owns which number.

**The calibration point, cited properly.** MFU was introduced in the PaLM paper (Chowdhery et al., 2022, arXiv:2204.02311) precisely because the field needed a throughput measure that a memory-bound kernel could not game. For scale: Meta reports roughly 38–43% model-FLOPs utilisation for Llama 3.1 pretraining on 16,384 H100s (arXiv:2407.21783). **If a team with that budget and that much tuning tops out below half of theoretical peak, a serving fleet at 0.011 tensor activity is not "a bit inefficient" — it is three orders of magnitude away, and `GPU_UTIL` reads ~100 for both.**

### 3. The spine, and the objection each beat closes

A contrarian post is an argument under attack. Every beat exists to close one specific objection, and knowing which beat closes which objection tells you what can be shortened and what can never be cut.

```
   THE POST AS AN ARGUMENT — beats, and the objection each one closes
  ══════════════════════════════════════════════════════════════════════════════

   BEAT                            CLOSES THE OBJECTION            CUTTABLE?
   ─────────────────────────────   ─────────────────────────────   ─────────
   1 · THE CLAIM                   —  (it *creates* the tension)   no
       one paragraph, the title
       restated as a fact
              │
              ▼
   2 · THE MECHANISM               "you're misreading the metric"  NEVER
       what field 203 IS: an       → quote the NVML struct
       NVML duty cycle, the           comment; name the field ID
       exact predicate
              │
              ▼
   3 · THE DERIVATION              "you had a broken exporter"     NEVER
       predict the gap BEFORE      → arithmetic that predicts
       measuring: 16 GB / 3.35        the result in advance
       TB/s vs 16 GFLOP / 989         cannot be an artefact
       TFLOP/s → 0.34%
              │
              ▼
   4 · THE EXHIBIT                 "show me"                       no
       one screenshot: same GPU,   → the payoff image
       same second, both numbers
              │
              ▼
   5 · THE REPRODUCTION            "works on your machine"         NEVER
       10-line spin kernel +       → 15 minutes on any GPU
       dcgmi dmon line
              │
              ▼
   6 · THE HONEST METRIC           "so what should I use?"         no
       SM_ACTIVE, its arithmetic,
       AND its limits (occupancy
       ≠ productivity)
              │
              ▼
   7 · THE ARITHMETIC              "your dollar figure is          shorten,
       ratio → GPU-hours, and       hand-waved"                    never cut
       the avg_over_time bug —     → publishing your own bug is
       INCLUDING that you got         the single strongest
       it wrong first                 credibility move available
              │
              ▼
   8 · THE DECOMPOSITION           "some of that is physics,       no
       six buckets, six owners,     not waste"
       six fixes                   → you said it first
              │
              ▼
   9 · THE VALIDATION              "how do you know your           NEVER
       five checks + the            measurement is right?"
       synthetic ground truth      → this is the section almost
                                      nobody writes
              │
              ▼
   10 · THE FIX                    "fine, but what do I DO?"       no
        what to alert on, what
        NOT to alert on, and the
        before/after that moves
        the honest metric
              │
              ▼
   11 · THE CAVEATS                "you overclaimed"               no
        time-slicing, MIG units,
        your number vs the
        industry number
              │
              ▼
   12 · THE TAKEAWAY               —  (manufactures the            no
        one repeatable line            quotable line)

   RULE: the four NEVER-CUT beats (2, 3, 5, 9) are exactly the four that
   convert a provocation into a citation. Everything else is compressible.
```

**Beat 7 deserves a note.** Publishing the mistake you made first — that `avg_over_time(x[24h]) * 24` overstates GPU-hours by exactly `window ÷ time-present` — feels like a weakness and is the opposite. Technical readers trust an author who shows their own error more than one who presents a clean result, because the clean result is exactly what a fabricated result looks like. It also inoculates you: nobody can "catch" you on a bug you published.

### 4. The draft

What follows is the post, written. Edit it into your voice, replace every number with one you measured, and keep the structure — the structure is doing the argumentative work mapped in §3.

---

> ## Your GPU dashboard is lying to you
>
> ### 1 · The claim
>
> Open any GPU dashboard. The panel at the top says utilisation, and right now it probably says
> something in the nineties. That number is true, and it is telling you almost nothing about
> whether the most expensive hardware you own is doing any work.
>
> On the fleet I measured, a serving namespace reported **99–100% GPU utilisation** while **16% of
> the streaming multiprocessors were lit**, the **tensor pipes ran at 1.1%**, and HBM was the only
> part of the chip working hard. Same GPU. Same second. The dashboard was solid green and 86% of
> the compute die was dark.
>
> This is not a broken exporter, and it is not a misconfiguration. It is what that metric has
> always measured. Here is the mechanism, the arithmetic that predicts it before you measure
> anything, a fifteen-minute reproduction on any GPU you can rent, and what to put on the dashboard
> instead.
>
> ### 2 · What the number actually measures
>
> The panel is almost certainly reading `DCGM_FI_DEV_GPU_UTIL`, field ID **203**, which is the same
> underlying counter as `nvidia-smi`'s `GPU-Util` column and the "GPU Utilization" panel in the
> official DCGM Grafana dashboard. DCGM does not compute it. In the cache manager, field 203 and
> its sibling `DCGM_FI_DEV_MEM_COPY_UTIL` share a single handler that calls NVML's
> `nvmlDeviceGetUtilizationRates()` once and takes `.gpu` and `.memory` from the result. No
> averaging, no scaling, no derivation. **Field 203 is `nvmlUtilization_t.gpu`, unmodified.**
>
> So its semantics are NVML's semantics, and NVML states them exactly, in the struct's own comment:
>
> ```c
> typedef struct nvmlUtilization_st
> {
>     unsigned int gpu;     // Percent of time over the past sample period during which
>                           // one or more kernels was executing on the GPU
>     unsigned int memory;  // Percent of time over the past sample period during which
>                           // global (device) memory was being read or written
> } nvmlUtilization_t;
> ```
>
> One sentence, no ambiguous words: **it is the percentage of a short, driver-chosen sample window
> during which at least one kernel was resident on the GPU.**
>
> Every clause is where an assumption dies.
>
> - **"at least one kernel"** — a threshold at one, not an intensity. One kernel and ten thousand
>   concurrent kernels both make the predicate true. The counter has no arity.
> - **"resident" / "executing"** — the kernel is scheduled on the device. A kernel whose every warp
>   is stalled waiting on a memory load is still executing by this definition, and will still be
>   executing four microseconds later when it is still waiting.
> - **"a short sample period"** — between one second and one sixth of a second, product-dependent,
>   per the header. You do not choose it and you cannot read it back.
> - **"percent"** — integer, 0–100. Not a ratio. The honest metrics below are doubles in 0.0–1.0,
>   and comparing them directly is off by 100×.
>
> Two more facts from the same header that almost nobody mentions: this call is **not supported on
> MIG-enabled GPUs**, so a panel that goes blank on the nodes you switched to MIG last week is this,
> not a broken exporter; and **ECC memory scrubbing at driver initialisation** produces high
> utilisation readings on a device that is running nothing.
>
> ### 3 · Predicting the gap before measuring it
>
> The strongest form of this argument is not a screenshot. It is arithmetic that tells you what the
> screenshot will show.
>
> Take an 8-billion-parameter model in BF16 on one H100 SXM5, generating at batch size 1. Three
> numbers from the datasheet and the tuning guide: **132 SMs**, **≈3.35 TB/s of HBM3 bandwidth**,
> **≈989 TFLOP/s of dense BF16 tensor throughput**.
>
> ```
>   MEMORY SIDE — every weight is read from HBM once per token (batch 1, no reuse)
>       bytes_moved = 8e9 params × 2 bytes           = 16 GB
>       t_memory    = 16 GB ÷ 3.35 TB/s              ≈ 4.78 ms
>
>   COMPUTE SIDE — a forward pass is ~2 FLOP per parameter per token
>       flops       = 2 × 8e9                        = 16 GFLOP
>       t_compute   = 16 GFLOP ÷ 989 TFLOP/s         ≈ 16.2 µs
>
>   RATIO       t_compute / t_memory = 16.2 / 4780   ≈ 0.0034   (0.34%)
>
>   ROOFLINE    arithmetic intensity = 16 GFLOP / 16 GB   ≈ 1 FLOP/byte
>               H100 ridge point     = 989e12 / 3.35e12   ≈ 295 FLOP/byte
>               → ~295× to the LEFT of the ridge: hard memory-bound
> ```
>
> Two conclusions fall straight out, and together they *are* the post.
>
> **The tensor pipes must be idle about 99.7% of the time.** There is no tuning that changes this
> at batch 1; it is arithmetic. Real measurements land a little higher — attention adds work and
> real kernels are not perfectly bandwidth-efficient — but the order of magnitude is derived, not
> observed.
>
> **And utilisation must read ~100.** During those 4.78 ms there is always exactly one kernel
> resident: a matrix-vector product against a weight matrix, then the next, then attention over the
> KV cache, then the next layer, enqueued back to back on one stream. The gap between kernels is a
> few microseconds of launch latency, far below the sample window's floor of about 167 ms. The
> predicate is true for essentially the whole window.
>
> **The field is not malfunctioning. It is correctly reporting a true fact that is useless.**
>
> ### 4 · The exhibit
>
> Here is the same GPU, over five consecutive one-second samples, with six fields side by side.
> `dcgmi dmon` prints each field's registered short name: `GPUTL` is 203, `SMACT` is 1002, `SMOCC`
> is 1003, `TENSO` is 1004, `DRAMA` is 1005, `FBUSD` is 252.
>
> ```console
> $ dcgmi dmon -e 203,1002,1003,1004,1005,252 -i 0 -d 1000
> #Entity        GPUTL       SMACT   SMOCC   TENSO   DRAMA   FBUSD
> ID
> GPU 0          99          0.163   0.088   0.011   0.706   58122
> GPU 0          100         0.157   0.085   0.009   0.718   58122
> GPU 0          100         0.171   0.093   0.013   0.699   58122
> GPU 0          99          0.160   0.087   0.010   0.712   58122
> GPU 0          100         0.166   0.090   0.012   0.704   58122
> ```
>
> Read it column by column:
>
> | Reading | Value | What it proves |
> |---|---|---|
> | `GPUTL` | 99–100 | at least one kernel resident essentially always. True. Uninformative. |
> | `SMACT` | 0.16 | about 21 of 132 SM-equivalents lit. **86% of the compute die is dark.** |
> | `SMOCC` | 0.09 | about 760 of 8,448 warp slots occupied. The grids are tiny. |
> | `TENSO` | 0.011 | tensor pipes issuing ~1% of cycles — matching the 0.34% floor derived above, plus attention and inefficiency. |
> | `DRAMA` | 0.71 | 0.71 × 3.35 TB/s ≈ **2.4 TB/s achieved**. HBM is the actual bottleneck. |
> | `FBUSD` | 58,122 MB | ~58 GB resident: weights plus the KV-cache arena. **This card cannot simply be reclaimed.** |
>
> One sentence of diagnosis: *this fleet is not compute-saturated, it is memory-bandwidth-bound at
> batch size 1, using 1% of its tensor throughput and 16% of its SM breadth — and buying more of the
> same hardware will reproduce the same 1%.*
>
> ### 5 · Reproduce it in fifteen minutes
>
> You do not need my fleet. You need one GPU and ten lines.
>
> ```python
> # util_lie.py — one block, one thread, doing nothing, forever.
> import torch, time
> s = torch.cuda.Stream()
> x = torch.zeros(1, device="cuda")
> t_end = time.time() + 900          # 15 minutes
> with torch.cuda.stream(s):
>     while time.time() < t_end:
>         for _ in range(10_000):
>             x.add_(1)              # a stream of tiny 1-element kernels
> ```
>
> An H100 has 132 SMs. A one-element kernel occupies one of them. Utilisation reports ~100 because
> a kernel is resident for essentially the whole sample window; `SM_ACTIVE` reports roughly
> 1/132 ≈ 0.008 because it averages resident-warp cycles across every SM. **Two orders of magnitude
> between two metrics that people believe measure the same thing, reproducible on any GPU in
> fifteen minutes.**
>
> One prerequisite, and it is the punchline of the whole post: **you probably cannot see the honest
> metric yet.** `DCGM_FI_PROF_SM_ACTIVE` ships **commented out** in dcgm-exporter's default counter
> CSV, and the profiling fields additionally need elevated privileges that the device-level fields
> do not. The vendor's own shipped Grafana dashboard has eight panels — temperature, average
> temperature, power, total power, SM clocks, GPU Utilization, framebuffer used, tensor-core
> utilisation — and **no `SM_ACTIVE` panel, no `SM_OCCUPANCY` panel, no `DRAM_ACTIVE` panel.**
>
> That is why every GPU dashboard shows the misleading metric. Not carelessness. **The misleading
> metric is the default and the honest one is opt-in.**
>
> ### 6 · The honest metric, and its own limits
>
> `DCGM_FI_PROF_SM_ACTIVE` (field 1002) is not a driver counter at all. It comes from differencing
> two hardware performance-monitor snapshots, and its definition is *the ratio of cycles an SM has
> at least one warp assigned*, computed per SM and averaged across the device:
>
> ```
>                      Σ over SMs ( active_cycles[sm] )
>    SM_ACTIVE   =    ─────────────────────────────────
>                      Σ over SMs ( elapsed_cycles[sm] )
>
>    On a 132-SM H100, a kernel occupying exactly 32 SMs continuously
>    for the whole interval yields  SM_ACTIVE = 32/132 = 0.24
>    — even though those 32 SMs are 100% busy.
> ```
>
> It is a **breadth** measure. That is exactly what you want for the question "did I allocate a GPU
> and schedule nothing on it," which is the money question.
>
> It is not a productivity measure, and I want to be precise about that, because the failure mode of
> this genre is replacing one over-trusted number with another. An SM with a single stalled warp
> counts as active for that cycle exactly as much as an SM with 64 running warps. A GPU running FP32
> elementwise kernels can sit at `SM_ACTIVE = 0.9` with `PIPE_TENSOR_ACTIVE = 0.04`. So:
>
> - **`SM_ACTIVE` is the right metric for the waste claim** — "you are holding a GPU and nothing is
>   scheduled on it" is a scheduling fact, and occupancy is the right evidence for it.
> - **`PIPE_TENSOR_ACTIVE` is the right metric for the efficiency claim** — "work is scheduled, but
>   not the work you bought this GPU for" is a different claim with a different owner: the tenant's
>   ML engineer, not the platform team.
>
> Ship both, label them differently, and never blend them into one "efficiency" number.
>
> **And one trap.** `DCGM_FI_PROF_GR_ENGINE_ACTIVE` (1001) looks like the honest family because it
> carries the same prefix. Its own definition is *a graphics or compute context is bound and the
> graphics or compute pipe is busy* — which is a presence duty cycle reimplemented on the hardware
> path. On the batch-1 decode server above it reads about 1.0 alongside `SM_ACTIVE = 0.16`. It also
> ships **enabled** by default. If your "real utilisation" panel is built on 1001, you have rebuilt
> the lie with a longer metric name.
>
> ### 7 · Turning a ratio into hours — and the bug I shipped first
>
> A ratio in [0,1] is not a quantity. To say anything about money you need GPU-hours, which is an
> integral:
>
> ```
>    GPU-hours ≈  Σ  value_i × Δ / 3600         Δ = sample spacing, seconds
>                 i
> ```
>
> The query almost everyone writes — and the one I wrote first — is:
>
> ```promql
> # ✗ WRONG for any workload that did not run for the whole window
> sum by (ns) (avg_over_time(gpu:sm_active:ratio[24h])) * 24
> ```
>
> `avg_over_time` averages **only over samples that exist**. If a job ran for 9 hours of a 24-hour
> window, its series exists for 9 hours, `avg_over_time` returns its mean over those 9 hours, and
> multiplying by 24 extrapolates it across the 15 hours it was not running.
>
> ```
>    Window: 24 h. Pod ran 06:00–12:00 at a steady SM_ACTIVE of 0.60.
>    True utilised GPU-hours = 6 h × 0.60 = 3.60
>
>    SM_ACTIVE
>      1.0 ┤
>      0.6 ┤          ████████████████
>      0.0 ┼──────────████████████████────────────────────────────────▶ t
>          0h        6h              12h                            24h
>          │◀── no series ──▶│◀ 6h of samples ▶│◀──── no series ────▶│
>
>    avg_over_time(x[24h]) * 24       = 14.40 GPU-h   ✗  4× too high
>    sum_over_time(x[24h]) * 30/3600  =  3.60 GPU-h   ✓  exact
> ```
>
> **The error is exactly `window ÷ time-present`.** On my own bursty namespace that was 2.67×. And
> notice the direction: the wrong query makes the fleet look *more* utilised and the waste look
> *smaller*, precisely on the workloads where the waste is worst. The naive query understates the
> problem you are trying to publish.
>
> Use `sum_over_time`, and pin the sample spacing by integrating a recording rule whose group
> `interval` you control, rather than a raw series whose spacing changes when someone edits a scrape
> config.
>
> ### 8 · One gap, six causes, six owners
>
> "Allocated minus utilised" is a single number hiding at least six distinct causes with six
> different owners and six different fixes. A dashboard that shows only the total is a complaint.
> One that decomposes it is a plan.
>
> | Bucket | Basis | Owner | Fix |
> |---|---|---|---|
> | ① productive | ∫ tensor-active | nobody | nothing |
> | ② busy but inefficient | ∫ (SM_ACTIVE − tensor) | tenant ML engineer | precision, kernel fusion, dataloader |
> | ③ **allocated-idle** ◀ the headline | ∫ (1 − SM_ACTIVE) while allocated | tenant + platform | right-size, TTL on idle notebooks, preemption |
> | ④ unattributable | allocated hours on time-sliced or MPS devices | platform | MIG, or dedicated devices |
> | ⑤ cordoned / drained | hours with a health flag | platform | remediation automation |
> | ⑥ unallocated | present but held by no pod | platform / capacity | scheduling, autoscaling, commitments |
>
> Two consequences matter more than the table. **Bucket ③ is the only one I promise to recover** —
> it is unambiguous waste with no physics defending it, whereas a memory-bandwidth-bound decode
> service sits in ② by physics and is not recoverable by better scheduling. And **buckets ⑤ and ⑥
> are the platform team's own number**: publishing them on the same chart as the tenant buckets is
> what makes this a diagnostic rather than an accusation. I am grading myself on the same graph.
>
> ### 9 · How I know the measurement is right
>
> This is the section that separates a measurement from an opinion, and it is the one almost nobody
> writes. Five checks, all of which passed before any figure above was published.
>
> **1 · Reconciliation.** The six buckets must sum to the physical ceiling: every GPU DCGM can see,
> integrated over the window. Agreement within about 1%. A larger discrepancy means scrape gaps, an
> unaccounted sharing mode, or a node whose exporter is down — which silently removes it from *both*
> sides and understates the total.
>
> **2 · Ordering.** Utilised must never exceed allocated, per namespace and per GPU. The query that
> tests it must return nothing; any result is a bug, and it means work is running on a GPU with no
> allocation record.
>
> **3 · Completeness.** At 30-second sampling a 24-hour window should contain 2,880 samples per
> series. Anything materially below is a gap, and gaps make waste look *smaller* — an error in the
> direction that makes you look wrong later. I accept nothing under 98%.
>
> **4 · Synthetic ground truth.** This is the one that actually proves the arithmetic. Saturate one
> GPU for exactly 600 seconds with a burn tool, then ask the pipeline what it cost:
>
> ```bash
> docker run --rm --gpus '"device=3"' oguzpastirmaci/gpu-burn 600
> # expected utilised GPU-hours = 600/3600 = 0.1667
> ```
>
> ```promql
> sum(sum_over_time(gpu:sm_active:ratio{UUID="GPU-…"}[30m])) * 30 / 3600
> ```
>
> Accept 0.150–0.184 — the quantisation error of a 30-second Riemann sum over a 600-second run is
> ±30 s at each edge. **Every arithmetic error in the pipeline shows up here:** 0.30 means double
> counting (two exporters, or a duplicated scrape job), ~0.08 means the sample spacing is 15 s not
> 30 s, ~4.0 means you reproduced the `avg_over_time` bug from §7. Ten minutes of one GPU, and it
> retires the entire "your measurement is wrong" objection.
>
> **5 · External agreement.** Total allocated GPU-hours against the invoice for the same instances
> over the same day. They will not match exactly — the invoice bills whole instances whether or not
> their GPUs are allocated to pods, which is bucket ⑥ — but the invoice must be ≥ your allocated
> hours, and the difference must equal buckets ⑤ plus ⑥. If it does not, you have found either a
> billing surprise or a bug.
>
> ### 10 · What to put on the dashboard instead
>
> Four rules, in the order I would apply them.
>
> **Page on `SM_ACTIVE`, gated by framebuffer.** The reclaim alert is "allocated, and doing no work
> for long enough that this is not a gap between steps."
>
> ```promql
> # Allocated GPUs doing essentially no SM work for 30 minutes, and not
> # holding a large resident model or KV-cache arena.
> #   0.05 : 5% of SM-cycles. A healthy training job's between-steps gap
> #          never sustains below this for 30m — tune against your fleet.
> #   2048 : MB. Below ~2 GB the card is not holding a served model.
> (
>     avg_over_time(DCGM_FI_PROF_SM_ACTIVE[30m]) < 0.05
>   and on (gpu, UUID, instance)
>     DCGM_FI_DEV_FB_USED < 2048
> )
> ```
>
> The gate is not optional. A paused-but-loaded serving replica reads `SM_ACTIVE ≈ 0` with 58 GB
> resident. Reclaim it and you destroy a warm replica, eat a multi-minute model load on the next
> request, and — more expensively — the reclaim system gets switched off and never comes back.
>
> **Warn, never page, on tensor activity.** Efficiency is a conversation, not an incident. It
> belongs in a weekly report, not in a pager at 03:00.
>
> **Never alert on `GPU_UTIL` or `GR_ENGINE_ACTIVE`.** Both are presence duty cycles. A reclaim rule
> built on either will *never fire on your most expensive waste*, because that waste is precisely
> the case that keeps a kernel resident. Keep 203 on exactly one panel: directly beside
> `SM_ACTIVE`, as the foil that makes the gap visible.
>
> **Alert on absence.** Because blank profiling values are *dropped* rather than zeroed, a
> permissions failure looks like silence, and silence looks like health:
>
> ```promql
> # SM_ACTIVE stopped being exported for a GPU still reporting device fields.
>   count by (instance, gpu) (DCGM_FI_DEV_GPU_UTIL)
> unless
>   count by (instance, gpu) (DCGM_FI_PROF_SM_ACTIVE)
> ```
>
> Absence is not zero in PromQL. `avg by (namespace)` silently skips a GPU that stopped reporting,
> so a fleet where half the cards went dark on the honest metric shows an unchanged, healthy-looking
> average. This is the single most common way an "everything looks fine" GPU dashboard lies by
> omission.
>
> **And the proof the fix works.** The team enabled continuous batching and raised the sequence
> limit. Across the rollout, on the same eight cards:
>
> | Phase | `GPUTL` | `SMACT` | `SMOCC` | `TENSO` | `DRAMA` | Throughput |
> |---|---|---|---|---|---|---|
> | before (effective batch 1) | 99 | 0.16 | 0.09 | 0.011 | 0.71 | 1.0× |
> | mid-rollout (batch ~8) | 99 | 0.34 | 0.21 | 0.11 | 0.74 | ~1.8× |
> | after (batch ~32) | **99** | **0.55** | **0.38** | **0.19** | **0.68** | **~2.9×** |
>
> Batching amortises each weight read across B sequences, so bytes-moved per token falls by roughly
> B while FLOPs per token stay constant — arithmetic intensity climbs from ~1 toward the ridge at
> 295, and the tensor pipes start seeing real matrix-matrix products instead of matrix-vector ones.
>
> **And utilisation never moved.** 99 before, 99 during, 99 after, while useful output nearly
> tripled. **The default metric is uninformative about the problem and uninformative about the
> solution.**
>
> ### 11 · What I am not claiming
>
> - **Time-sliced GPUs cannot be attributed per tenant, by anyone.** The profiling counter is scoped
>   to the device, not to the CUDA context that happened to be resident when it was sampled. N pods
>   sharing a device all read the same number. Any per-tenant split there is an estimate; mine emits
>   one with an explicit uncertainty label, and I would distrust any tool that does not.
> - **Under MIG the accounting unit is not a whole GPU.** A `1g.10gb` slice is one seventh of an
>   H100 and must be costed as such, or you will report seven cards' worth of allocation on one
>   card.
> - **Industry averages are directional.** Published figures putting typical fleet utilisation in
>   the 10–25% range are order-of-magnitude context, not a constant. **My own measured number is the
>   one I stand behind**, and mine came out *better* than the range often quoted, which is exactly
>   why I lead with it rather than with the more dramatic industry figure.
> - **This is a simulated fleet where labelled as such**, with the traces published, validated
>   against one rented GPU-afternoon.
>
> ### 12 · The takeaway
>
> **GPU utilisation is a liveness check, not an efficiency metric.** If it is the number on your
> dashboard, you are paying full price for a fraction of the work and the dashboard is structurally
> incapable of showing you which fraction. Turn on `SM_ACTIVE`, put it beside `GPU_UTIL` on one
> panel, and look at the gap. Then run the ten-line script above and watch two metrics that measure
> "the same thing" disagree by two orders of magnitude.

---

### 5. The reproduction kit

Three things belong in the post because each closes an objection no prose can close.

**The spin kernel** (beat 5) is the minimal reproduction: ten lines, no model, no dataset, any GPU.
Its job is to make the mechanism undeniable rather than to be realistic.

**The realistic workload** is the one to lead the exhibit with, because a reader recognises it:

```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct --port 8000 --max-num-seqs 1
curl -sN localhost:8000/v1/completions -H 'content-type: application/json' -d '{
  "model":"meta-llama/Llama-3.1-8B-Instruct",
  "prompt":"Write a 4000 word essay about the history of memory bandwidth.",
  "max_tokens":4000,"stream":true}' > /dev/null
```

Publish the realistic one because people recognise it; keep the ten-line one as the "here is why,
minimally" explanation.

**The detector query** is what a reader runs against their own fleet:

```promql
# GPUs claiming ≥90% busy while fewer than 20% of SMs are lit.
# Both sides must be present, so this also proves you enabled SM_ACTIVE.
  DCGM_FI_DEV_GPU_UTIL > 90
and on (UUID)
  DCGM_FI_PROF_SM_ACTIVE < 0.2
```

Join on `UUID` — it is the one label stable across every series, unlike `gpu` (an index, not unique
across nodes) or `instance` (changes if the exporter moves).

**The prerequisite paragraph is part of the kit.** Tell the reader that the query may return
nothing because the field is not collected, and that this is the finding rather than a problem with
their setup. Half your readers will discover their own fleet has never exported the honest metric,
and that discovery is what makes them share the post.

### 6. The two secondary posts

One flagship, two supporting posts that show range without competing for the lead. Sequence them
*after* the flagship everywhere, so the contrarian lead pulls readers in and the secondaries hold
the ones who want depth.

**Secondary A — "What a million tokens actually costs" (artifact 07).** The economics proof. The
spine: take attributed GPU-hours from the flagship's pipeline, multiply by a dated rate on a named
basis, divide by token counters from the serving stack, and split input from output by prefill and
decode compute time rather than by a fixed multiplier. The honest distinctions to make explicit are
(i) direct cost versus fully loaded, (ii) the allocation basis versus the usage basis — one
reconciles to the invoice and includes idle, the other reflects active compute only and explicitly
does not reconcile — and (iii) that a per-token figure computed on a MIG instance inherits whatever
error the whole-device numerator carried. Its job in the portfolio is to prove you connect the metal
to the P&L.

**Secondary B — "Surviving a dead GPU mid-run" (artifact 08).** The operations proof. The spine: a
distributed job loses a rank; what the failure looks like from each layer (the collective hangs
rather than errors, so the *symptom* is a stall, not a crash); what the checkpoint interval costs in
steady state versus what it saves in expectation; and how the platform degrades. The number to lead
with is a real recovery time and a real checkpoint overhead percentage from your own lab. Anchor the
motivation on published reliability data rather than on drama: eleven months of operations on two
large research clusters gives a **measured** mean-time-to-failure of 7.9 hours for 1,024-GPU jobs,
with shorter figures at larger scales given as the paper's **projections** (arXiv:2410.21680). Say
which is which — quoting a projection as a measurement is exactly the slip that costs you a careful
reader.

### 7. The five-minute demo

**Narrate the value chain, not the code.** Nobody hires off watching you scroll a file. They hire
off watching you reason about a system. Every minute must advance an argument.

```
   THE 5-MINUTE DEMO — screen, narration, and the payoff shot
  ══════════════════════════════════════════════════════════════════════════════

   0:00 ─┬─ ON SCREEN: the whole-platform diagram (lesson 03)
         │  SAY: "One reference GPU platform. Signals come off the silicon,
         │        get made honest, get attributed to a tenant, drive an
   0:45 ─┤        allocation decision, wrapped in a survivability layer."
         │
         │  ON SCREEN: the stock dashboard. Utilisation panel, solid green,
         │             99%. Do not touch anything for three seconds.
         │  SAY: "This is the panel every GPU fleet leads with, and it is the
   2:00 ─┤        one number I do not let anyone make a decision on. It means
         │        a kernel was resident. That's all it means."
         │
         │  ★★★ THE PAYOFF SHOT ★★★
         │  ON SCREEN: switch to the honest panel — SM_ACTIVE 0.16 and
         │             TENSOR 0.011 on the SAME GPU, SAME time range,
         │             plotted on the same 0–1 axis as GPU_UTIL/100.
   3:30 ─┤  SAY: "Same card. Same second. Eighty-six percent of the compute
         │        die is dark. Here's the arithmetic that predicted it before
         │        I measured: 16 GB of weights per token at 3.35 TB/s is
         │        4.8 milliseconds; 16 GFLOP at 989 TFLOP/s is 16 microseconds."
         │
         │  ON SCREEN: trace ONE GPU-hour. The attribution panel (pod →
         │             namespace → share), then the six-bucket decomposition,
         │             then the alert that should have fired.
   4:30 ─┤  SAY: "This hour belongs to that namespace, at that share, on that
         │        basis — and the shares on this physical card sum to exactly
         │        one, which is asserted continuously, not checked once."
         │
         │  ON SCREEN: the README front door.
   5:00 ─┴─ SAY: "Utilisation is a liveness check, not an efficiency metric.
                  The post has the ten-line reproduction; the repo has the
                  query pack and the validation."

   RULES
     · Not one frame of scrolling code or explaining directory layout.
       File layout is plumbing, not a decision, and a reviewer learns
       nothing about your reasoning from it.
     · The cut at 2:00 is the whole demo. Rehearse the transition until
       it lands cleanly — that single cut is the thesis made visible.
     · Record only once you can narrate without reading. The voiceover
       carries it; the screen is evidence.
     · Both series on ONE axis, with GPU_UTIL divided by 100. Plotting 99
       against 0.16 on one axis flattens the honest series into the floor.
```

### 8. Distribution, and the asymmetry that justifies it

```
   WHY THE POST LEADS AND THE REPO FOLLOWS
  ══════════════════════════════════════════════════════════════════════════════

   REPO — depth work                      POST — reach work
   ─────────────────────                  ─────────────────
   found by: someone already              found by: someone who has never
             evaluating you                         heard of you
        │                                      │
        ▼                                      ▼
   requires: a reason to look             requires: one interesting sentence
        │                                      │
        ▼                                      ▼
   produces: a considered judgement       produces: a forward, a bookmark,
             about depth                            a name in someone's head
        │                                      │
        └──────────────┬───────────────────────┘
                       ▼
             ┌──────────────────────┐
             │ the post creates the │
             │ reason to open the   │
             │ repo. The repo       │
             │ converts it.         │
             └──────────┬───────────┘
                        ▼
          THEREFORE: the post link goes at the top of the README,
          at the top of every application, and in the first line of
          your positioning sentence — not the repo URL.
```

Practical rules:

- **Own the canonical surface.** Publish on something you control, then syndicate. A post that lives
  only on a platform you do not own can vanish, and the link in your applications goes dead.
- **Publishable-by-default.** Write it so it *can* go public from the first draft. Retrofitting a
  scrub is where mistakes happen.
- **Scrub employer specifics before the first push, not before the publish.** Real fleet sizes
  become ratios or a stated simulated fleet; internal service names become generic tiers; negotiated
  rates never appear. The argument works identically on synthetic-but-realistic figures, so there is
  no upside to the exposure.
- **Date every rate and name its basis.** Your most challengeable sentence is the one with a dollar
  sign in it.
- **Link the repo from the post and the post from the repo**, so whichever one is discovered first
  leads to the other.

## Perspectives

**The reader-as-hiring-manager.** They are scanning for one line worth repeating to their own team.
They will not remember your prose, your formatting, or most of your argument — they remember the
sentence they said out loud in their next standup. The whole post is in service of manufacturing
that sentence, and every paragraph that does not sharpen it dilutes the thing that actually routes
you to a loop.

**The reader-as-skeptic.** A contrarian claim invites scrutiny by design; you are telling an
experienced reader that a number they trust is wrong, and their first instinct is to look for the
seam. Cleverness earns a skim. What earns trust is the derivation that predicts the result before
measuring, the ten-line reproduction, and the validation section — and trust is what turns a reader
into someone forwarding your post.

**The finance and leadership view.** Whether to frame a metric flatteringly or truthfully is a
political choice as much as a technical one, and this post is a public demonstration of which side
you land on. A dashboard reading 99% looks excellent in a leadership review; a decomposition showing
57% of allocated GPU-hours doing no SM work invites uncomfortable questions about spend. Publicly
choosing the harder, truer number — and explaining what it means for the P&L rather than only for
engineers — is itself the signal. So is refusing to promise recovery of the part that is physics.

**The vendor's view.** This argument has been made publicly before, by people selling products
around it, and that is a feature: it means the genre is proven and the claim is not eccentric. It
also sets your bar. Next to a vendor's version, the differentiators available to you are the ones
they usually skip — the roofline derivation that predicts the number in advance, the published bug
in your own arithmetic, and the synthetic ground-truth validation. Write those and yours is the
sharper post regardless of who has the bigger audience.

**The maintainer's view.** Be careful to criticise a *default*, not a project. "The honest metric
ships commented out" is an observation about a configuration file. "The exporter is bad" is an
opinion that will meet a maintainer in a comment thread and lose. The former is stronger *and*
safer, and the distinction between a defect and a scope boundary is itself a senior signal.

## Real-world use cases

- **NVIDIA `dcgm-exporter`, `etc/default-counters.csv`.** `DCGM_FI_DEV_GPU_UTIL` and
  `DCGM_FI_PROF_GR_ENGINE_ACTIVE` ship enabled; `DCGM_FI_PROF_SM_ACTIVE` and
  `DCGM_FI_PROF_SM_OCCUPANCY` ship commented out. **What it shows:** the single strongest exhibit in
  the post. The reason every GPU dashboard shows the misleading metric is not that everyone is
  careless — it is that the misleading metric is what the box ships with. One file, thirty seconds
  to verify, and it converts your claim from a provocation into an observation.

- **NVIDIA `dcgm-exporter`'s shipped Grafana dashboard** (`grafana/dcgm-exporter-dashboard.json`,
  published as grafana.com dashboard 12239). Eight panels: temperature, average temperature, power
  usage, total power, SM clocks, GPU Utilization, framebuffer used, tensor-core utilisation. No
  `SM_ACTIVE`, no `SM_OCCUPANCY`, no `DRAM_ACTIVE`. **What it shows:** the most widely installed GPU
  dashboard in existence leads with the presence metric and offers exactly one efficiency metric —
  from the vendor, in the box.

- **`NVIDIA/dcgm-exporter` issue #34 — profiling privileges.** An A100 node with MIG enabled fails
  to start the exporter, printing a warning that it lacks sufficient privileges to expose profiling
  metrics and needs `SYS_ADMIN`. **What it shows:** the honest metrics need elevated privileges the
  device-level fields do not, and the gap between them produces a half-populated dashboard rather
  than a loud failure. Current charts add the capability by default, but any hardened deployment
  that drops capabilities recreates the bug — which is why the post's "alert on absence" rule
  exists.

- **Chowdhery et al., *PaLM: Scaling Language Modeling with Pathways* (arXiv:2204.02311).** Where
  "Model FLOPs Utilization" was introduced, explicitly as a throughput measure that is not gameable
  the way a duty cycle is. **What it shows:** citing the origin signals you know the metric's
  provenance rather than treating it as a buzzword — and it is the right place to make the
  `SM_ACTIVE` ≠ MFU distinction explicit.

- **Meta, *The Llama 3 Herd of Models* (arXiv:2407.21783).** Roughly 38–43% model-FLOPs utilisation
  for Llama 3.1 pretraining on 16,384 H100s. **What it shows:** the calibration that makes your
  numbers meaningful. If a team with that budget and tuning effort tops out below half of
  theoretical peak, a serving fleet at 0.011 tensor activity is three orders of magnitude away — and
  the duty cycle reads ~100 for both.

- **Meta, *Revisiting Reliability in Large-Scale Machine Learning Research Clusters*
  (arXiv:2410.21680, HPCA 2025).** Measured MTTF of 7.9 hours for 1,024-GPU jobs across eleven
  months on two large A100 clusters; 1.8 h at 16,384 GPUs and 0.23 h at 131,072 are the paper's
  projections. **What it shows:** the anchor for secondary post B, and a live example of the
  citation discipline the flagship depends on — quoting a projection as a measurement is precisely
  the error a careful reader catches.

## Worked example

**The scenario:** the post goes up, and within a day someone with real expertise pushes back in
public. This is the outcome you want, and how you handle it is part of the artifact. Here are the
four objections that actually arrive, and the answer to each — all of which are already in the post,
which is the point of §3's objection map.

**Objection 1: "You just had profiling misconfigured."** *Answer:* the two families come from
independent collection paths. Field 203 is a straight passthrough of an NVML driver counter that
integrates a one-bit predicate; 1002 is a difference of two hardware performance-monitor snapshots
over the watch interval. Neither can be derived from the other and neither can be misconfigured into
the other. Also — beat 3 — I predicted the gap arithmetically before measuring it, and a prediction
cannot be a configuration artefact.

**Objection 2: "16% SM-active just means your batch size was 1. That's a workload problem, not a
metric problem."** *Answer:* agreed, and that is the post's point in its strongest form. The
workload problem is real and fixable — batching took `SM_ACTIVE` from 0.16 to 0.55 and throughput to
~2.9×. What I am claiming about the metric is narrower and worse: **the duty cycle read 99 before,
during and after the fix.** It could not see the problem and it could not see the solution. A
dashboard built on it would have shown a flat line while useful output nearly tripled.

**Objection 3: "Your dollar figure assumes you'd recover all of it."** *Answer:* it does not, and
beat 8 says so before you asked. The gap decomposes into six buckets; only bucket ③ is unambiguous
waste. A memory-bandwidth-bound decode service sits in bucket ② by physics and no scheduling change
recovers it. My projection moves one namespace from 0.11 to a conservative 0.40 mean `SM_ACTIVE` —
well below the 0.62 measured on another namespace of the *same* cluster — and the arithmetic is
published so you can disagree with the assumption rather than with the conclusion.

**Objection 4: "OK, but `SM_ACTIVE` is not utilisation either."** *Answer:* correct, and beat 6 says
so explicitly. It is a breadth measure, not a productivity measure: an SM with one stalled warp
counts as active exactly as much as one with 64 running warps, and a GPU doing FP32 elementwise work
can sit at 0.9 with tensor pipes at 0.04. That is why the post ships two metrics with two claims
attached — occupancy for the waste claim, tensor activity for the efficiency claim — and never
blends them into a single "efficiency" number.

**What this exercise proves.** Every objection was answered by a beat that was already in the post.
That is not luck; it is what §3's map is for. **Write the objection list first, then write the post
that closes it** — and when the objections arrive in public, answer with a pointer to the section
plus one sentence, which is far stronger than arguing.

The same four objections arrive in an interview, phrased as follow-ups, which is the other reason to
rehearse them. A candidate who answers "did you have profiling misconfigured?" with a derivation is
having a different conversation from one who answers with reassurance.

## Practice

Feeds [GPU platform capstone](../practice/gpu-platform-capstone/README.md).

1. **Write the objection list before the draft.** List every way a knowledgeable reader could
   attack the claim. Map each to a beat from §3. Any objection with no beat is a section you have
   not written; any beat closing no objection is a section you can cut.

2. **Draft the post from §4**, replacing every number with one you measured. Keep the four
   never-cut beats intact: the mechanism in field-ID terms, the arithmetic derivation, the ten-line
   reproduction, and the validation section with the synthetic ground-truth result.

3. **Run the synthetic ground-truth test and publish the actual result**, including the tolerance
   and what each failure value would have meant. This is the section that separates your post from
   every other post on this topic.

4. **Publish the bug you shipped first.** Write the `avg_over_time` section with the real
   overstatement factor from your own bursty namespace. If you did not make that mistake, write
   whichever one you did make instead — the beat is "here is where I was wrong," not that specific
   bug.

5. **Check the `SM_ACTIVE` ≠ MFU distinction** in your own draft, line by line. If you use the word
   MFU anywhere, either compute it properly from your workload's FLOPs-per-token or replace it with
   the field you actually measured.

6. **Outline the two secondaries** to one-paragraph-per-section depth. For secondary B, get the
   citation precision right: which reliability figures are measured and which are projections.

7. **Script and record the demo** on §7's grid. Rehearse the 2:00 cut until it lands cleanly — that
   single transition is the thesis made visible. Watch the recording once with the sound off: if the
   argument is not legible from the screen alone, the beats are wrong.

8. **Run the scrub.** No employer figures, no internal names, no negotiated rates; every rate dated
   with its basis; every simulated result labelled as simulated *first*.

9. **Get one hostile read.** Ask someone who will genuinely try to break it. Every hole they find is
   either a missing beat or a number you cannot defend.

**Acceptance:** a published post containing all four never-cut beats · your own measured exhibit
with the fields named by ID · a ten-line reproduction a stranger can run · a published validation
section including the synthetic ground-truth number and tolerance · at least one documented mistake
of your own · two secondary outlines · a recorded five-minute demo with no code-scrolling and a
clean payoff cut.

## Common pitfalls

1. **Conflating `SM_ACTIVE` with MFU.** **Mechanism:** they measure different things at different
   layers — one is a hardware occupancy ratio, the other a workload throughput ratio requiring the
   model's FLOPs-per-token. **Symptom:** a headline like "95% util, 31% MFU" when what you measured
   was a DCGM field. **Consequence:** a reader who works in the field stops reading, and the rest of
   your rigour never gets seen. **Fix:** name fields by ID and say which layer owns each number.

2. **A suspiciously clean reveal.** **Mechanism:** real measurements have noise and a window; a
   round pair of figures with no sample period, no workload description and no variance is what a
   fabricated result looks like. **Symptom:** "95% and 31%" with no context. **Fix:** state the
   window, the workload, the observed range, and whether the transcript is captured or
   representative.

3. **Ending on the reveal.** **Mechanism:** a post that stops at "look how bad this number is" is a
   complaint; the fix section is what proves you solved the problem rather than merely finding it.
   **Symptom:** no alerting rules, no before/after. **Fix:** beats 10 and the rollout table.

4. **Replacing one over-trusted number with another.** **Mechanism:** `SM_ACTIVE` is breadth, not
   productivity — a stalled warp counts as active. Presenting it as *the* truth reproduces the
   original error one level up. **Symptom:** an alert that fires on a well-tuned memory-bound
   service. **Fix:** two metrics, two claims, two owners, stated explicitly.

5. **Building the "real utilisation" panel on `GR_ENGINE_ACTIVE`.** **Mechanism:** field 1001 is a
   presence duty cycle wearing a `PROF` prefix — context bound and a pipe busy — and it ships
   enabled. **Symptom:** you "fixed" the dashboard, the number is still ~1.0 on an idle-but-resident
   decode server, and you conclude the whole argument was overstated.

6. **Treating a missing series as zero.** **Mechanism:** the exporter drops blank values entirely,
   so a permissions failure produces *absence*, and `avg by (namespace)` silently skips the GPU
   rather than dragging the average down. **Symptom:** a healthy-looking fleet average while half
   the cards stopped reporting. **Fix:** the absence alert, published in the post.

7. **Narrating code in the demo.** **Mechanism:** file layout is plumbing, not a decision, so a
   reviewer watching you scroll YAML learns nothing about how you reason. **Symptom:** three of your
   five minutes are a repository tour. **Fix:** every minute advances the argument; the payoff cut
   at 2:00 is non-negotiable.

8. **Plotting both series on one axis without dividing by 100.** **Mechanism:** field 203 is an
   integer percentage and the profiling fields are ratios in [0,1], so 99 against 0.16 flattens the
   honest series into the floor. **Symptom:** your payoff shot shows one line and a flat trace.

9. **Attacking the project instead of the default.** **Mechanism:** "the honest metric ships
   commented out" is a checkable observation about a file; "this tool is bad" is an opinion that
   loses an argument with a maintainer and reads as an inability to distinguish a defect from a
   scope boundary. **Symptom:** a comment thread you cannot win.

10. **Using real employer figures.** **Mechanism:** the argument holds identically on
    synthetic-but-realistic numbers, so the exposure buys nothing. **Symptom:** a post you have to
    take down.

## Self-check

- **Why is this a stronger flagship than "how I built my GPU cost exporter"?** *Answer:* three
  properties, all required. It is contrarian — it contradicts a number everyone trusts, and a claim
  strong enough to be wrong earns attention a build log never will. It is quotable — the thesis
  compresses to one sentence a reader can repeat: GPU utilisation measures whether a kernel was
  resident, not whether the silicon did work. And it is load-bearing on the specific competency a
  GPU-fleet operator screens for. A build post proves you can build; this proves you know something
  the reader's own team probably does not, which is what routes you to a loop. A fourth property
  makes it survive: it is reproducible in fifteen minutes on any GPU.

- **State precisely what `DCGM_FI_DEV_GPU_UTIL` measures, and why a batch-1 decode server pins it.**
  *Answer:* field 203 is an unmodified passthrough of NVML's `nvmlUtilization_t.gpu` — the
  percentage of a driver-chosen sample window (1 s to 1/6 s, product-dependent) during which at
  least one kernel was resident on the device. It is a threshold at one, with no notion of how many
  SMs exist or how full they are. Batch-1 decode emits an unbroken stream of tiny kernels — one
  matrix-vector product per weight matrix plus attention over the KV cache — enqueued back to back
  on one stream, with inter-kernel gaps of microseconds against a window floor of about 167 ms, so
  the predicate is true for essentially the whole window. Meanwhile each kernel is waiting on HBM:
  16 GB of weights at 3.35 TB/s is 4.78 ms against 16 GFLOP at 989 TFLOP/s, which is 16 µs. Presence
  is 100%; work is under 1%.

- **Which four beats can never be cut, and what does each defend against?** *Answer:* the mechanism
  (beat 2) defends against "you misread the metric" — you quote the struct comment and name the
  field ID. The derivation (beat 3) defends against "your exporter was broken" — arithmetic that
  predicts the result in advance cannot be a measurement artefact. The reproduction (beat 5) defends
  against "works on your machine" — ten lines and any GPU. The validation (beat 9) defends against
  "how do you know your measurement is right" — five checks including a synthetic ground-truth run
  with a stated tolerance. Those four convert a provocation into a citation; everything else is
  compressible.

- **How do `SM_ACTIVE` and MFU differ, and why does the platform layer usually publish the former?**
  *Answer:* `SM_ACTIVE` is field 1002, a hardware-counter ratio of summed active cycles over summed
  elapsed cycles averaged across all SMs — a breadth measure, so a kernel occupying 32 of 132 SMs
  reads 0.24. MFU is a workload-level ratio of achieved model FLOPs to the hardware's peak FLOPs,
  derived from tokens or samples per second and the model's FLOPs per token; it was introduced in
  the PaLM paper precisely because the field needed a throughput number a memory-bound kernel could
  not game. The platform layer publishes `SM_ACTIVE` because it needs nothing but the counter,
  whereas MFU needs the model's arithmetic, which lives with the tenant. So the honest platform-side
  claim is the occupancy claim — you allocated a GPU and almost nothing was scheduled on it — and
  MFU belongs in the post as calibration (roughly 38–43% for Llama 3.1 pretraining on 16,384 H100s),
  not as your headline.

- **Why publish a mistake you made, and which one?** *Answer:* because technical readers trust an
  author who shows their own error more than one presenting a clean result — a clean result is
  exactly what a fabricated one looks like — and because you cannot be "caught" on a bug you
  published. The specific one is the integration bug: `avg_over_time(x[24h]) * 24` averages only
  over samples that exist, so a job running 9 hours of a 24-hour window has its mean extrapolated
  across the whole window, overstating by exactly `window ÷ time-present` (2.67× in that case). The
  direction matters too — it makes the fleet look more utilised and the waste look smaller, on
  precisely the bursty workloads where waste is worst, so the naive query understates the very
  problem the post exists to expose.

- **What is the demo's payoff shot and what must be true for it to land?** *Answer:* the cut at
  about 2:00 from the stock green utilisation panel to the honest panel showing `SM_ACTIVE` 0.16 and
  tensor activity 0.011 for the *same GPU over the same time range*. Three things must be true: both
  series are on one axis with `GPU_UTIL` divided by 100, or the honest series flattens into the
  floor; the time ranges are visibly identical, or the comparison is arguable; and you narrate the
  arithmetic that predicted it rather than only pointing at it. Not one frame of the five minutes
  should be spent scrolling code — file layout is plumbing, and a reviewer learns nothing about your
  reasoning from a repository tour.

- **What are you explicitly not claiming, and why does saying so make the post stronger?**
  *Answer:* four things. That per-tenant attribution under time-slicing is possible — it is not, for
  anyone, because the profiling counter is device-scoped rather than context-scoped. That a MIG
  slice is a whole GPU for accounting purposes — a `1g.10gb` is one seventh of an H100 and must be
  costed as such. That published industry utilisation ranges are constants — they are directional
  order-of-magnitude context and your own measured number is the one you stand behind, which matters
  more because yours came out *better* than the range usually quoted. And that a simulated fleet is
  a production fleet. Stating these makes the post stronger because a skeptical reader's job is to
  find the boundary; arriving with the boundary already mapped, with the mechanism for each,
  converts their strongest probe into your evidence.

## Connections & what's next

This lesson is the public-facing counterpart to lesson 03's written portfolio: the same Context and
Risks material that produced the design doc's attribution hole produces the post's decomposition and
caveats, aimed at a reach-oriented audience instead of a review board, and the same architecture
diagram opens both the README and the demo's first forty-five seconds. It also arms the rest of the
module directly — the post's spine (mechanism → derivation → exhibit → validation → fix) is the same
shape lesson 06's debugging drills and lesson 07's artifact narration ask you to produce out loud,
under time pressure, without slides. The objection map in §3 becomes the follow-up bank you rehearse
against in lessons 05 and 07.

Next: **lesson 05** moves from published artifacts to live system-design drills — the same
volunteered numbers, tradeoffs and failure honesty this post demonstrates in writing now have to
hold up in real time, on a prompt you have not seen.

## References & further reading

**Primary sources**

- NVIDIA `dcgm-exporter`, `etc/default-counters.csv` — https://github.com/NVIDIA/dcgm-exporter/blob/main/etc/default-counters.csv — read for: which fields ship enabled and which ship commented out. The single strongest exhibit in the post.
- NVIDIA `dcgm-exporter`, `grafana/dcgm-exporter-dashboard.json` — https://github.com/NVIDIA/dcgm-exporter/blob/main/grafana/dcgm-exporter-dashboard.json — read for: the eight-panel default dashboard with no SM-breadth panel; published as grafana.com dashboard 12239.
- NVIDIA DCGM, `dcgmlib/dcgm_fields.h` — https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/dcgm_fields.h — read for: the exact field definitions quoted in beats 2 and 6, the numeric IDs, and the DCGM 4.x renaming aliases.
- NVIDIA `go-nvml`, `gen/nvml/nvml.h` — https://github.com/NVIDIA/go-nvml/blob/main/gen/nvml/nvml.h — read for: the `nvmlUtilization_t` struct comment that *is* the post's central quote, plus the MIG-unsupported and ECC-scrubbing notes.
- `NVIDIA/dcgm-exporter` issue #34 — https://github.com/NVIDIA/dcgm-exporter/issues/34 — read for: the documented `SYS_ADMIN` requirement for profiling metrics and the half-populated-dashboard failure mode behind the absence alert.
- Chowdhery et al. — *PaLM: Scaling Language Modeling with Pathways* (arXiv:2204.02311) — https://arxiv.org/abs/2204.02311 — read for: the origin of Model FLOPs Utilization and why a gaming-resistant throughput metric was needed. Cite this when you use the term.
- Meta — *The Llama 3 Herd of Models* (arXiv:2407.21783) — https://arxiv.org/abs/2407.21783 — read for: the ≈38–43% MFU calibration point on 16,384 H100s.
- Meta — *Revisiting Reliability in Large-Scale Machine Learning Research Clusters* (arXiv:2410.21680) — https://arxiv.org/abs/2410.21680 — read for: secondary post B's anchor. Note carefully which figures are measured (7.9 h MTTF at 1,024 GPUs) and which are projections.

**Course-internal sources — every number in the draft**

- `platform-eng/modules/05-gpu-observability/lessons/01-lie-and-truth.md` — field semantics, the two collection paths, the batch-1 derivation, the `dcgmi dmon` transcript, the alerting rules, and the batching before/after table.
- `platform-eng/modules/05-gpu-observability/lessons/08-capstone-allocated-vs-utilised.md` — the ratio-to-hours integration and its bug, the six-bucket decomposition, the five validation checks, and the spin-kernel and vLLM reproduction recipes.

**Not relied upon**

- Vendor and third-party blog posts making versions of the "GPU utilisation is misleading" argument
  were consulted as evidence that the genre exists and is proven. They are not cited for any
  specific number in the draft, and no figure above is taken from them — every measurement in §4
  traces to this course's own module 05 work or to a primary source listed above.
