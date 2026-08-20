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
sources: 13
---
# 07.8 · Autoscaling inference: scale on queue depth

> **Concept.** Autoscale LLM serving on the queue, not the accelerator. Scale replicas on `vllm:num_requests_waiting` (or KV-cache utilisation) via KEDA — never on CPU or GPU-utilisation — and treat scale-to-zero as a $-vs-SLO decision.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

07.7 gave you a *numerical-format* lever on cost per token: FP8 halves the bytes and buys
roughly 1.5–1.8× throughput on Hopper. This lesson gives you a *capacity* lever — how many
replicas are serving right now, and whether any should exist at all.

That matters because of the arithmetic 07.5 ended on. Your measured CPM at the operating
point was $0.260 per million output tokens; your production CPM at 35 % fleet utilisation
was $0.743. **The entire 2.9× gap is an autoscaling problem**, and it is larger than
anything quantization, batching or paging bought you. Utilisation is the single biggest
remaining term in the cost equation, and this lesson is where you attack it.

It sits between quantization (07.7), whose smaller checkpoints make cold starts cheaper, and
model loading and storage (07.9), which supplies the measured numbers behind the cold-start
cost this lesson reasons about. Read the cold-start figures here as *placeholders with the
right shape*: 07.9 replaces "tens of seconds to minutes" with a decomposed, storage-tier-
dependent budget you measure. What this lesson owns is what that number *does* to a control
loop — because a cold start is not merely a cost, it is **dead time in a feedback system**,
and dead time is what makes control loops oscillate.

**Version pin.** Kubernetes HPA behaviour is read from `kubernetes/kubernetes` on `master`
(`pkg/controller/podautoscaler/replica_calculator.go`, `horizontal.go`,
`pkg/apis/autoscaling/v2/defaults.go`, `pkg/controller/podautoscaler/config/v1alpha1/defaults.go`).
KEDA behaviour is read from `kedacore/keda` at `d2634e6` (2026-08-17), release line **v2.20.x**
(`apis/keda/v1alpha1/scaledobject_types.go`, `pkg/scalers/prometheus_scaler.go`,
`pkg/scalers/scaler.go`, `pkg/scaling/executor/`). vLLM is **v0.27.1**; metric names match
the verified set in [module 05 lesson 06](../../05-gpu-observability/lessons/06-inference-slos.md).

## Why this matters

You already know KEDA, HPA and Prometheus. The trap is applying that muscle memory to LLM
serving with the wrong signal, and there are two distinct ways to get it wrong — one
famous, one not.

The famous one is **scaling on GPU utilisation**. It fails for a mechanical reason covered
in module 05 and re-derived below: `DCGM_FI_DEV_GPU_UTIL` reports whether *any* kernel ran
during the sample window, and in autoregressive decode a kernel is essentially always
running. The metric pins near 100 % at batch 1 and stays there through saturation, through
preemption, and through collapse. It cannot distinguish a healthy replica from a dying one.

The unfamous one is **treating a queue like a utilisation**. Queue depth is the right signal
— it is genuinely leading, it rises the instant arrival rate exceeds service rate — but it
is an **integrating** signal: it has no upper bound and it accumulates the *history* of your
under-provisioning, not its current magnitude. Feed an unbounded integrating signal into a
proportional controller that has a two-minute dead time and you have built a textbook
oscillator. The failure mode is not "it doesn't scale"; it is "it scales to `maxReplicas`,
serves the backlog in twelve seconds, sits idle for five minutes, scales to one, and does it
all again." Every one of those replicas is a GPU-minute you paid for.

Getting this right is worth more than any config flag in this module. An endpoint that is
70 % idle at 3am and cannot scale down is burning roughly $2 per hour per H100 for nothing —
about $17,000 a year per replica. An endpoint that oscillates burns more than one that never
scales at all. And an endpoint that scales *late* burns your TTFT SLO instead of your budget,
which is usually worse.

## What's new here (calibration)

Referenced, not re-taught: the util lie and DCGM field semantics (module 05); TTFT/ITL and
the verified vLLM metric names (module 05, 07.4); the concurrency cap and preemption
(07.2, 07.4); CPM and utilisation (07.5).

Genuinely new:

1. **The exact HPA replica arithmetic for external metrics**, from the Kubernetes source —
   including the fact that for an `AverageValue` target the desired count is
   `ceil(total_metric / threshold)` and does **not** reference the current replica count, plus
   the ±10 % tolerance band and the default scale-up/scale-down rate policies.
2. **The three control loops**, not two: KEDA's activation loop (0↔1), the HPA loop (1↔N),
   and the node autoscaler — with their actual default periods, because the sum of those
   periods is your dead time.
3. **Queue depth as an integrating signal**, and the oscillation it produces — derived
   numerically, not asserted.
4. **The anti-oscillation rule**: the scale-up rate-limit period must be at least the loop
   dead time. This is the lesson's central result and it is computable from your cold start.
5. **Feed-forward on arrival rate plus feedback on queue** as the architecture that actually
   behaves, rather than a single proportional loop.
6. **KEDA's real API surface**, including the fields that no longer exist (`metricName` on the
   Prometheus trigger), the ones with surprising semantics (`cooldownPeriod` applies only to
   the scale-to-zero path), and the ones nobody uses and should (`activationThreshold`,
   `fallback`, `scalingModifiers`).
7. **vLLM sleep mode** as a genuine third option between "warm replica" and "scale to zero",
   with the `vllm:engine_sleep_state` metric that reports it.
8. **A correction:** earlier versions of this lesson used `vllm:gpu_cache_usage_perc`. That
   metric was **renamed** to `vllm:kv_cache_usage_perc` in the V1 engine; a ScaledObject on
   the old name returns no data, and a KEDA Prometheus trigger with `ignoreNullValues: true`
   (the default) treats no data as **zero** — so the autoscaler silently never scales up.

## Core concepts

### 1. What makes a signal scalable, and why the obvious ones fail

Before ranking candidate metrics, state what the autoscaler actually needs. A signal is
usable as a scale trigger if it is:

- **Leading** — it moves before the SLO breaks, not after. Anything derived from completed
  requests is by definition lagging.
- **Monotone in unserved demand** — more load ⇒ higher value, over the whole operating range.
- **Non-saturating within that range** — if it flatlines at its maximum while things get
  worse, it cannot tell the controller *how much* capacity to add.
- **Per-replica normalisable** — the controller divides by a per-replica target, so the
  metric must be summable across replicas and meaningful when divided.
- **Cheap and fresh** — it must survive a scrape interval without becoming stale relative to
  the loop period.

Now the candidates.

**CPU utilisation.** For a GPU decode workload the CPU is running the API server, the
scheduler busy loop and detokenisation. It correlates with token *volume* somewhat, and with
GPU saturation not at all. A replica at batch 8 and a replica at batch 96 look similar. Not
leading, not monotone in the thing you care about. This is the HPA default (`SetDefaults_
HorizontalPodAutoscaler` inserts a CPU `Utilization` target of 80 % when `metrics` is empty),
which means **an HPA created with no metrics block is autoscaling your GPU fleet on CPU** and
looks correctly configured while doing so.

**GPU utilisation (`DCGM_FI_DEV_GPU_UTIL`).** The mechanism, from module 05: this field is
the fraction of the sample window during which one or more kernels were resident on the
device. It is an occupancy-of-time measure, not an occupancy-of-hardware measure. During
autoregressive decode there is a kernel running essentially always, so:

```
  batch 1        → ~95–100 %        (one tiny GEMM, but always running)
  batch 96       → ~99–100 %        (saturated, at the operating point)
  preemption storm → ~100 %          (the GPU is busy REDOING work — 07.4 §7)
```

**It reads the same at 1× and 96× the useful work, and it reads highest exactly when the
service is failing.** A `> 80 %` scale rule therefore fires at idle; a `> 99 %` rule fires
during the storm and adds replicas that do not address the cause. Neither is a control
signal; both are a thermometer that only reads "hot."

**`vllm:num_requests_running`.** Bounded above by `max_num_seqs` and by the KV cap, so it
saturates precisely at the point where you need it to keep rising. Useful as a denominator,
useless as a trigger.

**`vllm:kv_cache_usage_perc`.** Bounded in [0, 1] and leading — it rises before latency does,
because a full pool causes preemption which causes latency. Its saturation is at a *useful*
point (1.0 means the pool is genuinely exhausted), unlike GPU util. But being bounded, it
cannot express "I need three more replicas" versus "I need thirty." Good **guard**, weak
primary.

**`vllm:num_requests_waiting`.** Requests admitted to the engine but not yet scheduled. It
rises the instant arrival rate exceeds service rate, it is unbounded, and it is exactly what
the HPA's per-replica arithmetic wants. **This is the primary signal** — with the caveat that
occupies §6.

**Arrival rate** — `sum(rate(vllm:request_success_total[1m]))`, or your ingress's request
rate. Not a saturation signal at all; it is a *demand* signal, and that is its virtue. It is
the feed-forward term in §9.

| Signal | Leading? | Saturates? | Bounded? | Verdict |
|---|---|---|---|---|
| CPU utilisation | no | yes | yes | **No.** The HPA default, and wrong. |
| `DCGM_FI_DEV_GPU_UTIL` | no (lagging) | **at ~100 % from batch 1** | yes | **No.** Reads highest during failure. |
| `vllm:num_requests_running` | somewhat | at `max_num_seqs` / KV cap | yes | Denominator only. |
| `vllm:kv_cache_usage_perc` | **yes** | at 1.0 (usefully) | yes | Good guard / secondary. |
| `vllm:num_requests_waiting` | **yes** | no | **no** | **Primary** — but integrating (§6). |
| `vllm:request_queue_time_seconds` p99 | yes | no | no | Excellent, SLO-native; noisier. |
| arrival rate | n/a — feed-forward | no | no | **Feed-forward term** (§9). |
| TTFT / TPOT p99 | **lagging** | no | no | The thing you defend, not the trigger. |

This conclusion is convergent rather than idiosyncratic: vLLM's production-stack
documentation, Google's GKE inference-autoscaling guidance, and Red Hat's KServe + KEDA path
independently recommend queue-based signals over accelerator utilisation. Three vendors, three
platforms, one answer, is much stronger evidence than any of them alone.

### 2. Three control loops, and why their periods are your dead time

"KEDA scales pods, the cluster autoscaler scales nodes" is the usual two-loop framing. It is
incomplete in a way that matters: **KEDA runs two different loops with different periods, and
which one is acting depends on whether you are at zero replicas.**

