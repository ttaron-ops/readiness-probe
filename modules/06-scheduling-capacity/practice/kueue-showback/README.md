# Kueue setup + per-queue showback — Module 06 deliverable

One repo, three joined artifacts — **all runnable on a GPU-less `kind` cluster with fake
`nvidia.com/gpu` extended resources.** Kueue's borrowing, cohorts, preemption, priorities,
and TAS all demonstrate correctly against extended-resource nodes, so no real GPUs are
needed.

## 1. A working Kueue setup (reproducible)

Checked-in manifests + a `Makefile`/script that stands it up on kind:
- 1+ `ResourceFlavor`, **2+ `ClusterQueue`s in a `Cohort` with borrowing**, `LocalQueue`s, priority classes.
- A **demonstrated preemption/reclaim**: a script that fills queue A idle, lets B borrow,
  then submits to A and captures the eviction event as A reclaims.
- A **gang-scheduling demo**: the L1→L2 deadlock-then-fix, scripted and reproducible.

## 2. Per-ClusterQueue showback report (the FinOps compounding)

*Extends `gpu-cost-operator`.* A small tool (Python or Go, reusing the operator's cost
model) that pulls Kueue `Workload`/`ClusterQueue` usage — admitted GPU-seconds per queue,
borrowed vs owned — and joins it to a $/GPU-hr rate to produce a per-queue showback table:

| Queue | Reserved quota | Actual usage | Borrowed | $ owed | Idle-quota cost |
|-------|---------------|--------------|----------|--------|-----------------|

This is the artifact that most differentiates you — it's the exact "charge teams for what
they used against what they reserved" story a FinOps-minded platform engineer tells.

## 3. Capacity + queue-design doc for a fixed 128-GPU fleet (the interview whiteboard)

Written up:
- **Quota split** for 3 research teams + 1 prod service, with the borrowing/preemption
  policy and its defense.
- A **fragmentation / effective-capacity calculation** (the L7 calculator applied to the
  128-GPU inventory).
- A **reserved/on-demand/spot commitment ladder** with blended cost and break-even utilisation.

## Suggested layout

```
kueue-showback/
├── manifests/            # ResourceFlavor, ClusterQueues, cohort, LocalQueues, priorities
├── scripts/              # stand-up + deadlock/gang demo + borrow-then-reclaim
├── showback/             # per-queue showback tool (reuses gpu-cost-operator cost model)
├── capacity/             # effective-capacity calculator (L7) + the 128-GPU design doc
└── README.md             # how to run it all on kind
```

## Acceptance criteria (matches the [checkpoint](../../checkpoint.md))

- [ ] `make`/script stands up Kueue on kind with 2+ ClusterQueues in a borrowing cohort
- [ ] the gang deadlock-then-fix and the borrow-then-reclaim-by-preemption both reproduce
- [ ] the showback tool emits a per-ClusterQueue table (reserved / used / borrowed / $ / idle-cost)
- [ ] the effective-capacity calculator runs on a 128-GPU inventory and shows the fragmentation gap
- [ ] the 128-GPU doc covers quota split + borrowing/preemption defense + commitment ladder

## Guardrails

- Everything runs locally on kind — no cloud credentials or real GPUs.
- Keep the showback tool importable — it folds back into `gpu-cost-operator` and feeds Module 11.
- No real cost rates or internal data committed (repo `.gitignore` guards these).
