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
sources: 10
---

# Commitment & procurement strategy for GPU capacity

[💰 11 — GPU cost and unit economics](../README.md) · ← [05 Unit economics](05-unit-economics.md) · → [07 Neocloud vs hyperscaler price gap](07-neocloud-vs-hyperscaler.md)

## Where this fits

Lesson 05 gave you the unit-cost formula, $\text{unit cost} = \frac{H_{\text{attr}} \times r_{\text{blend}}}{U}$, and treated $r_{\text{blend}}$ as a given input pulled from "the fleet's commitment mix." This lesson is where that number actually comes from: the procurement decision that sets how much capacity you commit to, at what duration, and against what fallback. Get this decision wrong and every unit-cost number downstream in lesson 05 is wrong regardless of how carefully you did the attribution and the join. This lesson also closes the loop on module 10's capex-vs-cloud crossover — own, commit, or rent are three points on one continuum, and this is where you learn to place a real fleet on it.

## Why this matters

You know how RIs and Savings Plans work in ordinary cloud: pre-commit spend or capacity, get a discount, eat some waste on the unused portion. That intuition transfers, but GPU procurement in 2025-2026 breaks one load-bearing assumption of general-cloud FinOps: **on-demand is not always available.** For CPUs you can burst to on-demand at a premium whenever you want; for H100/H200/GB200-class capacity, the reservation is frequently the *only* way to get the hardware at all. Procurement stops being purely a price-optimization problem and becomes a **supply-securing** problem. That inversion is the whole point of this lesson, and it's what an interviewer is checking when they ask "how would you procure capacity for a fleet that's about to 3×."

The dollars are enormous and the decisions are semi-irreversible. A multi-year neocloud reservation is a balance-sheet commitment measured in eight or nine figures; a wrong commitment level strands capital in idle silicon or forces you into a spot market you can't rely on for latency-serving. This is the exact terrain where platform teams get evaluated: not "did you get a discount," but "did you size the commitment to the *shape* of demand and defend it against utilization risk." The blended-rate number you produce here is the direct input to lesson 05's unit economics — get procurement wrong and every unit cost downstream is wrong.

It also sits on top of the module 10 capex-vs-cloud crossover. Owning, committing to a neocloud, and staying on-demand are three points on one build-vs-buy-vs-rent continuum, and the right point is a function of demand predictability, contract duration, and — the multiplier that dominates everything — utilization. This lesson makes that decision explicit and quantitative.

## What's new here (calibration)

- **You already know** generic RIs, Savings Plans, and spot mechanics, and the coverage/commitment tradeoff in the abstract. **Skip** the general FinOps theory.
- **You already know** the capex crossover model (mod 10), the two-ledger model (lesson 02), and checkpointed-training fault tolerance (mod 08). These are *referenced*, not re-taught.
- **New angle 1:** the GPU-specific landscape — **neocloud committed contracts** as a distinct instrument, and the **capacity-not-just-price** constraint that makes reservations about securing supply.
- **New angle 2:** the coverage math applied to a GPU demand distribution (commit the baseline, burst the peak) and the resulting **blended $/GPU-hr**.
- **New angle 3:** spot **specifically for GPUs** — fine for checkpointed training/batch, disqualified for latency inference — and the explicit **spot break-even** between discount and interruption overhead.
- **New angle 4:** **utilization as the multiplier on every option**, and the **build-vs-buy-vs-rent** decision framing as a function of predictability, duration, and utilization.

## Core concepts

### The GPU procurement landscape

| Instrument | $/GPU-hr posture | Commitment | Supply guarantee | Best for |
|---|---|---|---|---|
| On-demand (hyperscaler) | Highest | None | **Weak** for top-end GPUs — often unavailable | Unpredictable spikes, short experiments |
| Reserved / committed-use (CUD, capacity blocks) | Medium-low | 1 mo – 3 yr | Strong within the reservation | Predictable baseline |
| Spot / preemptible | Lowest (60-90% off) | None | **None** — reclaimed anytime | Checkpointed training, batch |
| Neocloud committed contract (CoreWeave, Lambda, Crusoe, Nebius, …) | Low $/GPU-hr, but multi-month/multi-year floor | Long, often take-or-pay | **Primary reason to sign** — it's how you *get* the GPUs | Large steady/growing baseline at scale |
| On-prem owned (mod 10) | Lowest marginal at high util; capex up front | Years (asset life) | You own it | High-utilization, long-duration, predictable |

