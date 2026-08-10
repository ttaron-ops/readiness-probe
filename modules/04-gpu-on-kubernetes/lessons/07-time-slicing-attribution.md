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
sources: 9
---

# 04.7 · Time-slicing and the attribution trap

> **Concept.** Time-slicing multiplies one GPU into N schedulable replicas that share it by taking turns — with no memory isolation, no fault isolation, and no per-pod allocation signal — so cost cannot be attributed from allocation counts and must fall back to DCGM per-pod utilisation.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Lesson 06 gave you the clean case: MIG carves a GPU into hardware partitions, each with its own UUID, so `nvidia.com/gpu` allocation counts translate directly into a defensible bill — one partition, one UUID, one tenant, one line item. This lesson gives you the *hard* case, and it's the one you'll actually meet most often, because MIG requires an Ampere-or-newer datacenter GPU and a drain, while time-slicing is one ConfigMap key on anything. Here the clean allocation-to-cost mapping breaks, and you'll prove exactly how and why, using a real GitHub issue filed by a practitioner who hit this on current-generation Blackwell hardware. What you build here — the fallback attribution path — is the piece of the capstone (lesson 10) that handles every GPU that isn't MIG-partitioned.

## Why this matters

Time-slicing is the cheapest, most common, most abused way to put more than one pod on a GPU: change `replicas: 1` to `replicas: 4` in a ConfigMap, and a node that advertised `nvidia.com/gpu: 1` now advertises `nvidia.com/gpu: 4`. Utilization on idle inference fleets jumps, the scheduler stops wedging pods in `Pending`, and everyone is happy — until finance asks the question this course's whole differentiator is built on: **who owes what for that GPU?**

Under time-slicing the honest answer is *you cannot tell from Kubernetes alone*. Every one of the N replicas resolves to the same physical GPU UUID. The allocation-count signal that works perfectly for whole-GPU and MIG scheduling — count `nvidia.com/gpu` requests per namespace, multiply by $/GPU-hour — silently produces garbage: it either double-counts the physical GPU up to N times, or, if you divide evenly by N, charges an idle tenant exactly as much as the tenant pinning every SM.

This is the module README's named interview probe, verbatim: **"attribute cost to a time-sliced GPU."** It is called out as "your home turf" for a reason — most candidates either don't know time-sliced GPUs share a UUID, or they know it but can't name the fallback signal cleanly under interview pressure. The senior answer is not "use time-slicing" or "avoid it" — it is naming precisely which signal died (per-pod allocation), which signal you fall back to (DCGM per-pod utilization), and how you'd wire that fallback into a real cost operator. That answer is what this lesson builds, and — as you'll see below — it isn't hypothetical: a practitioner on brand-new Blackwell hardware hit exactly this wall in production and filed it as a bug against NVIDIA's own reference exporter.

It matters twice over because time-slicing also removes the two isolation guarantees an operator quietly relies on. A neighbor pod can exhaust VRAM and OOM you — a fault your own workload did nothing to cause. If your cost report can't see the neighbor, your incident review can't either.

## What's new here (calibration)

Module 04's README already told this module to skip: device-plugin gRPC *API mechanics* (owned in module 02), the DRA *object model* (module 02), Topology Manager internals (module 02b), and GPU/MIG *at the silicon level* (module 03). Lesson 06 covered MIG's clean attribution case, and lesson 03 gave you the pod-resources API as a Go client. This lesson does **not** re-teach any of that. What it adds:

- **The specific mechanism by which the allocation signal collapses** under sharing — not "sharing is complicated" but the exact gRPC field (`device_ids`) that goes from unique-per-pod (MIG, lesson 06) to identical-across-pods (time-slicing, here).
- **A primary-source, currently-open GitHub issue** proving this isn't textbook theory — a real engineer on real Blackwell-class hardware hit it and documented the exact symptom.
- **The fallback attribution formula** — utilization-weighted cost splitting from DCGM profiling metrics — worked through with real numbers, plus which specific DCGM metric name is safe to use and which is not.
- **The isolation failure modes** (VRAM OOM, fault propagation) that a cost/reliability operator must encode as caveats on any time-sliced bill, because the bill and the incident are the same underlying fact.

## Core concepts

### 1 — Enabling time-slicing via the device-plugin ConfigMap

