---
lesson: 10
title: "FOCUS 1.x and designing a GPU cost schema"
module: 11
concept: "FOCUS-aligned GPU cost schema"
status: not-started
est_time: "6 hrs"
prev: "09-existing-tooling-limits.md"
next: null
artifacts: ["the module-capstone FOCUS-aligned GPU cost/usage schema (columns + types + provenance) carrying both ledgers, the attribution regime, and unit-cost joins, dropped into the module deliverable"]
sources: 11
---

# FOCUS 1.x and designing a GPU cost schema

*Prev: [09 — Where existing tooling fails: reading the OpenCost source](09-existing-tooling-limits.md) · Next: none — this lesson is the module capstone*

## Where this fits
Lesson 09 proved, from source, exactly where OpenCost's allocated-ledger, whole-GPU-request design throws away everything this module built: the fractional MIG basis, the utilised ledger, the app-level unit costs. This lesson takes that named gap and closes it with a schema — the module's single deliverable artifact. Every prior lesson becomes a column here: the four regimes (01) become `x_AttributionRegime`, the two ledgers (02) become the FOCUS cost columns plus the `x_Utilised*` extensions, unit economics (05/07/08) become the `x_AppTokens`/`x_AppRuns` joins, and the OpenCost gap (09) becomes the reason this schema needs to exist at all rather than just quoting a dashboard.

## Why this matters
This is the module capstone. Everything upstream — the four regimes (01), the two ledgers (02), idle detection (03), unit economics (04), fragmentation (05), and the proof that OpenCost bills the allocated ledger and drops utilisation (09) — converges here into one artifact: **a schema.** When a hiring panel says "design a cost and usage schema for a shared GPU fleet," they are testing whether you know the industry standard (FOCUS), whether you know its edges, and whether you can extend it *precisely* where GPUs still break it. The candidate who invents column names from scratch signals they've never shipped FinOps at scale; the candidate who says "start from FOCUS, know exactly which version added what, and add only the two things it genuinely still lacks for GPUs" signals they have.

FOCUS — the FinOps Open Cost & Usage Specification — matters because it is the answer to the landscape gap from lesson 09: it normalises cost and usage across AWS, Azure, GCP, OCI, and increasingly SaaS/on-prem into one column vocabulary, so your GPU cost table speaks the same language as the rest of finance's tooling. AWS, Azure, and GCP all ship FOCUS exports. If your GPU schema is FOCUS-aligned, it joins the corporate cost lake for free; if it's bespoke, it's a silo.

**The stake, stated precisely, matters more than ever here — because the spec has moved.** As of **FOCUS 1.3** (ratified December 2025), the standard gained a first-class **split-cost allocation** mechanism aimed explicitly at "compute nodes in a shared cluster (Kubernetes, databases)" — the GPU-sharing case this whole module is about. That means the old, simpler story — "FOCUS has no way to represent shared cost, full stop" — is **no longer true**, and repeating it in an interview in 2026 is itself a signal of stale knowledge. What's still true, and still the sharper interview point, is narrower: FOCUS 1.4 gives you a *place* to declare how a shared GPU's cost was split, but it still has **no column for how confident you are in that split**, and it still has **no native utilisation ledger** at all. Knowing exactly where the standard now stops — not where it stopped eighteen months ago — and extending it with disciplined `x_`-prefixed columns for exactly those two remaining gaps is the whole skill. This lesson produces the schema that *is* the deliverable's centerpiece.

## What's new here (calibration)
- **Already yours (skip):** the four regimes and the time-slice unsolvability proof (01); allocated-vs-utilised (02); cost-per-token / cost-per-run (07/08); the OpenCost allocation-basis defect (09). None re-taught — all become *columns*.
- **New angle 1:** the real, current FOCUS column vocabulary through **1.4** (ratified June 2026) — exact names and definitions (`BilledCost`, `EffectiveCost`, `ContractedCost`, `ConsumedQuantity`, `ChargeCategory`, `CommitmentDiscountId`, `AllocatedResourceId`, `AllocatedMethodId`, `Tags`, …), not invented ones, and not frozen at an old version.
- **New angle 2:** how a GPU-hour maps onto a FOCUS charge row under all three sharing regimes — including the corrected picture: MIG and even time-sliced sharing now have a **named, spec-defined place** to declare a split (`AllocatedMethodId`, since 1.3), but the *confidence* of a time-sliced split still has nowhere to live in core FOCUS.
- **New angle 3:** the capstone schema — a FOCUS-conformant core (including the *new* 1.3+ split-cost columns) plus a much smaller `x_`-prefixed extension block carrying only what genuinely remains missing: the utilised ledger and the attribution uncertainty.

## Core concepts