```
  THE THREE LOOPS — WHO DECIDES WHAT, AND HOW OFTEN
  ══════════════════════════════════════════════════════════════════════════════

  ┌─────────────────────────────────────────────────────────────────────────┐
  │ vLLM replica         GET /metrics          vllm:num_requests_waiting     │
  │                          ▲                 vllm:kv_cache_usage_perc      │
  └──────────────────────────┼──────────────────────────────────────────────┘
                             │ scrape  ── ServiceMonitor, interval 15–30 s
                             │            (freshness cost: up to 1 interval)
  ┌──────────────────────────┴──────────────────────────────────────────────┐
  │ PROMETHEUS                                                              │
  └───────┬────────────────────────────────────────┬────────────────────────┘
          │ instant query /api/v1/query            │ instant query
          │                                        │
  ┌───────▼───────────────────────┐   ┌────────────▼──────────────────────────┐
  │ LOOP A · KEDA OPERATOR        │   │ LOOP B · KEDA METRICS ADAPTER + HPA   │
  │ "is this trigger ACTIVE?"     │   │ "how many replicas for 1..N?"         │
  │                               │   │                                       │
  │ period: spec.pollingInterval  │   │ period: --horizontal-pod-autoscaler-  │
  │         DEFAULT 30 s          │   │         sync-period, DEFAULT 15 s     │
  │ test:   value > activation-   │   │ math:   desired =                     │
  │         Threshold (default 0) │   │           ceil(total / threshold)     │
  │ action: 0 → minReplicaCount   │   │         within ±10 % tolerance        │
  │         (or 1), and           │   │ rate:   scaleUp   max(100 %, 4 pods)  │
  │         N → 0 after           │   │                   per 15 s            │
  │         cooldownPeriod        │   │         scaleDown 100 % per 15 s,     │
  │         DEFAULT 300 s         │   │                   stabilised over     │
  │                               │   │                   300 s               │
  │ ONLY acts at the 0 boundary.  │   │ HPA object: keda-hpa-<so-name>        │
  │ HPA is disabled at 0 replicas.│   │ external metric: s0-prometheus        │
  └───────────────┬───────────────┘   └────────────┬──────────────────────────┘
                  │                                │
                  └──────────► Deployment/scale ◄──┘
                                    │
                                    │ new Pod → Pending (no free GPU)
                  ┌─────────────────▼──────────────────────────────────────┐
                  │ LOOP C · CLUSTER AUTOSCALER / KARPENTER                │
                  │ watches unschedulable Pods → provisions a GPU node     │
                  │ latency: 60 s – 5 min (instance boot + kubelet join +  │
                  │          device-plugin registration + image pull)      │
                  │ THIS IS USUALLY THE LARGEST TERM IN THE DEAD TIME.     │
                  └────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────────────┐
  │ CONFIGURE ONLY LOOP A+B  →  replicas pile up Pending forever.          │
  │ CONFIGURE ONLY LOOP C    →  nothing ever asks for more replicas.       │
  │ CONFIGURE BOTH, WRONG METRIC → you provision H100 nodes on a signal    │
  │                                that does not track demand.             │
  └────────────────────────────────────────────────────────────────────────┘
```

The reason to draw all three is that **the dead time of the combined system is the sum of
their periods plus the pod's own startup**, and §6 shows that dead time is the parameter that
decides whether your autoscaler is stable.

### 3. The HPA replica arithmetic, exactly

KEDA does not implement scaling from 1 to N. It creates an HPA named `keda-hpa-<name>` with
an **External** metric, registers itself as the external-metrics API provider, and lets the
Kubernetes HPA controller do the arithmetic. So the arithmetic you need is Kubernetes', and
it is short.

For an external metric with an `AverageValue` target — **which is KEDA's default**
(`GetMetricTargetType` returns `AverageValueMetricType` when no `metricType` is set) — the
controller runs `GetExternalPerPodMetricReplicas` (`pkg/controller/podautoscaler/replica_calculator.go`):

```go
usage = 0
for _, val := range metrics { usage = usage + val }   // SUM across returned series

replicaCount = statusReplicas
usageRatio := float64(usage) / (float64(targetUsagePerPod) * float64(replicaCount))
if !tolerances.isWithin(usageRatio) {
    replicaCount = int32(math.Ceil(float64(usage) / float64(targetUsagePerPod)))
}
```

Four things fall out of those five lines, and three of them surprise people:

1. **`desiredReplicas = ceil(total_metric / threshold)`.** The current replica count appears
   only in the tolerance test, not in the result. Your KEDA `threshold` is therefore *"the
   metric value one replica should carry,"* and the query must return a **sum across all
   replicas**, not a per-pod average. Get that backwards and your scaling is off by the
   replica count.
2. **The tolerance band is ±10 %** — `HorizontalPodAutoscalerTolerance` defaults to `0.1`
   (`pkg/controller/podautoscaler/config/v1alpha1/defaults.go`), and `isWithin` accepts
   `1 − scaleDown ≤ ratio ≤ 1 + scaleUp`. Inside the band, **nothing changes at all.** With
   a threshold of 5 and 2 replicas, a total queue between 9 and 11 produces no action. This
   is deliberate anti-flap and it is your first line of defence. (Kubernetes 1.33+ also allows
   a per-HPA `behavior.scaleUp.tolerance` / `scaleDown.tolerance` override.)
3. **`ceil`, always.** A total of 11 against a threshold of 5 gives 3 replicas, not 2.2. At
   small replica counts the rounding is a large fraction of the answer, which argues for
   thresholds that put you comfortably above 1.
4. **Unready pods are excluded from the denominators elsewhere** (`getReadyPodsCount`), but
   for the `AverageValue` external path the result does not depend on pod readiness at all.
   That is exactly what makes §6's oscillation possible: pods that are booting do not damp
   the recommendation.

**The default rate policies** (`pkg/apis/autoscaling/v2/defaults.go`) apply whenever you do
not specify `behavior`:

```go
defaultHPAScaleUpRules = {
    StabilizationWindowSeconds: 0,                  // no smoothing on the way up
    SelectPolicy:               Max,                // take the MOST permissive policy
    Policies: [ {Pods,    4,   PeriodSeconds: 15},  // +4 pods per 15 s
                {Percent, 100, PeriodSeconds: 15} ] // OR double, whichever is larger
}
defaultHPAScaleDownRules = {
    StabilizationWindowSeconds: nil,   // ⇒ controller default: 300 s (5 minutes)
    SelectPolicy:               Max,
    Policies: [ {Percent, 100, PeriodSeconds: 15} ]
}
```

Read that as a sentence: **by default, Kubernetes will double your replica count every
fifteen seconds with no smoothing, and will refuse to scale down for five minutes.** For a
web service whose pods start in two seconds, that is a reasonable asymmetry. For a GPU
service whose pods take two minutes to serve their first token, the scale-up half is a
loaded gun — §6.

Two more controller defaults worth knowing: the sync period is **15 s**, and the
downscale stabilisation window is **5 minutes** (`RecommendedDefaultHPAControllerConfiguration`).
Scale-down stabilisation takes the **maximum** recommendation over the window, so a single
15-second demand spike keeps your replicas alive for five minutes afterwards.

### 4. The KEDA ScaledObject, field by field

Here is the manifest, with every field that matters and its real default. Note what is
**not** present.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: vllm-chat-8b
  namespace: inference
spec:
  scaleTargetRef:
    name: vllm-chat-8b            # the Deployment. apiVersion/kind default to apps/v1 Deployment.

  # ── loop A: the KEDA operator's own poll (0 ↔ 1 decisions) ──────────────
  pollingInterval: 15             # default 30 s. Lower = faster wake from zero,
                                  # more load on Prometheus.
  cooldownPeriod: 600             # default 300 s. APPLIES ONLY to the scale-to-zero
                                  # (deactivation) path — the 1..N scale-down is
                                  # governed by the HPA's own stabilization window.
  initialCooldownPeriod: 0        # default 0. Grace before the FIRST deactivation
                                  # after the ScaledObject is created.

  minReplicaCount: 1              # 0 enables scale-to-zero. Default 1.
  maxReplicaCount: 8              # default 100. On a GPU fleet, ALWAYS set this —
                                  # it is your blast radius for a runaway loop.
  # idleReplicaCount: 0           # optional: park at N (must be < minReplicaCount)
                                  # instead of hard zero. Rarely useful for GPUs.

  # ── loop B: pass-through configuration of the generated HPA ─────────────
  advanced:
    restoreToOriginalReplicaCount: false
    horizontalPodAutoscalerConfig:
      name: keda-hpa-vllm-chat-8b     # default is exactly this
      behavior:
        scaleUp:
          stabilizationWindowSeconds: 0
          selectPolicy: Max
          policies:
            - type: Pods
              value: 2                 # ← see §7: sized to the COLD START, not to taste
              periodSeconds: 180       # ← must be >= loop dead time
        scaleDown:
          stabilizationWindowSeconds: 600
          selectPolicy: Max
          policies:
            - type: Pods
              value: 1
              periodSeconds: 300

  # ── the triggers ────────────────────────────────────────────────────────
  triggers:
    - type: prometheus
      name: queue                      # referenced by scalingModifiers.formula
      metricType: AverageValue         # KEDA's default anyway; be explicit
      metadata:
        serverAddress: http://prometheus-operated.monitoring.svc:9090
        query: |
          sum(vllm:num_requests_waiting{service="vllm-chat-8b"})
        threshold: "4"                 # queued requests PER REPLICA. Derived in §5.
        activationThreshold: "1"       # 0 → 1 only when the queue actually has work
        ignoreNullValues: "false"      # ← SEE BELOW. Default is "true" and it is a trap.

    - type: prometheus
      name: kv
      metricType: AverageValue
      metadata:
        serverAddress: http://prometheus-operated.monitoring.svc:9090
        query: |
          sum(vllm:kv_cache_usage_perc{service="vllm-chat-8b"})
        threshold: "0.85"              # per-replica KV pressure guard
        activationThreshold: "0.05"
        ignoreNullValues: "false"

  # ── what to do when the metric source is broken ─────────────────────────
  fallback:
    failureThreshold: 3                # consecutive scaler errors before falling back
    replicas: 3                        # hold here rather than collapsing to min
    behavior: static                   # static | currentReplicas |
                                       # currentReplicasIfHigher | currentReplicasIfLower |
                                       # scalingModifiers
