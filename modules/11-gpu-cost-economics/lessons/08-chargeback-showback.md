---
lesson: 08
title: "Chargeback, showback, and queue-wait billing"
module: 11
concept: "internal GPU cost allocation"
status: not-started
est_time: "3 hrs"
artifacts: ["a monthly showback/chargeback statement for 3 teams on a shared 64-GPU fleet with a blended internal rate and a queue-priority premium line"]
---

# Chargeback, showback, and queue-wait billing

## Why this matters

Once your org owns or commits to a GPU fleet, the fleet's cost is fixed and the internal question flips from "what does a GPU-hour cost the company" to "who inside the company pays for it, and how do we make them stop hoarding." This is the mechanism that turns the attribution plumbing from lessons 01-04 into behavior change. Attribution without allocation is just a dashboard nobody acts on; allocation without good attribution is a billing fight nobody wins.

The stakes are concrete and political. A research org with a shared 256-GPU cluster where teams reserve-and-idle is bleeding money in the most invisible way possible: the GPUs are "allocated" (so the fleet looks busy) but doing nothing (the util-lie, lesson 05). Whoever runs the platform gets asked, in the same breath, "why is our GPU bill so high" and "why can't my team ever get GPUs." Chargeback/showback design is how you answer both — by making the cost of hoarding land on the hoarder's budget and the cost of the fleet land honestly across tenants.

You know showback and chargeback generically from FinOps. What's new and genuinely hard here is applying them to a *shared GPU fleet* where the unit is scarce, lumpy (whole GPUs, often whole nodes of 8), fixed-cost, and where "allocated" and "utilised" can differ by 3x. The generic FinOps playbook — tag resources, split the bill — breaks on GPUs precisely at the allocated-vs-utilised fork, and picking the wrong ledger quietly rewards the worst behavior.

## What's new here

- Not re-teaching showback vs chargeback definitions — you have those. The new content is the **GPU-specific allocation decisions**: which ledger to charge, how to recover fixed cost across low utilisation, and how to price the queue.
- New: the **allocated-vs-utilised charging decision** and why "charge allocation, report utilisation" is the defensible standard.
- New: the **blended internal rate** that floats with fleet utilisation — the honest, unpopular signal that low utilisation makes internal GPU-hours *more* expensive, not less.
- New: **queue-wait / fair-share billing** — pricing the queue itself (priority tiers, fair-share quotas, preemptible discounts) in a Kueue-style shared cluster (ref mod 06) so flexible workloads subsidize urgent ones.
- Reused, not re-taught: per-pod attribution (04/05) is the input; idle reclaim (lesson 03) is the enforcement lever that makes allocation-based chargeback non-gameable.

## Core notes

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

An owned or committed fleet is a **largely fixed cost** (lesson 10's crossover math): the depreciation, the reserved-commit spend, the power and DC contracts are owed regardless of utilisation. Chargeback must recover that fixed cost across tenants — which means the internal rate cannot be a static sticker. It must **float with fleet utilisation**:

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
- **Fair-share quotas per team.** Each team gets a nominal share (cohort/ClusterQueue in Kueue terms); borrowing above your share is allowed when the fleet is idle but is the first to be preempted and can be priced at the premium tier.
- **Preemptible discount.** Workloads that tolerate preemption (checkpointing training, batch eval, offline data gen) run at a discount and backfill the gaps. They *subsidize* the urgent workloads: flexible demand fills troughs (raising fleet utilisation, lowering the blended rate for everyone), urgent demand pays the premium that funds keeping headroom available.

The incentive alignment is the whole point: **flexible workloads pay less and improve fleet economics; urgent workloads pay more and fund the responsiveness they demand.** This is a real mechanism at research platforms, not a thought experiment — it converts the queue from a source of political escalations into a priced market.

### Governance and anti-gaming

- **Budgets, quotas, alerts.** Per-team monthly GPU budgets; quota caps in the scheduler; alerts on projected overspend before month-end, not after.
- **Monthly showback per namespace/team.** Allocated GPU-h, utilised GPU-h, waste gap, blended rate, charged amount, queue-premium spend.
- **Anti-gaming.** Two gaming vectors: (1) **request less than you need** to lower your bill, then thrash on preemption — countered by the preemptible discount making honest best-effort declaration rational, and by priority pricing so that under-requesting means waiting. (2) **Pad requests** to guarantee capacity, then idle — countered directly by **allocation-based chargeback** (you pay for the pad) plus **idle reclaim** (lesson 03: sustained-idle GPUs get preempted back to the pool regardless of reservation). Allocation billing and idle reclaim together make both padding and hoarding lose money and lose the GPU.

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

## Self-check

- A team argues "we only used 40% of our reserved GPUs, so we should only pay for 40%." Why do you charge them for 100% of the allocation, and what do you show them instead? **Answer:** They held the other 60% out of the shared pool where nobody else could use it, and the fleet's cost is fixed regardless of their utilisation — charging only utilised would under-recover fleet cost and reward hoarding. You charge allocated (the hold) and *show* utilised, surfacing the 60% as a waste ledger, with idle reclaim (lesson 03) as the enforcement so the unused GPUs return to the pool if the idle persists.
- Fleet utilisation drops from 85% to 50% this month. What happens to the blended internal $/GPU-hr, and why is that the *correct* signal rather than a billing bug? **Answer:** It rises (the same fixed cost is divided by fewer chargeable GPU-h, so ~$5.66 → ~$9.62 in the example's shape). It's correct because the fleet's fixed cost must still be recovered — an idle fleet is *more* expensive per delivered GPU-hour, not less, and floating the rate makes the whole org feel the cost of collective under-utilisation, creating pressure to fix it.
- How does queue-wait pricing make a flexible batch-eval workload and an urgent serving-adjacent training job both behave the way the platform wants? **Answer:** The flexible workload takes the preemptible discount, so it's cheapest to declare its true flexibility, and it backfills idle troughs — raising fleet utilisation and lowering the blended rate for everyone. The urgent job takes the priority tier and pays a premium for guaranteed front-of-queue start, funding the reserved headroom. Flexible demand subsidizes fleet economics; urgent demand pays for the responsiveness it needs — incentives aligned without a scheduler fight.

## Resources

- Kueue documentation — quotas, cohorts, fair-sharing, preemption — https://kueue.sigs.k8s.io/docs/concepts/
- FinOps Foundation, "Chargeback and Showback" — https://www.finops.org/framework/capabilities/invoicing-chargeback/
- OpenCost — Kubernetes cost allocation and showback — https://www.opencost.io/docs/
- Kubernetes ResourceQuota and priority/preemption — https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/
- Run:ai / NVIDIA fair-share GPU scheduling and quota concepts — https://docs.run.ai/latest/Researcher/scheduling/the-runai-scheduler/

[💰 11 — GPU cost and unit economics](../README.md)
