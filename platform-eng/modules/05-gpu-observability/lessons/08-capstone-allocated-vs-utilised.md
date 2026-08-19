---
lesson: "05.8"
title: "Capstone — allocated vs utilised (your dashboard is lying)"
module: "05"
concept: "Capstone — allocated vs utilised (your dashboard is lying)"
status: not-started
est_time: "10h"
prev: "07-profiling-escalation.md"
next: null
artifacts: []
sources: 13
---

# 05.8 · Capstone — allocated vs utilised (your dashboard is lying)

> **Concept.** Ship the per-namespace **allocated-vs-utilised GPU-hours** dashboard, render the gap in **dollars**, and prove the `GPU_UTIL=100% / SM_ACTIVE≈0` lie from your own cluster. This is the flagship public artifact.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Where this fits

This capstone pulls every lesson in the module into one shipped artifact. 05.1–05.2 gave you the honest metric (`SM_ACTIVE` vs the `GPU_UTIL` duty-cycle lie) and the profiling-sampler mechanics behind it. 05.3 gave you the exporter configuration to turn it on safely at fleet scale, and the rule that a custom counters CSV *replaces* rather than extends the default set. 05.4 gave you the per-namespace attribution join and its hard limit under time-slicing. 05.5 gave you the health layer that keeps "idle" from silently meaning "degraded." 05.6 gave you the roofline arithmetic that explains *why* a decode-bound server pins `GPU_UTIL` at 100% with the tensor pipes idle, plus goodput as the honest efficiency number for serving. 05.7 gave you the escalation ladder that turns a finding into a fix, and the round-trip note that proves the honest metric can be moved.

Nothing here is a new mechanism. What is new is the **arithmetic that turns a 0–1 ratio into billable hours without lying**, the **decomposition that says where the gap actually comes from**, the **validation that proves your number is right before you publish it**, and the **packaging** that turns seven lessons' worth of correct metrics into one artifact a hiring panel, a CFO, and your own cost tooling can all consume.

Read this lesson as a build. Every query below is complete and runnable; every number is derived; every step has an acceptance check. If you follow it end to end you finish with the dashboard, the exhibit, the query pack and the blog draft.

## Why this matters

This is the artifact your career pivots on. Everyone applying for the same GPU-platform role can stand up `dcgm-exporter` and point Grafana at it. Almost none of them can walk into the room and say: *"Your GPU utilisation dashboard is lying to you. Here is the proof from a real cluster, here is the honest number, here is the dollar figure the gap represents, and here is how I validated it."* That sentence is simultaneously your interview opener, the thesis of a blog post that will get shared, the input dataset for module 11, and the core query of the `gpu-cost-operator`. One artifact, four payoffs.

It is not a course-invented thesis either. `nvidia-smi`'s "GPU-Util" — surfaced as `DCGM_FI_DEV_GPU_UTIL` — reads 100% whenever *at least one kernel was resident during the sample window*, regardless of whether that kernel used 1% or 100% of the SMs. Every default GPU dashboard shows this field. Every capacity-planning and chargeback decision built on it is built on sand. Companies whose entire product is closing this gap (ScaleOps) and observability vendors who built first-class GPU products around the same taxonomy (Datadog, whose metric set spans device-level `gpu.sm_active`, process-level `gpu.process.sm_active` explicitly for detecting idle or zombie allocations, Hopper-only pipeline metrics like `gpu.tensor_active`, and reliability metrics like `gpu.errors.xid.total` and `gpu.remapped_rows.*`) independently validate the framing.

Industry material repeatedly puts *average* fleet GPU utilisation somewhere in the **10–25%** range while the dashboards show "busy" — a **2026, order-of-magnitude industry figure**, not a precise universal constant, and you must present it that way. **Your own cluster's measured number is the one you stand behind.** That is the whole point of building the thing.

## What's new here (calibration)

You are certified on generic FinOps framing and you run Prometheus/PromQL/Grafana at 40+ clusters — neither is re-taught. Module 03 owns the hardware util-lie *concept*; module 04 owns the pod-resources→UUID join, which you *consume* here rather than rebuild. What is genuinely new:

- **Integrating a ratio gauge into hours, correctly.** `avg_over_time(SM_ACTIVE[24h]) * 24` is the query everyone writes and it is **wrong for any workload that did not run for the whole window** — which is most of them. §3 derives the right one and shows the size of the error.
- **The gap decomposition.** "Allocated minus utilised" is one number hiding six different causes with six different owners and six different fixes. A dashboard that shows only the total is a complaint; one that decomposes it is a plan.
- **The join key is not what you think.** Whether you join on `pod` or `exported_pod` depends on a single Prometheus scrape-config setting, and getting it wrong produces an empty panel that looks exactly like a healthy one.
- **Validation as a first-class step.** You are about to publish a dollar figure with your name on it. §7 is the set of checks that must pass first, including a synthetic ground-truth test that proves your integration arithmetic against a known workload.
- **The dollar translation layer** — parameterised by GPU model, not hard-coded — because a fleet with L40S and H100 in it has two rate cards and one dashboard.
- **Packaging discipline** — an interview screenshot, a reusable query pack, and a blog draft are three deliverables with three different bars, hit from one underlying dataset.

## Core concepts

### 1. Two numbers, two completely independent data paths

The entire artifact is a subtraction between two quantities that come from different systems and are true in different senses. Getting the artifact right starts with never confusing them.

| | **Allocated GPU-hours** | **Utilised GPU-hours** |
|---|---|---|
| **Definition** | GPU-time reserved by a workload, whether or not it did anything | GPU-time during which the SMs actually had work resident |
| **Source of truth** | kubelet's pod-resources API (via the 04 join) — an *allocation record* | `DCGM_FI_PROF_SM_ACTIVE` integrated over time — a *hardware measurement* |
| **Units** | GPUs × hours | GPUs × hours × mean SM-active fraction |
| **At 0% use** | **still counts — you pay** | zero |
| **Financial meaning** | the invoice | the value delivered |
| **Can it be faked?** | No — it is a scheduler fact | Yes, by using `GPU_UTIL` here. **That is the lie.** |

```
   ALLOCATION PATH (a scheduler fact)          MEASUREMENT PATH (a hardware fact)
   ═══════════════════════════════════          ══════════════════════════════════

   Pod spec                                     GPU silicon
     resources.limits:                            SM counters, HBM counters
       nvidia.com/gpu: 4                                 │
          │                                              ▼
          ▼                                       NVML / DCGM profiling module
   kube-scheduler binds pod → node                (module 8, ~1 Hz sampling)
          │                                              │
          ▼                                              ▼
   kubelet + device plugin Allocate()            nv-hostengine
     assigns SPECIFIC GPU UUIDs                    field cache: 1002 SM_ACTIVE
          │                                                     1004 TENSOR_ACTIVE
          ▼                                                     203  GPU_UTIL
   /var/lib/kubelet/pod-resources/kubelet.sock            │
     ListPodResources() →                                 ▼
       {namespace, pod, container, [GPU-UUID…]}    dcgm-exporter :9400
          │                                        (collect-interval 30000 ms)
          └──────────────┬─────────────────────────────────┘
                         │  dcgm-exporter -k joins them by UUID
                         ▼
        DCGM_FI_PROF_SM_ACTIVE{UUID="GPU-a1b2…", gpu="3", Hostname="node-17",
                               modelName="NVIDIA H100 80GB HBM3",
                               namespace="team-research", pod="llama-sft-0",
                               container="main"}  0.62
                         │
                         │   ⚠ Prometheus may rename these to exported_namespace /
                         │     exported_pod / exported_container — see §2.
                         ▼
                    Prometheus TSDB
                         │
          ┌──────────────┴──────────────┐
          ▼                             ▼
   ALLOCATED GPU-hours          UTILISED GPU-hours
   (presence of the series,     (integral of the VALUE
    integrated over time)        over time)
          └──────────────┬──────────────┘
                         ▼
                    GAP  ×  $/GPU-hour
```

**The single most important structural fact in this diagram: allocation and measurement meet exactly once, at the GPU UUID.** Everything downstream — chargeback, the reshuffle panel, the dollar figure — depends on that one join being correct. §2 is about making sure it is.

### 2. Getting "allocated" right

Three candidate sources, and they do not agree. Know which you are using and why.

| Source | What it actually measures | Fidelity | When to use |
|---|---|---|---|
| **pod-resources API** (via `dcgm-exporter -k`) | the specific GPU UUIDs the kubelet handed to a container, right now | **highest** — a real device assignment | the default; this is what the label on the DCGM series means |
| **kube-state-metrics** `kube_pod_container_resource_requests{resource="nvidia_com_gpu"}` | what the pod *asked for* in its spec | good for scheduling intent | when you want requested-vs-assigned drift, or pods that are Pending |
| **Cloud bill / node inventory** | GPUs you are paying for, allocated or not | the ground truth for money | the outer sanity check in §7, and the source of `unallocated` in §4 |

**Use the pod-resources path.** It is the only one that knows *which* GPU, which is what makes the UUID join possible at all. Enable it:

```yaml
# dcgm-exporter DaemonSet — the settings that matter for this build.
# Flags, env vars and defaults from dcgm-exporter's own CLI definition.
env:
  - name: DCGM_EXPORTER_KUBERNETES          # flag: --kubernetes / -k   default: false
    value: "true"                           # ← without this there are NO pod labels
  - name: DCGM_EXPORTER_KUBERNETES_GPU_ID_TYPE
    value: "uid"                            # flag default is GPUUID; match your join key
  - name: DCGM_EXPORTER_INTERVAL            # flag: --collect-interval / -c  default: 30000 ms
    value: "30000"
  - name: DCGM_EXPORTER_COLLECTORS          # flag: --collectors / -f
    value: "/etc/dcgm-exporter/custom-counters.csv"
  - name: DCGM_POD_RESOURCES_KUBELET_SOCKET # default: /var/lib/kubelet/pod-resources/kubelet.sock
    value: "/var/lib/kubelet/pod-resources/kubelet.sock"
volumeMounts:
  - name: pod-gpu-resources
    mountPath: /var/lib/kubelet/pod-resources
    readOnly: true
volumes:
  - name: pod-gpu-resources
    hostPath: { path: /var/lib/kubelet/pod-resources }
```

