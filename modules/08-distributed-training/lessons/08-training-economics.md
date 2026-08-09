---
lesson: "08.8"
title: "Training economics"
module: "08"
concept: "Training economics"
status: not-started
est_time: "6h"
artifacts: []
---

# 08.8 · Training economics

> **Concept.** Turn the reliability and efficiency math of this module into one dollar figure: the true cost of a successful run.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Why this matters

This is the capstone, and it is *your* differentiator. Everyone in this module learned the mechanics —
parallelism, NCCL, checkpointing, elasticity. You are the one who can put a **dollar figure on a
training run that includes what failures actually cost**, and defend it. That skill is the input to
module 11 (GPU cost & unit economics), and it is what a "cost/observability" platform engineer brings to
a GPU-heavy org that a distributed-systems generalist does not.

The naive number — `GPU-hours × $/GPU-hr` — is a lie in two directions. It ignores **efficiency**
(a run at 35% MFU burns ~29% more hours than the same run at 45%) and it ignores **failure overhead**
(checkpoint writes, redone work, restart time). At scale those are not rounding errors: Llama 3's
16,384-GPU run hit a hardware-caused interruption roughly **every ~3 hours**. A cost model that assumes
zero failures is off by the exact amount your reliability engineering is worth — which is precisely the
thing you want to be able to price.

## What's new here

- **One formula that ties the module together.** MFU (03) sets the hours; Young/Daly (04) and recovery
  (05) set the overhead; the rate is FinOps. We assemble them into cost-per-*successful*-run.
- **The failure-overhead multiplier.** Not "add 10% for slack" — a derived fraction
  `≈ √(2δ/M) + R/M` from the checkpoint cost `δ`, MTBF `M`, and restart time `R`.
- **The two levers, priced.** Raising MFU and shrinking restart both cut `$/run`, but by different
  mechanisms and different magnitudes. You will be able to say which buys more per engineering-dollar.
- **Effective training time** as the industry framing (Llama 3 §3.3): the fraction of wall-clock that
  was productive. Your cost multiplier is just its inverse, in dollars.

## Core notes

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
and halves this part of the bill. This is the biggest lever and it is why lesson 03 came first.

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
detection and restart, not just frequent checkpoints.

### Step 3 — Assemble: true cost-per-successful-run

```
$/successful_run = GPU_hours_raw × $/GPU-hr × (1 + failure_overhead_fraction)

               = [ C_flops / (P_peak × MFU) ] × $/GPU-hr × (1 + √(2δ/M) + R/M)
                    └── MFU lever (÷) ──┘                    └─ reliability lever (×) ─┘
```

Everything in this module is in that line. Raising MFU shrinks the first bracket; shrinking `δ` or `R`
shrinks the multiplier. The two are *multiplicative*, so the MFU win and the reliability win compound.

### Effective training time (Llama 3 framing)

Meta reports "effective training time" — productive time ÷ wall-clock — as their headline reliability
number; Llama 3 sustained **>90%** across the 405B run despite 466 interruptions in a 54-day window
(417 unexpected, ~78% hardware). Effective-training-time `= 1 / (1 + failure_overhead_fraction)`. Your
cost multiplier is literally the reciprocal of that KPI, denominated in dollars. When you present to a
platform org, quote both: the SRE cares about effective-training-time, the CFO cares about the multiplier.

## Worked example

**Run.** 1024 × H100 SXM, a pretraining job that needs `T = 720` productive GPU-*hours per GPU*
(30 days wall-clock at 100% effective time), i.e. `GPU_hours_raw = 1024 × 720 = 737,280 GPU-hr`.
Rate `$/GPU-hr = $12` (2026 hyperscaler on-demand H100 — **snapshot, substitute your own**).

**Raw (naive) cost:** `737,280 × $12 = $8.85M`.

**Reliability inputs.**
- MTBF `M ≈ 45 hr`. *Where this comes from:* Llama 3's 16,384-GPU job saw 466 interruptions over
  ~1,296 hr → a per-GPU failure rate of `466 / (1,296 × 16,384) ≈ 2.2×10⁻⁵` per GPU-hr. Scaled to 1,024
  GPUs: `M ≈ 1 / (1024 × 2.2×10⁻⁵) ≈ 45 hr`. (Failure rate scales with GPU count.)
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
**Saves ~$2.08M/run** — four times the entire failure overhead. MFU is the dominant lever.

