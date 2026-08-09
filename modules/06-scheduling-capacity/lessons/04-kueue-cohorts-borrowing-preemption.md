---
lesson: "06.4"
title: "Kueue cohorts: borrowing, lending, and preemption"
module: "06"
concept: "Kueue cohorts: borrowing, lending, and preemption"
status: not-started
est_time: "10h"
artifacts: []
---
# 06.4 · Kueue cohorts: borrowing, lending, and preemption
> **Concept.** A cohort lets queues lend idle floors to each other and reclaim them by preemption — utilisation without giving up guarantees.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + showback](../practice/kueue-showback/README.md)

## Why this matters

06.3 gave every team a hard floor. Floors are safe and wasteful: if research owns 8 A100s and is idle over the weekend, those 8 GPUs sit dark while product's queue backs up. On a fixed fleet, idle guaranteed capacity is money on fire — this is exactly the FinOps line you are hired to defend. **Cohorts** solve it: pool the ClusterQueues, let a busy queue *borrow* a neighbour's idle floor, and *reclaim* it by preemption the moment the owner comes back. You raise utilisation of a fixed fleet **without** anyone losing their guarantee.

This is the deepest, most interview-tested Kueue lesson. "Explain borrowingLimit vs lendingLimit," "when does fair-sharing pick a different victim than priority preemption," "why can't borrowWithinCohort run under fair sharing" — these are real screens at GPU-heavy shops. Master the mechanics *and* the cost/fairness reasoning behind each knob.

## What's new here (vs 06.3 and vs 04)

- **vs 06.3:** there, `nominalQuota` was a hard ceiling. Here it becomes a **guaranteed floor you can exceed** by borrowing from cohort siblings, and a **floor you may lend** to them. New knobs: `cohort`, `borrowingLimit`, `lendingLimit`, `preemption.*`, `WorkloadPriorityClass`, and fair sharing.
- **vs 04 (`ResourceQuota`):** `ResourceQuota` has no concept of lending idle capacity or of evicting a running pod to give an owner its guarantee back. Kueue preemption **evicts admitted Workloads** (re-suspends the Job) to reclaim borrowed quota. That is a scheduling decision in a controller, not an apiserver rejection.

## Core notes

### Cohorts: the borrowing group

A **cohort** is a set of ClusterQueues that share their unused quota. Two ways to declare it:

```yaml
# classic: a string on each ClusterQueue
spec:
  cohort: gpu-cohort         # any CQs with the same string share
```

or the explicit **Cohort API** (v1beta1, for hierarchy and cohort-level quota):

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: Cohort
metadata: { name: gpu-cohort }
spec: {}                     # can carry its own resourceGroups / parent
```

Within a cohort, a ClusterQueue can admit up to `nominalQuota + (borrowable idle from siblings)`. Its own nominal is always reserved *for it* — borrowing only taps quota siblings aren't currently using.

### `borrowingLimit` vs `lendingLimit` — the two caps

Both are optional per-resource fields inside a flavor's `resources`:

```yaml
resourceGroups:
- coveredResources: ["nvidia.com/gpu"]
  flavors:
  - name: a100
    resources:
    - name: "nvidia.com/gpu"
      nominalQuota: 8
      borrowingLimit: 4      # I may borrow at most 4 ON TOP of my 8 -> admit up to 12
      lendingLimit: 2        # I will lend at most 2 of my 8 -> I always keep >=6
