# 💰 Checkpoint — 11 · GPU cost and unit economics

The **signature gate**. Prove it with the [GPU cost synthesis](practice/gpu-cost-synthesis/)
and answer the probes cold. You've passed when you can, **unaided**:

## Pass criteria

- [ ] **1 · Attribute across all four regimes.** Given a GPU-hour, split it correctly under
      **exclusive**, **MIG-fractional**, **time-sliced**, and **DRA** sharing — and state the
      signal each relies on (pod-resources binding, per-MIG-UUID DCGM, aggregate-device DCGM, ResourceClaim).
- [ ] **2 · Defend the time-slicing limit.** Explain **why** time-sliced GPU cost is *not*
      attributable from driver/DCGM signals, name the approximations (equal / weighted-by-request
      / CUDA-context runtime), and give each one's error source.
- [ ] **3 · Two ledgers.** Compute **allocated** vs **utilised** GPU-hours for a workload and the
      waste gap in dollars; say which ledger the bill lives on and which you *charge*.
- [ ] **4 · Idle safely.** State a DCGM idle rule (SM_ACTIVE threshold + window) and the
      **false-positive cost** of reclaiming a latency-serving pod vs a batch job.
- [ ] **5 · Fragmentation.** Compute a fragmentation ratio for a request-shape histogram and name
      the packing/gang/topology levers that recover stranded GPU-hours.
- [ ] **6 · Unit economics.** Turn attributed GPU-hours × blended rate ÷ app counter into
      **$/1M tokens** and **$/run**, and explain direct vs fully-loaded (waste-allocated) unit cost.
- [ ] **7 · Name the OpenCost gap from source.** Point to the request-based whole-GPU allocation
      logic in `pkg/costmodel/` and state exactly what it can't see (MIG / time-slice / utilisation).
- [ ] **8 · Design the FOCUS schema.** Present a FOCUS 1.x-aligned GPU cost schema (real column
      names) extended with the utilisation ledger FOCUS lacks and the attribution-regime dimension.

## Depth probes (answer cold)

- [ ] How do you bill 4 pods time-slicing one A100 — and why is every answer an approximation?
- [ ] Which DCGM field is the correct utilisation basis, and why not `DCGM_FI_DEV_GPU_UTIL`? (ties 05)
- [ ] Why is charging **allocated** GPU-hours (not utilised) the defensible chargeback default?
- [ ] A reserved GPU at 30% SM_ACTIVE vs on-demand at 90% — which is cheaper per unit of work?
- [ ] Decompose the hyperscaler-vs-neocloud 3–6× sticker gap into ≥4 real drivers.
- [ ] What does OpenCost do when a GPU is MIG-sliced 7×1g — and what's the true cost basis?
- [ ] Which FOCUS columns carry a GPU-hour charge, and where does GPU *sharing* still not map?
- [ ] What's the false-positive cost of reclaiming a "idle" vLLM pod holding a warm KV cache? (ties 03/07)

## Interview-readiness proxy

- [ ] Your `gpu-cost-operator` emits **both** allocated and utilised GPU-hours + a unit cost.
- [ ] You have a **source-grounded** writeup naming the exact gap in the leading OSS cost tool.
- [ ] You have a **FOCUS-aligned schema** you can whiteboard for "design a GPU fleet cost system".

## Fail signal

- [ ] Quotes a single $/GPU-hr sticker as "the cost" · reports utilisation OR allocation but not
      both · treats OpenCost's number as ground truth · claims time-sliced attribution is exact.

## Answers / notes

_Record answers as you close each lesson; link the operator + tooling-gaps + FOCUS-schema for items 1–8._
