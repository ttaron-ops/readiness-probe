---
lesson: 06
title: "Commitment & procurement strategy for GPU capacity"
module: 11
concept: "coverage, spot, build-vs-buy-vs-rent"
status: not-started
est_time: "5 hrs"
prev: "05-unit-economics.md"
next: "07-neocloud-vs-hyperscaler.md"
artifacts: ["a coverage-optimization model that sets a commitment level against a demand distribution and outputs blended $/GPU-hr with a waste-vs-premium and utilization sensitivity table"]
sources: 15
---

# Commitment & procurement strategy for GPU capacity

[💰 11 — GPU cost and unit economics](../README.md) · ← [05 Unit economics](05-unit-economics.md) · → [07 Neocloud vs hyperscaler price gap](07-neocloud-vs-hyperscaler.md)

## Where this fits

Lesson 05 gave you the unit-cost identity — `unit_cost = (attributed_GPU_hours × blended_rate) / units_of_work` — and treated `blended_rate` as an input pulled from "the fleet's commitment mix." This lesson is where that number actually comes from. It is the procurement decision: how much capacity you commit to, for how long, under what obligation, and what you fall back to when demand exceeds or falls short of the commitment. Get it wrong and every unit-cost number downstream in lesson 05 is wrong no matter how carefully you did the attribution and the join.

It also closes the loop on module 10's capex-vs-cloud crossover. Own, commit, and rent are three points on one continuum, and this lesson is where you learn to place a real fleet on it with arithmetic rather than instinct. Module 03's capstone taught you that a `$/hr` figure is not a number until it carries four fields — price, tier, contract term, and date. This lesson is about the third of those four: what a *term* actually buys, what it costs when you are wrong, and how to size it.

## Why this matters

You already know how reserved instances and savings plans work in ordinary cloud: pre-commit spend or capacity, get a discount, eat some waste on the unused portion. That intuition transfers, but GPU procurement in 2025–2026 breaks one load-bearing assumption of general-cloud FinOps: **on-demand is not reliably available.** For CPUs you can burst to on-demand at a premium whenever you want; for H100/H200/GB200-class capacity, a reservation is frequently the only way to get the hardware at all. Procurement stops being purely a price-optimisation problem and becomes a **supply-securing** problem.

That is not just a war story — it is now encoded in the industry cost standard. FOCUS 1.3 introduced `ContractCommitmentBenefitCategory`, whose allowed values include **`Availability`**, defined by the spec as *"a contractual assurance of resource access and physical capacity,"* with capacity reservations and dedicated-host guarantees as its named use cases, and listed *separately from* `Discount` ("a financial reduction in the unit price or list rate"). The standard itself now says that a commitment can buy access rather than price. If you argue this point in a room, you are not offering an opinion; you are citing a schema.

The dollars are enormous and the decisions are semi-irreversible. Multi-year neocloud reservations are eight- and nine-figure balance-sheet commitments. CoreWeave's fiscal-2025 Form 10-K reported roughly **$5.13B of 2025 revenue** against a **revenue backlog of about $66.8B** and a weighted-average contract duration of roughly **five years** (up from the ~$15.1B of remaining performance obligations and ~4-year weighted-average duration disclosed at the time of the March 2025 S-1). That is a market in which buyers routinely sign away five years of capacity. Being able to size that commitment, defend the size, and state the utilisation at which it stops paying is the difference between a procurement recommendation and an expensive guess.

The cost of *not* knowing this is concrete and it lands twice. Commit too little and you either pay on-demand rates on a large share of the curve or — worse for GPUs — you cannot get the hardware at any price and your roadmap slips a quarter. Commit too much and you have converted a variable cost into a fixed one at exactly the moment demand softened, with no cancellation right and a depreciating asset behind it.

## What's new here (calibration)

- **You already know** generic RIs, savings plans, and spot mechanics, and the coverage/commitment tradeoff in the abstract. The general FinOps theory is not re-taught.
- **You already know** the capex crossover model (module 10), the two-ledger model (lesson 02), and checkpointed-training fault tolerance (module 08). These are referenced, not re-derived.
- **Genuinely new — the instrument catalogue with real terms.** Not "reservations exist" but: exactly what AWS Capacity Blocks, Savings Plans, GCP CUDs, Azure Reservations, and neocloud take-or-pay contracts each fix, for how long, with what exit, and what each looks like as FOCUS rows. Real durations, real caps, real cancellation policies.
- **Genuinely new — the break-even derived, not asserted.** The old rule of thumb ("commit to roughly P20–P50 of demand") is replaced by the exact result: **the optimal commitment level is the `d`-th percentile of the concurrent-demand distribution, where `d` is the discount fraction.** That falls out of a two-line marginal argument, and it is the single most useful thing in this lesson.
- **Genuinely new — the regret function.** Not just "over-committing is bad," but a closed-form expression for how many dollars you lose per point of shortfall in realised commitment utilisation, so the downside can be put in a memo.
- **Genuinely new — spot done properly.** The break-even is the same inequality as the commitment break-even, and the optimal checkpoint interval is the Young/Daly square-root formula, derived here with numbers rather than "checkpoint more often."
- **Genuinely new — three different utilisations, disentangled.** *Commitment* utilisation, *allocation* ratio, and *device* utilisation are three different denominators that all get called "utilisation," and conflating them is how a defensible model becomes a wrong one.

## Core concepts

### 1. What a commitment actually is: four independent knobs

Start by refusing the word "reservation," because it means at least three different products. Every capacity instrument fixes some subset of four things, and the interesting differences between instruments are exactly which subset:

```
   THE FOUR KNOBS A CAPACITY CONTRACT CAN TURN

   (1) PRICE       Is the unit rate fixed for the term?
                   → protects you from rate increases; costs you the
                     benefit of rate decreases (H100 cohort median fell
                     from >$7/hr in early 2024 to ~$3/hr by 2026)

   (2) QUANTITY    Is a specific amount of capacity physically held for you?
                   → this is the one that matters when supply is short,
                     and the one most "discount" instruments do NOT turn

   (3) TERM        How long, and at what granularity can it be resized?
                   → 1 hour ... 1 day ... 6 months ... 5 years

   (4) OBLIGATION  Do you owe the money whether or not you consume?
                   → "take-or-pay" is knob 4 set to ALWAYS.
                     Everything expensive about a commitment lives here.

   The classic mistake: buying an instrument that turns (1) and (4)
   when the problem you actually had was (2).
```

A **savings plan** turns knobs 1 and 4 and leaves 2 alone: you owe an hourly spend commitment, you get a discounted rate, and you get **no capacity guarantee whatsoever**. A **capacity reservation** turns knob 2 and (usually) 4, and may or may not turn 1. A **neocloud take-or-pay contract** turns all four at once, for years. Spot turns none of them — not even 1, since the price floats.

FOCUS gives you vocabulary for this, and it is worth learning because it is the vocabulary your billing data will speak. The `ContractCommitment` dataset (added in FOCUS 1.3, extended in 1.4) carries these dimensions:

| FOCUS column | Allowed values | What it tells you |
|---|---|---|
| `ContractCommitmentBenefitCategory` | `Discount`, `Entitlement`, `Availability`, `Other` | **Which knob you bought.** `Discount` = knob 1. `Availability` = knob 2 ("a contractual assurance of resource access and physical capacity"). |
| `ContractCommitmentModel` | `Continuous`, `Discontinuous` | `Continuous` = a flat hourly floor where any dip is *immediate, unrecoverable waste* (RIs, savings plans). `Discontinuous` = an aggregate bucket that tolerates spiky usage (EAs, minimum-spend deals). |
| `ContractCommitmentFulfillmentInterval` | `Hourly`, `Daily`, `Weekly`, `Monthly`, `Quarterly`, `Semi-Annual`, `Annual`, `Full Period`, `Transactional` | **How often the "use it or lose it" clock resets.** Continuous models are typically `Hourly`. This is the single field that decides how much your demand *shape* matters. |
| `ContractCommitmentPaymentModel` | `No Upfront`, `Partial Upfront`, `All Upfront` | The cash-flow shape. `All Upfront` carries the deepest discount and the largest sunk-cost trap. |
| `ContractCommitmentPaymentInterval` | `One-Time`, `Monthly`, `Quarterly`, `Semi-Annual`, `Annual`, `Custom` | Billing cadence — explicitly *not* the same as the fulfilment interval. A spend commitment can have an hourly fulfilment interval and a monthly payment interval. |
| `ContractCommitmentDurationType` | e.g. `1 Year`, `3 Years` | The *categorical* term, deliberately stable even if the record is exchanged or superseded mid-life. |
| `ContractCommitmentOfferCategory` | `Public`, `Negotiated` | Rate-card terms vs. a privately brokered deal. Relevant because you must never compare a negotiated rate against a rack rate (module 03 capstone, §8). |
| `ContractCommitmentLifecycleStatus` | `Proposed`, `Pending`, `Active`, `Exhausted`, `Expired`, `Canceled`, `Superseded` | The state machine. `Exhausted` means in-effect but fully consumed; `Superseded` means replaced mid-term by an exchange. |
| `ContractCommitmentDiscountPercentage` | decimal, 0.0–1.0 | The effective reduction against list. This is the `d` in every formula below. |

The `Continuous` / `Discontinuous` distinction is the one people get wrong. Read the spec's own words: a Continuous commitment is *"a flat, constant 'floor' of commitment… coverage is applied at a fixed rate per Contract Commitment Fulfillment Interval (usually hourly), and benefits are not carried over to subsequent intervals."* **Not carried over.** If you commit to 40 GPU-hours per hour and use 30 in one hour and 50 in the next, you do not net out. You waste 10 in hour one and pay on-demand for 10 in hour two. Everything in §4's math turns on this.