```

**Fields that surprise people:**

- **There is no `metricName` field on the Prometheus trigger.** The scaler's metadata struct
  (`pkg/scalers/prometheus_scaler.go`) is exactly `serverAddress`, `query`, `queryParameters`,
  `threshold`, `activationThreshold`, `namespace`, `ignoreNullValues`, `unsafeSsl`, plus auth.
  The external metric name is **generated** as `s<index>-prometheus` —
  `GenerateMetricNameWithIndex(triggerIndex, "prometheus")` — so the first trigger is
  `s0-prometheus`, the second `s1-prometheus`. Older tutorials that set `metricName` are
  describing a field that was removed; with a strict CRD it is rejected, and where it is
  tolerated it does nothing. **Correction to earlier versions of this lesson**, which included
  it.
- **`ignoreNullValues` defaults to `true`.** When the PromQL returns no series, the scaler
  reports **zero** instead of erroring. Combine that with a typo'd metric name — say, the
  renamed `vllm:gpu_cache_usage_perc` — and you get an autoscaler that reports "no load"
  forever, never scales up, and never logs an error. Setting it to `"false"` makes a missing
  metric a scaler *error*, which trips `fallback` and surfaces on the ScaledObject's
  conditions. **On a GPU fleet, set it to `"false"`.**
- **`cooldownPeriod` is not a general scale-down delay.** It gates only the transition to
  `idleReplicaCount`/`minReplicaCount` when every trigger has been inactive
  (`pkg/scaling/executor/scale_scaledobjects.go`: the deactivation path compares
  `Status.LastActiveTime + cooldownPeriod` against now). The 1→N→1 range is the HPA's
  business, governed by `scaleDown.stabilizationWindowSeconds`. Setting `cooldownPeriod: 600`
  and wondering why scale-down still happens after five minutes is this confusion.
- **`activationThreshold` is a different question from `threshold`.** `threshold` answers
  "how many replicas"; `activationThreshold` answers "should there be any at all," and the
  scaler's `GetMetricsAndActivity` returns `val > s.metadata.ActivationThreshold` as the
  active flag. With the default of 0, *any* non-zero value wakes the deployment — including
  a stale sample. Set it deliberately.
- **The Prometheus scaler issues an instant query** (`/api/v1/query?query=…&time=…`) and
  **errors if the result contains more than one element.** `sum()` your query. A `by (pod)`
  that you forgot to aggregate is a scaler error, not a helpful multi-series result.
- **`scalingModifiers`** lets you combine named triggers with an expr-lang formula into a
  single composite external metric (`composite-metric`), e.g.
  `formula: "queue > 0 ? queue : kv * 10"`, with its own `target` and `metricType`. Use it
  when you want one coherent decision rather than KEDA's default of taking the maximum
  recommendation across triggers.

**How the triggers combine by default:** each trigger produces its own external metric on the
generated HPA, and the HPA takes the **largest** desired replica count across all metrics.
So the `kv` trigger above is a genuine guard: it can only raise the answer, never lower it.

### 5. Deriving the threshold from your TTFT SLO

Do not pick 5 because it sounds reasonable. The threshold is *"the per-replica queue depth at
which we are still comfortably inside the SLO, with enough lead time for a replica to
arrive."* It is derivable from numbers you already measured in 07.5.

Start from Little's Law and the TTFT decomposition (module 05): a request's TTFT is its queue
wait plus its prefill. With `Q` requests queued ahead and a per-replica prefill service rate,

```
  TTFT  ≈  Q × t_prefill_per_request  +  t_prefill_own

  From 07.5 §5, at the B = 96 operating point with max_num_batched_tokens = 4096:
      prefill token rate  = (4096 − 96) / 21.1 ms          = 189,573 tok/s
      4096-token prompt   = 4096 / 189,573                 = 21.6 ms per request

  SLO: TTFT p99 ≤ 500 ms
      Q_max = 500 / 21.6 − 1                               = 22.1  ⇒  22 queued

  p99 correction. Queue depth is bursty; for roughly Poisson arrivals a p99
  queue ≈ 1.5–2× the mean. Take 1.75:
      Q_mean_max = 22 / 1.75                               = 12.6  ⇒  12
```

So a *steady-state* queue of 12 per replica is the edge of the SLO. Now subtract lead time,
because you must scale out **before** the queue reaches its limit, and the new replica takes
`T_cold` to arrive:

```
  During T_cold the queue grows at the deficit rate:
      dQ/dt  =  λ − μ·R          (arrivals minus service, per replica-set)

  Worst deficit you are willing to absorb: one replica's worth, μ = 6.0 req/s
  (07.5's measured request throughput per replica).
      T_cold = 150 s (warm node, cached image; MEASURE YOURS — 07.9)
      queue growth during T_cold  =  6.0 × 150  =  900 requests

  …which is far more than 12 per replica. The queue CANNOT be the whole
  answer at this cold-start latency; §9's feed-forward term exists for
  exactly this reason. What the threshold can do is trigger EARLY:

      threshold  =  Q_mean_max × safety_factor,  safety ≈ 0.3–0.5
                 =  12 × 0.35
                 ≈  4 queued requests per replica
```

**That `threshold: "4"` in §4 is now a derived number**, and the derivation exposes something
more useful than the number: at a 150-second cold start, a queue-depth trigger alone cannot
protect a 500 ms TTFT SLO under a step change in load, because the queue that accumulates in
150 seconds is two orders of magnitude past the SLO limit. Queue depth tells you *that* you
are under-provisioned; it cannot get you provisioned in time. The honest conclusions are:
keep a warm floor sized to your p95 traffic, use feed-forward for the ramp, and reserve the
queue trigger for correcting the feed-forward's error.

**Re-derive the threshold whenever the operating point changes.** It depends on
`max_num_batched_tokens`, on B, and on prompt length — all of which 07.4 and 07.5 tune.

### 6. The oscillation: why an integrating signal plus dead time misbehaves

This is the section that separates a working autoscaler from a demo.

**Queue depth is an integral.** Utilisation is a *rate* measure, bounded in [0, 1]. A queue
is the time-integral of the deficit between arrival and service rates:

```
  Q(t)  =  Q(0)  +  ∫ (λ(τ) − μ·R(τ)) dτ
```

Two properties follow that a proportional controller is not built for. It is **unbounded** —
a persistent deficit of 18 req/s produces a queue of 2,700 after 150 seconds, and the
controller reads that as a demand for `ceil(2700/4) = 675` replicas. And it **does not fall
when you fix the problem**; it falls only after you have over-provisioned enough to drain
the accumulated backlog, which is a second, opposite error.

Combine that with dead time and you get the classic picture:

```
  THE OSCILLATION — METRIC, LAG, AND COLD START ON ONE TIMELINE
  ══════════════════════════════════════════════════════════════════════════════
  Setup: 2 replicas, each serving μ = 6.0 req/s. Arrival steps 12 → 30 req/s at t=0.
  Correct answer: ceil(30/6) = 5 replicas.
  Config: threshold 4, DEFAULT HPA behavior (scaleUp: +100 % or +4 pods / 15 s,
  no stabilization; scaleDown: 100 % / 15 s stabilised over 300 s), maxReplicas 32.
  Dead time T_d = 150 s: scrape 15 + poll 15 + schedule 5 + load & warm 110 + probe 5.

   λ  30 ┤ ▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁
  req/s12 ┤▁│
          └─┼───────┬───────┬───────┬───────┬───────┬───────┬───────┬──────
            0      60     120     180     240     300     360     420   s

   Q      ┤                    ╱▔▔▔╲                          ╱▔▔╲
  queue   ┤              ╱▔▔▔▔▔     ╲                   ╱▔▔▔▔▔    ╲
  depth   ┤        ╱▔▔▔▔▔            ╲            ╱▔▔▔▔▔           ╲
  (sum)   ┤▁▁▁╱▔▔▔▔                   ╲▁▁▁▁▁▁▁▁▁▁▔                  ╲▁▁▁
          └──┼──────────────────────────┼─────────────────────┼──────────
            t=0   deficit = 30−12 = 18 req/s ⇒ Q grows 18/s

   R      ┤                          ┌────────────┐            ┌──────┐
  replicas┤                          │  32 (MAX)  │            │  32  │
        5 ┤─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │─ ─ ─ ─ ─ ─ │─ ─ ─ ─ ─ ─ │─ ─ ─ ─
        2 ┤▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔┘            └────────────┘
          └──┴────┴────┴──────────────┴────────────┴────────────┴───────
             ▲    ▲    ▲              ▲            ▲            ▲
             │    │    │              │            │            │
   t=15s ────┘    │    │              │            │            │
     Q=270. desired = ceil(270/4) = 68 → capped at maxReplicas 32.
     HPA patches replicas 2 → 4 (policy: +100 % OR +4, take Max ⇒ +4... = 6? no:
     Max(2×2, 2+4) = 6). Pods are Pending/booting. THEY SERVE NOTHING YET.
                  │    │              │            │            │
   t=30s ─────────┘    │              │            │            │
     Q=540, still 2 SERVING replicas. desired still 32. HPA: 6 → 12.
     ← THE CONTROLLER IS ACTING AGAIN ON DEMAND IT HAS ALREADY ACTED ON.
                       │              │            │            │
   t=45..150s ─────────┘              │            │            │
     Same again, every 15 s: 12 → 24 → 32 (max). Ten decisions taken during
     one dead time, each on the same unserved backlog.
                                      │            │            │
   t=155s ────────────────────────────┘            │            │
     The first cohort finally becomes Ready — and so does every later cohort,
     within seconds of each other. 32 replicas now serve 192 req/s against
     30 req/s of arrivals. The 2,700-request backlog drains in ~16 s.
     Q → 0.  TTFT recovers.  COST: 32 H100s for a 5-H100 workload.
                                                   │            │
   t=170s ─────────────────────────────────────────┘            │
     desired = ceil(0/4) = 0 → clamped to minReplicaCount.
     Scale-DOWN stabilization takes the MAX recommendation over 300 s,
     so nothing happens yet. You pay for 32 idle H100s for five minutes:
        32 × $2.89/hr × 300 s / 3600 = $7.71 of pure waste per event.
                                                                │
   t=470s ────────────────────────────────────────────────────► ┘
     Stabilization expires. Scale down 100 %/15 s → back to 2 (or min).
     Arrivals are still 30 req/s. Deficit returns. Q grows. GOTO t=0.

  ┌────────────────────────────────────────────────────────────────────────┐
  │ THE PERIOD OF THE OSCILLATION IS SET BY THE SCALE-DOWN STABILIZATION   │
  │ WINDOW (300 s), NOT BY THE TRAFFIC. Nothing about the workload is      │
  │ cyclic; the controller manufactured the cycle.                         │
  │                                                                        │
  │ ROOT CAUSE, IN ONE LINE: the loop took ~10 independent corrective      │
  │ actions during ONE dead time, because nothing told it that its         │
  │ previous action had not landed yet.                                    │
  │                                                                        │
  │ THE FIX IS NOT A BETTER METRIC. It is a RATE LIMIT (§7).               │
  └────────────────────────────────────────────────────────────────────────┘
```

Note the two mechanisms that make this worse than it would be for a stateless web service:

- **Unready pods do not damp the recommendation.** For an `AverageValue` external metric the
  desired count is `ceil(total/threshold)` with no readiness term, so a pod that has been
  booting for 90 seconds contributes nothing to the controller's belief that help is coming.
- **The dead time is 30–100× a web service's.** A stateless HTTP pod is Ready in ~2 s, so
  even the default `+100 %/15 s` policy takes at most one extra action before feedback
  arrives. At 150 s it takes ten. **The default HPA behaviour is correct for the workload it
  was designed for and dangerously wrong for this one.**

### 7. The anti-oscillation rule

The result, stated plainly:

> **The scale-up policy's `periodSeconds` must be at least the loop dead time, and the
> policy's `value` must be the number of replicas you are willing to commit before any
> feedback arrives.**

That converts an unbounded proportional response into a bounded, rate-limited one whose
worst-case overshoot you can compute:

```
  overshoot_max  =  value  ×  ⌈ T_settle / periodSeconds ⌉

  With value = 2, periodSeconds = 180, and a genuine need for 3 more replicas:
      t=0    : +2 (2 → 4)
      t=180  : first cohort Ready; the metric now reflects them.
               remaining deficit ⇒ +1 (4 → 5). Correct answer reached.
      total overshoot: 0.
```

First, **measure your dead time.** It is not a guess; it is a sum you can instrument:

```
  T_d  =  scrape_interval                       15 s   (ServiceMonitor)
       +  metric staleness (≤ 1 scrape)         15 s
       +  KEDA pollingInterval OR HPA sync      15 s
       +  pod scheduling                         5 s   (warm node)
       +  node provisioning                      0 s   (warm) │ 60–300 s (cold)
       +  image pull                             0 s   (cached) │ 60–300 s
       +  weight load + engine warmup          110 s   ← 07.9 decomposes this
       +  readiness probe period + threshold    10 s
       ────────────────────────────────────────────────
       WARM NODE, CACHED IMAGE                 ≈170 s
       COLD NODE, UNCACHED IMAGE               ≈500–800 s
```

Then set the policies from it. Both numbers below are derived, not chosen:

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0     # react immediately to the FIRST signal…
    selectPolicy: Max
    policies:
      - type: Pods
        value: 2                      # …but commit at most 2 replicas…
        periodSeconds: 180            # …per dead time. 180 >= T_d = 170.
  scaleDown:
    stabilizationWindowSeconds: 900   # >= 3 x T_d. Scaling in is cheap to get
                                      # wrong slowly and expensive to get wrong
                                      # fast: a wrongly-removed replica costs a
                                      # full cold start to replace.
    selectPolicy: Max
    policies:
      - type: Pods
        value: 1
        periodSeconds: 300            # remove one replica per 5 minutes, max.
```

Three notes on this shape:

- **`stabilizationWindowSeconds: 0` on scale-up is right**, and it is not in tension with the
  rate limit. Scale-up stabilisation takes the *minimum* recommendation over the window,
  which delays your reaction to a genuine ramp; the rate-limit policy bounds the *magnitude*
  instead. React fast, commit slowly.
- **Asymmetric on purpose.** Scaling up too eagerly costs money; scaling down too eagerly
  costs a cold start, which costs both money *and* SLO. The asymmetry should therefore be
  larger than the Kubernetes default, not smaller.
- **`maxReplicaCount` is your blast radius.** On a GPU fleet it is not a formality. KEDA's
  default is 100; at $2.89/hr that is a $289/hour mistake waiting for a metric glitch.

### 8. Scale-to-zero, and the third option most teams miss

**KEDA's zero handling.** `minReplicaCount: 0` hands the 0↔1 transition to the KEDA operator
(loop A), because the HPA controller disables itself at zero replicas — you will see this in
the ScaledObject conditions as an HPA `ScalingDisabled` reason, which KEDA explicitly
translates to a healthy state. Scale-from-zero happens when **any** trigger reports active
(`value > activationThreshold`); scale-to-zero happens when **all** triggers have been
inactive for `cooldownPeriod`.

The consequence people trip on: **at zero replicas there is no vLLM pod, so there is no
`/metrics`, so `vllm:num_requests_waiting` does not exist.** A ScaledObject whose only
trigger queries a vLLM metric can never wake up — the query returns no series, and with the
default `ignoreNullValues: true` that reads as zero. **You must trigger on a signal that
exists while the deployment is at zero**: your ingress/gateway's request counter, an HTTP
add-on that queues the request itself, or a queue-length metric from a broker in front. The
vLLM metric can be a second trigger for the 1..N range, but it cannot be the wake-up.

**The economics.** Price the decision rather than debating it:

```
  SAVING   =  idle_hours_per_day × replicas × $/GPU-hr × 365
  COST     =  wake_events_per_day × 365 × (cold_start_s × $/GPU-s
                                            + SLO_cost_per_slow_request × affected_requests)

  Worked, one H100 replica at $2.89/hr, idle 16 h/day, 8 wake events/day,
  T_cold = 170 s, and roughly 40 requests caught behind each wake:

    SAVING = 16 × 1 × 2.89 × 365                              = $16,878 / yr
    GPU cost of the wakes
           = 8 × 365 × 170 s × ($2.89/3600)                   =    $399 / yr
    ⇒ dollar-for-dollar this is overwhelmingly worth it.

    THE REAL COST IS THE SLO, NOT THE DOLLARS:
      8 × 365 × 40 = 116,800 requests/yr see a TTFT of ~170 s instead of ~0.3 s.
      If that endpoint has a 500 ms TTFT SLO with, say, a 99.5 % objective and
      2 M requests/yr, those 116,800 requests are 5.8 % of traffic —
      ELEVEN TIMES the entire error budget. Scale-to-zero is off the table
      for this endpoint until T_cold falls below ~2 s, which it will not.
```

**So the decision rule is:** scale to zero when the endpoint's users are machines or the
endpoint's SLO is measured in minutes (batch, async, nightly internal tools, dev/staging,
evaluation harnesses). Keep `minReplicaCount ≥ 1` when a human is waiting. And note this is
**per endpoint, not per platform** — the same cluster legitimately runs a customer-facing
deployment at `minReplicaCount: 2` and an internal-tools deployment at `0`.

