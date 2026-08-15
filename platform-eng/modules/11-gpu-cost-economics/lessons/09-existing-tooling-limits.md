---
lesson: 09
title: "Where existing tooling fails: reading the OpenCost source"
module: 11
concept: "tooling GPU attribution gap"
status: not-started
est_time: "5 hrs"
prev: "08-chargeback-showback.md"
next: "10-focus-spec.md"
artifacts: ["a sourced writeup naming the exact file/line where GPU cost attribution breaks in OpenCost, plus a tool-landscape gap matrix, dropped into the module deliverable"]
sources: 8
---

# Where existing tooling fails: reading the OpenCost source

*Prev: [08 — Chargeback, showback & queue-wait billing](08-chargeback-showback.md) · Next: [10 — FOCUS 1.x and designing a GPU cost schema](10-focus-spec.md)*

## Where this fits
Lesson 08 closed the theory arc: you can now attribute a GPU-hour under all four sharing
regimes, split allocated from utilised, and turn both into a chargeback statement. This lesson
asks the question a hiring panel actually asks next — "fine, but doesn't a tool already do this?"
— and answers it by reading the leading open-source cost tool's source code line by line. The gap
this lesson proves is the exact gap lesson 10 spends its whole schema closing, so treat this as
the bridge: everything you find here becomes a row in next lesson's capstone table.

## Why this matters
You will be asked, in an interview or in your first month, "why not just run OpenCost / Kubecost?" The weak answer is a vibe ("it's not great for GPUs"). The answer that gets you hired is a *source citation*: the exact function, the exact field, the exact multiply where the leading open-source cost tool commits to allocation-basis accounting and thereby throws away everything lessons 01–05 told you matters. This lesson is where you earn the right to say "here is line X in `costmodel.go`, and here is why no configuration flag fixes it."

The stake is credibility with two audiences. Engineering leadership will have already deployed OpenCost or bought Kubecost, and will assume the GPU cost problem is *solved*; you need to show, from the code, that the numbers on the dashboard are the **allocated** ledger (lesson 02) — what tenants *reserved* — and are structurally blind to the **utilised** ledger and to sub-device sharing. Finance will make chargeback decisions on those numbers; if a MIG-partitioned or time-sliced fleet is being billed as whole GPUs, someone is being over- or under-charged by a factor you can name.

The payoff is the single most reusable artifact in this module: a precise, sourced statement of *"here is the line where GPU cost attribution breaks in the leading tool, and here's exactly what it would take to fix it."* That paragraph goes straight into the deliverable and into your interview kit.

## What's new here (calibration)
- **Already yours (skip):** the two-ledger model (allocated vs utilised, lesson 02); the four sharing regimes and the time-slice unsolvability proof (lesson 01); the SM_ACTIVE-vs-GPU_UTIL util-lie (lesson 05); the pod-resources-API + DCGM per-UUID join mechanism (04/05); cost-per-token and cost-per-run (07/08). This lesson does **not** re-derive any of it — it shows *where OpenCost's code silently chooses the allocated ledger and drops the rest*.
- **New angle 1:** reading real Go source as a FinOps engineer — locating the exact `nvidia.com/gpu` request read and the `gpuCount × gpuPrice` multiply, and proving from the struct definitions that DCGM utilisation is captured as a **reporting sidecar**, never as a cost input.
- **New angle 2:** the *whole-GPU request* assumption as the root defect — why MIG fractions and time-slice sharers are invisible to a tool whose numerator is a Kubernetes resource request count.
- **New angle 3:** the landscape gap matrix — Kubecost, DCGM-exporter+Grafana, and cloud-native cost tools each hold *one* of {attributed $, utilisation, app counters} and **none join all three**, which is precisely the deliverable's thesis.

## Core concepts

### How OpenCost actually computes GPU cost (from source)
OpenCost is the CNCF cost-model reference implementation; Kubecost is built on the same engine, so reading OpenCost reads the whole lineage. The GPU cost path has two halves: **node-level price** and **pod-level allocation**.

**1. Node GPU price.** In `pkg/costmodel/costmodel.go`, `getGPUCount()` reads the node's advertised GPU capacity from Kubernetes node status — the resource keys `nvidia.com/gpu`, `nvidia.com/gpu.shared`, and `k8s.amazonaws.com/vgpu`. The node's GPU cost is then computed as roughly:

```
gpuPrice := cp.GpuPricing(nodeLabels)   // cloud-provider GPU $/hr, or configured default
newCnode.GPUCost = gpuCount * gpuPrice   // whole-GPU count × price
```

`cp.GpuPricing(nodeLabels)` is the cloud-provider hook. The `Provider` interface that declares it lives at **`pkg/cloud/models/models.go`** (`GpuPricing(map[string]string) (string, error)`, on the `Provider` interface around line 282), and `costmodel.go` imports the package under the alias `costAnalyzerCloud "github.com/opencost/opencost/pkg/cloud/models"`. It resolves a per-GPU hourly price from the provider's instance pricing (on-demand or spot, keyed off node labels like instance type / capacity type), falling back to a **configured default GPU price** when the provider can't price the node (the common on-prem / neocloud case). Note the shape: **one scalar $/GPU-hr for the whole physical device.** There is no notion of a per-MIG-instance or per-time-slice price anywhere in this resolution.

**2. Pod GPU allocation.** The pod's GPU claim is read from the container spec, in `costmodel.go`, from `container.Resources.Requests` (then `Limits`) for the keys `nvidia.com/gpu` and `k8s.amazonaws.com/vgpu`:

```
gpuReqCount = g.AsApproximateFloat64()   // the pod's requested nvidia.com/gpu count
```

That requested count, multiplied by the node's per-GPU price and the pod's running time, becomes the pod's `GPUCost`. **This is the load-bearing line of the whole lesson:** the numerator of GPU cost is a *Kubernetes resource request count*, an integer-valued whole-device claim.

### The struct that proves it: cost ≠ utilisation
In `core/pkg/opencost/allocation.go`, the billed cost fields are plain floats on the `Allocation`:

```go
GPUHours          float64 `json:"gpuHours"`
GPUCost           float64 `json:"gpuCost"`
GPUCostAdjustment float64 `json:"gpuCostAdjustment"`
GPUCostIdle       float64 `json:"gpuCostIdle"`
```

OpenCost *does* query DCGM. `ComputeAllocation` fires `QueryGPUsUsageAvg` / `QueryGPUsUsageMax` (applied via `applyGPUUsageAvg` / `applyGPUUsageMax`) against a DCGM engine-active utilisation metric (the `DCGM_FI_PROF_GR_ENGINE_ACTIVE` family — the SM-active signal from lesson 05, *not* the `DCGM_FI_DEV_GPU_UTIL` util-lie), plus `QueryIsGPUShared` and `QueryGPUInfo`. But look at *where that data lands* — the nested `GPUAllocation` struct:

```go
type GPUAllocation struct {
    GPUDevice         string   `json:"gpuDevice,omitempty"`
    GPUModel          string   `json:"gpuModel,omitempty"`
    GPUUUID           string   `json:"gpuUUID,omitempty"`
    IsGPUShared       *bool    `json:"isGPUShared"`
    GPUUsageAverage   *float64 `json:"gpuUsageAverage"`
    GPURequestAverage *float64 `json:"gpuRequestAverage"`
}
```

`GPUUsageAverage` and `GPURequestAverage` are pointers hung off the allocation as **metadata / efficiency reporting**. They are *not* multiplied into `GPUCost`. The billed `GPUCost` float is produced from the allocated (requested) whole-GPU count × node GPU price × hours. **DCGM utilisation is captured, displayed, and then not billed on.** This is lesson 02 made concrete in code: OpenCost's ledger is the *allocated* ledger; the utilised ledger exists only as a sidecar number for an efficiency panel.

One more signal worth clocking: the parent `Allocation` struct also still carries two fields named `deprecatedGPURequestAverage` and `deprecatedGPUUsageAverage` (tagged `@bingen:field[version=22]`), superseded by the `GPUAllocation` pointer block above (introduced at schema version 23). The schema has already churned once. That's a small but real methodological point — **read the current source before every writeup, don't cite field names from memory** — because the shape of `Allocation` you memorize today may already be the deprecated one next quarter.

### Where MIG and time-slicing fall through the floor
Trace the two shared regimes through the code above:

