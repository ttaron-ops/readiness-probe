---
lesson: 02
title: "Allocated vs utilised cost: the two ledgers"
module: 11
concept: "two-ledger cost model"
status: not-started
est_time: "2 hrs"
artifacts: ["a two-ledger (allocated vs utilised) computation for a fleet slice with the gap % and dollar figure, added to the module deliverable"]
---

# Allocated vs utilised cost: the two ledgers

## Why this matters
The bill you pay and the value you got are two different numbers, and almost every GPU cost mistake is a confusion between them. **Allocated cost** is what a workload *reserved* — it hits the invoice whether the GPU computed anything or sat warm and idle. **Utilised cost** is the slice of that reservation that did useful work. The difference between the two is the **waste ledger**, and on real GPU fleets the waste ledger is where essentially all the money is — reserved-but-idle H100s at $2–3+/hour each, multiplied by fleet size, is a seven-figure line item hiding behind a healthy-looking utilisation dashboard.

A platform engineer who reports only one of these numbers is lying by omission. Report only *allocated* and you look fully committed while burning cash on idle silicon. Report only *utilised* and you understate what procurement must actually pay for, so capacity planning collapses. Neocloud FinOps and platform-eng cost ownership are both judged on holding both ledgers at once and naming the gap out loud. This lesson is the conceptual hinge for the whole module — short by design, because the machinery (attribution regimes in lesson 01, the SM_ACTIVE signal in module 05) is already built; this just wires them into two ledgers.

## What's new here
- **Already yours (skip):** SM_ACTIVE vs GPU_UTIL as the correct utilisation signal — GPU_UTIL only says "a kernel was resident," SM_ACTIVE says "the SMs did work" (module 05, its dashboard). The attribution mechanism and sharing regimes (lesson 01, module 04/05).
- **New angle 1:** the explicit **two-ledger model** — allocated vs utilised as *separate books you always report together*, with the gap as a first-class number.
- **New angle 2:** the **reservation-vs-consumption asymmetry** — reservation is billed in whole GPU-hours the instant you bind; consumption is a fraction earned back only when SMs actually work, so the two ledgers drift apart structurally, not accidentally.

## Core notes

### The two ledgers
- **Allocated cost** = `reserved_GPU_hours × rate`. `reserved_GPU_hours` comes from the allocation record — device binding / requests held over time (the exact, deterministic side from lesson 01). This is what the bill charges no matter what.
- **Utilised cost** = `allocated_cost × useful_work_fraction`, where `useful_work_fraction` is measured from the correct hardware signal — **SM_ACTIVE** (module 05), *not* GPU_UTIL, and definitely not "is the pod Running." For a training job you may further gate on MFU (module 08) as the "useful" bar; for serving, on request throughput. The point: utilised is allocated *discounted by real work*.
- **The gap** = `allocated − utilised`, the **waste ledger**. `gap% = 1 − useful_work_fraction`.

Both ledgers use the *same* rate and the *same* attribution (lesson 01). The only difference is the multiplier: allocated multiplies by 1 (you reserved it), utilised multiplies by the SM_ACTIVE-derived work fraction.

### Why platform engineers must report BOTH
| Ledger | Drives | If you hide it |
|---|---|---|
| **Allocated** | Capacity planning, procurement, commitment sizing (lesson 06) | You understate what must be bought/reserved → stranded or short capacity |
| **Utilised** | Efficiency programs, showback/chargeback (lesson 08), right-sizing | You hide the utilisation lie → idle spend looks like committed spend |

Allocated answers *"how much iron do we need and pay for?"*. Utilised answers *"how much of it is actually working?"*. One without the other is a half-audited book.

### The reservation-vs-consumption asymmetry
Reservation is **coarse, up-front, and whole-unit**: bind 8 GPUs for 10 hours and you owe 80 GPU-hours the moment the pods schedule — no partial credit for kernels that never launched. Consumption is **fine-grained and earned**: it accrues only in the moments SM_ACTIVE is high. Because reservations are integer GPU-hours held over wall-clock time while consumption is a sub-unit fraction of that time, the two ledgers **diverge by construction** — the gap is the default state, not an anomaly.

