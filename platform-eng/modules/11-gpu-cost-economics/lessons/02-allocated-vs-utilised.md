---
lesson: 02
title: "Allocated vs utilised cost: the two ledgers"
module: 11
concept: "two-ledger cost model"
status: not-started
est_time: "3 hrs"
prev: "01-attribution-models.md"
next: "03-idle-detection.md"
artifacts: ["a two-ledger (allocated vs utilised) computation for a fleet slice with the gap % and dollar figure, added to the module deliverable"]
sources: 12
---

# Allocated vs utilised cost: the two ledgers

> Module: [💰 11 — GPU cost and unit economics](../README.md) · Deliverable: [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md)

## Where this fits

Lesson 01 gave you the attribution ladder and showed that exactly one rung is lossy. It also fell out of that ladder that GPU cost splits into two quantities with completely different provenance: **allocation cost**, derived from rungs 1–3 and exact in every sharing regime, and **utilisation cost**, derived from rung 4 and only as accurate as the regime permits. This lesson takes that split and turns it into an accounting model — two books you always close together, with a named, owned, dollar-denominated gap between them.

You already have the collection machinery. Module 04 built the pod-resources join and the two reconciliation identities. Module 05 built the honest utilisation signal and, critically, the correct integral: `sum_over_time(x[W]) × Δ/3600`, not `avg_over_time(x[W]) × W_hours`. This lesson does not rebuild any of that. It does three new things: it defines the ledgers precisely enough to survive an audit, it maps each ledger onto the industry's cost columns (FOCUS `BilledCost` / `EffectiveCost` / `ListCost` / `ContractedCost`, with the spec's real definitions inlined), and it settles the policy question that everything downstream depends on — **which ledger you charge and which you report**, with the incentive arithmetic that proves it.

Everything after this inherits the model. Lesson 03 splits the gap into idle states with owners. Lesson 04 adds the bucket that is neither allocated nor idle — paid-for capacity nobody can schedule. Lesson 05 divides a ledger by an application counter to get unit cost, and the choice of ledger is the difference between a direct and a fully-loaded number. Lesson 08 designs the chargeback policy on top of this split. Lesson 09 shows that OpenCost, by construction, reports only the first ledger.

## Why this matters

The bill you pay and the value you got are two different numbers, and almost every GPU cost mistake is a confusion between them.

Report only **allocated** and you look fully committed while burning cash on warm, idle silicon: the number is unimpeachable and tells you nothing about whether any of it was worth buying. Report only **utilised** and procurement never sees the capacity it must actually contract for, so commitment sizing collapses and you end up short at exactly the moment demand arrives. Report a single blended "utilisation %" and you have hidden which of the two moved — a jump from 40% to 55% could be an efficiency win or could be teams reserving less capacity while doing the same work, and those call for opposite responses.

The size of the thing is what makes the discipline non-negotiable. On a fleet where a GPU-hour costs `r`, the gap is `(1 − mean utilisation) × allocated GPU-hours × r`, and the mean utilisation term on real fleets is routinely well under half. That is not a rounding error you can defer; it is usually the largest single controllable line in an AI infrastructure budget. Whether it is 30% or 60% on *your* fleet is exactly what this lesson makes you able to state, defend, and re-derive next month.

And there is a subtler failure this lesson prevents. The obvious reaction to a large gap is "so charge people for what they use." That is the wrong policy, and the arithmetic in §6 shows why: charging utilisation makes over-reserving free, so every rational tenant reserves generously and runs lightly, the fleet's allocated hours climb, and the platform eats the difference. **Charge allocated, report utilised** is not a compromise — it is the only pairing where the tenant's incentive points at the behaviour you want.

## What's new here (calibration)

- **Already yours (skip):** `SM_ACTIVE` vs `GPU_UTIL` as the utilisation signal and why the latter lies (module 05); the pod-resources → UUID join (module 04); the `sum_over_time × Δ/3600` integral and the `avg_over_time` bug (module 05); the four sharing regimes and the exactness of the allocation side (lesson 01).
- **New angle 1 — the ledgers as an accounting model, not a pair of metrics.** Precise definitions with units, a conservation identity that must hold in dollars, and a rule for what may and may not be subtracted from what.
- **New angle 2 — three nested areas, not two.** Paid ⊃ allocated ⊃ utilised ⊃ *productive*. The gap between utilised and productive has a different owner and a different fix from the gap between allocated and utilised, and collapsing them is how you tell a well-tuned inference team they are 90% wasted.
- **New angle 3 — the asymmetry, derived.** Reservation is a step function over wall-clock time; consumption is the integral of a rate bounded above by 1. The gap is therefore structural, not accidental, and you can write down the two independent factors that produce it: *how long held* and *how busy while held*.
- **New angle 4 — the ledgers mapped onto FOCUS columns**, with the spec's actual definitions of `BilledCost`, `EffectiveCost`, `ListCost` and `ContractedCost` inlined, plus the precise reason a utilisation ledger has no FOCUS home and must be an extension.
- **New angle 5 — the charge-allocated / report-utilised policy, proved rather than asserted**, via the tenant's optimisation problem under each billing rule.
- **New angle 6 — where the leading tool actually stops.** OpenCost computes `GPUHours = request × hours` and `GPUCost = GPUHours × CostPerGPUHr`; it *does* collect a utilisation average and a shared-device flag, and uses neither in the cost. That is a sharper and more useful statement than "OpenCost ignores utilisation."

## Core concepts

### 1. Four nested quantities, three named gaps

Start with the picture, because the whole lesson is one containment relation.

```
  ONE PHYSICAL GPU, ONE WINDOW W (hours).  Areas are GPU-hours.
  Every outer ring is money you already paid.

  ┌─────────────────────────────────────────────────────────────────────┐
  │ ① PAID          = G × W GPU-hours,  cost = G × W × r                │
  │   you own or rent the device for the whole window, unconditionally  │
  │                                                                     │
  │   ┌─────────────────────────────────────────────────────────────┐   │
  │   │ ② ALLOCATED  = Σ_pods  share(p) × hours_held(p)              │   │
  │   │   some pod held the device (or a fraction of it)             │   │
  │   │                                                              │   │
  │   │   ┌──────────────────────────────────────────────────────┐   │   │
  │   │   │ ③ UTILISED  = ∫ SM_ACTIVE(t) dt   over held time      │   │   │
  │   │   │   the SMs had at least one warp resident             │   │   │
  │   │   │                                                      │   │   │
  │   │   │   ┌──────────────────────────────────────────────┐   │   │   │
  │   │   │   │ ④ PRODUCTIVE = ∫ PIPE_TENSOR_ACTIVE(t) dt    │   │   │   │
  │   │   │   │   the tensor pipes did the arithmetic you    │   │   │   │
  │   │   │   │   bought this class of GPU to do             │   │   │   │
  │   │   │   └──────────────────────────────────────────────┘   │   │   │
  │   │   │        ▲ GAP C: "busy but not at the work"           │   │   │
  │   │   └────────┼─────────────────────────────────────────────┘   │   │
  │   │            ▲ GAP B: "held but the SMs were empty"            │   │
  │   └────────────┼──────────────────────────────────────────────────┘   │
  │                ▲ GAP A: "paid for but nobody held it"                 │
  └───────────────────────────────────────────────────────────────────────┘

  ── THE THREE GAPS, WITH OWNERS AND FIXES ────────────────────────────────
  GAP A  ① − ②   unallocated + cordoned + unschedulable-by-shape
                 OWNER: platform / capacity
                 FIX:   scheduling, autoscaling, commitment sizing,
                        health MTTR, packing policy (lesson 04)
  GAP B  ② − ③   allocated-idle — THE HEADLINE WASTE NUMBER
                 OWNER: tenant, refereed by platform
                 FIX:   right-sizing, idle TTLs, preemption (lesson 03)
  GAP C  ③ − ④   busy-but-inefficient
                 OWNER: the tenant's ML engineer
                 FIX:   kernels, batching, dataloader — and PARTLY
                        IRREDUCIBLE: memory-bandwidth-bound decode sits
                        here by physics, not by mismanagement
  ─────────────────────────────────────────────────────────────────────────
  ① = ② + GAP A     ② = ③ + GAP B     ③ = ④ + GAP C
  and therefore   ① = ④ + GAP A + GAP B + GAP C     ← the conservation law
```

Three things follow immediately, and each is load-bearing for the rest of the module.

**Only ② and ③ are "the two ledgers."** ① is the invoice envelope and ④ is an efficiency diagnostic. The lesson's title names the two that must always be reported together because they are the two that a *tenant* can be told about.

**The containment is strict and directional.** `④ ≤ ③ ≤ ② ≤ ①` must hold at every level of aggregation, and any violation is a bug rather than a finding. Utilised exceeding allocated means work ran on a device with no allocation record — a host process outside Kubernetes, a failed label mapping, or a DRA path your exporter is not configured for (lesson 01, `--kubernetes-enable-dra` defaults to `false`). Under time-slicing there is one legitimate exception, and it is worth knowing: a pod holding 1 of 4 replicas can show a *utilised share* near 1.0 when the other three are idle, because the driver's scheduler is work-conserving. So the invariant holds strictly per-device, and per-pod only under exclusive or MIG allocation.

