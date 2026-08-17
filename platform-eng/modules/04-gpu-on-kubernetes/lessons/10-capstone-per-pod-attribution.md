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
sources: 12
---

# 04.10 · Capstone — per-pod GPU attribution

> **Concept.** Turn device UUID → pod → namespace → utilization → dollars, for MIG and time-sliced devices alike.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Every lesson in this module handed you one piece of a machine you are now assembling.

**04.1** gave you the GPU Operator's dependency chain — nothing downstream produces a number if the init-container chain is broken. **04.2** gave you the discipline of reading a failure from logs alone, and the failure-mode log that ships with this deliverable. **04.3** gave you the Go client this exporter is built around, the kubelet data structure behind it, and — critically — the table of what `device_ids` actually contains under each sharing mode. **04.4** explained how a device ID becomes a real `/dev/nvidia*` node inside the container, and why `NVIDIA_VISIBLE_DEVICES=all` silently destroys attribution by giving a container devices the kubelet never assigned it. **04.5** is why this exporter has to survive a rolling driver upgrade without emitting garbage mid-rollout. **04.6** gave you the clean case: a hardware partition with its own counters, plus the NVML hop that turns a `MIG-…` UUID into a DCGM join key. **04.7** gave you the hard case and proved it is real: the map survives sharing, the metric does not. **04.8** added MPS — the same fan-out with a *bounded* error, because the control daemon caps the thread percentage. **04.9** gave you a structurally cleaner ownership source in `ResourceClaim.status.allocation`, and the quota layer these numbers eventually feed.

This lesson introduces **no new Kubernetes mechanism**. It assembles all of it into one running exporter, and it is where you stop being someone who understands nine GPU-on-Kubernetes subsystems and become someone who can wire them into a system that answers a business question. That is the module's thesis, stated as a deliverable.

## Why this matters

Finance asks "what did team-a's training run cost last night?" and every off-the-shelf answer is wrong.

Cloud billing stops at the **node**: an 8×H100 box is one line item. `nvidia.com/gpu` counts **allocations**, not use — a pod holding an idle GPU is indistinguishable, to the scheduler, from one running it flat out. DCGM measures **devices**, and under any software sharing mode there are fewer devices than tenants. The only honest per-pod GPU number is a join across all three, and nobody ships it in a form that matches your fleet's exact mix of sharing modes and your rate card. You have to build it.

This is not a practice exercise adjacent to the deliverable — **this lesson is the deliverable.** Checkpoint item 8 ("Ship it") is exactly this exporter plus the failure-mode log. Checkpoint item 6 ("Attribution, live") is you demonstrating it against a real device UUID on a real pod. Item 5 is you explaining the cost-attribution consequence of each sharing mode, which is the thing this exporter encodes in a label.

There is a second reason, and it is the one that separates a good artifact from a great one. Most attribution pipelines are unfalsifiable: they emit numbers and nobody can check them. This one is falsifiable, because GPU-hours are conserved. **Attributed GPU-hours plus idle GPU-hours must equal physical GPUs times wall time, exactly.** If your exporter satisfies that identity continuously and can show the check, you have something an interviewer cannot poke a hole in — and, more importantly, something a finance team can dispute *against evidence* rather than against vibes. Building the check is a bigger part of this capstone than building the exporter.

## What's new here (calibration)

No new object model, no new API, no new sharing mode. What is genuinely new is **systems-integration judgement**, which is exactly what a capstone should test:

- **Composing four independent data sources** — pod-resources, NVML, DCGM, and node labels — each of which fails differently, into one process that degrades rather than lying.
- **A share model with an explicit basis.** "What fraction of the card does this pod hold?" has different answers depending on whether compute or memory is scarce, and lesson 04.6 showed they disagree by up to 16 % for the same MIG slice. Picking one and *saying so in a label* is the difference between a measurement and an assertion.
- **Two gauges with different denominators, and a signed gap.** Allocated cost and utilised cost do not measure the same thing, the gap between them can go *negative*, and the sign is a mechanically meaningful fact about the sharing mode rather than a bug.
- **The reconciliation identities as a shipped feature**, not a one-off sanity check: continuous PromQL assertions with alert rules, and the arithmetic showing what each identity catches.
- **Fleet-level error accounting** — what fraction of a chargeback total rests on estimates, and the bound on how wrong that fraction can make the total.
- **Degradation as a design requirement.** An exporter that blanks on a transient socket error trains people to distrust the data, which is worse than not having it.

## Core concepts

### 1 — The cardinality table, restated precisely

Everything the exporter does is a consequence of one table: how many pods claim each *measurement* identity. This is the lesson-04.3 device-ID table carried forward with the metric side attached.

| Sharing mode | pod-resources `device_ids` | DCGM measurement identity | pods per measurement | attribution |
|---|---|---|---|---|
| Whole GPU | `GPU-8f2c…` | `UUID="GPU-8f2c…"` | 1 : 1 | exact |
| **MIG** (04.6) | `MIG-b6ba…` (distinct per GI) | `UUID=<parent>` + `GPU_I_ID="2"` — via an NVML hop | 1 : 1 | exact |
| **Time-slicing** (04.7) | `GPU-8f2c…::0`, `::1`, `::2` — **distinct** | `UUID="GPU-8f2c…"` — **one** | N : 1 | estimate |
| **MPS** (04.8) | `GPU-8f2c…::0`, `::1` — **distinct** | `UUID="GPU-8f2c…"` — **one** | N : 1 | bounded estimate |
| **DRA** (04.9) | `devices[]` empty; see `dynamic_resources` | whatever the allocated device resolves to | depends on sharing config | as above |

Two corrections to folklore, both established earlier in this module and both load-bearing here:

**The map never collapses.** The common claim that time-sliced pods all receive the same device ID is false. The device plugin fabricates annotated IDs `GPU-<uuid>::<replica>` (`NewAnnotatedID` in `internal/rm/devices.go`), so pod-resources hands you a *distinct string per pod*. What that buys you is the **denominator**: you always know exactly which pods are contending for a physical device, and `count by (uuid)` of the holders is a number you measured rather than guessed. What collapses is one layer down: DCGM programs hardware counters per device, so there is one value for N claimants.

**MIG does not join on the MIG UUID.** dcgm-exporter labels a MIG sample with the *parent* GPU's `UUID` plus `GPU_I_PROFILE` and `GPU_I_ID`; its internal join key is `"<gpuIndex>-<GPU_I_ID>"`. Getting there from `MIG-b6ba…` requires NVML. Lesson 04.6 §8 has the full path; here it is a hard requirement on how you deploy the exporter, because NVML must be resolvable in-process.

**One update since lesson 04.7 was written that you should build against.** dcgm-exporter 4.6.0 does now split two fields per process when `--kubernetes-virtual-gpus` is on: `isPerProcessMetric()` returns true for exactly `DCGM_FI_DEV_GPU_UTIL` and `DCGM_FI_DEV_FB_USED`, and for those it emits the device-level series *plus* one series per holding pod with `pod_uid` and `vgpu` labels, values summed per pod from NVML per-PID data, and `"0"` for holders with no visible process. Everything else — including every `DCGM_FI_PROF_*` field — is still device-level and still fanned out identically across holders. So the situation as of the current release is: **you get a real per-pod split for coarse utilisation and framebuffer, and no split at all for the profiling fields.** Your exporter should prefer the per-PID path where it exists and label honestly where it does not.

### 2 — The architecture: four sources, two loops, one map

```
  gpu-attributor — COMPONENT LAYOUT (one DaemonSet pod per GPU node)

  ┌───────────────────────────────────────────────────────────────────────┐
  │ NODE  gpu-node-01                                                     │
  │                                                                       │
  │  ┌──────────────────┐   ┌───────────────────┐   ┌──────────────────┐  │
  │  │ kubelet          │   │ NVIDIA driver     │   │ nv-hostengine    │  │
  │  │ pod-resources    │   │ libnvidia-ml.so.1 │   │ (nvidia-dcgm DS) │  │
  │  │ unix socket      │   │                   │   │                  │  │
  │  └────────┬─────────┘   └─────────┬─────────┘   └────────┬─────────┘  │
  │           │ List()                │ NVML                 │ scrape     │
  │           │ ≤100 QPS              │ in-process           │ /metrics   │
  │           │ 16 MiB recv           │ needs driver mount   │ :9400      │
  │  ┌────────▼───────────────────────▼──────────────────────▼─────────┐  │
  │  │                        gpu-attributor                           │  │
  │  │                                                                 │  │
  │  │  LOOP A — ownership (10 s ticker + informer nudge)              │  │
  │  │    podresources.List()  ─┐                                      │  │
  │  │    inventory.Resolve()  ─┼─▶  ownerMap: joinKey → Owner         │  │
  │  │      · GPU-x       → "GPU-x"        (whole)                     │  │
  │  │      · GPU-x::N    → "GPU-x"        (shared, N holders)         │  │
  │  │      · MIG-y       → "0-2"          (NVML hop)                  │  │
  │  │      · DRA claim   → CDI name → UUID                            │  │
  │  │    + shareModel: allocated_share per (pod, device)              │  │
  │  │    + cached, timestamped, served stale on error                 │  │
  │  │                                                                 │  │
  │  │  LOOP B — utilisation (15 s ticker)                             │  │
  │  │    dcgm scrape        → per-device / per-GI utilisation         │  │
  │  │    nvml per-PID       → smUtil, usedGpuMemory  (shared devices) │  │
  │  │    /proc/<pid>/cgroup → podUID   (regex: pod([a-f0-9_-]+))      │  │
  │  │                                                                 │  │
  │  │  JOIN + COST → prometheus.GaugeVec                              │  │
  │  └──────────────────────────────┬──────────────────────────────────┘  │
  └─────────────────────────────────┼─────────────────────────────────────┘
                                    │ :9401/metrics
                                    ▼
                       Prometheus ──▶ recording rules ──▶ Grafana
                                  └─▶ RECONCILIATION ALERTS  (§8)

  RATE CARD comes from node labels, not from code:
     lab/provider=neocloud-a, lab/node-hourly-rate="32.00",
     nvidia.com/gpu.count="8"   →  $4.00 per physical GPU-hour
```

Two loops, not one, because the two data sources have different failure modes and different natural periods. Ownership changes on pod churn (seconds, bursty); utilisation changes continuously and is cheap to read. Coupling them means a pod-resources hiccup blanks your utilisation and vice versa.

**Why NVML must be in-process.** Three of the four resolution paths need it: the MIG hop (`DeviceGetHandleByUUID` → `GetDeviceHandleFromMigDeviceHandle` → `GetGpuInstanceId`), the per-PID fallback (`GetProcessUtilization`, `GetComputeRunningProcesses`), and the UUID→index mapping DCGM's `gpu` label uses. So the pod needs the driver: either request `nvidia.com/gpu: 1` (wasteful — it takes a schedulable unit), or mount the driver root the way the Operator's own operands do and set `NVIDIA_VISIBLE_DEVICES=all` with `NVIDIA_DRIVER_CAPABILITIES=utility`. **`utility` is the important part**: it brings NVML and `nvidia-smi` without CUDA, which is all you need and is the smallest capability that works. Lesson 04.4 covers why that env var is otherwise a trap; this is the one legitimate use of it, and it must be paired with a hard rule that no tenant workload ever sets it.

