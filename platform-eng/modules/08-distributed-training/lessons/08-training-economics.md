---
lesson: "08.8"
title: "Training economics"
module: "08"
concept: "Training economics"
status: not-started
est_time: "7h"
prev: "07-data-pipeline.md"
next: null
artifacts: []
sources: 16
---

# 08.8 · Training economics

> **Concept.** Turn the reliability and efficiency math of this module into one dollar figure: the true cost of a successful run.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Where this fits

This is the capstone, and it is the lesson every other lesson in the module feeds. 08.1–08.2
gave you the parallelism and collective vocabulary; 08.3 gave you MFU as the efficiency
report card; 08.4 gave you the checkpoint-interval math; 08.5 gave you the failure and
restart loop with its detection signals; 08.6 gave you the objects that express, admit and
restart the job — and, crucially, whether a job waiting in a queue is holding GPUs while it
waits; 08.7 gave you a third, usually unlabelled source of wasted GPU-hours. None of those
produce a number anyone outside the platform team can act on.

This one does. By the end you should be able to walk into a planning meeting with a single
model of what a training run costs, defend every term in it, say which two or three terms
actually move the answer at *your* scale, and price a specific engineering investment
against them.

It also hands off forward. Module 11 (GPU cost and unit economics) applies the same shape of
thinking across a whole fleet rather than one run, and this lesson is deliberately written to
compose with it: the run-level number produced here is exactly the *allocated* GPU-hours that
[11.5 · Unit economics](../../11-gpu-cost-economics/lessons/05-unit-economics.md) loads with
its multiplier `L = (P/A)(1+f)`. §7 makes that join explicit, including the double-count you
must not commit at the seam.

## Why this matters

The naive number — `GPU-hours × $/GPU-hr` — is wrong in three directions at once, and each
one is worth more than most cost-optimisation projects.

It ignores **efficiency**: a run at 35% MFU burns 29% more GPU-hours than the identical run
at 45%. It ignores **reliability**: Meta's own study of 4 million jobs across two production
research clusters measured a **mean time to failure of 7.9 hours for 1,024-GPU jobs**, and
projects **1.8 hours at 16,384 GPUs**. And it ignores **what the company actually pays** —
fragmented capacity nobody could hold, the fabric, the checkpoint tier, the dataset tier.

The gap between the naive number and the real one is precisely the amount your reliability
and efficiency engineering is worth, which is exactly the thing you want to be able to price
when someone asks whether the work was justified. It is also, in the other direction, the
number that stops a bad decision: "this neocloud is 40% cheaper per GPU-hour" is not an
argument until you know what its failure rate and its queue wait do to the multiplier.

**The differentiator this lesson gives you** is not the arithmetic. It is being the person in
the room who can say: *"at our scale the reliability term is worth 3% and MFU is worth 22%,
so we should not spend the quarter on faster restart — but at 16k GPUs that flips, and here
is the GPU count where it flips."*

## What's new here (calibration)

- **One model, assembled, with every term attributed to the lesson that derived it.** We do
  not re-derive Young/Daly (08.4 owns it), MFU's roofline (08.3), the failure loop (08.5),
  gang admission economics (module 06 / 08.6) or starvation (08.7). We assemble them and
  then do the thing none of them do: differentiate the result to find out which terms matter.
- **Real failure rates, measured rather than extrapolated.** Meta's 11-month, 150M-GPU-hour
  study gives an empirical MTTF-versus-scale curve. **An earlier version of this lesson
  extrapolated a single Llama-3 datapoint down to 1,024 GPUs and got 45 hours; the measured
  figure for that scale is 7.9 hours** — a 5.7× error in the input that dominates the
  reliability term. §3 explains why both numbers are real and which one you should use.
- **Queue time as a first-class term**, because Meta's ETTR definition includes it and
  because 08.6 showed that whether a waiting gang is *billed* depends on which admission
  layer you deployed. This is the term that decides the spot-versus-on-demand question in §9.
- **Goodput**, defined precisely against three different time bases, with the
  `$ per useful GPU-hour` figure that follows — the single most persuasive number in the
  whole module.
- **Sensitivity done properly**, as log-elasticities rather than one-off what-ifs, so you can
  read straight off the model which lever to fund, and compute the GPU count at which the
  answer changes.
- **The costs the model does not contain** — failed experiments, engineering time, data
  preparation — named explicitly in §10, because a model whose boundaries you cannot state
  will not survive a planning meeting.

## Core concepts

### 1. Three time bases, and which one you are billed for

Before any formula, fix the vocabulary. A run occupies time in three different senses and
almost every confused cost conversation is two people using different ones.

```
  THREE TIME BASES FOR ONE RUN
  ═══════════════════════════════════════════════════════════════════════════════

  WALL-CLOCK  W_wall   submission ──────────────────────────────▶ model exists
              ├──────────────┬────────────────────────────────────────────┤
              │  QUEUE       │  ALLOCATED                                 │
              │  waiting for │  the GPUs are yours                        │
              │  admission   │                                            │
              └──────────────┴────────────────────────────────────────────┘
                     ▲                        │
                     │                        ├─ productive compute      T_p
   BILLED?           │                        ├─ checkpoint stalls       T_ckpt
   depends on the    │                        ├─ redone work after crash T_redo
   admission layer   │                        ├─ restart / re-warm       T_restart
   (08.6 §6):        │                        └─ in-run idle (starvation) → folded
     Kueue/Volcano   │                                                      into MFU
       hold NOTHING  │
     raw coscheduling│
       holds GPUs at Permit  ⇒ queue time IS billed

  ALLOCATED  A_run  = the hours the cost system charges you.
                      This is the base for every dollar figure below.
  PRODUCTIVE T_p    = hours spent executing the training step, checkpoint-free,
                      failure-free.
```

**Bill on allocated hours.** That is the ledger module 11 lesson 02 establishes and the one
every cloud invoice uses: you pay for a GPU from the moment it is bound to your pod until it
is released, whether or not a kernel is running on it. Everything below computes `A_run` and
then multiplies by a rate.

The queue box is the subtle one and it is a direct consequence of 08.6. A gang parked at the
coscheduling plugin's `Permit` extension point holds real, assumed capacity on real nodes for
up to `scheduleTimeoutSeconds`. A Kueue Workload waiting for quota holds nothing — no pods
exist. **Same wait, same wall-clock, different invoice.** Meta's ETTR metric counts queue
time as unproductive regardless; your cost model should count it as *billed* only when the
admission layer holds capacity.

### 2. Layer 1 — the hours the work needs

The useful work of a pretraining run is a fixed FLOP budget, independent of how well you run
it. For a dense transformer (08.1):

```
  C = 6 · N_params · N_tokens          [FLOP]      forward 2ND + backward 4ND
```

Hardware time to execute it depends on the fraction of peak you sustain — **MFU**, from 08.3:

```
  GPU-hours_productive = C / (P_peak · MFU · 3600)          [GPU-hours]
                       ↑ note: INDEPENDENT of the GPU count.
                         Adding GPUs buys wall-clock, not GPU-hours —
                         until communication overhead starts eating MFU (08.3),
                         at which point adding GPUs costs you GPU-hours.

  W_productive = GPU-hours_productive / G                    [wall-hours]
```

Two facts to carry. First, **GPU-hours are inversely proportional to MFU**, so this is the
one term with a clean 1:1 cost elasticity and it is why 08.3 came before everything else.
Second, MFU is a *composite*: an exposed all-reduce (08.3), a starved input pipeline (08.7),
a bad kernel launch pattern and low occupancy all show up as the same depressed number. The
cost model does not care which; your diagnosis work in 08.3 and 08.7 is what tells you which
one to attack.

Reference constants: H100 SXM peak **BF16 dense ≈ 989 TFLOP/s** (the datasheet's 1,979
TFLOPS figure is quoted *with* 2:4 structured sparsity — halve it for dense work, a
2× error people make constantly). Published LLM pretraining runs typically sustain
**35–50% MFU**; Llama-3's 405B run reported 38–43%.

### 3. Layer 2 — the failure-overhead multiplier, with real inputs

A real run does not spend all of its allocated hours on `C`. Four leaks, all measured against
productive time `T_p`:

| Leak | Time cost over `T_p` | As a fraction of `T_p` | Owned by |
|---|---|---|---|
| Checkpoint **stalls** | `(T_p/τ) · δ` | `δ / τ` | 08.4 |
| **Redone** work after a crash | `(T_p/M) · (τ/2)` | `τ / (2M)` | 08.4 |
| **Restart / recovery** per crash | `(T_p/M) · R` | `R / M` | 08.5, 08.6 |
| **Billed queue** per restart | `(T_p/M) · q` | `q / M` | 06, 08.6 |

where `τ` = checkpoint interval, `δ` = the *blocking* portion of one checkpoint (not the
total write time — asynchronous sharded checkpointing decouples them, which is the whole
point of 08.4), `M` = mean time between failures **for this job at this GPU count**, `R` =
detect + drain + reschedule + reload + re-warm, and `q` = the billed part of the wait for
re-admission. Redone work is `τ/2` on average because a crash lands uniformly within an
interval.

Minimising `δ/τ + τ/(2M)` over `τ` gives **Young/Daly** (08.4), and substituting the optimum
back collapses two terms into one:

```
  τ*        = √(2 · M · δ)
  f_ovh     = √(2δ/M)  +  (R + q)/M
               └ checkpoint + redo ┘   └ recovery ┘

  A_run     = GPU-hours_productive × (1 + f_ovh)
  ETTR      = 1 / (1 + f_ovh)              ← "effective training time ratio"
```

Note the scaling exponents, because the whole scale story is in them: **the checkpoint term
falls off as `M^{-1/2}` and the recovery term as `M^{-1}`.** Since `M` shrinks roughly with
GPU count, recovery overtakes checkpointing as you scale. §8 computes exactly where.

**Now the inputs, which is where this lesson diverges from most treatments.**

`M` is not something to guess or to extrapolate from one headline. Meta's *Revisiting
Reliability in Large-Scale Machine Learning Research Clusters* analysed 11 months of two
production clusters — **4 million jobs, >150 million A100 GPU-hours** — and fitted a failure
model across scales:

| Job size | MTTF | Source |
|---|---|---|
| 8 GPUs | **47.7 days** (≈1,145 h) | measured |
| 1,024 GPUs | **7.9 hours** | measured |
| 16,384 GPUs | **1.8 hours** | model projection |
| 131,072 GPUs | **0.23 hours** (14 min) | model projection |

Two things to notice, and both matter for how you use this table.

**It is not a clean `1/N` law.** From 8 → 1,024 GPUs (128×), a pure inverse law predicts
8.9 h and the measurement is 7.9 h — slightly worse than linear. From 1,024 → 16,384 (16×) it
predicts 0.49 h and the model says 1.8 h — considerably better than linear. From 16,384 →
131,072 it is almost exactly linear. **Do not extrapolate this curve naively.** Use the
nearest measured point, or better, measure your own: `M = allocated_GPU_hours / interruptions`
over a month is a two-line PromQL query and it is the single highest-value input in the model.

**The same organisation reports two very different reliability populations, and both are
real.** Llama-3's 405B run saw **466 interruptions over 54 days on 16,384 H100s — 47 planned
and 419 unexpected, with roughly 78% attributed to hardware** — which is one unexpected
interruption every `1,296 / 419 = 3.1 hours`, against the research-cluster model's 1.8 h
projection at that scale. A flagship run on dedicated, burned-in, heavily instrumented
capacity is more reliable than the median job on a shared multi-tenant research cluster. When
you use a number from a published frontier run, you are quoting the *best* case; when you use
a fleet study, you are quoting the *median*. Say which one you used.

For `R`, the number that surprises people is that **detection usually dominates recovery**. A
crashed process is detected in seconds. A *hang* — the 08.2 failure mode where every rank sits
at 100% GPU utilisation inside a collective that will never complete — is detected when a
watchdog fires, and NCCL's default collective timeout is measured in *tens of minutes*. A shop
that relies on the default timeout for hang detection has an `R` an order of magnitude larger
than one that runs an active heartbeat. §8 prices that difference and it is startling.

### 4. Layer 3 — the rate, which is not a constant

`r` looks like a scalar you plug in. It is three decisions.

**Blended versus marginal** (module 11 §7). The *blended* rate is what capacity costs you on
average across your commitment mix:

```
  r_blended = Σ_i (share_i × rate_i)
  e.g. 70% committed at $1.90 + 30% on-demand at $3.20 ⇒ $2.29/GPU-hr
```

Use it for steady-state planning and for any number that goes on a price sheet. The
*marginal* rate is what one more GPU-hour costs — and on a fleet with committed capacity
sitting idle, the marginal rate of running an extra experiment on it is close to **zero**.
That distinction decides "should we run this ablation": on blended it looks like $8k, on
marginal it is free. Both are correct answers to different questions; quoting one as the
other is how planning meetings go wrong.

**On-demand versus committed versus spot.** The observed 2026-08 band for on-demand H100 SXM
runs roughly **$1.4–$7/GPU-hr** on specialist GPU clouds and up to **~$7–12/GPU-hr** at
hyperscaler list price; module 11 uses a blended **$2.99/GPU-hr** for the same hardware.
Committed pricing is lower; spot is lower still and changes the failure rate, which is why §9
treats it as a modelling question rather than a procurement one.

**Sticker rate is not total cost.** Two clusters quoting the same `$/GPU-hr` can differ
materially in delivered cost through node health, network quality, time-to-first-job, and
support responsiveness — all of which show up in this model as `M`, `R` and MFU rather than as
`r`. That is not a hand-wave: it is the observation that **cluster quality enters the formula
through three different terms, none of them the price**, and it is the correct way to
interrogate a vendor claim.

### 5. Layer 4 — assemble, and see what the run costs

```
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │  $_direct(run) =   6·N·D                                                      │
  │                 ─────────────  ×  r  ×  ( 1 + √(2δ/M) + (R+q)/M )             │
  │                 P_peak · MFU                                                  │
  │                 └─ 08.1, 08.3 ┘    ↑        └──── 08.4, 08.5, 08.6 ────┘      │
  │                  GPU-hours the      FinOps      the reliability multiplier    │
  │                  work needs         rate                                      │
  └──────────────────────────────────────────────────────────────────────────────┘

  $_loaded(run)  =  $_direct  ×  L,        L = (P/A)(1+f)        ← module 11 §6
```

Every lesson in the module is in that expression. The two brackets are *multiplicative*, so
an MFU win and a reliability win compound rather than add.

The factor tree, so you know where each number comes from and who owns it:

```
  $ PER RUN
  ═══════════════════════════════════════════════════════════════════════════════
  $_loaded
   │
   ├── L = (P/A)(1+f) ─────────────── platform loading .............. module 11 §6
   │      ├── P/A  fragmentation, cordoned, unallocatable ........... module 11 L04
   │      └── 1+f  fabric, checkpoint tier, dataset tier, control plane, obs
   │
   └── $_direct
        │
        ├── r   $/GPU-hr, blended on a stated commitment mix ........ module 11 §7
        │
        ├── (1 + f_ovh)   reliability multiplier
        │      ├── √(2δ/M)   δ = blocking checkpoint cost ........... 08.4
        │      │             M = MTBF at THIS GPU count ............. 08.5 + measurement
        │      ├── R/M       R = detect + reschedule + reload ....... 08.2, 08.5, 08.6
        │      └── q/M       q = BILLED queue per restart ........... 06, 08.6 §6
        │
        └── GPU-hours = 6·N·D / (P_peak · MFU)
               ├── 6·N·D     the fixed FLOP budget .................. 08.1
               ├── P_peak    989 TFLOP/s BF16 dense on H100 SXM ..... 03
               └── MFU       35–50% typical, and itself composed of:
                     ├── exposed communication ...................... 08.3
                     ├── input starvation ........................... 08.7
                     └── occupancy / kernel efficiency .............. 03
```

### 6. Goodput: the number to put on the slide

Three ratios, each answering a different question, and the third is the one that changes
minds.

```
  MFU      = useful tensor-core FLOP time / PRODUCTIVE time      "how well does a step run?"
  ETTR     = productive time / ALLOCATED time                    "how much of what we rent is
                                                                  spent on steps?"
  GOODPUT  = MFU × ETTR = useful FLOP time / ALLOCATED time      "how much of what we PAY FOR
                                                                  becomes model?"

  $ per useful GPU-hour  =  r / GOODPUT
```