```

- **`borrowingLimit`** caps how much this queue may take **from** the cohort, *above* its nominal. Absent = unlimited (up to whatever the cohort has). It protects the *cohort* from one queue hoovering all spare capacity.
- **`lendingLimit`** caps how much of this queue's nominal it will expose **to** the cohort. Absent = the whole idle nominal is lendable. It protects the *owner* — with `lendingLimit: 2`, even when idle you keep 6 GPUs instantly reclaimable, so your own bursty jobs admit fast without waiting on preemption.

Memory hook: **borrowingLimit = how much I can take; lendingLimit = how much I let others take.** One caps consumption above floor, the other caps generosity below floor.

### Preemption: reclaiming what was borrowed

When a queue needs quota it can't get (pool exhausted, or an owner wants its lent floor back), Kueue can **preempt** — evict admitted Workloads to free quota. Config lives on the ClusterQueue:

```yaml
spec:
  preemption:
    reclaimWithinCohort: Any          # reclaim BORROWED quota from other CQs
    borrowWithinCohort:               # may I preempt in order to borrow?
      policy: LowerPriority
      maxPriorityThreshold: 100
    withinClusterQueue: LowerPriority # preempt my own lower-prio workloads
```

Three independent levers:

- **`withinClusterQueue`** (`Never` | `LowerPriority` | `LowerOrNewerEqualPriority`): may a pending Workload preempt *this queue's own* admitted Workloads? Priority-driven.
- **`reclaimWithinCohort`** (`Never` | `LowerPriority` | `Any`): may this queue reclaim quota that *other* cohort queues borrowed from it (or that pushes them below their nominal)? `Any` = reclaim regardless of the victim's priority — this is how an owner *guarantees* its floor back. This is the star of the borrow-then-reclaim demo.
- **`borrowWithinCohort`**: may a Workload preempt cohort siblings *in order to borrow* (go above its own nominal)? Bounded by `maxPriorityThreshold`.

Eviction re-suspends the victim Job (`spec.suspend=true`, pods deleted) and re-queues its Workload with condition `Evicted`/`Preempted` and an event. The victim isn't lost — it goes back to waiting.

**Workload priority** comes from a `WorkloadPriorityClass` (separate from pod `PriorityClass`, so queueing priority ≠ node-preemption priority):

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: WorkloadPriorityClass
metadata: { name: high }
value: 1000
```
Reference it on the Job: `labels: { kueue.x-k8s.io/priority-class: high }`.

### Classic (priority) preemption vs fair sharing

**Classic** preemption is **priority + borrowing-state** driven. Candidates to evict are chosen by: is the queue over nominal (borrowing)? is the victim lower priority? It answers "who is using quota they don't own, cheapest to reclaim by priority." It does **not** track history — a queue that hogged the fleet all week has no disadvantage today.

**Fair sharing** changes the victim-selection *objective* from priority to **share equalisation**. Enabled cluster-wide in the Kueue Configuration:

```yaml
apiVersion: config.kueue.x-k8s.io/v1beta1
kind: Configuration
fairSharing:
  enable: true
  preemptionStrategies: [LessThanOrEqualToFinalShare, LessThanInitialShare]
```
and each ClusterQueue may carry a weight:
```yaml
spec:
  fairSharing:
    weight: 1            # higher weight = entitled to a larger share
```

Under fair sharing, Kueue computes each queue's **dominant resource share** (usage ÷ weight, normalised across the cohort) and preempts to *reduce the borrowing queue's share toward parity*. The `preemptionStrategies` gate whether a preemption is allowed by comparing the preemptor's and target's shares before/after: only evict if it doesn't overshoot fairness.

**When fair sharing picks a different victim than classic:** consider queues A (priority-100 workloads) and B (priority-1 workloads), both borrowing, but A has consumed far more of the cohort over time / holds a larger current share. A new Workload arrives needing to preempt one. **Classic** evicts B's workload — it is lower priority, end of story. **Fair sharing** evicts **A's** workload despite its higher priority, because A's *share* is larger and fairness says the over-consuming queue should give back first. Priority orders *within* a share bracket; fair sharing orders *across* queues by historical/current usage. That inversion — high-priority victim chosen because its queue is greedy — is the classic interview trap.

Cost framing: classic = "important work wins, and greedy history is free." Fair sharing = "sustained hogging is penalised; no queue can camp on borrowed GPUs indefinitely." Pick fair sharing when many teams share one fleet and you must defend equitable long-run access; pick classic when a strict priority order (prod > batch) must always hold.

### Why `borrowWithinCohort` doesn't combine with fair sharing

