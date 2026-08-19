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
sources: 16
---

# Where existing tooling fails: reading the OpenCost source

*Prev: [08 — Chargeback, showback & queue-wait billing](08-chargeback-showback.md) · Next: [10 — FOCUS 1.x and designing a GPU cost schema](10-focus-spec.md)*

## Where this fits

Lesson 08 finished the design. You can now attribute a GPU-hour under all four sharing regimes, split the allocated ledger from the utilised one, dispose of the unallocatable remainder, price the queue, and defend the resulting invoice line in a dispute. That is a specification for what a GPU chargeback system must do.

This lesson is the gap analysis against the incumbent. A hiring panel — and your future engineering director, who has already installed something — will ask the obvious next question: *"doesn't a tool already do this?"* The answer is a source reading, not an opinion. This lesson opens the leading open-source Kubernetes cost tool, follows a GPU dollar from the node's price metric to the pod's `gpuCost` field, and names every point where the design from lesson 08 is not implemented and why.

Two things make this lesson worth more than the usual "OpenCost isn't great with GPUs" line. First, everything here is a citation with a file, a function, and a line — reproducible against the tree, at a named commit. Second, the story has genuinely changed in 2026: OpenCost now ships a feature-gated module that *does* join dollars, utilisation and application token counters for vLLM inference. Knowing that, and being able to say precisely how far it goes and where it stops, is a sharper credential than repeating a gap that has partially closed.

Everything found here becomes a row in lesson 10's schema.

## Why this matters

The weak answer to "why not just run OpenCost or Kubecost?" is a vibe. The answer that gets you hired is a location in the source: *"`pkg/costmodel/allocation_helpers.go`, `applyNodesToPod` — `alloc.GPUCost = alloc.GPUHours * node.CostPerGPUHr`, and `GPUHours` is set thirty lines earlier from the pod's `nvidia.com/gpu` **request** count. That is the allocated ledger, and no configuration flag changes what the numerator means."* One sentence, three verifiable facts, and the conversation moves to what you would build instead.

The stakes are not rhetorical. Two audiences make decisions on these numbers:

- **Engineering leadership** has usually already deployed something and assumes the problem is solved. If your fleet is MIG-partitioned or time-sliced, the number on that dashboard can be wrong by an integer factor in *either direction* — the same tool over-bills a 7-way MIG card by 7× and, under one very common device-plugin configuration, bills time-sliced pods **zero**. Both errors are structural, both are derivable from the source, and neither announces itself.
- **Finance** will repeat those numbers as fact. A chargeback programme built on an unexamined tool inherits its basis silently: "team A spent $4,200 on GPUs" quietly means "team A *requested* GPU capacity nominally worth $4,200," which is a different claim with a different defence, and lesson 08's dispute protocol depends on knowing which claim you are making.

There is also a transferable skill here that outlives any one tool. Reading an unfamiliar cost engine is a three-question procedure — *what is the numerator, where does the price come from, and where does the interesting telemetry actually land?* — and three greps settle questions that a dashboard cannot. That procedure is the thing to demonstrate in an interview; OpenCost is just the case study it is applied to.

## What's new here (calibration)

- **Already yours (skip):** the four sharing regimes and the time-slice unsolvability argument (01); the two ledgers (02); the `SM_ACTIVE`-vs-`GPU_UTIL` distinction (05); the pod-resources × DCGM join (module 04); the chargeback design this lesson measures the tool against (08). None of it is re-derived.
- **Genuinely new — the complete GPU cost path in OpenCost, traced from source.** Not one line, the whole path: which Prometheus metric the emitter writes, the exact PromQL of all seven GPU queries, which `apply*` function consumes each, which struct field it lands in, and where the multiply happens. With a data-flow diagram you can hold in your head.
- **Genuinely new — six structural gaps, each with its mechanism.** Including three that are not in any write-up of this tool: the `nvidia.com/gpu.shared` blind spot that can bill time-sliced pods **$0.00**, a vGPU-scaling correction branch that is **unreachable as written**, and a headline efficiency metric that **excludes GPU cost from both its numerator and its denominator**.
- **Genuinely new — `pkg/inferencecost`, the 2026 development that partly closes the gap.** The feature-gated module that joins allocation cost to vLLM token counters and produces `llm_cost_per_million_tokens`, with its env vars, its two cost bases, its compute-time input/output split, and — precisely — the four things it still does not fix.
- **Genuinely new — the project's own issue tracker as corroboration.** Issue #3900 (filed 2026-07-05, closed 2026-07-09) states this lesson's argument in the maintainers' own bug tracker, down to the file and line; issue #3828 (open) is the fractional-GPU gap filed by a user who is working around it exactly the way this module's deliverable does.
- **Genuinely new — a capability matrix built to be quoted**, with the mechanism behind every "no" rather than an empty cell.

## Core concepts

### 1. How to read a cost engine: three questions

Before any specific source, fix the method, because it is the part that transfers. Every cost model, in every tool, reduces to:

```
    cost  =  QUANTITY  ×  PRICE  ×  TIME
             ↑            ↑
             │            └── where does the price come from, and what
             │                is it the price OF? (a whole device? a
             │                fraction? a fungible unit?)
             │
             └── what is measured or recorded to produce this number,
                 and is it a REQUEST, a LIMIT, an ALLOCATION, or a
                 MEASUREMENT? These are four different quantities and
                 tools use the words interchangeably.
```

So: **(1) find the numerator** — grep for the resource key (`nvidia.com/gpu`) and see what is read and from where. **(2) find the price** — trace the `$/hr` figure back to its provider hook and ask what unit it is denominated in. **(3) find where the interesting telemetry lands** — grep for the DCGM metric name and follow the value into the data model. Question 3 is the one that separates reporting from billing: telemetry that lands in a display field is decoration; telemetry that lands in a multiply is a cost basis.

Apply it to OpenCost. **Repository state for everything below: `github.com/opencost/opencost`, branch `develop`, commit `30608b8`, committed 2026-08-12, read on 2026-08-17.** Line numbers move; function names and struct fields are stable enough to cite. Re-verify before you put a line number in a writeup — the schema has already churned once (see §3, G4).

### 2. The GPU cost path, end to end

OpenCost is the CNCF cost-model reference implementation, donated by Kubecost and still the engine lineage the commercial product descends from. Its GPU cost path has three halves that meet in one multiply.

