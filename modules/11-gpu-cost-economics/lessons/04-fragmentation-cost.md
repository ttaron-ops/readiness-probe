---
lesson: 04
title: "Fragmentation: the cost of GPUs you can't schedule"
module: 11
concept: "Unschedulable free capacity"
status: not-started
est_time: "3 hrs"
artifacts: ["Fragmentation-ratio + stranded-$ calculation over a request-shape histogram, plus a packing-policy recovery estimate, added to the gpu-cost synthesis deliverable"]
---

# Fragmentation: the cost of GPUs you can't schedule

> Module: [💰 11 — GPU cost and unit economics](../README.md) · Deliverable: [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md)

## Why this matters

Idle (lesson 03) is capacity you *allocated* and didn't use. Fragmentation is worse and sneakier: capacity that is **free and paid-for but cannot be scheduled**, because its shape doesn't match anything in the pending queue. On a dashboard the fleet reads "12 GPUs free" and green; in reality those 12 are scattered two-here-three-there across nodes, and the job at the head of the queue needs 8 rail-aligned GPUs on one node, so it pends while you pay for all 12 idle cards. This is the number that makes a CFO ask why a $40M cluster is "80% utilised" yet has a week-long training backlog.

Fragmentation is the dominant hidden cost on large accelerator fleets precisely because GPU jobs have *rigid, chunky shapes* — 8-GPU gang for a tensor-parallel model, a specific MIG profile, a NVLink-local domain — where CPU jobs are fungible and fluid. A scheduler that packs CPU perfectly will strand GPUs badly. Interviewers at any org running multi-hundred-GPU clusters (the neoclouds, the frontier labs, internal ML platforms) probe this because it separates people who ran `kubectl top` from people who watched a topology-blind scheduler leave a rack unschedulable.

The payoff is that fragmentation is *recoverable by policy* — bin-packing vs spread, gang scheduling, topology-aware placement, dynamic MIG, and DRA all move the number — so being able to quantify it (a fragmentation ratio, a stranded-dollar figure) and tie each shape of it to the policy that fixes it is directly monetisable work.

## What's new here

- **Skip:** the topology fundamentals — NVLink domains, rail alignment, why bandwidth-sensitive collectives need locality (mod 02b/09) — and gang scheduling with Kueue (mod 06). We *use* these as the levers, not re-derive them.
- **New:** the precise **idle-vs-fragmentation distinction** (allocated-unused vs unallocated-unusable) and the four **shapes** of fragmentation (whole-GPU/gang, MIG-profile, in-GPU memory, topology).
- **New:** a quantification method — the **fragmentation ratio** = unschedulable-free / total-free — and stranded-$ = Σ(unschedulable-free GPU-hrs × rate).
- **New:** the **dynamic-vs-static MIG** tradeoff (reconfigure-to-fit vs strand) with its drain/churn cost, and the **DRA** forward-tie (structured parameters let the scheduler reason about partitionable devices).

## Core notes

### Idle vs fragmentation — keep them separate