With the GPU Operator (the standard path), time-slicing is a sharing config the `nvidia-device-plugin` consumes. You author a ConfigMap of named configs and tell the plugin which one to apply, cluster-wide or per-node via a node label. As of NVIDIA's [k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) `v0.17.1`, the schema below is current and confirmed directly against the plugin's own README.

```yaml
# time-slicing-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: gpu-operator
data:
  any: |-
    version: v1
    flags:
      migStrategy: none
    sharing:
      timeSlicing:
        # failRequestsGreaterThanOne: true makes a request of >1 replica fail,
        # so a pod cannot silently ask for "2 slices" and imply isolation it
        # will not get. Keep this true — it is a guardrail against the trap.
        failRequestsGreaterThanOne: true
        resources:
          - name: nvidia.com/gpu
            replicas: 4
```

Apply it and point the operator's `ClusterPolicy` (or the plugin, if run standalone) at it:

```bash
kubectl create -f time-slicing-config.yaml

# Tell the operator which config to use by default:
kubectl patch clusterpolicy/cluster-policy \
  -n gpu-operator --type merge \
  -p '{"spec":{"devicePlugin":{"config":{"name":"time-slicing-config","default":"any"}}}}'
```

You can also opt in **per node** so only a labelled pool is sliced, leaving training nodes exclusive:

```bash
kubectl label node <node> nvidia.com/device-plugin.config=any
```

After the plugin restarts, capacity multiplies. A single physical H100 node:

```bash
$ kubectl describe node <node> | grep -A3 Allocatable
Allocatable:
  nvidia.com/gpu:  4          # was 1 — one physical GPU, now 4 replicas
```

Confirm the labelled state and that this is *shared*, not partitioned, silicon:

```bash
$ kubectl get node <node> -o json \
    | jq '.metadata.labels | with_entries(select(.key|test("nvidia.com")))'
{
  "nvidia.com/gpu.count": "1",              # ONE physical GPU
  "nvidia.com/gpu.product": "NVIDIA-H100-80GB-HBM3",
  "nvidia.com/gpu.replicas": "4",           # advertised as 4
  "nvidia.com/gpu.sharing-strategy": "time-slicing"
}
```

`gpu.count: 1` next to `nvidia.com/gpu: 4` allocatable is the whole story in two lines: **one device, four tickets.**

### 2 — Property one: no memory or fault isolation

Time-slicing gives every client the **full framebuffer** and the full SM array on its turn. There is no VRAM cap, no MPS memory limit, no MIG wall. Two consequences a cost/reliability operator must encode:

- **VRAM is first-come, uncapped.** If pod A allocates 78 GB of an 80 GB card, pod B's next `cudaMalloc` returns `out of memory` and the framework aborts — B is OOM-killed by CUDA, not by the kubelet, so it shows up as a **CUDA error / non-zero exit**, not a Kubernetes `OOMKilled` (that condition only tracks host-RAM cgroups). Your incident tooling that greps for `OOMKilled` will miss it entirely.
- **A fatal fault is not contained.** An ECC double-bit error, an Xid 43/74, or a wedged kernel on the shared device degrades or halts *every* co-resident client. There is no blast-radius boundary — the boundary is the physical GPU.

This is why time-slicing is for **cooperative, trusted, bursty** workloads (dev notebooks, low-QPS inference of your own services) and never for hostile multi-tenancy.

### 3 — Property two: no per-pod allocation signal (the attribution trap)

Here is the mechanism that breaks cost attribution. Query the kubelet **pod-resources API** — the same gRPC service you built a Go client against in lesson 03, and the authoritative record of which device IDs each pod was handed — for two pods scheduled onto the sliced GPU:

```bash
# On the node, over the kubelet's pod-resources unix socket:
$ grpcurl -plaintext -unix \
    /var/lib/kubelet/pod-resources/kubelet.sock \
    v1.PodResourcesLister/List | jq '.pod_resources[]
      | select(.namespace=="tenants")
      | {pod: .name, dev: .containers[].devices[].device_ids}'
{ "pod": "infer-a", "dev": ["GPU-8f2c...-e91a"] }
{ "pod": "infer-b", "dev": ["GPU-8f2c...-e91a"] }   # SAME UUID
```