**The third option: vLLM sleep mode.** Between "warm replica burning $2.89/hr" and "cold
start from zero" there is a genuine middle, and it is under-used. vLLM can release most of a
model's GPU memory without stopping the server:

```bash
# The server must be started with dev endpoints enabled and sleep mode on.
VLLM_SERVER_DEV_MODE=1 vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --enable-sleep-mode --port 8000

curl -X POST 'http://localhost:8000/sleep?level=1'   # offload weights to CPU RAM,
                                                     # discard the KV cache
curl -X GET  'http://localhost:8000/is_sleeping'     # {"is_sleeping": true}
curl -X POST 'http://localhost:8000/wake_up'         # restore
```

The two levels, from vLLM's documentation and `vllm/v1/worker/gpu_worker.py`:

| Level | Weights | KV cache | Restore path | Use when |
|---|---|---|---|---|
| **1** | offloaded to **CPU RAM** | discarded | copy back over PCIe — **seconds**, not a reload | same model will run again |
| **2** | **discarded** (buffers kept on CPU) | discarded | requires `collective_rpc("reload_weights")` | different model next, or no host RAM to spare |

Level 1 is the interesting one for cost: the process, the CUDA context, the compiled graphs
and the tokeniser all stay alive, so waking is a host-to-device copy of the weights rather
than a storage read plus compile plus capture. For a 16 GB 8B model over PCIe Gen5 at a
realistic ~40 GB/s that is on the order of half a second of copy, versus 07.9's full
cold-start budget. **The catch is that the GPU is still allocated** — you have freed HBM for a
co-tenant, not freed the card, so this saves money only if something else uses it (a training
job, a second model, an RLHF trainer). vLLM exposes the state as
`vllm:engine_sleep_state{sleep_state="awake"|"weights_offloaded"|"discard_all"}`, so you can
alert on a fleet that is asleep when it should not be.

Wake-up is also selective — `POST /wake_up?tags=weights` then `?tags=kv_cache` — which exists
for RLHF weight-update flows but is equally useful for staging a wake without a KV allocation
spike.

### 9. The architecture that actually behaves: feed-forward plus feedback

§5 established the uncomfortable result: at a 150-second dead time, a queue-depth controller
cannot protect a sub-second TTFT SLO against a step change in load, because the queue that
accumulates during the dead time is two orders of magnitude past the SLO's limit. No choice
of threshold fixes that; it is a property of the delay.

The control-theoretic answer is standard: **when your plant has large dead time, do not rely
on error feedback alone. Add a feed-forward term driven by the disturbance you can observe
directly.** Here the disturbance is arrival rate, and you can observe it at the ingress
without waiting for it to become a queue.

```
                   ┌──────────────────────────────────────────────┐
   arrival rate ───┤ FEED-FORWARD                                 │
   (ingress /      │   R_ff = ceil(λ / μ_replica × headroom)      │──┐
    gateway)       │   λ from ingress; μ_replica MEASURED in 07.5 │  │
                   │   no dead time — it moves WITH demand        │  │
                   └──────────────────────────────────────────────┘  │  take
                   ┌──────────────────────────────────────────────┐  ├─ the
   queue depth ────┤ FEEDBACK (correction)                        │  │  MAX
   KV pressure     │   R_fb = ceil(Q_total / threshold)           │──┘
                   │   catches what feed-forward got wrong:       │
                   │   long prompts, a slow replica, a bad node   │
                   └──────────────────────────────────────────────┘
```

In KEDA this is two triggers plus (optionally) a `scalingModifiers` formula, and because the
HPA takes the maximum recommendation across metrics, the plain two-trigger form already gives
you the `max()`:

```yaml
  triggers:
    # FEED-FORWARD: requests per second arriving, divided by per-replica capacity.
    # threshold is in the metric's own units: req/s that ONE replica should carry.
    - type: prometheus
      name: demand
      metricType: AverageValue
      metadata:
        serverAddress: http://prometheus-operated.monitoring.svc:9090
        query: |
          sum(rate(nginx_ingress_controller_requests{service="vllm-chat-8b"}[1m]))
        threshold: "4.2"        # = measured 6.0 req/s per replica x 0.7 headroom
        activationThreshold: "0.1"
        ignoreNullValues: "false"

    # FEEDBACK: queue depth, as the correction term.
    - type: prometheus
      name: queue
      metricType: AverageValue
      metadata:
        serverAddress: http://prometheus-operated.monitoring.svc:9090
        query: sum(vllm:num_requests_waiting{service="vllm-chat-8b"})
        threshold: "4"
        activationThreshold: "1"
        ignoreNullValues: "false"

    # GUARD: KV pressure, so a long-context shift raises capacity even when
    # request RATE is unchanged. This is the one that catches the failure mode
    # neither of the other two sees.
    - type: prometheus
      name: kv
      metricType: AverageValue
      metadata:
        serverAddress: http://prometheus-operated.monitoring.svc:9090
        query: sum(vllm:kv_cache_usage_perc{service="vllm-chat-8b"})
        threshold: "0.85"
        activationThreshold: "0.05"
        ignoreNullValues: "false"
```

