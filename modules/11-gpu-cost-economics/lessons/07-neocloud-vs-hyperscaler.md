---
lesson: 07
title: "Decomposing the neocloud-vs-hyperscaler price gap"
module: 11
concept: "price-gap decomposition"
status: not-started
est_time: "3 hrs"
artifacts: ["a fully-loaded $/GPU-hr normalization sheet comparing two real-shaped H100 quotes at your utilisation"]
---

# Decomposing the neocloud-vs-hyperscaler price gap

## Why this matters

You will, within your first quarter on a GPU platform team, be handed two quotes for the "same" H100 and asked which to buy. One is from a hyperscaler (AWS `p5`, GCP `a3`, Azure `ND H100 v5`) at a headline on-demand rate; the other is from a neocloud (CoreWeave, Lambda, Crusoe, Nebius, Together, Voltage Park) at roughly half the sticker. The naive move — pick the cheap sticker — is how platform teams end up with a "cheap" fleet that can't run multi-node training because the interconnect was never in the quote, or a surprise five-figure egress bill, or a cluster that's down 6% of the month because the control plane is immature. The gap is real, but it is a *decomposable* gap, and your job is to explain it, not to be surprised by it.

As a 2026 snapshot: hyperscaler on-demand H100 lists roughly **$3-5+/GPU-hr** (AWS `p5.48xlarge` on-demand pencils out well above $5/GPU-hr at list; committed and Savings Plans pull it down); neoclouds advertise roughly **$2-2.6/GPU-hr**, and the cheapest committed neocloud or spot capacity can sit **3-6x below** hyperscaler on-demand list. Treat every dollar figure in this lesson as a mid-2026 snapshot — the ratios and the *reasons* are the durable content; the absolute numbers rot in months as B200/GB200 supply floods in and H100 depreciates.

The stake is that GPU spend is now the single largest line in most AI infra budgets, and procurement decisions are made on multi-year commitments (lesson 06). A 2x sticker error compounded over a 3-year reserved commit on a 512-GPU fleet is eight figures. Getting the decomposition right is the difference between a defensible procurement recommendation and an expensive learning experience.

The other reason this matters: the decomposition *is* the interview question. "Hyperscaler quotes $X, neocloud quotes $Y — which do you pick?" A junior answer picks the cheap number. A senior answer says "neither yet — normalize both to fully-loaded $/GPU-hr at our utilisation and interconnect needs, then decide," and then actually walks the axes.

## What's new here

- You already know per-pod GPU attribution (04/05), the util-lie (05), and capex-vs-cloud crossover (10). This lesson is upstream of all of them: it's about the **rate you're attributing in the first place**, and why two "H100-hours" can differ 3x in true cost.
- New: a **price-gap decomposition framework** — seven axes that explain the sticker delta, each of which either shrinks the gap (it was bundling) or is real savings you can bank.
- New: a **normalization procedure** — reduce any offer to *fully-loaded $/GPU-hr including networking, storage, egress, and support, adjusted for your utilisation and commitment risk* — so that A-vs-B becomes a single comparable number instead of two incommensurable stickers.
- Not re-taught: what an H100 is, what committed-use discounts are (lesson 06 owns commitment mechanics), what MFU/cost-per-run are (lesson 08 owns work-normalized cost). Here we stop at fully-loaded $/GPU-hr; converting that to $/unit-of-work is the earlier modules' job.

## Core notes

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

**(b) Capital structure & depreciation.** Neoclouds run a different financial model: aggressive depreciation schedules (some depreciate H100s over ~4-6 years but price as if faster to move capacity), high target utilisation (they *must* keep GPUs busy to service GPU-backed debt), and thinner overhead. They are single-product companies optimizing one SKU. This is real structural cost advantage, not an accounting trick — part of the gap is genuinely lower cost-to-serve.

**(c) Networking / interconnect.** The load-bearing axis. A single H100 is fungible; a *training cluster* needs rail-optimized InfiniBand (NDR 400G) or equivalent RoCE fabric with SHARP/rail topology (ref lesson 09 on collectives and fabric). Ask of every quote: **is non-blocking IB/rail-optimized fabric in the rate, or a separate line, or unavailable?** A $2/GPU-hr rate with only standard Ethernet is worthless for multi-node training — your all-reduce chokes and MFU collapses. Some neoclouds include IB in the headline (CoreWeave, Crusoe rail-optimized clusters); others sell it as a premium tier or only in large reserved blocks. If IB is extra, add it back before comparing.

