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
sources: 9
---

# 08.8 · Training economics

> **Concept.** Turn the reliability and efficiency math of this module into one dollar figure: the true cost of a successful run.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Where this fits

This is the capstone — the last lesson of the module, and the one every other lesson's math feeds
into. 01–02 gave you the parallelism/collective vocabulary; 03 gave you MFU as the efficiency
report card; 04 gave you the checkpoint-interval math; 05 gave you the failure/restart loop and its
detection signals; 06 gave you the object that expresses and re-rendezvouses the job; 07 gave you a
third source of wasted GPU-hours (starvation) that a naive cost model would silently miss. None of
those lessons individually produce a number a CFO or a hiring panel can act on. This one does:
**everything upstream is an input to a single formula for cost-per-successful-run.**

## Why this matters

This is the capstone, and it is *your* differentiator. Everyone in this module learned the mechanics —
parallelism, NCCL, checkpointing, elasticity, starvation. You are the one who can put a **dollar
figure on a training run that includes what failures actually cost**, and defend it. That skill is
not formally required by anything downstream, but it is the natural precursor to the cost thinking
Module 11 (GPU cost & unit economics) applies at fleet scale later in the course — and it is what a
"cost/observability" platform engineer brings to a GPU-heavy org that a distributed-systems
generalist does not.

The naive number — `GPU-hours × $/GPU-hr` — is a lie in two directions. It ignores **efficiency**
(a run at 35% MFU burns ~29% more hours than the same run at 45%) and it ignores **failure overhead**
(checkpoint writes, redone work, restart time). At scale those are not rounding errors: Llama 3's
16,384-GPU run hit a hardware-caused interruption roughly **every ~3 hours**. A cost model that assumes
zero failures is off by the exact amount your reliability engineering is worth — which is precisely the
thing you want to be able to price. And it isn't just a hyperscale phenomenon: Databricks/MosaicML's
own framing of their sub-$500K GPT-3-quality run explicitly credits *managing* MFU and failure
overhead, not ignoring them, as the reason the number is small.

## What's new here (calibration)

- **One formula that ties the module together.** MFU (03) sets the hours; Young/Daly (04) and recovery
  (05) set the overhead; 07's starvation is a second, independent tax on the same MFU term; the rate is
  FinOps. We assemble them into cost-per-*successful*-run. We do not re-derive Young/Daly or re-teach
  MFU's roofline mechanics here — 03 and 04 own those; this lesson only assembles what they already
  gave you.
- **The failure-overhead multiplier.** Not "add 10% for slack" — a derived fraction
  `≈ √(2δ/M) + R/M` from the checkpoint cost `δ`, MTBF `M`, and restart time `R`.
- **The two levers, priced.** Raising MFU and shrinking restart both cut `$/run`, but by different
  mechanisms and different magnitudes, and which one dominates is itself a function of scale. You will
  be able to say which buys more per engineering-dollar, and at what GPU count that answer flips.
- **Effective training time** as the industry framing (Llama 3 §3.3): the fraction of wall-clock that
  was productive. Your cost multiplier is just its inverse, in dollars. We also bring in two more
  independently-sourced real effective-training-time numbers (Meta's cluster-wide study and a
  100,000-GPU fault-tolerant-training paper) to show that Llama 3's >90% is a frontier result, not a
  typical one.
- **The rate itself is not a fixed input.** `$/GPU-hr` looks like a constant you plug in, but
  procurement-quality differences (hyperscaler vs. neocloud, setup/tuning/downtime overhead) mean the
  "same" nominal rate can hide a comparable-sized hidden cost to the failure-overhead term. We name this
  as a third lever the base formula doesn't explicitly price.

## Core concepts

### Step 1 — Raw GPU-hours from MFU (lesson 03)

The *useful* work of a training run is a fixed FLOP budget, independent of how efficiently you run it.
For a dense transformer the standard estimate is:

```
C_flops ≈ 6 × N_params × N_tokens        # fwd+bwd, the "6ND" rule
```