### 3 — Component one: the ownership map, and keeping it honest

The map is `joinKey → Owner`, where `joinKey` is whatever string will appear on the DCGM series. Building it is three steps: list, resolve, classify.

```go
// internal/attribute/types.go
package attribute

type DeviceKind string

const (
	KindWhole  DeviceKind = "whole"  // 1 pod : 1 physical GPU
	KindMIG    DeviceKind = "mig"    // 1 pod : 1 GPU instance
	KindShared DeviceKind = "shared" // N pods : 1 physical GPU (time-slicing / MPS)
)

// Owner is one (pod, container) holding one device.
type Owner struct {
	Namespace string
	Pod       string
	Container string
	PodUID    string

	Kind         DeviceKind
	ResourceName string // nvidia.com/gpu, nvidia.com/mig-3g.40gb, nvidia.com/gpu.shared
	RawDeviceID  string // exactly what pod-resources returned
	JoinKey      string // what will appear on the DCGM series
	PhysicalUUID string // the parent physical GPU, always
	Replica      int    // -1 unless Kind == KindShared

	// AllocatedShare is this pod's claim on ONE physical GPU, in [0,1].
	// Per physical GPU, the shares of all holders plus the unallocated
	// remainder MUST sum to exactly 1. That is identity A (§8).
	AllocatedShare float64
	ShareBasis     string // "exclusive" | "mig-compute" | "mig-memory" | "replica"
}

// Snapshot is what LOOP A publishes and LOOP B reads. Immutable once built.
type Snapshot struct {
	Built     time.Time
	Owners    map[string][]Owner // joinKey -> holders (len>1 only for KindShared)
	Devices   map[string]Device  // physicalUUID -> inventory
	Stale     bool               // true if this is a cached snapshot after an error
	NodeGPUs  int                // physical GPUs, from NVML, not from labels
}
```

`Owners` is `map[string][]Owner` and not `map[string]Owner`. That single type decision is the whole lesson-04.7 correction encoded in the data model: **a join key can have more than one holder, and the exporter must be able to see that it does.** A `map[string]Owner` silently keeps whichever holder it wrote last, which is the bug that produces "one pod is charged for the whole card and the others for nothing".

The resolution step:

```go
// internal/attribute/resolve.go
package attribute

import (
	"fmt"
	"strconv"
	"strings"
)

// Resolve turns one raw pod-resources device ID into a join key plus classification.
// inv is the NVML-backed inventory (§4).
func Resolve(inv Inventory, resourceName, rawID string) (Owner, error) {
	o := Owner{ResourceName: resourceName, RawDeviceID: rawID, Replica: -1}

	// (1) Time-sliced / MPS replica: "GPU-<uuid>::<n>". Strip and remember n.
	//     The plugin builds these as NewAnnotatedID(id, replica) -> "id::n".
	base := rawID
	if i := strings.LastIndex(rawID, "::"); i >= 0 {
		if n, err := strconv.Atoi(rawID[i+2:]); err == nil {
			base, o.Replica = rawID[:i], n
		}
	}

	switch {
	// (2) MIG: needs the NVML hop. DCGM does NOT label with the MIG UUID.
	case strings.HasPrefix(base, "MIG-"):
		mig, err := inv.MIGInfo(base) // handles both UUID formats, R470 boundary
		if err != nil {
			return o, fmt.Errorf("resolve mig %q: %w", base, err)
		}
		idx, ok := inv.IndexOf(mig.ParentUUID)
		if !ok {
			return o, fmt.Errorf("unknown parent %q for %q", mig.ParentUUID, base)
		}
		o.Kind = KindMIG
		o.PhysicalUUID = mig.ParentUUID
		o.JoinKey = fmt.Sprintf("%d-%d", idx, mig.GPUInstanceID) // matches dcgm-exporter
		return o, nil

	// (3) Whole or shared physical GPU. The join key is the bare UUID either way;
	//     what differs is how many holders it will end up with.
	case strings.HasPrefix(base, "GPU-"):
		o.PhysicalUUID = base
		o.JoinKey = base
		if o.Replica >= 0 {
			o.Kind = KindShared
		} else {
			o.Kind = KindWhole
		}
		return o, nil
	}

	// (4) Anything else — index strategy ("0"), GKE's "nvidia0/gi3", a second
	//     vendor. Do not guess. Count it and emit an unresolved counter so the
	//     gap is visible instead of silently dropped.
	return o, fmt.Errorf("unrecognised device id %q for resource %q", rawID, resourceName)
}
```

The `default` branch matters more than it looks. An exporter that silently `continue`s on an ID shape it does not understand under-reports allocation, and identity A (§8) will fail with no clue why. Emit `gpu_attributor_unresolved_device_ids_total{resource_name=...}` and alert on it being non-zero.

**Freshness.** The pod-resources API has no watch — `List` is the only call — so polling is native. The pattern that actually works:

- A **10-second ticker** rebuilds the whole snapshot. Cheap: one gRPC call plus NVML lookups that hit a cache.
- A **pod informer** on `spec.nodeName=<this node>` that, on add/delete, nudges a buffered channel to trigger an out-of-cycle rebuild. The informer never carries device IDs — it only tells you *that* something changed — so it augments the poll, it never replaces it. Debounce it (a 500 ms coalescing window) or a rolling deployment will make you hammer the socket.
- **Serve from cache on error**, with `Snapshot.Stale = true` and a `gpu_attributor_map_age_seconds` gauge. Blanking is worse than stale: a hole in a utilisation graph reads as "the GPUs stopped being used", which is a *wrong fact*, whereas stale-but-labelled is a known unknown.

```go
func (a *Attributor) ownershipLoop(ctx context.Context) {
	tick := time.NewTicker(10 * time.Second)
	defer tick.Stop()
	for {
		snap, err := a.buildSnapshot(ctx)
		if err != nil {
			a.errors.WithLabelValues("ownership", classifyErr(err)).Inc()
			if last := a.snapshot.Load(); last != nil {
				stale := *last            // copy
				stale.Stale = true
				a.snapshot.Store(&stale)  // keep serving, flagged
			}
		} else {
			a.snapshot.Store(snap)
		}
		select {
		case <-ctx.Done():
			return
		case <-tick.C:
		case <-a.nudge: // pod informer add/delete, debounced
		}
	}
}
```

`classifyErr` must distinguish the two `codes.ResourceExhausted` cases from lesson 04.3 — receive-size ceiling versus the kubelet's 100 QPS token bucket (`rejected by rate limit`) — because one needs a bigger buffer and the other needs backoff, and conflating them means you fix the wrong thing at 3 a.m.

### 4 — Component two: the inventory, and detecting the sharing mode

The inventory answers three questions the map needs and nothing else: what physical GPUs exist, what index does DCGM use for each, and what does this MIG UUID resolve to.

```go
// internal/inventory/inventory.go — NVML-backed, refreshed on driver change.
type Device struct {
	UUID        string
	Index       int      // DCGM's gpu="N" label
	Model       string   // "NVIDIA H100 80GB HBM3"
	TotalMemMiB uint64
	MIGEnabled  bool
	Instances   map[int]Instance // GPU instance id -> profile info
}

type Instance struct {
	GPUInstanceID int
	Profile       string // "3g.40gb"
	SMCount       int    // multiprocessorCount from the profile info
	MemMiB        uint64
}
```