MFU is an ML-engineering metric; ETTR is an SRE metric; **goodput is the finance metric**,
and `r / goodput` converts it into the only unit a budget owner reasons in. At the base case
of §8 — MFU 0.40, ETTR 0.919 — goodput is 0.368, so a $2.99/GPU-hr rate delivers useful work
at **$8.13 per useful GPU-hour**, and $10.91 fully loaded. That sentence — *"we pay $2.99 and
we get $10.91"* — is the entire module in eleven words, and it is a sentence a CFO can act on.

Note the audience translation, because you will need it: **the SRE's "we're at 92% effective
training time" and the CFO's "there's a 1.09× reliability multiplier on every run" are the
same number.** Being able to move between them mid-sentence is what makes the model useful in
a room with both people in it.

### 7. Joining to the fleet model without double-counting

This is the seam where run-level and fleet-level cost models are most often wired together
wrongly, so state it precisely.

Module 11's loading multiplier is `L = (P/A)(1+f)`, where `P` is the fleet's **paid**
GPU-hours, `A` its **allocated** GPU-hours, and `f` non-GPU spend as a fraction of GPU spend.
`P/A` recovers *platform* inefficiency — fragmentation, cordoned devices, capacity nobody
could hold. `(1+f)` recovers the fabric, storage and control plane. Module 11 is explicit that
the widely circulated `L = 1/utilisation` is **wrong** under a charge-allocated policy,
because a tenant's own idle hours are already inside `A` and are already paid for; loading
them again charges the same waste twice, and on module 11's own worked fleet that mistake
inflates a unit cost by 78%.

The same trap appears here in a slightly different costume, and it is worth naming as a rule:

> **`(1 + f_ovh)` and `L` recover disjoint waste. Never let them overlap.**
>
> `(1 + f_ovh)` is *this run's* own unproductive hours — its checkpoint stalls, its redone
> work, its restarts. Those hours are **inside** `A_run`: the scheduler allocated them to you
> and the invoice contains them. `L` recovers what the *platform* wasted around you.
> Multiplying `$_direct` by `L` is correct. Multiplying it by `1/ETTR` **on top of**
> `(1 + f_ovh)` is the same double-count module 11 warns about, wearing a training hat.

The practical consequence: `A_run` from §3 is precisely the quantity that enters module 11's
`H_attr`. The run-level model ends at `$_direct`; the fleet model takes it from there. And
the split of responsibility is the same as module 11's rule of thumb — **optimise on direct,
price on fully loaded, and never quote one as the other.**

