---
lesson: "10.8"
title: "Capex vs cloud: the GPU crossover model — and load-balancing without a cloud LB"
module: "10"
concept: "Capex vs cloud: the GPU crossover model — and load-balancing without a cloud LB"
status: not-started
est_time: "10h"
prev: "07-storage-for-ai.md"
next: null
artifacts: []
sources: 14
---

# 10.8 · Capex vs cloud: the GPU crossover model — and load-balancing without a cloud LB

> **Concept.** Build the owned-vs-rented $/GPU-hr crossover model that *is* the neocloud business thesis — capex, power×PUE, colo, InfiniBand, depreciation, staff — find break-even utilisation and payback month; then, as the on-prem corollary, serve LoadBalancer/ingress and the API VIP with MetalLB (BGP) and kube-vip, because there is no cloud LB to call.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Where this fits

Every lesson before this one built or protected a piece of the fleet: a hand-built control
plane (01), an etcd that survives a bad day (02), HA that survives a node loss (03),
declarative fleets that provision without hand-holding (04), a PXE pipeline that turns bare
metal into a `Ready` node in minutes rather than weeks (05), a health loop that catches
broken hardware in minutes instead of a month (06), and a storage tier sized so healthy
GPUs are not idling on I/O (07).

This lesson does two things with all of that. First, it turns every one of those
operational decisions into a single number — **owned $/GPU-hr** — and asks the question a
CFO actually asks: does owning this fleet beat renting it, and at what utilisation? Second,
it closes the last purely infrastructural gap: on a managed cloud, `Service type:
LoadBalancer` is an API call; on bare metal *you* are the cloud LB controller, and this is
where you build that too.

This is the module's **capstone**. The deliverable and the checkpoint are built around it.

## Why this matters

A neocloud is, financially, one arbitrage: buy GPUs at capex, rent them at $/GPU-hr, and
pocket the spread *if* utilisation stays high enough to beat the depreciation clock. If you
can build the model that says at what utilisation owning beats renting, and which
assumption moves the answer, you can reason about that business — and about whether your
own fleet should have been bought or rented. Almost nobody writes this rigorously. Hiring
managers at GPU-heavy shops notice a candidate who can.

The stakes are also just arithmetic. A 64-GPU fleet is a ~$2.6M capital commitment against
a rental alternative with no commitment at all. Get the break-even utilisation wrong by ten
points and you have made a seven-figure decision on a number you did not check. And the
error is systematically in one direction: the terms people forget — storage, spares,
support contracts, the fraction of an engineer, the facility charge — are all *costs of
owning*, so an incomplete model always flatters ownership.

The second half is the unglamorous reality that pairs with it. Once you own the metal there
is no ELB. A `Service type: LoadBalancer` sits `<pending>` forever unless you supply the
load balancer, and a multi-apiserver control plane has no stable address unless you supply
that too. MetalLB and kube-vip are how bare-metal clusters get those, and they are the
things managed clouds did for you and never itemised — which is exactly why they belong in
a lesson about what owning actually costs.

## What's new here (calibration)

No FinOps basics are re-taught. What is new:

- **A TCO model with every term named and derived**, not a list of plausible line items:
  hardware amortisation, financing, power at a stated PUE, facility, network, support,
  staff, storage, spares — each traced back to the lesson that produced it.
- **The distinction between nameplate power and measured power**, which decides two
  different things (circuit provisioning versus the energy bill) and is the classic place
  a model double-counts.
- **Break-even utilisation derived algebraically** rather than eyeballed off a chart, with
  the crossover computed against four different rental bases — and the discipline that a
  crossover number is meaningless without naming which base it crossed.
- **The payback month on a cash basis**, and the proof that "break-even utilisation" and
  "payback lands at end of life" are the same statement seen from two directions.
- **A sensitivity analysis that names the terms that do** *not* **matter**, which is harder
  and more useful than naming the ones that do. PUE and electricity price are, at US
  industrial rates, close to noise; two other terms dominate.
- **The economy-of-scale curve** — the same model run at 8, 64 and 512 GPUs — which turns
  out to be the quantified form of the neocloud thesis.
- **MetalLB in L2 vs BGP mode** with its current backend (which changed), and **kube-vip**
  with its real failover-timing defaults.

## Core concepts — the capex-vs-cloud model

> **Every dollar figure below is an August 2026 snapshot and several move meaningfully
> within a single year.** They are default *inputs*, not truths. The deliverable's entire
> point is that you replace them with your own and watch the crossover move.

### 1. The one equation, and why utilisation is in the denominator

```
                          monthly_fixed  +  monthly_opex
  owned $/GPU-hr(U)  =  ─────────────────────────────────────
                          N_gpu  ×  730 hr/month  ×  U
```

`730` is a month's hours (8,760/12). `U` is the fraction of those hours a GPU spends doing
work you value — billable hours at a provider, or genuinely-useful compute in-house.

Everything hard is in populating the numerator honestly, and in the fact that **utilisation
is in the denominator**: halve it and $/GPU-hr doubles, exactly. There is no other input in
the model with that leverage, which is why every operational lesson in this module — faster
provisioning (05), faster fault detection (06), storage that does not stall (07) — is,
financially, the same lever.

### 2. Every term, named and sourced

The parameter set. Cross-references point at the lesson that produced the number.

```
  FLEET
    n        nodes                                        =  8
    g        GPUs per node                                =  8
    N        GPUs = n × g                                 = 64

  CAPEX  (one-time, at t = 0)
    K_node   8×H100 SXM HGX node, all-in                  = $280,000   [200k – 400k]
    K_fab    rail-optimised fabric, per GPU               = $3,000     [2,000 – 4,500]
             (NIC + the node's share of leaf/spine switches, optics, cabling)
    K_spare  spares held on the shelf                     = 1 × $180,000   ← lesson 06 §10
             (a 64-GPU fleet at AFR 3%, 21-day lead time, 95% service level needs s = 1)
    S        residual / salvage fraction at end of life   = 0.10       [0 – 0.30]

  FINANCE
    L        depreciation life, years (straight line)     = 5          [3 / 4 / 5 / 6]
    i        blended cost of capital                      = 12%/yr     [6% – 18%]
             (GPU-backed debt is expensive; corporate cash is not. State which.)

  POWER — two different numbers, for two different purposes
    P_name   NAMEPLATE node power                         = 10.2 kW    ← lesson 02b.7
             (DGX H100 datasheet maximum: six 3,300 W Titanium PSUs, 4+2 redundant)
             USE FOR: circuit and rack provisioning, and the colo's billed kW.
    P_meas   MEASURED sustained node power under load     = 7.3 kW     [6.0 – 8.5]
             (empirical H100-node measurement: ~8.4 kW peak, ~7.3 kW typical)
             USE FOR: the energy bill. Nothing else.
    PUE      facility power usage effectiveness           = 1.30       [1.15 – 1.54]
             (Uptime Institute's 2025 global survey average was 1.54; a purpose-built
              modern AI facility is well below it; a colo contract states its own.)
    e        electricity                                  = $0.085/kWh [0.04 – 0.30]
             (US industrial average ran ~8.5–8.6 ¢/kWh in 2025–26; European and some
              Asian industrial rates are 2–4× higher.)

  FACILITY
    colo     $ per provisioned kW per month               = $200/kW/mo [150 – 250]
             (covers space, cooling plant, power distribution, remote hands.
              Q1-2026 US market: Chicago ~$200–230, Northern Virginia ~$190–235
              for 250–500 kW requirements.)

  OPERATIONS
    maint    vendor support + break/fix, % of hw capex/yr =  7%        [5% – 10%]
    FTE      fully-loaded platform engineer               = $275,000/yr
    f        FTE fraction attributable to this fleet      = 0.5        [0.25 – 1.5]
    stor     storage opex (parallel FS + object)          = $18,000/mo ← lesson 07 §6
    net_op   IP transit, cross-connects, DDoS             = $2,000/mo

  DEMAND
    U        sustained utilisation                        = the free variable
    R        the rental rate you are comparing against    = see §5 — this is NOT one
                                                            number and treating it as
                                                            one is the top error
```

Three modelling decisions inside that list are load-bearing, and getting any of them wrong
is worth more than getting a price quote wrong.

**Nameplate versus measured power.** These are not two estimates of the same quantity; they
answer different questions. A circuit must carry the worst case the hardware can draw, so
rack and PDU provisioning uses **nameplate** — lesson 02b.7 works this through: four
10.2 kW nodes per rack against three 415 V / 32 A three-phase circuits at 21.8 kW each,
N+1, which survives losing one circuit at 94% loaded and does *not* survive a fifth node.
The energy bill, by contrast, is an integral of actual draw, and actual sustained draw on
an 8×H100 node under real training load measures around 7.3 kW typical against an 8.4 kW
peak. **Billing energy at nameplate overstates the power line by roughly 40%.** In this
model that turns out to matter far less than you would expect (§7), but the *reason* to get
it right is that it also tells you when the colo's provisioned-kW charge — which is billed
on nameplate — is your real power cost rather than the meter.

**Do not apply PUE twice.** In the common colo structure you pay `$/kW/month` on
*provisioned critical (IT) power* — that charge is what buys you the cooling and the
distribution, i.e. it is the facility's PUE overhead sold to you as a fixed fee — and you
pay a metered energy bill separately. Whether that metered bill is IT-only or
facility-total varies by contract. This model assumes the metered bill is facility-total
(hence `× PUE`) and the `$/kW` charge covers space and plant, and **says so**, because the
alternative structures differ by 30% on a line that is small anyway. The rule: write down
which structure your contract uses before you compute anything.

**Spares are capital, and they are lesson 06's output.** Lesson 06's Poisson model gives
the spares count for a service level; at 64 GPUs, AFR 3% and a 21-day lead time,
`λ = 64 × 0.03 × 21/365 = 0.110`, and `P(demand ≤ 1) = 0.994`, so `s = 1`. That is $180,000
of idle capital in a $2.6M fleet. §8 asks whether it is worth it, and the answer is not the
one lesson 06 implied.

### 3. The base case, computed