Wall-clock and therefore GPU-hours depend on how much of the hardware's peak you sustain — the **MFU**
(Model FLOPs Utilization = achieved FLOP/s ÷ peak FLOP/s, from lesson 03):

```
GPU_hours_raw = C_flops / (P_peak × MFU) × ... = N_GPUs × wall_clock_hours
             ∝ 1 / MFU
```

The key fact for costing: **GPU-hours are inversely proportional to MFU.** H100 SXM peak is
~989 TFLOP/s BF16 dense; LLM pretraining typically sustains 35–50% MFU. Doubling MFU halves the hours
and halves this part of the bill. This is the biggest lever and it is why lesson 03 came first. Note
that MFU is a *composite* number: a comms-bound stall (03), a starved data pipeline (07), and low
occupancy from a bad kernel config all show up as the same depressed MFU. The formula below doesn't
care which one caused it — but your diagnosis work in 03/07 is what lets you attack the right one.

### Step 2 — The failure-overhead multiplier (lessons 04 + 05)

A real run does not spend 100% of its GPU-hours on `C_flops`. Three leaks, all measured against
productive time `T` (in GPU-hours or wall-hours — pick one and stay consistent):

| Leak | Time cost | As a fraction of `T` |
|------|-----------|----------------------|
| Checkpoint **writes** | `(T/τ) × δ` | `δ / τ` |
| **Redone** work after a crash | `(T/M) × (τ/2)` | `τ / (2M)` |
| **Restart / recovery** per crash | `(T/M) × R` | `R / M` |

where `δ` = time to write one checkpoint, `τ` = checkpoint interval, `M` = MTBF (mean time between
failures for *this job*, which shrinks as you add GPUs), `R` = detect + reschedule + reload + re-warm
time per failure. Expected redone work is `τ/2` because a crash lands, on average, halfway through an
interval.

Minimizing `δ/τ + τ/(2M)` over `τ` gives **Young/Daly** (lesson 04):

```
τ_opt = √(2 · M · δ)
```

Substitute it back and the minimized overhead fraction is:

```
failure_overhead_fraction  ≈  √(2δ/M)  +  R/M
                               └checkpoint+redo┘  └restart┘
```

This is the **multiplier**: a run at the optimal checkpoint interval costs
`(1 + √(2δ/M) + R/M)` times its failure-free cost. Note two scaling facts you will reuse: the checkpoint
term grows only as `√(1/M)` while the restart term grows as `1/M`. Since `M ∝ 1/N_GPUs`, **at very
large scale the restart term dominates** — which is exactly why hyperscalers pour engineering into fast
detection and restart, not just frequent checkpoints. This is not just a theoretical scaling argument:
Meta's own fault-tolerant-HSDP work at 100,000-GPU scale measured stall time from failure recovery
directly, and cut it from 10 minutes to 3 minutes per event by restarting only the affected data-parallel
replica instead of the whole job — moving effective training time from 44% to 80%. That is a
restart-engineering win, not a checkpoint-frequency win, and it is the real-world confirmation of the
`R/M`-dominates-at-scale claim above.

### Step 3 — Assemble: true cost-per-successful-run

```
$/successful_run = GPU_hours_raw × $/GPU-hr × (1 + failure_overhead_fraction)

               = [ C_flops / (P_peak × MFU) ] × $/GPU-hr × (1 + √(2δ/M) + R/M)
                    └── MFU lever (÷) ──┘                    └─ reliability lever (×) ─┘
```

Everything in this module is in that line. Raising MFU shrinks the first bracket; shrinking `δ` or `R`
shrinks the multiplier. The two are *multiplicative*, so the MFU win and the reliability win compound.
`$/GPU-hr` is drawn as a flat scalar here, but see Perspectives below for why it is not actually fixed
— procurement quality hides a cost structure of its own, parallel to the failure-overhead term.

### Effective training time (Llama 3 framing, and how typical it is)

