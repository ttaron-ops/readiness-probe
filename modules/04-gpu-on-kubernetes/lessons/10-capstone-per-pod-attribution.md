---
lesson: "04.10"
title: "Capstone — per-pod GPU attribution"
module: "04"
concept: "Capstone — per-pod GPU attribution"
status: not-started
est_time: "12h"
artifacts: []
---

# 04.10 · Capstone — per-pod GPU attribution

> **Concept.** Turn device UUID → pod → namespace → utilization → dollars, for MIG and time-sliced devices alike.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Why this matters

Finance asks "what did team-a's training run cost last night?" and every off-the-shelf answer is wrong. Cloud
billing stops at the *node*: an 8×H100 box is one line item. `nvidia.com/gpu` counts *allocations*, not *use* —
a pod holding an idle GPU looks identical to one running it flat out. The only honest per-pod GPU number is
**device UUID → pod → utilization**, and you have to build that join yourself. This capstone is that join, wired
into the `gpu-cost-operator` you started in Kubernetes-controllers (02). It is the differentiator on your
résumé: everyone can schedule a GPU; you can *bill* one. This is the reference pattern `dcgm-exporter` uses, and
you are re-implementing its spine so you understand every edge.

## What's new here

Every prior lesson produced an input; this lesson **assembles** them. 04.3 gave you a Go pod-resources client
(UUID→pod). 04.4/04.5/04.6 gave you MIG, time-slicing, MPS — the sharing modes that make attribution hard.
04.9 gave you DRA claims (UUID from the API). What's new is the **live map and the join**: a controller loop
that keeps `deviceUUID → {namespace, pod, container}` fresh as pods churn, joins it to DCGM utilization by
UUID, and emits a per-pod cost metric — correctly handling the case where the map is many-pods-to-one-device.

The hard problem is the **cardinality of UUID→pod**, which differs by sharing mode:

| Sharing mode | Device UUID reported by pod-resources | UUID→pod | Attribution |
|---|---|---|---|
| Whole GPU | `GPU-xxxx` | 1:1 | trivial |
| **MIG** | `MIG-xxxx` (distinct per instance) | 1:1 | clean — DCGM reports per-MIG metrics keyed by the same MIG UUID |
| **Time-slicing / MPS** | `GPU-xxxx` (same physical UUID for *all* sharers) | **many:1** | **ambiguous — UUID join alone cannot split it** |

MIG is the easy case *because the hardware partitions the UUID*. Time-slicing is the trap: the pod-resources
API returns the *same* `GPU-xxxx` for every pod on that card, so a naive UUID join attributes 100% of the
device to whichever pod you happened to map last. You need a fallback signal.

## Core notes

**The map: pod-resources, not the pod spec.** The kubelet pod-resources gRPC API
(`/var/lib/kubelet/pod-resources/kubelet.sock`, proto at `kubernetes/kubelet/pkg/apis/podresources`) is the
*only* source that reports the **actual device IDs assigned to each container** — the pod spec just says
`nvidia.com/gpu: 1`. `ListPodResources` returns, per container, `Devices[]` with
`ResourceName: nvidia.com/gpu` and `DeviceIds: ["GPU-xxxx"]` (or `MIG-xxxx`). Build the inverted index:

```go
// map[deviceUUID] -> {ns, pod, container}
func buildMap(resp *podresourcesv1.ListPodResourcesResponse) map[string]Owner {
    m := map[string]Owner{}
    for _, p := range resp.PodResources {
        for _, c := range p.Containers {
            for _, d := range c.Devices {
                if !strings.HasPrefix(d.ResourceName, "nvidia.com/") {
                    continue
                }
                for _, uuid := range d.DeviceIds { // GPU-... or MIG-...
                    m[uuid] = Owner{p.Namespace, p.Name, c.Name}
                }
            }
        }
    }
    return m
}
```

**Fresh vs stale — watch, not just poll.** Pods churn faster than a metrics scrape. Two honest options:

- **Poll pod-resources** on a short interval (10–15s) and rebuild the whole map each tick. The pod-resources API
  has **no watch** — `List` is the only call, so polling is native. Simple, and the map is at most one interval
  stale. This is what `dcgm-exporter` does.
- **Watch pods** via the Kubernetes API (informer) to know *when* something changed and trigger an
  out-of-cycle pod-resources `List` — the informer tells you *that* it changed; pod-resources tells you *which
  device*. The informer never has the device IDs, so you can't skip the `List`.

Practical answer: **poll pod-resources on a ticker; optionally use a pod informer to poll immediately on
add/delete** so a short-lived pod doesn't slip between ticks. Cache the last map; serve metrics from cache so a
transient socket error doesn't blank your dashboard.

**The join: pod-resources ⋈ DCGM by UUID.** `dcgm-exporter` runs a DCGM collector that emits, per device,
metrics labelled with the GPU/MIG UUID, then *decorates* each series with `exported_namespace`, `exported_pod`,
`exported_container` looked up from the pod-resources map (enabled via `DCGM_EXPORTER_KUBERNETES=true` and
`--kubernetes-gpu-id-type=uid`). You mirror that: for each DCGM sample, look up its UUID in your map and attach
the owner labels. The utilization field you want:

- `DCGM_FI_DEV_GPU_UTIL` — coarse % busy (SM had ≥1 kernel). Fine for whole-GPU/MIG.
- `DCGM_FI_PROF_GR_ENGINE_ACTIVE` — fraction of time the graphics/compute engine was active (0–1). Truer
  utilization; prefer it when profiling is available.

**MIG (clean 1:1).** DCGM reports per-MIG-instance metrics keyed by the `MIG-xxxx` UUID, and pod-resources
reports that *same* `MIG-xxxx` for the owning container. Join by UUID → done, one pod per series. No fallback
needed. This is why MIG is the "easy" attribution mode despite being the "hard" hardware mode.

**Time-slicing / MPS (many:1 → per-PID fallback).** All sharers show the same `GPU-xxxx`; DCGM's device-level
util is the *sum* across sharers and cannot be split by UUID. Fall back to **per-process accounting** and map
PID → container:

- Signal: NVML `nvmlDeviceGetProcessUtilization` (per-PID SM/mem %), or `nvidia-smi pmon` / DCGM per-PID
  accounting. This gives `PID → utilization` on the shared device.
- PID → container: resolve each PID's cgroup (`/proc/<pid>/cgroup`) to the pod/container ID, or match against
  the runtime. Then split the device's utilization across pods **proportional to per-PID SM activity**.
- If you can't get per-PID (older stack, MPS hiding PIDs), degrade gracefully: **attribute by fair share**
  (device util ÷ number of sharing pods) and **label the series `attribution=shared-estimate`** so downstream
  never mistakes an estimate for a measurement.

**Cost: from utilization to dollars.** Join three inputs:

```
per_pod_gpu_cost_per_hour =
    node_hourly_rate                      # $ for the whole node (billing/rate-card)
    ÷ gpus_per_node                       # $ per physical GPU-hour
    × pod_share_of_device                 # 1.0 whole GPU; 1/7 for a 1g.10gb MIG slice; per-PID fraction if shared
    × (utilization_fraction or 1.0)       # ×util for *efficiency* cost; ×1 for *allocated* cost
```

Emit **two** metrics, not one: **allocated cost** (what the pod reserved — bills even at 0% use, this is what
finance charges back) and **efficiency/utilised cost** (allocated × utilization — the waste signal). The gap
between them is your idle-GPU story. For a MIG slice, `pod_share_of_device` is the slice fraction (a `1g.10gb`
on an 80GB H100 ≈ 1/7 of compute); for a shared full GPU it's the per-PID fraction from the fallback.