`borrowWithinCohort` is a **priority-threshold** mechanism: "you may preempt to borrow, but only victims below `maxPriorityThreshold`." Its whole selection logic is expressed in *priority* terms. Fair sharing **replaces** priority-based victim selection with **share**-based selection — the two speak different currencies. There is no coherent way to honour a priority threshold while also equalising shares, so Kueue **rejects the combination**: enabling fair sharing forbids `borrowWithinCohort` (validation error). Under fair sharing, borrowing-driven preemption is already governed by the share strategies, so the priority-threshold knob is both redundant and contradictory.

### The tradeoff, stated

Hard floors (06.3) maximise *predictability*, minimise *utilisation*. Borrowing raises fixed-fleet utilisation by letting idle floors work, while `nominalQuota` + `reclaimWithinCohort: Any` keeps the guarantee (you always get your floor back, by force if needed). `lendingLimit` tunes how much guarantee you trade for reclaim-latency. Fair sharing adds historical-usage fairness on top. This is the FinOps dial: more borrowing = higher utilisation = lower $/token, at the cost of some jobs being evicted and re-queued. Showback is still per ClusterQueue — but now you report *nominal (owned)* vs *borrowed* GPU-hours separately, because borrowed hours are the cross-team subsidy you want visible.

## Worked example

Cohort `gpu-cohort`, 16 A100s: `cq-research` and `cq-product`, 8 each. Research lends freely; product may borrow up to +8; research reclaims with `Any`.

```yaml
# cq-research
spec:
  cohort: gpu-cohort
  preemption: { reclaimWithinCohort: Any, withinClusterQueue: LowerPriority }
  resourceGroups:
  - coveredResources: ["nvidia.com/gpu"]
    flavors:
    - name: a100
      resources: [{ name: "nvidia.com/gpu", nominalQuota: 8 }]   # lendingLimit omitted = lend all idle
---
# cq-product
spec:
  cohort: gpu-cohort
  preemption: { reclaimWithinCohort: Any, withinClusterQueue: LowerPriority }
  resourceGroups:
  - coveredResources: ["nvidia.com/gpu"]
    flavors:
    - name: a100
      resources: [{ name: "nvidia.com/gpu", nominalQuota: 8, borrowingLimit: 8 }]  # may reach 16
```

Timeline: research idle. Product submits a 16-GPU job → admits by using its 8 nominal + borrowing research's 8 idle. Then research submits an 8-GPU job → it wants its floor, cohort is full, so `reclaimWithinCohort: Any` **preempts product's borrowed workload** (evicts enough to free 8), product re-queues, research admits. You just watched utilisation stay at 100% *and* the guarantee honoured.

## Practice (kind + fake GPUs → deliverable)

Continue from 06.3's cluster (16 fake `nvidia.com/gpu`, `gpu-type=a100`, Kueue installed, two CQs + LQs).

```bash
# 1. Put both CQs in the cohort and set preemption + borrowing (patch the 06.3 CQs)
kubectl patch clusterqueue cq-research --type merge -p '{"spec":{"cohort":"gpu-cohort",
  "preemption":{"reclaimWithinCohort":"Any","withinClusterQueue":"LowerPriority"}}}'
kubectl patch clusterqueue cq-product  --type merge -p '{"spec":{"cohort":"gpu-cohort",
  "preemption":{"reclaimWithinCohort":"Any","withinClusterQueue":"LowerPriority"}}}'
# give product a borrowingLimit so it can reach 16
kubectl patch clusterqueue cq-product --type json -p \
  '[{"op":"add","path":"/spec/resourceGroups/0/flavors/0/resources/0/borrowingLimit","value":8}]'

# 2. Fill product to 16 while research is idle: submit a 16-GPU sleeper to lq-product
#    (one job requesting 16, or four requesting 4). It admits by BORROWING research's 8.
kubectl create -f job-product-16gpu.yaml
kubectl get workloads -n product   # ADMITTED=True, borrowing 8 above nominal

# 3. Now make research reclaim: submit an 8-GPU job to lq-research
kubectl create -f job-research-8gpu.yaml

# 4. WATCH the reclaim-by-preemption: product's borrowed workload is evicted
kubectl get workloads -A -w        # research flips to admitted; product's flips Evicted
kubectl get events -n product --sort-by=.lastTimestamp | grep -i -E 'preempt|evict'
# describe the victim to capture the condition + reason
kubectl describe workload <product-victim> -n product | sed -n '/Conditions/,/Events/p'
```

