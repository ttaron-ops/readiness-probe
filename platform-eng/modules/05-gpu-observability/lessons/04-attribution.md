---
lesson: "05.4"
title: "Attribution — from truthful metrics to per-namespace GPU-hours"
module: "05"
concept: "Attribution — from truthful metrics to per-namespace GPU-hours"
status: not-started
est_time: "5h"
prev: "03-dcgm-exporter-at-scale.md"
next: "05-health-and-xid.md"
artifacts: []
sources: 8
---
# 05.4 · Attribution — from truthful metrics to per-namespace GPU-hours

> **Concept.** Join dcgm-exporter's UUID-keyed truthful metrics to the pod-resources labels you built in 04, aggregate to namespace, and expose the gap between *who holds* GPUs and *who uses* them — the divergence that is the whole story.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Where this fits

05.3 got `DCGM_FI_PROF_SM_ACTIVE` actually landing in Prometheus — enabled in the counter CSV, labelled with `pod`/`namespace` via controlled enrichment, cardinality kept sane. That was necessary but not sufficient: a per-GPU truthful series still answers only "is *this device* busy." FinOps, your manager, and the interview panel don't ask about a device — they ask about a *team*. This lesson is the last hop: aggregate the honest per-GPU signal to per-namespace GPU-hours, and expose the number that actually drives a chargeback conversation — the gap between GPUs a team *holds* and GPUs a team *uses*. It's the direct input to the capstone's headline panel.

## Why this matters

The stakes here are concrete and dollar-shaped, not academic. Time-slicing is sold on real, large numbers — vendor material puts the savings at up to 90% ("10 jobs on 1 GPU vs. 10 GPUs") — which is exactly why you will be pressured to adopt it, and exactly why you need to know its attribution blind spot cold before you do. A platform vendor whose entire product is GPU cost optimization (ScaleOps) independently states the same hole this lesson teaches: under time-slicing, multiple pods can share one physical GPU while many DCGM metrics stay device-level, so duplicated labels should not be read as exact per-pod usage. That's not a course-invented caveat — it's an industry admission from a company that gets paid to solve this exact problem. Getting the "holder vs user" query right, and getting the time-slicing caveat right, is the difference between a chargeback number that survives a VP's scrutiny and one that gets you overruled the first time someone checks your math.

## What's new here (calibration)

Module 04 delivered the mechanism — device UUID → pod → namespace, live and correct, the same join dcgm-exporter does internally. What's new here is small, specific, and does not re-derive that join:

- **You consume the labels, you don't compute them.** dcgm-exporter has already stamped `pod`/`namespace`/`container` onto every DCGM series (fed by pod-resources, `DCGM_EXPORTER_KUBERNETES=true`); here it arrives for free on `DCGM_FI_PROF_SM_ACTIVE` and you PromQL over it.
- **Aggregation to namespace and to GPU-hours** — turning a fraction sampled every 30s into an integral over time (`sum_over_time`/`avg_over_time` × device count).
- **The 04 time-slicing hole, restated in metric terms.** 04 taught that pod-resources returns the *same physical `GPU-xxxx` UUID* for every pod sharing a time-sliced card. Here is what that does to a *metric*: DCGM emits one series per physical GPU keyed by that one UUID, so enrichment maps it to *whichever pod it saw last* — the SM-active signal for a shared card cannot be split per tenant at all, in metric terms, full stop.

## Core concepts

### The labels you inherit

dcgm-exporter's raw labels for the attribution join are `pod`, `namespace`, `container` (plus `UUID`, `gpu`, `device`, `Hostname`, `modelName`). When Prometheus scrapes via Kubernetes SD, its own target labels can collide with `pod`/`namespace`; with `honor_labels: true` the exporter's win, otherwise Prometheus prefixes the exporter's as **`exported_pod`/`exported_namespace`**. Know which your setup produces — it decides whether you group by `namespace` or `exported_namespace`. The label originates in the kubelet **pod-resources API** (`/var/lib/kubelet/pod-resources/kubelet.sock`), read by the exporter and matched to the DCGM series by GPU **UUID** — your 04 exporter is that path.