```
  OPENCOST'S GPU COST PATH  (repo @ develop, commit 30608b8, 2026-08-12)
  ══════════════════════════════════════════════════════════════════════════

  ┌── A · WHAT OPENCOST *EMITS* into Prometheus ─────────────────────────────┐
  │                                                                          │
  │  pkg/costmodel/metrics.go                                                │
  │    node_gpu_hourly_cost{node,instance_type,provider_id,...}              │
  │      ← node.GPUCost, resolved in costmodel.go from the cloud provider    │
  │        (cp.GpuPricing(nodeLabels)) or the configured default GPU price.  │
  │      ▸ UNIT: dollars per PHYSICAL GPU per hour. One scalar. There is no  │
  │        per-MIG-instance or per-replica price anywhere in this path.      │
  │                                                                          │
  │    container_gpu_allocation{namespace,pod,container,node,uid}            │
  │      ← gpualloc := costs.GPUReq[0].Value          ← THE REQUEST COUNT    │
  │      ▸ the source comment above it reads, in the project's own words,    │
  │        that allocation is set to the request because shared GPU usage    │
  │        is not yet supported.                                             │
  │      ▸ a vGPU correction `gpualloc *= gpu/vgpu` sits right below it and  │
  │        is UNREACHABLE as written — see §3 G3.                            │
  └────────────────────────────┬─────────────────────────────────────────────┘
                               │  (Prometheus, or the collector-source module)
                               ▼
  ┌── B · WHAT OPENCOST *QUERIES* back — 7 GPU queries ──────────────────────┐
  │  pkg/costmodel/allocation.go lines ~294–303 fire these concurrently;     │
  │  the PromQL lives in modules/prometheus-source/pkg/prom/metricsquerier.go│
  │                                                                          │
  │  QueryGPUsAllocated     avg(avg_over_time(container_gpu_allocation{…}))  │
  │  QueryGPUsRequested     avg(avg_over_time(kube_pod_container_resource_   │
  │                           requests{resource="nvidia_com_gpu", …}))       │
  │                           ▸▸ NOTE THE FILTER. `nvidia_com_gpu` ONLY.     │
  │  QueryIsGPUShared       kube_pod_container_resource_requests{unit=       │
  │                           "integer"} by (…, resource, …)                 │
  │  QueryGPUsUsageAvg      avg(avg_over_time(                               │
  │                           DCGM_FI_PROF_GR_ENGINE_ACTIVE{container!=""})) │
  │  QueryGPUsUsageMax      max(max_over_time(DCGM_FI_PROF_GR_ENGINE_ACTIVE))│
  │  QueryGPUInfo           avg(avg_over_time(DCGM_FI_DEV_DEC_UTIL{…}))      │
  │                           by (…, device, modelName, UUID, …)             │
  │                           ▸ queried for the LABELS (UUID/model), not the │
  │                             value — DEC_UTIL is the video-decoder gauge  │
  │  QueryNodeGPUPricePerHr avg(avg_over_time(node_gpu_hourly_cost{…}))      │
  └────────────────────────────┬─────────────────────────────────────────────┘
                               ▼
  ┌── C · WHERE EACH RESULT LANDS  (pkg/costmodel/allocation_helpers.go) ────┐
  │                                                                          │
  │  applyGPUsAllocated()   ~line 929                                        │
  │      alloc.GPUHours = res.Data[0].Value * hrs      ◀── THE NUMERATOR     │
  │      // source comment: GPUHours reflects the full reserved GPU          │
  │      // allocation (request × hours); for usage-based accounting,        │
  │      // apply GPUUsageAverage separately.                                │
  │      also sets  GPUAllocation.GPURequestAverage                          │
  │                                                                          │
  │  applyGPUUsageAvg()     →  GPUAllocation.GPUUsageAverage  (*float64)     │
  │  applyGPUUsageMax()     →  RawAllocationOnly.GPUUsageMax                 │
  │  applyGPUUsageShared()  →  GPUAllocation.IsGPUShared      (*bool)        │
  │  applyGPUInfo()         →  GPUAllocation.{GPUDevice,GPUModel,GPUUUID}    │
  │  applyNodeCostPerGPUHr()→  nodePricing.CostPerGPUHr                      │
  │                                                                          │
  │  applyNodeDiscount()    ~line 1910                                       │
  │      node.CostPerCPUHr    *= (1 - discount)                              │
  │      node.CostPerRAMGiBHr *= (1 - discount)                              │
  │      ▸▸ THERE IS NO GPU LINE HERE. See §3 G5.                            │
  │                                                                          │
  │  applyNodesToPod()      ~line 1929      ◀══ THE ENTIRE COST MODEL ══     │
  │      alloc.CPUCost = alloc.CPUCoreHours              * node.CostPerCPUHr │
  │      alloc.RAMCost = (alloc.RAMByteHours/1024³)      * node.CostPerRAMGiBHr
  │      alloc.GPUCost = alloc.GPUHours                  * node.CostPerGPUHr │
  │                      └── request × hours ──┘          └── $/physical GPU │
  └────────────────────────────┬─────────────────────────────────────────────┘
                               ▼
  ┌── D · THE DATA MODEL  (core/pkg/opencost/allocation.go) ─────────────────┐
  │  type Allocation struct {                                                │
  │      GPUHours          float64  `json:"gpuHours"`          ← BILLED      │
  │      GPUCost           float64  `json:"gpuCost"`           ← BILLED      │
  │      GPUCostAdjustment float64  `json:"gpuCostAdjustment"`               │
  │      GPUCostIdle       float64  `json:"gpuCostIdle"` //@bingen:ignore    │
  │      …                                                                   │
  │      deprecatedGPURequestAverage float64 //@bingen:field[version=22]     │
  │      deprecatedGPUUsageAverage   float64 //@bingen:field[version=22]     │
  │      GPUAllocation  *GPUAllocation       //@bingen:field[version=23]     │
  │  }                                                                       │
  │  type GPUAllocation struct {                                             │
  │      GPUDevice, GPUModel, GPUUUID  string                                │
  │      IsGPUShared       *bool                                             │
  │      GPUUsageAverage   *float64   ◀── the DCGM number lands HERE …       │
  │      GPURequestAverage *float64                                          │
  │  }                                                                       │
  │                                                                          │
  │  ▸▸ … and is read by exactly one thing in the allocation path:           │
  │     GPUEfficiency() = GPUUsageAverage / GPURequestAverage — a DISPLAY    │
  │     ratio. It is never multiplied into GPUCost.                          │
  └──────────────────────────────────────────────────────────────────────────┘
```

Read the diagram once more with the three questions in hand. **Numerator:** a Kubernetes resource *request* count, integer-valued, whole-device. **Price:** one scalar dollars-per-physical-GPU-hour, resolved from the provider or a configured default. **Telemetry:** DCGM's `DCGM_FI_PROF_GR_ENGINE_ACTIVE` — correctly the graphics/SM-engine active signal from lesson 05, not the `DCGM_FI_DEV_GPU_UTIL` util-lie — is genuinely queried, genuinely stored, and lands in a pointer field consumed only by a display ratio.

**That is lesson 02's allocated ledger, in code.** OpenCost is not broken and it is not lying: it is an allocated-ledger engine, and on a fleet of exclusively-held whole GPUs it is *correct*. Every gap below is a consequence of applying that one design to regimes it was not built for.

### 3. The six structural gaps

Each gap gets the same treatment: what breaks, the mechanism, and what it does to a dollar figure.

#### G1 — The numerator is a whole-GPU request count, so MIG is invisible

**What breaks.** A pod bound to a `1g.10gb` MIG instance requests `nvidia.com/gpu: 1`, because the NVIDIA device plugin advertises MIG instances as whole `nvidia.com/gpu` units. `applyGPUsAllocated` sets `GPUHours = 1 × hrs`, `applyNodesToPod` multiplies by the *physical card's* `CostPerGPUHr`, and the pod is billed the price of a whole H100. Partition the card seven ways and the seven pods bill **7× the card's actual cost**.

**Mechanism, stated precisely.** The fraction never enters the arithmetic because there is no term in the arithmetic that could carry it. `GPUCost = GPUHours × CostPerGPUHr` has exactly two factors: a request count that is an integer, and a price denominated per physical device. A MIG fraction is neither. It would need a third term (a fractional basis) or a differently-denominated price, and adding either changes the model's shape, not a config value.

**The strongest evidence is negative and it is easy to reproduce.** `grep -ri '\bmig\b\|multi-instance' --include='*.go' --include='*.md' .` across the whole repository at commit `30608b8` returns nothing outside the word "migration." **The concept of a MIG instance does not appear in the tool's source or its documentation.** OpenCost may capture a MIG instance's UUID in `GPUAllocation.GPUUUID` — `QueryGPUInfo` groups by `UUID` and DCGM reports MIG UUIDs — but nothing downstream interprets it as a fraction of anything.

**Corroboration from the project's own tracker.** Issue **#3828, "Support HAMi GPU core and memory allocation in cost calculations"** (filed 2026-06-02, still open as of this reading). A user running HAMi — a different fractional-GPU mechanism that advertises `nvidia.com/gpucores` and `nvidia.com/gpumem-percentage` alongside `nvidia.com/gpu: 1` — reports that OpenCost sees only `nvidia.com/gpu: 1` and charges a workload allocated 25% of a card as if it held the whole thing. Their stated workaround is to query OpenCost's allocation API, look up the intended fraction from HAMi's metadata, and multiply `gpuCost` by it outside the tool. **That is precisely the correction layer this module's deliverable builds**, filed as a feature request by a stranger, which is about as good as external validation of a thesis gets. The issue also usefully dates the surrounding work: #934 added support for fractional `nvidia.com/gpu` *values*, #2862 improved costing with the `nvidia.com/gpu.replicas` label, and #3039 requested vGPU core/memory usage support.

#### G2 — The `nvidia.com/gpu.shared` blind spot: time-sliced pods can bill $0.00

This is the sharpest finding in the lesson and it is a two-line read.

**What breaks.** NVIDIA's GPU Feature Discovery, when configured with `renameByDefault=true`, advertises time-sliced replicas under the resource key **`nvidia.com/gpu.shared`** rather than `nvidia.com/gpu`. Pods on such a node request `nvidia.com/gpu.shared: 1`. Now follow both of OpenCost's numerator paths:

```
  PATH 1 — the in-process emitter:
    pkg/costmodel/costmodel.go, ~line 388:
        gpuReqCount := 0.0
        if g, ok := container.Resources.Requests["nvidia.com/gpu"];    ok {…}
        else if g, ok := container.Resources.Limits["nvidia.com/gpu"];  ok {…}
        else if g, ok := container.Resources.Requests["k8s.amazonaws.com/vgpu"]; ok {…}
        else if g, ok := container.Resources.Limits["k8s.amazonaws.com/vgpu"];   ok {…}
        ▸ "nvidia.com/gpu.shared" is not in that chain. gpuReqCount stays 0.0.

  PATH 2 — the Prometheus query:
    QueryGPUsRequested's PromQL filters  resource="nvidia_com_gpu"
        avg(avg_over_time(kube_pod_container_resource_requests{
              resource="nvidia_com_gpu", container!="", container!="POD",
              node!="", …}[window])) by (container, pod, namespace, node, uid, …)
        ▸ kube-state-metrics labels the shared resource `nvidia_com_gpu_shared`.
          The filter excludes it. No series is returned for the pod.

  RESULT (both paths):  GPUHours = 0  →  GPUCost = 0 × CostPerGPUHr = $0.00
```

**The mechanism is a string mismatch, not a modelling choice**, which is what makes it so easy to miss: the tool is doing exactly what it was told, and the resource key it was told about is one of two keys the device plugin might use. `nvidia.com/gpu.shared` *is* known elsewhere in the codebase — `core/pkg/model/kubemodel/resource.go` defines `ResourceNvidiaShared = "nvidia.com/gpu.shared"`, `getGPUCount()` reads `n.Status.Capacity["nvidia.com/gpu.shared"]` to count a node's vGPUs, and `applyGPUUsageShared()` sets `IsGPUShared = true` when it sees a `nvidia_com_gpu_shared` series. The key is recognised for *counting* and for *flagging*, and not for *billing*.

**What it does to a number.** On a node whose GPUs are time-sliced with `renameByDefault=true`, tenant GPU cost is zero and the whole card's cost falls into idle. The chargeback statement from lesson 08 would show the sharing tenants holding nothing, and the platform absorbing 100% of a fully-utilised card. **A cost tool reporting $0.00 is more dangerous than one reporting the wrong non-zero number, because nobody disputes a zero.**

