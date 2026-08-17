---
lesson: "04.7"
title: "Time-slicing and the attribution trap"
module: "04"
concept: "Time-slicing and the attribution trap"
status: not-started
est_time: "10h"
prev: "06-mig-operations.md"
next: "08-mps-choosing-sharing.md"
artifacts: []
sources: 13
---

# 04.7 · Time-slicing and the attribution trap

> **Concept.** Time-slicing multiplies one GPU into N schedulable replicas that share it by taking turns — with no memory isolation, no fault isolation, and no per-pod allocation signal — so cost cannot be attributed from allocation counts and must fall back to DCGM per-pod utilisation.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Lesson 06 gave you the clean case. MIG carves a GPU into hardware partitions; each partition gets its own device UUID; DCGM reports metrics against that same UUID; the kubelet hands that same UUID to exactly one container. Four independent systems agree on one identifier, so `allocation count × $/GPU-hour` produces a number you can defend in a finance meeting.

This lesson gives you the case you will actually meet more often, because MIG needs an Ampere-or-newer datacenter GPU, a node drain, and a reconfiguration dance, while time-slicing needs one integer in one ConfigMap and works on a 2018 T4. Here that four-way agreement breaks — but **not in the way most write-ups claim**. The scheduling identity survives; the *measurement* identity does not. Getting that distinction exactly right is what separates a candidate who has read a blog post from one who has instrumented a shared GPU, and it is the mechanical core of the through-line that runs from here to the capstone: **sharing breaks attribution at the metric, not at the map.**

What you build here — the fallback attribution path plus the honesty labels that go with it — is the branch of the capstone (lesson 10) that handles every GPU on your fleet that is not MIG-partitioned.

## Why this matters

Time-slicing is the cheapest, most common and most abused way to put more than one pod on a GPU. Change `replicas: 1` to `replicas: 4`, and a node that advertised `nvidia.com/gpu: 1` advertises `nvidia.com/gpu: 4`. Pending pods schedule. Dashboard utilisation climbs. Everyone is pleased — until someone asks the question this entire course is built around: **who owes what for that GPU?**

Under time-slicing the honest answer is *you cannot compute it from allocation, and you cannot compute it from a per-GPU metric either*. Both of the two obvious signals fail, and they fail for different reasons:

- **Allocation fails because it is uniform by construction.** Four replicas are four identical tickets. Ticket count carries zero information about work done, so dividing the GPU's cost by ticket count charges an idle Jupyter notebook exactly what it charges a saturating training job.
- **The per-GPU metric fails because it is a device-wide counter.** `DCGM_FI_PROF_GR_ENGINE_ACTIVE` for a physical GPU is one number describing the whole device. When three pods share that device, three time series exist — one per pod — and **all three carry the same value.** Sum them and you report 300% of a GPU that only ever existed once.

That second failure is the one that bites in production, because it is silent. The metric is present. The pod label is present. Your PromQL evaluates. The dashboard renders. The number is simply three times too large, and nothing in the pipeline complains.

This is the module README's named interview probe, verbatim: **"attribute cost to a time-sliced GPU."** The senior answer is not "use time-slicing" or "avoid it". It is: name which signal died and why, name the fallback, state the accuracy the fallback actually buys, and — the part most candidates skip — say how you label the estimate so nobody downstream mistakes it for a measurement.

It matters twice over, because time-slicing also removes two isolation guarantees an operator quietly depends on. A neighbour can exhaust VRAM and kill you, and a fatal fault on the device takes down every co-resident pod. NVIDIA's own device-plugin documentation states this flatly: workloads granted replicas of the same GPU get no isolation, share the full GPU memory, and run "in the same fault-domain as of all the others (meaning if one workload crashes, they all do)" (NVIDIA k8s-device-plugin README, *Shared Access to GPUs*). If your cost report cannot see the neighbour, neither can your incident review — and they are the same physical fact.

## What's new here (calibration)

Module 04's README told this module to skip the device-plugin gRPC *API mechanics* (module 02), the DRA *object model* (module 02), Topology Manager internals (module 02b), and GPU/MIG *at the silicon level* (module 03). Lesson 06 covered MIG's clean attribution case; lesson 03 gave you a Go client for the pod-resources API. None of that is re-taught. What this lesson adds:

- **The hardware mechanism of a time slice** — channels, contexts, compute preemption, the scheduling granularity, and the one knob (`nvidia-smi compute-policy --set-timeslice`) that changes it. "They take turns" is not a mechanism; this is.
- **The complete sharing config schema**, every field, from the device plugin's own Go types — including the three fields most write-ups omit (`rename`, `devices`, and the real default of `failRequestsGreaterThanOne`).
- **The correction that reframes the whole problem.** Contrary to the common claim (and to the previous version of this lesson), the kubelet's pod-resources API does **not** hand every co-resident pod an identical string. The device plugin advertises *annotated* device IDs of the form `GPU-<uuid>::<replica>`, so pod-resources gives you a distinct ID per pod, all resolving to one physical UUID. The map is fine. The metric is the problem. Everything about how you build the exporter follows from getting this right.
- **The arithmetic of the error** — the exact over-report factor for N sharers, the exact under- and over-charge for each pod under a fair-share split, and a reconciliation identity you can assert in a unit test.
- **The honest fallback menu with accuracy attached** — per-PID NVML, framebuffer proration, energy proration, fair share — and what each one can and cannot support.

## Core concepts

### 1 — What a "time slice" is in silicon

Start below Kubernetes, because every property that follows is a consequence of the hardware model.

A CUDA context is the GPU-side equivalent of a process address space: page tables, a set of channels (hardware submission queues), stream state, module code. When two independent OS processes each create a context on the same GPU, the GPU cannot run both simultaneously in the default compute mode — the SM array executes work belonging to one context at a time. The driver's channel scheduler therefore multiplexes them in time: it lets context A's work run, then **preempts**, saves A's state, restores B's state, and lets B run.

Three facts about that preemption decide everything else in this lesson:

1. **It is whole-device.** The unit being switched is the context, and a context owns the whole SM array while it is resident. There is no notion of "context A gets 30% of the SMs". Under time-slicing a job that only needs 20% of the SMs still occupies 100% of the device during its slice, and the other 80% idles. (This is exactly the gap MPS closes — lesson 08.)
2. **It is preemptive and reasonably fine-grained on modern hardware.** Pascal introduced *compute preemption* at instruction-level granularity: a long-running kernel can be interrupted mid-kernel rather than having to run to completion. On pre-Pascal hardware, preemption happened at kernel boundaries, which meant one badly-written multi-second kernel could stall every other tenant for its full duration. On Pascal and later, that specific pathology is bounded by the time slice.
3. **The slice length is a driver policy, and it is tunable.** Since CUDA 11.1 / R455 drivers, `nvidia-smi compute-policy` exposes a per-GPU timeslice setting with four levels — `default`, `short`, `medium`, `long`:

```bash
# Inspect the current policy on every GPU
$ nvidia-smi compute-policy --list
GPU  0: Timeslice: Default
GPU  1: Timeslice: Default

# Longer slices: fewer context switches, more switch-amortisation,
# worse tail latency for co-residents.
$ sudo nvidia-smi compute-policy -i 0 --set-timeslice=long
```

NVIDIA publishes the four *levels* but not their millisecond values, and they are not guaranteed stable across driver branches. Do not put a number on them in a design doc; say "default/short/medium/long, tuned toward throughput or toward fairness" and measure the latency effect on your own SKU if it matters. (This is the same knob the DRA driver exposes declaratively as `timeSlicingConfig.interval` — lesson 09.)

Here is the timeline you need in your head. Two pods, one GPU, `replicas: 2`. Each pod's kernels need roughly 40% of the SM array:

```
  TIME-SLICING ON ONE GPU — WHAT THE SILICON ACTUALLY DOES
  (t = wall clock, one row per resource, ▓ = busy, · = idle)

                0ms      2ms      4ms      6ms      8ms     10ms
                |        |        |        |        |        |
  ctx A (pod-a)  ▓▓▓▓▓▓▓▓ ········ ▓▓▓▓▓▓▓▓ ········ ▓▓▓▓▓▓▓▓
  ctx B (pod-b)  ········ ▓▓▓▓▓▓▓▓ ········ ▓▓▓▓▓▓▓▓ ········
                 └──┬───┘↑└──┬───┘↑
                    │    │   │    └─ preempt: save B's context state,
                  slice  │  slice     restore A's. Cost is real but
                         │            small vs. the slice length.
                         └─ the switch. Whole-device: the SM array
                            flips ownership, it is not divided.

  SM array        ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓
   occupancy       ~40%     ~40%     ~40%     ~40%     ~40%
                  └─────────────────┬─────────────────┘
                   60% of the SM array is idle THE WHOLE TIME.
                   Time-slicing packs the time axis, never the
                   occupancy axis. (MPS packs occupancy — L8.)

  DCGM sees, for the physical GPU, over the full window:
     GR_ENGINE_ACTIVE ≈ 1.00   ("the engine had work ~always")
     SM_ACTIVE        ≈ 0.40   ("~40% of SM-cycles had a warp")
  Neither number is decomposable by pod. Both are ONE number
  for a device that TWO pods are billing against.
```

