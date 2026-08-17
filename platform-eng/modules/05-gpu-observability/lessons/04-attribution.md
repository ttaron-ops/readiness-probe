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
sources: 18
---
# 05.4 · Attribution — from truthful metrics to per-namespace GPU-hours

> **Concept.** Join dcgm-exporter's UUID-keyed truthful metrics to the pod-resources labels you built in 04, aggregate to namespace, and expose the gap between *who holds* GPUs and *who uses* them — the divergence that is the whole story.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Where this fits

05.3 got `DCGM_FI_PROF_SM_ACTIVE` actually landing in Prometheus — uncommented in a full
copy of the counter CSV, enriched with `pod`/`namespace`/`container` from the pod-resources
join, cardinality bounded by an anchored allowlist. That was necessary and not sufficient: a
per-GPU truthful series answers only *"is this device busy?"*. Finance, your director and the
interview panel do not ask about a device. They ask about a **team**.

This lesson is the last hop before the capstone. It takes the honest per-GPU signal and
turns it into two per-namespace numbers — **allocated GPU-hours** and **SM-active
GPU-hours** — whose difference, multiplied by a rate, is a dollar figure a non-engineer can
act on. It also draws the boundary of what the join can and cannot know, because the
fastest way to destroy trust in a chargeback dashboard is to confidently attribute
something you cannot actually attribute.

Everything below is checked against **dcgm-exporter 4.8.3** (built on DCGM 4.6.0),
**NVIDIA k8s-device-plugin v0.19.x**, the **kubelet pod-resources v1 API** as defined in
`k8s.io/kubelet/pkg/apis/podresources/v1/api.proto`, and **Prometheus 3.x**. The
source-level claims come from `internal/pkg/transformation/kubernetes.go`,
`internal/pkg/transformation/process_metrics.go`,
`internal/pkg/rendermetrics/render_metrics.go`, `internal/pkg/nvmlprovider/provider.go` and
`internal/rm/devices.go` in the device plugin.

## Why this matters

The stakes are dollar-shaped and the failure mode is reputational.

GPU sharing is sold on large, real numbers — time-slicing pitches "ten jobs on one GPU
instead of ten GPUs" — which is exactly why you will be pressured to adopt it, and exactly
why you need its attribution blind spot cold *before* you do. A platform vendor whose entire
product is GPU cost optimisation (ScaleOps) states the same hole this lesson teaches: under
time-slicing, multiple pods share one physical GPU while many DCGM metrics remain
device-level, so duplicated labels must not be read as exact per-pod usage. That is not a
course-invented caveat; it is an admission from a company with every commercial incentive to
undersell the limitations of GPU sharing.

The concrete cost of getting it wrong: **a chargeback number that does not survive
scrutiny.** You present a per-namespace waste ranking. A team lead checks one node, notices
their pod shares a time-sliced card with three others, and asks how you split the 24
GPU-hours. If the honest answer is "we credited whichever pod the exporter saw last", the
number is dead and so is the dashboard, permanently. If the honest answer is "those hours
are in an explicit `unattributable` bucket, here it is, here is why, and here is what
switching that node to MIG would buy us" — you have just made the case for MIG in the same
breath.

And getting it *right* is the direct input to Module 11's cost operator. Every dollar figure
that system emits is downstream of the two aggregates defined here.

## What's new here (calibration)

Module 04 delivered the mechanism — device UUID → pod → namespace, live and correct, the
same join dcgm-exporter performs internally. This lesson does not re-derive it. What is new:

- **You consume the labels, you do not compute them.** With `DCGM_EXPORTER_KUBERNETES=true`
  the exporter has already stamped `pod`/`namespace`/`container`; here they arrive free on
  `DCGM_FI_PROF_SM_ACTIVE` and you write PromQL over them.
- **The integral.** Turning a dimensionless 0–1 ratio sampled every 30 s into GPU-hours is a
  time integral, and the naïve `avg_over_time(...) * 24` form breaks the moment GPUs enter or
  leave the fleet mid-window. §5 derives the form that does not.
- **The sharing hole, mechanically, and corrected.** The device plugin does *not* hand the
  same UUID to co-tenants of a time-sliced GPU — it hands out **annotated replica IDs**
  `GPU-<uuid>::0`, `::1`, `::2`, `::3`. The collapse happens one layer later, inside the
  exporter's `map[string]PodInfo`, and it is a genuine **last-writer-wins over Go map
  iteration order**. Knowing which layer loses the information tells you which layer could
  in principle be fixed, and which cannot.
- **The one exception that proves the rule.** dcgm-exporter *can* produce per-process
  attribution under sharing — but only for `DCGM_FI_DEV_GPU_UTIL` and `DCGM_FI_DEV_FB_USED`,
  via NVML's per-process sampling. **The lying metric is the only one that can be split per
  tenant; the honest one cannot.**
- **`unattributable_gpu_hours` as a first-class output**, not a footnote.

## Core concepts

### 1. Two numerators, and why confusing them is the whole problem

There are two different "GPUs per namespace", they have different units, and the gap between
them is the product.

**Allocated.** How many GPUs a namespace *holds*. This is what the kubelet assigned, what
`nvidia.com/gpu` requests sum to, and — critically — **what you are billed on**. A GPU
allocated to a pod is unavailable to everyone else whether or not a single kernel runs on
it. Units: GPUs. Integrated over time: **GPU-hours**.

**Used.** How much work those GPUs did. `SM_ACTIVE` is a dimensionless ratio in [0, 1]
representing the fraction of SM-cycles with at least one warp assigned (05.1 §6). A GPU held
for one hour at `SM_ACTIVE = 0.05` contributes **1 allocated GPU-hour and 0.05 SM-active
GPU-hours**. Units: GPU-hours, but *of work*, not of possession.

```
    gap_gpu_hours  =  allocated_gpu_hours  −  sm_active_gpu_hours
    waste_dollars  =  gap_gpu_hours  ×  $/GPU-hour
```

Two properties of this definition are worth stating explicitly, because both come up when
someone challenges the number.

**It is deliberately harsh, and that is defensible.** No workload sustains
`SM_ACTIVE = 1.0`; even excellent training runs sit well below it, and a healthy inference
service is memory-bound by design. So `gap` is not "waste you could recover". It is **the
total distance between what you bought and what ran**, of which some is recoverable
(idle-but-allocated GPUs, batch-1 serving, forgotten notebooks) and some is not (kernel
launch gaps, memory-bound phases, synchronisation). Present it as a *ceiling on recoverable
waste* and a *trend line*, never as a bill.

**It is not MFU.** Model-FLOPs-utilisation compares achieved FLOPs to peak FLOPs and is the
right efficiency metric for a training job. `SM_ACTIVE` is breadth, not FLOPs. Use
`PIPE_TENSOR_ACTIVE` if you want the FLOPs conversation; use `SM_ACTIVE` for the
possession-versus-work conversation, which is the one finance is having.

### 2. The join path, end to end

Here is the chain from a hardware counter to a namespace label. Every arrow is a place the
join can fail, and §6 breaks the one that matters.

```
 ══════════════════════════════════════════════════════════════════════════════════════
   THE ATTRIBUTION JOIN — hardware counter to namespace, on one node
 ══════════════════════════════════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ ① GPU                                                                            │
  │    physical device, immutable identity:  GPU-8f2c1a44-9b0d-5e17-a3c8-6d21e4b7f095│
  │    performance monitors → SM_ACTIVE = 0.16                                       │
  └───────────────────────────────┬──────────────────────────────────────────────────┘
                                  │  DCGM entity: (DCGM_FE_GPU, gpuId=0)
                                  ▼
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ ② DCGM host engine (embedded in dcgm-exporter)                                   │
  │    field 1002 cached per entity; identity fields read via NVML                   │
  │    → metric{ gpu="0", UUID="GPU-8f2c…", pci_bus_id, device, modelName, hostname }│
  └───────────────────────────────┬──────────────────────────────────────────────────┘
                                  │  key = GPU UUID  (DCGM_EXPORTER_KUBERNETES_GPU_ID_TYPE
                                  │                   default "uuid"; "device-name" gives
                                  ▼                   "nvidia0" instead)
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ ③ kubelet pod-resources API — unix:///var/lib/kubelet/pod-resources/kubelet.sock │
  │    gRPC  v1.PodResourcesLister/List   (no arguments; returns the whole node)     │
  │                                                                                  │
  │    ListPodResourcesResponse {                                                    │
  │      pod_resources: [                                                            │
  │        { name: "vllm-7d9f-abc", namespace: "svc-chat",                           │
  │          containers: [                                                           │
  │            { name: "server",                                                     │
  │              devices: [ { resource_name: "nvidia.com/gpu",                       │
  │                           device_ids: ["GPU-8f2c1a44-…"] } ] } ] } ] }           │
  │                                                                                  │
  │    exporter-side: 10 s dial timeout, 16 MiB gRPC max receive size,               │
  │    resource_name must be "nvidia.com/gpu", a name in --nvidia-resource-names,    │
  │    or start with "nvidia.com/mig-"; everything else is skipped.                  │
  └───────────────────────────────┬──────────────────────────────────────────────────┘
                                  │  build map[deviceID] → PodInfo{Name,Namespace,Container,UID,Labels}
                                  ▼
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ ④ dcgm-exporter PodMapper.Process()                                              │
  │    for each metric: deviceID := metric.GPUUUID                                   │
  │                     podInfo, ok := deviceToPod[deviceID]                         │
  │                     if ok → metric.Attributes["pod"|"namespace"|"container"] = … │
  │    pod labels (if enabled) fetched from a node-scoped pod informer, not from     │
  │    pod-resources — the API returns names, never labels.                          │
  └───────────────────────────────┬──────────────────────────────────────────────────┘
                                  ▼
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ ⑤ /metrics on :9400                                                              │
  │    DCGM_FI_PROF_SM_ACTIVE{gpu="0",UUID="GPU-8f2c…",device="nvidia0",             │
  │      modelName="NVIDIA H100 80GB HBM3",hostname="gpu-node-07",                   │
  │      DCGM_FI_DRIVER_VERSION="580.126.20",                                        │
  │      pod="vllm-7d9f-abc",namespace="svc-chat",container="server"} 0.163          │
  └───────────────────────────────┬──────────────────────────────────────────────────┘
                                  │  Prometheus scrape, honor_labels: false  (the default
                                  ▼  in BOTH the standalone chart and the GPU Operator)
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ ⑥ Prometheus TSDB                                                                │
  │    server-side labels win; the exporter's colliding ones are RENAMED:            │
  │      pod       → exported_pod                                                    │
  │      namespace → exported_namespace                                              │
  │      container → exported_container                                              │
  │    …and Prometheus attaches its own pod/namespace/container = the EXPORTER's own │
  │    pod. Grouping by `namespace` gives you "gpu-operator" for every GPU on earth. │
  └──────────────────────────────────────────────────────────────────────────────────┘
 ══════════════════════════════════════════════════════════════════════════════════════
```