**Gaps do not net off.** Gap B is waste; a negative gap under time-slicing is *borrowing*. Never let a dashboard take the absolute value or sum signed gaps across regimes — report `clamp_min(gap, 0)` when you mean waste and the signed value when you mean borrowing, and label them differently.

### 2. The allocated ledger, precisely

```
  allocated_gpu_hours(p, W) = Σ over the devices p held:
                                 share(p, d) × hours_held(p, d, W)

  allocated_cost(p, W)      = Σ  share(p, d) × hours_held(p, d) × r(d)
```

with units carried: `share` is dimensionless in [0, 1], `hours_held` is hours, `r(d)` is currency per physical-GPU-hour for *that device's model*. The output is currency.

Four properties make this the number you can defend in a dispute:

1. **It is a scheduler fact.** It comes from an allocation record — the pod-resources API's device list, or `ResourceClaim.status.allocation` under DRA — not from a measurement. No counter, no sampling, no scrape gaps.
2. **`share` is exactly defined in every regime** (lesson 01): 1.0 exclusive; the documented fractional basis under MIG; `1/replicas` under time-slicing or MPS — divided by the *configured* replica count, not by the number of pods currently holding one, so unsold replicas stay platform capacity rather than being socialised onto tenants.
3. **It reconciles exactly.** Per physical GPU, `Σ_p share(p) + unallocated_share ≡ 1.0`. That is module 04's identity A, and it is what lets a non-expert verify your books without knowing what a compute slice is.
4. **`r(d)` must be per-model.** A fleet with H100 and L40S in it has two rate cards and one dashboard. Hard-coding one multiplier is the most common way an otherwise correct pipeline produces an indefensible number.

**The trap that eats a day:** `r(d)` is the node's hourly cost divided by the node's **physical** GPU count from NVML — never by `nvidia.com/gpu.count`, which counts MIG devices under a `single` MIG strategy and replicas under time-slicing. Get that denominator wrong and every dollar figure is off by 4× or 7× while every internal identity still passes, because the shares remain self-consistent. Only a comparison against the invoice catches it (§8).

### 3. The utilised ledger, and the integral that must be right

```
                    T
  utilised_gpu_hours(p) = ∫  a_p(t) dt          a_p(t) = SM-active fraction
                    0                                     attributable to p

  utilised_cost(p)      = utilised_gpu_hours(p) × r
```

`a_p(t)` is a dimensionless ratio in [0, 1] sampled at the collect interval, so the integral is a Riemann sum over the samples you actually have:

```
  utilised_gpu_hours ≈ Σ_i  a_i × Δ / 3600           Δ = sample spacing, seconds
                     = sum_over_time(series[W]) × Δ / 3600
```

Module 05 derives this and its failure mode; the reason it reappears here is that **this lesson is where the arithmetic becomes money**, and the error is systematically in the flattering direction. Restate it in one box so you can never accidentally ship the wrong form:

```
  ✗ WRONG:  sum by (ns) (avg_over_time(gpu:sm_active:ratio[24h])) * 24
  ✓ RIGHT:  sum by (ns) (sum_over_time(gpu:sm_active:ratio[24h])) * 30 / 3600

  WHY: avg_over_time averages ONLY over samples that exist. A workload
  present for a fraction f of the window contributes its mean over that
  present time, and multiplying by the full window extrapolates it over
  time when the series did not exist.

     overstatement factor = 1 / f = window / time-present

  A job that ran 9 of 24 hours is overstated by 24/9 = 2.67×.
  A 2-hour job in a 24-hour window is overstated by 12×.

  DIRECTION MATTERS: the wrong query makes utilisation look HIGHER and
  the waste gap look SMALLER — worst on exactly the bursty workloads
  that waste the most. It understates the problem you are trying to
  publish, on the workloads where the problem is worst.
```

Two operational rules follow. **Pin Δ to the rule group's `interval`**, not to a scrape config someone else can edit, and write `Δ/3600` once as a documented constant. And **validate against a synthetic ground truth** before publishing a dollar figure: run a full-load workload for exactly 600 s on one GPU, expect 600/3600 = 0.1667 GPU-hours, accept ±2 sample intervals. The failure signatures are diagnostic — ~0.33 means double counting, ~0.08 means Δ is 15 s not 30 s, ~4.0 means you shipped the `avg_over_time` form.

**One honesty caveat on calling `SM_ACTIVE` "utilised."** `SM_ACTIVE` is an *occupancy* measure: the fraction of cycles an SM had at least one warp resident, averaged across SMs. It is the right evidence for the claim "you held a GPU and nothing was scheduled on it," which is a scheduling and waste claim. It is *not* a productivity measure — a GPU running fp32 elementwise kernels can sit at `SM_ACTIVE ≈ 0.9` with `PIPE_TENSOR_ACTIVE ≈ 0.04`. That is why ④ exists as a separate area with a separate owner. Ship both; label them differently; never merge them into one "efficiency" number.

**And one caveat on attribution**, carried from lesson 01: `a_p(t)` is exact under exclusive and MIG allocation, an estimate under time-slicing, and a bounded estimate under MPS. The utilised ledger therefore carries an error band that the allocated ledger does not. Report the exposure fraction alongside it.

### 4. The asymmetry — why the gap is structural, not accidental

People treat the gap as a defect to be eliminated. It is not; it is the arithmetic consequence of two quantities with different shapes. Write them out.

**Reservation is a step function of wall-clock time.** The moment a pod binds, the allocation is 1 (or `share`) and stays there until release. It does not fall when the process idles, it does not fall during an image pull, it does not fall while a checkpoint is written to object storage. It is `1[t ∈ held]`.

**Consumption is the integral of a rate bounded above by 1.** `a(t) ∈ [0, 1]`, and it dips for every cause in the workload's life: model download, CUDA context creation, kernel autotuning, dataloader stalls, gradient synchronisation, checkpoint writes, request troughs.

So over a window:

```
  gap_B  =  ∫ (1 − a(t)) dt          integrated over HELD time only
        held

  and, factorising the two independent causes:

  utilised_hours = (mean a while present) × (hours present)
                 = ā                     × h

  allocated_hours = h                     (by definition, share = 1)

  gap_B = h × (1 − ā)
```

**Those are two separate findings with two separate fixes.** `h` too large is a *holding* problem — a job that finished but did not release, a notebook parked overnight, a reserved node pool held across a weekend. `ā` too small is a *running* problem — batching, dataloaders, over-provisioned replicas. A dashboard that reports only `gap_B` in dollars conflates them; one that reports `h` and `ā` separately tells you which team to talk to.

Draw the lifecycle, because the areas make the point better than the algebra:

```
  ONE 8×H100 TRAINING JOB, 10 h wall clock.  share = 1.0 per GPU.

  ALLOCATED  ┌──────────────────────────────────────────────────────┐
  (step)     │████████████████████████████████████████████████████████
             │                    8 GPUs × 10 h = 80 GPU-hours
  a(t)  1.0 ─┼──────────────────────────────────────────────────────
             │        ▄▄▄▄▄▄▄▄▄▄        ▄▄▄▄▄▄▄▄▄▄▄▄▄▄
        0.5 ─┤       ▄██████████▄      ▄██████████████▄     ▄▄▄▄▄
             │      ▄████████████▄    ▄████████████████▄   ▄█████▄
        0.0 ─┴──▁▁▁─██████████████────██████████████████───███████──▁▁▁▁─▶
               │   │              │  │                  │         │     │ t
               │   │              │  │                  │         │     │
             bind  │           step │              step │      last  release
             image │           loop │              loop │      step
             pull  │                │                   │
             model │            CHECKPOINT           CHECKPOINT
             load  │            to object store      to object store
             (22m) │            (7 min, a≈0.02)      (7 min, a≈0.02)
                   └── first kernel

  ── WHERE THE 80 ALLOCATED GPU-HOURS WENT ────────────────────────────
    image pull + model load      0.37 h × 8 =  2.9 GPU-h   a ≈ 0.00
    two checkpoint barriers      0.23 h × 8 =  1.9 GPU-h   a ≈ 0.02
    step loop at ā = 0.61        9.03 h × 8 = 72.2 GPU-h   → 44.1 utilised
    trailing idle before release 0.37 h × 8 =  2.9 GPU-h   a ≈ 0.00
                                              ─────────
    allocated                                  80.0 GPU-h
    utilised (the integral)                    44.5 GPU-h
    GAP B                                      35.5 GPU-h  = 44 %

  ── THE DECOMPOSITION THAT NAMES OWNERS ──────────────────────────────
    startup + teardown (h problem)              5.8 GPU-h   platform/tenant:
                                                            image cache,
                                                            faster weights
                                                            load, release
                                                            on completion
    checkpoint barriers (h problem, partly      1.9 GPU-h   tenant: async
      irreducible)                                          checkpointing
    in-loop inefficiency (ā problem)           28.1 GPU-h   tenant ML eng:
                                                            batching, kernels
                                              ─────────
                                               35.5 GPU-h
```

