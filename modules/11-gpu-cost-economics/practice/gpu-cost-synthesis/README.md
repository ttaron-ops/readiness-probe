# GPU cost synthesis — Module 11 deliverable (the signature artifact)

This is the capstone of the whole cost thread. It fuses everything from `01` (the
`gpu-cost-exporter`) through `11` into **one publishable body of work** that reads as
staff-level FinOps-for-GPUs evidence. Three artifacts, all finishable with the operator you
already built + published data — **no large GPU fleet required**.

## A) The finished `gpu-cost-operator` — both ledgers, attribution-aware

Take the `gpu-cost-operator` you carried from `02` → `04` and finish it so it emits, per
namespace/pod/team:

- **allocated** GPU-hours (from pod-resources API device binding — ref 04) **and**
- **utilised** GPU-hours (allocated × SM_ACTIVE fraction from DCGM — ref 05, *not* GPU_UTIL),
- tagged with the **attribution regime** (exclusive / MIG-fraction / time-sliced / DRA — L1),
- joined to an **application counter** (tokens ref 07, or runs ref 08) to emit a **unit cost**.

It must **degrade honestly**: on a time-sliced device it emits the approximation *and flags it
as unattributable from driver signals* (L1) rather than reporting a false precise number.

## B) "Where every GPU cost tool fails" — the sourced writeup

A publishable 3–5 page teardown, grounded in **source you actually read** (L9):

- Read OpenCost's `pkg/costmodel/costmodel.go` (+ `allocation.go`, `pkg/cloud/`); state, with
  the exact logic, that its GPU cost is **request-based whole-GPU allocation cost** — blind to
  MIG fractions, time-slice sharers, and DCGM utilisation.
- Name the **exact gap** and what it would take to close it (the DCGM × pod-resources × app-counter
  join your operator does).
- Cover the landscape (Kubecost, DCGM-exporter+Grafana, cloud billing granularity) and why none
  join *$ × utilisation × business unit*.

## C) The FOCUS-aligned GPU cost schema (L10)

A concrete schema — real **FOCUS 1.x** column names (BilledCost/EffectiveCost/ListCost,
ConsumedQuantity/UnitPrice, CommitmentDiscount*, the split/allocated-cost columns) — extended
with the **utilisation ledger FOCUS lacks** and the **attribution-regime** dimension. Show how
each row is produced from DCGM + pod-resources + billing, and name where GPU sharing **still**
doesn't map (the L1 time-slicing hole).

## Suggested layout

```
gpu-cost-synthesis/
├── operator/            # the finished gpu-cost-operator (allocated + utilised + unit cost)
│   ├── README.md        # what it emits, the metrics schema, how it degrades on time-slicing
│   └── ...              # controller / exporter code (carried from 02→04)
├── writeup/
│   ├── tooling-gaps.md  # "where every GPU cost tool fails" — OpenCost source-grounded (L9)
│   └── focus-schema.md  # the FOCUS-aligned schema + the utilisation extension (L10)
├── demo/                # sample dashboards / a chargeback-showback statement (L08)
└── README.md            # how it all fits + how to reproduce
```

## Acceptance criteria (matches the [checkpoint](../../checkpoint.md))

- [ ] operator emits **both** allocated and utilised GPU-hours per namespace/pod, regime-tagged
- [ ] operator joins to an app counter and emits a **unit cost** ($/1M tokens or $/run)
- [ ] operator **degrades honestly** on time-sliced devices (flags, doesn't fabricate)
- [ ] the OpenCost gap named **from the source**, with the exact fix
- [ ] a **FOCUS 1.x-aligned** schema with real column names + a utilisation ledger extension
- [ ] a monthly **chargeback/showback statement** for ≥3 teams (allocated vs utilised, blended rate)

## Guardrails

- No big fleet needed — the operator runs against DCGM on any GPU (even 1) or replayed/published
  metrics; the writeup and schema are pure analysis.
- Flag every $/rate figure as a dated 2026 snapshot; the durable content is the **method** (two
  ledgers, regime-aware attribution, the FOCUS mapping).
- **Publishable-by-default** — this is the portfolio centerpiece for `12`. Scrub any
  employer-specific rates or cluster specifics before posting.
