---
lesson: 07
title: "Decomposing the neocloud-vs-hyperscaler price gap"
module: 11
concept: "price-gap decomposition"
status: not-started
est_time: "4.5 hrs"
prev: "06-commitment-strategy.md"
next: "08-chargeback-showback.md"
artifacts: ["a fully-loaded $/GPU-hr normalization sheet comparing two real-shaped H100 quotes at your utilisation"]
sources: 14
---

# Decomposing the neocloud-vs-hyperscaler price gap

## Where this fits

Lesson 06 gave you the commitment mechanics — the four knobs a contract turns, the `u > 1 − d` break-even, the `d`-quantile sizing rule, spot goodput — for capacity from a *given* vendor. This lesson sits alongside that decision and upstream of everything else in the module: before you can attribute a GPU-hour (lessons 01–05) or size a commitment against it (lesson 06), you have to know what the **rate itself is made of**.

Module 03's capstone established the discipline this lesson operationalises: a `$/hr` figure is not a number until it carries **price, SKU, reliability tier, contract term, and date**, and comparing across mismatched tiers produced an apparent 5.6× "margin" on a real 1.7× hardware difference. That lesson applied the rule to a *hardware* comparison (H100 vs H200). This one applies it to a *vendor* comparison, where the tier field is doing even more work — and where four additional cost lines (interconnect, storage, egress, support) sit outside the sticker entirely.

The output is the number lesson 06 should have been consuming all along, and the number lesson 08's internal chargeback rate is built on: **fully-loaded, availability-adjusted, utilisation-adjusted $/GPU-hour.**

## Why this matters

Within your first quarter on a GPU platform team you will be handed two quotes for the "same" H100 and asked which to buy. One is from a hyperscaler (AWS `p5`, GCP `a3`, Azure `ND H100 v5`); the other from a neocloud (CoreWeave, Lambda, Crusoe, Nebius, Together, Voltage Park, FluidStack) at a fraction of the sticker. The naive move — pick the cheap sticker — is how platform teams end up with a "cheap" fleet that cannot run multi-node training because the fabric was never in the quote, or a surprise five-figure egress bill, or a cluster that is unusable 6% of the month because the control plane is immature and nobody replaces failed nodes at 2 a.m. except you.

The gap is real. It is also **decomposable**, and your job is to explain it rather than be surprised by it. Every axis you walk either *dissolves* part of the gap (it was never apples-to-apples) or *confirms* it as savings you can bank. What survives the full walk is the number that goes in the memo; what does not survive was bundling.

The stake compounds. GPU spend is the largest line in most AI-infrastructure budgets, and procurement decisions run on multi-year commitments (lesson 06) with no cancellation right. A 2× sticker error compounded over a 3-year commitment on a 512-GPU fleet is eight figures. And it is also, verbatim, the interview question: *"Hyperscaler quotes $X, neocloud quotes $Y — which do you pick?"* A junior answer picks the cheap number. A senior answer says "neither yet — normalise both to fully-loaded $/GPU-hour at our utilisation and our fabric requirement, then decide," and then actually walks the axes with arithmetic.

## What's new here (calibration)

- **Already yours:** per-pod GPU attribution (04/05), the util-lie (05), the capex-vs-cloud crossover (module 10), commitment mechanics and the `u > 1−d` break-even (lesson 06), the ClusterMAX tiering and dated-price discipline (module 03 capstone). None of that is re-taught.
- **Genuinely new — the price surface, correctly framed.** The "3–6× gap" this module's README names is real, but it is a comparison between two *deliberately mismatched* bases. On a **matched** on-demand basis the hyperscaler-versus-dedicated-cloud premium is closer to **~1.9×**. Knowing which comparison produces which number, and saying so out loud, is most of the skill.
- **Genuinely new — cost-to-serve built from the bottom.** Not "neoclouds have a structural advantage" but a line-by-line construction of what a GPU-hour costs to produce — depreciation, financing, power, fabric amortisation, facility, ops — with every input parameterised, so you can compute the floor below which no provider can profitably price and see why the cheapest committed rates sit exactly there.
- **Genuinely new — the depreciation lever, quantified.** Moving the assumed useful life from six years to four raises the depreciation component of cost-to-serve by roughly 50%. The schedule is not a neutral accounting fact; it is an input to the sticker.
- **Genuinely new — effective availability done correctly, including the multi-node compounding.** The single-node arithmetic is easy and the previous version of this lesson got it backwards. The interesting version is what per-node availability does to a 64-node *synchronous* job, where the exponent bites.
- **Genuinely new — the egress crossover, computed.** "Free egress can offset a $0.50–1.00/hr premium" is a claim you can check, and the answer depends on your fleet size and workload shape in a way that reverses the popular advice for text inference.
- **Not re-taught:** what an H100 is, what a committed-use discount is (lesson 06 owns commitment mechanics), how to convert $/GPU-hour into $/token or $/run (lesson 05 owns work-normalised cost). This lesson stops at fully-loaded $/GPU-hour.

## Core concepts

### 1. What is actually being compared: four kinds of seller

"Neocloud versus hyperscaler" is a two-bucket framing for a market with at least four structurally different sellers. Sorting the quote into the right bucket is step zero, because the axes that matter differ per bucket.

```
   THE GPU SUPPLY MARKET, BY WHAT THE SELLER ACTUALLY OWNS

   ┌───────────────────────────────────────────────────────────────┐
   │ HYPERSCALER            AWS / GCP / Azure / OCI                │
   │   owns: DCs, network, silicon, a 200-service platform         │
   │   sells: an all-in managed service; GPU is one SKU of many    │
   │   GPU-hour absorbs: fabric, VPC, IAM, support org, adjacent   │
   │                     service margin, global footprint          │
   │   ClusterMAX 2.0: rated alongside neoclouds, NOT automatically│
   │                   top-tier — several land mid-table           │
   ├───────────────────────────────────────────────────────────────┤
   │ TIER-1 NEOCLOUD        CoreWeave / Crusoe / Nebius / Lambda   │
   │   owns: DCs (or long leases), rail-optimised IB, silicon      │
   │   sells: GPU capacity as the product, K8s/Slurm on top        │
   │   single-SKU focus; debt-financed silicon; must keep GPUs busy│
   │   ClusterMAX 2.0: CoreWeave the sole Platinum; others Gold/   │
   │                   Silver                                      │
   ├───────────────────────────────────────────────────────────────┤
   │ COMMODITY / MARKETPLACE   RunPod Community / Vast.ai /        │
   │                           SF Compute / spot aggregators       │
   │   owns: often nothing — brokers third-party or consumer-grade │
   │         hosts; heterogeneous, no fabric guarantee             │
   │   sells: the cheapest possible GPU-hour, single-node shaped   │
   │   ClusterMAX: Bronze / UnderPerform territory                 │
   ├───────────────────────────────────────────────────────────────┤
   │ MANAGED AI PLATFORM     Together / Modal / Baseten / Fireworks│
   │   owns: usually NOT the hardware — rents from the above       │
   │   sells: $/token or $/second of a managed runtime             │
   │   FOCUS models this split explicitly:                          │
   │     ServiceProviderName ≠ HostProviderName                    │
   └───────────────────────────────────────────────────────────────┘

   A quote is only comparable to another quote in the same row, OR
   after you have added back everything the lower row omits.
```

That last note is not decoration — it is a FOCUS column. Since 1.3 the spec carries `ServiceProviderName` ("who made the service available") and `HostProviderName` ("who runs the underlying infrastructure the service sits on, when that differs"), precisely so a managed platform's bill can name both. When you normalise a managed-platform quote you are re-deriving the `HostProviderName`'s rate plus the `ServiceProviderName`'s margin.

### 2. The observed price surface (August 2026 snapshot)

Every figure below is a dated snapshot with a stated basis. **Re-pull before you quote it.** The durable content is the *shape* — the width of the ranges and the reasons for them — not the numbers.

| SKU | Basis | Observed range | Cohort median | Notes |
|---|---|---|---|---|
| H100 80GB | 1-yr reserved | **$1.45 – $2.99** (Bronze $1.45–2.00; Silver $2.10–2.99) | ~$3.15 across the whole rental cohort | The Bronze→Silver spread alone exceeds 2× for identical silicon |
| H100 80GB | on-demand | **$2.19 – $11.06** | ~$4.17 dedicated clouds / **~$7.89 hyperscalers** | Matched-basis hyperscaler premium ≈ **1.9×** |
| H200 141GB | reserved / listed | **$2.30 – $13.78** | ~$4.11 | Low end FluidStack; high end Azure list |
| B200 | on-demand | **$4.95 – $18.00** | — | Launch premium; wide and unstable |
| H100 | spot | ~41% below on-demand (observed) | — | FOCUS `PricingCategory = Dynamic` |