Read the last block twice. **The device-wide metric is high precisely because sharing worked.** The GPU is busy nearly all the time — that is the point of time-slicing. And that single high number is the only utilisation signal DCGM will hand you for that device. Every attribution problem in this lesson is a consequence of one number needing to be split N ways with no information about the split.

One more property of the hardware model, because it is the root of the isolation story: **contexts are switched, but device memory is not.** Framebuffer allocations persist across context switches — that is what makes the switch cheap. A pod that has `cudaMalloc`'d 70 GB holds those 70 GB during every other pod's slice too. There is no swap-out, no eviction, no per-context memory reservation. Compute is time-shared; memory is simply *co-resident*.

### 2 — How Kubernetes turns one GPU into N tickets

The kubelet learns about GPUs from a device plugin over the device-plugin gRPC API: the plugin `Register`s, then streams `ListAndWatch` responses containing a list of `Device{ID, Health}`. The kubelet stores that list, publishes its length as `Allocatable["nvidia.com/gpu"]`, and on pod admission calls `Allocate(device_ids)` with the specific IDs it picked.

Time-slicing is implemented entirely inside that plugin, in one place: **the plugin fabricates N `Device` entries per physical GPU.** From the plugin's own source (`internal/rm/device_map.go` and `internal/rm/devices.go` in NVIDIA/k8s-device-plugin):

```go
// For each physical device id selected for replication, and for each
// replica index i in [0, r.Replicas):
annotatedID := string(NewAnnotatedID(id, i))

// and:
func NewAnnotatedID(id string, replica int) AnnotatedID {
    return AnnotatedID(fmt.Sprintf("%s::%d", id, replica))
}

func (r AnnotatedID) HasAnnotations() bool {
    split := strings.SplitN(string(r), "::", 2)
    return len(split) == 2
}
```

So on a single-GPU node with `replicas: 4`, `ListAndWatch` advertises four devices whose IDs are:

```
GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763::0
GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763::1
GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763::2
GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763::3
```

The kubelet treats those four strings as four independent devices, because from its point of view they are: distinct IDs, allocated at most once each. `Allocatable` becomes 4. When a pod requests one, the kubelet picks one string, records it against that container, and passes it to `Allocate`.

Inside `Allocate`, the plugin strips the suffix before it builds the container's device list, because `/dev/nvidia*` and `NVIDIA_VISIBLE_DEVICES` need the *real* UUID (`internal/plugin/server.go`):

```go
func (plugin *nvidiaDevicePlugin) uniqueDeviceIDsFromAnnotatedDeviceIDs(ids []string) []string {
    var deviceIDs []string
    if *plugin.config.Flags.Plugin.DeviceIDStrategy == spec.DeviceIDStrategyUUID {
        deviceIDs = rm.AnnotatedIDs(ids).GetIDs()   // "GPU-8f2c…::2" -> "GPU-8f2c…"
    }
    if *plugin.config.Flags.Plugin.DeviceIDStrategy == spec.DeviceIDStrategyIndex {
        deviceIDs = plugin.rm.Devices().Subset(ids).GetIndices()
    }
    // …dedupe, return
}
```

That two-layer design — annotated IDs above the plugin, real UUIDs below it — is the whole trick, and it is where the confusion in most write-ups comes from. Draw it:

```
  ONE PHYSICAL GPU, replicas: 4 — WHO SEES WHICH IDENTIFIER

   ┌───────────────────────────────────────────────────────────────┐
   │ node.status.allocatable["nvidia.com/gpu"] = 4                 │
   │ node labels: gpu.count=1  gpu.replicas=4                      │
   │              gpu.sharing-strategy=time-slicing                │
   │              gpu.product=NVIDIA-H100-80GB-HBM3-SHARED         │
   └───────────────────────────────────────────────────────────────┘
                         ▲ length of the device list
   ┌─────────────────────┴─────────────────────────────────────────┐
   │ kubelet devicemanager        ListAndWatch → 4 Device{ID}      │
   │   GPU-8f2c…::0  GPU-8f2c…::1  GPU-8f2c…::2  GPU-8f2c…::3      │
   │                                                               │
   │   podDevices:  tenants/infer-a/ctr → ["GPU-8f2c…::0"]         │
   │                tenants/infer-b/ctr → ["GPU-8f2c…::1"]         │
   │                                     └── DISTINCT per pod.     │
   │                                         pod-resources API     │
   │                                         reports exactly this. │
   └─────────────────────┬─────────────────────────────────────────┘
                         │ Allocate(["GPU-8f2c…::1"])
   ┌─────────────────────▼─────────────────────────────────────────┐
   │ nvidia-device-plugin          strip "::N"                     │
   │   → NVIDIA_VISIBLE_DEVICES=GPU-8f2c1d90-…-5da9abb51763        │
   │   → CDI device / /dev/nvidia0                                 │
   └─────────────────────┬─────────────────────────────────────────┘
                         │
   ┌─────────────────────▼─────────────────────────────────────────┐
   │ THE SILICON — one GPU, one UUID, one framebuffer,             │
   │ one fault domain. DCGM keys every metric HERE.                │
   │   DCGM_FI_PROF_GR_ENGINE_ACTIVE{UUID="GPU-8f2c…"}  = 0.97     │
   │                                  └── ONE series. ONE value.   │
   └───────────────────────────────────────────────────────────────┘

   The fan-out is between the last two boxes:
       4 scheduling identities  →  1 measurement identity.
   Attribution dies exactly there, and nowhere else.
```

**State it precisely, because this is the sentence the interview turns on:** time-slicing preserves a per-pod *scheduling* identity (the annotated device ID, visible in pod-resources) and destroys the per-pod *measurement* identity (there is one device-level counter for all sharers). You know exactly which pods are contending. You do not know their shares.

Corroborating evidence that the annotated ID really does surface through pod-resources: NVIDIA's own `dcgm-exporter` contains a `stripVGPUSuffix` helper that removes a `::N` suffix from device IDs read off the pod-resources socket, and a `getSharedGPU` helper that records the replica index on the pod info (`internal/pkg/transformation/kubernetes.go`). Code that strips a suffix exists because the suffix arrives. And `dcgm-exporter` issue #151 ("no metrics labels about pod namespace/name when Pod uses time slicing GPU") is the bug you get when that stripping is *absent*: DCGM reports `GPU-8f2c…`, pod-resources reports `GPU-8f2c…::1`, the lookup misses, and the metric ships with empty pod labels.

### 3 — The sharing config, every field

With the GPU Operator (the standard path) you author a ConfigMap whose keys are named configs and whose values are device-plugin config documents. Here is a complete one, with every field the schema accepts, annotated. The field set is taken from the plugin's Go types (`api/config/v1/sharing.go`, `api/config/v1/replicas.go`), not from a tutorial:

```yaml
# time-slicing-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: gpu-operator
data:
  # ── config name "any": the default the plugin applies cluster-wide ──
  any: |-
    version: v1
    flags:
      migStrategy: none          # none | single | mixed. Time-slicing is
                                 # supported for nvidia.com/gpu and, under
                                 # migStrategy: mixed, for the per-profile
                                 # names like nvidia.com/mig-1g.10gb.
      failOnInitError: true
      plugin:
        deviceIDStrategy: uuid   # uuid | index. Governs what goes into
                                 # NVIDIA_VISIBLE_DEVICES after "::N" is
                                 # stripped. Keep uuid — your attribution
                                 # join needs stable UUIDs, not indices.
        deviceListStrategy: envvar   # envvar | volume-mounts | cdi-annotations
        passDeviceSpecs: false
    sharing:
      timeSlicing:
        # false (the DEFAULT, kept for backwards compatibility) lets a pod
        # ask for `nvidia.com/gpu: 2` and receive two *replicas* — possibly
        # two replicas of the SAME card — while believing it got two GPUs.
        # Set it true. The plugin then rejects the request at Allocate.
        failRequestsGreaterThanOne: true

        # false: the resource keeps its name (nvidia.com/gpu) and the node's
        #        product label gains a "-SHARED" suffix so nodeSelectors can
        #        target shared vs. exclusive pools.
        # true:  the resource is advertised as nvidia.com/gpu.shared instead,
        #        which is self-documenting in the pod spec and makes quota
        #        (lesson 09) able to fence shared and exclusive separately.
        renameByDefault: false

        resources:
          - name: nvidia.com/gpu
            # rename: nvidia.com/gpu.timesliced
            #   per-resource override of the advertised name; takes
            #   precedence over renameByDefault for this entry.
            devices: all       # "all" | <int count> | [list of refs]
                               # Which physical devices get replicated.
                               # Refs may be a GPU index ("0"), a MIG index
                               # ("0:1"), a GPU UUID ("GPU-b102…"), or a MIG
                               # UUID. Defaults to "all" if omitted — this is
                               # how you replicate only the inference cards
                               # on a mixed node.
            replicas: 4        # must be >= 2. Advertised count becomes
                               # replicas × (number of matched devices).
```

Two of those fields are worth dwelling on.

