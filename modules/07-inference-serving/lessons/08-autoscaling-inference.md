---
lesson: "07.8"
title: "Autoscaling inference: scale on queue depth"
module: "07"
concept: "Autoscaling inference: scale on queue depth"
status: not-started
est_time: "5h"
artifacts: []
---
# 07.8 · Autoscaling inference: scale on queue depth

> **Concept.** Autoscale LLM serving on the queue, not the accelerator. Scale replicas on `vllm:num_requests_waiting` (or KV-cache utilisation) via KEDA — never on CPU or GPU-utilisation — and treat scale-to-zero as a $-vs-SLO decision.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Why this matters

You are fluent in KEDA/HPA and Prometheus. The trap here is applying that muscle memory to LLM serving with the *wrong signal*. The default autoscaler reflex — scale on CPU, or "scale on GPU-utilisation, it's the GPU workload" — is actively wrong for LLM decode, and it costs you both ways: you either over-provision H100s that idle (CPM balloons) or you under-provision and blow your TTFT p99 SLO under a load spike.

The right signal is the one you already learned to read in 05: **queue depth** (`vllm:num_requests_waiting`) and **KV-cache utilisation** (`vllm:gpu_cache_usage_perc`). These are *leading* indicators of saturation. GPU-utilisation is a *lagging, saturating* one — the "util lie" from 05. Getting this right is a direct CPM lever: it lets you run each replica hot (high utilisation, low CPM) while still adding capacity *before* latency breaches, and it lets you scale to zero on idle endpoints to stop paying for GPUs that serve nothing at 3am.

## What's new here

1. **Custom-metric autoscaling for LLMs specifically** — the queue-depth / KV-cache signals, wired through KEDA's Prometheus scaler, and *why* the obvious signals lie.
2. **The two-layer division of labour** — KEDA scales *replicas*; Cluster Autoscaler/Karpenter scales *GPU nodes*. Confusing these is the #1 reason "my ScaledObject does nothing."
3. **Scale-to-zero as an explicit $-vs-SLO tradeoff**, including cold-start decomposition and how to price it.

## Core notes

### Why GPU-utilisation is a bad scale signal (the util-lie, from 05)

