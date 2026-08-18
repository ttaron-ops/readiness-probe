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
sources: 18
---

# FOCUS 1.x and designing a GPU cost schema

*Prev: [09 — Where existing tooling fails: reading the OpenCost source](09-existing-tooling-limits.md) · Next: none — this lesson is the module capstone*

## Where this fits

Lesson 09 proved from source where the leading open-source cost tool stops: a whole-GPU request count as the numerator, a per-physical-device price, DCGM utilisation captured and then not billed, and — under one common device-plugin configuration — time-sliced pods billing $0.00. That is a list of missing columns.

This lesson turns the list into a schema, which is the module's capstone artifact. Every prior lesson becomes a column here. The four sharing regimes (01) become an explicit dimension and a split methodology. The two ledgers (02) become the FOCUS cost columns plus the utilisation extension the standard does not carry. Idle detection (03) and fragmentation (04) become the unallocated rows. Unit economics (05) become the app-counter joins. Commitment strategy (06) becomes the commitment-discount columns and the `ContractCommitment` dataset. The neocloud/hyperscaler normalisation (07) becomes the provider-identity columns. Chargeback (08) becomes the split-cost rows and the remainder row. And lesson 09's gap becomes the reason this schema needs to exist at all rather than pointing at a dashboard.

There is no lesson 11. This schema, lesson 09's sourced teardown, and the finished `gpu-cost-operator` are the three artifacts of the module deliverable.

## Why this matters

When a hiring panel says *"design a cost and usage schema for a shared GPU fleet,"* they are running a three-part test. Do you know that an industry standard exists? Do you know it well enough to name real columns, their types, and their nullability rules? And can you say precisely where it still does not reach for GPUs — without over-claiming, which is the more common failure?

The candidate who invents column names has never shipped FinOps at scale. The candidate who says "start from FOCUS" and then cannot name a column has read a blog post. The candidate who says *"start from FOCUS 1.4, use the split-cost-allocation columns that landed in 1.3 — `AllocatedResourceId`, `AllocatedMethodId`, `AllocatedMethodDetails` with its `Elements` array whose `AllocatedRatio` values must sum to 1 — and then add exactly two `x_` extensions, because the standard still has no utilisation ledger and no confidence band on an estimated split"* has clearly read the specification. That last sentence is the deliverable of this lesson.

The business case is not philosophical. FOCUS is the FinOps Foundation's open specification for cost and usage data, and its job is normalisation: one column vocabulary that several providers' native billing exports map into, so a query written once works across all of them. A GPU cost table that speaks FOCUS joins the corporate cost lake finance already queries. A bespoke one is a silo somebody has to be taught, and silos are what get deprecated when headcount tightens.

And there is a specific trap worth naming up front, because it is where most write-ups on this subject are now wrong. The old, widely repeated claim — *"FOCUS has no way to represent shared or split cost, so cross-tenant GPU sharing can only be expressed through `Tags`"* — was true through FOCUS 1.2 and has been **false since 1.3** (December 2025). Repeating it in 2026 signals stale knowledge in exactly the area you claim as a specialism. The corrected claim is narrower, more useful, and more senior: **FOCUS now has a first-class split-cost primitive; what it still lacks is a utilisation ledger and any way to express how much to trust an estimated split.**

## What's new here (calibration)

- **Already yours (skip):** the four cost columns and why `EffectiveCost` is the allocation basis (lesson 02 §7); the `ContractCommitment` dataset's benefit-category / model / fulfilment-interval columns (lesson 06 §1); the chargeback design the schema serves (08); the tooling gap it closes (09). Referenced, not re-derived.
- **Genuinely new — the actual column definitions, reproduced.** Column ID, data type, column type, feature level, nullability, and version introduced, for every column a GPU-hour touches. Read from the specification repository, not from memory.
- **Genuinely new — the allowed-value enumerations.** `ChargeCategory`, `ServiceCategory`, `PricingCategory`, `ChargeClass`, `ChargeFrequency`, `CommitmentDiscountStatus`, `CapacityReservationStatus` — the real lists, so you never invent a value that fails validation.
- **Genuinely new — `UnitFormat`, and how to spell a GPU-hour.** The compound-unit rule that makes `GPU-Hours` correct and `gpu_hour` wrong, the allowed time-base units, and the fact that `Token` is on the spec's own recommended count-unit list — which matters for how you denominate the app-counter join.
- **Genuinely new and a correction — the real `AllocatedMethodDetails` object schema.** It is **not** a flat `{"AllocatedRatio": …}`. It is `{"Elements": [ {AllocatedRatio, UsageUnit, UsageQuantity}, … ]}`, the ratios across all allocated charges of one origin charge **must sum to 1**, and custom keys must be `x_PascalCase`. A previous version of this lesson had the flat form; that is fixed here.
- **Genuinely new — the conservation requirement as a normative rule.** `DataGeneratorCalculatedSplitCostAllocationHandling` requires that summable metrics sum across allocated charges to the origin charge. That is module 04's identity A, written into an industry standard.
- **Genuinely new — the unallocated-remainder rule.** `AllocatedResourceId` **MUST be null** on the row representing the unallocated portion of a split origin charge, which is how lesson 08's remainder gets a first-class row and a canonical query.
- **Genuinely new — version discipline, current.** 1.3 (Dec 2025) and 1.4 (June 2026) with their actual change lists, plus the fact that the spec repository's `main` branch currently carries the **1.5 candidate release**, which introduces `AllocatedServiceName` — so "which version" is a live question, not trivia.
- **Genuinely new — publishing the schema in FOCUS's own metadata format.** The `Schema` metadata collection with its `ColumnDefinition` list is how you declare your `x_` columns machine-readably, and almost nobody does it.
- **Genuinely new — the query pack.** Real SQL that reconciles to the invoice, produces the chargeback statement, isolates the remainder, and computes the waste gap and unit cost from one table.

## Core concepts

### 1. What FOCUS is, structurally

FOCUS is a **column specification**, not a database, an API, or a tool. It defines datasets (`CostAndUsage`, `ContractCommitment`, `BillingPeriod`, `InvoiceDetail`), and for each dataset a set of columns, each with a Column ID, a data type, a column type (`Dimension` or `Metric`), a feature level, a nullability rule, and normative `MUST`/`SHOULD`/`MAY` requirements in BCP-14 language. A conformant export is a table whose columns carry those IDs and obey those rules.

The value is entirely in the normalisation. Every provider bills differently — different line-item granularity, different names for the same idea, different treatment of prepaid commitments — and every one of those differences is a query rewrite for whoever consumes the data.

```
  FOCUS AS THE NORMALISATION LAYER
  ═══════════════════════════════════════════════════════════════════════════

   NATIVE BILLING EXPORTS                        each with its own schema,
   (all describing the same H100-hour)           granularity and vocabulary
   ┌───────────────────┐ ┌──────────────────┐ ┌──────────────────┐ ┌────────┐
   │ HYPERSCALER A     │ │ HYPERSCALER B    │ │ NEOCLOUD C       │ │ ON-PREM│
   │ line: instance-hr │ │ line: VM-hour    │ │ line: GPU-hour   │ │ depre- │
   │ 8 GPUs = ONE line │ │ reservation      │ │ (already per-GPU)│ │ ciation│
   │ savings-plan      │ │  applied inline  │ │ take-or-pay      │ │ + power│
   │  amortisation in  │ │ negotiated rate  │ │  minimum as a    │ │ + DC   │
   │  a separate file  │ │  in a 3rd table  │ │  separate invoice│ │ (L07)  │
   └─────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘ └───┬────┘
             │                    │                    │               │
             ▼                    ▼                    ▼               ▼
   ╔═══════════════════════════════════════════════════════════════════════╗
   ║                       F O C U S   1 . 4   R O W                       ║
   ║   BilledCost   EffectiveCost   ListCost   ContractedCost              ║
   ║   ConsumedQuantity/Unit        PricingQuantity/Unit                   ║
   ║   ChargeCategory  ChargeClass  ChargePeriodStart/End                  ║
   ║   ServiceProviderName  HostProviderName  InvoiceIssuerName            ║
   ║   ResourceId  ResourceType  Tags   CommitmentDiscount*                ║
   ║   AllocatedResourceId  AllocatedMethodId  AllocatedMethodDetails      ║
   ╚═══════════════════════════════════════════╤═══════════════════════════╝
                                               │
        ┌──────────────────────────────────────┴──────────────────────────┐
        │  ONE QUERY SURFACE                                              │
        │  "H100 EffectiveCost by team, last quarter, all providers"      │
        │  — written once, works across every source above.               │
        └─────────────────────────────────────────────────────────────────┘

   WHAT THE LAYER DOES NOT DO, and this is the GPU problem in one line:
   ┌─────────────────────────────────────────────────────────────────────┐
   │  It normalises the SELLER's view. Nothing above the line knows       │
   │  which of YOUR tenants was on the GPU — because the seller doesn't.  │
   │  Everything below the line is yours to produce: pod-resources for    │
   │  the split, DCGM for the utilisation, app counters for the unit.     │
   └─────────────────────────────────────────────────────────────────────┘
```

Two structural facts follow, and they shape the rest of the lesson.

**Fact one: the atom is a charge, not a resource.** One row is one charge for one thing over one charge period. A GPU node billed hourly produces one row per hour per charge type. This is why a shared GPU needs the split-cost machinery in §5 — one physical device, several tenants, and a row model that natively has one owner per row.

**Fact two: `x_` is a sanctioned extension path, not a hack.** The `CustomColumnHandling` attribute requires custom columns to carry an `x_` prefix "to identify it as an external custom column and to distinguish it from FOCUS columns to avoid conflicts in future releases," and recommends Pascal case after the prefix, alphanumeric characters only, an `Id` suffix for identifiers and a `Name` suffix for names, and no more than 50 characters. **Adding `x_` columns keeps you conformant.** Inventing a bare `utilisation` column does not.

### 2. Version discipline: why "which version" is a real question

FOCUS classifies every change to the specification under one of three **Change Impact Classifications**, and the classification tells a consumer what breaks:

| Classification | What it means for your pipeline |
|---|---|
| **Compatible** | new columns/datasets arrive additively and nullable; existing queries keep working untouched |
| **Migration Compatible** | something moved or changed shape; existing queries need a documented, bounded edit |
| **Incompatible** | a genuine break. **1.3 and 1.4 both list "None" here** |

Each column also carries a **Feature level**, derived mechanically from its normative language: `MUST` with no condition → **Mandatory**; `MUST` with a stated condition → **Conditional**; `SHOULD` → **Recommended**; `MAY` → **Optional**. So the feature level is not an editorial judgement, it is a summary of the requirement text — which means you can predict it, and you can read it as "how hard will a validator complain."

The two versions that matter, from the specification's own changelog:

**FOCUS 1.3 — announced December 2025.** Compatible additions: the columns supporting **Data Generator-Calculated Split Cost Allocation**, the whole **`ContractCommitment` dataset**, the **Participating Entity Identification** columns, and new **Dataset Instance** and **Recency** metadata. Eight new `CostAndUsage` columns: `AllocatedMethodDetails`, `AllocatedMethodId`, `AllocatedResourceId`, `AllocatedResourceName`, `AllocatedTags`, `ContractApplied`, `HostProviderName`, `ServiceProviderName`. One Migration Compatible change: the replacement of the `Provider`/`Publisher` columns, which deprecated `ProviderName` and `PublisherName` across nine dependent features. Incompatible changes: none.

**FOCUS 1.4 — announced June 2026, the current ratified release.** Adds two datasets (`BillingPeriod`, `InvoiceDetail`), 47 new columns, 6 new attributes, 2 new supported features (Invoice Reconciliation, Commitment Program Eligibility Details), and 17 new glossary entries. Migration Compatible changes worth knowing: `ContractApplied` changed to the JSON Object Schema format (so any query parsing it must be updated); the `ColumnHandling`, `DiscountHandling` and `InvoiceHandling` attributes were removed with their requirements redistributed; and **`ProviderName`/`PublisherName` were removed entirely** — deprecated in 1.3, gone in 1.4, a full release of runway. `InvoiceId` was tightened from Recommended to Conditional. Incompatible changes: none.

**And the live edge:** the specification repository's `main` branch currently carries **publication version 1.5** as a candidate release. Its most relevant addition for this schema is **`AllocatedServiceName`** (Version Introduced: 1.5), which closes a gap in the 1.3 split-cost design described in §5. So when you present this schema, say which version you targeted and check the changelog before you do — "FOCUS 1.4, with a note on what 1.5 changes for split cost" is a materially stronger answer than "FOCUS."

