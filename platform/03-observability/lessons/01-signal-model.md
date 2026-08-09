---
lesson: "A03.1"
title: "The signal model"
module: "A-03"
concept: "cardinality-as-constraint"
status: not-started
est_time: "3 hrs"
artifacts: ["cardinality budget worksheet"]
---

# A03.1 · The signal model

> **Concept.** Cardinality is the master constraint that decides which signal a question belongs to — and on a GPU fleet, the wrong choice is a series-count bomb.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Why this matters

At senior level you pick metrics vs logs vs traces by habit and it usually works. At staff level the question flips: you own the *budget*, and the budget is denominated in active time series, not gigabytes. A single badly-chosen label on a widely-scraped metric can 100x your TSDB head, blow out ingester memory, and page the observability team instead of the service team. The staff engineer is the person who can look at a proposed metric and say "that dimension has 10k values in prod, it does not belong on a label" before it ships — and justify it with a number.

The interview stake is that "which signal?" is a design question with a defensible, quantified answer, not a preference. Being able to derive that `series = ∏(distinct label values)` and reason about the per-series cost in the TSDB head is what separates "I'd use Prometheus" from "here is the cardinality budget and here is where it breaks." It is also the setup for every downstream decision in this module: alerting fan-in, recording rules, exemplar wiring, and the GPU-fleet labeling scheme in lesson 9.

On a GPU fleet this is not academic. `gpu_uuid`, `job_id`, `pod`, and `mig_instance` are all naturally high-cardinality, and DCGM will happily emit them as labels. At 10k GPUs times 7 MIG slices times per-tenant dimensions, naive labeling is a bomb that goes off the first time a large training job churns pod names. Deciding *here* what is a bounded label versus an exemplar reference is what keeps the fleet's own monitoring from becoming the fleet's biggest tenant.

## Core notes

**Skip (you already know):** the four signals exist (metrics/logs/traces + profiles/events); push vs pull collection; the RED and USE method names and when each applies.

### Cardinality as the master constraint, quantified

The governing identity: for one metric name,

```
series = ∏ (distinct values of each label)
```

Every series is a distinct entry in the TSDB head, costing roughly **1–3 KB of RAM** while active (index entries + the in-memory chunk), before any samples hit disk. So series count, not sample rate, is the first-order memory driver. Ten metrics at 100k series each is a million series is a few GB of head just to *exist*.

The dimensionality rule of thumb: **any dimension that could take >~10³–10⁴ distinct values in prod does not belong on a metric label.** `user_id`, `request_id`, `gpu_uuid`, `trace_id`, `pod` (when pods churn), full URL paths — these are unbounded or effectively-unbounded. The staff heuristic, memorize it: *"if it could have 10k values in prod, it's a span attribute (or a log field, or an exemplar), not a label."*

The reason it is the *master* constraint is that it multiplies. Two individually-borderline labels (tenant=200, model=30) are 6,000x, and that rides on top of the fleet fan-out (nodes x GPUs). Cardinality is combinatorial; cost intuition built on "how many of this one thing" is wrong.

### The signal-fit matrix

Each signal answers a different question and has a different cost/cardinality profile. Pick by fit, not habit:

| Signal | Answers | Cardinality tolerance | Cost profile |
|---|---|---|---|
| **Metrics** | aggregate trend, "is it healthy", alerting | **bounded** (this is the constraint) | cheap per value, no per-event detail |
| **Traces** | causality, latency attribution, "where did the time go" | per-event, **sampled** | expensive; sampling is the cost control |
| **Logs** | arbitrary high-cardinality context, "what exactly happened to *this* request" | **unbounded** | most-expensive-per-value; grep-at-scale |
| **Profiles** | "where does CPU/alloc go" across the fleet | continuous, whole-fleet | eBPF-cheap, always-on |
| **Events** | discrete state changes: deploys, XID errors, preemptions | discrete, annotating | cheap; they *annotate* the other signals |

Events are the connective tissue: a deploy marker, an XID error, a preemption — each is a discrete annotation that explains a step-change you see in the metrics and lets you jump to the trace/log for the "why".

### Cost/value inversion

The signals order **inversely** on value-per-byte and cost-per-byte:

```
value-per-byte:   metrics > profiles > traces > logs
cost-per-byte:    metrics < profiles < traces < logs   (inverse)
```

A metric byte is a pre-aggregated answer; a log byte is one raw event you still have to search. The **staff move** is to *demote* every question to the cheapest signal that can still answer it — answer "is p99 latency bad?" from a histogram metric, not by scanning logs — and then use **exemplars** to keep the expensive signals reachable: the metric carries a trace_id pointer on the sampled outlier, so you go metric → exemplar → trace only for the request that actually mattered. That is how you keep high-cardinality detail reachable without paying to index all of it.

The **OpenTelemetry convergence bet**: unify *collection* (one SDK/collector, one wire format, correlated resource attributes) while keeping *storage* separate and specialized (TSDB for metrics, columnar/object store for traces, log store for logs). Convergence is at the pipeline, not the backend — do not expect one database to be good at all four.