**(d) Storage & egress.** Frequently unbundled and a hidden cost. High-throughput parallel storage (VAST, WEKA, Lustre) for checkpoint/dataset I/O may be a separate SKU. **Egress** is the classic trap: hyperscaler egress is expensive but predictable; some neoclouds offer free/cheap egress (a real advantage for serving), others meter it. If your workload ships checkpoints or serves tokens out, model egress explicitly — it can dominate for inference-heavy patterns.

**(e) Utilisation & committed term.** The headline low neocloud rate almost always requires a long commitment (6-36 months) or is spot. Lesson 06 owns the commitment mechanics; here the point is that **commitment converts a variable cost into a fixed one**, and the "savings" only materialize if your utilisation is high enough to beat on-demand. A cheap committed rate at 30% fleet utilisation is more expensive per unit of work than a dearer on-demand rate at 90%. Normalize by *your expected utilisation*, not the vendor's implied 100%.

**(f) Reliability, SLA & support.** Neoclouds may have thinner SLAs, less mature control planes, slower node-replacement on GPU failure, and smaller support orgs. This is why bare-metal lifecycle and node-health skills matter (mod 10) — on a neocloud you may *be* the SRE for the hardware. Quantify: expected uptime and failed-node MTTR feed effective-availability, which inflates your true $/GPU-hr (you pay for hours you can't use). A 96%-effective fleet at $2 costs the same per usable hour as a 99.9%-effective fleet at ~$1.93.

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

## Self-check

- A neocloud quotes $1.80/GPU-hr and a hyperscaler quotes $4.50 on-demand for H100. Your teammate says "obvious, take the neocloud, we save 60%." What's the first question you ask before agreeing? **Answer:** "Is rail-optimized InfiniBand in that $1.80, and what's our expected utilisation and commitment term?" Interconnect (axis c) and utilisation/commitment (axis e) are the two axes most likely to erase the savings — a bare-Ethernet rate is useless for the multi-node training this presumably is, and a committed rate at low utilisation isn't actually cheaper per unit of work.
- Why does charging your organization's true cost require dividing the fully-loaded rate by effective availability *and* by utilisation realization, rather than just using the base rate? **Answer:** You pay for hours the fleet is unusable (node failures → effective availability < 1) and, on committed capacity, for hours you reserved but didn't fill (utilisation realization < 1). Both inflate the true cost *per unit of delivered work* above the sticker; ignoring them makes a cheap committed neocloud look better than it is at low utilisation.
- After normalizing, a 5x sticker gap collapses to 1.1x. What does that tell you, and what should the procurement recommendation be? **Answer:** The gap was almost entirely bundling and adjacent-service margin (axes a/d), not structural cost advantage — the neocloud's cheap sticker was a bare GPU-hour and the hyperscaler's rate absorbed IB, storage, egress, and support you'd otherwise buy separately. At 1.1x fully-loaded, recommend the hyperscaler: the tiny residual saving doesn't justify the thinner SLA, immature control plane, and commitment risk (axes e/f).

## Resources

- CoreWeave pricing and instance/fabric documentation — https://www.coreweave.com/pricing
- Lambda GPU cloud pricing — https://lambdalabs.com/service/gpu-cloud
- AWS `p5` (H100) instance and pricing reference — https://aws.amazon.com/ec2/instance-types/p5/
- Google Cloud A3 (H100) VM documentation — https://cloud.google.com/compute/docs/accelerator-optimized-machines#a3-vms
- SemiAnalysis, "The GPU Cloud Cost & Economics" analysis — https://www.semianalysis.com/p/the-gpu-cloud-clustermax-rating-system-how-to-rent-gpus
- Crusoe / Nebius reference-cluster and InfiniBand fabric specs — https://crusoe.ai/cloud/ and https://nebius.com/prices

[💰 11 — GPU cost and unit economics](../README.md)