Both pods received the **identical physical GPU UUID** `GPU-8f2c…-e91a`. The device plugin never minted per-replica identities — the replicas are indices into one device, and only the underlying UUID is reported. Contrast this directly with MIG (lesson 06), where each partition gets its own distinct UUID and this query would have returned two *different* strings — that single difference is the entire reason MIG attribution is "solved" and time-slicing attribution is not. Therefore:

```
allocation_count(tenant) × $per_gpu_hour   →  MEANINGLESS
```

- Sum the `nvidia.com/gpu` requests and you count the same physical GPU up to N times → you bill 4× the hardware that exists.
- Divide the GPU's cost evenly by N and an **idle** replica is charged the same as a replica saturating the SMs → you punish the frugal tenant and subsidize the greedy one. Allocation count carries **zero** information about who used the device, because allocation is uniform by construction.

The allocation signal is dead. Do not try to resurrect it with heuristics — there is no clever query over Kubernetes objects that recovers information the device plugin never reported.

### 4 — Proof this is a live, current problem: dcgm-exporter#642

This is not a corner case a lesson author invented to make a point. On **August 10, 2026** — checked directly against NVIDIA's own reference telemetry exporter — this exact failure mode is an open, unresolved GitHub issue: **[NVIDIA/dcgm-exporter#642](https://github.com/NVIDIA/dcgm-exporter/issues/642)**, titled *"Why are DCGM_FI_DEV_GPU_UTIL values not isolated per vGPU/Pod?"*

A practitioner running **Blackwell-generation hardware**, testing time-slicing with `KUBERNETES_VIRTUAL_GPUS=true`, reports — quoting the issue verbatim — that:

> "all telemetry values (Utilization, Power, Temp) are identical for all pods sharing the same GPU."

Read that carefully: it isn't just the *allocation* signal (pod-resources `device_ids`) that collapses under time-slicing — the reporter found that even the **basic DCGM utilization metric itself**, `DCGM_FI_DEV_GPU_UTIL`, reports the *same device-wide number* to every pod sharing the device. That's a stronger and more damaging finding than the pod-resources collision alone: it means the naive fix ("just read GPU_UTIL per pod from DCGM") doesn't work either, because `DCGM_FI_DEV_GPU_UTIL` is explicitly documented as **not MIG-compatible and not per-replica isolated** — it is a whole-device counter, full stop, regardless of how many logical pods are layered on top.

The fix, confirmed from the same source material, is to use a **different, more modern DCGM field**: **`DCGM_FI_PROF_GR_ENGINE_ACTIVE`** — part of DCGM's profiling module — which is the MIG-aware, engine-level activity metric designed to be attributable at finer granularity than the legacy `GPU_UTIL` counter. This is the load-bearing correction for anyone building a GPU cost exporter in 2026: don't reach for `DCGM_FI_DEV_GPU_UTIL` as your per-pod signal under any sharing mode — reach for the `DCGM_FI_PROF_*` family, and even then, verify empirically on your own SKU/driver combination that the field carries per-pod resolution, because dcgm-exporter#642 proves that "should be per-pod" and "is per-pod on this hardware" are not the same claim.

### 5 — The fallback: DCGM per-PID / per-pod utilization

Given the above, the only layer that can distinguish tenants on a shared device is **usage**, keyed to a process, sourced from DCGM's profiling metrics rather than its legacy device-wide counters. `dcgm-exporter` running as the GPU Operator's DaemonSet enriches per-GPU metrics with the owning pod, and — where the profiling module resolves correctly on your hardware — can attribute to **PID → container → pod**:

```bash
# Per-pod SM occupancy on the shared GPU (Prometheus form):
DCGM_FI_PROF_SM_ACTIVE{gpu="0", pod="infer-a", namespace="tenants"}  0.61
DCGM_FI_PROF_SM_ACTIVE{gpu="0", pod="infer-b", namespace="tenants"}  0.12
DCGM_FI_DEV_FB_USED{gpu="0", pod="infer-a"}                          74112   # MiB
DCGM_FI_DEV_FB_USED{gpu="0", pod="infer-b"}                          2048
```

Now attribution has a defensible basis: share the GPU's $/hr **in proportion to integrated SM-active time (or FB-used, per your policy) per pod**, not in proportion to allocation:

```
cost(pod) = gpu_hourly_cost
          × ∫ DCGM_FI_PROF_SM_ACTIVE[pod] dt  /  Σ_pods ∫ SM_ACTIVE dt
```

