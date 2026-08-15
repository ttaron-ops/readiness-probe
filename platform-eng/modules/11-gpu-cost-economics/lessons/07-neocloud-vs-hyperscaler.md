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
sources: 10
---

# Decomposing the neocloud-vs-hyperscaler price gap

## Where this fits

Lesson 06 gave you the commitment and procurement mechanics — coverage math, spot vs. reserved, the build-vs-buy-vs-rent continuum — for capacity you've already decided to acquire. This lesson sits *upstream* of that decision, and upstream of everything else in the module: before you can attribute a GPU-hour (lessons 01-05) or commit to buying it (lesson 06), you have to know what the **rate itself** is actually made of. Two "$X/GPU-hr" quotes from two vendors are rarely comparable on their face, and mistaking a sticker gap for a cost-structure fact is how procurement recommendations go wrong before the commitment math even starts. This lesson gives you the decomposition and normalization procedure that makes the number you feed into lesson 06 — and eventually into lesson 08's internal chargeback rate — defensible.

## Why this matters

You will, within your first quarter on a GPU platform team, be handed two quotes for the "same" H100 and asked which to buy. One is from a hyperscaler (AWS `p5`, GCP `a3`, Azure `ND H100 v5`) at a headline on-demand rate; the other is from a neocloud (CoreWeave, Lambda, Crusoe, Nebius, Together, Voltage Park) at roughly half the sticker. The naive move — pick the cheap sticker — is how platform teams end up with a "cheap" fleet that can't run multi-node training because the interconnect was never in the quote, or a surprise five-figure egress bill, or a cluster that's down 6% of the month because the control plane is immature. The gap is real, but it is a *decomposable* gap, and your job is to explain it, not to be surprised by it.

As a 2026 snapshot: hyperscaler on-demand H100 lists roughly **$3-5+/GPU-hr** (AWS `p5.48xlarge` on-demand pencils out well above $5/GPU-hr at list; committed and Savings Plans pull it down); neoclouds advertise roughly **$2-2.6/GPU-hr**, and the cheapest committed neocloud or spot capacity can sit **3-6x below** hyperscaler on-demand list. Treat every dollar figure in this lesson as a mid-2026 snapshot — the ratios and the *reasons* are the durable content; the absolute numbers rot in months as B200/GB200 supply floods in and H100 depreciates.

The stake is that GPU spend is now the single largest line in most AI infra budgets, and procurement decisions are made on multi-year commitments (lesson 06). A 2x sticker error compounded over a 3-year reserved commit on a 512-GPU fleet is eight figures. Getting the decomposition right is the difference between a defensible procurement recommendation and an expensive learning experience.

The other reason this matters: the decomposition *is* the interview question. "Hyperscaler quotes $X, neocloud quotes $Y — which do you pick?" A junior answer picks the cheap number. A senior answer says "neither yet — normalize both to fully-loaded $/GPU-hr at our utilisation and interconnect needs, then decide," and then actually walks the axes.

## What's new here (calibration)

- You already know per-pod GPU attribution (04/05), the util-lie (05), and capex-vs-cloud crossover (10). This lesson is upstream of all of them: it's about the **rate you're attributing in the first place**, and why two "H100-hours" can differ 3x in true cost.
- New: a **price-gap decomposition framework** — seven axes that explain the sticker delta, each of which either shrinks the gap (it was bundling) or is real savings you can bank.
- New: a **normalization procedure** — reduce any offer to *fully-loaded $/GPU-hr including networking, storage, egress, and support, adjusted for your utilisation and commitment risk* — so that A-vs-B becomes a single comparable number instead of two incommensurable stickers.
- Not re-taught: what an H100 is, what committed-use discounts are (lesson 06 owns commitment mechanics), what MFU/cost-per-run are (lesson 08 owns work-normalized cost). Here we stop at fully-loaded $/GPU-hr; converting that to $/unit-of-work is the earlier modules' job.

## Core concepts