Two flags worth knowing about even though this build does not need them: `--kubernetes-enable-dra` (env `KUBERNETES_ENABLE_DRA`, default `false`) captures GPUs handed out through Dynamic Resource Allocation rather than the classic device plugin, and `--kubernetes-virtual-gpus` (env `KUBERNETES_VIRTUAL_GPUS`, default `false`) captures virtual GPUs from sharing-aware device plugins. If your cluster uses either, the default configuration will show you GPUs with **no pod labels at all**, which your dashboard will silently treat as unallocated. Check before you trust the numbers.

#### The `exported_` trap

Here is the detail that eats an afternoon. `dcgm-exporter` emits labels named `namespace`, `pod` and `container`. Prometheus, when scraping via Kubernetes service discovery, *also* attaches target labels — commonly `namespace`, `pod`, and `container` describing **the dcgm-exporter pod itself**.

Prometheus resolves that collision with `honor_labels`:

- **`honor_labels: false` (the default)** — the target labels win, and the scraped labels are renamed with an `exported_` prefix. Your series carry `exported_namespace`, `exported_pod`, `exported_container`, while plain `namespace`/`pod` refer to the *exporter's* pod in the `gpu-operator` namespace.
- **`honor_labels: true`** — the scraped labels win, and your series carry plain `namespace`/`pod` meaning the *workload's* pod.

**Both are valid; only one matches your queries.** Grouping by `namespace` under the default setting produces a dashboard where every GPU-hour in the cluster belongs to `gpu-operator`, which looks plausible enough that people ship it.

Find out which you have, in one command, before writing anything else:

```bash
curl -s http://<prometheus>/api/v1/series \
  --data-urlencode 'match[]=DCGM_FI_PROF_SM_ACTIVE' | head -c 2000
```

Then define **one** recording rule that normalises the world, and write every subsequent query against its output. This is the single highest-leverage line in the whole query pack:

```yaml
groups:
- name: gpu-cost-normalise
  interval: 30s          # MUST match dcgm-exporter's collect-interval (§3)
  rules:

  # Canonical per-GPU SM-active ratio, with a stable `ns` label regardless of
  # honor_labels. `or` takes the first vector that has samples.
  - record: gpu:sm_active:ratio
    expr: |
        label_replace(
          DCGM_FI_PROF_SM_ACTIVE{exported_namespace!=""},
          "ns", "$1", "exported_namespace", "(.*)")
      or
        label_replace(
          DCGM_FI_PROF_SM_ACTIVE{namespace!="", namespace!~"gpu-operator|kube-system"},
          "ns", "$1", "namespace", "(.*)")
      or
        label_replace(
          DCGM_FI_PROF_SM_ACTIVE,
          "ns", "__unallocated__", "", "")
```

The third branch is doing real work: **a GPU with no pod labels is not an error, it is an unallocated GPU**, and §4 needs it as its own bucket. Sending it to `__unallocated__` rather than dropping it is what makes the totals reconcile.

#### MIG and sharing modes

Under **MIG**, `dcgm-exporter` emits per-instance series carrying `GPU_I_ID` (and `GPU_I_PROFILE`) alongside the physical `UUID`. Attribution is clean — a MIG instance belongs to exactly one container — but the *accounting unit* is no longer a whole GPU. A `1g.10gb` slice is one-seventh of an H100 and must be costed as such, or you will report seven H100s' worth of allocation on one card.

Under **time-slicing** or **MPS**, per-pod attribution was never in the series: `SM_ACTIVE` is a device-level counter, several pods share the device, and the exported label is whichever mapping the exporter resolved. §4 handles this with an explicit `unattributable` bucket. **Do not silently credit shared-GPU time to whichever pod's label landed last** — that is a second, quieter lie inside the artifact that exists to expose a lie.

### 3. Getting "utilised" right — the arithmetic everyone gets wrong

`DCGM_FI_PROF_SM_ACTIVE` is a **dimensionless ratio in [0, 1]**, sampled roughly every `collect-interval`. GPU-hours is an **integral**:

```
                    T
   GPU-hours(W) =  ∫  SM_ACTIVE(t) dt      summed over every GPU series
                    0
```

Prometheus has no integral operator for gauges, so you approximate with a Riemann sum over the samples you have. Each sample represents one scrape interval Δ of wall time:

```
   GPU-hours ≈  Σ  value_i × Δ / 3600        Δ = scrape interval, seconds
                i
```

which is exactly `sum_over_time(...) * Δ / 3600`.

#### Why the obvious query is wrong

Almost every write-up (including the previous version of this lesson) uses:

```promql
# ✗ WRONG for any workload that did not run for the whole window
sum by (ns) (avg_over_time(gpu:sm_active:ratio[24h])) * 24
```

`avg_over_time` averages **only over samples that exist**. If a pod ran for 6 hours of a 24-hour window, its series exists for 6 hours, `avg_over_time` returns its mean over those 6 hours, and multiplying by 24 inflates it by **4×**.

Draw it, because the picture makes the bug obvious:

```
   Window: 24 h.  Pod ran 06:00–12:00 at a steady SM_ACTIVE of 0.60.
                  True utilised GPU-hours = 6 h × 0.60 = 3.60

   SM_ACTIVE
     1.0 ┤
         │
     0.6 ┤          ████████████████
         │          ████████████████
     0.0 ┼──────────████████████████────────────────────────────────▶ t
         0h        6h              12h                            24h
         │◀── no series ──▶│◀ 6h of samples ▶│◀──── no series ────▶│

   avg_over_time(x[24h])           = 0.60      ← mean of the SAMPLES THAT EXIST
   avg_over_time(x[24h]) * 24      = 14.40 GPU-h   ✗  4× too high

   sum_over_time(x[24h])           = 0.60 × (6×3600/30)  = 432   (720 samples… no:
                                     6 h ÷ 30 s = 720 samples, × 0.60 = 432)
   sum_over_time(x[24h]) * 30/3600 = 432 × 0.008333      = 3.60 GPU-h   ✓ exact
```

**Use `sum_over_time`.** It is scrape-count-weighted, so absent samples contribute zero rather than being extrapolated, which is precisely the semantics "the pod was not running" requires.

#### The integration constant, and how to stop it drifting

`Δ` must be the actual spacing between samples. Two ways to make that reliable:

**(a) Pin it.** Set Prometheus's `scrape_interval` for the dcgm-exporter job equal to `dcgm-exporter`'s `--collect-interval` (default **30000 ms**). Then Δ = 30 s and `Δ/3600 = 1/120`. Scraping *faster* than the collect interval is harmless for the integral (you sample the same value twice, each representing half the time) but wastes storage; scraping *slower* loses resolution on short bursts.

**(b) Better: integrate a recording rule instead of the raw series.** A recording rule fires on the rule group's `interval`, which *you* control and which does not change when someone edits a scrape config:

```yaml
- name: gpu-cost-normalise
  interval: 30s                    # ← the integration constant lives HERE
```

Then Δ is 30 s by construction. Write `30 / 3600` once, as a comment-documented constant, and never think about it again.

**(c) Or derive it, defensively.** If you cannot pin the interval, compute the mean over present samples and multiply by the present-time:

```promql
# Δ-independent form: (mean while present) × (hours present)
  sum by (ns) (
      (sum_over_time(gpu:sm_active:ratio[24h]) / count_over_time(gpu:sm_active:ratio[24h]))
    * (count_over_time(gpu:sm_active:ratio[24h]) * 30 / 3600)
  )
```

…which algebraically collapses back to form (a). The point of writing it out is that it makes the two independent factors — *how busy while present* and *how long present* — visible, and those are two different findings.

#### Allocated GPU-hours, by the same integral

Allocation is a 0/1 indicator: the series exists (with a real namespace) or it does not. So:

```yaml
  # 1 for every GPU that currently has an allocation, labelled by namespace.
  - record: gpu:allocated:indicator
    expr: clamp_max(gpu:sm_active:ratio{ns!="__unallocated__"}, 0) + 1

  # 1 for every GPU that is powered on and known to DCGM, allocated or not.
  - record: gpu:present:indicator
    expr: clamp_max(gpu:sm_active:ratio, 0) + 1
```

`clamp_max(x, 0) + 1` is the idiomatic "turn any value into 1 while keeping all labels" trick — `x * 0 + 1` works too. Both are exact and neither depends on the value.

Then:

```promql
# Allocated GPU-hours per namespace over 24h
sum by (ns) (sum_over_time(gpu:allocated:indicator[24h])) * 30 / 3600

# Utilised (SM-active) GPU-hours per namespace over 24h
sum by (ns) (sum_over_time(gpu:sm_active:ratio[24h])) * 30 / 3600
```

**These two queries are the artifact.** Everything else is decomposition, dollars and presentation.

#### One honesty caveat about `SM_ACTIVE` as "utilised"