```
  CAPEX
    compute   8 × $280,000                                 = $2,240,000
    fabric   64 × $3,000                                   = $  192,000
    spares    1 × $180,000                                 = $  180,000
    ────────────────────────────────────────────────────────────────────
    K_total                                                = $2,612,000

  MONTHLY FIXED
    depreciation   K_total × (1 − S) / (L × 12)
                   2,612,000 × 0.90 / 60                   = $ 39,180
    financing      i × K_total × 0.55 / 12
                   0.12 × 2,612,000 × 0.55 / 12            = $ 14,366
                   (0.55 ≈ the average outstanding balance over an amortising term;
                    set it to 1.0 for an interest-only facility, 0 for cash purchase)
    ────────────────────────────────────────────────────────────────────
    monthly_fixed                                          = $ 53,546

  MONTHLY OPEX
    energy         n × P_meas × 730 × PUE × e
                   8 × 7.3 × 730 × 1.30 × 0.085            = $  4,711
    facility       n × P_name × colo
                   8 × 10.2 × 200                          = $ 16,320
    support        (K_node·n + K_fab·N) × maint / 12
                   2,432,000 × 0.07 / 12                   = $ 14,187
    staff          f × FTE / 12
                   0.5 × 275,000 / 12                      = $ 11,458
    storage        from lesson 07                          = $ 18,000
    network                                                = $  2,000
    ────────────────────────────────────────────────────────────────────
    monthly_opex                                           = $ 66,676

  TOTAL MONTHLY COST OF OWNERSHIP                          = $120,222

  owned $/GPU-hr(U) = 120,222 / (64 × 730 × U) = 120,222 / (46,720 · U)
                    = $2.573 / U
```

| Utilisation `U` | owned $/GPU-hr |
|---|---|
| 100% | **$2.57** |
| 90% | $2.86 |
| 80% | $3.22 |
| 70% | $3.68 |
| 60% | $4.29 |
| 50% | $5.15 |
| 40% | $6.43 |
| 30% | $8.58 |

### 4. Consistency check: the same model, a provider's parameters

A model you cannot reconcile against someone else's is a model nobody will trust. Lesson
11.7 builds a *provider's* cost-to-serve from the same structure. Run this model with
**11.7's parameters** — `K = $280,000`, `S = 0.10`, `L = 6`, `i = 12%` on 55% average
outstanding, `P = 10.2 kW`, `PUE = 1.30`, `e = $0.08/kWh`, fabric `$3,000/GPU`, and a single
combined "facility + staff + storage + ops" term `F = $1,800/node-month`, with fleet
availability `A = 0.95` and revenue utilisation `U = 0.90`:

```
    depreciation   280,000 × 0.90 / (6 × 8,760)            = $ 4.794 /node-hr
    financing      0.12 × 280,000 × 0.55 / 8,760           = $ 2.110
    power+cooling  10.2 × 1.30 × 0.08                      = $ 1.061
    fabric amort   8 × 3,000 / 52,560                      = $ 0.457
    facility+ops   1,800 / 730                             = $ 2.466
    ──────────────────────────────────────────────────────────────────
    cost to serve, per node-hour of CALENDAR                = $10.888
    ÷ 8 GPUs                                                = $ 1.361 /GPU-hr calendar
    ÷ (A × U) = ÷ 0.855                                     = $ 1.592 /GPU-hr SOLD
```

That reproduces lesson 11.7's figure to the cent. **The two lessons are the same equation.**

So why does this lesson's base case come out at `$20.59/node-hr` (=$120,222 ÷ 8 ÷ 730)
against 11.7's `$10.888` — 1.9× higher for what is nominally the same hardware? Decompose
it, because the answer is the most important finding in this lesson:

| Line | This lesson, per node-hr | 11.7, per node-hr | Difference and why |
|---|---|---|---|
| Depreciation | $6.71 | $4.794 | 5-year life instead of 6, plus the spare in the capital base |
| Financing | $2.46 | $2.110 | larger capital base |
| Fabric amortisation | $0.49 | $0.457 | same |
| Energy | $0.81 | $1.061 | measured 7.3 kW rather than nameplate 10.2 kW, at a higher $/kWh |
| Facility (colo $/kW) | $2.79 | — | } |
| Support / maintenance | $2.43 | — | } all five of these are inside 11.7's |
| Staff | $1.96 | — | } single `F = $1,800/node-month` = |
| Storage | $3.08 | — | } **$2.466/node-hr** |
| Network | $0.34 | — | } |
| **Sum of the five** | **$10.60** | **$2.466** | **4.3× — this is the entire gap** |

**The hardware costs the same. What differs by 4.3× is fixed operating expense divided by
fleet size.** A provider's storage engineer, facility contract and support agreement
amortise across thousands of nodes; an eight-node fleet carries a whole $18,000/month
storage tier and half an engineer on its own. Lesson 11.7's `F` is not wrong — it is a
provider's number, correctly labelled as such — and this lesson's is not wrong either. They
disagree because **fleet size is a cost input**, which §9 quantifies.

Hold onto that. It is why neoclouds exist, and it is a far better answer to "why is
renting sometimes cheaper than owning identical hardware" than anything about margins.

### 5. Break-even utilisation, derived

Set owned cost equal to the rental rate `R` and solve. This is one line of algebra and it
is the deliverable's headline:

```
      monthly_total
  ─────────────────────  =  R          ⟹     U*  =  ───────────────────────
    N × 730 × U*                                        monthly_total
                                                     ───────────────────────
                                                        N × 730 × R

  With monthly_total = $120,222, N = 64:

      U*(R)  =  120,222 / (46,720 × R)  =  2.573 / R
```

That is the whole model. `U*` is inversely proportional to the comparator rate — which
means **the single largest determinant of the answer is a number you do not control and
must not fudge: which rental rate you are comparing against.**

Using lesson 11.7's observed August-2026 price surface for H100 80GB:

| Comparator `R` | Basis (tier · term · bundle) | `U* = 2.573/R` | Verdict |
|---|---|---|---|
| **$7.89** | hyperscaler on-demand, cohort median | **32.6%** | owning wins easily |
| **$4.90** | hyperscaler, 1-year committed (illustrative shape) | **52.5%** | owning wins at moderate utilisation |
| **$4.17** | dedicated-cloud on-demand, cohort median | **61.7%** | owning wins above ~62% |
| **$3.15** | whole-cohort median, all bases mixed | 81.7% | owning wins only near saturation |
| **$2.10** | tier-1 neocloud, 1-year reserved | 122.5% | **owning never wins** |
| **$1.45** | Bronze-tier, 1-year committed | 177.5% | **owning never wins** |

**A crossover number without its comparator's tier, term and bundle is not a result.**
"Break-even is 62%" and "owning never breaks even" are both true statements about this
identical fleet; they differ only in which row you compared against. Lesson 11.7 makes the
same point about vendor comparisons; here it lands on your own build-versus-buy memo.

### 6. The crossover chart

```
            owned $/GPU-hr  =  $120,222 / (64 x 730 x U)

  9.0 |
  8.5 |        *
  8.0 +--------------------------------------------------- <- hyperscaler ON-DEMAND, median   $7.89   U* = 32.6%
  7.5 |           *
  7.0 |
  6.5 |              *
  6.0 |
  5.5 |                 *
  5.0 +--------------------*------------------------------ <- hyperscaler 1-yr COMMITTED      $4.90   U* = 52.5%
  4.5 |                       *  *
  4.0 +-----------------------------*--------------------- <- dedicated-cloud ON-DEMAND, med  $4.17   U* = 61.7%
  3.5 |                                *  *
  3.0 |                                      *  *  *
  2.5 |                                               *  *
  2.0 +--------------------------------------------------- <- neocloud 1-yr RESERVED          $2.10   never
      +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
        20    30    40    50    60    70    80    90   100
                      sustained utilisation U (%)

  READ IT AS THREE FACTS, NOT ONE CURVE:

  (1) The curve is a HYPERBOLA (cost ∝ 1/U), not a line. Between 100% and 80%
      utilisation you lose 65 cents per GPU-hour; between 40% and 20% you lose
      $6.44. The penalty for idle capacity ACCELERATES as utilisation falls, which
      is precisely the opposite of the intuition that "we'll grow into it."

  (2) The owned curve NEVER crosses the $2.10 line. No amount of utilisation makes
      this fleet cheaper than a tier-1 neocloud's committed rate, because that rate
      is at or below the provider's own cash cost to serve (lesson 11.7 puts the
      Bronze floor at ~$1.59/GPU-hr sold). You cannot out-operate someone selling
      near cost with 100× your scale.

  (3) Therefore the honest build-versus-buy question is never "own or rent." It is
      "own, or COMMIT." Owning beats on-demand at very ordinary utilisation (33–53%);
      owning beats a committed neocloud rate essentially never. What you are really
      choosing between is capital risk and contract risk.
```

### 7. Sensitivity: which terms move the answer, and which do not

Because `U*` is directly proportional to `monthly_total`, **the sensitivity of the answer to
any cost line is exactly that line's share of total cost.** That single observation makes
the whole sensitivity analysis a table you can read off:

| Cost line | $/month | Share of total | ⟹ a ±50% error in this line moves `U*` by |
|---|---|---|---|
| Depreciation | $39,180 | **32.6%** | ±16.3% relative (±10.1 points) |
| Storage opex | $18,000 | **15.0%** | ±7.5% relative (±4.6 points) |
| Facility (colo) | $16,320 | **13.6%** | ±6.8% relative (±4.2 points) |
| Support / maintenance | $14,187 | 11.8% | ±5.9% relative (±3.6 points) |
| Financing | $14,366 | 11.9% | ±6.0% relative (±3.7 points) |
| Staff | $11,458 | 9.5% | ±4.8% relative (±2.9 points) |
| **Energy** | **$4,711** | **3.9%** | **±2.0% relative (±1.2 points)** |
| Network | $2,000 | 1.7% | ±0.8% relative (±0.5 points) |

Now run the scenarios that matter, against the `$4.17` comparator (`U*` base = 61.7%):