### The observed fact (2026 snapshot)

| Offer class | Headline H100 $/GPU-hr | What's in the number |
|---|---|---|
| Hyperscaler on-demand list | ~$3-5+ | GPU + fast fabric + support + integrated storage/networking + margin |
| Hyperscaler 1-3yr committed / Savings Plan | ~$2-3.5 | Same bundle, commitment discount |
| Neocloud on-demand | ~$2-2.6 | GPU + *maybe* fabric; storage/egress/support often extra |
| Neocloud committed (long term) | ~$1.5-2.2 | Bare GPU-hours, everything else unbundled |
| Spot / preemptible / marketplace | ~$1-2 | No availability guarantee |

The headline gap between hyperscaler on-demand list and the cheapest committed neocloud is routinely 3-6x. That number is a **starting point for analysis, not a conclusion**.

### The decomposition axes

Walk these in order. Each axis either *explains away* part of the gap (it was never apples-to-apples) or *confirms* it as bankable savings.

**(a) Margin & bundling.** Hyperscalers price the GPU-hour as an all-in service: managed control plane, integrated VPC networking, support tiers, and — critically — the rate absorbs a lot of adjacent-service margin. Neoclouds sell bare GPU-hours and monetize the extras separately (or thinly). Much of the sticker gap is that you are comparing a bundle to an unbundled unit. Re-bundle before comparing.

**(b) Capital structure & depreciation.** Neoclouds run a different financial model: aggressive depreciation schedules, high target utilisation (they *must* keep GPUs busy to service GPU-backed debt), and thinner overhead. They are single-product companies optimizing one SKU. This is real structural cost advantage, not an accounting trick — part of the gap is genuinely lower cost-to-serve. It's also analyst-scrutinized in public filings: CoreWeave's IPO S-1 disclosed a **6-year straight-line depreciation schedule** for its GPU/technology equipment against a **weighted-average committed-contract duration of roughly 4 years** at the time of filing (2024 data) — a real, load-bearing tension between how fast the capital is written off and how long the revenue backing it is actually locked in (see Real-world use cases below). A shorter depreciation window (some rivals use 4 years) would raise reported cost and shrink the apparent advantage; the schedule chosen is itself a lever on the sticker price, not a neutral fact.

**(c) Networking / interconnect.** The load-bearing axis. A single H100 is fungible; a *training cluster* needs rail-optimized InfiniBand (NDR 400G) or equivalent RoCE fabric with SHARP/rail topology (ref lesson 09 on collectives and fabric). Ask of every quote: **is non-blocking IB/rail-optimized fabric in the rate, or a separate line, or unavailable?** A $2/GPU-hr rate with only standard Ethernet is worthless for multi-node training — your all-reduce chokes and MFU collapses. Some neoclouds include IB in the headline (CoreWeave, Crusoe rail-optimized clusters); others sell it as a premium tier or only in large reserved blocks. If IB is extra, add it back before comparing. This is the axis most likely to silently erase the entire savings story, because a missing-fabric mistake doesn't show up as a billing surprise — it shows up as a stalled training run weeks later.

**(d) Storage & egress.** Frequently unbundled and a hidden cost. High-throughput parallel storage (VAST, WEKA, Lustre) for checkpoint/dataset I/O may be a separate SKU. **Egress** is the classic trap: hyperscaler egress is expensive but predictable; some neoclouds offer free/cheap egress (a real advantage for serving), others meter it. If your workload ships checkpoints or serves tokens out, model egress explicitly — it can dominate for inference-heavy patterns.

**(e) Utilisation & committed term.** The headline low neocloud rate almost always requires a long commitment (6-36 months) or is spot. Lesson 06 owns the commitment mechanics; here the point is that **commitment converts a variable cost into a fixed one**, and the "savings" only materialize if your utilisation is high enough to beat on-demand. A cheap committed rate at 30% fleet utilisation is more expensive per unit of work than a dearer on-demand rate at 90%. Normalize by *your expected utilisation*, not the vendor's implied 100%.