**The gap is the default state.** A job that reaches `ā = 0.61` in its steady loop and still shows 44% gap over the whole allocation is not badly run — it is normally run. That is why the number to argue about is not "is there a gap" but "which component of it, owned by whom, is worth attacking."

### 5. Which ledger answers which question

| | **Allocated** | **Utilised** |
|---|---|---|
| **Definition** | reserved GPU-time × share, whether or not work happened | reserved time × measured SM-active fraction |
| **Source** | allocation record (pod-resources / ResourceClaim) | hardware counter integral (DCGM) |
| **Units** | GPU-hours | GPU-hours (a strictly smaller number on the same base) |
| **At 0% use** | **still counts — you pay** | zero |
| **Accuracy** | exact in every regime | exact under exclusive/MIG; estimate under time-slice/MPS |
| **Survives a scrape gap?** | yes | **no** — gaps silently reduce it, making waste look smaller |
| **Financial meaning** | the invoice | the value delivered |
| **Drives** | capacity planning, commitment sizing (lesson 06), chargeback | efficiency programmes, right-sizing, showback (lesson 08) |
| **If you hide it** | you under-buy and go short exactly when demand arrives | idle spend looks like committed spend and nobody investigates |
| **Can it be faked?** | no — it is a scheduler fact | yes, by using `GPU_UTIL` here. That is the lie module 05 exists to kill. |

Allocated answers *"how much iron must we buy and hold?"*. Utilised answers *"how much of it is doing anything?"*. One without the other is a half-audited book, and the missing half is always the expensive one.

### 6. Charge allocated, report utilised — the incentive proof

The instinct on seeing a 60% gap is "bill people for what they use." Work through what each rule does to a rational tenant.

Model a tenant choosing a reservation `h` (GPU-hours held) knowing they will do `u` GPU-hours of real work, with `u ≤ h` by construction. Holding extra capacity has a private benefit `B(h)` — headroom, faster starts, no queue wait, no risk of losing the slot — that increases with `h` and does not decrease.

```
  RULE 1 — CHARGE UTILISED:   cost_tenant = u × r
     ∂cost/∂h = 0.
     Reserving more is FREE to the tenant. Their optimum is to reserve
     as much as quota allows, indefinitely.
     Consequences that follow mechanically:
       · allocated hours rise until quota binds
       · the platform's ① − ② gap does NOT shrink; the ② − ③ gap grows
       · the platform absorbs the difference and cannot bill it to anyone
       · the fleet needs MORE hardware to serve the same real work
       · a tenant who genuinely improves efficiency (raises ā) sees their
         bill FALL while holding the same hardware — so the metric you
         wanted to move rewards the wrong thing at the wrong margin

  RULE 2 — CHARGE ALLOCATED:  cost_tenant = h × r
     ∂cost/∂h = r > 0.
     Every extra held hour has a price. The tenant reserves until the
     marginal private benefit equals the rate: B'(h*) = r.
     This is the textbook efficient allocation of a scarce, excludable
     resource — the tenant internalises the opportunity cost of denying
     the device to someone else, which IS the real cost to the platform.

  RULE 3 — CHARGE ALLOCATED, REPORT UTILISED
     Cost = h × r, and the tenant also sees ā = u/h and the dollar gap.
     Now BOTH margins point the right way:
       · reduce h (release earlier, right-size) → bill falls immediately
       · raise ā (batch better, fix the loader) → same bill, MORE WORK
         per dollar, which is the efficiency signal you wanted
     The reported utilised number carries no billing weight, so it can
     safely carry the regime error band from lesson 01 — you are not
     charging money on an estimate.
```

**Rule 3 is the answer, and the last line is why.** The allocated ledger is exact in every regime, so the number you *bill* never rests on a fair-share approximation. The utilised ledger, which does carry an error band under time-slicing, is used only for reporting and prioritisation, where a stated ±band is perfectly acceptable. The policy and the physics agree, which is rare and worth saying out loud in an interview.

Two riders that stop this becoming dogma. Bill **allocated** but do not bill *unallocatable* capacity to tenants — gap A is the platform's own number (fragmentation, cordons, MIG stranding), and pushing it onto tenants as an overhead surcharge without saying so is how a cost programme loses credibility. And publish the platform's own gap A on the same chart as the tenants' gap B: grading yourself on the same graph is what makes the artifact land as an analysis instead of an accusation.

### 7. Which *rate*, and the FOCUS columns that name it

Both ledgers multiply GPU-hours by a rate, and the rate is the most challengeable number in the room. The industry already has vocabulary for the distinctions, so use it: the FinOps Foundation's **FOCUS** specification (v1.0 June 2024; v1.1 Nov 2024; v1.2 June 2025; **v1.3 Dec 2025**; **v1.4 June 2026**) defines four cost columns whose differences are exactly the ones people muddle. Their definitions, from the spec's normative text:

| FOCUS column | What it means | Why it matters for a GPU-hour |
|---|---|---|
| **`ListCost`** | `ListUnitPrice × PricingQuantity`. The published, undiscounted price. Mandatory column. | the sticker rate. Useful only as the baseline you measure savings against |
| **`ContractedCost`** | `ContractedUnitPrice × PricingQuantity` — list price after *negotiated* discounts. Defaults to `ListCost` when no negotiated discount applies. Mandatory. | your enterprise-agreement rate, before commitments |
| **`BilledCost`** | the cost **as invoiced** in a billing period, in `BillingCurrency`. Excludes any portion covered by a separate covering charge (a prepaid commitment), and is **0** for a fully covered charge. | what actually hit the invoice this month — the cash view |
| **`EffectiveCost`** | the cost of a charge based on resources used or commitments *recognised* in the charge period; includes the amortised portion of prepayments drawn down by this usage. **0** for a purchase charge that exists to cover other charges. | **the accrual view, and the right rate for both ledgers.** It spreads a prepaid commitment across the hours it actually covers |

Two derived facts you should be able to state cold:

- **`EffectiveCost` is the rate to use.** `BilledCost` will read zero for GPU-hours covered by a prepaid commitment, which would make a heavily-committed fleet look free in one month and catastrophic in the month you paid. FOCUS is explicit that the sum of `EffectiveCost` across related covering and covered charges equals the sum of `BilledCost` over the same set — so `EffectiveCost` is the same money, correctly time-spread.
- **The quantity columns are two, not one, and the distinction is exactly ours.** `PricingQuantity` (+ `PricingUnit`) is the volume the provider *rates and prices*: for a GPU instance it is instance-hours, and it is what the invoice multiplies. `ConsumedQuantity` (+ `ConsumedUnit`) is the volume of the metered SKU *used*, often at finer granularity, and the spec states it "focuses on resource and service consumption, not pricing and cost". FOCUS also requires `ConsumedQuantity` to be null when `ChargeCategory` is not `Usage`, or when `CommitmentDiscountStatus` is `Unused` — a commitment you did not draw down has no consumption to report.

**And here is the gap in the standard that your deliverable exists to fill.** Even `ConsumedQuantity` is a *provider-metered* quantity — GPU-instance-hours the provider handed you — not a *hardware-measured* one. Nothing in FOCUS carries "of the GPU-hours you were charged for, how many had warps resident on the SMs." The utilised ledger has no FOCUS column, which is precisely why lesson 10's schema extends the standard with `x_Utilised*` fields and an attribution-regime dimension. FOCUS 1.3's split-cost-allocation columns (`AllocatedMethodId`, `AllocatedMethodDetails` with its `AllocatedRatio` / `UsageUnit` / `UsageQuantity` elements, `AllocatedResourceId`, `AllocatedTags`) get you a standard *place to record which split you used* — which is the right home for lesson 01's regime and basis — but they still describe an allocation of cost, not a measurement of work.

Practical rate hygiene, regardless of standard:

```
  # Expose the rate as a metric, not a constant in a query.
  # SOURCE and DATE are part of the number.
  # HELP gpu_hourly_rate_usd Effective $/physical-GPU-hour by model.
  #      BASIS: EffectiveCost / physical GPU-hours. SNAPSHOT: 2026-08.
  # TYPE gpu_hourly_rate_usd gauge
  gpu_hourly_rate_usd{modelName="NVIDIA H100 80GB HBM3"} 2.99
  gpu_hourly_rate_usd{modelName="NVIDIA A100-SXM4-80GB"} 1.75
  gpu_hourly_rate_usd{modelName="NVIDIA L40S"}           1.05
```

State the basis and the date every single time you say a rate out loud. As of **August 2026** published on-demand H100 SXM rates span roughly **$2–$7 per GPU-hour** across provider classes, with marketplace and spot capacity quoted below that and multi-year commitments pulling the effective rate lower still; the spread across providers for the *same silicon* is several-fold, which is lesson 07's entire subject. Every calculation in this lesson is written in terms of `r` so you can substitute your own.

