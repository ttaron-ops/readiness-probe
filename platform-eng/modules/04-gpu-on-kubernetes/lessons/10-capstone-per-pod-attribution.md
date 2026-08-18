---
lesson: "04.10"
title: "Capstone — per-pod GPU attribution"
module: "04"
concept: "Assembling per-pod GPU cost attribution: pod-resources join, MIG vs time-sliced, DRA claims"
status: not-started
est_time: "16h"
prev: "09-dra-driver-and-quotas.md"
next: null
artifacts: []
sources: 6
---

# 04.10 · Capstone — per-pod GPU attribution

> **Concept.** Turn device UUID → pod → namespace → utilization → dollars, for MIG and time-sliced devices alike.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Every lesson in this module handed you one piece of a machine you're now assembling. **L1** (GPU Operator components) gave you the dependency chain that gets a GPU node ready in the first place — nothing downstream works if the operator's init-container chain is broken. **L2** (crash-loop diagnosis) gave you the muscle to keep that chain healthy and the discipline of a written failure-mode log. **L3** (device-plugin recap + pod-resources API) gave you the actual Go client this exporter is built around — the only API that reports which physical device ID a container actually holds. **L4** (CDI) explained the mechanism by which that device ID becomes a real `/dev/nvidia*` node inside the container, and why `NVIDIA_VISIBLE_DEVICES=all` is a debugging trap. **L5** (driver lifecycle) is why your exporter has to survive rolling driver upgrades without producing garbage data mid-rollout. **L6** (MIG) gave you the easy case: a distinct UUID per hardware slice, clean 1:1 attribution. **L7** (time-slicing) gave you the hard case and the proof it's real: `dcgm-exporter#642`, a practitioner watching every time-sliced pod on a Blackwell GPU report identical utilization because the UUID join collapses. **L8** (MPS) added a second flavor of the same collapse, plus the operational caveat that MPS is still experimental and can't be combined with MIG. **L9** (DRA) gave you a second, structurally cleaner path to the same ownership map — a claim's `status.allocation` names the device without polling a socket.

This lesson does not introduce new Kubernetes mechanism. It **assembles** everything above into one running exporter, and it is where you stop being someone who understands nine separate GPU-on-Kubernetes subsystems and become someone who can wire them into a single system that answers a business question. That's the module's actual thesis, stated as a deliverable.

## Why this matters

Finance asks "what did team-a's training run cost last night?" and every off-the-shelf answer is wrong. Cloud billing stops at the *node*: an 8×H100 box is one line item, full stop. `nvidia.com/gpu` counts *allocations*, not *use* — a pod holding an idle GPU looks identical, from the scheduler's point of view, to one running it flat out. The only honest per-pod GPU number is **device UUID → pod → utilization → dollars**, and nobody ships this join for you off the shelf in a way that matches your fleet's exact sharing modes and rate card. You have to build it.

This is not a practice exercise adjacent to the deliverable — **this lesson is the deliverable.** Checkpoint item 8, "Ship it," is exactly this: produce the per-pod attribution exporter and the GPU Operator failure-mode log. Checkpoint item 6, "Attribution, live," is you demonstrating this exporter working against a real device UUID on a real pod. If you can build this correctly — including the honest part, where you label an estimate as an estimate instead of quietly presenting a guess as a measurement — you have the single strongest artifact on your résumé for a Senior/Staff GPU-fleet platform role: everyone in the room can schedule a GPU, very few of them can bill one.

## What's new here (calibration)

This module has already covered every piece of Kubernetes mechanism this capstone touches — there is no new object model, no new API, no new sharing mode to learn. What's genuinely new is the **systems-integration work**: keeping a live ownership map fresh under pod churn, joining it correctly to a second data source (DCGM) that has its own labeling quirks, choosing the right fallback when the primary join collapses, and turning all of that into two honestly-labeled dollar figures instead of one falsely-precise one. That's engineering judgment, not new facts — which is exactly what a capstone should test.

## Core concepts

### The cardinality problem, restated precisely

Every prior lesson produced an input; the hard problem this lesson solves is that **the cardinality of UUID→pod differs by sharing mode**, and your exporter has to detect which case it's in and choose the right code path:

| Sharing mode | Device UUID reported by pod-resources | UUID→pod | Attribution |
|---|---|---|---|
| Whole GPU | `GPU-xxxx` | 1:1 | trivial |
| **MIG** (L6) | `MIG-xxxx` (distinct per instance) | 1:1 | clean — DCGM reports per-MIG metrics keyed by the same MIG UUID |
| **Time-slicing / MPS** (L7/L8) | `GPU-xxxx` (same physical UUID for *all* sharers) | **many:1** | **ambiguous — UUID join alone cannot split it** |

MIG is the easy attribution case *because the hardware itself partitions the UUID* — the physical device boundary and the billing boundary coincide. Time-slicing and MPS are the trap: the pod-resources API returns the *same* `GPU-xxxx` for every pod on that card, so a naive UUID join attributes 100% of the device's utilization to whichever pod you happened to map last, or double-counts it across all of them. `dcgm-exporter#642` is not a hypothetical — it's a practitioner hitting exactly this on current-generation (Blackwell) hardware and reporting, verbatim, that *"all telemetry values (Utilization, Power, Temp) are identical for all pods sharing the same GPU."* You need a fallback signal for the many:1 case, and you need to label it as a fallback.

### The map: pod-resources, not the pod spec

The kubelet pod-resources gRPC API (`/var/lib/kubelet/pod-resources/kubelet.sock`, proto at `kubernetes/kubelet/pkg/apis/podresources`) is the *only* source that reports the **actual device IDs assigned to each container** — the pod spec just says `nvidia.com/gpu: 1`. `ListPodResources` returns, per container, `Devices[]` with `ResourceName: nvidia.com/gpu` and `DeviceIds: ["GPU-xxxx"]` (or `MIG-xxxx`). Build the inverted index:

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

### Fresh vs stale — watch, not just poll

Pods churn faster than a metrics scrape. Two honest options:

- **Poll pod-resources** on a short interval (10–15s) and rebuild the whole map each tick. The pod-resources API has **no watch** — `List` is the only call, so polling is native. Simple, and the map is at most one interval stale. This is what `dcgm-exporter` does.
- **Watch pods** via the Kubernetes API (informer) to know *when* something changed and trigger an out-of-cycle pod-resources `List` — the informer tells you *that* it changed; pod-resources tells you *which device*. The informer never has the device IDs, so you can't skip the `List`.

Practical answer: **poll pod-resources on a ticker; optionally use a pod informer to poll immediately on add/delete** so a short-lived pod doesn't slip between ticks. Cache the last map; serve metrics from cache so a transient socket error doesn't blank your dashboard.

### The join: pod-resources ⋈ DCGM by UUID

`dcgm-exporter` runs a DCGM collector that emits, per device, metrics labelled with the GPU/MIG UUID, then *decorates* each series with `exported_namespace`, `exported_pod`, `exported_container` looked up from the pod-resources map. You mirror that: for each DCGM sample, look up its UUID in your map and attach the owner labels. The utilization field you want:

- `DCGM_FI_DEV_GPU_UTIL` — coarse % busy (SM had ≥1 kernel). Fine for whole-GPU/MIG.
- `DCGM_FI_PROF_GR_ENGINE_ACTIVE` — fraction of time the graphics/compute engine was active (0–1). Truer utilization; prefer it when profiling is available.

**MIG (clean 1:1).** DCGM reports per-MIG-instance metrics keyed by the `MIG-xxxx` UUID, and pod-resources reports that *same* `MIG-xxxx` for the owning container. Join by UUID → done, one pod per series. No fallback needed. This is why MIG is the "easy" attribution mode despite being the "hard" hardware mode.

**Time-slicing / MPS (many:1 → per-PID fallback).** All sharers show the same `GPU-xxxx`; DCGM's device-level util is the *sum* across sharers and cannot be split by UUID. This is precisely the failure `dcgm-exporter#642` documents — a DCGM maintainer confirmed it's by-design (a device-aggregate NVML value, not a bug) on hardware as current as Blackwell. Fall back to **per-process accounting** and map PID → container:

- Signal: NVML `nvmlDeviceGetProcessUtilization` (per-PID SM/mem %), or `nvidia-smi pmon` / DCGM per-PID accounting. This gives `PID → utilization` on the shared device.
- PID → container: resolve each PID's cgroup (`/proc/<pid>/cgroup`) to the pod/container ID, or match against the runtime. Then split the device's utilization across pods **proportional to per-PID SM activity**.
- If you can't get per-PID (older stack, MPS hiding PIDs), degrade gracefully: **attribute by fair share** (device util ÷ number of sharing pods) and **label the series `attribution=shared-estimate`** so downstream never mistakes an estimate for a measurement.