**Lever B — cut restart `R` 0.5 → 0.1 hr.** `R/M` drops `0.0111 → 0.0022`; multiplier `1.058 → 1.049`.
Saves `≈ $8.85M × 0.0089 = $79k/run`. Real, but an order of magnitude below the MFU win *at this scale*.
Rerun at 16,384 GPUs (`M ≈ 2.8 hr`) and the restart term `R/M` balloons — then faster restart is where
the money is.

## Practice

Feeds the **Survive-a-failure lab**: this is its payoff figure and a module-11 input.

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
handoff artifact into module 11.

## Self-check

**(a) Given MTBF `M`, checkpoint cost `δ`, and `$/GPU-hr`, what is the failure-overhead multiplier on a
run? Show the reasoning.**
**Answer:** At the Young/Daly optimum `τ_opt = √(2Mδ)`, overhead comes from checkpoint writes (`δ/τ`),
redone work (`τ/2M`), and restart (`R/M`). The first two minimize to `√(2δ/M)`; adding restart gives
`overhead_fraction ≈ √(2δ/M) + R/M`, so the **multiplier is `1 + √(2δ/M) + R/M`**. E.g. `δ=0.05 hr`,
`M=45 hr`, `R=0.5 hr` → `√(0.1/45) + 0.5/45 = 0.047 + 0.011 = 0.058` → `×1.058`. `$/GPU-hr` scales the
whole thing linearly but does not change the *fraction*.

**(b) How does raising MFU from 35% to 45% change cost-per-run?**
**Answer:** GPU-hours are `∝ 1/MFU`, so cost scales by `35/45 = 0.778` → **−22.2%**, holding the failure
multiplier fixed (MFU doesn't touch `δ`, `M`, or `R`). On a $8.85M raw run that is ~$2.08M/run — the
largest single lever in the model, because it shrinks the base hours that *everything else multiplies*.

**(c) Cheaper to invest in more-frequent checkpoints or faster restart — and on what does it depend?**
**Answer:** It depends on (1) whether you are already at `τ_opt` and (2) the ratio of `R` to `τ_opt/2`.
Checkpointing *more often than `τ_opt`* wastes money (the `δ/τ` write term takes over) — so "more
frequent checkpoints" only helps if you are *below* optimal; past that, the lever is *cheaper*
checkpoints (smaller `δ` via async/sharded), which lowers `√(2δ/M)`. Restart investment lowers `R/M`.
Compare the per-failure loss terms: if `R > τ_opt/2`, restart dominates the loss → **faster restart
wins**; if `R < τ_opt/2`, redone work dominates → **cheaper/more-frequent checkpoints win**. And since
`M ∝ 1/N_GPUs`, restart (`R/M`, linear in scale) overtakes the checkpoint term (`√`, sub-linear) as you
scale up — which is why 16K-GPU jobs invest in fast restart above all.

## Resources

1. **The Llama 3 Herd of Models**, §3.3 (infrastructure, scaling, reliability) — effective-training-time
   framing, 466 interruptions / 78% hardware over the 54-day window. <https://arxiv.org/abs/2407.21783>
2. **MFU** — lesson [08.3 · Communication as the bottleneck](03-communication-bottleneck.md) and the
   roofline/MFU work from module 03; peak FLOP/s from the H100 datasheet.
3. **Young/Daly optimal checkpoint interval** — lesson [08.4 · Checkpointing](04-checkpointing.md),
   `τ_opt = √(2Mδ)`, the source of the `√(2δ/M)` overhead term.

> **Snapshot (2026-08).** `$/GPU-hr` used here (`$12` H100 on-demand) is illustrative; on-demand H100
> ranges ~$3–12 (neocloud → hyperscaler), and reserved/committed pricing is lower. The **formula is
> rate-agnostic** — the rate is a linear scalar. Re-pull your org's blended rate before quoting a number.
