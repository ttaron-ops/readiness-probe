# 🚀 Checkpoint — 07 · Inference serving

The **completion gate**. Prove it with the [Cost-per-million-tokens characterization](practice/cost-per-token/)
from your own GPU run, and answer the probes cold. You've passed when you can, **unaided**:

## Pass criteria

- [ ] **1 · Size it cold.** Given a 70B, a GPU, and a TTFT target, derive the concurrency cap
      from the KV-cache budget and state the minimum GPU config (TP / FP8) — on a whiteboard,
      no tools. *(the interview question)*
- [ ] **2 · Produce the curves.** Hand over the **CPM-vs-batch** and **FP8-vs-FP16** plots
      from your own GPU run, with the SLO knee identified.
- [ ] **3 · Justify the operating point.** Explain why your final config's batch / `max-num-seqs`
      is chosen — quantitatively tying batch → TTFT → CPM.
- [ ] **4 · Defend autoscaling.** Explain why HPA-on-CPU fails, what signal you scale on, and
      the scale-to-zero cold-start tradeoff with a measured number.
- [ ] **5 · Make the quant call.** State when FP8 vs INT8 vs AWQ, and name the calibration-set risk.
- [ ] **6 · Pick the engine.** Given a workload (RAG/agents vs unique-prompt vs peak-NVIDIA vs
      K8s-platform), choose vLLM / SGLang / TRT-LLM / KServe and say why — and know **TGI is dead**.

## Depth probes (answer cold)

- [ ] Why does batching help decode but barely help prefill?
- [ ] Write the `VRAM = weights + KV + activations` budget for 70B-FP16 on 1×H100-80GB — does it fit?
- [ ] Per-token KV bytes for a 70B (state your assumptions); double `max-model-len` → what happens to concurrency?
- [ ] Why non-contiguous fixed-size KV blocks? What does copy-on-write buy a shared system prompt?
- [ ] Symptom of `gpu-memory-utilization` set too high? When is swap preemption worse than recompute?
- [ ] Why does CPM fall 50–100× from batch 1→256, and why is CPM meaningless without utilisation?
- [ ] Expected FP8 throughput multiplier on H100? Why can a generic calibration set hurt a code workload?
- [ ] Why is GPU-utilisation a bad scale signal for LLM serving? (tie to the 05 util-lie)

## Fail signals (avoid these)

- [ ] Quoting CPM without utilisation · scaling on GPU-util · treating KV cache as "just memory"
      rather than the concurrency cap · asserting FP8 savings without a measured accuracy check.

## Interview-readiness proxy

- [ ] You have benchmarked vLLM yourself and produced a cost-per-million-tokens curve against batch size.
- [ ] You have measured cold-start latency and the quantization saving on real hardware.

## Answers / notes

_Record answers as you close each lesson; link the CPM report + benchmark JSON + plots for items 1–6._