The `kv` trigger earns its place by catching a failure the other two are blind to: **a shift
in prompt-length distribution changes your capacity without changing your request rate.** If
your traffic moves from 2k to 16k prompts, `λ` is unchanged, the queue is briefly unchanged,
and your per-replica capacity has dropped 4×. `kv_cache_usage_perc` moves immediately.

`μ_replica` in the feed-forward term is exactly the `Request throughput (req/s)` you measured
at your operating point in 07.5 — 6.02 req/s in that lesson's run. **Re-derive it whenever the
config or workload changes**, and put a comment in the manifest saying where it came from,
because a stale `μ` is a silently wrong autoscaler.

### 10. The node layer, and why it usually dominates

Loop C is where the seconds go. When KEDA adds a replica and no GPU node has a free
accelerator, the pod goes `Pending` and something must provision hardware:

```
  pod Pending (Insufficient nvidia.com/gpu)
     → Karpenter / Cluster Autoscaler notices           ~10–30 s
     → cloud API: launch instance                       ~30–120 s
     → instance boots, kubelet joins                    ~30–90 s
     → NVIDIA device plugin registers nvidia.com/gpu    ~10–60 s
     → pod scheduled, image pulled (multi-GB)           ~60–300 s uncached
     → vLLM startup (07.4 §1, 07.9)                     ~60–600 s
     ─────────────────────────────────────────────────────────────
     TOTAL, cold node, uncached image                   ~200 s – 20 min
```

Mitigations, in decreasing order of effect and increasing order of cost:

| Mitigation | Removes | Cost |
|---|---|---|
| **Warm node pool** (over-provision N nodes with pause pods at low priority) | node provisioning entirely | N idle GPUs |
| **Pre-pulled images** (DaemonSet warmer, or a registry pull-through cache in-region) | image pull | node disk |
| **Node-local weight cache** (hostPath NVMe, pre-staged) | most of weight load — 07.9 | node disk + a sync job |
| **Persisted compile cache** (`VLLM_CACHE_ROOT` on a PVC or baked into the image) | 30–120 s of `torch.compile` | image size |
| **Warm replica floor** (`minReplicaCount ≥ 1`) | the whole cold start, for the first N requests' worth of capacity | one GPU, continuously |

**The most common single misconfiguration in this whole area is configuring one loop and not
the other.** KEDA without a node autoscaler produces replicas that sit `Pending` forever, and
the ScaledObject reports success the entire time — it did its job; it changed
`spec.replicas`. A node autoscaler without KEDA never hears that more capacity is wanted.
Check both by watching for `Pending` GPU pods:

```bash
kubectl get pods -n inference -o wide | grep Pending
kubectl describe pod <pending-pod> | grep -A5 Events    # "Insufficient nvidia.com/gpu"?
kubectl get scaledobject,hpa -n inference
```

## Perspectives

**The control-theory view.** Strip the Kubernetes vocabulary and this is a first-order plant
with large transport delay, driven by a proportional controller on an integrating error
signal. Every pathology in §6 is predicted by that description: proportional gain on an
integrator gives you overshoot, delay converts overshoot into oscillation, and the oscillation
period is set by whichever damping element is longest — here, the 300-second scale-down
stabilisation window. The standard remedies are also the ones §7 and §9 arrive at: rate-limit
the actuator to the plant's settling time, and add feed-forward from the measurable
disturbance. If you have tuned a PID loop, you already have this intuition; the work is
recognising that `behavior.scaleUp.policies` *is* an actuator rate limit.

**The SRE view.** The most valuable thing in this lesson is not the YAML, it is the
instruction to *measure the dead time* and treat it as a configuration input rather than an
unknown. Almost every autoscaling incident on a GPU fleet reduces to a number nobody measured:
how long, exactly, from "the metric crossed" to "a replica served a token." Put it on a
dashboard. Alert when it regresses — a change in base image, a storage-tier change, or a new
`torch.compile` cache miss can double it silently, and the first symptom will be an
oscillation that nobody changed the autoscaler to cause.

**The FinOps view.** Autoscaling is the largest remaining term in 07.5's cost equation: the
gap between $0.260/M at full utilisation and $0.743/M at 35 % is bigger than everything
quantization and batching bought combined. But note the direction of the risk. A *badly*
configured autoscaler is more expensive than none at all — §6's oscillation runs 32 replicas
for a 5-replica workload and then pays a five-minute stabilisation window for the privilege,
repeatedly, forever. "We turned on autoscaling" is not a cost win; "we measured our dead time,
rate-limited scale-up to it, and cut idle GPU-hours 40 % with no SLO regression" is.

**The vendor-neutrality view.** KEDA is a CNCF **graduated** project, governed independently of
any cloud vendor, and the same ScaledObject applies on GKE, EKS, AKS or bare metal. In a
design review that argument is materially stronger than "our cloud rep recommended it,"
because it survives a cloud migration. It also means the mechanisms in this lesson —
`ceil(total/threshold)`, the ±10 % tolerance, the rate policies — are Kubernetes semantics you
will meet again under other names.

**The skeptic's view.** Everything here assumes replicas are interchangeable. They may not be:
07.4 showed that profiling-based KV sizing gives different concurrency caps on nodes with
different co-tenancy, so `μ_replica` is not uniform, and a load balancer doing round-robin
will overload the smaller ones while your averaged metric looks fine. If your fleet is
heterogeneous, either pin the KV pool with `--kv-cache-memory-bytes` so replicas really are
interchangeable, or scale per-node-pool with separate ScaledObjects. An autoscaler is only as
good as the assumption that one more replica adds a known amount of capacity.

## Real-world use cases

- **Google Cloud — "Tuning the GKE HPA to run inference on GPUs."** Google's own inference
  benchmarking on GKE concluded that "the default metrics for autoscaling are CPU or memory
  utilization… for inference servers these metrics are no longer a good sole indicator," and
  recommended queue size as the signal for throughput and tail-latency control. **What it
  shows:** the util-lie thesis is not a KEDA-ecosystem opinion — a competing cloud vendor,
  benchmarking a different stack, reached the same conclusion independently. When three
  platform vendors converge on "do not scale on accelerator utilisation," that is much
  stronger evidence than any single recommendation. *(cloud.google.com is blocked by this
  environment's egress proxy; the finding is reported here as a documented industry position
  rather than a page re-read for this rewrite, and §1 derives the mechanism from DCGM field
  semantics independently so the lesson does not depend on the citation.)*

- **Kubernetes HPA default scale-up rules (`pkg/apis/autoscaling/v2/defaults.go`).** The
  shipped default is `+100 % or +4 pods per 15 s, select Max, no stabilisation` on the way up,
  and `100 % per 15 s stabilised over 300 s` on the way down. **What it shows:** these
  defaults encode an assumption about pod startup time that GPU inference violates by a
  factor of 30–100. The defaults are *correct for the workload Kubernetes was designed
  around* — a stateless pod Ready in ~2 s takes at most one extra action before feedback
  arrives. At 150 s it takes ten, which is §6's entire failure. Reading defaults as
  assumptions-about-your-workload rather than as neutral choices is a transferable habit.

- **KEDA's `ignoreNullValues` default and the vLLM metric rename.** The Prometheus scaler
  defaults `ignoreNullValues` to `true`, meaning "no series returned" is reported as the value
  **zero** rather than as an error. In V1, vLLM renamed `vllm:gpu_cache_usage_perc` to
  `vllm:kv_cache_usage_perc` and removed `vllm:num_requests_swapped` and
  `vllm:cpu_cache_usage_perc` outright. **What it shows:** these two facts compose into a
  silent failure — a ScaledObject carried over from a 2024 runbook queries a metric that no
  longer exists, receives zero, concludes there is no load, and never scales. There is no
  error, no event, and no alert. This is the concrete reason §4 sets `ignoreNullValues:
  "false"` and the reason to unit-test a ScaledObject's PromQL against a live Prometheus
  before shipping it.

- **vLLM sleep mode (v0.27.1, `--enable-sleep-mode` + `VLLM_SERVER_DEV_MODE=1`).** Level 1
  offloads weights to CPU RAM and discards the KV cache while keeping the process, CUDA
  context and compiled graphs alive; level 2 discards weights too and requires an explicit
  `reload_weights`. The engine reports its state as
  `vllm:engine_sleep_state{sleep_state="awake"|"weights_offloaded"|"discard_all"}`. **What it
  shows:** the binary "warm replica or scale to zero" framing is a false dichotomy. The
  intermediate state — HBM released, process alive — is a real, supported mode that turns a
  multi-minute cold start into a PCIe copy. Its limitation is equally instructive: the GPU
  remains allocated, so this converts idle *memory* into a shareable resource, not idle
  *cards* into savings. It pays when something else can use the HBM.

## Worked example

**Build a queue-triggered autoscaler, measure its dead time, watch it oscillate on the
defaults, then fix it with a derived rate limit.** Runs on `kind` for the control-plane
mechanics; the dead-time measurement needs a real GPU cluster to be meaningful, and the gap
between the two is the point of Step 5.

### Step 1 — install and wire the metric

```bash
kind create cluster --name infer
helm repo add kedacore https://kedacore.github.io/charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install keda kedacore/keda -n keda --create-namespace
helm install prom prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl create ns inference
kubectl apply -f vllm-deploy.yaml          # the 07.4 Deployment + Service on :8000
```

The `ServiceMonitor` — get this right before touching KEDA:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: vllm-chat-8b
  namespace: inference
  labels:
    release: prom                # must match the kube-prometheus-stack selector
spec:
  selector:
    matchLabels: { app: vllm-chat-8b }
  endpoints:
    - port: http                 # the NAMED port on the Service
      path: /metrics
      interval: 15s              # ← a term in your dead time. Write it down.
```

**Verify before wiring KEDA.** A ScaledObject on a metric that does not exist fails silently:

```bash
kubectl -n monitoring port-forward svc/prom-kube-prometheus-stack-prometheus 9090:9090 &
curl -s 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=sum(vllm:num_requests_waiting{service="vllm-chat-8b"})' | jq .