On the Cost and Usage side, each hour of consumption then carries:

| FOCUS column | Values | Meaning for GPU procurement |
|---|---|---|
| `PricingCategory` | `Standard`, `Committed`, `Dynamic`, `Other` | `Committed` = this hour was covered by a commitment discount. `Dynamic` = *"variable rate determined by the service provider… interruptible or low priority resources"* — i.e. **spot lives here**. |
| `CommitmentDiscountStatus` | `Used`, `Unused` | `Unused` rows are the waste rows. A commitment's realised utilisation is `SUM(Used) / (SUM(Used) + SUM(Unused))`. |
| `CommitmentDiscountCategory` | `Spend`, `Usage` | Spend-based (savings plans: $/hr of commitment) vs usage-based (RIs/CUDs: units of capacity). |
| `CapacityReservationId` / `CapacityReservationStatus` | string / `Used`\|`Unused` | The capacity-holding instrument, tracked **separately** from the discount instrument, because they genuinely are separate products. |

That last row is the schema making the same point this lesson opened with: FOCUS models "I hold this hardware" and "I get a discount on this hardware" as two different columns with two different `Used`/`Unused` accountings, because on a GPU fleet you can easily have one without the other.

### 2. The instrument catalogue, with real terms

Prices move; *terms* move much more slowly, and terms are what you actually have to reason about. Everything below is as of **August 2026** and should be re-verified before it goes in a contract memo, but the shapes are stable.

**AWS — EC2 Capacity Blocks for ML.** The purest "knob 2 only" instrument in the market, and the clearest evidence that hyperscalers themselves model GPU procurement as booking, not discounting. You choose a start date and a duration and pay the whole thing up front, like a hotel reservation:

- Duration: **1 to 14 days**, or multiples of 7 days up to **182 days** (so 21, 28, … 182).
- Booking horizon: start time up to **eight weeks** in the future.
- Size: up to **64 instances** per Capacity Block, up to **256 instances** across all your Capacity Blocks.
- Payment: **charged in full up front**, billed within roughly 5 minutes to 12 hours of purchase. The reservation sits in `payment-pending` until the charge clears, then moves to `scheduled`.
- There is no discount story here at all. You are buying certainty that 64 H100 nodes exist for you on a named date.

**AWS — Savings Plans.** Knobs 1 and 4, not knob 2. You commit to an hourly dollar figure for 1 or 3 years, at No / Partial / All Upfront. Two flavours matter for GPUs:

- **Compute Savings Plans** apply across instance family, size, AZ, region and OS. Observed 2026 discounts on accelerated instances run roughly **25–45%** depending on family and term (P5 has been observed near the top of that band on 1-year Compute plans).
- **EC2 Instance Savings Plans** lock you to one instance family in one region for a deeper nominal discount — but coverage for the newest GPU families is inconsistent, and P5/P5en have at times shown **no** 1-year Instance-Savings-Plan discount at all while the Compute plan did. **Check coverage for your exact family before assuming the deeper instrument applies.**
- Neither buys you a single GPU. A savings plan with no capacity is a discount on hardware you cannot get.
- **On-Demand Capacity Reservations (ODCRs)** are the separate knob-2 instrument, billable whether or not instances run, and can be combined with a Savings Plan so that the reserved-but-idle hours are still discounted.

**Google Cloud — Committed Use Discounts.** Two shapes: *resource-based* CUDs (commit to vCPU/RAM/GPU quantities in a region, 1 or 3 years) and *spend-based* CUDs (commit to $/hr). Reservations are the separate capacity-holding construct and can be attached so the CUD applies to reserved-but-idle capacity. Two GPU-specific facts worth carrying:

- **Sustained use discounts do not apply to GPUs attached to accelerator-optimised machine types.** The automatic "you ran it a lot this month" discount that softens CPU pricing simply is not there.
- **GPU CUD discounts are shallow.** Where memory-optimised families reach ~70% and general-purpose ~55%, published accelerator-family GPU commitment discounts have been observed in the **single digits to low teens** (on the order of ~8% for 1-year and ~11% for 3-year on a G2-class SKU). The mechanism is straightforward: the GPU silicon dominates the instance price, it is supply-constrained, and there is no reason for the provider to discount the scarce component. **Do not assume a GPU commitment carries a CPU-sized discount.** This single fact flips a lot of naive commitment models, because `d` in every formula below might be 0.10, not 0.45.

**Azure — Reserved VM Instances.** 1 or 3 years, All Upfront or Monthly, applied automatically to matching VMs. Discounts on GPU families have been observed around **~35% for 1-year**, with Azure marketing up to ~60% for reserved capacity on some SKUs. The exit terms are unusually well documented and worth knowing precisely, because "can I get out?" is the question a CFO will ask:

- Reservations are exchangeable within compute families, and refundable — with named exclusions (Red Hat plans, SUSE Linux plans, and pre-purchase plans are neither exchangeable nor refundable).
- **The total cancelled commitment cannot exceed USD 50,000 in a rolling 12-month window** per billing profile or enrolment. That is the hard ceiling on "we'll just unwind it."
- Microsoft states it is **currently not charging an early-termination fee, but that a 12% fee may apply in future.** Model the 12%.
- Refunds are prorated on remaining days and calculated at **the lower of your purchase price or the current price** — so if the market fell (and in GPUs it has), your refund falls with it.
- **From 1 February 2027**, reservations purchased after that date are not eligible for exchange where the service is covered by savings plans; reservations bought before retain one final exchange. A three-year GPU reservation signed in 2026 is a decision you will be living with in 2029.

**Neoclouds — take-or-pay committed contracts.** This is where large GPU fleets actually live, and the terms are structurally different from hyperscaler instruments:

- Term: commonly **2–5 years**, fixed price for the duration.
- Obligation: **take-or-pay** — you owe the minimum whether or not you consume. Typically no right to cancel early.
- Often an upfront reservation fee or deposit on top.
- Discounts against the provider's own on-demand rate have been observed around **30–60%** (CoreWeave publicly advertises up to ~60% off for reserved commitments; Nebius has cited up to ~35% for long-term commitments with larger GPU quantities).
- Frequently include **curtailment credits** or flex terms that compensate you for provider-side downtime — the mirror image of the take-or-pay obligation, and a clause worth reading, because it is the only place effective-availability risk (lesson 07) gets priced back to you.
- At CoreWeave, take-or-pay contracts were roughly **96% of 2024 revenue** — this is the normal shape of the market, not an aggressive outlier.

Put together:

| Instrument | Term granularity | Fixes price? | Holds capacity? | Obligation | Exit | Typical `d` (2026) |
|---|---|---|---|---|---|---|
| On-demand | per second/hour | no | no | none | walk away | 0 (baseline) |
| Spot / preemptible | per second, revocable | **no** (floats) | no | none | provider revokes | 0.4–0.9, unstable |
| AWS Capacity Block | 1–14 d, then ×7 d to 182 d | yes, for the block | **yes** | prepaid in full | none — it's prepaid | n/a (not a discount product) |
| AWS ODCR | hourly, cancellable | no | **yes** | billed while active | cancel | 0 (pair with an SP) |
| AWS Savings Plan | 1 or 3 yr | yes | **no** | hourly $ floor | none | ~0.25–0.45 |
| GCP resource CUD | 1 or 3 yr | yes | no (attach a reservation) | continuous | limited | **~0.08–0.11 on GPUs** |
| Azure Reserved VM | 1 or 3 yr | yes | partial | continuous | exchange/refund, $50k/12mo cap, possible 12% fee | ~0.35–0.6 |
| Neocloud take-or-pay | 2–5 yr | yes | **yes** | take-or-pay minimum | typically none | ~0.30–0.60 |
| Own the hardware (mod 10) | asset life (~6 yr) | n/a — capex | **yes** | sunk | resale market | n/a |

That six-year figure in the last row is not arbitrary: CoreWeave's FY2025 Form 10-K sets the estimated useful life of technology equipment including GPUs at **six years**, straight-line. Whether you own or rent, somebody is amortising the silicon over about six years, and the difference between instruments is mostly who carries that risk.

### 3. The commitment ladder against a demand curve

Here is the picture the whole lesson is about. Demand for GPUs on a growing platform is not flat and it is not smooth; it steps up as projects launch and it dips when they finish or get cancelled. Commitments are horizontal lines you draw under it, months in advance, on a forecast.

```
  CONCURRENT GPUs DEMANDED vs COMMITTED CAPACITY  (18 months)

  GPUs
  180 |                                          ,--.
      |                                        ,'    `.        ← on-demand
  160 |                                      ,'        `.        peak, paid
      |                                    ,'            `.      at r_od
  140 |                      ,-.         ,'                `.
      |                    ,'   `.     ,'                    `-.__
  120 |==================,'=======`---'============================  T3 (12mo, +40)
      |                ,'                                        ▓▓▓
  100 |              ,'                                          ▓▓▓ ← STRANDED
      |            ,'                                            ▓▓▓   after M15
   80 |==========,'=================================================  T2 (24mo, +40)
      |        ,'
   60 |      ,'
      |    ,'
   40 |==,'==========================================================  T1 (36mo, 40)
      | ,'
   20 |'
      +----+----+----+----+----+----+----+----+----+----+----+----+--
       M1   M3   M5   M7   M9  M11  M13  M15  M17  M19  M21  M23
       |         |                                    |
       T1 signed T2 signed                            T3 expires
                      T3 signed (M7)                  ← ladder roll point

  ── demand curve         ══ committed floor (paid regardless)
  ▓▓ committed-and-unused = the money you set on fire

  READ IT THIS WAY:
   • Below the lowest ══ line: cheapest hours you will ever buy.
   • Between the top ══ line and the demand curve: on-demand premium,
     paid only when consumed — and only IF capacity exists to buy.
   • Above the demand curve, below a ══ line (the ▓▓ region): pure loss.
     This is `CommitmentDiscountStatus = 'Unused'` in your FOCUS export.
   • STAGGERED EXPIRIES (T1 36mo, T2 24mo, T3 12mo) mean you get a
     re-pricing decision every year instead of one enormous cliff.