**The transferable point.** A candidate who cites `ProviderName` in 2026 is citing a column that no longer exists in the specification they claim to know. A candidate who says "deprecated in 1.3, removed in 1.4, use `ServiceProviderName` / `HostProviderName` / `InvoiceIssuerName` instead depending on which entity you mean" has read the changelog. That is a two-minute investment with a large signal payoff, and the same discipline should govern your own schema: additive nullable extensions, a documented deprecation window, and a version stamp on every export.

### 3. The column vocabulary a GPU-hour touches

Reproduced from the column definition files in the specification repository (`specification/datasets/cost_and_usage/columns/`, read at `main` on 2026-08-17). Column type is `Dimension` (something you group by) or `Metric` (something you sum).

**Cost — the four-way comparison.** All four are Mandatory, `Decimal`, `Metric`, non-nullable, and accept any valid decimal value (including negatives, for credits and corrections):

| Column ID | Ver | The spec's definition, condensed | Use for a GPU-hour |
|---|---|---|---|
| `BilledCost` | 0.5 | cost as invoiced by the invoice issuer in a given billing period | the cash view. **Reads 0 for hours fully covered by a prepaid commitment** — which makes a committed fleet look free one month and catastrophic the next |
| `EffectiveCost` | 0.5 | cost based on resources/services used or contract commitments *recognised* in a given charge period | **the accrual view, and the column you allocate on.** Spreads prepaid commitments across the hours they actually cover |
| `ListCost` | 1.0-preview | `ListUnitPrice × PricingQuantity` | the public sticker. Only useful as the baseline you measure savings against |
| `ContractedCost` | 1.0 | `ContractedUnitPrice × PricingQuantity` | your negotiated rate before commitment discounts. Defaults to `ListCost` when no negotiated discount applies |

**Quantity and price — two quantities, and the distinction is exactly ours:**

| Column ID | Type | Col type | Level | Null | Ver | Meaning |
|---|---|---|---|---|---|---|
| `ConsumedQuantity` | Decimal | Metric | Conditional | yes | 1.0 | volume of the metered SKU **used**, in `ConsumedUnit`. This is where fractional GPU-hours live |
| `ConsumedUnit` | String | Dimension | Conditional | yes | 1.0 | the provider's measurement unit; `UnitFormat` recommended (see §4) |
| `PricingQuantity` | Decimal | Metric | Mandatory | yes | 1.0-preview | the volume the provider **rates and prices**, after block-pricing rules |
| `PricingUnit` | String | Dimension | Mandatory | yes | 1.0-preview | the unit the price is quoted in; `UnitFormat` required |
| `ListUnitPrice` | Decimal | Metric | Conditional | yes | 1.0-preview | published price per `PricingUnit`, non-negative |
| `ContractedUnitPrice` | Decimal | Metric | Conditional | yes | 1.0 | agreed price per `PricingUnit`, inclusive of negotiated discounts, **exclusive** of commitment discounts |

The `Consumed` / `Pricing` split is not pedantry. For a GPU instance the provider prices **instance-hours** (`PricingQuantity = 1`, `PricingUnit = "Hours"`) while what you consumed is **GPU-hours** (`ConsumedQuantity = 8`, `ConsumedUnit = "GPU-Hours"` for an 8-GPU node). Lesson 01's rung 2 — divide the node price by the *physical* GPU count — is precisely the mapping between these two columns.

**Classification and time:**

| Column ID | Type | Col type | Level | Null | Ver | Notes |
|---|---|---|---|---|---|---|
| `ChargeCategory` | String | Dimension | Mandatory | no | 0.5 | allowed values only — §4 |
| `ChargeClass` | String | Dimension | Mandatory | yes | 1.0 | only allowed value is `Correction`; null otherwise |
| `ChargeFrequency` | String | Dimension | Recommended | no | 1.0-preview | `One-Time` \| `Recurring` \| `Usage-Based` |
| `ChargeDescription` | String | Dimension | — | — | 0.5 | human-readable line description |
| `ChargePeriodStart` / `ChargePeriodEnd` | Date/Time | Dimension | Mandatory | no | 0.5 | the charge window — your metric window must align to it |
| `BillingPeriodStart` / `BillingPeriodEnd` | Date/Time | Dimension | Mandatory | no | 0.5 | the invoice window |
| `BillingCurrency` | String | Dimension | Mandatory | no | 0.5 | ISO currency of the cost columns |
| `ServiceCategory` | String | Dimension | Mandatory | no | 0.5 | allowed values only — §4. `Compute` or `AI and Machine Learning` |
| `ServiceSubcategory` | String | Dimension | Recommended | no | 1.1 | second-level classification |
| `ServiceName` | String | Dimension | Mandatory | no | 0.5 | the service, e.g. the instance family |

**Accounts, resource and location:**

| Column ID | Type | Level | Null | Ver | Notes |
|---|---|---|---|---|---|
| `BillingAccountId` / `Name` / `Type` | String | Mandatory | no | 0.5 | the payer |
| `SubAccountId` / `Name` / `Type` | String | Conditional | yes | 0.5 | the linked account / project / subscription the GPU node lives in |
| `ResourceId` | String | Conditional | yes | 0.5 | provider's resource identifier. **For a split, this is the parent** |
| `ResourceName` | String | Conditional | yes | 0.5 | display name |
| `ResourceType` | String | Conditional | yes | 1.0-preview | free-form string; no allowed-value list. Use it for `GPU` / `MIG Instance` |
| `RegionId` | String | Conditional | yes | 1.0 | region |
| `AvailabilityZone` | String | Recommended | yes | 0.5 | AZ — matters for GPU capacity, which is AZ-scoped |
| `SkuId` / `SkuPriceId` | String | Conditional | yes | 1.0-preview | the SKU and the specific price record |
| `SkuMeter` | String | Conditional | yes | 1.1 | what the SKU meters |
| `Tags` | JSON | Conditional | yes | 1.0-preview | key/value map; the general-purpose allocation dimension |

**Provider identity — the 1.3 replacement set.** Do not use `ProviderName` or `PublisherName`; they were removed in 1.4.

| Column ID | Level | Null | Ver | Which entity it names |
|---|---|---|---|---|
| `ServiceProviderName` | Mandatory | **no** | 1.3 | who made the service available — produced or sold it, directly or via a reseller |
| `HostProviderName` | Mandatory | **yes** | 1.3 | who runs the underlying infrastructure, when that differs from the service provider |
| `InvoiceIssuerName` | Mandatory | no | 0.5 | who issued the invoice — the marketplace or reseller case |

This triple is exactly lesson 07's normalisation problem given column names. A managed inference platform's `ServiceProviderName` is the platform; its `HostProviderName` is whichever neocloud actually owns the H100s; and `InvoiceIssuerName` may be a marketplace. Comparing "neocloud pricing" against "hyperscaler pricing" without distinguishing these three is how you end up comparing a reseller's margin to a bare-metal rate.

**Commitments and reservations (lesson 06):**

| Column ID | Level | Null | Ver | Notes |
|---|---|---|---|---|
| `CommitmentDiscountId` / `Name` / `Type` | Conditional | yes | 1.0-preview | which commitment covered this hour |
| `CommitmentDiscountCategory` | Conditional | yes | 1.0-preview | `Spend` \| `Usage` |
| `CommitmentDiscountStatus` | Conditional | yes | 1.0 | `Used` \| `Unused` — **`Unused` rows are the waste rows** |
| `CommitmentDiscountQuantity` / `Unit` | Conditional | yes | 1.1 | how much of the commitment this row drew down |
| `CapacityReservationId` | Conditional | yes | 1.1 | the capacity-holding instrument, tracked **separately** from the discount |
| `CapacityReservationStatus` | Conditional | yes | 1.1 | `Used` \| `Unused` |
| `PricingCategory` | Conditional | yes | 1.0-preview | `Standard` \| `Dynamic` \| `Committed` \| `Other` — spot lives in `Dynamic` |

Note the deliberate separation of `CommitmentDiscount*` from `CapacityReservation*`, with independent `Used`/`Unused` accountings. That is the standard encoding lesson 06's point that on a GPU fleet you can hold hardware without a discount and hold a discount without hardware.

**Split cost allocation (1.3, plus one 1.5 addition) — the subject of §5:**

| Column ID | Type | Col type | Level | Null | Ver |
|---|---|---|---|---|---|
| `AllocatedResourceId` | String | Dimension | Conditional | yes | 1.3 |
| `AllocatedResourceName` | String | Dimension | Conditional | yes | 1.3 |
| `AllocatedMethodId` | String | Dimension | Conditional | yes | 1.3 |
| `AllocatedMethodDetails` | JSON | Dimension / Metric | Recommended | yes | 1.3 |
| `AllocatedTags` | JSON | Dimension | Conditional | yes | 1.3 |
| `AllocatedServiceName` | String | Dimension | Conditional | yes | **1.5** |

### 4. The enumerations you must not invent, and how to spell a GPU-hour

Half of "knowing the spec" is knowing which columns are closed vocabularies. Getting one wrong fails validation and, worse, silently breaks a downstream filter.

**`ChargeCategory`** — the highest-level classification of a charge by how it is billed:

| Value | Meaning |
|---|---|
| `Usage` | charges based on the quantity of a service or resource consumed over time, including refunds |
| `Purchase` | charges for acquiring a service or resource, upfront or recurring, including refunds |
| `Tax` | applicable taxes levied by the relevant authorities |
| `Credit` | charges granted by the provider — promotional credits and their corrections |
| `Adjustment` | charges the provider applies that fall into no other category |

A GPU-hour is `Usage`. A prepaid Capacity Block is `Purchase`. A chargeback credit from lesson 08's dispute protocol is `Adjustment`.

**`ChargeClass`** — a single allowed value, `Correction`, meaning "a correction to a previously closed billing period." Null on every ordinary row. This is the column lesson 08's restatement rule is written against.

**`ChargeFrequency`** — `One-Time` (never repeats), `Recurring` (repeats on a cadence regardless of use), `Usage-Based` (repeats every time the service is used, and is not recorded when it is not). A GPU-hour is `Usage-Based`; a monthly take-or-pay minimum is `Recurring`.

**`ServiceCategory`** — 19 allowed values. The two that matter here: **`Compute`** ("virtual, containerized, serverless, or high-performance computing infrastructure and services") and **`AI and Machine Learning`** ("artificial intelligence and machine learning related technologies"). Raw GPU instances are `Compute`; a managed inference or training service is `AI and Machine Learning`. *Note a wrinkle: the specification's own contract-commitment appendix example filters on the string `"AI/ML"` inside an applicability JSON — a free-form filter value, not the enumerated `ServiceCategory`. Do not copy example strings into an enumerated column.*

**`PricingCategory`** — `Standard` (the agreed rate for the billing account, including negotiated discounts and tiering, excluding commitment discounts), `Dynamic` (a variable rate the provider can change without notice — the spec names "interruptible or low priority resources," so **spot lives here**), `Committed` (reduced pricing from the commitment named in `CommitmentDiscountId`), `Other`.

**`CommitmentDiscountStatus`** — `Used` (charges that drew down a commitment) and `Unused` (charges representing the unused portion). **`CapacityReservationStatus`** takes the same two values for the reservation instrument. Realised commitment utilisation is `SUM(Used) / (SUM(Used) + SUM(Unused))`, which is the single query lesson 06's coverage model is graded by.

**`UnitFormat` — how to spell a GPU-hour.** This is a genuine source of failed conformance and it takes two minutes to get right. The rules that matter:

- Compound units join base units with a **hyphen**, in `<singular>-<plural>` form: `GB-Hours`, `MB-Days`, `Request-Tokens`. So a GPU-hour is **`GPU-Hours`** — not `gpu_hour`, not `GPU hours`, not `GPUHr`.
- Ratio units use a **slash**: `GB/Hour`, `Core-Hours/Day`, `Units/3 Months`.
- Allowed time base units (singular/plural): `Year`, `Month`, `Day`, `Hour`, `Minute`, `Second`, `Millisecond`, `Microsecond`. Nothing else.
- Recommended count base units include `Count`, `Unit`, `Request`, **`Token`**, `Connection`, `Core`, `Job`, `Session`, and others; a base unit not on the list may be used **as long as it is capitalised**. `GPU` qualifies as such a capitalised noun.
- A unit quantity may prefix a simple unit: **`1000 Tokens`**.

Two consequences for this schema. First, `ConsumedUnit = "GPU-Hours"` is well-formed and is in fact the unit the specification itself uses in its `ContractCommitment` appendix example for an H100 reservation. Second, the app-counter join in §7 should denominate tokens as `Tokens` or `1000 Tokens` rather than inventing `MTok` — the standard already has an opinion about this, and following it is free.

### 5. Split cost allocation, in full

