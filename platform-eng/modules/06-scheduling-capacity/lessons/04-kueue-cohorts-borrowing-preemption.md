---
lesson: "06.4"
title: "Kueue II — cohorts: borrowing, lending, preemption, and fair sharing"
module: "06"
concept: "Kueue cohorts: borrowing, lending, preemption, and fair sharing"
status: not-started
est_time: "13h"
prev: "03-kueue-queueing-model.md"
next: "05-alternatives-volcano-kai.md"
artifacts: []
sources: 11
---
# 06.4 · Kueue II — cohorts: borrowing, lending, preemption, and fair sharing

> **Concept.** A cohort lets queues lend idle floors to each other and reclaim them by preemption — utilisation without giving up guarantees.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits

06.3 built the queueing model and gave every team a hard, isolated floor via `nominalQuota` — safe, legible, but wasteful the instant one team is idle while another queues. This lesson removes that waste without removing the guarantee: it groups ClusterQueues into a **cohort**, lets a busy queue **borrow** an idle sibling's unused floor, and lets the owner **reclaim** it by preemption the moment it needs it back. It also introduces the fairness math — dominant resource share, `fairSharing`, and its interaction with 06.3's `AdmissionFairSharing` — that decides *who* gets evicted when the fleet is tight. This is the deepest, most interview-tested Kueue lesson in the module; after this you have the full anchor and can move to 06.5's scheduler comparison with real footing.

## Why this matters

If research owns 8 A100s and is idle over the weekend, those 8 GPUs sit dark while product's queue backs up — on a fixed fleet, idle guaranteed capacity is money on fire, exactly the FinOps line this module exists to teach. Cohorts, borrowing, and preemption are how you raise utilisation of a fixed fleet **without** anyone losing their guarantee, and the mechanics are dense enough — and tested often enough — that they deserve to be over-learned, not just understood once. "Explain `borrowingLimit` vs `lendingLimit`," "when does fair-sharing preemption pick a different victim than classic priority preemption," "why can't `borrowWithinCohort` run under fair sharing" — these are real screens at GPU-heavy shops, not hypotheticals. CoreWeave's own JD for Principal/Staff Cluster Orchestration names "quota enforcement, fairness, pre-emption, and multi-tenant GPU isolation" as core to the role. IBM Research runs exactly this borrowing model in production on its Vela and Blue Vela clusters to solve idle-GPU utilisation on real, multi-hundred-A100/H100 research fleets — this is not academic. Master the mechanics *and* the cost/fairness reasoning behind each knob; interviewers probe both.

## What's new here (calibration)

- **You already know controllers and `ResourceQuota` (modules 02, 04)** — we don't re-teach either. `ResourceQuota` has no concept of lending idle capacity or of evicting a running pod to give an owner its guarantee back; that gap is the whole lesson.
- **You already have 06.3's ClusterQueue/LocalQueue/ResourceFlavor/`nominalQuota` model** — here `nominalQuota` stops being a hard ceiling and becomes a **floor you can exceed** (by borrowing) and a **floor you may lend**. New knobs: `cohort`, `borrowingLimit`, `lendingLimit`, `preemption.*`, `WorkloadPriorityClass`, and `fairSharing`.
- **Genuinely new depth:** the classic-vs-fair-sharing victim-selection inversion (the single most-tested trap in this space); the theoretical grounding of Kueue's fairness math in **Dominant Resource Fairness** (Ghodsi et al., NSDI 2011); and the interaction between cohort-level `fairSharing` and 06.3's `AdmissionFairSharing` — two fairness layers that now coexist in current Kueue releases and are easy to conflate under interview pressure.

## Core concepts

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

Within a cohort, a ClusterQueue can admit up to `nominalQuota + (borrowable idle from siblings)`. Its own nominal is always reserved *for it* — borrowing only taps quota siblings aren't currently using. Cohorts can also nest: a Cohort may itself have a parent Cohort, forming a **CohortTree**, so a platform can express "these three research-team ClusterQueues share within themselves first, and only then reach into the wider org-level pool." This hierarchy is where a live upstream design debate is actively unresolved — see the pitfalls section below.

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

