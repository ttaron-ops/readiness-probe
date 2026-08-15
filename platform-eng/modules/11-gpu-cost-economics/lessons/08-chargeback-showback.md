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
sources: 8
---

# Chargeback, showback, and queue-wait billing

## Where this fits

Lessons 01-07 built the full attribution and cost stack: the four sharing regimes and what's provably unattributable (01), the allocated-vs-utilised ledger split (02), idle detection as an enforcement lever (03), fragmentation cost (04), unit economics translating raw $/GPU-hr into $/token and $/run (05), commitment and procurement strategy (06), and — most recently — how to normalize a vendor's rate into a defensible, fully-loaded $/GPU-hr (07). This lesson is where all of that lands on a real team's budget. Everything upstream was about knowing the true cost; this lesson is about allocating it to tenants and turning it into behavior change via chargeback and showback.

## Why this matters

Once your org owns or commits to a GPU fleet, the fleet's cost is fixed and the internal question flips from "what does a GPU-hour cost the company" to "who inside the company pays for it, and how do we make them stop hoarding." This is the mechanism that turns the attribution plumbing from lessons 01-04 into behavior change. Attribution without allocation is just a dashboard nobody acts on; allocation without good attribution is a billing fight nobody wins.

The stakes are concrete and political. A research org with a shared 256-GPU cluster where teams reserve-and-idle is bleeding money in the most invisible way possible: the GPUs are "allocated" (so the fleet looks busy) but doing nothing (the util-lie, lesson 05). Whoever runs the platform gets asked, in the same breath, "why is our GPU bill so high" and "why can't my team ever get GPUs." Chargeback/showback design is how you answer both — by making the cost of hoarding land on the hoarder's budget and the cost of the fleet land honestly across tenants.

You know showback and chargeback generically from FinOps. What's new and genuinely hard here is applying them to a *shared GPU fleet* where the unit is scarce, lumpy (whole GPUs, often whole nodes of 8), fixed-cost, and where "allocated" and "utilised" can differ by 3x. The generic FinOps playbook — tag resources, split the bill — breaks on GPUs precisely at the allocated-vs-utilised fork, and picking the wrong ledger quietly rewards the worst behavior.

## What's new here (calibration)

- Not re-teaching showback vs chargeback definitions — you have those. The new content is the **GPU-specific allocation decisions**: which ledger to charge, how to recover fixed cost across low utilisation, and how to price the queue.
- New: the **allocated-vs-utilised charging decision** and why "charge allocation, report utilisation" is the defensible standard.
- New: the **blended internal rate** that floats with fleet utilisation — the honest, unpopular signal that low utilisation makes internal GPU-hours *more* expensive, not less.
- New: **queue-wait / fair-share billing** — pricing the queue itself (priority tiers, fair-share quotas, preemptible discounts) in a Kueue-style shared cluster (ref mod 06) so flexible workloads subsidize urgent ones.
- Reused, not re-taught: per-pod attribution (04/05) is the input; idle reclaim (lesson 03) is the enforcement lever that makes allocation-based chargeback non-gameable; the fully-loaded $/GPU-hr from lesson 07 is what feeds the fixed-cost numerator below.

## Core concepts

### Showback vs chargeback on a shared fleet

| | Showback | Chargeback |
|---|---|---|
| Mechanism | Report costs to teams, no invoice | Bill the team's actual budget |
| Drives behavior via | Visibility / social pressure | Accountability / real money |
| Best when | Early, trust-building, attribution still maturing | Attribution trusted, org wants hard incentives |
| Failure mode | Ignored dashboards | Billing disputes, gaming, padding |

The mature pattern is **both**: chargeback on the number that drives the incentive you want, showback on the number that exposes waste. That split is only meaningful once you decide *which ledger* the chargeback runs on.

### The central decision: charge allocated or utilised?

This is the fork that generic FinOps doesn't have to face, because a VM you rent is a VM you pay for. On a shared GPU fleet you can meter two very different things:

- **Allocated GPU-hours** — what the team *reserved* (requested/held via their pods, quotas, or reservations), whether or not the GPU did work.
- **Utilised GPU-hours** — what the team's jobs actually kept busy (SM-occupancy-weighted, ref lesson 05).

**Charge allocation. Report utilisation.** The argument:

- **Charging allocated is defensible and incentive-correct.** The team took the GPU out of the shared pool; nobody else could use it. Charging for the hold — busy or not — makes releasing idle GPUs the rational act. This directly attacks reserve-and-idle.
- **Charging utilised under-recovers and rewards hoarders.** If you only bill busy time, a team can reserve 32 GPUs, run them at 20%, and pay for 20% while denying 80% to everyone else *for free*. Utilised-billing subsidizes exactly the behavior you're trying to kill, and it fails to recover the fleet's fixed cost (the GPUs cost the same whether busy or idle).
- **Report utilisation in showback** so the waste is visible: allocated vs utilised per team, with the gap called out as the *waste ledger*. The team is charged for allocation; the utilisation report is the social/managerial pressure to shrink the gap. Pair this with idle reclaim (lesson 03) so that sustained low utilisation doesn't just get reported — it gets the GPU taken back.

The one exception: for genuinely elastic, best-effort, preemptible workloads you *want* to encourage (see queue pricing below), utilised or discounted billing is appropriate — they're not holding scarce capacity against anyone.

### Fixed-cost recovery and the blended internal rate

An owned or committed fleet is a **largely fixed cost** (lesson 10's crossover math, priced using lesson 07's fully-loaded $/GPU-hr): the depreciation, the reserved-commit spend, the power and DC contracts are owed regardless of utilisation. Chargeback must recover that fixed cost across tenants — which means the internal rate cannot be a static sticker. It must **float with fleet utilisation**:

```
blended_internal_$/GPU-hr = total_fleet_monthly_fixed_cost
                            / (fleet_GPUs * hours * fleet_utilisation)
```

The denominator is *utilised* (or chargeable-allocated) GPU-hours, not calendar GPU-hours. So:

- Fleet at 90% utilisation → cost spread over many hours → **low** internal rate.
- Fleet at 40% utilisation → same fixed cost spread over few hours → **high** internal rate.

This is the honest but unpopular signal: **low fleet utilisation makes every internal GPU-hour more expensive.** Teams instinctively expect an idle fleet to be "cheap"; the blended rate correctly tells them the opposite, because the fixed cost still has to be recovered. Publishing the blended rate monthly makes the whole org feel the cost of collective under-utilisation, which is exactly the pressure you want. (Alternative: recover fixed cost at 100% denominator and eat the under-recovery centrally as a "platform tax" — cleaner budgeting, but it hides the utilisation signal. Prefer the floating rate unless finance mandates a fixed transfer price.)

### Queue-wait billing / fair-share

In a Kueue-style shared cluster (ref mod 06), GPUs are scarce and jobs *wait*. Wait time is an economic signal you can price:

- **Priority tiers, priced differently.** A "front-of-queue / guaranteed-start" tier costs a premium over the blended rate; a "standard" tier runs at the blended rate; a "best-effort / preemptible" tier gets a discount. A team that needs its job to start *now* pays for the privilege of jumping the queue.
- **Fair-share quotas per team.** Each team gets a nominal share (`nominalQuota` on a `ClusterQueue`, in Kueue terms); borrowing above your share is allowed when the fleet is idle (up to a `borrowingLimit`) but is the first to be preempted (governed by `lendingLimit` and `reclaimWithinCohort`), and borrowed capacity can be priced at the premium tier.
- **Preemptible discount.** Workloads that tolerate preemption (checkpointing training, batch eval, offline data gen) run at a discount and backfill the gaps. They *subsidize* the urgent workloads: flexible demand fills troughs (raising fleet utilisation, lowering the blended rate for everyone), urgent demand pays the premium that funds keeping headroom available.

The incentive alignment is the whole point: **flexible workloads pay less and improve fleet economics; urgent workloads pay more and fund the responsiveness they demand.** This is a real mechanism at research platforms, not a thought experiment — it converts the queue from a source of political escalations into a priced market. CoreWeave runs exactly this pattern in production on top of Kueue for AI training workloads (see Real-world use cases below); the cohort/quota/lending mechanics are not a hypothetical scheduler design, they're a shipping primitive you can read about and reproduce.

### Governance and anti-gaming

- **Budgets, quotas, alerts.** Per-team monthly GPU budgets; quota caps in the scheduler; alerts on projected overspend before month-end, not after.
- **Monthly showback per namespace/team.** Allocated GPU-h, utilised GPU-h, waste gap, blended rate, charged amount, queue-premium spend.
- **Anti-gaming.** Two gaming vectors: (1) **request less than you need** to lower your bill, then thrash on preemption — countered by the preemptible discount making honest best-effort declaration rational, and by priority pricing so that under-requesting means waiting. (2) **Pad requests** to guarantee capacity, then idle — countered directly by **allocation-based chargeback** (you pay for the pad) plus **idle reclaim** (lesson 03: sustained-idle GPUs get preempted back to the pool regardless of reservation). Allocation billing and idle reclaim together make both padding and hoarding lose money and lose the GPU.

