---
lesson: 08
title: "Chargeback, showback, and queue-wait billing"
module: 11
concept: "internal GPU cost allocation"
status: not-started
est_time: "4.5 hrs"
prev: "07-neocloud-vs-hyperscaler.md"
next: "09-existing-tooling-limits.md"
artifacts: ["a monthly showback/chargeback statement for 3 teams on a shared 64-GPU fleet with a blended internal rate and a queue-priority premium line"]
sources: 13
---

# Chargeback, showback, and queue-wait billing

[💰 11 — GPU cost and unit economics](../README.md) · ← [07 Neocloud vs hyperscaler price gap](07-neocloud-vs-hyperscaler.md) · → [09 Where existing tooling fails](09-existing-tooling-limits.md)

## Where this fits

Lessons 01–07 built the measurement stack. Lesson 01 gave you the attribution ladder and the one rung where information is destroyed. Lesson 02 split the allocated ledger from the utilised ledger, proved that you charge the first and report the second, and named the FOCUS cost columns that carry the rate. Lesson 03 gave you an idle rule with a false-positive cost. Lesson 04 gave you fragmentation as a measurable quantity. Lesson 05 turned GPU-hours into $/token and $/run. Lesson 06 produced the blended rate from a commitment mix. Lesson 07 taught you to normalise a vendor's sticker into a fully-loaded $/GPU-hr.

This lesson is where all of that becomes **an invoice line with a team's name on it** — a number that leaves your system, lands in someone else's budget, and gets argued with. Everything upstream was about knowing a cost. This lesson is about *transferring* one: the rate policy that turns a fixed monthly fleet cost into a per-hour price, the disposal of the capacity nobody held, the pricing of the queue itself, and the dispute protocol that decides what happens when a team says the number is wrong.

Lesson 02 already settled *which ledger* you charge and proved it with a two-line marginal argument. That proof is not repeated here. What is new is everything that sits around it: rate policy, remainder disposal, tier design, restatement, and the mechanics of an argument you have to win with evidence.

## Why this matters

A chargeback system is the only part of the cost stack that has an adversary. A dashboard nobody's budget depends on is never audited; the moment a number moves money, somebody has a direct financial incentive to find its weakest joint, and they will find it faster than you expect. **The technical quality bar for a chargeback number is not "accurate" — it is "survives a motivated engineer with access to the same telemetry."**

That bar is why the allocated/utilised distinction from lesson 02 is load-bearing rather than academic. Allocation comes from rungs 1–3 of the ladder: an invoice line, an arithmetic division, and a scheduler record. Every one of those is a *recorded fact* that you can replay. Utilisation comes from rung 4, a measurement, and under time-slicing it is provably an estimate. Bill on a recorded fact and a dispute becomes "here is the pod-resources snapshot at 14:03, and here is the same snapshot from your own `kubectl`." Bill on an estimate and a dispute becomes a debate about counter semantics that you will lose in front of a finance partner, because you will be the only person in the room who can explain what `DCGM_FI_PROF_GR_ENGINE_ACTIVE` measures.

The money is not small and the politics are not optional. Take a single shared 64×H100 fleet, the worked example below: **$191,150/month** fully loaded, **$2.29M/year**. The gap between "charge allocation, report utilisation, publish the remainder" and "just split the bill by headcount" is, on that fleet, a difference of tens of thousands of dollars per team per year plus the second-order effect that actually matters — whether teams release GPUs they are not using. A platform team that cannot allocate its own fleet's cost gets asked the same two questions forever: *why is the GPU bill so high* and *why can't my team ever get a GPU*. Those are the same question. Chargeback is the mechanism that makes the answer legible.

There is a specific interview probe hiding here, and it is not "define showback." It is: *"You charge team A for 5,000 GPU-hours. They used 1,500. They escalate to your VP. Walk me through the next 48 hours."* The answer to that is a protocol, not an opinion, and this lesson gives you one.

## What's new here (calibration)

- **Already yours (skip):** the definitions of showback and chargeback; the allocated-vs-utilised charging proof and its incentive argument (lesson 02 §6); the FOCUS cost columns `ListCost` / `ContractedCost` / `BilledCost` / `EffectiveCost` and why `EffectiveCost` is the rate basis (lesson 02 §7); the fully-loaded $/GPU-hr normalisation (lesson 07); the reconciliation identities from module 04's capstone. All referenced, none re-derived.
- **Genuinely new — the chargeback data path as a system, with a trust boundary.** Where a number stops being a recorded fact and becomes a measurement, drawn explicitly, because that line is exactly where a dispute is won or lost.
- **Genuinely new — three rate policies, derived.** Fixed transfer price, floating full-absorption, and two-part tariff, with the arithmetic for each and the **floating-rate death spiral** written out as a feedback loop with numbers, plus the two standard dampers.
- **Genuinely new — the unallocatable remainder, taxonomised and disposed of.** Six named sources of remainder, four disposal policies costed on the same fleet, and an explicit decision about who eats it. This is the part of chargeback design most write-ups skip and every real programme has to answer in its first month.
- **Genuinely new — variance decomposition.** When the statement total does not equal the fleet's cost, the difference splits exactly into volume variance, mix variance, and rate-rounding variance. Being able to hand finance that three-line reconciliation is the difference between "the model is off by $8k" and "here is precisely why, in three terms that sum to the cent."
- **Genuinely new — queue pricing on real scheduler primitives.** Not "priority tiers exist," but the actual Kueue API objects that implement them (`nominalQuota`, `borrowingLimit`, `lendingLimit`, `reclaimWithinCohort`, `borrowWithinCohort`, `fairSharing.weight`, `AdmissionFairSharing.usageHalfLifeTime`), what each one does to the scheduler, and how a price tier maps onto them.
- **Genuinely new — the preemptible discount, bounded from both sides.** A floor from the tenant's expected goodput loss and a ceiling from the platform's avoided headroom, with the discount chosen inside that band rather than picked because 40% sounds generous.
- **Genuinely new — the dispute protocol**, worked from both sides on the same fleet, including which claims get conceded, what the credit looks like as a FOCUS row, and the restatement rule.

## Core concepts

### 1. The chargeback data path, and the one boundary that matters

A chargeback system is a pipeline whose output is a sentence a stranger will dispute. Before designing any of it, know exactly which stages produce *records* and which produce *measurements*, because those two have completely different behaviour under challenge. A record can be replayed; a measurement can only be re-argued.

```
  THE CHARGEBACK DATA PATH — from telemetry to a disputed invoice line
  ═══════════════════════════════════════════════════════════════════════════

  ┌── STAGE 0 · SOURCES ───────────────────────────────────────────────────┐
  │  kubelet pod-resources API        DCGM exporter          cloud billing │
  │  ListPodResources()               DCGM_FI_PROF_*         FOCUS export  │
  │  {ns,pod,ctr,[device_ids]}        per-UUID counters      EffectiveCost │
  │  ▸ a SCHEDULER RECORD             ▸ a MEASUREMENT        ▸ a RECORD    │
  └────────┬───────────────────────────────┬──────────────────────┬────────┘
           │                               │                      │
           ▼                               ▼                      ▼
  ┌── STAGE 1 · ATTRIBUTION (lesson 01 ladder, module 04 exporter) ────────┐
  │  join on GPU UUID · resolve GPU-<uuid>::N · book MIG stranding         │
  │  emit: gpu_allocated_share{team,pod,gpu_uuid}   ← exact, all regimes   │
  │        gpu_utilised_share{...}                  ← exact only in R1/R2  │
  │        gpu_unallocated_share{gpu_uuid}          ← makes identity A     │
  └────────┬───────────────────────────────┬───────────────────────────────┘
           │                               │
   ╔═══════╪═══════════════════════════════╪═══════════════════════════════╗
   ║       │        T R U S T   B O U N D A R Y                            ║
   ║       │                               │                               ║
   ║  LEFT: derived from recorded     RIGHT: derived from device counters.  ║
   ║  facts (bill line, GPU count,    Under time-slicing this is an         ║
   ║  scheduler binding). Replayable. ESTIMATE with an unbounded            ║
   ║  Survives audit unconditionally. per-tenant error (lesson 01 §5).      ║
   ║       │                               │                               ║
   ║  ⇒ EVERYTHING THAT CROSSES INTO A CHARGE MUST COME FROM THE LEFT.     ║
   ╚═══════╪═══════════════════════════════╪═══════════════════════════════╝
           │                               │
           ▼                               ▼
  ┌── STAGE 2 · LEDGERS ───────────┐  ┌── (report-only) ──────────────────┐
  │  allocated GPU-hours per team  │  │  utilised GPU-hours per team      │
  │  + the unallocatable remainder │  │  + the waste gap, with error band │
  └────────┬───────────────────────┘  └────────────┬──────────────────────┘
           │                                       │
           ▼                                       │
  ┌── STAGE 3 · RATING ────────────────────────────┼──────────────────────┐
  │  rate policy (§3) × tier multiplier (§6)       │                      │
  │  charge = allocated_hours × r_published × m_tier                      │
  │  remainder disposed per policy (§4)            │                      │
  └────────┬───────────────────────────────────────┼──────────────────────┘
           │                                       │
           ▼                                       ▼
  ┌── STAGE 4 · STATEMENT ─────────────────────────────────────────────────┐
  │  per-team line items · provenance columns · the utilisation call-out   │
  │  ▸ THE NUMBER LEAVES YOUR SYSTEM HERE. After this it is finance's.     │
  └────────┬───────────────────────────────────────────────────────────────┘
           │
           ▼
  ┌── STAGE 5 · DISPUTE (§8) ──────────────────────────────────────────────┐
  │  claim → evidence pull → adjudicate → credit as ChargeClass=Correction │
  └────────────────────────────────────────────────────────────────────────┘
```