```

Two structural points that the picture makes and prose does not.

First, **the stranded region is created by a forecast, not by an event.** Nothing goes wrong at month 15; demand simply comes in below where it was projected when T3 was signed at month 7. The loss was locked in eight months earlier. This is why the sizing question is a *distributional* question — what you are really choosing is how much of the demand distribution's upper tail you are willing to pay for whether or not it shows up.

Second, **laddering is not an aesthetic choice.** With three tranches expiring in different years, at any moment roughly one third of your committed base is up for renewal, so you re-price against the current market annually and never face a single cliff where 100% of your capacity and 100% of your rate exposure renew simultaneously. In a market where the H100 cohort median fell by more than half in two years, a single 3-year all-in commitment is a bet that prices will not fall — a bet that has recently been a losing one.

### 4. Coverage math from first principles

Now derive the sizing rule instead of quoting one.

**Setup and notation.** Over a term of `T` hours, let `D(t)` be the number of GPUs concurrently demanded at time `t`. Let `r_od` be the on-demand rate and `r_c` the committed rate, and define the discount fraction

```
    d = 1 - r_c / r_od          (so r_c = (1 - d) · r_od)
```

Commit to a flat `C` GPUs for the whole term (a `Continuous` commitment with an `Hourly` fulfilment interval — benefits do not carry across hours). Then:

```
    committed spend      = C · T · r_c                      (owed regardless)
    on-demand spend      = r_od · ∫ max(0, D(t) − C) dt     (paid when D > C)
    consumed-on-commit   = ∫ min(D(t), C) dt                (what you actually used
                                                             of what you committed)
```

**Define commitment utilisation** — and note that this is a *procurement* number, not an engineering one:

```
    u = ∫ min(D(t), C) dt  /  (C · T)      ∈ [0, 1]
      = "of the GPU-hours I bought outright, what fraction did I have a
         workload for?"
      = SUM(EffectiveCost WHERE CommitmentDiscountStatus='Used')
        ÷ SUM(EffectiveCost WHERE CommitmentDiscountStatus IN ('Used','Unused'))
```

**The break-even.** Savings versus the all-on-demand counterfactual:

```
    savings = r_od · ∫min(D,C)dt          [what those hours would have cost]
              − C · T · r_c               [what you actually owed]

            = r_od · [ u·C·T  −  C·T·(1−d) ]

            = r_od · C · T · [ u − (1 − d) ]

    ⟹  the commitment pays  ⟺  u  >  1 − d
```

**That is the whole rule, and it is worth memorising:** a commitment beats on-demand exactly when the fraction of committed hours you actually consume exceeds one minus the discount. Nothing else enters it — not the absolute price, not the fleet size, not the term length.

```
    d = 0.10  (GCP GPU CUD)      →  break-even u = 90%   ← brutally tight
    d = 0.25                     →  break-even u = 75%
    d = 0.35  (Azure 1-yr)       →  break-even u = 65%
    d = 0.45  (AWS Compute SP)   →  break-even u = 55%
    d = 0.60  (neocloud reserved)→  break-even u = 40%
```

Look at the first row. A ~10% GPU CUD requires you to consume **90%** of every committed hour just to break even. That is why the GCP accelerator CUD number matters so much: on a shallow discount, a commitment is a bet you almost certainly lose unless demand is essentially flat.

**The marginal rule, which is the one you actually size with.** Committing to a flat `C` is a portfolio of `C` independent decisions: should I commit the *k*-th GPU? The *k*-th committed GPU is consumed exactly when demand is at least `k`, i.e. for a fraction

```
    p_k = P( D ≥ k )
```

of the term. Apply the break-even to that single GPU: commit the *k*-th GPU iff `p_k > 1 − d`. Now let `F` be the CDF of concurrent demand, so `p_k = 1 − F(k)`:

```
    commit GPU k   ⟺   1 − F(k) > 1 − d   ⟺   F(k) < d

    ⟹  C* = the d-th percentile of the concurrent-demand distribution
```

**Commit up to the `d`-quantile of demand.** If your discount is 40%, commit to P40 of concurrent demand — the level exceeded 60% of the time. If your discount is 10%, commit to P10. If it is 60%, commit to P60. This is the classic newsvendor result wearing a FinOps hat, and it replaces the folk rule ("commit to P20–P50, ish") with something you can compute from a Prometheus range query and defend in a meeting.

Note what it says about the *shape* of demand: a fleet with a hard, flat baseline has a demand CDF that jumps at the baseline, so every quantile below the jump lands on the same number, and the answer is "commit the baseline" for any `d` above a threshold. A fleet with smoothly ramping demand has no such flat spot, and the discount genuinely moves the answer.

Here is the same result drawn as the picture that makes it obvious — the **demand duration curve**, which is just `D(t)` sorted from highest to lowest:

```
  DEMAND DURATION CURVE — sort every hour of the term by GPUs demanded

  GPUs
  180 |*
      | **
  160 |   ***
      |      ****                    Each horizontal slice is ONE GPU.
  140 |          ****                Its width = the fraction of hours
      |              ****            it is needed = p_k.
  120 |                  ****
      |                      ***     Commit the slice iff p_k > 1 − d.
  100 |························***·································
      |                          **  ← the (1−d) FRACTION-OF-TERM line
   80 |                            *    for d = 0.40 → cut at 60% of hours
      |                             *
   60 |█████████████████████████████|**                 ← C* = P40 ≈ 60 GPUs
      |█████ COMMIT THIS ███████████|  ***
   40 |█████████████████████████████|     ****
      |█████████████████████████████|         ******
   20 |█████████████████████████████|               *********
      +----+----+----+----+----+----+----+----+----+----+----+----+
       0%  10%  20%  30%  40%  50%  60%  70%  80%  90% 100%
                      fraction of hours in the term ──▶
                                    ↑
                          cut here: 1 − d = 0.60

  Everything LEFT of the cut and BELOW the curve: commit it (used > 60%
      of the term, so it clears the u > 1−d bar).
  Everything RIGHT of the cut: rent it. Those GPU-slices are needed too
      rarely to pay for outright at this discount.

  MOVE THE CUT: a deeper discount (d = 0.60) slides the line LEFT to 40%,
      which raises C* — the better the discount, the more of the peak is
      worth owning. A shallow GPU CUD (d = 0.10) slides it RIGHT to 90%,
      collapsing C* to the hard floor of demand.
```

**The regret function.** Sizing is only half the job; a memo needs the downside. Suppose you commit `C*` on a forecast implying utilisation `u_plan`, and realised utilisation comes in at `u_real`. Then

```
    regret = C* · T · r_od · ( u_plan − u_real )        [dollars of savings lost]

    and the commitment is outright worse than on-demand when

    u_real < 1 − d      with loss = C* · T · r_od · ( (1−d) − u_real )
```

Per point of shortfall, you lose `C* · T · r_od / 100` dollars. That is a one-line sensitivity a finance partner can hold in their head: *"every percentage point of utilisation shortfall on this 60-GPU, 12-month commitment costs us `60 × 8760 × $4.90 / 100 ≈ $25,750`."*

Finally, the **blended rate** that lesson 05 consumes:

```
    blended_rate = [ C·T·r_c  +  r_od·∫max(0, D−C)dt ]  ÷  ∫D(t)dt
                     └── owed regardless ──┘  └─ burst ─┘      └─ consumed ─┘
```

Note the asymmetry that trips people up: the numerator includes committed hours you did **not** consume, but the denominator counts only hours you **did**. That is deliberate and it is the honest construction — it is why the blended rate rises when a commitment strands, which is exactly the signal you want. If you divide by committed hours instead, waste becomes invisible.

### 5. Spot for GPUs: the same inequality, plus an optimal checkpoint interval

**Where spot is and is not admissible.** This is a binary, decided by one property: can the workload survive a kill-and-resume without a human or a customer noticing?

- **Admissible:** checkpointed training (module 08), batch and offline inference, hyperparameter sweeps, data generation, evaluation harnesses.
- **Inadmissible:** synchronous latency-serving. Not "risky" — inadmissible. An interruption drops in-flight requests and there is no guarantee replacement capacity exists at any price. A scheduler that does not enforce this at the workload-class level will eventually place a serving pod on spot during a capacity crunch, and the resulting incident is fully predictable in advance.

**The economics.** Define `d = 1 − r_spot/r_od` as before, and define **goodput** `g` as the fraction of paid wall-clock time that produces retained forward progress. Then cost per unit of useful work is `r_spot / g` on spot and `r_od / 1` on-demand, so:

```
    spot wins  ⟺  r_spot / g  <  r_od  ⟺  (1 − d) < g  ⟺  g > 1 − d