**With `renameByDefault=false`** — the other common configuration — replicas are advertised under `nvidia.com/gpu` instead, so the node's `Status.Capacity["nvidia.com/gpu"]` shows N× the physical count and each of the N sharing pods requests 1 and bills a full `CostPerGPUHr`. The error flips sign: **N× over-billing instead of zero.** Which of the two you get is decided by a device-plugin flag that no one in the cost conversation is usually aware of. That is the fact to carry into an interview: *the same tool, on the same hardware, with the same workload, produces either a zero or an N× over-charge depending on a GPU-operator setting.*

*Caveat, stated because it matters:* this is derived from reading the source paths above, not from a captured run. Reproduce it on a time-sliced node before quoting the dollar figures — and reproducing it is a five-minute experiment that makes the finding yours.

#### G3 — The vGPU correction branch is unreachable as written

Sitting directly under the `container_gpu_allocation` emit in `pkg/costmodel/metrics.go` is a correction intended to divide a shared-GPU request down to its physical fraction — the very fix G2's second case needs:

```go
// pkg/costmodel/metrics.go, in the GPUReq branch
// preceding comment: allocation here is set to the request because
// shared GPU usage is not yet supported; if vGPUs, request × (actual/virtual)
vgpu := 0.0
gpu  := 0.0
var err, verr error
if matchedNode, found := nodes[nodeName]; found {
    vgpu, verr = strconv.ParseFloat(matchedNode.VGPU, 64)
    gpu,  err  = strconv.ParseFloat(matchedNode.GPU,  64)
}

gpualloc := costs.GPUReq[0].Value
if verr != nil && err != nil && vgpu != 0 {          // ◀── read this condition
    gpualloc = gpualloc * (gpu / vgpu)
}
```

Walk the condition. It requires **both** `ParseFloat` calls to have *failed* **and** `vgpu` to be non-zero:

- Both parses succeed (the normal case, node found, values present) → `verr == nil` → branch skipped → **no scaling, request billed at full physical price.**
- Both parses fail on a syntax error (empty or malformed strings, the case the branch was presumably written for) → Go's `strconv.ParseFloat` returns **0** on a syntax error → `vgpu == 0` → branch skipped.
- The only way in is a *range* error on both, which returns ±`Inf` — and then `gpu / vgpu` is `Inf/Inf` = `NaN`.

**So the branch either does not execute or produces `NaN`.** The intent is legible and correct; the guard inverts it. This is the most useful kind of source finding to carry into an interview because it demonstrates something a changelog cannot: that you read the condition rather than the comment. State it carefully — "as written at commit `30608b8`, this branch is effectively unreachable" — and note that a maintainer may fix it the week after you read it, which is exactly why you cite a commit.

#### G4 — Utilisation is queried, stored, and not billed

**What breaks.** `GPUCost` is a function of `GPUHours` and price. `GPUUsageAverage` — the DCGM SM-active fraction — is not an input to it. This is visible at the type level rather than the behavioural level, which is why it is worth showing: the billed fields on `Allocation` are plain `float64`s, and the utilisation fields hang off a separate nested pointer struct that no cost function reads.

```go
// core/pkg/opencost/allocation.go — the two families, side by side
GPUHours float64 `json:"gpuHours"`   // billed
GPUCost  float64 `json:"gpuCost"`    // billed
…
GPUAllocation *GPUAllocation `json:"GPUAllocation"` //@bingen:field[version=23]

type GPUAllocation struct {
    GPUDevice, GPUModel, GPUUUID string
    IsGPUShared       *bool
    GPUUsageAverage   *float64   // reporting
    GPURequestAverage *float64   // reporting
}

// the ONLY consumer of GPUUsageAverage in the allocation path:
func (a *Allocation) GPUEfficiency() float64 {
    …
    return *a.GPUAllocation.GPUUsageAverage / *a.GPUAllocation.GPURequestAverage
}
```

**The mechanism is the design, not an oversight**, and the source says so in its own comment at `applyGPUsAllocated`: `GPUHours` reflects the full reserved allocation, and usage-based accounting is to be done by applying `GPUUsageAverage` separately. That is a deliberate ledger choice — which lesson 02 argues is the *right* one for billing. The failure mode is not the choice; it is that a viewer sees a true DCGM utilisation percentage rendered next to a dollar figure it had no effect on, and reasonably assumes one informed the other.

**Corroboration, and this is the one to quote.** OpenCost issue **#3900, "GPU 'usage' cost basis does not reflect actual GPU utilization"** (filed 2026-07-05, closed 2026-07-09). The reporter observes that the inference-cost API's two cost bases, `costBasis=usage` and `costBasis=allocation`, differ by typically under 1% even for a vLLM server running at ~11% GPU utilisation, and traces it to the root: both bases call `ComputeAllocation` with different `ShareIdle`/`ShareSplit` options, so the only difference between them is whether idle and shared-infrastructure cost is *redistributed* — not whether utilisation is factored in. Their expected-versus-actual is the cleanest possible statement of this gap:

```
  1 GPU × 1 h × $2.50/hr, DCGM-measured utilisation ≈ 11%
    costBasis=allocation :  ≈ $2.50   (correct by design)
    costBasis=usage      :  ≈ $0.275  (expected)
    costBasis=usage      :  ≈ $2.50   (actual, before the fix)
```

The issue also quotes the code comment that stood at `applyGPUsAllocated` at the time — that request would always equal allocation because using a GPU requires requesting the whole GPU, so `max(usage, request)` always equals request. **That comment is the entire lesson in one sentence, written by the maintainers.** It is also no longer in the tree: at commit `30608b8` it has been replaced by the "apply `GPUUsageAverage` separately" comment quoted above, which is the fix for #3900 landing. Cite the issue for the argument and the current comment for the state.

#### G5 — Node discounts apply to CPU and RAM, not GPU

In `applyNodeDiscount` (`pkg/costmodel/allocation_helpers.go`, ~line 1910), the configured discount and negotiated discount are combined per node and then applied:

```go
node.Discount        = cm.Provider.CombinedDiscountForNode(node.NodeType, node.Preemptible, discount, negotiatedDiscount)
node.CostPerCPUHr   *= (1.0 - node.Discount)
node.CostPerRAMGiBHr*= (1.0 - node.Discount)
// there is no CostPerGPUHr line
```