Read the boundary once more, because it is the whole design. Stage 1 emits two families of series. One family — `gpu_allocated_share` — is a function of the kubelet's own record of which device is bound to which pod. If a tenant disputes it, you replay `ListPodResources()` output for the disputed window and the argument is over in ninety seconds. The other family — `gpu_utilised_share` — is a function of hardware counters that, under time-slicing, cannot be resolved per tenant at all. If you bill on that family, a dispute becomes an argument about whether `DCGM_FI_PROF_GR_ENGINE_ACTIVE` is the right basis, whether the 15-second scrape aliased a bursty workload, and whether the fair-share split you chose was the honest one. You will be right and you will still lose, because the burden of proof landed on the party holding the estimate.

**Design rule that falls out: a chargeback pipeline should be able to produce a complete, correct invoice with the DCGM exporter switched off.** If it cannot, a measurement has leaked into the money path. Run that as a literal test — stop the exporter in staging, regenerate last month's statement, and diff. Any non-zero diff in a `charge` column is a bug in the *architecture*, not the code.

### 2. The maturity ladder: four rungs, not two

"Showback vs chargeback" is a false binary that hides the three decisions that actually matter: who holds the budget, whether the transfer is reversible, and who can veto a charge. Real programmes climb four rungs, and each rung changes exactly one of those.

| Rung | Name | Budget holder | Money moves? | Veto | What breaks here |
|---|---|---|---|---|---|
| 0 | **Informational showback** | platform | no | n/a | nobody reads it |
| 1 | **Published showback** — ranked, named, distributed to leadership | platform | no | n/a | teams dispute the numbers with no stakes, which is *good*: it debugs your pipeline for free |
| 2 | **Soft chargeback** — teams get a GPU budget in units, tracked against it, overage escalates | platform (dollars), team (units) | no | manager | budget becomes a quota nobody enforces |
| 3 | **Hard chargeback** — a general-ledger transfer, platform's cost centre is credited, team's is debited | team | yes | finance | disputes become formal; restatement rules become mandatory |

The temptation is to jump to rung 3 because it has the strongest incentive. Do not, and be able to say why in an interview: **rung 1 is a free correctness test for the pipeline, and rung 3 without a passed correctness test is a credibility event you do not recover from.** Run rung 1 for at least two full billing periods and require zero unexplained variance against the fleet's actual cost before any money moves. The reconciliation identities from module 04's capstone are the pass condition:

```
  IDENTITY A (allocation conservation) — must hold EXACTLY:
      Σ allocated GPU-hours + Σ unallocated GPU-hours  ≡  G × W

  For chargeback specifically, the same identity in dollars:
      Σ tenant charges + Σ remainder disposition + variance  ≡  fleet cost
```

The second form is the one finance cares about. It says: every dollar the fleet cost is either on somebody's statement, in a named remainder bucket, or in a named variance term. There is no fourth place for a dollar to be. If your statement generator cannot close that identity to the cent, it is not ready for rung 3.

Rung 2 deserves more respect than it usually gets. A **unit budget** — "your team has 12,000 GPU-hours this quarter" — is often better than a dollar budget on a GPU fleet, for a structural reason: the fleet's dollar cost per hour moves with the commitment mix (lesson 06) and the tenant cannot influence it, so a dollar budget makes teams accountable for a number they do not control. A GPU-hour budget makes them accountable for exactly the thing they *do* control. Charge dollars for accounting; budget hours for behaviour.

### 3. Rate policy: three ways to turn a fixed cost into a price

An owned or committed GPU fleet is a **largely fixed monthly cost**. The commitment (lesson 06) is owed whether or not you consume; depreciation on owned hardware runs on a straight-line schedule regardless of utilisation; the storage tier, the fabric, the DC contract and the platform headcount do not shrink when a team goes quiet. Chargeback's central arithmetic problem is therefore: *divide a fixed numerator by a variable denominator, monthly, without producing a price that destroys the behaviour you wanted.*

Three policies. They are not interchangeable, and the choice is the single most consequential thing in a chargeback design.

**(a) Floating full-absorption rate.** Recompute the rate every period from what actually happened:

```
    r_float = total_fleet_fixed_cost / Σ chargeable (allocated) GPU-hours
```

This has one enormous virtue — the fleet's books close exactly, every period, by construction — and one fatal dynamic:

```
  THE FLOATING-RATE DEATH SPIRAL  (why full absorption is unstable)
  ═════════════════════════════════════════════════════════════════

    fixed cost C = $191,150/mo, fleet = 64 GPUs × 730 h = 46,720 GPU-h

    period 1 ── allocated 39,700 h  →  r = C/39,700 = $4.8149/GPU-h
                     │
                     │  a team sees $4.81 and does the obvious thing:
                     ▼
    period 2 ── allocated 34,000 h  →  r = C/34,000 = $5.6221/GPU-h   (+17%)
                     │
                     │  the rate rose BECAUSE they cut. Others react.
                     ▼
    period 3 ── allocated 27,000 h  →  r = C/27,000 = $7.0796/GPU-h   (+26%)
                     │
                     ▼
    period 4 ── allocated 20,000 h  →  r = C/20,000 = $9.5575/GPU-h   (+35%)

    ┌──────────────────────────────────────────────────────────────────┐
    │  THE MECHANISM: each tenant's cut raises everyone else's price,  │
    │  including their own next period. The externality is POSITIVE    │
    │  in the wrong direction — cutting usage is individually rational │
    │  and collectively self-defeating, because the numerator is fixed.│
    │  This is a textbook fixed-cost allocation instability, and it is │
    │  why "just divide the bill by usage" fails on a committed fleet. │
    └──────────────────────────────────────────────────────────────────┘

    Note the shape: the price is C/x, a hyperbola. Its slope steepens
    as x falls, so the spiral ACCELERATES. There is no equilibrium
    short of x → 0 unless something outside the loop pins the rate.
```

Nothing about that is hypothetical or GPU-specific — it is what happens to any full-absorption transfer price on a fixed-cost shared asset. It is worse on GPUs than on CPUs for two reasons: the fixed fraction is higher (the silicon is the cost, and it is committed), and the demand is lumpier (whole nodes of 8, projects that start and stop), so a single team's decision can move the denominator 15% in one period.

**(b) Fixed internal transfer price.** Publish a rate for the year, computed from *planned* hours:

```
    r_pub = budgeted_fixed_cost / budgeted_chargeable_GPU-hours     (set once)
```

Now the tenant faces a constant price and the loop is broken: one team's cut does not raise another team's rate. The cost is that the books no longer close automatically — the platform runs a **variance** every period, which someone must own. That is a feature, not a bug, and §5 shows how to decompose it so the variance is explainable rather than mysterious.

**(c) Two-part tariff.** Split the fixed cost into a **capacity fee** (a subscription to a reserved slice, charged whether used or not) and a **usage fee** (per GPU-hour actually held above the reserved slice):

```
    charge_team = F_team              +   r_marginal × max(0, h_team − q_team)
                  ↑ capacity fee          ↑ overage on hours above quota
                  = (q_team / Σq) × C_fixed_recovered
```

This is what the cloud providers themselves sell you (a commitment plus on-demand — lesson 06), and it is the most economically correct answer for a fleet where the capacity was bought *for* named teams. It prices the two different things separately: holding a claim on capacity, and consuming it. It is also the most work to administer, because `q_team` has to be negotiated annually and re-negotiated whenever a team's roadmap changes.

**How to choose:**

| | Floating full-absorption | Fixed transfer price | Two-part tariff |
|---|---|---|---|
| Books close automatically | **yes** | no (variance) | no (variance on overage) |
| Rate stable for tenants | no | **yes** | **yes** |
| Death-spiral risk | **high** | none | none |
| Signals fleet-wide waste | **strongly** | weakly (via annual reset) | via capacity fee |
| Admin burden | low | low | **high** |
| Right when | usage is stable and the org is small | most shared research/platform fleets | capacity was bought per-team |

**The practical answer for a shared GPU fleet is (b), with the utilisation signal moved from the price into the report.** You publish `$4.66/GPU-hour` for the fiscal year; you publish, every month, the fleet's realised full-absorption rate `r_float` *next to it* as a headline efficiency metric with no billing weight. Teams get a stable price to plan against and still see, in bold, that the fleet ran at 85% and the money-equivalent rate was $4.81 rather than the $4.66 they paid — with the difference named as a platform under-recovery. **You keep the signal and lose the spiral, because the signal is now information rather than a price.**

Two dampers if you are forced into (a) by finance:

- **Floor the denominator.** `r = C / max(actual_hours, floor_hours)` with `floor_hours` set at, say, 80% of available capacity. The rate can fall when the fleet is busy but cannot rise past a stated ceiling when it is quiet. The platform eats the shortfall below the floor. This is a floating rate with the hyperbola's tail cut off.
- **Lag and smooth.** Set the rate from a trailing 3-period mean of allocated hours. This does not remove the instability, it just slows it — a tenant's cut still raises everyone's price, three months later. Prefer the floor.

### 4. The unallocatable remainder: six sources, four disposals

Between "what the fleet cost" and "what tenants held" there is always a gap, and it is bigger on GPUs than on any other resource. Naming its parts is the difference between an honest statement and a mystery surcharge.