This is the mechanism that lets one shared GPU's cost be divided across tenants inside a conformant dataset, and it is the correction that changes this lesson's thesis. Read it precisely.

**The feature.** "Data Generator-Calculated Split Cost Allocation," added in 1.3, described by the specification as tracking resources split by internal consumption metrics, "most common for resources supporting shared usage like compute nodes in a shared cluster (Kubernetes, databases) or storage engines that can share capacity between workloads." That is the MIG and time-slicing case, named.

**The vocabulary.** The specification's glossary defines an **origin charge** (the charge being split), an **allocated charge** ("the charge that was created as the result of an allocation operation"), and an **allocated method** ("the process or formula by which cost is being allocated from an origin charge to produce allocated charges… which requires documentation of the method to be provided for any and all allocated methods used").

**The row structure and the conservation rule:**

```
  SPLIT COST ALLOCATION — one shared GPU, one charge period
  ═══════════════════════════════════════════════════════════════════════════

   ORIGIN CHARGE                          ┌─────────────────────────────────┐
   ResourceId          = GPU-abc123…      │ DIMENSIONS on allocated charges │
   ServiceName         = "H100 Instance"  │ MUST MATCH the origin charge.   │
   ServiceCategory     = Compute          │ NON-SUMMABLE METRICS (unit      │
   AllocatedResourceId = NULL             │ prices) MUST MATCH the origin.  │
   AllocatedMethodId   = NULL             │ SUMMABLE METRICS (costs,        │
   EffectiveCost       = $2.10            │ quantities) MUST SUM to the     │
        │                                 │ origin charge's value.          │
        │  split by AllocatedMethodId     └─────────────────────────────────┘
        │  = "sm_slice"
        ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │ ALLOCATED CHARGES — one per consuming tenant                         │
   │                                                                      │
   │  ResourceId = GPU-abc123…   ◀── UNCHANGED. Still the parent.        │
   │  AllocatedResourceId = MIG-GPU-abc123…/1/0   ◀── the child          │
   │  AllocatedResourceName = "team-a inference slice"                    │
   │  AllocatedServiceName = "Inference Platform"   (1.5; see below)      │
   │  AllocatedMethodId   = "sm_slice"    ← YOUR id, YOUR documentation   │
   │  AllocatedMethodDetails = {"Elements":[{                             │
   │        "AllocatedRatio": 0.142857,                                   │
   │        "UsageUnit": "GPU",                                           │
   │        "UsageQuantity": 1 }]}                                        │
   │  AllocatedTags = {"team":"a"}   ← child tags NOT already in Tags     │
   │  EffectiveCost = $0.30                                               │
   │                                                                      │
   │  × 7 tenants …                                                       │
   └──────────────────────────────────────────────────────────────────────┘
        │
        ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │ THE UNALLOCATED REMAINDER — lesson 08's remainder, as a real row     │
   │  ResourceId          = GPU-abc123…                                   │
   │  AllocatedMethodId   = "sm_slice"      ◀── NOT null: it IS part of   │
   │                                             the split                │
   │  AllocatedResourceId = NULL            ◀── MUST be null on the       │
   │                                             unallocated portion      │
   │  EffectiveCost       = whatever nobody held                          │
   └──────────────────────────────────────────────────────────────────────┘

   ┌──────────────────────────────────────────────────────────────────────┐
   │  CONSERVATION:  Σ EffectiveCost(allocated) + unallocated ≡ origin.   │
   │  Σ AllocatedRatio across all allocated charges of one origin ≡ 1.    │
   │  ▸ THIS IS MODULE 04'S IDENTITY A, AS A NORMATIVE SPEC REQUIREMENT. │
   └──────────────────────────────────────────────────────────────────────┘
```

**Column-by-column, with the nullability rules that a validator checks:**

- **`AllocatedResourceId`** (String, Dimension, Conditional, nullable, 1.3). MUST be null when a charge is not related to a split; **MUST be null when a charge represents the unallocated portion** of the origin charge after split cost allocation; MUST NOT be null when it represents an allocated portion. SHOULD be locally unique within the associated `ResourceId` and `ChargePeriod`, and MAY NOT be unique across them. → For MIG, this is the MIG instance UUID. For time-slicing, a per-context or per-pod identifier.
- **`AllocatedResourceName`** (String, Dimension, Conditional, nullable, 1.3). The display name of the child.
- **`AllocatedMethodId`** (String, Dimension, Conditional, nullable, 1.3). "The unique identifier for the allocated method defined by the service provider… can be used to find how the allocated charge was calculated in the provider's documentation." MUST be null when the charge is not part of a split; MUST NOT be null when it is; and **method documentation MUST reference a single `AllocatedMethodId` value.** → FOCUS deliberately does **not** enumerate methodologies. You choose the identifier — `sm_slice`, `memory_fraction`, `context_runtime`, `request_weighted` — and you owe the world a document explaining each one. That obligation is the standard's way of making an arbitrary choice auditable instead of hidden.
- **`AllocatedMethodDetails`** (JSON, Dimension/Metric, **Recommended**, nullable, 1.3). MUST be null when the charge is not part of a split; SHOULD NOT be null when it is. **The object schema — this is the correction:**

```json
{
  "Elements": [
    { "AllocatedRatio": 0.05, "UsageUnit": "CPU",    "UsageQuantity": 0.5 },
    { "AllocatedRatio": 0.10, "UsageUnit": "Memory", "UsageQuantity": 4   }
  ]
}
```

  A top-level `Elements` array (Required) of entry objects. Each entry: `AllocatedRatio` (Numeric, **Required**) — "the percentage of the origin charge that this allocated charge derived from"; `UsageUnit` (String, Conditional — required if `UsageQuantity` is present) — the aspect of the documented method used to calculate the ratio; `UsageQuantity` (Numeric, optional) — the volume consumed in that unit. Normative constraints: **the sum of `AllocatedRatio` across all allocated charges related to a single origin charge MUST equal 1 (100%)**, `UsageUnit` SHOULD conform to `UnitFormat`, and custom properties — at the top level or inside an `Elements` entry — must be prefixed `x_` followed by PascalCase.

  *Correction noted:* a previous version of this lesson gave the object as a flat `{"AllocatedRatio": …, "UsageUnit": …, "UsageQuantity": …}`. It is not flat; it is `Elements`-wrapped, and the array exists precisely so a split can be justified by **several** dimensions at once. That matters for GPUs: a MIG split can carry both its compute-slice and its memory-slice basis as two elements, which is the honest representation of lesson 01's two-basis problem.
- **`AllocatedTags`** (JSON, Dimension, Conditional, nullable, 1.3). MUST be null when the charge is not part of a split. When present it **MUST NOT include resource tags already in `Tags`**, and MUST include all applicable user- and generator-defined tags for the `AllocatedResourceId`. Valueless tag keys take boolean `true`. → the per-tenant identity riding with the child, without duplicating the parent's tags.
- **`AllocatedServiceName`** (String, Dimension, Conditional, nullable, **1.5**). MUST be null when `AllocatedResourceId` is null and MUST NOT be null when it isn't. It exists because the handling attribute **requires the origin `ServiceName` and `ServiceCategory` to be preserved on allocated rows** so the dataset still reconciles to the invoice on the origin service — which has the side effect that the *consuming* service's identity has nowhere to go. Before 1.5 you inferred it by joining `AllocatedResourceId` to a mapping you maintained yourself.

**The handling attribute, which is the part to quote.** `DataGeneratorCalculatedSplitCostAllocationHandling` (1.3) states three requirements for columns in a split:

1. A dimension column MUST match the corresponding value in the origin charge when present on an allocated charge.
2. A non-summable metric (e.g. unit prices) MUST match the origin charge's value.
3. **The sum of a summable metric (e.g. costs and quantities) across allocated charges MUST match the origin charge's value.**

Requirement 3 is module 04's identity A, promoted to a normative requirement of an industry specification. When you say "the statement must close to the cent," you are quoting a spec. And it is the requirement OpenCost's MIG behaviour violates by a factor of 7 (lesson 09) — the tool emits seven whole-GPU charges against one physical GPU with no origin/allocated relationship at all, which is not a conformant split, it is seven unrelated charges.

**What the spec does *not* prescribe.** It does not say whether the origin row should be zeroed, omitted, or paired with a separate offsetting row — only that the allocated charges sum to the origin total. The specification's own worked example zeroes the origin row: a shared cluster `cluster-shared-01` with a $100.00 origin `EffectiveCost` split by measured vCPU-hours 40/35/25 emits four rows — origin at $0.00, plus three allocated rows at $40.00 / $35.00 / $25.00, all four carrying the same `ResourceId`, `ServiceName` and `ServiceCategory`. **Pick one convention, document it, and never mix them in one dataset**, because a consumer summing `EffectiveCost` without filtering will double-count if you do.

### 6. Mapping a GPU-hour onto rows, per regime

Now apply §5 to lesson 01's four sharing regimes. This is the section that turns a standard into an implementation.

```
  REGIME → ROW SHAPE  (one physical H100, one hour, $2.10 EffectiveCost)
  ══════════════════════════════════════════════════════════════════════════

   REGIME 1 · EXCLUSIVE                        rows: 1
   ┌────────────────────────────────────────────────────────────────────┐
   │ ResourceId = GPU-abc…      Allocated* = ALL NULL                   │
   │ ConsumedQuantity = 1.0     ConsumedUnit = "GPU-Hours"              │
   │ EffectiveCost = 2.10       Tags = {"team":"research"}              │
   │ x_AttributionRegime = "exclusive"   x_SplitConfidence = "exact"    │
   │ ▸ nothing is split, so no Allocated* columns. Clean, conformant.   │
   └────────────────────────────────────────────────────────────────────┘

   REGIME 2 · MIG 7 × 1g.10gb                  rows: 1 origin + 7 allocated
   ┌────────────────────────────────────────────────────────────────────┐
   │ origin:   ResourceId = GPU-abc…   EffectiveCost = 0.00 (zeroed)    │
   │           AllocatedResourceId = NULL   AllocatedMethodId = NULL    │
   │ child ×7: AllocatedResourceId = MIG-GPU-abc…/<gi>/<ci>             │
   │           AllocatedMethodId   = "sm_slice"                         │
   │           AllocatedMethodDetails = {"Elements":[                   │
   │              {"AllocatedRatio":0.142857,"UsageUnit":"GPU",         │
   │               "UsageQuantity":1}]}                                 │
   │           ConsumedQuantity = 0.142857   EffectiveCost = 0.30       │
   │           x_AttributionRegime = "mig"  x_SplitConfidence="exact"   │
   │ ▸ Σ ratios = 7 × 0.142857 ≈ 1.0 ✓   Σ cost = 7 × 0.30 = 2.10 ✓    │
   │ ▸ FOCUS is STRICTLY BETTER than lesson 09's tool here: the 7×      │
   │   inflation is impossible, because conservation is normative.      │
   │ ▸ FOCUS does NOT tell you 0.142857 is right — SM-slice vs memory   │
   │   basis is YOUR call, documented behind "sm_slice" (lesson 01 §4). │
   └────────────────────────────────────────────────────────────────────┘

   REGIME 3 · TIME-SLICED, 4 sharers            rows: 1 origin + 4 allocated
   ┌────────────────────────────────────────────────────────────────────┐
   │ child ×4: AllocatedResourceId = GPU-abc…::0 … ::3                  │
   │           AllocatedMethodId   = "context_runtime"                  │
   │           AllocatedMethodDetails = {"Elements":[                   │
   │              {"AllocatedRatio":0.70,"UsageUnit":"Seconds",         │
   │               "UsageQuantity":2520}], "x_EstimatorId":"nvml_pid"}  │
   │           EffectiveCost = 1.47 / 0.525 / 0.0525 / 0.0525           │
   │           x_AttributionRegime = "timeslice"                        │
   │           x_SplitConfidence  = "estimated"                         │
   │           x_SplitUncertainty = 0.90     ◀── THE COLUMN FOCUS LACKS │
   │ ▸ Fully conformant. Sums to 2.10. Ratios sum to 1.                 │
   │ ▸ AND THE RATIO IS A GUESS (lesson 01 §5): one device counter,     │
   │   N resident contexts, non-invertible. FOCUS gives you a place to  │
   │   name the METHOD and carry the RATIO — and no column anywhere     │
   │   for HOW MUCH TO TRUST IT. That is gap #2 in §7.                  │
   └────────────────────────────────────────────────────────────────────┘

   REGIME 4 · DRA (ResourceClaim)               rows: as the claim resolved
   ┌────────────────────────────────────────────────────────────────────┐
   │ ▸ DRA changes the ALLOCATION RECORD, not the counter. A claim      │
   │   resolves to a whole device, a partition, or a shared device —    │
   │   so the row shape is whichever of regimes 1–3 it landed on.       │
   │ ▸ What DRA adds is a BETTER AllocatedResourceId: the ResourceClaim │
   │   allocation result names the device and the request by name, so   │
   │   the child identifier is a first-class API object rather than a   │
   │   plugin-fabricated "::N" string.                                  │
   │ ▸ x_AttributionRegime = "dra"; x_SplitConfidence inherits from the │
   │   underlying regime. Do NOT let "dra" hide which one it resolved   │
   │   to — carry both.                                                 │
   └────────────────────────────────────────────────────────────────────┘
```

