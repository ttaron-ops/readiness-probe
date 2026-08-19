---
lesson: "09.7"
title: "Bandwidth as a cost input"
module: "09"
concept: "Bandwidth as a cost input"
status: not-started
est_time: "7h"
prev: "06-k8s-multi-nic.md"
next: null
artifacts: []
sources: 12
---

# 09.7 · Bandwidth as a cost input

> **Concept.** Turn topology into dollars — oversubscription as a capex lever, the IB premium vs RoCE reuse, SHARP as byte-reduction, and the widening scale-up-vs-scale-out $/GB/s gap — and convert a placement choice into a bandwidth-and-cost statement.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Where this fits

Lesson 06 gave you the *mechanism*: Multus, SR-IOV, the Network Operator, Topology Manager — the plumbing that puts a job's pods on the right NUMA node next to the right NIC VF. That lesson answered "how does a placement decision get enforced." This lesson answers the question procurement and your manager actually ask: **"what does that placement decision buy, in dollars, and what does the alternative cost?"**

It is the last lesson in the module because it is the synthesis. Every earlier lesson feeds one final move: turning a diagram into a number a procurement committee will act on. And unlike the earlier lessons, the deliverable here is not a mechanism you can point at — it is an *argument*, which means the numbers have to be derived rather than quoted, so that when someone changes an input you can re-derive rather than re-google. That is exactly what the module's [Network architecture read](../practice/network-architecture-read/README.md) asks you to produce.

## Why this matters

Everyone in this conversation can quote a switch datasheet. Almost nobody can answer the two questions that actually decide a fabric: **what does a GPU-hour of back-end bandwidth cost, and how much faster does the job have to run to justify a better fabric?** Those are arithmetic, and this lesson does the arithmetic end to end — from switch, NIC and optics capex, through amortisation and power, down to dollars per GPU-hour, and then across to the training-time saving on the other side of the trade.

The stakes are asymmetric and they are not what most people assume. On the model built here, moving from a full-bisection spine to a 1:7 oversubscribed one saves about **$0.14 per GPU-hour** — roughly 4.6% of a $2.99/GPU-hour blended rate. That is real money at scale (about **$700k a year across 512 GPUs**) but it is small enough that a comms-bound job spanning the oversubscribed tier destroys the saving several times over: the same model shows such a job running **2.2× slower**, which means full bisection pays for itself as soon as more than about **8.5% of your GPU-hours** run cross-tier. Getting that fraction wrong in either direction costs more than any switch discount you will ever negotiate. And a wrong bet does not page anyone — it shows up as a step time nobody can explain, months after the purchase order cleared.

## What's new here (calibration)

You already know how to read `nvidia-smi topo -m` (02b), what a Clos and an oversubscription ratio are (09.2), the IB-vs-RoCE mechanics (09.4), what SHARP does to the byte count (09.5), and how a placement is enforced (09.6). None of that is re-taught. What is new:

- **A derived cost model, not a quoted one.** A per-GPU fabric bill of materials built from switch radix and oversubscription ratio, converted to $/GPU-hour with a capital recovery factor, maintenance, power and utilisation — with a sensitivity table showing which inputs actually move the answer. You will find the biggest lever is not the switch price.
- **The break-even calculation**, in both directions: how much faster a job must run to justify a better fabric, and what fraction of your GPU-hours must be cross-tier before full bisection wins.
- **Two results that contradict the usual framing** and fall straight out of the algebra: oversubscription *raises* the cost per unit of delivered bisection even as it lowers the total, and the lever is nearly exhausted by 1:4 — past that you are paying the same irreducible floor for less bandwidth.
- **Where fabric cost belongs in the cost model you already have.** Module 11's loading multiplier is `L = (P/A)(1+f)`. This lesson shows exactly when fabric spend belongs in `f` and when putting it there double counts — the same discipline module 11 applies to idle.
- **Currency and provenance discipline.** Every dollar in this lesson is a *parameter with a stated value*, not a fact. The structure survives a hardware generation; the numbers do not, and the lesson is built so you can substitute your own quotes and re-run.

## Core concepts

### 1. The unit: dollars per GPU-hour of fabric

Start by fixing the unit, because most fabric arguments fail on units rather than on facts.

`$/port` is a purchasing unit and tells you nothing about performance. `$/Gb/s` of link rate is worse — it makes an oversubscribed fabric look identical to a non-blocking one, since the link rates are the same. **`$ per unit of bisection bandwidth` is the right performance-normalised metric**, and §4 shows it behaving in a way that surprises people.

But the unit that lets you argue with the rest of the business is **dollars per GPU-hour**, because that is the unit everything else in the fleet already speaks: your blended rate `r` is in $/GPU-hour (module 11), your unit costs are derived from it, and your margin conversation is denominated in it. A fabric argument stated in $/GPU-hour can be compared directly against the thing it competes with — more GPUs, better GPUs, or a lower price sheet.

So the whole lesson is one conversion:

```
   switch capex          ┐
   NIC capex             │
   optics capex          ├──▶  $ per GPU of fabric  ──┐
   installation          │      (a BOM, §2)           │
                         ┘                            │
                                                      ├──▶  $ per GPU-HOUR
   switch power          ┐                            │      (§3)
   optics power          ├──▶  W per GPU of fabric  ──┘
   PUE, $/kWh            ┘

   and then, on the other side of the trade:

   oversubscription ratio ──▶ effective bisection ──▶ step time ──▶ GPU-hours
```

### 2. The per-GPU fabric bill of materials

Build the BOM per GPU rather than per cluster. Per-cluster numbers do not compose; per-GPU numbers do, and they slot straight into a $/GPU-hour rate.

Take a rail-optimized two-tier Clos, one NIC port per GPU (lesson 01: the reference build actually has nine NICs on an eight-GPU node — eight rail NICs plus one off-rail for storage and management; we cost the eight, and you should add the ninth when you price a real build).

Let:

```
  R    switch radix                     ports per switch
  OS   oversubscription ratio           leaf downlinks : leaf uplinks
  d    leaf downlinks  = R·OS/(OS+1)    GPU-facing ports on a leaf
  u    leaf uplinks    = R/(OS+1)       spine-facing ports on a leaf
```

Every GPU consumes exactly one leaf downlink. So per GPU:

```
  leaf switches       1/d           of a switch
  leaf uplinks        u/d = 1/OS    of a spine-facing link
  spine switches      (1/OS)/R      of a switch
```

and the capex attributable to one GPU is

```
  C_gpu = C_nic                    the adapter
        + 2·C_opt_short            one GPU-facing link, a module at each end
        + C_sw / d                 its share of a leaf switch
        + (1/OS)·2·C_opt_long      its share of the leaf→spine links
        + (1/OS)·C_sw / R          its share of a spine switch
```

with the identical shape for power (`P_nic`, `P_opt`, `P_sw` in place of the costs). Drawn, with the worked values from the table below at full bisection:

```
  PER-GPU FABRIC BOM — rail-optimized 2-tier Clos, R = 64, 400 Gb/s
  ══════════════════════════════════════════════════════════════════

        ┌──────┐
        │ GPU  │   700 W, ~$30,000 of server+GPU share
        └──┬───┘
           │  PCIe                                     ┌───────────────────┐
        ┌──┴────────────┐                              │ IRREDUCIBLE FLOOR │
        │ ConnectX-7    │  $2,500      25 W  ──────────│ does NOT shrink   │
        │ 400 Gb/s NIC  │                              │ with              │
        └──┬────────────┘                              │ oversubscription  │
           │                                           │                   │
        ┌──┴────────────┐                              │  NIC     $2,500   │
        │ 2 × short     │  2×$250      2×12 W ─────────│  optics    $500   │
        │ optic (SR/DR) │  = $500      = 24 W          │  ───────────────  │
        └──┬────────────┘                              │  floor   $3,000   │
           │  1 link per GPU, always                   │  + leaf ≥ C_sw/R  │
    ═══════╪════════════════════════════════════       │          = $1,250 │
        ┌──┴───────────────────────┐                   │  ───────────────  │
        │  LEAF SWITCH  64 × 400G  │  $80,000  1,600 W │  TOTAL   $4,250   │
        │  d downlinks, u uplinks  │                   └───────────────────┘
        │  share per GPU = C_sw/d  │
        │    OS=1 → d=32 → $2,500     50 W
        │    OS=7 → d=56 → $1,429     29 W
        └──┬───────────────────────┘
           │  uplinks per GPU = 1/OS   ◀── THE ONLY TERM OS TOUCHES
           │     OS=1 → 1.00 link      OS=4 → 0.25      OS=7 → 0.143
        ┌──┴────────────┐
        │ 2 × long      │  (1/OS)·2×$600      (1/OS)·24 W
        │ optic (DR4)   │   OS=1 → $1,200         24.0 W
        └──┬────────────┘   OS=7 →   $171          3.4 W
           │
        ┌──┴───────────────────────┐
        │  SPINE SWITCH 64 × 400G  │  share = (1/OS)·C_sw/R
        │                          │   OS=1 → $1,250   25.0 W
        └──────────────────────────┘   OS=7 →   $179    3.6 W

  ══════════════════════════════════════════════════════════════════
   TOTAL PER GPU     OS=1  $7,950  148.0 W          $159 per GB/s
                     OS=2  $6,100  111.0 W          $244 per GB/s
                     OS=4  $5,175   92.5 W          $414 per GB/s
                     OS=7  $4,779   84.6 W          $669 per GB/s
                     OS→∞  $4,250   ~72 W             (no bisection)
```

