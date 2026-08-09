# 🎓 Checkpoint — 12 · Capstone & the whole-course readiness gate

This is the **final gate for the entire program**. Prove it with the
[GPU platform capstone](practice/gpu-platform-capstone/) and a **timed mock loop** (lesson 09).
You pass only when you can, **unaided and timed**, clear all five dimensions:

## Pass criteria (the readiness gate)

- [ ] **1 · System design.** Take any of **P1–P6** and cover all of that prompt's probe axes
      **while volunteering** scale, cost, failure modes, and SLOs — no hand-waving the
      GPU-specific layer (isolation model, gang scheduling, KV cache, lemon nodes, util-vs-goodput).
- [ ] **2 · Debugging.** Run **D2** ("GPUs show 100% util but throughput dropped") end-to-end,
      correctly pivoting util → MFU/goodput and bisecting compute / comms / data.
- [ ] **3 · Behavioral (staff).** Deliver **3 staff-level STAR stories**, each landing a **metric**
      and a **tradeoff/reversibility** note, and each reading as **multi-team scope** — not heroics.
- [ ] **4 · Artifact narration.** Walk the full capstone **value chain in 5 minutes**, connecting
      any single artifact to a **decision, a tradeoff, and a number**.
- [ ] **5 · Portfolio.** The repo + RFC design doc + flagship blog are **published and
      self-explanatory** to a hiring manager who never meets you.

## The prompts you must be able to take cold

- [ ] **P1** multi-tenant GPU platform · **P2** cost attribution/chargeback (your home field —
      volunteer it) · **P3** training-job scheduler · **P4** model-serving platform ·
      **P5** health detection & remediation · **P6** fleet observability.
- [ ] **D1** slow training job (data → synthetic-test → comms/`nsys` → sync stalls) ·
      **D2** 100%-util/low-goodput trap · **D3** lemon node that passes idle health checks.

## Depth probes (answer cold)

- [ ] "Bill 4 pods time-slicing one A100" — why is every answer an approximation? (ties 11)
- [ ] Why does K8s default scheduling silently hang a 4-rank job at 3 ranks? (gang; ties 06)
- [ ] `nvidia-smi` says 100% — what does that actually mean, and what do you look at instead? (05)
- [ ] MIG vs time-slicing vs passthrough — the isolation/utilisation tradeoff, with a verdict.
- [ ] Turn "co-locate this job in one pod" into a per-GPU bandwidth number. (ties 09)
- [ ] What's the staff tell in a behavioral answer vs the senior tell?

## Interview-readiness proxy

- [ ] Your capstone repo + design doc + flagship blog are live and self-explanatory.
- [ ] You have run a full timed mock loop and passed all five dimensions.
- [ ] You can *volunteer* the GPU cost-attribution design without being prompted.

## Fail signals

- [ ] Quotes a util number as ground truth · a STAR story with no metric or no tradeoff ·
      a system-design answer that never surfaces failure modes / cost / SLOs · a portfolio that
      needs you present to make sense.

## Answers / notes

_Record your mock-loop scorecard and remediation list here; link the capstone repo + blog + demo._
