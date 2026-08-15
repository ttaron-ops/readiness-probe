---
lesson: 02
title: "Allocated vs utilised cost: the two ledgers"
module: 11
concept: "two-ledger cost model"
status: not-started
est_time: "3 hrs"
prev: "01-attribution-models.md"
next: "03-idle-detection.md"
artifacts: ["a two-ledger (allocated vs utilised) computation for a fleet slice with the gap % and dollar figure, added to the module deliverable"]
sources: 7
---

# Allocated vs utilised cost: the two ledgers

## Where this fits
Lesson 01 established the four sharing regimes and, inside them, a two-sided cost equation: allocation cost (what a workload reserved) and utilisation cost (the fraction of that reservation that did useful work) — a "two ledgers" idea that ran through the whole previous lesson's worked example without yet being named as the module's core accounting model. This lesson formalizes exactly that: it takes the allocation/utilisation split lesson 01 used informally per-regime and turns it into an explicit, always-report-both-together model that applies uniformly across all four regimes. Everything from here forward in the module — idle detection, fragmentation, unit economics, chargeback — is built on top of this formalized two-ledger split.

## Why this matters
The bill you pay and the value you got are two different numbers, and almost every GPU cost mistake is a confusion between them. **Allocated cost** is what a workload *reserved* — it hits the invoice whether the GPU computed anything or sat warm and idle. **Utilised cost** is the slice of that reservation that did useful work. The difference between the two is the **waste ledger**, and on real GPU fleets the waste ledger is where essentially all the money is — reserved-but-idle H100s at $2–3+/hour each, multiplied by fleet size, is a seven-figure line item hiding behind a healthy-looking utilisation dashboard.

A platform engineer who reports only one of these numbers is lying by omission. Report only *allocated* and you look fully committed while burning cash on idle silicon. Report only *utilised* and you understate what procurement must actually pay for, so capacity planning collapses. Neocloud FinOps and platform-eng cost ownership are both judged on holding both ledgers at once and naming the gap out loud. This lesson is the conceptual hinge for the whole module — short by design, because the machinery (attribution regimes in lesson 01, the SM_ACTIVE signal in module 05) is already built; this just wires them into two ledgers.

## What's new here (calibration)
- **Already yours (skip):** SM_ACTIVE vs GPU_UTIL as the correct utilisation signal — GPU_UTIL only says "a kernel was resident," SM_ACTIVE says "the SMs did work" (module 05, its dashboard). The attribution mechanism and sharing regimes (lesson 01, module 04/05).
- **New angle 1:** the explicit **two-ledger model** — allocated vs utilised as *separate books you always report together*, with the gap as a first-class number.
- **New angle 2:** the **reservation-vs-consumption asymmetry** — reservation is billed in whole GPU-hours the instant you bind; consumption is a fraction earned back only when SMs actually work, so the two ledgers drift apart structurally, not accidentally.

## Core concepts

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

> **2026 snapshot — where the gap actually lands in production.** Uber reports managing 5,000+ GPUs across on-prem infrastructure plus OCI and GCP, and has explicitly invested in elastic cross-team GPU sharing so teams can opportunistically use other teams' idle reserved capacity — a direct, named-scale instance of exactly the allocated-vs-idle problem this lesson formalizes. Anyscale's account of the same failure mode gives it a dollar figure: one described deployment sat at 12.5% GPU utilization with 224 GPUs idle just to supply enough CPU alongside them, and the firm's broader industry read is that many AI clusters run at only 30–50% GPU utilization — meaning "a 30% utilization rate on a $100M GPU investment [means] $70M is sitting idle." Both figures are dated 2026-era snapshots from named companies, not universal constants, but they anchor how large the gap ledger gets at real scale.

## Perspectives

**Finance/procurement perspective.** Allocated cost is the number that sizes what has to be bought or reserved. A committed-use discount, a capacity reservation, or a new cluster order is sized against allocated GPU-hours, because that's what's contractually held regardless of whether it's busy. A procurement plan built off the utilised ledger would under-buy — it would assume the fleet only needs as much capacity as work is currently landing on, ignoring the reserved-but-idle capacity that's already committed and will be committed again.