### DRA makes the map easier — but you still need pod-resources for utilization

On a 04.9 DRA cluster the `ResourceClaim.status.allocation` already names the device, so you *could* build UUID→pod by watching claims instead of polling pod-resources — do it, a claim informer is a clean, event-driven source of the owner map, and it's the concrete payoff of having done 04.9. But DCGM keys utilization by the *runtime* GPU/MIG UUID, and pod-resources is still the ground truth that a container actually holds that UUID at scrape time, so keep the pod-resources poll as the join key even when DRA supplies the ownership. **DRA improves *where the map comes from*; it does not remove the utilization join** — the cardinality problem above is exactly as real on a claim-scheduled time-sliced device as on a device-plugin one, because DRA's per-claim sharing config still multiplexes pods onto one physical UUID.

### Cost: from utilization to dollars

Join three inputs:

```
per_pod_gpu_cost_per_hour =
    node_hourly_rate                      # $ for the whole node (billing/rate-card)
    ÷ gpus_per_node                       # $ per physical GPU-hour
    × pod_share_of_device                 # 1.0 whole GPU; 1/7 for a 1g.10gb MIG slice; per-PID fraction if shared
    × (utilization_fraction or 1.0)       # ×util for *efficiency* cost; ×1 for *allocated* cost
```

Emit **two** metrics, not one: **allocated cost** (what the pod reserved — bills even at 0% use, this is what finance charges back) and **efficiency/utilised cost** (allocated × utilization — the waste signal). The gap between them is your idle-GPU story. For a MIG slice, `pod_share_of_device` is the slice fraction (a `1g.10gb` on an 80GB H100 ≈ 1/7 of compute); for a shared full GPU it's the per-PID fraction from the fallback.

```go
// Prometheus gauges, labelled so PromQL can group by team/namespace:
allocated := ratePerGPUHour * podShare                 // bills regardless of util
utilised  := allocated * utilFraction                  // efficiency-adjusted
gpuCostAllocated.With(labels).Set(allocated)
gpuCostUtilised.With(labels).Set(utilised)
// labels: namespace, pod, container, gpu (UUID), device_type=mig|shared|whole, attribution=exact|shared-estimate
```

### Assemble the GPU Operator failure-mode log

As part of the deliverable, roll up the failure modes you hit across L2/L5/L6 into `practice/per-pod-attribution/failure-modes.md`: driver-container/toolkit mismatch (L2), a real crash-loop signature like the EKS-1.30 `failed to get sandbox runtime: no runtime for 'nvidia' is configured` family from `NVIDIA/gpu-operator#1220` if you hit something similar, time-slicing that oversubscribes and tanks per-pod throughput (L5/L7), MPS `defaultActiveThreadPercentage` starving a co-tenant (L8), and the attribution-specific ones — pod-resources socket unreadable (hostPath/permission), UUID present in DCGM but missing from the map (pod deleted mid-scrape → serve-from-cache), and a time-sliced device with no per-PID data (fall back + flag). The checkpoint requires **at least 5 real entries**, each with symptom → evidence → root cause → fix → prevention — this is the artifact that survives an interview follow-up ("tell me about a GPU incident you actually debugged") better than anything else in this module.

## Perspectives

**Systems-integration perspective.** Nothing in this lesson is individually hard — you've built or read every component already. What's hard is composing nine subsystems (Operator health, pod-resources, CDI, driver lifecycle, MIG, time-slicing, MPS, DRA, DCGM) into one process that doesn't fall over when any one of them misbehaves. This is the actual day-to-day work of a staff platform engineer: not inventing new mechanism, but building the thing that sits on top of everyone else's mechanism and has to keep working when theirs doesn't.

**FinOps/finance-consumer perspective.** The two-metric design (allocated vs utilised) exists because a single number lies by omission. If you emit only "allocated cost," you can chargeback correctly but you erase the waste signal — nobody discovers the idle H100. If you emit only "utilised cost," you undercount what finance actually pays, because the node is billed whether the GPU is busy or not. The gap between the two numbers is a report in itself: it's the dollar amount your organization is paying for GPUs that are scheduled but idle, and it's usually the single most actionable number this whole module produces.

**Site-reliability/operability perspective.** This exporter has to be more reliable than the thing it's measuring. A dashboard that goes blank because the pod-resources socket had a transient error, or emits a wildly wrong number because a pod was deleted between the DCGM scrape and the map lookup, is worse than no dashboard — it trains people to distrust the data. Serve-from-cache on socket error, and label stale-vs-fresh, are not nice-to-haves; they're the difference between an exporter people trust and one they route around.