Meta reports "effective training time" — productive time ÷ wall-clock — as their headline reliability
number; Llama 3 sustained **>90%** across the 405B run despite 466 interruptions in a 54-day window
(417 unexpected, ~78% hardware). Effective-training-time `= 1 / (1 + failure_overhead_fraction)`. Your
cost multiplier is literally the reciprocal of that KPI, denominated in dollars. When you present to a
platform org, quote both: the SRE cares about effective-training-time, the CFO cares about the multiplier.

Treat Llama 3's >90% as a frontier result, not a default. Two independently-sourced data points make
that concrete: Meta's own cluster-wide reliability study, analyzing **150M+ A100-GPU-hours** across
4 million jobs on two production research clusters, is the broader empirical base this module's
single-run MTBF numbers are drawn from — it's worth knowing that Llama 3's headline figure sits on top
of a much larger, messier population of jobs from 1 to 4k+ GPUs, most of them smaller and less
engineered than a flagship run. And the 100,000-GPU fault-tolerant-HSDP result above shows a
*baseline* of only 44% effective training time at fully-synchronous scale before the specific
restart-engineering fix this module teaches — i.e., without that investment, effective time can be
dramatically below 90% even at a well-resourced lab.

## Perspectives

**CFO / FinOps view.** The multiplicative structure — `(1/MFU) × $/GPU-hr × (1 + failure_overhead)` —
is the single formula tying every other lesson in the module together. Stress-test it against real
numbers, not just Llama-3 scale: Databricks/MosaicML's ~$450K GPT-3-quality run (2023 snapshot) and
MPT-7B's ~$200K, 9.5-day, zero-human-intervention run (2023 snapshot) are both consistent with this
formula at 40–60% MFU and small failure-overhead fractions — the formula predicts realistic magnitudes
at $200K-scale runs, not just $9M-scale ones.

**SRE / reliability view.** "Effective training time" is the SRE-facing framing of the *identical* math
the CFO sees as a cost multiplier — both audiences care about the same underlying number, just
denominated differently (a fraction vs. dollars). Being able to translate fluently between the two in
the same sentence — "we're at 92% effective time, which is about a 1.09× cost multiplier" — is a
genuinely useful thing to be able to say in an interview or a budget review.

**Scale-dependent-lever view.** MFU dominates the savings opportunity at ~1,024-GPU scale (Lever A in
the worked example below); the restart term dominates at ~16,384+ GPU scale, and the 100,000-GPU
FT-HSDP result (44%→80% effective time, driven by restart engineering rather than checkpoint-frequency
tuning) is strong independent confirmation that this is what hyperscalers actually observed and acted
on, not just an artifact of the algebra.

**Vendor / procurement view.** SemiAnalysis's TCO analysis of GPU cluster costs argues that a
"gold-tier" neocloud can beat a hyperscaler on *total* cost even at an equal or higher nominal
`$/GPU-hr`, because of hidden downtime, setup, and tuning costs baked into cluster quality — their
"Grand Unifying Theory of Goodput" framing. That is a third lever the base formula above doesn't
explicitly price: the `$/GPU-hr` term is not a fixed input, it's a procurement decision with the same
kind of hidden-cost structure as the `failure_overhead_fraction` term. Two clusters quoting the same
sticker rate can have meaningfully different true cost-per-successful-run.

## Real-world use cases

- **Databricks/MosaicML — "Mosaic LLMs (Part 2): GPT-3 quality for <$500k"** —
  <https://www.databricks.com/blog/gpt-3-quality-for-500k> — **2023 snapshot.** Reports GPT-3-quality
  training achievable for ~$450K, framed as "2–10x less than people think." What it shows: a concrete,
  real-world number consistent with this lesson's formula at 40–60% MFU — the low cost is presented as
  the direct payoff of managing MFU and reliability, not evidence that the naive-cost model was right.
