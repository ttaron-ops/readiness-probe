# Staff design portfolio — Distributed systems module deliverable

A portfolio of **5–6 staff-level design write-ups** (2–4 pages each) on real GPU-platform prompts.
Each doubles as an interview artifact *and* the checkpoint evidence. The point is not to draw
boxes — it's to **bound each system**: state the guarantee, cost it, name the failure, and put a
number on the deciding tradeoff.

## The required shape of every write-up (the Lesson 08 framework)

1. **Requirements** — functional, non-functional (availability / latency SLO / consistency /
   durability RPO-RTO), and the **scale envelope**; what's out of scope.
2. **Back-of-envelope estimate** — the dominant resource from first principles (in GPU units where
   apt: GPU-seconds, HBM-GB, NVLink/RDMA bandwidth, weight-load egress). The estimate must *choose*
   the architecture.
3. **API + data model** — access patterns first, then schema/partition key.
4. **Design + scale it** — find the first bottleneck the numbers exposed; shard/cache/replicate
   deliberately.
5. **A "guarantees & non-guarantees" table** — the exact consistency/availability class (PACELC),
   what it promises, and what it explicitly does *not*.
6. **Failure & blast radius** — the gray/correlated failure modes and the containment
   (cells / shuffle-sharding / checkpoint interval / backpressure).
7. **Tradeoffs** — the axes chosen *and rejected*, named (consistency↔latency, throughput↔tail,
   fair-share↔utilisation↔starvation, durability↔write-latency, blast-radius↔efficiency).

## The prompt set (pick 5–6, one per "plane" at least)

| Prompt | Plane | Draws on |
|--------|-------|----------|
| GPU scheduler / quota & fair-share (Kueue-shaped) | control | L2, L5 |
| Distributed-training checkpoint store | training | L3, L6 |
| Inference gateway / router (SLO + KV-cache locality) | serving | L1, L4, L5 |
| Fleet metrics / telemetry pipeline | control/data | L7 |
| Model-weight distribution (cold-start thundering herd) | training/serving | L4, L6 |
| Distributed lock / leader election (→ etcd) | control | L1, L2 |

> Do at least one **contrast rep**: the same system (e.g. the checkpoint store) designed twice
> under different binding constraints — once for restart RTO (fast local + async), once for
> zero-loss durability (sync/quorum) — and write up how the architecture *and the named tradeoff
> axis* flip. This trains the "name the axis" reflex the checkpoint tests.

## Suggested layout

```
staff-design-portfolio/
├── gpu-scheduler.md
├── checkpoint-store.md        # includes the RTO-vs-durability contrast rep
├── inference-gateway.md
├── telemetry-pipeline.md
├── weight-distribution.md
└── README.md                  # index + the framework each write-up follows
```

## Acceptance criteria (matches the [checkpoint](../../checkpoint.md))

- [ ] 5–6 write-ups, each following the 8-step framework
- [ ] every write-up has a **guarantees & non-guarantees** table (with the PACELC class)
- [ ] every write-up has **one back-of-envelope** estimate that chooses the architecture
- [ ] every write-up has a **failure & blast-radius** section naming the containment
- [ ] at least one **contrast rep** showing the tradeoff axis flip under a different constraint
- [ ] at least one prompt from each plane (control / training / serving)

## Guardrails

- Pure analysis — no infra required; these are design docs.
- **Publishable-by-default** — a threaded set of GPU-platform design docs is a strong portfolio
  centerpiece for module 12; scrub any employer-specific numbers before posting.