```
  ── (a) DEPRECIATION LIFE — a policy choice, and the largest controllable term ──
     L = 3 yr : dep $65,300 → total $146,342 → $3.132/U → U* = 75.1%
     L = 4 yr : dep $48,975 → total $130,017 → $2.783/U → U* = 66.7%
     L = 5 yr : (base)                       → $2.573/U → U* = 61.7%
     L = 6 yr : dep $32,650 → total $113,692 → $2.433/U → U* = 58.4%
     ── spread 16.7 POINTS of break-even utilisation, from an accounting policy.
        CoreWeave's FY2025 10-K uses six years straight-line for technology
        equipment including GPUs; three years is the conservative posture. The
        number you pick is an assumption about obsolescence, not a fact, and a
        reviewer will challenge it first — so state the reasoning, not just the
        number.

  ── (b) RESIDUAL VALUE — smaller than its reputation ──────────────────────────
     S = 0.00 : dep $43,533 → total $124,575 → U* = 63.9%
     S = 0.10 : (base)                       → U* = 61.7%
     S = 0.30 : dep $30,473 → total $111,515 → U* = 57.2%
     ── spread 6.7 points. Assuming a 30% residual instead of zero buys you less
        than a one-year change in depreciation life. Model residual as a RISK, not
        an asset: a next-generation glut is exactly the scenario where you both
        need the residual and cannot realise it.

  ── (c) ENERGY AND PUE — much less than the literature implies ────────────────
     e = $0.04/kWh : energy $2,217  → U* = 60.4%
     e = $0.085    : (base)         → U* = 61.7%
     e = $0.20     : energy $11,085 → U* = 65.0%
     e = $0.30     : energy $16,627 → U* = 67.8%      (European industrial rates)

     PUE = 1.15 : energy $4,168 → U* = 61.4%
     PUE = 1.30 : (base)        → U* = 61.7%
     PUE = 1.54 : energy $5,581 → U* = 62.2%          (Uptime 2025 global average)

     Using NAMEPLATE 10.2 kW instead of measured 7.3 kW for the energy bill:
                  energy $6,583 → U* = 62.7%          (+1.0 point)

     ── THE CORRECTION: at US industrial electricity rates, energy is ~4% of the
        total cost of ownership for this fleet, so PUE moves break-even by under a
        point and the whole vendor-versus-measured power-draw argument is worth
        1.0 point. An earlier version of this lesson ranked $/kWh as the third most
        sensitive input; on this fully-decomposed model it is seventh of eight.
        Energy becomes material only at 3–4× US rates, where it reaches 12–13% of
        cost and 7 points of break-even.

        BUT — and this is why the power lesson still matters — PUE and node power
        are not cost constraints, they are CAPACITY constraints. A PUE of 1.54
        instead of 1.15 means 34% more facility power for the same IT load, and at
        a power-constrained site that is the limit on how many nodes you can
        install at all. Lesson 02b.7's rack arithmetic is a provisioning tool, not
        a costing tool, and conflating the two is why this term gets mis-ranked.

  ── (d) THE COMPARATOR RATE — the dominant term, and not yours to choose ──────
     R = $7.89 → U* = 32.6%          R = $3.15 → U* = 81.7%
     R = $4.90 → U* = 52.5%          R = $2.10 → never
     R = $4.17 → U* = 61.7%          R = $1.45 → never
     ── spread: from 33% to NEVER. This single input swings the answer further
        than every internal cost line combined, and it is a market price that moved
        55% in the two years to 2026 (lesson 11.7). Any model that hard-codes one
        rental number has hidden its most sensitive assumption inside a constant.
```

**The defensible one-paragraph answer**, which is what an interview is actually asking for:
*the comparator rate dominates and must be stated with tier, term, bundle and date;
depreciation life is the largest term you control and swings break-even by ~17 points across
a defensible 3–6 year range; fixed opex divided by fleet size is the third (§9), and it is
the reason a provider's cost structure is not yours. Energy and PUE are close to noise at
US industrial rates — they constrain capacity, not cost.*

### 8. The spares line, and a correction to lesson 06

Lesson 06 concluded that spares pay for themselves. Re-run it inside this model, because
the framing there was incomplete in two ways.

```
  Lesson 06's argument, at 512 GPUs, AFR 3%, 21-day lead time, s = 3 baseboards:
      holding cost   = 3 × $180,000 × 12% cost of capital        = $ 64,800/yr
      capacity saved = 15.4 events × 8 GPUs × 21 d × 24 h        = 62,092 GPU-hr/yr
                       valued at $2.50/GPU-hr                    = $155,200/yr
      → spares win by 2.4×

  TWO PROBLEMS WITH THAT:

  (1) It counts only FINANCING as the holding cost. A spare baseboard depreciates
      on the same clock as a deployed one — arguably worse, because it earns
      nothing while the technology-obsolescence clock runs.

          full holding cost = depreciation + financing
                            = 540,000 × 0.90/5  +  540,000 × 0.12 × 0.55
                            = $97,200 + $35,640                  = $132,840/yr

  (2) It values the saved capacity at $2.50/GPU-hr without saying what that price
      IS. It matters enormously which one you pick:

          at your own cost to produce ($2.21/GPU-hr at 512 GPUs, §9):
              62,092 × 2.21                                      = $137,200/yr
              → a wash. Spares neither win nor lose.

          at the rental rate you would otherwise pay ($4.17):
              62,092 × 4.17                                      = $258,900/yr
              → spares win by 1.9×.

  THE ACTUAL RULE: hold spares when the marginal GPU-hour is worth more than your
  own cost of producing it — that is, when you are CAPACITY-CONSTRAINED and the
  work you cannot do has real value. If you have slack (U well below 1), a failed
  node's work moves to an idle one and the spare buys you very little.

  At 64 GPUs the arithmetic is worse still, because s = 1 spare is 6.9% of the
  capital base for a fleet that only loses ~1.9 GPU-events per year:
      holding cost = 180,000 × 0.90/5 + 180,000 × 0.12 × 0.55     = $44,280/yr
      capacity saved = 1.92 × 8 × 21 × 24 = 7,741 GPU-hr/yr
                       at $4.17                                   = $32,280/yr
      → at 64 GPUs, the spare LOSES on these parameters.

  → The right answer for a small fleet is usually a contractual one: buy a
    guaranteed-turnaround support tier instead of holding capital. Halving the lead
    time from 21 to 10 days halves the capacity loss AND drops lesson 06's Poisson
    spares requirement, which is why lesson 06's own sensitivity analysis named
    lead time, not AFR, as the lever. This model is where that recommendation gets
    a dollar value.
```

That is what a capstone is for: it is the instrument that scores the other lessons, and it
is allowed to send one of them back for revision.

### 9. Fleet size is a cost input: the economy-of-scale curve

§4 showed a 4.3× gap in fixed opex per node between this fleet and a provider's. Quantify
it by running the identical model at three scales, letting only the things that genuinely
scale sub-linearly do so — staff (0.25 / 0.5 / 2.5 FTE), storage ($4K / $18K / $72K per
month), and spares (0 / 1 / 3 baseboards):

| | **8 GPUs** (1 node) | **64 GPUs** (8 nodes) | **512 GPUs** (64 nodes) |
|---|---|---|---|
| Capex | $304,000 | $2,612,000 | $19,996,000 |
| Depreciation /mo | $4,560 | $39,180 | $299,940 |
| Financing /mo | $1,672 | $14,366 | $109,978 |
| Energy /mo | $589 | $4,711 | $37,687 |
| Facility /mo | $2,040 | $16,320 | $130,560 |
| Support /mo | $1,773 | $14,187 | $113,493 |
| Staff /mo | $5,729 (0.25 FTE) | $11,458 (0.5 FTE) | $57,292 (2.5 FTE) |
| Storage /mo | $4,000 | $18,000 | $72,000 |
| Network /mo | $500 | $2,000 | $6,000 |
| **Total /mo** | **$20,863** | **$120,222** | **$826,950** |
| GPU-hours/mo at U=1 | 5,840 | 46,720 | 373,760 |
| **owned $/GPU-hr at U=1** | **$3.573** | **$2.573** | **$2.212** |
| **`U*` vs $4.17** | **85.7%** | **61.7%** | **53.1%** |
| Staff as share of cost | 27.5% | 9.5% | 6.9% |

```
   BREAK-EVEN UTILISATION vs FLEET SIZE   (comparator $4.17/GPU-hr on-demand)

   100% ┤
        │  *  8 GPUs — 85.7%
    90% ┤
        │
    80% ┤
        │
    70% ┤
        │        *  64 GPUs — 61.7%
    60% ┤
        │                          *  512 GPUs — 53.1%
    50% ┤                                              (asymptote: the point where
        │                                               fixed opex per node stops
    40% ┤                                               falling — around $2.10/GPU-hr
        └───┬──────────┬──────────┬──────────┬────────  at these parameters)
            8         64        512       4,096
                        fleet size (GPUs, log scale)

   The gap between 85.7% and 53.1% is not hardware. It is the same $18,000 storage
   tier and the same engineer, divided by 8 versus 512.
```

Two conclusions fall out, and both are worth more than the numbers:

1. **Small owned fleets are structurally uncompetitive**, and not because of purchasing
   power. At 8 GPUs, staff alone is 27.5% of total cost — the single largest line after
   depreciation — and no amount of operational excellence removes it. This is the same
   result lesson 11.7 reaches from the other direction with its self-SRE addon
   ($0.78/GPU-hour at 16 GPUs for 0.4 FTE), and it is why sub-32-GPU teams should almost
   always rent.
2. **This is the neocloud thesis, quantified.** A provider pools demand from many customers
   to hit a utilisation no single team can, *and* amortises fixed opex across a fleet no
   single team has. The two effects compound: they operate further right on this curve and
   further right on the hyperbola in §6. That is a structural advantage, not a margin
   story, which is exactly the distinction lesson 11.7 insists on when it separates what
   survives normalisation from what was bundling.

### 10. Payback, on a cash basis

Break-even utilisation is an accounting statement. A CFO will also ask a cash question:
when does this stop being underwater? Depreciation is not cash, so drop it and use the
actual outflows.

```
                                K_total
  payback_month(U, R)  =  ─────────────────────────────────────────
                           N × 730 × U × R   −   monthly_cash_opex

  monthly_cash_opex = monthly_opex + financing = $66,676 + $14,366 = $81,042
  K_total = $2,612,000 ;  N × 730 = 46,720 GPU-hours/month at U = 1
```

```
  payback month (cash basis) vs sustained utilisation, comparator $4.17/GPU-hr
  each # = 1 month;  |  marks the end of the 5-year (60-month) asset life

  U=1.00    #######################  23 mo
  U=0.90    ############################  28 mo
  U=0.80    ###################################  35 mo
  U=0.70    ###############################################  47 mo
  U=0.65    #########################################################  57 mo
  U=0.62    ############################################################|#####  66 mo  <- PAST end of life
  U=0.60    ############################################################|############  73 mo
  U=0.55    ############################################################|#######################>>  100 mo
  U=0.50    ############################################################|#######################>>  160 mo
            +---------+---------+---------+---------+---------+---------+---------+---------+
                     10        20        30        40        50        60        70        80
                                          months owned
```

**Notice where payback crosses the 60-month life: at `U ≈ 0.62`.** That is the same 61.7%
this lesson derived algebraically in §5, arrived at by a completely different route.