`DCGM_FI_DEV_GPU_UTIL` reports the fraction of the sample window in which *at least one* kernel was running. During autoregressive decode a kernel is almost always running, so this metric **pins near 100% even at batch size 1** — long before the replica is actually saturated. It tells you the GPU is *busy*, not whether it has *headroom*. It is also **lagging**: by the time any accelerator counter reflects overload, requests are already queued and TTFT has already climbed. Autoscaling on it either never triggers (it's always ~100%, so a >80% rule scales out constantly or never meaningfully) or reacts after the SLO is already violated. CPU-utilisation is worse still — for a GPU decode workload it is mostly idle and correlates with nothing you care about.

**Queue depth is the leading signal.** `vllm:num_requests_waiting` counts requests admitted to vLLM but not yet running — it rises the instant arrival rate exceeds the replica's service rate, *before* latency degrades. `vllm:gpu_cache_usage_perc` (KV-cache utilisation) is the complementary signal: when it approaches 1.0 the scheduler starts preempting/recomputing, which spikes latency. Scale on either; queue depth is the usual primary.

| Signal | Leading or lagging? | Good scale trigger? |
|---|---|---|
| CPU utilisation | irrelevant | No |
| GPU utilisation (`DCGM..GPU_UTIL`) | lagging + saturates at ~100% | **No** |
| `vllm:num_requests_waiting` (queue depth) | **leading** | **Yes (primary)** |
| `vllm:gpu_cache_usage_perc` (KV util) | leading | Yes (secondary/guard) |
| TTFT / TPOT p99 | lagging (the SLO you protect) | No — it's the thing you defend, not the trigger |

### The division of labour: KEDA vs Cluster Autoscaler / Karpenter

Two independent control loops, one stacked on the other:

- **KEDA scales *replicas* (pods).** It reads `vllm:num_requests_waiting` from Prometheus and drives a Deployment's replica count up/down (KEDA creates and manages an HPA under the hood, with an external metric). KEDA is also what enables **scale-to-zero** — plain HPA cannot go below 1.
- **Cluster Autoscaler / Karpenter scales *nodes*.** When KEDA adds a replica and there is no GPU node with a free GPU, the new pod is `Pending` (unschedulable). The node autoscaler sees the pending GPU pod and **provisions a new GPU node**; when replicas scale back down and a node is empty, it deprovisions it.

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

The `threshold` is a **target value per replica**: KEDA's HPA math is `desiredReplicas = ceil(currentReplicas * currentMetric / threshold)`. A threshold of `5` means "hold ~5 queued requests per replica on average"; when the summed queue climbs past `5 × replicas`, it scales out.

### Deriving the queue-depth threshold from your TTFT SLO

Do not guess the threshold — measure it. It is the queue depth at which your **TTFT p99 SLO is about to break**, minus headroom:

1. **Load-test one replica** at increasing concurrency; record `vllm:num_requests_waiting` and TTFT p99 together (you have this tooling from 05).
2. **Find the knee** — the queue depth `Q*` at which TTFT p99 crosses your SLO (say 500 ms).
3. **Set threshold = Q* × safety_factor** (e.g. 0.5–0.7) so you scale out *before* the knee, leaving time for the pod-then-node cold start to complete.

Intuition via Little's law: `L = λ·W`. Queue wait `W_q` grows with queue length `L_q`; each queued request adds ≈ its prefill+scheduling time to the TTFT of requests behind it. If measurement shows TTFT p99 breaches at ~10 queued and your cold start is ~30s, pick a threshold around 5–6 so the new replica is serving before the queue reaches 10. The threshold is thus an SLO-derived number, not a folklore constant.

### Scale-to-zero: the $-vs-SLO decision

Set `minReplicaCount: 0` and KEDA parks the Deployment at zero replicas when the queue stays empty (an activation trigger wakes it on the first request). This **stops paying for idle GPU-hours** — decisive for spiky, bursty, or dev/internal endpoints that sit idle most of the day. An H100 at ~$3/hr left running overnight is ~$36 of pure waste per endpoint per night.

The cost is **cold-start latency** on the first request after idle:

```
cold start ≈ node provision (Karpenter, if no warm GPU node)   ~30–90s
           + image/weights pull (multi-GB, unless cached)        ~10–60s
           + model load into VRAM + CUDA/graph warmup            ~10–40s
```

That is tens of seconds to minutes of TTFT on the wake-up request — unacceptable for an interactive p99 SLO, fine for batch/async. **The decision is explicit:** scale-to-zero on endpoints where idle savings beat the occasional cold-start hit (internal tools, off-peak, async jobs); keep `minReplicaCount ≥ 1` (a warm floor) on latency-critical, always-on traffic. Mitigations that move the line: pre-pulled/cached images, node pre-warming / a warm GPU node pool, and smaller/quantized weights (07.7 — FP8 halves the VRAM you must load, shrinking the load step). Price it: `idle_savings = idle_hours × GPU_$/hr` vs `cold_start_SLO_cost × wake_events`.

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
Record three timestamps: **t0** queue crosses threshold → **t1** HPA bumps `desiredReplicas` → **t2** new pod `Ready`/serving. **Scale-out reaction time = t2 − t0.** On `kind` with a warm image expect single-digit seconds; on a real GPU cluster where a node must be provisioned, tens of seconds to minutes — which is exactly the number that justifies (or rules out) scale-to-zero for a given SLO. Then stop the load and watch scale-in respect `cooldownPeriod`.

**5. Record for the deliverable:** the ScaledObject YAML, a plot of `num_requests_waiting` vs replica count over time, and the measured **t2 − t0** reaction time.

## Practice

Feeds the **Cost-per-million-tokens** deliverable.

- Stand up `kind` + KEDA + a vLLM replica (or target the rented GPU), scrape `vllm:num_requests_waiting` into Prometheus, and apply a `ScaledObject` triggered on it.
- Drive load up and down; capture queue depth vs replicas and the **scale-out reaction time** (t2 − t0).
- (Stretch) flip `minReplicaCount` to `0`, measure the cold-start wake latency, and write the one-line $-vs-SLO verdict for this endpoint.

**Acceptance:** a working **KEDA queue-depth autoscaling demo** plus a **measured scale-out reaction time**, captured for the deliverable. Scaling on CPU or GPU-utilisation does not pass; the trigger must be `vllm:num_requests_waiting` (or KV-cache util).

## Self-check

**(a) Why is GPU-utilisation a bad / lagging scale signal for LLM serving (tie to the util-lie from 05)?**

**Answer:** `DCGM..GPU_UTIL` only reports whether a kernel ran during the sample window. In autoregressive decode a kernel almost always runs, so it **pins near 100% even at batch 1** — it shows the GPU is busy, not whether it has headroom, so it can't distinguish a healthy replica from a saturated one. It's also **lagging**: any accelerator counter reflecting overload appears only after requests are already queued and TTFT has already risen. Queue depth (`vllm:num_requests_waiting`) is leading — it rises the moment arrival rate exceeds service rate, before latency degrades.

**(b) What queue-depth threshold protects your TTFT p99 SLO, and how would you derive it?**

**Answer:** The per-replica queue depth just *below* the knee where TTFT p99 breaches. Derive it empirically: load-test one replica at rising concurrency, record `num_requests_waiting` and TTFT p99 together, find the queue depth `Q*` where p99 crosses the SLO, then set `threshold = Q* × 0.5–0.7` so you scale out with enough lead time for the pod-then-node cold start to finish before the knee. It's an SLO-derived, measured number (KEDA scales when the summed queue exceeds `threshold × replicas`), never a folklore constant.

**(c) What's the division of labour between KEDA and Cluster Autoscaler / Karpenter?**

**Answer:** **KEDA scales replicas (pods)** off the custom Prometheus metric (via an HPA it manages) and enables scale-to-zero. **Cluster Autoscaler / Karpenter scales nodes**: when KEDA adds a replica and no GPU node has a free GPU, the pod is `Pending`, and the node autoscaler provisions a GPU node (and deprovisions/consolidates idle ones on scale-down). Two stacked loops — replicas on top, GPU nodes underneath. Configure only KEDA and replicas pile up `Pending`; configure only the node layer and nothing tells it to grow.

## Resources

1. **vLLM production-stack — "Autoscaling with KEDA"** — canonical ScaledObject on `vllm:num_requests_waiting` with the Prometheus trigger and Helm integration: https://docs.vllm.ai/projects/production-stack/en/latest/use_cases/autoscaling-keda.html
2. **Red Hat Developer — "How to set up KServe autoscaling for vLLM with KEDA"** — the KServe + KEDA + custom-metric path end to end, with the node-autoscaler interplay: https://developers.redhat.com/articles/2025/09/23/how-set-kserve-autoscaling-vllm-keda