## Perspectives

**Platform-team / incentive-design.** From this seat, chargeback design is applied game theory, not accounting. Every rule you write is a payoff structure someone will optimize against; the job is choosing rules whose Nash equilibrium is the behavior you actually want (release idle GPUs, declare true flexibility honestly) rather than the behavior that's easiest to game (pad requests, hoard capacity, under-declare need). The allocated-vs-utilised decision and the preemptible discount are both, at heart, mechanism-design choices dressed up as billing policy.

**Finance.** From the CFO's-office view, the floating blended rate is a fixed-cost recovery mechanism, not a punishment. A fleet is a sunk, largely fixed monthly cost; someone has to absorb it, and the floating-rate approach simply makes that absorption visible and proportional to who's actually holding the scarce resource, rather than burying it as an unexplained "platform tax" line that hides the utilisation signal finance actually wants to see.

**Tenant / team.** From inside a team being billed, allocation-based chargeback can feel unfair in the moment: "we're paying for GPUs we're not using." That reaction is exactly the point — it's the political dimension of being charged for a *hold*, not a *use*. The discomfort is the mechanism working; a team that finds the bill uncomfortable has a rational reason to right-size its reservation or move to a preemptible tier, which is precisely the behavior change chargeback exists to produce.

**Scheduler / queue-economics.** From the scheduler's vantage point, an unpriced queue is just a waiting line that resolves by whoever escalates loudest to a manager. Pricing urgency (priority tiers) and pricing patience (preemptible discounts) replaces that political escalation path with a legible market: anyone can see the price of jumping the queue and decide whether their job is worth it, instead of the outcome depending on whose VP shows up in the Slack channel.

## Real-world use cases

- **CoreWeave, "Kueue: A Kubernetes-Native System for AI Training Workloads"** — <https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads> — what it shows: CoreWeave's own production use of Kueue for AI training workload queueing and fair-share scheduling, directly matching this lesson's queue-wait/fair-share mechanism — including the "all-at-once" gang-scheduling semantics GPU training needs that a plain Kubernetes scheduler doesn't provide.
- **Red Hat / OpenShift AI, "Lab Guide: Advanced GPU Quota Management and Preemption with Kueue"** — <https://redhat-ai-services.github.io/rhoai-platform-foundation-bootcamp-instructions/modules/95_kueue_fair_sharing.html> — what it shows: a concrete, worked-through reference for `ClusterQueue` cohorts, `nominalQuota`, `lendingLimit`, `borrowingLimit`, and `reclaimWithinCohort` — the exact primitives this lesson's fair-share-quota section names, demonstrated with a high-priority team preempting a low-priority team's workload inside a shared cohort.
- **NVIDIA developer blog, "Ensuring Balanced GPU Allocation in Kubernetes Clusters with Time-Based Fairshare"** — <https://developer.nvidia.com/blog/ensuring-balanced-gpu-allocation-in-kubernetes-clusters-with-time-based-fairshare/> — what it shows: a primary vendor source (NVIDIA Run:ai / KAI Scheduler) on the exact time-based fairshare pattern this lesson's queue-pricing section describes conceptually — the scheduler tracks historical usage so teams that recently consumed more over-quota capacity get lower priority next time, preventing a team with frequent small jobs from permanently starving a team with occasional large ones.

## Worked example

Shared **64-GPU H100 fleet**, owned/committed. Monthly fixed cost (depreciation + reserved-commit + power + DC + platform staff, ref lesson 10) = **$225,000/mo**. Calendar capacity = 64 * 730 = 46,720 GPU-h. This month, fleet-wide *chargeable* (allocated) utilisation = **68%** → chargeable GPU-h = 31,770.

Blended internal rate = $225,000 / 31,770 = **$7.08/GPU-h** (note this is well above any external sticker precisely because 32% of the fleet is unrecovered idle — the honest signal).

Three teams:

| Team | Allocated GPU-h | Utilised GPU-h | Util % | Queue tier | Base charge (alloc × $7.08) | Queue premium | Total charged |
|---|---|---|---|---|---|---|---|
| Research-LLM | 18,000 | 15,300 | 85% | Standard | $127,440 | $0 | $127,440 |
| Product-Infer | 8,770 | 8,330 | 95% | Priority (+20%) | $62,092 | +$12,418 | $74,510 |
| Exploratory | 5,000 | 1,500 | 30% | Best-effort (−15% on utilised) | see note | — | $23,020 |