- **Databricks — "Introducing MPT-7B"** — <https://www.databricks.com/blog/mpt-7b> — **2023 snapshot.**
  MPT-7B trained in 9.5 days on 440 GPUs for ~$200K, with the platform detecting and handling 4 hardware
  failures automatically ("zero human intervention"). What it shows: a small-model real cost figure
  where the low *human*-ops cost is presented as a direct consequence of the automated failure recovery
  this module teaches — a direct callback to 05's elasticity thesis.
- **BigScience — BLOOM-176B** — cost reported around €3–5M (~$3.5–5.5M, including engineering support,
  storage, and networking), trained over 117 days on 3,000+ GPUs of the Jean Zay supercomputer
  (figures **circa 2022**, per BigScience/Nature/HF reporting). What it shows: a mid-scale real budget
  figure where the "including engineering support" caveat is exactly this lesson's thesis — the naive
  GPU-hours × rate number, on its own, undercounts the real cost of a run.
- **SemiAnalysis — "How Much Do GPU Clusters Really Cost?"** —
  <https://newsletter.semianalysis.com/p/how-much-do-gpu-clusters-really-cost> — **published April
  2026, treat as a dated snapshot.** An industry-analyst treatment of cluster TCO arguing hyperscalers
  can be meaningfully more expensive than gold-tier neoclouds on a TCO-adjusted basis even at equal
  nominal `$/GPU-hr`, because of hidden downtime/setup/tuning costs. What it shows: an independent
  source reinforcing "the naive number is a lie," and introduces the "Grand Unifying Theory of Goodput"
  framing this lesson's Perspectives section draws on.

## Worked example

**Run.** 1024 × H100 SXM, a pretraining job that needs `T = 720` productive GPU-*hours per GPU*
(30 days wall-clock at 100% effective time), i.e. `GPU_hours_raw = 1024 × 720 = 737,280 GPU-hr`.
Rate `$/GPU-hr = $12` (2026 hyperscaler on-demand H100 — **snapshot, substitute your own**).

**Raw (naive) cost:** `737,280 × $12 = $8.85M`.

**Reliability inputs.**
- MTBF `M ≈ 45 hr`. *Where this comes from:* Llama 3's 16,384-GPU job saw 466 interruptions over
  ~1,296 hr → a per-GPU failure rate of `466 / (1,296 × 16,384) ≈ 2.2×10⁻⁵` per GPU-hr. Scaled to 1,024
  GPUs: `M ≈ 1 / (1024 × 2.2×10⁻⁵) ≈ 45 hr`. (Failure rate scales with GPU count.) Meta's own
  cluster-wide reliability study — 150M+ A100-GPU-hours across 4M jobs — is the broader empirical base
  such per-scale MTBF projections are drawn from in practice, rather than extrapolating from a single run.
- Checkpoint cost `δ = 0.05 hr` (3 min, async sharded distributed checkpoint).
- Restart `R = 0.5 hr` (detect + reschedule + reload + re-warm).

**Optimal interval (Young/Daly):** `τ_opt = √(2 × 45 × 0.05) = √4.5 ≈ 2.1 hr` → checkpoint every ~2 hr.

**Failure-overhead multiplier:**
```
√(2δ/M) = √(0.1/45)  = 0.0471   (checkpoint writes + redone work)
R/M     = 0.5/45     = 0.0111   (restart)
overhead_fraction    = 0.0583   → multiplier 1.058  (≈ 94.5% effective training time)
```

**True cost-per-successful-run:** `$8.85M × 1.058 = $9.36M`. The failure overhead is **~$0.51M/run** —
the number the naive model silently drops, and the direct payoff of your checkpointing/restart work.

**Sanity-check the multiplier by counting hours** (per GPU, over 720 hr): failures
`= 720/45 = 16`; each loses `τ/2 + R = 1.05 + 0.5 = 1.55 hr` → `24.8 hr` lost; checkpoint writes
`= (720/2.1) × 0.05 = 17.1 hr`. Total `≈ 41.9 hr / 720 = 5.8%`. Matches. ✓