One training-specific note on `f`. Module 11's worked fleet measured `f = 0.09`. For a
training platform the components are checkpoint storage (08.4 — high-bandwidth, short
retention, sized by `δ`), dataset storage (08.7 §4 — where the "buy capacity to get
bandwidth" effect on a per-TiB-provisioned parallel filesystem lives), the InfiniBand or
RoCE fabric, and observability. **Measure yours.** A 3D U-Net-class workload needing 25 GB/s
per node buys a very different storage tier from an LLM run needing 34 MB/s per *cluster*,
and the difference lands in `f`.

### 8. The waterfall: where every paid GPU-hour goes

```
  PAID GPU-HOURS → USEFUL FLOP TIME
  1,024 × H100 · 8B params · 2T tokens · MFU 0.40 · M 7.9 h · δ 0.01 h · R 0.25 h · q 0.05 h
  ═══════════════════════════════════════════════════════════════════════════════════════

  ALLOCATED (billed)                                            73,362 GPU-h   100.0%
  ████████████████████████████████████████████████████████████████████████████
   │
   ├─ restart + billed queue    8.33 failures × 0.30 h × 1024 =  2,560 GPU-h     3.5%
   │  ███                                                                        (R+q)/M
   ├─ redone work               8.33 failures × τ*/2  × 1024  =  1,696 GPU-h     2.3%
   │  ██                                                                         τ/2M
   ├─ checkpoint stalls         165.6 writes × 0.01 h × 1024  =  1,696 GPU-h     2.3%
   │  ██                                                                         δ/τ
   │                                                             ───────────
   └─ PRODUCTIVE (steps executing)                              67,410 GPU-h    91.9%  ← ETTR
      ███████████████████████████████████████████████████████████████████
       │
       │   ── the MFU decomposition below is ILLUSTRATIVE: the 40% is the
       │      model's input, the split of the other 60% must be MEASURED
       │      per run with 08.3's profiling and 08.7's attribution ──
       │
       ├─ exposed communication          ~18%                  ~12,100 GPU-h    16.5%   08.3
       │  ██████████
       ├─ input starvation / H2D gaps     ~9%                   ~6,100 GPU-h     8.3%   08.7
       │  █████
       ├─ memory-bound ops, launch gaps  ~33%                  ~22,200 GPU-h    30.3%   03
       │  ████████████████
       └─ USEFUL TENSOR-CORE FLOP TIME    40%                   26,964 GPU-h    36.8%  ← GOODPUT
          ████████████████████

  GOODPUT = MFU × ETTR = 0.400 × 0.919 = 0.368
  $ per allocated GPU-hour     r                     = $2.99
  $ per USEFUL GPU-hour        r / goodput           = $8.13
  $ per USEFUL GPU-hour loaded r / goodput × 1.3416  = $10.91
```

**Read the waterfall as a prioritisation.** The three reliability bands together are 8.1% of
the bill. The three MFU bands together are 55.1%. At this scale, an hour of engineering spent
on MFU is worth roughly seven times an hour spent on reliability — which is the opposite of
where most platform teams' instincts point, and it is why you do this calculation before
choosing a quarter's work rather than after.

### 9. Sensitivity: which terms actually move the answer

Do this as **log-elasticities** — `∂ln($)/∂ln(x)`, "a 1% change in x moves cost by e%" —
because that makes every term directly comparable regardless of units.

From the assembled model, three terms are trivially unit-elastic:

```
  ε(MFU) = −1.00        ε(r) = +1.00        ε(L) = +1.00
```

The reliability terms are not, and their elasticities depend on where you are:

```
  f_ovh = √(2δ/M) + K/M ,  K = R + q

  ε(M) = −[ ½·√(2δ/M) + K/M ] / (1 + f_ovh)
  ε(δ) = + ½·√(2δ/M)          / (1 + f_ovh)      ← SUB-LINEAR: halving δ cuts the
  ε(R) = +   R/M              / (1 + f_ovh)         checkpoint term by only 29%
  ε(q) = +   q/M              / (1 + f_ovh)
```

Evaluate at two scales, holding `δ = 0.01 h`, `R = 0.25 h`, `q = 0.05 h`:

```
  TORNADO — ELASTICITY OF $/RUN  (longer bar = bigger lever)
  ═══════════════════════════════════════════════════════════════════════════════

                       1,024 GPUs (M = 7.9 h)        16,384 GPUs (M = 1.8 h)
                       f_ovh = 0.088, ETTR 0.919      f_ovh = 0.272, ETTR 0.786

   MFU        −1.000   ████████████████████████       ████████████████████████  −1.000
   rate  r    +1.000   ████████████████████████       ████████████████████████  +1.000
   loading L  +1.000   ████████████████████████       ████████████████████████  +1.000
   MTBF  M    −0.058   █▍                             ████▏                     −0.173
   restart R  +0.029   ▊                              ██▋                       +0.109
   queue  q   +0.006   ▏                              ▋                         +0.022
   ckpt   δ   +0.023   ▌                              █                         +0.041
```

**The three conclusions you take to the meeting:**

1. **MFU, the rate, and the platform loading are always the top three.** Each is unit-elastic:
   a 10% MFU improvement is a 9.1% cost reduction, full stop, at any scale. If you can move
   MFU at all, move MFU.
2. **Reliability is a rounding error at 1,024 GPUs and a first-class term at 16,384.**
   Restart's elasticity is 0.029 at the smaller scale and 0.109 at the larger — a 3.8× swing
   from nothing but GPU count.
3. **Cheaper checkpoints are always a weaker lever than faster restart**, because `δ` enters
   under a square root and `R` enters linearly. Halving `δ` buys 1.2% at 16k GPUs; halving `R`
   buys 5.5%.

**The crossover, computed.** Define "reliability is worth a meeting" as `f_ovh > 0.10`. With
`δ = 0.01` and `K = 0.30`:

```
  √(0.02/M) + 0.30/M = 0.10
    M = 7.9 h  ⇒ 0.0503 + 0.0380 = 0.0883   below threshold
    M = 6.6 h  ⇒ 0.0550 + 0.0455 = 0.1005   at threshold
    M = 5.0 h  ⇒ 0.0632 + 0.0600 = 0.1232   above

  Meta's curve puts M ≈ 7.9 h at 1,024 GPUs and M ≈ 1.8 h at 16,384.
  Interpolating, M ≈ 6.6 h lands around 1,200–1,400 GPUs.

  ⇒ Below ~1,200 GPUs, fund MFU work.
    Above ~1,200 GPUs, reliability becomes a named line item and keeps growing.
```

**And the detection number, which is the one people remember.** Everything above assumed
`R = 0.25 h` — a shop with active hang detection. Suppose instead you rely on NCCL's default
collective timeout, tens of minutes, to notice a hung rank; call it `R = 0.75 h`. At 16,384
GPUs with `M = 1.8 h`:

```
  R = 0.05 h (aggressive heartbeat)   f_ovh = 0.1054 + 0.0556 = 0.1610   ×1.1610
  R = 0.25 h (baseline)               f_ovh = 0.1054 + 0.1667 = 0.2721   ×1.2721
  R = 0.75 h (default NCCL timeout)   f_ovh = 0.1054 + 0.4444 = 0.5498   ×1.5498

  ⇒ 1.5498 / 1.1610 = 1.335
```

**A 33% swing in the cost of every run, from nothing but how fast you notice a hang.** That
is the payoff of 08.2 and 08.5, denominated in dollars, and it is the single best answer to
"why did you spend a sprint on watchdogs."

### 10. What the model deliberately does not contain

A model whose boundaries you cannot state will lose an argument to someone who can. Four
things sit outside it, and the first is the largest.

**The experiment tax.** `$_run` prices *one* run. A shipped model is the survivor of a
sequence of failed and abandoned runs, hyperparameter sweeps, and ablations. The number a
planning meeting actually wants is:

```
  $ per shipped model = Σ over attempts of $_run(attempt)
                      ≈ $_run(final) × (1 + A_ratio)

  where A_ratio = (exploration + failed + abandoned GPU-hours) / (final run GPU-hours)
```

`A_ratio` is an *organisational* number, not a technical one, and it is frequently larger than
everything this lesson models. Measure it from your own job history — it is the same PromQL
join as `M` — and quote it separately. Never present `$_run(final)` as the cost of the model.

**Engineering time.** A quarter of platform-engineering effort has a fully loaded cost. The
elasticities in §9 tell you the *return*; you supply the *cost* and compute the ratio. At the
1,024-GPU base case, a 5-percentage-point MFU improvement is worth `1 − 40/45 = 11.1%` of a
$219k run — about $24k per run. Whether that is worth a quarter depends entirely on how many
runs there are, which is why run frequency belongs in the same slide.

**Data preparation and evaluation.** Tokenisation, deduplication, filtering, shard packing
(08.7 §7) and downstream evaluation are real GPU and CPU spend that is not in `6ND`.

**Inference.** Entirely out of scope here, and the point where module 11's per-token unit
economics takes over.

### 11. Spot capacity: the decision the model is built for

This is the archetypal planning-meeting question, and it is the cleanest demonstration that
the model earns its keep. Spot capacity is cheaper per GPU-hour and less reliable. Does it
win?

Take the §8 base case at 1,024 GPUs, and suppose spot is 60% off: `r_spot = $1.196` against
`r_od = $2.99`. Preemption adds a second, independent failure process. If each of the 128
nodes has a preemption hazard with a 72-hour mean, the gang's preemption MTBF is
`72 / 128 = 0.5625 h`, and hazards compose:

```
  1/M_total = 1/M_hardware + 1/M_preempt = 1/7.9 + 1/0.5625 = 0.1266 + 1.7778
  M_total   = 0.525 h        ← preemption now dominates by 14×

  Case A — the gang is re-admitted promptly (q = 0.05 h, spare capacity available)
    f_ovh = √(0.02/0.525) + 0.30/0.525 = 0.1952 + 0.5714 = 0.7666   ×1.7666
    τ*    = √(2 × 0.525 × 0.01) = 0.1025 h  ⇒ checkpoint every 6.2 minutes
    $     = 67,410 × 1.196 × 1.7666 = $142,435       vs on-demand $219,349
    ⇒ SPOT WINS by 35%

  Case B — the spot market is tight, so each restart waits (q = 0.5 h)
    f_ovh = 0.1952 + (0.25+0.5)/0.525 = 0.1952 + 1.4286 = 1.6238   ×2.6238
    $     = 67,410 × 1.196 × 2.6238 = $211,540       vs on-demand $219,349
    ⇒ SPOT BARELY WINS — 3.6%, well inside the error bars on every input
```

**The entire spot decision is a bet on `q`.** Not on the discount, not on the preemption rate
— on how long a 1,024-GPU gang waits to be re-admitted after each preemption. And `q` is the
term module 06 and 08.6 are about: it is set by your quota headroom, by whether the admission
layer holds capacity while waiting, and by whether the job can restart at reduced width
(08.5's elasticity, `--nnodes=min:max`) rather than waiting for the full gang.

Two structural mitigations follow directly from the algebra rather than from intuition. First,
`τ*` collapses to 6 minutes under spot's failure rate — so spot capacity is only viable at all
if `δ` is small, which means asynchronous sharded checkpointing (08.4) is a *precondition* for
spot, not an optimisation. Second, an elastic job that continues at reduced width instead of
waiting turns `q` from a stall into an MFU reduction, which the model prices as a much smaller
loss.

## Perspectives

**FinOps / CFO view.** The multiplicative structure — `(1/MFU) × r × (1 + f_ovh) × L` — is
the single line that ties the module together, and `$ per useful GPU-hour` is its output. Two
independently reported real budgets are consistent with it at the magnitudes it predicts:
Databricks/MosaicML reported GPT-3-quality training achievable for roughly **$450K** and
MPT-7B trained on **440 A100s in 9.5 days for about $200K** (both 2023 snapshots), with the
low numbers attributed to *managing* MFU and failure overhead rather than to ignoring them.
Sanity-check your model against figures like these before you present it: if your model says
$2M for an MPT-7B-class run, you have an input wrong.

**SRE view.** ETTR is the SRE-facing name for the identical arithmetic the CFO sees as a
multiplier, and Meta's definition — productive runtime over productive-plus-unproductive,
*including queue* — is the one to adopt because it makes the scheduler's behaviour visible in
the reliability metric. The three unproductive categories in that definition (catch-up from
checkpoint, restart overhead, checkpoint overhead) are exactly §3's table minus queue.

**Scale-dependent-lever view.** The tornado in §9 is the argument, and the FT-HSDP result is
the empirical confirmation: at O(100,000) GPUs, restarting only the affected data-parallel
replica instead of the whole job cut recovery stall from **10 minutes to 3 minutes** and moved
effective training time from **44% to 80%**. That is a pure `R` win, at a scale where `R/M`
dominates — and note the 44% baseline, which is what fully synchronous training looks like at
that scale before the engineering. It also shows up in this module's own APIs: JobSet's
`RestartJob` action (08.6 §7) is that idea expressed as a Kubernetes failure policy.

**Procurement view.** Cluster quality enters this model through `M`, `R` and MFU, not through
`r`. That reframes the vendor conversation from "what is your hourly rate" to three answerable
questions: what is your observed node failure rate per GPU-hour, what is your time from node
failure to a replacement node being schedulable, and what fabric bisection bandwidth am I
getting. A vendor who cannot answer those is quoting you a price for an unknown quantity.

**Interview view.** "Price a training run including failure overhead" is a standard probe, and
the answer that separates candidates is not the formula — it is naming which term dominates and
why it depends on scale, then giving a real MTTF number with its source. "MTTF is about 8 hours
for a 1,024-GPU job and around 2 hours at 16k, from Meta's cluster study" is a very different
answer from "failures happen sometimes."

## Real-world use cases

- **Meta — "Revisiting Reliability in Large-Scale Machine Learning Research Clusters"**
  (arXiv:2410.21680, HPCA 2025). 11 months, two production clusters (RSC-1, RSC-2),
  **4 million jobs, >150 million A100 GPU-hours**. Introduces **ETTR** — productive runtime
  over productive plus unproductive (including queue) — and identifies exactly three sources
  of unproductive scheduled time: catching up from the last checkpoint, restart overhead, and
  checkpoint overhead. Fits a failure model giving **MTTF 47.7 days at 8 GPUs, 7.9 hours at
  1,024 GPUs**, projecting **1.8 hours at 16,384** and **0.23 hours at 131,072**. Also notes
  that hardware reliability scales roughly inversely with GPU count above about 32 GPUs, and
  that small jobs dominate cluster job *counts* even though large jobs dominate failure
  exposure. **This is the empirical base for §3 and the correction to this lesson's previous
  extrapolated MTBF.** *arxiv.org is blocked from this environment; these figures were
  confirmed via search against the paper's abstract and published summaries, not by reading
  the PDF in this pass.*
- **Meta — "The Llama 3 Herd of Models", §3.3.** 16,384 H100s, 54 days, **466 interruptions
  (47 planned, 419 unexpected)**, roughly **78% attributed to hardware** — faulty GPUs 30.1%
  and HBM3 17.2% being the two largest categories — with **>90% effective training time**
  sustained and only a handful of events requiring significant manual intervention. What it
  shows: the frontier-run best case, and the contrast with the research-cluster median that
  §3 makes explicit. *arxiv.org blocked; search-verified. Note the correction: 419 unexpected
  interruptions, not 417.*
- **Meta — "Training LLMs with Fault Tolerant HSDP on 100,000 GPUs"** (arXiv:2602.00277).
  Uses data-parallel replicas as the unit of fault tolerance: on failure only the replica
  containing the failed device is taken offline and restarted while the others continue.
  **Recovery stall falls from 10 minutes to 3 minutes; effective training time rises from 44%
  to 80%**, with no meaningful accuracy degradation. What it shows: at extreme scale the win
  came from `R`, not from checkpoint frequency — the empirical confirmation of §9's
  elasticity ordering. *arxiv.org blocked; search-verified.*
- **Databricks/MosaicML — "GPT-3 quality for <$500k"** and **"Introducing MPT-7B"** (both
  **2023 snapshots**). MPT-7B: **440 A100-40GB, ~9.5 days, ~$200K, zero human intervention**,
  with the platform detecting and handling hardware failures automatically. What it shows: a
  real, checkable magnitude to sanity-check the model against — and the framing that the low
  cost is a *consequence* of managing MFU and failure overhead. *databricks.com is blocked
  from this environment; search-verified.*
- **SemiAnalysis — "How Much Do GPU Clusters Really Cost?"** (**2026 snapshot**). An
  analyst-side TCO treatment arguing that a well-run specialist cloud can beat a hyperscaler on
  total cost even at an equal or higher nominal `$/GPU-hr`, because of hidden downtime, setup
  and tuning costs — their "goodput" framing. What it shows: an independent statement of §4's
  claim that cluster quality enters through `M`, `R` and MFU rather than through the sticker
  rate. *Blocked from this environment; not fetched in this pass, and no number from it is
  relied upon above.*

## Worked example

Build the whole thing once, end to end, with every input named. **Every figure below is
parameterised — substitute your own and re-run.**

**Inputs.**

```
  MODEL / DATA        N_params = 8×10⁹         D = 2×10¹² tokens
  HARDWARE            G = 1,024 × H100 SXM     P_peak = 989 TFLOP/s (BF16 DENSE)
  EFFICIENCY          MFU = 0.40                            [08.3, within 35–50% band]
  RELIABILITY         M   = 7.9 h                           [Meta RSC, 1,024-GPU jobs]
                      δ   = 0.01 h (36 s blocking)          [08.4, async sharded ckpt]
                      R   = 0.25 h (15 min)                 [08.5 + 08.6, with active
                                                             hang detection]
                      q   = 0.05 h (3 min billed queue)     [08.6, Kueue with headroom]
  RATE                r   = $2.99/GPU-hr blended            [2026-08 snapshot; band
                                                             $1.4–7 specialist,
                                                             $7–12 hyperscaler list]
  PLATFORM LOADING    P/A = 1.2308   f = 0.09   ⇒ L = 1.3416   [module 11 worked fleet]
```

**Step 1 — the FLOP budget and the hours it needs.**

```
  C = 6 × 8×10⁹ × 2×10¹²                 = 9.60×10²²  FLOP
  GPU-seconds = C / (P_peak × MFU)
              = 9.60×10²² / (9.89×10¹⁴ × 0.40)
              = 9.60×10²² / 3.956×10¹⁴   = 2.4267×10⁸  GPU-s
  GPU-hours_productive                    = 67,410  GPU-h
  W_productive = 67,410 / 1,024           = 65.83 h   = 2.74 days
```

**Step 2 — the checkpoint interval, from Young/Daly.**

```
  τ* = √(2 · M · δ) = √(2 × 7.9 × 0.01) = √0.158 = 0.3975 h = 23.9 minutes
  checkpoints over the run = 65.83 / 0.3975 = 165.6
```

Sanity-check `δ` against the hardware rather than assuming it. A distributed sharded
checkpoint of an 8B model at 16 bytes/parameter (bf16 weights + fp32 master + Adam `m` and
`v`) is `8×10⁹ × 16 = 128 GB` total, i.e. **125 MB per rank** across 1,024 ranks. The
device-to-host copy of 125 MB over a pinned PCIe path at ~20 GB/s is ~6 ms; the blocking
cost is dominated by the collective barrier and metadata, not by bytes. **36 seconds is a
conservative figure for a well-implemented async sharded checkpoint and a pessimistic one if
you are still writing a monolithic `torch.save` from rank 0** — in which case `δ` is minutes,
`τ*` grows, and the redone-work term grows with it.

**Step 3 — the failure-overhead multiplier.**

```
  √(2δ/M) = √(0.02 / 7.9)  = √0.0025316 = 0.05032     ← checkpoint stalls + redone work
  (R+q)/M = 0.30 / 7.9                  = 0.03797     ← recovery + billed queue
  ─────────────────────────────────────────────────
  f_ovh                                 = 0.08829
  multiplier                            = 1.0883
  ETTR = 1/1.0883                       = 0.9189      (91.9% effective training time)
```

**Step 4 — cross-check by counting hours, not fractions.** Never present a multiplier you
have not verified against an hour count; this is the step that catches an algebra slip.

```
  per GPU, over the 65.83 productive hours:
    failures            = 65.83 / 7.9        = 8.333
    redone work         = 8.333 × 0.3975/2   = 1.656 h
    restart + queue     = 8.333 × 0.30       = 2.500 h
    checkpoint stalls   = 165.6 × 0.01       = 1.656 h
                                               ───────
    unproductive                               5.812 h
    allocated per GPU   = 65.83 + 5.812      = 71.64 h
    ⇒ ratio 71.64 / 65.83 = 1.0883  ✓  matches Step 3 exactly
    ⇒ wall-clock 71.64 h = 2.99 days (vs 2.74 failure-free)
```

**Step 5 — the money.**

```
  A_run (allocated GPU-hours) = 67,410 × 1.0883      =  73,362 GPU-h
  $_direct                    = 73,362 × $2.99       =  $219,352
     of which failure overhead = (73,362 − 67,410) × $2.99  =  $17,796   (8.1%)

  $_loaded                    = $219,352 × 1.3416    =  $294,285
     of which platform loading = $294,285 − $219,352 =  $74,933

  goodput                     = 0.40 × 0.9189        =  0.3676
  $ per useful GPU-hour       = 2.99 / 0.3676        =  $8.13
  $ per useful GPU-hour, loaded                      =  $10.91
```

**Step 6 — price the levers, in the order the elasticities say.**

```
  LEVER A — MFU 0.40 → 0.45  (fix an exposed all-reduce per 08.3, or a starved
                              loader per 08.7)
     GPU-hours_productive scales by 40/45 = 0.8889   ⇒ 59,920 GPU-h
     f_ovh is UNCHANGED (MFU does not touch δ, M, R, q)
     $_direct = 59,920 × 1.0883 × 2.99 = $194,979
     SAVES $24,373 per run  (11.1%)          ← the largest single lever here

  LEVER B — restart R 0.25 → 0.10 h  (a heartbeat watchdog instead of a long timeout)
     (R+q)/M = 0.15/7.9 = 0.01899 ;  f_ovh = 0.06931 ;  ×1.0693
     $_direct = 67,410 × 1.0693 × 2.99 = $215,536
     SAVES $3,816 per run  (1.7%)

  LEVER C — checkpoint δ 0.01 → 0.005 h  (async offload of the remaining barrier)
     √(2δ/M) = √(0.01/7.9) = 0.03558 ;  f_ovh = 0.07355 ;  ×1.0736
     τ* falls to √(2×7.9×0.005) = 0.281 h = 16.9 min
     $_direct = 67,410 × 1.0736 × 2.99 = $216,412
     SAVES $2,940 per run  (1.3%)   ← note: HALVING δ buys only 29% of the ckpt term,
                                      because δ enters under a square root

  LEVER D — rate 2.99 → 2.20 via a committed-capacity mix
     $_direct = 73,362 × 2.20 = $161,396
     SAVES $57,956 per run  (26.4%)          ← procurement beats every technical lever
                                               at this scale, and costs no engineering
```

**Step 7 — rerun at 16,384 GPUs and watch the ranking invert.** Same model, same MFU, same
`δ`, `R`, `q`; only `M` changes, to the study's 1.8 h projection.

```
  √(2δ/M) = √(0.02/1.8) = 0.10541
  (R+q)/M = 0.30/1.8    = 0.16667
  f_ovh   = 0.27208     ⇒ ×1.2721 ,  ETTR = 0.786 ,  τ* = 11.4 min

  A_run    = 67,410 × 1.2721 = 85,753 GPU-h        (vs 73,362 at 1,024 GPUs
                                                    — 12,391 extra GPU-hours for the
                                                    SAME work, purely from scale)
  $_direct = 85,753 × 2.99   = $256,401
  failure overhead            = $54,867            (21.4%, vs 8.1% before)

  LEVER B at this scale — R 0.25 → 0.10 h
     (R+q)/M = 0.15/1.8 = 0.08333 ; f_ovh = 0.18874 ; ×1.1887
     $_direct = 67,410 × 1.1887 × 2.99 = $239,591
     SAVES $16,810 per run  (6.6%)   ← 4.4× more valuable than the same fix at 1,024 GPUs

  LEVER A at this scale — MFU 0.40 → 0.45
     $_direct = 59,920 × 1.2721 × 2.99 = $227,912
     SAVES $28,489 (11.1%)            ← still the biggest, but restart is now within 2×
```

**Step 8 — write the four lines you would actually put on a slide.**

```
  Run: 8B params × 2T tokens on 1,024 H100.
  We pay for 73,362 GPU-hours; 26,964 of them become tensor-core FLOP.  Goodput 36.8%.
  Direct $219k, fully loaded $294k — $10.91 per useful GPU-hour against a $2.99 rate.
  Biggest levers, in order: commitment mix (−26%), MFU +5pp (−11%), restart time (−2%).
  At 16,384 GPUs the failure overhead goes 8.1% → 21.4% and restart time moves to #2.
```

Every number on that slide traces to a named input, and every input traces to a lesson or a
published measurement. That is what "defensible" means.

## Practice

Feeds the **Survive-a-failure lab**: this is its payoff figure. See
[`../practice/survive-a-failure/README.md`](../practice/survive-a-failure/README.md) for the
full lab spec — the steps below produce the number the lab reports and
[checkpoint.md](../checkpoint.md)'s criterion 6 asks you to defend.

1. **Measure MFU, do not assume it.** From the lab run, record sustained throughput and
   compute `MFU = achieved FLOP/s ÷ (G × P_peak)` using the **dense** peak for your precision.
   Note explicitly whether 08.7's `SM_active` figure is depressing it.
2. **Measure `δ` and `R`, do not assume them.** Time one checkpoint's *blocking* portion (not
   the total write). Time a full kill-to-first-step-after-recovery cycle and break it into
   detect / reschedule / reload / re-warm. The breakdown is the interesting part: whichever
   component dominates is your `R` lever.
3. **Get an `M` you can defend.** Either measure it (`allocated GPU-hours ÷ interruptions`
   over a month, if you have a cluster) or take the nearest measured point from §3's table and
   **say which**. Do not extrapolate the curve naively — §3 explains why.
4. **Compute `τ*` and check the run against it.** If your interval is not near
   `√(2Mδ)`, note the free win from fixing the interval before any other work.
5. **Build the waterfall.** Decompose allocated GPU-hours into productive / checkpoint /
   redone / restart+queue, cross-check the multiplier by hour count as in Worked example
   Step 4, and compute goodput and `$ per useful GPU-hour`.
6. **Do the sensitivity.** Compute the four elasticities at your scale and rank the levers.
   Then price *one* lever you could actually build in a sprint, and state its payback.
7. **State the boundaries.** Report your `A_ratio` (experiment tax) if you can estimate it,
   and name what the model excludes (§10).

**Acceptance:** a committed cost model containing (a) the formula with every input named and
dated, (b) the allocated-GPU-hours waterfall with the hour-count cross-check, (c) goodput and
`$ per useful GPU-hour` both direct and loaded with `L = (P/A)(1+f)` — **not** `1/utilisation`,
and with a one-line note on why, (d) the elasticity table and the resulting lever ranking, and
(e) one priced engineering lever with its payback. Keep it importable: it is a Module 11 input.

## Common pitfalls

- **"`GPU-hours × $/GPU-hr` is the cost."** *Mechanism:* it silently assumes MFU = 1 and
  zero failures. At MFU 0.40 and ETTR 0.92 the real figure is `1/(0.40 × 0.92) = 2.7×` the
  naive one per unit of model progress.
- **"Extrapolate MTBF from a published frontier run."** *Symptom:* your reliability term is
  a rounding error and reality is not. *Mechanism:* scaling Llama-3's 16,384-GPU rate down to
  1,024 GPUs gives ~45 h; Meta's measurement of that scale on shared research clusters is
  **7.9 h** — 5.7× worse. A flagship run on dedicated burned-in capacity is the best case, not
  the median. *Fix:* measure `M` on your own fleet, or use the nearest measured point and say
  so.
- **"More frequent checkpointing is the lever."** Only below `τ*`. Above it, more frequent
  checkpointing *costs* money — the `δ/τ` write term takes over. And past that, `δ` enters
  under a square root while `R` enters linearly, so **cheaper checkpoints are structurally a
  weaker lever than faster restart**, and increasingly so as you scale.
- **"Load the run with `1/utilisation` on top of the failure multiplier."** *Mechanism:* the
  run's unproductive hours are already inside its allocated hours, which `(1 + f_ovh)` already
  computed and the invoice already contains. Loading them again is module 11's documented
  double-count, which on its worked fleet overstates a unit cost by 78%. *Fix:* `L = (P/A)(1+f)`
  recovers *platform* waste only; the run's own waste is in `A_run`.
- **"Effective training time above 90% is normal."** It is a frontier result. FT-HSDP measured
  a **44%** baseline for fully synchronous training at O(100,000) GPUs before the specific
  restart engineering that took it to 80%. >90% is an achievement, not a default.
- **"Queue time is free."** It is free if your admission layer holds nothing while waiting
  (Kueue, or Volcano with delayed pod creation) and billed if it parks pods at the
  coscheduling `Permit` point holding assumed capacity (08.6 §6). Same wait, different
  invoice — and it is the term that decides §11's spot question.
- **"A cheaper `$/GPU-hr` provider is cheaper."** Cluster quality enters this model through
  `M`, `R` and MFU, none of which is the price. Ask for the failure rate per GPU-hour and the
  time-to-replacement-node before comparing rates.
- **"This is the cost of the model."** It is the cost of *one run*. The shipped model carries
  the whole failed-experiment tail (§10). Presenting `$_run(final)` as the model's cost
  understates it by a factor that is organisational, often large, and easy to measure.
- **"A vendor's all-in training quote is comparable to my GPU-hours estimate."** The vendor's
  price already contains *their* MFU and *their* failure overhead, which they bear and price
  in. Comparing it to a naive `hours × rate` compares two different quantities that share a
  dollar sign.
- **"Use the sparsity TFLOPS number."** H100 SXM's headline 1,979 BF16 TFLOPS is quoted with
  2:4 structured sparsity. Dense pretraining gets ~989 TFLOP/s. Using the wrong one halves
  your predicted GPU-hours and is the most common single error in a training cost estimate.

## Self-check

- **Write the full cost model, name every term, and say which lesson owns it.**
  **Answer:**
  `$_direct = [6·N·D / (P_peak · MFU)] × r × (1 + √(2δ/M) + (R+q)/M)`, and
  `$_loaded = $_direct × L` with `L = (P/A)(1+f)`.
  `6·N·D` is the FLOP budget for a dense transformer (08.1). `P_peak` is the accelerator's
  **dense** peak — 989 TFLOP/s BF16 on H100 SXM, not the sparsity figure (module 03). `MFU`
  is achieved over peak (08.3), itself depressed by exposed communication (08.3), input
  starvation (08.7) and occupancy. `r` is a blended rate on a stated commitment mix and date
  (module 11 §7). `δ` is the *blocking* portion of one checkpoint and `√(2δ/M)` is what the
  checkpoint-plus-redone-work terms collapse to at the Young/Daly optimum `τ* = √(2Mδ)`
  (08.4). `M` is MTBF at this GPU count (08.5, and measured). `R` is detect + reschedule +
  reload + re-warm (08.2 for detection, 08.5, 08.6). `q` is the *billed* queue per restart,
  which is zero if the admission layer holds no capacity while waiting and nonzero if it parks
  pods at `Permit` (module 06, 08.6). `L` recovers platform inefficiency and non-GPU spend
  (module 11 §6). `ETTR = 1/(1 + f_ovh)`; `goodput = MFU × ETTR`; `$ per useful GPU-hour =
  r / goodput`.

- **Which two or three terms actually move the answer, and how does that change with scale?**
  **Answer:** As log-elasticities, `MFU`, `r` and `L` are each unit-elastic — a 1% change is a
  1% change in cost, at any scale. The reliability terms are not:
  `ε(M) = −[½√(2δ/M) + (R+q)/M]/(1+f_ovh)`, `ε(R) = (R/M)/(1+f_ovh)`,
  `ε(δ) = ½√(2δ/M)/(1+f_ovh)`. At 1,024 GPUs with `M = 7.9 h`, `δ = 0.01 h`, `R = 0.25 h`,
  `q = 0.05 h`: `ε(M) = −0.058`, `ε(R) = +0.029`, `ε(δ) = +0.023` — all an order of magnitude
  below the unit-elastic three, so **fund MFU and procurement**. At 16,384 GPUs with
  `M = 1.8 h`: `f_ovh` rises from 0.088 to 0.272 and `ε(R)` rises to `+0.109`, so restart time
  becomes the second-biggest technical lever. The crossover — where `f_ovh` exceeds 10% — is
  around `M ≈ 6.6 h`, which on Meta's curve is roughly **1,200–1,400 GPUs**. And because `δ`
  enters under a square root while `R` enters linearly, faster restart always beats cheaper
  checkpoints, increasingly so with scale.

- **Compute the failure-overhead multiplier and the optimal interval for `M = 1.8 h`,
  `δ = 0.01 h`, `R = 0.25 h`, `q = 0.05 h`, and cross-check it by counting hours over a
  100-hour productive budget.**
  **Answer:** `τ* = √(2 × 1.8 × 0.01) = √0.036 = 0.1897 h = 11.4 min`.
  `√(2δ/M) = √(0.02/1.8) = 0.10541`; `(R+q)/M = 0.30/1.8 = 0.16667`; `f_ovh = 0.27208`, so the
  multiplier is **1.2721** and `ETTR = 0.786`. Cross-check over 100 productive hours:
  failures `= 100/1.8 = 55.6`; redone `= 55.6 × 0.1897/2 = 5.27 h`; restart+queue
  `= 55.6 × 0.30 = 16.67 h`; checkpoint stalls `= (100/0.1897) × 0.01 = 5.27 h`; total
  unproductive `= 27.21 h`, allocated `= 127.21 h`, ratio **1.2721** ✓. Note that redone work
  and checkpoint stalls are equal — that equality is the signature of being exactly at `τ*`,
  and it is the fastest way to check an interval by eye.

- **Define goodput against the three time bases and explain why it is the number to present.**
  **Answer:** `MFU` = useful tensor-core FLOP time ÷ **productive** time (how well a step
  runs). `ETTR` = productive time ÷ **allocated** time (how much of what you rent is spent on
  steps; Meta's definition also counts queue time as unproductive). `Goodput = MFU × ETTR` =
  useful FLOP time ÷ **allocated** time — the fraction of what you are *billed for* that
  becomes model. It is the number to present because `r / goodput` is the cost per unit of
  actual progress: at MFU 0.40 and ETTR 0.919, goodput is 0.368, so a $2.99/GPU-hr rate
  delivers work at **$8.13 per useful GPU-hour**, or $10.91 fully loaded. It also unifies the
  room: MFU is the ML engineer's metric, ETTR is the SRE's, goodput is finance's, and they are
  the same arithmetic.

- **Is spot capacity cheaper for a 1,024-GPU run? Show the work.**
  **Answer:** It depends almost entirely on `q`, not on the discount. Preemption is a second
  independent failure process and hazards add: with hardware `M = 7.9 h` and a 128-node gang
  whose per-node preemption mean is 72 h, preemption MTBF is `72/128 = 0.5625 h`, so
  `1/M_total = 1/7.9 + 1/0.5625 = 1.9044` and `M_total = 0.525 h` — preemption dominates by
  14×. At a 60% discount (`r = $1.196`): if the gang is re-admitted promptly (`q = 0.05 h`),
  `f_ovh = √(0.02/0.525) + 0.30/0.525 = 0.767`, multiplier 1.767, and the run costs **$142k
  against $219k on-demand — spot wins by 35%**. If the market is tight and each restart waits
  half an hour (`q = 0.5 h`), `f_ovh = 1.624`, multiplier 2.624, and the run costs **$212k —
  a 3.6% advantage, inside the error bars**. Two structural consequences fall out of the
  algebra: `τ*` collapses to `√(2 × 0.525 × 0.01) = 6.2 minutes`, so asynchronous sharded
  checkpointing with a small `δ` is a *precondition* for spot rather than an optimisation; and
  an elastic job that continues at reduced width (08.5's `--nnodes=min:max`) converts `q` from
  a stall into a smaller MFU reduction, which the model prices as a much cheaper loss.

- **Why can two providers quoting an identical `$/GPU-hr` produce different true costs per
  run, and what should you ask them?**
  **Answer:** Because `r` is only one of four factors, and cluster quality enters through the
  other three. Node failure rate sets `M`; time from failure to a replacement node being
  schedulable sets `R`; fabric bisection bandwidth, topology-aware placement and storage
  bandwidth set `MFU` (08.1, 08.3, 08.7); and fragmentation and unallocatable capacity set
  `P/A` inside `L` (module 11). So the questions to ask are: what is your observed
  interruption rate per GPU-hour at my job size, what is your median time-to-replacement-node,
  what bisection bandwidth and rail alignment do I actually get, and what fraction of a
  reserved block is typically unallocatable. A vendor who cannot answer those is quoting a
  price for an unknown quantity.

## Connections & what's next

This closes module 08. Every lesson fed the capstone: 08.1–08.2 determined which collectives
you pay for and over which link; 08.3 supplied MFU, the term with the largest elasticity;
08.4 supplied `δ` and Young/Daly, which collapse two overhead terms into `√(2δ/M)`; 08.5
supplied the failure loop that `M` and `R` describe and the elastic option that shrinks `q`;
08.6 supplied the objects whose failure policies set `R`'s blast radius and whose admission
layer decides whether `q` is billed at all; 08.7 supplied a second, unlabelled tax on the same
MFU term. Together they are exactly what the
[Survive-a-failure lab](../practice/survive-a-failure/README.md) asks you to build and what
[checkpoint.md](../checkpoint.md)'s six pass criteria — especially criterion 6, "price it",
and its "which lever" follow-up — test cold.

Forward, this is the on-ramp to **module 11**, and the join is deliberate and specific:
`A_run` from §3 is exactly the `H_attr` that
[11.5 · Unit economics](../../11-gpu-cost-economics/lessons/05-unit-economics.md) divides by
an application counter, `L = (P/A)(1+f)` is the multiplier this lesson borrows rather than
re-derives, and the double-count warning in §7 is the same one module 11 raises about
`1/utilisation`. The shape of thinking — separate a naive resource-hours number from the
*effective* number that failures and inefficiency actually cost — is identical; module 11
applies it across a fleet and a token counter rather than to one run.

Two habits to carry out of this module. **Always cross-check a multiplier against an hour
count** (Worked example Step 4) — it is the cheapest way to catch an algebra error before it
reaches a slide. And **always state the time base**: wall-clock, allocated, or productive.
Most cost disagreements are two people using different ones without noticing.

## References & further reading

**Primary sources — reliability data**

1. **Meta — "Revisiting Reliability in Large-Scale Machine Learning Research Clusters"** —
   <https://arxiv.org/abs/2410.21680> (HPCA 2025). The empirical base for §3: 11 months of two
   production clusters, **4M jobs, >150M A100 GPU-hours**; the **ETTR** definition (productive
   runtime ÷ productive-plus-unproductive, including queue) and its three unproductive
   categories (catch-up from checkpoint, restart overhead, checkpoint overhead); and the
   fitted MTTF curve — **47.7 days at 8 GPUs, 7.9 h at 1,024 GPUs**, projected **1.8 h at
   16,384** and **0.23 h at 131,072**, with hardware reliability scaling roughly inversely
   with GPU count above ~32 GPUs. **Read this one if you read only one.** *arxiv.org is
   blocked by this environment's egress proxy; these figures were confirmed via search against
   the paper's abstract and multiple published summaries, not by reading the PDF in this pass.*
   **Correction recorded:** an earlier version of this lesson derived `M ≈ 45 h` for a
   1,024-GPU job by scaling Llama-3's per-GPU rate; the measured figure is 7.9 h, and §3 now
   explains why both are real.
2. **Meta — "The Llama 3 Herd of Models", §3.3** — <https://arxiv.org/abs/2407.21783>. 16,384
   H100s over 54 days; **466 interruptions, 47 planned and 419 unexpected**; roughly **78%
   attributed to hardware**, led by faulty GPUs (30.1%) and HBM3 (17.2%); **>90% effective
   training time**. *arxiv.org blocked; search-verified. Correction: 419 unexpected
   interruptions, not 417 as previously stated here.*
3. **Meta — "Training LLMs with Fault Tolerant HSDP on 100,000 GPUs"** —
   <https://arxiv.org/abs/2602.00277>. Data-parallel replicas as the fault-tolerance unit;
   recovery stall **10 min → 3 min**, effective training time **44% → 80%**, no meaningful
   accuracy impact. The empirical confirmation that `R/M` dominates at extreme scale.
   *arxiv.org blocked; search-verified.*

**Primary sources — the model's other inputs**

4. **[08.3 · Communication as the bottleneck](03-communication-bottleneck.md)** — MFU's
   definition, the roofline, and the exposed-communication term. This lesson consumes MFU and
   does not re-derive it.
5. **[08.4 · Checkpointing](04-checkpointing.md)** — Young/Daly, `τ* = √(2Mδ)`, and the
   distinction between total checkpoint write time and the *blocking* portion `δ` that the
   model uses. The source of the `√(2δ/M)` collapse.
6. **[08.5 · Failure & elasticity](05-failure-and-elasticity.md)** — the detection → drain →
   re-rendezvous → resume loop that `R` measures, and the elastic option that converts `q`
   into an MFU reduction.
7. **[08.6 · Job orchestration](06-job-orchestration.md)** — the failure policies that set a
   restart's blast radius, and §6's distinction between admission layers that hold capacity
   while waiting and those that hold nothing, which is what decides whether `q` is billed.
8. **[08.7 · Data pipeline](07-data-pipeline.md)** — the starvation tax on MFU, and the
   storage-bandwidth budget that lands in `f`.
9. **[11.5 · Unit economics](../../11-gpu-cost-economics/lessons/05-unit-economics.md)** — the
   loading multiplier `L = (P/A)(1+f)`, the documented double-count in `L = 1/utilisation`
   (78% overstatement on that lesson's worked fleet), the blended-versus-marginal rate
   distinction, and the `$2.99/GPU-hr` blended H100 figure used here. **This lesson's §7 is
   written specifically to compose with it.**
10. **NVIDIA H100 datasheet figures** — H100 SXM at 80 GB HBM3, 3.35 TB/s memory bandwidth,
    700 W, with Tensor Core peaks quoted **with 2:4 structured sparsity** (1,979 TFLOPS BF16,
    3,958 TFLOPS FP8) — i.e. roughly **989 and 1,979 TFLOP/s dense**. *NVIDIA's hosted
    datasheet PDFs are blocked from this environment; these are the widely reproduced
    datasheet values, search-verified, and identical to the figures module 11 uses. Confirm
    against the datasheet for your exact SKU — PCIe and NVL variants differ.*

**Real-world cost figures — all dated snapshots**

11. **Databricks/MosaicML — "Mosaic LLMs (Part 2): GPT-3 quality for <$500k"** —
    <https://www.databricks.com/blog/gpt-3-quality-for-500k> (**2023**). ~$450K for
    GPT-3-quality training, framed as the payoff of managing MFU and reliability rather than
    of ignoring them. *databricks.com is blocked here; search-verified.*
12. **Databricks — "Introducing MPT-7B"** — <https://www.databricks.com/blog/mpt-7b>
    (**2023**). **440 A100-40GB, ~9.5 days, ~$200K, zero human intervention**, with the
    platform handling hardware failures automatically. The cleanest small-scale figure to
    sanity-check a model against. *Blocked here; search-verified.*
13. **BigScience — BLOOM-176B** — reported at roughly **€3–5M** including engineering support,
    storage and networking, over 117 days on 3,000+ GPUs of the Jean Zay supercomputer
    (**circa 2022**). The "including engineering support" caveat is this lesson's §10 thesis
    stated by someone else. *Not fetched in this pass.*
14. **SemiAnalysis — "How Much Do GPU Clusters Really Cost?"** (**2026 snapshot**). Cluster
    TCO and the "goodput" framing; argues a well-run specialist cloud can beat a hyperscaler
    on total cost at an equal or higher nominal rate. *Blocked from this environment; **not
    fetched in this pass and not relied upon for any number above** — cited only as an
    independent statement of §4's argument.*
15. **GPU rate survey, 2026-08.** On-demand H100 SXM spans roughly **$1.4–$7/GPU-hr** across
    specialist GPU clouds, with hyperscaler list prices reported up to **~$7–12/GPU-hr**;
    market medians reported around **$2.5/GPU-hr**. Module 11 uses a blended **$2.99/GPU-hr**
    for the same hardware, which is the figure used throughout this lesson. *Compiled from
    multiple public pricing surveys via search; vendor pricing pages are blocked from this
    environment. Treat as a dated snapshot and re-pull your own blended rate before quoting.*

**Deeper dives**

16. **stas00/ml-engineering** — <https://github.com/stas00/ml-engineering>. An open book on
    large-scale training written by a BLOOM engineer; the reliability and performance chapters
    are the best practitioner companion to this module's `M`, `R` and MFU terms.

> **Snapshot (2026-08).** Every dollar figure here is dated and every one is an input you
> should replace. `$/GPU-hr` is a linear scalar in the model, so substituting your own blended
> rate is a one-line change. The reliability inputs matter more: `M` in particular varies by
> more than 5× between a shared research cluster and a dedicated flagship run at the same GPU
> count, so **measure it** rather than quoting a paper. Re-verify the MTTF projections, the
> published run costs (2022–2023 snapshots) and the rate band before putting any of them in
> front of a budget owner.