Notes:
- **Research-LLM** and **Product-Infer** are charged on **allocation** at the blended rate. Product-Infer chose the Priority tier for guaranteed-start on its serving-adjacent training, paying a 20% premium ($12,418) — funding the headroom that keeps it front-of-queue.
- **Exploratory** is on the best-effort/preemptible tier, so it is billed on **utilised** GPU-h at a 15% discount: 1,500 × $7.08 × 0.85 = $9,027... but note its **allocated 5,000 vs utilised 1,500 → 70% waste gap**. Because it accepted preemptible terms it isn't penalized for the hold (its idle GPUs were reclaimable and backfilled), but the **showback report flags the 3,500 GPU-h waste** prominently. If Exploratory instead sat on a standard reservation at 30% utilisation, allocation billing would have charged it 5,000 × $7.08 = **$35,400** for 1,500 GPU-h of actual work — the ~$26k delta is exactly the reserve-and-idle penalty that pushes teams onto preemptible or off idle reservations. (Statement total reconciled to fixed cost via the platform under-recovery line for the unallocated 32%.)
- **Utilisation call-out (showback):** fleet at 68% means the blended rate is $7.08; at 85% it would fall to ~$5.66. The report states plainly: *collective under-utilisation cost every team ~25% on their rate this month.*

The statement each team receives: allocated GPU-h, utilised GPU-h, waste gap, tier, blended rate, base charge, queue premium/discount, total, and the fleet-utilisation-vs-rate call-out.

## Practice

Feeds [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md): build a monthly showback/chargeback statement generator for a shared fleet. Inputs: per-team allocated and utilised GPU-h, chosen queue tier, and the fleet's fixed monthly cost. Compute the blended internal rate from fleet-wide chargeable utilisation, apply allocation-based charging (with the preemptible-tier exception), add the queue premium/discount line, and produce per-team statements plus a fleet summary that surfaces the waste ledger and the utilisation-vs-rate signal. Then write a half-page defense of "charge allocation, report utilisation" and the floating blended rate that you could hand to a skeptical finance partner.

## Common pitfalls

- **Charging utilised GPU-hours instead of allocated.** The central argument of this lesson, and the top pitfall: billing only busy time lets a team hold 80% of a reservation idle for free, denying it to everyone else at no cost to themselves. Correction: charge allocation, report utilisation, and use idle reclaim (lesson 03) as the enforcement backstop.
- **Using a static (non-floating) internal rate.** A fixed sticker rate hides the true cost of collective under-utilisation — teams see no signal when fleet-wide waste rises, so nothing pressures anyone to fix it. Correction: recompute the blended rate from actual fleet-wide chargeable utilisation every billing period and publish it.
- **Believing showback alone changes behavior.** Dashboards nobody's budget depends on get ignored; visibility without a financial consequence rarely moves a team that's already resource-constrained on other priorities. Correction: pair showback (the waste ledger) with chargeback (the real bill) — visibility drives awareness, money drives action.
- **Applying identical rules to preemptible/best-effort workloads as to guaranteed ones.** Charging a preemptible workload full allocation-based rates kills the incentive to honestly declare flexibility — teams will just request guaranteed capacity instead, defeating the whole point of offering a discount tier. Correction: bill preemptible/best-effort workloads on utilised GPU-hours at a discount, explicitly rewarding the honest declaration.
- **Under-pricing (or not pricing at all) queue-wait/urgency.** Leaving priority unpriced means a scarce resource gets allocated by political escalation — whoever complains loudest to a manager — rather than through a legible, fair mechanism. Correction: implement priced tiers (priority premium, standard, preemptible discount) so urgency is a transaction, not a favor.

## Self-check

