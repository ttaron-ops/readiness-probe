---
lesson: "04.3"
title: "Device-plugin recap & the kubelet pod-resources API"
module: "04"
concept: "Device-plugin recap & the kubelet pod-resources API"
status: not-started
est_time: "8h"
artifacts: []
---
# 04.3 · Device-plugin recap & the kubelet pod-resources API
> **Concept.** The device plugin tells the kubelet *what* GPUs exist; the pod-resources API tells you *which pod holds which GPU* — the seam where per-pod attribution begins.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Why this matters

Every GPU cost or efficiency number your capstone operator emits is a join. On one
side you have hardware telemetry — utilization, memory, power, SM occupancy — keyed
by **GPU UUID**. On the other side you have the thing finance and capacity planning
care about — a **pod, a namespace, a team**. Nothing in DCGM, nothing in `nvidia-smi`,
nothing in the driver knows about Kubernetes. The kubelet is the *only* component on
the node that knows both the physical device IDs it handed out **and** the pod they
went to. The pod-resources API is how it tells you.

If you get this join wrong — stale mappings, MIG UUIDs confused with parent GPU UUIDs,
missing containers — every downstream dollar figure is wrong, silently. This lesson is
the foundation of the capstone: you will write the exact Go client that dcgm-exporter
uses internally, and reuse it as the attribution core of your cost operator.

## What's new here

Module **02** already taught the device-plugin gRPC contract end to end — how a plugin
registers with the kubelet, streams device health over `ListAndWatch`, and returns
mounts/env from `Allocate`. **We do not re-teach that.** Here is the one-diagram recap,
then we pivot to the *read* side the kubelet exposes back to you.

```
          DEVICE PLUGIN  (nvidia-device-plugin, one DaemonSet pod per node)
          ─────────────────────────────────────────────────────────────────
  (1) Register    ──►  kubelet   "I serve resource nvidia.com/gpu, here's my socket"
                       (plugin dials /var/lib/kubelet/device-plugins/kubelet.sock)

  (2) ListAndWatch ─►  kubelet   stream: [GPU-a1.., GPU-b2.., MIG-c3..] all Healthy
                       kubelet publishes Node.status.allocatable: nvidia.com/gpu: 8
                       └─► scheduler now schedules on that integer count

  (3) Allocate    ◄──  kubelet   "give me 1 device for this container"
                  ──►            returns env NVIDIA_VISIBLE_DEVICES=GPU-a1..,
                                 device mounts /dev/nvidia0, CDI device names
                       kubelet injects these into the container's CRI spec
```

The two facts from 02 that this lesson builds on:

1. **Device IDs are opaque strings the plugin chose.** The NVIDIA plugin uses the
   GPU's UUID (`GPU-<uuid>`) as the device ID — or the MIG device UUID
   (`MIG-<uuid>`) when MIG is enabled. That string is the primary key of your join.
2. **Extended resources are integer-only.** `nvidia.com/gpu` is an *extended
   resource*; a container requests whole units. There is no `0.5`. Understand *why*
   (self-check a) because it shapes everything about GPU sharing.

**New in this lesson:** the kubelet **pod-resources API** — a second, read-only gRPC
service the kubelet runs on `/var/lib/kubelet/pod-resources/kubelet.sock`. It answers:
"for every pod on this node, which device IDs (GPU/MIG UUIDs) does each container
hold?" That is the attribution mechanism. dcgm-exporter uses exactly this to stamp
`pod`, `namespace`, and `container` labels onto GPU metrics.

## Core notes

### The socket and the service

The kubelet serves the pod-resources gRPC service on a Unix domain socket:

```
/var/lib/kubelet/pod-resources/kubelet.sock
```

The proto lives in `k8s.io/kubelet/pkg/apis/podresources/v1`. The service is
`PodResourcesLister` with three RPCs:

| RPC | Since | Returns |
|-----|-------|---------|
| `List` | GA (v1, k8s 1.20) | Every pod on the node → its containers → the device IDs, cpu_ids, and memory *currently assigned* to each container. |
| `GetAllocatableResources` | GA (v1, k8s 1.23) | The node's *full* set of allocatable devices/CPUs/memory, assigned or not — the denominator. |
| `Get` | Alpha, gate `KubeletPodResourcesGet` | One pod by (name, namespace). Same shape as one `List` entry. |

To reach the socket your code must run **on the node** — as a DaemonSet with the
socket bind-mounted (`hostPath: /var/lib/kubelet/pod-resources`), which is exactly how
dcgm-exporter and your capstone will ship. No kube-apiserver call is involved; this is
node-local and fast, so you can poll it on your metrics interval.