Two market facts to carry with the table. First, the **H100 cohort median fell from more than $7/hr in early 2024 to roughly $3.15/hr** by this pass — a figure that moved 55% in two years is not quotable undated. Second, the *same H100* spans roughly $0.80/hr (specialty spot) to well north of $90/hr (hyperscaler reserved multi-GPU bundles with everything attached). **"H100 costs $X/hr" is not a well-formed sentence.**

**Now the correction that matters most, and the one to internalise before you repeat the module's own headline.** The "3–6× gap" is a real, observable ratio — and it is a ratio between *deliberately mismatched bases*:

```
   WHICH GAP IS WHICH  (H100, August 2026 snapshot)

   Hyperscaler on-demand LIST      $7.89 (median)  ─┐
                                                    ├─ 5.4×  ← the "3–6×"
   Cheapest Bronze COMMITTED       $1.45           ─┘          headline
     ↑ mismatched on THREE fields at once:
       tier (Platinum/Gold vs Bronze), term (on-demand vs 1-yr),
       and bundle (all-in vs bare GPU-hour)

   Hyperscaler on-demand median    $7.89           ─┐
                                                    ├─ 1.89×  ← the MATCHED
   Dedicated-cloud on-demand median $4.17          ─┘          basis gap

   Silver 1-yr reserved            $2.10–2.99      ─┐
                                                    ├─ ~1.4×  ← matched tier
   Bronze 1-yr reserved            $1.45–2.00      ─┘          AND term,
                                                               different tier

   THE DISCIPLINE: state which comparison you ran. A memo that quotes
   "5.4× cheaper" without saying it compared a Bronze committed rate
   against a hyperscaler on-demand list price has measured a
   PROCUREMENT decision and reported it as a VENDOR conclusion.
```

The rest of this lesson is the machinery for turning the top row into the bottom rows.

### 3. Axis (a) — bundling and margin: what is actually inside a GPU-hour

A hyperscaler prices the GPU-hour as an all-in service. A neocloud sells GPU-hours and monetises the rest separately, or thinly, or not at all. Much of the sticker gap is that you are comparing a bundle to a component.

```
   ANATOMY OF A GPU-HOUR — what the number covers

   HYPERSCALER  p5-class on-demand, ~$7.89/GPU-hr median
   ┌────────────────────────────────────────────────────────────┐
   │ GPU silicon amortisation + financing              ~$0.9    │
   │ Power, cooling, facility                          ~$0.15   │
   │ Rail-optimised fabric (EFA/IB), NICs, switches    ~$0.06   │  bundled
   │ Integrated block/object storage baseline          ~$0.1    │  — you
   │ Control plane: VPC, IAM, EKS, monitoring, quotas   ~$0.2    │  cannot
   │ Support organisation, SLA, on-call, RMA           ~$0.2    │  unbundle
   │ Global footprint, capacity buffer, idle inventory  ~$0.5    │  these
   │ ─────────────────────────────────────────────────────────  │
   │ Adjacent-service + platform margin           the remainder │
   └────────────────────────────────────────────────────────────┘

   TIER-1 NEOCLOUD  committed, ~$2.10/GPU-hr
   ┌────────────────────────────────────────────┐
   │ GPU silicon amortisation + financing  ~$0.86│
   │ Power, cooling, facility              ~$0.13│  in the rate
   │ Rail-optimised IB (reserved tier only)~$0.06│
   │ Thin control plane, K8s/Slurm         ~$0.05│
   │ Margin                                 rest │
   └────────────────────────────────────────────┘
        +  parallel filesystem        SEPARATE SKU   ~$0.20–0.35/GPU-hr
        +  egress (if metered)        SEPARATE       $0 – $0.10/GB
        +  support tier above basic   SEPARATE       negotiated
        +  YOUR on-call absorbing     NOT ON ANY     $0.10–0.20/GPU-hr
           node failures              INVOICE           (§8)

   The bracket contents are ILLUSTRATIVE decomposition parameters, not
   published figures — §4 derives the silicon/power/financing lines from
   first principles so you can re-run them with your own inputs. What is
   NOT illustrative is the structure: the neocloud rate has four lines
   living outside it, and three of them cost real money.
```

The operational instruction is simple and it is the top pitfall in this lesson: **re-bundle before you compare.** Every item the neocloud quote omits either has to be priced back in or explicitly declared not needed for this workload.

### 4. Axis (b) — capital structure and depreciation, built from the bottom

This is the axis where genuine, bankable advantage lives, and it is the one most people wave at rather than compute. Build the cost-to-serve floor from parameters you can source or estimate, then see what it explains.

**The model.** Everything is per 8-GPU HGX node, converted to $/GPU-hour at the end.

```
   INPUTS (parameterise all of these — the values shown are illustrative
   2026 mid-market estimates, not published figures)

     K     node capital cost, 8× H100 SXM HGX      = $280,000   [range 200k–400k]
     S     salvage fraction at end of life          = 0.10
     L     depreciation life, years                 = 6          [4 / 5 / 6]
     i     blended cost of capital (GPU-backed debt)= 12%/yr     [8–18%]
     P     node power draw incl. CPU/NIC/fans       = 10.2 kW    [8×700W + ~4.6kW]
     PUE   facility power usage effectiveness       = 1.30       [1.1–1.5]
     e     electricity                              = $0.08/kWh  [0.04–0.15]
     N     fabric capex per GPU (NIC + switch share)= $3,000     [2,000–4,500]
     F     facility, staff, storage, ops per node   = $1,800/mo  [parameter]
     A     fleet availability (node up and sellable)= 0.95
     U     revenue utilisation (of available hours) = 0.90

   DERIVED — per node, per calendar hour (8760 h/yr)

     depreciation = K(1−S) / (L × 8760)
                  = 280,000 × 0.90 / (6 × 8760)
                  = 252,000 / 52,560                    = $4.794 /node-hr

     financing    = i × K × (average outstanding ≈ 0.55) / 8760
                  = 0.12 × 280,000 × 0.55 / 8760        = $2.110 /node-hr

     power+cooling= P × PUE × e
                  = 10.2 × 1.30 × 0.08                  = $1.061 /node-hr

     fabric amort = 8 × N / (L × 8760)
                  = 8 × 3,000 / 52,560                  = $0.457 /node-hr

     facility+ops = F / 730                             = $2.466 /node-hr
                                                          ─────────────
     TOTAL COST TO SERVE, per node-hour of CALENDAR      = $10.888

   CONVERT to $/GPU-hour SOLD (only A×U of calendar hours earn revenue)

     per GPU-hr calendar = 10.888 / 8                    = $1.361
     per GPU-hr sold     = 1.361 / (0.95 × 0.90)
                         = 1.361 / 0.855                 = $1.592
```

**Read what that produces.** A cash cost-to-serve around **$1.59/GPU-hour sold**, before SG&A and before any margin. Now compare it to the observed price surface in §2: the cheapest Bronze 1-year committed rates sit at **$1.45–2.00**. **The Bronze floor is approximately cost.** That is the single most useful thing this model tells you, and it has two consequences:

1. The cheap end of the market cannot get much cheaper by cutting margin, because there is barely any margin there. It can only get cheaper by cutting *inputs* — older silicon, cheaper power, no redundancy, no fabric, no support, no spares. Which is exactly what a Bronze rating describes, and exactly why ClusterMAX's named Bronze gaps are inconsistent support, subpar networking performance, unclear SLAs, and limited orchestration integration.
2. Any quote materially *below* the floor is telling you something about the inputs. Consumer-grade cards in someone's garage, a provider burning investor money for market share, or capacity that is genuinely surplus (someone else's committed-but-idle hours resold). All three are real; all three have different risk profiles; none of them is "the same GPU, cheaper."

**Now the depreciation lever, which is the part that changes a sticker.** Hold everything else fixed and vary `L`:

| Depreciation life `L` | Depreciation $/node-hr | Total cost-to-serve $/node-hr | $/GPU-hr sold | Δ vs 6-year |
|---|---|---|---|---|
| 4 years | $7.192 | $13.286 | $1.942 | **+22%** |
| 5 years | $5.753 | $11.847 | $1.732 | +9% |
| **6 years** | **$4.794** | **$10.888** | **$1.592** | baseline |
| 7 years | $4.109 | $10.203 | $1.492 | −6% |

Moving from a six-year to a four-year assumed life raises the depreciation *line* by 50% and the total floor by 22%. CoreWeave's FY2025 Form 10-K sets the estimated useful life of technology equipment including GPUs at **six years**, straight-line; some competitors use shorter schedules. **The depreciation schedule is an input to the price, not a neutral accounting fact** — and a provider on a longer schedule can quote a lower rate for identical hardware while reporting the same margin.