**SRE/platform-efficiency perspective.** Utilised cost is the number an efficiency program is judged against. If your job is to raise fleet efficiency, the allocated ledger doesn't move when you fix a dataloader bottleneck or right-size an inference replica — the utilised ledger does, and the gap closing is the entire measure of whether the efficiency work mattered. Reporting allocated-only to an SRE efficiency team gives them no signal to act on.

**Workload-owner perspective.** From inside a workload, "my pod shows Running" feels like "I'm doing useful work" — but that's an allocation-side fact (the device is bound, the process is scheduled), completely invisible to whether SM_ACTIVE is anywhere above zero. A workload owner watching their own pod's status has no native visibility into which ledger they're actually costing the fleet on; they need the SM_ACTIVE-derived utilised number surfaced to them explicitly, or they will (reasonably, from their vantage point) assume a running pod is a working pod.

**Executive/reporting perspective.** A single blended "utilization %" headline number is actively dangerous, not just incomplete — it hides which ledger moved. A utilization percentage going from 40% to 55% could mean the fleet got more efficient (utilised ledger improved) or it could mean less capacity got reserved in the first place (allocated ledger shrank), and those call for opposite responses. An executive making a headcount or procurement decision off one blended KPI is, more often than the dashboard implies, deciding on the wrong signal.

## Real-world use cases

- **Uber Engineering, "Scaling AI/ML Infrastructure at Uber"** — https://www.uber.com/en-US/blog/scaling-ai-ml-infrastructure-at-uber/. What it shows: Uber manages 5,000+ GPUs across on-prem infrastructure plus OCI and GCP, and has invested specifically in elastic cross-team GPU sharing so teams can opportunistically use other teams' idle reserved capacity — a direct, named-scale instance of the allocated-vs-idle problem this lesson formalizes.
- **Anyscale, "GPU (In)efficiency in AI Workloads"** — https://www.anyscale.com/blog/gpu-in-efficiency-in-ai-workloads. What it shows: a concrete deployment example at 12.5% GPU utilization with 224 GPUs sitting idle just to supply enough CPU, plus the industry-level claim that many AI clusters run at only 30–50% GPU utilization — "a 30% utilization rate on a $100M GPU investment means $70M is sitting idle." A strong dollar-denominated illustration of the waste ledger.
- **Microsoft Research, the Philly trace (reused from lesson 01)** — https://www.usenix.org/conference/atc19/presentation/jeon. What it shows: per-minute GPU utilization data across a real multi-tenant production fleet, useful here as independent evidence of how much utilization varies job-to-job — the empirical texture behind "the gap is the default state, not an anomaly."

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

## Common pitfalls
1. **Treating `DCGM_FI_DEV_GPU_UTIL` as "close enough" to SM_ACTIVE.** It isn't — GPU_UTIL only reports kernel residency, not whether the SMs are actually doing work, and the gap between them is exactly where the Anyscale-reported 30%→$70M-idle stakes live. Using GPU_UTIL for the utilised ledger systematically overstates useful work.
2. **Believing allocated GPU-hours are themselves a waste signal.** They aren't — allocated is the reservation, the number the bill charges no matter what. The waste ledger is the *gap* between allocated and utilised, not the allocated figure on its own.
3. **Reporting one blended utilization % without splitting reserved-but-not-running from running-but-idle.** These have different owners and different fixes (capacity planning vs application right-sizing) and get conflated by a single number. Forward-ref lesson 03, which builds the full taxonomy this pitfall is missing.
4. **Assuming utilised-ledger billing is "fairer" to teams than allocated-ledger billing.** Charging purely on utilisation under-recovers the fixed cost of reserved capacity and actively rewards teams for reserving generously and running lightly — the opposite of the intended incentive. Forward-ref lesson 08's charge-allocated / report-utilised policy design.
5. **Comparing "allocated cost" across two teams on different commitment mixes without normalizing the rate.** A team on spot pricing and a team on a 3-year reserved rate will show very different allocated-cost totals for the same GPU-hour volume; comparing the dollar figures directly conflates volume and rate. Forward-ref lesson 06 (commitment strategy).