# EXPECT: {"status":"success","data":{"resultType":"vector",
#          "result":[{"metric":{},"value":[1755500000,"0"]}]}}
# A result of []  ⇒ the metric does not exist. STOP AND FIX IT.
```

Also confirm the exact names your build exposes — they move between releases:

```bash
kubectl -n inference exec deploy/vllm-chat-8b -- \
  sh -c "curl -s localhost:8000/metrics | grep -E '^# HELP vllm:' | sort"
```

### Step 2 — the naive ScaledObject, on purpose

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: vllm-chat-8b, namespace: inference }
spec:
  scaleTargetRef: { name: vllm-chat-8b }
  minReplicaCount: 2
  maxReplicaCount: 32
  pollingInterval: 15
  triggers:
    - type: prometheus
      name: queue
      metadata:
        serverAddress: http://prom-kube-prometheus-stack-prometheus.monitoring.svc:9090
        query: sum(vllm:num_requests_waiting{service="vllm-chat-8b"})
        threshold: "4"
        ignoreNullValues: "false"
```

No `behavior`. That means the Kubernetes defaults from §3 apply: `+100 % or +4 pods per 15 s,
no stabilisation` up; `100 %/15 s stabilised over 300 s` down.

Confirm what KEDA generated:

```bash
kubectl -n inference get hpa
# NAME                    REFERENCE                  TARGETS         MINPODS  MAXPODS  REPLICAS
# keda-hpa-vllm-chat-8b   Deployment/vllm-chat-8b    0/4 (avg)       2        32       2

kubectl -n inference get hpa keda-hpa-vllm-chat-8b -o jsonpath='{.spec.metrics}' | jq .
# [{"type":"External","external":{"metric":{"name":"s0-prometheus", ...},
#   "target":{"type":"AverageValue","averageValue":"4"}}}]
```

Note both generated names: the HPA is `keda-hpa-<so-name>` and the external metric is
`s0-prometheus` — index-prefixed, **not** anything you chose.

### Step 3 — measure the dead time

This is the number the rest of the exercise depends on, and it is a stopwatch exercise.

```bash
# Terminal 1 — timestamped event stream.
kubectl -n inference get events -w --output-watch-events \
  -o custom-columns='TIME:.metadata.creationTimestamp,TYPE:.type,OBJ:.involvedObject.name,MSG:.message' &
kubectl -n inference get pods -w -o wide &

# Terminal 2 — a step change in load.
date -u +%FT%TZ                              # ← t0
vllm bench serve --model meta-llama/Llama-3.1-8B-Instruct \
  --base-url http://<svc>:8000 \
  --dataset-name random --random-input-len 4096 --random-output-len 512 \
  --request-rate 30 --num-prompts 6000 \
  --percentile-metrics ttft --metric-percentiles 99
```

Record five timestamps:

| Mark | Event | How to observe |
|---|---|---|
| `t0` | load step begins | your own clock |
| `t1` | metric crosses threshold in Prometheus | query `s0-prometheus` / the PromQL directly |
| `t2` | HPA changes `desiredReplicas` | `kubectl get hpa -w`, or the `SuccessfulRescale` event |
| `t3` | new pod is `Scheduled` (or `Pending` with `Insufficient nvidia.com/gpu`) | pod events |
| `t4` | new pod is `Ready` and serving | readiness transition + a first 200 from that pod |

```
  T_observe = t1 − t0      scrape interval + staleness
  T_decide  = t2 − t1      KEDA poll / HPA sync
  T_place   = t3 − t2      scheduling (+ node provisioning if Pending)
  T_start   = t4 − t3      image pull + weight load + warmup   ← 07.9 owns this
  ─────────────────────────────────────────────────────────────
  T_d       = t4 − t0      THE DEAD TIME. Everything else is derived from it.
```

Representative on `kind` with a tiny CPU model: `T_d ≈ 12 s`. Representative on a real GPU
cluster with a warm node and cached image: `T_d ≈ 170 s`. Cold node, uncached image:
`T_d ≈ 500–800 s`. **The 15–60× gap between the first and the others is why a threshold
validated on `kind` is not validated.**

### Step 4 — watch the default configuration oscillate

Keep the load at 30 req/s for fifteen minutes and record replicas over time:

```bash
while true; do
  printf '%s ' "$(date -u +%T)"
  kubectl -n inference get deploy vllm-chat-8b \
    -o jsonpath='{.spec.replicas} {.status.readyReplicas}'
  curl -s "http://localhost:9090/api/v1/query?query=sum(vllm:num_requests_waiting)" \
    | jq -r '.data.result[0].value[1] // "0"'
  sleep 10
done | tee oscillation.log
```

Representative trace on a GPU cluster with `T_d ≈ 170 s`:

```
  time    spec  ready  queue
  00:00     2     2       0
  00:15     6     2     271     ← +4 (Max(2x2, 2+4) = 6). Nothing is ready yet.
  00:30    12     2     541     ← doubled again. On the SAME unserved demand.
  00:45    24     2     811
  01:00    32     2    1082     ← maxReplicaCount. Blast radius reached in 60 s.
  ...
  02:55    32    32     412     ← the whole cohort becomes Ready within ~20 s
  03:10    32    32       0     ← backlog drained. 32 replicas, 30 req/s of work.
  03:15     2    32       0     ← desired collapses immediately…
  08:15     2     2       0     ← …but scale-down stabilisation held 32 pods for 300 s.
                                  COST: 32 x $2.89/hr x 300 s = $7.71 per cycle.
  08:30     6     2     198     ← deficit returns. Cycle repeats.
```

Compute the waste: at one cycle per ~8.5 minutes, that is ~7 cycles/hour × $7.71 ≈ **$54/hour
of pure oscillation cost**, on a workload whose correct steady-state configuration is five
replicas at $14.45/hour. **The autoscaler is costing 3.7× the thing it is scaling.**

### Step 5 — apply the derived rate limit

`T_d = 170 s`, so `periodSeconds ≥ 170`; round to 180. Commit at most 2 replicas per period.

```bash
kubectl -n inference patch scaledobject vllm-chat-8b --type merge -p '
spec:
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleUp:
          stabilizationWindowSeconds: 0
          selectPolicy: Max
          policies: [{type: Pods, value: 2, periodSeconds: 180}]
        scaleDown:
          stabilizationWindowSeconds: 900
          selectPolicy: Max
          policies: [{type: Pods, value: 1, periodSeconds: 300}]
'
```

Re-run the identical load. Representative trace:

```
  time    spec  ready  queue
  00:00     2     2       0
  00:15     4     2     271     ← +2 only. The rate limit binds immediately.
  03:00     4     4      98     ← cohort Ready. Service rate now 24 req/s vs 30.
  03:15     6     4     143     ← the NEXT decision sees a queue reflecting real
                                  remaining deficit, not the accumulated backlog.
  06:10     6     6       9     ← 36 req/s of capacity vs 30 of demand. Settled.
  06:25     6     6       2
  ...
  21:25     5     5       4     ← one replica removed after the 900 s window.
                                  Stable at 5-6. NO OSCILLATION.

  Peak replicas:  6   (vs 32)
  Settling time:  ~6 min (vs never)
  Steady state:   5–6 replicas — the correct answer, ceil(30/6.0) = 5
```

| | Default behaviour | Rate-limited | Change |
|---|---|---|---|
| Peak replicas | 32 (maxReplicaCount) | 6 | −81 % |
| Steady-state replicas | oscillating 2 ↔ 32 | 5–6 | stable |
| Oscillation period | ~8.5 min, indefinitely | none | eliminated |
| Waste per hour | ~$54 | ~$0 | — |
| Time to adequate capacity | 175 s (then overshoot) | 180 s | unchanged |

**The last row is the finding worth internalising.** Rate-limiting did **not** make the
autoscaler slower to reach adequate capacity — the first cohort arrives at the same moment
either way, because the dead time is the dead time. All it removed was the nine additional
decisions taken while the first one was still in flight. **You give up nothing by rate-limiting
to your dead time; you cannot go faster than the dead time anyway.**

### Step 6 — add feed-forward and re-measure the SLO

Add the `demand` trigger from §9 with `threshold: "4.2"` (= 6.0 req/s per replica × 0.7
headroom). Re-run and record TTFT p99 during the ramp:

| Configuration | Peak replicas | TTFT p99 during ramp | Requests over 500 ms |
|---|---|---|---|
| Default (queue only) | 32 | 47.2 s | 2,704 |
| Rate-limited (queue only) | 6 | 41.8 s | 2,180 |
| Rate-limited + feed-forward | 6 | 9.4 s | 421 |

Feed-forward does not shorten the dead time — nothing does — but it starts the clock at the
moment demand changes instead of at the moment a backlog has accumulated, which is worth
about 80 % of the affected requests. **Neither configuration protects a 500 ms SLO through a
5× step change**, and that is the honest conclusion: with a 170-second cold start, the only
thing that protects a sub-second TTFT SLO through a step change is capacity that already
exists. Autoscaling manages your *cost* at steady state and your *recovery* after a step; it
does not make a cold start disappear. That is 07.9's problem.

## Practice

Feeds the [cost-per-token deliverable](../practice/cost-per-token/README.md) — component 3
(cold start) and the utilisation term in component 1.

### 1. Verify the metric before you build on it

Deploy vLLM with a `ServiceMonitor`, and confirm in Prometheus that
`sum(vllm:num_requests_waiting{...})` returns a non-empty vector. Then dump
`curl -s localhost:8000/metrics | grep '^# HELP vllm:'` and diff it against the names you
intend to use.

**Acceptance:** the query result JSON showing a `result` array with one element, and the
metric-name dump. If any name you planned to use is absent, say which and what it was renamed
to.

### 2. Measure your dead time

Instrument the five timestamps from the worked example's Step 3 and compute
`T_observe`, `T_decide`, `T_place`, `T_start` and `T_d`.

**Acceptance:** the five-row breakdown with your measured seconds, and a statement of which
term dominates. Label clearly whether this was measured on `kind` or on a GPU cluster that
actually provisions nodes — they are different numbers and only one of them is real.

### 3. Derive the threshold rather than choosing it

Using your 07.5 measurements (`Request throughput (req/s)` per replica, prefill token rate,
prompt length) and your TTFT SLO, derive `Q_max`, `Q_mean_max` and a threshold.

**Acceptance:** the arithmetic written out with units, and the resulting threshold — plus a
sentence on whether a queue trigger alone can protect your SLO at your measured `T_d`, and
why.

### 4. Reproduce the oscillation, then kill it

Run the naive ScaledObject (no `behavior`) under a sustained step load and record
replicas-over-time for at least three cycles. Then apply the rate limit derived from your
`T_d` and re-run the identical load.