**(f) Reliability, SLA & support.** Neoclouds may have thinner SLAs, less mature control planes, slower node-replacement on GPU failure, and smaller support orgs. This is why bare-metal lifecycle and node-health skills matter (mod 10) — on a neocloud you may *be* the SRE for the hardware. Independent buyer's-guide ratings exist precisely because this axis is hard to see from a pricing page alone: SemiAnalysis's ClusterMAX rating system scores dozens of neoclouds on reliability, orchestration, monitoring, and support maturity, not just price (see Real-world use cases below). Quantify: expected uptime and failed-node MTTR feed effective-availability, which inflates your true $/GPU-hr (you pay for hours you can't use). A 96%-effective fleet at $2 costs the same per usable hour as a 99.9%-effective fleet at ~$1.93.

**(g) Total cost of serving.** The synthesis: a fair comparison normalizes both offers to **$/unit-of-work at your utilisation**, not $/GPU-hr sticker. Sticker is the input; fully-loaded, utilisation-adjusted, availability-adjusted cost is the decision variable.

### The normalization framework

To compare offer A vs B, compute for each:

```
fully_loaded_$/GPU-hr =
    ( base_rate
      + interconnect_addon        # 0 if IB bundled; else amortized fabric cost
      + storage_addon             # parallel FS $/GPU-hr equivalent
      + egress_addon              # expected egress $ / expected GPU-hrs
      + support_addon )           # support tier $ / GPU-hrs
    / effective_availability      # e.g. 0.97 for a 97%-usable fleet
    / utilisation_realization     # your expected busy fraction vs committed hours
```

The last two divisors are where committed cheap offers get punished: if you commit to hours you don't use, the *effective* rate per unit of work climbs. Report the sticker, the fully-loaded rate, and the utilisation-adjusted rate as three columns so the gap's collapse (or persistence) is visible.

The decision rule: **the sticker gap is your hypothesis; the utilisation-adjusted fully-loaded gap is your answer.** If it survives normalization, bank it. If it collapses, you found the bundling.

## Perspectives

**Buyer / procurement.** Your job is not to find the cheapest sticker; it's to normalize every offer to the same fully-loaded, utilisation-adjusted basis before a decision is made. The sticker gap is a **hypothesis** — "maybe neocloud N is 3x cheaper" — not an answer. Treat every quote as an invitation to ask "what's not in this number," and don't let a procurement clock (vendor urgency, end-of-quarter discounting) short-circuit the normalization step.

**Networking / HPC.** From the fabric engineer's chair, axis (c) is the whole story. A GPU-hour without rail-optimized interconnect isn't a discount on the same product — it's a different, cheaper product that happens to share a part number with the one you need. The savings that look real on a spreadsheet evaporate the moment a multi-node all-reduce hits standard Ethernet and MFU collapses; the bill doesn't go up, but the *value received per dollar* does, silently, without anyone getting an alert.

**Financial structure.** From a capital-markets view, axis (b) is where genuine, bankable advantage lives — and it's visible in public filings, not just marketing. Depreciation schedule and capital structure (debt-financed silicon, single-SKU focus, high forced utilisation to service that debt) explain real cost-to-serve differences that persist even after every bundling adjustment. The discipline here is distinguishing *this* axis, which survives normalization, from axis (a), which usually doesn't.

**Reliability / SRE.** Effective availability is a cost line, not a footnote. On a neocloud with a thinner support org, "who replaces the failed node at 2am" is frequently *you* — and that staffing time is a real cost that never appears on the vendor's invoice. The self-SRE overhead line in this lesson's worked example is exactly this perspective made numeric: hardware-lifecycle labor (mod 10) hidden inside what looks like a pure GPU-hour rate.

## Real-world use cases

- **CoreWeave, Form S-1 (SEC EDGAR)** — <https://www.sec.gov/Archives/edgar/data/1769628/000119312525044231/d899798ds1.htm> — what it shows: a real, analyst-scrutinized instance of axis (b)'s capital-structure tension. The filing disclosed 6-year straight-line depreciation on GPU/technology equipment against a roughly 4-year weighted-average committed-contract duration at filing time — the depreciation clock runs longer than the revenue commitments backing it, a structural detail that changes how "cheap" the underlying capacity really is once you account for how the balance sheet, not just the sticker, is built.
- **SemiAnalysis, ClusterMAX rating system** — v1: <https://semianalysis.com/2025/03/26/the-gpu-cloud-clustermax-rating-system-how-to-rent-gpus/>, v2 (Nov 2025): <https://newsletter.semianalysis.com/p/clustermax-20-the-industry-standard>, live ratings: <https://www.clustermax.ai/> — what it shows: the industry-standard independent buyer's guide for neoclouds, scoring dozens of providers (84 in the November 2025 "2.0" update) across security, networking, storage, reliability, monitoring, and pricing — not price alone. CoreWeave is the only Platinum-rated provider in both the v1 and v2 cycles, a concrete signal for axis (f) that a cheap sticker and a mature, reliable platform are not the same claim.
- **AWS `p5` (H100) instance pricing** — <https://aws.amazon.com/ec2/instance-types/p5/> — and **Google Cloud A3 (H100) VMs** — <https://cloud.google.com/compute/docs/accelerator-optimized-machines#a3-vms> — what it shows: the official rate cards that anchor the hyperscaler side of every normalization sheet you'll build — the "what's bundled" baseline (integrated fabric, storage, support) that axis (a) asks you to strip out and compare against a neocloud's unbundled price.

## Worked example

Two real-shaped quotes for a 64-GPU H100 training workload, target 80% utilisation, multi-node → **needs InfiniBand**. Ships ~40 TB/month egress (checkpoints to a hyperscaler-hosted eval pipeline). All figures 2026-snapshot, illustrative.

**Offer H (hyperscaler, on-demand list):** $4.90/GPU-hr. IB-class fabric bundled. Integrated storage bundled. Support (Business) bundled. Egress metered at ~$0.08/GB. Effective availability 99.9%.

**Offer N (neocloud, 12-mo committed):** $2.05/GPU-hr base. Rail-optimized IB available *only* on the reserved cluster tier — included here. Parallel FS: +$0.25/GPU-hr. Egress: free/cheap (~$0.01/GB). Support: standard, thinner SLA; model effective availability 97.5% and add a self-SRE overhead of +$0.15/GPU-hr (your on-call time absorbing node failures, ref mod 10).

Egress addon per GPU-hr: 40 TB/mo = 40,960 GB. Fleet GPU-hrs/mo at 80% of 64 GPUs = 64 * 730 * 0.80 ≈ 37,376 GPU-hr.
- Offer H egress: 40,960 * $0.08 = $3,277/mo → /37,376 = **+$0.088/GPU-hr**
- Offer N egress: 40,960 * $0.01 = $410/mo → /37,376 = **+$0.011/GPU-hr**

| Line | Offer H | Offer N |
|---|---|---|
| Base rate | $4.90 | $2.05 |
| + Interconnect | 0 (bundled) | 0 (in reserved tier) |
| + Storage | 0 (bundled) | +$0.25 |
| + Egress | +$0.088 | +$0.011 |
| + Support / self-SRE | 0 | +$0.15 |
| Fully-loaded $/GPU-hr | **$4.99** | **$2.46** |
| / effective availability | /0.999 = $4.99 | /0.975 = $2.52 |
| Utilisation-adjusted (÷0.80 committed-hour realization) | on-demand, no penalty → $4.99 | committed hours you pay regardless → $2.52 / 0.80* |

\* The commitment subtlety: on Offer N you pay for 12 months of reserved hours whether or not you hit 80%. If actual utilisation lands at 80% as planned, the reserved rate holds at ~$2.52 usable. If it slips to 50%, effective usable cost balloons to ~$4.03 — closing most of the gap. On-demand Offer H has no such exposure.

**Decision:** At the planned 80% utilisation, Offer N is fully-loaded ~$2.52 vs Offer H ~$4.99 — a genuine ~2x that *survives normalization*. The 3-6x headline shrank to ~2x once IB, storage, egress, support, and availability were loaded in, and the residual 2x is real structural savings (axis b) plus the on-demand-vs-committed premium. **Pick N — but only with a utilisation floor**: the recommendation is conditional on hitting ~70%+ utilisation, because below ~55% the gap collapses and the hyperscaler's zero commitment risk and thicker SLA win. Attach that utilisation floor to the procurement memo; it's the load-bearing assumption.

## Practice

Feeds [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md): take two vendor quotes (one hyperscaler committed, one neocloud committed) for an H100 or H200 cluster and build the normalization sheet — fully-loaded $/GPU-hr including interconnect, storage, egress, support, and both an effective-availability and a utilisation-realization adjustment. Produce the three-column output (sticker / fully-loaded / utilisation-adjusted), identify which decomposition axis explains the largest share of the gap, and write the one-paragraph procurement recommendation with its explicit utilisation floor. Deliver the sheet and the memo.

## Common pitfalls

- **Comparing sticker rates without normalizing for bundling.** The central thesis of this lesson, and the top pitfall: a $1.80 neocloud rate and a $4.50 hyperscaler rate are not the same product until you've re-bundled them onto the same axes. Correction: always produce the three-column sheet (sticker / fully-loaded / utilisation-adjusted) before recommending anything.
- **Assuming the cheap neocloud rate is available on-demand at that price.** The headline low number is usually gated behind a long commitment or is spot-only with no availability guarantee. Correction: check the offer class row in the observed-fact table above — on-demand and committed neocloud rates can differ by 30-50%.
- **Ignoring effective availability / MTTR in the comparison.** A lower sticker at lower uptime can cost the same or more per usable hour — the "96%-effective at $2 ≈ 99.9%-effective at ~$1.93" framing from axis (f). Correction: always divide by effective availability, not just by nameplate GPU count.
- **Assuming the whole gap is structural cost advantage rather than partly bundling/margin.** A 5x sticker gap that collapses to 1.1x after normalization was mostly bundling (axes a/d), not a genuine capital-structure edge (axis b). Correction: don't bank the headline ratio — bank only the ratio that survives the full seven-axis walk.
- **Recommending a vendor without an attached utilization floor.** The lesson's own worked example gets this right by stating "Pick N, conditional on ≥70% utilisation" — the omission of that floor is exactly the unstated assumption that turns a good memo into an expensive surprise six months later when utilisation slips and the "savings" evaporate. Correction: every procurement recommendation on committed capacity must state the minimum utilisation at which the recommendation still holds.

## Self-check

- A neocloud quotes $1.80/GPU-hr and a hyperscaler quotes $4.50 on-demand for H100. Your teammate says "obvious, take the neocloud, we save 60%." What's the first question you ask before agreeing? **Answer:** "Is rail-optimized InfiniBand in that $1.80, and what's our expected utilisation and commitment term?" Interconnect (axis c) and utilisation/commitment (axis e) are the two axes most likely to erase the savings — a bare-Ethernet rate is useless for the multi-node training this presumably is, and a committed rate at low utilisation isn't actually cheaper per unit of work.
- Why does charging your organization's true cost require dividing the fully-loaded rate by effective availability *and* by utilisation realization, rather than just using the base rate? **Answer:** You pay for hours the fleet is unusable (node failures → effective availability < 1) and, on committed capacity, for hours you reserved but didn't fill (utilisation realization < 1). Both inflate the true cost *per unit of delivered work* above the sticker; ignoring them makes a cheap committed neocloud look better than it is at low utilisation.
- After normalizing, a 5x sticker gap collapses to 1.1x. What does that tell you, and what should the procurement recommendation be? **Answer:** The gap was almost entirely bundling and adjacent-service margin (axes a/d), not structural cost advantage — the neocloud's cheap sticker was a bare GPU-hour and the hyperscaler's rate absorbed IB, storage, egress, and support you'd otherwise buy separately. At 1.1x fully-loaded, recommend the hyperscaler: the tiny residual saving doesn't justify the thinner SLA, immature control plane, and commitment risk (axes e/f).
- A neocloud's reserved-cluster tier bundles rail-optimized InfiniBand, but its pure on-demand tier is standard Ethernet only. Which decomposition axis governs this, and why does it matter for a multi-node training decision specifically? **Answer:** Axis (c), networking/interconnect. It matters because the fabric requirement is workload-shaped, not price-shaped: a training job that needs a non-blocking all-reduce across nodes will stall or collapse in MFU on standard Ethernet regardless of how cheap the GPU-hour is, so the "cheap" on-demand tier is not a substitutable option for this workload at any price — you must price the reserved (IB-bundled) tier or add the fabric cost back explicitly.

## Connections & what's next

This lesson's fully-loaded, utilisation-adjusted rate is the number that lesson 06's commitment math should have been feeding on all along — normalize first, then decide coverage. It flows directly into **lesson 08**, where this same rate becomes the input to the internal **blended chargeback rate** that teams get billed against, and into **lesson 10**, where it populates the cost columns of the FOCUS-aligned schema. Next: **lesson 08 — Chargeback, showback & queue-wait billing** takes the external rate you just normalized and asks the next question: once you own or commit to capacity at this rate, who inside the org pays for it, and how do you make the incentives work?

## References & further reading

**Primary sources**
- AWS `p5` (H100) instance and pricing reference — <https://aws.amazon.com/ec2/instance-types/p5/> — read for the official hyperscaler on-demand rate card used as the Offer H baseline.
- Google Cloud A3 (H100) VM documentation — <https://cloud.google.com/compute/docs/accelerator-optimized-machines#a3-vms> — read for the GCP equivalent bundle (fabric, storage, support) that anchors axis (a).
- CoreWeave pricing and instance/fabric documentation — <https://www.coreweave.com/pricing> — read for a neocloud's own framing of what is and isn't bundled into its headline rate.
- Lambda GPU cloud pricing — <https://lambdalabs.com/service/gpu-cloud> — read for a second neocloud reference point on on-demand vs. reserved pricing structure.

**Real-world engineering blogs**
- CoreWeave, Form S-1 (SEC EDGAR) — <https://www.sec.gov/Archives/edgar/data/1769628/000119312525044231/d899798ds1.htm> — what it shows: the 6-year depreciation vs. ~4-year weighted-average contract-duration tension, a real capital-structure instance of axis (b).
- SemiAnalysis, "The GPU Cloud ClusterMAX Rating System" (v1) — <https://semianalysis.com/2025/03/26/the-gpu-cloud-clustermax-rating-system-how-to-rent-gpus/> — what it shows: the original methodology for rating neoclouds on reliability/support/networking, not just price.
- SemiAnalysis, "ClusterMAX 2.0: The Industry Standard GPU Cloud Rating System" — <https://newsletter.semianalysis.com/p/clustermax-20-the-industry-standard> — what it shows: the November 2025 update rating 84 neoclouds, with CoreWeave the sole repeat Platinum provider — a live signal for axis (f).

**Deeper dives**
- ClusterMAX live ratings and per-provider reviews — <https://www.clustermax.ai/> — browse individual neocloud scorecards across the ten rated criteria.
- Crusoe reference-cluster and InfiniBand fabric specs — <https://crusoe.ai/cloud/> — a concrete example of a neocloud's rail-optimized fabric offering (axis c).
- Nebius pricing — <https://nebius.com/prices> — a second neocloud's bundling choices to practice the decomposition against.

[💰 11 — GPU cost and unit economics](../README.md)
