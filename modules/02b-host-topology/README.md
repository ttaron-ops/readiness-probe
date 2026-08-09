---
id: "02b"
title: "Host hardware and topology"
notion: "https://app.notion.com/p/3b33abaeb82381b48bc9e928ef7cb695"
phase: "Phase 2 · Months 5–8 (precedes 03)"
effort: "~35 hrs ≈ 3.5 weeks @ 10–12 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: []
unlocks: ["03", "04", "09", "10"]
started: null
completed: null
---

# 🧬 02b — Host hardware and topology

> **Goal.** Understand the machine underneath the accelerator well enough to place
> and debug GPU workloads. The failure this module prevents: a GPU that looks busy
> but delivers far less than spec because of a host-side placement decision
> (NUMA/PCIe misalignment) that **no metric reports** — exactly the silent waste your
> cost work should catch.

- **Notion page:** https://app.notion.com/p/3b33abaeb82381b48bc9e928ef7cb695
- **Phase:** Phase 2 · **precedes 03**; assumed by 04, 09, 10 · **Est. effort:** ~35 hrs ≈ 3.5 weeks
- **Deliverable:** [Topology Teardown](practice/topology-teardown/) — reconcile a real
  GPU node's topology + measure a NUMA-misalignment cost.

## Why this module, and to what bar

Host-topology fluency is the literal job at neoclouds — and your differentiator stated as a hardware role:

- **CoreWeave** — *Sr HW Engineer, GPU & PCIe*: "troubleshoot complex **GPU and PCIe** failures… create **alerts**" (turn a silent placement fault into a monitored signal); *Sr GPU Infra SWE*: owns the "GPU performance testing platform."
- **NVIDIA** — *Sr SSE, Data Center GPU Compute Diagnostics*.
- **Interview probes:** *"GPU at 100% util but throughput is half spec — why?"* (host-side) · *"read this `nvidia-smi topo -m` — where does the NIC go?"* · *"guarantee a pod's CPU+mem+GPU on one NUMA node in k8s."*

## Calibrated to your background — what we skip

You have on-prem hardware background and just did the Linux module, so we **skip**:
what-is-a-CPU/RAM/cache, kernel NUMA mechanics (numa_balancing, first-touch — done in
01b), cgroups/hugepages/THP, datacenter/rack 101, PCIe history. We **start at
GPU-server topology + Kubernetes alignment** — "formalize + apply to accelerators."

## Lessons

Anchored on **Kubernetes topology alignment** (L5); ends in a real-node capstone (L8).

| # | Lesson | Hrs | Core skill |
|---|--------|-----|-----------|
| 01 | [Host as a topology tree (cpu-arch + NUMA, refresher)](lessons/01-host-topology-tree.md) | 2.5 | the tree devices hang off |
| 02 | [Memory subsystem (accelerator-relevant)](lessons/02-memory-subsystem.md) | 3 | local vs remote bandwidth = placement |
| 03 | [PCIe (gens/lanes/switches)](lessons/03-pcie.md) | 4 | read `LnkCap` vs `LnkSta` |
| 04 | [8-GPU node architecture (HGX/DGX)](lessons/04-server-architecture-8gpu.md) | 4 | draw the reference layout |
| 05 | [Topology / CPU / Memory Managers](lessons/05-topology-alignment-k8s.md) (anchor) | 6 | guarantee vs attempt; `TopologyInfo` trap |
| 06 | [Storage / NVMe placement + GDS](lessons/06-storage-nvme.md) | 3 | NVMe on the GPU's switch |
| 07 | [Power & thermals](lessons/07-power-and-thermals.md) (short) | 2.5 | throttling = undelivered FLOPs |
| 08 | [Capstone — read one real GPU node end to end](lessons/08-capstone-topology-teardown.md) | 7 | reconcile all four tools |

Total ≈ **35 hrs ≈ 3.5 weeks**. Spine = L5 + L8.

## Resource spine

- **Ronak Nathani — "Keeping GPU Workloads NUMA-Local in Kubernetes"** + **Frank
  Denneman — "Multi-GPU Topologies Within a Single Host"** (both required deep reads).
- **Kubernetes — Node Resource Managers & Memory Manager docs** (policy semantics).
- **NADDOD 8-card topology walkthrough** + **NVIDIA HGX/DGX whitepapers**.
- `lstopo` / `lspci` / `nvidia-smi topo -m` as tooling reference; Drepper §5 as a
  5-minute NUMA refresher only.

## Deliverable & checkpoint

- Build **[Topology Teardown](practice/topology-teardown/)** on a by-the-hour rented
  GPU box.
- The [**checkpoint**](checkpoint.md) is the gate — reconcile the four tools into one
  picture, decode `nvidia-smi topo -m` cold, configure Topology Manager, and quantify
  a misalignment's cost.

## How to work this module

1. Lessons in order; do every **Practice** — they compound into the teardown deliverable.
2. Rent a multi-GPU box for a focused weekend to do the hands-on labs (L2, L5, L7, L8).
3. Answer the [checkpoint](checkpoint.md) from memory; flip `status` and update Notion
   when the teardown + misalignment investigation is done.