**`failRequestsGreaterThanOne` defaults to `false`, and that default is a trap.** The plugin's own README recommends setting it to `true` and explains why: "requesting more than one shared GPU does not imply that you will get guaranteed access to a proportional amount of compute power. It only implies that you will get access to a GPU that is shared by other clients". With the flag off, a pod that requests 2 gets 2 tickets that may point at the same card, and it will run at roughly half the speed its author expects with no error anywhere. With the flag on, admission fails loudly:

```
$ kubectl describe pod gpu-pod
...
Events:
  Type     Reason                    Age   From      Message
  ----     ------                    ----  ----      -------
  Warning  UnexpectedAdmissionError  13s   kubelet   Allocate failed due to rpc
    error: code = Unknown desc = request for 'nvidia.com/gpu: 2' too large:
    maximum request size for shared resources is 1, which is unexpected
```

Note the failure shape: `UnexpectedAdmissionError`. The plugin's README is explicit that such a pod "will fail with an `UnexpectedAdmissionError` and need to be manually deleted, updated, and redeployed" — it does not reschedule itself. Your controllers and alerting need to know that reason string.

**`devices` is how you avoid slicing your training cards.** On an 8-GPU node where GPUs 6 and 7 serve low-QPS inference, `devices: ["6", "7"]` with `replicas: 4` advertises `2×4 + 6×1 = 14` — no, it does not: the entry only *replicates* the matched devices, so you get 8 physical devices where two of them contribute 4 entries each → `6 + 8 = 14` advertised. Verify that arithmetic against `kubectl describe node` on your own cluster before you write it into a capacity model; the exact interaction with `migStrategy` is worth confirming empirically.

Apply it and point the operator at it:

```bash
kubectl create -f time-slicing-config.yaml

# Cluster-wide default:
kubectl patch clusterpolicy/cluster-policy \
  -n gpu-operator --type merge \
  -p '{"spec":{"devicePlugin":{"config":{"name":"time-slicing-config","default":"any"}}}}'
```

The path `spec.devicePlugin.config.{name,default}` is the real ClusterPolicy CRD schema — `name` is "ConfigMap name for NVIDIA Device Plugin config including shared config between plugin and GFD", `default` is "Default config name within the ConfigMap" (verified against `nvidia.com_clusterpolicies.yaml` in NVIDIA/gpu-operator v26.3.3).

Per-node opt-in, so a labelled inference pool is sliced while training nodes stay exclusive:

```bash
kubectl label node <node> nvidia.com/device-plugin.config=any --overwrite
```

`nvidia.com/device-plugin.config` is a documented, *read* label — the plugin watches it and applies the named config on that node. That is the mechanism behind heterogeneous sharing policy in one cluster.

After the plugin restarts, capacity multiplies:

```bash
$ kubectl describe node <node> | grep -A4 Allocatable
Allocatable:
  cpu:             30
  memory:          230192Mi
  nvidia.com/gpu:  4          # was 1 — ONE physical GPU, four replicas
```

And the labels tell the whole story in four lines:

```bash
$ kubectl get node <node> -o json \
    | jq '.metadata.labels | with_entries(select(.key|test("nvidia.com")))'
{
  "nvidia.com/gpu.count": "1",
  "nvidia.com/gpu.replicas": "4",
  "nvidia.com/gpu.product": "NVIDIA-H100-80GB-HBM3-SHARED",
  "nvidia.com/gpu.sharing-strategy": "time-slicing",
  "nvidia.com/mig.capable": "true",
  "nvidia.com/mps.capable": "false"
}
```

`gpu.count: 1` next to `nvidia.com/gpu: 4` allocatable is the finding, in two numbers: **one device, four tickets.** The `-SHARED` product suffix is applied only when `renameByDefault=false` (the plugin appends it precisely so a `nodeSelector` can attract or avoid shared cards); with `renameByDefault=true` the suffix is dropped, because `nvidia.com/gpu.shared` in the pod spec already encodes the fact. Your cost operator should branch on `nvidia.com/gpu.sharing-strategy`, which is the label that exists for exactly this purpose — its documented values are `none`, `time-slicing`, and `mps`.

### 4 — Property one: no memory isolation, and why the symptom is confusing

Time-slicing gives every client the full framebuffer. There is no VRAM cap, no MPS-style pinned-memory limit, no MIG wall. Section 1 explained the mechanism: context switching swaps compute state, not allocations. So the framebuffer is a first-come, uncapped free-for-all across all N sharers.

The failure mode is a neighbour OOM, and its *shape* is what trips up incident tooling:

```
  NEIGHBOUR OOM UNDER TIME-SLICING — WHO KILLS WHOM

  t0   pod-a starts, allocates 74 GiB on an 80 GiB card
       nvidia-smi: 74000MiB / 81559MiB used     (one FB, no partitions)

  t1   pod-b starts, model load calls cudaMalloc(8 GiB)
       free FB ≈ 6.4 GiB  →  cudaMalloc returns cudaErrorMemoryAllocation

  t2   framework raises:
         torch.OutOfMemoryError: CUDA out of memory. Tried to allocate
         8.00 GiB. GPU 0 has a total capacity of 79.65 GiB of which
         6.41 GiB is free.
       process exits non-zero → container restarts → CrashLoopBackOff

  WHAT KUBERNETES RECORDS
    kubectl get pod pod-b
      NAME    READY   STATUS             RESTARTS
      pod-b   0/1     CrashLoopBackOff   3

    kubectl get pod pod-b -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
      Error                    ← NOT "OOMKilled"

  WHY: the kubelet's OOMKilled reason is set from the container
  runtime's report of a *host* memory-cgroup kill. VRAM exhaustion
  never touches memory.max on any cgroup, so no cgroup OOM event
  fires and the reason string stays "Error" with a non-zero exit code.

  CONSEQUENCE: every alert of the form
     kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}
  is BLIND to the single most common time-slicing incident.
```

The detector you actually need is a second path: non-zero exit **plus** a log match on `CUDA out of memory` / `cudaErrorMemoryAllocation`, correlated with `DCGM_FI_DEV_FB_USED` on the same GPU being near capacity at the time of death. Both halves matter — the log match alone fires on genuine single-tenant OOM (your own model is too big), and the FB-used correlation is what tells you it was a neighbour.

### 5 — Property two: no fault isolation

The blast radius under time-slicing is the physical GPU. There is no boundary to contain a fault to one replica, because a replica is not a hardware object — it is an integer in a plugin's device list.

Concretely, these events take out every co-resident pod:

| Event | Xid (typical) | Effect on co-residents |
|---|---|---|
| Uncorrectable (double-bit) ECC error | 48 | Device may be reset; all contexts lost |
| Illegal address / MMU fault in one client | 13, 31 | Faulting context dies; device can be left degraded |
| GPU has fallen off the bus / hardware fault | 79 | All contexts on the device die |
| Channel/robust-channel error, driver reset | 43, 45 | All contexts on the device die |
| Preemptive cleanup / GPU reset by driver | 74 | All contexts on the device die |

Xid numbers are stable identifiers from NVIDIA's Xid error documentation, but which Xids are *fatal to the device* versus *fatal only to the offending context* depends on driver branch and GPU generation. Do not memorise a fatal/non-fatal split; memorise the structural fact — **the fault domain is the device, so any device-fatal event is an N-pod incident** — and read `DCGM_FI_DEV_XID_ERRORS` plus `dmesg` on your own fleet to learn which Xids actually escalate there.

This is why time-slicing is for **cooperative, trusted, bursty** workloads: dev notebooks, batch experiments, low-QPS inference of your own services. It is never for hostile or arms-length multi-tenancy, and it is a poor fit for anything carrying an availability SLO, because you have just multiplied that SLO's exposure by N.

### 6 — The attribution hole, stated mechanically

Now put the two identity layers together with the metric pipeline and read off exactly what breaks.

**What the pod-resources API gives you.** The kubelet serves `v1.PodResourcesLister` on a Unix socket at `/var/lib/kubelet/pod-resources/kubelet.sock`. The relevant messages, from the proto (`kubernetes/kubelet/pkg/apis/podresources/v1/api.proto`):

```protobuf
service PodResourcesLister {
    rpc List(ListPodResourcesRequest) returns (ListPodResourcesResponse) {}
    rpc GetAllocatableResources(AllocatableResourcesRequest) returns (AllocatableResourcesResponse) {}
    rpc Get(GetPodResourcesRequest) returns (GetPodResourcesResponse) {}
}

message PodResources {
    string name                        = 1;
    string namespace                   = 2;
    repeated ContainerResources containers = 3;
    // …cpu_ids, memory
}

message ContainerResources {
    string name                        = 1;
    repeated ContainerDevices devices  = 2;
    // …cpu_ids, memory
    repeated DynamicResource dynamic_resources = 5;   // DRA — lesson 09
}

message ContainerDevices {
    string resource_name               = 1;   // "nvidia.com/gpu"
    repeated string device_ids         = 2;   // ← the annotated IDs
    TopologyInfo topology              = 3;
}
```

Query it for two pods on the sliced GPU:

```bash
$ grpcurl -plaintext -unix \
    /var/lib/kubelet/pod-resources/kubelet.sock \
    v1.PodResourcesLister/List | jq -c '.podResources[]
      | select(.namespace=="tenants")
      | {pod: .name, dev: [.containers[].devices[]?.deviceIds[]?]}'
{"pod":"infer-a","dev":["GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763::0"]}
{"pod":"infer-b","dev":["GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763::1"]}
```

Two *different* strings. Strip `::N` from both and they collapse to one physical UUID. That gives you, for free, the two things a naive account claims you cannot have:

- the **set** of pods contending for `GPU-8f2c…` (the denominator of any split), and
- a **replica index** per pod, if you ever want it for debugging.

**What DCGM gives you.** `dcgm-exporter` polls DCGM for fields listed in a counters CSV and emits one Prometheus series per (field, device). For a non-MIG GPU the device is the physical card. It then decorates each series with pod identity looked up from the same pod-resources socket — `--kubernetes` / `DCGM_EXPORTER_KUBERNETES=true`, socket path from `--pod-resources-kubelet-socket` (default `/var/lib/kubelet/pod-resources/kubelet.sock`), ID matching mode from `--kubernetes-gpu-id-type` (default `GPUUID`). Under sharing you must additionally set `--kubernetes-virtual-gpus` / `KUBERNETES_VIRTUAL_GPUS=true`, which is what enables the `::N` handling.

And here is the exact output shape that shape-shifts your cost model:

```
# HELP DCGM_FI_PROF_GR_ENGINE_ACTIVE Ratio of time the graphics engine is active.
DCGM_FI_PROF_GR_ENGINE_ACTIVE{gpu="0",UUID="GPU-8f2c…63",device="nvidia0",modelName="NVIDIA H100 80GB HBM3",Hostname="gpu-node-01",namespace="tenants",pod="infer-a",container="ctr"} 0.968
DCGM_FI_PROF_GR_ENGINE_ACTIVE{gpu="0",UUID="GPU-8f2c…63",device="nvidia0",modelName="NVIDIA H100 80GB HBM3",Hostname="gpu-node-01",namespace="tenants",pod="infer-b",container="ctr"} 0.968
                                                                                                                                                                                    ^^^^^
DCGM_FI_DEV_FB_USED{gpu="0",UUID="GPU-8f2c…63",namespace="tenants",pod="infer-a",container="ctr"} 74112
DCGM_FI_DEV_FB_USED{gpu="0",UUID="GPU-8f2c…63",namespace="tenants",pod="infer-b",container="ctr"} 74112
                                                                                                 ^^^^^
```

**Identical values, duplicated per pod.** Not because the exporter is broken, but because it is doing the only thing it can: DCGM measured the device, the exporter knows which pods hold replicas of that device, so it labels the same measurement once per holder. `DCGM_FI_DEV_FB_USED` = 74112 MiB is the *device's* framebuffer usage; it is ~100% attributable to `infer-a` and ~0% to `infer-b`, and the metric cannot tell you that.

This is the failure two independent, dated bug reports describe against NVIDIA's own reference exporter:

- **`NVIDIA/dcgm-exporter` #587** — *"--kubernetes-virtual-gpus exports identical values for all pods instead of per-pod utilization."* The title *is* the finding: enabling the flag gets you pod labels and duplicated device-level values, not per-pod values.
- **`NVIDIA/dcgm-exporter` #642** — *"Why are DCGM_FI_DEV_GPU_UTIL values not isolated per vGPU/Pod?"* A practitioner on Blackwell-generation hardware, time-slicing enabled via `KUBERNETES_VIRTUAL_GPUS=true`, reporting that all telemetry values — utilisation, power, temperature — are identical for every pod sharing the GPU.

There is also a matching report on NVIDIA's own developer forums, *"DCGM-Exporter: Missing Process-level Attribution for GPU Time-Slicing on Blackwell GB10"*, which frames it correctly as a **process-level attribution gap** rather than a labelling bug. (These three were located and corroborated via search this session; the GitHub HTML and forum pages could not be fetched directly through this environment's egress proxy, so treat the titles and framing as confirmed and spot-check the thread contents yourself before quoting details in an interview.)

The structural picture:

```
  THE ATTRIBUTION JOIN — WHERE IT WORKS AND WHERE IT LIES

  ┌──────────────────┐         ┌──────────────────────────────────┐
  │ kubelet          │         │ DCGM (nv-hostengine)             │
  │ pod-resources    │         │ programs HW counters per DEVICE  │
  │ /var/lib/kubelet/│         │                                  │
  │  pod-resources/  │         │ GR_ENGINE_ACTIVE(GPU-8f2c…)=0.97 │
  │  kubelet.sock    │         │ FB_USED(GPU-8f2c…)      =74112   │
  └────────┬─────────┘         └─────────────────┬────────────────┘
           │ List()                              │ field values
           ▼                                     ▼
  infer-a → GPU-8f2c…::0  ──strip "::N"──▶  GPU-8f2c…  ◀── key
  infer-b → GPU-8f2c…::1  ──strip "::N"──▶  GPU-8f2c…  ◀── key
  infer-c → GPU-8f2c…::2  ──strip "::N"──▶  GPU-8f2c…  ◀── key
                                                 │
                                         ┌───────┴────────┐
                                         │ ONE value,     │
                                         │ THREE holders  │
                                         └───────┬────────┘
                                                 ▼
   emitted series (dcgm-exporter, --kubernetes-virtual-gpus):
     …{pod="infer-a"} 0.97      ┐
     …{pod="infer-b"} 0.97      ├─ sum() = 2.91 "GPU-equivalents"
     …{pod="infer-c"} 0.97      ┘  on a node with ONE GPU.

   ✅ The MAP is correct and complete: three pods, one device.
   ❌ The METRIC is not decomposable: one number, three claimants.

   MIG, for contrast (lesson 06):
     pod-x → MIG-a1b2…  ▶ DCGM GR_ENGINE_ACTIVE(MIG-a1b2…) = 0.81
     pod-y → MIG-c3d4…  ▶ DCGM GR_ENGINE_ACTIVE(MIG-c3d4…) = 0.12
   Distinct keys all the way down. sum() means something. No fan-out.
```

### 7 — Why the naive PromQL double-counts, with the arithmetic

Here is the query a reasonable engineer writes on day one, and it is wrong:

```promql
# ❌ WRONG under any form of GPU sharing.
sum by (namespace) (
  DCGM_FI_PROF_GR_ENGINE_ACTIVE
)
* on() group_left() vector(4.00)      # $/GPU-hour
```

The `sum` fans out over the duplicated series. Let N be the number of pods sharing one device and let `u` be the device's true utilisation ratio. Then:

```
  reported "GPU-equivalents used" = N × u
  actual   "GPU-equivalents used" =     u
  ─────────────────────────────────────────
  over-report factor              = N     (exactly N, independent of u)
  relative error                  = (N−1)/1 = N−1  → +100% at N=2,
                                                     +200% at N=3,
                                                     +300% at N=4
```

That is an unusually clean and unusually dangerous error: it does not depend on load, it does not average out over time, and it scales linearly with the sharing factor you chose to improve efficiency. **The more aggressively you share to save money, the more your cost report over-states what you spent.** Cluster-wide, if your fleet runs mean sharing factor N̄, your utilisation dashboards are inflated by N̄× on exactly the nodes management is watching for efficiency wins.

The second naive fix — deduplicate, then split evenly — removes the double-count but introduces a different, load-dependent error:

```promql
# ⚠️ Correct in TOTAL, wrong PER POD. This is the "fair-share estimate".
# Denominator: how many pods hold a replica of each device, right now.
DCGM_FI_PROF_GR_ENGINE_ACTIVE
  / on(UUID) group_left()
    count by (UUID) (DCGM_FI_PROF_GR_ENGINE_ACTIVE)
```

Now `sum by (namespace)` reconciles: the shares add to `u`, once. But each pod is charged `u/N` regardless of what it did. Quantify the injustice. Let pod *i* have true share `s_i` with `Σ s_i = 1`. Fair-share assigns `1/N`. The absolute error for pod *i* is:

```
  err_i = u × (1/N − s_i)

  Worst case, one pod does everything (s_1 = 1, rest 0), N = 4, u = 0.97:
     pod 1 charged 0.2425 GPU-equivalents, owed 0.97   → −75% (under-billed)
     pods 2-4 each charged 0.2425, owed 0             → ∞% (billed for nothing)

  Best case, perfectly even load (s_i = 1/N):
     err_i = 0 for all i. Fair share is EXACT when load is even.
```

So fair share is not garbage — it is a estimator whose error is exactly the load skew, and whose total is always right. That is a defensible thing to ship **if you label it**, and it is the reason lesson 10's exporter carries an `attribution="shared-estimate"` label rather than silently emitting the number.

The third naive fix — "just use `max` instead of `sum`" — is worth naming because it comes up in review and it is subtly worse than either: `max by (namespace)` gives you the device's utilisation attributed wholly to whichever namespace has the largest value, which for identical values is arbitrary. It reconciles at the cluster level only by accident and produces non-deterministic per-namespace numbers between scrapes.

### 8 — The honest fallback menu