That is not a coincidence, it is a definition: break-even utilisation is *exactly* the
utilisation at which the asset finishes paying for itself on the day it finishes
depreciating. The small residual gap (66 months rather than 60) is precisely the 10%
salvage value that depreciation never wrote off plus the interest that the accounting view
spreads differently. **If your two calculations do not land in the same place, one of them
is wrong** — and this is the cheapest self-check in the whole model.

The other thing the chart shows is the shape of the risk. Between 100% and 80% utilisation
payback stretches by 12 months. Between 65% and 55% it stretches by 43. **The failure mode
of a capex decision is not a slightly worse return, it is a cliff** — and where you land on
that cliff is decided by exactly the operational things this module spent seven lessons on.

### 11. What each earlier lesson is worth, in this model

The capstone's job is to score the module. Every lesson maps to a term:

| Lesson | The term it moves | Concretely |
|---|---|---|
| **05 · PXE to Ready** | `U` at the start of life | A node un-provisioned is depreciating at `$280,000 × 0.9 / (5 × 365) = $138/day` with zero utilisation against it. Lesson 05's 17-minute automated bring-up versus days of manual work, across a 40-node delivery, is roughly `40 × 3 days × $138 = $16,560` of pure recovered depreciation per rack — before counting the revenue those nodes could have earned. |
| **06 · health and RMA** | `U`, continuously | A sick-but-schedulable node contributes ~0 useful work while accruing full cost. At 64 GPUs, one node undetected for a week is `8 × 168 = 1,344` GPU-hours of the month's 46,720 — **2.9 points of utilisation**, which at the $4.17 comparator is worth more than the entire energy line. Detection latency is a utilisation input. |
| **06 · spares** | capital, and `U` | §8: a spare is 6.9% of capex at this fleet size and is only worth holding when you are capacity-constrained. |
| **07 · storage** | a 15% cost line, and `U` | The storage tier is the third-largest line in the model. And a storage stall is a direct subtraction from `U` — Meta's 56% stall figure, read through this equation, is a ~2.3× multiplier on effective cost per useful GPU-hour. |
| **04 · declarative fleets** | `f`, the FTE fraction | Staff is 9.5% of cost here and 27.5% at 8 GPUs. CAPI/Talos automation is what keeps `f` sub-linear as the fleet grows — the difference between the 512-GPU column needing 2.5 FTE and needing 8. |
| **02 / 03 · etcd and HA** | `U`, catastrophically | A control-plane outage takes `U` to zero for the whole fleet at once. Every other lesson moves `U` by points; this one moves it to 0. |
| **02b.7 · power and thermals** | capacity, not cost | §7: PUE and node power move `U*` by under a point. What they decide is how many nodes fit on the circuits you have. |

**This is not a bolt-on economics unit. It is the instrument that scores every other lesson
in the module** — and the scoring is quantitative, in the same units, so the lessons are
comparable to each other.

## Core concepts — load-balancing without a cloud LB

### 12. The problem, stated precisely

On EKS/GKE/AKS, `Service type: LoadBalancer` triggers a cloud-controller-manager that
provisions an external load balancer and writes its address into
`status.loadBalancer.ingress`. On bare metal there is no such controller. Nothing writes
that field, so the Service's `EXTERNAL-IP` stays **`<pending>`** forever — not as an error,
just permanently unfinished.

There are two distinct jobs, and they need different tools because they have different
consistency requirements:

- **External IPs for workload Services and Ingress** → **MetalLB**. Many VIPs, allocated
  from pools, advertised to the network.
- **One stable address for the control-plane API**, so kubelets and `kubectl` can target a
  single endpoint across `N` apiservers → **kube-vip** (or keepalived + HAProxy). Exactly
  one VIP, and it has to exist *before* the cluster is fully up, which is why it runs as a
  static pod rather than a Deployment.

### 13. MetalLB: L2 versus BGP, and what each actually does

**L2 mode is failover, not load balancing.** One node is elected to answer ARP (IPv4) or
NDP (IPv6) for each service VIP; all traffic for that IP enters through that one node, and
`kube-proxy` then spreads it to pods. MetalLB's own documentation is explicit that this
"does not implement a load balancer… it implements a failover mechanism." Two limitations
follow directly from using ARP:

1. **Ingress bandwidth for a VIP is capped at one node's NIC.** This is fundamental to
   steering traffic with ARP, not an implementation gap.
2. **The VIP must be in the same L2 subnet as the nodes.** ARP does not cross a router, so
   a single L2-mode VIP cannot span racks in a routed fabric.

Failover uses `hashicorp/memberlist` to detect that a node is gone, then sends gratuitous
ARP/unsolicited NDP so clients update their neighbour caches. Modern operating systems
handle that correctly and failover completes in a few seconds; older or buggy clients can
be slower, which is why MetalLB's guidance is to keep the old leader up for a couple of
minutes after a *planned* leadership change. Leader election is stateless: every speaker
independently computes a sorted hash of `node+VIP` pairs and announces if it sorts first —
elegant, and it means removing a node does not disturb existing announcements, but it also
means a speaker with a wrong view of node liveness can produce a split brain where two
nodes (or none) announce the same VIP.

**BGP mode is real load balancing.** Each node runs a BGP speaker that peers with your
top-of-rack routers and advertises the service prefix. The routers install equal-cost
multipath routes and hash each *connection* to a next hop. Two things to understand:

- **Per-connection hashing is correct, not a limitation.** Spreading packets of one TCP
  flow across nodes would cause reordering, and on-node routing is not guaranteed identical
  across nodes, so packets of one connection could reach different pods. Routers typically
  offer 3-tuple `(protocol, src-ip, dst-ip)` or 5-tuple (adding ports) hashing; more
  entropy spreads connections more evenly.
- **The real BGP downside is rehashing.** ECMP hashes are usually not *stable*: when the
  backend set changes — a node's BGP session drops — existing connections get rehashed
  essentially at random, so most in-flight connections break with "connection reset by
  peer." It is a one-time clean break rather than ongoing loss, but it is a break. The
  mitigations MetalLB names: enable resilient ECMP / resilient LAG on the routers if they
  support it; pin service deployments to a smaller node set; put a stateful ingress
  controller in front so you only take the hit when the ingress deployment itself changes;
  or add client-side retry.

**The backend changed, and this is a live correction.** **FRR-K8s is now MetalLB's default
BGP backend** — a Kubernetes wrapper around FRR with its own API, which MetalLB configures
rather than driving FRR directly. It adds BFD support for BGP sessions, IPv6 for BGP and
BFD, multi-protocol BGP, and the ability to merge additional FRR configuration from other
controllers so several components can share one FRR instance and its sessions. The older
**FRR mode is deprecated and slated for removal**; the original native implementation
remains but lacks BFD and MP-BGP. Its FRR-based modes carry three constraints worth knowing
before you design: `routerID` and `myASN` may be overridden but must be identical across
all advertisements; an eBGP peer more than one hop away requires `ebgpMultiHop: true`; and
peering to a speaker on the same host is not supported (the native implementation allows
it). Verified against `metallb/metallb`, `website/content/concepts/`, commit `18469a1`,
read 2026-08-18.

The full BGP configuration, with the fields that matter:

```yaml
# Peer every node's speaker with the top-of-rack router.
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata: { name: tor-a, namespace: metallb-system }
spec:
  myASN: 64513
  peerASN: 64512
  peerAddress: 192.168.10.1
  holdTime: 90s                     # session declared dead after this
  keepaliveTime: 30s                # convention: holdTime = 3 × keepaliveTime
  passwordSecret:                   # MD5/TCP-AO session auth; prefer this over
    name: tor-bgp-password          # the inline `password` field
    namespace: metallb-system
  bfdProfile: fast                  # sub-second failure detection; FRR-K8s only
  enableGracefulRestart: true       # keep forwarding while the session re-establishes
  ebgpMultiHop: false               # true if the peer is more than one hop away
  nodeSelectors:                    # peer only from nodes in this rack
    - matchLabels: { rack: c14 }
---
apiVersion: metallb.io/v1beta1
kind: BFDProfile
metadata: { name: fast, namespace: metallb-system }
spec:
  receiveInterval: 300              # ms
  transmitInterval: 300
  detectMultiplier: 3               # session down after 3 × 300 ms ≈ 900 ms,
                                    # versus 90 s on BGP hold time alone
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata: { name: svc-pool, namespace: metallb-system }
spec:
  addresses: ["203.0.113.0/24"]
  autoAssign: true
  avoidBuggyIPs: true               # skip .0 and .255
---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata: { name: svc-bgp, namespace: metallb-system }
spec:
  ipAddressPools: ["svc-pool"]
  aggregationLength: 32             # advertise each VIP as a /32, not a summary
  localPref: 100
  communities: ["64512:100"]        # let the fabric policy on these
```

**`bfdProfile` is the line worth arguing for.** Without BFD, a dead peer is detected on the
BGP hold timer — 90 seconds by default, and your VIP blackholes for that entire window.
With a 300 ms interval and a multiplier of 3, detection is under a second. On a fabric
where your ingress path matters, that is a three-order-of-magnitude difference in outage
duration for one line of configuration.

### 14. kube-vip for the API VIP

The apiservers need one address, and it has to exist before there is a working cluster to
schedule a controller onto — which is why kube-vip runs as a **static pod** on the
control-plane nodes, read directly from `/etc/kubernetes/manifests` by the kubelet.

It floats a virtual IP to whichever node holds a leader-election lease, in ARP mode (the
node answers ARP for the VIP) or BGP mode (it advertises the VIP, like MetalLB). It can
also load-balance apiserver traffic across the control-plane nodes rather than merely
failing the VIP over, and optionally provide Service LoadBalancer functionality if you do
not want MetalLB.

The failover-timing defaults, which are the numbers to know
(`kube-vip/kube-vip`, `cmd/kube-vip.go` and `pkg/kubevip/config_generator.go`, commit
`85a8c94`, read 2026-08-17):

| Flag / env | Default | Meaning |
|---|---|---|
| `--leaseDuration` / `vip_leaseduration` | **15 s** | How long a leader may hold the lease without renewing. **This bounds VIP outage on an ungraceful leader loss.** |
| `--leaseRenewDuration` / `vip_renewdeadline` | **10 s** | How long the leader keeps trying to renew before giving up |
| `--leaseRetry` / `vip_retryperiod` | **2 s** | Interval between leader-election attempts |
| `--leaseName` | `plndr-cp-lock` | The Lease object's name |
| `--leaderElectionType` | `kubernetes` | Or `etcd` — which matters, see below |
| `--bgp` / `bgp_enable` | `false` | ARP mode unless you turn BGP on |
| `cp_enable` / `svc_enable` | — | Control-plane VIP and/or Service LoadBalancer |