**Honesty/epistemics perspective.** The `attribution=exact` vs `attribution=shared-estimate` label is the single most important design decision in this capstone, and it's easy to skip under time pressure. An estimate presented with the same confidence as a measurement is actively worse than no data, because it gets treated as ground truth in a chargeback dispute. The correct engineering response to "I can't measure this precisely" is not to fake precision — it's to compute the best available estimate and say, in the data itself, that it's an estimate. This is the same discipline this whole research process was built on: cite what you verified, flag what you didn't, never present a guess as a fact.

## Real-world use cases

- **[NVIDIA/dcgm-exporter](https://github.com/NVIDIA/dcgm-exporter) — the reference implementation.** Fetched and read this session. This capstone re-derives dcgm-exporter's actual spine: the pod-resources→per-pod-metric join, the `exported_pod/namespace/container` label pattern, and MIG UUID handling. Read its source, not just its README, to see how it structures the join.
- **[NVIDIA/dcgm-exporter#642](https://github.com/NVIDIA/dcgm-exporter/issues/642) — the exact failure this capstone's fallback path solves.** Fetched and read this session; already cited in L7 as the strongest evidence in this module, worth re-citing here because it's the concrete, dated, on-hardware proof that the many:1 case this lesson's fallback exists to handle is not a textbook exercise — it's a real gap in NVIDIA's own reference exporter, reported on current-generation (Blackwell) hardware.
- **CloudZero, ["GPU cost attribution for Kubernetes is here"](https://www.cloudzero.com/blog/gpu-cost-attribution-kubernetes/).** *Not independently fetched this session* (proxy-blocked; found via search). Directly on-topic per its title and snippet — a FinOps vendor's framing of exactly this problem ("DCGM reports at the physical GPU level... cannot attribute compute to individual virtual replicas"), useful for how a commercial vendor pitches the same gap you're building an exporter to close.
- **Honest gap:** this research could not find or corroborate a named-company engineering blog describing a homegrown per-pod GPU cost exporter matching this capstone's exact shape (pod-resources join + MIG/time-slicing branch + two-metric dollarization). Rather than invent one, lean on `dcgm-exporter`'s own source as the primary reference — it is the closest real, verifiable system doing this exact join, even though it stops short of the cost layer you're adding.

## Worked example

Two pods, one time-sliced H100 (`GPU-abc`), node rate $32/hr, 8 GPUs/node → $4/GPU-hr.

```
pod-resources ListPodResources:
  team-a/trainer  ctr -> DeviceIds: ["GPU-abc"]
  team-b/notebook ctr -> DeviceIds: ["GPU-abc"]     # SAME UUID — many:1
```

Naive UUID join → both pods get `GPU-abc`'s util; you'd double-bill or mislabel. This is the exact shape of the failure `dcgm-exporter#642` reports live. Fallback:

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

Contrast MIG: two `1g.10gb` slices on the same card show `MIG-x` and `MIG-y` in *both* pod-resources and DCGM → join by UUID, `attribution=exact`, `pod_share=1/7`, no per-PID needed. Same operator, two code paths chosen by whether the UUID is 1:1 or many:1.

```promql
# team monthly chargeback (allocated) vs waste (allocated - utilised):
sum by (namespace) (gpu_cost_allocated_per_hour) * 730
sum by (namespace) (gpu_cost_allocated_per_hour - gpu_cost_utilised_per_hour) * 730
```

## Practice

**This section is the module deliverable** — [../practice/per-pod-attribution/README.md](../practice/per-pod-attribution/README.md). "Shipping it" means all of the following are true, committed, and demonstrable on a live GPU node:

1. **Mapper:** poll the pod-resources API (04.3 client) on a ticker; build `UUID → {ns, pod, container}`; serve from cache; optionally trigger an immediate poll from a pod informer on add/delete.
2. **Join:** read DCGM (`DCGM_FI_PROF_GR_ENGINE_ACTIVE`, fallback `DCGM_FI_DEV_GPU_UTIL`) — or NVML/`nvidia-smi` for a minimal build — and attach `exported_namespace/pod/container` by UUID.
3. **Cost:** emit `gpu_cost_allocated_per_hour` and `gpu_cost_utilised_per_hour` = share × rate (× util), `rate = node_hourly_rate ÷ gpus_per_node`.
4. **MIG case:** distinct UUID → 1:1, `attribution=exact`, share = slice fraction.
5. **Time-sliced case:** shared UUID → per-PID (NVML `nvmlDeviceGetProcessUtilization`, PID→cgroup→pod) to split; if unavailable, fair-share and label `attribution=shared-estimate`.
6. **(Optional, if you completed 04.9) DRA-sourced map:** a claim informer building the same ownership map from `ResourceClaim.status.allocation` instead of polling pod-resources, with the utilization join unchanged.
7. **Failure-mode log:** `failure-modes.md` rolling up L2/L5/L6 (and any others you actually hit) plus the attribution-specific edges above — **≥5 real entries**, each with symptom → `kubectl`/log evidence → root cause → fix → prevention. This is not optional flavor text; it's an explicit checkpoint pass criterion (item 8, "Ship it") and the acceptance criteria in the deliverable spec.

**Acceptance = the module checkpoint** ([../checkpoint.md](../checkpoint.md)). Concretely: on a live GPU node, `curl` the exporter and show per-pod series carrying `namespace`, `pod`, `gpu` (UUID), `device_type`, `attribution`, and both cost gauges; demonstrate a **MIG** pod (`attribution=exact`) and a **time-sliced** pair (`attribution=shared-estimate`, shares summing to the device util within tolerance); commit the code, a scrape sample, and `failure-modes.md` with its ≥5 entries.

## Common pitfalls

1. **Presenting `attribution=shared-estimate` values with the same confidence as `attribution=exact` ones.** An unlabeled estimate in a chargeback dashboard becomes a false fact the moment someone screenshots it into a finance report. Label every series honestly and let the label survive into any dashboard built on top of these metrics.
2. **Blanking the dashboard on a transient pod-resources socket error instead of serving the last-known-good map from cache.** A metrics gap that looks like "the GPU stopped being used" is worse than slightly stale data that's clearly timestamped.
3. **Emitting only one cost number.** "Allocated" alone hides the waste signal (idle GPUs look invisible); "utilised" alone undercounts what's actually billed (the node is paid for whether the GPU is busy or not). You need both, and the gap between them, to tell the real story.
4. **Assuming DRA removes the need for the utilization join.** A DRA claim tells you who *holds* a device; it doesn't change the fact that a time-sliced device still reports one utilization number for all its sharers. The cardinality problem is orthogonal to which API supplied the ownership map.
5. **Skipping the failure-mode log because "the exporter works."** The checkpoint and the deliverable spec both require ≥5 real entries — this isn't a formality, it's the artifact that most directly demonstrates staff-level debugging depth in an interview, and it's the one piece of this capstone that can't be faked after the fact from memory.

## Self-check

- A time-sliced device has a shared UUID (many pods:1 device) — how do you attribute per-pod, and what's the fallback signal? **Answer:** The device UUID join collapses — pod-resources returns the same `GPU-xxxx` for every sharer, so a UUID join alone can't split the device; `dcgm-exporter#642` documents exactly this on real hardware. Fall back to per-process accounting: NVML `nvmlDeviceGetProcessUtilization` (or `nvidia-smi pmon` / DCGM per-PID) gives `PID → SM utilization` on that device; resolve each PID's cgroup to its pod/container; split the device's utilization proportional to per-PID SM activity. If per-PID data is unavailable, degrade to fair-share (device util ÷ sharer count) and label the series `attribution=shared-estimate` so it's never mistaken for a measurement.
- How do you keep the UUID→pod map fresh as pods churn — watch vs poll? **Answer:** The pod-resources API has no watch — `List` is the only call — so you poll it on a short ticker (10–15s) and rebuild the map, serving the last good map from cache on transient socket errors. To close the gap for short-lived pods, add a Kubernetes pod informer (watch) that triggers an immediate pod-resources `List` on add/delete — the informer signals *that* something changed but never carries device IDs, so it augments the poll, it doesn't replace it. On a DRA cluster (04.9), a ResourceClaim informer is an alternative event-driven ownership source, but you still need the pod-resources/DCGM poll for utilization.
- How do you turn a device UUID + utilization into a per-pod dollar figure — what inputs do you join? **Answer:** Join four things: the UUID→pod map (pod-resources or, on DRA, ResourceClaim status), the utilization for that UUID (DCGM `DCGM_FI_PROF_GR_ENGINE_ACTIVE`, per-PID for shared devices), the rate (`node_hourly_rate ÷ gpus_per_node`, a $/GPU-hour), and the pod's share of the device (1.0 whole GPU; slice fraction for MIG; per-PID fraction for time-slicing). Emit two numbers: allocated = rate × share (bills even when idle — the chargeback figure) and utilised = allocated × utilization (the efficiency figure). Their gap is the idle-waste signal.
- Why is MIG the "easy" attribution case despite being the "harder" hardware feature to operate, and why is time-slicing the reverse? **Answer:** MIG partitions the physical hardware itself, so each slice gets a genuinely distinct UUID (`MIG-xxxx`) that both pod-resources and DCGM report consistently — the join is a trivial 1:1 lookup, no fallback needed, even though *configuring* MIG (drain, reconfigure, redeploy) is operationally heavier than flipping a time-slicing ConfigMap. Time-slicing and MPS multiplex several pods onto one unpartitioned physical UUID at the software layer — trivial to configure, but the UUID join is structurally many:1 and cannot be split without a second, independent per-process signal (per-PID NVML data). Attribution difficulty tracks whether the hardware or the software drew the sharing boundary, not how hard the feature is to turn on.

## Connections & what's next

This capstone is the point where every earlier lesson in module 04 becomes one artifact: L1/L2 keep the node healthy enough to produce real data; L3 supplies the mapper; L4 explains why the device ID inside the container is trustworthy; L5 is why the exporter has to survive driver upgrades; L6/L7/L8 are the three sharing-mode branches the exporter has to detect and handle correctly; L9 supplies a second, cleaner ownership-map source and the multi-tenancy quota layer this exporter's numbers eventually feed into a chargeback policy for. If you can explain, cold, why time-slicing breaks the UUID join and MIG doesn't, and then show a working exporter that handles both — you've passed the module's hardest interview probe before anyone asks it.

Forward: the per-pod cost/efficiency signal you just built is the raw material for **module 05 (GPU observability)**, which turns single exporters like this one into fleet-wide dashboards, alerting, and SLOs — the same `gpu_cost_allocated_per_hour` / `gpu_cost_utilised_per_hour` gauges become the input to a Grafana board and an alert on sustained low utilization. It also feeds **module 06 (scheduling and capacity)**, where the attribution you built here — knowing precisely what each pod costs and how efficiently it used its device — becomes an input to bin-packing, autoscaling, and capacity-planning decisions: you can't schedule for cost efficiency until you can measure cost per pod, which is exactly what this capstone produces.

## References & further reading

**Primary sources**
- [NVIDIA/dcgm-exporter](https://github.com/NVIDIA/dcgm-exporter) — fetched and read this session. The reference implementation of the pod-resources→per-pod-metric join; read its source (not just the README) for the exact `exported_pod/namespace/container` label pattern and MIG-UUID handling this capstone re-derives.
- [kubelet pod-resources proto](https://github.com/kubernetes/kubelet/tree/master/pkg/apis/podresources) — fetched and read this session (via the cross-lesson primary-source check in L3/L9). The `ListPodResources` message shapes your mapper consumes directly.
- [kubernetes-sigs/dra-driver-nvidia-gpu](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu) — fetched and read this session (v0.4.1 latest confirmed, see L9). Read if you're building the optional DRA-sourced ownership-map path — the claim/allocation object shapes it consumes.

**Real-world engineering blogs**
- [NVIDIA/dcgm-exporter#642](https://github.com/NVIDIA/dcgm-exporter/issues/642) — fetched and read this session; cross-referenced from L7. The concrete, dated, on-hardware proof of the exact failure this capstone's per-PID fallback exists to solve — cited here again because it is the strongest single piece of evidence in the whole module for why the "easy" allocation-based join is not enough.
- CloudZero, ["GPU cost attribution for Kubernetes is here"](https://www.cloudzero.com/blog/gpu-cost-attribution-kubernetes/) — *not independently fetched this session (proxy-blocked)*. A FinOps vendor's framing of the same problem this capstone solves; useful for seeing how a commercial product pitches the gap you're closing yourself.

**Deeper dives**
- Google Cloud, ["Kubernetes device management with DRA (Dynamic Resource Allocation)"](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-device-management-with-dra-dynamic-resource-allocation) — fetched and read this session (see L9 for full detail). Worth a second read here specifically for its framing of DRA's claim model as the long-term structural replacement for the pod-resources join this capstone builds by hand today.