Under fair sharing, a ClusterQueue with a pending Workload can preempt other Workloads *in its cohort* until it obtains an equal or weighted share of the cohort's **borrowable resources** — the unused nominal quota of every ClusterQueue in the cohort combined. The `preemptionStrategies` gate whether a given preemption is allowed by comparing the preemptor's and the target's shares before and after the move: only evict if it doesn't overshoot fairness. As of Kueue v0.11, hierarchical cohorts (any Cohort with a parent) are compatible with fair sharing, so this share computation can run across a CohortTree, not just a flat cohort.

**The theory underneath it: Dominant Resource Fairness.** The "share" Kueue equalises is not a single number when a fleet has multiple resource types (GPUs, CPU, memory) in play — it is each queue's **dominant resource share**: for each queue, find the resource type it consumes the largest *fraction* of relative to its allocation, and that fraction is its dominant share. Kueue then tries to equalise dominant shares across queues, not raw GPU counts. This is a direct, practical instance of **Dominant Resource Fairness (DRF)**, formalised by Ghodsi, Zaharia, Hindman, Konwinski, Shenker, and Stoica in "Dominant Resource Fairness: Fair Allocation of Multiple Resource Types" (NSDI 2011) — the paper that gave multi-tenant cluster schedulers (originally Mesos) a principled way to define "fair" when tenants want different mixes of resources, not just different amounts of one resource. It matters here for a concrete reason: a queue that's GPU-light but memory-heavy can have a *high* dominant share (on memory) while looking GPU-idle, and DRF-aware fair sharing accounts for that; naive "equalise GPU count" fairness would not. You'll meet DRF again in 06.5 — Volcano implements it as its *primary*, native fairness model rather than layering it on top of a quota system the way Kueue does; same theory, two different architectural choices for where it lives.

**When fair sharing picks a different victim than classic:** consider queues A (priority-100 workloads) and B (priority-1 workloads), both borrowing, but A has consumed far more of the cohort over time / holds a larger current dominant share. A new Workload arrives needing to preempt one. **Classic** evicts B's workload — it is lower priority, end of story. **Fair sharing** evicts **A's** workload despite its higher priority, because A's *share* is larger and fairness says the over-consuming queue should give back first. Priority orders *within* a share bracket; fair sharing orders *across* queues by historical/current usage. That inversion — high-priority victim chosen because its queue is greedy — is the classic interview trap, and per this module's research it is the single most-tested one among target-company JDs.

Cost framing: classic = "important work wins, and greedy history is free." Fair sharing = "sustained hogging is penalised; no queue can camp on borrowed GPUs indefinitely." Pick fair sharing when many teams share one fleet and you must defend equitable long-run access; pick classic when a strict priority order (prod > batch) must always hold.

### Why `borrowWithinCohort` doesn't combine with fair sharing

`borrowWithinCohort` is a **priority-threshold** mechanism: "you may preempt to borrow, but only victims below `maxPriorityThreshold`." Its whole selection logic is expressed in *priority* terms. Fair sharing **replaces** priority-based victim selection with **share**-based selection — the two speak different currencies. There is no coherent way to honour a priority threshold while also equalising shares, so Kueue **rejects the combination**: enabling fair sharing forbids `borrowWithinCohort` (validation error). Under fair sharing, borrowing-driven preemption is already governed by the share strategies, so the priority-threshold knob is both redundant and contradictory.

### Two fairness layers, now: `fairSharing` vs 06.3's `AdmissionFairSharing`

06.3 introduced `AdmissionFairSharing` (beta-default since Kueue v0.15) as a *within-ClusterQueue* mechanism ordering which pending Workload from which LocalQueue gets admitted next, based on decayed historical consumption. Everything in this lesson — cohort `fairSharing` — is a *different, older, cross-ClusterQueue* mechanism that acts by **preemption**, not admission order. They are not mutually exclusive, and current Kueue versions genuinely run both at once on a busy fleet:

| | `AdmissionFairSharing` (06.3) | Cohort `fairSharing` (this lesson) |
|---|---|---|
| Acts on | pending Workloads, at admission time | admitted Workloads, via preemption |
| Scope | LocalQueues within one ClusterQueue | ClusterQueues within a cohort |
| Currency | decayed GPU-second consumption per LocalQueue | dominant resource share per ClusterQueue (DRF) |
| Can it evict a running job? | no | yes |