```
  THE REMAINDER WATERFALL — one month on a 64×H100 fleet
  ══════════════════════════════════════════════════════════════════════════

  CALENDAR CAPACITY                          64 GPUs × 730 h = 46,720 GPU-h
  ██████████████████████████████████████████████████████████████████ 100.0%
   │
   ├─▶ (1) UNAVAILABLE — node down, Xid-quarantined, in maintenance
   │       1,402 GPU-h  ███  3.0%     cause: hardware + platform ops
   │
   ├─▶ (2) FRAGMENTED — free but unschedulable: no 8-GPU hole left,
   │       topology constraint unmet, MIG geometry stranding (L04)
   │       2,240 GPU-h  █████  4.8%   cause: request-shape mix + packing
   │
   ├─▶ (3) HEADROOM/IDLE POOL — genuinely free, deliberately kept free
   │       3,378 GPU-h  ███████  7.2% cause: policy (burst absorption)
   │
   ├─▶ (4) SYSTEM OVERHEAD — daemonsets, device-plugin, DCGM, validators
   │       (on this fleet: 0 whole GPUs; charged as node overhead, not GPU)
   │
   ├─▶ (5) COMMITMENT UNUSED — FOCUS CommitmentDiscountStatus='Unused'
   │       hours you owe but no device existed to run (L06). Not shown
   │       here (commitment fully covered by the fleet), but this is
   │       where a mis-sized reservation lands, and it is NOT a GPU-hour
   │       — it is a dollar with no hour behind it.
   │
   └─▶ (6) ALLOCATED TO TENANTS
           39,700 GPU-h  ████████████████████████████████████████  85.0%

  IDENTITY CHECK:  1,402 + 2,240 + 3,378 + 39,700 = 46,720  ✓  exact.
  REMAINDER TOTAL: 7,020 GPU-h = 15.0% of calendar
                 = 7,020 × $4.66 = $32,713.20 at the published rate
```

Note the ordering: it is causal, not arbitrary. Unavailable capacity was never schedulable, so it cannot be fragmented. Fragmentation is measured against what *was* schedulable. Headroom is what was schedulable, unfragmented, and deliberately left alone. Each bucket has a different owner and a different remedy, which is exactly why lumping them into one "idle" number is the most common way a chargeback programme loses an argument — a tenant will correctly point out that the number contains both "your hardware broke" and "we chose not to run."

Now: who pays for 7,020 GPU-hours nobody held? Four policies, costed on the same fleet.

| Policy | Formula | Effect on the four teams (see §Worked example) | When it is right |
|---|---|---|---|
| **(i) Platform absorbs** | remainder → platform cost centre, published as a line | research $83,880 · infer $55,337 · eval $20,131 · expl $23,300; platform carries $8,501 variance | **default.** The platform chose the fleet size; the platform owns the sizing error |
| **(ii) Proportional** | surcharge = remainder$ / Σ allocated h, added to r | +$0.2141/h → research +$3,854 · infer +$2,034 · eval +$1,542 · expl +$1,071 | when finance mandates full recovery and teams are comparable |
| **(iii) Even split** | remainder$ / n_teams | +$2,125.33 to each — exploratory (5,000 h) pays the same as research (18,000 h) | almost never. Rewards the largest consumer |
| **(iv) Causal** | each bucket to its cause | (1) → platform. (2) → the teams whose request shape caused it. (3) → whoever asked for headroom | when fragmentation is dominated by one team's shape and you can prove it |

**Recommendation, and be able to defend it: (i) within a tolerance band, (ii) only above it, never (iii).** Concretely: the platform absorbs remainder up to ±5% of fleet cost as its own operating variance, and if the variance exceeds that band two periods running, the *published rate is reset* for the next fiscal period rather than a surcharge being retro-applied. That preserves the fixed-rate stability from §3(b) while keeping the platform honest about sizing. Retro-surcharges are the single fastest way to lose tenant trust, because they convert a price a team planned against into a price they cannot plan against.

Policy (iv) deserves one caveat that separates a careful engineer from a clever one. Fragmentation is *jointly* caused: an 8-GPU request only strands 3 GPUs because somebody else's 5-GPU job is sitting on the node. There is no non-arbitrary way to assign a joint cost to one party — this is a cooperative-game problem, and the principled answers (Shapley value over request shapes) are computable but unexplainable to a tenant. **Prefer publishing fragmentation as a platform metric with a named remedy (bin-packing policy, gang-scheduling, MIG geometry standardisation — lesson 04) over allocating it.** A number you can act on beats a number you can bill.

Finally, note where the remainder lives in the industry schema, because lesson 10 builds on this. FOCUS's split-cost-allocation feature models exactly this shape: the origin charge is the shared resource, allocated charges are the tenants' portions, and the spec requires that **`AllocatedResourceId` be null on the row representing the unallocated portion of the origin charge after split cost allocation.** The spec's own example query for finding it is:

```sql
-- FOCUS: total unallocated split cost, per shared resource
SELECT ResourceId, SUM(EffectiveCost) AS TotalEffectiveCost
FROM   focus_data_table
WHERE  ChargeCategory = 'Usage'
  AND  ChargePeriodStart >= ? AND ChargePeriodEnd <= ?
  AND  AllocatedMethodId IS NOT NULL
  AND  AllocatedResourceId IS NULL
GROUP BY ResourceId
```

That is the standard telling you the remainder is a **first-class row**, not a rounding error. Your statement should treat it the same way.

### 5. Rating and variance: making the books close, and explaining it when they don't

Rating is the step that turns `(team, allocated_hours, tier)` into a dollar figure. On a fixed transfer price it is one multiply, which makes it tempting to under-engineer. Resist: the value is not in the arithmetic, it is in the **provenance columns that ride alongside it**, because those are what a dispute consumes.

A statement line item that survives an audit carries, at minimum:

| Field | Example | Why it must be on the line |
|---|---|---|
| `charge_period_start` / `_end` | `2026-07-01T00:00Z` / `2026-08-01T00:00Z` | disputes are always scoped to a window |
| `team` / `cost_centre` | `research-llm` / `CC-4412` | the GL target |
| `allocated_gpu_hours` | `18000.00` | the billed quantity |
| `basis` | `allocated` | states which ledger — pre-empts the entire "we only used" argument |
| `attribution_regime` | `exclusive` \| `mig` \| `timeslice` \| `dra` | lesson 01; tells a reader whether the quantity was recorded or estimated |
| `rate_published` | `4.66` | the price |
| `rate_basis` / `rate_effective_from` | `EffectiveCost / planned chargeable h` / `2026-01-01` | pre-empts "where did 4.66 come from" |
| `tier` / `tier_multiplier` | `standard` / `1.00` | §6 |
| `charge` | `83880.00` | the money |
| `utilised_gpu_hours` (report-only) | `15300.00` | the efficiency signal, explicitly **not** a billing input |
| `utilisation_confidence` | `exact` \| `estimated±band` | honesty label; `estimated` for time-sliced rows |
| `source_snapshot_id` | `podres-2026-07-31T23:59Z` | the replay key. Without this the dispute protocol has no evidence step |

**Variance decomposition.** With a fixed published rate, the statement total will not equal the fleet's cost. That difference is not one number — it is exactly three, and they sum to the cent. On the worked-example fleet:

```
  FLEET FIXED COST                                          C = $191,150.00
  PLANNED chargeable hours (rate was set on these)          h_plan = 41,000
  PUBLISHED rate  r_pub = round(C / h_plan, 2)              = $4.66/GPU-h
                        ( exact:  191,150 / 41,000 = 4.66219512… )

  ACTUAL allocated hours                                    h_act  = 39,700
  BILLED at ×1.00 equivalent                39,700 × 4.66  = $185,002.00
  TIER MIX effect (priority premium − best-effort discount) = −$  2,353.30
  TOTAL BILLED                                              = $182,648.70

  UNDER-RECOVERY  =  191,150.00 − 182,648.70                = $  8,501.30

  ─── DECOMPOSITION ────────────────────────────────────────────────────────
   1. VOLUME variance  (h_plan − h_act) × r_pub
                       = (41,000 − 39,700) × 4.66           = $  6,058.00
      ▸ meaning: the fleet was less fully allocated than planned.
        Owner: platform (sizing) + demand forecast.

   2. MIX variance     tier discounts net of tier premiums
                       = 13,420.80 − 11,067.50              = $  2,353.30
      ▸ meaning: more best-effort hours than the tier plan assumed.
        Owner: platform (tier pricing), and it is a GOOD variance —
        it bought backfill and reclaimability. Report it as such.

   3. RATE-ROUNDING    h_plan × (C/h_plan − r_pub)
                       = 41,000 × 0.00219512…               = $     90.00
      ▸ meaning: you published 4.66 instead of 4.6621951…
        Owner: nobody. It is arithmetic. State it so it isn't mistaken
        for a leak.

   CHECK: 6,058.00 + 2,353.30 + 90.00 = 8,501.30  ✓  closes exactly.
```

That three-line reconciliation is the single most useful artifact in a chargeback programme, and it is the thing that converts a finance partner from adversary to ally. "We are $8.5k under-recovered" invites investigation. "We are $8.5k under-recovered: $6,058 volume because allocation ran 3.2% below plan, $2,353 mix because best-effort hours ran above plan — which is the tier working — and $90 rounding" ends the conversation.

