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
sources: 5
---

# 04.3 · Device-plugin recap & the kubelet pod-resources API

> **Concept.** The device plugin tells the kubelet *what* GPUs exist; the pod-resources API tells you *which pod holds which GPU* — the seam where per-pod attribution begins.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Lesson 04.2 taught you to read a crash-looping GPU Operator pod like a state machine — walk the dependency chain, don't start at the symptom. That skill assumes the fleet is healthy: drivers loaded, toolkit wired, device plugin `Running`. This lesson starts from that healthy state and asks a different question: given a node full of GPUs and pods that hold them, **how do you find out which pod holds which GPU?** That question is the hinge the whole rest of this module turns on — MIG accounting (04.6), the time-slicing attribution hole (04.7), and the capstone exporter (04.10) are all variations on the join this lesson builds. By the end you will have written the exact Go client dcgm-exporter uses internally, and you'll reuse it as the attribution core of your own operator.

## Why this matters

Every GPU cost or efficiency number your capstone operator emits is a join between two worlds that don't know about each other. On one side: hardware telemetry — utilization, memory, power, SM occupancy — keyed by **GPU UUID**, a string the driver assigns. On the other side: the thing finance and capacity planning care about — a **pod, a namespace, a team**. Nothing in DCGM, nothing in `nvidia-smi`, nothing in the driver knows what a Kubernetes pod is. The kubelet is the *only* component on the node that knows both the physical device IDs it handed out **and** the pod they went to. The pod-resources API is how it tells you that.

This is also a named interview probe in this module's calibration: *"why can't you request 0.5 GPU?"* is asked because the honest answer requires understanding extended-resource semantics at the device-plugin/kubelet boundary, not a hand-wave about "Kubernetes doesn't support it." A candidate who can additionally explain *how* you'd attribute a shared GPU back to a pod — the pod-resources API, not `nvidia-smi` — is answering the follow-up question the interviewer is actually screening for on a GPU-fleet team. Getting the join wrong in production is worse than not having it: a stale mapping, a MIG UUID confused with its parent GPU's UUID, or a missed container silently corrupts every downstream dollar figure your operator reports, and nobody notices until finance asks why the numbers don't reconcile.

## What's new here (calibration)

Module 02 already taught the device-plugin gRPC contract end to end — how a plugin registers with the kubelet over `/var/lib/kubelet/device-plugins/kubelet.sock`, streams device health via `ListAndWatch`, and returns mounts/env from `Allocate`. **We do not re-teach that.** What this lesson adds:

- **The pod-resources API in full** — a second, separate, read-only gRPC service the kubelet exposes, with all three of its RPCs (`List`, `GetAllocatableResources`, `Get`) precisely characterized by version/stability, not just the one (`List`) most tutorials mention.
- **The allocated-vs-allocatable delta as a first-class metric** — `List` alone only shows you what's in use; pairing it with `GetAllocatableResources` gives you idle/unrequested capacity, a distinct cost signal from idle-but-allocated capacity (which needs DCGM, covered in lesson 04.7).
- **The node-local operational consequence** — why this is a DaemonSet-shaped problem, not a service you can query from off-node, and what that implies for how your exporter has to be deployed.
- **Precisely where this API sits relative to scheduling** — it is a *reporting* layer on top of the kubelet's Device Manager, not a participant in scheduling or admission, which shapes what you can and can't infer from it.

## Core concepts

### Recap: the device-plugin gRPC contract (Module 02, condensed)

One diagram, no new teaching — this is the mechanism the rest of this lesson assumes:

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

Two facts from that lesson this one builds directly on:

1. **Device IDs are opaque strings the plugin chose.** The NVIDIA plugin uses the GPU's UUID (`GPU-<uuid>`) as the device ID — or the MIG device UUID (`MIG-<uuid>`) when MIG is enabled. That string is the primary key of your join.
2. **Extended resources are integer-only.** `nvidia.com/gpu` is an *extended resource*; a container requests whole units, never `0.5`. Self-check (a) below walks the mechanism; it's the standard "why not 0.5 GPU" interview answer.

### The pod-resources API: a second, separate socket

Everything above is the *write* side — the device plugin telling the kubelet what to hand out. The kubelet also exposes a *read* side: a second, independent gRPC service on its own Unix domain socket:

```
/var/lib/kubelet/pod-resources/kubelet.sock
```

This is not the device-plugin socket, and the device plugin has nothing to do with serving it — it's the kubelet's own Device Manager exposing its already-committed allocation state, read-only. The proto lives in `k8s.io/kubelet/pkg/apis/podresources`; confirmed directly from the source tree, the package directory holds exactly `v1` and `v1alpha1` — there is no `v1beta` anything on current upstream, so don't go looking for one. The service is `PodResourcesLister`, and it has exactly three RPCs:

| RPC | Stability | Returns |
|---|---|---|
| `List` | GA (`v1`, k8s 1.20) | Every pod on the node → its containers → the device IDs, `cpu_ids`, and memory *currently assigned* to each container. |
| `GetAllocatableResources` | GA (`v1`, k8s 1.23) | The node's *full* set of allocatable devices/CPUs/memory, assigned or not — the denominator. |
| `Get` | **Alpha**, gated by `KubeletPodResourcesGet` | One pod by (name, namespace). Same shape as a single `List` entry, without paying for the whole-node response. |

This RPC surface and stability level is confirmed straight from the `k8s.io/kubelet` source — not inferred from docs that may be stale by the time you read this. If you're studying this after a few Kubernetes releases have shipped, re-check whether `Get` has moved past alpha; it's the one RPC here still actively evolving.

To reach any of these three RPCs your code must run **on the node** — as a DaemonSet with the socket bind-mounted (`hostPath: /var/lib/kubelet/pod-resources`), which is exactly how dcgm-exporter and your capstone ship. No kube-apiserver call is involved; this is node-local and fast, so you can poll it on your metrics interval without apiserver load or rate-limiting concerns.

### What `List` gives you — the tree

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

`device_ids` is the payload you care about. For a whole-GPU pod it is the GPU UUID; for a MIG pod it is the MIG device UUID. The same tree also carries `cpu_ids` and `memory` per container — pod-resources is the single source for *all* topology-aware allocations, not just GPUs, which is why the capstone can later attribute CPU and hugepages through the identical mechanism.

### What `GetAllocatableResources` gives you — the denominator

`List` is the numerator of your allocation accounting; `GetAllocatableResources` is the denominator. It returns *all* devices the node can allocate, including idle ones no pod currently holds. With both in hand you compute node GPU allocation efficiency without touching DCGM at all:

```
free GPUs = { device_ids in GetAllocatableResources }  −  ∪ { device_ids in List }
```

That number — allocated vs. allocatable — is a headline metric for a cost operator: GPUs you are paying for but no pod has even *requested* are pure waste, distinct from GPUs that are allocated but sitting idle (which you only catch by joining to DCGM utilization data — that's lesson 04.7's territory). This lesson gives you the allocation half of that picture for free, from a single node-local socket, before you've touched a GPU metric at all.

### The `Get` RPC — alpha, single-pod lookup

`Get` answers a narrower question: "what does *this one pod* hold?" instead of the whole node. It exists for callers (admission-time tooling, a targeted debug query) that don't want to pay for and parse a full-node `List` response just to check one pod. It is gated behind the `KubeletPodResourcesGet` feature gate and is **still alpha** as confirmed from the current proto — meaning it may not be enabled on your cluster, its wire shape can still change between releases, and you should not build a production dependency on it without first confirming the gate is on and stable for your k8s version. For the capstone, `List` plus client-side filtering by pod name/namespace gets you the same answer with zero gate risk.

### How this powers dcgm-exporter — the reference join

dcgm-exporter, NVIDIA's own reference GPU-metrics exporter, reads DCGM fields keyed by GPU UUID, then calls `List` on this exact socket to build a `UUID → {pod, namespace, container}` map and attaches those as Prometheus labels. When you see `DCGM_FI_DEV_GPU_UTIL{pod="trainer-0",namespace="team-vision",...}`, the `pod` label was resolved through this API, not through any Kubernetes-aware code in the driver or DCGM itself. Your capstone performs the identical join, but keeps a different terminal dimension: `$/GPU-hour × allocated-GPU-hours per namespace` instead of a utilization gauge.

### Node-local by design — the operational consequence