The corresponding tension, which analysts scrutinise and you should understand: CoreWeave's weighted-average customer contract duration is roughly **five years** (up from ~4 at the March 2025 S-1) against that **six-year** depreciation life. The asset has to earn beyond the contracted window for the economics to close. That is normal infrastructure-business shape, not an accusation — and it tells you what your provider needs from the back half of the asset's life. It is also why re-contracting older silicon at reduced rates rather than retiring it is the expected behaviour, which is good news if you are shopping for A100-class capacity and relevant context if you are being asked to commit five years to today's flagship.

**And the risk side of the same coin.** The financing line above is $2.110/node-hour — roughly 19% of total cost-to-serve — and it exists because the silicon was bought with GPU-backed debt. That is a genuine structural cost *disadvantage* against a hyperscaler funding capex from operating cash flow, and it is the reason single-SKU providers must run high utilisation: the debt service does not pause when the GPUs are idle. It is also why their contracts are take-or-pay (lesson 06): the take-or-pay obligation is how the utilisation risk gets transferred to you.

### 5. Axis (c) — interconnect: the axis that silently voids the savings

A single H100 is fungible. A *training cluster* is not. Multi-node training needs rail-optimised InfiniBand (NDR 400G-class) or an equivalent RoCE fabric with the right topology, because the all-reduce that synchronises gradients every step is a bandwidth-and-latency problem, not a compute problem.

The mechanism, stated so you can reason about cases this lesson does not cover: a data-parallel step must sum every gradient across every rank before the optimizer can advance. Ring all-reduce moves `2(N−1)/N × M` bytes per rank for a message of size `M` across `N` ranks — asymptotically `2M` regardless of `N`, which is why the algorithm scales at all. But that `2M` has to cross the *slowest* link in the ring. On a 70B-parameter model in BF16, the gradient buffer is ~140 GB; even with bucketing and overlap, an all-reduce that would take ~0.7 s on 400 Gb/s rail-optimised IB takes ~28 s on a 10 Gb/s general-purpose Ethernet path. The GPUs are idle for that entire window. **Your bill does not go up; your MFU collapses, and the $/useful-work figure follows.** Nobody gets an alert. The symptom appears weeks later as "training is slower than we projected."

Practically, ask three questions of every quote:

1. **Is non-blocking, rail-optimised IB (or equivalent) included in this rate, sold separately, or unavailable?** Several neoclouds bundle it in the *reserved cluster* tier only, and sell a pure on-demand tier over commodity Ethernet. The two are not substitutable products at any price for a multi-node job.
2. **What is the blocking ratio and the topology?** "InfiniBand" with a 4:1 oversubscribed spine is not the same product as a non-blocking rail-optimised fabric, and the difference shows up exactly under the all-reduce load you care about.
3. **Can you run a NCCL all-reduce bandwidth test on the actual allocation before signing?** This is the only answer that is not marketing. A `nccl-tests` `all_reduce_perf` run across the number of nodes you intend to use, at the message sizes your model produces, is a two-hour test that settles the question.

**Costing it back in.** Fabric capex amortised is small — §4's model puts it at $0.457/node-hour, about **$0.057/GPU-hour**, roughly 3.6% of cost-to-serve. That asymmetry is the whole point: **the fabric is cheap to provide and catastrophic to omit.** A quote that saves you $2.00/GPU-hour by dropping a $0.06/GPU-hour component has not given you a discount; it has given you a different product.

### 6. Axis (d) — storage and egress, with the crossover computed

