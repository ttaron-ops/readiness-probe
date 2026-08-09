---
lesson: "05.4"
title: "Attribution — from truthful metrics to per-namespace GPU-hours"
module: "05"
concept: "Attribution — from truthful metrics to per-namespace GPU-hours"
status: not-started
est_time: "3h"
artifacts: []
---
# 05.4 · Attribution — from truthful metrics to per-namespace GPU-hours
> **Concept.** Join dcgm-exporter's UUID-keyed truthful metrics to the pod-resources labels you built in 04, aggregate to namespace, and expose the gap between *who holds* GPUs and *who uses* them — the divergence that is the whole story.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Why this matters

You now have honest metrics on the wire (05.3): `DCGM_FI_PROF_SM_ACTIVE` per GPU,
enriched with `pod`/`namespace`/`container` via the exact pod-resources join you built in
04. The question FinOps actually asks is not per-GPU — it is per-*team*: "team-vision
holds 80 GPUs; are they using them?" Answering that is one aggregation away, and the
answer is almost always uncomfortable: the namespace that *holds* the most GPUs is rarely
the one that *uses* them the most. That divergence — allocated GPU-hours vs SM-active
GPU-hours, sliced by namespace — is the single query that makes the "your dashboard is
lying" deliverable land. It converts "utilization is low" (unactionable) into "team-X is
sitting on 40 idle H100s" (a chargeback conversation).

This lesson is deliberately light: you are *consuming* the 04 join, not rebuilding it.
The new work is aggregation and one honest caveat — the time-slicing attribution hole,
now stated in metric terms.

## What's new vs 04

Module 04 delivered the mechanism: device UUID → pod → namespace, live and correct, the
same join dcgm-exporter does internally. It produced *per-pod* attribution. What's new
here is small and specific:

- **You consume the labels, you don't compute them.** dcgm-exporter has already stamped
  `pod`/`namespace`/`container` onto every DCGM series (`DCGM_EXPORTER_KUBERNETES=true`,
  fed by pod-resources). Your 04 exporter *is* the reference implementation of that
  decoration; here it arrives for free on `DCGM_FI_PROF_SM_ACTIVE` and you PromQL over it.
- **Aggregation to namespace and to GPU-hours** — turning a fraction sampled every 30s
  into an integral over time (`sum_over_time` / `avg_over_time` × device count).
- **The 04 time-slicing hole, restated in metric terms.** In 04 you learned that
  pod-resources returns the *same physical `GPU-xxxx` UUID* for every pod sharing a
  time-sliced card. Here is what that does to a *metric*: DCGM emits one series per
  physical GPU keyed by that one UUID, so the enrichment maps it to *whichever pod it
  saw last* — the SM-active signal for a shared card cannot be split per tenant at all.

## Core notes

### The labels you inherit

dcgm-exporter's raw labels for the attribution join are `pod`, `namespace`, `container`
(plus `UUID`, `gpu`, `device`, `Hostname`, `modelName`). When Prometheus scrapes via
Kubernetes SD, its own target labels can collide with `pod`/`namespace`; with
`honor_labels: true` the exporter's win, otherwise Prometheus prefixes the exporter's as
**`exported_pod` / `exported_namespace`**. Know which your setup produces — it decides
whether you group by `namespace` or `exported_namespace`. The label originates in the
kubelet **pod-resources API** (`/var/lib/kubelet/pod-resources/kubelet.sock`), read by the
exporter and matched to the DCGM series by GPU **UUID** — your 04 exporter is that path.

### Two numerators: allocated vs used

There are two different "GPU count per namespace," and confusing them is the mistake the
deliverable exposes:

- **Allocated** — how many GPUs a namespace *holds*. Count distinct devices that have an
  `(exported_)namespace` label, i.e. that pod-resources says are assigned. This is the
  denominator finance is billed on; it is what `nvidia.com/gpu` requests sum to.
- **Used** — how much those GPUs were *doing work*, integrated over time.
  `SM_ACTIVE` is a 0..1 fraction; a GPU held for an hour at `SM_ACTIVE=0.05` contributes
  1 allocated GPU-hour but 0.05 SM-active GPU-hours.

### PromQL — per-namespace SM_ACTIVE

Mean truthful utilization per namespace, right now:

```promql
avg by (namespace) (DCGM_FI_PROF_SM_ACTIVE)
```

(Use `exported_namespace` if your scrape prefixes.) That is the honest analogue of the
GPU_UTIL dashboard number, per team. Compare it against the misleading one to show the
gap:

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

SM-active GPU-hours per namespace over the last 24h (integrate the fraction, per device,
then sum). `avg_over_time` gives the mean fraction per series across the range; multiply
by hours (24) to get GPU-hours of actual SM activity:

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

Sort the fleet by *allocated* and separately by *SM-active GPU-hours*. The top of the two
lists is not the same namespace. That inversion is the story: the biggest GPU *holder* is
not the biggest GPU *user*.

### The time-slicing hole, in metric terms

Everything above assumes one tenant per physical GPU (whole-GPU or MIG). Under
**time-slicing** (04.7) it breaks, and you must say so in the deliverable:

- pod-resources returns the **same `GPU-xxxx` UUID** to every pod sharing the card. Your
  04 join built `map[UUID] → owner`, so the last writer wins — the map holds one owner
  for a device many pods share.