**Restatement.** Chargeback numbers change after the fact: a late-arriving billing correction from the provider, an exporter bug found in week three, a dispute credit. Do not silently rewrite a closed period. FOCUS gives you the vocabulary and you should adopt it internally even if you never emit a FOCUS file: `ChargeClass = 'Correction'` marks a charge that corrects a previously closed billing period, and `ChargeCategory = 'Adjustment'` covers charges the provider (here, you) applies that do not fall into another category. Concretely: a credit lands in the *current* period as a negative-value row tagged `Correction`, referencing the original period. The closed period's statement never changes. This is not bureaucracy — it is what makes month-over-month comparisons meaningful, and it is the rule finance already lives by.

### 6. Queue-wait billing: pricing urgency and patience on real scheduler primitives

On a fleet where demand exceeds supply, **the queue is a rationing mechanism whether or not you price it.** Unpriced, it rations by escalation: the job that starts first belongs to whoever complains hardest to whoever has the most seniority. That is not a neutral default; it is a specific allocation rule with terrible properties (opaque, unappealable, uncorrelated with value). Pricing the queue replaces it with a rule anybody can read.

Three tiers is the standard shape. What matters for an interview is that you can say **exactly which scheduler object implements each one**, so here it is against Kueue's API (`kueue.x-k8s.io`, `apis/kueue/v1beta2`), which is the CNCF-adjacent batch queueing layer most GPU platforms standardise on.

```
  PRICE TIER  →  SCHEDULER PRIMITIVE  →  WHAT THE SCHEDULER ACTUALLY DOES
  ══════════════════════════════════════════════════════════════════════════

  ┌─ PRIORITY (×1.25) ──────────────────────────────────────────────────┐
  │  ClusterQueue cq-priority, in cohort "fleet"                        │
  │    nominalQuota:      14 GPUs      ← reserved floor, never lent out │
  │    lendingLimit:      0            ← "reserves for its exclusive    │
  │                                       use nominalQuota − lending-   │
  │                                       Limit" ⇒ all 14 held back     │
  │    borrowingLimit:    8            ← may borrow up to 8 more        │
  │    preemption:                                                      │
  │      reclaimWithinCohort: Any      ← may preempt ANY workload in    │
  │                                       the cohort using above its    │
  │                                       nominal quota, ignoring       │
  │                                       priority                      │
  │      borrowWithinCohort: {policy: LowerPriority}                    │
  │  ⇒ THE PREMIUM BUYS: a floor nobody can borrow (lendingLimit 0)     │
  │     plus the right to reclaim. That floor is idle capacity the      │
  │     platform is holding — which is exactly what the +25% funds.     │
  └─────────────────────────────────────────────────────────────────────┘

  ┌─ STANDARD (×1.00) ──────────────────────────────────────────────────┐
  │  ClusterQueue cq-standard, same cohort                              │
  │    nominalQuota:      36 GPUs                                       │
  │    lendingLimit:      unset (null) ← all of it lendable when idle   │
  │    borrowingLimit:    12                                            │
  │    preemption:                                                      │
  │      reclaimWithinCohort: LowerPriority                             │
  │      withinClusterQueue:  LowerOrNewerEqualPriority                 │
  │  ⇒ ordinary citizenship: lends when idle, borrows when others are.  │
  └─────────────────────────────────────────────────────────────────────┘

  ┌─ BEST-EFFORT (×0.60) ───────────────────────────────────────────────┐
  │  ClusterQueue cq-besteffort, same cohort                            │
  │    nominalQuota:      0 GPUs       ← owns NOTHING                   │
  │    borrowingLimit:    64           ← may borrow the whole fleet     │
  │    (its workloads carry the lowest PriorityClass)                   │
  │  ⇒ pure backfill. Everything it runs is borrowed, so every other    │
  │     queue's reclaimWithinCohort can take it back at any moment.     │
  │     THE DISCOUNT IS THE PRICE OF THAT OPTION.                       │
  └─────────────────────────────────────────────────────────────────────┘

  ┌─ FAIR SHARING (cross-cutting, prevents one tier eating a cohort) ───┐
  │  fairSharing.weight on each ClusterQueue (default 1)                │
  │  status.fairSharing.weightedShare = max over resources of           │
  │        (usage above nominal quota) / (lendable in cohort) / weight  │
  │  Admission prefers the LOWEST weightedShare; preemption targets the │
  │  HIGHEST. weight 0 ⇒ infinite share ⇒ always last.                  │
  │                                                                     │
  │  AdmissionFairSharing (usage-based) adds HISTORY:                   │
  │    usageHalfLifeTime      — past usage decays by half after this    │
  │    usageSamplingInterval  — how often consumedResources updates     │
  │                             (default 5m)                            │
  │    resourceWeights        — per-resource weights (default 1)        │
  │  ⇒ a team that burned the cohort last week is admitted later this   │
  │     week, with the memory decaying exponentially. This is a real    │
  │     historical-usage ledger inside the scheduler.                   │
  └─────────────────────────────────────────────────────────────────────┘
```

The load-bearing insight is in the priority box: **the premium is not a fee for jumping a line, it is the price of the idle capacity that makes jumping possible.** `lendingLimit: 0` on a 14-GPU nominal quota means up to 14 GPUs can sit unused while other queues wait. Somebody pays for those hours. Charging the priority tier a multiplier is how you make the beneficiary pay rather than socialising it across the fleet. If you cannot explain a premium in terms of a specific resource being held out of the pool, you have invented a tax, not a price.

**Now derive the best-effort discount properly, from both directions.** Most write-ups pick a number. Bound it instead.

*Floor — what the tenant needs to be made whole.* A preemptible tenant loses work on each preemption: the progress since the last checkpoint, plus restart. Let `λ` be preemptions per GPU-hour held, `Δ` the checkpoint interval, and `δ` the restart cost. Expected lost work per preemption is `Δ/2 + δ`, so:

```
    goodput_fraction  =  1 − λ · (Δ/2 + δ)

    with λ = 0.05 preemptions per GPU-hour held (aggressive reclaim),
         Δ = 0.5 h  (30-minute checkpoints),
         δ = 0.1 h  (restart, re-warm, re-shard)

    goodput = 1 − 0.05 × (0.25 + 0.1) = 1 − 0.0175 = 0.9825
    ⇒ the tenant needs ≥ 1.75% off to be indifferent on goodput alone.
```

Add the engineering cost of making a workload genuinely preemptible — checkpointing, idempotent restarts, tolerating a node vanishing mid-step — amortised over the workload's life. A team that estimates that at 3% of its compute bill puts the **floor at roughly 5%**.

*Ceiling — what the option is worth to the platform.* Reclaimable hours substitute for headroom. If 7,200 best-effort GPU-hours let you run the fleet with 3,600 fewer GPU-hours of standing headroom per month, the avoided cost is `3,600 × $4.66 = $16,776`, against a best-effort bill at full rate of `7,200 × $4.66 = $33,552`. So a discount up to **50%** is still value-positive for the platform.

*Choose inside the band.* `[5%, 50%]` → pick **40%** (`×0.60`). Sit well above the floor so the tier is obviously attractive and teams actually declare flexibility instead of faking urgency; sit below the ceiling so the platform keeps some of the surplus to fund the under-recovery from §5. **That is the whole argument, and it is reproducible with your own λ, Δ, δ and headroom numbers** — which is the point. A discount you can derive is a discount you can defend when a team asks for 60%.

One correction to a common formulation, and it matters. The instinct is to say "bill preemptible tenants on *utilised* hours since they don't hold capacity against anyone." That breaks the §1 trust boundary for no gain, because **preemption already handles it on the allocated ledger**: when a best-effort pod is preempted, it stops holding the device, and its allocated hours stop accruing that instant. The allocated ledger automatically reflects interruption. You keep one billing rule for the whole fleet — charge allocated — and express the tier entirely as a multiplier. Simpler, and it never puts an estimate in the money path.

### 7. Anti-gaming: five exploits, and the mechanism that closes each

Every billing rule is a payoff function, and a team of smart engineers will optimise against it within one quarter. Design for that.

| # | Exploit | Why it works | The counter, and the mechanism |
|---|---|---|---|
| 1 | **Pad the reservation** — request 32 GPUs, use 12, guarantee capacity | under utilised-billing the pad is free | **allocation billing** makes the pad cost full price, and **idle reclaim** (lesson 03) takes the pad back after a sustained-idle window. Both are needed: billing makes it expensive, reclaim makes it impossible |
| 2 | **Under-request and thrash** — request 4, rely on preemption/retries to actually get 16 | a smaller allocated number is a smaller bill | priority pricing means under-requesting buys *waiting*, not savings; and `AdmissionFairSharing.usageHalfLifeTime` means the thrashing team's historical usage still counts against its admission priority even though its instantaneous allocation is low. **The scheduler remembers.** |
| 3 | **Sleeper pods** — keep a 1-GPU pod alive on each node to reserve topology | tiny allocated hours, large blocking effect | this is a fragmentation attack, not a cost attack. Counter with topology-aware packing plus a rule that fragmentation attributable to a single tenant's shape is *reported by tenant* in showback. Naming it is usually enough |
| 4 | **Tier arbitrage** — declare best-effort for the discount, then escalate when preempted | the discount is taken but the flexibility is not delivered | make the tier a *technical* commitment, not a declaration: best-effort queues carry the lowest `PriorityClass` and `borrowingLimit` covers their whole footprint, so preemption is automatic and unappealable. An escalation has no lever to pull. Track "preemption complaints per best-effort team" as a metric and move repeat offenders to standard |
| 5 | **Label laundering** — retag pods to another team's namespace at month end | attribution keys on labels | key attribution on the **namespace binding recorded by the kubelet at allocation time** (the pod-resources snapshot, §1), not on a mutable label read at report time. Snapshot ownership when the device is bound; a label edited afterwards changes nothing |