**Step ⑥ is the single most common way this query is written wrong**, and it fails silently
with a plausible-looking answer. `avg by (namespace) (DCGM_FI_PROF_SM_ACTIVE)` on a
Kubernetes-SD scrape with `honor_labels: false` returns exactly one row, labelled
`namespace="gpu-operator"`, holding the fleet-wide mean. It looks like a working query. It
is the average of everything, attributed to the monitoring namespace.

Establish which spelling you have, once, before writing anything else:

```promql
# Run this first. It tells you which labels actually exist on your series.
count by (__name__) (DCGM_FI_PROF_SM_ACTIVE)
topk(1, DCGM_FI_PROF_SM_ACTIVE)          # then read the label set off the result

# Or explicitly:
count(DCGM_FI_PROF_SM_ACTIVE{exported_namespace!=""})   # non-zero → you have exported_*
count(DCGM_FI_PROF_SM_ACTIVE{namespace!="gpu-operator"}) # non-zero → honor_labels is true
```

The rest of this lesson writes `exported_namespace`, because that is what both shipped
charts produce. Substitute `namespace` if you set `honorLabels: true`. **Do not** "fix" this
with `honor_labels: true` casually — it also lets a target override `job` and `instance`,
which breaks other things. If you want clean names, use `metric_relabel_configs` to rename
`exported_namespace` → `gpu_namespace` explicitly.

One more property of the join worth internalising: **it is refreshed every scrape from a
live gRPC call, and it fails soft.** `PodMapper.Process()` wraps the pod-resources call and,
on error, logs and returns `nil` — *"Don't fail the whole scrape, just skip enrichment"*.
There is a named case for the response exceeding the 16 MiB gRPC receive limit, which logs
*"Kubelet pod-resources response exceeded gRPC receive limit; pod metric labels will be
missing this scrape"*. **So the failure mode of the join is metrics that arrive with no
namespace label at all**, which in PromQL means they drop out of every `by (exported_namespace)`
aggregation — the same silent-undercount pattern as 05.1's missing-series problem, one layer
up. Alert on it:

```promql
# GPUs reporting SM_ACTIVE with no namespace label for 10 minutes.
# Either genuinely unallocated (fine) or the pod-resources join is failing (not fine).
# Compare against your known idle-node count.
count(DCGM_FI_PROF_SM_ACTIVE{exported_namespace=""})
  / ignoring(exported_namespace) count(DCGM_FI_PROF_SM_ACTIVE) > 0.5
```

### 3. The pod-resources API, concretely

Module 04 built this; here is the reference you need to reason about its limits, from
`api.proto` in `k8s.io/kubelet`:

```protobuf
service PodResourcesLister {
    rpc List(ListPodResourcesRequest) returns (ListPodResourcesResponse) {}
    rpc GetAllocatableResources(AllocatableResourcesRequest) returns (AllocatableResourcesResponse) {}
    rpc Get(GetPodResourcesRequest) returns (GetPodResourcesResponse) {}
}

message PodResources {
    string name = 1;
    string namespace = 2;
    repeated ContainerResources containers = 3;
    repeated int64 cpu_ids = 4;
    repeated ContainerMemory memory = 5;
}

message ContainerResources {
    string name = 1;
    repeated ContainerDevices devices = 2;
    repeated int64 cpu_ids = 3;
    repeated ContainerMemory memory = 4;
    repeated DynamicResource dynamic_resources = 5;   // DRA
}

message ContainerDevices {
    string resource_name = 1;              // "nvidia.com/gpu", "nvidia.com/mig-1g.10gb", …
    repeated string device_ids = 2;        // ["GPU-8f2c…"] or ["GPU-8f2c…::2"] or ["MIG-…"]
    TopologyInfo topology = 3;             // NUMA nodes
}

message DynamicResource {
    string claim_name = 2;
    string claim_namespace = 3;
    repeated ClaimResource claim_resources = 4;
}

message ClaimResource {
    repeated CDIDevice cdi_devices = 1;    // e.g. "nvidia.com/gpu=gpudevice1"
    string driver_name = 2;                // "gpu.nvidia.com"
    string pool_name = 3;
    string device_name = 4;
    optional string share_id = 5;
}
```

Five things this tells you that matter for attribution:

1. **It returns pod *name* and *namespace*, never labels.** Anything richer — `app`, `team`,
   `job-id` — requires a Kubernetes API read. That is why `enablePodLabels` needs RBAC
   (`pods: get,list,watch`) and a pod informer, and why it is a separate opt-in from the
   basic join (05.3 §7).
2. **It is node-scoped and whole-node.** `List()` takes no arguments and returns every pod
   with an allocation on that node. dcgm-exporter calls it **once per scrape**. On a node
   with many pods and many devices, that response grows — hence the 16 MiB receive limit the
   exporter sets and warns about.
3. **`device_ids` is a list of opaque strings whose format is the device plugin's choice.**
   Whole GPU: `GPU-<uuid>`. MIG: `MIG-<uuid>`. Time-sliced replica:
   **`GPU-<uuid>::<replica>`**. GKE virtual GPU: `<gpuid>/vgpu<n>`. GKE MIG:
   `nvidia<idx>/gi<instance>`. All five shapes are parsed in
   `internal/pkg/transformation/kubernetes.go`, and the shape is what decides whether
   attribution survives.
4. **`resource_name` is filtered.** The exporter skips any device whose resource name is not
   `nvidia.com/gpu`, not in `--nvidia-resource-names`, and does not start with
   `nvidia.com/mig-`. **If your cluster advertises GPUs under a custom resource name — some
   sharing operators do — the join silently produces nothing until you add it to
   `NVIDIA_RESOURCE_NAMES`.**
5. **DRA is a separate branch.** `dynamic_resources` carries a claim name/namespace and a
   driver/pool/device triplet rather than a device ID. dcgm-exporter reads it only with
   `KUBERNETES_ENABLE_DRA=true` and emits `dra_claim_name`, `dra_claim_namespace`,
   `dra_driver_name`, `dra_pool_name`, `dra_device_name` (plus `dra_mig_profile` /
   `dra_mig_device_uuid` for MIG-backed claims). **On a DRA cluster with that flag off, GPUs
   are allocated and your join returns nothing.** Module 04's lesson 09 owns DRA; the
   attribution consequence is: check the flag before concluding your GPUs are unallocated.

### 4. The labels you inherit

For an `FE_GPU` entity, the complete label set (05.3 §6 has the provenance):

| Label | From | Use in attribution |
|---|---|---|
| `gpu` | renderer | node-local index; **not unique across the fleet** |
| `UUID` | renderer | **the join key** — globally unique, survives reboots |
| `pci_bus_id`, `device`, `modelName` | renderer | grouping by GPU model for per-model rates |
| `hostname` | renderer | node grouping; suppressed by `--no-hostname` |
| `GPU_I_PROFILE`, `GPU_I_ID` | renderer, MIG only | MIG slice identity, e.g. `1g.10gb` |
| `DCGM_FI_DRIVER_VERSION` | `label`-typed CSV row | fleet hygiene |
| `exported_pod` / `exported_namespace` / `exported_container` | pod-resources | **the attribution target** |
| `pod_uid` | pod informer, opt-in | unbounded; avoid |
| `vgpu` | device-ID suffix | present under sharing — a useful flag |
| sanitised pod labels | pod informer, opt-in | `app_kubernetes_io_name`, etc. |
| `dra_*` | pod-resources DRA branch | DRA clusters only |

