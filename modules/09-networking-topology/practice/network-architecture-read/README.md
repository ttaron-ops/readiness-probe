# Network Architecture Read + Placement/Procurement Argument — Module 09 deliverable

A 3–5 page doc that reads a **real, published** GPU-cluster fabric, predicts where a job
bottlenecks, and argues placement + procurement in bandwidth and dollars. Finishable with
published architectures + a single **2-GPU `nccl-tests`** run — no big cluster needed.

## Contents

1. **Topology read.** Take the **Llama-3 24K-GPU RoCE cluster** (or the parallel Quantum-2 IB
   variant) and redraw its 3 Clos tiers with bandwidth/bisection labeled at each. State the
   **oversubscription ratio per tier** (pod = 1:1, aggregation = 1:7).
2. **Bottleneck prediction.** For a named job (e.g. a 2,048-GPU all-reduce-heavy training run),
   predict where it bottlenecks under **two placements** — within one 3,072-GPU pod vs spread
   across pods — and quantify the penalty using the 1:7 ratio.
3. **Placement argument.** Make the co-location case with concrete numbers ("keeping the job
   in-pod preserves ~X GB/s per GPU of bisection; spreading it caps effective all-reduce
   bandwidth at ~X/7").
4. **IB-vs-RoCE section.** For a procurement scenario, pick IB or RoCE/Spectrum-X and justify
   on latency, NCCL GB/s, PFC/ECN tuning risk, SHARP availability, cost, and lock-in.
5. **Oversubscription reasoning.** Argue what oversubscription ratio *this* workload can tolerate.
6. **Evidence appendix.** Output of a **2-GPU `nccl-tests all_reduce_perf`** run (single node,
   NVLink path) with a short note on how bus bandwidth would degrade over a NIC/fabric path vs
   NVLink — grounding the whole doc in one real measurement you can afford.

## Suggested layout

```
network-architecture-read/
├── read.md              # the 3–5 page doc (§1–5)
├── topology.png         # the redrawn 3-tier Clos with bisection labels
├── nccl-allreduce.log   # the 2-GPU nccl-tests output (evidence appendix)
└── README.md            # sources + how the bench was run
```

## Acceptance criteria (matches the [checkpoint](../../checkpoint.md))

- [ ] the Llama-3 (or IB variant) topology redrawn with per-tier bisection + oversubscription
- [ ] a bottleneck prediction for a named job under two placements, **quantified** with the ratio
- [ ] a placement argument using **real bandwidth numbers**, not intuition
- [ ] an IB-vs-RoCE verdict for a procurement scenario, defended on ≥4 axes
- [ ] an oversubscription-tolerance argument for the workload
- [ ] a 2-GPU `nccl-tests all_reduce_perf` capture in the evidence appendix

## Guardrails

- No multi-node cluster required — published architectures + one 2-GPU NCCL run suffice.
- Flag any $/optics figures as dated snapshots; the durable content is the *structure*
  (bisection, oversubscription, IB-tax-vs-RoCE-reuse).
- Publishable-by-default; scrub any real cluster specifics.