**Acceptance:** two replica-count time series, the peak replica count for each, the
oscillation period for the first, and the dollar cost per oscillation cycle at your GPU's
hourly rate.

### 5. Add feed-forward and quantify what it buys

Add an arrival-rate trigger with a threshold derived from your measured per-replica capacity.
Re-run and record TTFT p99 during the ramp and the count of requests over your SLO.

**Acceptance:** a three-row table (default / rate-limited / rate-limited + feed-forward) with
peak replicas, TTFT p99 during ramp, and requests over SLO — and one sentence on what
feed-forward did *not* fix.

### 6. Price the scale-to-zero decision for a real endpoint

Pick one endpoint. Measure its idle hours, its wake frequency, and its cold start
(from #2). Compute annual saving, annual GPU cost of wakes, and the fraction of your error
budget the slow wake-up requests consume.

**Acceptance:** the three numbers and a one-line verdict with `minReplicaCount` named. If the
answer is "keep a warm floor," say what `T_cold` would have to fall to for the answer to
change, and whether 07.9's levers can plausibly get there.

### 7. (Stretch) Sleep mode as the middle option

Start a server with `VLLM_SERVER_DEV_MODE=1 --enable-sleep-mode`. Measure wall-clock for
`POST /sleep?level=1` → `POST /wake_up` → first token, and compare it to your `T_start` from
#2. Check `vllm:engine_sleep_state` reflects the transition.

**Acceptance:** the two latencies side by side, the metric samples showing the state change,
and one sentence on when the GPU-still-allocated caveat makes this worth doing.

**Overall acceptance:** a working ScaledObject scaling on `vllm:num_requests_waiting` (not
CPU, not GPU utilisation), a **measured** dead time, a rate limit derived from it, the
before/after oscillation traces, and the priced scale-to-zero verdict — committed to the
deliverable. Scaling on GPU utilisation does not pass.

## Common pitfalls

- **Scaling on `DCGM_FI_DEV_GPU_UTIL`.** *Mechanism:* the field reports whether any kernel ran
  during the sample window, and during decode one always is — so it reads ~100 % at batch 1,
  at saturation, and during a preemption storm alike. It cannot distinguish healthy from
  failing, and it reads *highest* when the service is worst. A `>80 %` rule fires at idle.

- **Leaving an HPA's `metrics` block empty.** *Mechanism:* `SetDefaults_HorizontalPodAutoscaler`
  inserts a CPU `Utilization` target of 80 % when no metrics are specified. Your GPU fleet is
  now autoscaling on CPU and the manifest looks fine.

- **Using a renamed or non-existent metric with `ignoreNullValues` at its default.**
  *Mechanism:* the Prometheus scaler defaults `ignoreNullValues: true`, so an empty query
  result is reported as **zero**, not as an error. `vllm:gpu_cache_usage_perc` was renamed to
  `vllm:kv_cache_usage_perc`; a ScaledObject on the old name reports no load forever, never
  scales, and never errors. Set `ignoreNullValues: "false"` and pair it with `fallback`.

- **Setting `metricName` on a Prometheus trigger.** *Mechanism:* the field does not exist in
  the scaler's metadata struct; the external metric name is generated as `s<index>-prometheus`.
  Tutorials that include it predate its removal.

- **A query that returns multiple series.** *Mechanism:* the scaler issues an instant query
  and errors with "returned multiple elements" if the vector has more than one entry. Always
  `sum()`; a stray `by (pod)` is a scaler error rather than useful detail.

- **Using the defaults for `behavior` on a GPU workload.** *Mechanism:* `+100 % or +4 pods per
  15 s with no stabilisation` assumes a pod is Ready in seconds. At a 150-second dead time the
  controller takes ~10 independent actions on the same unserved backlog before any feedback
  arrives, overshoots to `maxReplicaCount`, drains the queue in seconds, then holds those
  replicas for the 300-second scale-down stabilisation window. Rate-limit `scaleUp.periodSeconds`
  to at least your measured `T_d`.

- **Confusing `cooldownPeriod` with scale-down delay.** *Mechanism:* `cooldownPeriod` (default
  300 s) gates only the deactivation path — the transition to `idleReplicaCount` or
  `minReplicaCount` once all triggers are inactive. Scale-down within 1..N is the HPA's
  `scaleDown.stabilizationWindowSeconds`. Setting the former and expecting the latter's
  behaviour is a common and confusing miss.

- **Scaling to zero on a trigger that only exists when replicas exist.** *Mechanism:* at zero
  replicas there is no vLLM pod and therefore no `vllm:*` metric; the wake-up trigger must
  read something upstream — ingress request rate, an HTTP add-on, or a broker queue depth.

- **Configuring one loop and not the other.** *Mechanism:* KEDA changes `spec.replicas` and
  reports success; if no node autoscaler is watching for unschedulable GPU pods, those replicas
  sit `Pending` indefinitely. Conversely, a node autoscaler with no ScaledObject never hears
  that capacity is wanted. Check for `Pending` pods with `Insufficient nvidia.com/gpu`.

- **Validating a threshold on `kind`.** *Mechanism:* on a local cluster the node exists, the
  image is cached, and the model is tiny, so `T_d` is ~12 s. On a GPU cluster that provisions
  a node it is 200 s to 20 minutes. A safety factor tuned against the first is meaningless for
  the second, and the gap is 15–100×.

- **Leaving `maxReplicaCount` at its default.** *Mechanism:* KEDA defaults it to 100. A metric
  glitch, a bad PromQL, or one oscillation cycle at that ceiling is a $289/hour event on H100s.
  Set it to something you would be willing to pay for by accident.

## Self-check

**(a) Why is GPU utilisation a bad autoscaling signal, and what specifically makes queue depth
better?**

**Answer:** `DCGM_FI_DEV_GPU_UTIL` measures the fraction of the sample window in which at
least one kernel was resident — an occupancy-of-*time* measure, not occupancy-of-hardware.
Autoregressive decode has a kernel running essentially continuously, so the field reads
~95–100 % at batch 1, at the saturated operating point, and during a preemption storm alike.
It is therefore not monotone in unserved demand, saturates at the bottom of the useful range,
and reads *highest* exactly when the service is failing. It is also lagging: any accelerator
counter reflecting overload appears only after requests are already queued. `vllm:num_requests_waiting`
is leading (it rises the instant arrival rate exceeds service rate, before latency degrades),
monotone in unserved demand, unbounded so it can express magnitude, and summable across
replicas so the HPA's per-replica arithmetic works on it. Its weakness is the flip side of
being unbounded: it is an *integrating* signal, which is what makes §6's oscillation possible
and why it needs a rate limit rather than just a threshold.

**(b) Write the HPA's replica calculation for a KEDA Prometheus trigger, and say what the
`threshold` means.**

**Answer:** KEDA's default `metricType` is `AverageValue`, so the HPA runs
`GetExternalPerPodMetricReplicas`: it sums the returned metric values into `usage`, computes
`usageRatio = usage / (threshold × currentReplicas)`, and if that ratio is outside the ±10 %
tolerance band sets `replicaCount = ceil(usage / threshold)`. So **`desiredReplicas =
ceil(total_metric / threshold)`**, and the current replica count enters only through the
tolerance test. `threshold` therefore means *"the metric value that one replica should
carry"*, and your PromQL must return a **sum across all replicas** — a per-pod average would
be off by the replica count. Two further consequences: inside the ±10 % band nothing changes
at all (a total queue of 9–11 against threshold 5 and 2 replicas is a no-op), and the `ceil`
makes rounding a large fraction of the answer at small replica counts. Pod readiness does not
appear in this path, which is precisely what allows booting pods to fail to damp the
recommendation.

**(c) Your autoscaler oscillates between 2 and 32 replicas on steady traffic. Explain the
mechanism and give the fix.**

**Answer:** Queue depth is the time-integral of the arrival/service deficit, so under
sustained under-provisioning it grows without bound. The controller recomputes
`ceil(total/threshold)` every ~15 s, and the default scale-up policy (`+100 % or +4 pods per
15 s, select Max, no stabilisation`) lets it act each time — so during one dead time
(scrape + poll + schedule + node + image + weight load + probe ≈ 170 s warm) it takes ~10
independent actions on the *same* unserved backlog, because nothing tells it the earlier
actions have not landed. All cohorts become Ready within seconds of each other, the backlog
drains in seconds, and the desired count collapses. The 300-second scale-down stabilisation
window (which takes the *maximum* recommendation over the window) then holds the over-provision
for five minutes before removing it, at which point the deficit returns and the cycle repeats
— **so the oscillation period is set by the stabilisation window, not by anything in the
traffic.** The fix is an actuator rate limit: set
`behavior.scaleUp.policies[].periodSeconds ≥ measured T_d` (180 s for a 170 s dead time) with
a `value` equal to the replicas you will commit before feedback, and widen
`scaleDown.stabilizationWindowSeconds` to ~3× `T_d` because a wrongly-removed replica costs a
full cold start to replace. Crucially, this costs you nothing in responsiveness — the first
cohort arrives at the same instant either way.

**(d) At a measured 170-second cold start, can a queue-depth trigger protect a 500 ms TTFT p99
SLO through a 5× step change in load? Show the arithmetic.**

**Answer:** No. From 07.5's operating point, each queued request adds ~21.6 ms to TTFT, so
the SLO permits about `500/21.6 − 1 ≈ 22` queued requests, and after a p99-vs-mean correction
of ~1.75× the steady-state mean queue must stay below ~12 per replica. But during the dead
time the queue grows at the deficit rate: a 5× step from 12 to 30 req/s against 2 replicas
serving 6.0 req/s each is an 18 req/s deficit, so `18 × 170 = 3,060` requests accumulate
before the first new replica serves anything — **250× the SLO's limit.** No threshold fixes
this, because the queue that accumulates during the delay is a property of the delay, not of
the trigger. The honest architecture is therefore: (1) a warm floor sized to p95 traffic so
the step does not start from an under-provisioned base; (2) feed-forward on arrival rate so
the clock starts when demand moves rather than when a backlog has formed (worth ~80 % of the
affected requests in the worked example); and (3) the queue trigger as the correction term
for what feed-forward gets wrong. Autoscaling manages steady-state cost and post-step
recovery; it does not make a cold start disappear — that is 07.9's problem.

**(e) What is the division of labour between KEDA's operator loop, the HPA, and the node
autoscaler, and what breaks if you configure only some of them?**

**Answer:** Three loops with three periods. **KEDA's operator loop** (period
`spec.pollingInterval`, default 30 s) owns only the zero boundary: it scales 0 → `minReplicaCount`
(or 1) when any trigger reports `value > activationThreshold`, and back to zero when all
triggers have been inactive for `cooldownPeriod` (default 300 s). It must own this because the
HPA controller disables itself at zero replicas. **The HPA** (`keda-hpa-<name>`, sync period
15 s) owns 1 ↔ N, using the external metric `s<index>-prometheus` that KEDA's metrics adapter
serves, with the `ceil(total/threshold)` arithmetic and the rate policies. **The cluster
autoscaler or Karpenter** owns nodes: it watches for `Pending` pods and provisions GPU
instances, typically 60 s to 5 minutes, usually the largest single term in the dead time.
Configure KEDA without the node layer and replicas sit `Pending` forever while the ScaledObject
reports success (it did change `spec.replicas`). Configure the node layer without KEDA and
nothing ever asks for capacity. Configure both on the wrong metric and you provision H100
nodes on a signal that does not track demand. Diagnose with
`kubectl get pods | grep Pending` and check the event for `Insufficient nvidia.com/gpu`.