### Two numerators: allocated vs used

There are two different "GPU count per namespace," and confusing them is the mistake the deliverable exposes:

- **Allocated** — how many GPUs a namespace *holds*. Count distinct devices with an `(exported_)namespace` label, i.e. that pod-resources says are assigned. This is the denominator finance is billed on; it is what `nvidia.com/gpu` requests sum to.
- **Used** — how much those GPUs were *doing work*, integrated over time. `SM_ACTIVE` is a 0..1 fraction; a GPU held for an hour at `SM_ACTIVE=0.05` contributes 1 allocated GPU-hour but only 0.05 SM-active GPU-hours.

### PromQL — per-namespace SM_ACTIVE

Mean truthful utilization per namespace, right now:

```promql
avg by (namespace) (DCGM_FI_PROF_SM_ACTIVE)
```

(Use `exported_namespace` if your scrape prefixes.) That's the honest analogue of the `GPU_UTIL` dashboard number, per team. Compare it against the misleading one to show the gap:

```promql
avg by (namespace) (DCGM_FI_DEV_GPU_UTIL) / 100      # the lie, per team
avg by (namespace) (DCGM_FI_PROF_SM_ACTIVE)          # the truth, per team
```

### PromQL — the divergence query (deliverable core)

Rank namespaces two ways over a window and show they disagree.

Allocated GPUs per namespace (count of held devices):
```promql
count by (namespace) (
  count by (namespace, gpu, UUID) (DCGM_FI_PROF_SM_ACTIVE)
)
```

SM-active GPU-hours per namespace over the last 24h (integrate the fraction per device, then sum). `avg_over_time` gives the mean fraction per series across the range; multiply by hours (24) to get GPU-hours of actual SM activity:

```promql
sum by (namespace) (
  avg_over_time(DCGM_FI_PROF_SM_ACTIVE[24h])
) * 24
```

Allocated GPU-hours over the same window (each held device = 1 GPU-hour/hour):
```promql
count by (namespace) (
  count by (namespace, UUID) (DCGM_FI_PROF_SM_ACTIVE)
) * 24
```

Efficiency (used ÷ held), the one number that starts a chargeback conversation:
```promql
(
  sum by (namespace) (avg_over_time(DCGM_FI_PROF_SM_ACTIVE[24h]))
)
/
(
  count by (namespace) (count by (namespace, UUID) (DCGM_FI_PROF_SM_ACTIVE))
)
```

Sort the fleet by *allocated* and separately by *SM-active GPU-hours*. The top of the two lists is not the same namespace. That inversion is the story: the biggest GPU *holder* is not the biggest GPU *user*.

### The time-slicing hole, in metric terms

Everything above assumes one tenant per physical GPU (whole-GPU or MIG). Under **time-slicing** (Module 04, lesson 07) it breaks, and you must say so in the deliverable:

- pod-resources returns the **same `GPU-xxxx` UUID** to every pod sharing the card. Your 04 join built `map[UUID] → owner`, so the last writer wins — the map holds one owner for a device many pods share.
- DCGM emits **one `DCGM_FI_PROF_SM_ACTIVE` series per physical GPU**, keyed by that one UUID. There is exactly one SM-active number for the whole card.
- Therefore enrichment stamps that single series with *one* `(namespace, pod)` — whichever pod-resources reported last — and **100% of the card's SM activity is attributed to one tenant while its co-tenants show zero.** The metric has no per-tenant dimension to split on; the physical GPU is the finest granularity DCGM can see. No PromQL fixes this, because the information was never in the series.

**MIG is the clean case:** the hardware partitions the device, pod-resources returns a *distinct* `MIG-xxxx` UUID per instance, DCGM emits per-MIG-instance series keyed by that same UUID, and the join is 1:1. MIG preserves per-tenant attribution; time-slicing and MPS destroy it — this is a hardware property (whether the device is physically partitioned), not a config oversight you can patch. In the divergence query, flag time-sliced UUIDs (those mapped to >1 pod by your 04 exporter) as an explicit *unattributable* bucket rather than silently crediting one tenant.