Because the socket lives on the node's filesystem and the kubelet does not expose pod-resources over the network, there is no way to build a single centralized service that polls every node's allocations from outside the cluster. The DaemonSet-plus-hostPath shape isn't a convenience choice — it's the only shape that works. Each DaemonSet pod sees only its own node's pods; cluster-wide attribution is assembled later (typically by a Prometheus scrape aggregating per-node exporter output), not inside any single pod-resources client.

## Perspectives

**Developer/exporter-builder perspective.** From the chair of the person writing the client, this API is refreshingly narrow: no watch semantics, no long-lived stream, just request/response gRPC you poll. That simplicity is deliberate — it bridges "Kubernetes objects" and "physical device identity" as a pure read, which shapes your architecture toward a stateless polling loop rather than an informer-style cache.

**Kubelet-internals perspective.** pod-resources is a *reporting* API layered on top of the kubelet's Device Manager, exposing state that was already decided during pod admission (lesson 04.4 covers exactly how `Allocate`'s output gets wired into the container). It does not participate in scheduling or admission itself — querying it can never change what's allocated, only observe it. That separation is what makes it safe to poll aggressively: you can't accidentally perturb the allocation state by reading it.

**Observability perspective.** dcgm-exporter is the reference implementation of exactly this join, and it is public, open source, and mirrors the exact pattern this lesson teaches — reading `List`, keying by device UUID, attaching pod/namespace/container as labels on physical-GPU metrics. Studying its source (linked below) after you've built your own client is the fastest way to sanity-check your design against NVIDIA's own.

**Economics perspective.** `GetAllocatableResources` minus the union of `List` is, in plain terms, "GPUs you're paying for that nobody has even asked for." On a fleet billed by node-hour regardless of GPU utilization, that idle-and-unrequested count is pure waste sitting in the accounting before you've even measured whether the *allocated* GPUs are being used. It's the cheapest signal in this whole module to compute, because it costs one extra RPC call on a socket you're already polling.

## Real-world use cases