- **MIG fractional (regime 2).** A pod bound to a `3g.40gb` MIG instance still requests `nvidia.com/gpu: 1` — the device plugin advertises MIG instances as *whole* `nvidia.com/gpu` units. So `gpuReqCount = 1`, and the pod is billed `1 × gpuPrice × hours` — the price of a **whole physical H100**, not 3/7 of one. The MIG *fraction* (lesson 01's SM-slice vs memory basis) never enters the math because the request count is 1 and `gpuPrice` is a whole-device scalar. OpenCost may capture the MIG UUID in `GPUUUID`, but the cost is whole-GPU. A card partitioned into 7 instances, each requesting `nvidia.com/gpu: 1`, would bill **7× the physical card's price.**
- **Time-sliced (regime 3).** With `sharing.timeSlicing`, the device plugin advertises N replicas as `nvidia.com/gpu.shared`; `IsGPUShared` goes true and OpenCost divides the whole-GPU price across the sharers by their **requested** share. That is exactly lesson 01's *request-weighted* approximation — it bills **intent, not use** — and OpenCost has no other option, because (lesson 01's proof) the per-tenant utilisation counter does not physically exist. `GPUUsageAverage` on a shared device is the *device* aggregate, identical across all sharers, so it can't even inform a better split. The time-slice attribution hole is *provably unsolvable from OpenCost's inputs*, and the code confirms it never tries.

### The exact gap, stated for the deliverable
> **OpenCost bills GPUs on the allocated ledger: numerator = `nvidia.com/gpu` request count (`gpuReqCount`, `costmodel.go`), price = whole-device `cp.GpuPricing()` scalar (`pkg/cloud/models/models.go`), cost = count × price × hours (`GPUCost` in `core/pkg/opencost/allocation.go`). DCGM utilisation is queried (`QueryGPUsUsageAvg`) but stored only in `GPUAllocation.GPUUsageAverage` as reporting metadata and is never multiplied into `GPUCost`. Because MIG instances and time-slices both surface as whole `nvidia.com/gpu` requests, sub-device attribution (MIG fraction, per-tenant time-slice utilisation) is structurally invisible — not a config gap, an input-model gap.**

**What it would take to fix it:** (a) a per-instance price basis so a MIG UUID prices at its fractional basis (lesson 01) rather than a whole-device scalar; (b) making `GPUCost` a function of `GPUUsageAverage` for an optional utilised-cost ledger (still bounded by the regime-3 unsolvability for time-slices); (c) a fractional request model (DRA `ResourceClaim` allocation results, lesson 01) so the numerator stops being an integer whole-GPU count. (a) and (c) are tractable; (b) for time-slicing is not, by the lesson-01 proof.

### The landscape gap matrix
None of the incumbents join all three of {attributed $, real utilisation, app-level counters}:

| Tool | GPU cost basis | Utilisation | App counters (tokens/runs) | The gap |
|---|---|---|---|---|
| **OpenCost** | Allocated: `nvidia.com/gpu` request × whole-GPU price | Queried from DCGM but **only reported**, not billed | None | Bills reserved whole-GPUs; blind to MIG fraction & time-slice use |
| **Kubecost** | Same OpenCost engine (allocation-basis) | Same reporting sidecar, plus (2024+) a dedicated GPU-utilisation view | None | Inherits every OpenCost billing gap; the newer utilisation view is visibility, not a new cost basis |
| **DCGM-exporter + Grafana** | **No $ at all** | Ground-truth SM/mem utilisation per UUID | None | Utilisation without dollars or attribution to tenant |
| **Cloud-native cost tools** (CUR/FOCUS exports, Cost Explorer, billing) | Real invoiced $ | None | None | Granularity **stops at the node/instance**, never the GPU-fraction or namespace |

The empty column that no single tool fills — **DCGM utilisation × attributed $ × app counters, joined per tenant** — is exactly the schema the module deliverable builds (lesson 10). That's the thesis: the gap is not "OpenCost is buggy," it's that no incumbent joins the three signals, and two of them (utilisation for time-slices) can't be joined at all under regime 3.

## Perspectives

**Source-reading / engineering perspective.** The transferable skill here isn't "OpenCost bills GPUs wrong" — it's *how you found that out*. The method generalizes to any unfamiliar codebase you're evaluated on in an interview or a first-month onboarding: find the numerator (what quantity gets multiplied by price — here, `gpuReqCount`), find the price (`GpuPricing`, and whether it's per-device or per-fraction), and find where the "interesting" telemetry (DCGM data) actually lands in the data model versus where it's merely displayed. Three greps and a struct read settled a question a dashboard vibe couldn't.