**Group by `exported_namespace`; join by `UUID`.** `gpu` is a node-local index — `gpu="0"`
exists on every node — so any `on (gpu)` join across nodes is wrong. Use
`on (UUID)`, or `on (instance, gpu)` if you must.

### 5. GPU-hours: doing the integral correctly

This is where most implementations are subtly wrong, and the error only shows up when the
fleet changes size mid-window — which is exactly when someone is looking.

**What you want, mathematically.** Over a window `W`:

```
                     ⌠
  allocated_GPU_h =  ⎮  N_alloc(t) dt         N_alloc = number of allocated GPUs
                     ⌡W                                  in this namespace at time t

                     ⌠
  sm_active_GPU_h =  ⎮  Σ  SM_ACTIVE_g(t) dt  summed over that namespace's GPUs
                     ⌡W  g
```

Both are integrals of an instantaneous quantity. In Prometheus you approximate an integral
as `mean × duration`, and the mean must be taken **over time of the aggregate**, not over
series of the time-average. Those two are different whenever the series set changes.

**The wrong form** (and the one in most write-ups, including the previous version of this
lesson):

```promql
# ✗ WRONG when GPUs enter or leave the namespace during the window
sum by (exported_namespace) (avg_over_time(DCGM_FI_PROF_SM_ACTIVE[24h])) * 24
```

Why it breaks: `avg_over_time(X[24h])` averages *the samples that exist* for each series. A
GPU allocated for the last hour of the window has ~120 samples, and `avg_over_time` divides
by 120, not by 2,880 — so its one hour of work is weighted as if it were 24. Then `sum`
operates on whichever series happen to exist at *evaluation* time, silently dropping GPUs
released before the query ran. On a stable fleet the error is small. On a training fleet with
job churn, it can be several-fold, in an unpredictable direction.

**The right form** uses a subquery so the aggregation happens at each step, and the time
average is taken of the *already-aggregated* series:

```promql
# ✓ SM-active GPU-hours per namespace over 24 h
#
#   inner:  sum by (ns) (SM_ACTIVE)  → instantaneous "GPU-equivalents of work" per ns
#   [24h:1m]: evaluate that every 1 minute across the window
#   avg_over_time: time-average of the aggregate
#   × 24:  mean × hours = GPU-hours
avg_over_time(
  sum by (exported_namespace) (DCGM_FI_PROF_SM_ACTIVE)[24h:1m]
) * 24
```

```promql
# ✓ Allocated GPU-hours per namespace over 24 h
#   Count distinct allocated devices at each step. Counting series that CARRY a
#   namespace label is the allocation signal: dcgm-exporter only stamps the label
#   when pod-resources reported the device as assigned.
avg_over_time(
  count by (exported_namespace) (
    DCGM_FI_PROF_SM_ACTIVE{exported_namespace!=""}
  )[24h:1m]
) * 24
```

```promql
# ✓ Efficiency: the fraction of held GPU-time that was SM-active
avg_over_time(sum   by (exported_namespace) (DCGM_FI_PROF_SM_ACTIVE)[24h:1m])
/
avg_over_time(count by (exported_namespace) (DCGM_FI_PROF_SM_ACTIVE{exported_namespace!=""})[24h:1m])
```

```promql
# ✓ The gap, in dollars. Rate injected as a recording rule or a Grafana variable —
#   NEVER hard-coded in a committed query (the repo .gitignore guards real rates).
(
  avg_over_time(count by (exported_namespace) (DCGM_FI_PROF_SM_ACTIVE{exported_namespace!=""})[24h:1m])
  -
  avg_over_time(sum   by (exported_namespace) (DCGM_FI_PROF_SM_ACTIVE)[24h:1m])
) * 24 * on() group_left() gpu_hourly_rate_usd
```

**A caution about subqueries.** `[24h:1m]` makes Prometheus evaluate the inner expression
1,440 times per query per namespace. On a 400k-series fleet that is slow enough to time out
a dashboard. **The production answer is recording rules**: pre-aggregate at scrape
resolution, then integrate over the recorded series, which are tiny.

```yaml
# prometheus-rules.yaml — evaluate at the scrape interval so no interpolation is needed
groups:
  - name: gpu-attribution
    interval: 30s
    rules:
      # Instantaneous GPU-equivalents of SM work, per namespace.
      - record: namespace:gpu_sm_active:sum
        expr: sum by (exported_namespace) (DCGM_FI_PROF_SM_ACTIVE)

      # Instantaneous count of allocated GPUs, per namespace.
      - record: namespace:gpu_allocated:count
        expr: count by (exported_namespace) (DCGM_FI_PROF_SM_ACTIVE{exported_namespace!=""})

      # Allocated GPUs whose SM work is ~zero right now. The reclaim candidate set,
      # gated on framebuffer so a loaded-but-paused serving replica is excluded.
      - record: namespace:gpu_idle_allocated:count
        expr: |
          count by (exported_namespace) (
              DCGM_FI_PROF_SM_ACTIVE{exported_namespace!=""} < 0.05
            and on (UUID)
              DCGM_FI_DEV_FB_USED < 2048
          )

      # GPU-equivalents whose SM signal exists but has no owner: unallocated, OR the
      # pod-resources join failed this scrape. Track it; do not fold it into a team.
      - record: gpu_unowned:count
        expr: count(DCGM_FI_PROF_SM_ACTIVE{exported_namespace=""}) or vector(0)
```

With those recorded, the dashboard queries become cheap and the integral is exact, because
the recorded series is already the aggregate:

```promql
# allocated GPU-hours, 24 h  — no subquery, evaluates in milliseconds
avg_over_time(namespace:gpu_allocated:count[24h]) * 24

# SM-active GPU-hours, 24 h
avg_over_time(namespace:gpu_sm_active:sum[24h]) * 24

# the gap
( avg_over_time(namespace:gpu_allocated:count[24h])
  - avg_over_time(namespace:gpu_sm_active:sum[24h]) ) * 24
```

**Sanity-check your arithmetic before trusting it.** Two invariants that catch most bugs:

```promql
# 1. Total allocated GPU-hours must not exceed fleet size × window hours.
sum(avg_over_time(namespace:gpu_allocated:count[24h])) * 24   <=   520 * 24

# 2. Per namespace, SM-active hours must never exceed allocated hours.
#    SM_ACTIVE ≤ 1 per GPU, so the sum is bounded by the count. If this is
#    ever violated you have a label-join bug, almost always double-counting
#    MIG instances alongside their parent GPU.
avg_over_time(namespace:gpu_sm_active:sum[24h])
  <= avg_over_time(namespace:gpu_allocated:count[24h])
```

### 6. Where attribution breaks: the sharing hole, mechanically

Everything above assumes **one tenant per DCGM entity**. Under whole-GPU allocation and
under MIG that holds. Under time-slicing and MPS it does not, and the reason is precise
enough to draw.

Start from what the device plugin emits. In `internal/rm/devices.go` (k8s-device-plugin):

```go
// AnnotatedID represents an ID with a replica number embedded in it.
type AnnotatedID string

func NewAnnotatedID(id string, replica int) AnnotatedID {
    return AnnotatedID(fmt.Sprintf("%s::%d", id, replica))
}
```

So a GPU configured for 4-way time-slicing is advertised as four schedulable devices with
IDs `GPU-8f2c…::0`, `::1`, `::2`, `::3`, and pod-resources faithfully reports each pod's
distinct replica ID. **The information is present at layer ③.** It is destroyed at layer ④,
and here is exactly how — `toDeviceToPod()` in `internal/pkg/transformation/kubernetes.go`:

```go
} else if strings.Contains(deviceID, "::") {
    gpuInstanceID := strings.Split(deviceID, "::")[0]   // "GPU-8f2c…"
    deviceToPodMap[gpuInstanceID] = podInfo             // ← every replica writes HERE
}
// Default mapping between deviceID and pod information
deviceToPodMap[deviceID] = podInfo                      // "GPU-8f2c…::2" (never read)
```

The map is `map[string]PodInfo` — one value per key. All four pods write to the key
`GPU-8f2c…`. **Last write wins, and the order is Go's map/slice iteration over the
pod-resources response, which is not stable between scrapes.** Meanwhile the metric side
looks up by the bare UUID (`val.GetIDOfType(uuid)` → `GPU-8f2c…`), so it finds whichever pod
landed last.