### FOCUS's own change discipline: why "which version" is a real question
FOCUS classifies every spec change as **Compatible**, **Migration Compatible**, or **Incompatible**, and version-gates features by a *feature level* (e.g. Conditional, Recommended) rather than silently breaking consumers. That discipline is why a FOCUS-aligned pipeline built in 2024 still parses in 2026 — new columns arrive nullable and additive; old columns get formally deprecated (kept, but discouraged) for a release before removal. Two versions matter for this lesson:

- **FOCUS 1.3 — ratified December 2025.** Added the `ContractCommitment` dataset, five new supported features including **"Data Generator-Calculated Split Cost Allocation"** (the shared-cost mechanism, below), eight new `CostAndUsage` columns (`AllocatedMethodDetails`, `AllocatedMethodId`, `AllocatedResourceId`, `AllocatedResourceName`, `AllocatedTags`, `ContractApplied`, `HostProviderName`, `ServiceProviderName`), and **deprecated** `ProviderName` and `PublisherName`.
- **FOCUS 1.4 — ratified June 2026, current as of this writing.** Added `BillingPeriod` and `InvoiceDetail` datasets, Invoice Reconciliation and Commitment Program Eligibility Details features, and **removed `ProviderName`/`PublisherName` entirely** — the columns this lesson's schema must not cite anymore.

This matters beyond trivia: a candidate who cites `ProviderName` in 2026 is citing a column that no longer exists in the spec they claim to know.

### The FOCUS column vocabulary you must know (through 1.4)
FOCUS is column-oriented: every charge is one row; every provider normalises to the same columns. The load-bearing set for GPU compute:

**Cost columns (the four-way cost comparison):**
| Column | Meaning |
|---|---|
| `BilledCost` | What was/will be invoiced for this row (post-discount, as billed). |
| `EffectiveCost` | Amortised real cost after all reduced rates, discounts, and the applicable portion of prepaid commitments. The one you allocate on. |
| `ListCost` | Cost at public list price (`ListUnitPrice` × `PricingQuantity`). |
| `ContractedCost` | Cost at negotiated/contracted price (`ContractedUnitPrice` × `PricingQuantity`). |

**Quantity & price:** `ConsumedQuantity` + `ConsumedUnit` (what was used — e.g. GPU-hours), `PricingQuantity` + `PricingUnit` (the unit the price is quoted in), `ListUnitPrice`, `ContractedUnitPrice`.

**Accounts:** `BillingAccountId` / `BillingAccountName` / `BillingAccountType`, `SubAccountId` / `SubAccountName` / `SubAccountType` — payer vs the linked account/project/subscription the GPU lives in.

**Resource:** `ResourceId`, `ResourceName`, `ResourceType` — the GPU node/instance (or, if you push it, the GPU/MIG UUID).

**Classification:** `ServiceCategory` (e.g. *Compute*, *AI and Machine Learning*), `ServiceSubcategory`, `ServiceName`; `ChargeCategory` (`Usage` | `Purchase` | `Tax` | `Credit` | `Adjustment`), `ChargeClass` (`Correction` or null), `ChargeDescription`, `ChargeFrequency`; `ChargePeriodStart` / `ChargePeriodEnd`, `BillingPeriodStart` / `BillingPeriodEnd`.

**Commitment discounts** (RIs / savings plans / CUDs — lesson 06): `CommitmentDiscountId`, `CommitmentDiscountName`, `CommitmentDiscountType`, `CommitmentDiscountCategory`, `CommitmentDiscountStatus` (`Used` | `Unused`), `CommitmentDiscountQuantity` + `CommitmentDiscountUnit`.

**Provider/invoice identity — corrected for 1.4:** the legacy `ProviderName` / `PublisherName` columns are **gone as of 1.4** (deprecated in 1.3, removed in 1.4). Use instead: `ServiceProviderName` (who actually made the resource/service available — the entity that produced or sold it, directly or via reseller), `HostProviderName` (who runs the underlying infrastructure the service sits on, when that differs from the service provider), and `InvoiceIssuerName` (who issued the invoice — relevant for marketplace/reseller GPU capacity). For a neocloud-vs-hyperscaler comparison (lesson 07) this split matters: a managed inference platform's `ServiceProviderName` may differ from the `HostProviderName` actually running the GPUs underneath it.

**Split-cost allocation columns — new in 1.3, core (not `x_`):** `AllocatedResourceId`, `AllocatedResourceName`, `AllocatedMethodId`, `AllocatedMethodDetails`, `AllocatedTags` — the mechanism the next section covers in full, because it is this lesson's central correction.

**SKU / location:** `SkuId`, `SkuPriceId`, `SkuMeter`, `SkuPriceDetails`; `RegionId` / `RegionName` / `AvailabilityZone`; `CapacityReservationId` / `CapacityReservationStatus` — relevant to reserved GPU capacity.

**Tags:** `Tags` — a key/value map, FOCUS's general-purpose allocation dimension (team, namespace, model). **Custom, non-standard properties anywhere in FOCUS must use an `x_` prefix** — this rule is unchanged and still explicitly mandated in the current (1.4) spec's custom-column-handling attribute.

