# The fake GPU fleet — a 200-node lab that costs nothing

**Shared lab.** Modules 04, 05, 06, 11 and the capstone all need a *fleet* to build against,
and renting 200 H100s to learn scheduling is absurd. This lab gives you a fleet that is fake in
exactly one place — the silicon — and completely real everywhere that matters: the scheduler,
the allocation accounting, the queueing, the attribution, the cost model.

> **Adapted from** [`k8s-learn/gpu-platform-tasks.md`](https://github.com/harut8/system-design/blob/main/k8s-learn/gpu-platform-tasks.md)
> in [`harut8/system-design`](https://github.com/harut8/system-design), re-aimed at this course's
> deliverables. See [`docs/EXTERNAL-DEPTH.md`](../../../../../docs/EXTERNAL-DEPTH.md) for attribution
> and licensing.

## The idea that makes it work

Kubernetes never talks to a GPU. **The kubelet does.** Everything above the kubelet —
the scheduler, quota, Kueue, your operator, your dashboards, your cost model — only ever sees
an *integer on a node object* and *a metric in Prometheus*. Both are forgeable:

```
   ┌─────────────────────────────────────────────┐
   │  scheduler · Kueue · your operator · Grafana │   ← 100% real
   ├─────────────────────────────────────────────┤
   │  node.status.capacity["nvidia.com/gpu"]      │   ← forged (§1)
   │  DCGM_FI_PROF_* in Prometheus                │   ← forged (§3)
   ├─────────────────────────────────────────────┤
   │  the actual silicon                          │   ← absent
   └─────────────────────────────────────────────┘
```

Everything you build in Modules 04–06 and 11 lives in the top box. **Only Module 04's
CDI/driver/MIG lessons and Module 05's DCGM-internals lessons need real hardware** — and
those need *one* GPU for *one* afternoon, not a fleet (§5).

---

## §1 · One node that claims to have GPUs

Extended resources are advertised through the node **status** subresource, so you can simply
patch them in.

```bash
kind create cluster --name gpulab
kubectl proxy &

curl --header "Content-Type: application/json-patch+json" \
  --request PATCH \
  --data '[{"op":"add","path":"/status/capacity/nvidia.com~1gpu","value":"8"}]' \
  http://localhost:8001/api/v1/nodes/gpulab-worker/status
```

`nvidia.com~1gpu` is `nvidia.com/gpu` with the `/` JSON-Pointer-escaped. Pods requesting
`nvidia.com/gpu` will now schedule and bind. Nothing runs on a GPU — which is fine, because
the scheduling decision, the `Allocated resources` accounting, and every quota check are real.

**Verify the forgery took:**

```bash
kubectl describe node gpulab-worker | grep -A3 'Allocatable'
kubectl run t --image=busybox --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"t","image":"busybox","command":["sleep","3600"],
    "resources":{"limits":{"nvidia.com/gpu":"2"}}}]}}'
kubectl describe node gpulab-worker | grep -A6 'Allocated resources'
```

> **Gotcha.** The kubelet re-posts node status roughly every 5 minutes and will **erase** a
> patched extended resource unless a device plugin keeps re-advertising it. For a persistent
> single node, either re-patch on a loop or move to `kwok` (§2), where the fake kubelet is
> yours and the capacity sticks.

## §2 · A fleet, not a node

One node hides every problem worth studying. Fragmentation, bin-packing, gang deadlock,
topology-aware placement, and stranded capacity are all **fleet-shaped** — they do not exist
on a single node.

Use [`kwok`](https://kwok.sigs.k8s.io/) (Kubernetes WithOut Kubelet), which simulates nodes and
transitions pods to `Running` without any container runtime. Stand up **50–200 nodes**, and make
them **heterogeneous on purpose**:

| Axis | Spread it across |
|---|---|
| Accelerator type | `nvidia.com/gpu` (8/node), `nvidia.com/mig-1g.10gb` (56/node), a second vendor |
| GPU count | 8-GPU HGX nodes, 4-GPU nodes, 1-GPU inference nodes |
| Region / zone | 2–3 invented regions, so topology and egress cost mean something |
| Provider label | 2 invented providers with **different billing models** (§4) |
| NVLink domain | a `topology.lab/nvlink-island` label — 2 islands of 4 on each 8-GPU node |

Homogeneous fleets make every scheduler look good. The heterogeneity **is** the lab.

**Acceptance for §2:** `kubectl get nodes -L topology.lab/nvlink-island,lab/provider` shows a
fleet you can describe in one sentence, and a job requesting 8 GPUs on one NVLink island can
only land on a subset of nodes.

## §3 · Synthetic DCGM — the part that teaches the most

Write an exporter that emits **real DCGM metric names** with **plausible fake values**, per fake
GPU. Field names and IDs are in the source repo's
[DCGM field-ID cheat sheet](https://github.com/harut8/system-design/blob/main/gpu-observability/appendix-b-field-ids.md);
cross-check against your Module 05 lesson 01 notes.

At minimum:

| Metric | What it means | Why you need it |
|---|---|---|
| `DCGM_FI_DEV_GPU_UTIL` | fraction of time ≥1 kernel was resident — **not** occupancy | the lie |
| `DCGM_FI_PROF_SM_ACTIVE` | fraction of time ≥1 warp was active on an SM, averaged over SMs | the truth |
| `DCGM_FI_PROF_SM_OCCUPANCY` | resident warps ÷ max warps | the deeper truth |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | tensor-core pipe utilization | MFU's numerator |
| `DCGM_FI_DEV_FB_USED` / `_FREE` | framebuffer memory | the KV-cache ceiling |
| `DCGM_FI_DEV_POWER_USAGE` | watts | the honest utilization proxy |
| `DCGM_FI_DEV_XID_ERRORS` | hardware faults | Module 05 L5 |

**Emit three distinct workload *shapes*** — this is the whole point, because telling them apart
is what Modules 05 and 11 are actually about:

| Shape | `GPU_UTIL` | `SM_ACTIVE` | `SM_OCCUPANCY` | Power | Memory |
|---|---|---|---|---|---|
| **Training job** | ~95%, steady | ~85% | ~60% | near TDP | high, flat |
| **Inference server** | ~90%, bursty | ~25% | ~8% | ~40% TDP | high (KV cache) |
| **Idle-but-allocated notebook** | ~2% | ~0% | ~0% | idle | **held** |

The inference row is the util-lie in one line: `GPU_UTIL` 90% against `SM_OCCUPANCY` 8%. The
notebook row is the money: allocated, billed, and doing nothing. Your Module 05 dashboard and
Module 11 cost model must distinguish all three **from metrics alone**.

> **Make it fail, too.** Add flags to kill the exporter mid-scrape, stall a single GPU's
> `SM_ACTIVE` at 30% of its peers (a fail-slow straggler), and fire an XID 48. Staleness that
> reads as *zero utilization* will silently corrupt every downstream cost number — finding that
> yourself is worth more than reading about it.

## §4 · Two providers, two billing models

Attach a rate to each node via labels, and make the two providers **structurally different**:

| Provider | Billing model |
|---|---|
| `lab/provider=neocloud-a` | on-demand, per-second, single rate |
| `lab/provider=hyperscaler-b` | committed-use discount tier + on-demand overflow |

Normalising these into one comparable `$/GPU-hour` is much harder than it sounds, and it is
exactly the Module 11 deliverable. Doing it against synthetic data first means you arrive at the
real thing with the schema already argued out.

## §5 · The one afternoon of real hardware

Rent **one** GPU for a few hours (Lambda, RunPod, Vast.ai) at the end of Module 05 and validate
that your synthetic metrics match reality in **name, unit, and cardinality**. Run the real
`dcgm-exporter`, diff its `/metrics` output against your synthetic exporter's, and fix every
discrepancy.

This converts *"I simulated this"* into *"I verified my simulation against a real device"* —
which is the difference between an artifact that survives an interview follow-up and one that
doesn't.

## Which modules this unblocks

| Module | What it can now build without GPUs |
|---|---|
| [04 GPU on Kubernetes](../../README.md) | allocation accounting, quota, the pod-resources join (mock the socket), DRA/ResourceClaim scheduling |
| [05 GPU observability](../../../05-gpu-observability/practice/gpu-dashboard-lie/) | the entire allocated-vs-utilised dashboard, cardinality budget, XID alerting |
| [06 Scheduling & capacity](../../../06-scheduling-capacity/practice/kueue-showback/) | Kueue cohorts/borrowing/preemption, gang deadlock, fragmentation math, a custom `Score` plugin |
| [11 GPU cost economics](../../../11-gpu-cost-economics/practice/gpu-cost-synthesis/) | attribution, stranded-GPU-hours, multi-provider normalisation, commitment laddering |
| [12 Capstone](../../../12-capstone-interview/practice/gpu-platform-capstone/) | the flagship operator, end to end, plus a replayable trace benchmark |

## Acceptance criteria

- [ ] **A fleet exists** — ≥50 nodes, heterogeneous on accelerator type, node size, region, provider, and NVLink island.
- [ ] **Pods bind** — a job requesting 8 GPUs schedules, and `Allocated resources` moves.
- [ ] **Three shapes are distinguishable from metrics alone** — training, inference, idle-but-allocated.
- [ ] **Failure injection works** — exporter death, a fail-slow straggler, and an XID 48 are all reproducible on demand.
- [ ] **Two billing models normalise** to one defensible `$/GPU-hour`.
- [ ] **Verified once against real hardware** — a written diff of synthetic vs real `dcgm-exporter` output.

## Publishing it honestly

This matters more than the lab itself. **Label the simulation, every time:**

> *"I built a GPU scheduler plugin and benchmarked it against the default on a 200-node
> **simulated** fleet"* — accurate, impressive, and cannot be punctured.
>
> *"I built GPU scheduling infrastructure"* — cannot survive one follow-up question.

The first sentence is a stronger interview artifact than the second, because it comes with a
published trace and a number. A scheduler without a benchmark is an opinion.

## Suggested layout

```
fake-gpu-fleet/
├── fleet/
│   ├── kwok-nodes.yaml          # the heterogeneous fleet generator
│   └── rates.yaml               # provider → billing model
├── cmd/synthetic-dcgm/main.go   # exporter: 3 workload shapes + failure injection
├── traces/
│   └── jobs.jsonl               # replayable job trace for scheduler benchmarking
├── verify/
│   └── real-vs-synthetic.md     # the §5 diff against a rented GPU
└── README.md                    # this file
```