- **[NVIDIA dcgm-exporter](https://github.com/NVIDIA/dcgm-exporter)** — the reference implementation of this exact join: DCGM fields keyed by GPU UUID, enriched with pod/namespace/container labels resolved through the pod-resources `List` RPC. Read the source, not just the README, to see the join in real Go.
- **NVIDIA Developer Blog, "Monitoring GPUs in Kubernetes with DCGM"** — a vendor write-up of this exact monitoring pattern in production dashboards. *(This URL was found via search but could not be fetched this session to verify current content — treat as high-confidence but unconfirmed, and spot-check before citing verbatim.)*
- **NVIDIA Developer Blog, "Get Real-Time Visibility into GPU Usage Across Kubernetes Clusters"** — another vendor write-up describing a production monitoring stack built on this API. *(Same caveat: found via search, not independently fetched this session.)*
- **Honest gap:** no non-NVIDIA company engineering blog specifically about consuming the pod-resources API directly turned up in research for this lesson. Rather than inventing one, lean on the two primary sources above — the kubelet proto itself and the dcgm-exporter source — as the strongest evidence available.

## Worked example

Poke the socket by hand first, then build the client.

```bash
# On the node (or a debug pod with the socket mounted). The service speaks gRPC,
# so the quickest sanity check is just that something's listening:
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

`main.go` — dial the socket, call both `List` and `GetAllocatableResources`, and print the allocated-vs-allocatable delta alongside the pod → container → GPU-UUID map:

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

const gpuResource = "nvidia.com/gpu"

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

	// 1) List: who holds what, right now.
	listResp, err := client.List(ctx, &podres.ListPodResourcesRequest{})
	if err != nil {
		fmt.Fprintln(os.Stderr, "List:", err)
		os.Exit(1)
	}

	allocated := map[string]bool{}
	for _, pod := range listResp.GetPodResources() {
		for _, c := range pod.GetContainers() {
			for _, dev := range c.GetDevices() {
				if dev.GetResourceName() != gpuResource {
					continue // ignore cpu/memory/other vendors
				}
				for _, id := range dev.GetDeviceIds() {
					allocated[id] = true
					fmt.Printf("%s/%s\tcontainer=%s\tgpu=%s\n",
						pod.GetNamespace(), pod.GetName(), c.GetName(), id)
				}
			}
		}
	}

	// 2) GetAllocatableResources: the node's total GPU capacity, the denominator.
	allocResp, err := client.GetAllocatableResources(ctx, &podres.AllocatableResourcesRequest{})
	if err != nil {
		fmt.Fprintln(os.Stderr, "GetAllocatableResources:", err)
		os.Exit(1)
	}

	total := 0
	free := []string{}
	for _, dev := range allocResp.GetDevices() {
		if dev.GetResourceName() != gpuResource {
			continue
		}
		for _, id := range dev.GetDeviceIds() {
			total++
			if !allocated[id] {
				free = append(free, id)
			}
		}
	}

	fmt.Printf("\n--- node summary ---\nallocatable=%d allocated=%d free=%d\n",
		total, len(allocated), len(free))
	for _, id := range free {
		fmt.Printf("  unrequested: %s\n", id)
	}
}
```

Build static and run it inside a DaemonSet (or `nsenter` onto the node). Expected output on a node with 4 GPUs, 2 currently held:

```
team-vision/trainer-0     container=cuda    gpu=GPU-a1b2c3d4-...
team-nlp/infer-7b-5c9f    container=server  gpu=MIG-e5f6a7b8-...

--- node summary ---
allocatable=4 allocated=2 free=2
  unrequested: GPU-c9d0e1f2-...
  unrequested: GPU-d3e4f5a6-...
```

Two things to notice. First, the MIG line is the tell that your join key must be treated as an opaque device UUID, not "the GPU at index 0" — a MIG-sliced card exposes multiple device IDs on one physical board, and only the pod-resources API (not `nvidia-smi -L` alone) tells you which slice went where. Second, the node summary is the allocated-vs-allocatable delta from Core concepts made concrete: two fully-billed GPUs on this node have no pod even requesting them — a cost signal you now have before touching a single DCGM metric.

## Practice — feeds the deliverable

**Task.** Turn the worked example into the seed of your capstone exporter.

1. Deploy the client as a DaemonSet that bind-mounts `hostPath: /var/lib/kubelet/pod-resources` (type `Directory`) read-only into the container, so it can reach `kubelet.sock`. Confirm in your notes *why* this must be a DaemonSet and cannot be a single centralized service (Core concepts, "Node-local by design").
2. Have it call `List` and print, for every GPU pod on that node, the tuple `namespace/pod · container · resource_name · device_id`.
3. Add the `GetAllocatableResources` call and print, for that node, the count of **allocatable** vs **allocated** GPU device IDs, plus the list of unrequested (free) device IDs — the allocated-vs-allocatable delta from the worked example.
4. (Stretch) If your cluster has `KubeletPodResourcesGet` enabled, call `Get` for a single known pod and confirm it returns the same devices as filtering `List` client-side for that pod — a good way to build intuition for why `Get` exists without depending on it being available everywhere.
5. Commit it under `practice/per-pod-attribution/` as the `podresources` package your cost operator will import.

**Acceptance.** A working Go pod-resources client, running on a real (or kind + fake device plugin) node, that prints correct `pod → container → GPU-UUID` mappings for every GPU pod, and reports node allocatable/allocated/free GPU counts. This binary is the seed of the [Per-pod GPU attribution](../practice/per-pod-attribution/README.md) deliverable — the attribution core everything else in this module joins onto.

## Common pitfalls

1. **Forgetting the socket is node-local only.** There is no network-reachable pod-resources service; a single centralized poller cannot reach it from off-node. Your client must run as a DaemonSet with the socket bind-mounted on every node you want data from.
2. **Depending on `Get` in production.** It is still alpha, gated by `KubeletPodResourcesGet`, and may simply not be enabled on your cluster. Use `List` and filter client-side unless you've explicitly confirmed the gate.
3. **Treating `device_ids` as positional or indexable** rather than opaque UUIDs. A MIG device's ID looks nothing like a plain GPU index; code that assumes `nvidia0`-style ordering silently mis-attributes MIG slices to the wrong pod.
4. **Confusing "allocated" with "utilized."** `List` tells you a pod *holds* a GPU, not whether it's doing any work on it. A GPU can be fully allocated and sitting idle — that's a DCGM question (lesson 04.7), not a pod-resources question.
5. **Polling without handling reconnects.** The kubelet can restart, which drops the socket connection. A DaemonSet client should retry the dial with backoff, not crash-loop on the first disconnect.

## Self-check

- Why are extended resources integer-only — why can't the scheduler give 0.5 GPU? **Answer:** Extended resources (like `nvidia.com/gpu`) are modeled as countable, whole devices. The device plugin advertises a set of discrete device IDs via `ListAndWatch`, and the kubelet's `Allocate` either hands a container a specific device or it doesn't — there is no kernel or kubelet mechanism to partition one device's compute/memory at request time, and the scheduler only tracks an integer `allocatable` count per node. Kubernetes therefore rejects fractional quantities for extended resources at admission. Sharing a GPU is done by advertising *more integer devices* instead — MIG exposes each slice as its own device ID, time-slicing/MPS advertises N virtual replicas of one physical GPU — so the request is still `1`, it just maps to a fraction of hardware behind the plugin.
- `List` vs `GetAllocatableResources` — what does each give you, and how do you combine them? **Answer:** `List` returns current assignments — every pod, its containers, and the device IDs each container actually holds — the numerator. `GetAllocatableResources` returns the node's full capacity, assigned or not — the denominator, including idle GPUs. Subtracting the union of `List` device IDs from `GetAllocatableResources` gives you free/unrequested GPUs, a cost signal distinct from idle-but-allocated capacity.
- Why must a pod-resources client run as a DaemonSet with the socket bind-mounted, instead of a single service polling every node from outside the cluster? **Answer:** The pod-resources service is exposed only as a Unix domain socket on the node's local filesystem (`/var/lib/kubelet/pod-resources/kubelet.sock`); the kubelet does not serve it over the network. The only way to reach it is a process running on that specific node with the socket path mounted in — which is exactly the DaemonSet + hostPath pattern dcgm-exporter and the capstone use. Cluster-wide attribution is assembled afterward, typically by a central scrape aggregating each node's exporter output, not by a single client reaching every node's socket directly.
- The `Get` RPC exists alongside `List` — why would you ever use it, and why should you be cautious about depending on it? **Answer:** `Get` answers a narrower question — "what does this one pod hold?" — without paying for a full-node `List` response, useful for a targeted lookup (e.g. admission-time tooling checking one pod). It's confirmed alpha and gated by `KubeletPodResourcesGet` directly from the current kubelet proto, meaning it may not be enabled on a given cluster and its shape can still change between releases; `List` plus client-side filtering achieves the same result without that risk, which is why the capstone exporter is built on `List`, not `Get`.

## Connections & what's next

This lesson recapped the device-plugin contract from module 02 and then went one layer deeper into the kubelet's *reporting* surface — the piece module 02 didn't need to cover because it's specific to attribution, not scheduling. The `List`/`GetAllocatableResources` split resurfaces directly in lesson 04.6 (MIG's clean, distinct-UUID-per-slice attribution) and lesson 04.7 (where time-slicing breaks the UUID-per-tenant assumption this lesson relies on, and DCGM has to fill the gap pod-resources can't). The capstone (04.10) is this lesson's Go client, extended.

Next: **[04.4 · Container runtime integration (CDI)](04-container-runtime-integration.md)** picks up exactly where `Allocate`'s output (env vars, device mounts) left off in the device-plugin recap above, and explains the mechanism — CDI or the legacy hook — that actually carries those decisions across the container boundary.

## References & further reading

**Primary sources**
- [kubelet pod-resources proto](https://github.com/kubernetes/kubelet/tree/master/pkg/apis/podresources) — the `PodResourcesLister` contract straight from source; read this for the exact RPC/stability table above.
- [NVIDIA k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) (confirmed v0.17.1) — the recap contract: what registers and advertises the device UUIDs this lesson's API reports on.

**Real-world engineering blogs**
- NVIDIA Developer Blog, "Monitoring GPUs in Kubernetes with DCGM" — production dashboard pattern built on this API. *(Found via search, not independently fetched this session — spot-check before citing.)*
- NVIDIA Developer Blog, "Get Real-Time Visibility into GPU Usage Across Kubernetes Clusters" — a second vendor write-up of the same production pattern. *(Same caveat.)*

**Deeper dives**
- [NVIDIA dcgm-exporter](https://github.com/NVIDIA/dcgm-exporter) — read the source (not just the README) for the exact `List`-based UUID→pod join your capstone re-derives.