### 8. The query pack: both ledgers, in dollars, reconciled

This drops into the deliverable. It assumes module 05's `gpu:sm_active:ratio` normalising rule, which gives every series a stable `ns` label and routes label-less GPUs to `__unallocated__`.

```yaml
# queries/two-ledgers.yaml
#
# INTEGRATION CONSTANT: 30 / 3600. It is the `interval:` of this rule
# group. Change one, change both, then re-run the synthetic ground-truth
# check before trusting any dollar figure.
groups:

- name: gpu-ledgers-base
  interval: 30s
  rules:

  # 1 for every GPU DCGM can see — the PAID envelope (area ①).
  - record: gpu:present:indicator
    expr: clamp_max(gpu:sm_active:ratio, 0) + 1

  # 1 for every GPU a pod holds — the ALLOCATED indicator (area ②).
  # Replace with the exporter's gpu_allocated_share for fractional
  # (MIG / time-sliced) devices, where the value is not 1.
  - record: gpu:allocated:indicator
    expr: clamp_max(gpu:sm_active:ratio{ns!="__unallocated__"}, 0) + 1

  # Per-model effective rate. BASIS + DATE live in this comment and in
  # the metric's HELP text; never inline a bare number in a query.
  - record: gpu:hourly_rate_usd
    # BASIS: FOCUS EffectiveCost ÷ physical GPU-hours. SNAPSHOT: 2026-08.
    expr: |
        (gpu:present:indicator{modelName=~"NVIDIA H100.*"} * 0 + 2.99)
      or (gpu:present:indicator{modelName=~"NVIDIA A100.*"} * 0 + 1.75)
      or (gpu:present:indicator{modelName=~"NVIDIA L40S.*"} * 0 + 1.05)
      or (gpu:present:indicator * 0 + 1.00)          # unknown model

- name: gpu-ledgers-hours
  interval: 5m
  rules:

  # ── AREA ②: allocated GPU-hours per namespace, per day ──────────────
  - record: ns:gpu_hours_allocated:1d
    expr: sum by (ns) (sum_over_time(gpu:allocated:indicator[1d])) * 30 / 3600

  # ── AREA ③: utilised (SM-active-weighted) GPU-hours ─────────────────
  - record: ns:gpu_hours_utilised:1d
    expr: sum by (ns) (sum_over_time(gpu:sm_active:ratio[1d])) * 30 / 3600

  # ── AREA ④: productive (tensor-active-weighted) GPU-hours ───────────
  - record: ns:gpu_hours_productive:1d
    expr: sum by (ns) (sum_over_time(gpu:tensor_active:ratio[1d])) * 30 / 3600

  # ── AREA ①: the paid envelope ───────────────────────────────────────
  - record: fleet:gpu_hours_present:1d
    expr: sum(sum_over_time(gpu:present:indicator[1d])) * 30 / 3600

  # ── THE TWO INDEPENDENT FACTORS behind the gap (§4) ─────────────────
  #    hours held, and mean SM-active fraction WHILE held.
  - record: ns:gpu_hours_held:1d
    expr: ns:gpu_hours_allocated:1d
  - record: ns:sm_active_mean_while_held:1d
    expr: ns:gpu_hours_utilised:1d / ns:gpu_hours_allocated:1d

  # ── THE GAPS, in GPU-hours ──────────────────────────────────────────
  - record: ns:gpu_hours_gap_b:1d        # allocated-idle: THE HEADLINE
    expr: ns:gpu_hours_allocated:1d - ns:gpu_hours_utilised:1d
  - record: ns:gpu_hours_gap_c:1d        # busy-but-inefficient
    expr: ns:gpu_hours_utilised:1d - ns:gpu_hours_productive:1d
  - record: fleet:gpu_hours_gap_a:1d     # paid but unallocated — PLATFORM'S
    expr: fleet:gpu_hours_present:1d - sum(ns:gpu_hours_allocated:1d)

- name: gpu-ledgers-money
  interval: 5m
  rules:

  # ── LEDGER 1: allocated cost, per namespace ─────────────────────────
  #    Model-parameterised: the group_left join applies each GPU's own
  #    rate, so a heterogeneous fleet costs correctly.
  - record: ns:gpu_cost_allocated:1d
    expr: |
      sum by (ns) (
        sum_over_time(
          ( gpu:allocated:indicator
            * on (modelName) group_left() gpu:hourly_rate_usd )[1d:30s]
        )
      ) * 30 / 3600

  # ── LEDGER 2: utilised cost, per namespace ──────────────────────────
  - record: ns:gpu_cost_utilised:1d
    expr: |
      sum by (ns) (
        sum_over_time(
          ( gpu:sm_active:ratio
            * on (modelName) group_left() gpu:hourly_rate_usd )[1d:30s]
        )
      ) * 30 / 3600

  # ── THE WASTE LEDGER, in money. clamp_min because a negative gap is
  #    BORROWING under time-slicing, not waste, and must not net off. ──
  - record: ns:gpu_cost_gap_b:1d
    expr: clamp_min(ns:gpu_cost_allocated:1d - ns:gpu_cost_utilised:1d, 0)
```

And the three checks that must pass before any of it is published:

```promql
# CHECK 1 — CONSERVATION (area ① = ② + gap A). Within ~1%.
fleet:gpu_hours_present:1d
  - ( sum(ns:gpu_hours_allocated:1d) + fleet:gpu_hours_gap_a:1d )

# CHECK 2 — CONTAINMENT. Must return NOTHING.
#   Any result means work ran on a device with no allocation record.
(ns:gpu_hours_utilised:1d - ns:gpu_hours_allocated:1d) > 0

# CHECK 3 — SAMPLE COMPLETENESS. Gaps silently SHRINK the utilised
#   ledger, i.e. they bias the waste number upward. Know your floor.
#   2880 = 24h ÷ 30s.
count_over_time(gpu:sm_active:ratio[1d]) < 2880 * 0.98
```

Note that check 3 has a direction: a scrape gap removes samples from ③ but not from ②, so an incomplete series *overstates* the gap. That is the opposite bias to the `avg_over_time` bug, and knowing which way each error pushes is how you defend a number under challenge.

### 9. What the leading tool actually does — and where it stops

OpenCost is the reference implementation of the allocated ledger, and reading it is more useful than reading anyone's summary of it. Two lines carry the whole model.

In `pkg/costmodel/allocation_helpers.go`, `applyGPUsAllocated` computes GPU-hours from the container's GPU **request**, multiplied by the hours the allocation existed:

```go
hrs := thisPod.Allocations[container].Minutes() / 60.0
// GPUHours reflects the full reserved GPU allocation (request × hours).
// For usage-based cost accounting, apply GPUUsageAverage separately.
thisPod.Allocations[container].GPUHours = res.Data[0].Value * hrs
```

and cost is that quantity times the node's per-GPU rate:

```go
alloc.GPUCost = alloc.GPUHours * node.CostPerGPUHr
```

The request itself is read in `costmodel.go` with a documented fallback order — `requests["nvidia.com/gpu"]`, then `limits["nvidia.com/gpu"]`, then `requests["k8s.amazonaws.com/vgpu"]`, then the corresponding limit.

**That is exactly this lesson's allocated ledger, computed correctly.** OpenCost is not wrong; it is *complete on one book*. What is genuinely interesting — and what makes the lesson-09 teardown sharper than the usual complaint — is that the same code path already collects the other book's inputs and does not use them:

- `GPUAllocation.GPUUsageAverage` is populated from a DCGM utilisation query (the code comments name `DCGM_FI_PROF_GR_ENGINE_ACTIVE`, a 0–1 float), with `GPUUsageMax` alongside it in `RawAllocationOnly`.
- `GPUAllocation.IsGPUShared` is populated from a dedicated `QueryIsGPUShared`, so the model *knows* a device has several claimants.
- `GPUAllocation.GPUUUID`, `GPUModel` and `GPUDevice` are populated from a GPU-info query.

So the precise statement is: **OpenCost measures utilisation, records that a device is shared, and then computes cost as request × hours × rate anyway.** The utilised ledger is one multiplication away and is not performed, and the shared flag is not used to adjust the share. Which means, for a fleet running seven `1g` MIG slices on one card or four pods time-slicing one device, its GPU cost is the request-count ledger under regime-1 assumptions — the exact gap lesson 09 documents from source and your operator closes.

The fix in your own pipeline is small and worth writing down explicitly, because it is the deliverable's core:

```
  cost_allocated(p) = share(p) × hours_held(p) × r          ← what tools give you
  cost_utilised(p)  = ∫ a_p(t) dt × r                       ← the missing multiply
  gap(p)            = clamp_min(cost_allocated − cost_utilised, 0)
  regime(p)         = exclusive | mig | time-sliced | mps | dra   ← the label that
                                                                    says how much
                                                                    to trust a_p
```

### 10. Ledger hygiene: five things that silently corrupt both books