## Perspectives

**Developer perspective.** A data scientist requesting a shared or time-sliced GPU has no visibility that their utilization is being silently merged with — or hidden behind — a co-tenant's. From inside their pod, everything looks normal; the attribution hole is invisible at the layer where the work actually happens.

**Operator/FinOps perspective.** The "who holds vs who uses" divergence is the single query that turns a vague "utilization is low" into an actionable, named chargeback conversation. That's exactly why this lesson is kept light — it's an aggregation over 05.3's honest metrics and 04's join, not a new mechanism, but it's the aggregation that actually gets read in a budget meeting.

**Hardware/isolation perspective.** MIG's hardware-level partitioning (a distinct `MIG-xxxx` UUID per instance) is what makes attribution *possible* at all. Time-slicing and MPS share one physical UUID and destroy attribution at the metric level — a property of the silicon and driver, not something a smarter exporter config could fix.

**Economics perspective.** Vendors selling GPU-sharing products openly acknowledge the attribution gap as a known limitation of their own offering (see ScaleOps below) — which validates that "time-sliced GPU-hours are unattributable, full stop" is industry-consensus, not a course-specific opinion invented for this module.

## Real-world use cases

- **NVIDIA GPU Operator — "Time-Slicing GPUs in Kubernetes"** — https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html (canonical URL; blocked by this sandbox's egress proxy on fetch, cited per SPEC's fallback rule). **What it shows:** NVIDIA's own documentation frames time-slicing vs MIG as a tradeoff between isolation and share-count — the same tradeoff this lesson frames as an attribution tradeoff, from the vendor that ships both mechanisms.
- **ScaleOps — "GPU Sharing in Kubernetes: MIG vs MPS vs Time-Slicing"** — https://scaleops.com/blog/kubernetes-gpu-sharing/ (canonical URL; blocked by this sandbox's egress proxy on fetch, cited per SPEC's fallback rule). Search-indexed content confirms a directly relevant admission: with time-slicing, multiple pods may share one physical GPU while many DCGM metrics remain device-level, so duplicated labels should not be interpreted as exact per-pod usage. **What it shows:** a company whose product *is* GPU cost optimization independently states the exact attribution hole this lesson teaches — strong corroboration from a vendor with every incentive to undersell the limitations of GPU sharing, not oversell them.
- **Red Hat — "Sharing is caring: how to make the most of your GPUs (part 1 — time-slicing)"** — https://www.redhat.com/en/blog/sharing-caring-how-make-most-your-gpus-part-1-time-slicing (canonical URL; blocked by this sandbox's egress proxy on fetch, cited per SPEC's fallback rule). **What it shows:** a major platform vendor's own explainer on the time-slicing/MIG tradeoff, aimed at the same platform-engineer audience as this module.

## Worked example

Fleet slice, 24h window, whole-GPU + MIG only (no time-slicing):

| namespace | allocated GPUs | allocated GPU-h | SM-active GPU-h | efficiency |
|---|---|---|---|---|
| team-vision | 80 | 1920 | 96 | 5% |
| team-nlp | 24 | 576 | 430 | 75% |
| team-research | 40 | 960 | 120 | 12% |

Rank by **allocated**: vision (80) > research (40) > nlp (24). Rank by **SM-active GPU-h**: nlp (430) > research (120) > vision (96). The lists invert at the top: the team holding 80 GPUs converts them to 96 GPU-hours of real work — a 24-GPU team out-*uses* them. `avg by (namespace)(DCGM_FI_DEV_GPU_UTIL)` would have shown team-vision at ~90% because their pods keep one SM ticking; `SM_ACTIVE` shows 5%. That delta, per namespace, is the deliverable's headline: **the dashboard said 90%; the fleet is 5% busy, and the biggest holder is the emptiest.**

Now add a fourth row, this time under time-slicing, to make the attribution hole concrete rather than abstract:

| namespace | logical slots | physical GPUs (time-sliced) | SM-active GPU-h (attributable) | unattributable GPU-h |
|---|---|---|---|---|
| team-shared | 8 | 2 (H100, time-sliced 4-way each) | — | 48 |

`team-shared` runs 8 pods across 2 physical H100s, each time-sliced 4 ways. DCGM emits exactly 2 `SM_ACTIVE` series — one per physical GPU — and the pod-resources join can only report *one* pod per physical UUID at any instant. Over the 24h window that's `2 GPUs × 24h = 48` GPU-hours of real SM activity happening on that hardware, but the query pack has no honest way to split those 48 GPU-hours across the 8 tenant pods that actually generated them. The correct move is **not** to guess and credit one tenant — it's to report those 48 GPU-hours in an explicit `unattributable_gpu_hours` bucket, separate from the per-namespace breakdown, so the dashboard stays honest about what it doesn't know rather than quietly corrupting a chargeback number.

## Practice

Feeds the module deliverable, ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md):

1. Confirm `DCGM_FI_PROF_SM_ACTIVE` carries a namespace label (`namespace` or `exported_namespace`) sourced from your 04 pod-resources path. If not, revisit 05.3's enrichment config.
2. Run `avg by (namespace)(DCGM_FI_PROF_SM_ACTIVE)` and the `GPU_UTIL` analogue; capture the per-namespace gap.
3. Build the divergence query: namespaces ranked by allocated GPU-hours vs by SM-active GPU-hours over 24h, plus the efficiency ratio. Confirm the two rankings disagree.
4. Enumerate any UUID mapped to >1 pod (time-sliced/MPS) via your 04 exporter; route those to an explicit `unattributable_gpu_hours` bucket in the query and note the GPU-hours parked there — don't silently drop or credit them.

**Acceptance:** a saved PromQL query (or recording rule) that outputs, per namespace over a window, **allocated GPU-hours, SM-active GPU-hours, and their ratio**, ranked so the holder-vs-user divergence is visible — plus an explicit `unattributable_gpu_hours` figure for time-sliced GPUs, not just a footnote. This is the deliverable's core query.

## Common pitfalls

1. **Assuming a PromQL trick (join, `label_replace`, etc.) can recover per-tenant attribution under time-slicing.** The information was never in the series — no query fixes missing data, no matter how clever the join.
2. **Treating MIG and time-slicing as interchangeable "GPU sharing" options when comparing cost savings.** MIG has a hard partition-count ceiling (fewer, larger, isolated slices); time-slicing offers a much higher share count but only one of the two preserves billing-grade attribution.
3. **Silently crediting a time-sliced GPU's SM-active work to whichever pod's label happened to land last, without flagging it.** This quietly corrupts the chargeback number rather than making it honestly incomplete — route it to an explicit unattributable bucket instead (see the worked example).
4. **Forgetting to caveat the divergence query's unattributable bucket when presenting results to stakeholders.** An unqualified per-namespace ranking under time-slicing can silently misattribute waste to the wrong team, undermining trust in the whole dashboard the first time someone checks a time-sliced node.

## Self-check

- Why can't you attribute per-pod under time-slicing, in metric terms? **Answer:** Because DCGM emits exactly one series per *physical* GPU, keyed by that GPU's single `GPU-xxxx` UUID, and pod-resources returns that same UUID to every pod sharing the card. There is one `SM_ACTIVE` value and one UUID for many pods, so enrichment can stamp only one `(namespace, pod)` on it — the physical GPU is the finest granularity in the series. The per-tenant split simply isn't present in the data; no query recovers it.
- Which dcgm-exporter label joins a DCGM series to a pod, and where does it come from? **Answer:** The `pod` label (with `namespace`/`container`; possibly `exported_`-prefixed after scrape). dcgm-exporter reads the kubelet **pod-resources API** (`/var/lib/kubelet/pod-resources/kubelet.sock`), which maps each container's assigned device UUIDs to its pod/namespace, and matches that to the DCGM series by GPU **UUID** — the exact join your Module-04 exporter implements.
- MIG vs time-slicing — which preserves clean per-tenant attribution, and why? **Answer:** MIG. The hardware partitions the GPU, so pod-resources returns a distinct `MIG-xxxx` UUID per instance and DCGM emits per-instance series keyed by that UUID — a 1:1 UUID→pod join. Time-slicing (and MPS) share one physical `GPU-xxxx` UUID across all tenants, collapsing them to one series and destroying per-tenant attribution. It's a hardware-partitioning property, not a config choice.
- What should the divergence query do with GPU-hours it can't attribute, rather than ignore the issue? **Answer:** Route them to an explicit `unattributable_gpu_hours` bucket, reported alongside the per-namespace breakdown, rather than silently crediting whichever pod's label happened to land last. An honestly incomplete number preserves trust in the dashboard; a silently misattributed one destroys it the first time someone audits a time-sliced node.