```go
// Prometheus gauges, labelled so PromQL can group by team/namespace:
allocated := ratePerGPUHour * podShare                 // bills regardless of util
utilised  := allocated * utilFraction                  // efficiency-adjusted
gpuCostAllocated.With(labels).Set(allocated)
gpuCostUtilised.With(labels).Set(utilised)
// labels: namespace, pod, container, gpu (UUID), device_type=mig|shared|whole, attribution=exact|shared-estimate
```

**DRA makes the map easier — but you still need pod-resources for util.** On a 04.9 DRA cluster the
`ResourceClaim.status.allocation` already names the device, so you *could* build UUID→pod by watching claims
instead of polling pod-resources. Do it — a claim informer is a clean, event-driven source of the owner map.
But DCGM keys utilization by the *runtime* GPU/MIG UUID, and pod-resources is still the ground truth that a
container actually holds that UUID at scrape time, so keep the pod-resources poll as the join key even when DRA
supplies the ownership. DRA improves *where the map comes from*; it does not remove the utilization join.

**Assemble the GPU Operator failure-mode log.** As part of the deliverable, roll up the failure modes you hit
in 02/05/06 into `practice/per-pod-attribution/failure-modes.md`: driver-container/toolkit mismatch (02),
time-slicing that oversubscribes and tanks per-pod throughput (05), MPS `defaultActiveThreadPercentage`
starving a co-tenant (06), and the attribution-specific ones — pod-resources socket unreadable
(hostPath/permission), UUID present in DCGM but missing from the map (pod deleted mid-scrape → serve-from-cache),
and a time-sliced device with no per-PID data (fall back + flag).

## Worked example

Two pods, one time-sliced H100 (`GPU-abc`), node rate $32/hr, 8 GPUs/node → $4/GPU-hr.

```
pod-resources ListPodResources:
  team-a/trainer  ctr -> DeviceIds: ["GPU-abc"]
  team-b/notebook ctr -> DeviceIds: ["GPU-abc"]     # SAME UUID — many:1
```

Naive UUID join → both pods get `GPU-abc`'s util; you'd double-bill or mislabel. Fallback:

```
nvmlDeviceGetProcessUtilization on GPU-abc:
  PID 4210 (cgroup -> team-a/trainer)   smUtil 82%
  PID 4990 (cgroup -> team-b/notebook)  smUtil  6%
device DCGM_FI_PROF_GR_ENGINE_ACTIVE = 0.88   # matches ~sum

shares:  trainer 82/(82+6)=0.93   notebook 0.07
allocated cost/hr:  each reserved a *share* of one GPU:
  trainer  $4 × 0.93 = $3.72        notebook $4 × 0.07 = $0.28
utilised cost/hr:   allocated × own util
  trainer  $3.72 × 0.88 = $3.27     notebook $0.28 × 0.88 = $0.25
labels: attribution=shared-estimate (per-PID), device_type=shared
```

Contrast MIG: two `1g.10gb` slices on the same card show `MIG-x` and `MIG-y` in *both* pod-resources and DCGM →
join by UUID, `attribution=exact`, `pod_share=1/7`, no per-PID needed. Same operator, two code paths chosen by
whether the UUID is 1:1 or many:1.

```promql
# team monthly chargeback (allocated) vs waste (allocated - utilised):
sum by (namespace) (gpu_cost_allocated_per_hour) * 730
sum by (namespace) (gpu_cost_allocated_per_hour - gpu_cost_utilised_per_hour) * 730
```

## Practice

**THE MODULE DELIVERABLE** — [../practice/per-pod-attribution/README.md](../practice/per-pod-attribution/README.md).
Extend the `gpu-cost-operator`/exporter:

1. **Mapper:** poll the pod-resources API (04.3 client) on a ticker; build `UUID → {ns, pod, container}`; serve
   from cache; optionally trigger an immediate poll from a pod informer on add/delete.
2. **Join:** read DCGM (`DCGM_FI_PROF_GR_ENGINE_ACTIVE`, fallback `DCGM_FI_DEV_GPU_UTIL`) — or NVML/`nvidia-smi`
   for a minimal build — and attach `exported_namespace/pod/container` by UUID.