`SM_ACTIVE` is *"the ratio of cycles an SM has at least one warp assigned"* — averaged over SMs. It is far more honest than `GPU_UTIL`, but it is still an **occupancy** measure, not a **productivity** measure. A GPU running fp32 elementwise kernels can show `SM_ACTIVE = 0.9` with `PIPE_TENSOR_ACTIVE = 0.04` (05.7's worked example did exactly that). So:

- **`SM_ACTIVE` is the right metric for the headline gap**, because the headline claim is "you allocated a GPU and nothing was scheduled on it." That is a scheduling/waste claim, and occupancy is the right evidence.
- **`PIPE_TENSOR_ACTIVE` is the right metric for the *efficiency* panel**, because "you scheduled work but it was not the work you bought this GPU for" is a different claim with a different owner (the tenant's ML engineer, not the platform team).

Ship both. Label them differently. Conflating them is how you end up telling a team their well-tuned inference service is 90% wasted when it is memory-bandwidth-bound by physics (05.6 §3).

### 4. Decomposing the gap — the panel that makes it a plan

`allocated − utilised` is one number with at least six distinct causes, different owners and different fixes. A dashboard that shows only the total is a complaint. One that decomposes it is a plan, and this decomposition is the thing that makes the blog post better than everyone else's.

```
  FLEET GPU-HOURS OVER 24 h  (N GPUs × 24 h = the ceiling; nothing may exceed it)

  GPU-h
   576 ┤━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ physical ceiling
       │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ⑥ UNALLOCATED
   500 ┤░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░     no pod holds it
       │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  ⑤ CORDONED / DRAINED
   470 ┤▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒  ④ UNATTRIBUTABLE
       │                                                            (time-slice/MPS)
   440 ┤╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬  ┐
       │╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬  │ ③ ALLOCATED-IDLE
       │╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬  │   SM_ACTIVE ≈ 0
       │╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬╬  │   ← THE HEADLINE
   242 ┤▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨  ┘
       │▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨  ② ALLOCATED-BUSY-
       │▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨     INEFFICIENT
       │▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨▨     SM high, TENSOR low
    97 ┤████████████████████████████████████████████████████████  ┐
       │████████████████████████████████████████████████████████  │ ① PRODUCTIVE
       │████████████████████████████████████████████████████████  │   tensor pipes hot
     0 ┼────────────────────────────────────────────────────────  ┘
       00:00        06:00        12:00        18:00        24:00

   ─────────────────────────────────────────────────────────────────────────────
   BUCKET                     QUERY BASIS                OWNER          FIX
   ─────────────────────────────────────────────────────────────────────────────
   ① productive               ∫ PIPE_TENSOR_ACTIVE       nobody         nothing
   ② allocated-busy-          ∫ (SM_ACTIVE − TENSOR)     tenant ML eng  05.7 ladder
      inefficient                                                       (autocast,
                                                                        fusion,
                                                                        dataloader)
   ③ allocated-idle           ∫ (1 − SM_ACTIVE) while    tenant +       right-size,
      ◀── THE HEADLINE          allocated                platform       TTL on idle
                                                                        notebooks,
                                                                        preemption
   ④ unattributable           allocated hours on time-   platform       MIG or
                              sliced / MPS GPUs                         dedicated
   ⑤ cordoned / drained       hours with GpuUnhealthy    platform       05.5 automation
                              (05.5)                                    (MTTR)
   ⑥ unallocated              present but no pod         platform /     scheduling,
                                                         capacity       autoscaling,
                                                                        commitments
   ─────────────────────────────────────────────────────────────────────────────
   ① + ② + ③ + ④  =  ALLOCATED         ⑤ + ⑥  =  paid for, never allocated
   ① + ② + ③ + ④ + ⑤ + ⑥  =  N GPUs × 24 h    ← THE RECONCILIATION IDENTITY
```

**That last line is the validation invariant, and it is what §7 checks.** If your six buckets do not sum to the physical ceiling, one of them is wrong, and you should find out which before you publish a dollar figure.

The two most important consequences of decomposing:

1. **Bucket ③ is the one you promise to recover.** It is unambiguous waste with a clear fix and no physics defending it. Buckets ① and ② are not waste in the same sense — ② is inefficiency you can improve with 05.7's ladder, but a memory-bandwidth-bound decode service (05.6) sits in ② by physics and is *not* recoverable.
2. **Buckets ⑤ and ⑥ are the platform team's own number.** Presenting them alongside the tenant buckets is what makes the artifact credible rather than accusatory: you are grading yourself on the same graph.

### 5. The dollar layer

Two rules: **parameterise by GPU model**, and **state the rate's provenance and date** every single time.

#### Attaching a rate to every GPU

A fleet with L40S and H100 in it has two rate cards. Hard-coding `* 2.50` is the mistake that makes the number indefensible. Two clean approaches:

**(a) The rate as a metric (preferred).** Expose a tiny static target — a textfile-collector file on any node, or a two-line HTTP endpoint — carrying one series per model:

```
# HELP gpu_hourly_rate_usd Effective $/GPU-hour by model. SOURCE: 2026-02 rate card, 1yr reserved.
# TYPE gpu_hourly_rate_usd gauge
gpu_hourly_rate_usd{modelName="NVIDIA H100 80GB HBM3"} 2.50
gpu_hourly_rate_usd{modelName="NVIDIA A100-SXM4-80GB"} 1.60
gpu_hourly_rate_usd{modelName="NVIDIA L40S"} 1.10
```

It is versioned in git, it is auditable, and finance can change it without a Prometheus reload of your rules.

**(b) Inline, for the blog-post build.** A recording rule using the `× 0 + rate` idiom:

```yaml
  - record: gpu:hourly_rate_usd
    # SOURCE: 2026-02 published rate cards, 1-yr reserved. Dated snapshot, not a constant.
    expr: |
        (gpu:present:indicator{modelName=~"NVIDIA H100.*"} * 0 + 2.50)
      or
        (gpu:present:indicator{modelName=~"NVIDIA A100.*"} * 0 + 1.60)
      or
        (gpu:present:indicator{modelName=~"NVIDIA L40S.*"} * 0 + 1.10)
      or
        (gpu:present:indicator * 0 + 1.00)          # fallback for unknown models
```

Either way, the money query is a `group_left` join on `modelName`, so a heterogeneous fleet costs correctly:

```promql
# $ of allocated-but-idle GPU-time per namespace, last 24h
sum by (ns) (
    sum_over_time(
      (
        (gpu:allocated:indicator - gpu:sm_active:ratio)      # idle fraction, per GPU
        * on (modelName) group_left() gpu:hourly_rate_usd    # × that model's rate
      )[24h:30s]
    )
) * 30 / 3600
```

Read the shape: `gpu:allocated:indicator − gpu:sm_active:ratio` is the per-GPU idle fraction (1 − SM_ACTIVE, but only where an allocation exists). Multiply by the rate to get dollars-per-hour, integrate to get dollars. The `[24h:30s]` subquery is what lets you `sum_over_time` an expression rather than a bare series.

#### Which rate, though?

Be explicit, because the number changes by more than 2× depending on the answer:

| Rate basis | Typical H100 figure (2026, directional) | Use when |
|---|---|---|
| Neocloud on-demand | ~$2–4 /GPU-hr | you actually buy on demand; the most defensible "we could stop paying this" number |
| 1-year reserved / committed | ~$2–2.6 /GPU-hr | you have commitments; **this is usually the honest one** for owned capacity |
| Hyperscaler on-demand | higher still, and varies by region and accelerator | you are on a hyperscaler |
| Amortised capex + power + DC | your own TCO model | you own the metal — module 11's territory |

**Present it as a dated, directional snapshot with the basis named**, e.g. *"$2.50/GPU-hr, 1-year-reserved basis, February 2026 rate card."* The fastest way to lose a CFO conversation is to have your single most challengeable number be undated and unsourced.

### 6. The exhibit — proving the lie on your own hardware

The dashboard is the argument; the exhibit is the evidence. One image: `DCGM_FI_DEV_GPU_UTIL` pinned near 100 while `DCGM_FI_PROF_SM_ACTIVE` crawls along the floor, **for the same GPU, at the same time, from your cluster.**

The detector query:

```promql
# THE UTIL-LIE DETECTOR
  DCGM_FI_DEV_GPU_UTIL > 90
and on (UUID)
  DCGM_FI_PROF_SM_ACTIVE < 0.2
```

Join on `UUID` — it is the one label that is stable across every series, unlike `gpu` (index, not unique across nodes) or `instance` (changes if you move the exporter).

You need a workload that produces the shape. Two reliable recipes:

**(a) Batch-1 LLM decode — the one to publish.** From 05.6 §3: a single-sequence decode step on an 8B bf16 model is ~4.8 ms of HBM traffic wrapped around ~16 µs of arithmetic. A kernel is resident essentially the whole time, so the occupancy duty cycle pins; the SMs are stalled on memory, so `SM_ACTIVE` (averaged over all SMs, most of which have nothing resident during a small decode kernel) stays low and `PIPE_TENSOR_ACTIVE` stays near zero.

```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct --port 8000 --max-num-seqs 1
# drive exactly one long stream:
curl -sN localhost:8000/v1/completions -H 'content-type: application/json' -d '{
  "model":"meta-llama/Llama-3.1-8B-Instruct",
  "prompt":"Write a 4000 word essay about the history of memory bandwidth.",
  "max_tokens":4000,"stream":true}' > /dev/null
```

**(b) A trivial spin kernel — the one to *understand* with.** Ten lines that pin `GPU_UTIL` at 100 while using a single SM:

```python
# util_lie.py — one block, one thread, doing nothing, forever.
import torch, time
s = torch.cuda.Stream()
x = torch.zeros(1, device="cuda")
t_end = time.time() + 900          # 15 minutes
with torch.cuda.stream(s):
    while time.time() < t_end:
        for _ in range(10_000):
            x.add_(1)              # a stream of tiny 1-element kernels
```

An H100 has 132 SMs. A one-element kernel occupies one. `GPU_UTIL` reports ~100% because a kernel was resident for essentially the whole sample window; `SM_ACTIVE` reports roughly 1/132 ≈ 0.008 because it averages resident-warp cycles across all SMs. **That ratio — two orders of magnitude between two metrics measuring "the same thing" — is the entire thesis, reproducible on any GPU in 15 minutes.**

Publish (a) because it is a real production workload people recognise. Keep (b) in the blog as the "here is why, minimally" explanation.

### 7. Validation — do this before you publish anything

You are about to put a dollar figure with your name on it in front of a hiring panel and the internet. Five checks. All of them must pass.

#### Check 1 — the reconciliation identity

The six buckets must sum to the physical ceiling:

```promql
# Physical ceiling: every GPU DCGM can see, integrated over the window.
sum(sum_over_time(gpu:present:indicator[24h])) * 30 / 3600

# Must equal (within scrape jitter):
#   allocated_gpu_hours + cordoned_gpu_hours + unallocated_gpu_hours
```

**Acceptance: agreement within ~1%.** Bigger discrepancies mean scrape gaps (check 3), a sharing mode you have not accounted for (§2), or GPUs whose exporter is down — which, note, silently *removes* them from both sides and understates your total.

#### Check 2 — the ordering invariant

Per namespace and per GPU, **utilised must never exceed allocated**:

```promql
# Must return NOTHING. Any result is a bug.
(
  sum by (ns) (sum_over_time(gpu:sm_active:ratio[24h]))
  -
  sum by (ns) (sum_over_time(gpu:allocated:indicator[24h]))
) > 0
```

A violation means work is running on a GPU with no allocation record: a process started on the host outside Kubernetes, a pod whose label mapping failed, or a DRA/virtual-GPU path the exporter is not configured for.

#### Check 3 — sample completeness

Your integral silently under-counts across scrape gaps. Measure the gaps:

```promql
# Expected samples in 24h at 30s = 2880. Anything materially below is a gap.
count_over_time(gpu:sm_active:ratio[24h]) < 2880 * 0.98
```

**Acceptance: no series below 98% completeness**, or an explicit note in the blog that the number is a floor. Gaps make waste look *smaller*, so this error is in the direction that makes you look wrong later.

#### Check 4 — synthetic ground truth (the one that actually proves the arithmetic)

This is the check almost nobody does, and it is the one that turns "I built a dashboard" into "I validated a measurement."

Run a workload with a **known, exactly computable** GPU-hour cost, then confirm the query returns it.

```bash
# Saturate ONE GPU for EXACTLY 10 minutes. gpu-burn drives essentially all SMs,
# so SM_ACTIVE should sit at ~1.0 for the duration.
#   Expected utilised GPU-hours = 10/60 = 0.1667
docker run --rm --gpus '"device=3"' oguzpastirmaci/gpu-burn 600
```

Then, immediately after:

```promql
# Restrict to that GPU and a window that comfortably contains the run.
sum(sum_over_time(gpu:sm_active:ratio{UUID="GPU-a1b2c3d4-..."}[30m])) * 30 / 3600
```

**Acceptance: within ±2 scrape intervals of 0.1667 GPU-hours — i.e. 0.150 to 0.184.** The tolerance is the quantisation error of a 30-second Riemann sum over a 600-second run (±30 s at each edge = ±0.0083 h each).

If you get **0.30** you have double counting (two exporters scraped, or a duplicated job in `scrape_configs`). If you get **~0.08** your Δ is 15 s, not 30 s. If you get **~4.0** you used `avg_over_time × window_hours` and just reproduced §3's bug. **This one test catches every arithmetic error in the pipeline**, and it costs ten minutes of one GPU.

#### Check 5 — external agreement

Compare **total allocated GPU-hours** against something outside Prometheus:

```
   Your query:  sum(sum_over_time(gpu:allocated:indicator[24h])) * 30/3600
   Compare to:  (number of GPU nodes) × (GPUs per node) × 24 × (measured allocation fraction)
   And to:      the cloud invoice line for those instances over the same day
```

They will not match exactly — the invoice bills whole instances whether or not the GPUs are allocated to pods, which is precisely bucket ⑥. **The invoice should be ≥ your allocated hours, and the difference should equal bucket ⑥ plus bucket ⑤.** If it does not, you have found either a billing surprise or a bug, and both are worth knowing before you publish.

### 8. The complete query pack

This is the reusable core. It drops verbatim into the `gpu-cost-operator` and module 11. Commit it as `queries/gpu-cost.yaml` alongside the dashboard.

```yaml
# queries/gpu-cost.yaml — Module 05 deliverable query pack.
#
# INTEGRATION CONSTANT: 30 / 3600.  The `interval:` of this rule group is the
# sample spacing for every sum_over_time below. If you change `interval`, change
# the constant. Validate with §7 check 4 after ANY change.
groups:

# ─────────────────────────────────────────────────────────────────────────────
- name: gpu-cost-normalise
  interval: 30s
  rules:

  - record: gpu:sm_active:ratio
    expr: |
        label_replace(DCGM_FI_PROF_SM_ACTIVE{exported_namespace!=""},
                      "ns","$1","exported_namespace","(.*)")
      or
        label_replace(DCGM_FI_PROF_SM_ACTIVE{namespace!="",namespace!~"gpu-operator|kube-system"},
                      "ns","$1","namespace","(.*)")
      or
        label_replace(DCGM_FI_PROF_SM_ACTIVE, "ns","__unallocated__","","")

  - record: gpu:tensor_active:ratio
    expr: |
        label_replace(DCGM_FI_PROF_PIPE_TENSOR_ACTIVE{exported_namespace!=""},
                      "ns","$1","exported_namespace","(.*)")
      or
        label_replace(DCGM_FI_PROF_PIPE_TENSOR_ACTIVE{namespace!="",namespace!~"gpu-operator|kube-system"},
                      "ns","$1","namespace","(.*)")
      or
        label_replace(DCGM_FI_PROF_PIPE_TENSOR_ACTIVE, "ns","__unallocated__","","")

  - record: gpu:present:indicator          # 1 per GPU DCGM can see
    expr: clamp_max(gpu:sm_active:ratio, 0) + 1

  - record: gpu:allocated:indicator        # 1 per GPU a pod holds
    expr: clamp_max(gpu:sm_active:ratio{ns!="__unallocated__"}, 0) + 1

  - record: gpu:hourly_rate_usd
    # SOURCE: 2026-02 rate cards, 1-yr reserved basis. Dated snapshot.
    expr: |
        (gpu:present:indicator{modelName=~"NVIDIA H100.*"} * 0 + 2.50)
      or (gpu:present:indicator{modelName=~"NVIDIA A100.*"} * 0 + 1.60)
      or (gpu:present:indicator{modelName=~"NVIDIA L40S.*"} * 0 + 1.10)
      or (gpu:present:indicator * 0 + 1.00)

# ─────────────────────────────────────────────────────────────────────────────
- name: gpu-cost-hours
  interval: 5m
  rules:

  - record: ns:gpu_hours_allocated:1d
    expr: sum by (ns) (sum_over_time(gpu:allocated:indicator[1d])) * 30 / 3600

  - record: ns:gpu_hours_utilised:1d       # SM-active-weighted
    expr: sum by (ns) (sum_over_time(gpu:sm_active:ratio[1d])) * 30 / 3600

  - record: ns:gpu_hours_productive:1d     # tensor-active-weighted
    expr: sum by (ns) (sum_over_time(gpu:tensor_active:ratio[1d])) * 30 / 3600

  - record: ns:gpu_hours_wasted:1d         # bucket ③ — THE HEADLINE
    expr: ns:gpu_hours_allocated:1d - ns:gpu_hours_utilised:1d

  - record: ns:gpu_hours_inefficient:1d    # bucket ② — busy but not tensor work
    expr: ns:gpu_hours_utilised:1d - ns:gpu_hours_productive:1d

  - record: fleet:gpu_hours_present:1d     # the physical ceiling
    expr: sum(sum_over_time(gpu:present:indicator[1d])) * 30 / 3600

  - record: fleet:gpu_hours_unallocated:1d # bucket ⑥
    expr: |
      sum(sum_over_time(gpu:sm_active:ratio{ns="__unallocated__"} * 0 + 1 [1d]))
      * 30 / 3600

  - record: ns:gpu_dollars_wasted:1d       # bucket ③, in money
    expr: |
      sum by (ns) (
        sum_over_time(
          ( (gpu:allocated:indicator - gpu:sm_active:ratio)
            * on (modelName) group_left() gpu:hourly_rate_usd
          )[1d:30s]
        )
      ) * 30 / 3600

  - record: ns:gpu_utilisation_ratio:1d    # the "how honest is this team" number
    expr: ns:gpu_hours_utilised:1d / ns:gpu_hours_allocated:1d
```

And the ad-hoc queries that go on panels rather than into rules:

```promql
# ── PANEL 1 · allocated vs utilised, per namespace, stacked ──────────────────
ns:gpu_hours_allocated:1d
ns:gpu_hours_utilised:1d

# ── PANEL 2 · the dollar gap, per namespace, sorted ──────────────────────────
sort_desc(ns:gpu_dollars_wasted:1d)

# ── PANEL 3 · THE RESHUFFLE — two rankings side by side ─────────────────────
topk(10, ns:gpu_hours_allocated:1d)      # who HOLDS the most GPU
topk(10, ns:gpu_hours_utilised:1d)       # who actually USES the most GPU

# ── PANEL 4 · the util-lie detector (feeds the exhibit) ─────────────────────
DCGM_FI_DEV_GPU_UTIL > 90 and on (UUID) DCGM_FI_PROF_SM_ACTIVE < 0.2

# ── PANEL 5 · allocated-but-idle beyond 15 minutes (the alertable one) ──────
#   Sustained near-zero SM activity on a GPU that a pod is holding.
avg_over_time(gpu:sm_active:ratio{ns!="__unallocated__"}[15m]) < 0.05

# ── PANEL 6 · fleet reconciliation (the validation panel — SHIP THIS) ───────
fleet:gpu_hours_present:1d
sum(ns:gpu_hours_allocated:1d)
fleet:gpu_hours_unallocated:1d
```

**Note the deliberate `avg_over_time` in Panel 5.** It is correct *there*: over a 15-minute window on a currently-allocated GPU you want the mean-while-present, not an integral. The rule is `sum_over_time` for **hours**, `avg_over_time` for **rates and thresholds**. Knowing which is which is the skill.

### 9. Packaging — three deliverables, three bars

One dataset, three artifacts with genuinely different standards:

| Artifact | Bar | Fails when |
|---|---|---|
| **The exhibit screenshot** | must be legible at thumbnail size and self-explanatory in one caption | it has six series, a legend nobody can read, and needs a paragraph |
| **The query pack** | must run unmodified against another cluster after changing one rate and (maybe) one label name | it hard-codes namespaces, rates or hostnames |
| **The blog post** | must survive a hostile reader who runs the queries themselves | the industry statistic is unsourced, the rate is undated, or the validation is absent |

The blog's spine, in order — this structure is doing work, do not shuffle it:

1. **The claim.** "Your GPU utilisation dashboard is lying to you." One paragraph.
2. **The mechanism.** What `GPU_UTIL` actually measures — the duty cycle of kernel residency — and why that is not utilisation. Include the 10-line spin kernel; it makes the claim reproducible in 15 minutes on any GPU.
3. **The exhibit.** The screenshot. `GPU_UTIL` 99, `SM_ACTIVE` 0.008, same GPU, same minute.
4. **The honest metric.** `SM_ACTIVE`, what it means, and the caveat that it is occupancy not productivity (hence `PIPE_TENSOR_ACTIVE` as a third line).
5. **The arithmetic.** How a 0–1 ratio becomes GPU-hours, and the `avg_over_time` bug — **including that you got it wrong first.** This is the section that makes technical readers trust you.
6. **The gap, decomposed.** The six-bucket chart on *your* numbers, with owners.
7. **The money.** Dollars per namespace, with the rate's basis and date stated.
8. **The validation.** The five checks, and the synthetic ground-truth result. This is the section that separates you from every other post on this topic.
9. **The fix.** 05.7's before/after round-trip: baseline shape → rung → finding → fix → **the honest metric moving while `GPU_UTIL` did not.**
10. **The caveats.** Time-slicing attribution, MIG accounting units, the industry statistic as directional, your own measured number as the headline.

## Perspectives

**Engineer.** "85% idle SMs" is the sentence that lands with a peer — a hardware occupancy fact, provable from your own cluster's series, with no framing required. What makes it land *harder* is the validation: any engineer's first reaction to a surprising number is "your measurement is wrong," and the synthetic ground-truth test (§7 check 4) answers that before it is asked.

**Finance / CFO.** "$X/year on the table" is the only sentence that gets budget approved. The dollar math exists to make the translation mechanical rather than persuasive-by-vibes: allocated GPU-hours (the invoice) minus SM-active GPU-hours (the value delivered), times a dated rate card, per namespace, per day. The decomposition matters here too — a CFO who hears "85% waste" and later learns that a third of it was memory-bandwidth-bound inference that physics will not give back stops trusting you.

**Platform-vendor.** ScaleOps and Datadog did not invent this gap; they built products around it because it is real and recurring enough to sell. That a paid observability product's entire GPU architecture is organised around exactly the `GPU_UTIL`-vs-`SM_ACTIVE` distinction — device-level, process-level *specifically for detecting idle and zombie allocations*, Hopper-only pipeline metrics, and XID/remapped-row reliability metrics — is independent commercial validation that this module's taxonomy is the industry's taxonomy.

**Tenant.** From inside a namespace this dashboard is a bill arriving. Two things stop it being adversarial: publish the platform's own buckets (⑤ cordoned, ⑥ unallocated) on the same chart, and pair every "you are wasting X" with 05.7's ladder so the message is "here is the finding *and* the first thing to try," not "here is your invoice."

**Failure-mode.** The gap is not a one-time finding you fix and move past — it recurs continuously as teams scale allocation faster than usage. The reshuffle, where the biggest *holder* of GPUs is not the biggest *user*, is a durable organisational pattern (teams over-request for headroom; inference services provision for peak and idle at the median), not a one-off bug. The dashboard's job is to keep surfacing it as the org grows, not to be a single audit.

## Real-world use cases

- **NVIDIA — `dcgm-exporter` CLI definition** — https://github.com/NVIDIA/dcgm-exporter — the flags and env vars this build depends on, with their real defaults: `--kubernetes` / `DCGM_EXPORTER_KUBERNETES` (default **`false`** — the pod labels this entire artifact rests on are **off** unless you turn them on), `--collect-interval` / `DCGM_EXPORTER_INTERVAL` (default **30000 ms**, which is your integration constant), `--collectors` / `DCGM_EXPORTER_COLLECTORS`, `DCGM_POD_RESOURCES_KUBELET_SOCKET` (default `/var/lib/kubelet/pod-resources/kubelet.sock`), plus `--kubernetes-enable-dra` and `--kubernetes-virtual-gpus`, both default `false`. **What it shows:** three of this lesson's most expensive failure modes — no pod labels, a wrong integration constant, and invisible DRA/vGPU allocations — are all *defaults*, not mistakes. Read the flags before you trust the graph.

- **NVIDIA — `dcgm-exporter` `etc/default-counters.csv`** — https://github.com/NVIDIA/dcgm-exporter/blob/main/etc/default-counters.csv — `DCGM_FI_DEV_GPU_UTIL` ships **enabled**; `DCGM_FI_PROF_SM_ACTIVE` ships **commented out**. **What it shows:** the lie is the default and the truth is opt-in. This one file is the strongest single piece of evidence in the blog post — the reason "every GPU dashboard shows the wrong metric" is not that everyone is careless, it is that the wrong metric is what the box ships with.

- **Prometheus — `scrape_config` `honor_labels`** — https://prometheus.io/docs/prometheus/latest/configuration/configuration/ — when a scraped label collides with a target label and `honor_labels` is false (the default), the scraped label is renamed with an `exported_` prefix. **What it shows:** the mechanism behind the `pod` vs `exported_pod` trap in §2, and why a dashboard can attribute an entire cluster's GPU-hours to the `gpu-operator` namespace while looking completely reasonable.

- **Datadog — GPU Monitoring Reference Architecture** — https://www.datadoghq.com/architecture/gpu-monitoring/ — a production metric taxonomy spanning device-level (`gpu.sm_active`), process-level per-pod attribution (`gpu.process.sm_active`, described as being for detecting **idle or zombie allocations**), Hopper-only advanced pipeline metrics (`gpu.tensor_active`, `gpu.fp16_active`, `gpu.sm_occupancy`), and reliability metrics (`gpu.errors.xid.total`, `gpu.remapped_rows.*`). **What it shows:** a paid product's entire GPU architecture mirrors this capstone's dashboard design and this module's lesson order — device metric, honest metric, per-pod attribution, health. Independent commercial confirmation you are building the right thing.

- **ScaleOps — "GPU Cost Optimization" and "GPU Sharing in Kubernetes: MIG vs MPS vs Time-Slicing"** — https://scaleops.com/blog/gpu-cost-optimization/ · https://scaleops.com/blog/kubernetes-gpu-sharing/ — a vendor whose product *is* the allocated-vs-used gap, and which states plainly that under time-slicing multiple pods may share one physical GPU while many DCGM metrics remain device-level, so duplicated labels should not be read as exact per-pod usage. **What it shows:** commercial validation of the framing, and independent confirmation of the §2 caveat that forces the `unattributable` bucket.

- **NVIDIA GPU Operator — "Time-Slicing GPUs in Kubernetes"** — https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html — the authoritative description of how time-sliced GPUs are advertised as multiple replicas of one physical device with no memory or fault isolation between them. **What it shows:** the hardware basis for why bucket ④ has to exist — it is not a tooling gap you can configure away.

## Worked example

**The cluster.** 3 nodes × 8 GPUs = **24 GPUs**, mixed: 16 × H100 80GB, 8 × A100 80GB. One day (24 h). Rates: **H100 $2.50/GPU-hr, A100 $1.60/GPU-hr** (1-year-reserved basis, February 2026 rate card — a dated snapshot). `dcgm-exporter` with `-k`, 30 s collect interval, PROF fields enabled per 05.3.

### Step 1 — the ceiling

```
   fleet:gpu_hours_present:1d
     = 24 GPUs × 24 h = 576 GPU-hours

   Cost of the fleet for the day (nothing to do with utilisation):
     16 H100 × 24 h × $2.50 = $  960.00
      8 A100 × 24 h × $1.60 = $  307.20
                              ─────────
                              $1,267.20 / day        = $462,528 / yr
```

**This is the invoice.** Every number below decomposes it.

### Step 2 — allocated hours, per namespace

Query output from `ns:gpu_hours_allocated:1d`:

| Namespace | GPUs held (mean) | Allocated GPU-h | Mix |
|---|---|---|---|
| `team-research` | 12.0 | 288.0 | 12 × H100 |
| `team-serving` | 8.0 | 192.0 | 4 × H100, 4 × A100 |
| `team-batch` | 3.0 | 72.0 | 3 × A100 |
| `__unallocated__` | 1.0 | 24.0 | 1 × A100 |
| **total** | **24.0** | **576.0** | |

**Check 1 passes:** 288 + 192 + 72 + 24 = 576 = the ceiling. ✓

### Step 3 — utilised hours, per namespace

From `ns:gpu_hours_utilised:1d` (the `sum_over_time` integral, not `avg_over_time × 24`):

| Namespace | Allocated GPU-h | Utilised GPU-h | Mean SM_ACTIVE | Utilisation |
|---|---|---|---|---|
| `team-research` | 288.0 | 178.6 | 0.620 | **62.0%** |
| `team-serving` | 192.0 | 21.1 | 0.110 | **11.0%** |
| `team-batch` | 72.0 | 31.7 | 0.440 | **44.0%** |
| `__unallocated__` | 24.0 | 0.0 | 0.000 | 0% |
| **total** | **576.0** | **231.4** | | **40.2% of the fleet** |

**Check 2 passes:** utilised ≤ allocated everywhere. ✓

40.2% is *better* than the 10–25% industry range often quoted — which is exactly why you lead with your own measured number and cite the industry figure only as directional context. A reader who catches you inflating your own waste to match a statistic stops reading.

### Step 4 — the same numbers, computed the wrong way (do this once, deliberately)

`team-batch`'s three A100s ran a batch job for **9 hours**, not 24. Its series only existed for those 9 hours, at a mean `SM_ACTIVE` of 0.440 while running.

```
   ✗ WRONG:  sum by (ns) (avg_over_time(gpu:sm_active:ratio[24h])) * 24
             = (0.440 + 0.440 + 0.440) × 24
             = 1.32 × 24
             = 31.68 GPU-h                      ← looks right by coincidence? No:

   Careful — avg_over_time returns 0.440 PER SERIES, three series:
             3 × 0.440 × 24 = 31.68 GPU-h

   ✓ RIGHT:  sum by (ns) (sum_over_time(...[24h])) * 30/3600
             samples present = 9 h ÷ 30 s = 1,080 per GPU
             = 3 GPUs × 1,080 × 0.440 × 30/3600
             = 3 × 1,080 × 0.440 × 0.0083333
             = 11.88 GPU-h

   ERROR:    31.68 vs 11.88  →  the wrong query is 2.67× too high,
             which is exactly 24 h ÷ 9 h.
```

**Generalise it: the `avg_over_time` form overstates utilised hours by exactly `window / time-present`.** For a namespace that runs bursty jobs, that is a 2–10× error, and it errs in the direction that makes your fleet look *more* utilised and the waste look *smaller*. Which is to say: the naive query understates the problem you are trying to publish, on precisely the workloads where the problem is worst.

(The table in step 3 uses the correct integral: `team-batch`'s honest figure is **31.7 GPU-h at 44% of its allocated 72 h** because its GPUs were *allocated* for all 24 h — a reserved node pool — while only *running* for 9. If those GPUs had also been *released* after 9 hours, allocated would be 27 GPU-h and utilised 11.9, a 44% ratio on a much smaller base. **Allocated-hours accounting is what distinguishes those two situations, and it is the whole reason both numbers exist.**)

### Step 5 — decompose the gap

Add `ns:gpu_hours_productive:1d` (the `PIPE_TENSOR_ACTIVE` integral):

| Namespace | Allocated | ① productive | ② inefficient | ③ idle | ratio ①/alloc |
|---|---|---|---|---|---|
| `team-research` | 288.0 | 121.0 | 57.6 | 109.4 | 42.0% |
| `team-serving` | 192.0 | 7.3 | 13.8 | 170.9 | 3.8% |
| `team-batch` | 72.0 | 19.1 | 12.6 | 40.3 | 26.5% |
| **subtotal** | **552.0** | **147.4** | **84.0** | **320.6** | |
| ⑥ unallocated | — | — | — | 24.0 | |
| **fleet total** | | **147.4** | **84.0** | **344.6** | |

Reconciliation: 147.4 + 84.0 + 344.6 = **576.0** = the ceiling. ✓ (Buckets ④ and ⑤ are zero here — no time-slicing, no cordons that day. On a real fleet they will not be, and their absence is itself worth stating.)

### Step 6 — the money

```
   BUCKET ③ — allocated-but-idle (THE HEADLINE), per namespace:

     team-research  109.4 GPU-h × $2.50 (all H100)          = $  273.50
     team-serving   170.9 GPU-h, mix 4×H100 + 4×A100 held
                      → 85.45 h H100 × $2.50 = $213.63
                      → 85.45 h A100 × $1.60 = $136.72       = $  350.35
     team-batch      40.3 GPU-h × $1.60 (all A100)          = $   64.48
                                                              ──────────
     tenant idle waste                                        $  688.33 / day

   BUCKET ⑥ — unallocated (the PLATFORM's number):
     24.0 GPU-h × $1.60                                     = $   38.40 / day

   TOTAL idle spend                                          $  726.73 / day
                                                             $265,256 / yr

   As a share of the $1,267.20/day invoice:                        57.3 %
```

**Both numbers go on the chart.** Publishing the tenant number without the platform number is how a cost dashboard becomes politically radioactive and stops being used.

### Step 7 — the reshuffle

```
   RANKED BY ALLOCATION            RANKED BY UTILISED GPU-HOURS
   ────────────────────            ────────────────────────────
   1. team-research   288.0   ┐    1. team-research   178.6
   2. team-serving    192.0   ├──▶ 2. team-batch       31.7   ▲ up one
   3. team-batch       72.0   ┘    3. team-serving     21.1   ▼ down one

   team-serving holds 2.7× the GPUs of team-batch and does 0.67× the work.
   It is 33% of the fleet's allocation and 9% of its output.
```

**`team-serving` is the reclaim target**: 8 GPUs held, 11.0% SM-active, **$350/day** of idle allocation. And the util-lie detector confirms *why* nobody noticed — its GPUs read `GPU_UTIL` in the 80s and 90s (an inference server always has a kernel resident) while `SM_ACTIVE` sits at 0.11. **On the dashboard everyone actually looks at, `team-serving` is the busiest namespace in the cluster.**

### Step 8 — the projection, done honestly

Do not promise 100% recovery. Promise a credible partial move, and show the arithmetic so the reader can disagree with the assumption rather than the conclusion.

```
   Assumption: team-serving's batching gets fixed (05.6 — raise max_num_seqs
   toward the roofline ridge; 05.7 — remove the fp32 path found by the profiler),
   lifting mean SM_ACTIVE from 0.110 to a conservative 0.400.

   Why 0.400 is conservative:
     • well below team-research's measured 0.620 on the SAME cluster
     • far below the >0.70 that teams report after a targeted batching fix
     • memory-bandwidth-bound decode (05.6 §3) means this namespace will NEVER
       reach a training workload's tensor utilisation — the ceiling is physics,
       and any projection that ignores that is not credible

   utilised:  192.0 × 0.400 = 76.8 GPU-h   (was 21.1)
   idle:      192.0 − 76.8  = 115.2 GPU-h  (was 170.9)
   Δ idle:    170.9 − 115.2 = 55.7 GPU-h/day recovered

   In money, at team-serving's 50/50 H100/A100 mix ($2.05 blended):
     55.7 × $2.05 = $114.19 / day = $41,678 / yr

   From ONE namespace, on a 24-GPU cluster, with no new hardware.
```

Scale the argument, clearly labelled as a scaling and not a measurement:

```
   Same 57.3% idle share on a 500-GPU H100 fleet at $2.50/GPU-hr:
     allocated:  500 × 24 × 365 × $2.50            = $10.95 M / yr
     idle at 57.3%                                 = $ 6.27 M / yr

   A 10-percentage-point fleet-wide utilisation improvement:
     500 × 24 × 365 × 0.10 × $2.50                 = $ 1.10 M / yr recovered
```

That last line is the CFO sentence: **"a ten-point utilisation improvement on this fleet is about $1.1M a year, and I can show you which three namespaces it comes from."**

### Step 9 — validate before publishing

Run all five checks from §7. The one that matters most:

```
   Synthetic ground truth: gpu-burn on GPU 3 for exactly 600 s.
     Expected:  600/3600 = 0.1667 GPU-hours
     Tolerance: ±2 scrape intervals = ±0.0167
     Measured:  0.1583 GPU-hours        ✓  (within tolerance; 19 samples at
                                            ~1.0 instead of the ideal 20)
```

**Now** you can publish. That single line in the blog post — *"validated against a synthetic 10-minute full-load run; the query returns 0.158 GPU-hours against a true 0.167, within one scrape interval"* — is worth more to a technical reader than the entire dollar section, because it says you know the difference between a graph and a measurement.

## Practice — THE MODULE DELIVERABLE

Build against [`../practice/gpu-dashboard-lie/README.md`](../practice/gpu-dashboard-lie/README.md); acceptance is the [module checkpoint](../checkpoint.md). Develop every query against the [fake GPU fleet lab](../../04-gpu-on-kubernetes/practice/fake-gpu-fleet/README.md) first, then spend the rented-GPU weekend **validating** rather than debugging.

### Step 1 — instrument (1 h)

1. Deploy the GPU Operator with `dcgm-exporter`, a **custom counters CSV** containing at minimum `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_PROF_SM_ACTIVE`, `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`, `DCGM_FI_DEV_FB_USED`, `DCGM_FI_DEV_XID_ERRORS` (remember: a custom CSV **replaces** the default set).
2. Set `DCGM_EXPORTER_KUBERNETES=true` and mount the pod-resources socket.
3. Set Prometheus `scrape_interval` for that job equal to `DCGM_EXPORTER_INTERVAL`.

**Acceptance:** `curl :9400/metrics | grep SM_ACTIVE` returns a series **with namespace/pod labels on it**.

### Step 2 — resolve the join key (10 min, saves an afternoon)

Query `DCGM_FI_PROF_SM_ACTIVE` in Prometheus and inspect the label set. Write down whether you have `namespace` or `exported_namespace`. Install the `gpu:sm_active:ratio` normalising rule from §8 and confirm every series carries a sensible `ns`.

**Acceptance:** `count by (ns) (gpu:sm_active:ratio)` lists your real workload namespaces plus `__unallocated__`, and **not** `gpu-operator`.

### Step 3 — install the query pack (30 min)

Load `queries/gpu-cost.yaml` from §8. Set your own rates with the basis and date in the comment.

**Acceptance:** `ns:gpu_hours_allocated:1d` and `ns:gpu_hours_utilised:1d` both return data for every namespace.

### Step 4 — validate (1 h) — **do this before building panels**

Run all five checks from §7, including the synthetic `gpu-burn` ground truth.

**Acceptance:** reconciliation within 1%; no `utilised > allocated`; sample completeness ≥ 98%; **synthetic test within ±2 scrape intervals of the true value**; external comparison explained.

### Step 5 — capture the exhibit (1 h)

Run the batch-1 decode workload (or the spin kernel) and screenshot `DCGM_FI_DEV_GPU_UTIL` and `DCGM_FI_PROF_SM_ACTIVE` on one panel for the same GPU, with the detector query firing.

**Acceptance:** one image, legible at thumbnail size, `GPU_UTIL` > 90 and `SM_ACTIVE` < 0.2 for the same `UUID` at the same timestamp, plus a one-sentence caption that stands alone.

### Step 6 — build the dashboard (3 h)

Six panels:

1. **Allocated vs utilised GPU-hours per namespace** (stacked bar or two series).
2. **The gap decomposition** — buckets ①–⑥, stacked area over time (the §4 chart).
3. **The dollar gap per namespace**, sorted descending, with the rate basis and date in the panel description.
4. **The reshuffle** — two `topk(10, …)` tables side by side.
5. **The util-lie detector** — the exhibit query as a live panel.
6. **The reconciliation panel** — ceiling vs sum-of-buckets. **Ship this one.** It is your ongoing proof that the numbers add up, and it is the panel that catches it when someone changes the scrape interval.

**Acceptance:** the dashboard JSON is committed, the panels render on your data, and panel 6 reconciles.

### Step 7 — write the blog (3 h)

Follow §9's ten-section spine. Non-negotiable inclusions:

- The industry ~10–25% figure explicitly labelled **dated and directional**, with **your own measured number as the headline**.
- The rate's **basis and date**.
- **The `avg_over_time` bug and that you hit it.**
- **The validation section**, with the synthetic ground-truth number.
- The **time-slicing / MIG caveat** if either applies to your cluster.
- 05.7's **before/after round-trip**, including the row showing `GPU_UTIL` barely moved across a 2× speedup.

### Step 8 — make it importable (30 min)

Ensure the query pack runs unmodified after changing only the rate values and (if needed) the `ns` normalisation branch. That is the module-11 / `gpu-cost-operator` handoff.

**Overall acceptance = the module checkpoint:** the per-namespace allocated-vs-utilised dashboard renders with a dollar gap; the util-lie exhibit exists from a real cluster with the detector query; the PromQL pack is committed and reusable; the blog draft ties exhibit + query + dollar figure into the argument; and **the validation checks pass and are documented.**

## Common pitfalls

1. **`avg_over_time(SM_ACTIVE[W]) * W_hours` for GPU-hours.** *Mechanism:* `avg_over_time` averages only over samples that exist, so a workload present for a fraction `f` of the window is inflated by `1/f`. Use `sum_over_time(...) * Δ / 3600`. The error is largest on bursty workloads — exactly the ones with the most waste — and it makes waste look *smaller*.

2. **Grouping by `namespace` when Prometheus renamed it to `exported_namespace`.** *Mechanism:* `honor_labels: false` (the default) makes target labels win, so `namespace` becomes the exporter's own namespace. Symptom: every GPU-hour in the cluster attributed to `gpu-operator`, on a chart that otherwise looks fine. Normalise once with `label_replace` and never touch the raw label again.

3. **Changing the scrape or rule interval without changing the integration constant.** *Mechanism:* `Δ/3600` is hard-coded in every hours query. Halve the interval and every hours figure doubles. Pin the constant to the rule group's `interval`, document it at the top of the file, and re-run §7 check 4 after any change.

4. **Quoting "~15% average utilisation" as a precise sourced statistic.** It is an industry-consensus order of magnitude from 2026-era material, not a measured constant for any fleet. Caveat it as dated and directional, and lead with your own number.

5. **Presenting the dollar gap as money you will definitely recover.** It is money currently wasted, with a *credible partial* recovery target. The worked example's 0.110 → 0.400 projection — explicitly short of the ceiling and explicitly bounded by the memory-bandwidth physics of decode — is the right level of promise. Oversell it once and the CFO conversation is over.

6. **Silently crediting time-sliced or MPS GPU-hours to one pod.** *Mechanism:* `SM_ACTIVE` is a device-level counter; several pods share the device; the exported label is whichever mapping the exporter resolved. Bucket ④ exists so the artifact does not tell a quieter lie of its own.

7. **Costing MIG instances as whole GPUs.** A `1g.10gb` slice is one-seventh of an H100. Seven allocated slices on one card is one card's worth of cost, not seven.

8. **Using `SM_ACTIVE` as a productivity metric.** It is occupancy. A memory-bandwidth-bound inference service can be well-tuned and still sit at low tensor activity by physics (05.6 §3). Ship `PIPE_TENSOR_ACTIVE` as a separate bucket with a separate owner, or you will tell a good team they are failing.

9. **Publishing without the reconciliation panel.** Without ①+②+③+④+⑤+⑥ = ceiling on the dashboard, you have no ongoing evidence the numbers add up, and no way to notice when a config change breaks them.

10. **Shipping the `{ns!="__unallocated__"}` presence filter as a substitute for the real allocation join.** The label filter is a reasonable Prometheus-only stand-in for the blog. The `gpu-cost-operator` and a rigorous chargeback number need the actual pod-resources allocation set — including pods that are allocated but whose GPU is idle enough that a metric gap could drop them. Ship the proxy, note the limitation explicitly.

11. **Forgetting DRA and virtual-GPU allocations.** `--kubernetes-enable-dra` and `--kubernetes-virtual-gpus` both default to `false`. If your cluster uses either, those GPUs appear with **no pod labels** and land in `__unallocated__`, understating tenant allocation and overstating the platform's own bucket.

## Self-check

- **Write, from memory, the PromQL for "GPUs allocated to a pod but idle for more than 15 minutes."**
  **Answer:**
  ```promql
  avg_over_time(gpu:sm_active:ratio{ns!="__unallocated__"}[15m]) < 0.05
  ```
  Load-bearing parts: **`SM_ACTIVE`** (the honest occupancy fraction, *not* `DCGM_FI_DEV_GPU_UTIL`, which would hide every idle GPU behind a pinned duty cycle); **`avg_over_time([15m])`** for sustained idleness rather than an instantaneous dip — and note this is one of the few places `avg_over_time` is *correct*, because you want a rate over a window, not an integral; **`< 0.05`** as "essentially dead SMs"; and the **`ns!="__unallocated__"`** filter meaning an allocation exists. In the operator you replace that filter with an explicit AND against the pod-resources allocation set.

- **Why is `avg_over_time(SM_ACTIVE[24h]) * 24` wrong for GPU-hours, and by how much?**
  **Answer:** `avg_over_time` averages only over the samples that exist. A workload present for `f` of the window contributes its mean over that present time, and multiplying by the full window extrapolates it across time when the series did not exist — inflating utilised hours by exactly `1/f`. A job that ran 9 of 24 hours is overstated by 24/9 = **2.67×**. The correct form is a Riemann sum: `sum_over_time(x[24h]) * Δ / 3600`, where Δ is the sample spacing in seconds, because absent samples contribute zero — which is precisely the semantics "the pod was not running" needs. The error direction matters: the wrong query makes utilisation look higher and waste look smaller, worst on exactly the bursty workloads that waste the most.

- **Explain the utilisation trap and the allocated-vs-utilised dollar gap to a CFO in two minutes.**
  **Answer:** "Our GPU dashboard says the fleet is about 90% busy. That number is a lie — not maliciously; it is how the metric is defined. It reads 'busy' if *any* work touched the GPU in the last second, even 1% of the chip. The honest metric — the fraction of the GPU's compute cores actually working — I measured directly on our cluster: **40%**, against an industry range usually quoted around 10–25%. So we allocate and pay for 100% of these GPUs and use 40% of them. On our fleet at $2.50 an hour for H100s that is about $1,270 a day of allocation, of which roughly **$730 a day — $265,000 a year — is idle**. I built a per-team dashboard showing exactly which teams hold GPUs they are not using: one team holds a third of the fleet and does 9% of the work. If we recover even ten points of utilisation on the worst offenders, that is about $1.1M a year on a 500-GPU fleet. It is not a purchase and it is not a vendor; it is reclaiming what we already own — and I validated the measurement against a controlled test before bringing it to you." *(Allocated = the invoice; utilised = the value; the gap = the waste, in dollars, per team.)*

- **What are the six buckets the gap decomposes into, and who owns each?**
  **Answer:** ① **productive** (tensor pipes active) — nobody owns it, it is the work; ② **allocated-busy-inefficient** (SM active but tensor idle — fp32 paths, elementwise-dominated graphs, dataloader-limited steps) — owned by the tenant's ML engineer, fixed with 05.7's ladder, and partly *irreducible* for memory-bandwidth-bound inference; ③ **allocated-idle** (SM_ACTIVE ≈ 0 while a pod holds the GPU) — **the headline**, shared between tenant and platform, fixed with right-sizing, idle-notebook TTLs and preemption; ④ **unattributable** (time-sliced or MPS GPUs where per-pod attribution was never in the series) — platform, fixed with MIG or dedicated GPUs; ⑤ **cordoned/drained** (health, 05.5) — platform, fixed by reducing remediation MTTR; ⑥ **unallocated** (powered on, no pod holds it) — platform/capacity, fixed with scheduling, autoscaling and commitment strategy. The invariant that validates the whole thing: ①+②+③+④+⑤+⑥ = number of GPUs × window hours.

- **How do you prove your GPU-hours arithmetic is correct?**
  **Answer:** A **synthetic ground-truth test**. Run a workload with an exactly computable cost — `gpu-burn` on one GPU for exactly 600 seconds, which drives essentially all SMs, so the true utilised figure is 600/3600 = **0.1667 GPU-hours**. Then run the query scoped to that UUID over a window containing the run. Acceptance is ±2 scrape intervals (±0.0167 h at 30 s), because a Riemann sum quantises at each edge. The failure signatures are diagnostic: **~0.33** means double counting (two exporters or a duplicated scrape job); **~0.08** means your Δ is 15 s not 30 s; **~4.0** means you used `avg_over_time × window_hours`. Alongside it run the four other checks: the reconciliation identity, `utilised ≤ allocated` per namespace, ≥98% sample completeness, and agreement with the cloud invoice (which should exceed your allocated hours by exactly buckets ⑤ + ⑥).

- **How does the time-slicing attribution hole change what the dashboard can honestly claim?**
  **Answer:** Under time-slicing, `DCGM_FI_PROF_SM_ACTIVE` is a **device-level** counter: several pods share one physical GPU, the counter stays attached to the device, and the exported pod label is whichever mapping the exporter resolved — so it cannot be read as exact per-pod usage. Consequently the dashboard cannot rank a time-sliced namespace's SM-active hours against a dedicated-GPU namespace's; it can only report an **`unattributable_gpu_hours`** bucket for that pool. Only **MIG-partitioned or dedicated** GPUs support a clean per-namespace ranking — MIG because the hardware partition means an instance belongs to exactly one container, though MIG then introduces its own accounting subtlety: a `1g.10gb` slice is one-seventh of a card and must be costed as such. Omitting the caveat means the artifact that exists to expose one lie quietly tells another.

- **Why is `SM_ACTIVE` the right metric for the headline gap but the wrong one for efficiency?**
  **Answer:** `SM_ACTIVE` is *the ratio of cycles an SM had at least one warp resident*, averaged over SMs — an **occupancy** measure. The headline claim is "you allocated a GPU and nothing was scheduled on it," which is a scheduling/waste claim, and occupancy is exactly the right evidence for it. But occupancy does not distinguish useful arithmetic from plumbing: a GPU running fp32 elementwise kernels shows `SM_ACTIVE ≈ 0.9` with `PIPE_TENSOR_ACTIVE ≈ 0.04`. So efficiency needs the tensor metric as a separate bucket with a separate owner — and even then, a memory-bandwidth-bound decode service is at low tensor activity by physics (05.6's ridge-point arithmetic: batch-1 decode sits ~300× below an H100's ~296 FLOP/byte ridge), not by mismanagement. Conflating occupancy with productivity means telling a well-tuned inference team they are 90% wasted.

- **Which single metric is simultaneously the interview answer, the blog thesis, and the module-11 input?**
  **Answer:** The **allocated-vs-utilised GPU-hours gap** — allocated GPU-hours from the pod-resources join, minus `SM_ACTIVE`-integrated utilised GPU-hours, rendered in dollars per namespace via a dated, model-parameterised rate. It is the interview opener, the blog's thesis, and the exact quantity module 11's cost modelling and the `gpu-cost-operator` consume. Its credibility rests on three things beyond the number itself: the correct integral (`sum_over_time`, not `avg_over_time`), the decomposition into owners, and the validation.

## Connections & what's next

This is the last lesson in the module — there is no next lesson, only the gate. Everything upstream feeds it: 05.1's metric semantics, 05.2's sampler mechanics, 05.3's counters CSV (the fields this build needs are opt-in), 05.4's UUID join (and the `exported_` trap that sits on top of it), 05.5's health layer (bucket ⑤, and the reason "idle" is not automatically "wasted"), 05.6's roofline arithmetic (why bucket ② is partly irreducible, and the goodput framing for serving namespaces), and 05.7's escalation ladder (the fix that turns the gap from an observation into a recovery projection).

Two places carry it forward. The [module checkpoint](../checkpoint.md) is where you prove — unaided — that you can state what `GPU_UTIL` measures without the word "utilisation," alert on the right metric, classify XIDs, define TTFT/TPOT, write the allocated-but-idle query from memory, pass the CFO test, and produce the artifact. And **module 11 (GPU cost economics)** takes this capstone's dollar-gap query and per-namespace waste numbers as its direct input dataset — the query pack in §8 is the handoff, which is why step 8 of the practice insists it run unmodified after changing only the rates.

## References & further reading

**Primary sources**

1. **NVIDIA — `dcgm-exporter` CLI and configuration** — https://github.com/NVIDIA/dcgm-exporter — every flag, env var and default this build depends on: `--kubernetes`/`DCGM_EXPORTER_KUBERNETES` (default `false`), `--collect-interval`/`DCGM_EXPORTER_INTERVAL` (default `30000` ms — your integration constant), `--collectors`/`DCGM_EXPORTER_COLLECTORS`, `DCGM_POD_RESOURCES_KUBELET_SOCKET`, `--kubernetes-gpu-id-type`, `--kubernetes-enable-dra`, `--kubernetes-virtual-gpus`.
2. **NVIDIA — `dcgm-exporter` `etc/default-counters.csv`** — https://github.com/NVIDIA/dcgm-exporter/blob/main/etc/default-counters.csv — `DCGM_FI_DEV_GPU_UTIL` enabled, `DCGM_FI_PROF_SM_ACTIVE` commented out. The single best exhibit in the blog post after the screenshot itself.
3. **NVIDIA — DCGM field-ID reference** — https://docs.nvidia.com/datacenter/dcgm/latest/dcgm-api/dcgm-api-field-ids.html — the exact definitions of field 203 (`GPU_UTIL`), 1001 (`GR_ENGINE_ACTIVE`), 1002 (`SM_ACTIVE`), 1003 (`SM_OCCUPANCY`), 1004 (`PIPE_TENSOR_ACTIVE`), 1005 (`DRAM_ACTIVE`). Quote 1002's definition verbatim in the post; it does the arguing for you.
4. **Prometheus — `scrape_config` reference (`honor_labels`)** — https://prometheus.io/docs/prometheus/latest/configuration/configuration/ — the `exported_` label-collision mechanism behind §2's join-key trap.
5. **Prometheus — Query functions** — https://prometheus.io/docs/prometheus/latest/querying/functions/ — `sum_over_time`, `avg_over_time`, `count_over_time`, `clamp_max`, `label_replace`, and subquery syntax (`[1d:30s]`), which are the whole arithmetic of §3 and §8.
6. **Prometheus — Recording rules** — https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/ — rule-group `interval` semantics, which is where §3's integration constant should live.
7. **Kubernetes — Pod-resources API / third-party device metrics** — https://kubernetes.io/blog/2020/12/16/third-party-device-metrics-reaches-ga/ — the kubelet API `dcgm-exporter` reads to map GPU UUIDs to pods; the mechanism behind the "allocated" side of the artifact.
8. **kube-state-metrics — metrics documentation** — https://github.com/kubernetes/kube-state-metrics/blob/main/docs/README.md — `kube_pod_container_resource_requests{resource="nvidia_com_gpu"}`, the alternative allocation source (scheduling *intent* rather than device assignment) worth cross-checking against.
9. **NVIDIA GPU Operator — "Time-Slicing GPUs in Kubernetes"** — https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html — why time-sliced GPUs are advertised as replicas of one device with no isolation, the hardware basis for bucket ④.

**Real-world engineering**

10. **Datadog — GPU Monitoring Reference Architecture** — https://www.datadoghq.com/architecture/gpu-monitoring/ — a paid product's device/process/pipeline/error taxonomy, including `gpu.process.sm_active` for detecting idle or zombie allocations. Independent validation that this dashboard's design is the industry's design.
11. **ScaleOps — "GPU Cost Optimization"** — https://scaleops.com/blog/gpu-cost-optimization/ — the allocated-vs-used framing and reclaim economics from a vendor whose product is this gap.
12. **ScaleOps — "GPU Sharing in Kubernetes: MIG vs MPS vs Time-Slicing"** — https://scaleops.com/blog/kubernetes-gpu-sharing/ — the source of the "DCGM metrics remain device-level under time-slicing, so duplicated labels are not exact per-pod usage" caveat that forces bucket ④.

**Deeper dives**

13. **NVIDIA — "Get Real-Time Visibility into GPU Usage Across Kubernetes Clusters"** — https://developer.nvidia.com/blog/get-real-time-visibility-into-gpu-usage-across-kubernetes-clusters/ — NVIDIA's own reference deployment of the DCGM + Prometheus + Grafana stack this capstone builds on, useful for cross-checking your panel design against the vendor's.