```

**The same inequality as §4.** A commitment pays when the fraction of committed hours you use beats `1 − d`; spot pays when the fraction of paid hours that survive beats `1 − d`. In both cases the discount is buying you tolerance for a specific kind of yield loss, and the break-even is the complement of the discount. Once you see that, the whole procurement surface becomes one rule applied to different yields.

**Deriving `g`.** Two independent losses eat goodput, and they pull the checkpoint interval in opposite directions:

```
   TIMELINE OF ONE SPOT-INTERRUPTED TRAINING RUN

   Δ = checkpoint interval        δ = checkpoint write time
   M = mean time between interruptions (M = 1/f)
   ρ = reload + requeue + re-warm time after a preemption

   ├──work──┤δ├──work──┤δ├──work───✂ PREEMPTED
   |◀── Δ ──▶|         |            |
                       |◀ lost work ▶|      ← expected Δ/2 of progress
                       last good ckpt         thrown away
                                    |
                                    ├─── ρ ───┤├──work──┤δ├──
                                     reload,   resume from
                                     requeue,  last checkpoint
                                     re-warm

   LOSS 1 — checkpoint overhead:  you spend δ out of every (Δ+δ) writing
            state.  Fraction retained = Δ / (Δ + δ) ≈ 1 − δ/Δ.
            → shrinking Δ makes this WORSE.

   LOSS 2 — interruption loss: per interruption you throw away Δ/2 of
            progress (uniform arrival within the interval) and burn ρ
            recovering. Expected cost per unit time = (Δ/2 + ρ)/M.
            → shrinking Δ makes this BETTER.

   g(Δ) ≈ (1 − δ/Δ) · (1 − (Δ/2 + ρ)/M)
```

The two losses trade off, so there is an interior optimum. Differentiating and keeping leading terms gives the classical result — Young's 1974 first-order approximation, refined by Daly (2006):

```
    Δ*  ≈  sqrt( 2 · δ · M )
```

**Worked, with units carried.** A 70B-parameter training job checkpointing sharded optimizer state to a parallel filesystem:

```
    δ (checkpoint write)            =  90 s
    f (spot interruptions)          =  0.30 /hr   →  M = 1/0.30 = 3.333 hr = 12,000 s
    ρ (reload + requeue + re-warm)  =  420 s      (7 min: 2 min requeue,
                                                   3 min state load, 2 min
                                                   re-establish collectives)
    r_od  = $4.90 /GPU-hr           r_spot = $1.47 /GPU-hr   →  d = 0.70

    Δ* = sqrt(2 · 90 · 12000) = sqrt(2,160,000) = 1,470 s ≈ 24.5 min

    g  = (1 − 90/1470) · (1 − (735 + 420)/12000)
       = (1 − 0.0612)  · (1 − 0.0963)
       = 0.9388 × 0.9037
       = 0.848            → 84.8% goodput

    break-even needs g > 1 − d = 0.30.   0.848 ≫ 0.30  →  SPOT WINS, hugely.

    effective cost per useful GPU-hour:
        spot       = 1.47 / 0.848 = $1.734
        on-demand  = 4.90 / 1.00  = $4.900
        saving     = 64.6%   (vs the 70% sticker discount — the 5.4 pp gap
                              IS the interruption tax, and now you can name it)
```

**And the case where spot loses**, which is the more interesting one to be able to construct:

```
    A capacity-crunch region with f = 4 /hr (M = 900 s), a model whose
    checkpoint takes δ = 300 s, and ρ = 600 s.

    Δ* = sqrt(2 · 300 · 900) = sqrt(540,000) = 735 s ≈ 12.2 min

    g  = (1 − 300/735) · (1 − (367 + 600)/900)
       = (0.592)       · (1 − 1.074)
       = 0.592 × (−0.074)        ← NEGATIVE

    A negative goodput term means the recovery cost per interruption exceeds
    the mean time between interruptions: the job is preempted, on average,
    before it finishes recovering from the previous preemption. It makes
    NO forward progress at any price. Spot is not "expensive" here — it is
    non-functional, and the correct output of the model is "do not run this
    workload on spot in this region," not a dollar figure.
```

The lever, stated precisely: `Δ*` scales as `sqrt(δ · M)`, and `g` improves with smaller `δ`. So the highest-leverage engineering investment for spot economics is **faster checkpoints**, not more frequent ones — asynchronous/overlapped checkpointing that cuts `δ` from 300 s to 30 s improves both terms at once, whereas simply halving `Δ` improves loss 2 and worsens loss 1. Automated detect-and-resume (MosaicML's "Node Doctor" and "Watchdog", §Real-world use cases) is an attack on `ρ`, the third term, and it is why teams running that tooling can tolerate interruption rates that would make an un-automated team's `g` collapse.

### 6. Three utilisations, and why conflating them produces wrong answers

"Utilisation" is used for at least three different ratios in a GPU cost conversation, and they multiply rather than substitute. Keep them apart:

```
   THE THREE DENOMINATORS  (they compose, they do not compete)

   u  COMMITMENT UTILISATION      ── procurement
      consumed committed GPU-hours / committed GPU-hours
      "Did I have a workload for the capacity I bought?"
      Source: FOCUS CommitmentDiscountStatus Used vs Unused
      Owner: procurement / FinOps.  Lever: commitment sizing.

   a  ALLOCATION RATIO            ── platform
      GPU-hours bound to a pod / GPU-hours available in the fleet
      "Did the scheduler place work on the capacity I hold?"
      Source: pod-resources API (lesson 04) vs node capacity
      Owner: platform team.  Levers: bin-packing, fragmentation (L04),
             queue policy (L08), idle reclaim (L03).

   s  DEVICE UTILISATION          ── engineering
      SM_ACTIVE-weighted busy time / allocated time
      "Did the silicon do work while the pod held it?"
      Source: DCGM DCGM_FI_PROF_GR_ENGINE_ACTIVE (lesson 05 — NOT
              DCGM_FI_DEV_GPU_UTIL, which measures kernel residency)
      Owner: model/serving engineers.  Levers: batching, precision,
             kernel efficiency.

   COMPOSED:

       $ per GPU-hour of USEFUL WORK  =   r_c
                                        ─────────
                                        u · a · s

   Each factor is a different team's problem. A procurement memo that
   quotes r_c / s (skipping a) or r_c / u (skipping both) will understate
   true cost, and it will point the fix at the wrong org.
```

Run the numbers on a plausible fleet and the point lands hard:

```
    r_c = $2.10/GPU-hr committed
    u = 0.92   (well-sized commitment, 8% Unused)
    a = 0.71   (fragmentation + queue gaps — lesson 04)
    s = 0.46   (SM_ACTIVE across a mixed training/serving fleet)

    $/useful-GPU-hr = 2.10 / (0.92 × 0.71 × 0.46)
                    = 2.10 / 0.3005
                    = $6.99

    versus the on-demand alternative at r_od = $4.90 with u = 1.0 (no
    commitment to strand), the same a and s:

                    = 4.90 / (1.00 × 0.71 × 0.46) = 4.90 / 0.3266 = $15.00

    The commitment still wins — but the headline "$2.10/hr" understated
    true cost by 3.3×, and 70% of the gap is a = 0.71 and s = 0.46, which
    procurement cannot fix and does not own.