A single Workload can be touched by both in one lifecycle: it can be admitted *later* than a sibling LocalQueue's job because `AdmissionFairSharing` ranks its LocalQueue as having consumed more recently, and — once admitted and borrowing — it can *still* be preempted later under cohort `fairSharing` if its whole ClusterQueue's dominant share grows too large relative to cohort siblings. The first is a queueing-order penalty; the second is an eviction. Interviewers who ask "how do these interact" are testing whether you understand that fairness in current Kueue is layered, not a single global mechanism — get this distinction crisp, because it's genuinely new since earlier course passes and easy to blur.

### The tradeoff, stated

Hard floors (06.3) maximise *predictability*, minimise *utilisation*. Borrowing raises fixed-fleet utilisation by letting idle floors work, while `nominalQuota` + `reclaimWithinCohort: Any` keeps the guarantee (you always get your floor back, by force if needed). `lendingLimit` tunes how much guarantee you trade for reclaim-latency. Fair sharing adds historical-usage fairness on top, at the cohort level; `AdmissionFairSharing` adds a second, finer-grained layer of it within each pool. This is the FinOps dial: more borrowing = higher utilisation = lower $/token, at the cost of some jobs being evicted and re-queued. Showback is still per ClusterQueue — but now you report *nominal (owned)* vs *borrowed* GPU-hours separately, because borrowed hours are the cross-team subsidy you want visible.

## Perspectives

**Developer/tenant.** From inside a borrowing team, admission just happens faster when a sibling is idle — but the tenant needs to understand their job can be *evicted* later when the owner reclaims. That means workloads that borrow should be checkpoint-tolerant by design, a direct bridge to lesson 08's preemption economics. A tenant that doesn't know this treats a mid-run eviction as a platform bug rather than the system working as designed.

**Operator/platform.** Choosing `reclaimWithinCohort: Any` vs a priority-gated reclaim is a fairness-vs-predictability call the platform team owns, not a default to accept blindly. `Any` gives owners a *hard* guarantee (good for prod-adjacent teams) at the cost of borrowers never being safe from eviction regardless of how important their work looked a moment ago — an operator has to be explicit about which teams get which promise.

**Systems/algorithms.** Fair sharing's dominant-resource-share computation is a direct, practical instance of Dominant Resource Fairness — worth stating explicitly because 06.5 introduces Volcano's *native* DRF, and a learner should recognise these as the same theory expressed two different architectural ways: Volcano treats DRF as its primary fairness model; Kueue layers a DRF-flavored share-equalisation mechanism on top of an otherwise quota-first system.

**Economics/FinOps.** The cohort is literally a shared insurance pool: nominal quota is the "premium" each team pays (in dedicated floor), and borrowing is the "payout" when idle. `lendingLimit` is the deductible — how much self-insurance a team keeps versus how much it pools with everyone else. This framing is a strong, concrete interview answer for "how do you think about quota design," because it maps every knob onto a financial instrument the interviewer already understands.

## Real-world use cases

- **IBM Research — Vela / Blue Vela** — arXiv 2407.05467 ("The infrastructure powering IBM's Gen AI model development") and the KubeCon EU 2025 tutorial "Build An AI Cluster Tutorial" — https://static.sched.com/hosted_files/kccnceu2025/9b/BuildAnAIClusterTutorial.pdf. What it shows: cohorts and borrowing used explicitly to let idle GPU floors serve bursty demand across research teams on a real multi-hundred-A100/H100 cluster — this lesson's borrowing/reclaim mechanics, not just 06.3's queueing, at production research-lab scale. (Egress-blocked at fetch time; search-confirmed.)
- **Netflix TechBlog — "How Netflix Simplified Batch Compute with Kueue"** (same post as 06.3, different angle here) — https://netflixtechblog.com/how-netflix-simplified-batch-compute-with-kueue-87860682629c. What it shows: at "millions of batch workloads" scale, cohort-level sharing is what makes shared infrastructure economical rather than every team getting a dedicated, over-provisioned pool — the utilisation argument this lesson makes, validated at hyperscale. (Egress-blocked at fetch time; search-confirmed.)
- **kubernetes-sigs/kueue GitHub issue #7016** — "Clusterqueues must prefer borrowing within a cohort before borrowing across cohorts" — https://github.com/kubernetes-sigs/kueue/issues/7016. What it shows: a live upstream design discussion about *hierarchical* cohort borrowing order — direct evidence that cohort-borrowing semantics are still actively evolving, worth a "verify against your version" flag consistent with the module README's TAS warning, and a good primary-source view of how these semantics get debated by the people who build them. **Fetched directly** this session: confirmed the issue proposes that a ClusterQueue with access to multiple resource flavors should exhaust borrowing within its own cohort before reaching across cohort boundaries, because the current behavior causes unnecessary cross-cohort preemptions when a parent cohort's own nominal quota was already sufficient.
- **Red Hat Developer — "Improve GPU utilization with Kueue in OpenShift AI"** — https://developers.redhat.com/articles/2025/05/22/improve-gpu-utilization-kueue-openshift-ai. What it shows: an enterprise platform vendor's own worked example of cohort borrowing solving idle-GPU utilisation on a shared OpenShift AI cluster — a more tutorial-style, vendor-neutral fourth account of the same mechanism. (Egress-blocked at fetch time; search-confirmed, and consistent in substance with the mechanism described in the fetched KEPs and issue above.)