The point to take from regime 4: an attribution-regime dimension with four mutually exclusive values is *wrong* for DRA, because DRA is an allocation mechanism layered over the same three hardware regimes. Carry the mechanism and the hardware regime as separate fields, or you will find yourself unable to answer "how confident is this row" for every DRA row in the fleet.

### 7. Exactly what FOCUS still lacks — two gaps and one scope boundary

Scope this narrowly. Over-claiming here is the most common way to sound like you have not read the current spec.

**Gap 1 — there is no utilisation ledger, anywhere in FOCUS.** Every cost and quantity column in the standard, including the new `Allocated*` ones, is an **allocated or billed** number. `ConsumedQuantity` is the closest candidate and it is still *provider-metered* — the GPU-hours the provider handed you, not a measurement of whether warps were resident on the SMs. There is no column for a duty cycle, no column for a utilised-cost figure, and nothing in `AllocatedMethodDetails` that changes this: `UsageQuantity` is the consumption metric that *drove a split ratio*, not a measure of device work. So the entire allocated-versus-utilised distinction from lesson 02 — the single most important idea in GPU cost — has no home in core FOCUS and must live in `x_` columns.

**Gap 2 — there is no confidence or uncertainty on a declared split.** `AllocatedMethodId` records *which* method was used and `AllocatedMethodDetails` can carry a ratio and the usage that produced it, but nothing carries an error band. For MIG that is fine and worth saying: the SM-slice fraction is a physically metered partition, so there is nothing to be uncertain about, and `x_SplitConfidence = "exact"` is the honest value. For time-slicing it is not fine: lesson 01's argument stands unchanged — the device exposes one counter, not one per CUDA context — so the ratio is the best available estimate and a downstream reader has no in-spec way to learn that. **This is the sharper, more senior version of the old claim.** The story is no longer "FOCUS cannot represent shared GPU cost"; it is "FOCUS names the methodology and still cannot carry the confidence."

**Not a gap — a scope boundary: application counters.** Tokens, runs, and experiments are not missing from FOCUS; they are *outside* it. FOCUS describes cost and usage as billed by a provider, and how many tokens your inference server emitted is not something any provider bills you for on a GPU rental. Say this explicitly when presenting the schema, because it is the difference between "the standard has a hole" and "I extended the standard at its boundary." (The `UnitFormat` recommended count units include `Token`, which is the specification acknowledging the *unit* exists without claiming the *counter* is in scope.)

Everything else a shared-GPU schema needs is core FOCUS as of 1.3 — including the split-cost mechanism itself. Two extensions, one boundary. That is the whole answer.

### 8. The capstone schema

**Grain: one row per (measurement identity × tenant × charge period)**, where a measurement identity is a physical GPU, a MIG instance, or a per-context share. Core columns are FOCUS 1.4-conformant so the table drops into any current FOCUS lake; the `x_` block carries only the two gaps and the out-of-scope joins.

| Column | Type | FOCUS? | Produced from |
|---|---|---|---|
| `ChargePeriodStart` / `ChargePeriodEnd` | timestamp | core, Mandatory | the metric window, aligned to the billing window |
| `BillingPeriodStart` / `BillingPeriodEnd` | timestamp | core, Mandatory | billing export |
| `BillingCurrency` | string | core, Mandatory | billing export |
| `BillingAccountId` / `SubAccountId` | string | core | billing export / cluster identity |
| `ServiceProviderName` | string | core (1.3), Mandatory | who sold the capacity (lesson 07) |
| `HostProviderName` | string, null | core (1.3) | who runs the hardware, when it differs |
| `InvoiceIssuerName` | string | core, Mandatory | marketplace/reseller case |
| `ServiceCategory` / `ServiceName` | string | core, Mandatory | `Compute` (or `AI and Machine Learning`) / instance family |
| `ChargeCategory` | enum | core, Mandatory | `Usage` |
| `ChargeClass` | enum, null | core, Mandatory | `Correction` on restatements (lesson 08 §5) |
| `ChargeFrequency` | enum | core, Recommended | `Usage-Based` |
| `ResourceId` | string | core | **pod-resources + DCGM** → the *parent* physical GPU UUID |
| `ResourceName` / `ResourceType` | string, null | core | `GPU` \| `MIG Instance` \| `Shared GPU` (free-form) |
| `RegionId` / `AvailabilityZone` | string, null | core | billing export |
| `ConsumedQuantity` | decimal | core, Conditional | **allocated GPU-hours** = pod-resources share × window |
| `ConsumedUnit` | string | core, Conditional | `GPU-Hours` (UnitFormat compound) |
| `PricingQuantity` / `PricingUnit` | decimal / string | core, Mandatory | billing export — instance-hours |
| `ListUnitPrice` / `ContractedUnitPrice` | decimal, null | core, Conditional | rate card / negotiated rate, per `PricingUnit` |
| `ListCost` / `ContractedCost` / `BilledCost` / `EffectiveCost` | decimal | core, Mandatory | **billing — the allocated-ledger dollars** |
| `PricingCategory` | enum, null | core, Conditional | `Standard` \| `Committed` \| `Dynamic` (spot) |
| `CommitmentDiscountId` / `Status` / `Category` | string / enum | core, Conditional | commitment coverage (lesson 06) |
| `CapacityReservationId` / `Status` | string / enum | core, Conditional | the capacity instrument, separate from the discount |
| `AllocatedResourceId` / `AllocatedResourceName` | string, null | **core (1.3)** | pod-resources → MIG UUID or per-context id. **NULL on the remainder row** |
| `AllocatedServiceName` | string, null | **core (1.5)** | the consuming service, when targeting 1.5 |
| `AllocatedMethodId` | string, null | **core (1.3)** | `sm_slice` \| `memory_fraction` \| `context_runtime` \| `request_weighted` — externally documented |
| `AllocatedMethodDetails` | JSON, null | **core (1.3)** | `{"Elements":[{AllocatedRatio,UsageUnit,UsageQuantity}]}`, ratios summing to 1 |
| `AllocatedTags` | JSON, null | **core (1.3)** | child-specific tags not already in `Tags` |
| `Tags` | JSON | core, Conditional | team / namespace / model — the top-level allocation dimension |
| `x_AttributionRegime` | enum | **ext** | `exclusive` \| `mig` \| `timeslice` \| `mps` (lesson 01) — the *hardware* regime |
| `x_AllocationMechanism` | enum | **ext** | `device_plugin` \| `dra` — layered over the regime, not a fifth regime |
| `x_UtilisationMean` | decimal 0–1 | **ext, gap 1** | DCGM `DCGM_FI_PROF_GR_ENGINE_ACTIVE` over the window (lesson 05) |
| `x_UtilisationMax` | decimal 0–1, null | **ext, gap 1** | max over the window — separates "bursty" from "idle" |
| `x_UtilisedQuantity` | decimal | **ext, gap 1** | `ConsumedQuantity × x_UtilisationMean` — the utilised ledger, in `GPU-Hours` |
| `x_UtilisedEffectiveCost` | decimal | **ext, gap 1** | `EffectiveCost × x_UtilisationMean` (lesson 02) |
| `x_SplitConfidence` | enum, null | **ext, gap 2** | `exact` \| `estimated` \| `unattributable` |
| `x_SplitUncertainty` | decimal, null | **ext, gap 2** | the band on `AllocatedRatio`; 0 for `exact`, populated for `timeslice` |
| `x_UtilisationScope` | enum | **ext, gap 2** | `instance` \| `device_aggregate` — **the honesty flag.** On a shared device `x_UtilisationMean` is the *device* number and cannot be resolved per tenant |
| `x_SourceSnapshotId` | string | **ext** | the pod-resources snapshot key — lesson 08's dispute-protocol replay key |
| `x_AppTokens` | bigint, null | **ext, boundary** | inference-server counter (lesson 05) |
| `x_AppRuns` | bigint, null | **ext, boundary** | orchestrator/job counter |
| `x_UnitCostPer1000Tokens` | decimal, null | **ext, boundary** | `EffectiveCost / (x_AppTokens/1000)` — note the `1000 Tokens` unit convention |
| `x_UnitCostPerRun` | decimal, null | **ext, boundary** | `EffectiveCost / x_AppRuns` |

**`x_UtilisationScope` is the field most schemas miss and the one an interviewer will notice.** Without it, a reader cannot tell whether `x_UtilisationMean = 0.62` means "this tenant's MIG instance ran at 62%" or "the physical device this tenant shares ran at 62%, and we have no idea what that tenant's share of it was." Those are different claims and the column that distinguishes them costs nothing.

**As DDL**, so it is a thing rather than a diagram:

```sql
CREATE TABLE gpu_cost_and_usage (
  -- ── core FOCUS 1.4: time, currency, accounts ─────────────────────────
  ChargePeriodStart        TIMESTAMP      NOT NULL,
  ChargePeriodEnd          TIMESTAMP      NOT NULL,
  BillingPeriodStart       TIMESTAMP      NOT NULL,
  BillingPeriodEnd         TIMESTAMP      NOT NULL,
  BillingCurrency          VARCHAR(3)     NOT NULL,
  BillingAccountId         VARCHAR(64)    NOT NULL,
  SubAccountId             VARCHAR(64),
  -- ── core FOCUS 1.3+: participating entities (NOT ProviderName) ───────
  ServiceProviderName      VARCHAR(128)   NOT NULL,
  HostProviderName         VARCHAR(128),
  InvoiceIssuerName        VARCHAR(128)   NOT NULL,
  -- ── core FOCUS: classification ──────────────────────────────────────
  ServiceCategory          VARCHAR(32)    NOT NULL,   -- 'Compute'
  ServiceName              VARCHAR(128)   NOT NULL,
  ChargeCategory           VARCHAR(16)    NOT NULL,   -- 'Usage'
  ChargeClass              VARCHAR(16),               -- 'Correction' | NULL
  ChargeFrequency          VARCHAR(16)    NOT NULL,   -- 'Usage-Based'
  ChargeDescription        VARCHAR(512),
  -- ── core FOCUS: resource ────────────────────────────────────────────
  ResourceId               VARCHAR(128),              -- PARENT physical GPU
  ResourceName             VARCHAR(128),
  ResourceType             VARCHAR(64),               -- 'GPU'|'MIG Instance'
  RegionId                 VARCHAR(64),
  AvailabilityZone         VARCHAR(64),
  -- ── core FOCUS: quantity & price (note the TWO quantities) ──────────
  ConsumedQuantity         DECIMAL(20,10),
  ConsumedUnit             VARCHAR(32),               -- 'GPU-Hours'
  PricingQuantity          DECIMAL(20,10),
  PricingUnit              VARCHAR(32),               -- 'Hours'
  ListUnitPrice            DECIMAL(20,10),
  ContractedUnitPrice      DECIMAL(20,10),
  -- ── core FOCUS: cost, all four, all NOT NULL ────────────────────────
  ListCost                 DECIMAL(20,10) NOT NULL,
  ContractedCost           DECIMAL(20,10) NOT NULL,
  BilledCost               DECIMAL(20,10) NOT NULL,
  EffectiveCost            DECIMAL(20,10) NOT NULL,   -- allocate on THIS
  -- ── core FOCUS: commitments & reservations (separate instruments) ───
  PricingCategory            VARCHAR(16),             -- 'Committed'|'Dynamic'
  CommitmentDiscountId       VARCHAR(128),
  CommitmentDiscountCategory VARCHAR(16),             -- 'Spend'|'Usage'
  CommitmentDiscountStatus   VARCHAR(8),              -- 'Used'|'Unused'
  CapacityReservationId      VARCHAR(128),
  CapacityReservationStatus  VARCHAR(8),              -- 'Used'|'Unused'
  -- ── core FOCUS 1.3+ (1.5 for AllocatedServiceName): the split ───────
  AllocatedResourceId      VARCHAR(128),  -- NULL on origin AND on remainder
  AllocatedResourceName    VARCHAR(128),
  AllocatedServiceName     VARCHAR(128),  -- FOCUS 1.5
  AllocatedMethodId        VARCHAR(64),   -- 'sm_slice'|'context_runtime'|…
  AllocatedMethodDetails   JSON,          -- {"Elements":[{AllocatedRatio,…}]}
  AllocatedTags            JSON,
  Tags                     JSON,
  -- ══ x_ EXTENSION 1: the utilised ledger FOCUS has no column for ═════
  x_UtilisationMean        DECIMAL(6,5),  -- DCGM_FI_PROF_GR_ENGINE_ACTIVE
  x_UtilisationMax         DECIMAL(6,5),
  x_UtilisationScope       VARCHAR(20)    NOT NULL,   -- 'instance'
                                          -- | 'device_aggregate'  ← honesty
  x_UtilisedQuantity       DECIMAL(20,10),            -- GPU-Hours
  x_UtilisedEffectiveCost  DECIMAL(20,10),
  -- ══ x_ EXTENSION 2: the split confidence FOCUS has no column for ════
  x_AttributionRegime      VARCHAR(16)    NOT NULL,   -- exclusive|mig|
                                          -- timeslice|mps
  x_AllocationMechanism    VARCHAR(16)    NOT NULL,   -- device_plugin|dra
  x_SplitConfidence        VARCHAR(16),   -- exact|estimated|unattributable
  x_SplitUncertainty       DECIMAL(6,5),  -- band on AllocatedRatio
  -- ══ x_ BOUNDARY: application counters, outside FOCUS's scope ════════
  x_AppTokens              BIGINT,
  x_AppRuns                BIGINT,
  x_UnitCostPer1000Tokens  DECIMAL(20,10),
  x_UnitCostPerRun         DECIMAL(20,10),
  -- ══ x_ provenance: the dispute-protocol replay key (lesson 08 §8) ═══
  x_SourceSnapshotId       VARCHAR(128)   NOT NULL
);
```

