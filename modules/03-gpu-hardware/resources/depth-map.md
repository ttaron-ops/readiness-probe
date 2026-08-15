# Depth map — Module 03 · GPU hardware

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **Thin match, one strong chapter.** The source repo has no GPU-architecture track — its GPU
> material is observability-shaped. The exception is `gpu-observability/00`, which is a genuinely
> good "how GPUs actually work" primer written specifically so that metric semantics make sense.

| Lesson | Go deeper in | Why |
|---|---|---|
| 01 Execution model & utilisation | [`gpu-observability/00-mental-models`](https://github.com/harut8/system-design/blob/main/gpu-observability/00-mental-models.md) | SMs, warps, occupancy, and the vocabulary every DCGM field assumes — the cleanest short statement of why `GPU_UTIL` isn't utilisation |
| 01 Execution model & utilisation | [`python-mastery/00-cpu-execution-model`](https://github.com/harut8/system-design/blob/main/python-mastery/00-cpu-execution-model.md) | the CPU contrast: pipelines, speculation, and what "one instruction" costs — sharpens why the GPU model is different rather than just bigger |
| 03 Memory hierarchy & HBM | [`python-mastery/01-memory-hierarchy-and-caches`](https://github.com/harut8/system-design/blob/main/python-mastery/01-memory-hierarchy-and-caches.md) | the general memory-hierarchy argument (latency vs bandwidth, why locality decides throughput) that the roofline model rests on |
| 05 Precision & tensor cores | [`gpu-observability/00-mental-models`](https://github.com/harut8/system-design/blob/main/gpu-observability/00-mental-models.md) | how tensor-pipe activity is *measured*, which is the practical half of the precision story |
| 07 Capstone — cost per useful work | [`gpu-observability/05-gpu-allocation-and-utilization-efficiency`](https://github.com/harut8/system-design/blob/main/gpu-observability/05-gpu-allocation-and-utilization-efficiency.md) | the allocation-vs-utilisation framing, one module early — useful for setting up the capstone's denominator |
| 07 Capstone — cost per useful work | [`python-mastery/31-measurement-methodology`](https://github.com/harut8/system-design/blob/main/python-mastery/31-measurement-methodology.md) | **read before benchmarking anything** — warm-up, variance, and how to know a measured improvement is real |

## Nothing there for

Roofline analysis, generational architecture comparisons (Ampere → Hopper → Blackwell), HBM
bandwidth arithmetic, the CUDA software-stack version matrix. This module's content is not
duplicated anywhere in the source; NVIDIA's architecture whitepapers and the module's own
references remain the material.