```
 ══════════════════════════════════════════════════════════════════════════════════════
   TIME-SLICING: WHERE THE TENANT IDENTITY IS LOST
 ══════════════════════════════════════════════════════════════════════════════════════

  ① SILICON            ONE physical H100.  ONE set of performance counters.
                       SM_ACTIVE = 0.72 for the card as a whole.
                       No per-tenant counters exist. Time-slicing is context
                       switching, not partitioning — nothing in hardware
                       attributes a cycle to a tenant.
                                    │
  ② DEVICE PLUGIN      advertises nvidia.com/gpu: 4  from ONE GPU
                       device IDs:  GPU-8f2c…::0  ::1  ::2  ::3
                                    │      ▲
                                    │      └── tenant identity STILL PRESENT
                                    ▼
  ③ POD-RESOURCES      pod-a → ["GPU-8f2c…::0"]        namespace: team-red
                       pod-b → ["GPU-8f2c…::1"]        namespace: team-red
                       pod-c → ["GPU-8f2c…::2"]        namespace: team-blue
                       pod-d → ["GPU-8f2c…::3"]        namespace: team-green
                                    │      ▲
                                    │      └── STILL PRESENT, and correct
                                    ▼
  ④ EXPORTER MAPPING   deviceToPodMap["GPU-8f2c…"] = pod-a   ┐
      (default path)   deviceToPodMap["GPU-8f2c…"] = pod-b   │  four writes,
                       deviceToPodMap["GPU-8f2c…"] = pod-c   │  ONE key
                       deviceToPodMap["GPU-8f2c…"] = pod-d   ┘
                                    │      ▲
                                    │      └── ✗ LOST HERE. map[string]PodInfo.
                                    ▼           Last writer wins, order unstable.
  ⑤ SERIES             DCGM_FI_PROF_SM_ACTIVE{UUID="GPU-8f2c…",
                         exported_pod="pod-d",exported_namespace="team-green"} 0.72
                                    │      ▲
                                    │      └── 100% of the card credited to ONE team.
                                    ▼           The other three show nothing.
  ⑥ CHARGEBACK         team-green: 0.72 GPU-equivalents · team-red: 0 · team-blue: 0
                       …and next scrape it may be team-red. The series flaps.

 ──────────────────────────────────────────────────────────────────────────────────────
   WHAT IF YOU SET  KUBERNETES_VIRTUAL_GPUS=true ?
 ──────────────────────────────────────────────────────────────────────────────────────

  ④' toDeviceToSharingPods() builds  map[string][]PodInfo  — it APPENDS instead of
     overwriting, so the key "GPU-8f2c…" holds all four PodInfos.

  ⑤' Process() emits ONE COPY OF THE DEVICE METRIC PER SHARING POD:

       SM_ACTIVE{UUID="GPU-8f2c…", exported_pod="pod-a", vgpu="0"} 0.72
       SM_ACTIVE{UUID="GPU-8f2c…", exported_pod="pod-b", vgpu="1"} 0.72
       SM_ACTIVE{UUID="GPU-8f2c…", exported_pod="pod-c", vgpu="2"} 0.72
       SM_ACTIVE{UUID="GPU-8f2c…", exported_pod="pod-d", vgpu="3"} 0.72
                                                                  ▲▲▲▲
       THE SAME VALUE, FOUR TIMES. This is not per-pod usage. Summing it gives
       2.88 GPU-equivalents of work on a card that has 1.0 to give — which is
       how you get an efficiency ratio above 100% and lose the room.

  ⑤'' EXCEPT for exactly two fields. isPerProcessMetric() returns true only for
      DCGM_FI_DEV_GPU_UTIL and DCGM_FI_DEV_FB_USED. For those, the exporter calls
      NVML per-process sampling and replaces the value per pod:

        nvmlDeviceGetProcessUtilization()  → per-PID smUtil   → GPU_UTIL
        nvmlDeviceGetComputeRunningProcesses() → per-PID bytes → FB_USED
        PID → pod resolved via the pid mapper

      ┌────────────────────────────────────────────────────────────────────────┐
      │  THE METRIC THAT LIES (GPU_UTIL) IS THE ONLY ONE THAT CAN BE SPLIT     │
      │  PER TENANT.  THE METRIC THAT TELLS THE TRUTH (SM_ACTIVE) CANNOT.      │
      └────────────────────────────────────────────────────────────────────────┘
      Because per-process data comes from NVML, which has per-process visibility
      but only presence-grade metrics, while SM_ACTIVE comes from device-scope
      hardware counters that have no process dimension at all (05.1 §2).
 ══════════════════════════════════════════════════════════════════════════════════════
```

**Could the exporter be fixed to preserve the mapping?** For the *mapping*, yes — the
replica identity survives to layer ③ and the exporter chooses to collapse it. For the
*measurement*, no. There is exactly one `SM_ACTIVE` number for the card because there is
exactly one set of counters, and time-slicing is context switching between full-device
contexts rather than a partition. Even a perfect exporter could only tell you *which four
pods share this 0.72*, not how the 0.72 divides. **The upstream limitation is physical; the
downstream one is a data structure.** Say it that way in an interview — it demonstrates you
know which parts of a system are negotiable.

**MPS** is the same conclusion by a different route: co-tenants share one CUDA context on
one device, DCGM sees one entity, and there is no per-client counter.

### 7. MIG: why it is different, in DCGM terms

MIG is not "better sharing"; it is a different mechanism, and the difference is visible in
DCGM's entity model (05.2 §4).

- The hardware partitions the GPU into **GPU instances** (memory slices, L2 slices, and a
  dedicated set of SMs) and, within them, **compute instances**.
- `dcgm_fields.cpp` registers `SM_ACTIVE`, `SM_OCCUPANCY`, `GR_ENGINE_ACTIVE` and
  `PIPE_TENSOR_ACTIVE` at entity level `DCGM_FE_GPU_CI`, and `DRAM_ACTIVE` at
  `DCGM_FE_GPU_I`. **The counters are reported per instance, not per card.**
- On Hopper, NVML's GPM has a dedicated `nvmlGpmMigSampleGet(device, gpuInstanceId, sample)`
  path, which DCGM calls when the entity is a `DCGM_FE_GPU_I`. It is a separate sample
  stream, not a share of the parent's.
- The device plugin advertises MIG instances under `nvidia.com/mig-<profile>` resource names
  with `MIG-<uuid>` device IDs, so pod-resources returns a **distinct UUID per instance**.
- dcgm-exporter resolves the MIG UUID through NVML to a `(parentUUID, gpuInstanceId)` pair
  and keys the mapping on that, emitting `GPU_I_PROFILE` and `GPU_I_ID` labels.

Result: a **1:1 join**, per instance, all the way down. Attribution is exact.

| | Whole GPU | MIG | Time-slicing | MPS |
|---|---|---|---|---|
| Isolation | full | hardware (memory + SM partition) | none (context switch) | none (shared context) |
| DCGM entity per tenant | yes (`FE_GPU`) | **yes** (`FE_GPU_I`/`FE_GPU_CI`) | **no** | **no** |
| pod-resources device ID | `GPU-<uuid>` | `MIG-<uuid>` | `GPU-<uuid>::<n>` | `GPU-<uuid>::<n>` |
| Distinct UUID per tenant | yes | **yes** | no (collapses at the exporter) | no |
| Per-tenant `SM_ACTIVE` | yes | **yes** | **no** | **no** |
| Per-tenant `GPU_UTIL` | yes | yes | only via NVML per-process | only via NVML per-process |
| Max tenants per card | 1 | 7 (A100/H100 profiles) | high (config) | high |
| Billing-grade attribution | yes | **yes** | **no** | **no** |

**The one-line version for a design review:** *"MIG partitions the counters, so attribution
survives. Time-slicing partitions the schedule, so it doesn't. If we need per-team billing
on shared cards, MIG is the only option that produces a defensible number; if we need more
than seven tenants per card, we need to accept an unattributable bucket."*

### 8. `unattributable_gpu_hours`: making the unknown explicit

Given all of the above, the correct engineering response is **not** to guess. It is to
partition the fleet into attributable and unattributable, report both, and let the size of
the second drive a hardware conversation.

Identify the shared GPUs. Three signals, in descending order of directness:

```promql
# (a) Best: the exporter told you. With KUBERNETES_VIRTUAL_GPUS=true a `vgpu` label
#     is stamped from the device-ID suffix. Its presence IS the sharing flag.
count by (UUID) (DCGM_FI_PROF_SM_ACTIVE{vgpu!=""})

# (b) Without that flag: a physical GPU whose label set flaps between pods across
#     scrapes. Count distinct pods seen per UUID over an hour — >1 with only one
#     series at a time means the last-writer-wins collapse.
count by (UUID) (
  count by (UUID, exported_pod) (
    max_over_time(DCGM_FI_PROF_SM_ACTIVE[1h])
  )
) > 1

# (c) Ground truth, from the node side: the device plugin's advertised capacity
#     exceeds the physical GPU count. Requires kube-state-metrics.
   kube_node_status_capacity{resource="nvidia_com_gpu"}
 > on (node) group_left()
   count by (node) (DCGM_FI_PROF_SM_ACTIVE)
```

Signal (c) is the one to build on, because it is a property of configuration rather than of
observed behaviour, and it is true even when the shared GPUs are idle.

Then split the aggregates. Maintain a recording rule flagging shared UUIDs and subtract them
from the per-namespace figures:

```yaml
groups:
  - name: gpu-attribution-sharing
    interval: 30s
    rules:
      # 1 for every GPU on a node whose advertised nvidia.com/gpu capacity exceeds
      # its physical GPU count — i.e. time-sliced or MPS-shared.
      - record: gpu:shared:flag
        expr: |
          count by (UUID, hostname) (DCGM_FI_PROF_SM_ACTIVE)
            * on (hostname) group_left()
              (
                  (kube_node_status_capacity{resource="nvidia_com_gpu"}
                     > on (node) group_left() count by (node) (DCGM_FI_PROF_SM_ACTIVE))
                > bool 0
              )

      # Attributable work only: exclude shared cards from the per-namespace sum.
      - record: namespace:gpu_sm_active_attributable:sum
        expr: |
          sum by (exported_namespace) (
            DCGM_FI_PROF_SM_ACTIVE unless on (UUID) (gpu:shared:flag == 1)
          )

      # Everything happening on shared cards, as one honest bucket.
      - record: gpu_unattributable:sum
        expr: |
          sum(DCGM_FI_PROF_SM_ACTIVE and on (UUID) (gpu:shared:flag == 1)) or vector(0)

      - record: gpu_unattributable_allocated:count
        expr: |
          count(DCGM_FI_PROF_SM_ACTIVE and on (UUID) (gpu:shared:flag == 1)) or vector(0)
```

Then the dashboard shows, side by side:

```
  per-namespace: allocated GPU-h │ SM-active GPU-h │ gap │ gap $
  ─────────────────────────────────────────────────────────────
  team-vision            1,920   │        96       │ 1,824 │ …
  team-nlp                 576   │       430       │   146 │ …
  team-research            960   │       120       │   840 │ …
  ─────────────────────────────────────────────────────────────
  UNATTRIBUTABLE (time-sliced)
    physical GPU-hours       48  │        34.6     │  13.4 │ …
    tenant pods sharing them: 8 across 3 namespaces
    → these hours are real and paid for; the per-tenant split
      does not exist in the data. Convert to MIG to attribute.
```

**That bottom block is the most credible thing on the dashboard.** It is the row that
survives an audit, and it is the row that makes the MIG business case without you having to
argue for it.

## Perspectives

**Developer.** A data scientist requesting a time-sliced GPU has no visibility that their
utilisation is being merged with — or hidden behind — a co-tenant's. From inside the pod
everything is normal: `nvidia-smi` shows the whole card, their kernels run, their throughput
is what it is. The attribution hole is invisible at the layer where the work happens, which
means the first time they encounter it is when a chargeback report says their team used
either far more or far less than they expected. Tell them up front, in the platform docs,
that time-sliced GPUs are not individually metered.

**Operator / FinOps.** The "who holds versus who uses" divergence is the single query that
turns a vague "utilisation is low" into a named conversation with a budget owner. That is
why this lesson is short on new mechanism and long on rigour: it is an aggregation over
05.3's honest metrics and 04's join, but it is the aggregation that actually gets read in a
budget meeting, and therefore the one where a subtle integral bug or an unqualified shared
bucket costs you your credibility rather than a dashboard panel.

**Hardware / isolation.** MIG's partitioning is what makes attribution *possible at all* —
a distinct set of SMs and a distinct GPU-instance UUID means distinct counters and a
`nvmlGpmMigSampleGet()` stream per instance. Time-slicing and MPS share one physical
counter domain, so the finest granularity that physically exists is the card. This is
silicon, not a smarter-exporter problem, and it will not be fixed by a future release.

**Economics.** Vendors selling GPU-sharing products acknowledge the attribution gap in their
own marketing material — a company whose revenue depends on people adopting sharing has
every incentive to *not* mention that it destroys per-tenant metering, and they mention it
anyway. Treat that as strong evidence the limitation is real. The commercial framing that
follows: time-slicing raises fleet utilisation and lowers per-tenant accountability, and the
two move in opposite directions. If your organisation charges back, you are choosing between
a higher utilisation number you cannot bill and a lower one you can.

## Real-world use cases

- **`NVIDIA/k8s-device-plugin` — `internal/rm/devices.go`, `NewAnnotatedID()`.** Time-sliced
  replicas are advertised as `fmt.Sprintf("%s::%d", id, replica)`. **What it shows:** the
  co-tenant identity *does* exist in pod-resources — the common claim that "pod-resources
  returns the same UUID for every sharing pod" is **wrong**, and the correction matters
  because it locates the information loss precisely one layer downstream, in the exporter's
  `map[string]PodInfo`. That is the difference between "the platform cannot know" and "this
  data structure discards it", and only the second is potentially fixable.

- **`NVIDIA/dcgm-exporter` — `toDeviceToPod()` vs `toDeviceToSharingPods()`.** The default
  path builds `map[string]PodInfo` (one owner per device, last writer wins); the
  `KUBERNETES_VIRTUAL_GPUS=true` path builds `map[string][]PodInfo` and emits one copy of the
  device metric per sharing pod. **What it shows:** the exporter offers exactly two
  behaviours under sharing, and *both* are wrong for chargeback in different directions —
  one under-counts three tenants to zero, the other multiplies the card's work by the tenant
  count. Neither is a bug; there is no correct third option available from device-scope
  counters.

- **`NVIDIA/dcgm-exporter` — `isPerProcessMetric()` and `internal/pkg/nvmlprovider/provider.go`.**
  Per-process attribution exists, backed by `nvmlDeviceGetProcessUtilization()` (which
  returns `nvmlProcessUtilizationSample_t` with `pid`, `smUtil`, `memUtil`, `encUtil`,
  `decUtil`) and `GetComputeRunningProcesses()`, mapped PID → pod. But
  `isPerProcessMetric()` returns true for exactly two field names:
  `DCGM_FI_DEV_GPU_UTIL` and `DCGM_FI_DEV_FB_USED`. **What it shows:** the module's central
  irony, in code. The presence metric can be split per tenant because NVML has a process
  dimension; the honest breadth metric cannot, because hardware counters do not. If someone
  proposes per-pod GPU billing on a time-sliced fleet, this is the file that ends the
  discussion.

- **ScaleOps — "GPU Sharing in Kubernetes: MIG vs MPS vs Time-Slicing."** A GPU cost
  optimisation vendor states that under time-slicing multiple pods share one physical GPU
  while many DCGM metrics remain device-level, so duplicated labels should not be read as
  exact per-pod usage. **What it shows:** independent industry corroboration from a party
  with an incentive to downplay it. Useful in an internal design review, where "the course I
  did says so" carries less weight than "the vendor selling us this says so".

- **NVIDIA GPU Operator — "Time-Slicing GPUs in Kubernetes."** NVIDIA's own documentation
  frames time-slicing versus MIG as a trade-off between isolation and share count, and is
  explicit that time-slicing provides no memory or fault isolation between replicas.
  **What it shows:** the vendor that ships both mechanisms frames the choice the same way
  this lesson does — the attribution consequence is simply the metering-shaped view of the
  isolation property they already document.

- **Red Hat — "Sharing is caring: how to make the most of your GPUs (part 1 —
  time-slicing)."** A major platform vendor's explainer aimed at exactly this audience.
  **What it shows:** the trade-off framing is industry-standard, so presenting the
  unattributable bucket as a normal, expected artifact of a known trade-off — rather than as
  a defect in your monitoring — is the accurate and the persuasive framing.

## Worked example

**Fleet slice, 24-hour window.** 144 GPUs across three namespaces on whole-GPU and MIG
allocation, plus two H100s configured for 4-way time-slicing serving a shared notebook
namespace. Rate: **$2.50/GPU-hour** — the mid-2026 H100 on-demand band, quoted as a dated
directional snapshot, not a constant. Read it as an order of magnitude.

**Step 1 — establish the label spelling.** Before anything:

```promql
topk(1, DCGM_FI_PROF_SM_ACTIVE)
# → DCGM_FI_PROF_SM_ACTIVE{ ..., exported_namespace="team-vision",
#     exported_pod="train-vit-9f2c", namespace="gpu-operator", ... }
```

`exported_namespace` it is. `namespace` is the exporter's own pod. Had you grouped by
`namespace` you would have produced one row for the whole fleet and never noticed.

**Step 2 — the attributable aggregates.**

```promql
avg_over_time(namespace:gpu_allocated:count[24h]) * 24
avg_over_time(namespace:gpu_sm_active:sum[24h])   * 24
```

| namespace | allocated GPUs | allocated GPU-h | SM-active GPU-h | efficiency | gap GPU-h | gap $ |
|---|---|---|---|---|---|---|
| `team-vision` | 80 | 1,920 | 96 | **5.0%** | 1,824 | $4,560 |
| `team-nlp` | 24 | 576 | 430 | **74.7%** | 146 | $365 |
| `team-research` | 40 | 960 | 120 | **12.5%** | 840 | $2,100 |
| **subtotal** | **144** | **3,456** | **646** | **18.7%** | **2,810** | **$7,025** |

**Step 3 — the divergence, which is the story.** Rank twice:

```
  by ALLOCATED GPUs            by SM-ACTIVE GPU-HOURS
  ─────────────────────        ──────────────────────
  1. team-vision    80         1. team-nlp        430
  2. team-research  40         2. team-research   120
  3. team-nlp       24         3. team-vision      96
     ▲                            ▲
     └── holds 3.3× more ─────────┘  …and does 4.5× less work.
```

