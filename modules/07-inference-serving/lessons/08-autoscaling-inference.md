---
lesson: "07.8"
title: "Autoscaling inference: scale on queue depth"
module: "07"
concept: "Autoscaling inference: scale on queue depth"
status: not-started
est_time: "6h"
prev: "07-quantization-ops.md"
next: "09-model-loading-storage.md"
artifacts: []
sources: 7
---
# 07.8 · Autoscaling inference: scale on queue depth

> **Concept.** Autoscale LLM serving on the queue, not the accelerator. Scale replicas on `vllm:num_requests_waiting` (or KV-cache utilisation) via KEDA — never on CPU or GPU-utilisation — and treat scale-to-zero as a $-vs-SLO decision.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

07 gave you a *numerical-format* lever on CPM (quantization); this lesson gives you a *capacity* lever — how many replicas are serving right now, and whether any should exist at all. It sits between quantization (07), whose smaller checkpoints make cold starts cheaper, and model loading & storage (09), which supplies the real numbers behind the cold-start cost this lesson only sketches. Read this lesson's cold-start section as a preview: 09 is where "tens of seconds to minutes" becomes a measured, storage-tier-dependent budget.

## Why this matters

You are fluent in KEDA/HPA and Prometheus. The trap here is applying that muscle memory to LLM serving with the *wrong signal*. The default autoscaler reflex — scale on CPU, or "scale on GPU-utilisation, it's the GPU workload" — is actively wrong for LLM decode, and it costs you both ways: you either over-provision H100s that idle (CPM balloons) or you under-provision and blow your TTFT p99 SLO under a load spike.

The right signal is the one you already learned to read in 05: **queue depth** (`vllm:num_requests_waiting`) and **KV-cache utilisation** (`vllm:gpu_cache_usage_perc`). These are *leading* indicators of saturation. GPU-utilisation is a *lagging, saturating* one — the "util lie" from 05. Getting this right is a direct CPM lever: it lets you run each replica hot (high utilisation, low CPM) while still adding capacity *before* latency breaches, and it lets you scale to zero on idle endpoints to stop paying for GPUs that serve nothing at 3am.

## What's new here (calibration)

1. **Custom-metric autoscaling for LLMs specifically** — the queue-depth / KV-cache signals, wired through KEDA's Prometheus scaler, and *why* the obvious signals lie.
2. **The two-layer division of labour** — KEDA scales *replicas*; Cluster Autoscaler/Karpenter scales *GPU nodes*. Confusing these is the #1 reason "my ScaledObject does nothing."
3. **Scale-to-zero as an explicit $-vs-SLO tradeoff**, including cold-start decomposition and how to price it.

## Core concepts

### Why GPU-utilisation is a bad scale signal (the util-lie, from 05)