Four options, in decreasing order of fidelity. Pick per-fleet, not per-lesson, and record which one each series used.

**(a) Per-PID SM utilisation from NVML.** The highest-fidelity signal available on a shared, unpartitioned device. NVML exposes:

```c
typedef struct nvmlProcessUtilizationSample_st
{
    unsigned int        pid;        //!< PID of process
    unsigned long long  timeStamp;  //!< CPU Timestamp in microseconds
    unsigned int        smUtil;     //!< SM (3D/Compute) Util Value
    unsigned int        memUtil;    //!< Frame Buffer Memory Util Value
    unsigned int        encUtil;    //!< Encoder Util Value
    unsigned int        decUtil;    //!< Decoder Util Value
} nvmlProcessUtilizationSample_t;
```

retrieved via `nvmlDeviceGetProcessUtilization(device, samples, &count, lastSeenTimeStamp)`. You then resolve each `pid` to a container by reading `/proc/<pid>/cgroup` and matching the container ID, which gives you `PID → container → pod`, and split the device's utilisation in proportion to `smUtil`.

The caveats are real and you must state them: the call is a *sampling* interface (it returns samples newer than `lastSeenTimeStamp` from a bounded internal buffer, so a slow poller silently loses samples); it returns `NVML_ERROR_NOT_SUPPORTED` on some device/driver combinations; and NVML does not attribute utilisation to MIG devices on Ampere-class hardware. The GB10/Blackwell forum report above is precisely a report of this signal being unavailable where it was expected. **Probe for it at exporter startup and record the answer** — do not assume it, and do not silently fall through to something less accurate without changing the label.

A second, more robust per-process signal is available even where `smUtil` is not: `nvmlDeviceGetComputeRunningProcesses_v3` returns, per process,

```c
typedef struct nvmlProcessInfo_v2_st
{
    unsigned int        pid;
    unsigned long long  usedGpuMemory;      // bytes
    unsigned int        gpuInstanceId;      // 0xFFFFFFFF if MIG off
    unsigned int        computeInstanceId;  // 0xFFFFFFFF if MIG off
} nvmlProcessInfo_v2_t;
```

which gives you an exact per-PID framebuffer figure. That is not compute attribution, but it is a *measurement* rather than an estimate, and for memory-bound tenancy (inference servers whose cost driver is KV-cache residency) it is often the better basis. It also gives you the neighbour-OOM detector from §4 for free.

**(b) Framebuffer proration.** Split the device cost by per-PID `usedGpuMemory` share. Honest when the binding constraint is memory (which, on 80 GB cards serving 15 GB models, it frequently is) and when tenants' compute duty cycles are similar. Dishonest when one tenant holds a large idle cache.

**(c) Energy proration.** `DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION` is a device-wide counter in millijoules, so it does not split either — but if you have per-PID SM utilisation you can weight *energy* rather than *time*, which is closer to marginal cost and much less sensitive to idle-but-resident tenants. Treat this as a refinement of (a), not an alternative to it.

**(d) Fair share, labelled.** `u/N` from §7. Always available, total always reconciles, per-pod error equals the load skew. This is the floor, not the goal, and it must ship with the estimate label.

**(e) The structural answer: don't share, or share with MIG.** If a tenant needs a defensible per-tenant bill — chargeback across cost centres, an external customer, a regulated workload — the correct engineering response is to stop time-slicing that pool and use MIG (distinct UUIDs, clean 1:1, lesson 06) or exclusive allocation. "We can measure this precisely if you accept MIG's fixed partition sizes; we cannot measure it precisely if you want elastic sharing" is a real trade to put in front of a stakeholder, and it is a better answer than an unlabelled estimate.

The decision, as a table:

| Basis | Available when | Per-pod fidelity | Total reconciles | Label to emit |
|---|---|---|---|---|
| MIG per-instance UUID | MIG-capable GPU, partitioned | exact | yes | `attribution=exact` |
| Per-PID `smUtil` (NVML) | driver/SKU supports it, PID→cgroup resolvable | high | yes | `attribution=per-pid` |
| Per-PID `usedGpuMemory` | almost always | high *for memory* | yes | `attribution=per-pid-memory` |
| Fair share `u/N` | always | = load skew | yes | `attribution=shared-estimate` |
| Naive `sum` of device metric | — | wrong by N× | **no** | never ship this |

## Perspectives

**Scheduler's-eye view.** To `kube-scheduler`, `replicas: N` is a pure advertisement multiplier. `NodeResourcesFit` decrements an integer; it has no concept that four allocations point at one piece of silicon, no concept of the framebuffer as a second dimension, and no way to express "do not co-schedule these two 70 GB models". Every guardrail against oversubscribing the framebuffer has to live outside the scheduler — in admission policy, in a custom `Score` plugin (module 06), or in the discipline of who is allowed to set `replicas`. The lie lives entirely in the plugin's advertisement.

**Tenant / incident-responder view.** From inside a pod, the two failure modes are both *invisible in origin*: you die of a memory error caused by a neighbour you cannot see, or you die of a device reset triggered by a neighbour's illegal address. Neither shows up in your logs as anything other than a CUDA error. The practical consequence for on-call is that a shared-GPU fleet needs a *co-residency view* — given a pod, which other pods hold replicas of the same physical UUID — and that view is exactly the pod-resources map from §6. Building the map for cost gives you the incident tool for free; they are the same join.

**FinOps / attribution view.** Allocation-based billing under time-slicing is not merely imprecise, it is *adversarial*: cost split by ticket count rewards the tenant who requests the most tickets and punishes the frugal one. Any chargeback model whose incentive gradient points toward over-requesting is a design bug, not a rounding error. And the `sum`-over-duplicated-series error from §7 is worse than a wrong number — it is a wrong number that flatters the sharing decision, because the nodes with the highest sharing factor report the highest apparent utilisation.

**Telemetry-engineer view.** The lesson generalises past GPUs. `dcgm-exporter` #587 and #642 are both instances of one anti-pattern: **a measurement whose cardinality is lower than the cardinality of the entities you want to bill, joined onto those entities by fan-out.** The moment a metrics pipeline duplicates a value across a one-to-many join, every aggregate downstream is wrong by the fan-out degree, and nothing in Prometheus will tell you. Grep your own exporters for `group_left` over a non-unique key; this is where that class of bug lives.

**Hardware view.** Notice that the thing that makes time-slicing cheap is exactly the thing that makes it unattributable. Context switching is fast *because* it does not touch memory and does not partition the SM array — no state to migrate, no reconfiguration, no drain. MIG is attributable *because* it does partition, which is why it needs a drain and a reconfigure. There is no free lunch here: attribution fidelity and partition cost are the same axis, and every sharing mode in lessons 06–09 sits somewhere on it.

## Real-world use cases

- **`NVIDIA/dcgm-exporter` #587 — "--kubernetes-virtual-gpus exports identical values for all pods instead of per-pod utilization."** The clearest statement of this lesson's thesis in a bug tracker. The flag that makes pod labels appear on shared-GPU metrics does not make the values per-pod; it duplicates the device value across every holder. Anyone building a cost exporter who enables `KUBERNETES_VIRTUAL_GPUS=true` and then sums the result reproduces this bug inside their own pipeline. *(Located and corroborated via search this session; GitHub HTML was not directly fetchable through this environment's proxy — read the thread yourself for current status.)*
- **`NVIDIA/dcgm-exporter` #642 — "Why are DCGM_FI_DEV_GPU_UTIL values not isolated per vGPU/Pod?"** A practitioner on Blackwell-generation hardware, time-slicing via `KUBERNETES_VIRTUAL_GPUS=true`, reporting that utilisation, power and temperature are all identical across pods sharing one GPU. The value of this report is that it is *dated, on current silicon, and filed against NVIDIA's own reference exporter* — i.e. this is not a legacy limitation someone forgot to fix, it is the current state of device-level telemetry under software sharing. *(Same fetch caveat.)*
- **NVIDIA Developer Forums — "DCGM-Exporter: Missing Process-level Attribution for GPU Time-Slicing on Blackwell GB10."** The same gap named correctly: the missing capability is *process-level* attribution, not pod labelling. Useful because it points at the fallback (§8a) as the actual fix rather than asking for the impossible. *(Same fetch caveat.)*
- **`NVIDIA/dcgm-exporter` #151 — "no metrics labels about pod namespace/name when Pod uses time slicing GPU."** The earlier, opposite bug: before `::N` handling existed, time-sliced pods got *no* pod labels at all, because pod-resources reported `GPU-…::1` and DCGM reported `GPU-…`. Worth knowing because it is the diagnostic signature you will see if `--kubernetes-virtual-gpus` is not set: metrics present, pod labels empty. *(Same fetch caveat.)*
- **NVIDIA k8s-device-plugin README, *Shared Access to GPUs*.** The vendor's own statement of the isolation properties, fetched and read this session against release v0.19.2: workloads granted replicas of the same GPU are not isolated, share GPU memory, and share a fault domain such that "if one workload crashes, they all do". Also the source of the `UnexpectedAdmissionError` transcript and the `failRequestsGreaterThanOne` default and rationale. When someone claims time-slicing is "basically safe", this is the primary source that says otherwise.