**Lever A — MFU 35% → 45%.** `GPU_hours_raw ∝ 1/MFU`, so cost × `35/45 = 0.778` → **−22.2%**. If the
$8.85M raw assumed 35% MFU, at 45% raw becomes `$6.88M`, and true cost `$6.88M × 1.058 = $7.28M`.
**Saves ~$2.08M/run** — four times the entire failure overhead. MFU is the dominant lever at this scale.
(A starved data pipeline, uncaught by 07's diagnosis, is exactly the kind of hidden MFU tax that would
eat into this lever without ever showing up as a "failure.")

**Lever B — cut restart `R` 0.5 → 0.1 hr.** `R/M` drops `0.0111 → 0.0022`; multiplier `1.058 → 1.049`.
Saves `≈ $8.85M × 0.0089 = $79k/run`. Real, but an order of magnitude below the MFU win *at this scale*.
Rerun at 16,384 GPUs (`M ≈ 2.8 hr`) and the restart term `R/M` balloons — then faster restart is where
the money is. This is exactly the pattern the 100,000-GPU FT-HSDP result confirms empirically: at
sufficient scale, restart engineering (not checkpoint tuning) is what moved effective time from 44% to
80%, a far bigger swing than either lever produces at 1,024-GPU scale.

## Practice

Feeds the **Survive-a-failure lab**: this is its payoff figure. See
[`../practice/survive-a-failure/README.md`](../practice/survive-a-failure/README.md) for the full lab
spec — this lesson's steps below produce the number the lab reports.

1. **Instrument GPU-hours.** From the run (or the 08.7 before/after artifact), record sustained
   throughput and MFU, and compute `GPU_hours_raw` for a fixed epoch/token budget.
2. **Cost per epoch.** Multiply by a real `$/GPU-hr` (flag the rate + date as a snapshot). Report
   `$/epoch` and `$/run` at zero-failure.
3. **Add failure overhead.** Plug your measured/observed `M` (from the lab's injected failures or the
   Llama-scaled rate), your checkpoint `δ`, and restart `R` into `1 + √(2δ/M) + R/M`. Confirm you are at
   `τ_opt`; if not, note the free win from fixing the interval first.
4. **Report true cost-per-successful-run** and break out the failure-overhead dollars. Then show one
   lever moved (MFU +10pp *or* restart −Δ) and its `$/run` delta.

**Acceptance:** a committed **cost-per-successful-run figure that includes failure overhead**, with the
formula, inputs, the rate+date snapshot, and one priced lever. This is the deliverable's payoff and the
figure the checkpoint gate ("Price it") asks you to defend.

## Common pitfalls

- **"`GPU-hours × $/GPU-hr` is the cost."** Ignores both MFU (real runs sustain 35–45%, not 100%) and
  failure overhead. Databricks/MosaicML's own framing of their sub-$500K number is explicit that it's
  achievable *because* they manage both terms, not because they ignored them — treat that framing as
  the correction, not the naive multiplication.
- **"Cheaper `$/GPU-hr` providers are always cheaper overall."** SemiAnalysis's TCO analysis shows
  meaningful swings from downtime, setup, and tuning costs at an *equal* nominal rate. The rate is a
  necessary input to true cost-per-run, not a sufficient one — ask about cluster quality, not just
  the sticker price.
- **"More frequent checkpointing is the primary lever at any scale."** The FT-HSDP 100,000-GPU result
  is the clean counter-evidence: the restart term (`R/M`, linear in scale) overtakes the checkpoint term
  (`√(2δ/M)`, sub-linear) as GPU count grows, so at very large scale, fast/partial restart — not
  checkpoint frequency — is where the engineering effort and the measured 44%→80% win actually went.
- **"The Llama 3 >90% effective-training-time number is typical."** It's a frontier, heavily-engineered
  result at a specific scale and time. FT-HSDP's own pre-fix baseline of 44% effective time at
  fully-synchronous 100K-GPU scale, and Meta's own 150M-GPU-hour cluster-wide study spanning jobs from
  1 to 4k+ GPUs, both show effective-training-time can sit dramatically lower without the specific
  engineering investments this module teaches. >90% is an achievement, not a default assumption.
