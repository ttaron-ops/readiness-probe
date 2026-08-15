# Depth map — Module 07 · Inference serving

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **Weak on serving internals, strong on the two things around it:** how to *observe* an inference
> service, and how to *measure* it honestly. The source has no vLLM/PagedAttention material —
> that's this module's own.

| Lesson | Go deeper in | Why |
|---|---|---|
| 01 Inference workload shape | [`gpu-observability/14-llm-inference-observability`](https://github.com/harut8/system-design/blob/main/gpu-observability/14-llm-inference-observability.md) | prefill vs decode as seen from the metrics, and the instrumentation points |
| 01 Inference workload shape | [`gpu-observability/04-batch-vs-stateless-workloads`](https://github.com/harut8/system-design/blob/main/gpu-observability/04-batch-vs-stateless-workloads.md) | why serving and batch need different signals — sharpens the "shape" argument |
| 05 Batching economics | [`python-mastery/31-measurement-methodology`](https://github.com/harut8/system-design/blob/main/python-mastery/31-measurement-methodology.md) | **the most valuable chapter for this module.** Warm-up, variance, confidence, and how to know a CPM improvement is real. Your cost-per-token curve is a benchmark, and an unrigorous benchmark is the fastest way to lose credibility on a published chart. |
| 05 Batching economics | [`sre-observability/13-slo-engineering`](https://github.com/harut8/system-design/blob/main/sre-observability/13-slo-engineering.md) | the SLO discipline behind "the TTFT-p99 knee" — how to set the constraint you optimise under |
| 08 Autoscaling inference | [`kubernetes/22-autoscaling`](https://github.com/harut8/system-design/blob/main/kubernetes/22-autoscaling.md) | HPA/VPA/CA/Karpenter/KEDA internals — scale-to-zero mechanics and the metrics pipeline behind custom-metric scaling |
| 09 Model loading & storage | [`kubernetes/02-container-images-and-registries`](https://github.com/harut8/system-design/blob/main/kubernetes/02-container-images-and-registries.md) | layers, registries, and pull-time — the dominant term in a multi-GB cold start |
| 09 Model loading & storage | [`kubernetes/19-storage-csi-pv-pvc`](https://github.com/harut8/system-design/blob/main/kubernetes/19-storage-csi-pv-pvc.md) | volume lifecycle and access modes for shared weight volumes |
| 09 Model loading & storage | [`kubernetes/43-python-containers-with-uv`](https://github.com/harut8/system-design/blob/main/kubernetes/43-python-containers-with-uv-performance-and-cold-start.md) | a focused treatment of Python image size and cold start — directly applicable to a vLLM image |
| 10 Multi-model / LoRA | [`sre-observability/26-llm-and-ai-observability`](https://github.com/harut8/system-design/blob/main/sre-observability/26-llm-and-ai-observability.md) | per-model, per-tenant token accounting — how you attribute cost when N adapters share one base |

## Adjacent, and a judgement call: the RAG track

[`ai-rag/`](https://github.com/harut8/system-design/tree/main/ai-rag) is 9 chapters plus runnable
labs on embeddings, chunking, vector indexes, hybrid retrieval, reranking, and — the strongest
part — [evaluation methodology](https://github.com/harut8/system-design/blob/main/ai-rag/08-evaluation-methodology.md)
(golden sets, recall@k, nDCG, LLM-as-judge calibration, CI regression gates).

**This is out of scope for a GPU-platform role and deliberately not imported.** Read it only if you
end up targeting *inference-platform* roles where the team owns the retrieval pipeline as well as
the serving layer. If you do, two chapters transfer regardless of RAG:

- [`appendix-e-deployment-and-compute`](https://github.com/harut8/system-design/blob/main/ai-rag/appendix-e-deployment-and-compute.md)
  — per-request, per-tenant, per-model token cost attribution. That is Module 11's problem in a
  different costume.
- [`08-evaluation-methodology`](https://github.com/harut8/system-design/blob/main/ai-rag/08-evaluation-methodology.md)
  — the general lesson (build a golden set, gate on it in CI, don't eyeball outputs) applies to
  your FP8-vs-FP16 accuracy delta in Lesson 07, where "measured, not asserted" is the whole claim.

Note the track is incomplete — its own index promises chapters 05–07 and 09–14 that don't exist yet.