## Self-check
- Two workloads each hold 4 GPUs for 5 hours (20 allocated GPU-hours each) at the same rate; one runs SM_ACTIVE 80%, the other 20%. Do they have the same allocated cost? The same utilised cost? **Answer:** Same allocated cost (both reserved 20 GPU-hours × the same rate — reservation is billed regardless of use). Different utilised cost: 16 vs 4 useful GPU-hours, a 4× gap. Reporting only allocated would make the two look identical while one wastes 80% of its spend.
- Why is SM_ACTIVE, not GPU_UTIL, the correct multiplier for the utilised ledger? **Answer:** GPU_UTIL only reports that a kernel was resident on the device (a process is "on" the GPU), which stays high even when the SMs are starved; SM_ACTIVE reports that the streaming multiprocessors actually did work. Using GPU_UTIL would credit running-but-idle time as useful and collapse the gap — the exact module-05 utilisation lie.
- Where is "all the money" in a typical GPU fleet, and which ledger surfaces it? **Answer:** In the gap between allocated and utilised — the waste ledger, dominated by idle-but-allocated GPUs (reserved-but-not-running plus running-but-idle), all billed at full allocated rate. Only reporting both ledgers together surfaces it; either number alone hides it.
- A dashboard reports "utilization improved from 40% to 55% this quarter." Why is that number, on its own, ambiguous about whether the fleet actually got more efficient? **Answer:** A single blended utilization percentage doesn't say which ledger moved. It could mean SM_ACTIVE-derived useful work genuinely increased against the same reservation (real efficiency gain), or it could mean allocated GPU-hours shrank — teams reserved less capacity — while the absolute useful work stayed flat or fell, which inflates the ratio without anyone doing anything more efficiently. You need both ledgers reported separately to tell the two apart.

## Connections & what's next
This lesson connects backward to lesson 01: both ledgers use the same rate and the same attribution regime, differing only by the multiplier (1 for allocated, the SM_ACTIVE-derived fraction for utilised). It connects forward to lesson 03, which splits the gap this lesson defines into a taxonomy of idle states (reserved-but-not-running vs running-but-idle, and the confidence with which each can be detected per sharing regime); to lesson 05, which discounts a utilised GPU-hour by this same fraction to get cost-per-token and cost-per-run; to lesson 08, which designs a charge-allocated-report-utilised billing policy directly on top of this split; and to lesson 09, which shows that OpenCost, by construction, only ever reports the allocated ledger. Next: **lesson 03** takes the single "gap%" number this lesson produces and breaks it into the idle-state taxonomy needed to actually act on it.

## References & further reading

**Primary sources**
- NVIDIA DCGM field identifiers — SM_ACTIVE (`DCGM_FI_PROF_SM_ACTIVE`) vs GPU_UTIL (`DCGM_FI_DEV_GPU_UTIL`), the exact fields this lesson's utilised ledger depends on: https://docs.nvidia.com/datacenter/dcgm/latest/dcgm-api/dcgm-api-field-ids.html
- OpenCost — allocation and idle-cost model concepts, read for how (and how narrowly) the leading OSS tool models this split today: https://www.opencost.io/docs/specification
- FinOps Foundation — allocated vs utilised / cost allocation practices, the industry framework's take on the same two-ledger idea: https://www.finops.org/framework/capabilities/allocation/
- Kubernetes resource requests and the reservation model — read for why device requests *are* the allocation record this lesson's allocated ledger is built from: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

**Real-world engineering blogs**
- Uber Engineering, "Scaling AI/ML Infrastructure at Uber": https://www.uber.com/en-US/blog/scaling-ai-ml-infrastructure-at-uber/ — what it shows: 5,000+ GPUs across on-prem/OCI/GCP and elastic cross-team sharing built specifically to address idle reserved capacity.
- Anyscale, "GPU (In)efficiency in AI Workloads": https://www.anyscale.com/blog/gpu-in-efficiency-in-ai-workloads — what it shows: named deployments at 12.5% utilization with 224 idle GPUs, and the "$70M idle on a $100M investment at 30% utilization" industry framing.

**Deeper dives**
- Jeon et al., "Analysis of Large-Scale Multi-Tenant GPU Clusters for DNN Training Workloads" (USENIX ATC'19): https://www.usenix.org/conference/atc19/presentation/jeon — real per-minute utilization data across a production multi-tenant fleet, useful for building your own gap-distribution analysis.

---
Module backlink: [💰 11 — GPU cost and unit economics](../README.md)