## Connections & what's next

This lesson is the payoff of the chain that started in 05.1 (the lie) and ran through 05.2 (why profiling costs what it costs) and 05.3 (getting the honest metric onto the wire without OOMing Prometheus) — here it becomes an actual per-team dollar-shaped number, and it directly reuses Module 04's pod-resources join and time-slicing lesson (04.7) rather than rebuilding either. The `unattributable_gpu_hours` discipline you build here resurfaces verbatim in the capstone (05.8), where an unqualified per-namespace ranking under time-slicing would otherwise misattribute waste to the wrong team. Next: [05.5 — Health & errors / XID](05-health-and-xid.md) shifts from "is this GPU being used" to "is this GPU healthy" — a different question, but one that draws on the same DCGM series and the same fleet-scale operational discipline.

## References & further reading

**Primary sources**
- Your Module-04 pod-resources exporter — [`../../04-gpu-on-kubernetes/lessons/03-device-plugin-recap-pod-resources.md`](../../04-gpu-on-kubernetes/lessons/03-device-plugin-recap-pod-resources.md) — the UUID→pod→namespace join this lesson consumes directly; read to refresh the mechanism before trusting its output.
- Module-04 time-slicing attribution lesson — [`../../04-gpu-on-kubernetes/lessons/07-time-slicing-attribution.md`](../../04-gpu-on-kubernetes/lessons/07-time-slicing-attribution.md) — read for the pod-resources-level explanation of why time-sliced UUIDs collide, which this lesson restates in metric/PromQL terms.
- Module-04 capstone — [`../../04-gpu-on-kubernetes/lessons/10-capstone-per-pod-attribution.md`](../../04-gpu-on-kubernetes/lessons/10-capstone-per-pod-attribution.md) — read for the full worked attribution build this lesson's PromQL sits on top of.
- `NVIDIA/dcgm-exporter` — repository — https://github.com/NVIDIA/dcgm-exporter — read for how the exporter reads the pod-resources socket and applies `pod`/`namespace` labels internally.
- NVIDIA GPU Operator — "Time-Slicing GPUs in Kubernetes" — https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html — read for the authoritative isolation-vs-share-count tradeoff between time-slicing and MIG.

**Real-world engineering blogs**
- ScaleOps — "GPU Sharing in Kubernetes: MIG vs MPS vs Time-Slicing" — https://scaleops.com/blog/kubernetes-gpu-sharing/ — what it shows: a GPU-cost-optimization vendor independently confirms the device-level-metrics-under-time-slicing attribution hole this lesson teaches.
- Red Hat — "Sharing is caring: how to make the most of your GPUs (part 1 — time-slicing)" — https://www.redhat.com/en/blog/sharing-caring-how-make-most-your-gpus-part-1-time-slicing — what it shows: a major platform vendor's own explainer on the same time-slicing/MIG tradeoff, aimed at platform engineers.

**Deeper dives**
- ScaleOps — "GPU Cost Optimization" — https://scaleops.com/blog/gpu-cost-optimization/ — a company whose product is exactly the allocated-vs-used gap this lesson's query surfaces; useful for seeing the commercial framing of the same problem, and a direct precursor to the capstone's dollar math.