1. **Window and timezone alignment.** Both ledgers and any application counter you later divide by (lesson 05) must come from one window definition. GPU-hours on a UTC billing-day boundary joined to tokens from a dashboard on local time is a skew of several hours on a bursty workload — double-digit percent error in a number that still looks plausible.
2. **You pay for instances, not GPUs.** The invoice bills a whole `p5.48xlarge` whether or not any GPU on it is allocated. That is area ① and it is the reason gap A exists as a separate bucket. Your allocated ledger will always be ≤ the invoice, and **the difference should equal gap A exactly**. If it does not, you have found either a billing surprise or a bug.
3. **Cordoned and unhealthy GPUs.** A device drained for a driver upgrade or an XID storm is paid-for, not allocated, and not idle in the tenant sense. Give it its own reason label inside gap A or the platform's own number looks worse than it is.
4. **MIG accounting units.** Seven `1g` slices on one card are one card's worth of cost. And the shares under a memory basis do not sum to 1 — book the remainder (≈17% of framebuffer at a 7×1g geometry on A100-40GB, from lesson 01) as platform overhead, with its own reason, or the conservation check fails with a symptom identical to an unresolved device ID.
5. **Time-sliced fan-out.** `SM_ACTIVE` is device-level; the exporter labels the same value with every holder. Summing across holders inflates the utilised ledger by exactly the holder count. Deduplicate on the measurement identity before summing, and carry the regime label so the error band travels with the number.

## Perspectives

**Finance / procurement.** Allocated is the number that sizes what must be bought or reserved, because it is what is contractually held regardless of activity. A procurement plan built off the utilised ledger under-buys by exactly the gap — it assumes the fleet only needs as much capacity as work currently lands on, ignoring reserved-but-idle capacity that is already committed and will be committed again next quarter. Finance also cares about which cost column you used: `EffectiveCost` spreads a prepaid commitment across the hours it covers, while `BilledCost` reads zero for those same hours and puts a spike in the month you paid. Mixing the two across months is how a "GPU spend by team" chart acquires a cliff nobody can explain.

**SRE / platform efficiency.** Utilised is the number an efficiency programme is judged on. Fix a dataloader bottleneck or right-size an inference replica and the allocated ledger does not move at all — the utilised ledger does, and the gap closing is the entire measure of whether the work mattered. Reporting allocated-only to an efficiency team gives them nothing to act on; reporting the two factors separately (`h` and `ā`) tells them whether the win is in releasing capacity or in using it harder, which are different projects with different owners.

**Workload owner.** From inside a namespace, "my pod shows Running" feels like "I am doing useful work." That is an allocation-side fact — the device is bound, the process is scheduled — and it is completely blind to whether any warp was resident. A workload owner has no native visibility into which ledger they are costing the fleet on, and will reasonably assume a running pod is a working pod until you surface `ā` to them explicitly. Pair every "you are wasting X" with a first thing to try, or the number reads as an accusation and the dashboard stops being opened.

**Executive / reporting.** A single blended "utilisation %" headline is worse than incomplete; it is ambiguous in a way that inverts decisions. Going from 40% to 55% could mean the same reservation now does more work (efficiency win, keep going) or that teams reserved less while doing the same work (allocated ledger shrank, the ratio improved, nothing got better). Those call for opposite responses, and only reporting both ledgers separately distinguishes them. The executive-safe framing is three numbers and a sentence: allocated GPU-hours, utilised GPU-hours, the gap in dollars, and which of `h` or `ā` moved.

**Auditor.** The allocated ledger is auditable without any GPU knowledge: it reconciles to 1.0 per physical device and to the invoice at the fleet level. The utilised ledger cannot be audited the same way — it is a measurement, with a scrape-completeness caveat and, on shared devices, an attribution error band. Publishing them with the same visual weight and no distinction invites a challenge that lands on the wrong ledger. Label the utilised number as measured, state its completeness percentage and its exposure fraction, and the challenge becomes a conversation instead of a credibility problem.

## Real-world use cases

- **OpenCost — `pkg/costmodel/allocation_helpers.go` and `costmodel.go`.** **What it shows:** the allocated ledger implemented in production OSS, in two lines: `GPUHours = <GPU request> × hours` and `GPUCost = GPUHours × node.CostPerGPUHr`, with requests read in the order `requests["nvidia.com/gpu"]` → `limits["nvidia.com/gpu"]` → `requests["k8s.amazonaws.com/vgpu"]` → limits. The sharper finding is what sits *next to* it: the same pipeline populates `GPUUsageAverage` from a DCGM engine-activity query and `IsGPUShared` from a dedicated shared-device query, and neither is used in the cost. The leading OSS tool has the utilised ledger's inputs in hand and does not perform the multiplication.

- **FinOps Foundation — the FOCUS specification, v1.0 (June 2024) through v1.4 (June 2026).** **What it shows:** the industry now has normative definitions for the cost columns people previously argued about. `BilledCost` is what the invoice charged and is **0** for a charge fully covered by a prepaid commitment; `EffectiveCost` includes the amortised, recognised portion of that commitment and is **0** on the covering purchase itself, so the two sum to the same money over the covering charge's period. `ListCost` and `ContractedCost` are `unit price × PricingQuantity` before and after negotiated discounts. `PricingQuantity`/`PricingUnit` is what gets rated; `ConsumedQuantity`/`ConsumedUnit` is what was consumed, and must be null when `CommitmentDiscountStatus` is `Unused`. **The gap this exposes:** every one of those quantities is provider-metered. None of them is hardware-measured, so the utilised ledger has no home in the standard — which is exactly why lesson 10's schema extends it.

- **FOCUS 1.3 split cost allocation (ratified 5 December 2025).** **What it shows:** the standard's answer to shared resources — `AllocatedMethodId` (which documented method produced the split), `AllocatedMethodDetails` (a JSON object whose `Elements[]` carry `AllocatedRatio`, `UsageUnit` and `UsageQuantity`, with the normative requirement that the ratios across all allocated charges from one origin charge **sum to 1**), `AllocatedResourceId` and `AllocatedTags`. That is the industry's version of lesson 01's redistributive-error framing: the total is conserved and the split is documented. It gives you a standard place to record *which* approximation a shared GPU-hour used — and still no place to record how much of that hour had warps resident.

- **NVIDIA `dcgm-exporter` — `etc/default-counters.csv`.** **What it shows:** `DCGM_FI_DEV_GPU_UTIL` ships **enabled** and `DCGM_FI_PROF_SM_ACTIVE` ships **commented out**. The reason "every GPU dashboard shows the wrong utilisation metric" is not carelessness — the wrong metric is the default and the right one is opt-in. Any utilised ledger built on a default install is measuring kernel residency, not SM occupancy, and will report a gap far smaller than the real one.

- **Uber Engineering — "Scaling AI/ML Infrastructure at Uber."** **What it shows:** a named operator at 5,000+ GPUs across on-prem plus OCI and GCP, which built elastic cross-team GPU sharing specifically so teams could opportunistically use *other teams' idle reserved capacity*. That is this lesson's gap B addressed at the platform layer rather than the tenant layer: rather than trying to make every tenant's reservation match their usage, let unused reservations be borrowed. It is also an implicit endorsement of charging allocated — the mechanism only makes sense if holding capacity has a cost the holder feels.

- **Anyscale — "GPU (In)efficiency in AI Workloads."** **What it shows:** a vendor teardown of *why* the gap exists in production, with the failure archetypes named: prefill and decode phases stalling each other in LLM serving, Python dataloaders starving training GPUs, and CPU-heavy pipeline stages bottlenecking GPU-heavy ones — including a described deployment holding hundreds of GPUs largely to supply enough CPU alongside them. Treat the specific percentages as dated vendor snapshots rather than constants; the durable content is the causal taxonomy, which maps one-to-one onto §4's `h`-versus-`ā` decomposition.

## Worked example

**The fleet slice.** One namespace, `team-vision`, over one billing day (24 h). Eight H100 80GB GPUs held for the full day by a training job. Rate `r = $2.99/GPU-hour` on an `EffectiveCost` basis — an August 2026 snapshot for H100 SXM in the middle of the observed $2–$7 on-demand band. **Substitute your own `r`; every line below is linear in it.**

### Step 1 — the allocated ledger

```
  allocated_gpu_hours = Σ share × hours_held
                      = 8 GPUs × 1.0 × 24 h
                      = 192.0 GPU-hours

  allocated_cost      = 192.0 GPU-h × $2.99/GPU-h
                      = $574.08
```

This is the invoice line for that namespace, unconditionally. It does not depend on a counter, a scrape, or a sharing regime. If Prometheus were down all day, this number would be unchanged.

### Step 2 — the utilised ledger

DCGM at Δ = 30 s gives 2,880 samples per GPU per day. The integral:

```
  Σ_i a_i  (summed over all 8 GPUs, all samples)      = 8,164.6
  utilised_gpu_hours = 8,164.6 × 30 / 3600            = 68.0 GPU-hours

  cross-check via the two factors:
    hours held  h  = 192.0 GPU-hours
    mean SM-active while held  ā = 68.0 / 192.0       = 0.354
    utilised = h × ā = 192.0 × 0.354                  = 68.0  ✓

  utilised_cost = 68.0 × $2.99                        = $203.32
```

