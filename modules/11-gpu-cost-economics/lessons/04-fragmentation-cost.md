---
lesson: 04
title: "Fragmentation: the cost of GPUs you can't schedule"
module: 11
concept: "Unschedulable free capacity"
status: not-started
est_time: "4.5 hrs"
prev: "03-idle-detection.md"
next: "05-unit-economics.md"
artifacts: ["Fragmentation-ratio + stranded-$ calculation over a request-shape histogram, plus a packing-policy recovery estimate, added to the gpu-cost synthesis deliverable"]
sources: 11
---

# Fragmentation: the cost of GPUs you can't schedule

> Module: [💰 11 — GPU cost and unit economics](../README.md) · Deliverable: [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md)

## Where this fits

Lesson 03 covered idle: capacity that is **allocated** — bound to a pod — but not being used. It gave you a taxonomy and a decision rule for when it's safe to reclaim that unused allocation. This lesson covers a structurally different problem that looks similar on a dashboard but has nothing to do with utilisation: capacity that is **free** (never allocated to anyone) and yet **cannot be scheduled**, because its shape doesn't match anything in the pending queue.

Where lesson 03 was about *is this GPU being used*, this lesson is about *can this GPU be used at all, given how it's laid out*. That distinction matters because the fix is completely different — idle is a reclaim/pack-more problem; fragmentation is a scheduler-policy/topology/shape problem. Getting them confused, as many cost dashboards do, produces a "we have headroom" number that is actively wrong.

## Why this matters

Idle (lesson 03) is capacity you *allocated* and didn't use. Fragmentation is worse and sneakier: capacity that is **free and paid-for but cannot be scheduled**, because its shape doesn't match anything in the pending queue. On a dashboard the fleet reads "12 GPUs free" and green; in reality those 12 are scattered two-here-three-there across nodes, and the job at the head of the queue needs 8 rail-aligned GPUs on one node, so it pends while you pay for all 12 idle cards. This is the number that makes a CFO ask why a $40M cluster is "80% utilised" yet has a week-long training backlog.

Fragmentation is the dominant hidden cost on large accelerator fleets precisely because GPU jobs have *rigid, chunky shapes* — 8-GPU gang for a tensor-parallel model, a specific MIG profile, a NVLink-local domain — where CPU jobs are fungible and fluid. A scheduler that packs CPU perfectly will strand GPUs badly. Interviewers at any org running multi-hundred-GPU clusters (the neoclouds, the frontier labs, internal ML platforms) probe this because it separates people who ran `kubectl top` from people who watched a topology-blind scheduler leave a rack unschedulable.

The payoff is that fragmentation is *recoverable by policy* — bin-packing vs spread, gang scheduling, topology-aware placement, dynamic MIG, and DRA all move the number — so being able to quantify it (a fragmentation ratio, a stranded-dollar figure) and tie each shape of it to the policy that fixes it is directly monetisable work.

## What's new here (calibration)

- **Skip:** the topology fundamentals — NVLink domains, rail alignment, why bandwidth-sensitive collectives need locality (mod 02b/09) — and gang scheduling with Kueue (mod 06). We *use* these as the levers, not re-derive them.
- **New:** the precise **idle-vs-fragmentation distinction** (allocated-unused vs unallocated-unusable) and the four **shapes** of fragmentation (whole-GPU/gang, MIG-profile, in-GPU memory, topology).
- **New:** a quantification method — the **fragmentation ratio** = unschedulable-free / total-free — and stranded-$ = Σ(unschedulable-free GPU-hrs × rate).
- **New:** the **dynamic-vs-static MIG** tradeoff (reconfigure-to-fit vs strand) with its drain/churn cost, and the **DRA** forward-tie (structured parameters let the scheduler reason about partitionable devices).

## Core concepts

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

### This is not a toy problem — real numbers

Two production/academic data points anchor why fragmentation gets staff-level attention rather than being dismissed as an edge case:

- **Meta's MAST scheduler (OSDI'24).** Before MAST's global, fleet-aware scheduling, Meta's most-overloaded training region ran at a **GPU demand-to-supply ratio of 2.63** for high-priority workloads — i.e. demand for GPUs in that region was 2.63× the locally available supply, even while other regions plausibly had unusable slack. After MAST, that ratio dropped to **0.98** — essentially matched. That's a scheduling/placement fix, not a hardware purchase, closing a gap of that scale.
- **The "Beware of Fragmentation" study (USENIX ATC'23), on Alibaba's production PAI cluster.** Analysis of a real multi-tenant GPU-sharing cluster found that partial-GPU allocation (the norm once you allow MIG/time-slicing/fractional requests) can convert **21–42% of unallocated GPU resources into fragmentation** — i.e. that fraction of "free" capacity was structurally unusable by the pending workload mix. Their scheduler, Fragmentation Gradient Descent (FGD), recovered a large share of it in production trace replay.

Both numbers are dated to their publication years (2023–2024) and specific to the fleets studied — treat them as evidence of *magnitude*, not a universal constant, but they establish that fragmentation losses in the tens-of-percent range are observed on real, large, heterogeneous GPU fleets, not a hypothetical.

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

## Perspectives

**Scheduler-policy perspective.** Bin-pack vs spread is not a minor tuning knob for GPU fleets — it's a first-class lever with the single largest effect on fragmentation of anything covered in this lesson. Kubernetes' spread-favouring default (`LeastAllocated`) exists for good reasons on CPU fleets (blast-radius, resilience), but for chunky GPU gang-shapes it is the primary mechanism that turns "12 free GPUs" into "0 usable GPUs," as the worked example below shows starkly.

**Hardware/topology perspective.** "Enough GPUs" and "the right GPUs" are different claims once NVLink domains and rail alignment enter the picture. A scheduler that only counts device integers can place a job onto GPUs that satisfy the count but not the topology — the job runs, but a bandwidth-sensitive collective crawls across PCIe instead of NVLink/NVSwitch. That's fragmentation with no pending-pod symptom at all; it hides inside a degraded MFU number instead, which is why topology-blind placement is a genuinely dangerous default, not just a suboptimal one.

**Finance/capital perspective.** Stranded fragmented capacity is paid-for capital sitting idle by *shape* — a distinct financial category from lesson 02/03's utilisation waste. Utilisation waste says "we bought the right amount but ran it inefficiently." Fragmentation says "we bought the right amount, and it's structurally unusable given how it's arranged" — which is a scheduling-policy failure, not a workload-efficiency failure, and gets reported and fixed differently to finance stakeholders. Conflating the two in a cost review misattributes blame (to workload owners instead of platform/scheduler config) and misdirects the fix.

**Capacity-planning perspective.** The request-shape histogram isn't only an observability output for measuring today's fragmentation — it's a forecasting input for tomorrow's procurement and geometry decisions. If the pending queue is persistently dominated by 8-GPU gangs, that's a signal to buy/allocate whole-node blocks and bias MIG geometry toward larger profiles *before* the fragmentation shows up, not just to react to it after the fact. Fragmentation is partly solvable by designing the demand mix (standardizing on fewer request shapes) as well as by fixing the scheduler.

## Real-world use cases

- **Meta — "MAST: Global Scheduling of ML Training across Geo-Distributed Datacenters at Hyperscale" (OSDI'24)** — <https://www.usenix.org/conference/osdi24/presentation/choudhury>. Meta's real production global scheduler for ML training. Before MAST, the most overloaded region ran a GPU demand-to-supply ratio of 2.63 for high-priority workloads; MAST brought that down to 0.98. What it shows: fleet-aware, cross-region scheduling can eliminate the equivalent of a 2.6× local capacity shortfall without new hardware — the highest-magnitude real-world validation of "shape/placement policy is worth as much as buying more GPUs" in this lesson.
- **"Beware of Fragmentation: Scheduling GPU-Sharing Workloads with Fragmentation Gradient Descent" (USENIX ATC'23)** — slides: <https://www.usenix.org/system/files/atc23_slides_weng.pdf>, paper: <https://www.cse.ust.hk/~weiwa/papers/fgd-atc23.pdf>. Academic study built directly on a real multi-tenant GPU-sharing production cluster (Alibaba PAI); finds that partial-GPU allocation can convert 21–42% of unallocated GPU resources into fragmentation, and proposes a scheduler (FGD) that materially reduces it. What it shows: on-topic, quantified confirmation that fragmentation — not idle — is the dominant hidden loss once you allow fractional/shared GPU allocation at scale.
- **Alibaba PAI GPU cluster trace 2020** — <https://github.com/alibaba/clusterdata/tree/master/cluster-trace-gpu-v2020>. A public production trace from a real ~6,500-GPU Alibaba cluster running a mixed training-and-inference workload over July–August 2020 (the same cluster family studied by the FGD paper above). What it shows: the exact workload heterogeneity — mixed training + inference, varied request shapes — that produces shape fragmentation in the first place, and gives you real request-shape data to replay a fragmentation-ratio calculation against instead of a synthetic histogram.

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

## Common pitfalls

1. **Conflating idle and fragmentation as one waste bucket.** They look identical on a naive dashboard ("$X of GPUs not doing anything") but idle is allocated-and-unused (a utilisation problem, fixed by reclaim/pack-more) while fragmentation is unallocated-and-unusable (a scheduling/shape problem, fixed by changing the scheduler or the shapes). Reporting them together produces a number with no single owner and no single fix.
2. **Assuming Kubernetes' default spread/`LeastAllocated` scoring is a safe, neutral default for GPU workloads.** It's a sensible default for fungible CPU workloads but is the primary driver of gang-fragmentation on chunky GPU jobs — this lesson's own worked example shows a 0.0 → 1.0 fragmentation-ratio swing from a scoring flip alone, with no change in the amount of free capacity.
3. **Believing more free GPUs always means more usable capacity.** The worked example's headline result — the same 12 free GPUs go from fragmentation ratio 1.0 (fully stranded) to 0.0 (fully usable) purely by rearranging *layout*, not count — is the single most important, most quotable fact in this lesson. "GPUs free" and "GPUs usable" are different numbers.
4. **Treating dynamic MIG reconfiguration as a free fix for fragmentation.** It ignores the whole-GPU drain cost (evict all slices, reset, rewrite) every reconfigure incurs, and the thrash risk if demand oscillates — repeatedly draining a GPU to chase a shifting request mix can cost more than the stranding it's meant to fix.
5. **Ignoring topology fragmentation because "the GPU count fits."** A job can schedule onto the right *number* of GPUs that are non-rail-aligned or span NVLink domains and silently run at degraded MFU — no pending pod, no obvious symptom, just quietly worse throughput per dollar, undetectable without explicit topology-aware monitoring.

## Self-check

- How does fragmentation differ from idle, and why does the same free-GPU count give different fragmentation? **Answer:** Idle is capacity *allocated to a pod but unused* — a utilisation problem fixed by reclaim/pack-more. Fragmentation is capacity that is *free/unallocated but unschedulable* because its shape doesn't match any pending request's constraints (gang size, MIG profile, contiguous HBM, topology domain) — a scheduling/shape problem fixed by changing the scheduler or the shapes. The same free count fragments differently because schedulability depends on the *layout*: 12 free GPUs smeared 2-and-3 across nodes satisfy no 8-GPU gang (fragmentation_ratio 1.0), whereas the same 12 coalesced into one full node + one half node satisfy the gang queue (ratio 0.0). Only the shape changed.
- Name the four shapes of fragmentation and the lever that reduces each. **Answer:** (a) whole-GPU/gang — free GPUs scattered so an N-on-one-node gang can't place → **bin-pack scoring + gang scheduling (Kueue)**; (b) MIG-profile — a fixed slice geometry can't host a different-profile request without a whole-GPU-draining reconfigure → **dynamic MIG / DRA**; (c) in-GPU memory — HBM free but not contiguous/large enough for the next model → **allocator/packing changes, DRA partitioning**; (d) topology — free GPUs exist but aren't rail-aligned/NVLink-local so a bandwidth-sensitive job won't place or runs degraded → **topology-aware scheduling**.
- State the dynamic-vs-static MIG tradeoff. **Answer:** Static MIG fixes each GPU's profile geometry: simple, no runtime churn, but **strands** capacity whenever the request mix drifts from the chosen geometry (steady-state fragmentation). Dynamic MIG reconfigures geometry to match live demand, cutting fragmentation, **but every reconfigure drains the entire physical GPU** (evict all slices, reset, rewrite) — a transition cost that becomes thrash if demand oscillates. So dynamic MIG trades steady-state stranding for transition cost + churn risk: it wins for diverse but slow-moving demand (hysteresis/rate-limit the reconfigures) and loses for fast-oscillating demand where repeated drains cost more than the stranding they save.
- What do the MAST (Meta) and FGD (Alibaba PAI) real-world numbers add to the "fragmentation is a toy problem" question, and why must you still flag them as dated snapshots? **Answer:** MAST shows a real hyperscaler cut its most-overloaded region's GPU demand-to-supply ratio from 2.63 to 0.98 through scheduling alone, and FGD shows a real multi-tenant cluster converting 21–42% of unallocated GPU resources into fragmentation under partial-GPU allocation — both are magnitude evidence, from named production/academic sources, that fragmentation losses run into double-digit percentages and multi-× demand/supply gaps on real fleets, not a hypothetical corner case. They must be flagged as dated (2023–2024 publication, specific fleets) because both the request-shape mix and the fixes available (e.g. DRA didn't exist in its current form yet) change over time — the *magnitude* is evidence the problem is real, not a constant you can port unchanged into any fleet's math.

## Connections & what's next

This lesson deliberately contrasts with lesson 03: idle and fragmentation both show up as "wasted GPU-hours" on a naive report, but they are different problems (utilisation vs shape) with different owners and different fixes, and this lesson exists to keep that distinction sharp. It builds on the topology and gang-scheduling fundamentals from mod 02b/06/09, applying them specifically to quantify and price stranded capacity.

Next, lesson 05 (unit economics) shows how fragmentation inflates your *effective* cost per unit — a $/token or $/experiment number computed only from allocated-vs-utilised GPU-hours understates true cost once stranded capacity is factored in, because you're still paying for the whole fleet including the unschedulable slice. Lesson 06 (commitment & procurement strategy) picks up the capacity-planning thread from this lesson's Perspectives section: fragmentation directly affects how much capacity you actually need to buy or commit to, since a fleet with high structural fragmentation needs *more* headroom than its raw utilisation numbers would suggest.

## References & further reading

**Primary sources**
1. **NVIDIA MIG user guide** — <https://docs.nvidia.com/datacenter/tesla/mig-user-guide/> — profile geometry rules, why reconfigure drains the GPU, valid profile tilings behind shape-(b) fragmentation.
2. **Kubernetes Dynamic Resource Allocation (DRA) / structured parameters** — <https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/> — partitionable devices and scheduler-side partitioning that reduce MIG/in-GPU fragmentation.
3. **Kueue (gang / all-or-nothing scheduling)** — <https://kueue.sigs.k8s.io/docs/concepts/> — the gang-scheduling lever against whole-GPU strand (ties to mod 06).
4. **kube-scheduler scoring — `NodeResourcesFit` MostAllocated vs LeastAllocated** — <https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins> — bin-pack vs spread, the highest-leverage default flip.
5. **NVIDIA GPU Operator MIG management (dynamic reconfiguration)** — <https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-mig.html> — the mechanics and drain cost of dynamic MIG.
6. **Kubernetes topology-aware scheduling / NVIDIA GPU topology** — <https://github.com/NVIDIA/k8s-device-plugin#gpu-feature-discovery> — topology labels feeding rail-aligned placement (shape (d), mod 09).

**Real-world engineering blogs**
7. **Meta — "MAST: Global Scheduling of ML Training across Geo-Distributed Datacenters at Hyperscale" (OSDI'24)** — <https://www.usenix.org/conference/osdi24/presentation/choudhury> — real production global scheduler; read for the 2.63 → 0.98 demand/supply ratio result.
8. **"Beware of Fragmentation: Scheduling GPU-Sharing Workloads with Fragmentation Gradient Descent" (USENIX ATC'23)** — slides: <https://www.usenix.org/system/files/atc23_slides_weng.pdf>, paper: <https://www.cse.ust.hk/~weiwa/papers/fgd-atc23.pdf> — on-topic academic study on a real multi-tenant GPU-sharing cluster; read for the 21–42% stranded-fragmentation figure and the FGD scheduling approach.
9. **Alibaba PAI GPU cluster trace 2020** — <https://github.com/alibaba/clusterdata/tree/master/cluster-trace-gpu-v2020> — real mixed training+inference production trace data; read for a real request-shape distribution to replay a fragmentation-ratio calculation against.

**Deeper dives**
10. **USENIX ATC '23 talk video — "Beware of Fragmentation: Scheduling GPU-Sharing Workloads"** — <https://www.youtube.com/watch?v=-37_clawjdc> — the FGD paper's conference talk, for a faster way into the same material than the full paper.