**Read the 15-second default as an SLO.** If a control-plane node loses power, the API VIP
is unreachable for up to `leaseDuration` before another node takes it. Kubelets tolerate
that (they retry), but `kubectl`, CI systems and anything doing synchronous API calls will
see errors for up to 15 seconds. Shortening it makes failover faster and makes spurious
failovers more likely under load — the standard leader-election trade, and one you should
set deliberately rather than inherit.

**`leaderElectionType: kubernetes` has a bootstrap circularity worth seeing.** Leader
election through the Kubernetes API requires a reachable API server; the API server is
reached through the VIP; the VIP is held by the leader. During initial cluster bring-up
(lesson 01) this resolves because the first control-plane node runs kube-vip pointing at
its own local apiserver before the VIP is the only path. The `etcd` election backend exists
precisely to break that dependency for people who would rather not rely on it.

```yaml
# /etc/kubernetes/manifests/kube-vip.yaml  (static pod, ARP mode, control plane VIP)
apiVersion: v1
kind: Pod
metadata: { name: kube-vip, namespace: kube-system }
spec:
  hostNetwork: true                      # it must own an address on the host
  containers:
    - name: kube-vip
      image: ghcr.io/kube-vip/kube-vip:v1.0.0
      args: ["manager"]
      env:
        - { name: vip_interface,     value: "eno1" }
        - { name: address,           value: "10.0.9.100" }   # the API VIP
        - { name: port,              value: "6443" }
        - { name: cp_enable,         value: "true" }
        - { name: svc_enable,        value: "false" }        # MetalLB owns Services
        - { name: vip_arp,           value: "true" }
        - { name: vip_leaderelection,value: "true" }
        - { name: vip_leaseduration, value: "15" }
        - { name: vip_renewdeadline, value: "10" }
        - { name: vip_retryperiod,   value: "2" }
      securityContext:
        capabilities: { add: ["NET_ADMIN", "NET_RAW"] }       # to add the VIP and gARP
      volumeMounts:
        - { name: kubeconfig, mountPath: /etc/kubernetes/admin.conf }
  volumes:
    - name: kubeconfig
      hostPath: { path: /etc/kubernetes/admin.conf }
```

### 15. Where MetalLB and kube-vip sit in the cluster

```
        OFF-CLUSTER CLIENTS                     kubectl / kubelets / CI
                 │                                        │
                 ▼                                        ▼
        ┌────────────────────┐                  ┌─────────────────────┐
        │  ToR ROUTER(S)     │                  │  API VIP 10.0.9.100 │
        │  ASN 64512         │                  │  (kube-vip, ARP or  │
        │  ECMP over the     │                  │   BGP, leader-held) │
        │  advertised /32s   │                  └──────────┬──────────┘
        └─────────┬──────────┘                             │ :6443
      BGP sessions │ (one per node, MD5 + BFD)             │
     ┌─────────────┼─────────────┐            ┌────────────┼────────────┐
     ▼             ▼             ▼            ▼            ▼            ▼
 ┌────────┐   ┌────────┐   ┌────────┐    ┌────────┐  ┌────────┐  ┌────────┐
 │ node 1 │   │ node 2 │   │ node N │    │  cp-1  │  │  cp-2  │  │  cp-3  │
 │ speaker│   │ speaker│   │ speaker│    │ apisrv │  │ apisrv │  │ apisrv │
 │ +FRR-K8s   │ +FRR-K8s   │ +FRR-K8s│   │ +etcd  │  │ +etcd  │  │ +etcd  │
 │ announces  │ announces  │ announces│  │+kube-vip│ │+kube-vip│ │+kube-vip│
 │ 203.0.113.7│ 203.0.113.7│203.0.113.7│ │ (static │ │ (static │ │ (static │
 └────┬───┘   └────┬───┘   └────┬───┘    │  pod)  │  │  pod)  │  │  pod)  │
      │            │            │        └────────┘  └────────┘  └────────┘
      └────────────┴────────────┘             ▲          ▲          ▲
              kube-proxy / CNI                └──────────┴──────────┘
                     │                          exactly one holds the lease
                     ▼                          (leaseDuration 15 s bounds
              ┌─────────────┐                    the failover window)
              │ Service pods│
              └─────────────┘

  L2 MODE, for contrast — the same picture minus the routers:
      one elected node answers ARP for 203.0.113.7; ALL ingress traffic for that
      VIP enters through that node's NIC and is then spread by kube-proxy.
      No router configuration required. Bandwidth capped at one NIC. Cannot cross
      a subnet, so it cannot span racks in a routed fabric.

  WHY BGP ONCE YOU HAVE MORE THAN ONE RACK: nodes in different racks are in
  different L2 domains, so one ARP-advertised VIP cannot span them at all — and
  even inside one rack, L2 funnels the VIP through a single node. BGP advertises
  the /32 into the ROUTED fabric, so every ToR can ECMP it across every node in
  every rack. Routed, not bridged, is how ingress scales past one rack — the same
  conclusion module 09 reached for compute and storage traffic.
```

## Perspectives

**Financial / CFO view.** The equation in §1 is the neocloud pitch, inverted: "we can run
this fleet at higher utilisation *and* amortise fixed opex across more nodes than any
single customer, so we can buy at capex prices and still sell below what your own owned
economics would achieve." Your job is a number finance can defend in a budget review, with
every figure dated and every assumption named. The one-sentence answer they can act on is
"the comparator rate and the depreciation life decide this; utilisation decides whether we
regret it."

**Platform-operator view.** The model is only as good as the operations feeding it. A slow
RMA loop (06) or a stalling storage tier (07) does not merely create a reliability problem
— it creates a *utilisation* problem, and utilisation is the denominator. §11 puts numbers
on it: one node sick and undetected for a week is 2.9 points of fleet utilisation, worth
more than the entire energy line. Every automation you build is, in this lesson's units,
denominator work.

**Network-engineer view.** MetalLB and kube-vip are not add-ons; without them a bare-metal
cluster cannot serve external traffic and its control plane has no stable address. The
BGP-versus-L2 decision is the routed-versus-bridged trade-off module 09 taught, applied to
the control path. And the two most consequential settings are timers, not topology: BFD's
sub-second detection versus a 90-second BGP hold, and kube-vip's 15-second lease duration.
Both are outage-duration SLOs hiding in a config file.

**Failure-mode view.** Both halves of this lesson fail quietly. The economic model is wrong
when someone reuses a `$/GPU-hr` figure without its tier, term, bundle and date — lesson
11.7's data point of a one-year contract rate rising ~40% in five months proves a "current"
number goes stale inside a fiscal year. The networking half is wrong when someone deploys
L2 mode across multiple racks and is surprised that traffic does not spread: `EXTERNAL-IP`
looks fine, the service works, and one node is carrying everything until you look at where
packets actually land.

## Real-world use cases

- **MetalLB's production adopters, verified in-repo.** `ADOPTERS.md` in the MetalLB
  repository (commit `18469a1`, read 2026-08-18) names **1&1 Mail & Media** as an end-user
  since 2018 — MetalLB is "the main way to get network traffic into the clusters driving
  most of the internal services for WEB.DE, GMX and mail.com"; **Deutsche Telekom AG**
  since 2021 in its Das SCHIFF / T-CaaS telco cloud platform; **Red Hat** since 2022,
  shipping MetalLB with OpenShift as the supported bare-metal LoadBalancer implementation;
  **SUSE** since 2023 as a core part of SUSE Edge, explicitly including load balancing the
  Kubernetes API in HA topologies; and **Nokia EDA** since 2024, implementing the
  LoadBalancer controller role inside Nokia's Event Driven Automation product. **What it
  shows:** this is the primary ingress path at a large consumer email provider and a
  component inside two vendors' shipping products — not a lab tool. It is also a rare case
  where the adopter list is a maintained file in the repository rather than a marketing
  page, so you can verify it yourself.
- **CoreWeave's filings as the depreciation-policy data point.** CoreWeave's FY2025 Form
  10-K sets the estimated useful life of technology equipment including GPUs at **six years,
  straight-line**, against a weighted-average customer contract duration of about **five
  years** — the asset must earn beyond the contracted window for the economics to close.
  **What it shows:** §7(a)'s 16.7-point sensitivity to depreciation life is not academic.
  A provider on a six-year schedule can quote a materially lower rate for identical hardware
  than one on four years and report the same margin, which means part of an apparent price
  advantage can be an accounting-policy difference. It also tells you what your provider
  needs from the back half of the asset's life, and why previous-generation capacity gets
  re-contracted cheaply rather than retired. *(Sourced via lesson 11.7, which cites
  CoreWeave's S-1 and FY2025 10-K; `sec.gov` is blocked by this session's egress proxy and
  the filings were not re-fetched here.)*
- **The colocation market as the facility-cost anchor.** As of Q1 2026, committed
  high-density GPU colocation in major US markets priced at roughly **$150–250 per kW per
  month**, with Chicago around $200–230 and Northern Virginia around $190–235 for 250–500 kW
  requirements, and the consistent market observation that **power availability, not floor
  space, is the binding constraint** for high-density deployments. **What it shows:** the
  facility line in §3 is 13.6% of total cost — the third-largest single line — and it is
  priced per provisioned kW, which is why lesson 02b.7's nameplate arithmetic ends up in a
  financial model. *(Figures from trade-press and broker summaries of the Q1-2026 market;
  individual provider pricing pages were not fetched.)*
- **Uptime Institute's 2025 Global Data Center Survey as the PUE reality check.** The
  weighted-average annual PUE reported by survey respondents was **1.54**, essentially
  unchanged for six consecutive years, with rack densities rising into the 10–30 kW band
  and few facilities above 30 kW. **What it shows:** the PUE numbers used in vendor TCO
  models (1.1–1.2) describe purpose-built new facilities, not the industry. And per §7(c),
  it barely matters to *cost* at US electricity rates — but a facility averaging 1.54 while
  your GPU racks need 40+ kW is telling you something important about whether it can host
  you at all. *(`uptimeinstitute.com` was not fetched; figures from search extracts of the
  2025 survey summary and press release.)*

## Worked example — the build-versus-buy memo

The task: a team wants 64 H100s for an 18–36 month programme. Own or rent? Produce the
answer the way it should be handed over.

**Step 1 — establish the comparator, with all five fields.** This comes first because §7(d)
showed it dominates.