3. **Cost:** emit `gpu_cost_allocated_per_hour` and `gpu_cost_utilised_per_hour` = share × rate (× util),
   `rate = node_hourly_rate ÷ gpus_per_node`.
4. **MIG case:** distinct UUID → 1:1, `attribution=exact`, share = slice fraction.
5. **Time-sliced case:** shared UUID → per-PID (NVML `nvmlDeviceGetProcessUtilization`, PID→cgroup→pod) to split;
   if unavailable, fair-share and label `attribution=shared-estimate`.
6. **Failure-mode log:** `failure-modes.md` rolling up 02/05/06 + the attribution edges above.

**Acceptance = the module checkpoint** ([../checkpoint.md](../checkpoint.md)). Concretely: on a live GPU node,
`curl` the exporter and show per-pod series carrying `namespace`, `pod`, `gpu` (UUID), `device_type`,
`attribution`, and both cost gauges; demonstrate a **MIG** pod (`attribution=exact`) and a **time-sliced** pair
(`attribution=shared-estimate`, shares summing to the device util within tolerance); commit the code, a scrape
sample, and `failure-modes.md`.

## Self-check

**(a) A time-sliced device has a shared UUID (many pods:1 device) — how do you attribute per-pod, and what's the
fallback signal?**
**Answer:** The device UUID join collapses — pod-resources returns the same `GPU-xxxx` for every sharer, so a
UUID join alone can't split the device. Fall back to **per-process accounting**: NVML
`nvmlDeviceGetProcessUtilization` (or `nvidia-smi pmon` / DCGM per-PID) gives `PID → SM utilization` on that
device; resolve each PID's cgroup to its pod/container; split the device's utilization proportional to per-PID
SM activity. If per-PID data is unavailable, degrade to fair-share (device util ÷ sharer count) and label the
series `attribution=shared-estimate` so it's never mistaken for a measurement.

**(b) How do you keep the UUID→pod map fresh as pods churn — watch vs poll?**
**Answer:** The pod-resources API has **no watch** — `List` is the only call — so you **poll** it on a short
ticker (10–15s) and rebuild the map, serving the last good map from cache on transient socket errors. To close
the gap for short-lived pods, add a Kubernetes **pod informer** (watch) that triggers an *immediate* pod-resources
`List` on add/delete — the informer signals *that* something changed but never carries device IDs, so it
augments the poll, it doesn't replace it.

**(c) How do you turn a device UUID + utilization into a per-pod dollar figure — what inputs do you join?**
**Answer:** Join four things: the **UUID→pod** map (pod-resources), the **utilization** for that UUID (DCGM
`DCGM_FI_PROF_GR_ENGINE_ACTIVE`, per-PID for shared devices), the **rate** (`node_hourly_rate ÷ gpus_per_node`,
a $/GPU-hour), and the **pod's share of the device** (1.0 whole GPU; slice fraction for MIG; per-PID fraction
for time-slicing). Emit two numbers: **allocated** = rate × share (bills even when idle — the chargeback figure)
and **utilised** = allocated × utilization (the efficiency figure). Their gap is the idle-waste signal.

## Resources

1. **dcgm-exporter** — https://github.com/NVIDIA/dcgm-exporter — the reference implementation of the
   pod-resources→per-pod-metric join (`DCGM_EXPORTER_KUBERNETES`, `--kubernetes-gpu-id-type`, the
   `exported_pod/namespace/container` labels). Read how it maps UUID and MIG IDs.
2. **kubelet pod-resources proto** —
   https://github.com/kubernetes/kubelet/tree/master/pkg/apis/podresources — the `ListPodResources`
   message shapes your mapper consumes.
3. **DCGM field reference** — the DCGM API/field-ID guide for `DCGM_FI_DEV_GPU_UTIL` vs
   `DCGM_FI_PROF_GR_ENGINE_ACTIVE` and the per-PID accounting fields used in the shared-device fallback.