## Worked example

Start from the two-queue example (research/product, 16 A100s) and extend it into a **three-queue hierarchical cohort** — research, product, and best-effort — to show a *chain* of borrowing and reclaim, not just a single hand-off.

```yaml
# cq-research: nominalQuota 8, reclaimWithinCohort: Any, lends freely
# cq-product:  nominalQuota 6, reclaimWithinCohort: Any, borrowingLimit: 10
# cq-besteffort: nominalQuota 2, reclaimWithinCohort: Never, borrowingLimit: 14
#   (best-effort never gets to force a reclaim from anyone — it's the cheapest to evict)
```

Timeline:

1. Fleet is idle except best-effort's own 2. Best-effort submits work that borrows from **both** research's and product's idle floors, climbing to 16 admitted GPUs.
2. Product submits an 8-GPU job. It needs to reclaim above its own nominal 6. With `reclaimWithinCohort: Any`, product's reclaim preempts **best-effort's** borrowed workloads first — best-effort is the cheapest victim (lowest priority, `reclaimWithinCohort: Never` on its own side means it can never *initiate* a reclaim against anyone, but it can still *be* reclaimed against). Product admits at 8.
3. Research submits its own 8-GPU job, needing its full floor back. Whatever best-effort capacity remains gets reclaimed first; if that's insufficient, `reclaimWithinCohort: Any` on research's side lets it reclaim from product's *borrowed* (above-nominal) portion too — product's own nominal 6 stays untouched, only the borrowed excess is fair game.

The reason `reclaimWithinCohort` differs per queue (`Any` for research and product, `Never` for best-effort) is exactly the ordering lever: **best-effort is designed to be evicted first, every time**, because it never had a real claim on the capacity — it only ever ran on other queues' idle floors. This is the direct answer to "design a `reclaimWithinCohort` policy per queue that guarantees best-effort is always evicted before product or research": give best-effort the smallest `nominalQuota`, the largest `borrowingLimit` (so it can actually use idle capacity), and set the *other two* queues' `reclaimWithinCohort: Any` so they can always take back what best-effort borrowed from them, regardless of best-effort's priority. Best-effort's own `reclaimWithinCohort` setting is close to irrelevant here — it never owns enough nominal quota to have anything worth reclaiming from someone else.

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

**Stretch:** enable fair sharing in the Kueue Configuration (`fairSharing.enable: true`, restart the controller), set unequal `fairSharing.weight`, and show a *higher-priority* borrowed workload getting preempted because its queue's share is larger — the classic-vs-fair victim inversion. **Second stretch:** add the third `cq-besteffort` queue from the worked example above and reproduce the borrowing chain end-to-end — this is the closest thing in the module to the actual 128-GPU, 3-team design the checkpoint asks you to defend.

## Common pitfalls