**The lists invert at the top.** The team holding 80 GPUs converts them into 96 GPU-hours of
SM work; a team with 24 GPUs out-*uses* them 4.5×. That inversion is the headline, and it is
invisible on any dashboard built on field 203.

To prove it is invisible, run the lie for comparison:

```promql
avg by (exported_namespace) (DCGM_FI_DEV_GPU_UTIL) / 100
```

| namespace | `GPU_UTIL`/100 (the lie) | `SM_ACTIVE` (the truth) | ratio |
|---|---|---|---|
| `team-vision` | **0.91** | 0.050 | **18×** |
| `team-nlp` | 0.97 | 0.747 | 1.3× |
| `team-research` | **0.88** | 0.125 | **7×** |

`team-vision` runs batch-1 inference and idle notebooks: kernels are always resident, so the
presence metric reads 91%, while 5% of SM-cycles have a warp. **This table is the blog
post's second exhibit** — the first is the single-GPU screenshot from 05.1, and this is the
fleet-level version with names and dollars attached.

**Step 4 — the unattributable bucket, done honestly.** The notebook namespace runs 8 pods
across 2 time-sliced H100s.

```promql
# The shared cards, and what they did.
avg_over_time(gpu_unattributable_allocated:count[24h]) * 24    → 48    GPU-hours allocated
avg_over_time(gpu_unattributable:sum[24h])             * 24    → 34.6  GPU-hours SM-active
```

Note that 34.6 / 48 = **72% efficiency** — the shared cards are the *best-utilised hardware
in the fleet*, which is exactly what time-slicing is for and exactly why teams want it. And
you still cannot say which of the 8 pods produced it. Report it as:

```
  UNATTRIBUTABLE (2× H100, 4-way time-sliced, 8 tenant pods, 3 namespaces)
      allocated GPU-hours ......... 48.0
      SM-active GPU-hours ......... 34.6      (72% — the best in the fleet)
      gap ......................... 13.4      ≈ $34
      per-tenant split ............ NOT AVAILABLE
          DCGM reports one SM_ACTIVE series per physical GPU. Time-slicing
          context-switches whole-device contexts; no per-tenant counters exist.
          Converting these 2 cards to MIG (up to 7 instances each) would restore
          per-tenant attribution at the cost of 8 → 7 tenants per card and hard
          memory partitioning.
```

**Step 5 — what you do about it.** The fleet total: 3,504 allocated GPU-hours (3,456 + 48),
680.6 SM-active, a gap of 2,823 GPU-hours ≈ **$7,059/day ≈ $2.58M/year** at $2.50/GPU-hour.
Do not present that as recoverable — present it as the ceiling and then decompose it:

```promql
# How much of the gap is GPUs that did NOTHING, rather than GPUs that
# were merely inefficient? This is the recoverable part.
avg_over_time(namespace:gpu_idle_allocated:count[24h]) * 24
```

| namespace | idle-allocated GPU-h | recoverable $ /day | action |
|---|---|---|---|
| `team-vision` | 1,392 | $3,480 | reclaim: 58 GPUs held with `SM_ACTIVE < 0.05` and `FB_USED < 2 GB` for the whole window — almost certainly abandoned notebooks |
| `team-research` | 648 | $1,620 | reclaim: idle training pods between runs; add a TTL controller |
| `team-nlp` | 12 | $30 | nothing to do — 75% efficient, this is normal launch-gap overhead |

**That table is the CFO answer**: "$3.5k/day of GPUs in one namespace are allocated, holding
under 2 GB of memory, and doing under 5% SM work for 24 hours straight. Not slow —
*untouched*. The dashboard everyone was looking at showed that namespace at 91%."

**Step 6 — the honesty check before you send it.** Run the invariants:

```promql
sum(avg_over_time(namespace:gpu_allocated:count[24h])) * 24  → 3,456   ≤ 144 × 24 = 3,456 ✓
avg_over_time(namespace:gpu_sm_active:sum[24h])
  <= avg_over_time(namespace:gpu_allocated:count[24h])                                    ✓
count(DCGM_FI_PROF_SM_ACTIVE{exported_namespace=""})         → 0       (join healthy)     ✓
```

If the second ever fails, you are double-counting MIG instances against their parent GPU —
filter with `GPU_I_ID` before you publish anything.

## Practice

Feeds ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md).
Steps 1–6 work against the
[fake GPU fleet lab](../../04-gpu-on-kubernetes/practice/fake-gpu-fleet/README.md); steps 7–8
want a real time-sliced or MIG node.

1. **Establish the label spelling first.** Run `topk(1, DCGM_FI_PROF_SM_ACTIVE)` and read the
   label set. Write down whether you have `namespace` or `exported_namespace`, and check
   `honorLabels` in your ServiceMonitor to confirm you understand *why*. Every query below
   depends on this.

2. **Verify the join is alive.** `count(DCGM_FI_PROF_SM_ACTIVE{exported_namespace=""})`
   should equal your genuinely-unallocated GPU count. If it is your whole fleet, the
   pod-resources path is broken — check the socket mount, the resource-name filter, and
   whether the cluster uses DRA (in which case you need `KUBERNETES_ENABLE_DRA=true`).

3. **Write the recording rules** from §5 — `namespace:gpu_sm_active:sum`,
   `namespace:gpu_allocated:count`, `namespace:gpu_idle_allocated:count`, `gpu_unowned:count`
   — at `interval: 30s` matching your scrape.

4. **Prove the integral matters.** Compute SM-active GPU-hours over 24 h both ways — the
   naïve `sum(avg_over_time(SM_ACTIVE[24h])) * 24` and the recording-rule form — on a window
   during which you deliberately scale a GPU workload up and back down. Record the
   difference. On a stable fleet they will agree; the point is to see them disagree.

5. **Build the divergence.** Rank namespaces by allocated GPU-hours and separately by
   SM-active GPU-hours over the same window. Confirm the orderings differ. Put both rankings
   in one Grafana table panel side by side — the visual inversion is what makes the argument
   land.

6. **Build the CFO panel.** Allocated GPU-hours, SM-active GPU-hours, and the gap in dollars,
   per namespace. Inject the rate as a Grafana variable or a recording rule; **do not commit
   a real rate** (the repo `.gitignore` guards this).

7. **Reproduce the collapse.** On a node with time-slicing configured
   (`nvidia.com/gpu` capacity > physical GPU count), schedule 4 pods from 3 different
   namespaces onto one card. Then: (a) confirm exactly one `SM_ACTIVE` series exists for that
   UUID; (b) sample `exported_pod` on that series every 30 s for 10 minutes and record
   whether it changes; (c) turn on `KUBERNETES_VIRTUAL_GPUS=true`, confirm four series appear
   with identical values and distinct `vgpu` labels, and confirm that summing them exceeds
   1.0 GPU-equivalents. **Screenshot (c)** — four identical numbers under four pod names is a
   very persuasive image.

8. **Confirm the per-process exception.** On the same node, query
   `DCGM_FI_DEV_GPU_UTIL` and `DCGM_FI_DEV_FB_USED` with virtual GPUs enabled and observe
   that those two fields *do* carry different values per pod, while `SM_ACTIVE` does not.
   This is the sharpest possible demonstration of the module's thesis and it takes one query.

9. **Ship the unattributable bucket.** Implement `gpu:shared:flag` and the attributable /
   unattributable split, and render the bucket as an explicit row on the dashboard with the
   pod count and namespace count, not as a footnote.

**Acceptance:** a saved PromQL pack (or recording-rule group) that outputs, per namespace
over a window, **allocated GPU-hours, SM-active GPU-hours, their ratio, and the gap in
dollars**, ranked so the holder-versus-user divergence is visible — **plus** an explicit
`unattributable_gpu_hours` figure with the count of physical GPUs and tenant pods behind it.
Include the two invariant checks from step 6 of the worked example as panel-level alerts, so
the dashboard tells you when it is lying. This is the deliverable's core query and Module
11's cost-operator input.

## Common pitfalls

1. **Grouping by `namespace` instead of `exported_namespace`.** **Symptom:** one row,
   labelled with your monitoring namespace, holding the fleet-wide mean — a query that runs,
   returns data, and is completely wrong. **Mechanism:** `honor_labels: false` (the default in
   Prometheus, in the standalone chart, and in the GPU Operator) renames the exporter's
   colliding labels to `exported_*` and attaches Prometheus' own service-discovery values.
   **Fix:** read the label set off a real sample before writing the first query.

2. **`sum(avg_over_time(SM_ACTIVE[24h])) * hours` as the GPU-hours integral.**
   **Symptom:** GPU-hours that do not reconcile against fleet size, in an unpredictable
   direction. **Mechanism:** `avg_over_time` divides by the number of samples each series
   actually has, so a GPU present for one hour of a 24-hour window is weighted as if present
   throughout; and the outer `sum` only sees series existing at evaluation time. **Fix:**
   aggregate first, average over time second — via a subquery, or better, a recording rule.

