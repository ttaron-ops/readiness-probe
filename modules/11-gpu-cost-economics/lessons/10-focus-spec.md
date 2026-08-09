---
lesson: 10
title: "FOCUS 1.x and designing a GPU cost schema"
module: 11
concept: "FOCUS-aligned GPU cost schema"
status: not-started
est_time: "4 hrs"
artifacts: ["the module-capstone FOCUS-aligned GPU cost/usage schema (columns + types + provenance) carrying both ledgers, the attribution regime, and unit-cost joins, dropped into the module deliverable"]
---

# FOCUS 1.x and designing a GPU cost schema

## Why this matters
This is the module capstone. Everything upstream — the four regimes (01), the two ledgers (02), idle detection (03), unit economics (04), fragmentation (05), and the proof that OpenCost bills the allocated ledger and drops utilisation (09) — converges here into one artifact: **a schema.** When a hiring panel says "design a cost and usage schema for a shared GPU fleet," they are testing whether you know the industry standard (FOCUS), whether you know its edges, and whether you can extend it *precisely* where GPUs break it. The candidate who invents column names from scratch signals they've never shipped FinOps at scale; the candidate who says "start from FOCUS 1.2, then add exactly these three things it lacks for GPUs" signals they have.

FOCUS — the FinOps Open Cost & Usage Specification — matters because it is the answer to the landscape gap from lesson 09: it normalises cost and usage across AWS, Azure, GCP, OCI, and increasingly SaaS/on-prem into one column vocabulary, so your GPU cost table speaks the same language as the rest of finance's tooling. AWS, Azure, and GCP all ship FOCUS exports. If your GPU schema is FOCUS-aligned, it joins the corporate cost lake for free; if it's bespoke, it's a silo.

The stake is that FOCUS, as of 1.2, has **no native mechanism for splitting shared cost across tenants** and **no column for a utilised ledger.** Those are exactly the two things a shared-GPU-fleet schema must carry. Knowing where the standard stops — and extending it with disciplined `x_`-prefixed columns rather than abandoning it — is the whole skill. This lesson produces the schema that *is* the deliverable's centerpiece.

## What's new here
- **Already yours (skip):** the four regimes and the time-slice unsolvability proof (01); allocated-vs-utilised (02); cost-per-token / cost-per-run (07/08); the OpenCost allocation-basis defect (09). None re-taught — all become *columns*.
- **New angle 1:** the real FOCUS 1.x column vocabulary — exact names and definitions (`BilledCost`, `EffectiveCost`, `ContractedCost`, `ConsumedQuantity`, `ChargeCategory`, `CommitmentDiscountId`, `Tags`, …), not invented ones.
- **New angle 2:** how a GPU-hour maps onto a FOCUS charge row, where fractional (MIG) allocation maps cleanly, and where per-tenant time-slice sharing **still doesn't map** — the same lesson-01 hole, now expressed as "FOCUS has no split-cost primitive."
- **New angle 3:** the capstone schema — a FOCUS-conformant core plus `x_`-prefixed extensions carrying the utilised ledger, the attribution regime + basis, an uncertainty band, and joins to app counters for unit cost.

## Core notes

### The FOCUS column vocabulary you must know (1.1 / 1.2)
FOCUS is column-oriented: every charge is one row; every provider normalises to the same columns. The load-bearing set for GPU compute:

**Cost columns (the four-way cost comparison):**
| Column | Meaning |
|---|---|
| `BilledCost` | What was/will be invoiced for this row (post-discount, as billed). |
| `EffectiveCost` | Amortised real cost after all reduced rates, discounts, and the applicable portion of prepaid commitments. The one you allocate on. |
| `ListCost` | Cost at public list price (`ListUnitPrice` × `PricingQuantity`). |
| `ContractedCost` | Cost at negotiated/contracted price (`ContractedUnitPrice` × `PricingQuantity`). |

**Quantity & price:** `ConsumedQuantity` + `ConsumedUnit` (what was used — e.g. GPU-hours), `PricingQuantity` + `PricingUnit` (the unit the price is quoted in), `ListUnitPrice`, `ContractedUnitPrice`. The spec requires `ContractedUnitPrice × PricingQuantity == ContractedCost` (when non-null and `ChargeClass != "Correction"`).

**Accounts:** `BillingAccountId` / `BillingAccountName` / `BillingAccountType` (1.2), `SubAccountId` / `SubAccountName` / `SubAccountType` (1.2) — payer vs the linked account/project/subscription the GPU lives in.

**Resource:** `ResourceId`, `ResourceName`, `ResourceType` — the GPU node/instance (or, if you push it, the GPU/MIG UUID).