- **Setting `lendingLimit` to 0 "to be safe."** This defeats the entire point of a cohort; a queue that lends nothing behaves like a hard floor (06.3), and the platform loses the utilisation gains cohorts exist to provide, while still paying the operational complexity of running cohorts.
- **Enabling fair sharing without understanding the `borrowWithinCohort` incompatibility.** This is a *validation-time* error, not a silent misbehaviour — Kueue rejects the config outright — but engineers who don't know the incompatibility exists waste time debugging a rejected manifest they assumed was a syntax problem.
- **Assuming preemption always targets the "obviously wrong" victim.** Under fair sharing, the victim-selection inversion — higher priority, larger share, still evicted — surprises engineers who reason purely in priority terms. This is the single most-tested interview trap in this space per the target JDs' own emphasis on "fairness, pre-emption."
- **Conflating `AdmissionFairSharing` (06.3) with cohort `fairSharing` (this lesson).** They operate at different scopes and by different mechanisms (admission order vs. preemption); a Workload can be affected by both in the same lifecycle. Treating them as one knob under interview pressure reads as surface-level knowledge.
- **Ignoring that cohort-borrowing-order semantics are still moving upstream.** Issue #7016 shows the project itself is actively debating whether a ClusterQueue should exhaust its own cohort before reaching across a hierarchy — verify the exact behaviour against the Kueue version you're running rather than assuming last year's mental model still holds, consistent with the module README's "verify feature gates" warning.

## Self-check