### What `List` gives you

`ListPodResourcesResponse` is a tree:

```
PodResources
├── name         "trainer-0"
├── namespace    "team-vision"
└── containers []ContainerResources
    ├── name     "cuda"
    └── devices []ContainerDevices
        ├── resource_name  "nvidia.com/gpu"
        ├── device_ids   [ "GPU-a1b2c3..." ]      ← the join key
        └── topology      { nodes: [{ id: 0 }] }   ← NUMA affinity
```

`device_ids` is the payload you care about. For a whole-GPU pod it is the GPU UUID; for
a MIG pod it is the MIG device UUID. Note the same tree also carries `cpu_ids` and
`memory` per container — the pod-resources API is the single source for *all* topology-
aware allocations, not just GPUs, which is why the capstone can later attribute CPU and
hugepages the same way.

### What `GetAllocatableResources` gives you

`List` is the numerator of your allocations; `GetAllocatableResources` is the
denominator. It returns *all* devices the node can allocate — including idle ones no pod
holds. With both you can compute node GPU allocation efficiency directly on the node:

```
free GPUs = { device_ids in GetAllocatableResources }  −  ∪ { device_ids in List }
```

That number — allocated vs. allocatable — is a headline metric for a cost operator:
GPUs you are paying for but no pod has even *requested* are pure waste, distinct from
GPUs that are allocated but idle (which you catch later by joining to DCGM utilization).

### How this powers dcgm-exporter

dcgm-exporter reads DCGM fields keyed by GPU UUID, then calls `List` on this socket to
build `UUID → {pod, namespace, container}` and attaches those as Prometheus labels. When
you see `DCGM_FI_DEV_GPU_UTIL{pod="trainer-0",namespace="team-vision",...}`, the `pod`
label was resolved through this exact API. Your capstone does the same join, but keeps
the *cost* dimension: `$/GPU-hour × allocated-GPU-hours per namespace`.

## Worked example

Poke the socket by hand first, then build the client.

```bash
# On the node (or a debug pod with the socket mounted). The service speaks gRPC,
# so grpcurl needs the proto; the quickest sanity check is just that it's listening:
sudo ls -l /var/lib/kubelet/pod-resources/kubelet.sock
# srw-rw---- 1 root root 0 ... kubelet.sock
```

Now the real client. `go.mod`:

```
module podres-probe
go 1.22
require (
    google.golang.org/grpc v1.66.0
    k8s.io/kubelet v0.31.0
)
```

`main.go` — dial the socket, call `List`, print `pod → container → GPU-UUID`:

```go
package main

import (
	"context"
	"fmt"
	"net"
	"os"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	podres "k8s.io/kubelet/pkg/apis/podresources/v1"
)

const socket = "/var/lib/kubelet/pod-resources/kubelet.sock"

func main() {
	// grpc.NewClient is the current constructor; it connects lazily on first RPC.
	conn, err := grpc.NewClient(
		"unix://"+socket,
		grpc.WithTransportCredentials(insecure.NewCredentials()),
		grpc.WithContextDialer(func(ctx context.Context, addr string) (net.Conn, error) {
			return (&net.Dialer{}).DialContext(ctx, "unix", socket)
		}),
	)
	if err != nil {
		fmt.Fprintln(os.Stderr, "dial:", err)
		os.Exit(1)
	}
	defer conn.Close()

	client := podres.NewPodResourcesListerClient(conn)
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	resp, err := client.List(ctx, &podres.ListPodResourcesRequest{})
	if err != nil {
		fmt.Fprintln(os.Stderr, "List:", err)
		os.Exit(1)
	}

	for _, pod := range resp.GetPodResources() {
		for _, c := range pod.GetContainers() {
			for _, dev := range c.GetDevices() {
				if dev.GetResourceName() != "nvidia.com/gpu" {
					continue // ignore cpu/memory/other vendors
				}
				for _, id := range dev.GetDeviceIds() {
					fmt.Printf("%s/%s\tcontainer=%s\tgpu=%s\n",
						pod.GetNamespace(), pod.GetName(), c.GetName(), id)
				}
			}
		}
	}
}
```

Build static and run it inside a DaemonSet (or `nsenter` onto the node). Expected output
on a node running two GPU pods:

```
team-vision/trainer-0     container=cuda    gpu=GPU-a1b2c3d4-...
team-nlp/infer-7b-5c9f    container=server  gpu=MIG-e5f6a7b8-...
```