## Worked example — bill three tenants on one sliced H100, four ways

Setup: one H100-80GB node, `replicas: 4`, three tenant pods co-resident for a 1-hour window. Node rate $32.00/hr for 8 GPUs → **$4.00 per physical GPU-hour**. (Use your own rate card; the arithmetic is what matters.)

Measured over the hour:

| Pod | Namespace | Per-PID mean `smUtil` | Peak FB used | What it is |
|---|---|---|---|---|
| `infer-a` | `team-a` | 74% | 61 GiB | busy 7B inference server |
| `infer-b` | `team-b` | 18% | 9 GiB | low-QPS internal service |
| `nb-c` | `team-c` | 2% | 4 GiB | Jupyter notebook, mostly idle |

Device-level, from DCGM: `DCGM_FI_PROF_GR_ENGINE_ACTIVE = 0.94`, `DCGM_FI_DEV_FB_USED = 76 288 MiB`.

**Step 0 — the number that must never ship.** Naive `sum` over the duplicated series:

```
  3 pods × 0.94 = 2.82 GPU-equivalents on ONE GPU
  → 2.82 × $4.00 = $11.28/hr charged for $4.00/hr of hardware
  over-report factor = 3 (= N), i.e. +182% relative error
```

Note it is `N=3`, the number of *co-resident pods*, not `replicas=4` — the fourth replica is unallocated and produces no series. This is a good sanity check for your exporter: the fan-out degree equals the number of *holders*, which is exactly `count by (UUID)` of the series.

**Step 1 — allocated cost (the chargeback figure).** Each pod holds one of four replicas, so each reserved a quarter of the card:

```
  allocated_share = 1 / replicas = 1/4 = 0.25
  allocated cost/hr = $4.00 × 0.25 = $1.00  per pod, for all three
  total allocated   = 3 × $1.00 = $3.00
  unallocated       = 1 × $1.00 = $1.00   ← the 4th replica nobody took
```

Two things fall out. First, allocated cost is *uniform* — that is the whole problem, restated as money: the notebook doing 2% of the work is billed identically to the inference server doing 74%. Second, the unallocated replica's $1.00 must go somewhere. Charge it to a platform/overhead bucket; do not silently socialise it across the three tenants, because that turns an idle-capacity problem (your problem, fixable by scheduling) into a tenant cost (their problem, not fixable by them).

**Step 2 — fair-share utilised cost.** Dedupe and split evenly:

```
  per-pod utilised share = u / N = 0.94 / 3 = 0.3133
  per-pod utilised cost  = $4.00 × 0.3133 = $1.2533
  total                  = 3 × $1.2533 = $3.76 = $4.00 × 0.94  ✓ reconciles
```

Correct in aggregate, and wrong for every individual tenant. Compute the error against the per-PID truth (step 3) to see how wrong.

**Step 3 — per-PID utilised cost (the honest split).**

```
  Σ smUtil = 74 + 18 + 2 = 94   (a satisfying coincidence: it lands at
                                 94, close to the device's 0.94 — see
                                 the reconciliation note below)

  share(infer-a) = 74/94 = 0.7872
  share(infer-b) = 18/94 = 0.1915
  share(nb-c)    =  2/94 = 0.0213
                          ─────
                          1.0000  ✓

  utilised cost/hr = $4.00 × u × share
    infer-a = $4.00 × 0.94 × 0.7872 = $2.960
    infer-b = $4.00 × 0.94 × 0.1915 = $0.720
    nb-c    = $4.00 × 0.94 × 0.0213 = $0.080
                                      ──────
                              total = $3.760 = $4.00 × 0.94  ✓
```

**The error fair share introduced, per tenant:**

| Pod | Fair share | Per-PID | Error | Direction |
|---|---|---|---|---|
| `infer-a` | $1.2533 | $2.960 | −$1.707 | under-billed 58% |
| `infer-b` | $1.2533 | $0.720 | +$0.533 | over-billed 74% |
| `nb-c` | $1.2533 | $0.080 | +$1.173 | over-billed **1467%** |

The notebook is billed nearly fifteen times what it used. That is the number to quote when someone argues fair share is "close enough": it is close enough *in total* and catastrophically wrong *at the tail*, and the tail is where chargeback disputes come from.

**Step 4 — the reconciliation check you must assert.** Two identities, both cheap to test, both of which catch real bugs:

```
  IDENTITY 1 — allocation conservation (must hold exactly)
    Σ_pods allocated_share(pod)  +  unallocated_share
      =  (number of replicas allocated + number free) / replicas
      =  4/4 = 1.00  per physical GPU
    In GPU-hours over a window W:
      Σ attributed GPU-hours + idle GPU-hours = (#physical GPUs) × W
    Here: (0.25×3 + 0.25) × 1 h = 1.00 GPU-hour = 1 GPU × 1 h  ✓

  IDENTITY 2 — utilisation conservation (must hold to within scrape jitter)
    Σ_pods utilised_share(pod)  =  device utilisation u
    Here: 0.7872+0.1915+0.0213 = 1.0000, × u = 0.94  ✓
    Equivalently in PromQL, this must be ~1 for every UUID:
      sum by (UUID) (gpu_util_attributed) / on(UUID) group_left()
        max by (UUID) (DCGM_FI_PROF_GR_ENGINE_ACTIVE)
```

Identity 1 failing means your share arithmetic is broken or you are double-counting replicas. Identity 2 failing by a factor of exactly N means you forgot to dedupe. Identity 2 failing by a small amount is normal — the DCGM device counter and the NVML per-PID samples are collected on different clocks over different windows — and you should pick a tolerance (5–10% is reasonable at 15 s scrape / 1 s DCGM update) and alert outside it rather than trying to make them agree exactly.

**Step 5 — the isolation footnote that must ride along with the bill.** `infer-a` at 61 GiB plus `infer-b` at 9 GiB plus `nb-c` at 4 GiB is 74 GiB of an ~79.6 GiB usable framebuffer. The fourth replica is *schedulable* — the scheduler will happily place a pod on it — and that pod will die on model load with `CUDA out of memory`, status `Error`, not `OOMKilled`. **The bill and the incident are the same physical fact.** A cost report for a time-sliced pool that does not carry the co-residency and headroom data alongside the dollar figures is only telling half the story, and it is the less actionable half.

**Step 6 — what actually ships.** Two gauges per pod, plus the labels that make them honest:

```promql
# Monthly chargeback per team (allocated — bills even at 0% utilisation):
sum by (namespace) (gpu_cost_allocated_per_hour) * 730

# Monthly waste per team (the allocated-but-unused gap):
sum by (namespace) (
  gpu_cost_allocated_per_hour - gpu_cost_utilised_per_hour
) * 730

# Trust filter: only the numbers you can defend in a dispute.
sum by (namespace) (gpu_cost_utilised_per_hour{attribution="per-pid"})

# Everything that is an estimate, so you can quantify your own exposure:
sum by (namespace) (gpu_cost_utilised_per_hour{attribution="shared-estimate"})
```

That last query is the one nobody writes and everybody needs: **what fraction of my chargeback total rests on estimates?** If the answer is 60%, the right next project is not a better dashboard, it is moving those workloads to MIG.

## Practice

Produce three artifacts for the [Per-pod GPU attribution](../practice/per-pod-attribution/README.md) deliverable: a **replica-identity proof**, a **duplicated-metric proof**, and a **neighbour-OOM demonstration**. All three feed the failure-mode log. A sub-$1/hr single-GPU rental is sufficient — you do not need an H100 for any of this, only a card the device plugin will slice.

1. **Enable slicing and capture the state.** Apply the `replicas: 4` ConfigMap from §3 with `failRequestsGreaterThanOne: true`, patch the ClusterPolicy (or label a node), and capture: `kubectl describe node | grep -A4 Allocatable`, the full `nvidia.com/*` label dump, and the device-plugin pod's log lines showing which config it loaded. Record the device-plugin version (`kubectl get ds -n gpu-operator nvidia-device-plugin-daemonset -o jsonpath='{.spec.template.spec.containers[0].image}'`) — every claim in this lesson is version-sensitive and yours may differ.

2. **Prove the replica identity survives (the correction).** Schedule two pods, each requesting `nvidia.com/gpu: 1`. From the node, query the pod-resources socket with the `grpcurl` form in §6 and capture the raw `deviceIds`. Confirm they are **distinct strings differing only in the `::N` suffix**, and that stripping the suffix collapses them to one UUID which matches `nvidia-smi -L` on the host. Write down both facts. This is the evidence that your exporter's job is a *metric* problem, not a *map* problem — and it is the finding most write-ups get wrong.

3. **Prove `failRequestsGreaterThanOne` works.** Submit a pod requesting `nvidia.com/gpu: 2`. Capture the `UnexpectedAdmissionError` event verbatim and note that the pod does not self-heal. Then flip the flag to `false`, resubmit, and confirm the pod is admitted with two replicas — possibly of the same card. Document both behaviours; the second one is the silent-halving trap.