**Declare the extensions machine-readably.** FOCUS's `Schema` metadata collection exists for exactly this and almost nobody uses it. It carries `SchemaId`, `FocusVersion`, `CreationDate`, `DatasetInstanceId`, and a `ColumnDefinition` list whose entries carry `ColumnName`, `DataType`, and — as applicable — `NumericPrecision`, `NumberScale`, `StringMaxLength`, `StringEncoding`, `ProviderTagPrefixes`, `Deprecated`, and `PreviousColumnName`. `FocusVersion` MUST match a published version of the specification and MUST match the version the dataset conforms to; `Deprecated` MUST be `true` when a column is planned for removal, which gives you the same deprecation runway FOCUS gives its own consumers. Published alongside your export at, for example, `/focus/metadata/schemas/<schema-id>.json`:

```json
{
  "SchemaId": "gpu-cost-and-usage-2026-08",
  "FocusVersion": "1.4",
  "CreationDate": "2026-08-01T00:00:00Z",
  "DatasetInstanceId": "prod-gpu-fleet-01",
  "DataGeneratorVersion": "2.3.0",
  "ColumnDefinition": [
    { "ColumnName": "ChargePeriodStart", "DataType": "DATETIME" },
    { "ColumnName": "EffectiveCost", "DataType": "DECIMAL",
      "NumericPrecision": 20, "NumberScale": 10 },
    { "ColumnName": "ConsumedQuantity", "DataType": "DECIMAL",
      "NumericPrecision": 20, "NumberScale": 10 },
    { "ColumnName": "AllocatedMethodDetails", "DataType": "JSON" },
    { "ColumnName": "Tags", "DataType": "JSON",
      "ProviderTagPrefixes": ["gpufleet", "gf"] },
    { "ColumnName": "x_UtilisationMean", "DataType": "DECIMAL",
      "NumericPrecision": 6, "NumberScale": 5 },
    { "ColumnName": "x_UtilisationScope", "DataType": "STRING",
      "StringMaxLength": 20, "StringEncoding": "UTF-8" },
    { "ColumnName": "x_SplitUncertainty", "DataType": "DECIMAL",
      "NumericPrecision": 6, "NumberScale": 5 },
    { "ColumnName": "x_AttributionRegime", "DataType": "STRING",
      "StringMaxLength": 16, "StringEncoding": "UTF-8" }
  ]
}
```

Publishing that file is what turns "I added some custom columns" into "I extended a standard, declared the extension, and versioned it."

### 9. Where every value comes from — the production pipeline

A schema without provenance is a spreadsheet. Every column has exactly one authoritative source, and the order the sources arrive in decides what you can emit when one is missing.

```
  PRODUCING ONE ROW — three streams, in dependency order
  ═══════════════════════════════════════════════════════════════════════════

  t0  ┌─ STREAM A · BILLING / FOCUS EXPORT ───────── arrives: hours to days ─┐
      │  fills: BilledCost EffectiveCost ListCost ContractedCost            │
      │         PricingQuantity/Unit  *UnitPrice  BillingAccountId          │
      │         ServiceProviderName HostProviderName InvoiceIssuerName      │
      │         PricingCategory CommitmentDiscount* CapacityReservation*    │
      │         RegionId AvailabilityZone ServiceCategory ServiceName       │
      │  grain: ONE INSTANCE. 8 GPUs = 1 line. Divide by PHYSICAL count.    │
      └───────────────────────────────┬─────────────────────────────────────┘
                                      │  ✗ missing ⇒ NO ROW. You cannot
                                      │    invent a cost. Emit quantities
                                      │    only and backfill on arrival.
                                      ▼
  t1  ┌─ STREAM B · POD-RESOURCES API ───────────── arrives: seconds ───────┐
      │  fills: ResourceId(parent GPU UUID)  AllocatedResourceId(child)     │
      │         AllocatedResourceName  ConsumedQuantity  ConsumedUnit       │
      │         Tags AllocatedTags  x_AttributionRegime                     │
      │         x_AllocationMechanism  x_SourceSnapshotId                   │
      │         AllocatedMethodId + AllocatedMethodDetails.AllocatedRatio   │
      │  ▸ EXACT IN EVERY REGIME. A scheduler record, not a measurement.    │
      └───────────────────────────────┬─────────────────────────────────────┘
                                      │  ✗ missing ⇒ emit the ORIGIN row
                                      │    only, unsplit. Never guess a
                                      │    tenant.
                                      ▼
  t2  ┌─ STREAM C · DCGM ────────────────────────── arrives: 1–15 s ────────┐
      │  fills: x_UtilisationMean  x_UtilisationMax  x_UtilisedQuantity     │
      │         x_UtilisedEffectiveCost  x_UtilisationScope                 │
      │         GPU model / UUID cross-check against stream B               │
      │  ▸ EXACT for exclusive + MIG (per-instance counters).               │
      │    DEVICE-AGGREGATE for time-slicing ⇒ x_UtilisationScope =         │
      │    'device_aggregate', x_SplitConfidence = 'estimated'.             │
      └───────────────────────────────┬─────────────────────────────────────┘
                                      │  ✗ missing ⇒ x_Utilised* NULL and
                                      │    the CHARGEBACK IS STILL VALID.
                                      │    (Lesson 08's architecture test:
                                      │     kill DCGM, regenerate, diff.
                                      │     Any diff in a cost column is a
                                      │     design bug.)
                                      ▼
  t3  ┌─ STREAM D · APPLICATION COUNTERS ─────────── arrives: seconds ──────┐
      │  fills: x_AppTokens x_AppRuns x_UnitCostPer1000Tokens               │
      │         x_UnitCostPerRun                                            │
      │  ▸ OUTSIDE FOCUS'S SCOPE — not a gap in the standard.               │
      │  ✗ missing ⇒ unit-cost columns NULL. Everything else stands.        │
      └─────────────────────────────────────────────────────────────────────┘

  DEGRADATION ORDER (read bottom-up): the schema loses its OPTIONAL columns
  before its BILLABLE ones. That ordering is the design — it is the trust
  boundary from lesson 08 §1, expressed as a null-propagation rule.
```

Two rules fall out of the diagram and both belong in the deliverable's README:

1. **Cost columns depend only on streams A and B**, both of which are recorded facts. Streams C and D can be entirely absent and the invoice is still correct and still conformant. Test it by disabling the DCGM exporter and diffing the regenerated statement.
2. **`AllocatedMethodDetails.AllocatedRatio` comes from stream B, not stream C.** This surprises people. Under MIG the ratio is the partition geometry — a scheduler and hardware-configuration fact, not a measurement. Under time-slicing, if you choose `request_weighted` the ratio is also stream B; only if you choose `context_runtime` does the ratio depend on a measurement, and that is precisely the case where `x_SplitConfidence` must read `estimated`. **The confidence column is a function of which method you chose, which is a function of which stream the ratio came from.**

### 10. The query pack

Six queries that make the schema a system rather than a table. All are ANSI-ish SQL and should run with minor dialect edits.

```sql
-- 1 ── RECONCILE TO THE INVOICE. Run this before anything else, every period.
--      Origin rows carry the invoice truth; allocated rows re-express it.
--      With the zeroed-origin convention, summing everything is already
--      correct. With a non-zeroed origin, filter to origin rows only.
SELECT BillingPeriodStart,
       SUM(EffectiveCost) AS total_effective,
       SUM(BilledCost)    AS total_billed
FROM   gpu_cost_and_usage
WHERE  ChargeCategory = 'Usage'
GROUP BY BillingPeriodStart;

-- 2 ── THE CHARGEBACK STATEMENT (lesson 08). Allocated ledger, per team.
SELECT JSON_VALUE(COALESCE(AllocatedTags, Tags), '$.team') AS team,
       SUM(ConsumedQuantity)  AS allocated_gpu_hours,
       SUM(EffectiveCost)     AS charge
FROM   gpu_cost_and_usage
WHERE  ChargeCategory = 'Usage'
  AND  ChargePeriodStart >= ? AND ChargePeriodEnd <= ?
  AND  AllocatedResourceId IS NOT NULL     -- allocated portions only
GROUP BY 1
ORDER BY charge DESC;

-- 3 ── THE UNALLOCATABLE REMAINDER (lesson 08 §4). The spec's own pattern:
--      part of a split (AllocatedMethodId NOT NULL) but allocated to no one.
SELECT ResourceId,
       SUM(ConsumedQuantity) AS unallocated_gpu_hours,
       SUM(EffectiveCost)    AS unallocated_cost
FROM   gpu_cost_and_usage
WHERE  ChargeCategory = 'Usage'
  AND  ChargePeriodStart >= ? AND ChargePeriodEnd <= ?
  AND  AllocatedMethodId   IS NOT NULL
  AND  AllocatedResourceId IS NULL
GROUP BY ResourceId;

-- 4 ── THE WASTE LEDGER (lesson 02). Both ledgers, side by side, honest
--      about which utilisation numbers can be trusted per tenant.
SELECT JSON_VALUE(COALESCE(AllocatedTags, Tags), '$.team') AS team,
       SUM(ConsumedQuantity)                        AS allocated_gpu_hours,
       SUM(x_UtilisedQuantity)                      AS utilised_gpu_hours,
       SUM(ConsumedQuantity - x_UtilisedQuantity)   AS waste_gpu_hours,
       SUM(EffectiveCost - x_UtilisedEffectiveCost) AS waste_cost,
       -- the honesty column: what share of this team's hours have a
       -- per-tenant utilisation number that is actually per-tenant?
       SUM(CASE WHEN x_UtilisationScope = 'instance'
                THEN ConsumedQuantity ELSE 0 END)
         / NULLIF(SUM(ConsumedQuantity), 0)         AS frac_util_trustworthy
FROM   gpu_cost_and_usage
WHERE  ChargeCategory = 'Usage'
  AND  ChargePeriodStart >= ? AND ChargePeriodEnd <= ?
GROUP BY 1;

-- 5 ── UNIT ECONOMICS (lesson 05), per model, at the 1000-token grain.
SELECT JSON_VALUE(Tags, '$.model')            AS model,
       SUM(EffectiveCost)                     AS cost,
       SUM(x_AppTokens)                       AS tokens,
       SUM(EffectiveCost) / NULLIF(SUM(x_AppTokens) / 1000.0, 0)
                                              AS cost_per_1000_tokens
FROM   gpu_cost_and_usage
WHERE  ChargeCategory = 'Usage'
  AND  x_AppTokens IS NOT NULL
  AND  ChargePeriodStart >= ? AND ChargePeriodEnd <= ?
GROUP BY 1;

-- 6 ── THE CONFORMANCE ASSERTION. Run in CI. This is identity A, in SQL.
--      Every origin charge's allocated ratios must sum to 1.
WITH ratios AS (
  SELECT ResourceId, ChargePeriodStart,
         SUM(CAST(JSON_VALUE(AllocatedMethodDetails,
                  '$.Elements[0].AllocatedRatio') AS DECIMAL(12,10))) AS s
  FROM   gpu_cost_and_usage
  WHERE  AllocatedResourceId IS NOT NULL
  GROUP BY ResourceId, ChargePeriodStart
)
SELECT * FROM ratios WHERE ABS(s - 1.0) > 0.000001;   -- MUST return 0 rows
```