### Idle-but-allocated is the dominant cost
The waste ledger has two distinct shapes, both billed at full allocated rate:
- **Reserved-but-not-running** — the device is bound to a pod but no process is on it (job scheduled, model still downloading; notebook pod parked overnight; a claim held "just in case"). SM_ACTIVE ≈ 0 and even GPU_UTIL ≈ 0.
- **Running-but-idle** — a process *is* on the GPU but the SMs are starved (dataloader-bound, tiny batch, blocked on the host, over-provisioned inference replica). GPU_UTIL can look busy while **SM_ACTIVE is low** — this is exactly the module-05 utilisation lie, and it is the one that fools single-number dashboards.

On most fleets these two together dominate total spend. Distinguishing and quantifying them is the job of lesson 03 (idle detection methodology); here they are simply the components of the gap. Unit economics (lesson 05) then converts a utilised GPU-hour into cost-per-token / cost-per-run.

## Worked example
A fleet slice: one job requests **8× H100** for **10 hours** at **$3.00/GPU-hour**.

- **Allocated ledger:** `8 × 10 × $3.00 = $240.00` for `80` GPU-hours. This is the invoice line, unconditionally.
- **Utilisation signal:** DCGM reports average **SM_ACTIVE = 30%** over the run (GPU_UTIL read ~85% — the lie; the job is dataloader-bound).
- **Utilised ledger:** `80 × 0.30 = 24` GPU-hours of useful work → `24 × $3.00 = $72.00`.
- **Gap (waste ledger):** `$240 − $72 = $168`, i.e. `gap% = 70%`.

So **$168 of this single $240 job is waste** — bound H100-hours that produced no useful SM work. Report only allocated and it looks like a fully-committed 80-GPU-hour job; report only utilised and procurement never sees the 80 GPU-hours it must actually buy and commit against. The honest report is both lines plus the 70% gap and its **$168** dollar figure — and that $168, scaled across a fleet of such jobs, is the number that justifies a right-sizing or scheduling program.

## Practice
Feeds the module deliverable at [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md):
1. **Compute both ledgers** for a real (or plausible) fleet slice: pull reserved GPU-hours from allocation records and SM_ACTIVE from DCGM, produce allocated $, utilised $, gap $, and gap %.
2. **Split the gap** into reserved-but-not-running vs running-but-idle using SM_ACTIVE ≈ 0 with no process vs SM_ACTIVE low with a process present. Report each as a dollar figure (sets up lesson 03).
3. **Write the two-line report** a platform lead would send: both ledgers, the gap %, and the dollar waste — the number, not the vibe.

## Self-check
- Two workloads each hold 4 GPUs for 5 hours (20 allocated GPU-hours each) at the same rate; one runs SM_ACTIVE 80%, the other 20%. Do they have the same allocated cost? The same utilised cost? **Answer:** Same allocated cost (both reserved 20 GPU-hours × the same rate — reservation is billed regardless of use). Different utilised cost: 16 vs 4 useful GPU-hours, a 4× gap. Reporting only allocated would make the two look identical while one wastes 80% of its spend.
- Why is SM_ACTIVE, not GPU_UTIL, the correct multiplier for the utilised ledger? **Answer:** GPU_UTIL only reports that a kernel was resident on the device (a process is "on" the GPU), which stays high even when the SMs are starved; SM_ACTIVE reports that the streaming multiprocessors actually did work. Using GPU_UTIL would credit running-but-idle time as useful and collapse the gap — the exact module-05 utilisation lie.
- Where is "all the money" in a typical GPU fleet, and which ledger surfaces it? **Answer:** In the gap between allocated and utilised — the waste ledger, dominated by idle-but-allocated GPUs (reserved-but-not-running plus running-but-idle), all billed at full allocated rate. Only reporting both ledgers together surfaces it; either number alone hides it.

## Resources
- NVIDIA DCGM field identifiers (SM_ACTIVE = `DCGM_FI_PROF_SM_ACTIVE` vs `DCGM_FI_DEV_GPU_UTIL`): https://docs.nvidia.com/datacenter/dcgm/latest/dcgm-api/dcgm-api-field-ids.html
- OpenCost — allocation (idle/utilisation) cost model concepts: https://www.opencost.io/docs/specification
- FinOps Foundation — allocated vs utilised / cost allocation practices: https://www.finops.org/framework/capabilities/allocation/
- Kubernetes resource requests & the reservation model (device requests are the allocation record): https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

---
Module backlink: [💰 11 — GPU cost and unit economics](../README.md)