3. **Believing a PromQL trick can recover per-tenant attribution under time-slicing.**
   **Symptom:** hours spent on `label_replace`, `group_left`, and increasingly baroque joins.
   **Mechanism:** there is one `SM_ACTIVE` value for the physical card because there is one
   set of counters; the split was never measured. No query recovers data that was not
   collected.

4. **Reading `KUBERNETES_VIRTUAL_GPUS=true` duplicated series as per-pod usage.**
   **Symptom:** namespace efficiency above 100%, or fleet SM-active GPU-hours exceeding fleet
   allocated GPU-hours. **Mechanism:** `Process()` deep-copies the *device-level* metric once
   per sharing pod; four pods on one card produce four series carrying the identical value.
   Summing multiplies the card's work by its tenant count. **Fix:** the second invariant check
   in §5 catches this on the first run.

5. **Silently crediting a shared card's work to whichever pod landed last.**
   **Symptom:** a namespace's usage jumps between scrapes for no workload-related reason; a
   team disputes a chargeback and turns out to be right. **Mechanism:** `map[string]PodInfo`,
   overwritten once per replica, in unstable iteration order. **Fix:** route those GPU-hours
   to an explicit unattributable bucket.

6. **Treating MIG and time-slicing as interchangeable "GPU sharing".** **Symptom:** a
   sharing rollout that quietly destroys the chargeback system it was justified by.
   **Mechanism:** MIG creates distinct DCGM entities (`FE_GPU_I`/`FE_GPU_CI`) with their own
   counters and their own `MIG-<uuid>`; time-slicing creates schedulable replicas of one
   entity. **Fix:** decide isolation-and-attribution versus share-count explicitly, and write
   the decision down.

7. **Double-counting MIG instances against their parent GPU.** **Symptom:** SM-active
   GPU-hours exceed allocated GPU-hours for a MIG namespace. **Mechanism:** the exporter can
   emit series at both device and instance scope; summing across both counts the same silicon
   twice. **Fix:** filter on the presence or absence of `GPU_I_ID` and be explicit about which
   scope your aggregate is at.

8. **Forgetting the DRA branch.** **Symptom:** a modern cluster where every GPU shows an
   empty `exported_namespace` and you conclude the fleet is idle. **Mechanism:** DRA
   allocations arrive in `dynamic_resources`, not `devices`, and dcgm-exporter only reads them
   with `KUBERNETES_ENABLE_DRA=true`. **Fix:** check the flag before you check anything else.

9. **Presenting the gap as recoverable waste.** **Symptom:** you promise $2.5M and deliver
   $1.2M, and nobody believes the next number. **Mechanism:** `SM_ACTIVE` never reaches 1.0
   even on healthy work — kernel launch gaps, memory-bound phases and synchronisation are
   real. **Fix:** present the gap as a ceiling and a trend, and decompose the *recoverable*
   part with the idle-allocated query (`SM_ACTIVE < 0.05` **and** `FB_USED < threshold`).

## Self-check

- **Why can't you attribute per-pod under time-slicing, in metric terms?** *Answer:* because
  DCGM emits exactly one series per *physical* GPU, keyed by that GPU's single
  `GPU-<uuid>`, and time-slicing is context switching between whole-device contexts — no
  per-tenant hardware counters exist. Note the common formulation is subtly wrong: the device
  plugin *does* hand out distinct annotated replica IDs (`GPU-<uuid>::0`, `::1`, …), and
  pod-resources reports them faithfully. The collapse happens inside dcgm-exporter's
  `toDeviceToPod()`, which strips the `::N` suffix and writes every replica into the same key
  of a `map[string]PodInfo` — last writer wins, in unstable iteration order. So the *mapping*
  is discarded by a data structure, while the *measurement* was never per-tenant to begin
  with. Only the second is unfixable.

- **Which dcgm-exporter label joins a DCGM series to a pod, and where does it come from?**
  *Answer:* `pod` (with `namespace` and `container`), typically arriving as `exported_pod` /
  `exported_namespace` / `exported_container` after a `honor_labels: false` scrape. The
  exporter calls the kubelet **pod-resources API** over
  `unix:///var/lib/kubelet/pod-resources/kubelet.sock`, gRPC service
  `v1.PodResourcesLister/List`, once per scrape, with a 10 s dial timeout and a 16 MiB
  receive limit. The response maps each container's assigned `device_ids` to its pod name and
  namespace; the exporter keys that against the DCGM series' GPU `UUID`. Pod *labels* are not
  in the API — they come from a separate pod informer and need RBAC.

- **MIG vs time-slicing — which preserves clean per-tenant attribution, and why?**
  *Answer:* MIG. The hardware partitions SMs, L2 and memory into GPU instances, DCGM
  registers the activity fields at entity levels `DCGM_FE_GPU_CI` and `DCGM_FE_GPU_I`, and on
  Hopper NVML provides `nvmlGpmMigSampleGet(device, gpuInstanceId, …)` as a separate sample
  stream per instance. The device plugin advertises `nvidia.com/mig-<profile>` with distinct
  `MIG-<uuid>` device IDs, so pod-resources returns a unique UUID per tenant and the join is
  1:1. Time-slicing and MPS share one physical entity and one counter domain, so the finest
  granularity that exists is the card. It is a hardware-partitioning property, not a config
  choice.

- **Write the allocated-but-idle-beyond-N-minutes query from memory.** *Answer:*
  ```promql
  count by (exported_namespace) (
      avg_over_time(DCGM_FI_PROF_SM_ACTIVE{exported_namespace!=""}[15m]) < 0.05
    and on (UUID)
      DCGM_FI_DEV_FB_USED < 2048
  )
  ```
  Three parts, each load-bearing: `exported_namespace!=""` scopes it to *allocated* GPUs
  (the label is only stamped when pod-resources reported the device as assigned);
  `avg_over_time(...[15m]) < 0.05` is the sustained-idle test on the honest breadth metric,
  not on `GPU_UTIL`, which would never fire; and the `FB_USED` gate excludes a
  loaded-but-paused serving replica holding a KV-cache. Join on `UUID`, never on `gpu`, which
  is a node-local index.

- **Derive per-namespace GPU-hours correctly, and say why the obvious form is wrong.**
  *Answer:* GPU-hours is a time integral of an instantaneous aggregate, so aggregate first
  and average over time second:
  `avg_over_time(sum by (exported_namespace)(DCGM_FI_PROF_SM_ACTIVE)[24h:1m]) * 24` for work,
  and the same with `count by (…)(…{exported_namespace!=""})` for allocation. The obvious
  form, `sum by (ns)(avg_over_time(SM_ACTIVE[24h])) * 24`, averages each series over the
  samples it happens to have — so a GPU present for one hour is weighted as if present for
  all 24 — and the outer `sum` only sees series that exist at evaluation time, dropping
  released GPUs entirely. In production, replace the subquery with recording rules evaluated
  at the scrape interval: subqueries at `[24h:1m]` evaluate the inner expression 1,440 times
  per query.

- **What should the divergence query do with GPU-hours it cannot attribute?** *Answer:*
  report them in an explicit `unattributable_gpu_hours` bucket alongside the per-namespace
  breakdown, with the count of physical GPUs and the count of tenant pods behind it, never
  silently credited to whichever pod's label landed last. Identify the shared cards from
  `kube_node_status_capacity{resource="nvidia_com_gpu"}` exceeding the physical GPU count on
  that node (a configuration fact, true even when the cards are idle), or from a `vgpu` label
  if virtual GPUs are enabled. An honestly incomplete number survives an audit; a silently
  misattributed one destroys the dashboard the first time someone checks a shared node.

- **Under `KUBERNETES_VIRTUAL_GPUS=true`, four pods share a card and you see four
  `SM_ACTIVE` series. Can you now bill per pod?** *Answer:* no, and it is worse than before.
  `Process()` deep-copies the device-level metric once per sharing pod and stamps distinct
  `exported_pod` / `vgpu` labels on **the identical value**. Four pods on a card at
  `SM_ACTIVE = 0.72` produce four series each reading 0.72, so summing gives 2.88
  GPU-equivalents of work on hardware that has 1.0 to give — efficiency above 100%. The only
  fields that carry genuine per-tenant values under sharing are `DCGM_FI_DEV_GPU_UTIL` and
  `DCGM_FI_DEV_FB_USED`, because `isPerProcessMetric()` returns true only for those two and
  the exporter fills them from NVML's per-process sampling
  (`nvmlDeviceGetProcessUtilization()` → per-PID `smUtil`, `GetComputeRunningProcesses()` →
  per-PID bytes, PID mapped to pod). **The lying metric is the only one that splits per
  tenant; the honest one does not.**