`DCGM_FI_DEV_GPU_UTIL` reports the fraction of the sample window in which *at least one* kernel was running. During autoregressive decode a kernel is almost always running, so this metric **pins near 100% even at batch size 1** — long before the replica is actually saturated. It tells you the GPU is *busy*, not whether it has *headroom*. It is also **lagging**: by the time any accelerator counter reflects overload, requests are already queued and TTFT has already climbed. Autoscaling on it either never triggers (it's always ~100%, so a >80% rule scales out constantly or never meaningfully) or reacts after the SLO is already violated. CPU-utilisation is worse still — for a GPU decode workload it is mostly idle and correlates with nothing you care about.

**Queue depth is the leading signal.** `vllm:num_requests_waiting` counts requests admitted to vLLM but not yet running — it rises the instant arrival rate exceeds the replica's service rate, *before* latency degrades. `vllm:gpu_cache_usage_perc` (KV-cache utilisation) is the complementary signal: when it approaches 1.0 the scheduler starts preempting/recomputing, which spikes latency. Scale on either; queue depth is the usual primary.

This is not a KEDA-ecosystem opinion. It is independently corroborated by three different vendors building three different inference stacks: the vLLM project itself, Google's own GKE guidance for HPA on GPU inference, and Red Hat's KServe/OpenShift path — see Perspectives and Real-world use cases below. When every major Kubernetes inference platform vendor converges on the same "don't scale on GPU-util" conclusion independently, that is a much stronger signal than any one of them saying it alone.

| Signal | Leading or lagging? | Good scale trigger? |
|---|---|---|
| CPU utilisation | irrelevant | No |
| GPU utilisation (`DCGM..GPU_UTIL`) | lagging + saturates at ~100% | **No** |
| `vllm:num_requests_waiting` (queue depth) | **leading** | **Yes (primary)** |
| `vllm:gpu_cache_usage_perc` (KV util) | leading | Yes (secondary/guard) |
| TTFT / TPOT p99 | lagging (the SLO you protect) | No — it's the thing you defend, not the trigger |

### The division of labour: KEDA vs Cluster Autoscaler / Karpenter

Two independent control loops, one stacked on the other:

- **KEDA scales *replicas* (pods).** It reads `vllm:num_requests_waiting` from Prometheus and drives a Deployment's replica count up/down (KEDA creates and manages an HPA under the hood, with an external metric). KEDA is also what enables **scale-to-zero** — plain HPA cannot go below 1. KEDA is a **CNCF-graduated project** and deliberately cloud-agnostic — worth stating explicitly when you justify the choice: "cloud-agnostic and CNCF-governed," not "because a vendor's docs told me to."
- **Cluster Autoscaler / Karpenter scales *nodes*.** When KEDA adds a replica and there is no GPU node with a free GPU, the new pod is `Pending` (unschedulable). The node autoscaler sees the pending GPU pod and **provisions a new GPU node**; when replicas scale back down and a node is empty, it deprovisions it. Google's GKE documentation frames the identical idea in its own vocabulary — GKE's HPA (pod-level) alongside GKE's node auto-provisioning (node-level) — the same two-loop split under different vendor names.

```
load spike → vllm:num_requests_waiting ↑
   → KEDA: +1 replica  → pod Pending (no free GPU)
      → Karpenter/CA: +1 GPU node  → pod schedules → serves traffic
```

If you configure KEDA but forget the node layer, replicas pile up `Pending` forever and nothing scales. If you configure Karpenter but scale on the wrong metric, you provision expensive GPU nodes for a signal that doesn't reflect real demand. You need both, wired to the *right* metric. (Karpenter's consolidation is also what actually captures the cost saving on scale-down — it bin-packs and removes idle GPU nodes.)

### The KEDA ScaledObject

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: vllm-llama8b
spec:
  scaleTargetRef:
    name: vllm-llama8b            # the Deployment
  minReplicaCount: 1              # set 0 for scale-to-zero (see below)
  maxReplicaCount: 8
  cooldownPeriod: 300            # wait before scaling *down* — guards against thrash
  pollingInterval: 15
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:                  # tame flapping; scale out fast, in slow
        scaleUp:   { stabilizationWindowSeconds: 0,   policies: [{ type: Pods, value: 2, periodSeconds: 30 }] }
        scaleDown: { stabilizationWindowSeconds: 300, policies: [{ type: Pods, value: 1, periodSeconds: 60 }] }
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-operated.monitoring.svc:9090
        metricName: vllm_num_requests_waiting
        query: sum(vllm:num_requests_waiting{service="vllm-llama8b"})
        threshold: "5"           # target avg queued reqs per replica — DERIVE this