```

**The blunt version, which is the checkpoint's own depth probe:** a reserved GPU at 30% device utilisation is worse per unit of useful work than an on-demand GPU at 90%. At `r_c = $1.80` and `r_od = $3.20`, that is `1.80/0.30 = $6.00` against `3.20/0.90 = $3.56`. A 44% rate discount is annihilated by a 3× utilisation gap. **A commitment discount is never the number to compare; the number to compare is the discount divided by the yield you will actually sustain over the term.**

### 7. Supply as the product, and the risks that only appear at term length

Three risks that a one-period model cannot see, and that a multi-year GPU commitment definitely has.

**(a) Generational obsolescence.** The commitment fixes your *rate*; it does not fix your *competitiveness*. A 3-year H100 commitment signed in mid-2024 at what was then a good rate was, by 2026, a contract to keep paying above the current H100 market while B200-class parts delivered more tokens per dollar. Your `$/useful-GPU-hour` did not move, but your competitors' fell. The quantitative form: over the term, the relevant comparison is not `r_c` versus `r_od(t=0)` but `r_c` versus the *expected path* of `r_od(t)` — and in a market where the H100 cohort median more than halved in two years, assuming a flat path is an active bet. Laddering (§3) is the hedge, and shorter terms at a shallower discount are frequently the better risk-adjusted trade.

**(b) The depreciation/contract-duration mismatch, which is your counterparty's risk and therefore also yours.** CoreWeave depreciates GPU equipment straight-line over **six years** while its weighted-average customer contract runs about **five**. The asset must earn beyond the contracted window for the economics to close. That is not an accusation — it is the normal shape of an infrastructure business — but it tells you what your provider needs from the back half of the asset's life, and it is why re-contracting older silicon at a lower rate (rather than retiring it) is the expected behaviour. As a buyer, it is also your best argument for a mid-term re-price clause.

**(c) Take-or-pay is a covenant-adjacent obligation.** A multi-year take-or-pay commitment is a fixed future cash outflow that appears in the same conversations as debt: runway modelling, covenant headroom, downside scenarios. The relevant question at signing is not "is the rate good" but "at what demand level does this obligation become the binding constraint on the company," and the regret function in §4 is precisely the tool that answers it.

Against those risks sits the reason people sign anyway, and it is knob 2. FOCUS names it `Availability`. AWS built an entire product (Capacity Blocks) around it with no discount attached. When frontier-class accelerators are allocation-constrained, the counterfactual to a commitment is not "pay more" — it is **"do not run the workload."** A procurement team optimising purely for discount percentage will lose the capacity race to a team that signed earlier for supply reasons, and the discount-optimiser will be right about the arithmetic and wrong about the outcome.

### 8. Build vs buy vs rent, as a procedure

Three variables decide it: demand **predictability**, commitment **duration** you can honestly forecast, and sustained **utilisation** (`u · a · s`).

```
   THE PROCUREMENT DECISION TREE

   START: characterise the demand distribution over the horizon
          (percentiles of concurrent GPUs, from your own metrics)
     │
     ├─ Is there a floor exceeded ≥ (1−d) of the time?
     │    NO ──▶ commit nothing. Rent on-demand / spot.
     │           A shallow-discount GPU CUD (d≈0.10) needs u>90%;
     │           almost no real fleet clears that on a rising curve.
     │    YES ─▶ continue
     │
     ├─ Is the workload interruption-tolerant?
     │    YES ─▶ compute g = (1−δ/Δ*)(1−(Δ*/2+ρ)/M), Δ*=sqrt(2δM)
     │           g > 1−d_spot ?  ──▶ serve that tranche on SPOT
     │           g ≤ 1−d_spot ?  ──▶ it is not spot-eligible in practice
     │    NO ──▶ this tranche needs guaranteed capacity; continue
     │
     ├─ Do you need CAPACITY or just a PRICE?
     │    CAPACITY ─▶ knob 2 instruments only: Capacity Blocks / ODCR /
     │                neocloud reserved cluster. A savings plan will not
     │                conjure a GPU.
     │    PRICE ────▶ savings plan / CUD, layered on top.
     │
     ├─ Can you forecast ≥ 24 months with the demand floor holding?
     │    NO ──▶ 1-year or shorter, laddered. Accept the smaller d.
     │    YES ─▶ continue
     │
     ├─ Is sustained u·a·s high enough that owning beats renting at the
     │  module-10 crossover, AND do you have the capital, the lead time,
     │  and the datacentre skills (module 10)?
     │    YES ─▶ OWN the floor. Lowest marginal cost, you control supply,
     │           you carry obsolescence and ~6-year depreciation.
     │    NO ──▶ NEOCLOUD TAKE-OR-PAY for the floor. Someone else carries
     │           the capital and the DC ops; you carry the obligation.
     │
     └─ ALWAYS: rent the tail. Commit the d-quantile, never the peak.
```

Most real fleets end up a **portfolio**: own or take-or-pay the confident floor, savings-plan or CUD the next tranche where a price-only instrument suffices, spot the interruption-tolerant middle, and on-demand the spiky tail. The portfolio exists because the four knobs are independent and different parts of the demand curve need different subsets of them.

## Perspectives

**Supply-chain / procurement view.** In generic cloud FinOps a reservation is purely a discount instrument: get the coverage wrong and you fall back to on-demand and eat a premium, but you always get the machine. For frontier GPUs in 2025–2026 that fallback frequently does not exist. The commitment is not optional risk-hedging against price; it is the mechanism by which the hardware exists for you. The tell that this is structural rather than anecdotal is that both the standards body and the largest hyperscaler have productised it: FOCUS carries `Availability` as a benefit category distinct from `Discount`, and AWS ships Capacity Blocks — a prepaid, non-discounted, date-bounded booking product. A procurement team that treats a GPU reservation as a savings plan, optimising discount percentage, will be outbid for capacity by a team that understood it was buying knob 2.

**Finance / treasury view.** A multi-year take-or-pay commitment is a balance-sheet decision, not a line item. It is a committed future cash outflow regardless of whether demand materialises, so it enters covenant calculations, runway planning, and downside scenarios the same way debt does — and unlike an office lease, the underlying asset depreciates on a hardware refresh cycle. The exit terms are what treasury will ask about first, which is why the specifics matter: Azure's rolling **$50,000 / 12-month** cancellation ceiling and its signalled **12% future early-termination fee** are the difference between "we can unwind this" and "we cannot." Neocloud take-or-pay contracts typically have no cancellation right at all. Read the curtailment-credit clause; it is the only place your provider's downtime gets priced back to you.

**Workload-scheduling view.** The spot-eligible / spot-ineligible line is a hard binary determined by kill-and-resume tolerance, and it has to be enforced at the workload-class level in the scheduler rather than left to a per-team decision. But the more interesting scheduling insight from this lesson is the Young/Daly optimum: checkpoint interval is a *tunable* that the platform can set on the workload's behalf from two measurable inputs (`δ`, the observed checkpoint write time; `M`, the observed MTBF for that node pool). A platform that measures both and sets `Δ* = sqrt(2δM)` automatically is delivering a materially better spot yield than one that ships a hard-coded "checkpoint every hour."

**Risk-management view.** The regret function is a stress test in the same spirit as a risk desk stress-testing a portfolio, and it answers the question a signature actually needs answered: *at what realised utilisation does this commitment become the more expensive option, not merely a less-good one?* The answer is `u < 1 − d`, it is computable before signing, and it is monitorable afterwards from a single FOCUS query over `CommitmentDiscountStatus`. Turning "we think utilisation will be fine" into "we breach at 55% and we are currently at 78%, tracked monthly" is the entire discipline.

## Real-world use cases

- **CoreWeave, Form S-1 (SEC EDGAR, filed 2025-03-03) and FY2025 Form 10-K.** The public record of what a GPU capacity commitment looks like on someone's books. The S-1 disclosed roughly **$15.1B of remaining performance obligations as of 31 December 2024** with a weighted-average contract duration of about **four years**; the FY2025 10-K reported **~$5.13B of 2024→2025 revenue (up ~170%)**, a **revenue backlog of ~$66.8B**, a weighted-average contract duration of roughly **five years**, and an estimated useful life for technology equipment including GPUs of **six years**, straight-line. What it shows: multi-year take-or-pay is the *normal* shape of large GPU procurement (~96% of CoreWeave's 2024 revenue came from take-or-pay contracts), and the six-year-depreciation-versus-five-year-contract gap is the structural tension §7(b) describes.
- **AWS EC2 Capacity Blocks for ML.** A hyperscaler productising knob 2 with no discount attached: reserve GPU capacity for a future window of 1–14 days or multiples of 7 up to 182 days, starting up to 8 weeks ahead, up to 64 instances per block and 256 across blocks, **paid in full up front** with the reservation held in `payment-pending` until the charge clears. What it shows: even AWS models GPU procurement as booking a slot rather than buying a discount — direct vendor evidence for this lesson's central reframing.
- **Databricks / MosaicML, "Introducing MPT-7B."** A 440-GPU training run over 9.5 days during which the platform detected and handled **4 hardware failures** and auto-resumed with **zero human intervention**. What it shows: real numbers for the `ρ` term in the goodput model. Four events across ~4,180 GPU-days is a low `f`, and — more importantly — an automated recovery loop drives `ρ` toward the floor, which is what makes an interruption-tolerant capacity strategy economically viable rather than merely survivable.
- **Databricks / MosaicML, "Training Stable Diffusion from Scratch for $50k (Part 2)."** Describes "Node Doctor" and "Watchdog," platform components that automatically detect failed or preempted nodes and resume training without a human. What it shows: `ρ` is an *engineered* quantity, not a fact of nature. The team could run large jobs on interruptible capacity economically because they had driven recovery cost toward zero — the exact lever §5 identifies as higher-leverage than tightening the checkpoint interval.
- **Google Cloud accelerator CUDs.** Sustained-use discounts do not apply to GPUs on accelerator-optimised machine types, and published GPU commitment discounts have been observed in the single-digit to low-teens range (on the order of 8% for 1-year and 11% for 3-year on a G2-class SKU) against ~55–70% for CPU/memory families. What it shows: `d` is a *per-SKU* input, not a category constant. At `d = 0.10` the break-even is `u > 90%`, which almost no real fleet on a rising demand curve will clear — so the correct recommendation for that SKU is frequently "commit nothing," and the model tells you so.

## Worked example

A platform team runs a fleet with a **40-GPU steady floor** and spiky peaks to **100 GPUs**. Concurrent demand over a representative month: 40 GPUs are needed essentially 100% of hours; the increment from 40 to 100 is needed progressively less, averaging about 30% of hours across those 60 marginal GPUs. The month has 720 hours. Rates, flagged as an **August 2026 snapshot, illustrative shapes** — substitute your own quotes:

```
    r_od    = $3.20 /GPU-hr      on-demand
    r_c     = $1.80 /GPU-hr      1-year committed  →  d = 1 − 1.80/3.20 = 0.4375
    r_spot  = $0.90 /GPU-hr      spot              →  d_spot = 1 − 0.90/3.20 = 0.7188
```

**Step 0 — what does the break-even rule say before we compute anything?**

```
    d = 0.4375  →  commit only where u > 1 − 0.4375 = 0.5625
    Marginal rule: C* = the 43.75th percentile of concurrent demand.

    Demand CDF: P(D = 40) ≈ 0.70 for the flat floor region and the
    marginal GPUs above 40 each have p_k ≈ 0.30.
    → F(40) ≈ 0.0 (demand is never below 40), F(k>40) ≈ 0.70.
    → The d-quantile with d = 0.4375 lands inside the flat floor: C* = 40.

    The rule says: commit the floor, nothing above it. Note the marginal
    check directly — GPU #41 has p_41 ≈ 0.30, and 0.30 < 0.5625, so it
    fails the bar by a wide margin. No arithmetic below can rescue it.