For your Go cost operator this is the time-sliced hard case: watch pods with the `nvidia.com/gpu.sharing-strategy=time-slicing` node label, join pod-resources (who is on which physical UUID — this tells you the denominator, the set of pods contending for the same device) with DCGM profiling series (how much each used — this splits the numerator), and emit a *utilization-weighted* charge.

> **Note on the profiling metrics — this is exactly what dcgm-exporter#642 warns you about.** `DCGM_FI_PROF_*` fields (SM active/occupancy, graphics-engine active, tensor active) come from the profiling module and on some driver/GPU combinations may still be collected device-wide rather than per-process, especially on newer architectures where support is still maturing. **Verify on your own SKU** that the field you're relying on actually carries a distinct value per pod under your sharing configuration before you trust it in a bill — the dcgm-exporter maintainers had not resolved issue #642 as of this writing, so "should work" is not "confirmed working" on every generation. If a metric only resolves per-GPU on your hardware, fall back to `DCGM_FI_DEV_GPU_UTIL` with best-effort PID association and **state the precision limit in the report**. Naming the limit explicitly, rather than presenting a falsely-precise number, is the senior move — and it's the same standard of honesty the issue's own reporter is holding NVIDIA to.

## Perspectives

**Scheduler's-eye view.** To `kube-scheduler`, `replicas: N` is a pure advertisement multiplier — it inflates `Allocatable.nvidia.com/gpu` on the node object and nothing else. The scheduler's bin-packing math (`NodeResourcesFit`) treats four time-sliced replicas exactly like four independent whole GPUs would be treated; it has no concept of "these four allocations point at the same silicon." The lie lives entirely in the device plugin's advertisement, not in the scheduler.

**Tenant/incident-responder view.** From inside a pod on a sliced GPU, the failure modes are counter-intuitive: your process can die from a CUDA out-of-memory error caused by a neighbor you can't see and didn't request. Because the kubelet's own `OOMKilled` reason is scoped to host-RAM cgroup evictions, standard alerting built around that reason string is silently blind to the single most common time-slicing incident. Anyone building on-call runbooks for a shared-GPU fleet has to add a second detector for CUDA-level OOM (non-zero exit + `CUDA out of memory` in logs) that has nothing to do with the kubelet's own OOM machinery.

**FinOps/attribution view.** Allocation-based billing under time-slicing isn't merely imprecise — it's actively adversarial. It systematically rewards the tenant who requests the most replicas regardless of use, and punishes the tenant who is frugal, because cost is split by ticket count, not by work done. Any billing model that produces perverse incentives (encouraging tenants to over-request "slices" just to lower their per-slice share) is a bug in the FinOps design, not a quirk to route around after the fact.

**Telemetry/observability-engineer view.** dcgm-exporter#642 matters beyond this one lesson because it's evidence that even NVIDIA's own reference GPU-metrics exporter — the tool most GPU-fleet operators build their cost and health dashboards on top of — has an acknowledged, open gap on exactly the sharing mode this lesson covers. If you are building or evaluating a GPU observability stack, the practical takeaway is: don't assume "NVIDIA ships it, so it must resolve correctly" for shared-GPU telemetry. Verify field-by-field on your generation of hardware.

## Real-world use cases