```
  Candidate comparators, August 2026 snapshot:
    (a) hyperscaler on-demand, p5-class            $7.89/GPU-hr   median, no commitment
    (b) hyperscaler 1-yr committed                 $4.90/GPU-hr   1-yr, take-or-pay
    (c) dedicated-cloud on-demand, cohort median   $4.17/GPU-hr   no commitment
    (d) tier-1 neocloud 1-yr reserved              $2.10/GPU-hr   1-yr take-or-pay,
                                                                  parallel FS extra
  The team cannot forecast demand 12 months out (that is why they are asking), so a
  take-or-pay commitment carries utilisation risk they would also bear by owning.
  → the honest apples-to-apples comparator is (c), $4.17 on-demand: no commitment on
    either side, matched tier, matched bundle. Comparators (a) and (d) are reported
    as bounds, not as the decision basis.
```

**Step 2 — build the cost stack.** §3, with this team's actual inputs: they have colo space
at $210/kW/month, US-Midwest power at $0.079/kWh, a 5-year depreciation policy, corporate
cash at an 8% internal hurdle rate rather than GPU-backed debt, and they already employ the
engineer (so `f = 0.5` is the incremental fraction, not a new hire).

```
    depreciation  2,612,000 × 0.90 / 60                       = $39,180
    financing     0.08 × 2,612,000 × 0.55 / 12                = $ 9,577   (8%, not 12%)
    energy        8 × 7.3 × 730 × 1.30 × 0.079                = $ 4,379
    facility      8 × 10.2 × 210                              = $17,136
    support       2,432,000 × 0.07 / 12                       = $14,187
    staff         0.5 × 275,000 / 12                          = $11,458
    storage       lesson 07 sizing sheet                      = $18,000
    network                                                   = $ 2,000
    ────────────────────────────────────────────────────────────────────
    monthly_total                                             = $115,917
    owned $/GPU-hr(U) = 115,917 / (46,720 · U)                = $2.481 / U
```

**Step 3 — derive break-even and payback.**

```
    U*  vs $4.17  = 2.481 / 4.17                              = 59.5%
    U*  vs $7.89  = 2.481 / 7.89                              = 31.4%
    U*  vs $2.10  = 2.481 / 2.10                              = 118.1%  → never

    cash opex/month = 115,917 − 39,180                        = $76,737
    payback(U) = 2,612,000 / (46,720 · U · 4.17 − 76,737)
        U = 0.90 → 2,612,000 / 98,603                         = 26.5 months
        U = 0.70 → 2,612,000 / 59,638                         = 43.8 months
        U = 0.60 → 2,612,000 / 40,156                         = 65.0 months  ← past life
```

**Step 4 — sensitivity, run on the terms §7 identified.**

| Scenario | `U*` vs $4.17 | Payback at `U = 0.80` |
|---|---|---|
| **Base** (L=5, S=0.10, i=8%) | **59.5%** | 33.4 mo |
| Depreciation life 3 yr | 72.7% | 33.4 mo (cash unchanged) |
| Depreciation life 6 yr | 56.3% | 33.4 mo |
| Residual 0% instead of 10% | 61.7% | 33.4 mo |
| Storage costs 2× the estimate | 68.9% | 41.5 mo |
| Support contract at 10% not 7% | 63.6% | 37.0 mo |
| Energy at $0.20/kWh | 61.1% | 34.4 mo |
| Comparator is $4.90 committed | 50.6% | 27.3 mo |
| Comparator is $2.10 committed | never | never |

Two things jump out of that table. **Depreciation life moves break-even but not payback**,
because payback is a cash calculation and depreciation is not cash — which is exactly why
you compute both: they answer different questions and a stakeholder who conflates them will
argue about the wrong number. And **the storage line, doubled, moves break-even by 9.4
points** — more than a three-year change in depreciation life. If lesson 07's sizing sheet
is wrong, this memo is wrong.

**Step 5 — the memo.**

> **Recommendation: rent on demand for the first 9–12 months, then re-evaluate with
> measured utilisation.**
>
> At our inputs — $2.61M capex (compute $2.24M, fabric $192K, one spare baseboard $180K),
> $115.9K/month all-in, 5-year straight-line to a 10% residual, 8% internal hurdle rate,
> colo at $210/kW-month on 81.6 kW nameplate, US-Midwest power at $0.079/kWh, storage per
> the lesson-07 sizing sheet — owned cost is **$2.48/GPU-hr ÷ U**. Against the matched
> comparator (dedicated-cloud on-demand, cohort median **$4.17/GPU-hr**, August 2026,
> no commitment on either side), **break-even is 59.5% sustained utilisation** and payback
> at 80% utilisation is **33 months**.
>
> We do not have a defensible forecast above 59.5%. We have never measured our own
> utilisation on dedicated capacity, the programme's duration is uncertain at 18–36 months,
> and the failure mode is asymmetric: at 65% we pay back in 51 months against a 60-month
> life, and at 55% we never pay back at all. Renting on demand costs more per hour and
> costs nothing per hour we do not use.
>
> **Three conditions would reverse this, in priority order.** (1) **Measured sustained
> utilisation above 65% for two consecutive quarters** on rented capacity — that is the
> single number the decision turns on, and we can buy it for the price of nine months of
> rental. (2) **A defensible six-year depreciation policy**, which moves break-even to
> 56.3%; three years moves it to 72.7% and settles the question in the other direction.
> (3) **Storage costing materially less than the lesson-07 estimate** — that line is 15.5%
> of monthly cost and doubling it moves break-even by 9.4 points, more than any
> depreciation assumption.
>
> **What would not change it:** energy price and PUE. Even at $0.20/kWh — more than double
> our rate — break-even moves 1.6 points. Power is a capacity constraint on this design
> (81.6 kW nameplate needs three 21.8 kW circuits at N+1 per four nodes, per the rack
> arithmetic), not a cost driver.
>
> **What we are explicitly not comparing against:** the $2.10/GPU-hr tier-1 neocloud
> reserved rate. Owning never beats it at any utilisation, because that rate sits close to
> the provider's own cost to serve at a scale we do not have. If we are willing to sign a
> one-year take-or-pay, the real comparison is neocloud-committed versus hyperscaler, and
> that is lesson 11.7's normalisation sheet, not this model.
>
> All figures are an August 2026 snapshot at the stated tier, term and bundle. Re-verify
> the comparator before signing anything; it moved ~40% in five months during 2025–26.

**Step 6 — the networking half, deployed.** On the cluster from earlier lessons, with a ToR
router at `192.168.10.1` in ASN 64512 and MetalLB in ASN 64513:

```
$ kubectl apply -f metallb-bgp/                    # BGPPeer, BFDProfile, IPAddressPool,
                                                   # BGPAdvertisement from §13
$ kubectl -n metallb-system get bgppeers
NAME    ASN     PEER ASN   ADDRESS         BFD PROFILE   STATUS
tor-a   64513   64512      192.168.10.1    fast          Established

$ kubectl expose deploy web --type=LoadBalancer --port=80 --target-port=8080
$ kubectl get svc web
NAME   TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE
web    LoadBalancer   10.96.44.12    203.0.113.7    80:31544/TCP   6s
```

And the check that distinguishes working BGP from working L2 — **multiple next hops**:

```
# on the ToR router
router# show ip bgp 203.0.113.7/32
BGP routing table entry for 203.0.113.7/32
  Paths: (3 available, best #1, multipath)
    64513, (received & used)  10.0.9.101 from 10.0.9.101   ← node 1
    64513, (received & used)  10.0.9.102 from 10.0.9.102   ← node 2
    64513, (received & used)  10.0.9.103 from 10.0.9.103   ← node 3

router# show bfd peers brief
SessionId  LocalAddress   PeerAddress   Status
1          192.168.10.1   10.0.9.101    up
2          192.168.10.1   10.0.9.102    up
3          192.168.10.1   10.0.9.103    up

# from off-cluster, across a router
$ for i in $(seq 20); do curl -s http://203.0.113.7/whoami; done | sort | uniq -c
      7 web-6d9f4-2xk9p
      6 web-6d9f4-h7t2m
      7 web-6d9f4-qw84z
```

Three next hops on the router and traffic landing on three pods is the tell. In L2 mode you
would see exactly one next hop, the same `EXTERNAL-IP`, a working service, and every packet
entering through one node's NIC — a difference that is invisible from `kubectl` and obvious
from the router.

## Practice — feeds the deliverable (both halves)

**(a) Build the capex-vs-cloud model.** A spreadsheet or notebook for a 64-GPU fleet with
**your own** inputs. It must:

1. **Name every term** in §2, with the source of each value, and state explicitly which
   power figure (nameplate or measured) each line uses and why. State your colo contract
   structure — provisioned-kW plus metered energy, or all-in — and confirm you have not
   applied PUE twice.
2. **Carry forward the numbers from earlier lessons**: the spares count and lead time from
   lesson 06's Poisson model, the storage cost from lesson 07's sizing sheet. If you skipped
   those, do them first — they are 15% and 3% of the answer.
3. **Compute `owned $/GPU-hr(U)`** and produce the **crossover chart** with the owned curve
   and *at least three* rental lines, each labelled with tier, term, bundle and date.
4. **Derive `U*` algebraically** for each comparator — `U* = monthly_total / (N × 730 × R)`
   — rather than reading it off the chart.
5. **Produce the payback chart** on a cash basis, and **verify the consistency check** from
   §10: payback should cross your asset life at the same utilisation your algebra gave for
   break-even. If it does not, find the error before going further.
6. **Run the sensitivity table**, and state which terms move the answer *and which do not*.
   The second list is the harder one and it is what makes the model defensible. Run at
   minimum: depreciation life at 3/4/5/6 years, residual at 0/10/30%, the comparator across
   its full observed band, and your two largest opex lines at ±50%.
7. **Run the fleet-size scaling** from §9 at three sizes with staff, storage and spares
   scaled sub-linearly, and state your break-even at each. This is the number that tells a
   small team the answer immediately.
8. **Cross-check** your midpoint against at least one dated external figure — lesson 11.7's
   cost-to-serve model is the natural one, and §4 shows how to reconcile the two — and
   explain any delta rather than hiding it.

**(b) Deploy MetalLB in BGP mode.** On the cluster from earlier lessons: install MetalLB
(FRR-K8s backend, the current default), configure a `BGPPeer` against a router — an FRR
container as the peer is fine in a lab — plus a `BFDProfile`, an `IPAddressPool` and a
`BGPAdvertisement`. Expose a `type: LoadBalancer` Service. **Capture the router's view**
showing the /32 with **multiple next hops** and the BFD sessions up, not just
`kubectl get svc`. Then, for contrast, switch the pool to an `L2Advertisement` and capture
the single-next-hop / single-ARP-owner behaviour, and write one paragraph on which you
would run in a multi-rack fleet and why.