```

**Option A — commit the floor (40), burst on-demand.**

```
    committed GPU-hrs   = 40 × 720            = 28,800 h   (owed regardless)
    committed spend     = 28,800 × $1.80      = $51,840
    committed consumed  = 28,800 h            (floor is always used)
    u_A                 = 28,800/28,800       = 1.00   ✔ (≫ 0.5625)

    peak GPU-hrs        = 60 × 720 × 0.30     = 12,960 h
    on-demand spend     = 12,960 × $3.20      = $41,472

    total               = $93,312
    consumed GPU-hrs    = 28,800 + 12,960     = 41,760 h
    blended rate        = 93,312 / 41,760     = $2.234 /GPU-hr
    commitment waste    = 0 h                 ($0)
```

**Option B — over-commit to 70 GPUs.** The intuition being tested is "more coverage is safer."

```
    committed GPU-hrs   = 70 × 720            = 50,400 h
    committed spend     = 50,400 × $1.80      = $90,720

    committed consumed  = floor 40 always      = 28,800 h
                        + GPUs 41–70 at ~30%   = 30 × 720 × 0.30 = 6,480 h
                        = 35,280 h
    u_B                 = 35,280 / 50,400      = 0.700
    UNUSED              = 50,400 − 35,280      = 15,120 h  →  $27,216 burned

    Check the rule: is u_B = 0.700 > 1 − d = 0.5625?  Yes — so the BLENDED
    commitment still beats all-on-demand in aggregate. But that is the wrong
    test. The right test is MARGINAL: GPUs 41–70 individually run at
    u = 0.30 < 0.5625, so each of those 30 GPUs is a loss. The profitable
    floor is subsidising them inside the average.

    peak above 70       = 30 × 720 × ~0.15     = 3,240 h
    on-demand spend     = 3,240 × $3.20        = $10,368

    total               = $101,088
    consumed GPU-hrs    = 35,280 + 3,240       = 38,520 h
    blended rate        = 101,088 / 38,520     = $2.624 /GPU-hr
```

**Option B costs $7,776 more than Option A** while serving *less* demand, and its blended rate is 17% worse. This is the concrete refutation of "more coverage is always safer": past the marginal break-even, every additional committed GPU converts a discount into waste. Notice how the aggregate `u` test was misleading and the marginal test was not — **always size on the marginal rule, then report the aggregate.**

**Option C — commit the floor, serve the interruption-tolerant peak on spot.** The peak here is hyperparameter sweeps and checkpointed fine-tuning, so it is spot-eligible. Compute the goodput first rather than assuming it:

```
    δ = 60 s   (small models, fast checkpoints)
    f = 0.5 /hr  →  M = 7,200 s
    ρ = 300 s

    Δ* = sqrt(2 × 60 × 7200) = sqrt(864,000) = 930 s ≈ 15.5 min
    g  = (1 − 60/930) · (1 − (465 + 300)/7200)
       = 0.9355 × 0.8938 = 0.836

    break-even: g > 1 − d_spot = 0.2812.   0.836 ≫ 0.2812  →  spot admissible.

    effective spot rate per useful hour = 0.90 / 0.836 = $1.077

    peak GPU-hrs (paid)  = 12,960 h × $0.90    = $11,664
    but USEFUL hrs       = 12,960 × 0.836      = 10,834 h

    total  = $51,840 + $11,664                 = $63,504
    consumed-useful GPU-hrs = 28,800 + 10,834  = 39,634 h
    blended USEFUL rate     = 63,504 / 39,634  = $1.602 /GPU-hr

    (The naive figure, dividing by paid rather than useful hours, is
     63,504/41,760 = $1.521 — a 5% understatement. Report the useful
     denominator; the interruption tax is real and it belongs in the number.)