- **"A vendor's quoted training price is directly comparable to my self-hosted GPU-hours estimate."**
  A vendor-quoted all-in price already bundles in *their* assumed MFU and failure-overhead — they bear
  that risk and price it in. Comparing a naive self-hosted GPU-hours × rate estimate to a vendor's
  all-in quote, without adjusting for MFU and failure overhead on both sides, compares two different
  quantities that happen to share a dollar sign.

## Self-check

- **Given MTBF `M`, checkpoint cost `δ`, restart `R`, and `$/GPU-hr`, what is the failure-overhead
  multiplier on a run? Show the reasoning.**
  **Answer:** At the Young/Daly optimum `τ_opt = √(2Mδ)`, overhead comes from checkpoint writes (`δ/τ`),
  redone work (`τ/2M`), and restart (`R/M`). The first two minimize to `√(2δ/M)`; adding restart gives
  `overhead_fraction ≈ √(2δ/M) + R/M`, so the **multiplier is `1 + √(2δ/M) + R/M`**. E.g. `δ=0.05 hr`,
  `M=45 hr`, `R=0.5 hr` → `√(0.1/45) + 0.5/45 = 0.047 + 0.011 = 0.058` → `×1.058`. `$/GPU-hr` scales the
  whole thing linearly but does not change the *fraction*.