Query 6 is the one to wire into CI. The specification says the ratios must sum to 1 and that summable metrics must sum to the origin charge; an assertion that fails the build is how you keep that true as the pipeline evolves. Add a sibling assertion on cost conservation and you have made lesson 09's 7×-inflation class of bug impossible in your own system rather than merely diagnosable in someone else's.

## Perspectives

**Standards-body view.** FOCUS's change discipline is worth studying independently of GPUs: every change is classified Compatible / Migration Compatible / Incompatible, feature levels are derived mechanically from the requirement language rather than assigned by taste, and `ProviderName`/`PublisherName` were deprecated in 1.3 and removed in 1.4 — a full release of runway with a documented successor for each use case. Both 1.3 and 1.4 list "Incompatible changes: None." Borrow the posture for your own schema: additive nullable extensions, a `Deprecated` flag in the published schema metadata before anything disappears, and a version stamp on every export.

**Data-engineering view.** The `Produced from` column and the four-stream diagram in §9 are the design, not documentation of it. A schema that does not record where a value came from cannot be debugged when it is wrong and cannot be defended when a tenant disputes it — which is why `x_SourceSnapshotId` is `NOT NULL` and why the degradation order is specified rather than emergent. The single most valuable property of this table is that its billable columns depend only on recorded facts, so it degrades by losing optional columns rather than by producing quietly wrong money.

**Multi-cloud finance view.** The business case for starting from FOCUS is not aesthetic. Hyperscalers ship FOCUS-shaped exports; a table that speaks the same column vocabulary joins the cost lake finance already queries, for free. A bespoke schema — however well designed — is a silo somebody has to be taught and maintain, and silos lose budget arguments. The `ServiceProviderName` / `HostProviderName` / `InvoiceIssuerName` triple is the concrete payoff: it is the only way to compare a managed platform's rate against the bare-metal rate underneath it without accidentally comparing a reseller margin to a cost of goods.

**GPU-extension view — precisely scoped, because scope is the signal.** Where does current FOCUS not reach for GPUs? Exactly two places: no utilisation ledger anywhere in the standard, and no confidence band on a declared split. Everything else a shared-GPU schema needs — including the split-cost mechanism itself — has been core FOCUS since 1.3. App counters are a scope boundary, not a gap. Scoping the answer that narrowly, instead of repeating the now-inaccurate "FOCUS can't represent shared cost," is itself the evidence that you read the changelog rather than a summary of it.

**Interview view.** "Design a cost schema for a shared GPU fleet" is a question with a wrong-shaped good answer and a right-shaped good answer. The wrong shape is a beautiful bespoke schema. The right shape is: name the standard and the version, name the grain, name the columns that carry the money and the split, name the two extensions and justify each against a specific spec gap, name the three provenance streams, and name the one thing that is unsolvable and what you emit instead of a false number. Six moves, and every one of them is checkable.

## Real-world use cases

- **The FOCUS `ContractCommitment` appendix — an H100 reservation, fully specified.** Scenario 2, Commitment 1 in `specification/appendix/contract_commitments/contract_commitments.md`: `CC Description = "H100 GPU Reservation - Q1"`, `CC Unit = "GPU-Hours"`, `CC Quantity = 5000.00`, `CC Cost = 250000.00`, `CC Discount % = 0.30`, `CC Duration Type = "3 Months"`, `CC Payment Model = "All Upfront"` with `CC Payment Upfront % = 1.00`, `CC Model = "Continuous"`, `CC Category = "Usage"`, `CC Benefit Category = "Discount"`, `CC Type = "Resource Reservation"`, `CC Fulfillment Interval = "Monthly"`, `CC Offer Category = "Negotiated"`, and an applicability filter restricting it to an AI/ML service category. What it shows: the specification's own illustrative example of a GPU commitment is a short-duration, all-upfront, take-or-pay-shaped reservation denominated in `GPU-Hours` — i.e. lesson 06's instrument catalogue rendered in FOCUS columns, confirming both the unit spelling and the commitment shape. *The implied unit price is $50.00 per `GPU-Hour`, which is far above any observed 2026 rate for a single H100 (lesson 07: roughly $2–$11); treat the figures as synthetic illustration and the column semantics as the thing being taught.*

- **The FOCUS split-cost-allocation appendix — a shared cluster split three ways.** `specification/appendix/split_cost_allocation_examples.md`: origin `EffectiveCost` of $100.00 on `cluster-shared-01` for one charge period, split by measured vCPU-hours 40 / 35 / 25 into $40.00 / $35.00 / $25.00 across three consuming services, with the origin row zeroed to $0.00 and `ServiceName` / `ServiceCategory` / `ResourceId` preserved on all four rows. What it shows: the exact row layout this lesson's §5 diagram reproduces, the conservation property, the zeroed-origin convention, and — via `AllocatedServiceName` — why 1.5 needed a column for the *consuming* service, since origin preservation means the consuming service's identity would otherwise have to be reconstructed from an external `AllocatedResourceId` lookup.

- **The FOCUS Validator** — the FinOps Foundation maintains a conformance validator (`finopsfoundation/focus_validator`), which the specification's own overview names as its reference implementation for conformance assessment while explicitly not mandating any particular tool. What it shows: conformance is checkable by machine, which changes how you should treat the enumerations and the `UnitFormat` rules — they are not stylistic advice, they are things that fail a validator run. It also means "is this schema FOCUS-conformant?" has an answer you can produce in CI rather than argue about.

- **OpenCost's `CustomCost` protobuf, and where a real tool got the column names roughly right** — `protos/customcost/` in the OpenCost repository annotates its fields with FOCUS column names: `billed_cost`, `list_cost`, `list_unit_price`, `usage_quantity`, `usage_unit`, `charge_category`, `resource_name`, `resource_type`, plus an extended-attributes message carrying `billing_period_start/end`, `charge_frequency`, `commitment_discount_*`, and `effective_cost`. What it shows: FOCUS alignment is already how a major OSS cost tool models third-party cost, so a FOCUS-shaped GPU table is joinable to real tooling and not a paper exercise. It also shows a trap worth noticing — one field is commented "FOCUS Charge Subcategory," which is not a current `CostAndUsage` column (the current second-level classification is `ServiceSubcategory`) — so **check names against the spec repository rather than against another tool's comments.**

## Worked example

**One H100 80GB node, one hour, four tenancy shapes.** `EffectiveCost` for the node's GPU component is **$16.80/hour** for 8 physical GPUs → **$2.10 per physical GPU-hour** (Silver-tier, 1-year reserved, August 2026 snapshot — module 03 capstone anchor). Rows below are for **one** of those eight GPUs.

**(a) Exclusive — pod `train-a`, one whole GPU.** One row, no split:

| Column | Value |
|---|---|
| `ResourceId` | `GPU-a1b2c3d4-…` |
| `ResourceType` | `GPU` |
| `ConsumedQuantity` / `ConsumedUnit` | `1.0` / `GPU-Hours` |
| `PricingQuantity` / `PricingUnit` | `1.0` / `Hours` (the instance-hour, on the parent row) |
| `EffectiveCost` | `2.10` |
| `ChargeCategory` / `ChargeFrequency` | `Usage` / `Usage-Based` |
| `Allocated*` | all `NULL` |
| `Tags` | `{"team":"research","model":null}` |
| `x_AttributionRegime` / `x_AllocationMechanism` | `exclusive` / `device_plugin` |
| `x_UtilisationMean` / `x_UtilisationScope` | `0.40` / `instance` |
| `x_UtilisedQuantity` / `x_UtilisedEffectiveCost` | `0.40` / `0.84` |
| `x_SplitConfidence` / `x_SplitUncertainty` | `exact` / `0.0` |

Both ledgers on one row: charged $2.10, worked $0.84, waste $1.26. Nothing is split, so no `Allocated*` columns, which is exactly right — using them here would be non-conformant, since `AllocatedResourceId` MUST be null when a charge is not part of a split.

**(b) MIG — 7 × `1g.10gb`, seven tenants.** Eight rows: one origin, seven allocated.

```
  row 0 (origin):  ResourceId=GPU-a1b2…  AllocatedResourceId=NULL
                   AllocatedMethodId=NULL   EffectiveCost=0.00
                   ConsumedQuantity=0.0     (zeroed-origin convention)

  rows 1–7 (allocated), each:
      ResourceId              = GPU-a1b2c3d4-…      ← parent, unchanged
      ResourceType            = MIG Instance
      AllocatedResourceId     = MIG-GPU-a1b2c3d4-…/<gi>/0
      AllocatedResourceName   = "1g.10gb #<n>"
      AllocatedMethodId       = "sm_slice"
      AllocatedMethodDetails  = {"Elements":[
                                  {"AllocatedRatio":0.1428571429,
                                   "UsageUnit":"GPU","UsageQuantity":1},
                                  {"AllocatedRatio":0.1428571429,
                                   "UsageUnit":"GB","UsageQuantity":10,
                                   "x_BasisNote":"memory basis, reported
                                                  for comparison"}]}
      AllocatedTags           = {"team":"team-<n>"}
      ConsumedQuantity        = 0.1428571429   ConsumedUnit = "GPU-Hours"
      EffectiveCost           = 0.30
      x_AttributionRegime     = "mig"
      x_UtilisationScope      = "instance"   ← per-MIG DCGM counters exist
      x_UtilisationMean       = 0.55 (say)
      x_UtilisedQuantity      = 0.0786   x_UtilisedEffectiveCost = 0.165
      x_SplitConfidence       = "exact"   x_SplitUncertainty = 0.0

  CONSERVATION CHECKS:
      Σ EffectiveCost(allocated) = 7 × 0.30      = $2.10   ≡ origin ✓
      Σ ConsumedQuantity         = 7 × 0.142857  = 1.0     ≡ 1 GPU-hour ✓
      Σ AllocatedRatio (compute) = 7 × 0.142857  = 1.0     ≡ required 1 ✓

  CONTRAST WITH LESSON 09: OpenCost bills these seven pods $2.10 each,
  $14.70 total, for a $2.10 card. In this schema that is not a bug you
  could ship — it violates a NORMATIVE conservation requirement, and
  query 6 in §10 fails the build.
```

Note what the two-element `AllocatedMethodDetails` buys: the compute basis (1/7 of the SM slices) and the memory basis (10 GB of 80 GB = 0.125) disagree, which is lesson 01 §4's whole point. The array lets you *declare the basis you billed on and show the other one*, rather than silently picking. Only the elements belonging to the billed method participate in the sum-to-1 requirement; a second, comparative element is a documented extension and should be marked as such (here via an `x_BasisNote` custom key, which the spec permits inside an `Elements` entry).

**(c) Time-sliced — `replicas: 4`, four tenants.** Five rows: one origin, four allocated.

```
  each allocated row:
      ResourceId             = GPU-a1b2c3d4-…
      ResourceType           = "Shared GPU"
      AllocatedResourceId    = GPU-a1b2c3d4-…::0 | ::1 | ::2 | ::3
      AllocatedMethodId      = "context_runtime"
      AllocatedMethodDetails = {"Elements":[{"AllocatedRatio":0.70,
                                 "UsageUnit":"Seconds",
                                 "UsageQuantity":2520}],
                                "x_EstimatorId":"nvml_pid_sampling",
                                "x_EstimatorWindowSeconds":3600}
      ConsumedQuantity       = 0.70 | 0.25 | 0.025 | 0.025   GPU-Hours
      EffectiveCost          = 1.47 | 0.525 | 0.0525 | 0.0525
      x_AttributionRegime    = "timeslice"
      x_UtilisationScope     = "device_aggregate"   ◀◀ THE HONEST FLAG
      x_UtilisationMean      = 0.83  ← the DEVICE's number, identical on
                                       all four rows. NOT this tenant's.
      x_SplitConfidence      = "estimated"
      x_SplitUncertainty     = 0.90  ← see the derivation below

  CONSERVATION:  1.47 + 0.525 + 0.0525 + 0.0525 = $2.10 ✓
                 ratios 0.70 + 0.25 + 0.025 + 0.025 = 1.0 ✓
  ⇒ FULLY CONFORMANT. And still an estimate.

  DERIVING x_SplitUncertainty — make it reproducible, not decorative:
    the two defensible methods disagree, so bound the ratio by their span.
      request_weighted (equal shares)  → 0.25 each
      context_runtime  (measured)      → 0.70 / 0.25 / 0.025 / 0.025
    for the smallest tenant:
      |0.25 − 0.025| / 0.25 = 0.90   ⇒ the ratio could be wrong by 90%
                                        of the alternative method's value
    for the largest tenant:
      |0.70 − 0.25| / 0.70 = 0.64
    Publish the METHOD SPAN as the band, and say so in the method doc
    behind "context_runtime". A number you can re-derive beats a number
    that looks precise.
```