That MIG line is the tell that your join key must be treated as an opaque device UUID,
not "the GPU at index 0" — a MIG-sliced card exposes multiple device IDs on one physical
board, and only the pod-resources API (not `nvidia-smi -L` alone) tells you which slice
went where.

## Practice — feeds the deliverable

**Task.** Turn the worked example into the seed of your capstone exporter.

1. Deploy the client as a DaemonSet that bind-mounts
   `hostPath: /var/lib/kubelet/pod-resources` (type `Directory`) read-only into the
   container, so it can reach `kubelet.sock`.
2. Have it call `List` and print, for every GPU pod on that node, the tuple
   `namespace/pod · container · resource_name · device_id`.
3. Add a second call to `GetAllocatableResources` and print, for that node, the count of
   **allocatable** vs **allocated** GPU device IDs (allocated = union of `List` device
   IDs for `nvidia.com/gpu`).
4. Commit it under `practice/per-pod-attribution/` as the `podresources` package your
   cost operator will import.

**Acceptance.** A working Go pod-resources client, running on a real (or kind + fake
device plugin) node, that prints correct `pod → container → GPU-UUID` mappings for every
GPU pod and reports node allocatable-vs-allocated GPU counts. This binary is the seed of
the capstone exporter — the attribution core everything else joins onto.

## Self-check

**(a) Why are extended resources integer-only — why can't the scheduler give 0.5 GPU?**

**Answer:** Extended resources (like `nvidia.com/gpu`) are modeled as *countable, whole
devices*. The device plugin advertises a set of discrete device IDs via `ListAndWatch`,
and the kubelet's `Allocate` either hands a container a specific device *or it doesn't* —
there is no kernel or kubelet mechanism to partition one device's compute/memory at
request time, and the scheduler only tracks an integer `allocatable` count per node.
Kubernetes therefore *rejects* fractional quantities for extended resources at
admission. This is not a limitation you route around with `0.5`; sharing a GPU is done by
advertising **more integer devices** — MIG exposes each slice as its own device ID,
time-slicing/MPS advertises N virtual replicas of one physical GPU — so the request is
still `1`, it just maps to a fraction of hardware behind the plugin.

**(b) What does `Allocate` return to the kubelet, and at what point in pod admission?**

**Answer:** After the scheduler has bound the pod to the node (using the integer
`allocatable` count), the kubelet's Device Manager, during **pod admission / container
setup on the node and before it asks the CRI runtime to create the container**, picks
concrete healthy device IDs and calls the plugin's `Allocate` RPC with them. `Allocate`
returns a `ContainerAllocateResponse` per container containing: environment variables
(e.g. `NVIDIA_VISIBLE_DEVICES=GPU-<uuid>`), device nodes to expose (`/dev/nvidia0`,
`/dev/nvidiactl`), host mounts (driver libs), and/or CDI device names/annotations. The
kubelet merges these into the container's runtime spec so the runtime creates the
container with the GPU already wired in. Scheduling used the *count*; `Allocate` binds
the *specific* devices — and those bound IDs are exactly what `List` later reports.

**(c) `List` vs `GetAllocatableResources` — what does each give you?**

**Answer:** `List` returns the *current assignments*: every pod on the node, its
containers, and the device IDs/cpu_ids/memory each container was actually allocated — the
numerator, and the source of your pod→GPU-UUID map. `GetAllocatableResources` returns the
*node's full capacity*: all devices the node can allocate whether or not any pod holds
them — the denominator, including idle GPUs. Subtract the union of `List` device IDs from
`GetAllocatableResources` to get free/unrequested GPUs. `List` changes as pods come and
go; `GetAllocatableResources` changes only when devices appear or go unhealthy.

## Resources

1. **pod-resources API — kubelet proto** (the contract you compile against):
   https://github.com/kubernetes/kubelet/tree/master/pkg/apis/podresources — package
   `v1`, `PodResourcesLister` service. Read this first; the Go types come straight from
   here.
2. **Third-party device metrics reaches GA** (why this API exists, the dcgm-exporter
   pattern): https://kubernetes.io/blog/2020/12/16/third-party-device-metrics-reaches-ga/
   and the monitoring section of the device-plugin docs:
   https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#monitoring-device-plugin-resources
3. **Reference consumers** — NVIDIA k8s-device-plugin
   https://github.com/NVIDIA/k8s-device-plugin (what advertises the UUIDs) and
   dcgm-exporter https://github.com/NVIDIA/dcgm-exporter (the join you are re-building).