**Acceptance:** committed to the deliverable folder — (1) the **capex-vs-cloud model**
producing the crossover chart, the algebraically-derived break-even, the payback chart with
its consistency check, the sensitivity table naming both the terms that matter and the ones
that do not, and the fleet-size scaling; every dollar figure carrying tier, term, bundle and
date; **and** (2) a **MetalLB BGP service VIP** with router-side evidence of ECMP, plus the
L2 contrast. Together with lesson 01–02's KTHW/etcd writeup, this is the full module
deliverable — see [`practice/capex-vs-cloud/README.md`](../practice/capex-vs-cloud/README.md)
for the layout.

## Common pitfalls

- **Quoting a break-even utilisation without its comparator.** *Symptom:* two people
  reading the same model reach opposite conclusions. *Mechanism:* `U* = C/(N·730·R)` is
  inversely proportional to `R`, and the observed H100 rate band spans $1.45 to $11.06.
  "Break-even is 62%" and "owning never wins" are both true about the same fleet. State
  tier, term, bundle and date, every time.
- **Treating utilisation as a fixed input rather than the model's dominant variable.**
  *Symptom:* a fleet that looked great at a planned 85% and is a disaster at the 55% it
  actually achieves. *Mechanism:* cost is `∝ 1/U`, a hyperbola — the penalty accelerates as
  utilisation falls, so the error is worst exactly where you are least prepared for it.
  Always show the curve, never a point estimate.
- **Omitting the terms that only exist when you own.** *Symptom:* a model that flatters
  ownership by 30% or more. *Mechanism:* storage (15%), support contracts (12%), staff
  (9.5%) and spares appear in no rental quote, so an incomplete model is always biased in
  the same direction. Lesson 11.7's normalisation sheet solves the mirror-image problem on
  the rental side; both are the same discipline.
- **Applying PUE on top of an all-in colo charge, or omitting it entirely.** *Symptom:* a
  power line 30% wrong in either direction. *Mechanism:* a `$/kW/month` charge on
  provisioned power typically *is* the facility's cooling and distribution overhead sold as
  a fee. Write down which structure your contract uses before computing anything.
- **Billing energy at nameplate power.** *Symptom:* an energy line ~40% too high.
  *Mechanism:* nameplate (10.2 kW for a DGX H100) is a provisioning figure — six PSUs, 4+2
  redundant, worst case. Measured sustained draw under real training load is around 7.3 kW.
  Provision circuits at nameplate; bill energy at measured.
- **Over-weighting energy and PUE.** *Symptom:* a week spent negotiating $/kWh while the
  depreciation policy goes unexamined. *Mechanism:* at US industrial rates energy is ~4% of
  total cost of ownership for this fleet, so PUE moves break-even by under a point.
  Sensitivity to any line equals that line's share of total cost — compute the shares first
  and let them tell you where to spend your attention.
- **Assuming an aggressive depreciation life or residual to make the answer come out
  right.** *Symptom:* a reviewer challenges it in the first five minutes. *Mechanism:*
  3-year versus 6-year moves break-even by 16.7 points, so it is the single most effective
  place to put a thumb on the scale, and everyone knows it. State the reasoning about
  obsolescence, not just the number.
- **Confusing break-even utilisation with payback month.** *Symptom:* an argument in which
  two people are both right. *Mechanism:* break-even is an accounting statement including
  depreciation; payback is a cash statement excluding it. Depreciation life changes the
  first and not the second. Compute both, and use §10's consistency check — they must agree
  on where payback crosses the asset life.
- **Building the economic model in isolation from the operational ones.** *Symptom:* a
  model that looks fine on paper while the fleet underperforms it. *Mechanism:* a slow RMA
  loop (06) or a storage stall (07) appears in this model only as a lower `U`. §11 gives the
  conversion: one node sick for a week is 2.9 points of fleet utilisation.
- **Deploying MetalLB in L2 mode and expecting load balancing across racks.** *Symptom:*
  one node's NIC saturated while the others idle; nothing wrong in `kubectl`. *Mechanism:*
  L2 is failover through one ARP-elected node within a single subnet — it does not spread
  traffic and cannot cross a router. Check the *router's* next-hop count, not the Service's
  `EXTERNAL-IP`.
- **Running BGP without BFD.** *Symptom:* a 90-second ingress blackhole when a node dies.
  *Mechanism:* without BFD, a dead peer is detected on the BGP hold timer. A `BFDProfile`
  with a 300 ms interval and a multiplier of 3 detects it in under a second. One block of
  YAML, three orders of magnitude.
- **Not knowing kube-vip's lease duration.** *Symptom:* unexplained ~15-second windows of
  API unavailability after a control-plane node loss. *Mechanism:* `leaseDuration` defaults
  to **15 s**, which bounds how long the old leader's VIP claim persists. It is an SLO; set
  it deliberately.

## Self-check

**(a) At what sustained utilisation does a 64-GPU owned fleet beat renting, and what is the
model most sensitive to?**
**Answer:** It depends entirely on what you are comparing against, and refusing to give one
number is the correct answer. At the base parameters — $2.61M capex, $120,222/month all-in,
5-year straight-line to a 10% residual — owned cost is `$2.573/U` per GPU-hour, so
`U* = 2.573/R`: **32.6%** against a hyperscaler on-demand median of $7.89, **52.5%** against
a $4.90 committed hyperscaler rate, **61.7%** against a $4.17 dedicated-cloud on-demand
median, and **never** against a $2.10 tier-1 neocloud reserved rate, which sits at or below
that provider's own cost to serve. On sensitivity: because `U*` is directly proportional to
total monthly cost, each line's sensitivity equals its share of cost. The three that
matter are **the comparator rate** (swings the answer from 33% to never — and it is not
yours to choose), **depreciation life** (3 vs 6 years moves break-even 16.7 points, and it
is a policy choice), and **fixed opex divided by fleet size** (85.7% at 8 GPUs, 61.7% at 64,
53.1% at 512). The ones that do *not* matter at US industrial electricity rates are **PUE**
(0.8 points across 1.15–1.54) and **the power-draw figure** (1.0 point between nameplate
and measured) — energy is ~4% of total cost here.

**(b) Why does your model produce $2.57/GPU-hr while lesson 11.7's cost-to-serve model
produces $1.59 for the same hardware?**
**Answer:** They are the same equation with different parameters, and the difference is
almost entirely **fixed operating expense divided by fleet size**. Run this model with
11.7's inputs — $280K node, 10% salvage, 6-year life, 12% cost of capital on 55% average
outstanding, 10.2 kW at PUE 1.30 and $0.08/kWh, $3,000/GPU fabric, and a single combined
"facility + staff + storage + ops" term of $1,800/node-month — and it reproduces
$10.888/node-hour, $1.361/GPU-hour calendar, $1.592/GPU-hour sold at `A × U = 0.855`,
exactly. The gap to this lesson's $20.59/node-hour comes from two places: a 5-year rather
than 6-year life (worth ~$1.9/node-hour) and, dominantly, that this lesson's colo, support,
staff, storage and network lines total **$10.60/node-hour** against 11.7's single
$2.466/node-hour `F` term — a 4.3× difference. 11.7's number is a provider's, amortised
across thousands of nodes; an 8-node fleet carries a whole storage tier and half an engineer
by itself. That is not a contradiction between the lessons; it is the quantified reason
neoclouds exist.

**(c) Break-even utilisation and payback month — how do they relate, and why compute both?**
**Answer:** Break-even utilisation is an **accounting** statement: at what `U` does total
cost including depreciation equal the rental you would otherwise pay. Payback month is a
**cash** statement: capex is spent at `t = 0`, depreciation is not cash, so
`payback = K_total / (N·730·U·R − monthly_cash_opex)`. They are two views of one fact, and
the consistency check is that **payback crosses the asset life at exactly the break-even
utilisation** — in the base case, 61.7% algebraically and ~62% from the payback chart
crossing 60 months. If yours do not agree, one of them is wrong, and that is the cheapest
error-check in the model. Compute both because they answer different questions and respond
to different inputs: changing depreciation life from 5 to 3 years moves break-even by 11
points and moves payback not at all, so a stakeholder arguing about depreciation is arguing
about the accounting answer only. And the payback curve shows the shape of the risk that
the break-even number hides: 100%→80% utilisation stretches payback by 12 months, but
65%→55% stretches it by 43.

**(d) Name three costs that cloud pricing hides, and size them.**
**Answer:** From this model's base case, as shares of a $120,222/month total: **storage**
(**15.0%**, $18,000/month for the parallel filesystem and object tier sized in lesson 07 —
the third-largest line and one that appears in no rental quote as a separate item);
**support and maintenance contracts** (11.8%, at 7% of hardware capex per year — vendor
break/fix, which the provider absorbs into their rate); and **staff** (9.5% here at 0.5 FTE
of a $275K fully-loaded engineer, rising to **27.5%** of total cost at 8 GPUs, which is what
makes small owned fleets structurally uncompetitive). Two more worth naming: **spares**
(capital held idle — §8 shows a spare baseboard actually loses money at 64 GPUs unless you
are capacity-constrained), and **idle time itself**, since owning means paying depreciation,
facility and the power floor whether or not a GPU is busy, while rental is pay-per-use.
Every one of these biases an incomplete model in the same direction: towards owning.