The 2025-2026 reality: for frontier-class accelerators, **capacity is the binding constraint, not price.** Reservations and neocloud contracts are signed to *secure supply* through a shortage; the discount is secondary. This flips the general-cloud framing where on-demand is the always-available fallback.

### Coverage optimization on a demand distribution

Model demand as a distribution of concurrent GPUs over time: a steady baseline plus bursty peaks.

- **Cover the baseline with commitments** (cheapest $/GPU-hr, and the part of demand you're most confident about).
- **Serve the peak with on-demand/spot** (premium rate, but only paid when needed).
- Standard FinOps coverage rule, applied to GPUs: **commit to roughly the P-something of your demand distribution** — commit up to the level you're confident you'll consume nearly always (e.g. P20-P50 of concurrent demand), leave the tail to on-demand.

The tradeoff at the margin:

- Commit **too high** → **commitment waste**: paying the reserved rate for GPUs sitting idle below-baseline.
- Commit **too low** → **on-demand premium** on more of the curve (and, for GPUs, *supply risk* if on-demand isn't there).

Blended rate for a chosen commitment level $c$ (fraction of GPU-hours served off commitment):

$$
r_{\text{blend}} = c \cdot r_{\text{commit}} + (1-c)\cdot r_{\text{ondemand/spot}}
$$

The optimum minimizes total cost = committed-hours × $r_{\text{commit}}$ + burst-hours × $r_{\text{burst}}$, where committed hours you *pay for regardless* (waste = committed − consumed-below-baseline). This is exactly the general coverage optimization; what's GPU-specific is (a) the supply-risk penalty on the on-demand term and (b) the utilization multiplier below.

### Spot / preemptible for GPUs

- **Viable:** checkpointed training (mod 08 fault tolerance) and batch/offline inference — anything that tolerates a kill-and-resume.
- **Disqualified:** latency-sensitive online inference — an interruption drops in-flight requests and you can't guarantee replacement capacity.
- **Interruption cost** per preemption = checkpoint reload + recompute of progress lost since last checkpoint (+ requeue/scheduling latency). Let $d$ = spot discount fraction vs on-demand, $f$ = interruptions per hour, $t_{\text{lost}}$ = hours of work lost per interruption (≈ time since last checkpoint + reload). Spot pays off when:

$$
\underbrace{d}_{\text{saved fraction}} \;>\; \underbrace{f \cdot t_{\text{lost}}}_{\text{fraction of time re-doing lost work}}
$$

i.e. spot is worth it while the discount exceeds the wasted-work fraction. **Lever:** checkpoint more frequently to shrink $t_{\text{lost}}$ and push the break-even in spot's favor — until checkpoint I/O overhead itself eats the gain. Typical: 70% discount ($d=0.7$) with 0.3 interruptions/hr and 0.5 h lost each → wasted fraction 0.15 ≪ 0.7, spot clearly wins.

### Utilization is the multiplier on everything

A commitment only pays off at sufficient utilization — tie back to the two-ledger model (lesson 02) and the capex crossover (mod 10). Effective $/useful-GPU-hr = $r / \text{utilization}$ (and "useful" should mean SM_ACTIVE, not GPU_UTIL — the util-lie of 05). Consequence, stated bluntly:

> **A reserved GPU at 30% SM_ACTIVE is more expensive per unit of work than an on-demand GPU at 90%.** ($r_{\text{commit}}/0.30$ vs $r_{\text{ondemand}}/0.90$ — a 50% commit discount is wiped out by a 3× utilization gap.)

So the commitment decision is never "is the reserved rate lower" — it's "is the reserved rate lower *after dividing by the utilization I'll actually sustain over the commitment term.*" Low, uncertain utilization argues for on-demand even at a nominal premium.

### Build-vs-buy-vs-rent framing

A decision in three variables — demand **predictability**, **duration**, **utilization**:

- **Own (build, mod 10):** high predictability + long duration + high sustained utilization → cross the capex crossover, lowest marginal cost, you control supply. Penalty: capital, lead time, obsolescence risk (next-gen silicon).
- **Neocloud committed contract (buy capacity):** large, steady-or-growing baseline you can't/won't capex, and where securing supply matters more than squeezing the last dollar. Medium duration, you offload datacenter ops. Penalty: take-or-pay waste if demand softens.
- **On-demand / spot (rent):** low predictability, short duration, or spiky/experimental workloads; utilization too low or too uncertain to justify a commitment. Penalty: highest rate, and for top-end GPUs, real supply risk.

Steady base → commit or own; spiky tail → rent. Most real fleets are a **portfolio**: own/commit the confident baseline, rent the tail.

## Perspectives

**Supply-chain/procurement perspective.** In generic cloud FinOps, an RI is purely a discount instrument — if you get the coverage wrong, you fall back to on-demand and eat a premium, but you always get the machine. In 2025-2026 GPU procurement, that fallback frequently does not exist: on-demand H100/H200/GB200 capacity is allocation-constrained, sometimes with multi-month waitlists at hyperscalers. This inverts the standard CPU-RI mental model — the commitment isn't optional risk-hedging against price, it's the only reliable path to the hardware existing for you at all. A procurement team that treats a GPU reservation like a Savings Plan, purely optimizing for discount percentage, will lose the actual capacity race to a competitor who signed earlier for supply reasons.

**Finance/treasury perspective.** A multi-year neocloud commitment is not a line item, it's a balance-sheet decision. Take-or-pay contracts in the eight-to-nine-figure range show up as committed future cash outflows regardless of whether demand materializes, which affects covenant calculations, cash runway planning, and how a CFO models downside scenarios. Treasury has to weigh the commitment against the counterfactual — capacity risk if you don't sign, financial risk if demand softens after you do — the same tension a startup weighs when signing a multi-year office lease, except the "lease" here depreciates on a hardware refresh cycle instead of appreciating like real estate.

**Workload-scheduling perspective.** The line between spot-eligible and spot-ineligible workloads is not fuzzy — it's a hard binary determined by one property: can the workload tolerate a kill-and-resume without a human or a customer noticing? Checkpointed batch training passes; a synchronous, latency-critical inference request does not, full stop, because there is no way to guarantee replacement capacity mid-request. A scheduler that doesn't enforce this distinction at the workload-class level will eventually put a serving pod on spot during a capacity crunch, and the resulting dropped-request incident is entirely predictable in hindsight and entirely avoidable in advance.

**Risk-management perspective.** The utilization-sensitivity table in the worked example below is not decoration — it's a downside-scenario stress test in the same spirit as a risk desk stress-testing a portfolio. The question it answers is: "if our actual utilization comes in below plan, at what point does this commitment become the *more expensive* option, not just a less-good one?" Running that table before signing turns "we think utilization will be fine" into a quantified breakeven point you can actually monitor against — the same discipline as running a covenant-breach scenario before taking on debt.

## Real-world use cases

- **CoreWeave — Form S-1 (SEC EDGAR, filed 2025-03-03)** ([sec.gov/Archives/edgar/data/1769628/000119312525044231/d899798ds1.htm](https://www.sec.gov/Archives/edgar/data/1769628/000119312525044231/d899798ds1.htm)) — a real public filing. CoreWeave depreciates GPU equipment straight-line over 6 years, and as of December 31, 2024 held a contracted revenue backlog reported in the $66B-$88B range across public summaries of the filing, with a weighted-average contract duration of roughly 4 years; contracts are generally fixed-price per contracted-GPU-hour, billed monthly against reserved usage. This is the best grounding available for "a reservation is a supply-securing, semi-irreversible, balance-sheet decision" — it shows what that decision looks like on an actual company's books, not a hypothetical.
- **Databricks/MosaicML — "Training Stable Diffusion from Scratch for $50k with MosaicML (Part 2)"** ([databricks.com/blog/stable-diffusion-2](https://www.databricks.com/blog/stable-diffusion-2)) — describes "Node Doctor" and "Watchdog," platform features that automatically detect failed or preempted nodes and resume training with zero human intervention. This demonstrates the checkpoint-driven spot-viability lever directly: the reason this team could run large training jobs on interruptible-capable infrastructure economically is that the interruption-recovery cost was engineered down to near-zero, not just accepted as a discount tradeoff.
- **Databricks/MosaicML — "Introducing MPT-7B"** ([databricks.com/blog/mpt-7b](https://www.databricks.com/blog/mpt-7b)) — during a training run on 440 GPUs over 9.5 days, the platform detected and addressed 4 hardware failures and auto-resumed with zero human intervention. Concrete, real numbers for the interruption-recovery side of the spot break-even math: a handful of failures over hundreds of GPU-days did not meaningfully change the economics because the recovery loop was automated.
- **AWS News Blog — "Announcing Amazon EC2 Capacity Blocks for ML"** ([aws.amazon.com/blogs/aws/announcing-amazon-ec2-capacity-blocks-for-ml-to-reserve-gpu-capacity-for-your-machine-learning-workloads](https://aws.amazon.com/blogs/aws/announcing-amazon-ec2-capacity-blocks-for-ml-to-reserve-gpu-capacity-for-your-machine-learning-workloads)) — the official hyperscaler framing of GPU reservation explicitly as a supply-securing instrument (reserve a cluster of H100s for a fixed future window, up to eight weeks ahead), closer to booking a hotel room for a date than to a generic cloud discount product. Direct evidence, from the hyperscaler itself, that the capacity-not-price framing in this lesson is how even AWS designed the product.

## Worked example

A fleet with a **40-GPU steady baseline** and spiky peaks to **100 GPUs**. Demand distribution over a month: 40 GPUs needed ~100% of hours; the increment 40→100 is needed progressively less (say the 60 peak-GPUs are each needed ~30% of hours on average). Rates: committed \$1.80/GPU-hr, on-demand \$3.20/GPU-hr, spot \$0.90/GPU-hr (training-eligible peak only).

**Option A — commit the baseline (40), burst on-demand.**
- Committed GPU-hrs/mo: $40 \times 720 = 28{,}800$ h, all paid: $28{,}800 \times 1.80 = \$51{,}840$.
- Peak GPU-hrs consumed: $60 \times 720 \times 0.30 = 12{,}960$ h on-demand: $\times 3.20 = \$41{,}472$.
- Total \$93,312 for $28{,}800 + 12{,}960 = 41{,}760$ consumed GPU-hrs → **blended \$2.23/GPU-hr.** Commitment waste ≈ 0 (baseline nearly always used).

**Option B — over-commit to 70 GPUs.**
- Committed: $70 \times 720 \times 1.80 = \$90{,}720$, but only ~$(40\times720) + (30\times720\times0.30)=28{,}800+6{,}480=35{,}280$ h of it consumed → **6,480 committed-but-idle hrs of waste** (\$11,664).
- Peak above 70 on-demand: remaining $30\times720\times\sim0.15 = 3{,}240$ h × 3.20 = \$10,368.
- Total ≈ \$101,088 — **worse**, because the incremental committed GPUs sit idle more than their discount saves. Over-committing the spiky tail loses, and it's worth stating the intuition this defeats: "more coverage is always safer" feels true but is exactly backwards once you commit past the confident baseline — every additional committed GPU that isn't consistently used converts a nominal discount into pure waste.

**Option C — commit 40, serve checkpointed-training peak on spot.**
- If the peak is training (spot-eligible): $12{,}960$ h × 0.90 = \$11,664 instead of \$41,472.
- Total \$63,504 → **blended \$1.52/GPU-hr** — a 32% cut vs Option A, *if* the peak workload tolerates preemption.

**Utilization sensitivity.** Re-price Option A's *useful* cost at different baseline SM_ACTIVE:

| Baseline SM_ACTIVE | Effective \$/useful GPU-hr (committed portion) |
|---|---|
| 90% | 1.80 / 0.90 = **\$2.00** |
| 60% | 1.80 / 0.60 = **\$3.00** |
| 30% | 1.80 / 0.30 = **\$6.00** — worse than on-demand at 90% (\$3.56) |

The commitment's value **inverts** below ~56% utilization vs on-demand-at-90%. So the recommendation is Option A (or C if training-heavy), commit **only the confident 40-GPU baseline**, never the tail — and *conditional on holding the baseline above ~60% SM_ACTIVE over the term*; if you can't, shrink the commitment.

## Practice

Feeds `../practice/gpu-cost-synthesis/README.md`:

1. Build a coverage model: given an hourly concurrent-GPU demand series and rates for committed/on-demand/spot, compute total cost and blended $/GPU-hr as a function of commitment level $c$; find the cost-minimizing $c$ and plot cost vs $c$ (identify the waste-vs-premium turning point).
2. Add the utilization multiplier: parameterize sustained SM_ACTIVE on the committed portion and report effective $/useful-GPU-hr; find the utilization at which the chosen commitment stops beating on-demand.
3. Spot break-even: given discount $d$, interruption rate $f$, and checkpoint interval, compute lost-work fraction and decide spot-vs-on-demand for a training peak; show how halving the checkpoint interval moves the decision.
4. Write a one-page build-vs-buy-vs-rent recommendation for the 40/100 fleet as a function of demand predictability and expected utilization, citing the crossover from module 10.

## Common pitfalls

1. **Treating GPU procurement like generic cloud RIs, where on-demand is always a fallback.** This is false for frontier-class GPUs in 2025-2026 — the CoreWeave S-1 backlog figures above are direct evidence that buyers lock in years of capacity specifically because the on-demand fallback isn't reliably there, not purely to capture a discount.
2. **Comparing $/GPU-hr commitment discount without dividing by expected utilization.** This is the single most testable interview trap in the module: a nominally cheaper reserved rate can be the *more expensive* option per unit of useful work if sustained utilization is low, exactly as the worked example's sensitivity table shows.
3. **Putting latency-serving workloads on spot.** An interruption mid-request drops live traffic with no guaranteed replacement capacity; this is disqualifying, not a tradeoff to weigh — spot belongs to checkpointed training and batch/offline inference only.
4. **Over-committing to the peak instead of the baseline.** Option B in the worked example is the concrete demonstration: committing 70 GPUs instead of the confident 40-GPU baseline produced *more* total cost than committing less, because the extra committed capacity sat idle more than its discount saved. "More coverage is always safer" is the wrong intuition — coverage past the confident baseline converts a discount into waste.
5. **Treating the checkpoint interval as fixed rather than as a lever on the spot break-even.** The MosaicML Node Doctor/Watchdog story above shows this is actively engineered, not passively accepted: shrinking $t_{\text{lost}}$ by checkpointing more frequently directly improves the $d > f \cdot t_{\text{lost}}$ inequality and can turn a marginal spot decision into a clear win, right up until checkpoint I/O overhead itself starts eating the gain.

## Self-check

- Why is GPU procurement in 2025-2026 not simply a price-optimization problem the way CPU RIs are? **Answer:** For frontier-class GPUs (H100/H200/GB200) on-demand capacity is frequently unavailable, so reservations and neocloud committed contracts are signed primarily to *secure supply* through a shortage; capacity, not price, is the binding constraint, which inverts the general-cloud assumption that on-demand is an always-available fallback.
- A reserved GPU costs \$1.80/GPU-hr and runs at 30% SM_ACTIVE; on-demand is \$3.20/GPU-hr at 90%. Which is cheaper per unit of useful work, and what's the lesson? **Answer:** Reserved effective = 1.80/0.30 = \$6.00/useful-hr; on-demand = 3.20/0.90 = \$3.56/useful-hr — the on-demand GPU is cheaper. Lesson: a commitment discount is meaningless until divided by the utilization you'll actually sustain (SM_ACTIVE, not GPU_UTIL); low or uncertain utilization can make a nominally cheaper reservation the more expensive option.
- Spot GPUs are 70% cheaper. When is spot the right choice and when is it disqualified, and what's the break-even condition? **Answer:** Right for checkpointed training and batch/offline inference (kill-and-resume tolerant); disqualified for latency-sensitive online inference (interruptions drop in-flight requests with no guaranteed replacement). Break-even: spot pays off while discount $d$ > lost-work fraction $f\cdot t_{\text{lost}}$ (interruptions/hr × hours of progress lost each); more frequent checkpointing shrinks $t_{\text{lost}}$ and tips it toward spot.
- In the worked example, why did committing to 70 GPUs (Option B) cost more than committing to 40 (Option A), even though the committed rate is the same \$1.80/GPU-hr in both cases? **Answer:** Because the extra 30 committed GPUs above the confident 40-GPU baseline were only consumed a fraction of the time (~15-30% of hours), so most of their committed-rate hours were paid for but unused — 6,480 hours of committed-but-idle waste (\$11,664) that outweighed the on-demand premium it saved. The lesson generalizes: commitment should track the confident baseline of the demand distribution, not the peak, because coverage beyond the baseline converts a discount into waste rather than savings.

## Connections & what's next

This lesson closes the loop lesson 05 opened: the blended rate $r_{\text{blend}}$ that lesson's unit-cost formula treats as an input is exactly the output of the coverage-optimization and spot-eligibility decisions made here. It also extends module 10's capex-vs-cloud crossover into a three-way build-vs-buy-vs-rent framing, adding the neocloud committed contract as a distinct middle option and utilization as the multiplier that can flip the ranking of all three. The next lesson, 07, decomposes the 3-6× price gap between neocloud and hyperscaler GPU rental rates — and it leans on the exact same rate-normalization discipline built here (blended vs marginal, dividing by utilization before comparing) to make sure that gap is measured honestly rather than by comparing mismatched sticker prices.

## References & further reading

- **Primary sources**
  - AWS — Capacity Blocks for ML documentation: [docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-blocks.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-blocks.html) — read for the mechanics of a hyperscaler's fixed-window GPU reservation product.
  - Google Cloud — Committed Use Discounts and reservations for accelerators: [cloud.google.com/compute/docs/instances/reservations-overview](https://cloud.google.com/compute/docs/instances/reservations-overview) — read for a second hyperscaler's reservation model, for comparison against AWS's.
  - FinOps Foundation — Rate Optimization capability: [finops.org/framework/capabilities/rate-optimization](https://www.finops.org/framework/capabilities/rate-optimization/) — read for the general coverage-optimization framework this lesson specializes to GPUs.
  - Kubernetes docs — scheduling and eviction, preemption handling: [kubernetes.io/docs/concepts/scheduling-eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/) — read for how spot/preemptible node interruption is actually handled at the orchestration layer.
  - CoreWeave — Form S-1, SEC EDGAR (filed 2025-03-03): [sec.gov/Archives/edgar/data/1769628/000119312525044231/d899798ds1.htm](https://www.sec.gov/Archives/edgar/data/1769628/000119312525044231/d899798ds1.htm) — read for real contracted-backlog, contract-duration, and depreciation figures from a public GPU-cloud filing (dated 2025 snapshot).
- **Real-world engineering blogs**
  - AWS News Blog — "Announcing Amazon EC2 Capacity Blocks for ML": [aws.amazon.com/blogs/aws/announcing-amazon-ec2-capacity-blocks-for-ml-to-reserve-gpu-capacity-for-your-machine-learning-workloads](https://aws.amazon.com/blogs/aws/announcing-amazon-ec2-capacity-blocks-for-ml-to-reserve-gpu-capacity-for-your-machine-learning-workloads) — what it shows: official hyperscaler framing of reservation as a supply-securing instrument, not a price-optimization one.
  - Databricks/MosaicML — "Training Stable Diffusion from Scratch for $50k with MosaicML (Part 2)": [databricks.com/blog/stable-diffusion-2](https://www.databricks.com/blog/stable-diffusion-2) — what it shows: automated node-failure detection and resume (Node Doctor/Watchdog) as the engineering lever behind spot viability.
  - Databricks/MosaicML — "Introducing MPT-7B": [databricks.com/blog/mpt-7b](https://www.databricks.com/blog/mpt-7b) — what it shows: concrete interruption-recovery numbers (4 hardware failures over 440 GPUs / 9.5 days, zero human intervention).
  - CoreWeave — pricing model for reserved capacity / committed contracts: [coreweave.com/pricing](https://www.coreweave.com/pricing) — what it shows: a neocloud's own framing of committed-contract pricing against on-demand.
- **Deeper dives**
  - SemiAnalysis — GPU cloud economics and rental-rate analysis: [semianalysis.com](https://www.semianalysis.com/) — longer-form, frequently-updated analysis of GPU cloud pricing and supply dynamics; useful for tracking how the capacity-vs-price balance in this lesson shifts over time.