| | Bound to a pod? | Being used? | Cost problem |
|---|---|---|---|
| **Idle** (lesson 03) | Yes | No | Allocated-but-unused — a *utilisation* problem |
| **Fragmentation** | No | (n/a — it's free) | Unallocated-but-**unusable** — a *scheduling/shape* problem |

Fragmented capacity is genuinely free at the device level; the failure is that no pending request can *consume* it given placement constraints. Same dollar bleed, completely different fix: idle is reclaim/pack-more; fragmentation is change-the-scheduler/change-the-shapes.

### The four shapes of fragmentation

**(a) Whole-GPU / gang-packing fragmentation.** The pending job needs *N GPUs on one node* (a tensor-parallel or NVLink-domain gang, mod 06). Free GPUs exist but are scattered: node A has 2 free, node B has 3 free, and an 8-on-one-node request can't be satisfied even though 5 are free fleetwise. The 5 are stranded. This is the classic gang-scheduling strand and it ties directly to topology/NVLink locality (mod 02b/09) — the gang usually must also be *co-located and rail-aligned*, tightening the constraint further.

**(b) MIG-profile fragmentation.** An A100/H100 sliced into 7× `1g.10gb` cannot host a `3g.40gb` request without a **reconfigure**, and reconfiguring MIG **drains the entire physical GPU** (all existing slices must be evicted, the GPU reset, geometry rewritten). Mixed-profile geometry constraints strand slices: MIG geometry is not arbitrary — valid profiles must tile the GPU under placement rules, so a GPU carrying one `3g` + some `1g` may have free capacity that no allowed profile can occupy. You end up with slices that are free but unrequestable.

**(c) In-GPU memory fragmentation.** On a shared GPU (MPS, or a serving stack packing multiple models), HBM is free in aggregate but not **contiguous / not a single block large enough** for the next model's weights + activations/KV. 12 GB free across gaps doesn't load a 10 GB model that needs a contiguous allocation. This is the allocator-level analogue of the node-level strand.

**(d) Topology fragmentation.** Free GPUs exist and are even on the right node count, but they are **not rail-aligned / not in the same NVLink domain** (mod 09). A bandwidth-sensitive collective (all-reduce over NVLink/NVSwitch) won't place, or places and runs at a fraction of throughput because it's crossing PCIe/inter-rail. The scheduler that ignores topology "successfully" schedules the job into a slow corner — a *silent* fragmentation cost that shows up as degraded MFU (mod 08) rather than a pending pod.

### Quantifying fragmentation

**Stranded cost:**
```
fragmentation_$  =  Σ (free-but-unschedulable GPU-hours) × $rate
```
Free-but-unschedulable = free GPUs that no request in the pending queue can consume given its shape + placement constraints. You compute it by replaying the pending request-shape histogram against the free-capacity layout.

**Fragmentation ratio** — a scale-free health metric:
```
fragmentation_ratio  =  unschedulable_free_GPUs / total_free_GPUs
```
`0.0` = every free GPU is placeable (perfect packing). `1.0` = all free capacity is stranded. A fleet at 15% free with a fragmentation ratio of 0.6 is effectively at 6% *usable* free — the ratio is what turns "we have headroom" into "we don't."

**Scheduler behaviour that drives it:**

| Policy | Effect on fragmentation |
|---|---|
| **Bin-pack / `mostAllocated` (MostAllocated scoring)** | Concentrates load, leaves *whole nodes empty* → large contiguous free blocks → **low** whole-GPU/gang fragmentation. Preferred for GPU. |
| **Spread / `leastAllocated`** (k8s default) | Scatters pods for resilience → free GPUs smeared thin across nodes → **high** gang fragmentation. Wrong default for chunky GPU jobs. |
| **Gang scheduling (Kueue, mod 06)** | All-or-nothing placement prevents partial placement that would half-fill nodes and strand the rest; enables reserving contiguous blocks. |
| **Topology-aware scheduling** (topology-aware plugin / NVIDIA topology labels) | Places gangs within an NVLink domain / rail → removes shape (d), and by preferring domain-complete placement reduces (a). |

The single highest-leverage default change on most GPU clusters is flipping the scoring from spread to **bin-pack**, so empty capacity coalesces into schedulable whole-node blocks instead of unusable dust.

### The MIG reconfiguration tradeoff

- **Static MIG geometry** — fix each GPU's profile layout ahead of time (e.g. all `1g.10gb`, or all `3g.40gb`). Simple, no runtime churn, predictable. **Strands** whenever the request mix drifts from the chosen geometry: shape-(b) fragmentation grows as demand and geometry diverge.
- **Dynamic MIG** — reconfigure geometry to match live demand (the NVIDIA GPU Operator's MIG manager can rewrite profiles). Reduces fragmentation by fitting supply to the request histogram, **but** each reconfigure **drains the whole physical GPU** (evict all slices, reset, rewrite) — a real cost in lost work + churn, and pathological if demand oscillates (you drain repeatedly). 

**The tradeoff, explicitly:** dynamic MIG trades *steady-state stranding* for *transition cost + churn risk*. It wins when demand is diverse but slow-moving (reconfigure occasionally, run stable); it loses when demand is fast-oscillating (constant drains cost more than the stranding they save). Rate-limit reconfigures and hysteresis-gate them, or you convert a fragmentation problem into a thrash problem.

### DRA — the forward-looking lever

Dynamic Resource Allocation (structured parameters; forward-tie to lesson 01's DRA regime) lets the device driver advertise a GPU as a **partitionable device** with structured attributes, and lets the *scheduler itself* reason about how to carve it to fit a claim — rather than the device-plugin pre-slicing into fixed integer resources the scheduler can only count. Because the scheduler sees the partition structure, it can pick a partitioning that *matches the pending claim* at schedule time, reducing shape-(b) MIG fragmentation and shape-(c) in-GPU strand without a separate drain-and-reconfigure step. It's the structural fix for "the slicing was decided before we knew the demand."

## Worked example

A **64-GPU** fleet (8 nodes × 8 GPUs, H100, $3.20/GPU-hr). Snapshot: **52 allocated, 12 free**. Pending-queue request-shape histogram:

| Request shape | Pending count | Placement constraint |
|---|---|---|
| 8-GPU gang (tensor-parallel) | 3 | one node, rail-aligned |
| 4-GPU gang | 2 | one node, NVLink-local |
| 1-GPU | 6 | any |

Free-capacity layout (spread scheduler has smeared it):

| Node | Free GPUs |
|---|---|
| n1 | 3 |
| n2 | 2 |
| n3 | 1 |
| n4 | 2 |
| n5 | 0 |
| n6 | 1 |
| n7 | 2 |
| n8 | 1 |
| **Total** | **12** |

**Schedulability:** No node has ≥4 free, so **zero** of the 3×8-gang and 2×4-gang requests place — all 5 gang jobs pend. Only the 6 1-GPU requests can consume free capacity, and there are 12 free, so 6 GPUs get used, 6 remain free-and-unschedulable.

- `total_free = 12`, after placing the six 1-GPU jobs, `free = 6`, all unschedulable (no gang fits).
- **fragmentation_ratio (post 1-GPU placement) = 6/6 = 1.0** — every remaining free GPU is stranded.
- Measuring at the snapshot instant against the *gang* demand: 12/12 free are unusable by the gang queue → ratio **1.0** for the workload that actually matters (the gangs are the backlog).
- **Stranded $:** those 6 free GPUs sit idle across, say, an 8-hr window while gangs pend → 6 × 8 × $3.20 = **$153.60** stranded in one window; annualised at that steady state ≈ 6 × 8760 × $3.20 ≈ **$168k/yr** of capacity paid-for-but-unschedulable — *plus* the opportunity cost of 5 gang jobs blocked in the backlog.

**Policy change — flip spread → bin-pack + gang scheduling.** Re-pack the 52 allocated GPUs to concentrate them, coalescing free capacity into whole nodes. Suppose bin-packing consolidates the allocated load so free capacity becomes: n7 and n8 fully free (16 GPUs? — here we only have 12 free, so) → n1..n2 drained to yield **one fully-free node (8) + one node with 4 free**:

| Node | Free GPUs (after bin-pack) |
|---|---|
| n8 | 8 |
| n7 | 4 |
| others | 0 |

Now the layout satisfies **one 8-gang** (n8) **and one 4-gang** (n7) immediately — 12 free GPUs, all placeable against the gang queue.

- **fragmentation_ratio → 0/12 = 0.0.**
- **Recovered:** the 12 previously-stranded GPU-hours/hr are now schedulable, and 2 of the 5 backlogged gang jobs start immediately. Over the same 8-hr window that is **12 × 8 = 96 GPU-hours** moved from stranded to productive, ≈ **$307** recovered per window (idle avoided + backlog drained), before counting the throughput value of unblocking the gang jobs.

The lesson: the *free GPU count didn't change* (12 both times) — only the **shape** of the free capacity did, and that shape is a scheduler-policy choice worth six figures a year on a single 64-GPU fleet.

## Practice

Feeds `../practice/gpu-cost-synthesis/README.md`.

1. **Build the request-shape histogram.** From the pending queue (or a synthesised one), tabulate request shapes and their placement constraints (gang size, topology).
2. **Compute the fragmentation ratio and stranded $.** Given a free-capacity layout, replay the histogram: count free-but-unschedulable GPUs, compute `fragmentation_ratio = unschedulable_free/total_free` and `stranded_$ = unschedulable_free_GPU_hrs × rate` for a stated window.
3. **Show a packing-policy recovery.** Re-lay-out the free capacity under bin-pack (+ gang scheduling), recompute the ratio and the GPU-hours recovered, and state the $ delta.
4. **Classify one instance of each shape** (whole-GPU/gang, MIG-profile, in-GPU memory, topology) in your fleet or a synthetic one, and name the specific lever that fixes each (bin-pack, gang, dynamic MIG, topology-aware, DRA).
5. **Deliverable:** add a "Fragmentation cost" section to the synthesis README with the histogram, the ratio + stranded-$ figure, and the before/after packing-policy recovery.

## Self-check

- How does fragmentation differ from idle, and why does the same free-GPU count give different fragmentation? **Answer:** Idle is capacity *allocated to a pod but unused* — a utilisation problem fixed by reclaim/pack-more. Fragmentation is capacity that is *free/unallocated but unschedulable* because its shape doesn't match any pending request's constraints (gang size, MIG profile, contiguous HBM, topology domain) — a scheduling/shape problem fixed by changing the scheduler or the shapes. The same free count fragments differently because schedulability depends on the *layout*: 12 free GPUs smeared 2-and-3 across nodes satisfy no 8-GPU gang (fragmentation_ratio 1.0), whereas the same 12 coalesced into one full node + one half node satisfy the gang queue (ratio 0.0). Only the shape changed.
- Name the four shapes of fragmentation and the lever that reduces each. **Answer:** (a) whole-GPU/gang — free GPUs scattered so an N-on-one-node gang can't place → **bin-pack scoring + gang scheduling (Kueue)**; (b) MIG-profile — a fixed slice geometry can't host a different-profile request without a whole-GPU-draining reconfigure → **dynamic MIG / DRA**; (c) in-GPU memory — HBM free but not contiguous/large enough for the next model → **allocator/packing changes, DRA partitioning**; (d) topology — free GPUs exist but aren't rail-aligned/NVLink-local so a bandwidth-sensitive job won't place or runs degraded → **topology-aware scheduling**.
- State the dynamic-vs-static MIG tradeoff. **Answer:** Static MIG fixes each GPU's profile geometry: simple, no runtime churn, but **strands** capacity whenever the request mix drifts from the chosen geometry (steady-state fragmentation). Dynamic MIG reconfigures geometry to match live demand, cutting fragmentation, **but every reconfigure drains the entire physical GPU** (evict all slices, reset, rewrite) — a transition cost that becomes thrash if demand oscillates. So dynamic MIG trades steady-state stranding for transition cost + churn risk: it wins for diverse but slow-moving demand (hysteresis/rate-limit the reconfigures) and loses for fast-oscillating demand where repeated drains cost more than the stranding they save.

## Resources

1. **NVIDIA MIG user guide** — <https://docs.nvidia.com/datacenter/tesla/mig-user-guide/> — profile geometry rules, why reconfigure drains the GPU, valid profile tilings behind shape-(b) fragmentation.
2. **Kubernetes Dynamic Resource Allocation (DRA) / structured parameters** — <https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/> — partitionable devices and scheduler-side partitioning that reduce MIG/in-GPU fragmentation.
3. **Kueue (gang / all-or-nothing scheduling)** — <https://kueue.sigs.k8s.io/docs/concepts/> — the gang-scheduling lever against whole-GPU strand (ties to mod 06).
4. **kube-scheduler scoring — `NodeResourcesFit` MostAllocated vs LeastAllocated** — <https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins> — bin-pack vs spread, the highest-leverage default flip.
5. **NVIDIA GPU Operator MIG management (dynamic reconfiguration)** — <https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-mig.html> — the mechanics and drain cost of dynamic MIG.
6. **Kubernetes topology-aware scheduling / NVIDIA GPU topology** — <https://github.com/NVIDIA/k8s-device-plugin#gpu-feature-discovery> — topology labels feeding rail-aligned placement (shape (d), mod 09).