Sample completeness: `count_over_time` returns 2,867 of an expected 2,880 per series — 99.5%, above the 98% floor, so no note needed. Had it been 90%, the utilised ledger would be understated by ~10% and the gap correspondingly *overstated*; say so in the footnote.

### Step 3 — the gap, and its two causes

```
  GAP B (allocated-idle) = 192.0 − 68.0    = 124.0 GPU-hours
                         = $574.08 − $203.32 = $370.76
  gap %                  = 1 − ā             = 64.6 %
```

Now split it by the §4 factorisation, using the timeline the job actually ran:

```
  phase                     GPU-h    ā      utilised   idle    owner
  ────────────────────────────────────────────────────────────────────────
  image pull + weights       12.6   0.00       0.0     12.6    platform
    (94 min × 8 GPUs)                                          (image cache,
                                                                weights on
                                                                local NVMe)
  warm-up / autotune          4.0   0.10       0.4      3.6    tenant
  steady step loop          150.4   0.44      66.2     84.2    tenant ML eng
    (18.8 h × 8 GPUs)                                          (batching,
                                                                dataloader)
  checkpoint barriers        16.0   0.05       0.8     15.2    tenant
    (2 h × 8 GPUs)                                             (async ckpt)
  trailing hold before        9.0   0.07       0.6      8.4    tenant
    release (67 min × 8)                                       (release on
                                                                completion)
  ────────────────────────────────────────────────────────────────────────
  total                     192.0   0.354     68.0    124.0
```

Read the columns and the story writes itself:

```
  h-problems (held but structurally could not compute):
      image pull + weights            12.6 GPU-h  = $37.67
      trailing hold                    9.0 GPU-h  = $26.91
      checkpoint barriers             16.0 GPU-h  = $47.84   (partly
                                                              irreducible)
                                      ─────────────────────
                                      37.6 GPU-h  = $112.42   30 % of the gap

  ā-problem (held AND running, but the SMs were mostly empty):
      steady loop at ā = 0.44         84.2 GPU-h  = $251.76   68 % of the gap
      warm-up                          3.6 GPU-h  = $10.76
```

**Two different projects.** The `h` bucket is attacked with an image cache, weights on local NVMe, asynchronous checkpointing and a controller that releases on completion — all platform-ish, all mechanical, all low-risk. The `ā` bucket is attacked with batching and kernel work, which is the tenant's ML engineer and a much longer conversation. A dashboard that says "$370 wasted" points at neither.

### Step 4 — the third area, and the sentence it prevents

Add `PIPE_TENSOR_ACTIVE`:

```
  productive_gpu_hours = 41.2 GPU-h        (∫ PIPE_TENSOR_ACTIVE dt)
  GAP C = utilised − productive = 68.0 − 41.2 = 26.8 GPU-h = $80.13

  ratios:
    utilised / allocated   = 35.4 %      ← the waste conversation
    productive / utilised  = 60.6 %      ← the efficiency conversation
    productive / allocated = 21.5 %      ← the number that sounds alarming
                                           and must NOT be reported alone
```

That last ratio is true and, on its own, misleading: it invites "this team wastes 78% of its GPUs," which is not a statement anyone can act on and is not fair to a workload whose steady loop may be partly bandwidth-bound by physics. Report ② → ③ → ④ as three areas with three owners, or do not report ④ at all.

### Step 5 — scale it to the fleet, and grade the platform too

Extend to a 24-GPU cluster for the same day: 16 H100 at `r_H = $2.99` and 8 A100 at `r_A = $1.75`.

```
  AREA ① — PAID (the envelope)
    16 × 24 × $2.99 = $1,148.16
     8 × 24 × $1.75 = $  336.00
                      ──────────
                      $1,484.16 / day     = $541,718 / yr

  AREA ② — ALLOCATED, per namespace (from the allocation records)
    team-vision    192.0 GPU-h  (8 H100)          $574.08
    team-serving   192.0 GPU-h  (4 H100 + 4 A100) $455.04
    team-batch      72.0 GPU-h  (3 A100)          $126.00
                   ───────────                    ─────────
                   456.0 GPU-h                    $1,155.12

  GAP A — PAID BUT UNALLOCATED (THE PLATFORM'S OWN NUMBER)
    24 GPUs × 24 h = 576 GPU-h paid
                   − 456 GPU-h allocated
                   = 120 GPU-h  (5 GPUs' worth, all day)
    at the mix's blended rate ≈ $2.74                $329.04 / day
    reasons: 3 GPUs unallocated (bin-packing, lesson 04)
             1 GPU cordoned for a driver upgrade
             1 GPU-equivalent of MIG memory stranding

  AREA ③ — UTILISED, per namespace
    team-vision     68.0 GPU-h   ā = 0.354           $203.32
    team-serving    21.1 GPU-h   ā = 0.110           $ 50.03
    team-batch      31.7 GPU-h   ā = 0.440           $ 55.48
                   ───────────                       ─────────
                   120.8 GPU-h                       $308.83

  GAP B — ALLOCATED-IDLE, per namespace
    team-vision    124.0 GPU-h                       $370.76
    team-serving   170.9 GPU-h                       $405.01
    team-batch      40.3 GPU-h                       $ 70.52
                   ───────────                       ─────────
                   335.2 GPU-h                       $846.29 / day

  ── CONSERVATION CHECK ──────────────────────────────────────────────
    paid          $1,484.16
    = utilised     $308.83
    + gap B        $846.29
    + gap A        $329.04
                  ──────────
                  $1,484.16    ✓ closes to the cent
```

**Both numbers go on the chart.** Tenant waste is $846/day; the platform's own unallocated and stranded capacity is $329/day. Publishing the first without the second is how a cost dashboard becomes politically radioactive and stops being consulted.

### Step 6 — the reshuffle, which is the finding people remember

```
  RANKED BY ALLOCATION ($)          RANKED BY UTILISED GPU-HOURS
  ────────────────────────          ────────────────────────────
  1. team-vision    $574.08    ┐    1. team-vision   68.0 GPU-h
  2. team-serving   $455.04    ├──▶ 2. team-batch    31.7 GPU-h   ▲ up one
  3. team-batch     $126.00    ┘    3. team-serving  21.1 GPU-h   ▼ down one

  team-serving holds 3.6× the GPU-dollars of team-batch and does
  0.67× the work: 39 % of the namespace allocation, 17 % of the output.

  And on the dashboard everyone actually looks at — the one built on
  DCGM_FI_DEV_GPU_UTIL, which ships enabled — team-serving is the
  BUSIEST namespace in the cluster, because an inference server always
  has a kernel resident. ā = 0.110 on the honest metric.
```

### Step 7 — sensitivity, so the reader can disagree with an input rather than the conclusion

Because everything is linear in `r` and in `ā`, publish the sensitivity rather than a single number:

```
  GAP B for team-serving (192.0 allocated GPU-h, mix-blended rate $2.37):

    ā        utilised GPU-h    gap GPU-h    gap $/day    gap $/yr
    ────────────────────────────────────────────────────────────────
    0.11  (measured)   21.1        170.9      $405.01    $147,829
    0.20               38.4        153.6      $364.03    $132,871
    0.35               67.2        124.8      $295.78    $107,960
    0.50               96.0         96.0      $227.52    $ 83,045

  At r = $2.00 instead of $2.37, multiply every $ figure by 0.844.
  At r = $5.00, multiply by 2.11.

  READ IT AS: "moving team-serving from ā = 0.11 to a conservative
  ā = 0.35 recovers ~$110/day, ~$40k/yr, at our current rate — and
  the projection is stated as a rate change so you can challenge the
  0.35 rather than the conclusion."
```

Do not promise the ceiling. A memory-bandwidth-bound decode service will never reach a training workload's occupancy, and any projection that ignores that is not credible in front of someone who knows it.

## Practice

Feeds the module deliverable at [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md).

1. **Compute both ledgers for a real or plausible fleet slice.** Pull allocated GPU-hours from allocation records (pod-resources join, or `kube_pod_container_resource_requests` as a stand-in) and utilised GPU-hours from the `sum_over_time` integral. Emit: allocated GPU-h, utilised GPU-h, allocated $, utilised $, gap $, gap %, per namespace.
   **Acceptance:** every dollar figure carries the rate, its basis (`EffectiveCost` / list / TCO) and its date; utilised ≤ allocated everywhere; sample completeness reported.

2. **Factorise every gap into `h` and `ā`.** For each namespace report hours held and mean SM-active-while-held as *separate* numbers, and state which of the two is the bigger contributor to that namespace's gap.
   **Acceptance:** `h × ā = utilised GPU-hours` reconciles for every row.

3. **Close the conservation identity in dollars.** Compute area ① from the node inventory and the rate card, then show `paid = utilised + gap B + gap A` to within 1%, with gap A broken into reasons (unallocated / cordoned / MIG-stranded).
   **Acceptance:** the identity closes and gap A has at least two named reasons.