**Classification:** `ServiceCategory` (e.g. *Compute*, *AI and Machine Learning*), `ServiceSubcategory` (1.1), `ServiceName`; `ChargeCategory` (`Usage` | `Purchase` | `Tax` | `Credit` | `Adjustment`), `ChargeClass` (`Correction` or null), `ChargeDescription`, `ChargeFrequency`; `ChargePeriodStart` / `ChargePeriodEnd`, `BillingPeriodStart` / `BillingPeriodEnd`.

**Commitment discounts** (RIs / savings plans / CUDs — lesson 06): `CommitmentDiscountId`, `CommitmentDiscountName`, `CommitmentDiscountType`, `CommitmentDiscountCategory`, `CommitmentDiscountStatus` (`Used` | `Unused`), and (added 1.1) `CommitmentDiscountQuantity` + `CommitmentDiscountUnit`.

**SKU / location / provider:** `SkuId`, `SkuPriceId`, `SkuMeter` (1.1), `SkuPriceDetails` (1.1); `RegionId` / `RegionName` / `AvailabilityZone`; `ProviderName`, `PublisherName`, `InvoiceIssuerName`. `CapacityReservationId` / `CapacityReservationStatus` (1.1) — relevant to reserved GPU capacity.

**Tags:** `Tags` — a key/value map. **This is FOCUS's only allocation dimension.** Custom, non-standard properties anywhere in FOCUS must use an `x_` prefix (mandated since 1.1).

*Version notes:* 1.1 added capacity-reservation, commitment-discount quantity/unit, `ServiceSubcategory`, and the SKU meter/price-detail columns. 1.2 added dual-currency pricing (`PricingCurrency` + `PricingCurrency*` cost variants), `InvoiceId`, and account-type columns, and relaxed `CurrencyFormat` for virtual currencies. **Crucially: neither 1.1 nor 1.2 added any split-cost, shared-cost, or allocation mechanism** — allocation across tenants remains something you do *outside* the spec, expressed only through `Tags`.