- **How does raising MFU from 35% to 45% change cost-per-run?**
  **Answer:** GPU-hours are `∝ 1/MFU`, so cost scales by `35/45 = 0.778` → **−22.2%**, holding the failure
  multiplier fixed (MFU doesn't touch `δ`, `M`, or `R`). On a $8.85M raw run that is ~$2.08M/run — the
  largest single lever in the model at 1,024-GPU scale, because it shrinks the base hours that
  *everything else multiplies*.

- **Cheaper to invest in more-frequent checkpoints or faster restart — and on what does it depend?**
  **Answer:** It depends on (1) whether you are already at `τ_opt` and (2) the ratio of `R` to
  `τ_opt/2`. Checkpointing *more often than `τ_opt`* wastes money (the `δ/τ` write term takes over) — so
  "more frequent checkpoints" only helps if you are *below* optimal; past that, the lever is *cheaper*
  checkpoints (smaller `δ` via async/sharded), which lowers `√(2δ/M)`. Restart investment lowers `R/M`.
  Compare the per-failure loss terms: if `R > τ_opt/2`, restart dominates the loss → **faster restart
  wins**; if `R < τ_opt/2`, redone work dominates → **cheaper/more-frequent checkpoints win**. And since
  `M ∝ 1/N_GPUs`, restart (`R/M`, linear in scale) overtakes the checkpoint term (`√`, sub-linear) as you
  scale up — the real 100,000-GPU FT-HSDP result (44%→80% effective time via restart engineering) is
  direct confirmation of exactly this crossover.

- **Why can two GPU providers quoting the identical `$/GPU-hr` still produce different true
  cost-per-successful-run?**
  **Answer:** Because `$/GPU-hr` is only one factor in the formula, and it is not even the only "hidden
  cost" term — procurement/cluster quality (setup time, tuning support, unplanned downtime) behaves like
  a second failure-overhead multiplier that the nominal rate doesn't disclose. SemiAnalysis's TCO
  analysis argues this swing can be large enough that a "gold-tier" neocloud beats a hyperscaler on true
  cost even at an equal or higher sticker rate. The formula's `$/GPU-hr` term should be treated as a
  procurement decision to interrogate, not a fixed constant to plug in.

## Connections & what's next

This closes module 08. Every lesson fed this capstone: 01–02 gave the parallelism/collective vocabulary
that determines which comms pattern you're paying for; 03 gave MFU, the efficiency term; 04 gave
Young/Daly, the checkpoint-interval math inside the failure-overhead term; 05 gave the failure/restart
loop that `M` and `R` describe; 06 gave the object that expresses and re-rendezvouses the job you're
costing; 07 gave a second, easy-to-miss tax on the same MFU term. Together they are exactly what the
[Survive-a-failure lab](../practice/survive-a-failure/README.md) asks you to build — a real DDP job,
killed and recovered with and without checkpointing, priced with this lesson's formula — and exactly
what [checkpoint.md](../checkpoint.md)'s six pass criteria and depth probes test cold, especially
probe 6, "price it," and the "which lever" follow-up.

This cost-per-successful-run pattern — separating a naive resource-hours number from the *effective*
number failures and inefficiency actually cost you — is not formally required by anything later in this
course, since this module has `unlocks: []`. But it is the natural precursor to the fleet-scale GPU cost
and unit-economics work in Module 11, which arrives later in the course and is not yet enriched at the
time of writing. Treat this lesson as the conceptual on-ramp, not a hard prerequisite: the formula you
built here — `(1/MFU) × $/GPU-hr × (1 + failure_overhead)` — is the same shape of thinking Module 11
applies across a whole fleet rather than one run.

## References & further reading

- **Primary sources**
  - **The Llama 3 Herd of Models**, §3.3 (infrastructure, scaling, reliability) — effective-training-time
    framing, 466 interruptions / 78% hardware over the 54-day window; the anchor for this module's `M`
    estimate. <https://arxiv.org/abs/2407.21783>
  - **Meta — "Revisiting Reliability in Large-Scale Machine Learning Research Clusters"** — 150M+
    A100-GPU-hours across 4M jobs on two production clusters (1 to 4k+ GPUs); the broader empirical base
    behind this lesson's per-scale MTBF projections and Effective Training Time Ratio framing.
    <https://arxiv.org/abs/2410.21680>
  - **Meta — "Training LLMs with Fault Tolerant HSDP on 100,000 GPUs"** — the 44%→80% effective-training-
    time result from restart-focused (not checkpoint-frequency) engineering at 100K-GPU scale; the
    real-world confirmation of the restart-term-dominates-at-scale claim. <https://arxiv.org/abs/2602.00277>
  - MFU — lesson [08.3 · Communication as the bottleneck](03-communication-bottleneck.md) and the
    roofline/MFU work from module 03; peak FLOP/s from the H100 datasheet.
  - Young/Daly optimal checkpoint interval — lesson [08.4 · Checkpointing](04-checkpointing.md),
    `τ_opt = √(2Mδ)`, the source of the `√(2δ/M)` overhead term.

- **Real-world engineering blogs**
  - Databricks/MosaicML — "Mosaic LLMs (Part 2): GPT-3 quality for <$500k" — **2023 snapshot**.
    <https://www.databricks.com/blog/gpt-3-quality-for-500k>
  - Databricks — "Introducing MPT-7B" — **2023 snapshot**, $200K / 9.5 days / zero human intervention.
    <https://www.databricks.com/blog/mpt-7b>

- **Deeper dives**
  - SemiAnalysis — "How Much Do GPU Clusters Really Cost?" — **April 2026 snapshot**; the ClusterMAX
    TCO framework and "Grand Unifying Theory of Goodput," the closest existing industry artifact to
    this lesson's formula. <https://newsletter.semianalysis.com/p/how-much-do-gpu-clusters-really-cost>
  - BigScience/BLOOM-176B cost reporting (~€3–5M, **2022 snapshot**) — a mid-scale real budget with an
    explicit "including engineering support" caveat mapping onto this lesson's thesis.

> **Snapshot (2026-08).** `$/GPU-hr` used here (`$12` H100 on-demand) is illustrative; on-demand H100
> ranges ~$3–12 (neocloud → hyperscaler), and reserved/committed pricing is lower. The **formula is
> rate-agnostic** — the rate is a linear scalar. Re-pull your org's blended rate before quoting a number.
> Every dollar figure in this lesson (MosaicML, MPT-7B, BLOOM, SemiAnalysis) is a dated snapshot from
> its own source year — re-verify before citing in a live conversation.
