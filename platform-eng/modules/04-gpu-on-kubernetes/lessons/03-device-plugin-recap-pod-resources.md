---
lesson: "04.3"
title: "Device-plugin recap & the kubelet pod-resources API"
module: "04"
concept: "Device-plugin recap & the pod-resources attribution API"
status: not-started
est_time: "12h"
prev: "02-crash-loop-diagnosis.md"
next: "04-container-runtime-integration.md"
artifacts: []
sources: 13
---

# 04.3 · Device-plugin recap & the kubelet pod-resources API

> **Concept.** The device plugin tells the kubelet *what* GPUs exist; the pod-resources API tells you *which pod holds which GPU* — the seam where per-pod attribution begins.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Lesson 04.2 taught you to read a crash-looping GPU Operator pod as a state machine: walk the dependency chain, do not start at the symptom. That skill assumes the fleet is healthy — drivers loaded, toolkit wired, device plugin `Running`. This lesson starts from that healthy state and asks a different question: given a node full of GPUs and pods holding them, **how do you find out which pod holds which GPU?**

That question is the hinge the rest of this module turns on. MIG accounting (04.6), the time-slicing attribution hole (04.7), MPS (04.8), DRA claims (04.9) and the capstone exporter (04.10) are all variations on the join this lesson builds. By the end you will have written the exact Go client `dcgm-exporter` uses internally, understood the kubelet-side data structure it reads out of, and know precisely what the returned device ID means under each of the four sharing modes this module covers.

## Why this matters

Every GPU cost or efficiency number your capstone operator emits is a join between two worlds that do not know about each other.

On one side: hardware telemetry — utilisation, framebuffer, power, SM occupancy — keyed by **GPU UUID**, a string the driver assigns and burns into `nvidia-smi -L` output. On the other side: the thing finance and capacity planning care about — a **pod, a namespace, a team**. Nothing in DCGM, nothing in NVML, nothing in the driver knows what a Kubernetes pod is. The driver has never heard of a namespace.

The kubelet is the **only** component on the node that knows both the physical device IDs it handed out *and* the pod they went to. Not the API server: `Pod.spec.containers[].resources.limits` says `nvidia.com/gpu: 1`, a count, never an identity. Not the scheduler: it decremented an integer on a node object. Not the container runtime: it received a `/dev/nvidia3` path with no Kubernetes context attached. The mapping from *device identity* to *pod identity* is created inside the kubelet's Device Manager during pod admission, and the pod-resources API is the only way to read it back out.

This is also the module's first named interview probe: *"why can't you request 0.5 GPU?"* The reason it is asked is that the honest answer requires understanding extended-resource semantics at the device-plugin/kubelet boundary rather than hand-waving "Kubernetes doesn't support it". A candidate who can additionally explain *how* you would attribute a shared GPU back to a pod — the pod-resources API, not `nvidia-smi` — is answering the follow-up the interviewer is actually screening for.

Getting the join wrong in production is worse than not having it. A stale mapping, a MIG UUID confused with its parent GPU's UUID, or a container silently skipped will corrupt every downstream dollar figure your operator reports, and nobody notices until finance asks why the numbers do not reconcile. The failure is silent by construction: your PromQL still evaluates, the dashboard still renders, the number is just wrong.

## What's new here (calibration)

Module 02 already taught the device-plugin gRPC contract as an *API*: registration over a Unix socket, a health stream, an allocation call. This lesson does not re-teach it as an API — it re-teaches it as a **mechanism with state**, because the state is what you are about to read. What is genuinely new:

- **The full wire contract, both services, every message field**, reproduced from `k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1/api.proto` — including the three RPCs most write-ups skip (`GetDevicePluginOptions`, `GetPreferredAllocation`, `PreStartContainer`) and what each one is actually for.
- **The kubelet-side data structure** the pod-resources API reads: `podDevices`, its three-level map, the checkpoint file it survives a restart in, and the garbage-collection pass that runs on every `List`.
- **The pod-resources API in full** — all three RPCs, their current stability (all three of the relevant gates are now GA and locked; the "alpha `Get`" claim you will find in older write-ups, *including the previous version of this lesson*, is out of date), the server-side rate limit, and the kubelet metrics that tell you your client is being throttled.
- **What `device_ids` actually contains under each sharing mode** — whole GPU, MIG, time-sliced replica, DRA claim — because that string is the primary key of your entire attribution pipeline and it is *not* the same shape in all four cases.
- **The extra hop MIG requires.** DCGM does not label MIG metrics with the MIG device UUID. Getting from a pod-resources `MIG-…` string to a DCGM series requires an NVML lookup. This is the single most commonly-botched step in a homegrown exporter and it is spelled out here.
- **The allocated-vs-allocatable delta** as a first-class metric you get for one extra RPC call.

## Core concepts

### 1 — The problem: two identity systems that never meet

Before any mechanism, the shape of the problem.

A GPU has exactly one durable identity: the UUID the driver derives and reports, e.g. `GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763`. Every hardware-facing tool keys on it. `nvidia-smi -L` prints it. NVML's `nvmlDeviceGetHandleByUUID` accepts it. DCGM tags every field value with it. It survives reboots, driver reloads and node renames, because it is a property of the board.

A Kubernetes workload has a completely different identity: `(namespace, pod name, container name)`, plus a UID. It has no notion of a physical device — a pod requesting `nvidia.com/gpu: 1` is expressing a *quantity*, and the API server stores exactly that quantity and nothing else.

```
   HARDWARE IDENTITY SPACE                 KUBERNETES IDENTITY SPACE
   ───────────────────────                 ─────────────────────────
   GPU-8f2c1d90-…                          team-vision/trainer-0/cuda
   MIG-b6ba6b2a-…                          team-nlp/infer-7b/server
   PCI 0000:1b:00.0                        uid 4f3a…, node gpu-node-01
   /dev/nvidia3

   Known to: driver, NVML, DCGM,           Known to: apiserver, scheduler,
             nvidia-smi, your GPU vendor             your controllers, Prometheus

              ╲                                        ╱
               ╲          ONE COMPONENT                ╱
                ╲         KNOWS BOTH                   ╱
                 ▼                                    ▼
        ┌────────────────────────────────────────────────────┐
        │  kubelet — Device Manager                          │
        │  podDevices[podUID][container][resource] = {ids}   │
        │  built at admission, checkpointed to disk          │
        └────────────────────────────────────────────────────┘
                 │                              │
       written by│ Allocate()          read by  │ pod-resources API
                 │ (device plugin)              │ (your exporter)
```

Everything in this lesson is about that box: how the mapping gets written, where it lives, and how you read it.

### 2 — The device-plugin contract, completely

The device-plugin API is versioned `v1beta1` and has been since Kubernetes 1.10. The constant is literally in the source (`k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1/constants.go`):

```go
const (
    Healthy   = "Healthy"
    Unhealthy = "Unhealthy"

    Version          = "v1beta1"
    DevicePluginPath = "/var/lib/kubelet/device-plugins/"
    KubeletSocket    = DevicePluginPath + "kubelet.sock"

    KubeletPreStartContainerRPCTimeoutInSecs = 30
)

var SupportedVersions = [...]string{"v1beta1"}
```

Two facts hide in that block that matter operationally. First, `DevicePluginPath` is **hard-coded** — it is not derived from any kubelet flag, so a node with a relocated kubelet root still serves device-plugin registration from `/var/lib/kubelet/device-plugins/`. Second, `Version` is still `v1beta1` in current Kubernetes; the upstream docs are explicit that "the device plugin API is not stable" even though it has been beta for a very long time. Do not assume field stability the way you would for a GA workload API.

There are **two** gRPC services, and confusing them is the source of a lot of muddled explanations. The kubelet serves one; the plugin serves the other.

```protobuf
// SERVED BY THE KUBELET, on /var/lib/kubelet/device-plugins/kubelet.sock
service Registration {
    rpc Register(RegisterRequest) returns (Empty) {}
}

// SERVED BY THE PLUGIN, on a socket of its choosing in the same directory
// (the NVIDIA plugin uses nvidia-gpu.sock)
service DevicePlugin {
    rpc GetDevicePluginOptions(Empty) returns (DevicePluginOptions) {}
    rpc ListAndWatch(Empty) returns (stream ListAndWatchResponse) {}
    rpc GetPreferredAllocation(PreferredAllocationRequest) returns (PreferredAllocationResponse) {}
    rpc Allocate(AllocateRequest) returns (AllocateResponse) {}
    rpc PreStartContainer(PreStartContainerRequest) returns (PreStartContainerResponse) {}
}
```

The registration payload is four fields:

```protobuf
message RegisterRequest {
    string version              = 1;   // "v1beta1"
    string endpoint             = 2;   // socket file NAME, e.g. "nvidia-gpu.sock"
    string resource_name        = 3;   // "nvidia.com/gpu"
    DevicePluginOptions options = 4;
}

message DevicePluginOptions {
    bool pre_start_required                = 1;
    bool get_preferred_allocation_available = 2;
}
```

`endpoint` is a **file name, not a path** — the kubelet joins it to its own `DevicePluginPath`. That is why the plugin's socket must live in the same directory the kubelet's does, and why a device-plugin DaemonSet always bind-mounts `/var/lib/kubelet/device-plugins` read-write rather than read-only.

`resource_name` must be a fully-qualified extended-resource name of the form `vendor-domain/resourcetype`. `nvidia.com/gpu` is one. Under MIG `mixed` strategy the same plugin registers several: `nvidia.com/mig-1g.10gb`, `nvidia.com/mig-2g.20gb`, and so on — **one `Register` call per resource name**, each with its own endpoint socket. That detail matters when you are debugging a MIG node: you should see several sockets, not one.