**Mechanism and consequence.** On a GPU node the GPU is the overwhelming majority of the instance's price — for a p5-class node, the eight H100s dominate the CPU and RAM components by more than an order of magnitude. Applying an enterprise-agreement or negotiated discount to CPU and RAM only means the discount barely moves the node's total, and a fleet configured with, say, a 30% negotiated discount will see its GPU cost reported at **full list**. Whether this is intentional (many GPU rate cards genuinely discount differently — lesson 06 showed GCP's accelerator CUDs in the single digits to low teens while memory-optimised families reach ~70%) or an omission, the operational consequence is the same: **if you configure a blanket discount, check whether your GPU line moved.** This is an easy, high-signal thing to verify on a live install, and a good question to ask in an interview about someone's existing setup.

#### G6 — The headline efficiency metric excludes GPU entirely

```go
// core/pkg/opencost/allocation.go
func (a *Allocation) TotalEfficiency() float64 {
    if a.RAMTotalCost()+a.CPUTotalCost() > 0 {
        ramCostEff := a.RAMEfficiency() * a.RAMTotalCost()
        cpuCostEff := a.CPUEfficiency() * a.CPUTotalCost()
        return (ramCostEff + cpuCostEff) / (a.CPUTotalCost() + a.RAMTotalCost())
    }
    return 0.0
}
```

`TotalEfficiency` is a **cost-weighted** blend of CPU and RAM efficiency — and GPU appears in neither the numerator nor the denominator. On a node where the GPU is ~95% of the cost, the composite "efficiency" figure is computed entirely from the ~5% that is not the GPU.

**Mechanism.** The function pre-dates GPU support and its guard (`if RAM+CPU cost > 0`) means it never returns zero on a GPU node — it returns a confident-looking number derived from the wrong resource. A workload holding 8 idle H100s while comfortably using its CPU request will report high total efficiency. `GPUEfficiency()` exists and is correct; it simply is not composed in. **If you take one operational rule from this lesson: on a GPU fleet, never rank workloads by a composite efficiency score without checking which resources it composes.**

### 4. What changed in 2026: `pkg/inferencecost`

Here is where a stale write-up becomes a liability. OpenCost at `develop` ships a module that joins infrastructure cost, GPU utilisation, and **application-level token counters** — the three-way join this module's thesis says nobody performs. Be precise about what it is and is not.

**What it is.** `pkg/inferencecost/`, feature-gated behind `INFERENCE_COST_ENABLED` (**default `false`**, `pkg/env/costmodel.go`). When enabled it collects vLLM Prometheus metrics, pulls infrastructure cost from OpenCost's own allocation layer, and exports per-model cost metrics.

| Environment variable | Default | Purpose |
|---|---|---|
| `INFERENCE_COST_ENABLED` | `false` | master switch |
| `INFERENCE_MODEL_LABEL` | `llm-d.ai/model` | pod label whose value must match vLLM's `model_name` metric label |
| `INFERENCE_SHARED_INFRA_LABEL` | `llm-d.ai/inference-shared` | marks EPP/gateway pods as shared infra |
| `INFERENCE_SHARED_INFRA_LABEL_VALUE` | `true` | the value that marks it |
| `INFERENCE_COLLECTION_INTERVAL` | `2m` | background collection cadence |

**The vLLM metrics it requires** (`modules/prometheus-source/pkg/prom/inference_queries.go`, all read as counter deltas over the window, all needing `model_name` and `namespace` labels):

| Metric | Used for |
|---|---|
| `vllm:prompt_tokens_total` | input token count, blended rate denominator |
| `vllm:generation_tokens_total` | output token count |
| `vllm:request_prefill_time_seconds_sum` | prefill (input) time, for the cost split |
| `vllm:request_time_per_output_token_seconds_sum` | decode (output) time, for the cost split |
| `vllm:prefix_cache_hits_total` | `cacheSavingsFraction` — token-level, not block-level |
| `vllm:cache_config_info` | detects `enable_prefix_caching=false` only; not used in arithmetic |

**What it emits** (`pkg/inferencecost/exporter.go`): three Prometheus gauges — `llm_total_hourly_cost`, `llm_cost_per_million_tokens`, `llm_cache_savings_fraction` — plus two REST endpoints, `/inferenceCost/total` and `/inferenceCost/timeseries`. Labels carry `model_name`, `model_version`, `namespace`, `cost_basis`, `workload_type` and, on the per-token metric, `phase` ∈ {*(empty, blended)*, `prompt`, `generation`} and `allocation_method`.

**The methodology, which is genuinely good work.** Two cost bases are collected by calling `ComputeAllocation` twice: `allocation` uses `ShareIdle: ShareWeighted, ShareSplit: ShareWeighted` and reconciles to the infrastructure bill; `usage` uses `ShareNone` for both and then — this is the #3900 fix — scales each component by measured utilisation:

```go
// pkg/inferencecost/collector.go — the usage basis, GPU term
if alloc.GPUAllocation != nil && alloc.GPUAllocation.GPUUsageAverage != nil {
    util := *alloc.GPUAllocation.GPUUsageAverage
    if util < 0 { util = 0 } else if util > 1 { util = 1 }   // clamped
    scaledGPUCost := alloc.GPUTotalCost() * util
    cost = cost - alloc.GPUTotalCost() + scaledGPUCost
}
```

CPU is scaled by `CPUCoreUsageAverage / CPUCoreRequestAverage` and RAM likewise; network, PV, load-balancer and external costs are deliberately left unscaled, with the source naming the reason for each (network is already billed by bytes; PVs are billed by provisioned capacity with no IO utilisation metric; LBs are billed per hour of existence; external is an opaque pass-through). The input/output split is by **compute time**: `InputCost = TotalCost × PrefillTime / (PrefillTime + DecodeTime)`, with a fallback to a fixed **2.5× output/input multiplier** when the vLLM timing metrics are unavailable, and the path taken is recorded in the `allocation_method` label as `compute_time`, `prefix_caching_off`, or `multiplier`.

The project's own documented worked example (`docs/inference-cost-tracking.md`) — a Qwen3-32B model over one hour at $3.20/hr allocation cost, 12M prompt tokens, 3M generation tokens, prefill and decode each 50% of compute time — produces a blended `$0.213/M tokens`, an input rate of `$0.133/M prompt tokens`, an output rate of `$0.533/M output tokens`, and a `cacheSavingsFraction` of 6.7% from 800,000 cached prompt tokens.

**So: does that close the gap?** Four things it does not fix, and this is the answer to give:

1. **The numerator is unchanged.** `GPUHours` is still `request × hours` from G1/G2. The usage basis multiplies a *whole-GPU allocation* by a utilisation fraction; it does not make the allocation fractional. On a MIG card the 7× over-count is still 7×, now scaled by a duty cycle.
2. **`GPUUsageAverage` under sharing is the device aggregate.** `DCGM_FI_PROF_GR_ENGINE_ACTIVE` is a device-scoped counter (lesson 01 §5). Where several pods time-slice one device, every sharer gets the same value, so the usage basis scales all of them by an identical, non-discriminating number. It cannot even rank them.
3. **It is inference-only, and vLLM-only.** The `workload_type` label is documented as always `inference` today. Training, fine-tuning, and any non-vLLM serving stack are outside it. The join runs on a pod label matching vLLM's `model_name`; a mismatch silently produces zero cost with an empty `allocation_method` — the docs' own troubleshooting section is about exactly that failure.
4. **The `usage` basis does not reconcile to the bill, by design.** The source and the metric help text both say so: `cost_basis=usage` excludes idle and shared infrastructure. It is an efficiency lens, not a chargeback basis — which is precisely the lesson-02/08 distinction, now shipped as a label rather than left to the reader.

**The honest 2026 framing, and the one to use in an interview:** *"OpenCost's allocation ledger is still request-based whole-GPU allocation, blind to MIG and to the shared-resource key. But as of 2026 it does ship a feature-gated inference-cost module that joins allocation cost to vLLM token counters and emits a real `llm_cost_per_million_tokens` under two named cost bases. So the three-way join is no longer 'nobody does it' — it is 'one tool does it, off by default, for vLLM inference only, on top of a numerator that is still whole-GPU.' The remaining gap is the numerator and the sharing regimes, not the join."* That sentence is worth more than the older, simpler claim, because it is current and it survives someone in the room having read the repo last month.

### 5. The capability matrix

The matrix is the deliverable's central exhibit, so it has to be specific enough that every cell is a claim someone could check. Three capability columns, because those are the three legs of the join:

- **A — attributed $:** a dollar figure keyed to a tenant.
- **U — real utilisation in the cost basis:** measured device work multiplied into a cost, not shown beside one.
- **C — application counters:** tokens, runs, experiments — the denominator of a unit cost.

And two GPU-specific columns that decide whether A is even meaningful: **frac** (does the numerator represent a MIG fraction?) and **share** (does it correctly split a time-sliced device?).

```
  ═══════════════════════════════════════════════════════════════════════════
   TOOL                       │ A   U   C │frac│share│ WHERE IT STOPS
  ────────────────────────────┼───────────┼────┼─────┼───────────────────────
   OpenCost — allocation      │ ●   ○   ○ │ ✗  │  ✗  │ numerator is a whole-
     (pkg/costmodel)          │           │    │     │ GPU request count;
                              │           │    │     │ DCGM lands in a
                              │           │    │     │ display-only field
  ────────────────────────────┼───────────┼────┼─────┼───────────────────────
   OpenCost — inferencecost   │ ●   ◐   ● │ ✗  │  ✗  │ joins all three, but
     (INFERENCE_COST_ENABLED, │           │    │     │ vLLM inference only,
      default OFF)            │           │    │     │ off by default, and
                              │           │    │     │ scales an unchanged
                              │           │    │     │ whole-GPU numerator
  ────────────────────────────┼───────────┼────┼─────┼───────────────────────
   Kubecost (commercial)      │ ●   ◐   ○ │ ?  │  ?  │ same OpenCost lineage;
                              │           │    │     │ GPU views added 2.x are
                              │           │    │     │ visibility. NOT source-
                              │           │    │     │ verifiable — closed.
  ────────────────────────────┼───────────┼────┼─────┼───────────────────────
   DCGM-exporter + Grafana    │ ○   ●   ○ │ ●  │  ✗  │ ground-truth per-UUID
                              │           │    │     │ (and per-MIG-instance)
                              │           │    │     │ counters — and no
                              │           │    │     │ dollars, no tenant map
  ────────────────────────────┼───────────┼────┼─────┼───────────────────────
   Cloud billing / FOCUS      │ ●   ○   ○ │ ✗  │  ✗  │ granularity stops at
     export (CUR, Cost Expl.) │           │    │     │ the instance. Eight
                              │           │    │     │ H100s are ONE line.
  ────────────────────────────┼───────────┼────┼─────┼───────────────────────
   Kueue / Run:ai / KAI       │ ○   ○   ○ │ ●  │  ●  │ knows exactly who holds
     (schedulers)             │           │    │     │ what — in QUOTA UNITS,
                              │           │    │     │ never in dollars
  ────────────────────────────┼───────────┼────┼─────┼───────────────────────
   HAMi / vGPU layers         │ ○   ◐   ○ │ ●  │  ◐  │ enforces a fraction it
                              │           │    │     │ never prices; see
                              │           │    │     │ OpenCost issue #3828
  ────────────────────────────┼───────────┼────┼─────┼───────────────────────
   THIS MODULE'S DELIVERABLE  │ ●   ●   ● │ ●  │  ◐  │ ◐ on share is HONEST:
                              │           │    │     │ regime-3 is unsolvable
                              │           │    │     │ (L01), so it emits an
                              │           │    │     │ approximation + a flag
  ═══════════════════════════════════════════════════════════════════════════
   ● yes   ◐ partial   ○ no   ✗ structurally cannot   ? not verifiable
```

The same content as a table you can lift into a writeup, with the mechanism spelled out:

| Tool | GPU cost numerator | What it does with DCGM | App counters | The mechanism of the gap |
|---|---|---|---|---|
| **OpenCost, allocation path** | `nvidia.com/gpu` request count × hours (`applyGPUsAllocated`) | queries `DCGM_FI_PROF_GR_ENGINE_ACTIVE`, stores in `GPUAllocation.GPUUsageAverage`, consumes only in `GPUEfficiency()` | none | `GPUCost = GPUHours × CostPerGPUHr` has no term that can carry a fraction or a per-tenant share |
| **OpenCost, `pkg/inferencecost`** | same, then scaled by clamped `GPUUsageAverage` on the `usage` basis | multiplied into cost — genuinely | **yes**: `vllm:prompt_tokens_total`, `vllm:generation_tokens_total`, prefill/decode times, prefix-cache hits | off by default; vLLM inference only; scales an unchanged whole-GPU numerator; the device-aggregate counter cannot separate sharers |
| **Kubecost** | OpenCost lineage | GPU utilisation views exist in 2.x | none documented | closed source from 2.x — **say "not source-verifiable" rather than asserting**; the OSS engine's findings are the defensible claim |
| **DCGM-exporter + Grafana** | n/a — no cost model | *is* the ground truth, per GPU UUID and per MIG `GPU_I_ID` | none | it is an exporter. No price, no tenant map, no ledger. Correct and insufficient |
| **Cloud billing / FOCUS export** | instance-hours | nothing | none | the invoice's atom is an instance. Eight GPUs are one line, and no provider-side split exists for in-cluster tenants |
| **Kueue / Run:ai / KAI** | quota units | nothing | none | schedulers hold the *best* allocation record in the stack (`nominalQuota`, `borrowingLimit`, decayed usage) and never multiply it by a price |
| **HAMi and vGPU layers** | enforces `gpucores`/`gpumem` fractions | per-container limits | none | the fraction exists in the enforcement layer and never reaches the cost layer — issue #3828 is a user asking for exactly that bridge |

**Two conclusions to state out loud.** First, the empty three-way join is now *nearly* filled by one feature-gated module, for one workload class — so the honest thesis is about the **numerator and the sharing regimes**, not about the join. Second, look down the `frac`/`share` columns: the components that *know* the fraction (device plugin, DCGM, HAMi, the scheduler) all sit below the cost layer, and the components that know the dollars all sit above it. **That vertical split is the structural reason the gap persists across the whole industry rather than in one project** — billing teams default to the orchestrator's native unit, observability teams default to the driver's counters, and ML platform teams default to the framework's counters. Nobody who owns one leg naturally owns the other two.

### 6. The gap paragraph, and the fix spec

The one-paragraph statement to carry into the deliverable and the interview:

> **OpenCost bills GPUs on the allocated ledger.** The numerator is a Kubernetes resource-request count — `container_gpu_allocation`, emitted in `pkg/costmodel/metrics.go` from `costs.GPUReq[0].Value`, read back by `QueryGPUsAllocated`/`QueryGPUsRequested` (the latter filtered to `resource="nvidia_com_gpu"`), and turned into hours by `applyGPUsAllocated`: `GPUHours = value × hrs`. The price is one scalar dollars-per-**physical**-GPU-hour from the provider hook `GpuPricing()` or a configured default. The cost is one multiply in `applyNodesToPod`: `alloc.GPUCost = alloc.GPUHours × node.CostPerGPUHr`. DCGM utilisation *is* queried (`DCGM_FI_PROF_GR_ENGINE_ACTIVE`, correctly not `DCGM_FI_DEV_GPU_UTIL`) and stored in `GPUAllocation.GPUUsageAverage`, but in the allocation path it is read only by the display ratio `GPUEfficiency()`. **Because MIG instances and time-slice replicas both surface as whole-device Kubernetes requests — and because the shared resource key `nvidia.com/gpu.shared` is absent from both numerator paths — sub-device attribution is structurally invisible: not a configuration gap, an input-model gap.** As of 2026 the feature-gated `pkg/inferencecost` module (`INFERENCE_COST_ENABLED`, default off) does multiply `GPUUsageAverage` into cost and joins vLLM token counters to emit `llm_cost_per_million_tokens` — a real three-way join, for vLLM inference only, on top of the same unchanged whole-GPU numerator.

**What it would take to close it**, with tractability marked:

| # | Fix | What it changes | Tractable? |
|---|---|---|---|
| 1 | Add `nvidia.com/gpu.shared` to both numerator paths (the request chain in `costmodel.go` and the `resource=` filter in `QueryGPUsRequested`) | ends the $0.00 case (G2) | **yes — a few lines.** The lowest-hanging fix in the tool |
| 2 | Repair the guard on the vGPU scaling branch (G3) so `request × (physical/virtual)` actually runs | replicas divide down to a physical fraction; ends the N× case | **yes — one condition** |
| 3 | Make the numerator fractional: accept a per-pod fraction from a MIG-profile lookup, a HAMi `gpucores` read, or a DRA `ResourceClaim` allocation result | MIG and vGPU price at their real basis | **yes**, and it is what issue #3828 asks for. Needs a decision on the fractional basis (SM-slice vs memory vs blended — lesson 01) that FOCUS deliberately does not prescribe |
| 4 | Price per measurement identity: `CostPerGPUHr` per MIG UUID, not per physical device | removes the whole-device scalar assumption | **yes**, mostly plumbing once (3) exists |
| 5 | Extend the usage basis beyond vLLM inference to training/other counters | unit cost per run, per experiment | **yes**, and it is the generalisation of a module that already exists |
| 6 | A defensible per-tenant utilisation split under time-slicing | a true utilised ledger for regime 3 | **NO — blocked by physics.** Lesson 01's argument: the device exposes one counter, not one per CUDA context. The correct engineering answer is an approximation plus an explicit uncertainty flag, which is exactly what lesson 10's schema carries and what no tool models today |

That last row is the durable part of the answer. Five of six gaps are ordinary engineering; the sixth is a property of the hardware, and knowing which is which is the whole differentiator.

## Perspectives

**Source-reading view.** The transferable skill is not "OpenCost bills GPUs wrong," it is the three-question procedure: find the numerator, find the price, find where the interesting telemetry lands. On this codebase that was four greps — `nvidia.com/gpu`, `GPUCost =`, `DCGM_`, `GPUUsageAverage` — and reading two functions. Notice which findings that procedure produced that no changelog or blog post would: an unreachable branch (G3), a string mismatch between the emitter and the query (G2), and a composite metric that omits the dominant resource (G6). **Reading the guard rather than the comment is what produced all three.**

**FinOps-practitioner view.** Every dashboard number is a claim with a basis, and the basis is usually invisible. "Team A spent $4,200 on GPUs" means "team A requested nominal whole-GPU capacity priced at $4,200," which is a defensible claim if you say it that way and an indefensible one if you say it the short way and a tenant reads it as consumption. Lesson 08's dispute protocol works only if you know which claim you are making before finance repeats it. Reading the source is how you find out.

**Maintainer's view — and be fair here, because fairness is more credible than criticism.** Nothing in §3 is incompetence. OpenCost's model was designed when GPUs were exclusively held, and every gap is that one correct assumption meeting hardware that changed underneath it. The project queries the *right* DCGM field, deliberately separates ledgers, documents its cost bases honestly in metric help text, tracks these exact issues publicly, and shipped a fix to #3900 within four days of it being filed. **The right framing for an interview is "here is a design that was correct for its era and here is precisely where the hardware moved," not "here is a bad tool."** The second framing tells a panel you cannot distinguish a defect from a scope boundary.

**Tooling-landscape view.** Look at the `frac`/`share` columns in §5 again. The layers that know the fraction — device plugin, DCGM, HAMi, the scheduler — are all *below* the cost layer; the layers that know the dollars — billing exports, cost engines — are all *above* it. The gap is at the seam, and seams belong to nobody. That is why it persists industry-wide rather than in one project, and it is why the deliverable's value is the *join*, not any single component of it.

**Interview view.** In a room where every candidate says "OpenCost isn't great with GPUs," the differentiator is a location and a number: a file, a function, a struct field, a commit hash, and "seven pods on a MIG-partitioned H100 bill $14.70 for a card that costs $2.10." Adjectives are indistinguishable between candidates; a citation is not. The habit generalises: always answer "why not tool X" with a location in tool X's source.

## Real-world use cases

- **OpenCost issue #3900 — "GPU 'usage' cost basis does not reflect actual GPU utilization"** (filed 2026-07-05, closed 2026-07-09). The reporter shows that `/inferenceCost/total?costBasis=usage` and `?costBasis=allocation` return figures within ~1% of each other even for a vLLM server confirmed at ~11% GPU utilisation, and traces it precisely: both bases call `ComputeAllocation`, differing only in `ShareIdle`/`ShareSplit`, while `GPUHours` is set from the *request* in `applyGPUsAllocated` regardless. Their expected/actual is `$0.275` versus `$2.50` on a 1-GPU, 1-hour, $2.50/hr workload. **What it shows:** this lesson's central argument, stated in the project's own tracker with the same file and function citations, and fixed four days later by the `GPUUsageAverage` scaling in `pkg/inferencecost/collector.go`. It is the strongest possible corroboration — you are not offering an outside opinion, you are describing something the maintainers agreed with and shipped against.

- **OpenCost issue #3828 — "Support HAMi GPU core and memory allocation in cost calculations"** (filed 2026-06-02, **open**). A HAMi user reports that a pod requesting `nvidia.com/gpu: 1` plus `nvidia.com/gpucores: 25` and `nvidia.com/gpumem-percentage: 25` — physically allocated a quarter of a card — is costed as a whole GPU, because OpenCost reads only `nvidia.com/gpu`. Their documented workaround: query the allocation API, look up the intended fraction from HAMi metadata, multiply `gpuCost` by it externally. **What it shows:** the fractional-GPU gap is live, acknowledged, unfixed, and the community's workaround is structurally identical to this module's deliverable. It also anchors the history — #934 (fractional `nvidia.com/gpu` values), #2862 (`nvidia.com/gpu.replicas` costing), #3039 (vGPU core/memory usage) — so you can speak about the trajectory rather than a snapshot.

- **OpenCost issue #3160 — "Custom Pricing ignored on GPU nodes"** (filed 2025-05-27, **open**, reported against 1.114.0 and 1.115.0). An on-prem operator sets every custom price to `1` and finds that on GPU-equipped nodes `node_cpu_hourly_cost`, `node_ram_hourly_cost` and `node_gpu_hourly_cost` all come back as small fractions (0.000828, 0.010364, …) while `node_total_hourly_cost` lands near `1` — i.e. the total node cost was treated as 1 and split across components by ratio, instead of each component being priced at 1. CPU-only nodes in the same cluster report exactly `1`. **What it shows:** the on-prem and neocloud case — precisely where a GPU platform engineer lives, because there is no cloud rate card to fall back on — has a live, multi-version pricing defect on GPU nodes. Combined with `GpuPricing()`'s fallback-to-configured-default path in `costmodel.go`, it means the price factor is as worth auditing as the quantity factor. Do not assume the dollars are right just because you configured them.

- **OpenCost's own inference-cost documentation** (`docs/inference-cost-tracking.md`, in-repo at commit `30608b8`). Names its two cost bases with unusual honesty in the metric help text itself: `cost_basis=allocation` "reconciles to the infrastructure bill (includes idle and shared infra costs)", `cost_basis=usage` "reflects active compute only… does NOT reconcile to the bill." **What it shows:** the allocated/utilised distinction from lesson 02 has been adopted into a shipping tool as a first-class metric label with the reconciliation property spelled out. That is the direction of travel, and it is a good sign for the schema in lesson 10 — the industry is converging on modelling both ledgers explicitly rather than picking one.

## Worked example

One H100 80GB node. Provider price **$2.10 per physical GPU-hour** (Silver-tier, 1-year reserved, August 2026 snapshot — module 03 capstone anchor). One hour. Four tenancy configurations, same hardware, same tool.

*These figures are derived by tracing the source paths in §2 and §3, not captured from a running install. Reproduce before quoting.*

**(a) Exclusive — one pod, `nvidia.com/gpu: 1`.**

```
  QueryGPUsAllocated → container_gpu_allocation = 1
  applyGPUsAllocated → GPUHours = 1 × 1.0 h = 1.0
  applyNodesToPod    → GPUCost  = 1.0 × $2.10 = $2.10
  TRUTH              → $2.10.   ERROR: none. OpenCost is CORRECT here.
```

This row matters as much as the broken ones. The tool is right in regime 1, which is why it is trusted, which is why the other three rows surprise people.

**(b) MIG — 7 × `1g.10gb`, seven tenants, each requesting `nvidia.com/gpu: 1`.**

```
  per pod:  GPUHours = 1 × 1.0 = 1.0  →  GPUCost = 1.0 × $2.10 = $2.10
  seven pods                          →  $14.70 attributed
  the physical card actually costs    →  $ 2.10
  ──────────────────────────────────────────────────────────────────────
  ERROR: 7× per tenant AND 7× in aggregate.  $12.60/hr of phantom cost
         on ONE card. On a 64-GPU fleet MIG-partitioned 7× throughout:
         64 × $12.60 × 730 h = $588,672/month of cost that does not exist.

  TRUTH on the SM-slice basis (lesson 01 §4):
         each 1g.10gb = 1/7 of the card → $2.10/7 = $0.30/tenant/hour
         seven tenants sum to exactly $2.10. ✓ conservation holds.

  WHY NO SETTING FIXES IT: `grep -ri '\bmig\b' --include='*.go' .` at
  commit 30608b8 returns nothing. There is no code path to configure.
```

**(c) Time-sliced, `replicas: 4`, GFD `renameByDefault=true` — four tenants requesting `nvidia.com/gpu.shared: 1`.**

```
  costmodel.go:388 request chain   → key not matched  → gpuReqCount = 0
  QueryGPUsRequested filter        → resource="nvidia_com_gpu" excludes
                                      nvidia_com_gpu_shared → no series
  container_gpu_allocation         → 0
  ──────────────────────────────────────────────────────────────────────
  per pod:  GPUHours = 0  →  GPUCost = $0.00
  four pods              →  $0.00 attributed
  the card costs         →  $2.10, ALL of which lands in idle.

  ERROR: total. Four tenants saturating an H100 are billed nothing, and
  the platform absorbs 100% of a fully-used card. On the lesson-08
  statement this appears as a large unexplained remainder — which is
  exactly why lesson 08 insists the remainder be itemised by cause.

  Meanwhile the pods DO get an IsGPUShared = true flag (applyGPUUsage-
  Shared reads the nvidia_com_gpu_shared series) and DO get a
  GPUUsageAverage. The tool knows they are sharing a GPU, knows how
  busy that GPU is, and bills zero.
```

**(d) Same, GFD `renameByDefault=false` — replicas advertised as `nvidia.com/gpu`.**

```
  node Status.Capacity["nvidia.com/gpu"] = 4   (4 replicas, 1 card)
  node label nvidia.com/gpu.count        = 1
  getGPUCount() → (resultGPU=1, resultVGPU=4)

  per pod:  request 1 → GPUHours = 1.0 → GPUCost = 1.0 × $2.10 = $2.10
  four pods                                        →  $8.40 attributed
  the card costs                                   →  $2.10
  ERROR: 4× (in general, N× for N replicas).

  The intended correction — gpualloc × (gpu/vgpu) = ×(1/4) — exists in
  metrics.go directly below the emit and is unreachable (§3 G3):
      if verr != nil && err != nil && vgpu != 0
    · both parse OK (normal) → verr == nil            → skipped
    · both fail on syntax    → ParseFloat returns 0   → vgpu == 0 → skipped
    · both fail on range     → ±Inf → gpu/vgpu = NaN
  So the request-weighted split lesson 01 calls the honest approximation
  is written into this codebase and does not execute.

  TRUTH, request-weighted (lesson 01's regime-3 approximation):
      $2.10 / 4 = $0.525 per tenant, summing to $2.10.
  TRUTH, if the real context-runtime split were 70/25/2.5/2.5:
      $1.47 / $0.525 / $0.0525 / $0.0525 — and lesson 01's argument says
      you CANNOT recover this from device counters. Even a repaired
      OpenCost would be off by up to ~10× on the small tenants, which is
      not a bug: it is the regime-3 limit, and the correct engineering
      response is an uncertainty flag (lesson 10), not more precision.
```

**(e) The inference-cost path, for contrast.** Same card, one vLLM pod serving Qwen3-32B, `INFERENCE_COST_ENABLED=true`, measured `GPUUsageAverage = 0.62`, one hour, 12M prompt + 3M generation tokens, prefill and decode each 50% of compute time:

```
  allocation basis: GPUCost = 1.0 × $2.10                     = $2.10
                    + shared-infra share (EPP/gateway, if any) = …
  usage basis:      GPUCost = $2.10 × clamp(0.62)             = $1.302
                    (idle and shared infra EXCLUDED — does not
                     reconcile to the bill, and the metric says so)

  llm_cost_per_million_tokens{cost_basis="allocation", phase=""}
        = $2.10 / 15 M tokens × 1e6                  = $0.140 /M tokens
  phase="prompt",  allocation_method="compute_time"
        = ($2.10 × 0.5) / 12 M × 1e6                 = $0.0875/M prompt tok
  phase="generation"
        = ($2.10 × 0.5) /  3 M × 1e6                 = $0.350 /M output tok
  llm_cache_savings_fraction  = cached/prompt tokens (vllm:prefix_cache_hits_total)

  ▸ This is a genuine $ × utilisation × app-counter join, in the tool.
  ▸ And it sits on top of case (a). Run the same vLLM pod on a MIG
    instance and the numerator is still case (b)'s — every one of these
    per-token figures inherits the 7×.
```

**The summary line for the writeup:** on the same H100 at the same price, the same tool reports **$2.10 (correct), $14.70 (7× high), $0.00 (infinitely low), or $8.40 (4× high)** depending only on how the device is shared and which flag the GPU operator was installed with. The spread is not noise; it is four different code paths, each traceable to a named line.

## Practice

Feeds [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md). All four items go into `writeup/tooling-gaps.md`.

1. **Clone and re-verify, then cite a commit.** `git clone --depth 1 https://github.com/opencost/opencost`. Confirm each of G1–G6 against the tree yourself and record the commit hash you read. Specifically: run `grep -ri '\bmig\b' --include='*.go' .` and record the result; read the condition in `metrics.go` that G3 describes and decide for yourself whether it is reachable; check whether `applyNodeDiscount` has grown a GPU line. **If a gap has been fixed since this lesson was written, say so and cite both commits — a writeup that tracks a moving codebase is worth more than one that froze.**
2. **The sourced gap paragraph, in your own words.** Reproduce §6's box with your own commit hash, citing at minimum: the emitter (`pkg/costmodel/metrics.go`, `costs.GPUReq[0].Value`), the PromQL filter (`QueryGPUsRequested`, `resource="nvidia_com_gpu"`), the numerator (`applyGPUsAllocated`), the multiply (`applyNodesToPod`), and the landing field (`GPUAllocation.GPUUsageAverage`). This is the interview paragraph.
3. **Reproduce the four-way error table.** Take the §Worked-example structure with your own fleet's price and MIG geometry and produce cases (a)–(d) with the error multiplier for each. If you have access to a GPU, actually run case (c) — set `renameByDefault=true`, deploy two sharing pods, and check whether OpenCost's `/allocation/compute` reports a non-zero `gpuCost`. A captured transcript beats a derivation, and it is a five-minute experiment.
4. **The capability matrix, for the stack in front of you.** Fill §5's table for the exact tools your target company runs. Circle the cells that are `✗` rather than `○` — structurally impossible versus merely absent is the distinction that shows you understand the mechanism. Then write the fix spec (§6) with your own tractability calls, and mark row 6 as blocked by the lesson-01 argument.

**Acceptance criteria:** every claim carries a file and function · a commit hash is named · at least one gap re-verified rather than repeated from this lesson · the inference-cost module is covered accurately, including that it is off by default and vLLM-only · the time-slicing limit is attributed to hardware, not to the tool.

## Common pitfalls

- **Treating a Kubecost/OpenCost GPU number as chargeback ground truth.** Symptom: a MIG fleet's cost report totals several times the invoice. Mechanism: the numerator is a whole-GPU request count and MIG instances present as whole requests, so the sum over pods exceeds the physical cost by the partition factor. Correction: reconcile the tool's attributed GPU cost against the actual invoice before any of it becomes a chargeback line — the identity from module 04's capstone catches this in one query.
- **Assuming the error is always upward.** Symptom: nobody investigates because "we know it over-counts." Mechanism: G2 — with `renameByDefault=true` the shared resource key is absent from both numerator paths, so time-sliced pods bill `$0.00` and the cost silently becomes platform idle. Correction: check your GPU-operator configuration for `renameByDefault` before predicting the direction of the error.
- **Believing a config flag or version bump fixes MIG.** Symptom: an upgrade ticket that closes nothing. Mechanism: MIG does not appear anywhere in the source; the cost expression has no term that could carry a fraction. Correction: it is an input-model change (fix #3 in §6), not a setting.
- **Reading the comment instead of the guard.** Symptom: reporting that OpenCost divides vGPU requests down to the physical fraction, because a comment says it does. Mechanism: G3's condition requires both `ParseFloat` calls to have failed *and* the result to be non-zero, which `strconv`'s syntax-error return value makes contradictory. Correction: trace the condition, not the intent, and quote the condition in the writeup.
- **Assuming the DCGM percentage shown next to a cost informed that cost.** Symptom: a stakeholder concludes low utilisation is "already priced in." Mechanism: `GPUUsageAverage` is a pointer field on a nested struct read only by `GPUEfficiency()`; the billed `GPUCost` float is a separate multiply. Correction: the type-level proof in G4 settles it without needing a behavioural test.
- **Ranking workloads by `TotalEfficiency` on a GPU fleet.** Symptom: a pod holding eight idle H100s ranks as highly efficient. Mechanism: G6 — `TotalEfficiency` is a cost-weighted blend of CPU and RAM only, and its guard means it returns a plausible number rather than zero. Correction: rank on `GPUEfficiency()` explicitly, or compute your own composite that includes GPU cost in the denominator.
- **Repeating "no tool joins $ × utilisation × app counters" in 2026.** Symptom: someone in the room says "OpenCost ships `llm_cost_per_million_tokens`" and the rest of your argument loses credibility. Mechanism: `pkg/inferencecost` landed and does perform that join. Correction: make the sharper claim — the join exists, feature-gated, for vLLM inference, on top of an unchanged whole-GPU numerator; the remaining gap is the numerator and the sharing regimes.
- **Framing the teardown as "the tool is bad."** Symptom: a panel hears criticism instead of analysis. Mechanism: five of six gaps are one correct design assumption (exclusive whole-GPU tenancy) meeting hardware that changed; the sixth is physics. Correction: separate defects from scope boundaries explicitly, and give the project credit for querying the right DCGM field and for fixing #3900 in four days.

## Self-check

- **What is the numerator of GPU cost in OpenCost, and why does that make MIG fractions invisible?** *Answer:* The pod's whole-device GPU **request** count. It is emitted as `container_gpu_allocation` from `costs.GPUReq[0].Value` in `pkg/costmodel/metrics.go`, read back by `QueryGPUsAllocated` (or `QueryGPUsRequested`, filtered to `resource="nvidia_com_gpu"`), turned into hours in `applyGPUsAllocated` as `GPUHours = value × hrs`, and multiplied in `applyNodesToPod` as `alloc.GPUCost = alloc.GPUHours × node.CostPerGPUHr`, where the price is one scalar per *physical* GPU. MIG instances are advertised by the device plugin as whole `nvidia.com/gpu` units, so a `1g.10gb` instance reads as request count 1 and bills the whole card: seven tenants on a 7-way partition bill 7× the card. The mechanism is that the cost expression has exactly two factors — an integer count and a per-device price — and a fraction is neither; there is no term to put it in. Confirmed negatively: `grep -ri '\bmig\b'` over the tree at commit `30608b8` returns nothing outside "migration."

- **A colleague says OpenCost over-bills shared GPUs. Under what configuration does it under-bill them to zero, and why?** *Answer:* When NVIDIA GPU Feature Discovery runs with `renameByDefault=true`, time-sliced replicas are advertised as `nvidia.com/gpu.shared` and pods request that key. Both of OpenCost's numerator paths miss it: the request chain in `costmodel.go` (~line 388) checks only `nvidia.com/gpu` and `k8s.amazonaws.com/vgpu`, and `QueryGPUsRequested`'s PromQL filters `resource="nvidia_com_gpu"`, which excludes kube-state-metrics' `nvidia_com_gpu_shared`. So `GPUHours = 0` and `GPUCost = $0.00`, and the whole card's cost lands in idle. With `renameByDefault=false` the replicas surface as `nvidia.com/gpu`, each sharer requests 1, and the error flips to N× over-billing. The key is not unknown to the codebase — `ResourceNvidiaShared` is defined, `getGPUCount()` reads the shared capacity, and `applyGPUUsageShared()` sets `IsGPUShared` from it — it is recognised for counting and flagging and not for billing.

- **OpenCost queries DCGM. So why is it still blind to utilisation in its allocation ledger, and what changed in 2026?** *Answer:* `QueryGPUsUsageAvg` runs `avg(avg_over_time(DCGM_FI_PROF_GR_ENGINE_ACTIVE{container!=""}[w]))` — the correct SM-engine-active signal, not the `DCGM_FI_DEV_GPU_UTIL` util-lie — and `applyGPUUsageAvg` stores it in `GPUAllocation.GPUUsageAverage`, a `*float64` on a nested struct. In the allocation path the only consumer is `GPUEfficiency() = GPUUsageAverage / GPURequestAverage`, a display ratio; the billed `GPUCost` float never sees it. Issue #3900 (2026-07-05 → 2026-07-09) states exactly this and shows the consequence: `costBasis=usage` and `costBasis=allocation` differing by <1% on a workload at ~11% utilisation, expected $0.275 versus actual $2.50. What changed: the fix added utilisation scaling in `pkg/inferencecost/collector.go`, which clamps `GPUUsageAverage` to [0,1] and computes `scaledGPUCost = GPUTotalCost() × util` for the usage basis — but only inside the feature-gated inference-cost module, and on top of an unchanged whole-GPU `GPUHours`.

- **Which tools join attributed dollars × real utilisation × application counters, and what does the answer imply for the deliverable?** *Answer:* Exactly one, partially: OpenCost's `pkg/inferencecost`, behind `INFERENCE_COST_ENABLED` (default `false`), which joins allocation cost to `vllm:prompt_tokens_total` / `vllm:generation_tokens_total` and emits `llm_cost_per_million_tokens` under `cost_basis` ∈ {`allocation`, `usage`} with an input/output split by vLLM prefill/decode time (fallback: a fixed 2.5× multiplier). It is vLLM inference only, `workload_type` is always `inference`, the `usage` basis explicitly does not reconcile to the bill, and the numerator is still whole-GPU. Everyone else holds one leg: DCGM-exporter has ground-truth per-UUID utilisation and no dollars or tenant map; cloud billing has real dollars at instance granularity; schedulers have the best allocation record in the stack and never multiply it by a price; HAMi enforces a fraction it never prices. So the deliverable's thesis in 2026 is not "nobody joins the three" — it is **"the join exists for one workload class on a numerator that cannot represent sub-device sharing,"** and the remaining work is the fractional numerator, the regime tagging, and an explicit uncertainty flag for regime 3.

- **Give one finding from this teardown that no changelog would have told you, and say how you found it.** *Answer:* Any of three. (i) The vGPU scaling branch in `pkg/costmodel/metrics.go` is unreachable: its guard is `verr != nil && err != nil && vgpu != 0`, and `strconv.ParseFloat` returns 0 on a syntax error, so "both parses failed" and "`vgpu` non-zero" are effectively contradictory — the only way in is a double range error, which makes `gpu/vgpu` = `NaN`. Found by reading the condition rather than the comment above it. (ii) The `nvidia.com/gpu.shared` key is present in the codebase for counting and flagging but absent from both numerator paths — found by grepping the key and noticing which call sites use it. (iii) `TotalEfficiency()` blends only CPU and RAM efficiency, cost-weighted, so on a node where the GPU is ~95% of cost the headline efficiency number is computed from the other ~5% — found by reading the function rather than trusting its name.

- **Why is the time-slicing gap the one item on the fix list you mark as blocked?** *Answer:* Because it is not a property of the tool. `DCGM_FI_PROF_GR_ENGINE_ACTIVE` is scoped to the *device*, not to the CUDA context that happened to be resident when the counter was sampled, so N pods time-slicing one GPU all read the same aggregate number (lesson 01 §5). No amount of code in a cost engine recovers N per-tenant values from one device value — the information was never produced. Fixes 1–5 in §6 are ordinary engineering (add a resource key, repair a guard, make the numerator fractional, price per measurement identity, generalise beyond vLLM); fix 6 is not, and the correct engineering response is to emit the best approximation together with an explicit uncertainty flag, which is precisely the column lesson 10's schema adds because FOCUS does not have one.

## Connections & what's next

This lesson converts lessons 01–08 from a design into a verified claim about production tooling. The four regimes (01) reappear as the four rows of the worked example, each with its own error mode. The two ledgers (02) reappear as the type-level split between `GPUCost` and `GPUAllocation.GPUUsageAverage` — and, newly, as the `cost_basis` label on `llm_total_hourly_cost`. Idle and fragmentation (03/04) reappear as the destination of the cost that case (c) fails to attribute. Unit economics (05/07) reappear as `llm_cost_per_million_tokens`, which the tool now computes and which inherits every numerator error above it. The chargeback design (08) is the specification this teardown measures against.

Next, **lesson 10 — FOCUS 1.x and designing a GPU cost schema** — takes each gap named here and turns it into a column. The whole-GPU numerator becomes a fractional `ConsumedQuantity` declared through FOCUS's split-cost-allocation columns; the regime becomes an explicit dimension; the utilised ledger and the regime-3 uncertainty become the two `x_` extensions the standard genuinely still lacks; and the reconciliation property that OpenCost's MIG case violates by 7× becomes a normative requirement of the spec. The sourced gap paragraph and the capability matrix from this lesson are the direct inputs.

## References & further reading

**Primary sources — OpenCost, read at `develop`, commit `30608b8` (2026-08-12), on 2026-08-17**

- **`pkg/costmodel/metrics.go`** — <https://github.com/opencost/opencost/blob/develop/pkg/costmodel/metrics.go> — the emitter. Source for `node_gpu_hourly_cost` (per physical GPU, from `node.GPUCost`) and `container_gpu_allocation` (`gpualloc := costs.GPUReq[0].Value`), for the project's own comment that allocation is set to the request because shared GPU usage is not yet supported, and for the unreachable vGPU-scaling guard analysed in §3 G3.
- **`pkg/costmodel/costmodel.go`** — <https://github.com/opencost/opencost/blob/develop/pkg/costmodel/costmodel.go> — the container-request read chain at ~line 388 (`nvidia.com/gpu` and `k8s.amazonaws.com/vgpu` only, no `nvidia.com/gpu.shared`), `getGPUCount()` and its three cases (standard NVIDIA, GPU-Feature-Discovery with `nvidia.com/gpu.replicas`, AWS vGPU), and the node-price resolution path including the `cp.GpuPricing(nodeLabels)` hook and the configured-default fallback.
- **`pkg/costmodel/allocation_helpers.go`** — <https://github.com/opencost/opencost/blob/develop/pkg/costmodel/allocation_helpers.go> — `applyGPUsAllocated` (~line 929: `GPUHours = res.Data[0].Value * hrs`, and the "apply `GPUUsageAverage` separately" comment that replaced the comment quoted in issue #3900), `applyGPUUsageShared`, `applyNodeCostPerGPUHr`, `applyNodeDiscount` (~1910 — CPU and RAM only, G5), and `applyNodesToPod` (~1929 — the single multiply that is the whole GPU cost model).
- **`pkg/costmodel/allocation.go`** — <https://github.com/opencost/opencost/blob/develop/pkg/costmodel/allocation.go> — lines ~294–303, where the seven GPU-related queries are fired concurrently: `QueryIsGPUShared`, `QueryGPUsAllocated`, `QueryGPUsRequested`, `QueryGPUsUsageAvg`, `QueryGPUsUsageMax`, `QueryGPUInfo`, `QueryNodeGPUPricePerHr`.
- **`modules/prometheus-source/pkg/prom/metricsquerier.go`** — <https://github.com/opencost/opencost/blob/develop/modules/prometheus-source/pkg/prom/metricsquerier.go> — the literal PromQL for every query reproduced in §2, including the `resource="nvidia_com_gpu"` filter on `QueryGPUsRequested` that produces gap G2, and the fact that `QueryGPUInfo` selects on `DCGM_FI_DEV_DEC_UTIL` purely to carry the `device`/`modelName`/`UUID` labels.
- **`core/pkg/opencost/allocation.go`** — <https://github.com/opencost/opencost/blob/develop/core/pkg/opencost/allocation.go> — the `Allocation` struct (`GPUHours`, `GPUCost`, `GPUCostAdjustment`, `GPUCostIdle`, the `deprecatedGPURequestAverage`/`deprecatedGPUUsageAverage` fields tagged `@bingen:field[version=22]`, and `GPUAllocation` at `version=23`), the `GPUAllocation` struct, `GPUEfficiency()` with its comment that the value is zero without the NVIDIA DCGM exporter, and `TotalEfficiency()` — which composes CPU and RAM only (G6).
- **`pkg/inferencecost/` and `docs/inference-cost-tracking.md`** — <https://github.com/opencost/opencost/tree/develop/pkg/inferencecost> and <https://github.com/opencost/opencost/blob/develop/docs/inference-cost-tracking.md> — the 2026 module: `INFERENCE_COST_ENABLED` default `false` (`pkg/env/costmodel.go`), the env-var table, the two cost bases and their `ShareIdle`/`ShareSplit` settings, the clamped `GPUUsageAverage` scaling in `collector.go`, the compute-time input/output split with the 2.5× multiplier fallback, `cacheSavingsFraction` from `vllm:prefix_cache_hits_total`, and the three `llm_*` gauges in `exporter.go`.
- **`modules/prometheus-source/pkg/prom/inference_queries.go`** — <https://github.com/opencost/opencost/blob/develop/modules/prometheus-source/pkg/prom/inference_queries.go> — the six vLLM metric names read as counter deltas, and the `vllm:cache_config_info` join used only to detect `enable_prefix_caching=false`.

**Real-world engineering — the project's own tracker**

- **OpenCost issue #3900, "GPU 'usage' cost basis does not reflect actual GPU utilization"** — <https://github.com/opencost/opencost/issues/3900> — filed 2026-07-05, closed 2026-07-09. What it shows: the maintainers' own statement of this lesson's argument, with the `applyGPUsAllocated` citation, the two-cost-basis diagnosis, and the $2.50-vs-$0.275 expected/actual at ~11% utilisation. *Issue metadata and body retrieved via the GitHub API on 2026-08-18; `github.com` HTML pages are egress-blocked from this environment, so the rendered thread and its comments were not read — the body text is quoted from the API response.*
- **OpenCost issue #3828, "Support HAMi GPU core and memory allocation in cost calculations"** — <https://github.com/opencost/opencost/issues/3828> — filed 2026-06-02, open. What it shows: the fractional-GPU gap reported by a user with `nvidia.com/gpucores: 25` on a whole-GPU request, and a workaround that is structurally this module's deliverable. Also the history pointers to #934, #2862, #3039, #3061. Same API-retrieval caveat.
- **OpenCost issue #3160, "Custom Pricing ignored on GPU nodes"** — <https://github.com/opencost/opencost/issues/3160> — filed 2025-05-27, open, against 1.114.0/1.115.0. What it shows: with all custom prices set to `1`, GPU nodes report `node_cpu/ram/gpu_hourly_cost` as small fractions while `node_total_hourly_cost` lands near `1`, whereas CPU-only nodes report exactly `1` — a live on-prem pricing defect on the *price* factor, not the quantity factor. Same API-retrieval caveat.
- **Kubecost / Apptio GPU monitoring announcement** — <https://www.apptio.com/blog/gpu-monitoring/> — *`apptio.com` is egress-blocked from this environment and was **not** read in this pass. Carried forward as a URL only. Kubecost 2.x is closed source, so no claim in this lesson's matrix about Kubecost's internal cost basis is source-verified — the matrix marks those cells `?` deliberately, and you should too.*

**Deeper dives**

- **NVIDIA `k8s-device-plugin`** — <https://github.com/NVIDIA/k8s-device-plugin> — how MIG instances and time-slice replicas are advertised, and specifically what GPU Feature Discovery's `renameByDefault` flag does to the resource key (`nvidia.com/gpu` versus `nvidia.com/gpu.shared`) and to the `nvidia.com/gpu.replicas` / `nvidia.com/gpu.count` labels. This is the upstream behaviour that decides which of G2's two error directions you get, and OpenCost's `getGPUCount()` links to its resource tests directly.
- **Project HAMi** — <https://github.com/Project-HAMi/HAMi> — the `nvidia.com/gpucores` / `nvidia.com/gpumem` / `nvidia.com/gpumem-percentage` resource model behind issue #3828, and a concrete example of an enforcement layer that knows a fraction the cost layer never sees.
- **Module 04 capstone, per-pod attribution** — [04 L10 — Capstone: per-pod attribution](../../04-gpu-on-kubernetes/lessons/10-capstone-per-pod-attribution.md) — identity A (`Σ attributed + Σ idle ≡ G × W`) is the one-query test that catches the MIG 7× and the time-slice `$0.00` immediately, on any tool, without reading its source. Run it against whatever is installed before you read anything.
- **Lesson 01, four sharing regimes** — [01 — GPU cost attribution: four sharing regimes](01-attribution-models.md) — the argument that regime 3 is unrecoverable from device counters, which is why fix #6 in §6 is marked blocked rather than hard.

---
Module backlink: [💰 11 — GPU cost and unit economics](../README.md)