**FinOps-practitioner perspective.** A chargeback dashboard number is a claim, and every claim has a basis. "Team A spent $4,200 on GPUs this month" silently means "Team A *requested* GPU capacity worth $4,200" — not "Team A's workload *consumed* $4,200 of GPU compute." A viewer who doesn't know the difference will treat the two as identical, and a FinOps engineer who can't articulate the difference has no defense when a team disputes a bill on the grounds that "we weren't even using it." Reading the source is how you learn which claim you're actually making before finance repeats it as fact.

**Tooling-landscape perspective.** The gap matrix isn't a criticism of any one vendor's engineering — it's a systemic observation. Cost tools are built by billing teams who default to the metering system's native unit (a Kubernetes resource request); utilisation tools are built by observability teams who default to the driver's native counters (DCGM); app-counter instrumentation is built by ML platform teams who default to inference/training frameworks. Nobody who owns one leg of the join naturally owns the other two, so the three-way join stays undone across the whole industry, not just in one tool. That's the opening the deliverable exploits.

**Interview / credibility perspective.** In a room full of candidates who will all say "OpenCost isn't great with GPUs," the differentiator is precision: a file name, a function name, a struct field, a line-adjacent number ("$21 billed for a $3/hr card"). Vibes are indistinguishable between candidates; a citation is not. This is the cheapest, highest-leverage habit this lesson teaches — always answer "why not tool X" with a location in tool X's source, not an adjective.

## Real-world use cases