```

The `threshold` is a **target value per replica**: KEDA's HPA math is `desiredReplicas = ceil(currentReplicas * currentMetric / threshold)`. A threshold of `5` means "hold ~5 queued requests per replica on average"; when the summed queue climbs past `5 × replicas`, it scales out. (The `prometheus` trigger type and its parameters — `serverAddress`, `metricName`, `query`, `threshold` — are documented in KEDA's own Prometheus scaler reference, the most authoritative primary source for this YAML shape; see References.)

### Deriving the queue-depth threshold from your TTFT SLO

Do not guess the threshold — measure it. It is the queue depth at which your **TTFT p99 SLO is about to break**, minus headroom:

1. **Load-test one replica** at increasing concurrency; record `vllm:num_requests_waiting` and TTFT p99 together (you have this tooling from 05).
2. **Find the knee** — the queue depth `Q*` at which TTFT p99 crosses your SLO (say 500 ms).
3. **Set threshold = Q* × safety_factor** (e.g. 0.5–0.7) so you scale out *before* the knee, leaving time for the pod-then-node cold start to complete.

Intuition via Little's law: `L = λ·W`. Queue wait `W_q` grows with queue length `L_q`; each queued request adds ≈ its prefill+scheduling time to the TTFT of requests behind it. If measurement shows TTFT p99 breaches at ~10 queued and your cold start is ~30s, pick a threshold around 5–6 so the new replica is serving before the queue reaches 10. The threshold is thus an SLO-derived number, not a folklore constant.

**The safety factor must account for a two-stage cold start**, and the gap between environments is easy to underestimate: KEDA's own reaction time (polling interval + HPA evaluation) is on the order of seconds, but *node provisioning* — a fresh GPU node coming up via Karpenter/Cluster Autoscaler — is tens of seconds to minutes. If you derived your safety factor from a `kind` test cluster where the "node" is already there, you measured only the first stage. The real gap between a local dev loop and a production GPU cluster is often **10–100×**, and it is invisible until you test against a cluster that actually has to provision a GPU node from zero.

### Scale-to-zero: the $-vs-SLO decision

Set `minReplicaCount: 0` and KEDA parks the Deployment at zero replicas when the queue stays empty (an activation trigger wakes it on the first request). This **stops paying for idle GPU-hours** — decisive for spiky, bursty, or dev/internal endpoints that sit idle most of the day. An H100 at ~$3/hr left running overnight is ~$36 of pure waste per endpoint per night.

The cost is **cold-start latency** on the first request after idle:

```
cold start ≈ node provision (Karpenter, if no warm GPU node)   ~30–90s
           + image/weights pull (multi-GB, unless cached)        ~10–60s
           + model load into VRAM + CUDA/graph warmup            ~10–40s