**(e) Why BGP over L2 for MetalLB once you have more than one rack, and what does BGP cost
you?**
**Answer:** L2 mode advertises the VIP by ARP or NDP within a single subnet, and all traffic
for a given VIP enters through one elected node — MetalLB's documentation states plainly
that it is a failover mechanism, not a load balancer. Two consequences: ingress bandwidth
for a VIP is capped at one node's NIC, and the VIP cannot cross a router, so nodes in
different racks (different L2 domains) cannot share one L2-mode VIP at all. **BGP** peers
each node with the top-of-rack routers and advertises the service prefix into the routed
fabric, so routers ECMP it across every node in every rack — real horizontal ingress
bandwidth and cross-rack reach. What BGP costs you is **rehashing**: router ECMP hashes are
usually not stable, so when the backend set changes (a node's session drops) most existing
connections are re-hashed to a different next hop and break with a connection reset. It is a
one-time clean break, not ongoing loss, and the mitigations are resilient ECMP on the
routers, a stateful ingress controller in front, pinning deployments to fewer nodes, or
client retry. Two operational notes: **FRR-K8s is now the default BGP backend** (plain FRR
mode is deprecated), and **configure a `BFDProfile`** — without it a dead peer is detected
on the 90-second BGP hold timer instead of in under a second.

**(f) How does lesson 06's health loop concretely change the number in this model?**
**Answer:** It changes `U`, the denominator, and the conversion is arithmetic. A node that
is sick but still `schedulable` contributes approximately zero useful work while accruing
its full share of depreciation, facility, support and staff cost. At 64 GPUs one node
undetected for a week is `8 GPUs × 168 h = 1,344` GPU-hours out of the month's 46,720 —
**2.9 points of fleet utilisation**, which at the $4.17 comparator is worth ~$5,600 of
rental-equivalent value, more than the entire monthly energy line of $4,711. CoreWeave's
"up to a month" manual worst case is 12 points of a month's utilisation from a single node.
Reading the whole module through this equation: lesson 05 protects `U` at the start of life
(a node un-provisioned depreciates at $138/day with nothing against it), lesson 06 protects
it continuously, lesson 07 protects it against a subsystem that stalls without erroring, and
lessons 02–03 protect against the case where `U` goes to zero fleet-wide at once. Every one
of them is denominator work, and this lesson is what makes them comparable in the same
units.

## Connections & what's next

This is the module's last lesson and its job is to close the loop the module opened. Lesson
01 handed you a hand-built control plane with no cloud anything in front of it; §14 supplies
the missing load balancer. Lesson 02 made etcd's uptime your problem; the kube-vip VIP is
what makes that control plane addressable once there is more than one apiserver, and its
15-second lease duration is the outage bound. Lesson 03's stacked-etcd HA design is exactly
what kube-vip fronts. Lesson 04's declarative fleets are what keep the staff line sub-linear
— 9.5% of cost at 64 GPUs, 27.5% at 8. Lesson 05's PXE pipeline turns capex sitting in a
rack into utilisation. Lesson 06's health loop is a direct input to the denominator, and §8
sends its spares model back for one correction. Lesson 07's storage sizing is 15% of the
numerator and a second-order effect on the denominator.

Forward, this feeds **module 11**, which takes the same discipline outward. Lesson 11.7
normalises *vendor* quotes onto a common basis — the mirror image of what this lesson does
for build-versus-buy — and §4 above shows the two models are literally the same equation, so
you can reconcile them line by line instead of arguing about whose spreadsheet is right.
Where they differ is instructive rather than contradictory, and being able to explain *why*
a provider's cost structure is not yours is a stronger answer than either number alone.

**Close the module here.** Build both halves of the
[capex-vs-cloud + KTHW/etcd deliverable](../practice/capex-vs-cloud/README.md), then work
the [module checkpoint](../checkpoint.md) cold: provision from scratch, recover etcd inside
30 minutes, diagram the HA design, close the health loop, defend your own economics with
your own numbers, and stand up a MetalLB BGP VIP without notes. That checkpoint is the gate
this whole module has been building toward.

## References & further reading

**Primary sources (verified against upstream source this session)**

- **MetalLB** — <https://github.com/metallb/metallb> — read 2026-08-18 at commit `18469a1`.
  Source for the L2 and BGP concept material in §13 (`website/content/concepts/layer2.md`
  and `bgp.md`): L2 as a failover rather than load-balancing mechanism, the single-node
  bandwidth cap, memberlist-based failure detection with gratuitous ARP, the stateless
  hash-based leader election and its split-brain behaviour; BGP per-connection packet
  hashing (3-tuple and 5-tuple), the ECMP rehashing behaviour on backend-set changes and
  its five named mitigations; and the `BGPPeer` v1beta2 field set
  (`api/v1beta2/bgppeer_types.go`) including `holdTime`, `keepaliveTime`, `bfdProfile`,
  `passwordSecret`, `enableGracefulRestart`, `ebgpMultiHop` and `nodeSelectors`.
  **Correction applied from this source:** **FRR-K8s is now the default BGP backend** (it
  adds BFD, IPv6 BGP/BFD, MP-BGP and configuration merging); the plain **FRR mode is
  deprecated and slated for removal**. An earlier version of this lesson said MetalLB
  "bundles FRR". *`metallb.universe.tf` is blocked by this session's egress proxy; the
  website content was read from the repository that generates it.*
- **MetalLB `ADOPTERS.md`** —
  <https://github.com/metallb/metallb/blob/main/ADOPTERS.md> — read directly at commit
  `18469a1`. Source for the named production adopters in Real-world use cases: 1&1 Mail &
  Media (end-user, 2018), Deutsche Telekom AG (2021), Red Hat (vendor, 2022), SUSE (vendor,
  2023), Nokia EDA (vendor, 2024), CLASTIX (2022), Kiratech, Westnet.
- **kube-vip** — <https://github.com/kube-vip/kube-vip> — read 2026-08-17 at commit
  `85a8c94`. Source for the failover-timing defaults in §14 (`cmd/kube-vip.go` and
  `pkg/kubevip/config_generator.go`): `leaseDuration` **15 s**, `leaseRenewDuration`
  **10 s**, `leaseRetry` **2 s**, `leaseName` `plndr-cp-lock`, `leaderElectionType`
  `kubernetes` or `etcd`, `bgp` defaulting to false; and the environment-variable names
  (`vip_arp`, `vip_leaderelection`, `vip_leaseduration`, `vip_renewdeadline`,
  `vip_retryperiod`, `vip_interface`, `address`, `cp_enable`, `svc_enable`,
  `bgp_peeraddress`) in `pkg/kubevip/config_envvar.go`. *`kube-vip.io` is blocked by this
  session's egress proxy; all facts are from the source tree.*
- **Lesson 02b.7 — Power and thermals** —
  [`../../02b-host-topology/lessons/07-power-and-thermals.md`](../../02b-host-topology/lessons/07-power-and-thermals.md)
  — the source of every power figure this model uses and of the distinction that makes it
  correct: DGX H100 nameplate 10.2 kW from six 3,300 W Titanium PSUs in 4+2 redundancy; the
  bottoms-up node budget showing GPUs are ~55% of node power; the 415 V three-phase / 32 A
  circuit at ~21.8 kW and the four-nodes-per-rack N+1 arithmetic; and the empirical
  measurement of ~8.4 kW peak / ~7.3 kW typical for an 8×H100 node under real training load.
- **Lesson 11.7 — Decomposing the neocloud-vs-hyperscaler price gap** —
  [`../../11-gpu-cost-economics/lessons/07-neocloud-vs-hyperscaler.md`](../../11-gpu-cost-economics/lessons/07-neocloud-vs-hyperscaler.md)
  — the price surface used for every comparator in §5 (H100 on-demand $2.19–$11.06 with a
  dedicated-cloud median of $4.17 and a hyperscaler median of $7.89; 1-year reserved
  $1.45–$2.99; whole-cohort median ~$3.15), the cost-to-serve model that §4 reconciles
  against line by line, the depreciation-life sensitivity, and the normalisation discipline
  (price, tier, term, date, bundle) that §5's comparator table applies.
- **Lesson 06 and lesson 07 of this module** — the spares model
  ([`06-hardware-health-remediation-rma.md`](06-hardware-health-remediation-rma.md) §10) and
  the storage sizing ([`07-storage-for-ai.md`](07-storage-for-ai.md) §6 of the worked
  example) that supply the `K_spare` and `stor` terms directly. §8 above revises the spares
  conclusion.

**Market and facility data (August 2026 snapshots)**

- **Uptime Institute Global Data Center Survey 2025** —
  <https://uptimeinstitute.com/resources/research-and-reports/uptime-institute-global-data-center-survey-results-2025>
  — **what it shows:** a weighted-average annual PUE of **1.54** across respondents,
  essentially flat for six consecutive years, with rack densities rising into the 10–30 kW
  band and few facilities exceeding 30 kW. The reality check against the 1.1–1.2 PUE figures
  that appear in vendor TCO models. *Domain not fetched this session; figures from search
  extracts of the survey summary and Uptime's press release.*
- **US industrial retail electricity price** — EIA data, ~**8.6 ¢/kWh** actual for 2025 with
  a ~8.5 ¢/kWh forecast for 2026, and a reported ~8.6% year-over-year increase in early
  2026. The anchor for the `e` parameter, and the reason §7(c) concludes energy is a small
  share of TCO at US rates. *`eia.gov` was not fetched this session; figures from search
  extracts of EIA data summaries.*
- **GPU colocation pricing, Q1 2026** — committed high-density colocation at roughly
  **$150–250/kW/month** in US markets (Chicago ~$200–230, Northern Virginia ~$190–235 for
  250–500 kW requirements), with power availability rather than floor space as the binding
  constraint. The anchor for the `colo` parameter, which is 13.6% of total cost in the base
  case. *From trade-press and broker market summaries; individual provider pricing pages
  were not fetched.*
- **CoreWeave Form S-1 (March 2025) and FY2025 Form 10-K** — six-year straight-line
  depreciation on technology equipment including GPUs, against a weighted-average customer
  contract duration of ~5 years. The real-world anchor for §7(a)'s depreciation-life
  sensitivity. *`sec.gov` is blocked by this session's egress proxy; cited via lesson 11.7,
  which sourced it from search extracts of the filings and CoreWeave's investor-relations
  releases.*

**Deeper dives**

- **SemiAnalysis GPU rental price index and ClusterMAX ratings** —
  <https://gpu-index.semianalysis.com/> and <https://www.clustermax.ai/overview> — the live
  price surface and the independent reliability tiering behind §5's comparator table.
  **Re-pull before quoting anything**: lesson 11.7 documents a one-year contract rate rising
  ~40% in five months during 2025–26. *Both domains are blocked by this session's egress
  proxy and were not fetched; listed as the place to refresh your own inputs.*
- **FRR-K8s** — <https://github.com/metallb/frr-k8s> — MetalLB's current BGP backend, with
  its own API for merging additional FRR configuration from other controllers. Read it if
  you need MetalLB and another BGP-speaking component (a CNI in BGP mode, for instance) to
  share one FRR instance and its sessions on the same nodes.
- **Kubernetes cloud-controller-manager and the `Service` load-balancer contract** —
  <https://kubernetes.io/docs/concepts/architecture/cloud-controller/> — the interface
  MetalLB stands in for, and why `EXTERNAL-IP` stays `<pending>` when nothing implements it.
  *`kubernetes.io` is blocked by this session's egress proxy; listed as optional depth and
  not relied upon for any claim above.*