4. **Add the third area.** Integrate `PIPE_TENSOR_ACTIVE` for one namespace and report gap C separately from gap B, with a one-line note on how much of gap C you believe is irreducible for that workload class and why.
   **Acceptance:** the report never states productive/allocated as a standalone "waste" number.

5. **Write the two-line report a platform lead would actually send.** Both ledgers, the gap in dollars, which factor moved, and one recommended action with its owner. No adjectives.
   **Acceptance:** a non-engineer can act on it; an engineer can reproduce it from the query pack.

6. **Map one namespace-day onto FOCUS columns.** Produce a row with `BilledCost`, `EffectiveCost`, `ListCost`, `ContractedCost`, `PricingQuantity`/`PricingUnit`, `ConsumedQuantity`/`ConsumedUnit`, plus your `x_UtilisedGpuHours` extension and the attribution regime. Note in one sentence why the utilised figure cannot be a standard column.
   **Acceptance:** `EffectiveCost` is used as the ledger rate, and the extension fields are clearly marked as extensions.

## Common pitfalls

1. **Using `DCGM_FI_DEV_GPU_UTIL` for the utilised ledger.** *Mechanism:* `GPU_UTIL` reports the duty cycle of *kernel residency* — it reads ~100% if any kernel was resident during the sample window, whether it used one SM or all of them. It ships enabled in `dcgm-exporter`'s default counters CSV while `SM_ACTIVE` ships commented out, so this is the default outcome, not a mistake someone made. *Symptom:* a fleet that looks 85% utilised with a small, uninteresting gap. Use `DCGM_FI_PROF_SM_ACTIVE`.

2. **`avg_over_time(SM_ACTIVE[W]) × W_hours` for GPU-hours.** *Mechanism:* `avg_over_time` averages only over samples that exist, so a workload present for a fraction `f` of the window is inflated by `1/f`. *Direction:* it makes utilisation look higher and waste look smaller, worst on bursty workloads. Use `sum_over_time(...) × Δ/3600` and validate with a synthetic full-load run of known duration.

3. **Treating allocated GPU-hours as a waste signal.** *Mechanism:* allocated is the reservation — the number the bill charges regardless. The waste is the *gap*, not the allocation. A team with high allocated hours and `ā = 0.9` is your best tenant, and a report that ranks by allocation alone will name them as the problem.

4. **Netting a negative gap against a positive one.** *Mechanism:* under time-slicing the driver's scheduler is work-conserving, so a pod holding 1 of 4 replicas can show a utilised share near 1.0 when its co-tenants are idle — a negative gap, which is *borrowing*, not waste. Summing signed gaps across a shared pool cancels real waste against real borrowing. Use `clamp_min(gap, 0)` for waste and report the signed value separately.

5. **Charging tenants on the utilised ledger because it feels fairer.** *Mechanism:* it makes over-reserving free (`∂cost/∂h = 0`), so the rational tenant reserves to quota and runs lightly; allocated hours climb, the platform absorbs the difference, and the fleet needs more hardware for the same work. Charge allocated, report utilised — and note the bonus: the number you bill is then exact in every regime, so you never charge money on a fair-share estimate.

6. **Comparing allocated cost across teams on different commitment mixes.** *Mechanism:* a team on spot capacity and a team on a three-year commitment show very different dollar totals for identical GPU-hour volumes, so the comparison conflates volume with rate. Compare GPU-hours for volume questions and hold the rate fixed; use `EffectiveCost` and state the blend when you must compare dollars (lesson 06).

7. **Mixing `BilledCost` and `EffectiveCost` across months.** *Mechanism:* `BilledCost` is **0** for usage covered by a prepaid commitment and carries the full purchase in the month it was invoiced; `EffectiveCost` amortises it into the hours it covers. Mixing them gives a per-team chart with a cliff in the purchase month and an artificial trough afterwards, and destroys trust in every other number on the page.

8. **Reporting one blended utilisation percentage.** *Mechanism:* the ratio can move because the numerator rose (real efficiency) or the denominator fell (less capacity reserved). The two demand opposite responses. Always publish the two ledgers as absolute GPU-hours alongside any ratio.

9. **Ignoring scrape completeness when publishing the gap.** *Mechanism:* missing samples remove work from the utilised ledger and nothing from the allocated ledger, so an incomplete series *overstates* waste. This is the one error that biases in your favour, which is exactly why a hostile reviewer will find it. Report completeness with the number.

10. **Costing a MIG slice or a time-slice replica as a whole GPU.** *Mechanism:* the resource names count instances or replicas, not cards. Seven `1g` slices on one card is one card's cost; four replicas is one card's cost. Both errors inflate the fleet's apparent spend by the sharing factor and, worse, do so consistently enough to look like a real trend.

## Self-check

- **Two workloads each hold 4 GPUs for 5 hours at the same rate; one runs at `SM_ACTIVE` 0.80, the other 0.20. Same allocated cost? Same utilised cost?**
  **Answer:** Identical allocated cost — both reserved `4 × 5 = 20` GPU-hours and reservation is billed regardless of activity, so at rate `r` both owe `20r`. Utilised cost differs 4×: 16 GPU-hours versus 4 GPU-hours of SM-active work, i.e. `16r` versus `4r`. Gap B is `4r` versus `16r`. Reporting only allocated makes the two indistinguishable while one wastes 80% of its spend; reporting only utilised understates by 4 and 16 GPU-hours respectively the capacity procurement must actually contract for. Also note what is *not* different: their attribution accuracy, if both are exclusive allocations, is exact — the difference is entirely a utilisation finding.

- **Why is `SM_ACTIVE`, not `GPU_UTIL`, the multiplier for the utilised ledger — and why is `SM_ACTIVE` still not a productivity metric?**
  **Answer:** `GPU_UTIL` reports the duty cycle of kernel residency: it reads high whenever any kernel was resident in the sample window, regardless of how much of the chip that kernel used, so a single-SM or memory-stalled workload looks fully busy. It also ships enabled by default in `dcgm-exporter` while `SM_ACTIVE` ships commented out, which is why the wrong metric is on most dashboards. `SM_ACTIVE` measures the fraction of cycles an SM had at least one warp resident, averaged across SMs — real occupancy, and the right evidence for the claim "you held this GPU and nothing was scheduled on it." But occupancy is not productivity: a GPU running fp32 elementwise kernels can sit at `SM_ACTIVE ≈ 0.9` with `PIPE_TENSOR_ACTIVE ≈ 0.04`. That is why the model has four nested areas, not three: gap C (utilised minus productive) has a different owner and is partly irreducible for memory-bandwidth-bound work.

- **Where is "all the money" in a typical GPU fleet, and which ledger surfaces it?**
  **Answer:** In the gaps, and there are three of them with different owners. Gap B (allocated minus utilised) is the tenant-facing headline — reserved-but-idle capacity billed at full rate, driven by two independent factors: hours held `h` and mean SM-active-while-held `ā`, with `gap_B = h(1 − ā)`. Gap A (paid minus allocated) is the platform's own number — unallocated capacity, cordoned devices, fragmentation, MIG stranding. Gap C (utilised minus productive) is the efficiency finding. Neither ledger alone surfaces any of this: allocated alone shows a fully committed fleet; utilised alone shows a small number with nothing to compare it against. Reporting both, plus the conservation identity `paid = productive + gap A + gap B + gap C`, is what makes the money visible *and* assignable.

- **State the two-factor decomposition of gap B and say why it matters operationally.**
  **Answer:** `utilised = h × ā`, where `h` is GPU-hours held and `ā` is the mean SM-active fraction while held; therefore `gap_B = h − h·ā = h(1 − ā)`. The two factors are independent and have different owners. A large `h` with reasonable `ā` is a *holding* problem — jobs not releasing on completion, notebooks parked overnight, reserved pools held across weekends, long image pulls and weight loads before the first kernel — and it is fixed with caching, faster loading, TTLs and release-on-completion, mostly mechanically and at low risk. A reasonable `h` with small `ā` is a *running* problem — batching, dataloaders, over-provisioned replicas, synchronous checkpointing — and it is fixed by the tenant's ML engineers over a much longer horizon. A single "$X wasted" figure points at neither; publishing `h` and `ā` separately tells you which project to fund.

- **Why is charging the allocated ledger the defensible default, and what would go wrong if you charged utilised?**
  **Answer:** Two independent arguments converge. *Incentives:* under utilisation billing, `∂cost/∂h = 0` — reserving more capacity is free — so a rational tenant reserves to quota and runs lightly, allocated hours climb, and the platform absorbs a cost it cannot bill to anyone. Under allocation billing, `∂cost/∂h = r`, so the tenant reserves until marginal private benefit equals the rate, which internalises the opportunity cost of denying the device to someone else. *Provenance:* the allocated ledger is an exact scheduler fact in every sharing regime, while the utilised ledger is an estimate on time-sliced or MPS devices — so charging allocated means you never bill money on a fair-share approximation. Reporting utilised alongside it closes the loop: the tenant sees `ā` and the dollar gap, so improving efficiency is visible and rewarded even though it does not change the invoice. Two riders: do not silently push gap A onto tenants, and publish the platform's own gap on the same chart.