- A team argues "we only used 40% of our reserved GPUs, so we should only pay for 40%." Why do you charge them for 100% of the allocation, and what do you show them instead? **Answer:** They held the other 60% out of the shared pool where nobody else could use it, and the fleet's cost is fixed regardless of their utilisation — charging only utilised would under-recover fleet cost and reward hoarding. You charge allocated (the hold) and *show* utilised, surfacing the 60% as a waste ledger, with idle reclaim (lesson 03) as the enforcement so the unused GPUs return to the pool if the idle persists.
- Fleet utilisation drops from 85% to 50% this month. What happens to the blended internal $/GPU-hr, and why is that the *correct* signal rather than a billing bug? **Answer:** It rises (the same fixed cost is divided by fewer chargeable GPU-h, so ~$5.66 → ~$9.62 in the example's shape). It's correct because the fleet's fixed cost must still be recovered — an idle fleet is *more* expensive per delivered GPU-hour, not less, and floating the rate makes the whole org feel the cost of collective under-utilisation, creating pressure to fix it.
- How does queue-wait pricing make a flexible batch-eval workload and an urgent serving-adjacent training job both behave the way the platform wants? **Answer:** The flexible workload takes the preemptible discount, so it's cheapest to declare its true flexibility, and it backfills idle troughs — raising fleet utilisation and lowering the blended rate for everyone. The urgent job takes the priority tier and pays a premium for guaranteed front-of-queue start, funding the reserved headroom. Flexible demand subsidizes fleet economics; urgent demand pays for the responsiveness it needs — incentives aligned without a scheduler fight.
- A team consistently under-requests GPUs to keep its allocation-based bill low, then relies on preemption/thrash to get its work done anyway. What two mechanisms from this lesson close that loophole, and how? **Answer:** Priority pricing and the preemptible discount together close it. If the team's workload genuinely needs guaranteed capacity, under-requesting just means it waits or gets preempted repeatedly — costing it time, not money, which isn't actually cheaper for a team that needs the work done on schedule. If the workload can genuinely tolerate preemption, the honest move is to *declare* it best-effort and take the preemptible discount rather than pretend to need a guaranteed slot — so under-declaring flexibility to dodge the bill has no upside over honestly declaring it and being rewarded with a lower rate.

## Connections & what's next

This lesson is the synthesis point for lessons 01-07: attribution (01) and the allocated-vs-utilised ledgers (02) are the metering inputs; idle detection (03) is the enforcement lever that makes allocation billing non-gameable; fragmentation cost (04) and unit economics (05) shape what "a GPU-hour" and "a unit of work" mean on the statement; commitment strategy (06) and the normalized rate from lesson 07 set the fixed cost that the blended rate has to recover. Next: **lesson 09 — Where existing tooling fails: reading the OpenCost source** is a deliberate contrast. This lesson described what internal chargeback *should* do on a shared GPU fleet; lesson 09 shows what the leading open-source cost tool actually does — and names, from its source code, exactly where that gap opens up between the allocation-based, utilisation-aware chargeback you just designed and what ships by default.

## References & further reading

**Primary sources**
- Kueue documentation — quotas, cohorts, fair-sharing, preemption — <https://kueue.sigs.k8s.io/docs/concepts/> — read for the authoritative `ClusterQueue`/cohort/quota API this lesson's fair-share section is built on.
- Kubernetes ResourceQuota and priority/preemption — <https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/> — read for the underlying preemption primitive Kueue and Run:ai both build on.
- OpenCost — Kubernetes cost allocation and showback — <https://www.opencost.io/docs/> — read for the leading OSS showback tool's data model, which lesson 09 examines in depth for its GPU gaps.
- Run:ai / NVIDIA fair-share GPU scheduling and quota concepts — <https://docs.run.ai/latest/Researcher/scheduling/the-runai-scheduler/> — read for a second production fair-share scheduler's quota and priority model, alongside Kueue's.

**Real-world engineering blogs**
- CoreWeave, "Kueue: A Kubernetes-Native System for AI Training Workloads" — <https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads> — what it shows: production use of Kueue for AI training workload queueing and fair-share at a neocloud.
- Red Hat / OpenShift AI, "Lab Guide: Advanced GPU Quota Management and Preemption with Kueue" — <https://redhat-ai-services.github.io/rhoai-platform-foundation-bootcamp-instructions/modules/95_kueue_fair_sharing.html> — what it shows: a worked, reproducible lab for `nominalQuota`/`lendingLimit`/`borrowingLimit`/`reclaimWithinCohort` in a shared cohort with preemption.
- NVIDIA developer blog, "Ensuring Balanced GPU Allocation in Kubernetes Clusters with Time-Based Fairshare" — <https://developer.nvidia.com/blog/ensuring-balanced-gpu-allocation-in-kubernetes-clusters-with-time-based-fairshare/> — what it shows: a primary vendor source on time-based fairshare, the exact queue-pricing pattern this lesson describes conceptually.

**Deeper dives**
- FinOps Foundation, "Chargeback and Showback" — <https://www.finops.org/framework/capabilities/invoicing-chargeback/> — the general FinOps framework this lesson extends to the GPU-specific allocated-vs-utilised fork.

[💰 11 — GPU cost and unit economics](../README.md)