```

**Sensitivity 1 — commitment utilisation.** Option A assumed the 40-GPU floor holds all month. What if it does not?

| Realised `u` on the committed 40 | Committed spend | Consumed committed h | Effective committed rate | vs on-demand $3.20 |
|---|---|---|---|---|
| 1.00 | $51,840 | 28,800 | $1.80 | 44% cheaper |
| 0.80 | $51,840 | 23,040 | $2.25 | 30% cheaper |
| **0.5625** | $51,840 | 16,200 | **$3.20** | **break-even (= 1 − d) ✔** |
| 0.45 | $51,840 | 12,960 | $4.00 | 25% **more expensive** |
| 0.30 | $51,840 | 8,640 | $6.00 | 88% more expensive |

The break-even row lands exactly on `1 − d = 0.5625` and reproduces `r_od` to the cent, which is the arithmetic check that the derivation in §4 is right. **Regret per point of shortfall** = `40 × 720 × $3.20 / 100 = $921.60/month`, or about **$11,059 per point per year** — the number that goes in the memo.

**Sensitivity 2 — device utilisation, the multiplier procurement does not control.**

| Sustained SM_ACTIVE `s` on the committed floor | Effective $/useful-GPU-hr (committed) | On-demand at `s = 0.90` |
|---|---|---|
| 0.90 | 1.80 / 0.90 = **$2.00** | $3.56 |
| 0.60 | 1.80 / 0.60 = **$3.00** | $3.56 |
| **0.5625** | 1.80 / 0.5625 = **$3.20** | $3.56 |
| 0.50 | 1.80 / 0.50 = **$3.60** | $3.56 — **inverted** |
| 0.30 | 1.80 / 0.30 = **$6.00** | $3.56 |

Note the coincidence and do not be fooled by it: the commitment ties on *device* utilisation at 0.5625 only because we happened to compare against an on-demand GPU running at 100%. Against on-demand at a realistic `s = 0.90`, the crossover is `1.80 / (3.20/0.90) = 0.506` — the commitment stops winning below about **51% sustained SM_ACTIVE**. Two different break-evens, two different owners, and a memo must say which one it means.

**Recommendation, written the way it should be handed over.**

> *Commit the 40-GPU floor on a 1-year term at $1.80/GPU-hr and serve the peak on spot with a 15-minute checkpoint interval (Option C): blended cost per useful GPU-hour $1.60 against $2.23 for on-demand bursting and $2.62 for the over-committed alternative. Do not commit above 40 — GPU #41 is needed only ~30% of hours against a 56.25% break-even, so every GPU above the floor loses money at this discount. The recommendation is conditional on two floors, both monitored monthly: **commitment utilisation ≥ 56%** (below which the commitment is worse than on-demand; regret is ~$11,059/year per point of shortfall) and **sustained SM_ACTIVE ≥ 51%** on the committed portion (below which the utilisation gap eats the rate advantage). Spot eligibility is conditional on measured interruption rate staying below roughly 3/hr for this checkpoint profile; re-derive `Δ* = sqrt(2δM)` if either δ or M moves by more than 2×. Rates are an August 2026 snapshot at a matched tier and term; re-verify before signing.*

## Practice

Feeds [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md):

1. **Build the coverage model.** Given an hourly concurrent-GPU demand series and rates for committed / on-demand / spot, compute total cost and blended $/GPU-hr as a function of commitment level `C`. Plot cost vs `C` and identify the minimum. Then verify numerically that the minimum lands at the **`d`-th percentile** of the demand distribution — this is the acceptance test that your model implements the marginal rule rather than a heuristic.
2. **Emit the regret curve.** For your chosen `C*`, produce a table of realised commitment utilisation from 1.00 down to 0.20 in steps, with effective committed rate and dollars of regret at each step, and mark the `u = 1 − d` break-even row. Confirm that the break-even row reproduces `r_od` exactly; if it does not, your discount fraction and your rates disagree.
3. **Add the three-utilisation decomposition.** Parameterise `u`, `a`, and `s` separately and report `r_c / (u·a·s)`. State which team owns each factor and which lever moves it. Show at least one case where the ranking of two options flips when `a` and `s` are included.
4. **Implement the spot goodput model.** Given `δ`, `M`, and `ρ`, compute `Δ* = sqrt(2δM)` and `g`, decide spot-vs-on-demand against `g > 1 − d_spot`, and show how the decision moves when `δ` is halved (faster checkpoints) versus when `Δ` is halved at fixed `δ` (more frequent checkpoints). Include at least one parameter set where `g` goes non-positive and confirm your model returns "not spot-eligible" rather than a dollar figure.
5. **Map to FOCUS.** For your chosen instrument, write out the `ContractCommitment` row you would expect — `ContractCommitmentType`, `Category`, `Model`, `FulfillmentInterval`, `PaymentModel`, `DurationType`, `OfferCategory`, `DiscountPercentage` — and the shape of the corresponding `CostAndUsage` rows including `PricingCategory` and `CommitmentDiscountStatus`. This is the row you will actually query in lesson 10; producing it here means lesson 10's schema is not starting from a blank page.
6. **Write the one-page recommendation** for the 40/100 fleet as a function of demand predictability and expected utilisation, with both floors stated explicitly (commitment-utilisation floor and device-utilisation floor), the regret-per-point figure, and the dated pricing basis. Cite the module 10 crossover for the own-versus-rent boundary.

**Acceptance criteria.** The model takes a demand series and a rate card and emits: cost-vs-`C` with the optimum, the `d`-quantile check, a regret table with a correct break-even row, a three-factor utilisation decomposition, a spot goodput decision with a derived `Δ*`, and a recommendation carrying two numeric floors and a date.

## Common pitfalls

1. **Treating GPU procurement like generic cloud RIs, where on-demand is always the fallback.** For frontier-class parts the fallback frequently is not there, which is why FOCUS carries `Availability` as a benefit category distinct from `Discount` and why AWS shipped a prepaid, non-discounted Capacity Block product. *Symptom:* a procurement plan that optimises discount percentage and has no answer to "and if we cannot get the hardware?" *Correction:* separate the knob-2 decision (capacity) from the knob-1 decision (price) and make them independently.
2. **Buying a savings plan when you needed a capacity reservation.** A savings plan discounts hardware; it does not conjure it. *Symptom:* good coverage metrics and an empty cluster. *Correction:* check `CapacityReservationId` coverage separately from `CommitmentDiscountId` coverage — FOCUS models them as different columns precisely because they are different products.
3. **Assuming a GPU commitment carries a CPU-sized discount.** Accelerator CUDs on GCP have been observed in the single digits to low teens while CPU/memory families reach 55–70%, and sustained-use discounts do not apply to accelerator-optimised GPUs at all. *Symptom:* a model built with `d = 0.45` that shows a comfortable margin and a real contract at `d = 0.10` that needs 90% utilisation to break even. *Correction:* pull `d` per SKU from the actual quote before running any of this arithmetic.
4. **Sizing on aggregate utilisation instead of the marginal rule.** Option B in the worked example has an aggregate `u = 0.70` that clears the `0.5625` bar, and it still loses $7,776, because the profitable floor is subsidising 30 unprofitable marginal GPUs inside the average. *Symptom:* "our commitment utilisation is 70%, we're fine" while the blended rate rises. *Correction:* size with `p_k > 1 − d` per marginal GPU, i.e. `C* = d`-quantile; report aggregate `u` only as a monitoring metric afterwards.
5. **Comparing a commitment discount without dividing by the utilisation you will actually sustain.** The single most testable trap in this module. A reserved GPU at 30% SM_ACTIVE (`$1.80/0.30 = $6.00`) is worse per unit of useful work than on-demand at 90% (`$3.20/0.90 = $3.56`). *Correction:* always report `r / (u·a·s)`, and name the owner of each factor.
6. **Conflating the three utilisations.** Commitment utilisation, allocation ratio, and device utilisation multiply; using one where you mean another silently changes the answer by a factor of two or more and points the remediation at the wrong team. *Correction:* label every ratio with its numerator, its denominator, and its data source.
7. **Putting latency-serving workloads on spot.** Disqualifying, not a tradeoff — an interruption drops in-flight requests with no guaranteed replacement capacity. *Correction:* enforce the spot-eligibility binary at the workload-class level in the scheduler, not per-team.
8. **Treating the checkpoint interval as fixed, or "just checkpoint more often."** Shrinking `Δ` at fixed `δ` improves the interruption loss and *worsens* the checkpoint-overhead loss; there is an interior optimum at `Δ* = sqrt(2δM)`. *Symptom:* checkpoint frequency tuned by superstition, and goodput that gets worse when you tighten it. *Correction:* measure `δ` and `M`, compute `Δ*`, and spend engineering effort on shrinking `δ` and `ρ` — those improve both terms.
9. **Reporting spot savings against paid hours rather than useful hours.** In Option C the difference is 5% ($1.521 vs $1.602 blended). *Correction:* divide by `g`-adjusted hours; the interruption tax is a real cost and hiding it makes the spot decision look better than it is.
10. **Ignoring the exit terms until you need them.** Azure caps cancelled commitment at USD 50,000 per rolling 12 months, has signalled a possible 12% early-termination fee, refunds at the *lower* of purchase price or current price, and closes general exchange eligibility for post-1-February-2027 purchases. Neocloud take-or-pay contracts typically have no cancellation right at all. *Correction:* write the exit terms into the recommendation memo alongside the rate, and model the 12%.
11. **Assuming the rate you locked stays competitive.** The commitment fixes your price, not the market's. The H100 cohort median more than halved in two years. *Correction:* ladder expiries so roughly one third of the base re-prices annually, and treat a single long all-in commitment as an explicit bet that prices will not fall.

## Self-check

- **Derive the condition under which a commitment beats on-demand, and state it in one sentence.** *Answer:* Over a term of `T` hours committing `C` GPUs, you owe `C·T·r_c` regardless and consume `u·C·T` of it, where `u` is commitment utilisation. Those same hours on-demand would have cost `r_od·u·C·T`. Savings `= r_od·C·T·[u − (1−d)]` where `d = 1 − r_c/r_od`. So the commitment pays **exactly when `u > 1 − d`**: the fraction of committed hours you actually consume must exceed one minus the discount. Nothing else — not fleet size, not term length, not absolute price — enters the condition. At `d = 0.10` you need 90% utilisation; at `d = 0.60` you need 40%.
- **What commitment level is optimal against a known demand distribution, and why is "commit the baseline" only sometimes right?** *Answer:* The *k*-th committed GPU is used a fraction `p_k = P(D ≥ k)` of the term, so it pays iff `p_k > 1 − d`, i.e. `F(k) < d`. Therefore **`C*` is the `d`-th percentile of the concurrent-demand distribution**: at `d = 0.40`, commit to P40. "Commit the baseline" is right when the demand CDF has a flat floor — every quantile below the jump lands on the same number, so the answer is insensitive to `d`. On a smoothly ramping curve there is no such flat spot and `d` genuinely moves `C*`, which is why a shallow GPU CUD (`d ≈ 0.10`) can push the right answer down to almost nothing.
- **A vendor offers a 3-year commitment at 55% off. Name three things that could still make it the wrong purchase.** *Answer:* (1) **Yield.** 55% off needs `u > 45%` on commitment utilisation alone, and the true comparison is `r_c/(u·a·s)` — a fleet at `u=0.9, a=0.7, s=0.45` is paying 3.5× its sticker per useful hour, and the commitment only wins if the on-demand alternative suffers the same `a` and `s`. (2) **Generational risk.** The rate is fixed for three years; the market is not. The H100 cohort median more than halved in two years, so a 55% discount against a 2026 rate can be above the 2028 market. Laddered shorter terms at a shallower discount are frequently better risk-adjusted. (3) **Exit and obligation.** If it is take-or-pay with no cancellation right, the regret is unhedged; if it is an Azure-style reservation, the $50k rolling cancellation cap, the signalled 12% termination fee, and refunds at the lower of purchase-or-current price bound how much you can unwind. Also check whether it buys capacity at all (`Availability`) or only price (`Discount`) — a discount on hardware you cannot obtain is worth nothing.
- **Spot GPUs are 70% cheaper. Work out whether to use them, and what you would engineer to improve the answer.** *Answer:* Spot wins iff goodput `g > 1 − d = 0.30`, where `g ≈ (1 − δ/Δ)·(1 − (Δ/2 + ρ)/M)` with `δ` = checkpoint write time, `Δ` = checkpoint interval, `ρ` = reload+requeue+re-warm, `M = 1/f` = mean time between interruptions. The interval that maximises `g` is **`Δ* = sqrt(2δM)`** (Young/Daly). Worked: `δ=90 s`, `f=0.3/hr` (`M=12,000 s`), `ρ=420 s` → `Δ* = 1,470 s ≈ 24.5 min`, `g = 0.9388 × 0.9037 = 0.848`. Since `0.848 ≫ 0.30`, spot wins; effective cost `$1.47/0.848 = $1.734` vs `$4.90` on-demand, a 64.6% saving against a 70% sticker — the 5.4 pp gap is the interruption tax. To improve it, attack `δ` and `ρ`, not `Δ`: halving `Δ` at fixed `δ` makes the checkpoint-overhead term worse, whereas asynchronous checkpointing (smaller `δ`) and automated detect-and-resume (smaller `ρ`, as in MosaicML's Node Doctor/Watchdog) improve both terms. And spot is inadmissible for synchronous latency serving regardless of the arithmetic.
- **Why did committing 70 GPUs cost more than committing 40, at the same $1.80/GPU-hr rate?** *Answer:* Because the extra 30 GPUs above the confident 40-GPU floor were consumed only ~30% of hours, well under the `1 − d = 56.25%` break-even, so 15,120 of the 50,400 committed hours went unused — $27,216 burned, against $31,104 of on-demand premium avoided, for a net loss of $7,776 and a blended rate 17% worse ($2.624 vs $2.234) while serving *less* demand. The generalisable trap is that Option B's *aggregate* utilisation was 0.700, which clears the 0.5625 bar and looks healthy; the profitable floor was subsidising the unprofitable margin inside the average. **Size on the marginal rule (`p_k > 1 − d`), report the aggregate.**
- **Distinguish the three ratios that get called "utilisation," and show how they compose.** *Answer:* **Commitment utilisation `u`** = consumed committed GPU-hours ÷ committed GPU-hours, from FOCUS `CommitmentDiscountStatus` `Used`/`Unused`, owned by procurement, fixed by right-sizing the commitment. **Allocation ratio `a`** = GPU-hours bound to pods ÷ GPU-hours available, from the pod-resources API against node capacity, owned by the platform team, fixed by bin-packing, fragmentation work (lesson 04), queue policy (lesson 08) and idle reclaim (lesson 03). **Device utilisation `s`** = SM_ACTIVE-weighted busy ÷ allocated, from DCGM `DCGM_FI_PROF_GR_ENGINE_ACTIVE` (not `DCGM_FI_DEV_GPU_UTIL`, which measures kernel residency), owned by model and serving engineers, fixed by batching and precision. They **multiply**: `$/useful-GPU-hr = r_c / (u·a·s)`. At `r_c = $2.10, u = 0.92, a = 0.71, s = 0.46` the true figure is `$6.99` — 3.3× the sticker, with most of the gap in factors procurement neither owns nor can fix.
- **Which FOCUS columns tell you what a commitment actually bought, and how would you monitor the break-even from billing data alone?** *Answer:* `ContractCommitmentBenefitCategory` distinguishes `Discount` (price) from `Availability` ("a contractual assurance of resource access and physical capacity" — capacity reservations, dedicated hosts) — that column answers "which knob did we buy." `ContractCommitmentModel` (`Continuous` vs `Discontinuous`) plus `ContractCommitmentFulfillmentInterval` tell you whether unused capacity in one interval is unrecoverable waste (Continuous/Hourly: yes, benefits are not carried over) or nets out across a longer window. `ContractCommitmentDiscountPercentage` gives you `d`. To monitor: compute `u = SUM(EffectiveCost WHERE CommitmentDiscountStatus='Used') ÷ SUM(EffectiveCost WHERE CommitmentDiscountStatus IN ('Used','Unused'))` per `CommitmentDiscountId` per billing period, and alert when `u` approaches `1 − ContractCommitmentDiscountPercentage`. Separately track `CapacityReservationStatus` `Used`/`Unused`, because a reservation can be idle while the discount is fully consumed elsewhere.

## Connections & what's next

This lesson closes the loop lesson 05 opened: the blended rate `r_blend` that lesson's unit-cost formula treats as an input is exactly the output of the coverage-optimisation and spot-eligibility decisions made here, and the `r/(u·a·s)` decomposition is what ties it back to lesson 02's two ledgers and lesson 04's fragmentation. It extends module 10's capex-vs-cloud crossover into a three-way build-vs-buy-vs-rent framing with the neocloud take-or-pay contract as a distinct middle option, and it inherits module 03's discipline that no dollar figure is a number without its tier, term, and date — this lesson is the deep treatment of the *term* field.

It also plants the FOCUS vocabulary you will need twice more. Lesson 08's internal chargeback rate has to recover exactly the fixed cost this lesson's commitments create, including the `Unused` rows, and lesson 10's capstone schema carries `CommitmentDiscountId`, `CommitmentDiscountStatus`, `PricingCategory`, and the whole `ContractCommitment` dataset as core columns — the row you sketched in Practice step 5 is the row that schema emits.

Next: **lesson 07 — Decomposing the neocloud-vs-hyperscaler price gap** takes the same normalisation discipline built here (never compare across mismatched bases; always divide by realised yield before ranking) and applies it across *vendors* rather than across *terms*, turning two incomparable stickers into one fully-loaded, availability-adjusted, utilisation-adjusted number.

## References & further reading

**Primary sources**

- **FOCUS Specification, `ContractCommitment` dataset column definitions** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/tree/main/specification/datasets/contract_commitment/columns> — read the individual column files for the exact allowed values quoted in §1: `contractcommitmentbenefitcategory.md` (`Discount`/`Entitlement`/`Availability`/`Other`), `contractcommitmentmodel.md` (`Continuous`/`Discontinuous`, and the "benefits are not carried over" requirement), `contractcommitmentfulfillmentinterval.md`, `contractcommitmentpaymentmodel.md`, `contractcommitmentoffercategory.md`, `contractcommitmentlifecyclestatus.md`. *Verified against the spec repository at `main`, commit `7f19ccb`, read 2026-08-17.*
- **FOCUS Specification, `CostAndUsage` commitment columns** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/tree/main/specification/datasets/cost_and_usage/columns> — `pricingcategory.md` (note that `Dynamic` is defined to cover *"interruptible or low priority resources"*, i.e. spot), `commitmentdiscountstatus.md` (`Used`/`Unused`), `commitmentdiscountcategory.md` (`Spend`/`Usage`), `capacityreservationstatus.md`. These are the columns the monitoring query in Self-check runs against.
- **FOCUS Specification CHANGELOG** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/CHANGELOG.md> — version history confirming the `ContractCommitment` dataset landed in **1.3 (December 2025)** and was extended in **1.4 (June 2026)** with the payment/lifecycle/applicability columns. Read this before quoting any column name; the spec moves.
- **AWS EC2 Capacity Blocks for ML documentation** — <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-blocks.html> and the pricing/billing page at `.../capacity-blocks-pricing-billing.html` — the source for the 1–14 day / multiples-of-7-to-182-day durations, the 8-week booking horizon, the 64-per-block / 256-total instance caps, and the prepaid `payment-pending` → `scheduled` flow. *`docs.aws.amazon.com` is egress-blocked from this environment; the figures above were taken from search-result extracts of these pages rather than a direct read. Re-verify on the live page before quoting in a contract memo.*
- **Google Cloud, Committed use discounts for Compute Engine** — <https://cloud.google.com/compute/docs/instances/committed-use-discounts-overview> and the accelerator-optimized pricing page — the source for resource-based vs spend-based CUDs, the 1/3-year terms, and the fact that **sustained use discounts do not apply to GPUs on accelerator-optimized machine types**. *`docs.cloud.google.com` is egress-blocked here; the single-digit-to-low-teens accelerator CUD figures are from search-result extracts and secondary pricing analyses, not a direct read of the rate card. Pull your own SKU's number before modelling.*
- **Azure reservation exchange and refund policy** — <https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/cost-management-billing/reservations/exchange-and-refund-azure-reservations.md> — read directly from the docs source repository (the rendered `learn.microsoft.com` page is egress-blocked). Source for the **USD 50,000 per rolling 12-month** cancellation ceiling, the *"currently not charging an early termination fee, but in the future there might be a 12% early termination fee"* statement, refunds calculated at the **lower of purchase price or current price**, the Red Hat / SUSE / pre-purchase exclusions, and the **1 February 2027** change to exchange eligibility.
- **CoreWeave, Form S-1 (SEC EDGAR, filed 2025-03-03)** — <https://www.sec.gov/Archives/edgar/data/1769628/000119312525044231/d899798ds1.htm> — the ~$15.1B remaining-performance-obligations figure as of 31 December 2024, the ~4-year weighted-average contract duration at filing, the six-year straight-line GPU depreciation policy, and the take-or-pay contract structure. *`sec.gov` is egress-blocked from this environment; figures verified via search extracts of the filing and of CoreWeave's investor-relations releases. **This corrects the previous version of this lesson**, which stated a "$66B–$88B" backlog "as of December 31, 2024" — that conflated the S-1's $15.1B RPO with the much later FY2025 backlog.*
- **CoreWeave, FY2025 Form 10-K and Q4/FY2025 results release** — <https://investors.coreweave.com/> — 2025 revenue ~$5.13B (up ~170%), revenue backlog ~$66.8B, RPO ~$60.7B as of 31 December 2025, weighted-average contract duration ~5 years, six-year useful life for technology equipment including GPUs. Same egress caveat as above.

**Real-world engineering blogs**

- **AWS News Blog, "Announcing Amazon EC2 Capacity Blocks for ML"** — <https://aws.amazon.com/blogs/aws/announcing-amazon-ec2-capacity-blocks-for-ml-to-reserve-gpu-capacity-for-your-machine-learning-workloads> — what it shows: the hyperscaler's own framing of GPU reservation as booking a future window rather than buying a discount. The clearest single piece of vendor evidence for §7's "supply is the product" argument.
- **Databricks/MosaicML, "Introducing MPT-7B"** — <https://www.databricks.com/blog/mpt-7b> — what it shows: 4 hardware failures across 440 GPUs over 9.5 days with zero human intervention. Real numbers for the `f` and `ρ` terms in the goodput model, and evidence that automated recovery is what makes interruption-tolerant capacity economically usable.
- **Databricks/MosaicML, "Training Stable Diffusion from Scratch for $50k with MosaicML (Part 2)"** — <https://www.databricks.com/blog/stable-diffusion-2> — what it shows: "Node Doctor" and "Watchdog" as engineered attacks on the `ρ` term. Recovery cost is a design variable, not a constant.
- **SemiAnalysis, ClusterMAX ratings and GPU rental price index** — <https://clustermax.semianalysis.com/> and <https://gpu-index.semianalysis.com/> — what they show: the tier-and-term structure of GPU pricing and the market-level price path (H100 cohort median falling from >$7/hr in early 2024 to roughly $3/hr by 2026) that makes §7(a)'s generational-risk argument quantitative rather than rhetorical. **Time-sensitive — refresh every figure before citing.**

**Deeper dives**

- **J. W. Young, "A first order approximation to the optimum checkpoint interval," CACM 17(9), 1974**, and **J. T. Daly, "A higher order estimate of the optimum checkpoint interval for restart dumps," Future Generation Computer Systems 22(3), 2006** — the origin and the refinement of `Δ* = sqrt(2δM)` used in §5. Daly's higher-order form matters when `δ/M` is not small; at the GPU-training ratios in this lesson the first-order form is within a few percent.
- **FinOps Foundation, Rate Optimization capability** — <https://www.finops.org/framework/capabilities/rate-optimization/> — the general coverage-optimisation framework this lesson specialises to GPUs. *`finops.org` is egress-blocked from this environment; the framework's shared-cost distribution vocabulary (proportional / fixed / even-split) referenced in lesson 08 was taken from search extracts rather than a direct read.*
- **Kubernetes scheduling and eviction documentation** — <https://kubernetes.io/docs/concepts/scheduling-eviction/> — how spot/preemptible node interruption is actually surfaced and handled at the orchestration layer, and where the spot-eligibility binary from §5 has to be enforced.
