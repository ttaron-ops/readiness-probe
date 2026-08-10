---
id: "04"
title: "GPU on Kubernetes"
notion: "https://app.notion.com/p/3b33abaeb82381ff9762c82e3302fdbe"
phase: "Phase 2 · Months 5–8"
effort: "~109 hrs ≈ 9 weeks @ 10–12 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["01", "02", "03"]
unlocks: ["05", "06"]
started: null
completed: null
---

# 📦 04 — GPU on Kubernetes

> **Goal.** Own the GPU integration layer end to end — **the core operational module,
> the one a hiring manager probes hardest.** This is where your `gpu-cost-operator`
> starts doing real per-pod GPU attribution against live GPU nodes.

- **Notion page:** https://app.notion.com/p/3b33abaeb82381ff9762c82e3302fdbe
- **Phase:** Phase 2 · requires 01–03 · **Est. effort:** ~109 hrs ≈ 9 weeks (~$40–80 GPU rental)
- **Deliverable:** [Per-pod GPU attribution](practice/per-pod-attribution/) + a GPU
  Operator failure-mode log.

## Why this module, and to what bar

The JDs signal the surface; the depth is probed live:

- **CoreWeave** — *Sr Eng, Kubernetes Platforms*: "controllers/operators that automate infrastructure testing… visibility into system metrics/performance/health," "tens of thousands of kubelets."
- **NVIDIA** — *Sr DevOps, Platform Eng*: "troubleshoot… GPU-accelerated servers to minimize downtime."
- **Interview probes:** *"why can't you request 0.5 GPU?"* · *"walk the device-plugin gRPC lifecycle"* · *"name the GPU Operator components + debug a crash-looping driver pod"* · *"**MIG vs time-slicing vs MPS** — mechanism/isolation/when"* · *"design a driver upgrade for 200 nodes"* · *"**attribute cost to a time-sliced GPU**"* (your home turf).

## Calibrated to your background — what we skip

02/02b/03 already own the theory, so we **reference, not re-teach**: the device-plugin
gRPC *API mechanics* and DRA *object model* (02), Topology Manager / the `TopologyInfo`
trap (02b), and GPU/MIG *at the silicon level* (03). This module is the **operational
integration layer** — the GPU Operator as a running system, driver lifecycle,
partitioning, and per-pod attribution.

## Lessons

Anchored on **the GPU Operator + crash-loop debugging** (L1–L2); ends in the capstone wiring (L10).

| # | Lesson | Hrs | Ties to |
|---|--------|-----|---------|
| 01 | [GPU Operator: the component system](lessons/01-gpu-operator-components.md) (anchor) | 10 | install from scratch, the dependency chain |
| 02 | [Break it / fix it: crash-loop diagnosis](lessons/02-crash-loop-diagnosis.md) | 12 | seeds the failure-mode log |
| 03 | [Device-plugin recap + **pod-resources API**](lessons/03-device-plugin-recap-pod-resources.md) | 12 | the attribution hinge (Go client) |
| 04 | [Container-runtime integration (CDI)](lessons/04-container-runtime-integration.md) | 9 | debug "CUDA driver insufficient" |
| 05 | [Driver lifecycle + fleet upgrades](lessons/05-driver-lifecycle-upgrades.md) | 11 | the upgrade state machine, 200-node design |
| 06 | [MIG operations](lessons/06-mig-operations.md) | 10 | clean per-slice attribution |
| 07 | [Time-slicing + **the attribution hole**](lessons/07-time-slicing-attribution.md) | 10 | shared UUID → can't bill from allocation |
| 08 | [MPS + choosing a sharing mode](lessons/08-mps-choosing-sharing.md) | 7 | the cold comparison |
| 09 | [DRA driver (real install) + multi-tenancy quotas](lessons/09-dra-driver-and-quotas.md) | 12 | ResourceClaim scheduling; namespace quotas |
| 10 | [**Capstone — per-pod attribution into gpu-cost-operator**](lessons/10-capstone-per-pod-attribution.md) | 16 | the deliverable |

Total ≈ **109 hrs ≈ 9 weeks**. Spine = L1–L2 + L10.

## Resource spine

- **NVIDIA GPU Operator docs** (deep on gpu-sharing + driver-upgrades) · **MIG User Guide**.
- **k8s-device-plugin** + the **kubelet pod-resources API proto** · **dcgm-exporter source**
  (the exact UUID→pod pattern the capstone mirrors).
- **NVIDIA `k8s-dra-driver-gpu`** (now a CNCF sig repo) + the AKS DRA explainer.
- **scaleops'** MIG-vs-MPS-vs-time-slicing comparison for the cold interview answers.

> ⚠️ **Version-sensitive — re-verify at study time:** DRA GA in k8s 1.34, gate locks in
> 1.35, **1.34.0/.1 are buggy — use 1.33.x or ≥1.34.2**. The DRA driver is **not** folded
> into the GPU Operator — there was never a public GPU Operator "25.8" release; that number
> belonged to the DRA driver's own old CalVer tag scheme. The driver still installs as a
> **separate companion Helm chart**; its repo/chart moved to
> `kubernetes-sigs/dra-driver-nvidia-gpu` / `dra-driver-nvidia-gpu`, adopting semver from
> v0.4.0 (latest confirmed release **v0.4.1**). Full in-tree folding is a roadmap item, not
> shipped — check the exact Operator version's release notes rather than assuming.

## Deliverable & checkpoint

- Build **[Per-pod GPU attribution](practice/per-pod-attribution/)** — the pod-resources
  → UUID→pod→namespace exporter, handling MIG (clean) and time-sliced (per-PID fallback),
  plus the **failure-mode log**.
- The [**checkpoint**](checkpoint.md) is the gate — install/debug/upgrade the Operator
  unaided, MIG-vs-time-slicing-vs-MPS cold, map a UUID to a pod live, schedule via DRA.

## How to work this module

1. Lessons in order; every hands-on break/fix becomes a failure-mode-log entry.
2. Batch GPU rental: a cheap sub-$1/hr GPU for most; one A100/H100 session for MIG (L6).
3. Answer the [checkpoint](checkpoint.md) from memory; flip `status` and update Notion
   when the attribution exporter runs and the failure-mode log is written.