**Storage.** High-throughput parallel storage (VAST, WEKA, Lustre, or the provider's own) for checkpoint and dataset I/O is frequently a separate SKU on a neocloud and bundled on a hyperscaler. Price it per GPU-hour so it lands on the same axis:

```
   storage_addon ($/GPU-hr) = monthly_storage_$ ÷ monthly_fleet_GPU-hours
```

A 1 PB parallel filesystem at, say, $25–70/TB-month lists at $25,000–70,000/month. On a 64-GPU fleet at 80% utilisation (`64 × 730 × 0.80 = 37,376` GPU-hours/month) that is **$0.67–1.87/GPU-hour** — larger than the entire fabric line and, at the top of the range, comparable to the GPU rate itself on a cheap Bronze quote. Storage is not a rounding error on small fleets. Note the shape: storage cost is *fixed monthly* while GPU-hours scale with utilisation, so the per-GPU-hour storage addon **rises as your fleet gets less busy** — the opposite direction from intuition.

**Egress.** The classic trap, and the one where popular advice is quantitatively wrong. As of this snapshot, **Lambda, Nebius, Crusoe and SF Compute have offered free egress; CoreWeave, RunPod, Vast.ai and all three hyperscalers meter it**, with AWS's first-tier internet data-transfer-out rate around **$0.09/GB** (tiered down at volume, with a monthly free allowance).

Compute the crossover rather than quoting the folk rule:

```
   egress_addon ($/GPU-hr) = E_GB × rate_$/GB ÷ fleet_GPU-hours

   For a 64-GPU fleet at 80% utilisation → 37,376 GPU-hr/month:

       per TB/month of egress at $0.09/GB:
           1024 GB × $0.09 ÷ 37,376 = $0.002466 /GPU-hr per TB/month

   ⟹ to move the effective rate by $0.50/GPU-hr you need
           0.50 ÷ 0.002466 ≈ 203 TB/month of egress on 64 GPUs
     to move it by $1.00/GPU-hr you need ≈ 406 TB/month.
```

**Now sanity-check that against real workload shapes**, because this is where the popular claim ("free egress offsets a $0.50–1.00/hr premium") falls apart:

| Workload | Monthly egress on a 64-GPU fleet | Egress addon at $0.09/GB | Verdict |
|---|---|---|---|
| Text inference serving (tokens out) | ~0.5–5 TB — a 1,000-token response is ~4 KB; 200 TB would be ~50 billion responses | $0.001–0.012/GPU-hr | **Negligible. Free egress is not a reason to switch.** |
| Training with checkpoint replication to another cloud | 40–150 TB (a 70B FP8 checkpoint ~70 GB, replicated hourly ≈ 50 TB/mo) | $0.10–0.37/GPU-hr | Material; worth a line but rarely decisive |
| Dataset staging out / cross-cloud data lake | 100–500 TB | $0.25–1.23/GPU-hr | **Decisive.** Free egress can flip the ranking. |
| Image/video generation serving | 200 TB–1 PB+ | $0.49–2.5+/GPU-hr | **Dominant.** Model it before anything else. |

The correction worth carrying: **egress is a first-order axis for media generation and cross-cloud data movement, and a rounding error for text inference.** Advice that does not name the workload shape is advice you cannot apply.

**Support.** Hyperscaler support is a percentage of spend on a declining marginal schedule — observed bands in the low single digits at the top: roughly **3% of monthly charges above the ~$250k tier**, rising to 5%, 7%, and ~9–10% in lower bands, with plan minimums. (AWS restructured its support tiers and minimums during 2025–2026; the *structure* — marginal percentage bands declining with spend — is the durable part, the exact percentages and minimums are not.) At GPU-fleet spend levels you are almost always in the 3% marginal band, so:

```
   support_addon ≈ 0.03 × base_rate       (hyperscaler, large spend)
                 = 0.03 × $4.90 = $0.147 /GPU-hr
```

Neocloud support is typically a negotiated line or bundled thinly, and the honest way to price the gap is **not** as a support fee — it is as your own engineering time, which appears on no invoice. See §8.

### 7. Axis (e) — utilisation and term: the normalisation lesson 06 already gave you

The headline low neocloud rate almost always requires a long commitment (6–36 months, take-or-pay) or is spot. Lesson 06 owns the mechanics; the point here is that **a committed rate and an on-demand rate are not comparable numbers** and must be brought onto a common basis before ranking.

The clean way to do it, avoiding the double-counting that trips people up: compute **cost per GPU-hour actually delivered to a workload**.

```
   For an ON-DEMAND offer, you pay only for hours consumed:
        cost_per_delivered_hr  =  fully_loaded_rate  ÷  effective_availability

   For a COMMITTED offer, you pay for all committed hours regardless:
        cost_per_delivered_hr  =  fully_loaded_rate  ÷  (effective_availability
                                                          × commitment_utilisation u)

   Do NOT divide the on-demand offer by u. It has no commitment to strand.
   Dividing both by the same u is the single most common error in these
   sheets, and it makes the committed offer look artificially good by
   applying a penalty symmetrically that only one side actually bears.
```

Lesson 06's break-even carries straight over: the committed offer beats its own on-demand alternative when `u > 1 − d`. Here we are comparing across *vendors*, so the ranking can flip at a different `u` — and the whole point of the sheet is to find that `u` and put it in the memo as a floor.

### 8. Axis (f) — reliability, SLA and support, priced properly

**ClusterMAX is the independent instrument for this axis**, and it exists precisely because reliability is invisible from a pricing page. SemiAnalysis scores providers across **ten criteria — security, lifecycle, orchestration, storage, networking, reliability, monitoring, pricing, partnerships, availability — into five tiers: Platinum, Gold, Silver, Bronze, UnderPerform.** In the ClusterMAX 2.0 cycle, SemiAnalysis tracked more than **209** GPU cloud providers, hands-on evaluated **84**, drew on **140+** customer surveys, and awarded **37** medallion ratings. Platinum required roughly **90+/100 in nearly every category**, and **only CoreWeave achieved it**. Bronze is defined as meeting the minimum criteria and still being recommended, with named common gaps: inconsistent support, subpar networking performance, unclear SLAs, and limited Kubernetes/Slurm integration.

**The silicon is identical across tiers.** What differs is whether it stays up, whether the interconnect delivers under load, and whether a ticket gets answered.

**Single-node effective availability.** Divide the fully-loaded rate by the fraction of paid hours the capacity is actually usable:

```
   cost per USABLE GPU-hour = fully_loaded_rate ÷ effective_availability

   A 96%-available fleet at $2.00       →  2.00 / 0.96  = $2.083
   A 99.9%-available fleet at $2.081    →  2.081 / 0.999 = $2.083   ← equal

   ⟹ 96% availability at $2.00 is equivalent to 99.9% availability
     at $2.08. The LESS reliable fleet must be CHEAPER by ~4% to tie.
```

**This corrects the previous version of this lesson**, which asserted the equivalent price was ~$1.93 — obtained by multiplying by `0.96/0.999` instead of dividing. The sign of the correction matters: an availability shortfall makes a cheap fleet effectively *dearer*, and a formula that moves the other way would recommend the unreliable vendor.

**Multi-node synchronous availability, which is where the axis actually bites.** For a single serving replica, 96% availability costs you 4%. For a 64-node synchronous training job it costs you far more, because *any* node failure kills the step and forces a restart from the last checkpoint. Node failures are approximately independent, so the job-level mean time between failures divides:

```
   M_job  =  M_node / N          (N nodes, independent failures)

   M_node = 20,000 h (a well-run Gold/Platinum fleet), N = 64
        →  M_job = 312 h ≈ 13 days between job-killing events

   M_node =  5,000 h (a Bronze fleet with thin RMA and no health gating)
        →  M_job =  78 h ≈ 3.3 days

   Now feed that into lesson 06's goodput model, with δ = 90 s checkpoint
   write and ρ = 420 s recovery:

     Δ* = sqrt(2 δ M_job)

     Gold  : M_job = 1.123e6 s → Δ* = sqrt(2·90·1.123e6) = 14,230 s ≈ 3.95 h
             g = (1 − 90/14230)(1 − (7115+420)/1.123e6)
               = 0.99368 × 0.99329 = 0.9870        →  98.7% goodput

     Bronze: M_job = 2.808e5 s → Δ* = sqrt(2·90·2.808e5) =  7,110 s ≈ 1.98 h
             g = (1 − 90/7110)(1 − (3555+420)/2.808e5)
               = 0.98734 × 0.98584 = 0.9733        →  97.3% goodput

   Goodput gap = 1.4 percentage points, so on this axis alone the Bronze
   fleet must be ~1.4% cheaper to tie.  That is SMALL — and it is small
   *because* checkpointing works.
```

The honest conclusion is more interesting than "reliability is expensive": with a competent checkpoint-and-resume loop, a 4× worse node MTBF costs you only about 1.4 points of goodput on a 64-node job. **Reliability tier is not primarily a goodput tax — it is a human-time tax and a tail-risk tax.** The 4× MTBF difference means 4× as many 2 a.m. pages, 4× as many nodes to cordon and RMA, and a far worse tail: a fleet that can lose 8 nodes at once and take three days to replace them is a different risk than one that has hot spares. That cost is real and it lands on your headcount, not the invoice.

**So price it as headcount.** If a thinner-support provider costs you 0.4 FTE of platform on-call absorbing node failures, at a $220k fully-loaded engineer cost:

```
   self_SRE_addon = 0.4 × 220,000 ÷ 12 ÷ 37,376 GPU-hr/mo = $0.196 /GPU-hr
```

That is the line that never appears on any invoice and routinely exceeds the fabric and support lines combined. On a small fleet it is much worse: the same 0.4 FTE spread over a 16-GPU fleet is **$0.78/GPU-hour**, which is why sub-32-GPU fleets are usually better off on a higher tier even at a worse sticker. **The self-SRE addon scales inversely with fleet size, and that alone determines the right answer for small teams.**

### 9. The normalisation procedure

Put it together as a repeatable pipeline, with the three columns that make the gap's collapse or survival visible.

```
   THE NORMALISATION PIPELINE

   ┌──────────────┐
   │  STICKER     │  base_rate, as quoted            column 1: SNAPSHOT
   │  $/GPU-hr    │  + tier + term + date            ← never compare here
   └──────┬───────┘
          │  ADD BACK everything the quote omits (per GPU-hour):
          ├─ + interconnect_addon    0 if bundled; else amortised fabric
          ├─ + storage_addon         monthly parallel-FS $ ÷ fleet GPU-hrs
          ├─ + egress_addon          expected GB × $/GB ÷ fleet GPU-hrs
          ├─ + support_addon         support tier $ ÷ fleet GPU-hrs
          ├─ + self_SRE_addon        FTE fraction × loaded cost ÷ GPU-hrs
          ▼
   ┌──────────────┐
   │ FULLY-LOADED │                                  column 2: COMPARABLE
   │  $/GPU-hr    │  ← now both quotes cover the same product
   └──────┬───────┘
          │  DIVIDE by the yields:
          ├─ ÷ effective_availability   (both offers — everyone has downtime)
          ├─ ÷ commitment_utilisation u (COMMITTED offers only — an
          │                              on-demand offer strands nothing)
          ▼
   ┌──────────────────────┐
   │ COST PER DELIVERED   │                          column 3: DECISION
   │ GPU-HOUR             │  ← the only number you may rank on
   └──────┬───────────────┘
          │  (lesson 05 continues from here)
          ├─ ÷ device utilisation s  → $/useful-GPU-hour
          └─ ÷ app counter           → $/1M tokens, $/run
```

**The decision rule, in one sentence: the sticker gap is your hypothesis; the cost-per-delivered-GPU-hour gap is your answer.** If the gap survives normalisation, bank it. If it collapses, you found the bundling. And whatever survives, attach a **utilisation floor** — the `u` at which the ranking flips — because that floor is the load-bearing assumption in the recommendation and the thing you will monitor for the next three years.

### 10. The decision tree

```
   NEOCLOUD vs HYPERSCALER — walk the branches, do not skip

   START: what is the workload shape?
     │
     ├─ MULTI-NODE SYNCHRONOUS TRAINING?
     │    │
     │    ├─ Is non-blocking rail-optimised IB/RoCE in the quoted tier?
     │    │    NO ──▶ ✖ DISQUALIFIED at any price. Not a discount;
     │    │            a different product. Re-quote the fabric tier.
     │    │    YES ─▶ demand a nccl-tests all_reduce_perf run on the
     │    │            actual allocation at your model's message sizes
     │    │            BEFORE signing. Continue.
     │    │
     │    └─ Node MTBF and RMA turnaround known?
     │         NO ──▶ ✖ cannot compute M_job = M_node/N. Ask, or assume
     │                   the Bronze figure and price the pessimism.
     │         YES ─▶ compute goodput (lesson 06 model). Continue.
     │
     ├─ SINGLE-NODE / EMBARRASSINGLY PARALLEL (sweeps, batch inference)?
     │    fabric is NOT load-bearing → the commodity/marketplace tier
     │    becomes genuinely admissible. Continue on price.
     │
     └─ LATENCY-SERVING?
          spot is disqualified (lesson 06); effective availability is a
          direct SLO input, not a cost line. Weight tier heavily.
     │
     ▼
   FLEET SIZE?
     ├─ < ~32 GPUs ──▶ self_SRE_addon dominates ($0.78/GPU-hr at 16 GPUs
     │                  for 0.4 FTE). Higher tier usually wins DESPITE a
     │                  worse sticker. Do the arithmetic; do not assume.
     └─ > ~128 GPUs ─▶ self_SRE_addon amortises to noise; structural
                        cost-to-serve advantage (axis b) dominates.
     │
     ▼
   EGRESS SHAPE?
     ├─ Text inference ─────▶ egress ≈ 0. Ignore this axis.
     ├─ Checkpoint replication ▶ 40–150 TB/mo → $0.10–0.37/GPU-hr. A line.
     └─ Media gen / cross-cloud data ▶ 200 TB–1 PB+ → can FLIP the ranking.
                                        Model it first, not last.
     │
     ▼
   COMMITMENT APPETITE?
     ├─ Can you honestly forecast the demand floor 12+ months?
     │    NO ──▶ hyperscaler on-demand or short neocloud terms. The
     │            committed rate is not available to you at any u you
     │            can defend (lesson 06: u > 1 − d).
     │    YES ─▶ committed neocloud is on the table. Compute the u at
     │            which the ranking flips; that is your memo's floor.
     │
     ▼
   NORMALISE BOTH → three columns → rank on column 3 ONLY
   → recommendation + explicit utilisation floor + date + tier + term
```

## Perspectives

**Buyer / procurement view.** Your job is not to find the cheapest sticker; it is to refuse to rank anything until both offers sit on the same basis. The sticker gap is a *hypothesis* — "maybe vendor N is 3× cheaper" — and the fully-loaded, availability-adjusted, delivered-hour gap is the answer. Treat every quote as an invitation to ask "what is not in this number," and do not let a procurement clock (vendor urgency, end-of-quarter discounting) short-circuit the normalisation. The cheapest thing you can do to protect a nine-figure decision is spend two days building a spreadsheet.

**Networking / HPC view.** From the fabric engineer's chair, axis (c) is the whole story and the asymmetry is the argument: rail-optimised fabric is ~3.6% of cost-to-serve to provide and it is the difference between a training cluster and a pile of GPUs. A GPU-hour without it is not a discount on the same product; it is a cheaper product that shares a part number with the one you need. The failure mode is uniquely nasty because it is silent — the bill does not go up, no alert fires, and the *value received per dollar* falls by whatever factor your all-reduce slows down. The only defensible check is an all-reduce bandwidth test on the actual allocation.

**Financial-structure view.** From a capital-markets seat, axis (b) is where advantage that survives normalisation lives, and it is visible in filings rather than marketing. Depreciation schedule, debt-financed silicon, single-SKU focus, and the forced utilisation that servicing that debt requires are real cost-to-serve differences. But note the direction of each: the longer depreciation life is a genuine advantage (22% lower floor at six years than four), while the debt financing is a genuine *disadvantage* (19% of cost-to-serve in the §4 model, against a hyperscaler funding capex from operating cash flow). The discipline is distinguishing structural advantage from bundling, and then distinguishing structural *advantage* from structural *difference*.

**Reliability / SRE view.** Effective availability is a cost line, not a footnote — but the interesting part of the analysis is that with competent checkpointing, a 4× worse node MTBF costs only ~1.4 points of goodput on a 64-node job. The real cost of a thinner-support provider is not lost compute; it is that "who replaces the failed node at 2 a.m." is *you*, and that expense scales inversely with fleet size. At 16 GPUs a 0.4-FTE on-call burden is $0.78/GPU-hour, which dwarfs every other adjustment in the sheet. Small teams should read that number first.

**Standards / data view.** FOCUS 1.3's split of `ServiceProviderName` from `HostProviderName` exists because the market this lesson describes has layers: a managed inference platform's cost row names one entity as the service provider and a different one as the host. When you normalise a managed-platform quote you are inferring the host's rate plus the platform's margin, and the schema gives you a place to record both — which matters when the same GPU appears in your cost lake twice under two vendors' names.

## Real-world use cases

- **SemiAnalysis, ClusterMAX rating system (v1 March 2025; v2.0 November 2025; live ratings at clustermax.ai).** What it shows: the independent instrument for axis (f). Ten scored criteria (security, lifecycle, orchestration, storage, networking, reliability, monitoring, pricing, partnerships, availability) mapped to five tiers (Platinum / Gold / Silver / Bronze / UnderPerform). In the 2.0 cycle SemiAnalysis tracked **209+** providers, hands-on evaluated **84**, drew on **140+** customer surveys, and awarded **37** medallion ratings, with **CoreWeave the sole Platinum**. The concrete lesson for this decomposition: a cheap sticker and a mature platform are different claims, and the tiering is a market-structure fact rather than a pricing gimmick — a Bronze box and a Platinum box running the identical H100 SXM chip are genuinely different purchases.
- **CoreWeave, Form S-1 (March 2025) and FY2025 Form 10-K.** What it shows: axis (b) with real numbers. Six-year straight-line depreciation on technology equipment including GPUs; weighted-average contract duration ~4 years at S-1 and ~5 years by FY2025; ~$15.1B remaining performance obligations at 31 December 2024 growing to ~$66.8B revenue backlog by 31 December 2025 on ~$5.13B of 2025 revenue; take-or-pay contracts ~96% of 2024 revenue. The depreciation-life-versus-contract-duration gap is the structural tension §4 quantifies, and the take-or-pay share is how utilisation risk gets transferred to the buyer.
- **CoreWeave A100 re-contracting to 2029.** What it shows: the practical consequence of the six-year schedule. Older silicon gets re-contracted at reduced rates rather than retired, because the asset still has depreciated life to earn against. For a buyer this is the mechanism that makes previous-generation capacity cheap, and it is the reason "the GPU will be worthless in three years" is a weaker argument than it sounds.
- **Free-egress versus metered-egress provider split.** What it shows: axis (d) as an actual fork in the market rather than a footnote. As of this snapshot Lambda, Nebius, Crusoe and SF Compute have offered free egress while CoreWeave, RunPod, Vast.ai and all three hyperscalers meter it (AWS first-tier internet DTO around $0.09/GB). The arithmetic in §6 turns that into a decision rule instead of a slogan: ~203 TB/month on a 64-GPU fleet to move the effective rate by $0.50/GPU-hour, which text inference will never reach and media generation exceeds routinely.
- **AWS `p5` (H100) and Google Cloud A3 rate cards.** What they show: the official hyperscaler anchors for the bundled side of every normalisation sheet — the "what is included" baseline (integrated fabric, storage, IAM/VPC control plane, support tier) that axis (a) asks you to itemise before comparing against a neocloud's unbundled price. They are also the reminder that the hyperscaler number in your sheet should be a *committed* rate if you are comparing against a committed neocloud rate; comparing on-demand list to committed is the mismatch §2 is about.

## Worked example

Two real-shaped quotes for a **64-GPU H100 training and fine-tuning workload**, target 80% commitment utilisation, **multi-node — InfiniBand is load-bearing**. The workload replicates checkpoints to a hyperscaler-hosted evaluation pipeline: **40 TB/month egress**. All figures are an **August 2026 snapshot with illustrative vendor shapes** — substitute your own quotes.

**Fleet denominators, computed once:**

```
   fleet GPU-hours/month at 80% of 64 GPUs = 64 × 730 × 0.80 = 37,376 GPU-hr
   egress volume                            = 40 TB = 40,960 GB
```

**Offer H — hyperscaler, 1-year committed.** $4.90/GPU-hr. EFA/IB-class fabric bundled. Integrated storage baseline bundled (a separate parallel FS is not required for this workload's checkpoint rate). Support at the 3% marginal band. Egress metered at ~$0.09/GB. Effective availability 99.9%, node MTBF ~20,000 h. Self-SRE burden ~0.05 FTE (vendor handles hardware).

**Offer N — tier-1 neocloud, 12-month take-or-pay.** $2.05/GPU-hr base. Rail-optimised IB included, but **only in the reserved cluster tier**, which is the tier quoted. Parallel filesystem: **+$0.25/GPU-hr** (a separate SKU). Egress free. Support standard, thinner SLA; effective availability 97.5%, node MTBF ~8,000 h. Self-SRE burden **0.4 FTE** at $220k fully loaded.

**Step 1 — compute each addon on the same per-GPU-hour axis.**

```
   EGRESS
     H:  40,960 GB × $0.09 = $3,686.40/mo  ÷ 37,376 = +$0.0986 /GPU-hr
     N:  free                                        = +$0.0000 /GPU-hr

   STORAGE
     H:  bundled                                     = +$0.0000 /GPU-hr
     N:  quoted per-GPU-hour SKU                     = +$0.2500 /GPU-hr

   SUPPORT
     H:  0.03 × $4.90                                = +$0.1470 /GPU-hr
     N:  bundled thin tier                           = +$0.0000 /GPU-hr

   SELF-SRE (never on any invoice)
     H:  0.05 × 220,000 ÷ 12 ÷ 37,376                = +$0.0245 /GPU-hr
     N:  0.40 × 220,000 ÷ 12 ÷ 37,376                = +$0.1962 /GPU-hr

   INTERCONNECT
     H:  bundled                                     = +$0.0000 /GPU-hr
     N:  in the quoted reserved tier                 = +$0.0000 /GPU-hr
         ⚠ if you had quoted N's on-demand tier instead, this workload
           is DISQUALIFIED, not "+$X" — see §5 and the decision tree.
```

**Step 2 — the three-column sheet.**

| Line | Offer H (hyperscaler, 1-yr committed) | Offer N (neocloud, 12-mo take-or-pay) |
|---|---|---|
| Base rate (sticker) | **$4.9000** | **$2.0500** |
| + Interconnect | 0 (bundled) | 0 (in reserved tier) |
| + Storage | 0 (bundled) | +$0.2500 |
| + Egress | +$0.0986 | 0 (free) |
| + Support | +$0.1470 | 0 |
| + Self-SRE | +$0.0245 | +$0.1962 |
| **Fully-loaded $/GPU-hr** | **$5.1701** | **$2.4962** |
| ÷ effective availability | ÷0.999 = $5.1753 | ÷0.975 = $2.5602 |
| ÷ commitment utilisation (0.80) | ÷0.80 = **$6.4691** | ÷0.80 = **$3.2003** |
| **Cost per delivered GPU-hour** | **$6.4691** | **$3.2003** |

Both offers are committed here, so both bear the utilisation divisor — that symmetry is correct in this case. (If Offer H were on-demand, it would **not** take that divisor, and the gap would narrow to $5.1753 vs $3.2003.)

**Step 3 — read what happened to the gap.**

```
   sticker ratio                4.9000 / 2.0500 = 2.39×
   fully-loaded ratio           5.1701 / 2.4962 = 2.07×
   delivered-hour ratio         6.4691 / 3.2003 = 2.02×

   The gap moved from 2.39× to 2.02×. It did NOT collapse.
   Where did the 0.37× go?
       storage    −0.25  (N absorbs a real unbundled cost)
       self-SRE   −0.17  (N's thin support becomes your headcount)
       egress     +0.10  (H pays, N does not)
       support    +0.15  (H's percentage-of-spend)
       availability −0.06 net (N's 2.4 pp shortfall costs it more)

   RESIDUAL 2.02× = genuine structural cost-to-serve advantage
       (axis b: capital structure, single-SKU focus, longer depreciation
        schedule) + the bundle H sells that this workload does not use.
```

**Step 4 — the goodput check, because this is multi-node training.**

```
   N = 8 nodes (64 GPUs at 8/node)

   Offer H:  M_node = 20,000 h → M_job = 2,500 h = 9.0e6 s
             Δ* = sqrt(2 × 90 × 9.0e6) = 40,250 s ≈ 11.2 h
             g  = (1 − 90/40250)(1 − (20125+420)/9.0e6)
                = 0.99776 × 0.99772 = 0.9955

   Offer N:  M_node =  8,000 h → M_job = 1,000 h = 3.6e6 s
             Δ* = sqrt(2 × 90 × 3.6e6) = 25,456 s ≈ 7.1 h
             g  = (1 − 90/25456)(1 − (12728+420)/3.6e6)
                = 0.99646 × 0.99635 = 0.9928

   goodput-adjusted cost per USEFUL GPU-hour:
       H:  6.4691 / 0.9955 = $6.4983
       N:  3.2003 / 0.9928 = $3.2235
       ratio 2.02× — unchanged to two decimals.

   Finding: at 8 nodes with working checkpointing, the MTBF difference
   is worth 0.27 percentage points. Reliability is NOT the axis that
   decides this comparison; it is the axis that decides how many nights
   your on-call gets paged, and that cost is already in the self-SRE line.
```

**Step 5 — sensitivity, which is what makes it defensible.**

| Scenario | H delivered $/GPU-hr | N delivered $/GPU-hr | Winner | Margin |
|---|---|---|---|---|
| **Base case, `u = 0.80`** | $6.469 | $3.200 | **N** | 2.02× |
| `u` slips to 0.55 | $9.400 | $4.539 | **N** | 2.07× |
| `u` slips to 0.55, H is **on-demand** (no strand) | $5.175 | $4.539 | **N** | 1.14× |
| `u` slips to 0.40, H on-demand | $5.175 | $6.241 | **H** | 1.21× |
| Fleet is 16 GPUs, not 64 (self-SRE 4×) | $6.607 | $4.111 | **N** | 1.61× |
| Fleet is 16 GPUs **and** `u = 0.50`, H on-demand | $5.286 | $5.185 | **N** | 1.02× — a tie |
| N's on-demand tier quoted (no IB) | $6.469 | ✖ disqualified | **H** | n/a |
| Egress rises to 300 TB/mo | $7.088 | $3.200 | **N** | 2.22× |

**Read the table as three findings, not one.**

*Finding 1, robust:* at the planned 80% utilisation the neocloud advantage is **~2×** and it survives every adjustment. The 2.39× sticker was not wildly misleading here — but that is a fact you *established*, not one you assumed, and the 0.37× that did dissolve was real money that would have been a surprise.

*Finding 2, the floor:* the ranking flips only in a specific corner — **when N is committed and H is on-demand and realised utilisation falls below roughly 45%**. That is the memo's load-bearing assumption. Lesson 06's rule gives the same answer from the other direction: with `d = 1 − 2.05/4.90 = 0.582`, N's commitment needs `u > 0.418` just to beat its own on-demand alternative.

*Finding 3, the trap that is not about price at all:* quoting N's cheaper **on-demand** tier disqualifies the workload entirely, because that tier is commodity Ethernet. It does not appear as a worse number; it appears as a stalled training run six weeks later. **The disqualification branch must be walked before any arithmetic.**

**Recommendation, written the way it should be handed over.**

> *Contract Offer N's **reserved cluster tier** (InfiniBand-inclusive) on a 12-month take-or-pay at $2.05/GPU-hr. Fully loaded — parallel filesystem, free egress, thin support absorbed as 0.4 FTE of platform on-call, 97.5% effective availability, 80% commitment utilisation — the cost per delivered GPU-hour is **$3.20 against $6.47** for the matched hyperscaler committed offer, a genuine 2.02× that survives full normalisation. The 2.39× sticker gap narrowed by 0.37× once storage and self-SRE were loaded in; the residual is structural cost-to-serve, not bundling. **Conditional on three floors, monitored monthly: (i) commitment utilisation ≥ 45%, below which a hyperscaler on-demand posture wins; (ii) the reserved tier's rail-optimised IB is contractually specified and validated by an `all_reduce_perf` run on the delivered allocation before the first payment; (iii) the 0.4-FTE on-call assumption holds — if the burden reaches 1.0 FTE the advantage falls to ~1.6× and should be renegotiated.** If the fleet drops below ~24 GPUs the self-SRE line dominates and this recommendation should be re-run. Pricing is an August 2026 snapshot at stated tier and term; re-verify before signing.*

## Practice

Feeds [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md).

Take two vendor quotes — one hyperscaler committed, one neocloud committed — for an H100 or H200 cluster, and build the normalisation sheet:

1. **Classify both sellers** into the §1 taxonomy and record `ServiceProviderName` and `HostProviderName` for each. If they differ (a managed platform), state whose margin you are seeing.
2. **Build the three-column sheet**: sticker (with tier, term, date), fully-loaded, and cost per delivered GPU-hour. Compute every addon on the per-GPU-hour axis with the fleet denominator shown explicitly. Include the **self-SRE line** — the sheet is incomplete without it, and it is the line most people omit.
3. **Compute the cost-to-serve floor** from §4's model with your own parameters for `K`, `L`, `i`, `P`, `PUE`, `e`, and `F`. State where each quote sits relative to the floor. If a quote is *below* your computed floor, say what that implies about the inputs (older silicon, subsidised pricing, resold surplus) rather than treating it as free money.
4. **Run the depreciation sensitivity**: re-derive the floor at `L = 4, 5, 6, 7` and report how much of the observed gap the schedule alone could explain.
5. **Compute the egress crossover** for your own fleet size and workload shape: the TB/month at which egress moves the effective rate by $0.25 and $0.50/GPU-hour, and where your actual volume sits.
6. **Run the multi-node availability model**: `M_job = M_node/N`, then lesson 06's `Δ* = sqrt(2δM)` and `g`, for both vendors' node MTBFs. Report the goodput-adjusted delivered rate and state whether reliability actually decides this comparison or merely changes the pager volume.
7. **Identify which axis explains the largest share of the gap**, and separate the part that survives normalisation from the part that dissolved.
8. **Write the one-paragraph procurement recommendation** with (a) the winner on cost per delivered GPU-hour, (b) an explicit utilisation floor, (c) a fabric-validation gate, and (d) at least one other named condition that would reverse it. Every dollar figure carries tier, term, and date.

**Acceptance criteria.** The sheet ranks only on column 3; the self-SRE line is present and non-zero for the thinner-support vendor; the fabric branch is walked as a disqualification, not a price adjustment; the recommendation carries a numeric utilisation floor and a date.

## Common pitfalls

1. **Comparing sticker rates without re-bundling.** The central thesis and the top pitfall: a $2.05 neocloud rate and a $4.90 hyperscaler rate are not the same product until storage, egress, support, self-SRE, and fabric are on the same axis. *Correction:* always produce the three-column sheet before recommending anything, and rank only on column 3.
2. **Quoting the headline "3–6× gap" without saying which bases it compares.** That ratio compares a Bronze *committed* rate against a hyperscaler *on-demand list* — mismatched on tier, term, and bundle simultaneously. On a matched on-demand basis the hyperscaler-versus-dedicated-cloud premium is closer to **1.9×**. *Correction:* name the two rows you divided, every time.
3. **Getting the availability adjustment backwards.** You **divide** by effective availability. A 96%-available fleet at $2.00 costs $2.083 per usable hour, equivalent to a 99.9% fleet at $2.081 — so the *less* reliable fleet must be *cheaper* to tie. Multiplying instead gives $1.93 and recommends the unreliable vendor. *Correction:* sanity-check the sign — lower availability must always make a quote look worse.
4. **Applying the utilisation divisor symmetrically to a committed and an on-demand offer.** Only committed capacity strands hours you did not use. Dividing both by `u` penalises the on-demand offer for exposure it does not have and flatters the committed one. *Correction:* divide by `u` only where there is a take-or-pay obligation.
5. **Treating a missing fabric as a price adjustment instead of a disqualification.** A quote without rail-optimised IB is not "the same thing minus $0.06/GPU-hour"; for multi-node synchronous training it is not the product at all. And the failure is silent — no alert, no billing surprise, just collapsed MFU weeks later. *Correction:* walk the disqualification branch first, and validate with `nccl-tests all_reduce_perf` on the delivered allocation before the first payment clears.
6. **Assuming the cheap neocloud rate is available on-demand at that price.** The headline number is usually gated behind a long take-or-pay commitment or is spot-only. Neocloud on-demand and committed rates commonly differ by 30–60%. *Correction:* check the offer class before you write it in the sheet.
7. **Omitting the self-SRE line.** It never appears on an invoice and it routinely exceeds fabric and support combined. At 16 GPUs, 0.4 FTE is **$0.78/GPU-hour**. *Correction:* estimate the FTE fraction honestly, divide by your fleet's monthly GPU-hours, and note that this line scales inversely with fleet size — which is frequently the whole answer for a small team.
8. **Quoting the "free egress offsets $0.50–1.00/hr" rule without checking the workload.** On a 64-GPU fleet you need ~203 TB/month to move the rate by $0.50/GPU-hour. Text inference will never get there; media generation exceeds it easily. *Correction:* compute `E_GB × rate ÷ fleet_GPU-hours` with your own numbers before treating egress as decisive or dismissing it.
9. **Assuming the whole gap is structural cost advantage.** A 5× sticker gap that collapses to 1.1× after normalisation was bundling (axes a/d), not capital structure (axis b). *Correction:* bank only the residual that survives the full walk, and name which axis it came from.
10. **Treating the depreciation schedule as a neutral fact.** Six-year versus four-year straight-line changes the cost-to-serve floor by 22% for identical hardware. *Correction:* when a provider quotes materially below peers, ask about the schedule before assuming operational excellence.
11. **Recommending a vendor without an attached utilisation floor.** The floor is the load-bearing assumption and the thing that turns a good memo into an expensive surprise when demand slips. *Correction:* every recommendation on committed capacity states the minimum utilisation at which it still holds, plus at least one non-price condition (fabric validation, on-call burden, fleet size) that would reverse it.

## Self-check

- **A neocloud quotes $1.80/GPU-hr and a hyperscaler quotes $4.50 on-demand for H100. Your teammate says "obvious, take the neocloud, we save 60%." What do you ask first, and why that first?** *Answer:* "Is rail-optimised, non-blocking InfiniBand in the quoted tier, and can we run `all_reduce_perf` on the delivered allocation before we pay?" It goes first because it is a **disqualification**, not an adjustment: for multi-node synchronous training a commodity-Ethernet tier is not a cheaper version of the product, it is a different product, and the failure is silent — no billing surprise, no alert, just a collapsed MFU discovered weeks later. Second question: "is $1.80 an on-demand rate or a take-or-pay committed rate, and what utilisation do we honestly forecast?" — because a committed rate at low utilisation is not cheaper per delivered hour (lesson 06: it beats its own on-demand alternative only when `u > 1 − d`). Only after those two do you start adding storage, egress, support, and self-SRE to build the fully-loaded number.
- **Why divide the fully-loaded rate by effective availability *and* by commitment utilisation — and when is it wrong to do the second one?** *Answer:* You pay for hours the fleet is unusable (node failures, maintenance → effective availability < 1) and, on committed capacity, for hours you reserved but did not fill (`u` < 1). Both inflate true cost per *delivered* GPU-hour above the sticker. It is wrong to apply the `u` divisor to an on-demand offer, which strands nothing — doing so penalises it for exposure it does not carry and flatters the committed offer. In the worked example, applying `u` symmetrically at `u = 0.55` shows N winning 2.07×; applying it correctly (only to N) shows 1.14×, and at `u = 0.40` the ranking flips to H. Same inputs, opposite recommendation.
- **After normalising, a 5× sticker gap collapses to 1.1×. What does that tell you and what do you recommend?** *Answer:* The gap was almost entirely **bundling and unbundled adjacent costs** (axes a and d) rather than structural cost-to-serve (axis b) — the neocloud's cheap sticker was a bare GPU-hour and the hyperscaler's rate absorbed fabric, storage, control plane, and support you would otherwise buy separately. At 1.1× fully loaded, recommend the hyperscaler: the residual saving does not compensate for the thinner SLA, the less mature control plane, the take-or-pay commitment risk, and the on-call burden that the self-SRE line only partially captures. State the finding explicitly in the memo — "the gap was bundling, not cost structure" — because that is the reusable conclusion.
- **Build the cost-to-serve floor for an 8×H100 node and say what it explains.** *Answer:* With `K = $280k`, salvage 10%, `L = 6 yr`, `i = 12%` on ~55% average outstanding, `P = 10.2 kW`, `PUE = 1.30`, `e = $0.08/kWh`, fabric `$3,000/GPU`, facility+ops `$1,800/node-month`: depreciation $4.794, financing $2.110, power+cooling $1.061, fabric amortisation $0.457, facility+ops $2.466 → **$10.888/node-hour of calendar**, `÷8 = $1.361/GPU-hr calendar`, `÷ (0.95 × 0.90) = $1.592/GPU-hr sold`. That explains the observed Bronze 1-year committed floor of **$1.45–2.00** — the cheap end of the market is priced at roughly cash cost, which is why it cannot fall further on margin compression and must instead cut inputs (older silicon, no fabric, no spares, no support). It also flags that any quote materially below the floor is telling you something about the inputs rather than offering free money.
- **How much of the neocloud advantage is the depreciation schedule?** *Answer:* Holding all else fixed in the §4 model, the cost-to-serve floor is $1.942/GPU-hr sold at a 4-year life, $1.732 at 5 years, $1.592 at 6 years, $1.492 at 7 — so moving from four to six years cuts the floor by **22%** (~$0.35/GPU-hour) for identical hardware. CoreWeave's FY2025 10-K uses **six years, straight-line** for technology equipment including GPUs, against a weighted-average customer contract duration of about **five years** — the asset must earn beyond the contracted window. So a meaningful slice of an apparent price advantage can be an accounting-policy difference rather than an operational one, and the tell is a provider whose rate is well below peers with no obvious input difference.
- **Does reliability tier decide the neocloud-versus-hyperscaler comparison? Show the arithmetic.** *Answer:* Usually not, and the arithmetic is the interesting part. For a 64-GPU (8-node) synchronous job, `M_job = M_node/N`: a 20,000-hour node MTBF gives `M_job = 2,500 h`, an 8,000-hour MTBF gives `1,000 h`. With `δ = 90 s` and `ρ = 420 s`, lesson 06's `Δ* = sqrt(2δM)` and goodput formula give `g = 0.9955` and `g = 0.9928` respectively — a **0.27 percentage point** difference, which moves the delivered-hour ratio not at all to two decimals. Reliability tier is therefore mostly a *human-time and tail-risk* tax rather than a goodput tax: 2.5× the node failure rate is 2.5× the pages, the cordons, and the RMAs, plus a much worse correlated-failure tail. That cost belongs in the **self-SRE line** ($0.196/GPU-hour for 0.4 FTE on 64 GPUs; $0.78 on 16 GPUs), which is exactly where the worked example puts it — and where it is large enough to flip small-fleet decisions.
- **When does free egress actually matter?** *Answer:* When `E_GB × rate ÷ fleet_GPU-hours` is large relative to the rate gap you are deciding on. On a 64-GPU fleet at 80% utilisation (37,376 GPU-hours/month) at $0.09/GB, each TB/month of egress is worth **$0.002466/GPU-hour**, so you need ~**203 TB/month** to move the effective rate by $0.50. Text inference produces roughly 4 KB per 1,000-token response, so 203 TB would be ~50 billion responses — free egress is irrelevant there. Checkpoint replication to another cloud lands around 40–150 TB/month ($0.10–0.37/GPU-hour: a line item, rarely decisive). Dataset staging and image/video generation reach 200 TB–1 PB+ and can flip the ranking outright. **The rule is workload-shaped, and advice that does not name the workload cannot be applied.**

## Connections & what's next

This lesson's cost per delivered GPU-hour is the number lesson 06's commitment math should be consuming — normalise across vendors first, then size the commitment against the winner — and the two lessons share one discipline: never rank across mismatched bases, and always divide by realised yield before comparing. It inherits module 03's four-field price rule (price, tier, term, date) and adds a fifth field that only appears at fleet scale: **bundle**.

It flows directly into **lesson 08**, where this rate becomes the fixed-cost numerator of the internal **blended chargeback rate** that teams get billed against — the fully-loaded, availability-adjusted figure is what has to be recovered, not the sticker. And it flows into **lesson 10**, where the vendor structure from §1 becomes real FOCUS columns: `ServiceProviderName` and `HostProviderName` for the seller layering, `PricingCategory` for the on-demand/committed/spot basis, `CommitmentDiscountId` for the term, and the cost columns (`ListCost`, `ContractedCost`, `EffectiveCost`, `BilledCost`) for the four-way price comparison that makes a normalisation sheet queryable rather than a spreadsheet someone rebuilt by hand.

Next: **lesson 08 — Chargeback, showback & queue-wait billing** takes the external rate you just normalised and asks the next question: once you own or commit to capacity at this rate, who inside the organisation pays for it, what happens to the part nobody used, and how do you make the incentives point the right way?

## References & further reading

**Primary sources**

- **SemiAnalysis, ClusterMAX live ratings and methodology** — <https://www.clustermax.ai/overview> and <https://www.clustermax.ai/cloudreview> — the ten scored criteria, the five tiers, and the per-provider scorecards. Source for the 2.0-cycle figures in §8 (209+ providers tracked, 84 hands-on evaluated, 140+ customer surveys, 37 medallion ratings, CoreWeave sole Platinum). **Time-sensitive: tier assignments and provider counts change every cycle — a 2.1 update is already published. Re-check before quoting.** *Direct fetch of `clustermax.ai` and `semianalysis.com` is egress-blocked from this environment; the figures above come from search-result extracts of these pages and are consistent with the ClusterMAX facts established in [module 03 lesson 07](../../03-gpu-hardware/lessons/07-capstone-cost-per-useful-work.md).*
- **SemiAnalysis, GPU rental price index** — <https://gpu-index.semianalysis.com/> — live $/hr across SKUs, tiers, and contract terms. The source for §2's observed ranges and the cohort-median path (>$7/hr early 2024 → ~$3.15/hr by 2026). **Refresh every number before you cite it.**
- **CoreWeave, Form S-1 (SEC EDGAR, filed 2025-03-03)** — <https://www.sec.gov/Archives/edgar/data/1769628/000119312525044231/d899798ds1.htm> — six-year straight-line depreciation on technology equipment including GPUs, ~4-year weighted-average contract duration at filing, ~$15.1B remaining performance obligations as of 31 December 2024, take-or-pay contract structure (~96% of 2024 revenue). *`sec.gov` is egress-blocked from this environment; figures verified via search extracts of the filing and CoreWeave's investor-relations releases.*
- **CoreWeave, FY2025 Form 10-K and Q4/FY2025 results** — <https://investors.coreweave.com/> — ~$5.13B 2025 revenue (up ~170%), ~$66.8B revenue backlog and ~$60.7B RPO as of 31 December 2025, weighted-average contract duration ~5 years, six-year useful life for technology equipment. **This updates the previous version of this lesson**, which cited the S-1's ~4-year duration as current; the FY2025 figure is ~5 years, and the backlog is an order of magnitude larger than the S-1's RPO. Same egress caveat.
- **FOCUS Specification, participating-entity columns** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/tree/main/specification/datasets/cost_and_usage/columns> — `serviceprovidername.md`, `hostprovidername.md`, `invoiceissuername.md`, and `pricingcategory.md`. Read directly from the spec repository (`main`, commit `7f19ccb`, read 2026-08-17) for the service-provider-versus-host-provider split that §1's taxonomy maps onto, and for the `Dynamic` pricing category that covers spot. Note that `ProviderName` and `PublisherName` were deprecated in 1.3 and **removed in 1.4** — do not cite them.
- **AWS `p5` (H100) instance and pricing reference** — <https://aws.amazon.com/ec2/instance-types/p5/> — the hyperscaler rate-card anchor for the bundled side of the sheet. *`aws.amazon.com` is egress-blocked from this environment; use it as the live check when you build your own sheet.*
- **Google Cloud A3 (H100) accelerator-optimized VM documentation** — <https://cloud.google.com/compute/docs/accelerator-optimized-machines#a3-vms> — the GCP equivalent bundle (fabric, storage, control plane) that axis (a) asks you to itemise. Same egress caveat.
- **AWS Support plans pricing** — <https://aws.amazon.com/premiumsupport/pricing/> — the marginal-percentage-band structure behind §6's support addon. **The tiers and minimums were restructured during 2025–2026** (Business-tier top band observed at ~9% and Enterprise minimums reduced); treat the *structure* (declining marginal bands, ~3% at large spend) as durable and the percentages as a snapshot. Same egress caveat; figures from search extracts.

**Real-world engineering blogs**

- **SemiAnalysis, "The GPU Cloud ClusterMAX Rating System — How to Rent GPUs" (v1, March 2025)** — <https://newsletter.semianalysis.com/p/the-gpu-cloud-clustermax-rating-system-how-to-rent-gpus> — what it shows: the original methodology and the argument that raw $/hr comparison across GPU clouds is misleading without a reliability tier attached. The founding document for axis (f).
- **SemiAnalysis, "ClusterMAX 2.0: The Industry Standard GPU Cloud Rating System" (November 2025)** — <https://newsletter.semianalysis.com/p/clustermax-20-the-industry-standard> — what it shows: the expanded cycle (209+ tracked, 84 evaluated, 37 rated, 140+ customer surveys) and the tier assignments quoted in §8, with CoreWeave the sole Platinum in both cycles.
- **CoreWeave, "CoreWeave ranks as #1 AI Cloud, backed by SemiAnalysis's Platinum ClusterMAX rating"** — <https://www.coreweave.com/blog/coreweave-ranks-as-1-ai-cloud-backed-by-semianalysiss-platinum-clustermax-tm-rating> — what it shows: the vendor's own framing of what a Platinum rating is worth commercially, which is the demand-side evidence that tier is a priced attribute and not marketing garnish.
- **Crusoe and Nebius reference-cluster and fabric specifications** — <https://crusoe.ai/cloud/> and <https://nebius.com/prices> — what they show: two concrete examples of how neoclouds bundle (or unbundle) rail-optimised fabric, parallel storage, and egress, useful for practising the axis-(a) decomposition against a real quote structure rather than a hypothetical one.

**Deeper dives**

- **NVIDIA NCCL tests (`nccl-tests`), `all_reduce_perf`** — <https://github.com/NVIDIA/nccl-tests> — the validation gate §5 and the decision tree insist on. Run it across the node count you intend to use, at your model's gradient-bucket message sizes, on the *delivered* allocation. It is the only answer to "is the fabric real" that is not a marketing claim.
- **Module 03 lesson 07 — Capstone: cost per unit of useful work** — [`../../03-gpu-hardware/lessons/07-capstone-cost-per-useful-work.md`](../../03-gpu-hardware/lessons/07-capstone-cost-per-useful-work.md) — the origin of the price surface in §2 and of the four-field price rule. Read §6 and §8 there for the hardware-side version of the mismatched-basis error this lesson generalises to vendors.

[💰 11 — GPU cost and unit economics](../README.md)