### GPU-fleet tie

On the fleet the fatal high-cardinality labels are `gpu_uuid`, `job_id`, `pod`, and `mig_instance`. At 10k GPUs x up to 7 MIG instances x per-tenant labels, naive DCGM labeling is a cardinality bomb. The design decision to make *here* — which this lesson sets up and lesson 9 resolves — is: for each of those dimensions, is it a **bounded label** (keep it, e.g. a stable `node`/`gpu_index`), or is it an **exemplar/log reference** (drop from the label set, attach as an exemplar or emit to logs)? `gpu_uuid` in particular is stable per-device but 10k-wide fleet-wide; it is usually an identity you resolve out-of-band, not a metric label you carry on every series. (The SM_ACTIVE-vs-GPU_UTIL util-lie and MFU/goodput math live in the separate GPU-observability artifact — reference it, do not re-derive here.)

## Worked example

**Cardinality budget for a single DCGM metric.**

Proposed: `DCGM_FI_DEV_GPU_UTIL` scraped across **4000 nodes x 8 GPUs**, labeled with `{tenant (200 distinct), model (30 distinct)}`.

Naive series count:

```
series = nodes x gpus x tenant x model
       = 4000 x 8 x 200 x 30
       = 32000 x 6000
       = 192,000,000 series
```

~192M series for *one gauge*. At ~2 KB/series head that is roughly **384 GB of RAM** just to hold the index and active chunks — a non-starter; a single metric would dwarf the rest of the TSDB.

**The fix.** Two moves:

1. **Drop `model` from the label set** — it is not needed at per-GPU granularity (a GPU runs one job at a time; model is derivable from job metadata / resolvable out-of-band). That removes the 30x:

   ```
   4000 x 8 x 200 = 6,400,000 series
   ```

2. **Demote `tenant` off the raw series; keep it only in a recording rule / exemplar.** The raw per-GPU utilization only needs identity you'll actually slice on live — node and gpu_index:

   ```
   raw series = 4000 x 8 = 32,000 series
   ```

   Then a **recording rule** pre-aggregates `sum by (tenant) (...)` for the per-tenant dashboard (200 output series), and tenant attribution for a *specific* hot GPU is reachable via an **exemplar** or a join against a low-churn `gpu_info{gpu, tenant}` mapping metric.

Result: **32,000 raw series** for the per-GPU signal (well under a 256k budget) plus a couple hundred recording-rule series for the per-tenant view. From ~192M to ~32k — four orders of magnitude — by moving `model` and `tenant` off the hot label set. Compute both numbers yourself; the multiplication *is* the lesson.

## Practice

Feeds the [fleet observability design](../practice/fleet-observability/README.md).

Build a **cardinality budget worksheet** for the fleet's core GPU signals. For each of `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_DEV_SM_ACTIVE`, `DCGM_FI_DEV_FB_USED`, `DCGM_FI_DEV_POWER_USAGE`, and one XID-error event stream:

1. List every candidate label and estimate its prod distinct-value count at 10k GPUs / 7 MIG / N tenants.
2. Compute `series = ∏(distinct values)` for the naive labeling.
3. Classify each label: **bounded keep**, **recording-rule aggregate**, or **exemplar/log reference** — with the 10³–10⁴ rule as your cutoff and a one-line justification each.
4. Recompute the post-fix series count and check it lands under a stated fleet budget (e.g. 1M total active series for the GPU subsystem).
5. Note which questions now require an exemplar hop or a log query instead of a label filter — this is the cost you accepted.

Carry the label/exemplar decisions forward; lesson 9 turns them into the concrete DCGM relabel config.

## Self-check

- Why is series count, not sample rate or byte volume, the first-order cost driver for a metrics backend? **Answer:** Because each distinct series is a separate live entry in the TSDB head costing ~1–3 KB of RAM (index + in-memory chunk) just to exist, and series count multiplies combinatorially as `∏(distinct label values)`; samples append cheaply to an existing series, so a high-cardinality label set explodes memory long before sample throughput does.
- A teammate wants to add `request_id` as a label on an HTTP request-rate metric "so we can find slow requests." What do you tell them, and where does that data belong? **Answer:** No — `request_id` is unbounded (far past the 10³–10⁴ cutoff), so it belongs as a span attribute on a trace (or a log field), not a metric label. Keep the metric bounded and wire an exemplar so the sampled slow request's trace_id rides along, giving you metric → exemplar → trace without indexing every request.
- State the cost/value inversion and the staff move it implies. **Answer:** Value-per-byte runs metrics > profiles > traces > logs while cost-per-byte runs the inverse; the staff move is to demote each question to the cheapest signal that still answers it and use exemplars to keep the expensive signals reachable only for the events that matter.

## References

- https://opentelemetry.io/docs/concepts/signals/
- https://prometheus.io/docs/practices/naming/
- https://last9.io/blog/how-to-manage-high-cardinality-metrics-in-prometheus/