The two booleans in `DevicePluginOptions` are the plugin telling the kubelet which optional RPCs it will actually answer. If `pre_start_required` is false the kubelet never calls `PreStartContainer`; if `get_preferred_allocation_available` is false it never calls `GetPreferredAllocation`. This is a capability handshake, not a config setting — the plugin decides.

**The registration sequence, with payloads.** This is the diagram to hold in your head, because every device-plugin failure in lesson 04.2 is a step in it that did not happen:

```
  DEVICE-PLUGIN REGISTRATION AND ALLOCATION — MESSAGE SEQUENCE
  (── ▶ = gRPC call; ═══▶ = long-lived stream)

  nvidia-device-plugin                                  kubelet
  (DaemonSet pod, hostPath                              (Device Manager +
   /var/lib/kubelet/device-plugins)                      podDevices store)
        │                                                     │
   (0)  │ serve DevicePlugin on                               │
        │ .../device-plugins/nvidia-gpu.sock                  │
        │ ── MUST be listening BEFORE step 1 ──               │
        │                                                     │
   (1)  │ Register(RegisterRequest{                           │
        │   version:       "v1beta1"                          │
        │   endpoint:      "nvidia-gpu.sock"   ← name only    │
        │   resource_name: "nvidia.com/gpu"                   │
        │   options: {pre_start_required:false,               │
        │             get_preferred_allocation_available:true}│
        │ }) ────────────────────────────────────────────────▶│
        │                                          ┌──────────┤
        │◀── Empty ────────────────────────────────┤ validate:│
        │                                          │ version, │
        │                                          │ name fmt │
        │                                          └──────────┤
   (2)  │◀── GetDevicePluginOptions(Empty) ───────────────────┤
        │ ── DevicePluginOptions{…} ─────────────────────────▶│
        │        (kubelet dials BACK on nvidia-gpu.sock)      │
        │                                                     │
   (3)  │◀── ListAndWatch(Empty) ─────────────────────────────┤
        │ ═══ stream: ListAndWatchResponse{                   │
        │       devices: [                                    │
        │         Device{ID:"GPU-8f2c1d90-…", health:"Healthy",
        │                topology:{nodes:[{ID:0}]}},          │
        │         Device{ID:"GPU-c9d0e1f2-…", health:"Healthy",
        │                topology:{nodes:[{ID:1}]}},          │
        │       ]} ══════════════════════════════════════════▶│
        │   (one message NOW, then one on EVERY change;       │
        │    the stream stays open for the plugin's life)     │
        │                                          ┌──────────┤
        │                                          │ len(devs)│
        │                                          │   ↓      │
        │                        node.status.capacity["nvidia.com/gpu"]    = 2
        │                        node.status.allocatable["nvidia.com/gpu"] = 2
        │                                          │  (healthy only)
        │                                          └──────────┤
        │                                                     │
        ····· scheduler binds team-vision/trainer-0 (wants 1) ·····
        │                                                     │
   (4)  │◀── GetPreferredAllocation(PreferredAllocationRequest{
        │       container_requests: [{                        │
        │         available_deviceIDs:   ["GPU-8f2c…","GPU-c9d0…"],
        │         must_include_deviceIDs: [],                 │
        │         allocation_size:        1 }]}) ─────────────┤
        │ ── PreferredAllocationResponse{                     │
        │      container_responses:[{deviceIDs:["GPU-8f2c…"]}]}▶
        │      (a HINT. the kubelet may ignore it.)           │
        │                                                     │
   (5)  │◀── Allocate(AllocateRequest{                        │
        │       container_requests:[{devices_ids:["GPU-8f2c…"]}]})
        │ ── AllocateResponse{ container_responses: [{        │
        │      envs: {"NVIDIA_VISIBLE_DEVICES":"GPU-8f2c1d90-…"}
        │      devices: [DeviceSpec{host_path:"/dev/nvidia0", │
        │                container_path:"/dev/nvidia0",       │
        │                permissions:"rw"}]                   │
        │      mounts: [...]                                  │
        │      annotations: {...}                             │
        │      cdi_devices:[CDIDevice{name:"nvidia.com/gpu=0"}]│
        │    }]} ────────────────────────────────────────────▶│
        │                                          ┌──────────┤
        │                                          │ RECORD:  │
        │                    podDevices[uid]["cuda"]["nvidia.com/gpu"]
        │                          = {deviceIds:["GPU-8f2c…"],│
        │                             allocResp: <the above>} │
        │                                          │ then     │
        │                                          │ checkpoint
        │                                          │ to disk  │
        │                                          └──────────┤
        │                                                     │
        │                          ── this record is what the pod-resources
        │                             API returns. §5.
```

Read step (5) twice. **The allocation record — the thing you will later query — is written as a side effect of `Allocate` succeeding, and it is written by the kubelet, not the plugin.** The plugin is stateless with respect to ownership; it never learns which pod got which device. That asymmetry is why you must ask the kubelet and cannot ask the plugin.

The remaining messages, for completeness:

```protobuf
message ListAndWatchResponse { repeated Device devices = 1; }

message Device {
    string ID           = 1;   // opaque, plugin-chosen
    string health       = 2;   // "Healthy" | "Unhealthy"
    TopologyInfo topology = 3; // NUMA hint, consumed by Topology Manager
}

message AllocateRequest  { repeated ContainerAllocateRequest container_requests = 1; }
message ContainerAllocateRequest { repeated string devices_ids = 1; }

message AllocateResponse { repeated ContainerAllocateResponse container_responses = 1; }
message ContainerAllocateResponse {
    map<string, string> envs   = 1;   // NVIDIA_VISIBLE_DEVICES lands here
    repeated Mount mounts      = 2;
    repeated DeviceSpec devices = 3;  // /dev nodes
    map<string, string> annotations = 4;
    repeated CDIDevice cdi_devices  = 5;  // DevicePluginCDIDevices, GA in 1.31
}

message Mount      { string container_path = 1; string host_path = 2; bool read_only = 3; }
message DeviceSpec { string container_path = 1; string host_path = 2; string permissions = 3; }

message PreStartContainerRequest  { repeated string devices_ids = 1; }
message PreStartContainerResponse {}
```

`cdi_devices` is the modern injection path and is what lesson 04.4 picks up; the `DevicePluginCDIDevices` feature gate that governs it went **GA in Kubernetes 1.31**, so on any cluster you will realistically run, fully-qualified CDI device names from `AllocateResponse` are honoured.

`PreStartContainer` is a hook for devices that need initialisation between allocation and container start — FPGA reprogramming is the canonical example. NVIDIA's GPU plugin does not use it. Its timeout is fixed at `KubeletPreStartContainerRPCTimeoutInSecs = 30` seconds, which is worth knowing because a plugin that advertises `pre_start_required: true` and then hangs will stall pod startup for exactly 30 seconds per container.

`GetPreferredAllocation` is where a plugin expresses *topology preference within a resource*: given eight free GPUs and a request for four, return the four that share an NVLink island. It is a hint. The kubelet is free to ignore it, and it is only called if the plugin advertised the capability.

**Health.** `ListAndWatch` is a stream, not a poll — the plugin pushes a full device list whenever anything changes. When a device flips to `Unhealthy`, the kubelet removes it from **allocatable but leaves capacity unchanged**. That asymmetry is the diagnostic signature of a sick GPU on an otherwise-fine node:

```
Capacity:
  nvidia.com/gpu:  8
Allocatable:
  nvidia.com/gpu:  7      ← one device is Unhealthy; capacity did not move
```

If you ever see capacity drop instead, the plugin stopped reporting the device entirely (a re-enumeration), which is a different failure with a different cause.

**Kubelet restart.** On startup the kubelet **deletes every socket in `/var/lib/kubelet/device-plugins/`** except its own, then recreates `kubelet.sock`. Plugins are expected to watch for their socket disappearing and re-register. This is why the NVIDIA plugin runs an fsnotify watch on that directory, and why a plugin that crashes without cleaning up leaves a stale socket the kubelet will happily clear on next boot.

### 3 — Why there is no fractional GPU, mechanically

The interview answer people give is "Kubernetes only supports integer extended resources". That is a restatement, not a mechanism. Here is the mechanism, in three independent layers that each independently forbid it.

**Layer 1 — API validation.** Extended resources are validated at admission. `pkg/apis/core/validation/validation.go` defines `isNotIntegerErrorMsg = "must be an integer"` and applies it to any resource name that is not a native one. So this pod never reaches a scheduler:

```console
$ kubectl apply -f half-gpu.yaml
The Pod "half-gpu" is invalid: spec.containers[0].resources.limits[nvidia.com/gpu]:
  Invalid value: "500m": must be an integer
```

Note it is `500m`, not `0.5` — `resource.Quantity` normalises `0.5` to `500m` before validation, which is why the error looks like it does.

**Layer 2 — the device list is a set.** `ListAndWatchResponse.devices` is a list of `Device{ID}`. Allocatable is `len(devices)` after filtering for health. There is no field anywhere in that message expressing "0.5 of device X". A quantity that is not a whole number of set elements is not representable on the wire.

**Layer 3 — `Allocate` takes IDs, not amounts.** `ContainerAllocateRequest.devices_ids` is `repeated string`. The kubelet either hands the plugin a device ID or it does not. There is no partial handoff, and there is nothing in the kernel or the kubelet that could enforce a fractional claim on a device the container has been given a `/dev` node for.

**So how does GPU sharing exist at all?** By making the *set* bigger. Every sharing mode in this module is the same trick applied differently:

| Mode | What the plugin puts in `ListAndWatch` | Allocatable on 1 physical GPU | Lesson |
|---|---|---|---|
| Exclusive | one `Device{ID: "GPU-8f2c…"}` | 1 | — |
| MIG `mixed` | one `Device{ID: "MIG-b6ba…"}` per GPU instance, under per-profile resource names | e.g. `nvidia.com/mig-1g.10gb: 7` | 04.6 |
| MIG `single` | same devices, all under `nvidia.com/gpu` | 7 | 04.6 |
| Time-slicing | `replicas` fabricated `Device{ID: "GPU-8f2c…::0" … "::3"}` | 4 | 04.7 |
| MPS | same fabrication, plus a control daemon that caps memory/threads | 4 | 04.8 |

**The request is always an integer. The integer just means less hardware.** That is the complete answer, and the follow-up question — "then how do you bill it?" — is the rest of this module.

### 4 — Inside the kubelet: where the mapping actually lives

The pod-resources API is a thin reader over a data structure. Knowing the structure tells you exactly what the API can and cannot answer.

From `pkg/kubelet/cm/devicemanager/pod_devices.go`:

```go
type deviceAllocateInfo struct {
    deviceIds checkpoint.DevicesPerNUMA          // map[int64][]string, NUMA node → IDs
    allocResp *pluginapi.ContainerAllocateResponse  // the cached Allocate reply
}

type resourceAllocateInfo map[string]deviceAllocateInfo // keyed by resourceName
type containerDevices     map[string]resourceAllocateInfo // keyed by containerName

type podDevices struct {
    sync.RWMutex
    devs map[string]containerDevices              // keyed by pod UID
}
```

Three levels: **pod UID → container name → resource name → {device IDs by NUMA node, cached AllocateResponse}**. The Device Manager additionally keeps flat sets:

- `allDevices` — everything any plugin has ever reported
- `healthyDevices` — the subset currently `Healthy`
- `allocatedDevices` — the union of everything in `podDevices`

```
  KUBELET DEVICE MANAGER — WHAT IS WHERE

  ┌──────────────────────────────────────────────────────────────────┐
  │ /var/lib/kubelet/device-plugins/                                 │
  │   kubelet.sock              ← Registration service (kubelet)     │
  │   nvidia-gpu.sock           ← DevicePlugin service (the plugin)  │
  │   kubelet_internal_checkpoint  ← JSON + checksum, see below      │
  └──────────────────────────────────────────────────────────────────┘
                              ▲ persisted on every allocation change
                              │
  ┌───────────────────────────┴──────────────────────────────────────┐
  │ ManagerImpl (in memory)                                          │
  │                                                                  │
  │   allDevices       {GPU-8f2c…, GPU-c9d0…, GPU-d3e4…, GPU-a1b2…}  │
  │   healthyDevices   {GPU-8f2c…, GPU-c9d0…, GPU-d3e4…}   ← a1b2 sick│
  │   allocatedDevices {GPU-8f2c…}                                   │
  │                                                                  │
  │   podDevices                                                     │
  │     "4f3a-…-uid"                          (pod UID)              │
  │        └─ "cuda"                          (container name)       │
  │             └─ "nvidia.com/gpu"           (resource name)        │
  │                  ├─ deviceIds  {0: ["GPU-8f2c…"]}   (NUMA 0)     │
  │                  └─ allocResp  {envs, devices, cdi_devices, …}   │
  └───────────────────────┬──────────────────────────────────────────┘
                          │
            GetDevices(podUID, containerName) ──┐
            GetAllocatableDevices()             │  read by the
                                                ├─ pod-resources
                                                │  gRPC server
                                                ┘
```

**The checkpoint.** `pkg/kubelet/cm/devicemanager/checkpoint/checkpoint.go` defines what gets written to `kubelet_internal_checkpoint`:

```go
type PodDevicesEntry struct {
    PodUID        string
    ContainerName string
    ResourceName  string
    DeviceIDs     DevicesPerNUMA
    AllocResp     []byte
}

type checkpointData struct {
    PodDeviceEntries  []PodDevicesEntry
    RegisteredDevices map[string][]string
}

type Data struct {
    Data     checkpointData
    Checksum checksum.Checksum
}
```

It is JSON with a checksum, and it is the reason a kubelet restart does not lose the pod→device mapping while plugins re-register. It is also worth knowing as a debugging artifact: if the pod-resources API is returning nothing sane, `cat /var/lib/kubelet/device-plugins/kubelet_internal_checkpoint | jq` shows you the ground truth the kubelet is working from. A checksum mismatch there causes the kubelet to refuse to load it, which surfaces as every GPU on the node appearing free while pods are demonstrably running.

Two mechanical consequences you can now derive rather than memorise:

1. **The mapping is keyed by pod UID, not pod name.** A pod deleted and recreated with the same name gets a new UID and a new entry. The pod-resources server translates UID back to `(name, namespace)` when it responds, which is why *it* needs the pod list and the Device Manager does not.
2. **`allocResp` is cached in full.** That is how the kubelet can re-inject the same environment and device nodes on a container restart without calling `Allocate` again — and it is why a device plugin restarting does not disturb running containers.

### 5 — The pod-resources API: a second, separate socket

The read side is a completely separate gRPC server on its own socket:

```
/var/lib/kubelet/pod-resources/kubelet.sock
```

Different directory, different service, different proto package. The device plugin has nothing to do with serving it. The full contract, from `k8s.io/kubelet/pkg/apis/podresources/v1/api.proto`:

```protobuf
service PodResourcesLister {
    rpc List(ListPodResourcesRequest) returns (ListPodResourcesResponse) {}
    rpc GetAllocatableResources(AllocatableResourcesRequest) returns (AllocatableResourcesResponse) {}
    rpc Get(GetPodResourcesRequest) returns (GetPodResourcesResponse) {}
}

message ListPodResourcesRequest  {}
message ListPodResourcesResponse { repeated PodResources pod_resources = 1; }

message AllocatableResourcesRequest  {}
message AllocatableResourcesResponse {
    repeated ContainerDevices devices = 1;
    repeated int64            cpu_ids = 2;
    repeated ContainerMemory  memory  = 3;
}

message GetPodResourcesRequest  { string pod_name = 1; string pod_namespace = 2; }
message GetPodResourcesResponse { PodResources pod_resources = 1; }

message PodResources {
    string name                            = 1;
    string namespace                       = 2;
    repeated ContainerResources containers = 3;
    repeated int64             cpu_ids     = 4;
    repeated ContainerMemory   memory      = 5;
}

message ContainerResources {
    string name                              = 1;
    repeated ContainerDevices devices        = 2;
    repeated int64            cpu_ids        = 3;
    repeated ContainerMemory  memory         = 4;
    repeated DynamicResource  dynamic_resources = 5;   // DRA — lesson 04.9
}

message ContainerDevices {
    string   resource_name    = 1;   // "nvidia.com/gpu", "nvidia.com/mig-1g.10gb"
    repeated string device_ids = 2;  // ← THE JOIN KEY
    TopologyInfo topology      = 3;
}

message ContainerMemory { string memory_type = 1; uint64 size = 2; TopologyInfo topology = 3; }
message TopologyInfo    { repeated NUMANode nodes = 1; }
message NUMANode        { int64 ID = 1; }

message DynamicResource {
    string claim_name                    = 2;
    string claim_namespace               = 3;
    repeated ClaimResource claim_resources = 4;
}

message ClaimResource {
    repeated CDIDevice cdi_devices = 1;
    string driver_name             = 2;
    string pool_name               = 3;
    string device_name             = 4;
    optional string share_id       = 5;   // consumable capacity — lesson 04.9
}

message CDIDevice { string name = 1; }
```

Note `DynamicResource.claim_name` starts at field number **2**, not 1 — field 1 was removed. Harmless, but it is the sort of thing that makes you double-check you are reading the real proto and not someone's transcription.

**Stability, with versions.** This is where older material — including the previous version of this lesson — is now wrong. From `pkg/features/kube_features.go` in `kubernetes/kubernetes` (read August 2026):

| Gate | Alpha | Beta | GA |
|---|---|---|---|
| `KubeletPodResourcesGet` | 1.27 (off) | **1.34** (on) | **1.36**, locked, gate removed in 1.37 |
| `KubeletPodResourcesDynamicResources` | 1.27 (off) | **1.34** (on) | **1.36**, locked, gate removed in 1.37 |
| `KubeletPodResourcesListUseActivePods` | — | — | GA; default flipped to `true` and **deprecated** in 1.34 |