4. **Prove the metric is duplicated (the core proof).** Run `dcgm-exporter` with `--kubernetes` and `--kubernetes-virtual-gpus=true`, then `curl` its `/metrics` and grep `DCGM_FI_PROF_GR_ENGINE_ACTIVE` and `DCGM_FI_DEV_FB_USED`. Capture the two series with **identical values and different `pod` labels**. Then run the naive `sum by (namespace)` query and show it exceeding the physical GPU count. Finally run the deduped fair-share query from §7 and show it reconciling to the device value. Three PromQL outputs; the contrast is the artifact.

5. **Establish which fallback your hardware supports.** Write a short probe (Go with `go-nvml`, or Python with `pynvml`) that calls `nvmlDeviceGetProcessUtilization` and `nvmlDeviceGetComputeRunningProcesses_v3` against the shared device with two workloads running. Record, explicitly: does `smUtil` come back per-PID with plausible distinct values, or does it return `NVML_ERROR_NOT_SUPPORTED` / a single aggregate? Does `usedGpuMemory` come back per-PID? Your answer determines which row of the §8 table your exporter can claim, and it must be written down with the driver version and SKU, because it is not portable.

6. **Demonstrate no isolation (neighbour OOM).** In pod A, allocate ~90% of the framebuffer (`torch.empty(n, dtype=torch.float16, device='cuda')` sized from `torch.cuda.mem_get_info()`, or a `cudaMalloc` loop). In pod B, attempt a normal model load. Capture pod B's `CUDA out of memory` traceback, `kubectl get pod` showing `Error`/`CrashLoopBackOff`, and the `lastState.terminated.reason` jsonpath output proving it is **not** `OOMKilled`. Then write the two-detector alert rule you would ship instead.

**Acceptance:** four pieces of evidence, committed. (a) The raw pod-resources output showing distinct `::N` device IDs collapsing to one UUID, with a one-line statement of what that means for attribution. (b) A `/metrics` excerpt showing the same DCGM value duplicated across pod labels, plus the naive-`sum` query over-reporting by exactly the number of holders. (c) A written statement of which per-PID signal your SKU/driver supports, with versions. (d) The neighbour-OOM transcript with the `reason` field proving Kubernetes did not do the killing. Plus a one-paragraph conclusion in your own words: *the scheduling identity survives sharing, the measurement identity does not; allocation count is uniform and therefore uninformative; the fallback is per-PID data where available and a labelled fair-share estimate where it is not.*

## Common pitfalls

1. **Believing all sharers get the identical device ID from pod-resources.** They do not — they get `GPU-<uuid>::0`, `::1`, `::2`. Repeating the myth costs you the co-residency map, which is the single most useful thing you *can* build under sharing: it gives you the exact denominator for any split and doubles as the on-call blast-radius view. The thing that collapses is the DCGM key, one layer down.
2. **Running `sum` over shared-GPU DCGM series.** The over-report factor is exactly the number of pods holding replicas, it does not average out, and it scales with the sharing factor — so your efficiency dashboards are most wrong on the nodes you sliced hardest to be efficient. Always divide by `count by (UUID)` of the series, or filter to a deduplicating label, before aggregating.
3. **Leaving `failRequestsGreaterThanOne` at its default.** The default is `false` for backwards compatibility, not because it is a good idea. With it off, `nvidia.com/gpu: 2` is admitted and silently yields two replicas of possibly one card — the pod runs at half speed with no error, forever. NVIDIA's own README recommends `true`.
4. **Alerting only on Kubernetes `OOMKilled` for GPU memory pressure.** VRAM exhaustion never touches a memory cgroup, so the reason stays `Error`. You need a second detector: non-zero exit + `CUDA out of memory` in logs + `DCGM_FI_DEV_FB_USED` near capacity on that GPU at that time. Without it, the most common time-slicing incident is invisible to your alerting.
5. **Assuming a `DCGM_FI_PROF_*` field is per-pod because it is "fine-grained".** `PROF` fields are finer-grained *in what they measure* (SM cycles, pipe activity) than the legacy `DCGM_FI_DEV_GPU_UTIL`, not finer-grained *in whom they attribute to*. Both are device-scoped on a non-MIG GPU. Issues #587 and #642 are exactly this confusion, filed twice.
6. **Shipping a fair-share estimate without the label.** Fair share reconciles in total and is off by the load skew per pod — up to fifteen-fold in the worked example. An unlabelled estimate becomes ground truth the moment someone screenshots the dashboard into a finance deck, and you will be defending a number you never claimed was a measurement.
7. **Treating MIG and time-slicing as interchangeable "GPU sharing" in a design doc.** They present the same `nvidia.com/gpu` resource name and have opposite attribution regimes. Naming the resource without naming the strategy — and without reading `nvidia.com/gpu.sharing-strategy` in your tooling — is how a cost model silently mixes exact and estimated numbers in one column.
8. **Forgetting the unallocated replicas.** With `replicas: 4` and three pods, one replica's worth of cost has no tenant. Socialising it across the three tenants converts your idle-capacity problem into their bill and hides the signal that would have told you to lower `replicas`. Book it to platform overhead and report it.

## Self-check

- **Why can a neighbour pod OOM you under time-slicing, and why does Kubernetes not report it as `OOMKilled`?** *Answer:* Time-slicing multiplexes *contexts* in time; it does not partition the framebuffer. Context switching deliberately leaves device memory allocations in place — that is what makes the switch cheap — so every sharer's `cudaMalloc`s are simultaneously resident with no per-client cap. A neighbour holding 74 of 80 GiB leaves your next allocation to fail with `cudaErrorMemoryAllocation`, surfacing as a framework `CUDA out of memory` and a non-zero exit. The kubelet's `OOMKilled` reason is populated from a *host* memory-cgroup kill reported by the runtime; VRAM exhaustion never touches `memory.max` on any cgroup, so no cgroup OOM event fires and the reason stays `Error`. Alerting on the reason string is therefore blind to this, and you need a second detector: non-zero exit + `CUDA out of memory` in logs + high `DCGM_FI_DEV_FB_USED` on that GPU. Fault isolation is missing for the same structural reason — a replica is an integer in a plugin's device list, not a hardware object, so the fault domain is the whole physical GPU and a device-fatal Xid is an N-pod incident.

- **Precisely which signal does time-slicing destroy, and which does it preserve?** *Answer:* It preserves the per-pod **scheduling** identity and destroys the per-pod **measurement** identity. The device plugin advertises `replicas` fabricated devices per GPU with IDs of the form `GPU-<uuid>::<replica>` (`NewAnnotatedID`), so the kubelet allocates a distinct string to each pod and the pod-resources API reports distinct strings — you can always recover the exact *set* of pods sharing a device. What has only one value is the metric: DCGM programs hardware counters per device, so `DCGM_FI_PROF_GR_ENGINE_ACTIVE` for the physical GPU is one number, and `dcgm-exporter` (with `--kubernetes-virtual-gpus`) can only duplicate it across every holder. Allocation count is separately useless because it is uniform by construction — N identical tickets carry zero information about work done. So: the denominator of a split is knowable, the split itself is not.

- **Someone shows you `sum by (namespace) (DCGM_FI_PROF_GR_ENGINE_ACTIVE) * 4.00` as a cost dashboard. What is wrong, by how much, and how do you fix it?** *Answer:* The metric is duplicated once per pod holding a replica of each physical GPU, so `sum` fans out. The over-report factor is exactly N, the number of *holders* (not the configured `replicas` — unallocated replicas emit no series), independent of load, and it does not average out: +100% at N=2, +200% at N=3. It is also the most flattering possible error, since the most heavily shared nodes report the highest apparent utilisation. The minimal fix is to divide by the fan-out degree before aggregating: `DCGM_FI_PROF_GR_ENGINE_ACTIVE / on(UUID) group_left() count by (UUID) (DCGM_FI_PROF_GR_ENGINE_ACTIVE)`. That reconciles the total but charges every pod `u/N`, so it must be labelled `attribution="shared-estimate"`. The real fix is per-PID data (§8).

- **What is the accuracy cost of a fair-share split, quantified?** *Answer:* For pod *i* with true share `s_i` on a device at utilisation `u`, fair share assigns `u/N` and the error is `u × (1/N − s_i)`. The total always reconciles (`Σ u/N = u`), so aggregate cluster spend is right — the error is entirely redistributive. It is zero when load is even and maximal when one tenant does everything: with N=4 and one busy tenant, that tenant is under-billed 75% and each idle tenant is billed for work it did not do. In the lesson's worked example (74/18/2 percent SM), a nearly-idle notebook is billed $1.25/hr against a true $0.08/hr — a 1467% over-charge — while the busy server is under-billed 58%. Fair share is a legitimate estimator with a stated error, not a measurement, and the distinction has to live in the metric's labels.