**Every dollar and watt above is an assumption, not a fact.** They are stated here so the arithmetic is auditable, and §3's sensitivity table tells you which of them are worth chasing a real quote for:

| Parameter | Symbol | Value used | Note |
|---|---|---:|---|
| Switch, 64 × 400G | `C_sw` | $80,000 | street price varies enormously with volume and vendor |
| NIC, ConnectX-7 400G | `C_nic` | $2,500 | |
| Short-reach optic (in-rack/in-row) | `C_opt_short` | $250 | DAC/AOC is cheaper still where reach allows |
| Long-reach optic (leaf↔spine, single-mode) | `C_opt_long` | $600 | the priciest module in the build |
| Switch power | `P_sw` | 1,600 W | |
| NIC power | `P_nic` | 25 W | |
| Optic power, 400G | `P_opt` | 12 W | 800G OSFP modules run higher — vendor datasheets put DR8 around 14–16 W, with linear-drive (LPO) variants materially lower |
| Server + GPU share, per GPU | — | $30,000 | for the "fabric as % of system" check only |

**One external consistency check exists**, and the model passes it. Fabric spend as a share of total system cost comes out at `7,950 / (30,000 + 7,950)` = **20.9%** at full bisection and **13.7%** at 1:7. The widely-quoted rule of thumb for rail-optimized GPU builds is 10–20% of system cost. The model lands in that band from independent inputs, which is the only validation available without a real BOM — take it as "the parameters are not crazy," not as proof.

### 3. From capex to $/GPU-hour

Capex is a lump; a rate is what you can argue with. Three conversions.

**(a) Amortisation.** Do not simply divide by the life. Use a **capital recovery factor**, the annuity that repays the capital plus the cost of that capital over `n` years at discount rate `i`:

```
              i·(1+i)ⁿ
    CRF  =  ──────────────
              (1+i)ⁿ − 1

    n = 5 years, i = 10%  →  CRF = 0.2638
```

Straight-line depreciation (`1/n = 0.20`) understates the annual charge by about 24% here, because it prices the capital as free. Use CRF when you are arguing a purchase; use straight-line only when you are reconciling to an accounting statement. For calibration: CoreWeave's FY2025 Form 10-K sets the useful life of technology equipment including GPUs at **six years, straight-line** (module 11.6). Network gear is often held on a shorter book life than the accelerators, which is why 5 years is the base case here and §3(d) shows what 3 and 6 do.

**(b) Maintenance and support.** Vendor support, sparing and RMA typically run about 10% of capex per year on network equipment. Carry it as `m = 0.10` and treat it as a parameter, because it is one of the few line items you can actually negotiate.

**(c) Power.** `P_gpu` watts, grossed up by facility PUE, at your energy rate:

```
    annual_power_$ = (P_gpu / 1000) × PUE × 8760 × $/kWh
```

Base case: PUE 1.25, $0.08/kWh.

Putting it together, and this is the formula to memorise:

```
  ══════════════════════════════════════════════════════════════════════
    fabric $ per GPU-hour

              C_gpu · (CRF + m)  +  (P_gpu/1000) · PUE · 8760 · $/kWh
      =     ─────────────────────────────────────────────────────────
                                 8760 · U

    U = the fraction of wall-clock hours the GPU is actually sold or
        allocated.  The fabric is paid for whether or not the GPU is
        earning, so every idle hour reprices every busy hour.
  ══════════════════════════════════════════════════════════════════════

  WORKED, at U = 1.0 (i.e. the pure cost of ownership per elapsed hour):

    FULL BISECTION (OS = 1)
      capex charge   $7,950 × (0.2638 + 0.10)      =  $2,892 / yr
      power charge   0.148 kW × 1.25 × 8760 × 0.08 =    $130 / yr
                                                      ─────────
                                                      $3,022 / yr
                                          ÷ 8760  →  $0.345 / GPU-hour

    1:7 OVERSUBSCRIBED (OS = 7)
      capex charge   $4,779 × 0.3638                =  $1,738 / yr
      power charge   0.0846 kW × 1.25 × 8760 × 0.08 =     $74 / yr
                                                      ─────────
                                                      $1,813 / yr
                                          ÷ 8760  →  $0.207 / GPU-hour

    ΔFABRIC  =  $0.138 / GPU-hour
             =  4.6 % of a $2.99 / GPU-hour blended rate
             =  $700,000 / year across 512 GPUs
```

**(d) Which inputs actually matter.** Vary one at a time from the full-bisection base of $0.345/GPU-hour:

| Change | $/GPU-hour | Δ |
|---|---:|---:|
| **Amortisation life 5 y → 3 y** | $0.471 | **+36.4%** |
| Switch price $80k → $120k | $0.423 | +22.6% |
| Switch price $80k → $50k | $0.287 | −16.9% |
| Maintenance 10% → 5% | $0.300 | −13.2% |
| NIC $2,500 → $1,500 | $0.303 | −12.0% |
| Long optic $600 → $1,000 | $0.378 | +9.6% |
| Discount rate 10% → 15% | $0.376 | +9.1% |
| Amortisation life 5 y → 6 y | $0.314 | −9.0% |
| Energy $0.08 → $0.14/kWh | $0.356 | **+3.2%** |
| PUE 1.25 → 1.50 | $0.348 | **+0.9%** |

Read the two ends of that table together, because they overturn a common belief. **The single biggest lever on fabric cost per GPU-hour is how long you amortise it over** — a 3-year life costs 36% more per hour than a 5-year one, more than a 50% switch price increase. And **power is nearly irrelevant as a cost line**: at $0.08/kWh it is 4.3% of fabric TCO, and even doubling PUE moves the answer by under 1%. If you have been told to "price the fabric as capex plus power," the second term is a rounding error at these rates.

That does not make power unimportant — it makes it the *wrong kind* of important, which is §9.

**(e) The utilisation term is not decoration.** Because `U` sits alone in the denominator, fabric cost per *sold* GPU-hour is hyperbolic in it:

| Utilisation `U` | Full bisection | 1:7 | Δ |
|---:|---:|---:|---:|
| 1.00 | $0.345 | $0.207 | $0.138 |
| 0.85 | $0.406 | $0.243 | $0.162 |
| 0.70 | $0.493 | $0.296 | $0.197 |
| 0.55 | $0.627 | $0.376 | $0.251 |

A fleet running at 55% allocation pays 82% more per sold GPU-hour for the same fabric than one at 100%. **The fabric argument and the scheduling argument are the same argument** — which is why lessons 06 of both this module and the platform module are prerequisites for this one, and why module 11's gap analysis is where the other half of the money is.

### 4. Oversubscription: two results that are not the usual story

Now use the model rather than the slogan.

**Result 1 — oversubscription *raises* the cost per unit of bisection.** Look at the last column of §2's table again:

| `OS` | fabric $/GPU | per-GPU bisection | **$ per GB/s** |
|---:|---:|---:|---:|
| 1:1 | $7,950 | 50.0 GB/s | **$159** |
| 1:2 | $6,100 | 25.0 GB/s | $244 |
| 1:4 | $5,175 | 12.5 GB/s | $414 |
| 1:7 | $4,779 | 7.14 GB/s | **$669** |
| 1:16 | $4,481 | 3.13 GB/s | $1,434 |

Total spend falls 40% from 1:1 to 1:7; cost per delivered GB/s rises **4.2×**. Both statements are true and they are about different things. The reason is structural: the NIC, the GPU-facing optics and a floor share of the leaf switch are bought per GPU no matter what, and none of them shrink when you cut uplinks. **Full bisection is the cheapest bandwidth you will ever buy; it is simply the most bandwidth.** If someone justifies oversubscription with "better $/GB/s," they have it exactly backwards — the correct justification is "we do not need those GB/s," which is a workload claim, not a cost claim.

**Result 2 — the lever is nearly exhausted by 1:4.** The floor, as `OS → ∞`, is `C_nic + 2·C_opt_short + C_sw/R` = $2,500 + $500 + $1,250 = **$4,250 per GPU**. So:

```
  $8,000 ┤ ●  1:1  $7,950
         │  ╲
  $7,000 ┤   ╲
         │    ╲
  $6,000 ┤     ● 1:2  $6,100
         │      ╲
         │       ╲
  $5,000 ┤        ●─1:4 $5,175
         │          ╲──●─1:7 $4,779
         │              ╲──────●─1:16 $4,481
  $4,250 ┼┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄  IRREDUCIBLE FLOOR
         └────┬─────┬─────┬─────┬─────┬──
             1:1   1:2   1:4   1:7  1:16

     1:1 → 1:4  saves $2,775 per GPU   (35% of the total)
     1:4 → 1:7  saves   $396 per GPU   (5%)   ← for 1.75× less bisection
     1:7 → 1:16 saves   $298 per GPU   (4%)   ← for 2.3× less bisection
```

**Going past 1:4 trades a lot of bandwidth for very little money.** That is a directly actionable procurement result and it is invisible until you build the BOM: the aggressive ratios that sound like big savings are mostly grinding against a floor you already paid. If your build is being pushed toward 1:8 or beyond, ask what the incremental saving actually is in the model — it is usually a rounding error against the training-time risk in §5.

*(A caveat that keeps this honest: the model assumes one switch tier above the leaf. A real cluster large enough to need three tiers adds another spine layer, and oversubscription then applies per tier, so the savings recur. The shape of the curve is the same; the floor is a little higher.)*

### 4b. Extending the model to three tiers, and what changes

The two-tier model covers a pod. Real clusters past a few thousand GPUs need a third tier, and the arithmetic composes cleanly once you see the pattern: **each tier contributes a switch-share term and a link term, scaled by the product of the oversubscription ratios below it.**

A leaf switch of radix `R` at ratio `OS₁` gives each GPU `1/OS₁` of a leaf uplink. If the spine tier is itself oversubscribed at `OS₂` toward a super-spine, each GPU's share of a super-spine-facing link is `1/(OS₁·OS₂)`:

```
  C_gpu = C_nic + 2·C_opt_short                       ← per-GPU floor, tier-independent
        + C_sw/d₁                                     ← leaf share,  d₁ = R·OS₁/(OS₁+1)
        + (1/OS₁)·(2·C_opt_long + C_sw/d₂)            ← leaf→spine link + spine share,
                                                        d₂ = R·OS₂/(OS₂+1)
        + (1/(OS₁·OS₂))·(2·C_opt_long + C_sw/R)       ← spine→super-spine link
                                                        + super-spine share
```

Two consequences worth carrying into a large-cluster argument:

- **The effective end-to-end oversubscription is the product**, `OS₁ · OS₂`. A "1:2 at the leaf, 1:4 at the spine" build is a 1:8 fabric for any traffic that has to reach the top tier — and people routinely quote only the tier they are looking at. Always ask for the ratio *per tier* and multiply.
- **Cost per delivered GB/s degrades faster than in the two-tier case**, because each additional tier adds a fixed switch-share term while the bisection is cut multiplicatively. This is precisely why the standard large-cluster design is *not* uniform oversubscription: it is a **non-blocking pod plus one oversubscribed tier between pods**, so that the multiplicative penalty applies only to traffic that leaves the pod — which is a topology decision that only pays off if placement holds it, i.e. §5's `f`.

The Llama 3-class design (non-blocking inside a 3,072-GPU pod, oversubscribed at the aggregation tier between pods) is exactly this shape, and SemiAnalysis's published analysis of hypothetical 100,000-GPU builds walks the same lever one order of magnitude up: either a fully non-blocking design that needs a fourth Clos tier, or four internally-non-blocking pods joined by an oversubscribed top tier. *(Both of those source documents were unreachable from this environment — see References. The structural claim is the model's; the attributions are carried forward from lesson 09.2 and should be verified before you cite the specific figures.)*

### 4c. The decision table

Before the step-time analysis, it is worth laying out what the options actually are, because "how much do we oversubscribe" is only one of four levers and it is not the strongest one.

| Lever | What it costs | What it buys | When it is the right move |
|---|---|---|---|
| **Buy full bisection** | +$0.138/GPU-h (§3) | cross-tier traffic runs at line rate; no placement dependency | `f` above the break-even (§5), or a scheduler you cannot constrain |
| **Oversubscribe to 1:4** | −$0.142/GPU-h; cross-tier jobs 37% slower | ~35% of fabric capex back | `f` reliably small, and topology-aware scheduling in place |
| **Oversubscribe past 1:4** | −$0.011/GPU-h more; cross-tier jobs 118% slower at 1:7 | almost nothing (§4 Result 2) | essentially never on this parameter set — check your own floor first |
| **Improve placement (`f`)** | scheduler engineering; no hardware | moves the break-even without spending capex; also the DRANET result | always. It is the only lever with no capex and no bandwidth loss |
| **Add SHARP (IB)** | the IB premium (§6) | halves `δ`, roughly doubling tolerable `f` (§7) | reduction-shaped collectives, offloadable op and dtype, not MoE |
| **Change the parallelism plan** | model-engineering time | moves bytes into the NVLink domain where marginal cost is zero (§8) | always worth checking before buying bandwidth |

The row that should discomfort a procurement conversation is the fourth. **Placement is the only lever that costs no capex, loses no bandwidth, and moves the break-even directly** — and on the published DRANET measurement, topology-aware GPU-and-NIC scheduling recovered up to ~58% of collective bandwidth with no hardware change at all, which is larger than the entire full-bisection-versus-1:7 decision. If the scheduling work is not done, buying a better fabric is buying your way around a software problem at $0.138/GPU-hour.

### 5. What oversubscription costs in step time — and the break-even

The other side of the trade. A fabric is only cheap if the workload does not need what you removed.

**Setup**, stated so you can re-run it: 512 GPUs (64 nodes × 8), H100 SXM, one 400 Gb/s rail per GPU, data-parallel training of an 8-billion-parameter model with bf16 gradients. Gradient buffer `S = 8×10⁹ × 2 B = 16 GB` per rank.

**Collective time.** Ring all-reduce moves `2(N−1)/N · S` per rank (09.5 §11):

```
  bytes on the wire per rank = 2 × (511/512) × 16 GB = 31.94 GB
  achieved bus bandwidth, in-pod, non-blocking       = 45 GB/s   (assumption:
                                                       ~90% of the 50 GB/s
                                                       line rate — measure yours)
  t_comm(in-pod)  = 31.94 / 45  =  0.710 s
  t_comm(1:7)     = 0.710 × 7   =  4.968 s
```

**Step time with overlap.** Assume the framework overlaps up to 60% of the collective with backward compute, and that overlap obviously cannot exceed the compute time itself. With `t_compute = 2.00 s`:

```
  step = t_compute + ( t_comm − min(0.6·t_comm, t_compute) )
```

```
  IN-POD, NON-BLOCKING                       SPREAD ACROSS A 1:7 TIER
  ════════════════════                       ════════════════════════

  compute  ████████████████████ 2.00 s       compute  ████████████████████ 2.00 s
  comms    ▒▒▒▒▒▒▒░░░           0.710 s      comms    ▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒░░░░░░░░
           └overlapped┘ └exposed┘                     └── overlapped ──┘└─ exposed ─┘
                       0.284 s                              2.00 s        2.968 s

  STEP  ├──────────────────────┤ 2.284 s     STEP  ├────────────────────────────────┤
                                                                              4.968 s

                              ── 2.18× SLOWER ──▶
```

Note what happened to the overlap: at 1:7 the collective is longer than the entire compute phase, so overlap saturates at `t_compute` and every additional second of communication lands fully exposed. **Beyond a threshold, oversubscription penalties stop being partially hidden and become linear.** That threshold is `t_comm = t_compute / 0.6`, i.e. 3.33 s here — and 1:7 blows straight through it. This is why "we overlap our collectives" is not a defence against an undersized spine.

**The break-even.** Let `f` be the fraction of your GPU-hours running jobs that span the oversubscribed tier, and `δ` the slowdown those jobs suffer. The GPU-hours reclaimed by full bisection, per GPU-hour of fleet time, are `f · δ/(1+δ)`. Full bisection pays for itself when the value of those hours exceeds the extra fabric cost:

```
      f · [ δ / (1+δ) ] · r   ≥   Δfabric

      with δ = 1.176, r = $2.99/GPU-h, Δfabric = $0.138/GPU-h:

      f ≥ 0.138 / (0.5405 × 2.99)  =  0.0854

  ⇒  FULL BISECTION PAYS IF MORE THAN ~8.5% OF GPU-HOURS RUN CROSS-TIER.
```

That is the sentence to walk into the review with. It is small — which tells you something important: **the fabric saving is thin enough that it only survives if placement is genuinely reliable.** A scheduler that keeps 95% of comms-heavy work in-pod makes the oversubscribed fabric correct. One that manages 85% does not, and nobody will notice for a quarter.

Two ways to move the break-even in your favour:

- **Make `δ` smaller** — SHARP (§7), better overlap, gradient compression, or a parallelism plan that puts less traffic on the scale-out fabric (§8).
- **Make `f` smaller** — topology-aware scheduling (09.6, and the platform module's Kueue topology-aware scheduling), which is free by comparison and is the highest-leverage thing on this list.

### 6. The IB "tax" vs RoCE "reuse", priced

The fabric-technology choice is a second axis, orthogonal to oversubscription. Run it through the same model rather than trading adjectives.

| | InfiniBand | RoCEv2 / Spectrum-X Ethernet |
|---|---|---|
| Enters `C_sw`, `C_nic`, `C_opt` as | a premium; single-vendor supply since NVIDIA acquired Mellanox | multi-vendor optics and merchant switch silicon |
| Control plane | a dedicated **subnet manager** to run, patch and staff — an opex line with no Ethernet equivalent | reuses the EVPN/BGP estate and the team you already have |
| Congestion control | credit-based flow control in the fabric; works out of the box | PFC + ECN + DCQCN, which must be tuned and can be got wrong (09.4) |
| In-network reduction | **SHARP** — enters the model as `δ` reduction (§7) | no equivalent; NVLink SHARP is a scale-*up* mechanism (09.5 §12) |
| Lock-in | high | low |
| Current generation | Quantum-2, 64 × 400G (NDR); Quantum-X800 at 800G with SHARPv4 | Spectrum-4 SN5600, 64 × OSFP 800GbE, 51.2 Tb/s, 2U; SN5400 at 64 × 400GbE |

**A correction worth making explicitly, because the wrong version of it circulates widely.** You will see the claim that Spectrum-X fits "128 × 400G against InfiniBand's 64 × 400G" — double the port density, therefore fewer spine switches for the same bisection. That comparison is not like-for-like. SN5600 reaches 128 × 400G by *breaking out* its 64 native 800G ports; the InfiniBand switch it is being compared against, Quantum-2, is the previous 400G generation. The current-generation InfiniBand comparison is Quantum-X800, which is also an 800G platform. Compare at **matched aggregate switching capacity (Tb/s) and matched direction convention**, never at raw port count — vendors quote both unidirectional and bidirectional totals for the same box, which alone can produce a spurious 2×.

The model also tells you *where* a port-density advantage would even show up, and it is smaller than the rhetoric suggests. Switch cost enters `C_gpu` only through `C_sw/d` and `(1/OS)·C_sw/R` — that is, through **cost per port**, not port count. A switch with twice the ports at twice the price changes nothing. At full bisection the two switch terms are $2,500 + $1,250 = $3,750 of $7,950, so a genuine 20% reduction in cost-per-port is worth $750/GPU, about $0.033/GPU-hour — roughly a quarter of the entire oversubscription lever. Real, worth having, not decisive.

**The honest way to run the IB-vs-RoCE comparison** is to put both through §3 with your own quotes and then adjust `δ` for the differences that are not capex:

```
  Δ(IB − RoCE) per GPU-hour  =  [ ΔC_gpu · (CRF + m) + ΔP · PUE · 8760 · $/kWh ] / (8760·U)
                                 └─ capex and power delta ─┘
                              +  subnet-manager staffing ÷ (GPU count × 8760 × U)
                                 └─ the opex line RoCE does not have ─┘
                              −  f · [ δ_SHARP/(1+δ_SHARP) ] · r
                                 └─ the training time SHARP gives back (§7) ─┘
                              +  risk premium for PFC/ECN/DCQCN tuning on RoCE
                                 └─ not a number; state it as a named risk ─┘
```

Two of those four terms are computable from datasheets and payroll; one is computable from a benchmark; the fourth is a judgement you should state as a judgement. That is a defensible procurement argument. "IB is faster" is not.

For scale calibration: one full-time subnet-manager-and-fabric engineer at a fully-loaded $250k/year, spread over a 512-GPU cluster at `U = 0.85`, is `250,000 / (512 × 8760 × 0.85)` = **$0.066/GPU-hour** — about half the entire oversubscription lever, from one headcount. On a 5,000-GPU cluster the same headcount is $0.0067/GPU-hour and disappears. **The subnet manager is a real cost at small scale and a rounding error at large scale**, which is a large part of why IB's case strengthens as clusters grow.

### 7. SHARP as a byte-reduction lever, in dollars

Lesson 05 §11 derived the traffic ratio: ring all-reduce puts `2(N−1)/N · S` on each rank's link; SHARP puts `S`. At `N = 512` that is a factor of **1.996**. Feed it back into §5's model:

```
  WITHOUT SHARP                             WITH SHARP (÷1.996 on the wire)
  ─────────────                             ───────────────────────────────
  t_comm in-pod        0.710 s              t_comm in-pod        0.356 s
  t_comm across 1:7    4.968 s              t_comm across 1:7    2.490 s

  step in-pod          2.284 s              step in-pod          2.142 s
  step across 1:7      4.968 s              step across 1:7      2.996 s

  δ = 1.176  (118% slower)                  δ = 0.398  (40% slower)

  break-even f = 8.5%                       break-even f = 16.2%
```

**SHARP roughly doubles the fraction of cross-tier work an oversubscribed fabric can absorb before full bisection becomes the cheaper choice.** That is the precise, quantitative form of "SHARP lets you buy a cheaper fabric" — and note that the mechanism is not that SHARP saves money directly, but that it *widens the operating envelope in which the cheap fabric is still correct*.

Three conditions on that number, all from 05:

- It is **InfiniBand-only** at the scale-out tier, so it is part of the IB premium in §6 and cannot be assumed for a RoCE build.
- It applies only to **reduction-shaped** collectives with an offloadable op (`SUM`/`MAX`/`MIN`) and an offloadable dtype. **bf16 offloads only when the fabric reports v3 datatype support** — and bf16 is what the worked example above reduces. If that support is absent, `δ` reverts to 1.176 and the break-even reverts to 8.5%, with a single `WARN` line as the only evidence. Check before you count it.
- It does **nothing** for all-to-all. On an expert-parallel MoE workload the SHARP term in §6's equation is zero, and the IB premium has to be justified on determinism and tuning risk alone — a much weaker case. As MoE-shaped architectures take a larger share of production training, this is a live procurement risk, not a footnote: a fabric bought on SHARP's numbers for a dense model may be running a workload SHARP cannot touch.

### 8. Scale-up vs scale-out — the marginal-cost argument, stated precisely

There are two fabrics with an order-of-magnitude different price per GB/s, and the gap widens each generation:

| | H100 generation | GB200/GB300 (Blackwell Ultra) |
|---|---|---|
| Scale-up per GPU (NVLink) | ~900 GB/s (NVLink 4) | ~1.8 TB/s (NVLink 5) |
| Scale-out per GPU | 400 Gb/s = 50 GB/s (ConnectX-7) | 800 Gb/s = 100 GB/s (ConnectX-8) |
| Ratio | 18 : 1 | 18 : 1 |

The ratio is flat, so the interesting statement is not about raw bandwidth — it is about **cost structure**. Scale-out bandwidth carries the full §2 BOM: $159 per GB/s of bisection at full bisection, more when oversubscribed. NVLink bandwidth carries no separately-procurable BOM at all: the NVSwitch silicon is inside the board price, and there is no SKU in which you buy less NVLink for less money.

Be precise about what that means, because "NVLink is free" is wrong and the correct version is more useful. NVLink is not free — its cost is real and is inside the ~$30,000/GPU server line. It is **not marginal**: there is no decision at which you trade NVLink bandwidth for dollars. Scale-out bandwidth *is* marginal — every uplink you delete is money you keep, which is the entire subject of §4. So at the margin, which is where all procurement decisions live:

```
    marginal $ per GB/s, scale-out  =  $159  (full bisection)  …  $669  (1:7)
    marginal $ per GB/s, scale-up   =  $0    — there is no lever to pull
```

**The FinOps consequence: the cheapest bytes are the ones that never leave the NVLink domain, and a parallelism plan is therefore also a cost plan.** Putting tensor parallelism inside the 8-GPU NVLink island and reserving the scale-out fabric for the lighter pipeline- and data-parallel traffic maximises use of bandwidth you cannot decline to buy and minimises demand on the tier where every GB/s has a price tag. When you argue an oversubscription ratio in §4, you are implicitly asserting that the parallelism plan keeps most bytes off the scale-out fabric — so state the parallelism plan as part of the fabric argument, or the argument has a hole in it.

### 9. The opex tail: power is a capacity constraint, not a cost line

§3(d) showed power is 4.3% of fabric TCO at $0.08/kWh and that doubling PUE moves the answer by under 1%. So drop the "capex plus power-years" framing — as a *cost* argument it is noise.

Power matters for a different reason: **modern GPU builds are constrained by megawatts, not by dollars.** In a facility with a fixed power envelope, every watt the fabric draws is a watt not available to a GPU, and GPUs are what earn.

```
  PER-GPU DELIVERED POWER (incl. PUE 1.25)

    GPU (H100 SXM, 700 W TDP — Meta's Llama 3 model card)      700 W
    server share: CPU, DRAM, NVSwitch, fans, PSU loss          200 W   (assumption)
    fabric, full bisection                                     148 W
    fabric, 1:7                                                 85 W
                                                             ───────
    total × PUE 1.25   full bisection   1,048 × 1.25  =  1,310 W / GPU
                       1:7               985 × 1.25  =  1,231 W / GPU

  IN A 10 MW FACILITY
    full bisection  →  10,000,000 / 1,310  =  7,634 GPUs
    1:7             →  10,000,000 / 1,231  =  8,125 GPUs
                                              ────────
                                              +491 GPUs, +6.4%
```

**In a power-capped build, oversubscribing the spine buys you 6.4% more GPUs.** At $2.99/GPU-hour and 85% utilisation that is `491 × 8760 × 0.85 × 2.99` ≈ **$10.9M/year of additional revenue capacity** — nearly an order of magnitude more than the $700k/year of capex saving from the same decision at 512-GPU scale. The power argument for oversubscription is far stronger than the cost argument, and it is the one almost nobody makes.

The rest of the tail, briefly and honestly: **cabling labour and structured cabling** scale with spine link count and therefore with `1/OS`, and are a real installation line but not a recurring one. The **subnet manager** on the IB side is priced in §6 and is the one opex item with a genuinely different shape between the two fabrics. **Optics failure and RMA** are inside the 10% maintenance parameter — if your operational history says optics fail more often than that, raise `m` and re-run rather than adding a separate term.

### 10. Where this sits in the cost model you already have

Module 11 defines the fully-loaded unit cost with

```
    L = (P / A) × (1 + f)

    P  paid GPU-hours,  A  allocated GPU-hours
    f  non-GPU overhead as a fraction of GPU spend
       (control plane, storage, networking, egress, observability)
```

The fabric $/GPU-hour derived here looks like it belongs in `f`. **Usually it does not, and putting it there double counts** — the same discipline module 11 applies to a tenant's own idle:

- **If you rent GPU capacity** at a blended rate `r` (the module-11 fleet's $2.99/GPU-hour), the provider's back-end fabric is already inside `r`. It was in their BOM, their amortisation and their margin. Adding this lesson's $0.345 to `f` charges for it twice. What belongs in `f` is only fabric spend **you are separately billed for** — egress, a managed-interconnect line item, a dedicated interconnect charge.
- **If you own or colocate the hardware**, there is no `r` to hide inside, and this lesson's number is exactly how you *build* the GPU-hour rate: `r = (GPU + server capex charge + fabric charge + power + facility) / (8760 · U)`. The fabric term is $0.345 or $0.207 depending on §4, and `U` is the same `A/P` your allocation ledger already measures.

Stated as a rule that matches module 11's phrasing: **fabric cost is inside `r` for a renter and a component of `r` for an owner; it is in `f` only when it arrives as a separate invoice line.** Say which regime you are in before you quote the number, exactly as module 11 requires a rate to arrive with its basis and its date.

One consistency note worth carrying: module 11's `U` (or `A/P`) and this lesson's `U` are the same quantity, and §3(e) showed the fabric rate is hyperbolic in it. So an allocation improvement from module 11 shows up here as a fabric cost reduction with no hardware change at all — which is, per GPU-hour, often larger than anything you can win from a switch vendor.

## Perspectives

**Developer.** A model developer never sees the fabric invoice, but makes the single decision with the largest effect on it: the parallelism plan. How much communication volume is tensor-parallel (stays inside the NVLink island, marginal cost zero) versus data- or pipeline-parallel (crosses the scale-out fabric at $159–$669 per GB/s)? §8 makes that a costed decision rather than a performance-tuning one. The developer-side lever that shows up directly in §5 is the overlap fraction — and the threshold `t_comm > t_compute/overlap`, past which the oversubscription penalty stops being partially hidden and becomes linear, is worth knowing by name.

**Operator / procurement.** Your job is to turn "how much bisection do we need" into a signed BOM line, and §5 says that number is only defensible if it is tied to a named workload's traffic shape and a measured cross-tier fraction `f`. Two things to bring to the meeting. First, the break-even: at 8.5% cross-tier hours the decision flips, so ask what `f` actually is rather than assuming it is small. Second, §4 Result 2: past 1:4 the saving is a rounding error, so if the build is being pushed to 1:8 or beyond, the incremental money is not there and the bandwidth risk is.

**Hardware.** The fixed costs are fixed. NIC plus GPU-facing optics is $3,000/GPU and no topology decision touches it; the leaf floor adds $1,250 more. Everything the oversubscription lever can reach is the $3,700/GPU between full bisection and the floor, and §4 shows most of it is gone by 1:4. Meanwhile power — 148 W/GPU of fabric at full bisection against 700 W of GPU — is 21% on top of the accelerator, which is invisible in a cost model and decisive in a megawatt-limited facility (§9).

**Economics / FinOps.** Know which kind of number you are holding before you cite it. A switch's port count and aggregate Tb/s are datasheet facts you can defend line by line. A street price for a 400G single-mode transceiver is a volatile quote with a date on it. A third-party "RoCE is 2.3× cheaper" TCO estimate is a soft, dated, unreproducible summary of somebody else's assumptions — useful for framing a conversation, indefensible under scrutiny. This lesson deliberately builds from the first kind and parameterises the second, so that the *structure* of the argument survives when every price in it goes stale, which it will within a year.

**Failure mode.** A wrong oversubscription bet does not raise an alert. There is no dashboard tile that reads "spine undersized." It appears as a slower step time or a lower MFU, and the first three hypotheses in most organisations are the model, the data pipeline, and "GPU flakiness" — long before anyone traces the regression to a collective crossing a tier it was never supposed to touch. Being the engineer who can walk that chain backwards, from a step time to a specific tier's bandwidth cap to the procurement decision that created it, is the entire point of this module.

## Real-world use cases

- **The capital recovery factor versus straight-line depreciation.** *What it shows:* the same $7,950 of fabric costs $2,892/year under a 5-year CRF at 10% and $1,590/year under 5-year straight-line — an 82% difference in the annual charge, from a modelling choice alone. *Why it matters:* it is the most common way two people arrive at incompatible fabric TCOs while both being "right." Straight-line is what appears in the accounts (CoreWeave's FY2025 10-K uses six years, straight-line, for technology equipment including GPUs); CRF is what belongs in a purchase decision, because it prices the capital the purchase consumes. State which one you used, every time.

- **Meta's Llama 3 cluster, as a procurement case study.** *What it shows:* a named organisation running frontier-scale training on **RoCE rather than InfiniBand**, with a topology-aware scheduler and an oversubscribed aggregation tier, and publishing an H100-80GB fleet at 700 W TDP consuming 39.3M GPU-hours. *Why it matters:* it is the existence proof that a frontier run does not *require* IB — it requires the software discipline to keep `f` small, which is precisely the term §5's break-even turns on. **Provenance:** the paper itself (arXiv 2407.21783 §3.3.1) could not be fetched from this environment; the 700 W TDP and 39.3M GPU-hour figures are from the Llama 3.1 model card, which was fetched. The specific topology figures (24K GPUs, three Clos tiers, 3,072-GPU non-blocking pods, 1:7 aggregation) are carried forward from lesson 09.2 and are **not independently verified here** — verify them against §3.3.1 before putting them in a document.

- **The DRANET measurement, read as a cost number.** *What it shows:* the upstream Kubernetes network DRA driver reports up to **59.6% higher bus bandwidth for `all_gather` and 58.1% for `all_reduce`** purely from topology-aware GPU-and-NIC scheduling, with no hardware change. *Why it matters:* put that through §5's break-even and it is the same size as the entire fabric decision. A scheduling fix that recovers ~58% of collective bandwidth is worth more per GPU-hour than the gap between a full-bisection and a 1:7 fabric — which is the strongest possible argument that lesson 06's plumbing is a *cost* project, not just a correctness one.

- **Switch capacity quoted two ways.** *What it shows:* the same InfiniBand switch is described as "64 × 400G" and as "51.2 Tb/s" in different places — the first is unidirectional port arithmetic (25.6 Tb/s), the second counts both directions. Spectrum-4's SN5600 is 64 × OSFP 800GbE, quoted as 51.2 Tb/s on the same unidirectional basis. *Why it matters:* mixing the conventions produces a spurious 2× in either direction, and a 2× error in `C_sw/R` moves fabric $/GPU-hour by more than 20% (§3(d)). Before comparing two switches, normalise both to ports × per-port rate, unidirectional, and say so.

## Worked example

**The scenario.** You are asked to sign off the back-end fabric for a **512-GPU H100 cluster** that will run mixed workloads: mostly single-pod data-parallel training, with an unknown fraction of larger jobs that span pods. Two proposals are on the table: full bisection, and 1:4 oversubscription at the spine. Produce a recommendation with a number.

**Step 1 — build the per-GPU BOM for both.** From §2, with `R = 64` and the parameter table:

| | full bisection | 1:4 |
|---|---:|---:|
| NIC | $2,500 | $2,500 |
| GPU-facing optics (2) | $500 | $500 |
| leaf switch share (`C_sw/d`) | $2,500 | $1,563 |
| spine-facing optics (`(1/OS)·2·C_opt_long`) | $1,200 | $300 |
| spine switch share (`(1/OS)·C_sw/R`) | $1,250 | $313 |
| **per GPU** | **$7,950** | **$5,175** |
| **× 512 GPUs** | **$4.07M** | **$2.65M** |
| power per GPU | 148.0 W | 92.5 W |

**Step 2 — convert to $/GPU-hour.** CRF(5 y, 10%) = 0.2638, `m` = 0.10, PUE 1.25, $0.08/kWh, `U` = 0.85 (this fleet's measured allocation ratio, from the module-11 ledger):

```
  full bisection   ($7,950 × 0.3638 + 0.148×1.25×8760×0.08) / (8760 × 0.85)
                 = ($2,892 + $130) / 7,446   =  $0.406 / sold GPU-hour

  1:4              ($5,175 × 0.3638 + 0.0925×1.25×8760×0.08) / (8760 × 0.85)
                 = ($1,883 +  $81) / 7,446   =  $0.264 / sold GPU-hour

  Δ = $0.142 / GPU-hour  =  4.7% of r = $2.99
                         =  $1.06M over 512 GPUs × 5 years at U = 0.85
```

**Step 3 — price the performance risk.** From §5, with the 1:4 ratio instead of 1:7:

```
  t_comm in-pod                            0.710 s
  t_comm across the 1:4 tier  = ×4      =  2.840 s
  overlap cap                 = t_compute = 2.000 s;  0.6 × 2.840 = 1.704 s < 2.0
  step in-pod    = 2.000 + (0.710 − 0.426) = 2.284 s
  step across    = 2.000 + (2.840 − 1.704) = 3.136 s
  δ = 3.136 / 2.284 − 1                    = 0.373   (37% slower)
```

At 1:4 the overlap has not yet saturated, so the penalty is 37% rather than 1:7's 118% — a direct illustration of §5's threshold.

**Step 4 — the break-even, which is the recommendation.**

```
      f ≥ Δfabric / ( [δ/(1+δ)] · r )
        = 0.142 / ( 0.2717 × 2.99 )
        = 0.175

  ⇒  FULL BISECTION PAYS IF MORE THAN ~17.5% OF GPU-HOURS SPAN PODS.
```

**Step 5 — go and measure `f`.** This is the step people skip, and it is the only one that decides the answer. `f` is not an opinion; it is a query against the scheduler's history: of the GPU-hours consumed over the last 90 days, what fraction belonged to jobs whose pods spanned more than one non-blocking domain? Three outcomes:

- **`f` well under 17.5%** (say 5%) → **buy 1:4**, and spend a fraction of the $1.06M saving on making the topology-aware scheduler good enough to keep it there. Add a monitor on `f`; it is now a financial metric.
- **`f` well over 17.5%** (say 40%) → **buy full bisection**, and be able to say why: the oversubscribed fabric would cost `0.40 × 0.2717 × 2.99 = $0.325/GPU-hour` in lost throughput against a $0.142 saving, a net loss of $0.183/GPU-hour, or about $1.36M over five years.
- **`f` near 17.5%** → the decision is a coin flip on capex and should be made on the other axes: blast radius, future workload mix (an MoE shift raises `f` and removes SHARP's help — §7), and how much you trust the scheduler.

**Step 6 — sanity-check against the alternatives you are implicitly rejecting.** The $1.06M saving from 1:4 buys roughly 35 more H100 GPUs at $30,000 each. Is 512 GPUs on a full-bisection fabric worth more than 547 GPUs on a 1:4 fabric? For a comms-bound trainer where `f` is high, no. For a fleet of independent single-pod jobs, obviously yes. **State the fabric decision as this trade**, because it is the trade a CFO can evaluate, and it is the one being made whether or not anyone names it.

**Step 7 — write down what would change the answer.** Three things, each with its effect already computed: SHARP support on a bf16 workload roughly doubles the tolerable `f` (§7); an amortisation life shortened to 3 years raises the fabric charge 36% and therefore the break-even (§3(d)); a shift toward MoE raises `f` *and* removes SHARP, moving both terms the wrong way at once. A recommendation that names its own invalidation conditions is the one that survives the meeting.

## Practice

For the deliverable's procurement section ([network-architecture-read](../practice/network-architecture-read/README.md)), take a **stated job on a stated fabric** and produce the full argument. Show the arithmetic; the point is that someone else can re-run it.

1. **Build the BOM.** Using §2's formulas and your own switch radix and oversubscription ratio, compute fabric capex and power **per GPU** for at least three ratios (include full bisection and your proposed one). Substitute real quotes where you have them and mark the ones you do not. Report the `$ per GB/s of bisection` column and say in one sentence why it moves the direction it does.
2. **Convert to $/GPU-hour.** Apply §3's formula with your own `n`, `i`, `m`, PUE, energy rate, and your fleet's measured `U`. State which amortisation convention you used and why. Report the delta between full bisection and your proposal in $/GPU-hour, as a percentage of your blended rate `r`, and in annual dollars across the fleet.
3. **Placement → bandwidth → step time.** For a named job, compute per-GPU effective all-reduce bandwidth (a) co-located in one non-blocking domain and (b) spread across the oversubscribed tier, then run §5's overlap model to get `δ`. State explicitly whether the overlap saturates, and at what `t_comm` it would.
4. **Compute the break-even `f`** and state, in one sentence, the recommendation it implies. Then say how you would *measure* `f` from your scheduler's history — the actual query or data source, not "we'd look at it."
5. **Sensitivity.** Reproduce §3(d)'s table for your own inputs by varying one parameter at a time. Name the two parameters that move your answer most and what you would do to pin them down.
6. **IB-vs-RoCE verdict** for the same scenario, defended on ≥4 axes (capex delta, subnet-manager opex per GPU-hour at your scale, SHARP's effect on `δ` given your collective mix and dtype, PFC/ECN tuning risk, lock-in). Use §6's decomposition; put a number on the two terms that have one and label the others as judgements.
7. **Oversubscription tolerance.** Argue what ratio *this workload's traffic shape* can tolerate, tied to the parallelism plan: what stays in the NVLink domain, what is pipeline-parallel and light, and what the data-parallel all-reduce actually demands. Reference §4 Result 2 — say where on the diminishing-returns curve your proposal sits.
8. **The power check.** Compute delivered watts per GPU including fabric and PUE, and the GPU count your facility's power envelope supports at each ratio. If your build is power-capped, this may dominate everything above; say so if it does.

**Acceptance:** a written **BOM → $/GPU-hour → step time → break-even `f` → recommendation** chain for a named job and fabric, with a sensitivity table, an IB-vs-RoCE verdict on ≥4 axes, a stated oversubscription tolerance tied to the parallelism plan, and every dollar figure carrying its provenance and date. Done when someone can change one input — the switch quote, the amortisation life, the measured `f` — and re-derive the recommendation without asking you anything.

## Common pitfalls

- **Quoting a fabric TCO without saying which amortisation convention produced it.** *Mechanism:* CRF(5 y, 10%) = 0.2638 against straight-line 1/5 = 0.20 is an 82% difference in the annual capital charge once maintenance is excluded, which is larger than most of the differences people argue about. *Fix:* state `n`, `i`, and whether capital cost is included, every time. Use CRF for a purchase decision and straight-line only when reconciling to the accounts.

- **Justifying oversubscription on "$/GB/s."** *Mechanism:* it moves the wrong way. Total spend falls 40% from 1:1 to 1:7 while cost per delivered GB/s rises 4.2×, because the NIC, the GPU-facing optics and a floor share of the leaf switch are per-GPU costs that oversubscription cannot reach. *Fix:* justify oversubscription as a *workload* claim — "this traffic shape does not need that bisection" — and support it with a measured `f`, not with a cost ratio.

- **Pushing oversubscription past 1:4 for the savings.** *Mechanism:* the curve flattens against an irreducible floor. 1:1 → 1:4 saves $2,775/GPU; 1:4 → 1:7 saves $396 for 1.75× less bisection; 1:7 → 1:16 saves $298 for another 2.3× less. *Fix:* compute the *incremental* saving of each step, not the total, and compare it against the incremental `δ`.

- **Treating "we overlap our collectives" as protection against an undersized spine.** *Mechanism:* overlap is capped by the compute time. Once `t_comm > t_compute/overlap_fraction` (3.33 s in §5's example), every additional second of communication is fully exposed and the penalty becomes linear. A 1:7 tier blows through that threshold, which is why the slowdown is 118% rather than the ~40% people expect. *Fix:* compute the threshold and check which side of it your job lands on.

- **Pricing the fabric as "capex plus power-years" and expecting power to matter.** *Mechanism:* at $0.08/kWh, power is 4.3% of fabric TCO, and moving PUE from 1.25 to 1.50 changes fabric $/GPU-hour by under 1%. Meanwhile amortisation life changes it by 36%. *Fix:* spend your modelling effort on `n`, `i` and cost-per-port. Treat power as a **capacity** constraint (§9), where it genuinely dominates in a megawatt-limited facility, not as a cost line.

- **Comparing switches on raw port count.** *Mechanism:* "128 × 400G vs 64 × 400G" compares an 800G-generation Ethernet switch in breakout mode against a previous-generation 400G InfiniBand switch, and vendors additionally quote aggregate capacity both unidirectionally and bidirectionally for the same box. Either mistake alone produces a spurious 2×. *Fix:* normalise to unidirectional ports × per-port rate, compare same-generation platforms, and note that only **cost per port** enters the model anyway — a switch with twice the ports at twice the price changes nothing.

- **Counting SHARP's benefit without checking it applies.** *Mechanism:* SHARP's effect on `δ` (§7) requires an InfiniBand fabric, a reduction-shaped collective, an offloadable op (`SUM`/`MAX`/`MIN` only), and an offloadable dtype — bf16 offloads only when the fabric reports v3 datatype support. Any one missing and `δ` reverts, taking the break-even from 16.2% back to 8.5%. *Fix:* verify against the NCCL init log (09.5 §11) before the number enters a procurement document, and state the MoE risk explicitly.

- **Double-counting the fabric in the loading multiplier.** *Mechanism:* if you rent GPU capacity at a blended rate, the provider's back-end fabric is already inside that rate; adding this lesson's $/GPU-hour to `f` in `L = (P/A)(1+f)` charges for it twice, exactly as loading a tenant's own idle on top of its allocated hours does (module 11.5). *Fix:* fabric is inside `r` for a renter and a component of `r` for an owner; it belongs in `f` only when it arrives as a separate invoice line.

## Self-check

**1. Derive fabric cost per GPU-hour from first principles, and state which input matters most.**

**Answer:** Per GPU on a rail-optimized two-tier Clos with switch radix `R` and oversubscription `OS`, the leaf carries `d = R·OS/(OS+1)` downlinks, so each GPU consumes one downlink, `1/OS` of a spine-facing link, `1/d` of a leaf switch and `(1/OS)/R` of a spine switch. Capex is therefore `C_gpu = C_nic + 2·C_opt_short + C_sw/d + (1/OS)·2·C_opt_long + (1/OS)·C_sw/R`, with the identical shape for power. Convert with `$/GPU-h = [C_gpu·(CRF + m) + (P_gpu/1000)·PUE·8760·$/kWh] / (8760·U)`, where `CRF = i(1+i)ⁿ/((1+i)ⁿ−1)` prices the capital the purchase consumes and `U` is the fraction of hours the GPU is sold. On the worked parameters (`R`=64, $80k switch, $2,500 NIC, $250/$600 optics, 1,600 W switch, 5 years at 10%, `m`=0.10, PUE 1.25, $0.08/kWh) that is **$7,950 and 148 W per GPU at full bisection → $0.345/GPU-hour**, and **$4,779 and 85 W at 1:7 → $0.207/GPU-hour**. The dominant input is **amortisation life**: 5 y → 3 y raises the rate 36.4%, more than a 50% switch-price increase (+22.6%). Power is nearly irrelevant as a cost — 4.3% of the total, and a PUE change from 1.25 to 1.50 moves the answer 0.9%.

**2. Total fabric spend falls when you oversubscribe, but cost per GB/s of bisection rises. Explain both, and say what follows for procurement.**

**Answer:** Only the uplink-derived terms scale with `1/OS` — the spine-facing optics and the spine switch share — plus a mild improvement in the leaf share as downlinks per switch rise. The NIC, the GPU-facing optics and a floor share of the leaf switch are bought once per GPU regardless of topology, so they form an irreducible floor of `C_nic + 2·C_opt_short + C_sw/R` = **$4,250/GPU** on the worked parameters. Total spend therefore falls from $7,950 to $4,779 (−40%) going 1:1 → 1:7, while per-GPU bisection falls from 50 GB/s to 7.14 GB/s (−86%), so $/GB/s rises from $159 to $669 — a 4.2× increase. What follows: **full bisection is the cheapest bandwidth per unit, just the most of it**, so a $/GB/s argument for oversubscription is backwards; the correct justification is a workload claim. And because the curve flattens against the floor, the incremental saving collapses — 1:1 → 1:4 saves $2,775/GPU while 1:4 → 1:7 saves only $396 for 1.75× less bisection. **Oversubscription past about 1:4 trades a lot of bandwidth for very little money.**

**3. Turn "co-locate this 512-GPU job in one pod" into a bandwidth number, a step time and a dollar number.**

**Answer:** On a 400 Gb/s back-end, co-located in a non-blocking domain each GPU drives ~50 GB/s of line rate (≈45 GB/s achieved bus bandwidth), so aggregate bisection is ≈512 × 400 Gb/s = **204.8 Tb/s**. Spread across a 1:7 aggregation tier the crossing traffic is capped at ~57 Gb/s per GPU, ≈**29 Tb/s** aggregate. For an 8B-parameter bf16 data-parallel step, the ring all-reduce moves `2(511/512) × 16 GB = 31.94 GB` per rank: 0.710 s in-pod, 4.968 s across the 1:7 tier. With 2.00 s of compute and 60% overlap — capped at the compute time — the step is 2.284 s in-pod and 4.968 s spread, a **2.18× slowdown**, because at 1:7 the collective exceeds the compute phase and every extra second is fully exposed. In dollars: co-location costs $0 incremental fabric and spends bisection already bought, while the alternative that would make spreading safe — full bisection — costs $0.138/GPU-hour more, about $700k/year across 512 GPUs. So co-location converts a cheap fabric into an expensive-fabric experience for traffic that stays inside the blast radius, and the scheduling work that guarantees it is the cheapest bandwidth on the price list.

**4. Give the break-even condition for buying a better fabric, and show how SHARP changes it.**

**Answer:** Let `f` be the fraction of GPU-hours whose jobs span the oversubscribed tier, `δ` the slowdown they suffer there, and `r` the blended $/GPU-hour. Full bisection reclaims `f·δ/(1+δ)` GPU-hours per fleet GPU-hour, so it pays when **`f · [δ/(1+δ)] · r ≥ Δfabric`**. With `Δfabric = $0.138`, `r = $2.99` and `δ = 1.176` (the 1:7 case above), `f ≥ 0.138/(0.5405 × 2.99) = 0.085` — **full bisection pays above ~8.5% cross-tier hours.** SHARP reduces wire bytes by `2(N−1)/N = 1.996` at N=512, halving both collective times: the in-pod step becomes 2.142 s and the 1:7 step 2.996 s, so `δ` falls to 0.398 and the break-even rises to **16.2%**. SHARP therefore roughly doubles the cross-tier fraction an oversubscribed fabric can absorb before the expensive fabric wins — but only on InfiniBand, only for `SUM`/`MAX`/`MIN`, and for bf16 only when the fabric reports v3 datatype support. On an all-to-all-heavy MoE workload the term is zero and the break-even reverts to 8.5%.

**5. Why is scale-up bandwidth "free" at the margin when it clearly is not free, and what should you do about it?**

**Answer:** NVLink bandwidth costs real money — the NVSwitch silicon is inside the ~$30,000/GPU server line — but it is **not marginal**: there is no SKU in which you buy less NVLink for less money, so no decision trades NVLink GB/s against dollars. Scale-out bandwidth *is* marginal: every leaf uplink you delete is money you keep, which is the whole oversubscription lever. So the two marginal rates are `$0` for scale-up and `$159–$669 per GB/s` for scale-out depending on the ratio, even though the physical bandwidth ratio (~18:1 per GPU) has been flat from NVLink 4/400G to NVLink 5/800G. The consequence is that **the parallelism plan is a cost plan**: putting tensor parallelism inside the 8-GPU NVLink island and reserving the scale-out fabric for lighter pipeline- and data-parallel traffic maximises use of bandwidth you cannot decline to buy and minimises demand on the tier where every GB/s is priced. When you argue an oversubscription ratio, you are implicitly asserting that the parallelism plan keeps most bytes off the scale-out fabric — so state the parallelism plan as part of the fabric argument.

**6. Where does the fabric cost belong in module 11's `L = (P/A)(1+f)`?**

**Answer:** Usually **not in `f`**. If you rent GPU capacity at a blended rate `r`, your provider's back-end fabric is already inside `r` — it was in their BOM, amortisation and margin — so adding this lesson's $0.345/GPU-hour to `f` charges for it twice, exactly the double-count module 11.5 corrects when it replaces `L = 1/utilisation` with `L = (P/A)(1+f)`. What belongs in `f` is only fabric spend you are *separately invoiced* for: egress, a managed-interconnect line, a dedicated interconnect charge. If you own or colocate, there is no `r` to hide inside and this lesson's number is how you *build* `r`: `r = (GPU + server + fabric + power + facility charges) / (8760·U)`. Note also that the `U` here and module 11's `A/P` are the same quantity, and the fabric rate is hyperbolic in it — a fleet at `U = 0.55` pays 82% more per sold GPU-hour for identical hardware than one at `U = 1.0`, which means an allocation improvement is a fabric cost reduction with no hardware change at all.

## Connections & what's next

This lesson closes the module's arc, so name the whole thread. **02b** gave you the topology matrix inside one node. **L1–L2** extended it past the NIC into the inter-node fabric — Clos structure, rail optimization, oversubscription. **L3–L5** covered the protocol riding that fabric — RDMA's kernel bypass, IB vs RoCEv2 and lossless Ethernet, GPUDirect's engage conditions and SHARP's in-network reduction. **L6** was the Kubernetes mechanism that turns a placement decision into an enforced pod constraint. This lesson is the **cost layer** on all of it: the same topology, protocol and placement facts, re-expressed as $/GPU-hour, a break-even, and a recommendation. Every number here traces to an earlier lesson — bisection from L2, the IB/RoCE axes from L4, SHARP's `2(N−1)/N` from L5, and the placement mechanism from L6 that makes an oversubscribed fabric safe. It also joins outward: `U` is module 11's allocation ratio, `r` is module 11's blended rate, and §10 fixes exactly where the fabric sits in `L = (P/A)(1+f)`.

What is next is not another lesson — it is the module's proof of work. Produce the **[Network architecture read](../practice/network-architecture-read/README.md)**: redraw a real published topology with per-tier bisection and oversubscription labelled, predict where a named job bottlenecks under two placements and quantify the penalty, make the co-location argument with real bandwidth numbers, defend an IB-vs-RoCE verdict on ≥4 axes, argue an oversubscription tolerance for the workload, and ground it in a real 2-GPU `nccl-tests all_reduce_perf` capture. Then close the module against the **[checkpoint](../checkpoint.md)**, which gates on exactly this chain: read a `topo -m`, compute an oversubscription ratio, explain lossless RoCE, argue IB vs RoCE two ways, define rail-optimized, trace GPUDirect end to end, explain the Kubernetes path, and — the module's whole point — turn a topology into a bandwidth number and a placement argument into a dollar figure.

## References & further reading

**Primary sources — read directly and relied upon**

1. **NVIDIA NCCL, source tree at v2.31.2-1** — https://github.com/NVIDIA/nccl — cloned and read. Used here indirectly, via lesson 09.5: `NCCL_BUFFSIZE` (4 MiB) and `NCCL_STEPS` (8) set the chunk size behind the collective-time model, and `ncclTopoCheckGdr`'s level ladder is what makes the achieved-bandwidth assumption in §5 conditional on GDR actually engaging.

2. **`nccl-tests`, `src/all_reduce.cu`** — https://github.com/NVIDIA/nccl-tests — cloned and read. `busBw = algBw × 2(N−1)/N` for all-reduce. This is the factor behind §5's `31.94 GB` of per-rank wire traffic and §7's `1.996` SHARP ratio at N=512, and it is also the reason a busbw figure above the link's line rate indicates in-network reduction rather than a faster link.

3. **`nccl-rdma-sharp-plugins`, `src/sharp_plugin.c`** — https://github.com/Mellanox/nccl-rdma-sharp-plugins — cloned and read. The offloadable reduction ops (`SUM`, `MAX`, `MIN` only) and the `sharp_coll_caps_query()` gate on bf16/int8/uint8 — the two conditions §7 attaches to SHARP's effect on `δ`, and the reason a bf16 workload can see none of it.

4. **stas00, *ml-engineering*, network chapter** — https://github.com/stas00/ml-engineering — cloned and read (commit 139708e), with its own vendor citations. Switch platforms: Quantum-2 at 64 × 400G (NDR), Quantum-X800 at 800G with SHARPv4, Spectrum-4 SN5600 at 64 × OSFP 800GbE / 51.2 Tb/s / 2U and SN5400 at 64 × 400GbE. Adapters: ConnectX-7 at 400 Gb/s, ConnectX-8 at 800 Gb/s. InfiniBand generations (EDR/HDR/NDR/XDR 4× port rates). Also the warning that switch throughput is fabric capacity and must not be compared against per-node injection bandwidth — the basis for §6's port-density correction.

5. **Meta, Llama 3.1 model card** — https://github.com/meta-llama/llama-models — fetched and read. H100-80GB at **700 W TDP** and **39.3M GPU-hours** of training compute — the GPU power figure used in §9's power-envelope calculation. The card explicitly does not disclose GPU count or network topology; see entry 10 for the paper.

6. **DRANET (`kubernetes-sigs/dranet`)** — https://github.com/kubernetes-sigs/dranet — cloned and read (commit 9760b79). The project's own reported **59.6% `all_gather` / 58.1% `all_reduce`** bus-bandwidth improvement from topology-aware GPU-and-NIC scheduling, used in Real-world use cases as a scale comparison for the fabric decision itself.

7. **Course-internal: module 11 lesson 05, unit economics** — [`../../11-gpu-cost-economics/lessons/05-unit-economics.md`](../../11-gpu-cost-economics/lessons/05-unit-economics.md) — the loading multiplier `L = (P/A)(1+f)`, the double-count it corrects, the `EffectiveCost` rate basis, and the module's worked blended rate of **$2.99/GPU-hour** used throughout this lesson's break-even arithmetic. §10 states precisely when fabric cost belongs in `f` and when it double counts, following that lesson's own discipline.

8. **Course-internal: module 11 lesson 06, commitment strategy** — [`../../11-gpu-cost-economics/lessons/06-commitment-strategy.md`](../../11-gpu-cost-economics/lessons/06-commitment-strategy.md) — CoreWeave's FY2025 Form 10-K useful life for technology equipment including GPUs: **six years, straight-line**. The calibration point for §3(a)'s amortisation discussion and for the CRF-versus-straight-line contrast in Real-world use cases.

9. **Course-internal: module 09 lessons 01, 02, 04 and 05** — the rail model and the 2-8-9-800 reference NIC count (01), Clos structure and oversubscription (02), the IB/RoCE mechanism axes (04), and SHARP's `2(N−1)/N` derivation and engage conditions (05). This lesson prices what those establish and does not re-derive them.

**Optional depth — could not be fetched from this environment, and no claim in this lesson rests on them**

10. **Meta, "The Llama 3 Herd of Models", §3.3.1 Network** — https://arxiv.org/abs/2407.21783 — `arxiv.org` is blocked by this environment's egress proxy. **Not relied upon.** The topology figures attributed to it elsewhere in this module (24K-GPU RoCE cluster, three Clos tiers, non-blocking 3,072-GPU pods, 1:7 aggregation oversubscription, topology-aware scheduling) are carried forward from lesson 09.2 and are **not independently verified here** — read §3.3.1 yourself before putting those numbers in a procurement document. The 700 W TDP and 39.3M GPU-hour figures used in §9 come from the model card (entry 5), which was fetched.

11. **NVIDIA, HGX AI Factory enterprise reference architecture** — https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/ — `docs.nvidia.com` is blocked. **Not relied upon.** The "2-8-9-800" NIC-count detail referenced in §2 is carried forward from lesson 09.1, not verified here. If you can reach it, read it for the vendor's current full-bisection, rail-optimized design point and note *which fabric* it specifies, since "the ideal fabric" is no longer automatically an InfiniBand answer and that changes what you are contrasting an oversubscribed build against.

12. **SemiAnalysis, "100,000 H100 Clusters: Power, Network Topology, Ethernet vs InfiniBand, Reliability, Failures, Checkpointing"** — https://newsletter.semianalysis.com/p/100000-h100-clusters-power-network — not fetched from this environment. **Not relied upon.** Referenced in §4b only for the structural point that a 100K-GPU build chooses between a fully non-blocking four-tier Clos and internally-non-blocking pods joined by an oversubscribed top tier — a claim the model in §4b derives independently. Any specific pod sizes, ratios or port-density figures attributed to it should be read there before you cite them.

**A standing note on the numbers.** Every dollar and watt in this lesson is a **stated parameter**, not a measured fact: switch, NIC and optics prices are order-of-magnitude figures chosen so the arithmetic is auditable, and vendor datasheet pages for transceiver power were unreachable from this environment (the 14–16 W figure mentioned for 800G OSFP DR8 modules comes from search-result summaries and is flagged as such, not relied upon for any calculation). The model's only external validation is that it reproduces the widely-quoted 10–20% fabric share of system cost from independent inputs. **Substitute your own quotes and re-run** — that is why §2 and §3 are given as formulas with a sensitivity table rather than as a conclusion. What survives a hardware generation is the structure: fabric cost per GPU is a floor plus a `1/OS` term; amortisation life dominates the rate; oversubscription raises $/GB/s while lowering total spend and exhausts itself around 1:4; and the decision is settled by a break-even on the fraction of GPU-hours that cross the tier.
