# 🔌 Checkpoint — 03 · GPU hardware fundamentals

The **completion gate**. Prove it with the [GPU Efficiency & Cost Report](practice/gpu-efficiency-report/)
and answer the probes cold. You've passed when you can, **unaided**:

## Pass criteria

- [ ] **1 · Interpret every `nvidia-smi` field**, and name the DCGM metric that reveals
      *useful* work (tensor-active / SM-occupancy / MFU) versus the misleading `GPU-Util`.
- [ ] **2 · Explain the 1-SM 100%-util trap** and which metric you'd fleet-alert on instead.
- [ ] **3 · Compute arithmetic intensity** for a matmul and for LLM decode, and place both
      on a roofline — stating which roof each is under and how far from it.
- [ ] **4 · Estimate the tokens/sec ceiling** for a 70B model at FP16 on an H100 from HBM
      bandwidth (≈24 tok/s) — and redo it at FP8.
- [ ] **5 · Give bytes/param and the throughput multiple** for FP32/TF32/BF16/FP16/FP8/INT8,
      and state FP8's effect on cost and batch size.
- [ ] **6 · Recite current memory/bandwidth/FP8 numbers** for A100/H100/H200/B200 and argue,
      for a stated workload, which SKU is cheaper **per unit of useful work**.
- [ ] **7 · Diagnose a driver/CUDA/NCCL version mismatch** from symptoms and state the
      compatibility rule that governs it.
- [ ] **8 · Ship the report** with a defensible cost-per-useful-work conclusion.

## Depth probes (answer cold)

- [ ] What does `nvidia-smi` GPU-Util actually measure, in one sentence, without the word "utilisation"?
- [ ] Define MFU and a good target.
- [ ] You're at 15% of peak TFLOPS but HBM is saturated — will a bigger GPU help? Why/why not?
- [ ] What is the ridge-point arithmetic intensity for an H100?
- [ ] Does a 70B model at FP16 fit on one H100? one H200? Show the math.
- [ ] Why does batching help decode but not a single long prefill?
- [ ] Why doesn't FP8 speed up a memory-bound elementwise kernel proportionally?
- [ ] Why is H200 (same compute die as H100) often the better inference buy?
- [ ] A CUDA 12.x app on a host with driver 525 — will it run?

## Interview-readiness proxy

- [ ] Given a workload description, state whether it's memory- or compute-bound and what you'd change first.
- [ ] Given two SKUs and their $/hr, argue which is cheaper per unit of useful work — with numbers.

## Answers / notes

_Record answers as you close each lesson; link the report evidence for items 1–8._