- **Every GPU shows an empty `exported_namespace` on a cluster you know is busy. Rank your
  hypotheses.** *Answer:* (1) the cluster uses **DRA**, so allocations arrive in
  `dynamic_resources` rather than `devices`, and the exporter ignores them unless
  `KUBERNETES_ENABLE_DRA=true`. (2) `DCGM_EXPORTER_KUBERNETES` is not set — the whole
  enrichment path is off. (3) The pod-resources socket is not mounted, or the path differs
  (`--pod-resources-kubelet-socket`), so the gRPC dial fails; the exporter logs and continues
  without enrichment rather than failing the scrape. (4) GPUs are advertised under a custom
  resource name that is neither `nvidia.com/gpu` nor prefixed `nvidia.com/mig-` and is not in
  `NVIDIA_RESOURCE_NAMES`, so every device is skipped by the filter. (5) The pod-resources
  response exceeded the 16 MiB gRPC receive limit — look for the explicit
  *"Kubelet pod-resources response exceeded gRPC receive limit"* warning. Check the exporter
  logs first; all five paths log, and none of them fails the scrape.

## Connections & what's next

This lesson is the payoff of the chain that began in 05.1 (the lie), ran through 05.2 (what
the truth costs to collect) and 05.3 (getting it onto the wire without OOMing Prometheus):
here it becomes a per-team, dollar-shaped number. It consumes Module 04's pod-resources join
(lesson 04.3) and time-slicing work (04.7) rather than rebuilding either, and it consumes
05.2's entity model directly — `SM_ACTIVE` at `DCGM_FE_GPU_CI` and `DRAM_ACTIVE` at
`DCGM_FE_GPU_I` is *why* MIG attribution works and device-level sharing does not. The
`unattributable_gpu_hours` discipline built here resurfaces verbatim in the capstone (05.8),
where an unqualified per-namespace ranking under time-slicing would misattribute waste to the
wrong team on the first shared node someone checks, and the recording rules become the core
logic of Module 11's `gpu-cost-operator`. Next:
[05.5 — Health & errors / XID](05-health-and-xid.md) shifts from "is this GPU being used" to
"is this GPU healthy" — a different question drawing on the same DCGM series, the same
entity model, and the same discipline about what a metric can and cannot prove.

## References & further reading

**Primary sources — the join**

- Kubernetes — `k8s.io/kubelet/pkg/apis/podresources/v1/api.proto` — https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/kubelet/pkg/apis/podresources/v1/api.proto — read for: the `PodResourcesLister` service (`List`, `GetAllocatableResources`, `Get`), the `PodResources` / `ContainerResources` / `ContainerDevices` message shapes reproduced in §3, and the `DynamicResource` / `ClaimResource` / `CDIDevice` DRA branch with its driver/pool/device triplet.
- `NVIDIA/dcgm-exporter` — `internal/pkg/transformation/kubernetes.go` — https://github.com/NVIDIA/dcgm-exporter/blob/main/internal/pkg/transformation/kubernetes.go — read for: `toDeviceToPod()` (the `map[string]PodInfo` last-writer-wins collapse, and the `::` / `/vgpu` / `nvidia<i>/gi<j>` / `MIG-` device-ID parsing), `toDeviceToSharingPods()` (the `map[string][]PodInfo` duplication path), `Process()` (fail-soft enrichment and the 16 MiB gRPC limit warning), `connectToServer()` (10 s timeout, unix dialer), and `createPodInfo()` / `copyPodLabels()`.
- `NVIDIA/dcgm-exporter` — `internal/pkg/transformation/process_metrics.go` — https://github.com/NVIDIA/dcgm-exporter/blob/main/internal/pkg/transformation/process_metrics.go — read for: `isPerProcessMetric()` returning true for exactly `DCGM_FI_DEV_GPU_UTIL` and `DCGM_FI_DEV_FB_USED`, and the per-process collector's MIG and non-MIG paths. This is the single file behind §6's central irony.
- `NVIDIA/dcgm-exporter` — `internal/pkg/nvmlprovider/provider.go` — https://github.com/NVIDIA/dcgm-exporter/blob/main/internal/pkg/nvmlprovider/provider.go — read for: `GetDeviceProcessUtilization()` (`nvmlDeviceGetProcessUtilization` → per-PID `smUtil`) and `GetDeviceProcessMemory()` (`GetComputeRunningProcesses` → per-PID `UsedGpuMemory`), the two NVML calls that make per-process attribution possible for presence-grade fields only.
- `NVIDIA/dcgm-exporter` — `internal/pkg/transformation/const.go` and `internal/pkg/rendermetrics/render_metrics.go` — https://github.com/NVIDIA/dcgm-exporter/tree/main/internal/pkg — read for: the complete attribute name list (`pod`, `namespace`, `container`, `pod_uid`, `vgpu`, `hpc_job`, the seven `dra_*` names, and the `pod_name`/`pod_namespace`/`container_name` legacy spellings under `--use-old-namespace`), and the renderer-owned label set used in §4.
- `NVIDIA/k8s-device-plugin` — `internal/rm/devices.go` — https://github.com/NVIDIA/k8s-device-plugin/blob/main/internal/rm/devices.go — read for: `AnnotatedID`, `NewAnnotatedID(id, replica)` producing `"%s::%d"`, and `Split()` / `GetID()`. *Correction vs. the previous version of this lesson: pod-resources does **not** return the same UUID to every time-slicing co-tenant; it returns distinct annotated replica IDs. The collapse is downstream, in the exporter.*
- `NVIDIA/go-nvml` — `gen/nvml/nvml.h` — https://github.com/NVIDIA/go-nvml/blob/main/gen/nvml/nvml.h — read for: `nvmlProcessUtilizationSample_t` (`pid`, `timeStamp`, `smUtil`, `memUtil`, `encUtil`, `decUtil`) and `nvmlGpmMigSampleGet()`, the per-GPU-instance GPM sample stream that makes MIG attribution exact.
- `NVIDIA/DCGM` — `dcgmlib/src/dcgm_fields.cpp` — https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/src/dcgm_fields.cpp — read for: the entity levels registered per field — `SM_ACTIVE`, `SM_OCCUPANCY`, `GR_ENGINE_ACTIVE`, `PIPE_TENSOR_ACTIVE` at `DCGM_FE_GPU_CI`; `DRAM_ACTIVE` at `DCGM_FE_GPU_I` — which is the mechanism behind §7's MIG table.

**Primary sources — the query layer**

- Prometheus — querying basics and operators — https://prometheus.io/docs/prometheus/latest/querying/basics/ — read for: subquery syntax `expr[range:resolution]`, the vector-matching rules behind `on (UUID) group_left()`, and `unless` / `and` set operators used throughout §5 and §8.
- Prometheus — configuration reference — https://prometheus.io/docs/prometheus/latest/configuration/configuration/ — read for: `honor_labels` (default `false`) and the exact `exported_<label>` renaming rule that produces `exported_namespace`.

**Module 04 cross-references (the mechanism this lesson consumes)**

- Module-04 pod-resources lesson — [`../../04-gpu-on-kubernetes/lessons/03-device-plugin-recap-pod-resources.md`](../../04-gpu-on-kubernetes/lessons/03-device-plugin-recap-pod-resources.md) — the UUID → pod → namespace join this lesson consumes; read to refresh the gRPC mechanics before trusting its output.
- Module-04 time-slicing lesson — [`../../04-gpu-on-kubernetes/lessons/07-time-slicing-attribution.md`](../../04-gpu-on-kubernetes/lessons/07-time-slicing-attribution.md) — the pod-resources-level view of replica IDs, restated here in metric and PromQL terms.
- Module-04 DRA lesson — [`../../04-gpu-on-kubernetes/lessons/09-dra-driver-and-quotas.md`](../../04-gpu-on-kubernetes/lessons/09-dra-driver-and-quotas.md) — the object model behind the `dynamic_resources` branch of the pod-resources response and the `KUBERNETES_ENABLE_DRA` flag.
- Module-04 capstone — [`../../04-gpu-on-kubernetes/lessons/10-capstone-per-pod-attribution.md`](../../04-gpu-on-kubernetes/lessons/10-capstone-per-pod-attribution.md) — the full worked attribution build this lesson's PromQL sits on top of.

**Real-world engineering blogs**

- NVIDIA GPU Operator — "Time-Slicing GPUs in Kubernetes" — https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html — what it shows: the vendor's own isolation-versus-share-count framing, and the explicit statement that time-slicing provides no memory or fault isolation between replicas — the property that this lesson reads as an attribution consequence.
- ScaleOps — "GPU Sharing in Kubernetes: MIG vs MPS vs Time-Slicing" — https://scaleops.com/blog/kubernetes-gpu-sharing/ — what it shows: a GPU-cost-optimisation vendor independently stating that DCGM metrics remain device-level under time-slicing and that duplicated labels must not be read as per-pod usage.
- Red Hat — "Sharing is caring: how to make the most of your GPUs (part 1 — time-slicing)" — https://www.redhat.com/en/blog/sharing-caring-how-make-most-your-gpus-part-1-time-slicing — what it shows: a major platform vendor's explainer on the same trade-off, aimed at the same audience, useful as neutral third-party framing in a design review.

**Deeper dives**

- ScaleOps — "GPU Cost Optimization" — https://scaleops.com/blog/gpu-cost-optimization/ — go deeper on: the commercial framing of exactly the allocated-versus-used gap this lesson's query surfaces; a direct precursor to the capstone's dollar math and a useful template for how to present it to a non-engineering audience.
</content>
