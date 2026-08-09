---
lesson: 06
title: "Commitment & procurement strategy for GPU capacity"
module: 11
concept: "coverage, spot, build-vs-buy-vs-rent"
status: not-started
est_time: "3 hrs"
artifacts: ["a coverage-optimization model that sets a commitment level against a demand distribution and outputs blended $/GPU-hr with a waste-vs-premium and utilization sensitivity table"]
---

# Commitment & procurement strategy for GPU capacity

[💰 11 — GPU cost and unit economics](../README.md)

## Why this matters

You know how RIs and Savings Plans work in ordinary cloud: pre-commit spend or capacity, get a discount, eat some waste on the unused portion. That intuition transfers, but GPU procurement in 2025-2026 breaks one load-bearing assumption of general-cloud FinOps: **on-demand is not always available.** For CPUs you can burst to on-demand at a premium whenever you want; for H100/H200/GB200-class capacity, the reservation is frequently the *only* way to get the hardware at all. Procurement stops being purely a price-optimization problem and becomes a **supply-securing** problem. That inversion is the whole point of this lesson, and it's what an interviewer is checking when they ask "how would you procure capacity for a fleet that's about to 3×."

The dollars are enormous and the decisions are semi-irreversible. A multi-year neocloud reservation is a balance-sheet commitment measured in eight or nine figures; a wrong commitment level strands capital in idle silicon or forces you into a spot market you can't rely on for latency-serving. This is the exact terrain where platform teams get evaluated: not "did you get a discount," but "did you size the commitment to the *shape* of demand and defend it against utilization risk." The blended-rate number you produce here is the direct input to lesson 05's unit economics — get procurement wrong and every unit cost downstream is wrong.

It also sits on top of the module 10 capex-vs-cloud crossover. Owning, committing to a neocloud, and staying on-demand are three points on one build-vs-buy-vs-rent continuum, and the right point is a function of demand predictability, contract duration, and — the multiplier that dominates everything — utilization. This lesson makes that decision explicit and quantitative.

## What's new here

- **You already know** generic RIs, Savings Plans, and spot mechanics, and the coverage/commitment tradeoff in the abstract. **Skip** the general FinOps theory.
- **You already know** the capex crossover model (mod 10), the two-ledger model (lesson 02), and checkpointed-training fault tolerance (mod 08). These are *referenced*, not re-taught.
- **New angle 1:** the GPU-specific landscape — **neocloud committed contracts** as a distinct instrument, and the **capacity-not-just-price** constraint that makes reservations about securing supply.
- **New angle 2:** the coverage math applied to a GPU demand distribution (commit the baseline, burst the peak) and the resulting **blended $/GPU-hr**.
- **New angle 3:** spot **specifically for GPUs** — fine for checkpointed training/batch, disqualified for latency inference — and the explicit **spot break-even** between discount and interruption overhead.
- **New angle 4:** **utilization as the multiplier on every option**, and the **build-vs-buy-vs-rent** decision framing as a function of predictability, duration, and utilization.

## Core notes

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

## Worked example

A fleet with a **40-GPU steady baseline** and spiky peaks to **100 GPUs**. Demand distribution over a month: 40 GPUs needed ~100% of hours; the increment 40→100 is needed progressively less (say the 60 peak-GPUs are each needed ~30% of hours on average). Rates: committed \$1.80/GPU-hr, on-demand \$3.20/GPU-hr, spot \$0.90/GPU-hr (training-eligible peak only).

**Option A — commit the baseline (40), burst on-demand.**
- Committed GPU-hrs/mo: $40 \times 720 = 28{,}800$ h, all paid: $28{,}800 \times 1.80 = \$51{,}840$.
- Peak GPU-hrs consumed: $60 \times 720 \times 0.30 = 12{,}960$ h on-demand: $\times 3.20 = \$41{,}472$.
- Total \$93,312 for $28{,}800 + 12{,}960 = 41{,}760$ consumed GPU-hrs → **blended \$2.23/GPU-hr.** Commitment waste ≈ 0 (baseline nearly always used).

**Option B — over-commit to 70 GPUs.**
- Committed: $70 \times 720 \times 1.80 = \$90{,}720$, but only ~$(40\times720) + (30\times720\times0.30)=28{,}800+6{,}480=35{,}280$ h of it consumed → **6,480 committed-but-idle hrs of waste** (\$11,664).
- Peak above 70 on-demand: remaining $30\times720\times\sim0.15 = 3{,}240$ h × 3.20 = \$10,368.
- Total ≈ \$101,088 — **worse**, because the incremental committed GPUs sit idle more than their discount saves. Over-committing the spiky tail loses.

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

## Self-check

- Why is GPU procurement in 2025-2026 not simply a price-optimization problem the way CPU RIs are? **Answer:** For frontier-class GPUs (H100/H200/GB200) on-demand capacity is frequently unavailable, so reservations and neocloud committed contracts are signed primarily to *secure supply* through a shortage; capacity, not price, is the binding constraint, which inverts the general-cloud assumption that on-demand is an always-available fallback.
- A reserved GPU costs \$1.80/GPU-hr and runs at 30% SM_ACTIVE; on-demand is \$3.20/GPU-hr at 90%. Which is cheaper per unit of useful work, and what's the lesson? **Answer:** Reserved effective = 1.80/0.30 = \$6.00/useful-hr; on-demand = 3.20/0.90 = \$3.56/useful-hr — the on-demand GPU is cheaper. Lesson: a commitment discount is meaningless until divided by the utilization you'll actually sustain (SM_ACTIVE, not GPU_UTIL); low or uncertain utilization can make a nominally cheaper reservation the more expensive option.
- Spot GPUs are 70% cheaper. When is spot the right choice and when is it disqualified, and what's the break-even condition? **Answer:** Right for checkpointed training and batch/offline inference (kill-and-resume tolerant); disqualified for latency-sensitive online inference (interruptions drop in-flight requests with no guaranteed replacement). Break-even: spot pays off while discount $d$ > lost-work fraction $f\cdot t_{\text{lost}}$ (interruptions/hr × hours of progress lost each); more frequent checkpointing shrinks $t_{\text{lost}}$ and tips it toward spot.

## Resources

- AWS EC2 Capacity Blocks for ML (reserve GPU capacity for a fixed window): https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-blocks.html
- Google Cloud Committed Use Discounts + reservations for accelerators: https://cloud.google.com/compute/docs/instances/reservations-overview
- CoreWeave reserved-capacity / committed-contract pricing model: https://www.coreweave.com/pricing
- FinOps Foundation — Rate Optimization (commitment coverage) capability: https://www.finops.org/framework/capabilities/rate-optimization/
- Semianalysis — GPU cloud economics & rental-rate analysis: https://www.semianalysis.com/
- Kubernetes docs — handling preemption/spot node interruptions for training workloads: https://kubernetes.io/docs/concepts/scheduling-eviction/