- `borrowingLimit` vs `lendingLimit` — what does each cap? **Answer:** `borrowingLimit` caps how much a ClusterQueue may take **from** the cohort *above* its own `nominalQuota` (protects the cohort from one greedy queue; absent = unlimited). `lendingLimit` caps how much of its **own nominal** the queue will expose to siblings (protects the owner; absent = lend all idle). One limits consumption above the floor, the other limits generosity below the floor. With `lendingLimit: 2` on an 8-GPU floor you always keep ≥6 instantly reclaimable.
- When does fair-sharing preemption pick a different victim than classic priority-based preemption? **Answer:** When the queue with the *larger dominant resource share* holds the *higher-priority* workloads. Classic preemption evicts the lowest-priority borrowed workload regardless of which queue has been hogging the fleet. Fair sharing selects by DRF-style share, so it will evict a **higher-priority** workload belonging to the over-consuming queue to pull that queue's share back toward parity. Priority orders within a bracket; fair sharing orders across queues by usage — so a greedy queue's important job can be chosen over a frugal queue's trivial one.
- Why does `borrowWithinCohort` not combine with fair sharing? **Answer:** `borrowWithinCohort` selects preemption victims by a **priority threshold** (`maxPriorityThreshold`) — its logic is expressed entirely in priority terms. Fair sharing **replaces** priority-based victim selection with **share**-equalisation and its own `preemptionStrategies`. The two use incompatible currencies (priority vs share), so honouring a priority threshold while equalising shares is incoherent; Kueue rejects the combination with a validation error. Under fair sharing, borrow-driven preemption is already governed by the share strategies, making the knob redundant.
- How does `AdmissionFairSharing` (06.3, v0.15+) differ from cohort-level `fairSharing`, and could a single workload be affected by both simultaneously? **Answer:** `AdmissionFairSharing` acts at admission time, within one ClusterQueue, ordering pending Workloads from different LocalQueues by decayed historical consumption — it never evicts a running job. Cohort `fairSharing` acts across ClusterQueues in a cohort, using DRF-style dominant-resource-share comparisons, and *does* evict admitted Workloads via preemption. Yes — a single Workload could be admitted later than a sibling because its LocalQueue ranked poorly under `AdmissionFairSharing`, and, once admitted and borrowing, later be preempted under cohort `fairSharing` because its whole ClusterQueue's share grew too large. They're independent layers that both touch the same Workload at different points in its lifecycle.
- In a three-queue cohort (research/product/best-effort), design a `reclaimWithinCohort` policy per queue that ensures best-effort is always evicted before product or research — what field(s) enforce that ordering? **Answer:** Give best-effort the smallest `nominalQuota` and largest `borrowingLimit` (so it only ever runs on borrowed idle capacity, never owns much of its own), and set `reclaimWithinCohort: Any` on **research's and product's** ClusterQueues (not best-effort's) so they can always reclaim what best-effort borrowed from them, regardless of best-effort's priority. Best-effort's own `reclaimWithinCohort` value barely matters, since it rarely has meaningful nominal quota of its own for anyone to reclaim.

## Connections & what's next

This lesson completes the Kueue anchor: 06.3 gave you the queueing/quota vocabulary, 06.4 gave you the borrowing/preemption/fairness mechanics that make a fixed fleet economically efficient rather than merely safe. The DRF theory introduced here is the same theory 06.5 revisits when it covers Volcano's *native* DRF scheduler — you'll be able to compare "DRF as the whole scheduling model" (Volcano) against "DRF-flavored fairness layered on a quota-first controller" (Kueue) directly, having already learned the concept once. The checkpoint-tolerance implication of borrowing (a borrowed workload can be evicted mid-run) is the direct setup for lesson 08's preemption economics, where you'll quantify exactly how expensive an eviction is without frequent checkpointing. Next: **06.5 — Alternatives: Volcano & KAI**, where the same fairness and gang-scheduling problems get solved by different architectural choices, and you learn when Kueue is the wrong tool.

## References & further reading

**Primary sources**
- Kueue Preemption concept — https://kueue.sigs.k8s.io/docs/concepts/preemption/ — read for `reclaimWithinCohort`, `borrowWithinCohort`, `withinClusterQueue`, and the full candidate-selection algorithm. (Egress-blocked at fetch time; search-confirmed canonical doc URL.)
- Kueue Fair Sharing concept — https://kueue.sigs.k8s.io/docs/concepts/fair_sharing/ — read for dominant-resource share computation, weights, `preemptionStrategies`, and the `borrowWithinCohort` incompatibility, plus the hierarchical-cohort compatibility note (v0.11+). (Egress-blocked at fetch time; search-confirmed.)
- Kueue Cohort concept — https://kueue.sigs.k8s.io/docs/concepts/cohort/ — read for cohort semantics, `borrowingLimit`/`lendingLimit`, and the hierarchical Cohort/CohortTree API. (Egress-blocked at fetch time; search-confirmed.)
- Kueue Admission Fair Sharing concept — https://kueue.sigs.k8s.io/docs/concepts/admission_fair_sharing/ — read as the direct cross-reference back to 06.3's mechanism. (Egress-blocked at fetch time; search-confirmed.)
- Ghodsi, Zaharia, Hindman, Konwinski, Shenker, Stoica — "Dominant Resource Fairness: Fair Allocation of Multiple Resource Types" (USENIX NSDI 2011) — https://www.usenix.org/conference/nsdi11/dominant-resource-fairness-fair-allocation-multiple-resource-types (paper PDF also mirrored at https://amplab.cs.berkeley.edu/wp-content/uploads/2011/06/Dominant-Resource-Fairness-Fair-Allocation-of-Multiple-Resource-Types.pdf) — read for the formal theory behind Kueue's (and Volcano's, 06.5) fair-sharing math: how "fair" is defined when tenants want different resource mixes, not just different amounts of one resource. (Egress-blocked at fetch time; both URLs search-confirmed as the canonical paper and a legitimate academic mirror.)
- kubernetes-sigs/kueue GitHub issue #7016 — https://github.com/kubernetes-sigs/kueue/issues/7016 — read for the live, unresolved design debate on hierarchical cohort borrowing order. **Fetched directly** this session.

**Real-world engineering blogs**
- IBM Research — Vela/Blue Vela — arXiv 2407.05467, and KubeCon EU 2025 tutorial: https://static.sched.com/hosted_files/kccnceu2025/9b/BuildAnAIClusterTutorial.pdf — what it shows: cohorts and borrowing solving real idle-GPU utilisation on a multi-hundred-GPU research supercomputer.
- Netflix TechBlog — "How Netflix Simplified Batch Compute with Kueue" — https://netflixtechblog.com/how-netflix-simplified-batch-compute-with-kueue-87860682629c — what it shows: cohort-level sharing making shared infrastructure economical at millions-of-workloads scale.
- Red Hat Developer — "Improve GPU utilization with Kueue in OpenShift AI" — https://developers.redhat.com/articles/2025/05/22/improve-gpu-utilization-kueue-openshift-ai — what it shows: a vendor-neutral, tutorial-style walkthrough of cohort borrowing solving idle-GPU utilisation on a shared cluster.

**Deeper dives**
- Kueue release notes / GitHub Releases — https://github.com/kubernetes-sigs/kueue/releases — check the fair-sharing and hierarchical-cohort feature-gate status for your installed version before relying on any specific behaviour described above.
- KEP-4136, Admission Fair Sharing — https://github.com/kubernetes-sigs/kueue/tree/main/keps/4136-admission-fair-sharing — the design doc underlying the interaction discussed in this lesson's "two fairness layers" section; fetched and cited fully in 06.3.