- **What signals *can* you attribute time-sliced usage with, and what is the catch on each?** *Answer:* Four, in decreasing fidelity. (1) Per-PID SM utilisation via NVML `nvmlDeviceGetProcessUtilization`, which returns `nvmlProcessUtilizationSample_t{pid, timeStamp, smUtil, memUtil, encUtil, decUtil}`; resolve PID→cgroup→pod and split proportionally. Catch: it is a sampling interface with a bounded buffer (a slow poller loses samples), it returns `NVML_ERROR_NOT_SUPPORTED` on some driver/SKU combinations, and it does not attribute to MIG devices on Ampere. (2) Per-PID framebuffer via `nvmlDeviceGetComputeRunningProcesses_v3` → `nvmlProcessInfo_v2_t{pid, usedGpuMemory, gpuInstanceId, computeInstanceId}` — a real measurement, widely available, but memory rather than compute. (3) Energy weighting, which needs (1) anyway and is closer to marginal cost. (4) Fair share, always available, error = load skew, must be labelled. And the structural option: if a tenant needs a defensible bill, stop time-slicing that pool and use MIG.

- **How do you prove your attribution arithmetic is not silently broken?** *Answer:* Assert two conservation identities continuously, not once. **Allocation conservation:** attributed GPU-hours plus idle GPU-hours must equal (physical GPUs × window), exactly — per physical GPU, the allocated shares of all holders plus the free replicas' shares equal 1.0. Failure means double-counted replicas or bad share arithmetic. **Utilisation conservation:** `sum by (UUID)` of your attributed utilisation must equal the device's utilisation to within scrape jitter — in PromQL, `sum by (UUID) (attributed) / on(UUID) group_left() max by (UUID) (DCGM_FI_PROF_GR_ENGINE_ACTIVE)` should sit near 1. Failing by a factor of exactly N means you forgot to dedupe. Failing by 5–10% is normal, because the DCGM device counter and NVML per-PID samples use different clocks and windows — pick a tolerance and alert outside it rather than forcing exact agreement.

## Connections & what's next

This lesson is the hard half of a pair with lesson 06. Together they bracket the attribution spectrum the capstone's cost operator must branch on, keyed off `nvidia.com/gpu.sharing-strategy`: MIG gives distinct UUIDs end to end and an exact 1:1 join; time-slicing gives distinct scheduling IDs, one measurement key, and a labelled estimate. Lesson 03's pod-resources client is the exact tool used here — same API, answering a different question: in lesson 03 it told you *which device*, here it tells you *who else*.

Lesson 08 adds MPS, which looks superficially identical at the Kubernetes API (same sharing block, same `replicas`, same shared physical UUID, same DCGM fan-out) but differs on every axis underneath: it packs the *occupancy* dimension instead of the time dimension, it does enforce a memory partition through the control daemon, and its fault story is different in a way worth getting exactly right. Carry the conclusion from here forward unchanged — **the map survives, the metric does not** — because it applies to MPS verbatim.

Lesson 09 then shows the only structural fix: DRA's claim model, which puts the allocation in an API object, exposes per-share identity (`shareID`) when consumable capacity is in play, and lets sharing policy be per-workload instead of per-node. It does not repeal this lesson's arithmetic — a time-sliced claim still multiplexes pods onto one physical UUID — but it changes where the ownership map comes from and what quota can fence.

Next: **[04.8 · MPS and choosing a sharing mode](08-mps-choosing-sharing.md)** — the third point in the MIG / time-slicing / MPS triangle, and the one most engineers misplace.

## References & further reading

**Primary sources**

- [NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) — **fetched and read this session at release v0.19.2.** The authoritative `sharing.timeSlicing` schema, the `failRequestsGreaterThanOne` default (`false`) and its rationale, the `UnexpectedAdmissionError` transcript, the isolation statement ("no memory or fault isolation… if one workload crashes, they all do"), the node-label catalogue (`gpu.sharing-strategy`, `gpu.replicas`, `-SHARED` product suffix, `device-plugin.config`), and the list of time-sliceable resources per SKU. **Correction to the previous version of this lesson:** the current release line is v0.19.x, not v0.17.1 — the v0.19.2 README still contains a stale `v0.17.1` install URL in one example, which is how that number propagates.
- [NVIDIA/k8s-device-plugin — `internal/rm/devices.go`, `internal/rm/device_map.go`, `internal/plugin/server.go`, `api/config/v1/sharing.go`, `api/config/v1/replicas.go`](https://github.com/NVIDIA/k8s-device-plugin) — **read this session.** The `"::"` separator, `NewAnnotatedID`, `uniqueDeviceIDsFromAnnotatedDeviceIDs`, and the full `ReplicatedResources`/`ReplicatedResource`/`ReplicatedDevices` field set including `rename` and `devices` (`all` | count | list of index/UUID refs) and the `Replicas >= 2` constraint. **This is the source that corrects the "identical UUID" claim** in the earlier version of this lesson.
- [kubelet pod-resources API proto (`v1.PodResourcesLister`)](https://github.com/kubernetes/kubelet/tree/master/pkg/apis/podresources) — **fetched and read this session.** The exact gRPC contract: `List` / `GetAllocatableResources` / `Get`, and the `ContainerDevices{resource_name, device_ids, topology}` and `DynamicResource`/`ClaimResource` message shapes your mapper consumes.
- [NVIDIA/dcgm-exporter — `pkg/cmd/app.go`, `internal/pkg/transformation/kubernetes.go`, `etc/default-counters.csv`](https://github.com/NVIDIA/dcgm-exporter) — **fetched and read this session.** The complete flag set (`--kubernetes`, `--kubernetes-gpu-id-type` default `GPUUID`, `--pod-resources-kubelet-socket` default `/var/lib/kubelet/pod-resources/kubelet.sock`, `--kubernetes-virtual-gpus`/`KUBERNETES_VIRTUAL_GPUS`, `--kubernetes-enable-pod-labels`, `--collect-interval` default 30000 ms), the `stripVGPUSuffix`/`getSharedGPU` handling of `::N`, the 16 MiB `kubeletPodResourcesMaxRecvMsgSize`, and the exact field names and help text for `DCGM_FI_PROF_GR_ENGINE_ACTIVE`, `DCGM_FI_PROF_SM_ACTIVE`, `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_DEV_FB_USED`.
- [NVML API reference — `nvmlDeviceGetProcessUtilization`, `nvmlDeviceGetComputeRunningProcesses_v3`](https://docs.nvidia.com/deploy/nvml-api/) — the `nvmlProcessUtilizationSample_t` and `nvmlProcessInfo_v2_t` struct definitions quoted in §8 were read this session from the published `nvml.h` header. Read the reference guide for the sampling semantics of `lastSeenTimeStamp` and the `NVML_ERROR_NOT_SUPPORTED` conditions before relying on per-PID data.
- [NVIDIA/gpu-operator — `nvidia.com_clusterpolicies.yaml`](https://github.com/NVIDIA/gpu-operator) — **fetched and read this session at v26.3.3.** Confirms the `spec.devicePlugin.config.{name,default}` path used in the patch command, and the field descriptions quoted in §3.

**Bug reports and field evidence**

- [NVIDIA/dcgm-exporter #587 — "--kubernetes-virtual-gpus exports identical values for all pods instead of per-pod utilization"](https://github.com/NVIDIA/dcgm-exporter/issues/587) — the sharpest single statement of this lesson's thesis. *(Corroborated via search this session; GitHub HTML not fetchable through this environment's proxy.)*
- [NVIDIA/dcgm-exporter #642 — "Why are DCGM_FI_DEV_GPU_UTIL values not isolated per vGPU/Pod?"](https://github.com/NVIDIA/dcgm-exporter/issues/642) — Blackwell-generation hardware, `KUBERNETES_VIRTUAL_GPUS=true`, identical utilisation/power/temperature across all sharers. *(Same caveat.)*
- [NVIDIA/dcgm-exporter #151 — "no metrics labels about pod namespace/name when Pod uses time slicing GPU"](https://github.com/NVIDIA/dcgm-exporter/issues/151) — the inverse failure, and the diagnostic signature of `--kubernetes-virtual-gpus` being unset. *(Same caveat.)*
- NVIDIA Developer Forums, ["DCGM-Exporter: Missing Process-level Attribution for GPU Time-Slicing on Blackwell GB10"](https://forums.developer.nvidia.com/) — names the gap as process-level attribution, which points at the correct fallback. *(Same caveat.)*

**Deeper dives**

- NVIDIA GPU Operator docs, ["Time-Slicing GPUs in Kubernetes"](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html) — the ClusterPolicy wiring and per-node config reference. The isolation statement quoted in this lesson ("Unlike MIG, there is no memory or fault-isolation between replicas") appears here and in the device-plugin README; the latter is what was fetched directly this session. *(This domain is blocked by the current environment's egress proxy — verify field names against your installed Operator version.)*
- NVIDIA Technical Blog, ["Improving GPU Utilization in Kubernetes"](https://developer.nvidia.com/blog/improving-gpu-utilization-in-kubernetes/) — NVIDIA's own framing of time-slicing as an oversubscription tool, and the reference for `nvidia-smi compute-policy --set-timeslice` being available from CUDA 11.1 / R455 onward with four levels (`0=DEFAULT, 1=SHORT, 2=MEDIUM, 3=LONG`). Note that the millisecond values behind those levels are not published; do not invent them.
- [Lesson 06 — MIG operations](06-mig-operations.md) — the contrasting case, worth re-reading side by side with §6's diagram: the same join, with distinct keys all the way down.