Exploit 5 is the one that quietly invalidates a whole programme, and it is worth stating the general principle it comes from: **attribution must key on a fact recorded at the moment the resource was consumed, not on metadata read at reporting time.** Anything queried at report time can be changed between consumption and report. This is the same reason the statement line carries `source_snapshot_id`.

### 8. The dispute protocol

Disputes are not failures of the system; they are the system working. Have a written protocol before you need one, because inventing one under pressure produces concessions you cannot generalise.

```
  DISPUTE TIMELINE — the 48 hours after a team escalates
  ═════════════════════════════════════════════════════════════════════════

  T+0h   CLAIM RECEIVED
         ▸ require: team, charge_period, line item, the specific quantity
           disputed, and the claimed correct value. A claim without a
           number is a complaint; route it to showback, not to dispute.
         ▸ FREEZE nothing. The closed period stays closed (§5). Any
           remedy lands as a Correction in the CURRENT period.
              │
              ▼
  T+2h   EVIDENCE PULL  (this is why source_snapshot_id exists)
         ┌──────────────────────┬─────────────────────────────────────────┐
         │ PLATFORM BRINGS      │ TENANT BRINGS                           │
         ├──────────────────────┼─────────────────────────────────────────┤
         │ pod-resources        │ their own kubectl/job records for the   │
         │  snapshots, per hour │  same window                            │
         │ allocated-share      │ specific hours they claim they did not  │
         │  series, per GPU UUID│  hold a device                          │
         │ identity-A check for │ evidence of platform-caused holds       │
         │  the disputed nodes  │  (cordons, failed drains, stuck PVCs)   │
         │ node event log       │                                         │
         └──────────────────────┴─────────────────────────────────────────┘
              │
              ▼
  T+8h   ADJUDICATE — sort every disputed hour into exactly one bucket
         ┌───────────────────────────────────────────────────────────────┐
         │ A · TENANT HELD IT, DIDN'T USE IT       → NO CREDIT.          │
         │     The hold is the charge. This is the policy, not a bug.    │
         │ B · TENANT DIDN'T HOLD IT               → CREDIT + fix the    │
         │     (attribution defect)                  exporter. Also a P1.│
         │ C · PLATFORM FORCED THE HOLD            → CREDIT. Cordon,     │
         │     (couldn't release / couldn't run)     failed drain, an    │
         │                                           un-drainable node.  │
         │ D · HARDWARE FAULT ON A HELD DEVICE     → CREDIT for the      │
         │     (Xid, thermal quarantine)             fault window only.  │
         └───────────────────────────────────────────────────────────────┘
              │
              ▼
  T+24h  RESPOND — in writing, per bucket, with the hour counts and the
         evidence key for each. Bucket A gets a *reason*, not just a
         refusal: "you held 5,000 h; here is the release mechanism and
         here is the best-effort tier that would have cost you 40% less."
              │
              ▼
  T+48h  REMEDY — credit posted as a NEGATIVE line in the current period:
             ChargeCategory = 'Adjustment', ChargeClass = 'Correction',
             referencing the original charge_period.
         ▸ Plus, if bucket B was non-empty: a fleet-wide re-check, because
           an attribution defect that hit one team hit others silently.
```

The design property that makes this protocol work is that **bucket A is decided by policy and buckets B–D are decided by evidence.** You never negotiate bucket A — conceding it once destroys allocation billing for everyone, because the concession generalises instantly and every team will discover it by the next period. You concede B, C and D readily and fast, because conceding them cheaply is what buys you the credibility to refuse A.

## Perspectives

**Platform / mechanism-design view.** Every chargeback rule is a payoff function that a team of clever engineers will optimise against inside one quarter, so the design question is never "is this fair?" but "what is the equilibrium?" Charge utilised and the equilibrium is maximum hoarding at zero cost. Charge allocated and the equilibrium is `B'(h*) = r` — teams hold capacity up to the point where the marginal private benefit equals the price, which is the efficient outcome. Add a preemptible discount inside a derived band and you have created a second equilibrium for flexible work that fills the troughs. None of that is accounting; it is market design wearing an accountant's coat.

**Finance view.** The floating full-absorption rate is the intuitive design and the wrong one, and knowing *why* is the thing that earns credibility with a CFO's office. Finance's real requirement is not "the books close every month" — it is "variances are explainable and someone owns them." A fixed transfer price with a three-term variance decomposition delivers that better than a floating rate that closes automatically while quietly destabilising demand. Say it in their vocabulary: **you are choosing a standard cost with variance analysis over an actual-cost absorption method, for the same reason manufacturing did decades ago.**

**Tenant view.** Being charged for a *hold* rather than a *use* feels wrong the first time, and the discomfort is the mechanism working. The tenant's legitimate complaint is not about the rule; it is about controllability. So the platform's obligation, in exchange for allocation billing, is to make releasing capacity genuinely easy and fast: a working preemptible tier, a clean release path, no penalty for right-sizing mid-quarter, and a queue that gives capacity back promptly when asked. **Allocation billing without an easy release path is extraction, not incentive design**, and a tenant is right to say so.

**Scheduler view.** From inside the queue, a price tier is not a billing construct — it is `nominalQuota`, `lendingLimit`, `borrowingLimit`, and the three preemption policies, and nothing else. If a price tier does not translate into a change in one of those fields, it changes nothing about who runs first and it is fiction. That is the test to apply to any queue-pricing proposal: name the field it changes. Conversely, every one of those fields already *is* a price in disguise — `lendingLimit: 0` is a purchase of idle capacity, made by the scheduler, that somebody's budget is silently funding until you make it explicit.

**Auditor view.** The only question an auditor asks is "show me how you got this number," and it has exactly two acceptable answers: replay a snapshot, or point at an identity. `Σ tenant charges + Σ remainder + Σ variance ≡ fleet cost` is the identity; `source_snapshot_id` is the replay. A chargeback system with both is boring to audit, which is the highest compliment the function can pay.

## Real-world use cases

- **The FOCUS specification's own split-cost-allocation worked example** (`specification/appendix/split_cost_allocation_examples.md`, spec repo at `main`, read 2026-08-17). A shared container cluster `cluster-shared-01` with an origin `EffectiveCost` of $100.00 for one charge period is split across three consuming workloads by measured vCPU-hours: 40 / 35 / 25 vCPU-hours → $40.00 / $35.00 / $25.00. The generator emits **four** rows — one origin row zeroed to $0.00 plus three allocated rows — and the origin dimensions (`ServiceName`, `ServiceCategory`, `ResourceId`) are preserved on every row so the dataset still reconciles to the invoice on the origin service. What it shows: the industry standard models internal chargeback exactly as this lesson does — a shared parent, named children, an explicit method, and a conservation requirement that the children sum to the parent. It also shows the alternative the spec explicitly permits: it does **not** prescribe whether the origin row is zeroed, omitted, or paired with an offsetting row, only that the allocated charges sum to the origin total. Your statement generator has the same freedom and should document which it chose.

- **The FOCUS `DataGeneratorCalculatedSplitCostAllocationHandling` attribute** (spec 1.3+). Three normative requirements, and they are precisely the invariants a chargeback engine needs: dimension columns on an allocated charge must match the origin charge; non-summable metrics (unit prices) must match the origin; and **summable metrics (costs and quantities) must sum across allocated charges to the origin charge's value**. What it shows: the conservation identity you already know from module 04's capstone is not a local convention — it is a normative requirement in the industry cost standard. When you say "the statement must close to the cent," you are quoting a spec, not expressing a preference.

- **Kueue's `AdmissionFairSharing` configuration** (`kubernetes-sigs/kueue`, `apis/config/v1beta2`, read 2026-08-17). `usageHalfLifeTime` — "the time after which the current usage will decay by a half," with 0 meaning usage resets immediately — plus `usageSamplingInterval` (default 5m) and per-resource `resourceWeights`. What it shows: the "a team that consumed more recently gets admitted later" mechanism is not a scheduler-design thought experiment; it is a shipped, configurable exponential-decay usage ledger inside the queue. It is also the closest thing in the OSS ecosystem to a *non-monetary* chargeback — the currency is admission order rather than dollars, and it composes with a priced tier rather than competing with it.

- **Kueue's `ClusterQueue` preemption surface** (`apis/kueue/v1beta1`/`v1beta2`, `clusterqueue_types.go`). `reclaimWithinCohort` ∈ {`Never` (default), `LowerPriority`, `Any`}; `withinClusterQueue` ∈ {`Never` (default), `LowerPriority`, `LowerOrNewerEqualPriority`}; `borrowWithinCohort.policy` ∈ {`Never`, `LowerPriority`} with an optional `maxPriorityThreshold`; and a CEL validation rule that rejects `reclaimWithinCohort: Never` combined with a non-`Never` `borrowWithinCohort`. What it shows: the defaults are *all* `Never`. A cluster that has not deliberately configured preemption cannot support a best-effort tier at all, because nothing will ever reclaim the borrowed capacity — so the discount would be pure giveaway. Check the defaults before pricing the tier.

## Worked example

**The fleet.** 64 × H100 SXM, one cluster, four tenant teams, committed capacity. Every dollar figure is an **August 2026 snapshot**; the durable content is the method.