**This is the row that demonstrates seniority in an interview.** It is fully conformant — parent, named children, named method, ratios summing to 1, costs summing to the origin — and it still cannot tell a reader how much to trust itself, because FOCUS has no column that could. The two `x_` fields, `x_SplitConfidence` and `x_SplitUncertainty`, plus `x_UtilisationScope = 'device_aggregate'`, are the entire content of gap 2, and they are three columns.

**(d) Unit economics, joined.** The MIG tenant `team-3` runs an inference server that served **1.2 million tokens** in the hour:

```
  x_AppTokens             = 1_200_000
  x_UnitCostPer1000Tokens = EffectiveCost / (x_AppTokens / 1000)
                          = 0.30 / 1200
                          = $0.00025 per 1000 tokens
                          = $0.25 per million tokens
  cross-check against the utilised ledger (the efficiency view, not the bill):
  x_UtilisedEffectiveCost / (tokens/1000) = 0.165 / 1200 = $0.0001375
                          = $0.1375 per million tokens
```

The gap between $0.25/M billed and $0.1375/M "at perfect utilisation" is the headroom in the unit cost. Neither number is FOCUS's business — both live entirely in `x_` columns — and both are the number a business actually asks for. That is the scope boundary in §7, made concrete.

**(e) The same GPU-hour from two providers, normalised.** This is why the standard is worth the trouble:

| | Hyperscaler export | Neocloud export |
|---|---|---|
| native line | `p5.48xlarge` instance-hour, $X, savings-plan amortisation in a separate file | `H100-SXM` GPU-hour, $Y, take-or-pay minimum on a separate invoice |
| `ServiceProviderName` | the hyperscaler | the neocloud |
| `HostProviderName` | the hyperscaler | the neocloud (or its colo partner) |
| `PricingQuantity` / `Unit` | `1` / `Hours` (one instance) | `1` / `Hours` (one GPU) |
| `ConsumedQuantity` / `Unit` | `8` / `GPU-Hours` after rung-2 division | `1` / `GPU-Hours` natively |
| `BilledCost` | `0` for hours covered by a prepaid commitment | the take-or-pay draw-down |
| `EffectiveCost` | the amortised true cost of the hour | the amortised true cost of the hour |
| `PricingCategory` | `Committed` | `Committed` |
| `CommitmentDiscountStatus` | `Used` | `Used` |

**Query 2 in §10 runs unchanged across both.** That is the entire value proposition of a normalisation layer, and it is why the answer to "design a GPU cost schema" starts with "start from FOCUS" rather than with a column list of your own.

## Practice

Feeds [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md) as `writeup/focus-schema.md` plus the emitter in `operator/`.

1. **Emit the schema.** Write §8's DDL for your own fleet, splitting core FOCUS columns from `x_` extensions with a comment banner between them. Then produce the FOCUS **schema metadata JSON** (§8) declaring every column with its `DataType`, precision/scale or string length, and the `FocusVersion` you targeted. Publishing that file is the step that separates "custom columns" from "a declared extension."
2. **Produce the four regime row-sets.** Generate cases (a)–(d) from the worked example for a real or plausible fleet hour. Assert, in code: costs sum to the origin, `ConsumedQuantity` sums to the physical GPU-hours, and `AllocatedRatio` sums to 1 per origin charge. **Wire query 6 into CI so a regression fails the build.**
3. **Write the allocated-method documents.** `AllocatedMethodId` obliges you to document each method externally, and the spec requires the documentation to reference a single ID value. Write one short page each for `sm_slice`, `memory_fraction`, `context_runtime` and `request_weighted`: what the ratio is computed from, which data stream supplies it, whether it is exact or estimated, and how `x_SplitUncertainty` is derived when estimated. This is a small artifact with outsized credibility — it is the thing that makes an arbitrary basis auditable.
4. **The provenance map.** For every column in §8, name its stream (billing / pod-resources / DCGM / app counters) and its behaviour when that stream is missing. Then run lesson 08's architecture test: disable the DCGM exporter, regenerate a period, and diff. **Any difference in a cost column is a design bug, not a data bug.**
5. **Run a conformance check.** Point the FOCUS Validator (or your own assertions derived from §3 and §4) at a generated export. Every enumerated column must contain only allowed values; `ConsumedUnit` must be `GPU-Hours` in `UnitFormat` compound form; `AllocatedResourceId` must be null on origin and remainder rows and non-null on allocated ones; `ChargeClass` must be null or `Correction`.
6. **The interview paragraph.** Six sentences: the standard and version, the grain, the columns carrying money and split, the two extensions with the gap each closes, the three provenance streams, and the one thing that is unsolvable plus what you emit instead. Practise saying it in ninety seconds.

**Acceptance criteria:** real FOCUS column IDs only, verified against the spec repository, with no removed columns (`ProviderName`, `PublisherName`) · the split-cost block used correctly, including the null rules and the `Elements`-wrapped `AllocatedMethodDetails` · conservation asserted in CI · exactly two claimed gaps, each justified against a named absence in the spec · time-sliced rows carrying an uncertainty and a `device_aggregate` utilisation scope rather than a false per-tenant number.

## Common pitfalls

- **Claiming FOCUS cannot represent shared or split cost.** Symptom: someone in the room says "1.3 added split cost allocation" and your credibility on the standard evaporates. Mechanism: the claim was accurate through 1.2 and the feature landed in December 2025 with five columns and a normative handling attribute. Correction: the current claim is "FOCUS names the methodology and still cannot carry the confidence."
- **Getting `AllocatedMethodDetails` shape wrong.** Symptom: a validator rejects the JSON, or a `JSON_TABLE` extraction over `$.Elements[*]` returns nothing. Mechanism: the object is `{"Elements":[{…}]}`, not a flat map — the array exists so a split can cite several dimensions. Correction: wrap in `Elements`; ensure ratios sum to 1 across the origin charge's allocated rows; prefix any custom key `x_` + PascalCase.
- **Populating `AllocatedResourceId` on the remainder row.** Symptom: the remainder query in §10 returns nothing and the unallocatable capacity vanishes from the statement. Mechanism: the spec requires `AllocatedResourceId` to be **null** on the unallocated portion of a split, and `AllocatedMethodId` to be non-null because the row is still part of the split. Correction: null the child id, keep the method id — that combination *is* the remainder's identity.
- **Mixing origin-row conventions.** Symptom: totals are double the invoice. Mechanism: the spec permits zeroing the origin, omitting it, or pairing it with an offsetting row, and a consumer summing `EffectiveCost` without filtering double-counts if the origin retained its value. Correction: choose one convention, document it in the schema metadata, and never mix within a dataset.
- **Assuming FOCUS prescribes an allocation algorithm.** Symptom: looking for "the FOCUS-correct" MIG basis. Mechanism: `AllocatedMethodId` is a *provider-defined* identifier whose only requirement is that documentation exists and references a single value; the standard deliberately does not choose between SM-slice, memory-fraction, and runtime weighting. Correction: choose, document, and cite the doc — the choice is yours and the accountability is the point.
- **Believing split-cost allocation solved time-slicing.** Symptom: emitting time-sliced rows with no uncertainty because they are "now standard-compliant." Mechanism: a place to declare a ratio is not a way to measure one; lesson 01's non-invertibility argument is untouched by any schema change. Correction: `x_SplitConfidence`, `x_SplitUncertainty`, and `x_UtilisationScope = 'device_aggregate'`.
- **Assuming utilisation became native because split cost did.** Symptom: looking for a FOCUS utilisation column. Mechanism: the two features are unrelated; every FOCUS cost and quantity column, including the `Allocated*` ones, is allocated or billed, and `AllocatedMethodDetails.UsageQuantity` is the metric that *drove a ratio*, not a measure of device work. Correction: keep the `x_Utilised*` block.
- **Citing `ProviderName` or `PublisherName`.** Symptom: a column that does not exist in the version you claim to target. Mechanism: deprecated in 1.3, removed entirely in 1.4. Correction: `ServiceProviderName` (who sold it), `HostProviderName` (who runs the hardware), `InvoiceIssuerName` (who invoiced) — and know which one you mean.
- **Spelling the unit freely.** Symptom: `ConsumedUnit` values that fail `UnitFormat` and quietly break cross-provider aggregation. Mechanism: compound units must be hyphenated `<singular>-<plural>`, and only eight time base units are allowed. Correction: `GPU-Hours`, `Hours`, `Tokens` or `1000 Tokens` — and take the free win of using the same spelling the spec's own GPU commitment example uses.

## Self-check