- DCGM emits **one `DCGM_FI_PROF_SM_ACTIVE` series per physical GPU**, keyed by that one
  UUID. There is exactly one SM-active number for the whole card.
- Therefore enrichment stamps that single series with *one* `(namespace, pod)` — whichever
  pod-resources reported last — and **100% of the card's SM activity is attributed to one
  tenant while its co-tenants show zero.** The metric has no per-tenant dimension to split
  on; the physical GPU is the finest granularity DCGM can see. No PromQL fixes this,
  because the information was never in the series.

**MIG is the clean case:** the hardware partitions the device, pod-resources returns a
*distinct* `MIG-xxxx` UUID per instance, DCGM emits per-MIG-instance series keyed by that
same UUID, and the join is 1:1. MIG preserves per-tenant attribution; time-slicing and MPS
destroy it. In the divergence query, flag time-sliced UUIDs (those mapped to >1 pod by
your 04 exporter) as an *unattributable* bucket rather than silently crediting one tenant.

## Worked example

Fleet slice, 24h window, whole-GPU + MIG (no time-slicing):

| namespace | allocated GPUs | allocated GPU-h | SM-active GPU-h | efficiency |
|---|---|---|---|---|
| team-vision | 80 | 1920 | 96 | 5% |
| team-nlp | 24 | 576 | 430 | 75% |
| team-research | 40 | 960 | 120 | 12% |

Rank by **allocated**: vision (80) > research (40) > nlp (24). Rank by **SM-active
GPU-h**: nlp (430) > research (120) > vision (96). The lists invert at the top: the team
holding 80 GPUs converts them to 96 GPU-hours of real work — a 24-GPU team out-*uses*
them. `avg by (namespace)(DCGM_FI_DEV_GPU_UTIL)` would have shown team-vision at ~90%
because their pods keep one SM ticking; `SM_ACTIVE` shows 5%. That delta, per namespace,
is the deliverable's headline: **the dashboard said 90%; the fleet is 5% busy, and the
biggest holder is the emptiest.**

## Practice — feeds the deliverable

1. Confirm `DCGM_FI_PROF_SM_ACTIVE` carries a namespace label (`namespace` or
   `exported_namespace`) sourced from your 04 pod-resources path. If not, revisit 05.3
   enrichment.
2. Run `avg by (namespace)(DCGM_FI_PROF_SM_ACTIVE)` and the `GPU_UTIL` analogue; capture
   the per-namespace gap.
3. Build the divergence query: namespaces ranked by allocated GPU-hours vs by SM-active
   GPU-hours over 24h, plus the efficiency ratio. Confirm the two rankings disagree.
4. Enumerate any UUID mapped to >1 pod (time-sliced/MPS) via your 04 exporter; route those
   to an explicit *unattributable* bucket in the query and note the GPU-hours parked there.

**Acceptance:** a saved PromQL query (or recording rule) that outputs, per namespace over
a window, **allocated GPU-hours, SM-active GPU-hours, and their ratio**, ranked so the
holder-vs-user divergence is visible — plus a one-line caveat naming the time-sliced
GPU-hours that cannot be attributed. This is the deliverable's core query.

## Self-check

**(a) Why can't you attribute per-pod under time-slicing, in metric terms?**
**Answer:** Because DCGM emits exactly one series per *physical* GPU, keyed by that GPU's
single `GPU-xxxx` UUID, and pod-resources returns that same UUID to every pod sharing the
card. There is one `SM_ACTIVE` value and one UUID for many pods, so enrichment can stamp
only one `(namespace, pod)` on it — the physical GPU is the finest granularity in the
series. The per-tenant split simply isn't present in the data; no query recovers it.

**(b) Which dcgm-exporter label joins a DCGM series to a pod, and where does it come from?**
**Answer:** The `pod` label (with `namespace`/`container`; possibly `exported_`-prefixed
after scrape). dcgm-exporter reads the kubelet **pod-resources API**
(`/var/lib/kubelet/pod-resources/kubelet.sock`), which maps each container's assigned
device UUIDs to its pod/namespace, and matches that to the DCGM series by GPU **UUID** —
the exact join your module-04 exporter implements.

**(c) MIG vs time-slicing — which preserves clean per-tenant attribution?**
**Answer:** MIG. The hardware partitions the GPU, so pod-resources returns a distinct
`MIG-xxxx` UUID per instance and DCGM emits per-instance series keyed by that UUID — a 1:1
UUID→pod join. Time-slicing (and MPS) share one physical `GPU-xxxx` UUID across all
tenants, collapsing them to one series and destroying per-tenant attribution.

## Resources

1. **Your module-04 pod-resources exporter** — the UUID→pod→namespace join you built;
   this lesson consumes its labels directly:
   `../../04-gpu-on-kubernetes/lessons/03-device-plugin-recap-pod-resources.md` and the
   capstone `../../04-gpu-on-kubernetes/lessons/10-capstone-per-pod-attribution.md`.
2. **dcgm-exporter — Kubernetes / pod-resources integration** — how the exporter reads the
   pod-resources socket and applies `pod`/`namespace` labels:
   https://github.com/NVIDIA/dcgm-exporter