So on the versions this module targets (1.33.x, or ≥1.34.4 — see lesson 04.9's version table), all three `PodResourcesLister` RPCs are available by default, and DRA claims are reported in `dynamic_resources`. `Get` being "alpha and probably off" was true through 1.33 and is not true now. The capstone can use `Get`; it still does not need to, for reasons in §7.

`KubeletPodResourcesListUseActivePods` is the subtle one. With it on (the default since 1.34), `List` enumerates **active** pods only — terminal pods are filtered out. Before that flip, `List` could return stale entries for completed pods, which is precisely the bug that made naive exporters report GPUs as held by pods that had exited. Upstream issue #119423 tracked it; the gate exists so you can restore the old behaviour, and it is deprecated because nobody should.

### 6 — How `List` is actually served, line by line

The server is small enough to read completely, and reading it answers questions the docs do not. From `pkg/kubelet/apis/podresources/server_v1.go`:

```go
func (p *v1PodResourcesServer) List(ctx context.Context, req *podresourcesv1.ListPodResourcesRequest) (
    *podresourcesv1.ListPodResourcesResponse, error) {

    metrics.PodResourcesEndpointRequestsTotalCount.WithLabelValues("v1").Inc()
    metrics.PodResourcesEndpointRequestsListCount.WithLabelValues("v1").Inc()

    var pods []*v1.Pod
    if p.useActivePods {
        pods = p.podsProvider.GetActivePods()   // terminal pods excluded
    } else {
        pods = p.podsProvider.GetPods()
    }

    podResources := make([]*podresourcesv1.PodResources, len(pods))
    p.devicesProvider.UpdateAllocatedDevices(logger)     // ← garbage collection

    for i, pod := range pods {
        pRes := podresourcesv1.PodResources{Name: pod.Name, Namespace: pod.Namespace}

        // restartable init containers (sidecars) count; ordinary init containers do not
        for _, container := range pod.Spec.InitContainers {
            if !podutil.IsRestartableInitContainer(&container) { continue }
            pRes.Containers = append(pRes.Containers, p.getContainerResources(logger, pod, &container))
        }
        for _, container := range pod.Spec.Containers {
            pRes.Containers = append(pRes.Containers, p.getContainerResources(logger, pod, &container))
        }
        podResources[i] = &pRes
    }
    return &podresourcesv1.ListPodResourcesResponse{PodResources: podResources}, nil
}
```

Five things fall out of that, all of which will bite an exporter author:

1. **`List` returns every pod on the node, GPU or not.** A pod with no devices appears with `containers[].devices` empty. Filter client-side; do not assume the response is GPU-shaped.
2. **`UpdateAllocatedDevices` runs on every call.** It compares `podDevices` against the active pod list and deletes entries for pods that are gone, then rebuilds `allocatedDevices`. So `List` is not a pure read — it triggers a GC pass. Harmless, but it means a stuck `activePods` provider makes stale entries persist, and it means you should not call `List` at 1 Hz "just in case".
3. **Restartable init containers are included; plain init containers are not.** If a sidecar (restart policy `Always` on an init container) holds a GPU, you will see it. A one-shot init container that held a GPU during setup will not appear, because by the time you look it has exited.
4. **Response size scales with total pods on the node, not GPU pods.** On a dense node this response is large. `dcgm-exporter` raises the gRPC receive ceiling for exactly this reason (`kubeletPodResourcesMaxRecvMsgSize = 16 * 1024 * 1024`, sixteen mebibytes, versus the gRPC default of four). Your client must do the same or it will fail with `ResourceExhausted` on big nodes and silently lose all pod labels for that scrape.
5. **The counters are real Prometheus metrics on the kubelet.** `kubelet_pod_resources_endpoint_requests_total{server_api_version="v1"}`, plus `..._requests_list`, `..._requests_get`, `..._requests_get_allocatable`, and matching `..._errors_*`. Scrape them and you can see your own exporter's call rate from the other side — invaluable when you are debugging whether the client is actually connecting.

**The server-side rate limit.** This is the fact nobody mentions and everybody eventually hits. The pod-resources server is constructed with a token-bucket interceptor (`pkg/kubelet/server/server.go`):

```go
server := grpc.NewServer(apisgrpc.WithRateLimiter(ctx, "podresources",
    podresources.DefaultQPS, podresources.DefaultBurstTokens))
```

and the constants (`pkg/kubelet/apis/podresources/constants.go`) are:

```go
const (
    Socket             = "kubelet"
    DefaultQPS         = 100   // "unlikely that there is a legitimate need
                               //  to query podresources more than 100/s"
    DefaultBurstTokens = 10
)
```

Exceed it and you get `codes.ResourceExhausted` with the message `rejected by rate limit`. One hundred QPS is enormous relative to a sane 10–15 s poll, so you will never hit it by polling — you hit it by writing a retry loop with no backoff, at which point your client rate-limits itself into a permanent outage. Handle `ResourceExhausted` explicitly and back off.

Note also that both `v1alpha1` and `v1` `PodResourcesLister` implementations are registered on the same socket. `v1alpha1` still exists for very old clients; use `v1`.

### 7 — The three RPCs, and what each is for

**`List` — who holds what, right now.** The numerator of everything. Returns the full tree above. This is what `dcgm-exporter` calls, and what your capstone calls.

**`GetAllocatableResources` — the denominator.** Returns the node's full device inventory regardless of assignment. The implementation is one line:

```go
func (m *ManagerImpl) GetAllocatableDevices(logger klog.Logger) ResourceDeviceInstances {
    return m.allDevices.Filter(m.healthyDevices)
}
```

**Read that carefully: it is `allDevices` filtered by `healthyDevices`.** An unhealthy GPU disappears from this response entirely. That is correct behaviour — you cannot allocate it — but it means your "free GPUs" calculation silently shrinks the denominator when hardware goes bad, rather than reporting a fault. If you want to *see* the fault you need `Node.status.capacity` (which does not shrink) alongside this, and the capacity-vs-allocatable gap is your unhealthy-device count. That gap is a genuinely useful alert and it costs nothing.

With both calls you get the allocation-efficiency picture from one socket, no DCGM involved:

```
  free device IDs = { ids from GetAllocatableResources }
                    − ⋃ { ids from List }

  unhealthy count = Node.status.capacity − len(GetAllocatableResources ids)
```

GPUs you are paying for that no pod has even *requested* are pure waste, and they are a completely different problem from GPUs that are allocated and idle. The first is a scheduling/capacity problem you fix by shrinking the fleet or improving bin-packing; the second is a tenant-behaviour problem you fix with chargeback. Conflating them produces bad decisions, and this API separates them for one extra RPC.

**`Get` — one pod, by name.** GA and locked as of 1.36. It exists so that admission-time or debug tooling can ask about a single pod without parsing a whole-node response. The implementation looks up the pod by `(namespace, name)` and returns `pod %s in namespace %s not found` if it is not there. For the capstone, `List` plus client-side filtering does the same job with one round trip for all pods rather than N, so `List` remains the right call for an exporter — `Get` is for the one-shot case.

### 8 — What is actually *in* `device_ids` — the four cases

This is the section to reread before you write a line of exporter code. `device_ids` is an opaque string chosen by the plugin, and its shape differs by sharing mode. Treating it as one thing is the number-one source of broken attribution.

```
  WHAT pod-resources RETURNS, BY SHARING MODE — AND THE KEY YOU NEED

  ┌──────────────┬──────────────────────────────────┬────────────────────────┐
  │ MODE         │ resource_name / device_ids       │ TO JOIN TO DCGM YOU NEED│
  ├──────────────┼──────────────────────────────────┼────────────────────────┤
  │ Whole GPU    │ nvidia.com/gpu                   │ nothing — DCGM labels   │
  │              │ "GPU-8f2c1d90-3e5a-…"            │ UUID="GPU-8f2c1d90-…"   │
  │              │                                  │ 1:1. done.              │
  ├──────────────┼──────────────────────────────────┼────────────────────────┤
  │ MIG (mixed)  │ nvidia.com/mig-1g.10gb           │ AN NVML LOOKUP.         │
  │              │ "MIG-b6ba6b2a-1cd4-5f1e-…"       │ DCGM does NOT label     │
  │              │  (driver ≥ R470 format)          │ with the MIG UUID —     │
  │              │ "MIG-GPU-8f2c…/1/0"              │ it labels UUID=<parent> │
  │              │  (driver < R470 format:          │ + GPU_I_PROFILE="1g.10gb"│
  │              │   MIG-<parentUUID>/<gi>/<ci>)    │ + GPU_I_ID="1".         │
  │              │                                  │ See §9.                 │
  ├──────────────┼──────────────────────────────────┼────────────────────────┤
  │ Time-slicing │ nvidia.com/gpu (or .shared)      │ strip "::N" → parent    │
  │ / MPS        │ "GPU-8f2c1d90-…::0"              │ UUID. The map is FINE;  │
  │              │ "GPU-8f2c1d90-…::1"              │ the METRIC is one value │
  │              │  ← DISTINCT per pod              │ for N holders. 04.7/04.8│
  ├──────────────┼──────────────────────────────────┼────────────────────────┤
  │ DRA (04.9)   │ devices[] is EMPTY.              │ read dynamic_resources: │
  │              │ ownership is in                  │ claim_name/_namespace,  │
  │              │ dynamic_resources[]              │ driver/pool/device name,│
  │              │                                  │ cdi_devices[], share_id │
  └──────────────┴──────────────────────────────────┴────────────────────────┘

  If your exporter only handles row 1, it silently emits nothing for MIG
  nodes, wrong numbers for sliced nodes, and empty labels for DRA nodes.
```

The DRA row is the one that surprises people: a container whose GPU came from a `ResourceClaim` has **no entry in `devices[]` at all**. Its allocation lives in `dynamic_resources`, a different repeated field with a different shape. An exporter that iterates `container.GetDevices()` and stops there will report a DRA-scheduled GPU node as completely idle. Lesson 04.9 covers the claim object model; what matters here is that the pod-resources response has two independent places ownership can live, and you must read both.

### 9 — The MIG hop nobody tells you about

This deserves its own section because it is where homegrown exporters break, and because the previous version of this lesson (and of lesson 04.6, and of the capstone) asserted the opposite.

**Claim you will see everywhere:** "DCGM reports per-MIG metrics keyed by the MIG UUID, so join on the MIG UUID."

**What `dcgm-exporter` actually emits.** From `internal/pkg/rendermetrics/render_metrics.go` (dcgm-exporter 4.6.0), the label set for an `FE_GPU` sample is built as:

```go
builder.add("gpu",        metric.GPU)          // physical index, e.g. "0"
builder.add(metric.UUID,  metric.GPUUUID)      // label name "UUID", value = PARENT GPU UUID
builder.add("pci_bus_id", metric.GPUPCIBusID)
builder.add("device",     metric.GPUDevice)    // "nvidia0"
builder.add("modelName",  metric.GPUModelName)
if metric.MigProfile != "" {
    builder.add("GPU_I_PROFILE", metric.MigProfile)    // "1g.10gb"
    builder.add("GPU_I_ID",      metric.GPUInstanceID) // "1"
}
```

There is no MIG-UUID label. A MIG series looks like this (this is the shape the exporter's own render test asserts):

```
DCGM_FI_PROF_GR_ENGINE_ACTIVE{gpu="0",UUID="GPU-8f2c1d90-…",pci_bus_id="…",device="nvidia0",modelName="NVIDIA H100 80GB HBM3",GPU_I_PROFILE="1g.10gb",GPU_I_ID="1",hostname="gpu-node-01"} 0.81
```

**So the join key on the DCGM side is `(gpu index, GPU_I_ID)`, and on the pod-resources side it is a `MIG-…` UUID string. Those do not match.** The bridge is NVML. `dcgm-exporter`'s pod mapper does exactly this (`internal/pkg/transformation/kubernetes.go`):

```go
if strings.HasPrefix(deviceID, "MIG-") {
    migUUID := stripVGPUSuffix(deviceID)                      // drop any "::N"
    migDevice, err := nvmlprovider.Client().GetMIGDeviceInfoByID(migUUID)
    if err == nil && migDevice.GPUInstanceID >= 0 {
        giIdentifier := deviceinfo.GetGPUInstanceIdentifier(
            deviceInfo, migDevice.ParentUUID, uint(migDevice.GPUInstanceID))
        deviceToPodMap[giIdentifier] = podInfo               // key: "<gpuIndex>-<giID>"
    }
    gpuUUID := migUUID[len("MIG-"):]
    deviceToPodMap[gpuUUID] = podInfo                        // fallback key
}
```

and the metric side produces the matching key:

```go
func (m Metric) GetIDOfType(idType appconfig.KubernetesGPUIDType) (string, error) {
    if m.MigProfile != "" {
        return fmt.Sprintf("%s-%s", m.GPU, m.GPUInstanceID), nil   // e.g. "0-1"
    }
    ...
}
```

`GetMIGDeviceInfoByID` handles both MIG UUID formats. On drivers ≥ R470 (470.42.01+) a MIG device has its own UUID starting `MIG-`, and NVML resolves it directly:

```go
device, _ := nvml.DeviceGetHandleByUUID(uuid)
parent, _ := device.GetDeviceHandleFromMigDeviceHandle()
parentUUID, _ := parent.GetUUID()
gi, _ := device.GetGpuInstanceId()
ci, _ := device.GetComputeInstanceId()
// → MIGDeviceInfo{ParentUUID, GPUInstanceID, ComputeInstanceID}
```

On older drivers the ID is `MIG-<GPU-UUID>/<gpuInstanceID>/<computeInstanceID>` and is parsed by string-splitting. Your exporter needs the same two-format handling if you might run on a mixed-driver fleet.

**The practical consequence:** an exporter that wants MIG attribution must have NVML available in-process. That is why `dcgm-exporter` requires Kubernetes mode to initialise NVML, and why a MIG join built from pod-resources alone — with no NVML — cannot work. Plan for it when you scope the capstone: MIG attribution needs the container to see the driver, which means the exporter pod must itself be a GPU-visible pod (or mount the driver root), not a plain sidecar.

### 10 — Node-local by design, and what that forces

The kubelet does not serve pod-resources over the network. There is no port, no TLS, no authentication — the security boundary is the Unix socket's file permissions and the fact that reaching it requires being on the node with a hostPath mount. Upstream is explicit that monitoring agents should mount **the directory** `/var/lib/kubelet/pod-resources/`, not the socket file, because the socket's inode is recreated on kubelet restart and a file-level bind mount would be left pointing at a dead inode.

That single design decision determines your entire deployment shape:

- Your client is a **DaemonSet**, one pod per GPU node. There is no centralised alternative.
- It needs `hostPath: /var/lib/kubelet/pod-resources` with `type: Directory`, and a privileged-enough security context to read a root-owned socket.
- Each instance sees only its own node. Cluster-wide attribution is assembled *afterwards*, by Prometheus scraping every instance — never inside one client.
- It needs **no Kubernetes RBAC at all** for the pod-resources call itself. There is no API-server involvement. (You will want RBAC if you add a pod informer for labels, but that is a separate concern.)
- Because there is no apiserver in the path, polling is cheap: no rate limiting from the apiserver's side, no etcd read amplification, no watch cache pressure. The only limit is the 100 QPS token bucket from §6.

## Perspectives

**Exporter-builder's view.** This API is refreshingly narrow: no watch, no long-lived stream, no pagination, no resource version. Just request/response you poll. That simplicity is deliberate and it shapes your architecture toward a stateless polling loop with a cached last-known-good map, rather than an informer with a work queue. The one thing it costs you is that you cannot know *when* something changed — which is why the standard pattern (lesson 04.10) is a ticker plus an optional pod informer used only as a "poll now" trigger.

**Kubelet-internals view.** pod-resources is a *reporting* surface over the Device Manager. It is downstream of admission and takes part in no decision. Querying it cannot change what is allocated, which is what makes it safe to poll aggressively — the only side effect is the `UpdateAllocatedDevices` GC pass, which was going to happen anyway. Contrast that with the device-plugin socket, where a badly-behaved client can genuinely wedge pod admission.

**Scheduler's view.** The scheduler never sees any of this. It sees `node.status.allocatable["nvidia.com/gpu"] = 8` and decrements. It has no concept of device identity, no concept of which specific GPU a pod will get, and therefore no ability to reason about NVLink islands, MIG geometry, or framebuffer contention. Every guardrail on *which* device a pod lands on lives below the scheduler (in the plugin's `GetPreferredAllocation` and the Topology Manager) or above it (in admission policy). This is the structural gap DRA was built to close, and lesson 04.9 is where you see it closed.

**FinOps view.** `GetAllocatableResources` minus the union of `List` is, in plain language, "GPUs you are paying for that nobody has even asked for". On a fleet billed by node-hour it is pure waste sitting in the accounting before you have measured whether the *allocated* GPUs are doing anything. It is the cheapest signal in this entire module — one extra RPC on a socket you are already polling — and it is usually the first number that makes a platform team's GPU spend legible to someone outside the platform team.

**Security view.** Anything that can read this socket can enumerate every pod on the node and its device assignments. It is not a secret store, but it is an information disclosure surface, and the mitigation is entirely filesystem permissions plus the privilege needed to get a hostPath mount. Treat a pod-resources client the way you treat a node-exporter: privileged, minimal, audited, and not running arbitrary user code.

## Real-world use cases

- **[NVIDIA/dcgm-exporter](https://github.com/NVIDIA/dcgm-exporter) — the reference implementation.** Read at release `4.6.0-4.8.3` (August 2026). Everything in §6, §8 and §9 above is drawn from its source: the 16 MiB receive-size bump, the `ResourceExhausted` handling that degrades to unlabelled metrics rather than failing the scrape, the `stripVGPUSuffix` handling of time-sliced `::N` IDs, the NVML MIG hop, the DRA branch, and the `pod`/`namespace`/`container` attribute names. If you build a homegrown exporter, this is the file to diff yourself against: `internal/pkg/transformation/kubernetes.go`.
- **`dcgm-exporter`'s degradation behaviour is itself a case study.** When the pod-resources call fails, `Process()` logs a warning and returns `nil` — the scrape continues, but every series ships without pod labels for that interval. A dashboard built on `sum by (namespace)` will show a hole that looks like "usage dropped to zero" rather than "labels were missing". That is a real failure mode you inherit if you copy the pattern, and the fix (serve the last-known-good map from cache, and expose a staleness gauge) is a capstone requirement in lesson 04.10.
- **Upstream issue `kubernetes/kubernetes#119423` — stale pods in `List`.** The `KubeletPodResourcesListUseActivePods` gate exists because `List` historically returned terminal pods, and the interaction with the Memory Manager made the stale entries impossible for a client to filter out (the response carries no phase information). The fix was to restrict the output to active pods, defaulted on in 1.34 and the old behaviour deprecated. The lesson generalises: **an API that returns entities without their lifecycle state forces every client to guess.** If your own exporter exposes a map, expose its freshness alongside it.
- **Honest gap.** No non-vendor company engineering blog specifically about consuming the pod-resources API turned up in research for this lesson, and this environment's egress proxy blocks `kubernetes.io`, `docs.nvidia.com` and most blog domains. Rather than cite something unverified, the primary sources above — the kubelet source tree and the dcgm-exporter source tree, both read directly this session — carry the weight. They are stronger evidence than a blog post anyway.

## Worked example

Build the client, run it against a node, and read every line of the output. This is the seed of the capstone exporter, so build it properly.

### Step 0 — confirm the socket exists

```console
$ sudo ls -l /var/lib/kubelet/pod-resources/
srw-rw---- 1 root root 0 Aug 17 09:12 kubelet.sock

$ sudo ls -l /var/lib/kubelet/device-plugins/
srw-rw---- 1 root root      0 Aug 17 09:12 kubelet.sock
srw-rw---- 1 root root      0 Aug 17 09:12 nvidia-gpu.sock
-rw------- 1 root root   1893 Aug 17 11:40 kubelet_internal_checkpoint
```

Both sockets present, plus the checkpoint. If `nvidia-gpu.sock` is missing, the plugin never registered and you are back in lesson 04.2 territory — nothing below will work.

The checkpoint is readable and is the ground truth:

```console
$ sudo jq '.Data.PodDeviceEntries[] | {PodUID, ContainerName, ResourceName, DeviceIDs}' \
    /var/lib/kubelet/device-plugins/kubelet_internal_checkpoint
{
  "PodUID": "4f3a9c21-77b1-4e0e-9a4c-1f2b3c4d5e6f",
  "ContainerName": "cuda",
  "ResourceName": "nvidia.com/gpu",
  "DeviceIDs": { "0": ["GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763"] }
}
```

The `"0"` key is the NUMA node. This is `podDevices` from §4, serialised.

### Step 1 — the Go client

`go.mod` — pin the `k8s.io/kubelet` minor to your cluster's minor where you can; the proto is stable but the Go module tracks Kubernetes versioning:

```
module github.com/you/per-pod-attribution/podresources

go 1.24

require (
	google.golang.org/grpc v1.79.3
	k8s.io/kubelet v0.36.3
)
```

`main.go` — dial the socket, call both `List` and `GetAllocatableResources`, classify each device ID by sharing mode, and print the allocated-vs-allocatable delta:

```go
package main

import (
	"context"
	"fmt"
	"net"
	"os"
	"sort"
	"strings"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/status"

	podres "k8s.io/kubelet/pkg/apis/podresources/v1"
)

const (
	socket = "/var/lib/kubelet/pod-resources/kubelet.sock"

	// Match dcgm-exporter: the stock 4 MiB gRPC ceiling is not enough on a
	// dense node, because List returns EVERY pod, not just GPU pods.
	maxRecvMsgSize = 16 * 1024 * 1024

	gpuResource    = "nvidia.com/gpu"
	migResourcePfx = "nvidia.com/mig-"
)

// Owner is the terminal value of the attribution map.
type Owner struct {
	Namespace, Pod, Container, Resource string
}

// classify reports what kind of device ID this is, which decides the join path.
func classify(id string) string {
	switch {
	case strings.Contains(id, "::"):
		return "time-sliced" // GPU-<uuid>::<replica>, lesson 04.7
	case strings.HasPrefix(id, "MIG-"):
		return "mig" // needs the NVML hop, §9
	case strings.HasPrefix(id, "GPU-"):
		return "whole"
	default:
		return "unknown" // GKE nvidia0/gi3, vendor variants, index strategy
	}
}

func dial(ctx context.Context) (*grpc.ClientConn, error) {
	return grpc.NewClient(
		"unix://"+socket,
		grpc.WithTransportCredentials(insecure.NewCredentials()),
		grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(maxRecvMsgSize)),
		grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) {
			var d net.Dialer
			return d.DialContext(ctx, "unix", socket)
		}),
	)
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	conn, err := dial(ctx)
	if err != nil {
		fmt.Fprintln(os.Stderr, "dial:", err)
		os.Exit(1)
	}
	defer conn.Close()
	client := podres.NewPodResourcesListerClient(conn)

	// ── 1) List: the numerator ────────────────────────────────────────────
	listResp, err := client.List(ctx, &podres.ListPodResourcesRequest{})
	if err != nil {
		// The server rate-limits at 100 QPS / burst 10. Back off; do not retry hot.
		if status.Code(err) == codes.ResourceExhausted {
			fmt.Fprintln(os.Stderr, "List: rate limited by kubelet, back off")
		}
		fmt.Fprintln(os.Stderr, "List:", err)
		os.Exit(1)
	}

	owners := map[string]Owner{} // deviceID -> Owner
	claims := 0

	for _, pod := range listResp.GetPodResources() {
		for _, c := range pod.GetContainers() {
			// (a) the device-plugin path
			for _, dev := range c.GetDevices() {
				rn := dev.GetResourceName()
				if rn != gpuResource && !strings.HasPrefix(rn, migResourcePfx) {
					continue // cpu/memory/hugepages/other vendors
				}
				for _, id := range dev.GetDeviceIds() {
					owners[id] = Owner{pod.GetNamespace(), pod.GetName(), c.GetName(), rn}
				}
			}
			// (b) the DRA path — devices[] is EMPTY for claim-scheduled pods
			for _, dr := range c.GetDynamicResources() {
				for _, cr := range dr.GetClaimResources() {
					if cr.GetDriverName() != "gpu.nvidia.com" {
						continue
					}
					claims++
					fmt.Printf("%-28s %-10s claim=%s/%s pool=%s device=%s shareID=%s\n",
						pod.GetNamespace()+"/"+pod.GetName(), c.GetName(),
						dr.GetClaimNamespace(), dr.GetClaimName(),
						cr.GetPoolName(), cr.GetDeviceName(), cr.GetShareId())
				}
			}
		}
	}

	ids := make([]string, 0, len(owners))
	for id := range owners {
		ids = append(ids, id)
	}
	sort.Strings(ids)
	for _, id := range ids {
		o := owners[id]
		fmt.Printf("%-28s %-10s %-24s %-11s %s\n",
			o.Namespace+"/"+o.Pod, o.Container, o.Resource, classify(id), id)
	}

	// ── 2) GetAllocatableResources: the denominator ───────────────────────
	allocResp, err := client.GetAllocatableResources(ctx, &podres.AllocatableResourcesRequest{})
	if err != nil {
		fmt.Fprintln(os.Stderr, "GetAllocatableResources:", err)
		os.Exit(1)
	}

	total, free := 0, []string{}
	for _, dev := range allocResp.GetDevices() {
		rn := dev.GetResourceName()
		if rn != gpuResource && !strings.HasPrefix(rn, migResourcePfx) {
			continue
		}
		for _, id := range dev.GetDeviceIds() {
			total++
			if _, held := owners[id]; !held {
				free = append(free, id)
			}
		}
	}

	fmt.Printf("\n--- node summary ---\n")
	fmt.Printf("allocatable(healthy)=%d  allocated=%d  free=%d  dra-claims=%d\n",
		total, len(owners), len(free), claims)
	for _, id := range free {
		fmt.Printf("  unrequested: %s\n", id)
	}
}
```

Three lines carry most of the weight and deserve comment:

- `grpc.MaxCallRecvMsgSize(maxRecvMsgSize)` — without it, a node with a few hundred pods returns a response larger than gRPC's 4 MiB default and every call fails. The symptom is total, not partial: you get no map at all, so every metric loses its pod label at once.
- `codes.ResourceExhausted` — two different things produce it: the receive-size ceiling above, and the kubelet's 100 QPS token bucket. Distinguish them by the message (`rejected by rate limit` for the latter) and treat them differently, because one needs a bigger buffer and the other needs backoff.
- The `GetDynamicResources()` loop — omit it and DRA nodes report as empty. This is the single most common thing missing from homegrown clients written before 1.34.

### Step 2 — run it as a DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: podres-probe
  namespace: gpu-operator
spec:
  selector:
    matchLabels: { app: podres-probe }
  template:
    metadata:
      labels: { app: podres-probe }
    spec:
      # Only where the GPU Operator says there are GPUs.
      nodeSelector:
        nvidia.com/gpu.present: "true"
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      containers:
        - name: probe
          image: your-registry/podres-probe:v0.1.0
          securityContext:
            # Reading a root-owned socket. Narrow this as far as your cluster allows.
            privileged: true
          volumeMounts:
            - name: pod-resources
              # Mount the DIRECTORY, not the socket file: the socket's inode is
              # recreated on kubelet restart and a file bind-mount would go stale.
              mountPath: /var/lib/kubelet/pod-resources
              readOnly: true
      volumes:
        - name: pod-resources
          hostPath:
            path: /var/lib/kubelet/pod-resources
            type: Directory
```

No `serviceAccountName` beyond the default is required: the pod-resources call never touches the API server.

### Step 3 — read the output

On a node with 8 GPUs where one card is MIG-partitioned, one is time-sliced ×4, one is unhealthy, and one pod is DRA-scheduled:

```console
$ kubectl -n gpu-operator logs ds/podres-probe --tail=40
team-vision/trainer-0        cuda       nvidia.com/gpu           whole       GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763
team-nlp/infer-7b-5c9f       server     nvidia.com/mig-1g.10gb   mig         MIG-b6ba6b2a-1cd4-5f1e-8a03-77d19e02c4b1
team-nlp/infer-7b-9d2a       server     nvidia.com/mig-1g.10gb   mig         MIG-4e91cc07-2b8a-51d3-9f76-0ac5518be7d2
tenants/notebook-a           jupyter    nvidia.com/gpu           time-sliced GPU-c9d0e1f2-8a4b-4c6d-b1e3-2f5a6b7c8d9e::0
tenants/notebook-b           jupyter    nvidia.com/gpu           time-sliced GPU-c9d0e1f2-8a4b-4c6d-b1e3-2f5a6b7c8d9e::1
team-ml/dra-trainer          main       claim=team-ml/gpu-claim-0 pool=gpu-node-01 device=gpu-3 shareID=

--- node summary ---
allocatable(healthy)=17  allocated=5  free=12  dra-claims=1
  unrequested: GPU-d3e4f5a6-...
  unrequested: MIG-71c2ab34-...
  ...
```

Now read it line by line, because every line teaches something.

**Line 1, whole GPU.** `resource_name` is the generic `nvidia.com/gpu`, the ID is a plain `GPU-` UUID. This is the trivial case: DCGM emits `…{UUID="GPU-8f2c1d90-…"}` and the join is a map lookup.

**Lines 2–3, MIG.** Two distinct `MIG-` UUIDs under a per-profile resource name. Two things to notice. First, the resource name itself carries the profile — `nvidia.com/mig-1g.10gb` — which is already an attribution dimension before you have looked at a single metric. Second, and critically, **these IDs will not match any DCGM label.** You need §9's NVML hop to turn `MIG-b6ba…` into `(gpu index 0, GPU_I_ID 1)`.

**Lines 4–5, time-slicing.** Two *different* strings that differ only in the `::N` suffix, both resolving to the same physical UUID `GPU-c9d0e1f2-…`. This is the correction lesson 04.7 makes its centrepiece: the scheduling identity survives sharing. You can always recover the exact *set* of pods contending for a physical device — that is the denominator of any split — and you get a replica index for free. What you cannot recover is each pod's share, because DCGM measures the device.

**Line 6, DRA.** `devices[]` was empty for this container; everything came from `dynamic_resources`. `shareID` is blank because this claim is not using consumable capacity; with `DRAConsumableCapacity` in play it would carry a per-share identity, which is the first time in this module's history that a *share* gets its own identifier (lesson 04.9, §8).

**The summary line.** `allocatable(healthy)=17` counts 6 plain GPUs + 7 MIG instances + 4 time-slice replicas = 17. It does **not** count the unhealthy card: compare against `kubectl describe node`:

```console
$ kubectl describe node gpu-node-01 | grep -A3 'Capacity:\|Allocatable:'
Capacity:
  nvidia.com/gpu:           10
  nvidia.com/mig-1g.10gb:   7
Allocatable:
  nvidia.com/gpu:            9      ← one Unhealthy device
  nvidia.com/mig-1g.10gb:    7
```

Capacity 10 vs allocatable 9 on `nvidia.com/gpu` is your unhealthy-device count, and it is invisible in the pod-resources response alone. **Alert on `capacity − allocatable > 0` per GPU resource name**; it is a one-line rule that catches a failing card before a tenant does.

### Step 4 — the allocation-efficiency number, computed

With the two RPCs you can now state, for this node, the thing finance actually asks about, without touching a single GPU metric:

```
  physical GPUs on node            = 10   (Node.status.capacity, nvidia.com/gpu)
  healthy + allocatable units      = 17   (GetAllocatableResources, all GPU resources)
  units currently held by a pod    =  5   (List)
  units free and healthy           = 12
  units unhealthy                  =  1   (capacity − allocatable)

  ALLOCATION EFFICIENCY = 5 / 17 = 29.4 %

  At a node rate of $32.00/hr over 8 billable GPUs = $4.00 per physical GPU-hour:
     paid for            10 × $4.00 = $40.00/hr   (you rent the box)
     ... of which nothing has even REQUESTED 12 units.
```

That 29.4 % is not utilisation — the five held units may be completely idle, which is a separate and worse problem you need DCGM for (lessons 04.7 and 04.10). It is *allocation* efficiency, and it is the cheapest honest number in this module. Emitting it, clearly labelled as allocation rather than use, is the first metric your capstone should ship, because it is the only one that cannot be wrong.

## Practice — feeds the deliverable

**Task.** Turn the worked example into the `podresources` package of your capstone exporter.

1. **Build and deploy the client** as a DaemonSet bind-mounting `hostPath: /var/lib/kubelet/pod-resources` (`type: Directory`) read-only. In your notes, write one paragraph stating *why* it must be a DaemonSet and cannot be a centralised service — cite the mechanism (§10), not the conclusion.
2. **Print the full mapping** for every GPU pod on the node: `namespace/pod · container · resource_name · device_id · classification`. Classification must distinguish `whole` / `mig` / `time-sliced` / `dra`.
3. **Add the denominator.** Call `GetAllocatableResources` and print allocatable / allocated / free counts plus the free device IDs. Separately fetch `Node.status.capacity` and report `capacity − allocatable` as the unhealthy-device count. Confirm the two numbers differ if and only if a device is `Unhealthy`.
4. **Prove the failure modes deliberately.** (a) Remove the `MaxCallRecvMsgSize` option and schedule enough pods on the node to exceed 4 MiB of response; capture the `ResourceExhausted` error. (b) Write a tight retry loop with no backoff and capture the `rejected by rate limit` error at 100 QPS. Both go in the failure-mode log with symptom → evidence → mechanism → fix.
5. **Restart the kubelet** (`systemctl restart kubelet` on a lab node) with a GPU pod running, and confirm your client survives it: the socket inode changes, so a client that bind-mounted the *file* rather than the *directory* dies, and one that cached its `grpc.ClientConn` must redial. Record what your implementation actually did.
6. **Inspect the checkpoint** at `/var/lib/kubelet/device-plugins/kubelet_internal_checkpoint` and confirm the entries match what `List` returned. This is your independent verification that the API is telling the truth.
7. **Commit it** under `practice/per-pod-attribution/internal/podresources/` as an importable package with a `List() (map[string]Owner, error)` surface. Lesson 04.10 imports this exact package.

**Acceptance.** A working Go pod-resources client, running on a real node (or on kind with a fake device plugin), that prints correct `pod → container → resource → device-ID` mappings for every GPU pod, classifies each device ID by sharing mode, reports allocatable/allocated/free/unhealthy counts, and handles both `ResourceExhausted` cases without crashing. This binary is the seed of the [Per-pod GPU attribution](../practice/per-pod-attribution/README.md) deliverable.

## Common pitfalls

1. **Leaving the gRPC receive size at the default.** The stock 4 MiB ceiling is exceeded on dense nodes because `List` returns *every* pod, not just GPU pods. The failure is all-or-nothing — you lose the entire map, so every metric ships unlabelled and your dashboard looks like usage went to zero. `dcgm-exporter` sets 16 MiB; match it.
2. **Bind-mounting the socket file instead of the directory.** The kubelet recreates `kubelet.sock` on restart with a new inode. A file-level bind mount points at the dead one forever, and your client hangs or errors until the pod is recreated. Mount the directory.
3. **Assuming `device_ids` has one shape.** Whole-GPU (`GPU-…`), MIG (`MIG-…` in two possible formats), time-sliced (`GPU-…::N`) and index-strategy (`0`, `1`) IDs all appear in the wild, plus GKE's `nvidia0/gi3` form. Code that string-matches one shape mis-attributes or drops the rest.
4. **Joining MIG device UUIDs directly to DCGM series.** They do not match. DCGM labels a MIG sample with the *parent* `UUID` plus `GPU_I_PROFILE` and `GPU_I_ID`; the bridge is `nvmlDeviceGetHandleByUUID` → `GetDeviceHandleFromMigDeviceHandle` → `GetGpuInstanceId`. Without NVML in-process, MIG attribution is not possible from pod-resources alone.
5. **Ignoring `dynamic_resources`.** A DRA-scheduled pod has an *empty* `devices[]`. An exporter that only walks `GetDevices()` reports a fully-utilised DRA node as idle, and nothing in the response signals that you missed something.
6. **Confusing "allocated" with "utilised".** `List` tells you a pod *holds* a device, never whether it is doing work. A GPU can be fully allocated and completely idle. That distinction is the entire point of emitting two cost metrics in lesson 04.10, and collapsing it into one number destroys the waste signal.
7. **Retrying without backoff.** The server allows 100 QPS with a burst of 10 and returns `ResourceExhausted` beyond that. A crash-retry loop turns a transient error into a self-inflicted permanent one.
8. **Believing the "`Get` is alpha" folklore.** It was alpha through 1.33; it is beta from 1.34 and GA-and-locked from 1.36, with the gate slated for removal in 1.37. Check `pkg/features/kube_features.go` for your cluster's minor rather than trusting any write-up, including this one.

## Self-check

- **Why are extended resources integer-only — why can't the scheduler give 0.5 GPU?** *Answer:* Three independent layers forbid it. (1) **API validation**: extended resources are validated as integers; `pkg/apis/core/validation/validation.go` carries `isNotIntegerErrorMsg = "must be an integer"`, so `nvidia.com/gpu: 0.5` is normalised to `500m` and rejected at admission with `Invalid value: "500m": must be an integer`. (2) **The wire format**: `ListAndWatchResponse` is a list of `Device{ID, health, topology}` and allocatable is the count of healthy entries — there is no field capable of expressing a fraction of a device. (3) **Allocation semantics**: `ContainerAllocateRequest.devices_ids` is a `repeated string`; the kubelet either hands over a device ID or does not, and nothing in the kernel could enforce a partial claim on a `/dev` node the container already has. Sharing therefore works by enlarging the *set* rather than dividing a member: MIG advertises one device per hardware instance, time-slicing and MPS fabricate `replicas` synthetic devices per physical GPU. The request is always `1`; the `1` just buys less hardware.

- **Walk the device-plugin lifecycle from process start to a container getting `/dev/nvidia0`.** *Answer:* (0) The plugin starts serving the `DevicePlugin` service on its own socket inside `/var/lib/kubelet/device-plugins/` — it must be listening *before* registering. (1) It dials the kubelet's `kubelet.sock` and calls `Register` with `{version: "v1beta1", endpoint: "nvidia-gpu.sock" (a file name, not a path), resource_name: "nvidia.com/gpu", options}`. (2) The kubelet dials *back* on that endpoint and calls `GetDevicePluginOptions` to learn whether `PreStartContainer` and `GetPreferredAllocation` are supported. (3) It opens a `ListAndWatch` stream; the plugin pushes the full device list immediately and again on every change. The kubelet publishes `len(devices)` as capacity and the healthy subset as allocatable. (4) On pod admission, if the plugin advertised the capability, the kubelet calls `GetPreferredAllocation` with the available IDs and the requested size and receives a *hint*. (5) It calls `Allocate` with the chosen IDs; the response carries `envs` (`NVIDIA_VISIBLE_DEVICES`), `devices` (`DeviceSpec` entries for `/dev/nvidia0` etc.), `mounts`, `annotations`, and `cdi_devices`. The kubelet merges those into the container's OCI spec and — the part that matters here — **records the assignment in `podDevices[podUID][container][resource]` and checkpoints it to `kubelet_internal_checkpoint`.** That record is what the pod-resources API later returns.

- **`List` vs `GetAllocatableResources` vs `Get` — what does each give you, and how do you combine them?** *Answer:* `List` returns current assignments for every active pod on the node — pod, namespace, containers (including restartable init containers), and per-container `devices[]` with `resource_name` + `device_ids`, plus `cpu_ids`, `memory` and `dynamic_resources`. It is the numerator, and it triggers a `UpdateAllocatedDevices` garbage-collection pass as a side effect. `GetAllocatableResources` returns the node's device inventory — implemented as `allDevices.Filter(healthyDevices)`, so **unhealthy devices are absent**. It is the denominator. Subtracting the union of `List`'s IDs gives free-and-unrequested capacity. Because unhealthy devices vanish from `GetAllocatableResources` but not from `Node.status.capacity`, `capacity − allocatable` is a free unhealthy-device counter you should alert on. `Get` returns a single pod by `(name, namespace)`, is GA-and-locked from 1.36, and exists for one-shot lookups; an exporter still prefers `List` because it needs everything anyway.

- **Why must a pod-resources client run as a DaemonSet, and what exactly does it need mounted?** *Answer:* The kubelet serves `PodResourcesLister` only on a Unix domain socket in the node's filesystem — no network listener, no port, no TLS. The only way to reach it is a process on that node with the path mounted in. So the shape is a DaemonSet with `hostPath: /var/lib/kubelet/pod-resources`, `type: Directory` — **the directory, not the socket file**, because the kubelet recreates the socket inode on restart and a file bind-mount would be left pointing at a dead inode. It needs enough privilege to read a root-owned socket, and it needs *no* Kubernetes RBAC for the call itself since the API server is not involved. Each instance sees only its own node; cluster-wide attribution is assembled afterwards by Prometheus scraping every instance.

- **You wrote an exporter, it works in a kind cluster, and on the production node every metric ships without pod labels. Name three mechanisms that produce exactly that symptom.** *Answer:* (1) **Response size** — production nodes run hundreds of pods and `List` returns all of them, exceeding gRPC's 4 MiB default receive limit; the call fails with `ResourceExhausted` and you get *no* map, so every series loses its labels at once. Fix: `grpc.MaxCallRecvMsgSize(16 << 20)`. (2) **Self-inflicted rate limiting** — the server runs a 100 QPS / burst-10 token bucket and returns `ResourceExhausted` with `rejected by rate limit`; a no-backoff retry loop keeps itself permanently throttled. (3) **Wrong device-ID shape** — kind has no real plugin, so you never exercised MIG (`MIG-…`, which additionally needs an NVML hop to match DCGM's `UUID` + `GPU_I_ID` labels), time-sliced (`GPU-…::N`), or DRA (empty `devices[]`, ownership in `dynamic_resources`) paths, and your lookup misses. A fourth possibility worth checking: mounting the socket *file* rather than its directory, so the client is talking to a dead inode after a kubelet restart.

- **What does the kubelet keep on disk about device allocation, and why does it matter to you?** *Answer:* `/var/lib/kubelet/device-plugins/kubelet_internal_checkpoint`, a JSON `Data{Data: checkpointData, Checksum}` where `checkpointData` holds `PodDeviceEntries []PodDevicesEntry{PodUID, ContainerName, ResourceName, DeviceIDs (map NUMA→[]string), AllocResp []byte}` plus `RegisteredDevices map[resourceName][]deviceID`. It exists so a kubelet restart does not lose the pod→device mapping while plugins re-register, and so a container restart can be given the same environment and device nodes without calling `Allocate` again (the whole `AllocateResponse` is cached). For you it is two things: an independent ground truth to verify your client against, and a failure mode — a checksum mismatch makes the kubelet refuse to load it, which presents as every GPU on the node appearing free while pods are demonstrably running on them.

## Connections & what's next

This lesson recapped the device-plugin contract as a mechanism with state, then went one layer deeper into the kubelet's *reporting* surface — the part module 02 did not need, because it is specific to attribution rather than scheduling.

Everything downstream keys off §8's table. Lesson 04.6 takes the `MIG-…` row and shows where those UUIDs come from, why the partition cannot be changed while a pod holds one, and how the per-profile resource name doubles as an attribution dimension. Lesson 04.7 takes the `::N` row and proves the exact thing this lesson set up: the map survives sharing, the metric does not. Lesson 04.8 shows MPS reproducing the same fan-out with a bounded error. Lesson 04.9 takes the `dynamic_resources` row and replaces the whole ownership question with an API object. Lesson 04.10 is this lesson's Go client, extended with the DCGM join, the two cost gauges, and the reconciliation identity that proves the arithmetic.

Next: **[04.4 · Container runtime integration (CDI)](04-container-runtime-integration.md)** picks up exactly where `AllocateResponse` left off — the `envs`, `devices` and `cdi_devices` fields above — and explains the mechanism that actually carries those decisions across the container boundary, and why `NVIDIA_VISIBLE_DEVICES=all` is a debugging trap that will also silently destroy your attribution.

## References & further reading

**Primary sources — read directly this session (August 2026)**

- [kubelet device-plugin proto — `k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1/api.proto`](https://github.com/kubernetes/kubelet/tree/master/pkg/apis/deviceplugin/v1beta1) — the complete two-service contract reproduced in §2: `Registration.Register`, and `DevicePlugin.{GetDevicePluginOptions, ListAndWatch, GetPreferredAllocation, Allocate, PreStartContainer}` with every message field.
- [`k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1/constants.go`](https://github.com/kubernetes/kubelet/tree/master/pkg/apis/deviceplugin/v1beta1) — `Healthy`/`Unhealthy`, `Version = "v1beta1"`, the hard-coded `DevicePluginPath`, and the 30-second `PreStartContainer` timeout.
- [kubelet pod-resources proto — `k8s.io/kubelet/pkg/apis/podresources/v1/api.proto`](https://github.com/kubernetes/kubelet/tree/master/pkg/apis/podresources) — the `PodResourcesLister` contract and every message in §5, including `DynamicResource`/`ClaimResource` with `share_id`.
- [`kubernetes/kubernetes` — `pkg/kubelet/apis/podresources/server_v1.go`, `constants.go`](https://github.com/kubernetes/kubernetes/tree/master/pkg/kubelet/apis/podresources) — the `List`/`Get`/`GetAllocatableResources` implementations walked in §6, plus `DefaultQPS = 100` / `DefaultBurstTokens = 10`.
- [`kubernetes/kubernetes` — `pkg/kubelet/cm/devicemanager/{manager.go, pod_devices.go, checkpoint/checkpoint.go}`](https://github.com/kubernetes/kubernetes/tree/master/pkg/kubelet/cm/devicemanager) — `podDevices`, `GetAllocatableDevices() = allDevices.Filter(healthyDevices)`, `UpdateAllocatedDevices`, and the `kubelet_internal_checkpoint` format in §4.
- [`kubernetes/kubernetes` — `pkg/features/kube_features.go`](https://github.com/kubernetes/kubernetes/blob/master/pkg/features/kube_features.go) — the gate table in §5. **Correction to the previous version of this lesson:** `KubeletPodResourcesGet` is not alpha. It went beta (on by default) in 1.34 and GA-and-locked in 1.36, with removal scheduled for 1.37; `KubeletPodResourcesDynamicResources` followed the identical path.
- [`kubernetes/kubernetes` — `pkg/apis/core/validation/validation.go`](https://github.com/kubernetes/kubernetes/blob/master/pkg/apis/core/validation/validation.go) — `isNotIntegerErrorMsg = "must be an integer"`, the admission-layer half of the "no 0.5 GPU" answer in §3.
- [kubernetes/website — `content/en/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins.md`](https://github.com/kubernetes/website/blob/main/content/en/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins.md) — the upstream device-plugin concept page, read from the repository because `kubernetes.io` is blocked by this environment's egress proxy. Source of "a plugin MUST start serving gRPC before registering", the socket-deletion-on-restart behaviour, the mount-the-directory-not-the-file guidance, and `DevicePluginCDIDevices` being GA in 1.31.
- [NVIDIA/dcgm-exporter](https://github.com/NVIDIA/dcgm-exporter) — read at tag `4.6.0-4.8.3`. `internal/pkg/transformation/kubernetes.go` (the 16 MiB receive ceiling, the `ResourceExhausted` degradation path, `stripVGPUSuffix`, the NVML MIG hop, the DRA branch), `internal/pkg/nvmlprovider/provider.go` (`GetMIGDeviceInfoByID` and both MIG UUID formats), `internal/pkg/rendermetrics/render_metrics.go` (the `UUID` / `GPU_I_PROFILE` / `GPU_I_ID` label set), `internal/pkg/collector/types.go` (`GetIDOfType` producing the `"<gpu>-<giID>"` MIG key), `internal/pkg/transformation/const.go` (the `pod`/`namespace`/`container` attribute names).
- [NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) — read on `main` at the v0.19.x line (v0.19.3 is the newest CHANGELOG entry as of August 2026). The resource-name catalogue, the MIG `single`/`mixed` behaviour, and the node-label catalogue. Note the README still contains a stale `v0.17.1` install URL in one example; do not take that as the current version.
- [NVML API header (`nvml.h`)](https://docs.nvidia.com/deploy/nvml-api/) — `nvmlReturn_t` including `NVML_ERROR_IN_USE = 19`, and the MIG structures used in §9's hop. Read from the header shipped with the DCGM source tree.

**Deeper dives**

- [Lesson 04.7 — Time-slicing and the attribution trap](07-time-slicing-attribution.md) — the `::N` row of §8's table, taken all the way to the arithmetic of the error.
- [Lesson 04.9 — The NVIDIA DRA driver and GPU quotas](09-dra-driver-and-quotas.md) — the `dynamic_resources` row, and why a claim is structurally better than an integer for attribution.