**Sharing mode is a node label, not an inference.** The device plugin writes `nvidia.com/gpu.sharing-strategy` with documented values `none`, `time-slicing`, `mps`, plus `nvidia.com/gpu.replicas` and `nvidia.com/mig.strategy`. Read them; do not try to deduce the mode from the shape of a device ID, because `::N` appears under both time-slicing and MPS and they have different error bounds (04.8: MPS's `defaultActiveThreadPercentage` caps a tenant's SM share, which is why its estimate is *bounded* and time-slicing's is not).

```go
type NodeConfig struct {
	SharingStrategy string  // none | time-slicing | mps
	Replicas        int     // nvidia.com/gpu.replicas
	MIGStrategy     string  // none | single | mixed
	GPUCount        int     // nvidia.com/gpu.count
	RatePerGPUHour  float64 // derived: node rate / physical GPU count
}
```

**The rate card belongs in labels, not in the binary.** Put `lab/node-hourly-rate` on the node (or look it up from an instance-type table keyed on `node.kubernetes.io/instance-type`) and divide by the *physical* GPU count from NVML — never by `nvidia.com/gpu.count`, which under `single` MIG strategy counts MIG devices and under time-slicing counts replicas. Getting that denominator wrong is a silent 7× or 4× error in every dollar figure, and it is the single most likely arithmetic bug in this build.

### 5 — Component three: the share model

`AllocatedShare` is this pod's claim on **one physical GPU**, in [0, 1]. It is a property of the allocation, not of any measurement, and it must be computable without reading a single metric. Three cases:

```go
// internal/attribute/share.go

// AllocatedShare computes a pod's claim on one physical GPU.
// Invariant: per physical GPU, Σ holders' shares + unallocated ≡ 1.0
func AllocatedShare(o Owner, dev Device, cfg NodeConfig, basis string) (float64, string) {
	switch o.Kind {
	case KindWhole:
		return 1.0, "exclusive"

	case KindMIG:
		inst := dev.Instances[instanceIDOf(o.JoinKey)]
		full := dev.FullCardProfile() // the 7g profile: SMCount, MemMiB
		switch basis {
		case "memory":
			// 3g.40gb on A100-80GB: 40192/80384 = 0.5000
			return float64(inst.MemMiB) / float64(full.MemMiB), "mig-memory"
		default:
			// 3g.40gb on A100-80GB: 42/98 = 0.4286  (== 3/7)
			return float64(inst.SMCount) / float64(full.SMCount), "mig-compute"
		}

	case KindShared:
		// One replica of cfg.Replicas. NOT 1/(number of pods currently
		// holding one) — the unallocated replicas are real spare capacity
		// and belong to platform overhead, not to the tenants (04.7 §step 1).
		if cfg.Replicas < 1 {
			return 0, "replica"
		}
		return 1.0 / float64(cfg.Replicas), "replica"
	}
	return 0, "unknown"
}
```

Three decisions embedded there, each of which you should be able to defend:

**MIG's basis is a choice and must be labelled.** Lesson 04.6 showed compute-share and memory-share disagree: a `1g.10gb` on an A100-80GB is 14/98 = 14.3 % of the compute but 9856/80384 = 12.3 % of the framebuffer — a 16 % relative gap. For a memory-bound inference tenant, billing on compute over-charges. Pick per-fleet, emit `share_basis` as a label, and document it. Do not average the two; that is a number with no meaning.

**Under MIG, the shares do not sum to 1.** Under `7 × 1g.10gb`, compute shares sum to 7 × (1/7) = 1.000 but memory shares sum to 7 × 0.1226 = 0.858. The missing 14.2 % is lesson 04.6 §4's stranded memory slice — real capacity that physically cannot be allocated. **Book it to platform overhead explicitly.** It is not rounding, it is a geometry decision someone made, and surfacing it is how that decision gets revisited.

**Shared devices divide by `replicas`, not by holder count.** With `replicas: 4` and three co-resident pods, each pod's share is 0.25 and the fourth replica's 0.25 is unallocated. Dividing by 3 instead would socialise idle capacity onto the tenants — converting a platform problem (you over-provisioned the replica count) into a tenant cost they cannot act on.

### 6 — Component four: utilisation, and the ladder of honesty

`UtilisedShare` is the pod's share of the device's *actual work*, expressed in GPU-equivalents of one physical card. Unlike `AllocatedShare` it depends on measurement, and there is a strict ladder of sources ordered by fidelity. Walk down it and record which rung you landed on.

```
  THE ATTRIBUTION LADDER — pick the highest rung that works, LABEL the rung

  ┌──────────────────────────────────────────────────────────────────────┐
  │ rung │ available when              │ utilised_share       │ label    │
  ├──────┼─────────────────────────────┼──────────────────────┼──────────┤
  │  1   │ Kind == Whole               │ u_device             │ exact    │
  │      │ (1 holder, 1 device)        │                      │          │
  ├──────┼─────────────────────────────┼──────────────────────┼──────────┤
  │  2   │ Kind == MIG                 │ share × u_instance   │ exact    │
  │      │ DCGM GPU_I_ID series exists │  (per-GI counters)   │          │
  ├──────┼─────────────────────────────┼──────────────────────┼──────────┤
  │  3   │ Kind == Shared AND NVML     │ u_device ×           │ per-pid  │
  │      │ GetProcessUtilization works │   smUtil_i / Σ smUtil│          │
  │      │ AND cgroup→podUID resolves  │                      │          │
  ├──────┼─────────────────────────────┼──────────────────────┼──────────┤
  │  3b  │ as 3, but only per-PID      │ u_device ×           │ per-pid- │
  │      │ MEMORY resolves             │   fbUsed_i / Σ fbUsed│ memory   │
  ├──────┼─────────────────────────────┼──────────────────────┼──────────┤
  │  4   │ Kind == Shared, MPS         │ u_device / N_holders │ shared-  │
  │      │ (activeThreadPercentage     │  error BOUNDED by    │ capped-  │
  │      │  caps each tenant — 04.8)   │  the cap             │ estimate │
  ├──────┼─────────────────────────────┼──────────────────────┼──────────┤
  │  5   │ Kind == Shared, everything  │ u_device / N_holders │ shared-  │
  │      │ else failed                 │  error = load skew   │ estimate │
  └──────┴─────────────────────────────┴──────────────────────┴──────────┘

  N_holders = len(snapshot.Owners[joinKey]) — MEASURED, not configured.
  It is the number of pods actually holding a replica right now, which is
  ≤ replicas. This is why the map surviving sharing (04.7) matters: rungs
  4 and 5 need an exact denominator, and you have one.

  RUNG 1–2 sum EXACTLY to the device utilisation. Rungs 3–5 sum to it by
  construction (they are fractions of it). So identity B (§8) holds on
  every rung — which means identity B tests your ARITHMETIC, not your
  fidelity. Fidelity is what the label is for.
```

Two mechanics worth spelling out, because they are where implementations go wrong.

**Per-PID is a sampling interface with a bounded buffer.** `nvmlDeviceGetProcessUtilization(device, samples, &count, lastSeenTimeStamp)` returns samples newer than `lastSeenTimeStamp` from an internal ring. A slow poller silently loses samples, so shares computed from it drift. dcgm-exporter calls it with `lastSeenTimeStamp = 0` (`device.GetProcessUtilization(0)`), taking everything in the buffer, and treats `NVML_ERROR_NOT_SUPPORTED` as "no data" rather than an error. Do the same, and **probe for support at startup**, recording the answer with your driver version and SKU — it is not portable, and lesson 04.7's Blackwell GB10 forum report is exactly this call not being available where it was expected.

**PID → pod goes through the cgroup path, and the regex matters.** dcgm-exporter parses `/proc/<pid>/cgroup` (unified and per-subsystem lines) and extracts the pod UID with `pod([a-f0-9_-]+)`, then replaces `_` with `-` and requires ≥32 characters. The underscore substitution is not optional: systemd cgroup drivers write `pod4f3a9c21_77b1_4e0e_9a4c_1f2b3c4d5e6f.slice` while cgroupfs writes `pod4f3a9c21-77b1-...`. An implementation that handles only one driver works perfectly in kind and fails on every RHEL node.

```go
var podUIDRe = regexp.MustCompile(`pod([a-f0-9_-]+)`)

func podUIDFromCgroup(pid uint32) (string, error) {
	b, err := os.ReadFile(fmt.Sprintf("/proc/%d/cgroup", pid))
	if err != nil {
		return "", err // process exited between the NVML sample and now — normal
	}
	for _, line := range strings.Split(string(b), "\n") {
		if m := podUIDRe.FindStringSubmatch(line); len(m) == 2 {
			uid := strings.ReplaceAll(m[1], "_", "-")
			if len(uid) >= 32 {
				return uid, nil
			}
		}
	}
	return "", nil // host process (dcgm, the driver itself) — not a tenant
}
```

Requires `hostPID: true` on the DaemonSet, because you are reading another namespace's `/proc`. That is a real privilege escalation and it is the price of rung 3. If your security posture forbids it, you are capped at rung 5 for every shared device — which is a legitimate choice, provided the label says so.

**Which DCGM field.** Prefer `DCGM_FI_PROF_GR_ENGINE_ACTIVE` (fraction of time the graphics/compute engine was active, 0–1) over `DCGM_FI_DEV_GPU_UTIL` (percentage of time ≥1 kernel was resident). `GR_ENGINE_ACTIVE` is finer-grained *in what it measures*; neither is finer-grained *in whom it attributes to* on a non-MIG device. Fall back to `DCGM_FI_DEV_GPU_UTIL` where profiling is unavailable, and note in a label which you used — they are not the same number and mixing them in one series is how a dashboard develops an unexplained step change.

### 7 — Component five: two gauges, and why the gap is signed

Emit two metrics with different denominators, and resist every instinct to collapse them.

```go
// internal/metrics/metrics.go
var labels = []string{
	"namespace", "pod", "container",
	"gpu_uuid",      // the PHYSICAL GPU, always — the reconciliation key
	"join_key",      // what matched the DCGM series
	"device_type",   // whole | mig | shared
	"profile",       // 3g.40gb, or "" 
	"attribution",   // exact | per-pid | per-pid-memory | shared-capped-estimate | shared-estimate
	"share_basis",   // exclusive | mig-compute | mig-memory | replica
	"util_field",    // DCGM_FI_PROF_GR_ENGINE_ACTIVE | DCGM_FI_DEV_GPU_UTIL
}

var (
	// Dimensionless shares — these are what the reconciliation identities test.
	AllocatedShare = prom.NewGaugeVec(prom.GaugeOpts{
		Name: "gpu_allocated_share",
		Help: "Pod's claim on one physical GPU, [0,1]. Per gpu_uuid, sum + unallocated == 1.",
	}, labels)
	UtilisedShare = prom.NewGaugeVec(prom.GaugeOpts{
		Name: "gpu_utilised_share",
		Help: "Pod's share of one physical GPU's actual work, in GPU-equivalents.",
	}, labels)

	// Dollars — share × rate. Derived, but emitted so PromQL stays readable.
	CostAllocated = prom.NewGaugeVec(prom.GaugeOpts{
		Name: "gpu_cost_allocated_per_hour",
		Help: "USD/hour this pod is charged for reserving GPU capacity.",
	}, labels)
	CostUtilised = prom.NewGaugeVec(prom.GaugeOpts{
		Name: "gpu_cost_utilised_per_hour",
		Help: "USD/hour of GPU work this pod actually performed.",
	}, labels)

	// The other side of the ledger. Without these, identity A cannot be tested.
	UnallocatedShare = prom.NewGaugeVec(prom.GaugeOpts{
		Name: "gpu_unallocated_share",
		Help: "Capacity on this GPU held by no pod, [0,1]. Includes MIG stranding.",
	}, []string{"gpu_uuid", "reason"}) // reason: free | mig-stranded | unhealthy

	DeviceUtilisation = prom.NewGaugeVec(prom.GaugeOpts{
		Name: "gpu_device_utilisation",
		Help: "Device-level utilisation as measured, for identity B.",
	}, []string{"gpu_uuid", "join_key", "util_field"})

	// Operational health of the exporter itself.
	MapAgeSeconds = prom.NewGauge(prom.GaugeOpts{
		Name: "gpu_attributor_map_age_seconds",
		Help: "Age of the ownership snapshot currently being served.",
	})
	Unresolved = prom.NewCounterVec(prom.CounterOpts{
		Name: "gpu_attributor_unresolved_device_ids_total",
		Help: "pod-resources device IDs the resolver did not recognise.",
	}, []string{"resource_name"})
)
```

**The two gauges answer different questions and have different denominators.**

- `gpu_allocated_share` is an **entitlement**. It comes from the allocation and nothing else. Summed per GPU with the unallocated remainder it is exactly 1. This is the chargeback number: the tenant reserved capacity and the platform could not sell it to anyone else, so the tenant pays whether or not they used it.
- `gpu_utilised_share` is **work performed**, in GPU-equivalents. Summed per GPU it equals the device's utilisation.

Their gap is therefore **signed**, and the sign is mechanically meaningful:

```
  gap = allocated_share − utilised_share

  gap > 0   the pod reserved more than it used.  IDLE WASTE.
            Possible under every mode. This is the number that finds
            the notebook holding an H100 at 2 % for three weeks.

  gap < 0   the pod used MORE than it reserved.
            IMPOSSIBLE under MIG or exclusive allocation — a hardware
            partition physically cannot exceed itself.
            EXPECTED under time-slicing and MPS, because the driver's
            channel scheduler is work-conserving: when a co-tenant has
            no work queued, the tenant that does gets the whole device
            for that slice. A pod holding 1 of 4 replicas can reach a
            utilised share of ~1.0 if the other three are idle.

  ⇒ A NEGATIVE GAP ON A device_type="mig" OR "whole" SERIES IS A BUG
    IN YOUR EXPORTER, NOT A FINDING. Alert on it. It means either a
    join key collided, or you divided by the wrong GPU count (§4), or
    a MIG series is being matched to the parent device's utilisation.
```

That last box is worth more than it looks: it is a **free correctness oracle** that costs one alert rule and catches the two most common arithmetic bugs in this build.

So report the gap as `clamp_min(allocated − utilised, 0)` when you want "waste", and report the raw signed value when you want "borrowing", and never let a dashboard silently take the absolute value.

### 8 — The reconciliation identities: the part that makes this defensible

This is the section that turns an exporter into an artifact. Two identities, both continuously testable, each catching a different class of bug.

```
  IDENTITY A — ALLOCATION CONSERVATION   (must hold EXACTLY)

  For every physical GPU g:
      Σ  allocated_share(p, g)   +   unallocated_share(g)   ≡   1.0
     p ∈ holders(g)

  For a node with G physical GPUs over a window W:
      Σ attributed GPU-hours  +  Σ idle GPU-hours  ≡  G × W

  ┌───────────────────────────────────────────────────────────────────┐
  │  ONE PHYSICAL GPU = 1.000 SHARE. ALWAYS. NO EXCEPTIONS.           │
  │                                                                   │
  │  whole GPU, held:      [████████████████████████████████] 1.000   │
  │  whole GPU, free:      [░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░] 1.000   │
  │                                                     ↑ unallocated │
  │                                                                   │
  │  MIG 2g+2g+3g, all held (compute basis):                          │
  │                        [██████▏██████▏████████████████] 1.000     │
  │                          .2857  .2857        .4286                │
  │                                                                   │
  │  MIG 7×1g.10gb, all held (MEMORY basis — note the stranding):     │
  │                        [██▏██▏██▏██▏██▏██▏██▏░░░░░░░░░] 1.000     │
  │                         7 × .1226 = .858    .142 mig-stranded     │
  │                                                                   │
  │  time-sliced replicas:4, 3 pods holding:                          │
  │                        [███████▏███████▏███████▏░░░░░░░] 1.000    │
  │                          .25     .25     .25    .25 free          │
  └───────────────────────────────────────────────────────────────────┘

  WHAT A FAILURE MEANS:
    sum > 1  → double-counted holders. Almost always a map[string]Owner
               where it should be map[string][]Owner, or a container
               counted twice because you walked initContainers wrongly.
    sum < 1  → an unresolved device ID (§3 branch 4), or MIG stranding
               you forgot to book, or a device that went Unhealthy and
               vanished from GetAllocatableResources (04.3 §7).


  IDENTITY B — UTILISATION CONSERVATION   (holds to within scrape jitter)

  For every measurement identity k (a physical GPU, or a GPU instance):
      Σ  utilised_share(p, k)   ≈   device_utilisation(k)
     p ∈ holders(k)

  WHAT A FAILURE MEANS:
    ratio = N exactly  → you forgot to deduplicate. The classic: sum()
                         over DCGM series that were fanned out across
                         N holders. Over-report factor is exactly the
                         holder count, independent of load (04.7 §7).
    ratio drifts 5–10% → NORMAL. The DCGM device counter and the NVML
                         per-PID samples use different clocks over
                         different windows. Pick a tolerance, alert
                         outside it, do not chase exactness.
```

As shipped PromQL — these go in a recording-rule file and an alert file, not in someone's browser history:

```promql
# ─── IDENTITY A: allocation conservation, per physical GPU ───────────────
# Must be 1.0. Anything else is a bug in the exporter.
record: gpu:allocation_conservation:ratio
expr: |
  (
      sum by (node, gpu_uuid) (gpu_allocated_share)
    + sum by (node, gpu_uuid) (gpu_unallocated_share)
  )

# ─── IDENTITY A, fleet form: GPU-hours over 24h ─────────────────────────
record: gpu:attributed_gpu_hours:24h
expr: sum by (node) (avg_over_time(gpu_allocated_share[24h])) * 24
record: gpu:idle_gpu_hours:24h
expr: sum by (node) (avg_over_time(gpu_unallocated_share[24h])) * 24
record: gpu:physical_gpu_hours:24h
expr: sum by (node) (gpu_physical_count) * 24
# gpu:attributed + gpu:idle  ==  gpu:physical.  Exactly.

# ─── IDENTITY B: utilisation conservation, per measurement identity ─────
record: gpu:utilisation_conservation:ratio
expr: |
    sum by (node, join_key) (gpu_utilised_share)
  / on (node, join_key) group_left()
    max by (node, join_key) (gpu_device_utilisation)

# ─── The alerts ─────────────────────────────────────────────────────────
- alert: GPUAllocationConservationViolated
  expr: abs(gpu:allocation_conservation:ratio - 1) > 0.001
  for: 5m
  annotations:
    summary: "GPU {{ $labels.gpu_uuid }} shares sum to {{ $value }}, not 1.0"
    runbook: "sum>1 → duplicate holders; sum<1 → unresolved ID or unbooked stranding"

- alert: GPUUtilisationConservationViolated
  expr: |
    gpu:utilisation_conservation:ratio > 1.10
      or gpu:utilisation_conservation:ratio < 0.90
  for: 15m

- alert: GPUUtilisedExceedsAllocatedOnPartitionedDevice
  # Physically impossible. A partition cannot exceed itself.
  expr: |
    (gpu_utilised_share - gpu_allocated_share) > 0.01
      and on(namespace, pod, container) gpu_allocated_share{device_type=~"mig|whole"}
  for: 5m

- alert: GPUAttributionMapStale
  expr: gpu_attributor_map_age_seconds > 120
  for: 5m

- alert: GPUAttributorUnresolvedDeviceIDs
  expr: increase(gpu_attributor_unresolved_device_ids_total[1h]) > 0
```

And the two queries that actually get looked at:

```promql
# Monthly chargeback per namespace. Bills reserved capacity, idle or not.
sum by (namespace) (gpu_cost_allocated_per_hour) * 730

# Monthly idle waste per namespace. clamp_min because a negative gap is
# borrowing, not waste, and must not be netted off against real waste.
sum by (namespace) (
  clamp_min(gpu_cost_allocated_per_hour - gpu_cost_utilised_per_hour, 0)
) * 730

# The number nobody computes and everybody needs:
# what fraction of the chargeback total rests on an ESTIMATE?
  sum(gpu_cost_allocated_per_hour{attribution=~"shared.*"})
/ sum(gpu_cost_allocated_per_hour)
```

If that last query returns 0.6, the correct next project is not a better dashboard — it is moving those workloads to MIG.

### 9 — Fleet-level error accounting: how wrong can the total be?

Lesson 04.7 quantified the per-pod error of a fair-share split on one device. The capstone needs the aggregate version, because that is the number you put in front of a stakeholder.

Start from the per-device result. For a device at utilisation `u` shared by `N` pods with true shares `s_i` (Σ s_i = 1), a fair-share estimator assigns `u/N` and the error for pod *i* is `err_i = u × (1/N − s_i)`.

Two properties follow immediately, and both are load-bearing:

```
  1. THE ERROR IS PURELY REDISTRIBUTIVE.
       Σ err_i = u × (Σ 1/N − Σ s_i) = u × (1 − 1) = 0
     The device total is EXACTLY right. Every dollar is accounted for;
     some of them are on the wrong invoice.

  2. THE PER-POD ERROR IS BOUNDED BY THE DEVICE'S SHARE OF THE FLEET.
       |err_i| ≤ u × max(1/N, 1 − 1/N) ≤ u
     A pod on a shared device can be mis-billed by at most that
     device's entire utilised cost, and only in the degenerate case
     where one tenant does all the work.
```

Now aggregate. Let the fleet have `G` physical GPUs, of which `G_s` run a software sharing mode. Let `f = G_s / G` be the **shared fraction**, and let `ū_s` be mean utilisation on shared devices.

```
  FLEET CHARGEBACK ERROR BOUND

    total chargeback        = G × rate × mean allocated share
    dollars resting on an
    estimate                = G_s × rate × ū_s          ← the "exposure"
    exposure fraction       = (G_s × ū_s) / (G × ū_all)

    MAXIMUM MISATTRIBUTION between any two namespaces
                            ≤ exposure          (redistributive, so the
                                                 total is never wrong)

  WORKED — a 64-GPU fleet at $4.00/GPU-hour:
    40 GPUs exclusive or MIG,  mean util 0.62   →  attribution="exact"
    24 GPUs time-sliced ×4,    mean util 0.91   →  attribution="shared-estimate"

    exact utilised GPU-equivalents   = 40 × 0.62 = 24.80
    estimated utilised GPU-equiv.    = 24 × 0.91 = 21.84
                                                   ─────
    total                                          46.64

    exposure fraction = 21.84 / 46.64 = 46.8 %

    → 46.8 % of every utilisation-based chargeback dollar on this fleet
      is an estimate. At $4.00/GPU-hour × 730 h that is
      21.84 × $4.00 × 730 = $63,773/month sitting on rung 5.

    IF per-PID (rung 3) works on those 24 GPUs, exposure drops to ~0 %
    and the same $63,773 becomes attribution="per-pid".
    THAT is the business case for hostPID and the NVML probe (§6),
    stated in dollars rather than in engineering preference.
```

Note the shape of the argument: you are not claiming the total is wrong. You are claiming a specific, quantified fraction of the *split* is an estimate, and offering a concrete change that reduces it. That is a defensible position in a chargeback dispute, and it is unavailable to anyone who emits one unlabelled number.

**The allocation side has no such exposure**, which is why `gpu_cost_allocated_per_hour` is the metric you actually bill from. Allocated shares come from the allocation, not from a measurement — identity A holds exactly, and the number survives any argument about counters. Utilised cost is the *efficiency* signal, and it is where the honesty labels live.

### 10 — Degradation: what the exporter does when its inputs lie

Each of the four sources fails differently. Design for each, and expose which one is broken.

| Source | Failure | Symptom if unhandled | Correct behaviour |
|---|---|---|---|
| pod-resources | `ResourceExhausted` (16 MiB ceiling) | **all** pod labels vanish at once; graphs read as "GPUs went idle" | serve cached snapshot, `Stale=true`, bump `map_age_seconds` |
| pod-resources | `ResourceExhausted` (100 QPS bucket) | same, but self-inflicted and permanent under a retry loop | exponential backoff, distinct error label |
| pod-resources | kubelet restart, socket inode replaced | client hangs on a dead FD forever | mount the *directory*; redial on error |
| NVML | `NOT_SUPPORTED` on `GetProcessUtilization` | rung 3 silently degrades to rung 5 with no label change | probe at startup, record result, force the label down |
| NVML | driver upgrade mid-scrape (04.5) | UUIDs re-enumerate; join keys go stale | invalidate inventory on `nvidia.com/cuda.driver-version.full` label change |
| DCGM | exporter pod restarting | series disappear; utilised cost → 0 | keep emitting allocated cost; do **not** emit utilised as 0 |
| cgroup | PID exited between NVML sample and `/proc` read | log spam, or a dropped share | treat as normal, skip the PID, renormalise |
| node labels | rate label missing | cost is 0 or NaN | emit shares only; alert; never guess a rate |

**The rule that covers all of them: never emit a zero you did not measure.** A missing utilisation is not 0.0 utilisation. Absent series are honest; zero-valued series are a claim. This is the single most important operational property of this exporter, because a false zero in a cost dashboard is indistinguishable from a real efficiency win and will be reported as one.

### 11 — The failure-mode log

The deliverable requires `failure-modes.md` with **at least five real entries**, each `symptom → evidence → root cause → fix → prevention`. Roll up what you actually hit across the module — the entries below are the ones this build reliably produces, but only write up the ones you genuinely reproduced:

- **04.2** — driver/toolkit version mismatch, or the `no runtime for 'nvidia' is configured` family.
- **04.4** — a container with `NVIDIA_VISIBLE_DEVICES=all` seeing devices the kubelet never assigned it. This one is *also* an attribution failure: identity A fails because a pod is using a GPU it does not hold, and no map can see it. Worth writing up from the attribution angle specifically.
- **04.5** — a driver upgrade that re-enumerated UUIDs mid-window and produced a discontinuity in the cost series.
- **04.6** — `mig.config.state=failed` with `In use by another client`, plus the discovery that `mig-manager` pauses operands but does not drain your workloads.
- **04.7** — the naive `sum()` over fanned-out DCGM series over-reporting by exactly the holder count.
- **Attribution-specific** — pod-resources socket unreadable (hostPath or permissions); a UUID present in DCGM but missing from the map (pod deleted mid-scrape → serve-from-cache); a shared device with no per-PID data (probe, degrade, relabel); an unresolved device-ID shape.

This is the artifact that survives an interview follow-up ("tell me about a GPU incident you actually debugged") better than anything else in the module, and it is the one piece that cannot be reconstructed after the fact from memory.

## Perspectives

**Systems-integration view.** Nothing here is individually hard — you have built or read every component. What is hard is composing four sources that fail independently into one process that stays useful when any one of them misbehaves. That is the daily work of a staff platform engineer: not inventing mechanism, but building the thing that sits on everyone else's mechanism and has to keep working when theirs does not. The `Stale` flag and the `attribution` label are both instances of the same discipline — make the degraded state visible in the data rather than silently indistinguishable from the healthy one.

**FinOps view.** The two-gauge design exists because one number lies by omission. Allocated-only gives correct chargeback and erases the waste signal — nobody discovers the idle H100. Utilised-only undercounts what finance pays, because the node is billed whether the GPU is busy or not. The gap between them, clamped at zero, is usually the single most actionable number this whole module produces. And the *negative* gap — a tenant using more than it reserved — is a second finding hiding in the same subtraction: it says your sharing is working and your replica count could probably come down.

**Auditor's view.** A number you cannot check is not a measurement, it is an assertion. Identity A is what makes this pipeline auditable: hand someone `sum(allocated) + sum(unallocated)` per GPU and they can verify it equals the physical GPU count without understanding a single thing about MIG or DCGM. That is the property that makes the artifact defensible in a room where nobody else knows what a compute slice is.

**Site-reliability view.** This exporter must be more reliable than the thing it measures. A dashboard that blanks on a transient socket error, or emits a wildly wrong number because a pod was deleted between the DCGM scrape and the map lookup, is worse than no dashboard — it teaches people to route around the data. Serve-from-cache, label staleness, and never emit an unmeasured zero. Those are not polish; they are the difference between an exporter people trust and one they ignore.

**Epistemics view.** The `attribution` label is the most important design decision in this capstone and the easiest to skip under time pressure. An estimate presented with the same confidence as a measurement is *worse* than no data, because it becomes ground truth the moment someone screenshots it into a finance deck. The correct engineering response to "I cannot measure this precisely" is not to fake precision and not to refuse to answer — it is to compute the best available estimate, quantify its error (§9), and say in the data itself that it is an estimate.

## Real-world use cases

- **[NVIDIA/dcgm-exporter](https://github.com/NVIDIA/dcgm-exporter) — the reference implementation, read at tag `4.6.0-4.8.3`.** This capstone re-derives its spine: the pod-resources join, the `pod`/`namespace`/`container` attributes, the NVML MIG hop, the 16 MiB receive ceiling, the `stripVGPUSuffix` handling of `::N`, and the cgroup PID→pod regex. Read `internal/pkg/transformation/kubernetes.go` and `process_metrics.go` and diff your implementation against them. What it *stops short of* is the cost layer, the share model, and the reconciliation identities — which is precisely the gap this deliverable fills.
- **`dcgm-exporter`'s own degradation choice is a case study.** When the pod-resources call fails, `Process()` logs a warning and returns `nil`: the scrape succeeds, but every series ships without pod labels for that interval. A `sum by (namespace)` dashboard then shows a hole that reads as "usage dropped to zero". You inherit that failure mode by copying the pattern, and §10's serve-from-cache is the fix.
- **[NVIDIA/dcgm-exporter#642](https://github.com/NVIDIA/dcgm-exporter/issues/642) and #587** — the dated, on-hardware evidence that the many:1 case is not a textbook exercise. #587 states the finding in its title (`--kubernetes-virtual-gpus` exports identical values for all pods rather than per-pod values); #642 is a practitioner on Blackwell reporting utilisation, power and temperature all identical across sharers. Cross-referenced from lesson 04.7. *(Located via search across this module's research; GitHub HTML is not fetchable through this environment's egress proxy — read the threads yourself for current status, since 4.6.0's per-process split for the two `DEV` fields may have changed them.)*
- **The 4.6.0 per-process split, as an example of reading source over docs.** `isPerProcessMetric()` is nine lines and it tells you something no release note does: exactly two fields get per-pod treatment, and every `PROF` field still fans out. That asymmetry determines which rung of §6's ladder your fleet can reach, and you would not learn it from a changelog.
- **Honest gap.** No independently verified named-company engineering blog describing a homegrown per-pod GPU cost exporter of this exact shape could be fetched this session — this environment's egress proxy blocks essentially all blog and vendor-doc domains. Rather than cite something unread, this lesson's mechanisms all come from sources that *were* read directly: the kubelet source, the device-plugin source, NVIDIA's NVML header and profile definitions, and dcgm-exporter's source. Commercial FinOps vendors do sell products in this space and their marketing describes the same gap; treat that as corroboration to check, not as a source.

## Worked example

A complete, end-to-end build on one node, with every number reconciled. This is the deliverable; follow it.

### The lab

One node, `gpu-node-01`, 8 × H100 80GB, deliberately heterogeneous so all three code paths are exercised:

| GPU | mode | configuration | holders |
|---|---|---|---|
| 0 | MIG `mixed` | `2g.20gb ×2 + 3g.40gb` (zero-stranding, 04.6 §2) | 3 pods |
| 1 | time-sliced | `replicas: 4` | 3 pods |
| 2 | exclusive | — | 1 pod |
| 3–7 | free | — | 0 |

Node rate **$32.00/hr** across **8 physical GPUs** → **$4.00 per physical GPU-hour**.

```bash
kubectl label node gpu-node-01 lab/node-hourly-rate=32.00 --overwrite
```

### Step 1 — the layout

```
practice/per-pod-attribution/
├── cmd/gpu-attributor/main.go        # two loops, /metrics, /healthz
├── internal/
│   ├── podresources/                 # from 04.3 — List() → []RawDevice
│   ├── inventory/                    # NVML: devices, MIG hop, per-PID
│   ├── attribute/                    # Resolve, AllocatedShare, the ladder
│   └── metrics/                      # the GaugeVecs from §7
├── deploy/
│   ├── daemonset.yaml
│   ├── rules-recording.yaml          # the identities
│   └── rules-alerting.yaml
├── failure-modes.md                  # ≥5 entries
└── README.md                         # run instructions + the caveats
```

### Step 2 — the main loop

```go
// cmd/gpu-attributor/main.go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"

	"github.com/you/per-pod-attribution/internal/attribute"
	"github.com/you/per-pod-attribution/internal/inventory"
	"github.com/you/per-pod-attribution/internal/metrics"
	"github.com/you/per-pod-attribution/internal/podresources"
)

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	// NVML first: if it is unavailable, MIG and per-PID are both impossible
	// and we must know that at startup, not discover it silently at scrape time.
	inv, err := inventory.New(ctx)
	if err != nil {
		slog.Error("NVML init failed; MIG and per-PID attribution unavailable", "err", err)
		os.Exit(1)
	}
	defer inv.Close()

	// PROBE, do not assume. Record the answer; it decides the ladder rung
	// this node can reach and it is not portable across driver/SKU.
	caps := inv.ProbeCapabilities()
	slog.Info("attribution capabilities probed",
		"driver", caps.DriverVersion, "model", caps.Model,
		"per_pid_sm", caps.ProcessUtilization, // NVML_ERROR_NOT_SUPPORTED?
		"per_pid_mem", caps.ProcessMemory,
		"host_pid", caps.HostPIDVisible)

	pr, err := podresources.New(podresources.DefaultSocket)
	if err != nil {
		slog.Error("pod-resources dial", "err", err)
		os.Exit(1)
	}
	defer pr.Close()

	a := attribute.New(pr, inv, caps, attribute.Config{
		NodeName:       os.Getenv("NODE_NAME"),
		DCGMEndpoint:   "http://127.0.0.1:9400/metrics",
		OwnershipEvery: 10 * time.Second,
		UtilEvery:      15 * time.Second,
		ShareBasis:     os.Getenv("SHARE_BASIS"), // "compute" | "memory"
	})

	go a.OwnershipLoop(ctx) // LOOP A
	go a.UtilisationLoop(ctx) // LOOP B

	reg := prometheus.NewRegistry()
	metrics.MustRegister(reg)

	mux := http.NewServeMux()
	mux.Handle("/metrics", promhttp.HandlerFor(reg, promhttp.HandlerOpts{}))
	mux.HandleFunc("/healthz", func(w http.ResponseWriter, _ *http.Request) {
		// Healthy != fresh. Report age; let the alert decide.
		if age := a.MapAge(); age > 2*time.Minute {
			http.Error(w, "map stale: "+age.String(), http.StatusServiceUnavailable)
			return
		}
		w.Write([]byte("ok"))
	})

	srv := &http.Server{Addr: ":9401", Handler: mux}
	go func() { _ = srv.ListenAndServe() }()
	<-ctx.Done()
	shutdown, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	_ = srv.Shutdown(shutdown)
}
```

And the join, which is the heart of it:

```go
// internal/attribute/join.go

// Publish joins the ownership snapshot to utilisation and emits every series.
func (a *Attributor) Publish(snap *Snapshot, util UtilSample) {
	metrics.MapAgeSeconds.Set(time.Since(snap.Built).Seconds())

	// Per physical GPU, accumulate allocated share so we can emit the
	// unallocated remainder and make identity A testable.
	allocByGPU := map[string]float64{}

	for joinKey, holders := range snap.Owners {
		u, haveUtil := util.ByJoinKey[joinKey] // device- or instance-level

		// Per-PID split, only for shared devices and only if the ladder allows.
		var perPod map[string]float64 // podUID -> normalised share of u
		rung := "shared-estimate"
		if len(holders) > 1 {
			switch {
			case a.caps.ProcessUtilization && a.caps.HostPIDVisible:
				if s := util.PerPIDSM[joinKey]; len(s) > 0 {
					perPod, rung = normaliseByPod(s, a.pidToPodUID), "per-pid"
				}
			case a.caps.ProcessMemory && a.caps.HostPIDVisible:
				if s := util.PerPIDMem[joinKey]; len(s) > 0 {
					perPod, rung = normaliseByPod(s, a.pidToPodUID), "per-pid-memory"
				}
			}
			if perPod == nil && a.cfg.SharingStrategy == "mps" {
				rung = "shared-capped-estimate" // 04.8: bounded by the thread cap
			}
		} else {
			rung = "exact"
		}

		for _, o := range holders {
			allocByGPU[o.PhysicalUUID] += o.AllocatedShare
			lv := labelValues(o, rung, util.Field)

			metrics.AllocatedShare.WithLabelValues(lv...).Set(o.AllocatedShare)
			metrics.CostAllocated.WithLabelValues(lv...).
				Set(o.AllocatedShare * a.cfg.RatePerGPUHour)

			// NEVER emit a zero you did not measure: if utilisation is
			// unavailable, delete the utilised series rather than setting 0.
			if !haveUtil {
				metrics.UtilisedShare.DeleteLabelValues(lv...)
				metrics.CostUtilised.DeleteLabelValues(lv...)
				continue
			}

			var us float64
			switch {
			case len(holders) == 1 && o.Kind == KindMIG:
				us = o.AllocatedShare * u // instance counters × slice share
			case len(holders) == 1:
				us = u // whole GPU
			case perPod != nil:
				us = u * perPod[o.PodUID] // measured split
			default:
				us = u / float64(len(holders)) // fair share, labelled
			}
			metrics.UtilisedShare.WithLabelValues(lv...).Set(us)
			metrics.CostUtilised.WithLabelValues(lv...).Set(us * a.cfg.RatePerGPUHour)
		}
	}

	// The other side of identity A. Without this the identity is untestable.
	for uuid, dev := range snap.Devices {
		alloc := allocByGPU[uuid]
		if s := dev.MIGStrandedShare(a.cfg.ShareBasis); s > 0 {
			metrics.UnallocatedShare.WithLabelValues(uuid, "mig-stranded").Set(s)
			alloc += s
		}
		metrics.UnallocatedShare.WithLabelValues(uuid, "free").Set(1.0 - alloc)
	}
}
```

### Step 3 — deploy

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-attributor
  namespace: gpu-operator
spec:
  selector:
    matchLabels: { app: gpu-attributor }
  template:
    metadata:
      labels: { app: gpu-attributor }
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9401"
    spec:
      serviceAccountName: gpu-attributor      # needs: get/list/watch pods (informer)
      nodeSelector:
        nvidia.com/gpu.present: "true"
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      # Required for /proc/<pid>/cgroup on tenant PIDs (§6 rung 3).
      # Drop this and every shared device is capped at rung 5.
      hostPID: true
      containers:
        - name: attributor
          image: your-registry/gpu-attributor:v0.1.0
          ports: [{ name: metrics, containerPort: 9401 }]
          env:
            - name: NODE_NAME
              valueFrom: { fieldRef: { fieldPath: spec.nodeName } }
            - name: SHARE_BASIS
              value: "compute"
            # NVML without CUDA. This is the ONE legitimate use of
            # NVIDIA_VISIBLE_DEVICES=all (04.4) — pair it with a policy
            # that forbids it on tenant workloads.
            - name: NVIDIA_VISIBLE_DEVICES
              value: "all"
            - name: NVIDIA_DRIVER_CAPABILITIES
              value: "utility"
          securityContext:
            privileged: true
          volumeMounts:
            # Mount the DIRECTORY: the socket inode is recreated on
            # kubelet restart (04.3 pitfall 2).
            - name: pod-resources
              mountPath: /var/lib/kubelet/pod-resources
              readOnly: true
      volumes:
        - name: pod-resources
          hostPath: { path: /var/lib/kubelet/pod-resources, type: Directory }
```

### Step 4 — the raw scrape

```console
$ kubectl -n gpu-operator exec ds/gpu-attributor -- \
    curl -s localhost:9401/metrics | grep -E '^gpu_(allocated|utilised)_share|^gpu_unallocated' | sort
```

```
# GPU 0 — MIG, three instances, all held. attribution=exact.
gpu_allocated_share{namespace="team-a",pod="ft-0",container="cuda",gpu_uuid="GPU-0a…",join_key="0-1",device_type="mig",profile="3g.40gb",attribution="exact",share_basis="mig-compute",util_field="DCGM_FI_PROF_GR_ENGINE_ACTIVE"} 0.428571
gpu_allocated_share{namespace="team-b",pod="srv-0",container="s",gpu_uuid="GPU-0a…",join_key="0-7",device_type="mig",profile="2g.20gb",attribution="exact",share_basis="mig-compute",util_field="DCGM_FI_PROF_GR_ENGINE_ACTIVE"} 0.285714
gpu_allocated_share{namespace="team-c",pod="srv-1",container="s",gpu_uuid="GPU-0a…",join_key="0-8",device_type="mig",profile="2g.20gb",attribution="exact",share_basis="mig-compute",util_field="DCGM_FI_PROF_GR_ENGINE_ACTIVE"} 0.285714
gpu_utilised_share{…pod="ft-0"…}   0.338571     # 0.428571 × u(GI 1)=0.79
gpu_utilised_share{…pod="srv-0"…}  0.157143     # 0.285714 × u(GI 7)=0.55
gpu_utilised_share{…pod="srv-1"…}  0.014286     # 0.285714 × u(GI 8)=0.05
gpu_unallocated_share{gpu_uuid="GPU-0a…",reason="free"} 0.000000

# GPU 1 — time-sliced replicas:4, three holders, per-PID resolved.
gpu_allocated_share{namespace="team-a",pod="nb-0",…,gpu_uuid="GPU-1b…",join_key="GPU-1b…",device_type="shared",attribution="per-pid",share_basis="replica",…} 0.250000
gpu_allocated_share{namespace="team-b",pod="nb-1",…,attribution="per-pid",…} 0.250000
gpu_allocated_share{namespace="team-c",pod="nb-2",…,attribution="per-pid",…} 0.250000
gpu_utilised_share{…pod="nb-0"…}   0.587708     # 0.91 × 62/96
gpu_utilised_share{…pod="nb-1"…}   0.237083     # 0.91 × 25/96
gpu_utilised_share{…pod="nb-2"…}   0.085313     # 0.91 ×  9/96
gpu_unallocated_share{gpu_uuid="GPU-1b…",reason="free"} 0.250000

# GPU 2 — exclusive.
gpu_allocated_share{namespace="team-a",pod="train-0",…,device_type="whole",attribution="exact",share_basis="exclusive",…} 1.000000
gpu_utilised_share{…pod="train-0"…} 0.880000
gpu_unallocated_share{gpu_uuid="GPU-2c…",reason="free"} 0.000000

# GPUs 3–7 — nobody asked for them.
gpu_unallocated_share{gpu_uuid="GPU-3d…",reason="free"} 1.000000
… (×5)
```

### Step 5 — verify identity A by hand, then in PromQL

```
  IDENTITY A, per GPU:

  GPU 0 (MIG):    0.428571 + 0.285714 + 0.285714 + 0.000000 = 1.000000  ✓
                  (2/7 + 2/7 + 3/7 — the zero-stranding geometry, 04.6 §2)
  GPU 1 (sliced): 0.250000 × 3       + 0.250000            = 1.000000  ✓
                  (the 4th replica is PLATFORM overhead, not tenant cost)
  GPU 2 (whole):  1.000000           + 0.000000            = 1.000000  ✓
  GPUs 3–7:       0.000000           + 1.000000 × 5        = 5.000000  ✓
                                                             ─────────
  NODE TOTAL:     2.750000 attributed + 5.250000 idle     = 8.000000
                                                             = 8 PHYSICAL GPUs ✓

  IN GPU-HOURS over a 1-hour window:
      2.75 attributed GPU-hours + 5.25 idle GPU-hours = 8.00 GPU-hours
      = 8 GPUs × 1 hour.   THE IDENTITY HOLDS EXACTLY.

  IN DOLLARS at $4.00 per physical GPU-hour:
      tenants   2.75 × $4.00 = $11.00 /hr
      platform  5.25 × $4.00 = $21.00 /hr
                              ────────
      total                    $32.00 /hr  = THE NODE RATE.  ✓
      Every dollar the cloud bills has exactly one owner.
```

```console
$ promtool query instant http://prometheus:9090 \
    'sum by (gpu_uuid) (gpu_allocated_share) + sum by (gpu_uuid) (gpu_unallocated_share)'
{gpu_uuid="GPU-0a…"} => 1 @[…]
{gpu_uuid="GPU-1b…"} => 1 @[…]
{gpu_uuid="GPU-2c…"} => 1 @[…]
{gpu_uuid="GPU-3d…"} => 1 @[…]
… 8 series, all exactly 1
```

### Step 6 — verify identity B, and see the naive query fail

```
  IDENTITY B, per measurement identity:

  GPU 0, instance 0-1:  0.338571 / 0.428571 = 0.790  vs u(GI 1) = 0.79  ✓
    (for MIG each instance has ONE holder, so the check is per instance
     and reduces to "did you multiply by the right share")
  GPU 1 (one device, three holders):
        0.587708 + 0.237083 + 0.085313 = 0.910104
        device u = 0.91                             ratio 1.0001  ✓
  GPU 2:  0.880000 / 0.88 = 1.000                              ✓
```

Now reproduce the bug this whole design exists to prevent, so you have it on record:

```console
# ❌ THE NAIVE QUERY — sum over fanned-out DCGM series.
$ promtool query instant http://prometheus:9090 \
    'sum(DCGM_FI_PROF_GR_ENGINE_ACTIVE{Hostname="gpu-node-01"})'
{} => 5.62 @[…]

# ✅ WHAT IT SHOULD BE:
$ promtool query instant http://prometheus:9090 \
    'sum(gpu_utilised_share{node="gpu-node-01"})'
{} => 3.79 @[…]
```

Read the difference. The device-level series for GPU 1 (`0.91`) is emitted once per holder, so the naive sum counts it three times: `3 × 0.91 = 2.73` where the truth is `0.91`. Over-report `1.82`. Plus the MIG instances, which are legitimately separate series and are *not* double-counted. `5.62 − 3.79 = 1.83`, matching the over-count to within rounding. **The over-report factor on the affected device is exactly the holder count, independent of load** (lesson 04.7 §7) — which is why it never averages out and why the nodes you shared hardest to be efficient are the ones your dashboard flatters most.

### Step 7 — the money queries

```console
$ promtool query instant http://prometheus:9090 \
    'sum by (namespace) (gpu_cost_allocated_per_hour) * 730'
{namespace="team-a"} => 4562.0   # 1.00 + 0.25 + 0.4286 = 1.6786 × $4 × 730
{namespace="team-b"} => 1564.3   # 0.2857 + 0.25       = 0.5357 × $4 × 730
{namespace="team-c"} => 1564.3   # 0.2857 + 0.25       = 0.5357 × $4 × 730
$ # platform overhead: 5.25 × $4 × 730 = $15,330

$ promtool query instant http://prometheus:9090 \
    'sum by (namespace) (clamp_min(gpu_cost_allocated_per_hour - gpu_cost_utilised_per_hour, 0)) * 730'
{namespace="team-a"} => 613.2
{namespace="team-b"} => 1204.5
{namespace="team-c"} => 2036.5   ← team-c's 2g.20gb slice at 5 % util
```

`team-c` is paying $1,564/month for a MIG slice running at 5 %, and the exporter says so with `attribution="exact"` — no estimate involved, nothing to dispute. **That single row is the business case for the whole build.**

And the borrowing case, which only exists under sharing:

```console
$ promtool query instant http://prometheus:9090 \
    'gpu_utilised_share - on(namespace,pod,container) gpu_allocated_share'
{namespace="team-a",pod="nb-0",device_type="shared"} => 0.337708   ← used 0.59 of
                                                          a 0.25 reservation
{namespace="team-a",pod="ft-0",device_type="mig"}    => -0.090000
{namespace="team-c",pod="srv-1",device_type="mig"}   => -0.271428
```

`nb-0` is positive: it borrowed idle time from its co-tenants, which is exactly what a work-conserving time-slice scheduler is supposed to do. Every `device_type="mig"` and `"whole"` row is negative, as it must be — a hardware partition cannot exceed itself. **A positive value on a MIG or whole row would be a bug in the exporter**, and that is the alert from §7.

### Step 8 — the exposure number

```console
$ promtool query instant http://prometheus:9090 \
    'sum(gpu_cost_allocated_per_hour{attribution=~"shared.*"}) / sum(gpu_cost_allocated_per_hour)'
{} => 0 @[…]
```

Zero, because per-PID resolved on all three shared holders — every series on this node is `exact` or `per-pid`. Now prove that the number is real by breaking it: remove `hostPID: true` from the DaemonSet, redeploy, and re-run:

```console
$ promtool query instant … 'sum(gpu_cost_allocated_per_hour{attribution=~"shared.*"}) / sum(gpu_cost_allocated_per_hour)'
{} => 0.272727 @[…]     # 0.75 of 2.75 attributed shares
```

27.3 % of the chargeback total on this node now rests on a fair-share estimate whose per-pod error, from the measured shares above, is up to **0.34 GPU-equivalents** — $1.35/hour, ~$986/month, misattributed between `team-a` and its neighbours. That is the dollar value of `hostPID: true`, measured rather than argued.

## Practice

**This section is the module deliverable** — [../practice/per-pod-attribution/README.md](../practice/per-pod-attribution/README.md). "Shipping it" means all of the following are true, committed, and demonstrable on a live GPU node.

1. **Mapper.** Poll pod-resources (the 04.3 client) on a ticker; build `joinKey → []Owner` — a *slice* of owners, so a many:1 device is representable. Add a pod informer on this node that nudges an out-of-cycle rebuild, debounced. Serve the cached snapshot on error with `Stale=true` and expose `gpu_attributor_map_age_seconds`.
2. **Resolver.** Handle all four ID shapes: `GPU-…`, `GPU-…::N`, `MIG-…` (both UUID formats, via the NVML hop to `"<gpuIndex>-<GPU_I_ID>"`), and DRA `dynamic_resources`. Emit `gpu_attributor_unresolved_device_ids_total` for anything else — do not silently drop.
3. **Capability probe.** At startup, probe `nvmlDeviceGetProcessUtilization`, `nvmlDeviceGetComputeRunningProcesses`, and whether `/proc/<pid>/cgroup` is readable for a tenant PID. Log the result with driver version and SKU. **Write it down in the deliverable README** — it is not portable and it determines which ladder rung this fleet reaches.
4. **Join.** Read DCGM (`DCGM_FI_PROF_GR_ENGINE_ACTIVE`, fallback `DCGM_FI_DEV_GPU_UTIL`) and attach owner labels. MIG joins on `"<gpuIndex>-<GPU_I_ID>"`, not on the MIG UUID.
5. **Share model.** `AllocatedShare` per §5, with `share_basis` emitted as a label; MIG stranding booked to `gpu_unallocated_share{reason="mig-stranded"}`; shared devices divided by `replicas`, not by holder count.
6. **Ladder.** Implement rungs 1–5 with the `attribution` label, and make the label move when a rung fails. Never emit a zero you did not measure.
7. **Two gauges plus two cost gauges**, with the full label set from §7, and the rate derived from a node label divided by the *physical* GPU count from NVML.
8. **The identities, shipped.** `deploy/rules-recording.yaml` and `deploy/rules-alerting.yaml` containing both identities and all five alerts from §8. Demonstrate each alert firing at least once — including by deliberately introducing the `map[string]Owner` bug and watching `GPUAllocationConservationViolated` catch it.
9. **The naive-query proof.** Capture `sum(DCGM_FI_PROF_GR_ENGINE_ACTIVE)` over-reporting against `sum(gpu_utilised_share)`, and show the difference equals `(holders − 1) × u` on the shared device.
10. **The exposure measurement.** Compute the estimate-exposure fraction with and without per-PID available, and convert the difference into dollars per month at your rate.
11. **`failure-modes.md`** — **≥5 real entries**, each `symptom → kubectl/log evidence → root cause → fix → prevention`, rolled up from 04.2 / 04.5 / 04.6 plus the attribution-specific ones from §11.

**Acceptance = the module checkpoint** ([../checkpoint.md](../checkpoint.md)). Concretely, on a live GPU node:

- `curl` the exporter and show per-pod series carrying `namespace`, `pod`, `gpu_uuid`, `join_key`, `device_type`, `profile`, `attribution`, `share_basis` and both share gauges plus both cost gauges.
- Demonstrate a **MIG** pod (`attribution="exact"`, share = the profile fraction) and a **time-sliced** set (`attribution="per-pid"` or `"shared-estimate"`, shares summing to the device utilisation within tolerance).
- Show **identity A returning exactly 1.0 for every physical GPU**, and the node total equalling the physical GPU count.
- Show **identity B within your stated tolerance**, and the naive `sum()` failing by exactly the holder count.
- Show the **allocated-minus-utilised gap negative on every MIG/whole series** and positive where a shared pod borrowed — and explain why the reverse would be a bug.
- Commit the code, a scrape sample, the two rule files, and `failure-modes.md` with its ≥5 entries.

## Common pitfalls

1. **`map[string]Owner` instead of `map[string][]Owner`.** The single highest-impact bug in this build. A shared device has several holders; a scalar map keeps whichever was written last, so one pod is charged for the whole card and the others for nothing. Identity A catches it immediately (sums to less than 1) — which is why you build identity A first.
2. **Dividing the node rate by `nvidia.com/gpu.count`.** Under MIG `single` that label counts MIG devices (56 on an 8-GPU node); under time-slicing it counts replicas. Divide by the *physical* GPU count from NVML. Getting this wrong is a silent 7× or 4× error in every dollar figure and identity A will not catch it, because the shares are still internally consistent.
3. **Joining MIG on the MIG UUID.** DCGM labels MIG samples with the *parent* `UUID` plus `GPU_I_PROFILE` / `GPU_I_ID`. Without the NVML hop your MIG series simply never match and MIG pods get no utilisation at all — which looks like "the MIG pods are idle", a plausible and completely wrong finding.
4. **Emitting a zero you did not measure.** A missing utilisation is not `0.0`. Absent series are honest; zero-valued series are a claim, and a false zero in a cost dashboard is indistinguishable from a real efficiency win.
5. **Netting the signed gap.** `allocated − utilised` goes negative under work-conserving sharing. Summing signed gaps across a namespace lets a borrowing pod cancel out a genuinely idle one, hiding the waste. Use `clamp_min(…, 0)` for waste and report borrowing separately.
6. **Dividing shared allocation by holder count instead of `replicas`.** With `replicas: 4` and 3 pods, dividing by 3 socialises the unallocated replica onto tenants, converting a platform over-provisioning problem into a tenant cost they cannot act on. Book it to overhead.
7. **Presenting `shared-estimate` values with the same confidence as `exact` ones.** An unlabelled estimate becomes a false fact the moment someone screenshots it into a finance report. The label must survive into every dashboard built on these metrics, and the exposure query in §8 must be on the same dashboard.
8. **Blanking on a transient pod-resources error.** A gap that reads "the GPUs stopped being used" is worse than clearly-timestamped stale data. Serve from cache, flag staleness, alert on age.
9. **Assuming DRA removes the utilisation join.** A claim tells you who *holds* a device; it does not change the fact that a time-sliced device reports one number for all its sharers. The cardinality problem is orthogonal to which API supplied the ownership map.
10. **Skipping the failure-mode log because "the exporter works".** It is an explicit checkpoint criterion and the one artifact that cannot be reconstructed afterwards from memory.

## Self-check

- **A time-sliced device has three holders. Walk the attribution, name the fallback signal, and state what it costs you if it is unavailable.** *Answer:* The **map does not collapse** — the device plugin fabricates annotated IDs `GPU-<uuid>::<n>`, so pod-resources returns three distinct strings and you know exactly who the three holders are. Strip `::n` to get the physical UUID, which is the DCGM join key. The **metric** collapses: DCGM programs counters per device, so `DCGM_FI_PROF_GR_ENGINE_ACTIVE{UUID="GPU-…"}` is one value for three claimants and a naive `sum()` over-reports by exactly three, independent of load. Fall back to per-process accounting: NVML `nvmlDeviceGetProcessUtilization` gives `pid → smUtil` (sampling interface, bounded ring buffer, `NVML_ERROR_NOT_SUPPORTED` on some driver/SKU combinations), resolve each PID to a pod by reading `/proc/<pid>/cgroup` and matching `pod([a-f0-9_-]+)` with `_`→`-` substitution, then split the device utilisation proportionally and label `attribution="per-pid"`. If it is unavailable, fall to `u / holders` and label `attribution="shared-estimate"` — the total still reconciles exactly (the error is purely redistributive, `Σ err_i = 0`), but each pod's error is `u × (1/N − s_i)`, up to the device's entire utilised cost in the degenerate case. In the worked example that was 27.3 % of the node's chargeback total and ~$986/month of misattribution — which is the measured cost of not having `hostPID: true`.

- **Why does the allocated-minus-utilised gap go negative, and what does a negative gap on a MIG series mean?** *Answer:* The two gauges have different denominators. `allocated_share` is an entitlement derived from the allocation — for a time-sliced pod, `1/replicas`. `utilised_share` is work performed, in GPU-equivalents. Under time-slicing and MPS the driver's channel scheduler is **work-conserving**: when co-tenants have nothing queued, the tenant that does gets the whole device for that slice. So a pod holding one of four replicas can reach a utilised share near 1.0 while its allocated share stays 0.25 — a large negative gap, meaning it borrowed idle capacity. That is correct and is a signal your replica count could come down. Under MIG or exclusive allocation it is **physically impossible**: a hardware partition cannot exceed itself, and the instance's counters only ever measure that instance. So a negative gap on `device_type="mig"` or `"whole"` is a bug in the exporter, not a finding — most likely a join-key collision, a wrong physical-GPU denominator, or a MIG series matched against the parent device's utilisation. It is worth an alert, because it is a free correctness oracle that catches the two most likely arithmetic errors in the build.

- **State both reconciliation identities, say what each one catches, and give the PromQL.** *Answer:* **Identity A — allocation conservation, exact.** For every physical GPU, `Σ allocated_share(holders) + unallocated_share ≡ 1.0`; over a window, attributed GPU-hours plus idle GPU-hours equals physical GPUs × wall time. PromQL: `sum by (gpu_uuid) (gpu_allocated_share) + sum by (gpu_uuid) (gpu_unallocated_share)`, alerting on `abs(… − 1) > 0.001`. A sum above 1 means duplicated holders — almost always a `map[string]Owner` where a slice was needed, or a container counted twice. A sum below 1 means an unresolved device-ID shape, MIG stranding you did not book, or a device that went `Unhealthy` and vanished from `GetAllocatableResources`. **Identity B — utilisation conservation, within jitter.** For every measurement identity, `Σ utilised_share(holders) ≈ device_utilisation`. PromQL: `sum by (join_key) (gpu_utilised_share) / on(join_key) group_left() max by (join_key) (gpu_device_utilisation)`, alerting outside 0.90–1.10. A ratio of exactly N means you forgot to deduplicate fanned-out DCGM series. A 5–10 % drift is normal — the DCGM device counter and the NVML per-PID samples run on different clocks over different windows — so pick a tolerance and alert outside it rather than chasing exactness. Note that identity B tests your *arithmetic*, not your *fidelity*: rungs 3–5 all satisfy it by construction because they are fractions of the same device number. Fidelity is what the `attribution` label is for.

- **How do you turn a device UUID plus utilisation into a per-pod dollar figure, and what exactly do you join?** *Answer:* Four inputs. (1) The **ownership map** — pod-resources `List` (or `ResourceClaim.status.allocation` on a DRA cluster), resolved to a join key: the bare UUID for whole/shared devices, `"<gpuIndex>-<GPU_I_ID>"` for MIG via the NVML hop. (2) The **rate**: a node-label hourly rate divided by the *physical* GPU count from NVML — never by `nvidia.com/gpu.count`, which counts MIG devices under `single` strategy and replicas under time-slicing. (3) The **allocated share**: 1.0 exclusive; the MIG profile fraction on a stated basis (`42/98` compute or `40192/80384` memory for a `3g.40gb` on an A100-80GB — these differ by up to 16 %, so emit `share_basis`); `1/replicas` for a shared device. (4) The **utilisation** for that join key — `DCGM_FI_PROF_GR_ENGINE_ACTIVE` preferred, `DCGM_FI_DEV_GPU_UTIL` as fallback — split per-PID where the device is shared and NVML supports it. Emit two gauges: `allocated = rate × allocated_share` (bills whether or not the GPU is busy; this is the chargeback number and it carries no measurement exposure) and `utilised = rate × utilised_share` (the efficiency number). Their clamped positive gap is idle waste; the negative direction is borrowing.

- **Your dashboard shows the fleet using 5.62 GPU-equivalents on an 8-GPU node where you know only three GPUs are held. What happened and how do you fix it?** *Answer:* You summed DCGM series that were fanned out across holders. On a shared device, `dcgm-exporter` labels the *same* device measurement once per pod holding a replica, so `sum()` counts it once per holder. The over-report factor is exactly the holder count, independent of load, and it does not average out — which means the nodes you shared hardest to be efficient are the ones the dashboard flatters most. In the worked example, GPU 1 at `u = 0.91` with three holders contributed `2.73` instead of `0.91`, an over-count of `1.82`. Two fixes. The minimal one is to divide by the fan-out degree before aggregating: `DCGM_FI_PROF_GR_ENGINE_ACTIVE / on(UUID) group_left() count by (UUID) (DCGM_FI_PROF_GR_ENGINE_ACTIVE)`, which reconciles the total but charges every pod `u/N` and must therefore be labelled as an estimate. The real fix is to sum your own `gpu_utilised_share`, which is constructed to sum to the device utilisation on every rung of the ladder, and to ship identity B as an alert so this class of bug cannot reach a dashboard again.

- **Why is MIG the "easy" attribution case despite being the harder feature to operate — and what still requires a judgement call even there?** *Answer:* MIG partitions the hardware, so each GPU instance has its **own performance counters**: DCGM measures instance 2 and only instance 2, giving one number per billable unit and a 1:1 join with no estimator. Time-slicing and MPS multiplex several pods onto one unpartitioned device at the software layer — trivial to configure, but structurally many:1 in the measurement, requiring a second independent per-process signal to split. Attribution difficulty tracks **whether the hardware or the software drew the sharing boundary**, not how hard the feature is to turn on. What still requires judgement under MIG: the *share* term. A `1g.10gb` on an A100-80GB is 14/98 = 14.3 % of the compute but 9856/80384 = 12.3 % of the framebuffer — a 16 % relative disagreement — so "exact" applies to the utilisation term, not to the share, and you must pick a basis and emit it as a label. And under a stranding geometry such as `7 × 1g.10gb`, the memory-basis shares sum to 0.858, not 1.0; the missing 14.2 % is real unallocatable capacity that must be booked to platform overhead or identity A will fail.

## Connections & what's next

This capstone is where every earlier lesson becomes one artifact. 04.1 and 04.2 keep the node healthy enough to produce real data and supply the failure-mode discipline. 04.3 supplies the mapper, the device-ID taxonomy, and the two `ResourceExhausted` traps. 04.4 explains why the device ID inside the container is trustworthy — and why `NVIDIA_VISIBLE_DEVICES=all` on a tenant workload breaks identity A in a way no map can detect. 04.5 is why the exporter must invalidate its inventory on a driver-version label change. 04.6, 04.7 and 04.8 are the three sharing-mode branches the ladder detects, with 04.6 also supplying the NVML hop and the stranding term. 04.9 supplies the DRA ownership path and the quota layer these numbers eventually justify.

If you can explain, cold, why time-slicing breaks the metric but not the map, and then show an exporter that handles both *and* proves its own arithmetic with a conservation identity — you have answered this module's hardest interview probe before anyone asks it.

Forward: the per-pod cost and efficiency signal you just built is the raw material for **module 05 (GPU observability)**, which turns single exporters into fleet-wide dashboards, SLOs and alerting — and where the `gpu:utilisation_conservation:ratio` recording rule becomes a first-class data-quality SLI rather than a debugging aid. It also feeds **module 06 (scheduling and capacity)**, where knowing precisely what each pod costs and how efficiently it used its device becomes an input to bin-packing, preemption and capacity planning: you cannot schedule for cost efficiency until you can measure cost per pod, which is exactly what this capstone produces. And module 11 (GPU cost economics) takes `gpu_cost_allocated_per_hour` across providers with different billing models and asks the harder question of what a GPU-hour is actually worth.

## References & further reading

**Primary sources — read directly this session (August 2026)**

- [NVIDIA/dcgm-exporter](https://github.com/NVIDIA/dcgm-exporter) — read at tag `4.6.0-4.8.3`. The reference implementation this capstone re-derives and extends. `internal/pkg/transformation/kubernetes.go` (the pod-resources join, `stripVGPUSuffix` for `::N`, the NVML MIG hop, the DRA branch, the 16 MiB `kubeletPodResourcesMaxRecvMsgSize`, and the `ResourceExhausted` degradation path that drops all pod labels for a scrape); `process_metrics.go` (`isPerProcessMetric` — **only** `DCGM_FI_DEV_GPU_UTIL` and `DCGM_FI_DEV_FB_USED` get a per-process split; every `PROF` field still fans out); `pidmapper.go` (the `pod([a-f0-9_-]+)` cgroup regex and the `_`→`-` substitution); `nvmlprovider/provider.go` (`GetProcessUtilization(0)`, `GetComputeRunningProcesses`, `GetMIGDeviceInfoByID` with both UUID formats); `rendermetrics/render_metrics.go` (the MIG label set: parent `UUID` + `GPU_I_PROFILE` + `GPU_I_ID`, and no MIG-UUID label); `collector/types.go` (`GetIDOfType` producing the `"<gpu>-<GPU_I_ID>"` key); `transformation/const.go` (the `pod` / `namespace` / `container` / `pod_uid` / `vgpu` attribute names and the DRA label names).
- [kubelet pod-resources API — `k8s.io/kubelet/pkg/apis/podresources/v1`](https://github.com/kubernetes/kubelet/tree/master/pkg/apis/podresources) and [`kubernetes/kubernetes` `pkg/kubelet/apis/podresources/`](https://github.com/kubernetes/kubernetes/tree/master/pkg/kubelet/apis/podresources) — the `ListPodResources` message shapes the mapper consumes, and the server-side `DefaultQPS = 100` / `DefaultBurstTokens = 10` rate limit. Full treatment in lesson 04.3.
- [NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) — read on `main` at the v0.19.x line. `internal/rm/devices.go` (`NewAnnotatedID` producing `GPU-<uuid>::<replica>`, the fact the whole many:1 design rests on), the `nvidia.com/gpu.sharing-strategy` / `gpu.replicas` / `mig.strategy` node labels the exporter branches on, and the MPS-experimental / MPS-not-on-MIG constraints.
- [NVIDIA/go-nvml — `pkg/nvml/mock/gpus/`](https://github.com/NVIDIA/go-nvml) and the NVML header — the per-SKU GPU-instance profile info (`multiprocessorCount`, `memorySizeMB`) behind the MIG share arithmetic, and the `nvmlProcessUtilizationSample_t` / `nvmlProcessInfo_v2_t` structs behind the per-PID fallback. Full tables in lesson 04.6 §3.
- [kubernetes-sigs/dra-driver-nvidia-gpu](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu) — read if you build the optional DRA-sourced ownership path; the claim and allocation shapes, and `shareID` for consumable capacity. Full treatment in lesson 04.9.

**Bug reports and field evidence**

- [NVIDIA/dcgm-exporter#587 — "--kubernetes-virtual-gpus exports identical values for all pods instead of per-pod utilization"](https://github.com/NVIDIA/dcgm-exporter/issues/587) — the thesis of this capstone's fallback path stated in a title. *(Located via search across this module's research; GitHub HTML is not fetchable through this environment's egress proxy.)*
- [NVIDIA/dcgm-exporter#642 — "Why are DCGM_FI_DEV_GPU_UTIL values not isolated per vGPU/Pod?"](https://github.com/NVIDIA/dcgm-exporter/issues/642) — Blackwell-generation hardware, identical utilisation/power/temperature across all sharers. Note that 4.6.0's per-process split for `DCGM_FI_DEV_GPU_UTIL` may have changed the status of this specific report — check before quoting it as current. *(Same fetch caveat.)*
- [NVIDIA/dcgm-exporter#151 — "no metrics labels about pod namespace/name when Pod uses time slicing GPU"](https://github.com/NVIDIA/dcgm-exporter/issues/151) — the inverse failure, and the diagnostic signature of `::N` handling being absent. *(Same fetch caveat.)*

**Deeper dives**

- [Lesson 04.3 — Device-plugin recap & the kubelet pod-resources API](03-device-plugin-recap-pod-resources.md) — the mapper, and the four device-ID shapes the resolver must handle.
- [Lesson 04.6 — MIG operations & per-slice attribution](06-mig-operations.md) — the NVML hop, the profile tables behind the share arithmetic, and the stranding term that must be booked for identity A to hold.
- [Lesson 04.7 — Time-slicing and the attribution trap](07-time-slicing-attribution.md) — the per-device error arithmetic this lesson aggregates to the fleet in §9.
- [Lesson 04.8 — MPS and choosing a sharing mode](08-mps-choosing-sharing.md) — why MPS earns `shared-capped-estimate` rather than `shared-estimate`, and the three-way decision matrix these numbers eventually inform.