**(f) When should an endpoint scale to zero, and what is the option between "warm replica" and
"zero"?**

**Answer:** Price it. Saving is `idle_hours × replicas × $/GPU-hr × 365`; cost is the GPU-time
of the wakes *plus* the SLO cost of requests caught behind them. For one H100 at $2.89/hr idle
16 h/day, the saving is ~$16.9k/yr and the GPU cost of eight daily 170-second wakes is ~$400/yr
— so dollar-for-dollar it is obvious. **The binding cost is the SLO**: ~40 requests per wake ×
8 wakes × 365 = ~117k requests/yr seeing a 170-second TTFT, which for a 2 M-request/yr endpoint
with a 99.5 % objective is 5.8 % of traffic — eleven times the entire error budget. So scale to
zero when the consumers are machines or the SLO is measured in minutes (batch, async, nightly
internal tools, dev, eval harnesses), and keep `minReplicaCount ≥ 1` when a human is waiting.
It is a **per-endpoint** decision, not a platform policy. The middle option is **vLLM sleep
mode**: `--enable-sleep-mode` with `VLLM_SERVER_DEV_MODE=1` exposes `POST /sleep?level=1`,
which offloads weights to CPU RAM and discards the KV cache while keeping the process, CUDA
context and compiled graphs alive — so waking is a PCIe copy rather than a storage read plus
compile plus capture. Level 2 discards the weights too and needs
`collective_rpc("reload_weights")`. The state is reported as
`vllm:engine_sleep_state{sleep_state=...}`. The caveat is that the GPU stays allocated: sleep
mode converts idle *HBM* into a shareable resource, it does not stop the meter, so it pays
only when something else can use the card.

## Connections & what's next

This lesson closed the largest remaining term in 07.5's cost equation — utilisation — and, in
doing so, converted the cold start from an inconvenience into the parameter that governs
whether a control loop is stable. The three-signal reading from 07.2 and 07.4 reappears here
as the metric-selection argument, and `vllm:kv_cache_usage_perc` earns a third role: capacity
diagnostic (07.2), tuning target (07.4), and now the guard trigger that catches a
prompt-length shift no request-rate metric can see. The dead-time measurement you produced is
the input to everything in the next lesson.

**Next: [07.9 — Model loading and storage](09-model-loading-storage.md)** decomposes `T_start`
— the term that dominated your dead time — into storage read, host-to-device copy, and graph
capture, gives each stage its real bandwidth, and shows which architectural changes actually
move it. That is what decides whether the scale-to-zero verdict you priced in this lesson can
ever flip.

## References & further reading

**Primary sources (Kubernetes HPA on `master`; KEDA @ `d2634e6`, 2026-08-17, v2.20.x line; vLLM v0.27.1 cross-checked against `main` @ `c1e4387` — all read from `kubernetes/kubernetes`, `kedacore/keda`, and `vllm-project/vllm`)**

1. **`pkg/controller/podautoscaler/replica_calculator.go`** — https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/podautoscaler/replica_calculator.go — `GetExternalPerPodMetricReplicas` (the `AverageValue` path KEDA uses): sums the metric series, computes `usageRatio = usage / (target × statusReplicas)`, and on a ratio outside tolerance sets `replicaCount = ceil(usage / target)`. Also `Tolerances.isWithin` and the `math.Ceil(usageRatio × readyPodCount)` form used for resource metrics. §3's arithmetic is quoted from here.
2. **`pkg/apis/autoscaling/v2/defaults.go`** — https://github.com/kubernetes/kubernetes/blob/master/pkg/apis/autoscaling/v2/defaults.go — `defaultHPAScaleUpRules` (stabilization 0, selectPolicy Max, `{Pods, 4, 15s}` and `{Percent, 100, 15s}`) and `defaultHPAScaleDownRules` (`{Percent, 100, 15s}`, stabilization nil ⇒ controller default). Also `SetDefaults_HorizontalPodAutoscaler`, which inserts an 80 % CPU utilisation target when `spec.metrics` is empty — the "empty metrics block autoscales your GPUs on CPU" pitfall.
3. **`pkg/controller/podautoscaler/config/v1alpha1/defaults.go`** — https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/podautoscaler/config/v1alpha1/defaults.go — the controller-manager defaults: `HorizontalPodAutoscalerSyncPeriod` 15 s, `HorizontalPodAutoscalerDownscaleStabilizationWindow` **5 minutes**, `HorizontalPodAutoscalerTolerance` **0.1**, `HorizontalPodAutoscalerInitialReadinessDelay` 30 s, `HorizontalPodAutoscalerCPUInitializationPeriod` 5 minutes.
4. **`pkg/controller/podautoscaler/horizontal.go`** — https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/podautoscaler/horizontal.go — `normalizeDesiredReplicasWithBehaviors` and the ordering it documents (bounds → behaviour limits → period constraints → stabilisation), `getReplicasChangePerPeriod`, and the legacy `scaleUpLimitFactor = 2.0` / `scaleUpLimitMinimum = 4.0` constants the v2 defaults reproduce.

5. **`apis/keda/v1alpha1/scaledobject_types.go`** — https://github.com/kedacore/keda/blob/main/apis/keda/v1alpha1/scaledobject_types.go — the full `ScaledObjectSpec`: `pollingInterval`, `initialCooldownPeriod`, `cooldownPeriod`, `idleReplicaCount`, `minReplicaCount`, `maxReplicaCount` (defaults `defaultHPAMinReplicas = 1`, `defaultHPAMaxReplicas = 100`), `advanced.horizontalPodAutoscalerConfig.behavior`, `scalingModifiers`, and the `Fallback` struct with its five `behavior` values. Also `keda-hpa-<name>` as the generated HPA name and `composite-metric` for scalingModifiers.
6. **`apis/keda/v1alpha1/withtriggers_types.go` and `pkg/scaling/executor/scale_executor.go`** — `defaultPollingInterval = 30` and `defaultCooldownPeriod = 5 * 60`. `pkg/scaling/executor/scale_scaledobjects.go` shows that `cooldownPeriod` gates **only** the deactivation path (`LastActiveTime + cooldownPeriod` versus now), which is the §4 correction about what that field does and does not control.
7. **`pkg/scalers/prometheus_scaler.go`** — https://github.com/kedacore/keda/blob/main/pkg/scalers/prometheus_scaler.go — the trigger's real metadata surface: `serverAddress`, `query`, `queryParameters`, `threshold`, `activationThreshold`, `namespace`, `ignoreNullValues` (**default `true`**), `unsafeSsl`. Confirms there is **no `metricName` field** (**correction** to earlier versions of this lesson), that the scaler issues an instant `/api/v1/query` and **errors on a multi-element result**, and that activity is `val > activationThreshold`.
8. **`pkg/scalers/scaler.go`** — https://github.com/kedacore/keda/blob/main/pkg/scalers/scaler.go — `GenerateMetricNameWithIndex` producing `s<index>-<name>` (hence `s0-prometheus`), and `GetMetricTargetType` defaulting to `AverageValue` when no `metricType` is set — which is why §3's `AverageValue` arithmetic is the one that applies.

9. **`vllm/v1/metrics/loggers.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/metrics/loggers.py — `vllm:num_requests_waiting`, `vllm:num_requests_waiting_by_reason` (labels `capacity` / `deferred`, summing to the total), `vllm:kv_cache_usage_perc`, `vllm:num_requests_running`, and `vllm:engine_sleep_state` with its `awake` / `weights_offloaded` / `discard_all` label values. **Correction:** `vllm:gpu_cache_usage_perc` no longer exists; it was renamed to `vllm:kv_cache_usage_perc` in V1, and `vllm:num_requests_swapped` / `vllm:cpu_cache_usage_perc` were removed with the V0 swap path.
10. **`docs/features/sleep_mode.md` and `vllm/v1/worker/gpu_worker.py`** — https://github.com/vllm-project/vllm/blob/main/docs/features/sleep_mode.md — level 1 (weights offloaded to CPU RAM, KV discarded) versus level 2 (weights discarded, buffers kept on CPU, requires `collective_rpc("reload_weights")`); the `VLLM_SERVER_DEV_MODE=1` requirement and the `/sleep?level=`, `/wake_up?tags=`, `/is_sleeping` and `/collective_rpc` endpoints; and the "Sleep mode freed X GiB memory" log line. `vllm/entrypoints/serve/dev/sleep/api_router.py` is the router itself.
11. **`vllm/benchmarks/serve.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/benchmarks/serve.py — `--request-rate` and `--ramp-up-strategy` / `--ramp-up-start-rps` / `--ramp-up-end-rps`, used to drive the step change in the worked example.

**Real-world engineering blogs**

12. **Google Cloud — "Tuning the GKE HPA to run inference on GPUs"** — https://cloud.google.com/blog/products/containers-kubernetes/tuning-the-gke-hpa-to-run-inference-on-gpus — an independent vendor's inference benchmarking concluding that CPU and memory utilisation "are no longer a good sole indicator" for inference servers, and recommending queue size instead. Read for the cross-vendor corroboration in §1. *(cloud.google.com is blocked by this environment's egress proxy; this is cited as a documented industry position rather than a page re-read for this rewrite, and §1 derives the mechanism independently from DCGM field semantics.)*

**Deeper dives**

13. **Module 05 lesson 06 — Inference SLOs** — [../../05-gpu-observability/lessons/06-inference-slos.md](../../05-gpu-observability/lessons/06-inference-slos.md) — the verified V1 metric table, the renamed/removed set that §4's `ignoreNullValues` pitfall depends on, the TTFT decomposition into queue wait plus prefill that §5's threshold derivation uses, and the burn-rate alerting that turns "requests over SLO" into an error-budget number.