- **OpenCost issue #2029, "Incorrect cpu ram and gpu hourly cost on GPU nodes"** — https://github.com/opencost/opencost/issues/2029. Filed July 2023 against a SUSE Rancher RKE2 cluster on AWS (`g4dn.xlarge`, NVIDIA T4): CPU/RAM/GPU hourly costs came out far lower than expected on GPU nodes, with the reporter's own diagnosis pointing at a cost-splitting defect that only manifests once a GPU resource is present. The issue was later closed via fix PR #2299. What it shows: this is the project's *own issue tracker* independently surfacing a GPU-costing defect — a different bug from the one this lesson derives from first principles (a pricing-split bug vs. an allocation-basis/utilisation gap), but confirming, from real users, that GPU cost accuracy in OpenCost is an active, ongoing problem area rather than a solved corner of the tool.
- **Kubecost (Apptio/IBM), "Kubecost Brings NVIDIA GPU cost monitoring for AI workloads in 2.4"** — https://www.apptio.com/blog/gpu-monitoring/ *(fetch blocked by this environment's egress policy for `apptio.com`; canonical URL cited, not independently re-verified in this session — treat as reported, not quoted verbatim)*. The vendor's own release notes describe that prior to Kubecost 2.4, Kubecost could show teams the GPU capacity they had *requested* but not what they had actually *used* — the vendor confirming, in its own words, this lesson's exact "allocated-only, utilisation-as-sidecar" argument. Note the scope carefully: 2.4 added GPU *utilisation visibility* in the UI; nothing in the source read above (`GPUCost` still not a function of `GPUUsageAverage`) suggests the underlying billed-cost formula was re-architected. Read this as "visibility, not re-architecture."
- **OpenCost project (CNCF cost-model reference; Kubecost's engine)** — https://github.com/opencost/opencost. The upstream project itself: confirms the shared lineage between OpenCost and Kubecost (both quoted above) and is the canonical home for every file/line citation in this lesson.

## Worked example
A neocloud runs one H100 80GB node, on-demand priced by the provider at **$3.00/GPU-hour**, MIG-partitioned into 7× `1g.10gb` instances, each rented to a different tenant pod requesting `nvidia.com/gpu: 1`, for one hour.

**What OpenCost reports.** For each pod: `gpuReqCount = 1`, node `gpuPrice = $3.00`, so `GPUCost = 1 × $3.00 × 1h = $3.00`. Seven pods → **$21.00 of attributed GPU cost for a physical card that costs $3.00/hr.** OpenCost has multiplied the real cost by 7 because each MIG instance presents as a whole `nvidia.com/gpu`. The dashboard will also show each pod a `GPUUsageAverage` (say 30%) — a true DCGM number — sitting *next to* a cost figure it had no effect on.

**What's actually true (lesson 01).** SM-slice basis: each `1g.10gb` = 1/7 of the card, so `cost = (1/7) × $3.00 = $0.4286` per tenant, summing to $3.00. OpenCost is off by **7× per tenant** and **7× in aggregate**, and no OpenCost setting corrects it, because the defect is the whole-GPU request count in the numerator.

**The time-slice variant.** Reconfigure the card as `timeSlicing.replicas: 4`. Now `IsGPUShared = true`; OpenCost splits the $3.00 by requested share — request-weighted — billing $0.75 each. If the real context-runtime split is 70/25/2.5/2.5 (lesson 01), the true bills are $2.10 / $0.75 / $0.075 / $0.075. OpenCost's $0.75-each is the *intent* split, wrong by up to ~10× per pod, and — per the lesson-01 proof — un-fixable from its inputs because the per-tenant utilisation counter doesn't exist. The `GPUUsageAverage` it could show is the device aggregate, identical for all four, so it can't even rank them.

## Practice
Feed these into the module deliverable at [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md):
1. **The sourced gap paragraph.** Reproduce the "exact gap" box above *in your own words*, citing the three code locations (`gpuReqCount` in `costmodel.go`; `cp.GpuPricing()` in `pkg/cloud/models/models.go`; `GPUCost` / `GPUAllocation.GPUUsageAverage` in `core/pkg/opencost/allocation.go`). This is the interview writeup.
2. **Reproduce the 7× MIG error.** Take a real or plausible MIG layout on one card, compute what OpenCost would bill (count × whole-GPU price) vs the lesson-01 fractional-basis truth, and report the multiplier of error.
3. **The gap matrix, filled for your stack.** Fill the landscape table for the exact tools your target company runs, and circle the empty cell (joined $ × utilisation × counters) that motivates the deliverable schema.
4. **The fix spec.** Write the three-part "what it would take to fix it" list, and mark which parts are tractable and which are blocked by the regime-3 proof.

## Common pitfalls
1. **Trusting a Kubecost/OpenCost GPU dashboard number as chargeback ground truth.** It's the allocated ledger, not the utilised one, and issue #2029 shows even the allocated number has had its own accuracy bugs. Correction: treat any dashboard $ figure as "what was reserved," verify the basis in source before it goes into an invoice.
2. **Believing a version bump or config flag fixes the MIG/time-slice gap.** It's an input-model defect (whole-GPU request as numerator), not a settings problem — no flag changes what `gpuReqCount` means. Correction: the tool *is* actively evolving (Kubecost 2.4 GPU monitoring, ongoing bug fixes like PR #2299), so re-verify against current source before repeating a finding rather than treating this lesson's line numbers as permanently frozen.
3. **Assuming the DCGM utilisation number shown next to cost in the UI is used in the cost calculation.** It's sidecar metadata — the `GPUAllocation.GPUUsageAverage` struct proof in Core concepts settles this at the type level, not just behaviorally.
4. **Assuming "GPU cost monitoring was added" means MIG/time-slice attribution is solved.** Visibility (showing a utilisation percentage) and billing (using that percentage to compute `GPUCost`) are different claims; Kubecost 2.4 addressed the former, not the latter.
5. **Assuming Kubecost's enterprise tier uses a different cost engine than OpenCost.** It doesn't — Kubecost is built on the OpenCost engine, so every structural finding in this lesson (allocation-basis billing, sidecar utilisation) applies to both.

## Self-check
- In OpenCost, what is the numerator of GPU cost, and why does that make MIG fractions invisible? **Answer:** The numerator is the pod's `nvidia.com/gpu` **resource request count** (`gpuReqCount = g.AsApproximateFloat64()` in `costmodel.go`), multiplied by a whole-device `cp.GpuPricing()` scalar and running hours. MIG instances are advertised by the device plugin as whole `nvidia.com/gpu: 1` units, so a fractional `3g.40gb` instance reads as request count 1 and is billed at the whole physical GPU's price — the fraction (3/7, or 40/80) never enters the math, so a 7-way partition bills 7× the card. It's an input-model gap, not a config flag.
- OpenCost queries DCGM utilisation — so why is it still "blind to the util-lie"? **Answer:** Because the DCGM value lands in `GPUAllocation.GPUUsageAverage` (a `*float64` reporting field in `core/pkg/opencost/allocation.go`) and is **never multiplied into the billed `GPUCost` float**. Cost is the allocated ledger (request count × price × hours); utilisation is a sidecar shown next to it. OpenCost thus reports what you *reserved*, exactly the lesson-02 allocated ledger, with utilisation as decoration rather than a cost basis.
- Which incumbent tool joins attributed-$ × utilisation × app-counters, and what does the answer imply for the deliverable? **Answer:** None. OpenCost/Kubecost have attributed-$ (allocation basis) but only report utilisation and have no app counters; DCGM-exporter+Grafana has utilisation but no $ and no attribution; cloud-native billing has real $ but granularity stops at the node/instance. The empty three-way join is the deliverable's reason to exist — and for time-sliced tenants the utilisation leg of the join is unrecoverable in principle (lesson 01), which the schema must represent as an explicit uncertainty rather than a false number.
- The cloud-provider pricing hook used to be cited as living in `pkg/cloud/`. Where does the `GpuPricing` method actually live, and why does the correction matter? **Answer:** In `pkg/cloud/models/models.go`, as a method on the `Provider` interface (`GpuPricing(map[string]string) (string, error)`), imported into `costmodel.go` as `costAnalyzerCloud "github.com/opencost/opencost/pkg/cloud/models"`. It matters for the same reason as the deprecated `GPURequestAverage`/`GPUUsageAverage` fields on `Allocation`: source layout and schema shift over time, so a citation that was accurate at one commit can silently go stale — always re-verify a path against current source rather than trusting a memorized (or previously-written) citation.

## Connections & what's next
This lesson is where lessons 01–08 stop being theory and become a verified claim about production tooling: the four regimes (01), the two ledgers (02), idle/fragmentation (03/04), and unit economics (05/07/08) all reappear here as the exact things OpenCost's allocated-ledger, whole-GPU-request design throws away. The sourced gap paragraph and the gap matrix are the direct inputs to lesson 10 — the next lesson takes this "here's what's missing" finding and turns it into "here's the schema that supplies it," starting from the FOCUS industry standard and extending it with exactly the columns this lesson proved OpenCost can't produce.

## References & further reading

**Primary sources**
- OpenCost cost-model GPU logic (`gpuReqCount`, `getGPUCount`, node `GPUCost` = count × price) — read for the numerator/price multiply: https://github.com/opencost/opencost/blob/develop/pkg/costmodel/costmodel.go
- OpenCost `Allocation` / `GPUAllocation` struct (`GPUCost`, `GPUUsageAverage`, `GPURequestAverage`, `IsGPUShared`, and the deprecated pre-v23 fields) — read for the cost-vs-utilisation type-level proof: https://github.com/opencost/opencost/blob/develop/core/pkg/opencost/allocation.go
- OpenCost cloud-provider pricing interface (`GpuPricing`, spot/on-demand/default GPU price) — corrected path, read for the whole-device price scalar: https://github.com/opencost/opencost/blob/develop/pkg/cloud/models/models.go
- OpenCost project (CNCF cost-model reference; Kubecost's engine) — read for project scope and lineage: https://github.com/opencost/opencost

**Real-world engineering blogs**
- OpenCost GitHub issue #2029, "Incorrect cpu ram and gpu hourly cost on GPU nodes" — a real, project-tracked GPU costing defect (filed 2023, closed via PR #2299): https://github.com/opencost/opencost/issues/2029
- Kubecost/Apptio, "Kubecost Brings NVIDIA GPU cost monitoring for AI workloads in 2.4" — vendor confirmation of the pre-2.4 "requested, not used" gap, and the scope of what 2.4 actually fixed (visibility, not the billing formula): https://www.apptio.com/blog/gpu-monitoring/

**Deeper dives**
- NVIDIA k8s-device-plugin — how MIG and time-slicing are advertised as `nvidia.com/gpu` / `nvidia.com/gpu.shared`, and where MPS space-partitioning fits as a third sharing mode: https://github.com/NVIDIA/k8s-device-plugin
- DCGM metrics reference (`DCGM_FI_PROF_GR_ENGINE_ACTIVE` vs `DCGM_FI_DEV_GPU_UTIL`) — the utilisation signal OpenCost queries but doesn't bill on: https://docs.nvidia.com/datacenter/dcgm/latest/

---
Module backlink: [💰 11 — GPU cost and unit economics](../README.md)