*Fixed monthly cost, built bottom-up (lesson 07's normalisation):*

| Component | Basis | $/month |
|---|---|---|
| GPU capacity commitment | 64 GPUs × 730 h × $2.10/GPU-h (Silver-tier, 1-yr reserved — module 03 capstone anchor) | $98,150.40 |
| Shared storage tier | parallel FS, training + checkpoints | $18,000.00 |
| Networking + egress | fabric amortisation + observed egress | $6,000.00 |
| Platform staff | 3.0 FTE fully loaded @ $260k/yr | $65,000.00 |
| Observability + tooling | Prometheus/Grafana/DCGM stack, cost pipeline | $4,000.00 |
| **Total fixed cost `C`** | | **$191,150.00** |

Calendar capacity `= 64 × 730 = 46,720 GPU-h`. Fully-loaded cost per **calendar** GPU-hour `= 191,150 / 46,720 = $4.0914`. Note that already: the raw silicon rate is $2.10 and the fully-loaded calendar rate is $4.09 — **a 1.95× loading**, before a single idle hour. A team that benchmarks your internal rate against a rental sticker is comparing two different things, and the statement should say so in a footnote.

*Capacity accounting for July 2026:*

```
  46,720  calendar GPU-h
 −  1,402  unavailable (3.0%: node failures, Xid quarantine, maintenance)
 ─────────
   45,318  available
 −  2,240  fragmented / unschedulable (4.8% — lesson 04)
 −  3,378  headroom & idle pool (7.2%)
 ─────────
   39,700  ALLOCATED to tenants (85.0% of calendar)

  Identity A:  39,700 + 3,378 + 2,240 + 1,402 = 46,720  ✓
```

*Rate policy.* Fixed transfer price, set at the start of the fiscal year on planned chargeable hours of **41,000/month**:

```
  r_exact = 191,150 / 41,000 = $4.66219512…/GPU-h
  r_pub   = $4.66/GPU-h          (published, fixed for the year)
```

*Tiers.* Priority `×1.25`, Standard `×1.00`, Best-effort `×0.60` (derived in §6).

*The statement.* All four teams are billed on **allocated** GPU-hours (basis: `allocated`), at the published rate, times their tier multiplier.

| Team | Tier | Allocated GPU-h | Utilised GPU-h | Util % | Base (×$4.66) | Tier effect | **Charge** |
|---|---|---:|---:|---:|---:|---:|---:|
| `research-llm` | Standard ×1.00 | 18,000 | 15,300 | 85.0% | $83,880.00 | $0.00 | **$83,880.00** |
| `product-infer` | Priority ×1.25 | 9,500 | 8,265 | 87.0% | $44,270.00 | +$11,067.50 | **$55,337.50** |
| `eval-batch` | Best-effort ×0.60 | 7,200 | 5,760 | 80.0% | $33,552.00 | −$13,420.80 | **$20,131.20** |
| `exploratory` | Standard ×1.00 | 5,000 | 1,500 | 30.0% | $23,300.00 | $0.00 | **$23,300.00** |
| **Total** | | **39,700** | **30,825** | **77.6%** | **$185,002.00** | **−$2,353.30** | **$182,648.70** |

*Reconciliation.* `C = $191,150.00`, billed `= $182,648.70`, under-recovery `= $8,501.30`, decomposing exactly as in §5: volume $6,058.00 + mix $2,353.30 + rounding $90.00. **The platform absorbs it**, because 4.4% is inside the ±5% tolerance band, and the statement says so on its own line rather than hiding it in a rate.

*The remainder, published (not billed):*

| Bucket | GPU-h | At $4.66 | Owner | Named remedy |
|---|---:|---:|---|---|
| Unavailable (hardware/maintenance) | 1,402 | $6,533.32 | platform + vendor | RMA cadence; Xid triage SLA (module 03) |
| Fragmented / unschedulable | 2,240 | $10,438.40 | jointly caused | gang scheduling, MIG geometry standardisation (lesson 04) |
| Headroom / idle pool | 3,378 | $15,741.48 | platform (policy) | shrink as best-effort backfill grows |
| **Total remainder** | **7,020** | **$32,713.20** | | |

*The utilisation call-out (report-only, no billing weight).* Fleet allocated 85.0% of calendar; tenants utilised 77.6% of what they allocated. The realised full-absorption rate — what the fleet's hours *actually* cost — is `191,150 / 39,700 = $4.8149/GPU-h`, **3.3% above the $4.66 you were charged**; the difference is the platform's under-recovery, published here rather than passed through. And the number that should embarrass everyone: `39,700 − 30,825 = 8,875` allocated-but-unused GPU-hours, worth **$41,357.50** at the published rate. `exploratory` alone accounts for 3,500 of those hours ($16,310).

*Per-team unit economics (lesson 05's join, one line each).* `product-infer` served 1.62 billion tokens: `$55,337.50 / 1,620 M-tokens = $0.03416 per million tokens`. `research-llm` completed 46 training runs: `$83,880.00 / 46 = $1,823.48 per run`. These are the numbers that make a statement legible to someone who does not care about GPU-hours, and they belong on the statement.

---

### The dispute, worked from both sides

**T+0h — the claim.** `exploratory` disputes its $23,300.00. Written claim: *"We allocated 5,000 GPU-hours and utilised 1,500. We should be charged for 1,500 hours = $6,990.00. Requested credit: $16,310.00."*

**The tenant's case, in its strongest form** (state it well before answering it — an argument you have not steelmanned is one you will lose):

1. *Utilisation is what the platform itself measures and publishes.* The platform's own showback shows 30%. Charging on a number the platform doesn't feature is inconsistent.
2. *Some of the hold was not ours.* 412 GPU-hours fell in a window where the nodes were cordoned for a platform-driven driver upgrade — jobs could not be rescheduled, and the pods could not be evicted cleanly, so the devices stayed bound to us while we could not use them.
3. *Some of the hardware was broken.* 1,100 GPU-hours show `SM_ACTIVE = 0` in the platform's own DCGM series. We were charged for hardware that did nothing.
4. *No one else was waiting.* The fleet had 3,378 GPU-hours of headroom. Our hold denied nobody anything, so the opportunity-cost argument for allocation billing does not apply this month.

**The platform's case, point by point:**

1. **Rejected (bucket A).** Utilisation is published as an efficiency signal precisely because it is *not* the billing basis, and the statement's `basis: allocated` field says so on every line. The reason is structural, not preferential: under time-slicing the utilised number is an estimate with an unbounded per-tenant error (lesson 01), and money does not move on estimates (§1 trust boundary). The reciprocal commitment is real, though, and should be stated in the response: the platform guarantees a working release path and a cheaper tier — which is the substance of point 4's answer.
2. **Conceded (bucket C).** Evidence pull confirms it: node events show `cordon` at 2026-07-14T09:12Z across 4 nodes, `SchedulingDisabled` for 6h11m, and the pod-resources snapshots show `exploratory` pods still bound throughout. Recomputed from the snapshots the platform-forced hold is **412 GPU-h**, not an estimate. Credit `412 × $4.66 = $1,919.92`.
3. **Partly conceded (bucket D), and this is where evidence beats narrative.** Of the 1,100 zero-activity GPU-hours: the node event log shows `Xid 79` (GPU fell off the bus) on 2 GPUs for 6 hours = **12 GPU-h** of genuine hardware fault. Credit `12 × $4.66 = $55.92`. The remaining **1,088 GPU-h** show a healthy device with a pod in `CrashLoopBackOff` — a container image with a CUDA-runtime mismatch, visible in the tenant's own pod logs. The device was held, healthy, and available to their workload; the workload could not use it. That is bucket A. No credit, and the response should say plainly which log line settles it.
4. **Rejected, with the correct counter-argument.** "Nobody was waiting" is an argument about *this* month's realised demand, and it does not generalise: the platform cannot know at allocation time whether a hold will turn out to be blocking, so a rule conditional on hindsight is not implementable. More decisively, the fixed cost was incurred regardless — headroom does not make a GPU-hour free, it makes it *unrecovered*, and the statement already shows the platform eating $8,501.30 of exactly that. The constructive answer is the tier: on best-effort, the same 5,000 allocated hours would have cost `5,000 × 4.66 × 0.60 = $13,980.00` — **$9,320.00 less** — and the workload profile (exploratory, restartable, 30% duty cycle) is a textbook fit.

**T+48h — the remedy.**

```
  CREDIT MEMO — posted to the CURRENT period, referencing 2026-07
  ChargeCategory = Adjustment
  ChargeClass    = Correction
  ────────────────────────────────────────────────────────────────────
   platform-forced hold (cordon, 4 nodes, 6h11m)   412 GPU-h  −$1,919.92
   hardware fault on held device (Xid 79, 2 GPUs)   12 GPU-h  −$   55.92
  ────────────────────────────────────────────────────────────────────
   TOTAL CREDIT                                     424 GPU-h  −$1,975.84
   exploratory, effective July charge:  23,300.00 − 1,975.84 = $21,324.16
```

The July statement itself is **not** rewritten (§5). And two follow-ups leave the dispute with more value than it cost: a **process change** — cordon-driven holds are now auto-detected and credited before the statement goes out, since if it happened to one team it happened to others silently — and a **tier migration**, with `exploratory` moved to `cq-besteffort` for August.

## Practice

Feeds [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md). Build the statement generator and the policy defence.

1. **The generator.** Inputs: per-team allocated and utilised GPU-hours by regime, each team's tier, the fleet's fixed cost components, planned chargeable hours, and the capacity accounting (calendar / unavailable / fragmented / headroom). Outputs: one statement per team carrying every field in the §5 line-item table, plus a fleet summary carrying the remainder waterfall and the three-term variance decomposition.
2. **Close the identity, in code.** Assert `Σ tenant charges + Σ remainder disposition + Σ variance == fleet cost` to the cent, and make it a test that fails the build. Assert identity A on hours separately. A generator that cannot fail is not a generator, it is a spreadsheet.
3. **Derive your own tier multipliers.** Compute the best-effort floor from your own `λ`, `Δ`, `δ`, and the ceiling from your own avoided headroom. Publish the band and where you sat in it. Do the same for the priority premium: name the GPUs held out of the pool by `lendingLimit: 0` and show that the premium recovers their cost.
4. **Model the death spiral.** Run four periods of full-absorption rating against a demand curve that responds to price, and plot `r` against allocated hours. Then re-run with a floored denominator. This is a 30-line script and it is the most convincing single exhibit you can put in front of a finance partner.
5. **Write the dispute response.** Take the §Worked-example dispute, and write the actual T+24h reply: four numbered points, hour counts, evidence keys, the credit memo, and the tier recommendation. Half a page. This is the artifact that proves you have run a chargeback programme rather than read about one.
6. **The defence memo.** One page for a sceptical finance partner: why a fixed transfer price with variance analysis beats full absorption, why the utilisation number is published but not billed, and who eats the remainder. Frame it in standard-cost-versus-actual-cost language.

**Acceptance criteria:** statements for ≥3 teams reconciling to the cent · both ledgers on every statement with the billing basis stated explicitly · the remainder itemised into named buckets with owners · the variance decomposed into three terms that sum exactly · at least one tier multiplier derived from a stated band rather than asserted.

## Common pitfalls

- **Charging utilised GPU-hours.** Symptom: fleet allocation climbs quarter over quarter while utilisation falls, and the platform keeps buying hardware to serve flat real work. Mechanism: under utilised-billing `∂cost/∂h = 0`, so holding extra capacity is free to the tenant and their optimum is to hold everything quota allows (lesson 02 §6). Correction: charge allocated, report utilised, back it with idle reclaim.
- **A pure floating full-absorption rate.** Symptom: the internal rate ratchets up every period and teams start describing the platform as "getting more expensive." Mechanism: `r = C/x` with fixed `C`; each tenant's cut shrinks `x` and raises everyone's price including their own next period, and the hyperbola steepens as `x` falls, so it accelerates. Correction: fixed published rate, with the realised absorption rate reported alongside as a signal.
- **A single "idle" bucket.** Symptom: a tenant disproves your remainder number by finding one hour in it that was a node failure. Mechanism: the bucket mixes causes with different owners — hardware failure, packing, and deliberate policy — so a counterexample from any one of them discredits the whole figure. Correction: the six-source waterfall, each with an owner and a remedy.
- **Silently redistributing the remainder into the rate.** Symptom: teams compute `charge / allocated_hours` and get a number that is not the published rate, then ask why. Mechanism: the surcharge is invisible in the rate but visible in the division, and being caught doing arithmetic you did not disclose costs more credibility than the surcharge was worth. Correction: if you redistribute, do it as a **named line item** on the statement.
- **Retro-applying a rate change to a closed period.** Symptom: a team's approved budget is blown by a number that did not exist when they planned. Mechanism: it converts a price into a random variable, which destroys the planning value the fixed rate existed to provide. Correction: reset the rate forward-looking only; land everything else as `ChargeClass = Correction` in the current period.
- **Pricing a tier that the scheduler does not implement.** Symptom: priority-tier teams pay a premium and still wait. Mechanism: Kueue's `reclaimWithinCohort` and `withinClusterQueue` both default to `Never`, so a queue with no explicit preemption configuration cannot reclaim anything — the premium bought a label. Correction: every tier must name the field it changes, and you should be able to demonstrate a preemption in a test cluster before charging for one.
- **Attributing on labels read at report time.** Symptom: month-end namespace churn that always moves cost off the busiest team. Mechanism: any metadata queried after consumption can be edited between consumption and report. Correction: snapshot ownership at device-bind time and key the ledger on the snapshot.
- **Conceding bucket A once.** Symptom: within one period, every team disputes on utilisation grounds. Mechanism: a concession on policy generalises instantly and is discovered immediately, because tenants talk to each other far faster than a chargeback programme can re-issue its policy. Correction: concede B, C and D fast and generously; never negotiate A.

## Self-check

- **A team argues "we used 40% of our reserved GPUs, so we should pay 40%." Answer them, and say what you offer instead.** *Answer:* You charge for the hold, not the use, for two independent reasons. Incentive: under utilised-billing `∂cost/∂h = 0`, so reserving more is free and the tenant's optimum is to hold everything quota allows — the exact behaviour causing the shortage they are complaining about. Structural: the allocated ledger comes from rungs 1–3 of the attribution ladder (bill line, arithmetic division, kubelet binding), all recorded facts that replay under audit, while the utilised number comes from rung 4 and is, under time-slicing, an estimate with unbounded per-tenant error — money does not move on estimates. What you offer instead: the utilisation gap published prominently as a waste ledger, an easy and fast release path, idle reclaim so the capacity actually returns to the pool, and a best-effort tier at 40% off that fits their duty cycle. On the worked fleet that tier would have cut their bill from $23,300 to $13,980.

- **Fleet utilisation drops from 85% to 50%. What happens to the internal rate under each of the three rate policies, and which do you run?** *Answer:* Under **floating full-absorption**, allocated hours fall from 39,700 to ~23,360 and the rate rises from $4.81 to `191,150/23,360 = $8.18/GPU-h` — a 70% price increase caused entirely by other people's behaviour, which triggers further cuts, because `r = C/x` is a hyperbola whose slope steepens as `x` falls. Under a **fixed transfer price** the rate stays $4.66 and the platform absorbs a much larger volume variance — `(41,000 − 23,360) × 4.66 = $82,202` — which is now well outside any tolerance band and forces an explicit conversation about fleet sizing, which is the correct conversation. Under a **two-part tariff** the capacity fees are unaffected (teams pre-committed to their slices) and only the overage component falls, so recovery degrades least. Run the fixed price, publish the realised absorption rate ($8.18) beside it as a signal with no billing weight, and treat a sustained variance as a trigger to resize the fleet or reset the rate forward — never to retro-surcharge.

- **Your statement bills $182,648.70 against a fleet cost of $191,150.00. Finance asks where the $8,501.30 went. Answer in three terms.** *Answer:* Volume variance `(41,000 − 39,700) × $4.66 = $6,058.00` — allocation ran 3.2% below the plan the rate was set on; owner is the platform's sizing plus the demand forecast. Mix variance `$13,420.80 − $11,067.50 = $2,353.30` — more best-effort discount than the tier plan assumed, net of priority premium; owner is tier pricing, and it is a *good* variance because it bought backfill and reclaimability. Rate-rounding variance `41,000 × ($4.66219512… − $4.66) = $90.00` — you published two decimal places. They sum to $8,501.30 exactly. The platform absorbs it because 4.4% is inside the ±5% band; two consecutive periods outside the band trigger a forward-looking rate reset, not a retro-surcharge.

- **Justify a 40% best-effort discount with numbers, from both directions.** *Answer:* Floor — what the tenant needs to be whole. Goodput under preemption is `1 − λ(Δ/2 + δ)`; with λ = 0.05 preemptions per GPU-hour held, Δ = 0.5 h checkpoint interval and δ = 0.1 h restart, goodput = `1 − 0.05 × 0.35 = 0.9825`, so 1.75% covers the lost work; add ~3% amortised engineering cost of making the workload genuinely restartable and the floor is about 5%. Ceiling — what the option is worth to the platform. 7,200 reclaimable GPU-hours let the fleet carry ~3,600 fewer GPU-hours of standing headroom, worth `3,600 × $4.66 = $16,776` against a full-rate best-effort bill of `7,200 × $4.66 = $33,552`, so a discount up to 50% is still value-positive. Choose 40% inside `[5%, 50%]`: well above the floor so teams genuinely opt in rather than faking urgency, below the ceiling so the platform retains surplus to fund under-recovery. Substitute your own λ, Δ, δ and headroom to move the band.

- **Name the four sorting buckets in a dispute and say which one you never negotiate, and why.** *Answer:* (A) tenant held it and didn't use it — **no credit, never negotiated**; (B) tenant didn't hold it — credit, plus a P1 exporter bug and a fleet-wide re-check because the same defect hit others silently; (C) platform forced the hold, e.g. a cordon or failed drain — credit; (D) hardware fault on a held device, e.g. an Xid quarantine — credit for the fault window only. A is never negotiated because it is decided by *policy* rather than evidence, and a policy concession generalises instantly: every team learns it by the next period and allocation billing collapses. Conceding B, C and D quickly and generously is what buys the credibility to hold A. Remedies always land as a negative row in the current period tagged `ChargeCategory = Adjustment`, `ChargeClass = Correction`; the closed period's statement is never rewritten.

- **Which Kueue fields implement a priority tier, and what exactly does the premium pay for?** *Answer:* `nominalQuota` sets the tier's reserved floor; `lendingLimit: 0` makes that whole floor non-lendable, since a ClusterQueue reserves for exclusive use `nominalQuota − lendingLimit`; `borrowingLimit` caps how far it can exceed the floor; `preemption.reclaimWithinCohort: Any` lets it preempt any cohort workload running above that workload's nominal quota, irrespective of priority; `preemption.borrowWithinCohort.policy` governs preemption while borrowing. Note the defaults: both `reclaimWithinCohort` and `withinClusterQueue` default to `Never`, so an unconfigured cluster cannot preempt at all and a priority tier there is a label with a surcharge. The premium pays for the idle GPU-hours held out of the shared pool by `lendingLimit: 0` — that is the real, nameable resource being purchased, and if you cannot name it the premium is a tax rather than a price.

- **Why is "nobody else was waiting this month" not a valid reason to waive an allocation charge?** *Answer:* Three reasons, in ascending order of force. It is hindsight — the platform cannot know at allocation time whether a hold will turn out to be blocking, so a rule conditional on realised contention is not implementable at the moment the charge accrues. It is unstable — the same held hours would be chargeable or free depending on other teams' behaviour, which makes a tenant's own bill unpredictable from their own actions, destroying exactly the planning value the fixed rate provides. And it is factually wrong about the cost: the fleet's cost is fixed, so headroom does not make a GPU-hour free, it makes it *unrecovered* — and on the worked fleet the statement already shows the platform absorbing $8,501.30 of precisely that, publicly, rather than passing it to tenants.

## Connections & what's next

This lesson is the landing point for lessons 01–07. Attribution (01) supplies the allocated share and the trust boundary; the two ledgers (02) supply the charging rule this lesson operationalises rather than re-proves; idle detection (03) is the enforcement backstop that makes allocation billing non-gameable; fragmentation (04) is a named remainder bucket with an owner and a remedy; unit economics (05) supplies the $/token and $/run lines that make a statement legible to a non-engineer; commitment strategy (06) and the normalised fully-loaded rate (07) supply the fixed cost `C` that the rate policy has to recover.

Next, **lesson 09 — Where existing tooling fails: reading the OpenCost source** — is the deliberate contrast. This lesson specified what a chargeback system on a shared GPU fleet must do: bill from the recorded side of the trust boundary, itemise the remainder, decompose the variance, and price the queue on real scheduler primitives. Lesson 09 opens the leading open-source cost tool's source and shows, function by function, how much of that it actually implements — where its GPU numerator comes from, where the DCGM utilisation it queries actually lands, and what its own issue tracker says about the gap. The statement you just designed is the specification; lesson 09 is the gap analysis against the incumbent, and lesson 10 is the schema that closes it.

## References & further reading

**Primary sources**

- **FOCUS Specification — `DataGeneratorCalculatedSplitCostAllocationHandling` attribute** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/specification/attributes/data_generator_calculated_split_cost_allocation_handling.md> — the three normative requirements quoted in §4 and the Real-world section: dimensions match the origin charge, non-summable metrics match the origin charge, and summable metrics (costs and quantities) must **sum** across allocated charges to the origin charge. Introduced in FOCUS 1.3. *Read directly from the spec repository at `main`, commit `7f19ccb`, on 2026-08-17; `focus.finops.org` itself is egress-blocked from this environment.*
- **FOCUS Specification — `AllocatedResourceId` column** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/specification/datasets/cost_and_usage/columns/allocatedresourceid.md> — the source for §4's key rule: `AllocatedResourceId` **MUST be null when a charge represents the unallocated portion of the origin charge after split cost allocation**, and must not be null on an allocated portion. Type String, Dimension, Conditional feature level, nullable, introduced 1.3. This is the standard's model of the unallocatable remainder as a first-class row.
- **FOCUS Specification — "Data Generator-Calculated Split Cost Allocation" supported feature** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/specification/supported_features/data_generator_calculated_split_cost_allocation.md> — scoped by the spec to "resources supporting shared usage like compute nodes in a shared cluster (Kubernetes, databases)". Source of the remainder query reproduced verbatim in §4 (`AllocatedMethodId IS NOT NULL AND AllocatedResourceId IS NULL`) and of the per-consumer aggregation queries a statement generator needs.
- **FOCUS Specification — split-cost-allocation worked example** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/specification/appendix/split_cost_allocation_examples.md> — the $100.00 / 40-35-25 vCPU-hour example reproduced in Real-world use cases, including the origin row zeroed to $0.00 and the explicit statement that the spec does not prescribe whether the origin row is zeroed, omitted, or offset.
- **FOCUS Specification — `ChargeClass` and `ChargeCategory` columns** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/tree/main/specification/datasets/cost_and_usage/columns> — `ChargeClass` has the single allowed value `Correction` ("correction to a previously closed billing period"), Mandatory, nullable; `ChargeCategory` allows `Usage` / `Purchase` / `Tax` / `Credit` / `Adjustment`, Mandatory, non-nullable. These are the two columns §5's restatement rule and §8's credit memo are written against.
- **Kueue `ClusterQueue` API types** — <https://github.com/kubernetes-sigs/kueue/blob/main/apis/kueue/v1beta2/clusterqueue_types.go> (and the `v1beta1` file, now marked deprecated in favour of `v1beta2`) — source for §6's quota and preemption semantics: `nominalQuota`; `borrowingLimit` ("workloads can consume nominalQuota+borrowingLimit"); `lendingLimit` ("reserves for its exclusive use nominalQuota − lendingLimit"); `reclaimWithinCohort` ∈ {`Never` default, `LowerPriority`, `Any`}; `withinClusterQueue` ∈ {`Never` default, `LowerPriority`, `LowerOrNewerEqualPriority`}; `borrowWithinCohort.policy` ∈ {`Never`, `LowerPriority`} with `maxPriorityThreshold`; and the CEL rule rejecting `reclaimWithinCohort: Never` with a non-`Never` `borrowWithinCohort`. *Read from the repository at `main` on 2026-08-17; `kueue.sigs.k8s.io` is egress-blocked from this environment, so the rendered docs were not consulted — the Go types are the normative source anyway.*
- **Kueue fair-sharing API types** — <https://github.com/kubernetes-sigs/kueue/blob/main/apis/kueue/v1beta2/fairsharing_types.go> — `FairSharing.weight` (default 1; zero weight ⇒ infinite share ⇒ always at a disadvantage; when non-zero must exceed 10⁻⁹), `FairSharingStatus.weightedShare` (max over resources of usage-above-nominal-quota divided by cohort lendable, divided by weight; returns 9223372036854775807 for a zero-weight borrowing node), and `AdmissionScope.admissionMode` ∈ {`UsageBasedAdmissionFairSharing`, `NoAdmissionFairSharing`}.
- **Kueue `AdmissionFairSharing` configuration** — <https://github.com/kubernetes-sigs/kueue/blob/main/apis/config/v1beta2/configuration_types.go> — `usageHalfLifeTime` (usage decays by half after this duration; 0 resets usage immediately), `usageSamplingInterval` (default 5m), `resourceWeights` (default 1), plus the fair-sharing `preemptionStrategies` list (`LessThanOrEqualToFinalShare`, `LessThanInitialShare`, or both in that order — any other combination fails validation). The mechanism behind the "the scheduler remembers" counter to exploit #2 in §7.

**Real-world engineering blogs**

- **CoreWeave, "Kueue: A Kubernetes-Native System for AI Training Workloads"** — <https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads> — what it shows: a neocloud running the exact cohort/quota/preemption pattern this lesson prices, in production, for AI training — including the all-or-nothing gang-scheduling semantics that make partial GPU admission useless for a training job. *`coreweave.com` is egress-blocked from this environment; cited as the canonical URL and carried forward from the previous version of this lesson rather than re-read in this pass.*
- **NVIDIA developer blog, "Ensuring Balanced GPU Allocation in Kubernetes Clusters with Time-Based Fairshare"** — <https://developer.nvidia.com/blog/ensuring-balanced-gpu-allocation-in-kubernetes-clusters-with-time-based-fairshare/> — what it shows: a second production scheduler (NVIDIA Run:ai / KAI) implementing the same historical-usage idea that Kueue's `usageHalfLifeTime` implements, i.e. a team that recently consumed over-quota capacity is admitted later. Useful as evidence that decayed-usage fairness is a convergent design rather than one project's choice. *`developer.nvidia.com` is egress-blocked here; cited as reported, not re-read in this pass — the Kueue API types above are the claim this lesson actually rests on.*

**Deeper dives**

- **FinOps Foundation, Invoicing & Chargeback capability** — <https://www.finops.org/framework/capabilities/invoicing-chargeback/> — the general framework this lesson specialises to a shared GPU fleet. *`finops.org` is egress-blocked from this environment and was not read in this pass; no claim in this lesson depends on it. Where this lesson needed normative cost-allocation language it quotes the FOCUS specification instead, which is the same foundation's machine-readable output and is readable from the spec repository.*
- **Kubernetes pod priority and preemption** — <https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/> — the primitive underneath Kueue's preemption policies and the thing that actually evicts a best-effort pod. Read it if you intend to price a preemptible tier, because the eviction semantics (graceful termination period, PDB interaction) determine your `δ` in the §6 discount derivation.
- **Module 04 capstone, per-pod attribution** — [04 L10 — Capstone: per-pod attribution](../../04-gpu-on-kubernetes/lessons/10-capstone-per-pod-attribution.md) — identity A and identity B, the PromQL that asserts them continuously, and the `source_snapshot_id`-style provenance that the dispute protocol in §8 consumes. This lesson's "the statement must close to the cent" is that lesson's identity A, denominated in dollars.

[💰 11 — GPU cost and unit economics](../README.md)