- **A dashboard reports "utilisation improved from 40% to 55% this quarter." Why is that ambiguous, and what would you ask for?**
  **Answer:** Because a ratio can move from either end. The numerator may have risen — the same reservations now do more real work, a genuine efficiency gain. Or the denominator may have fallen — teams reserved less capacity while doing the same or less work, which improves the ratio without anyone becoming more efficient, and typically also means the fleet has more unallocated capacity sitting in gap A. Those call for opposite responses: keep funding the efficiency work, versus go find out why allocation dropped and whether you are now over-provisioned. Ask for the two ledgers as absolute GPU-hours over both periods, plus gap A, plus the completeness percentage on the utilised series — because a drop in scrape coverage also raises nothing and lowers the utilised ledger, which would move the ratio the *other* way and is worth ruling out.

- **Which FOCUS column should the ledgers be priced on, and why does the utilised ledger have no FOCUS column at all?**
  **Answer:** Price both ledgers on **`EffectiveCost`** — the cost of a charge based on resources used or commitments recognised in the charge period, including the amortised portion of prepayments drawn down by that usage. `BilledCost` is the invoiced amount and reads **0** for usage fully covered by a prepaid commitment, so a committed fleet looks free in most months and terrible in the purchase month; `ListCost` and `ContractedCost` are `unit price × PricingQuantity` before and after negotiated discounts, useful for savings analysis but not for what capacity actually costs you. As for the utilised ledger: every FOCUS quantity column is provider-metered. `PricingQuantity` is what the provider rates (instance-hours); `ConsumedQuantity` is the metered SKU volume consumed, explicitly "not pricing and cost" but still a provider meter. Neither can express "of the GPU-hours you were charged for, how many had warps resident on the SMs," because no provider meters that. FOCUS 1.3's split-cost columns (`AllocatedMethodId`, `AllocatedMethodDetails` with `AllocatedRatio` summing to 1 across a charge's allocations, `AllocatedResourceId`, `AllocatedTags`) standardise *how a shared cost was divided* — the right home for lesson 01's regime and basis — but a division of cost is still not a measurement of work. Hence the `x_Utilised*` extension in lesson 10.

## Connections & what's next

Backward: this lesson consumes lesson 01's regimes wholesale — the allocated ledger's `share` term is defined per regime, and the utilised ledger's error band *is* the regime's attribution accuracy. It consumes module 04's identity A (shares plus unallocated sum to 1.0 per physical GPU) as the conservation check on area ②, and module 05's `sum_over_time` integral as the only correct way to build area ③.

Forward: lesson 03 takes gap B and splits it into idle states with owners, remediations and — crucially — a decision rule for when closing it is safe, because a warm inference replica at `ā ≈ 0` is not the same finding as an abandoned notebook. Lesson 04 takes gap A and shows that "free" capacity is not the same as *usable* capacity, which is where the fragmentation ratio comes from. Lesson 05 divides a ledger by an application counter: divide by the utilised ledger and you get a direct unit cost, load in gaps A and B and you get the fully-loaded one, and the ratio between them is exactly the tax this lesson measures. Lesson 08 turns the charge-allocated / report-utilised policy into an actual statement a tenant receives. Lesson 09 reads the OpenCost source lines quoted in §9 in full. Lesson 10 gives both ledgers a schema.

Next: **lesson 03** takes the single gap-B number this lesson produces and breaks it into the idle-state taxonomy you need to act on it without breaking production.

## References & further reading

**Primary sources — the standard**

1. **FinOps Foundation — FOCUS specification, cost columns** — https://focus.finops.org/focus-specification/ — normative definitions of `BilledCost` (invoiced amount; 0 for fully covered charges), `EffectiveCost` (accrual view including recognised commitment drawdown; 0 on covering purchases), `ListCost` and `ContractedCost` (`unit price × PricingQuantity`, before and after negotiated discounts). Verified against the column definition files in the `FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec` repository (`specification/datasets/cost_and_usage/columns/`), because focus.finops.org and finops.org were unreachable from this build environment.
2. **FOCUS — `PricingQuantity`/`PricingUnit` vs `ConsumedQuantity`/`ConsumedUnit`** — same repository — the spec's own statement that pricing quantity "focuses on pricing and cost, not resource and service consumption" and vice versa, plus the nullability rules (`ConsumedQuantity` null when `ChargeCategory` is not `Usage`, or when `CommitmentDiscountStatus` is `Unused`). This is the distinction that shows why neither column can carry a hardware-measured utilisation figure.
3. **FOCUS — version history and split cost allocation** — `CHANGELOG.md` in the same repository — v1.0 (June 2024), v1.1 (Nov 2024, +7 columns incl. capacity reservation and SKU meter), v1.2 (June 2025, +7 columns incl. pricing-currency support), **v1.3 (Dec 2025**, 14 columns, the `ContractCommitment` dataset, and the split-cost-allocation columns `AllocatedMethodId` / `AllocatedMethodDetails` / `AllocatedResourceId` / `AllocatedResourceName` / `AllocatedTags`**)**, **v1.4 (June 2026**, 47 columns, invoice reconciliation and commitment-program eligibility**)**. The normative requirement that `AllocatedRatio` sums to 1 across all allocated charges from one origin charge is in `allocatedmethoddetails.md`.
4. **FinOps Foundation — Allocation capability** — https://www.finops.org/framework/capabilities/allocation/ — the framework's treatment of cost allocation, showback and chargeback, which is the organisational context for §6's policy argument. (finops.org was unreachable from this build environment; the framework's structure is summarised here from the capability's published scope rather than quoted.)

**Primary sources — the signals**

5. **NVIDIA — DCGM field-ID reference** — https://docs.nvidia.com/datacenter/dcgm/latest/dcgm-api/dcgm-api-field-ids.html — field 203 (`DCGM_FI_DEV_GPU_UTIL`, kernel-residency duty cycle), 1001 (`GR_ENGINE_ACTIVE`), 1002 (`SM_ACTIVE`, warp-resident cycles averaged over SMs), 1004 (`PIPE_TENSOR_ACTIVE`). (docs.nvidia.com unreachable here; the semantics used are those established and verified in module 05.)
6. **NVIDIA `dcgm-exporter` — `etc/default-counters.csv` and CLI flags** — https://github.com/NVIDIA/dcgm-exporter — `DCGM_FI_DEV_GPU_UTIL` enabled by default, `DCGM_FI_PROF_SM_ACTIVE` commented out; `--collect-interval` default **30000 ms** (your integration constant); `--kubernetes` default `false`.
7. **Prometheus — query functions and recording rules** — https://prometheus.io/docs/prometheus/latest/querying/functions/ and https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/ — `sum_over_time`, `count_over_time`, `clamp_max`/`clamp_min`, subquery syntax `[1d:30s]`, and rule-group `interval` semantics, which is where the integration constant should live.
8. **Kubernetes — managing resources for containers** — https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/ — why a device request *is* the allocation record the allocated ledger is built from, and why extended resources like `nvidia.com/gpu` must have request equal to limit.

**Reading the tools' source**

9. **OpenCost — `pkg/costmodel/allocation_helpers.go`** — https://github.com/opencost/opencost — `applyGPUsAllocated`: `GPUHours = <request> × hours`, with the in-code comment stating that GPU-hours reflect the full reserved allocation and that usage-based accounting would apply `GPUUsageAverage` separately; and `alloc.GPUCost = alloc.GPUHours * node.CostPerGPUHr`. Also `applyGPUUsageAvg`, `applyGPUUsageShared` and `applyGPUInfo`, which populate `GPUUsageAverage`, `IsGPUShared` and the GPU UUID/model — collected, and unused in the cost.
10. **OpenCost — `core/pkg/opencost/allocation.go`** — the `Allocation` struct's `GPUHours`, `GPUCost`, `GPUCostAdjustment`, `GPUCostIdle` fields and the `GPUs()` helper (`GPUHours / (Minutes()/60)`), i.e. the shape of the allocated ledger as a data model.
11. **OpenCost — specification** — https://www.opencost.io/docs/specification — the project's own definition of allocation and idle cost, useful for seeing how narrowly the OSS consensus models this split today.

**Real-world engineering**

12. **Uber Engineering — "Scaling AI/ML Infrastructure at Uber"** — https://www.uber.com/en-US/blog/scaling-ai-ml-infrastructure-at-uber/ — 5,000+ GPUs across on-prem, OCI and GCP, with elastic cross-team sharing built specifically so idle *reserved* capacity can be borrowed. Read for gap B addressed at the platform layer rather than by policing tenants. **What it shows:** the mechanism only makes economic sense if holding capacity carries a cost the holder feels — i.e. an implicit endorsement of charging the allocated ledger.

---
Module backlink: [💰 11 — GPU cost and unit economics](../README.md)