Expected: product's Workload shows `Evicted=True` reason `Preempted` / message referencing reclamation within cohort; its Job re-suspends (pods deleted); research's Job unsuspends. Product re-queues and re-admits only when research frees the 8 again.

**Acceptance (feeds the deliverable):**
- Both ClusterQueues in `gpu-cohort` with `reclaimWithinCohort: Any`; `cq-product` has `borrowingLimit: 8`.
- A captured **borrow event**: product admitted above its nominal 8 (its Workload/CQ status shows borrowing).
- A captured **reclaim-by-preemption**: research submits, product's borrowed Workload shows `Evicted`/`Preempted`, and the eviction **event** is saved (`kubectl get events` line + the `describe` condition block).
- One paragraph in the deliverable README explaining the utilisation-vs-guarantee tradeoff you just demonstrated, in $/GPU-hour terms.

**Stretch:** enable fair sharing in the Kueue Configuration (`fairSharing.enable: true`, restart the controller), set unequal `fairSharing.weight`, and show a *higher-priority* borrowed workload getting preempted because its queue's share is larger — the classic-vs-fair victim inversion.

## Self-check

**(a) `borrowingLimit` vs `lendingLimit` — what does each cap?**
**Answer:** `borrowingLimit` caps how much a ClusterQueue may take **from** the cohort *above* its own `nominalQuota` (protects the cohort from one greedy queue; absent = unlimited). `lendingLimit` caps how much of its **own nominal** the queue will expose to siblings (protects the owner; absent = lend all idle). One limits consumption above the floor, the other limits generosity below the floor. With `lendingLimit: 2` on an 8-GPU floor you always keep ≥6 instantly reclaimable.

**(b) When does fair-sharing preemption pick a different victim than classic priority-based preemption?**
**Answer:** When the queue with the *larger share* holds the *higher-priority* workloads. Classic preemption evicts the lowest-priority borrowed workload regardless of which queue has been hogging the fleet. Fair sharing selects by dominant-resource share, so it will evict a **higher-priority** workload belonging to the over-consuming queue to pull that queue's share back toward parity. Priority orders within a bracket; fair sharing orders across queues by usage — so a greedy queue's important job can be chosen over a frugal queue's trivial one.

**(c) Why does `borrowWithinCohort` not combine with fair sharing?**
**Answer:** `borrowWithinCohort` selects preemption victims by a **priority threshold** (`maxPriorityThreshold`) — its logic is expressed entirely in priority terms. Fair sharing **replaces** priority-based victim selection with **share**-equalisation and its own `preemptionStrategies`. The two use incompatible currencies (priority vs share), so honouring a priority threshold while equalising shares is incoherent; Kueue rejects the combination with a validation error. Under fair sharing, borrow-driven preemption is already governed by the share strategies, making the knob redundant.

## Resources

1. **Kueue Preemption concept** — `reclaimWithinCohort`, `borrowWithinCohort`, `withinClusterQueue`, and the candidate-selection algorithm. https://kueue.sigs.k8s.io/docs/concepts/preemption/
2. **Kueue Fair Sharing concept** — dominant-resource share, weights, `preemptionStrategies`, and the borrowWithinCohort incompatibility. https://kueue.sigs.k8s.io/docs/concepts/fair_sharing/
3. **Kueue Cohort concept** — cohort semantics, `borrowingLimit`/`lendingLimit`, hierarchical Cohort API. https://kueue.sigs.k8s.io/docs/concepts/cohort/