### Mapping a GPU-hour onto a FOCUS row
- **Whole GPU (regime 1) — clean.** One row per GPU-node charge period: `ConsumedQuantity` = GPU-hours, `ConsumedUnit` = `GPU-hour`, `ContractedUnitPrice` = $/GPU-hr, `ContractedCost`/`EffectiveCost`/`BilledCost` filled from billing, `ServiceCategory` = *Compute*, `ChargeCategory` = `Usage`, `ResourceId` = node/instance, tenant in `Tags`. FOCUS handles this with room to spare.
- **MIG fractional (regime 2) — clean, via fractional quantity.** A `1g.10gb` instance over an hour is `ConsumedQuantity = 0.1429` GPU-hours (SM-slice basis, lesson 01) at the whole-GPU unit price, or `ResourceType = "MIG Instance"` with `ResourceId` = MIG UUID. A fractional `ConsumedQuantity` is perfectly legal FOCUS — this is where FOCUS is *strictly better than OpenCost* (lesson 09's 7× bug disappears because the quantity is 1/7, not 1). The catch: **FOCUS doesn't tell you how to compute 0.1429** — the fractional basis (SM vs memory vs blended) is your decision, out of spec.
- **Time-sliced (regime 3) — does NOT map cleanly.** To emit per-tenant rows you need a per-tenant `ConsumedQuantity`, i.e. a split of one device's GPU-hours across N contexts. Lesson 01 proved that split is *unrecoverable* from DCGM. FOCUS gives you no split-cost primitive to even declare "this is an allocated share of a shared parent" — the best you can do is emit N rows whose `ConsumedQuantity` sums to the device's GPU-hours, splitting by your chosen approximation (equal / request-weighted / context-runtime-weighted) and tagging tenant in `Tags`. **The number in `ConsumedQuantity` is then an estimate with an error band FOCUS has no column to carry.** This is the spec's edge, and it's the same lesson-01 hole. Articulating it — "FOCUS normalises the *shape* of the row but cannot manufacture the missing per-tenant counter, and has no allocation/uncertainty primitive" — is the senior signal.
- **The other missing leg — the utilised ledger.** Every FOCUS cost column is an *allocated/billed* number. There is **no FOCUS column for utilisation** (SM-active, lesson 05) or for a utilised-cost ledger (lesson 02). If your schema is to show "you paid for X GPU-hours and burned Y% of them," that lives in `x_` extensions.

### The capstone schema — FOCUS-conformant core + GPU extensions
One row per **(GPU-or-MIG UUID × tenant × charge period)**. Core columns are FOCUS 1.2-conformant so the table drops into any FOCUS lake; the `x_` block carries the two-ledger, attribution, and unit-economics data FOCUS lacks.

| Column | Type | FOCUS? | Produced from |
|---|---|---|---|
| `ChargePeriodStart` / `ChargePeriodEnd` | timestamp | core | metric/billing window |
| `BillingAccountId` / `SubAccountId` | string | core | cloud billing / cluster id |
| `ResourceId` | string | core | **pod-resources API** + DCGM → GPU or MIG UUID |
| `ResourceType` | string | core | `GPU` / `MIG Instance` (device-plugin) |
| `ServiceCategory` / `ServiceName` | string | core | billing (*Compute* / instance type) |
| `ChargeCategory` | enum | core | `Usage` |
| `ChargeClass` | enum, null | core | `Correction` on restatements |
| `ConsumedQuantity` | decimal | core | **allocated GPU-hours** — pod-resources × regime fraction |
| `ConsumedUnit` | string | core | `GPU-hour` |
| `PricingQuantity` / `PricingUnit` | decimal / string | core | billing |
| `ContractedUnitPrice` | decimal | core | pricing ($/GPU-hr; spot/on-demand/committed) |
| `ListCost` / `ContractedCost` / `EffectiveCost` / `BilledCost` | decimal | core | **billing — the allocated-ledger $** |
| `CommitmentDiscountId` | string, null | core | reservation/CUD coverage (lesson 06) |
| `Tags` | map | core | tenant / team / namespace / model → **allocation dimension** |
| `x_AttributionRegime` | enum | ext | device-plugin/pod-resources: `exclusive`\|`mig`\|`timeslice`\|`dra` (lesson 01) |
| `x_FractionalBasis` | enum, null | ext | `sm_slice`\|`memory`\|`blended`\|`request`\|`context_runtime` (lesson 01) |
| `x_AllocatedGPUHours` | decimal | ext | pod-resources — the reserved ledger (= `ConsumedQuantity`) |
| `x_UtilisationMean` | decimal 0–1 | ext | **DCGM `DCGM_FI_PROF_GR_ENGINE_ACTIVE`** (SM-active, lesson 05) |
| `x_UtilisedGPUHours` | decimal | ext | `x_AllocatedGPUHours × x_UtilisationMean` — the **utilised ledger** |
| `x_EffectiveUtilisedCost` | decimal | ext | `EffectiveCost × x_UtilisationMean` — utilised-cost $ FOCUS omits (lesson 02) |
| `x_AttributionUncertainty` | decimal | ext | regime-3 split error band (lesson 01); 0/null for exclusive & MIG |
| `x_AppTokens` | bigint, null | ext | inference server counter (lesson 07) |
| `x_AppRuns` | bigint, null | ext | orchestrator/job counter (lesson 08) |
| `x_UnitCostPerMTokens` | decimal, null | ext | `EffectiveCost / (x_AppTokens/1e6)` (lesson 07) |
| `x_UnitCostPerRun` | decimal, null | ext | `EffectiveCost / x_AppRuns` (lesson 08) |

**Why this shape wins:** the core row is what finance and the FOCUS lake consume (allocated ledger, standard columns, joinable across clouds). The `x_` block carries the three things FOCUS structurally can't: the **utilised ledger** (`x_Utilised*`), the **attribution basis** (`x_AttributionRegime` / `x_FractionalBasis`) so a reader knows whether the split is exact (exclusive/MIG) or estimated (timeslice), the **honest uncertainty** on the regime-3 rows, and the **unit-economics joins** to app counters. Three provenance streams feed it — **pod-resources API** (who was allocated what device, the regime), **DCGM** (utilisation, UUID), and **cloud billing / FOCUS export** (the dollars) — which is exactly the three-way join lesson 09 said no incumbent tool performs.

## Worked example
One H100 node, `EffectiveCost` $3.00/GPU-hr, over a 1-hour charge period. Three tenancy shapes, three sets of rows.

**(a) Exclusive — pod `train-A`.** One row: `ConsumedQuantity = 1.0` GPU-hour, `EffectiveCost = 3.00`, `x_AttributionRegime = exclusive`, `x_FractionalBasis = null`, `x_AllocatedGPUHours = 1.0`, `x_UtilisationMean = 0.40` (DCGM), `x_UtilisedGPUHours = 0.40`, `x_EffectiveUtilisedCost = 1.20`, `x_AttributionUncertainty = 0`, `Tags = {team: research}`. Both ledgers, one clean row.

**(b) MIG — 7× `1g.10gb`, seven tenants.** Seven rows, each `ConsumedQuantity = 0.1429` GPU-hour, `EffectiveCost = 0.4286`, `x_AttributionRegime = mig`, `x_FractionalBasis = sm_slice`, `x_AttributionUncertainty = 0`, `ResourceId` = each MIG UUID. The seven `EffectiveCost` values sum to $3.00 — **no 7× inflation** (contrast lesson 09's OpenCost, which bills $21). FOCUS's fractional `ConsumedQuantity` did the work.

**(c) Time-sliced — 4 pods, `replicas: 4`.** Four rows, `EffectiveCost` summing to $3.00, split by context-runtime weighting → `2.10 / 0.75 / 0.075 / 0.075`, `x_AttributionRegime = timeslice`, `x_FractionalBasis = context_runtime`. Because the split is an estimate (lesson 01), each row carries `x_AttributionUncertainty` = the spread to the equal-split alternative (up to ~10× on the small pods). `x_UtilisationMean` is the *device* aggregate — flagged, because it can't be resolved per tenant. **This is the row where you show the interviewer you know FOCUS's edge:** the schema still emits conformant rows, but the honesty lives entirely in the `x_` columns FOCUS doesn't define.

**Unit cost.** For the inference tenant serving 8.4M tokens on its `EffectiveCost` of $0.4286: `x_AppTokens = 8_400_000`, `x_UnitCostPerMTokens = 0.4286 / 8.4 = $0.051 / M-tokens` (lesson 07) — a business-legible number produced by joining the app counter onto the FOCUS row.

## Practice
Feed these into the module deliverable at [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md):
1. **Emit the schema.** Write the capstone table above as a concrete DDL (or a typed CSV/Parquet header) for your fleet, splitting core FOCUS columns from `x_` extensions. This is the deliverable's centerpiece.
2. **Three-regime rows.** Produce the (a)/(b)/(c) rows for a real or plausible fleet hour, showing that MIG rows sum without inflation and time-slice rows carry `x_AttributionUncertainty`.
3. **Provenance map.** For every column, name its source stream (pod-resources / DCGM / billing / app counter) — the three-way join lesson 09 said nobody does.
4. **The interview paragraph.** Write the answer to "design a cost/usage schema for a shared GPU fleet": start from FOCUS 1.2 as the industry standard, name the two things it lacks for GPUs (utilised ledger, split-cost/uncertainty primitive), and state the regime-3 limit.

## Self-check
- Why is a MIG instance easy to represent in FOCUS but a time-slice tenant is not? **Answer:** A MIG instance has a real per-instance counter and a defensible fraction (lesson 01), so it becomes a normal FOCUS row with a *fractional* `ConsumedQuantity` (e.g. 0.1429 GPU-hours) — fully in spec, and it fixes OpenCost's 7× whole-GPU bug. A time-slice tenant has no per-tenant counter (regime-3 proof), so its `ConsumedQuantity` is an *estimate* of a split FOCUS gives no primitive to declare, and the error band has no FOCUS column — you can emit conformant rows only by adding `x_AttributionUncertainty` and flagging the device-level `x_UtilisationMean`.
- What two things does FOCUS 1.2 structurally lack for a shared-GPU schema, and how do you add them without leaving the standard? **Answer:** (1) A **utilised ledger** — every FOCUS cost column is allocated/billed, none carries utilisation or utilised-cost; (2) a **split/shared-cost and uncertainty primitive** — 1.1/1.2 added no allocation mechanism, so cross-tenant sharing lives only in `Tags`. You add both as `x_`-prefixed columns (`x_UtilisationMean`, `x_UtilisedGPUHours`, `x_EffectiveUtilisedCost`, `x_AttributionRegime`, `x_AttributionUncertainty`), which is the FOCUS-sanctioned extension path (custom properties must use `x_`), keeping the core row conformant and lake-joinable.
- Which three data streams produce a row in the capstone schema, and why does that matter? **Answer:** The **pod-resources API** (device binding, sharing regime, allocated GPU-hours), **DCGM** (per-UUID utilisation, `x_UtilisationMean`), and **cloud billing / FOCUS export** (the `EffectiveCost`/`BilledCost` dollars), optionally joined to **app counters** (tokens/runs) for unit cost. It matters because lesson 09 showed no incumbent tool joins all three — OpenCost has $ + reported-but-unbilled utilisation but no counters, DCGM+Grafana has utilisation but no $, billing stops at the node — so this schema *is* the missing join, expressed in the industry-standard vocabulary.

## Resources
- FOCUS Specification v1.2 (current; column definitions): https://focus.finops.org/focus-specification/v1-2/
- FOCUS specification GitHub repo (working draft, per-column source, JSON schema): https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec
- FOCUS changelog (what 1.1 and 1.2 added; confirms no native split-cost mechanism): https://focus.finops.org/focus-specification/changelog/
- FinOps Foundation — FOCUS overview and adoption (AWS/Azure/GCP exports): https://www.finops.org/focus/
- OpenCost allocation source (the allocated-ledger basis this schema extends, lesson 09): https://github.com/opencost/opencost/blob/develop/core/pkg/opencost/allocation.go
- DCGM metrics (`DCGM_FI_PROF_GR_ENGINE_ACTIVE`, the utilised-ledger source): https://docs.nvidia.com/datacenter/dcgm/latest/

---
Module backlink: [💰 11 — GPU cost and unit economics](../README.md)