- **["Why are DCGM_FI_DEV_GPU_UTIL values not isolated per vGPU/Pod?" — NVIDIA/dcgm-exporter#642](https://github.com/NVIDIA/dcgm-exporter/issues/642).** A practitioner on Blackwell-generation hardware, time-slicing enabled via `KUBERNETES_VIRTUAL_GPUS=true`, reports that "all telemetry values (Utilization, Power, Temp) are identical for all pods sharing the same GPU." **Primary-source, currently-open proof of this lesson's entire thesis** — not a synthesized example, a real engineer's real bug report against real hardware.
- **[ScaleOps — "Kubernetes GPU Sharing"](https://scaleops.com/blog/kubernetes-gpu-sharing/).** A practitioner comparison of time-slicing, MPS, and MIG covering isolation and attribution trade-offs — useful as a second, independent voice reaching the same conclusions as this lesson. *(Not independently fetched this session — the research proxy blocked the domain; the URL and its subject were corroborated via search but content could not be directly re-verified. Spot-check before relying on specifics.)*
- **["GPU cost attribution for Kubernetes is here" — CloudZero](https://www.cloudzero.com/blog/gpu-cost-attribution-kubernetes/).** A cost-observability vendor's post that states the same problem in FinOps language: DCGM reports at the physical-GPU level and, in their framing, "cannot attribute compute to individual virtual replicas" — a useful non-NVIDIA voice independently arriving at the identical conclusion this lesson does. *(Not independently fetched this session — proxy-blocked; treat specifics as unverified until you can load the page yourself.)*

## Worked example — bill two tenants on one sliced H100

Setup: one H100-80GB, `replicas: 4`, two tenant pods land on it for a 1-hour window. Node $/hr = $2.40 (a worked placeholder for 2026 — refresh against your actual cloud/on-prem rate at study time).

Observed, integrated over the hour from DCGM:

| Pod       | ∫ SM_ACTIVE (SM-hours) | Peak FB used |
|-----------|------------------------|--------------|
| `infer-a` | 0.55                   | 74 GB        |
| `infer-b` | 0.10                   | 2 GB         |

**Naive allocation split (the trap):** each pod holds 1 replica of 4 → `$2.40 / 4 = $0.60` each, and the other two idle replicas' `$1.20` vanishes or gets mis-socialized. `infer-b`, nearly idle, is billed the same as `infer-a`. Finance cannot defend this and neither can you.

**Utilization-weighted split (the fallback, correct):**

```
total SM-hours = 0.55 + 0.10 = 0.65
infer-a = $2.40 × 0.55/0.65 = $2.03
infer-b = $2.40 × 0.10/0.65 = $0.37
```

The greedy tenant carries the cost; the frugal one pays for its ~15% of the work.

Note the **isolation footnote you must attach**: `infer-a` at 74 GB left `infer-b` only 6 GB of headroom — if `infer-b` had tried to load an 8 GB model it would have hit `CUDA out of memory` and died with a non-zero exit, *not* `OOMKilled`. The bill and the incident are the same physical fact: no isolation.

**And it's not hypothetical.** This exact class of problem — telemetry that can't distinguish co-resident pods on a shared GPU — is precisely what someone hit live and filed as [dcgm-exporter#642](https://github.com/NVIDIA/dcgm-exporter/issues/642) on current-generation hardware. If your utilization-weighted split above relies on a `DCGM_FI_PROF_*` field that turns out not to be per-pod-resolved on your specific driver/GPU combination, you will reproduce that exact bug inside your own cost operator — which is why the verification step in Core concepts §5 is not optional polish, it's the difference between a defensible bill and a silently wrong one.

## Practice

Produce two artifacts for the [Per-pod GPU attribution](../practice/per-pod-attribution/README.md) deliverable: a **neighbor-OOM demonstration** and a **same-UUID proof**. These feed directly into the module's failure-mode log.

1. **Enable slicing.** Apply the `replicas: 4` ConfigMap above, patch the ClusterPolicy (or label a node), and confirm `nvidia.com/gpu: 4` allocatable with `nvidia.com/gpu.count: 1`. Capture the `describe node` output and the label dump. (Any sub-$1/hr single-GPU rental is sufficient — you don't need an H100 for this, just a node the device plugin will slice.)
2. **Demonstrate no isolation (neighbor OOM).** Schedule two pods on the sliced GPU, each requesting `nvidia.com/gpu: 1`. In pod A, allocate most of the framebuffer (`torch.empty(int(N), dtype=torch.float16, device='cuda')` sized to ~90% of the card, or a `cudaMalloc` loop). In pod B, attempt a normal model load. Capture pod B's `CUDA out of memory` / non-zero exit and show `kubectl get pod` — note the status is **not** `OOMKilled`. Document that CUDA, not the kubelet, killed it.
3. **Prove no per-pod allocation signal (same-UUID).** From the node, query the pod-resources socket (the `grpcurl` form above) for both pods and show the `device_ids` are the **identical physical GPU UUID**. This is the proof that allocation counts cannot attribute cost — screenshot or paste this output; it's your evidence, not just an assertion.
4. **Show the fallback works — and verify its precision.** Scrape `dcgm-exporter` and capture `DCGM_FI_PROF_SM_ACTIVE` (or the field you determine is actually per-pod-resolved on your hardware) split per pod; compute the utilization-weighted charge as in the worked example. As part of this step, explicitly check whether the metric you're using is genuinely per-pod on your SKU or whether you've reproduced dcgm-exporter#642 locally — write down which case you're in.

**Acceptance:** a documented neighbor-OOM (pod B's CUDA OOM with pod status proving it was not a k8s `OOMKilled`) **and** the same-UUID proof (both pods' pod-resources `device_ids` equal to one physical GPU UUID), plus a one-paragraph written conclusion: allocation count is unusable under time-slicing; attribution uses DCGM per-pod utilization, with an explicit statement of whether your DCGM field resolved per-pod on your test hardware. These are the time-sliced hard case for the operator.

## Common pitfalls

1. **Believing `DCGM_FI_DEV_GPU_UTIL` gives you per-pod resolution under sharing.** It doesn't — it's explicitly documented as not MIG-compatible and not per-replica isolated, and dcgm-exporter#642 shows it reporting identical values across all co-resident pods on Blackwell hardware. Use `DCGM_FI_PROF_GR_ENGINE_ACTIVE` (or `DCGM_FI_PROF_SM_ACTIVE`) instead, and still verify per-pod resolution on your own generation.
2. **Trying to fix allocation-based billing with a cleverer formula.** There is no arithmetic transform of "4 identical allocations" that recovers who actually used the GPU. The information was never captured; no downstream query can resurrect it. You need a different signal (DCGM utilization), not a different formula over the same signal.
3. **Alerting only on Kubernetes `OOMKilled` for GPU memory pressure.** CUDA-level OOM from a time-sliced neighbor exits non-zero without ever setting that reason — your incident detection needs a second path (log-grep for `CUDA out of memory`, or exit-code + GPU-error correlation) that doesn't depend on the kubelet's cgroup-scoped OOM machinery.
4. **Assuming a `DCGM_FI_PROF_*` field "should" be per-pod because the docs describe it as fine-grained.** dcgm-exporter#642 is proof that "should" and "is, on this hardware" diverge in practice, on current-generation GPUs, in an issue that was still open as of this writing. Always spot-check on your actual SKU/driver combination before shipping a bill based on it.
5. **Treating time-slicing and MIG as interchangeable "GPU sharing" without naming which attribution regime you're in.** They produce the same `nvidia.com/gpu` resource name at the API surface but completely different attribution stories underneath — MIG gives distinct UUIDs and allocation-based billing works; time-slicing gives a shared UUID and allocation-based billing is meaningless. Conflating them in a design doc is a fast way to fail the "attribute cost to a time-sliced GPU" interview probe.

## Self-check

- Why can a neighbor pod OOM you under time-slicing — what isolation is missing? **Answer:** Time-slicing shares the device by time-multiplexing only; it provides no memory isolation and no fault isolation. Every client sees the full framebuffer with no per-client VRAM cap, so a neighbor that allocates most of the card leaves your next `cudaMalloc` to fail with `CUDA out of memory`. It surfaces as a CUDA error / non-zero exit, not a Kubernetes `OOMKilled` (that condition tracks host-RAM cgroups, not VRAM). A fatal GPU fault (Xid error, double-bit ECC) is likewise uncontained and hits all co-resident clients — the blast radius is the whole physical GPU.
- Why can't you attribute cost to a tenant from allocation counts under time-slicing? **Answer:** All N replicas are advertisements of one device; the pod-resources API hands every pod the same physical GPU UUID — confirmed both by direct `grpcurl` inspection and by the real-world dcgm-exporter#642 report of identical telemetry across co-resident pods. Allocation is uniform by construction, so counting `nvidia.com/gpu` requests either counts one GPU up to N times (over-bills the hardware that exists) or, split evenly, charges an idle replica the same as a saturating one. The allocation signal carries zero information about who actually used the SMs, so it cannot form a defensible bill.
- What signal can you attribute time-sliced usage with, and where does it come from — and what's the catch? **Answer:** Per-pod (per-PID) utilization from DCGM's profiling module — fields like `DCGM_FI_PROF_SM_ACTIVE` or `DCGM_FI_PROF_GR_ENGINE_ACTIVE` (the MIG-aware replacement for the legacy `DCGM_FI_DEV_GPU_UTIL`), associated to PID → container → pod, giving you a basis to split the GPU's $/hr in proportion to each pod's integrated SM-active time. The catch, proven by dcgm-exporter#642: these profiling fields aren't guaranteed to resolve per-pod on every GPU generation — you must verify empirically on your own hardware before trusting the number in a bill, and state the precision limit if it doesn't resolve cleanly.
- What's the concrete difference between `DCGM_FI_DEV_GPU_UTIL` and `DCGM_FI_PROF_GR_ENGINE_ACTIVE`, and why does it matter for this lesson? **Answer:** `DCGM_FI_DEV_GPU_UTIL` is a legacy, device-wide counter that is explicitly documented as not MIG-compatible and not per-replica isolated — under sharing, it reports the same number to every co-resident pod, exactly as dcgm-exporter#642 documents happening on Blackwell hardware. `DCGM_FI_PROF_GR_ENGINE_ACTIVE` comes from DCGM's newer profiling module and is the MIG-aware, finer-grained engine-activity metric meant to support attribution at sub-device granularity. It matters because reaching for the wrong one silently reproduces the exact bug in the GitHub issue inside your own exporter.

## Connections & what's next

This lesson is the "hard case" half of a pair with lesson 06: together, MIG (clean, allocation-based, distinct UUIDs) and time-slicing (collapsed, utilization-based, shared UUID) are the two ends of the attribution spectrum that the capstone's cost operator must branch on, keyed off the `nvidia.com/gpu.sharing-strategy` node label. Lesson 03's pod-resources Go client is the exact tool used here to *prove* the UUID collision — the same API, applied to answer a different question than it answered in lesson 03. The next lesson, MPS, adds a third sharing mode that looks superficially similar (same ConfigMap shape, same "no distinct UUID" attribution problem) but solves a different problem (throughput on under-filling jobs, not just overcommit) while making the fault-domain story *worse*, not better — carry the "allocation is dead, fall back to DCGM utilization" conclusion from this lesson straight into it, because it applies unchanged.

Next: **[04.8 · MPS and choosing a sharing mode](08-mps-choosing-sharing.md)** — the third point in the MIG / time-slicing / MPS decision triangle, and the one most engineers misplace.

## References & further reading

**Primary sources**
- [NVIDIA/dcgm-exporter#642 — "Why are DCGM_FI_DEV_GPU_UTIL values not isolated per vGPU/Pod?"](https://github.com/NVIDIA/dcgm-exporter/issues/642) — read for the primary-source proof of this lesson's entire thesis: a real practitioner on Blackwell hardware confirming identical telemetry across pods sharing a time-sliced GPU.
- [NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) (confirmed v0.17.1) — read for the authoritative `sharing.timeSlicing` ConfigMap schema and `failRequestsGreaterThanOne` semantics behind the GPU Operator wiring shown above.
- [NVIDIA/dcgm-exporter](https://github.com/NVIDIA/dcgm-exporter) — read as the reference implementation of the pod-label enrichment this lesson's fallback path depends on; the source to grep if you need exact flag names for your own exporter.
- [kubelet pod-resources API (`v1.PodResourcesLister`)](https://github.com/kubernetes/kubelet/tree/master/pkg/apis/podresources) — read for the exact gRPC contract (`List`, `GetAllocatableResources`, `Get`) used to pull the `device_ids` proof in this lesson's practice section.

**Real-world engineering blogs**
- ScaleOps, ["Kubernetes GPU Sharing"](https://scaleops.com/blog/kubernetes-gpu-sharing/) — practitioner comparison of the three sharing modes' isolation/attribution trade-offs. *(Not independently fetched — proxy-blocked; corroborated via search, spot-check before citing specifics.)*
- CloudZero, ["GPU cost attribution for Kubernetes is here"](https://www.cloudzero.com/blog/gpu-cost-attribution-kubernetes/) — a FinOps vendor's independent framing of the same DCGM device-vs-replica attribution gap. *(Not independently fetched — proxy-blocked; corroborated via search, spot-check before citing specifics.)*

**Deeper dives**
- NVIDIA GPU Operator docs, ["Time-Slicing GPUs in Kubernetes"](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html) — canonical config reference and ClusterPolicy wiring for the sharing config shown in this lesson. *(Not independently fetched — proxy-blocked this session; treat field names as needing a spot-check against your installed Operator version.)*
- The GitHub issue thread on dcgm-exporter#642 itself is worth reading past the opening report — maintainer/community replies (if any accumulate) are the fastest way to learn whether the profiling-metric gap has since been resolved on your hardware generation.