### Data Generator-Calculated Split Cost Allocation (1.3+) — the correction, in full
This is the fact that changes this lesson's thesis, so read it precisely. FOCUS 1.3 added a named feature, **"Data Generator-Calculated Split Cost Allocation,"** built to let a data generator (the cloud/tool producing the FOCUS export) allocate the cost of one shared resource across multiple consumers based on actual usage — and the spec's own feature documentation names the target scenario as **"compute nodes in a shared cluster (Kubernetes, databases)"** — i.e., exactly the MIG/time-slice GPU-sharing case this module has been analyzing since lesson 01.

The mechanism is a parent/child relationship expressed with five new columns, all core FOCUS, all nullable, all required to be null on rows that aren't part of a split:

- **`ResourceId`** stays the *parent* — the physical GPU or shared node whose total cost is being split.
- **`AllocatedResourceId`** (+ `AllocatedResourceName`) identifies the *child* — the specific MIG instance, time-slice tenant, or workload receiving a portion of the parent's cost. It must be locally unique within a `(ResourceId, ChargePeriod)` pair.
- **`AllocatedMethodId`** is a **provider-defined identifier for the split methodology** — a string the data generator emits so a reader can look up, in that provider's own documentation, exactly how the split was computed. FOCUS does not enumerate the methodologies; a provider could use identifiers like `sm_slice` (physical SM-partition fraction, MIG's natural basis) or `context_runtime` (measured GPU-context wall-time share, time-slicing's best available approximation) — FOCUS just requires that *some* identifier be present whenever a charge is part of a split, and that it be resolvable to an external description of the method.
- **`AllocatedMethodDetails`** is a JSON object carrying the numeric detail of that split — the spec's own worked example shows fields like `AllocatedRatio` (the fraction assigned to this child), `UsageUnit`, and `UsageQuantity` (the consumption metric that drove the ratio).
- **`AllocatedTags`** carries tags specific to the allocated child that aren't already present in the parent row's `Tags` — the per-tenant identity that rides along with the split.

**What this genuinely fixes:** a MIG-fractional row, and even a time-sliced row, can now be represented with **proper core-FOCUS split-cost columns** — a parent resource, a named child, a named method, and a ratio — not merely through `Tags` the way 1.2 forced you to. The old claim that "allocation across tenants remains something you do outside the spec, expressed only through `Tags`" is **false as of 1.3.**

**What this does *not* fix, and is the sharper, corrected edge case:** `AllocatedMethodId` records *which* method was used and `AllocatedMethodDetails` can carry a ratio, but there is still **no column for the uncertainty or confidence of that ratio**. For a MIG instance, that's fine — the SM-slice fraction is a physically metered fact, not an estimate, so there's nothing to be uncertain about. For a time-sliced tenant, it is not fine: the lesson-01 proof that per-tenant utilisation is unrecoverable from driver/DCGM signals under time-slicing still holds exactly as before. FOCUS 1.3+ now gives you a *place* to declare "this row's cost was split by context-runtime weighting" and a JSON slot for the resulting ratio — but it still cannot tell a reader **how much to trust that ratio**. The story is no longer "FOCUS can't represent this at all"; it is now the more precise, more senior point: **"FOCUS 1.3+ names the methodology but still can't carry the confidence."**

### Mapping a GPU-hour onto a FOCUS row
- **Whole GPU (regime 1) — clean, unaffected by 1.3.** One row per GPU-node charge period: `ConsumedQuantity` = GPU-hours, `ConsumedUnit` = `GPU-hour`, `ContractedUnitPrice` = $/GPU-hr, `ContractedCost`/`EffectiveCost`/`BilledCost` filled from billing, `ServiceCategory` = *Compute*, `ChargeCategory` = `Usage`, `ResourceId` = node/instance, tenant in `Tags`. No `Allocated*` columns needed — nothing is split. FOCUS handles this with room to spare.
- **MIG fractional (regime 2) — clean, now via a genuine split-cost primitive.** A `1g.10gb` instance over an hour gets its own row: `ResourceId` = the physical H100 (the parent), `AllocatedResourceId` = the MIG instance's UUID, `AllocatedMethodId` = `sm_slice`, `AllocatedMethodDetails` = `{"AllocatedRatio": 0.142857, "UsageUnit": "GPU", "UsageQuantity": 1}`, `ConsumedQuantity` = 0.1429 GPU-hours at the whole-GPU unit price. This is where FOCUS is *strictly better than OpenCost*: lesson 09's 7× bug disappears because the split is declared explicitly, both as a fractional `ConsumedQuantity` **and** as a named, spec-conformant allocation from a stated parent. The catch, unchanged from before: **FOCUS doesn't tell you how to compute 0.1429** — the fractional basis (SM vs memory vs blended) is still your decision, documented externally via whatever `sm_slice` resolves to in your own provider documentation.
- **Time-sliced (regime 3) — now maps, but the honesty still lives outside the spec.** To emit per-tenant rows you split one device's GPU-hours across N contexts: `ResourceId` = the physical GPU (parent), `AllocatedResourceId` = a per-context/per-pod identifier, `AllocatedMethodId` = `context_runtime`, `AllocatedMethodDetails` carrying the observed ratio. Lesson 01 proved that ratio is *unrecoverable exactly* from DCGM — it's the best available estimate, not a metered fact — and FOCUS 1.3+ still gives **no column for the error band on that estimate**. You can now emit fully conformant, properly-linked split-cost rows for time-sliced tenants; what you still cannot do in-spec is tell a downstream reader how much to trust the split. **That's the residual gap, and it's narrower and more precise than "FOCUS can't represent this."**
- **The other missing leg — the utilised ledger.** Every FOCUS cost column, including the new `Allocated*` ones, is still an *allocated/billed* number — a quantity or a dollar figure, never a utilisation fraction. There is still **no FOCUS column for utilisation** (SM-active, lesson 05) or for a utilised-cost ledger (lesson 02), in 1.3 or 1.4. If your schema is to show "you paid for X GPU-hours and burned Y% of them," that still lives entirely in `x_` extensions.

### The capstone schema — FOCUS-conformant core (incl. 1.3+ split-cost) + genuine GPU extensions
One row per **(GPU-or-MIG-or-context UUID × tenant × charge period)**. Core columns are FOCUS 1.4-conformant, including the split-cost block, so the table drops into any current FOCUS lake; the much smaller `x_` block now carries only the two things FOCUS still structurally lacks: the utilised ledger and the attribution-confidence band.

| Column | Type | FOCUS? | Produced from |
|---|---|---|---|
| `ChargePeriodStart` / `ChargePeriodEnd` | timestamp | core | metric/billing window |
| `BillingAccountId` / `SubAccountId` | string | core | cloud billing / cluster id |
| `ServiceProviderName` / `InvoiceIssuerName` | string | core (1.3+; replaces the deprecated-then-removed `ProviderName`/`PublisherName`) | billing — who ran the hardware vs who issued the invoice (lesson 07) |
| `ResourceId` | string | core | **pod-resources API** + DCGM → the *parent* GPU or node UUID |
| `ResourceType` | string | core | `GPU` / `MIG Instance` (device-plugin) |
| `ServiceCategory` / `ServiceName` | string | core | billing (*Compute* / instance type) |
| `ChargeCategory` | enum | core | `Usage` |
| `ChargeClass` | enum, null | core | `Correction` on restatements |
| `ConsumedQuantity` | decimal | core | allocated GPU-hours — pod-resources × regime fraction |
| `ConsumedUnit` | string | core | `GPU-hour` |
| `PricingQuantity` / `PricingUnit` | decimal / string | core | billing |
| `ContractedUnitPrice` | decimal | core | pricing ($/GPU-hr; spot/on-demand/committed) |
| `ListCost` / `ContractedCost` / `EffectiveCost` / `BilledCost` | decimal | core | **billing — the allocated-ledger $** |
| `CommitmentDiscountId` | string, null | core | reservation/CUD coverage (lesson 06) |
| `AllocatedResourceId` / `AllocatedResourceName` | string, null | **core (1.3+)** | pod-resources API → the child MIG UUID or per-tenant context id receiving a split of `ResourceId`'s cost |
| `AllocatedMethodId` | string, null | **core (1.3+)** | provider-defined split-methodology id: `sm_slice` (MIG) or `context_runtime` (time-slice) — documented externally |
| `AllocatedMethodDetails` | JSON, null | **core (1.3+)** | the split ratio/usage detail, e.g. `{AllocatedRatio, UsageUnit, UsageQuantity}` |
| `AllocatedTags` | JSON, null | **core (1.3+)** | per-tenant tags specific to the allocated child, not already in `Tags` |
| `Tags` | map | core | tenant / team / namespace / model → top-level allocation dimension |
| `x_AttributionRegime` | enum | ext | device-plugin/pod-resources: `exclusive`\|`mig`\|`timeslice`\|`dra` (lesson 01) |
| `x_FractionalBasis` | enum, null | ext | `sm_slice`\|`memory`\|`blended`\|`request`\|`context_runtime` (lesson 01) — mirrors `AllocatedMethodId` for internal tooling that hasn't yet standardised on reading it |
| `x_UtilisationMean` | decimal 0–1 | ext | **DCGM `DCGM_FI_PROF_GR_ENGINE_ACTIVE`** (SM-active, lesson 05) — FOCUS still has no utilisation column |
| `x_UtilisedGPUHours` | decimal | ext | `ConsumedQuantity × x_UtilisationMean` — the **utilised ledger** FOCUS still doesn't carry |
| `x_EffectiveUtilisedCost` | decimal | ext | `EffectiveCost × x_UtilisationMean` — utilised-cost $ FOCUS still omits (lesson 02) |
| `x_AttributionUncertainty` | decimal, null | ext | the confidence band `AllocatedMethodDetails` has no column for (lesson 01); 0/null for `exclusive` & `mig` (metered, exact), populated for `timeslice` (estimated) |
| `x_AppTokens` | bigint, null | ext | inference server counter (lesson 07) |
| `x_AppRuns` | bigint, null | ext | orchestrator/job counter (lesson 08) |
| `x_UnitCostPerMTokens` | decimal, null | ext | `EffectiveCost / (x_AppTokens/1e6)` (lesson 07) |
| `x_UnitCostPerRun` | decimal, null | ext | `EffectiveCost / x_AppRuns` (lesson 08) |

**Why this shape wins, and why it's a smaller `x_` block than an earlier version of this schema would have needed:** the core row — now including the split-cost block — is what finance and the FOCUS lake consume directly: allocated ledger, standard columns, a spec-conformant declaration of *how* a shared GPU's cost was split, all joinable across clouds. The `x_` block has shrunk to carry only the two things FOCUS structurally still can't: the **utilised ledger** (`x_Utilised*`), and the **honest confidence** on an estimated split (`x_AttributionUncertainty`) — plus the **unit-economics joins** to app counters, which are out of FOCUS's scope entirely rather than a gap in it. `x_AttributionRegime` / `x_FractionalBasis` are kept as a redundant, machine-simple mirror of the regime and method for tooling that hasn't yet standardised on parsing `AllocatedMethodId`/`AllocatedMethodDetails`. Three provenance streams still feed it — **pod-resources API** (who was allocated what device, the regime, the allocated-child identity), **DCGM** (utilisation, UUID), and **cloud billing / FOCUS export** (the dollars, and now the split-cost declaration) — exactly the three-way join lesson 09 said no incumbent tool performs.

## Perspectives

**Standards-body / spec-versioning perspective.** FOCUS classifies every change as Compatible, Migration Compatible, or Incompatible, and gates new columns behind feature levels rather than breaking existing consumers overnight — `ProviderName`/`PublisherName` were *deprecated* in 1.3 before being *removed* in 1.4, a full release of runway. That discipline is worth studying independent of GPUs: it's a working example of how a serious industry-wide data standard evolves without stranding every pipeline built against last year's version. A schema you design today should borrow the same posture — additive, nullable extensions, a documented deprecation window before anything is removed.

**Data-engineering / schema-design perspective.** Every column in the capstone table has a named provenance stream (pod-resources API, DCGM, billing, app counters) — and that's a design choice, not an afterthought. A schema that doesn't record *where* a value came from can't be debugged when it's wrong, and can't be defended when a tenant disputes a bill. Naming provenance as a first-class design concern — the `Produced from` column above — is what turns a spreadsheet of column names into an auditable system.

**Multi-cloud finance perspective.** The business reason to start from FOCUS instead of building a bespoke schema isn't philosophical — it's that AWS, Azure, and (increasingly) GCP already export FOCUS-shaped data, and the FinOps Foundation's own adoption tracking shows this converging further each year. A GPU cost table that speaks FOCUS joins the corporate cost lake — the one finance already queries — for free. A bespoke schema, however well-designed, is a silo finance has to learn separately, and silos are what get deprecated when headcount tightens.

**GPU-extension perspective — precisely scoped.** Where does the *current* (1.4) spec still not reach for GPUs? Exactly two places, and it's important to say only these two, not more: (1) there is no utilisation ledger anywhere in FOCUS — every cost column is allocated/billed, never a measured-use fraction; (2) there is no confidence/uncertainty column on a declared split — `AllocatedMethodDetails` can carry a ratio, but not an error band on that ratio. Everything else a shared-GPU schema needs — including the split-cost mechanism itself — is now core FOCUS as of 1.3. Scoping the gap this narrowly, instead of the older and now-inaccurate "FOCUS can't represent shared cost at all," is itself the signal of someone who reads the spec's changelog rather than repeating received wisdom.

## Real-world use cases

- **FinOps Foundation, FinOps X cloud-provider FOCUS announcements** — https://www.finops.org/insights/finops-x-2025-cloud-announcements/ *(fetch blocked by this environment's egress policy for `finops.org`; canonical URL cited, not independently re-verified in this session)*. AWS added native FOCUS data-export support in the AWS Billing console (announced at FinOps X 2024), Google Cloud shipped a BigQuery view and Looker template aligned to the FOCUS 1.0 GA release, and by 2025–2026 AWS and Azure export FOCUS-shaped billing data natively with GCP continuing to build out coverage. What it shows: this is the real, dated, multi-vendor adoption story that makes "start from FOCUS" a business argument, not just a technically tidy one — the schema you design joins exports that already exist in production at all three major clouds.
- **OpenCost, "OpenCost Supports the FinOps Open Cost and Usage Specification (FOCUS)"** — https://opencost.io/blog/focus/ *(fetch blocked by this environment's egress policy for `opencost.io`; canonical URL cited, not independently re-verified in this session)*. OpenCost's own first-party post on adopting FOCUS terminology and mapping its Kubernetes cost model onto FOCUS columns. What it shows: this directly ties lesson 09 (OpenCost, the tool that only bills the allocated ledger) to lesson 10 (FOCUS, the schema this capstone builds) — the same open-source project this module spent a whole lesson reading source for is itself converging on the standard this lesson teaches.
- **Kubecost, "Kubecost Launches Cloud Cost Optimization and Support Through OpenCost and the FinOps Foundation's FOCUS Specification"** (GlobeNewswire, October 31, 2023) — https://www.globenewswire.com/en/news-release/2023/10/31/2769876/0/en/Kubecost-Launches-Cloud-Cost-Optimization-and-Support-Through-OpenCost-and-the-FinOps-Foundation-s-FOCUS-Specification.html *(fetch blocked by this environment's egress policy for `globenewswire.com`; canonical URL cited, not independently re-verified in this session)*. A dated press-release anchor for when the OpenCost/Kubecost/FOCUS ecosystem started converging — useful for placing this lesson's material on a timeline: FOCUS-Kubernetes convergence began in 2023, the split-cost mechanism this lesson centers on didn't land until FOCUS 1.3 in December 2025, over two years later.

## Worked example
One H100 node, `EffectiveCost` $3.00/GPU-hr, over a 1-hour charge period. Three tenancy shapes, three sets of rows — now written against the FOCUS 1.4 schema above, including the split-cost columns.

**(a) Exclusive — pod `train-A`.** One row: `ConsumedQuantity = 1.0` GPU-hour, `EffectiveCost = 3.00`, `AllocatedResourceId = null` / `AllocatedMethodId = null` (nothing is split — the whole parent resource belongs to one tenant), `x_AttributionRegime = exclusive`, `x_FractionalBasis = null`, `x_UtilisationMean = 0.40` (DCGM), `x_UtilisedGPUHours = 0.40`, `x_EffectiveUtilisedCost = 1.20`, `x_AttributionUncertainty = 0`, `Tags = {team: research}`. Both ledgers, one clean row, no split needed.

**(b) MIG — 7× `1g.10gb`, seven tenants.** Seven rows. `ResourceId` = the physical H100's UUID (the parent, shared across all seven rows); each row's `AllocatedResourceId` = that instance's MIG UUID, `AllocatedMethodId = "sm_slice"`, `AllocatedMethodDetails = {"AllocatedRatio": 0.142857, "UsageUnit": "GPU", "UsageQuantity": 1}`, `ConsumedQuantity = 0.1429` GPU-hour, `EffectiveCost = 0.4286`, `x_AttributionRegime = mig`, `x_FractionalBasis = sm_slice`, `x_AttributionUncertainty = 0` — the SM-slice fraction is a physically metered fact, so there's no confidence band to carry. The seven `EffectiveCost` values sum to $3.00 — **no 7× inflation** (contrast lesson 09's OpenCost, which bills $21). This row is now doubly correct: right by construction (fractional `ConsumedQuantity`) *and* declared through FOCUS's own split-cost primitive rather than smuggled through `Tags`.

**(c) Time-sliced — 4 pods, `replicas: 4`.** Four rows, sharing the same parent `ResourceId`. Each row's `AllocatedResourceId` = a per-context identifier, `AllocatedMethodId = "context_runtime"`, `AllocatedMethodDetails` carrying the observed runtime ratio; `EffectiveCost` split by that context-runtime weighting → `2.10 / 0.75 / 0.075 / 0.075`, summing to $3.00. `x_AttributionRegime = timeslice`, `x_FractionalBasis = context_runtime`. **This is the row where you show the interviewer you know FOCUS's current edge, precisely:** the split is now declared through genuine core-FOCUS columns (`AllocatedMethodId`/`AllocatedMethodDetails`), not an out-of-spec workaround — but because the underlying ratio is an *estimate* (lesson 01's unrecoverability proof), each row still carries `x_AttributionUncertainty` set to the spread against the equal-split alternative (up to ~10× on the small pods), because FOCUS 1.4 has no column that could carry that number itself. `x_UtilisationMean` is flagged as the *device* aggregate, since it can't be resolved per tenant.

**Unit cost.** For the inference tenant serving 8.4M tokens on its `EffectiveCost` of $0.4286: `x_AppTokens = 8_400_000`, `x_UnitCostPerMTokens = 0.4286 / 8.4 = $0.051 / M-tokens` (lesson 07) — a business-legible number produced by joining the app counter onto the FOCUS row, entirely outside anything FOCUS itself defines.

## Practice
Feed these into the module deliverable at [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md):
1. **Emit the schema.** Write the capstone table above as a concrete DDL (or a typed CSV/Parquet header) for your fleet, splitting core FOCUS columns (including the 1.3+ split-cost block) from the genuine `x_` extensions. This is the deliverable's centerpiece.
2. **Three-regime rows.** Produce the (a)/(b)/(c) rows for a real or plausible fleet hour, showing that MIG rows sum without inflation using `AllocatedMethodId = sm_slice`, and that time-slice rows use `AllocatedMethodId = context_runtime` while still carrying `x_AttributionUncertainty`.
3. **Provenance map.** For every column, name its source stream (pod-resources / DCGM / billing / app counter) — the three-way join lesson 09 said nobody does.
4. **The interview paragraph.** Write the answer to "design a cost/usage schema for a shared GPU fleet": start from current FOCUS as the industry standard, name the *two* things it still lacks for GPUs (utilised ledger, split-confidence/uncertainty), and state precisely that the split-cost mechanism itself is no longer one of them as of 1.3 — then state the regime-3 limit on the confidence of that split.

## Common pitfalls
1. **Believing "FOCUS has no way to represent shared/split cost."** This was true only through FOCUS 1.2. It is **false as of 1.3** (ratified December 2025), which added the `AllocatedResourceId`/`AllocatedMethodId`/`AllocatedMethodDetails`/`AllocatedTags` split-cost columns, explicitly targeting shared compute nodes including Kubernetes. This is the central correction of this lesson, not a footnote — repeating the old claim is now a stale-knowledge signal in an interview.
2. **Assuming FOCUS mandates a specific allocation algorithm now that split-cost columns exist.** It doesn't. `AllocatedMethodId` records *which* method was used (documented externally, by the provider/data generator); FOCUS still doesn't prescribe SM-slice vs memory-fraction vs equal-split vs context-runtime weighting. Choosing the method and defending it remains the engineer's job — FOCUS just gives you a standard place to declare the choice.
3. **Believing FOCUS 1.3's split-cost feature solves the regime-3 (time-slicing) attribution problem.** It gives you a place to declare a split and a JSON slot for a ratio — it does not give you a column for the *confidence* of that ratio, and it cannot manufacture a per-tenant counter that doesn't physically exist. The regime-3 unsolvability proof from lesson 01 is completely unchanged; only the *representation* of the (still-estimated) result improved.
4. **Assuming the utilisation ledger is now native to FOCUS because split-cost was added.** The two are unrelated features. Utilisation (SM-active, allocated-vs-utilised) is still entirely absent from FOCUS in 1.4; the `x_Utilisation*` / `x_UtilisedGPUHours` / `x_EffectiveUtilisedCost` extensions remain necessary and unchanged.
5. **Citing or inventing the deprecated column names `ProviderName`/`PublisherName`.** They were deprecated in 1.3 and **removed entirely in 1.4**. Use their replacements: `ServiceProviderName`, `HostProviderName`, `InvoiceIssuerName`.

## Self-check
- Why is a MIG instance easy to represent in FOCUS, and is a time-slice tenant still impossible to represent? **Answer:** A MIG instance has a real per-instance counter and a defensible, physically metered fraction (lesson 01), so as of FOCUS 1.3 it becomes a fully conformant split-cost row: `ResourceId` (parent) + `AllocatedResourceId` (the MIG UUID) + `AllocatedMethodId = "sm_slice"` + `AllocatedMethodDetails` carrying the exact ratio, with `x_AttributionUncertainty = 0` because there's nothing to be uncertain about. A time-slice tenant is **no longer impossible to represent** — it uses the same core split-cost columns (`AllocatedMethodId = "context_runtime"`) — but its ratio is an *estimate*, not a metered fact (regime-3 proof, lesson 01), and FOCUS has no column for the confidence of that estimate. So the correct answer is: representable, since 1.3, but the honesty about *how much to trust* the number still lives only in `x_AttributionUncertainty`.
- What does FOCUS 1.4 still structurally lack for a shared-GPU schema, and how do you add it without leaving the standard? **Answer:** Exactly two things, no more: (1) a **utilisation ledger** — every FOCUS cost/quantity column, including the new `Allocated*` ones, is still allocated/billed, never a measured-use fraction; (2) a **confidence/uncertainty column on a declared split** — `AllocatedMethodDetails` can carry a ratio but not an error band on it. (Split-cost allocation itself is *not* on this list anymore — that gap closed in 1.3.) You add both as `x_`-prefixed columns (`x_UtilisationMean`, `x_UtilisedGPUHours`, `x_EffectiveUtilisedCost`, `x_AttributionUncertainty`), the FOCUS-sanctioned extension path, keeping the core row fully conformant and lake-joinable.
- Which three data streams produce a row in the capstone schema, and why does that matter? **Answer:** The **pod-resources API** (device binding, sharing regime, the allocated-child identity that becomes `AllocatedResourceId`), **DCGM** (per-UUID utilisation, `x_UtilisationMean`), and **cloud billing / FOCUS export** (the `EffectiveCost`/`BilledCost` dollars, and now the split-cost declaration itself), optionally joined to **app counters** (tokens/runs) for unit cost. It matters because lesson 09 showed no incumbent tool joins all three — OpenCost has $ + reported-but-unbilled utilisation but no counters, DCGM+Grafana has utilisation but no $, billing stops at the node — so this schema *is* the missing join, expressed in the industry-standard vocabulary, now including the standard's own split-cost mechanism rather than a workaround around its absence.
- Is it still accurate to say "FOCUS has no split-cost primitive, so cross-tenant GPU sharing can only be expressed through `Tags`"? **Answer:** No — that statement was accurate through FOCUS 1.2 and is **false as of FOCUS 1.3** (ratified December 2025, current spec 1.4 as of mid-2026). FOCUS now has a first-class split-cost primitive: `AllocatedResourceId` names the child receiving a portion of a parent `ResourceId`'s cost, `AllocatedMethodId` names the methodology, `AllocatedMethodDetails` carries the ratio, and `AllocatedTags` carries child-specific tags. `Tags` alone is no longer the only allocation dimension. The corrected, still-true claim is narrower: FOCUS has a place to *declare* a split, but no column to carry the *confidence* of an estimated one.

## Connections & what's next
This is the module capstone: it does not open a new thread, it closes every thread this module opened. Regime 01 becomes `x_AttributionRegime`/`x_FractionalBasis`/`AllocatedMethodId`; the two ledgers of 02 become the FOCUS cost columns plus `x_Utilised*`; idle detection (03) and fragmentation (04) inform which rows get flagged for reclamation once this schema is queried; unit economics (05/07/08) become the `x_AppTokens`/`x_AppRuns`/`x_UnitCost*` joins; commitment strategy (06) becomes `CommitmentDiscountId`; and lesson 09's sourced OpenCost gap is precisely the gap this schema exists to close — the three-way join (pod-resources × DCGM × billing) no incumbent tool performs. There is no lesson 11 in this module: this schema, the sourced OpenCost writeup from lesson 09, and the finished operator are the three artifacts of the **[GPU cost synthesis](../practice/gpu-cost-synthesis/README.md)** deliverable, which becomes the portfolio centerpiece you lead with in **Module 12** — the interview-readiness module this whole thread has been building toward.

## References & further reading

**Primary sources**
- FOCUS Spec CHANGELOG — the full authoritative version history, including 1.3 (December 2025, split-cost allocation) and 1.4 (June 2026, current; `BillingPeriod`/`InvoiceDetail`, `ProviderName`/`PublisherName` removed): https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/CHANGELOG.md
- FOCUS "Data Generator-Calculated Split Cost Allocation" feature — the shared-cost mechanism, explicitly scoped to shared compute nodes (Kubernetes, databases): https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/specification/supported_features/data_generator_calculated_split_cost_allocation.md
- FOCUS column docs — `AllocatedMethodId` (the split-methodology identifier) and its sibling files `allocatedresourceid.md`, `allocatedresourcename.md`, `allocatedmethoddetails.md`, `allocatedtags.md` in the same directory: https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/specification/datasets/cost_and_usage/columns/allocatedmethodid.md
- FOCUS custom-column-handling attribute — confirms the `x_` prefix rule for non-standard columns is still current in 1.4: https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/specification/attributes/custom_column_handling.md
- FOCUS Specification GitHub repository (`main` branch — current spec source): https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec
- FOCUS Specification official site — the published, versioned spec: https://focus.finops.org/

**Real-world engineering blogs**
- FinOps Foundation, FinOps X cloud-provider FOCUS adoption announcements (AWS Billing console export, GCP BigQuery/Looker template) — the multi-cloud business case for starting from FOCUS: https://www.finops.org/insights/finops-x-2025-cloud-announcements/
- OpenCost, "OpenCost Supports the FinOps Open Cost and Usage Specification (FOCUS)" — OpenCost's own adoption of FOCUS terminology, tying lesson 09 to lesson 10: https://opencost.io/blog/focus/
- Kubecost, GlobeNewswire press release (Oct 31, 2023) — dated anchor for when OpenCost/Kubecost/FOCUS convergence began: https://www.globenewswire.com/en/news-release/2023/10/31/2769876/0/en/Kubecost-Launches-Cloud-Cost-Optimization-and-Support-Through-OpenCost-and-the-FinOps-Foundation-s-FOCUS-Specification.html

**Deeper dives**
- OpenCost `Allocation`/`GPUAllocation` source (lesson 09) — the allocated-ledger basis this schema extends: https://github.com/opencost/opencost/blob/develop/core/pkg/opencost/allocation.go
- DCGM metrics reference (`DCGM_FI_PROF_GR_ENGINE_ACTIVE`) — the utilised-ledger source FOCUS still has no column for: https://docs.nvidia.com/datacenter/dcgm/latest/

---
Module backlink: [💰 11 — GPU cost and unit economics](../README.md)