```

That is tens of seconds to minutes of TTFT on the wake-up request — unacceptable for an interactive p99 SLO, fine for batch/async. **The decision is explicit:** scale-to-zero on endpoints where idle savings beat the occasional cold-start hit (internal tools, off-peak, async jobs); keep `minReplicaCount ≥ 1` (a warm floor) on latency-critical, always-on traffic. Mitigations that move the line: pre-pulled/cached images, node pre-warming / a warm GPU node pool, and smaller/quantized weights (07 — FP8 halves the VRAM you must load, shrinking the load step). Price it: `idle_savings = idle_hours × GPU_$/hr` vs `cold_start_SLO_cost × wake_events`. The full, measured version of this decomposition — with real bandwidth numbers per storage tier — is 09; this lesson gives you the shape of the tradeoff, 09 gives you the numbers.

**This is also a per-endpoint decision, not a platform-wide one.** The same platform can run a customer-facing chat endpoint with `minReplicaCount: 2` and a nightly internal-tools endpoint with `minReplicaCount: 0` — the $-vs-SLO answer depends on that endpoint's traffic shape and who's waiting on the other end, not on a single organization-wide policy.

## Perspectives

**Multi-cloud validation.** The "don't scale on GPU-util" thesis isn't a single vendor's opinion — it's converged wisdom across every major Kubernetes inference platform. The vLLM production-stack docs recommend queue-depth. Google's own GKE guidance, published independently, states plainly that "the default metrics for autoscaling are CPU or memory utilization... for inference servers these metrics are no longer a good sole indicator," and recommends queue size instead. Red Hat's KServe+KEDA path reaches the same conclusion for OpenShift. Three vendors, three different platforms, one answer — that convergence is stronger evidence than any single source's recommendation.

**Two-loop systems view.** KEDA-vs-Karpenter is one instance of a pattern you'll see under different names everywhere: a fast, metric-driven loop that decides "more capacity is needed" stacked on a slower, infrastructure-driven loop that decides "here is where that capacity comes from." Google's GKE terminology (HPA vs. node auto-provisioning) is the identical split with different labels. Recognizing this as a *pattern*, not a KEDA-specific quirk, is what lets you reason about the same failure mode ("the top loop asked for capacity the bottom loop can't produce fast enough") in any autoscaling system you meet later.

**Cost-vs-SLO tradeoff view.** Scale-to-zero viability is fundamentally a storage question in disguise: the entire decision hinges on how long the cold start takes, and that number is set almost entirely by where the weights live (09), not by GPU choice or KEDA configuration. This lesson gives you the framework for the decision; treat the actual go/no-go verdict as provisional until you have 09's measured cold-start numbers for your specific model and storage tier.

**Vendor-neutral foundation view.** KEDA is a CNCF project (graduated), governed independently of any single cloud vendor. That matters as more than trivia: when you justify a platform choice in a design review, "cloud-agnostic, CNCF-governed, and works identically on GKE/EKS/AKS/bare metal" is a materially stronger argument than "our cloud rep recommended it." It's also why the same ScaledObject YAML works whether the GPU nodes underneath are Google's, AWS's, or on-prem.

## Real-world use cases

- **Google Cloud Blog — "Tuning the GKE HPA to run inference on GPUs"** (Oct 2024, dated snapshot): [cloud.google.com/blog/products/containers-kubernetes/tuning-the-gke-hpa-to-run-inference-on-gpus](https://cloud.google.com/blog/products/containers-kubernetes/tuning-the-gke-hpa-to-run-inference-on-gpus). Independently corroborates the util-lie thesis from a different cloud vendor's own inference-server benchmarking, explicitly recommending queue size over CPU/GPU utilization for throughput and tail-latency control.
- **Google Cloud — "Scale to zero using KEDA" (GKE tutorial)**: [docs.cloud.google.com/kubernetes-engine/docs/tutorials/scale-to-zero-using-keda](https://docs.cloud.google.com/kubernetes-engine/docs/tutorials/scale-to-zero-using-keda). A concrete, runnable walkthrough of the scale-to-zero half of this lesson on a managed Kubernetes platform.
- **vLLM production-stack — "Autoscaling with KEDA"**: [docs.vllm.ai/projects/production-stack/en/latest/use_cases/autoscaling-keda.html](https://docs.vllm.ai/projects/production-stack/en/latest/use_cases/autoscaling-keda.html). The canonical ScaledObject wired directly to `vllm:num_requests_waiting`, from the project whose metrics this lesson is built around.
- **CNCF Blog — "GPU autoscaling on Kubernetes with KEDA: Building an external scaler"** (2026, dated snapshot): [cncf.io/blog/2026/05/27/gpu-autoscaling-on-kubernetes-with-keda-building-an-external-scaler](https://www.cncf.io/blog/2026/05/27/gpu-autoscaling-on-kubernetes-with-keda-building-an-external-scaler/). A vendor-neutral, foundation-level write-up (from KEDA's own home foundation) of why default K8s autoscaling metrics don't see the GPU at all, and how to build a custom scaler that does — a good next-step read once the Prometheus-scaler pattern in this lesson feels routine.

## Worked example

Goal for the deliverable: a **KEDA queue-depth autoscaling demo + a measured scale-out reaction time.** Runs on `kind` with a vLLM replica (CPU/tiny model, or point at the rented GPU).

**1. Install the pieces:**
```bash
kind create cluster --name infer
helm repo add kedacore https://kedacore.github.io/charts && helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install keda kedacore/keda -n keda --create-namespace
helm install prom prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl apply -f vllm-deploy.yaml        # Deployment + Service exposing :8000/metrics
```

**2. Scrape vLLM** — a `ServiceMonitor` (or PodMonitor) so `vllm:num_requests_waiting` lands in Prometheus. Confirm in the Prometheus UI before wiring KEDA; a ScaledObject on a non-existent metric silently never scales.

**3. Apply the `ScaledObject` above** (`minReplicaCount: 1`, `maxReplicaCount: 8`, `threshold: "5"`).

**4. Drive load up, then down**, and timestamp the reaction:
```bash
# ramp concurrency to overwhelm one replica
vllm bench serve --model <m> --host <svc> --port 8000 \
  --dataset-name random --num-prompts 4000 --max-concurrency 200 &