- **Is it still true that GPU sharing can only be expressed through `Tags` in FOCUS?** *Answer:* No. That was accurate through FOCUS 1.2 and is false as of **1.3 (December 2025)**, which added a first-class split-cost primitive: `AllocatedResourceId` (the child receiving a portion of the parent `ResourceId`'s cost), `AllocatedResourceName`, `AllocatedMethodId` (a provider-defined methodology identifier that MUST resolve to external documentation), `AllocatedMethodDetails` (a JSON object with an `Elements` array of `{AllocatedRatio, UsageUnit, UsageQuantity}`, ratios summing to 1 across the origin charge), and `AllocatedTags`. FOCUS 1.5 adds `AllocatedServiceName` for the consuming service. The still-true, narrower claim is: FOCUS has a standard place to *declare* a split and no column to carry the *confidence* of an estimated one.

- **A MIG instance and a time-sliced tenant both get conformant split rows. What is different about them, and which columns express the difference?** *Answer:* Structurally nothing — both are `ResourceId` (parent) + `AllocatedResourceId` (child) + `AllocatedMethodId` + `AllocatedMethodDetails` + costs that sum to the origin. Epistemically everything. The MIG ratio is a hardware partition fact read from the device geometry, so `AllocatedMethodId = "sm_slice"`, `x_SplitConfidence = "exact"`, `x_SplitUncertainty = 0`, and `x_UtilisationScope = "instance"` because per-MIG DCGM counters genuinely exist. The time-slice ratio is an estimate — one device counter, N resident CUDA contexts, non-invertible (lesson 01 §5) — so `AllocatedMethodId = "context_runtime"`, `x_SplitConfidence = "estimated"`, `x_SplitUncertainty` set from the span between the defensible methods, and `x_UtilisationScope = "device_aggregate"` because the utilisation figure is the device's and is identical on all N rows. All five distinguishing fields are `x_` extensions; core FOCUS cannot tell the two cases apart.

- **What does FOCUS 1.4 still lack for a shared-GPU schema, and how do you add it without leaving the standard?** *Answer:* Exactly two things. (1) **No utilisation ledger** — every cost and quantity column, including the `Allocated*` ones, is allocated or billed; `ConsumedQuantity` is provider-metered, not hardware-measured, and `AllocatedMethodDetails.UsageQuantity` is the metric that drove a split ratio, not device work. (2) **No confidence on a declared split** — the ratio has a home, its error band does not. A third item is a scope boundary rather than a gap: application counters (tokens, runs) are not something a provider bills, so they were never in scope. Add all of it through the `CustomColumnHandling` extension path — `x_` prefix, PascalCase, alphanumeric, ≤50 characters, `Id`/`Name` suffixes where applicable — which keeps the core row fully conformant and lake-joinable, and declare each new column in the `Schema` metadata's `ColumnDefinition` list with its data type and precision.

- **Which three data streams produce a row, and what happens when each is missing?** *Answer:* **Billing / FOCUS export** supplies the four cost columns, pricing quantity and unit, the participating-entity names, the commitment and reservation columns, and region; missing it means no row at all, because you cannot invent a cost — emit quantities and backfill. **Pod-resources API** supplies the parent and child resource ids, `ConsumedQuantity`, the tags, the regime, the snapshot key, and — for `sm_slice` and `request_weighted` — the `AllocatedRatio` itself; missing it means emitting the unsplit origin row only, never a guessed tenant. **DCGM** supplies `x_UtilisationMean`/`Max`, the derived `x_Utilised*` columns, and `x_UtilisationScope`; missing it leaves those columns null and **the chargeback is still valid and still conformant** — which is the architecture test from lesson 08 §1. Optionally, **application counters** supply `x_AppTokens`/`x_AppRuns` and the unit costs. The degradation order is the design: the schema loses optional columns before billable ones.

- **Walk the seven-way MIG card through this schema and say why lesson 09's 7× error is impossible here.** *Answer:* One origin row for the physical GPU (`AllocatedResourceId` null, `EffectiveCost` zeroed under the zeroed-origin convention) plus seven allocated rows, each carrying the same `ResourceId` as parent, its own MIG UUID as `AllocatedResourceId`, `AllocatedMethodId = "sm_slice"`, `AllocatedMethodDetails` with `AllocatedRatio = 0.142857…`, `ConsumedQuantity = 0.142857…` GPU-Hours and `EffectiveCost = $0.30` against a $2.10 card. The 7× is impossible for two independent reasons. Normatively, `DataGeneratorCalculatedSplitCostAllocationHandling` requires summable metrics to sum across allocated charges to the origin charge, and the `AllocatedRatio` values to sum to 1 — seven whole-GPU charges violate both. Operationally, query 6 in §10 asserts the ratio sum in CI and fails the build. OpenCost's seven rows are not a non-conformant split; they are seven unrelated charges with no origin/allocated relationship at all, which is why nothing in its model detects the inflation.

- **Why is `x_UtilisationScope` worth a column of its own?** *Answer:* Because without it `x_UtilisationMean = 0.62` is ambiguous between two claims with completely different strength: "this tenant's own MIG instance ran at 62%," which is a per-instance measurement from a per-instance DCGM counter, and "the physical device this tenant shares ran at 62%, and this tenant's share of that is unknown," which is a device aggregate that will be identical across every sharer of that device. A reader who cannot distinguish them will compute a per-tenant waste figure for time-sliced tenants that is meaningless, and a chargeback conversation built on it will collapse the first time a tenant asks how the number was derived. Two allowed values — `instance` and `device_aggregate` — remove the ambiguity for free, and they make the honest answer under time-slicing machine-readable rather than a footnote.

- **Someone asks "which FOCUS columns carry a GPU-hour charge?" Answer in one breath.** *Answer:* `ChargePeriodStart`/`End` bound it; `ChargeCategory = 'Usage'` and `ChargeFrequency = 'Usage-Based'` classify it; `ServiceCategory = 'Compute'` and `ServiceName` say what it is; `ResourceId` names the GPU (or its parent, if split); `ConsumedQuantity` and `ConsumedUnit = 'GPU-Hours'` carry the quantity while `PricingQuantity`/`PricingUnit` carry what the provider actually rated; `ListUnitPrice` and `ContractedUnitPrice` carry the rates; `ListCost`, `ContractedCost`, `BilledCost` and `EffectiveCost` carry the four dollar views, and **you allocate on `EffectiveCost`** because `BilledCost` reads zero for hours covered by a prepaid commitment; `PricingCategory` says whether it was `Standard`, `Committed` or `Dynamic` (spot); `CommitmentDiscountId`/`Status` and `CapacityReservationId`/`Status` say which instrument covered it, tracked separately because a discount and a capacity hold are different products; `Tags` carries the tenant; and if the device was shared, `AllocatedResourceId`, `AllocatedMethodId`, `AllocatedMethodDetails` and `AllocatedTags` carry the split.

## Connections & what's next

This is the module capstone; it closes threads rather than opening one. Lesson 01's regimes become `x_AttributionRegime`, `x_AllocationMechanism` and the choice behind `AllocatedMethodId`. Lesson 02's two ledgers become the FOCUS cost columns plus the `x_Utilised*` block, with `EffectiveCost` as the allocation basis exactly as that lesson argued. Lesson 03's idle detection and lesson 04's fragmentation become the unallocated-remainder rows that query 3 isolates. Lesson 05's unit economics become the `x_App*` and `x_UnitCost*` joins. Lesson 06's commitment strategy becomes `CommitmentDiscount*`, `CapacityReservation*`, `PricingCategory`, and the `ContractCommitment` dataset. Lesson 07's provider comparison becomes `ServiceProviderName` / `HostProviderName` / `InvoiceIssuerName`. Lesson 08's chargeback becomes the split rows, the remainder row, and `ChargeClass = 'Correction'` for restatements. Lesson 09's sourced gap becomes the reason the schema exists — and the conservation requirement in §5 is the rule that tool's MIG behaviour violates by a factor of seven.

Those three artifacts — this schema, lesson 09's teardown, and the finished `gpu-cost-operator` emitting both ledgers with an honest degradation on time-slicing — are the **[GPU cost synthesis](../practice/gpu-cost-synthesis/README.md)** deliverable, which is the portfolio centrepiece you lead with in **Module 12**. Before you call it done, run the module [checkpoint](../checkpoint.md) cold: attribute across all four regimes, defend the time-slicing limit, compute both ledgers, name the OpenCost gap from source, and whiteboard this schema from memory.

## References & further reading

**Primary sources — the FOCUS specification, read from the spec repository at `main`, commit `7f19ccb`, on 2026-08-17.** *`focus.finops.org` and `finops.org` are egress-blocked from this environment; every claim below was verified against the specification's own markdown source in the repository rather than the rendered site. The repository is the normative source, so this is the stronger read, but note that `main` currently carries the **1.5 candidate release** — check the `specification/versions/` files and the CHANGELOG before quoting a version number.*

- **FOCUS Specification repository** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec> — the whole spec as markdown, one file per column, attribute, supported feature and glossary term. This is where you should verify column names before any writeup.
- **CHANGELOG** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/CHANGELOG.md> — source for §2: v1.3 announced December 2025 (5 new supported features, the `ContractCommitment` dataset, 8 new `CostAndUsage` columns, the Provider/Publisher replacement as Migration Compatible, no Incompatible changes) and v1.4 announced June 2026 (2 datasets, 47 columns, 6 attributes, 2 supported features, 17 glossary entries; `ContractApplied` reformatted to JSON Object Schema; `ColumnHandling`/`DiscountHandling`/`InvoiceHandling` attributes removed; `ProviderName`/`PublisherName` removed; `InvoiceId` tightened to Conditional; no Incompatible changes).
- **`CostAndUsage` column definitions** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/tree/main/specification/datasets/cost_and_usage/columns> — every data type, column type, feature level, nullability and Version-Introduced value in §3's tables came from these files, one per column. Read `billedcost.md`, `effectivecost.md`, `consumedquantity.md`, `consumedunit.md`, `pricingquantity.md`, `pricingunit.md`, `resourceid.md`, `chargecategory.md`, `chargeclass.md`, `chargefrequency.md`, `servicecategory.md`, `pricingcategory.md`, `commitmentdiscountstatus.md`, `capacityreservationstatus.md`, `serviceprovidername.md`, `hostprovidername.md`, `invoiceissuername.md` for the allowed-value tables in §4.
- **`AllocatedMethodDetails` column definition** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/specification/datasets/cost_and_usage/columns/allocatedmethoddetails.md> — the `AllocatedMethodDetailsObject` schema quoted in §5: the required top-level `Elements` array, the `AllocatedRatio` (Required) / `UsageUnit` (Conditional) / `UsageQuantity` (optional) entry keys, the requirement that ratios sum to 1 across a single origin charge, and the `x_` + PascalCase rule for custom properties. **This corrects the previous version of this lesson**, which gave the object as a flat `{AllocatedRatio, UsageUnit, UsageQuantity}` map.
- **`AllocatedResourceId` column definition** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/specification/datasets/cost_and_usage/columns/allocatedresourceid.md> — the nullability rules that make the unallocated remainder a first-class row: null when not part of a split, **null on the unallocated portion**, non-null on an allocated portion, and locally unique within `(ResourceId, ChargePeriod)`.
- **`AllocatedMethodId` and `AllocatedServiceName` column definitions** — same directory — the requirement that method documentation reference a single `AllocatedMethodId` value, and `AllocatedServiceName`'s **Version Introduced: 1.5** with its rationale (origin `ServiceName` is preserved on allocated rows, so the consuming service needed its own column). **This corrects the previous version of this lesson**, which folded `AllocatedServiceName` into the 1.3 block.
- **`DataGeneratorCalculatedSplitCostAllocationHandling` attribute** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/specification/attributes/data_generator_calculated_split_cost_allocation_handling.md> — the three normative requirements in §5: dimensions match the origin, non-summable metrics match the origin, and **summable metrics sum across allocated charges to the origin charge**. Introduced 1.3.
- **"Data Generator-Calculated Split Cost Allocation" supported feature** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/specification/supported_features/data_generator_calculated_split_cost_allocation.md> — the feature's scoping to "compute nodes in a shared cluster (Kubernetes, databases)", its dependent-column list, and the example SQL that queries 1, 3 and 6 in §10 are adapted from — including the `JSON_TABLE(… '$.Elements[*]' …)` extraction pattern.
- **Split-cost-allocation appendix example** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/specification/appendix/split_cost_allocation_examples.md> — the $100.00 / 40-35-25 vCPU-hour split across three consuming services, the zeroed origin row, origin-dimension preservation, and the explicit note that the spec does not prescribe whether the origin row is zeroed, omitted, or offset.
- **Contract-commitments appendix, Scenario 2** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/specification/appendix/contract_commitments/contract_commitments.md> — the "H100 GPU Reservation - Q1" example with `CC Unit = GPU-Hours`, quantity 5,000, cost $250,000, 30% discount, 3-month duration, All Upfront, Continuous model, Monthly fulfilment interval, Negotiated offer. Confirms the `GPU-Hours` unit spelling; the implied $50/GPU-Hour is synthetic.
- **`UnitFormat` attribute** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/specification/attributes/unit_format.md> — §4's rules: hyphenated compound units in `<singular>-<plural>` form, slash-separated ratio units, the eight allowed time base units, the recommended count base units (including `Token`), and the "capitalised noun" allowance that makes `GPU` a valid base unit.
- **`CustomColumnHandling` and `FocusColumnHandling` attributes** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/tree/main/specification/attributes> — the `x_` prefix requirement, PascalCase and alphanumeric recommendations, the ≤50-character guidance, and the `Id`/`Name` suffix conventions that the schema in §8 follows. Both were revised in 1.4 when the old `ColumnHandling` attribute was removed.
- **Schema metadata** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/tree/main/specification/metadata/schema> and the worked example in `specification/appendix/metadata_examples/schema_metadata_example.md` — `SchemaId`, `FocusVersion` (MUST match a published version and the version the dataset conforms to), `CreationDate`, `DatasetInstanceId`, `DataGeneratorVersion`, and the `ColumnDefinition` list with `ColumnName`, `DataType`, `NumericPrecision`, `NumberScale`, `StringMaxLength`, `StringEncoding`, `ProviderTagPrefixes`, `Deprecated` and `PreviousColumnName`. The JSON in §8 follows the appendix example's shape.
- **Specification overview — feature levels and validators** — <https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec/blob/main/specification/overview.md> — the mechanical derivation of Mandatory / Conditional / Recommended / Optional from `MUST` / conditional `MUST` / `SHOULD` / `MAY`, the BCP-14 keyword convention, and the FinOps Foundation's **FOCUS Validator** (<https://github.com/finopsfoundation/focus_validator>) as the reference conformance implementation, with the explicit statement that no particular tool is mandated.

**Deeper dives**

- **OpenCost `CustomCost` protobuf** — <https://github.com/opencost/opencost/tree/develop/protos/customcost> — a real OSS tool's FOCUS-annotated model (`billed_cost`, `list_cost`, `list_unit_price`, `usage_quantity`, `usage_unit`, `charge_category`, `effective_cost`, `commitment_discount_*`), showing that FOCUS alignment is already how third-party cost is modelled in the ecosystem — and containing at least one stale name ("Charge Subcategory") that illustrates why you verify against the spec repository rather than another tool's comments. Read at commit `30608b8`, 2026-08-17.
- **Lesson 01, four sharing regimes** — [01 — GPU cost attribution: four sharing regimes](01-attribution-models.md) — the non-invertibility argument that makes `x_SplitConfidence`/`x_SplitUncertainty` necessary, and the SM-slice-versus-memory-basis choice that sits behind `AllocatedMethodId`.
- **Module 04 capstone, per-pod attribution** — [04 L10 — Capstone: per-pod attribution](../../04-gpu-on-kubernetes/lessons/10-capstone-per-pod-attribution.md) — identity A, which the split-cost handling attribute independently promotes to a normative requirement, and the pod-resources snapshot that fills stream B in §9.
- **DCGM metrics reference** — <https://docs.nvidia.com/datacenter/dcgm/latest/> — `DCGM_FI_PROF_GR_ENGINE_ACTIVE`, the source for `x_UtilisationMean`, and the per-MIG-instance counter identity that makes `x_UtilisationScope = 'instance'` legitimate under MIG and impossible under time-slicing. *`docs.nvidia.com` is egress-blocked from this environment; the field semantics used here are those established in module 05 and are unchanged.*

---
Module backlink: [💰 11 — GPU cost and unit economics](../README.md)