# watch the loop react
kubectl get scaledobject,hpa,deploy -w
watch -n2 'kubectl get pods -l app=vllm-llama8b'
```
Record three timestamps: **t0** queue crosses threshold → **t1** HPA bumps `desiredReplicas` → **t2** new pod `Ready`/serving. **Scale-out reaction time = t2 − t0.** On `kind` with a warm image expect single-digit seconds; on a real GPU cluster where a node must be provisioned, tens of seconds to minutes — which is exactly the number that justifies (or rules out) scale-to-zero for a given SLO, and exactly the 10–100× gap flagged above. Then stop the load and watch scale-in respect `cooldownPeriod`.

**5. Record for the deliverable:** the ScaledObject YAML, a plot of `num_requests_waiting` vs replica count over time, and the measured **t2 − t0** reaction time — labelled clearly as `kind`-measured vs GPU-cluster-measured if you have both.

## Practice

Feeds the **Cost-per-million-tokens** deliverable at `../practice/cost-per-token/README.md`.

- Stand up `kind` + KEDA + a vLLM replica (or target the rented GPU), scrape `vllm:num_requests_waiting` into Prometheus, and apply a `ScaledObject` triggered on it.
- Drive load up and down; capture queue depth vs replicas and the **scale-out reaction time** (t2 − t0).
- (Stretch) flip `minReplicaCount` to `0`, measure the cold-start wake latency, and write the one-line $-vs-SLO verdict for this endpoint.

**Acceptance:** a working **KEDA queue-depth autoscaling demo** plus a **measured scale-out reaction time**, captured for the deliverable. Scaling on CPU or GPU-utilisation does not pass; the trigger must be `vllm:num_requests_waiting` (or KV-cache util).

## Common pitfalls

- **Scaling on GPU or CPU utilization.** The central mistake this lesson exists to prevent — now independently corroborated by Google's own GKE guidance, not just the KEDA/vLLM ecosystem. If your ScaledObject's `metricName` is anything DCGM- or CPU-derived, stop and re-derive it from queue depth.
- **Configuring KEDA without the node-autoscaler layer, or vice versa.** The classic "my ScaledObject does nothing" failure: replicas requested by KEDA pile up `Pending` forever because no node autoscaler is watching for unschedulable GPU pods — or a node autoscaler is running but nothing is telling it (via KEDA) that more replicas are needed in the first place.
- **Guessing the queue-depth threshold instead of deriving it from a load test.** A threshold picked "because 5 sounded reasonable" has no relationship to your actual TTFT SLO. Always derive it from the Little's-Law-style knee-finding exercise in Core concepts.
- **Setting the safety factor from a `kind`-only test.** KEDA's reaction time on a local cluster with warm nodes measures only the pod-scheduling stage, not node provisioning. The real production gap can be 10–100× larger; validate the safety factor against a cluster that actually provisions GPU nodes before trusting it.
- **Treating scale-to-zero as an all-or-nothing platform decision.** It's a per-endpoint $-vs-SLO call. A customer-facing endpoint and an internal batch endpoint on the same platform can — and usually should — have different `minReplicaCount` policies.

## Self-check

- **(a) Why is GPU-utilisation a bad / lagging scale signal for LLM serving (tie to the util-lie from 05)?**
  **Answer:** `DCGM..GPU_UTIL` only reports whether a kernel ran during the sample window. In autoregressive decode a kernel almost always runs, so it **pins near 100% even at batch 1** — it shows the GPU is busy, not whether it has headroom, so it can't distinguish a healthy replica from a saturated one. It's also **lagging**: any accelerator counter reflecting overload appears only after requests are already queued and TTFT has already risen. Queue depth (`vllm:num_requests_waiting`) is leading — it rises the moment arrival rate exceeds service rate, before latency degrades. This conclusion is independently reached by the vLLM project, Google's GKE team, and Red Hat's KServe team — not a single vendor's opinion.

- **(b) What queue-depth threshold protects your TTFT p99 SLO, and how would you derive it?**
  **Answer:** The per-replica queue depth just *below* the knee where TTFT p99 breaches. Derive it empirically: load-test one replica at rising concurrency, record `num_requests_waiting` and TTFT p99 together, find the queue depth `Q*` where p99 crosses the SLO, then set `threshold = Q* × 0.5–0.7` so you scale out with enough lead time for the pod-then-node cold start to finish before the knee. It's an SLO-derived, measured number (KEDA scales when the summed queue exceeds `threshold × replicas`), never a folklore constant — and the safety factor must be validated against real node-provisioning time, not just `kind`-cluster pod scheduling.

- **(c) What's the division of labour between KEDA and Cluster Autoscaler / Karpenter?**
  **Answer:** **KEDA scales replicas (pods)** off the custom Prometheus metric (via an HPA it manages) and enables scale-to-zero. **Cluster Autoscaler / Karpenter scales nodes**: when KEDA adds a replica and no GPU node has a free GPU, the pod is `Pending`, and the node autoscaler provisions a GPU node (and deprovisions/consolidates idle ones on scale-down). Two stacked loops — replicas on top, GPU nodes underneath. Configure only KEDA and replicas pile up `Pending`; configure only the node layer and nothing tells it to grow. GKE's own HPA-plus-node-auto-provisioning is the same split under different vendor terminology.

- **(d) Why can a threshold that works perfectly on a `kind` test cluster fail in production?**
  **Answer:** On `kind` the "node" already exists — KEDA's reaction (polling interval + HPA math + pod scheduling) is the entire observed latency, typically single-digit seconds. In production, if no GPU node has a free slot, the node autoscaler must provision an entirely new GPU node first — tens of seconds to minutes. That gap is often 10–100× larger than the `kind`-measured number, and a safety factor derived only from local testing leaves no headroom for it, so the replica arrives well after the queue has already blown the SLO.

- **(e) Is scale-to-zero a single yes/no decision for a platform? Why or why not?**
  **Answer:** No — it's a per-endpoint $-vs-SLO tradeoff. The saving is idle-GPU-hours avoided; the cost is cold-start latency on the wake-up request. An internal or off-peak/batch endpoint can tolerate that latency and should scale to zero; a customer-facing, latency-critical endpoint usually can't and should keep `minReplicaCount ≥ 1`. The same platform legitimately runs both policies simultaneously, decided per endpoint based on its own traffic pattern and who is waiting on the response.

## Connections & what's next

This lesson gave you the *shape* of the scale-to-zero decision — idle savings vs. cold-start cost — and the KEDA/Karpenter machinery to act on it. What it deliberately left approximate is the cold-start number itself: "tens of seconds to minutes" is a placeholder that 09 (Model loading & storage) replaces with a measured, storage-tier-dependent budget, because it turns out the GPU and the autoscaler are not what determines whether scale-to-zero is viable — where the weights live is. Carry the KEDA ScaledObject and the scale-out reaction time you measured here forward into 09's cold-start breakdown; they're two halves of the same number.

## References & further reading

**Primary sources**
1. **KEDA — Prometheus scaler docs** — the authoritative reference for the `prometheus` trigger type, `serverAddress`/`metricName`/`query`/`threshold` semantics used in the ScaledObject above: https://keda.sh/docs/2.19/scalers/prometheus/
2. **KServe — Autoscaling docs (KPA & HPA)** — the KServe-native autoscaler paths, relevant since 06 introduced KServe as the platform layer these engines can run under: https://kserve.github.io/website/docs/model-serving/predictive-inference/autoscaling/kpa-autoscaler · https://kserve.github.io/website/docs/model-serving/predictive-inference/autoscaling/hpa-autoscaler
3. **vLLM production-stack — "Autoscaling with KEDA"** — canonical ScaledObject on `vllm:num_requests_waiting` with the Prometheus trigger and Helm integration: https://docs.vllm.ai/projects/production-stack/en/latest/use_cases/autoscaling-keda.html

**Real-world engineering blogs**
4. **Google Cloud Blog — "Tuning the GKE HPA to run inference on GPUs"** (dated snapshot, Oct 2024) — independent, cross-vendor corroboration of queue-size over utilization metrics: https://cloud.google.com/blog/products/containers-kubernetes/tuning-the-gke-hpa-to-run-inference-on-gpus
5. **Google Cloud — "Scale to zero using KEDA"** (GKE tutorial) — a runnable walkthrough of this lesson's scale-to-zero half: https://docs.cloud.google.com/kubernetes-engine/docs/tutorials/scale-to-zero-using-keda
6. **Red Hat Developer — "How to set up KServe autoscaling for vLLM with KEDA"** — the KServe + KEDA + custom-metric path end to end, with the node-autoscaler interplay: https://developers.redhat.com/articles/2025/09/23/how-set-kserve-autoscaling-vllm-keda

**Deeper dives**
7. **CNCF Blog — "GPU autoscaling on Kubernetes with KEDA: Building an external scaler"** (dated snapshot, 2026) — vendor-neutral, foundation-level next step: building a custom KEDA external scaler on raw GPU telemetry (NVML/DCGM) rather than a pre-built Prometheus metric: https://www.cncf.io/blog/2026/05/27/gpu-autoscaling-on-kubernetes-with-keda-building-an-external-scaler/
