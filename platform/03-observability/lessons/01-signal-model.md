---
lesson: "A03.1"
title: "The signal model"
module: "A-03"
concept: "cardinality-as-constraint"
status: not-started
est_time: "3.5 hrs"
prev: null
next: "02-prometheus-and-promql.md"
artifacts: ["cardinality budget worksheet"]
sources: 7
---

# A03.1 · The signal model

> **Concept.** Cardinality is the master constraint that decides which signal a question belongs to — and on a GPU fleet, the wrong choice is a series-count bomb.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

This is lesson one, so it opens the module's through-line rather than following from anything: **cardinality is the master constraint, and delivered work (goodput) is the master SLI.** Every later lesson — Prometheus's failure modes at scale, the OTel Collector's role as an integration point, Loki's log-cardinality sleeper failures, the GPU/ML synthesis in lesson 9 — is a variation on "which signal, at what cost, answers this question." Get the cardinality budget wrong here and every downstream decision inherits the debt: alerting fan-in, recording-rule design, exemplar wiring, and the DCGM labeling scheme all either work with this constraint or fight it. This lesson gives you the quantified version of that constraint, not the intuition you already run on daily.

## Why this matters

At senior level you pick metrics vs logs vs traces by habit and it usually works. At staff level the question flips: you own the *budget*, and the budget is denominated in active time series, not gigabytes. A single badly-chosen label on a widely-scraped metric can 100x your TSDB head, blow out ingester memory, and page the observability team instead of the service team. The staff engineer is the person who can look at a proposed metric and say "that dimension has 10k values in prod, it does not belong on a label" before it ships — and justify it with a number.

The interview stake is that "which signal?" is a design question with a defensible, quantified answer, not a preference. Being able to derive that `series = ∏(distinct label values)` and reason about the per-series cost in the TSDB head is what separates "I'd use Prometheus" from "here is the cardinality budget and here is where it breaks." It is also the setup for every downstream decision in this module: alerting fan-in, recording rules, exemplar wiring, and the GPU-fleet labeling scheme in lesson 9.

On a GPU fleet this is not academic. `gpu_uuid`, `job_id`, `pod`, and `mig_instance` are all naturally high-cardinality, and DCGM will happily emit them as labels. At 10k GPUs times 7 MIG slices times per-tenant dimensions, naive labeling is a bomb that goes off the first time a large training job churns pod names. Deciding *here* what is a bounded label versus an exemplar reference is what keeps the fleet's own monitoring from becoming the fleet's biggest tenant.

## What's new here (calibration)

- **Skip (you already know):** the four signals exist (metrics/logs/traces + profiles/events); push vs pull collection; the RED and USE method names and when each applies.
- **New:** the quantified cost model behind "high cardinality is bad" — what actually consumes the 1–3KB/series (the labels index, not the chunk), why it's combinatorial *within* a metric but additive *across* metrics, and why a shorter-retention fix doesn't touch it.
- **New:** cardinality as a *governance* problem, not just a technical one — the enforcement mechanisms (CI linting, relabel-config backstops, Collector-side admission control) that keep the 10³–10⁴ rule from being theoretical.
- **New:** cardinality as a *time-varying* property of a label — the realistic incident is a label that was bounded becoming unbounded, not a label that was always a bomb.

## Core concepts

### Cardinality as the master constraint, quantified

The governing identity: for one metric name,

```
series = ∏ (distinct values of each label)
```

Every series is a distinct entry in the TSDB head, costing roughly **1–3 KB of RAM** while active (index entries + the in-memory chunk), before any samples hit disk. The dominant term in that cost is the **labels index** — a hashmap plus an inverted index per label pair that lets Prometheus resolve `{tenant="x", model="y"}` back to a series fast — not the chunk of samples itself. That's why the fix that actually moves the number is dropping label cardinality, not shortening the sample interval: doubling the scrape interval barely touches series count at all, because it's a different axis entirely (I/O and sample count are time-bound; series count is RAM-bound and cardinality-bound). See the [Prometheus storage docs](https://prometheus.io/docs/prometheus/latest/storage/) for the head-block mechanics behind the number.

So series count, not sample rate, is the first-order memory driver. Ten metrics at 100k series each is a million series is a few GB of head just to *exist*.

The dimensionality rule of thumb: **any dimension that could take >~10³–10⁴ distinct values in prod does not belong on a metric label.** `user_id`, `request_id`, `gpu_uuid`, `trace_id`, `pod` (when pods churn), full URL paths — these are unbounded or effectively-unbounded. The staff heuristic, memorize it: *"if it could have 10k values in prod, it's a span attribute (or a log field, or an exemplar), not a label."*

The reason it is the *master* constraint is that it multiplies — and it multiplies two different ways that are easy to conflate:

- **Within a metric, cardinality is combinatorial.** Two individually-borderline labels (tenant=200, model=30) are 6,000x, and that rides on top of the fleet fan-out (nodes x GPUs): `series = ∏(distinct label values)`.
- **Across metrics, cost is additive, not combinatorial.** A scrape target exposing ten metrics with their own label sets contributes `Σ over metrics of ∏(label cardinalities for that metric)` — a common mistake is multiplying *across* metrics as if they shared a cardinality space. They don't; each metric name has its own independent series space.

Cost intuition built on "how many of this one thing" is wrong on both counts — you have to track the product within a metric and the sum across metrics separately.

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

### Per-series cost is not uniform across backends

The 1–3KB figure is a single-node-Prometheus number, not a law of physics. Columnar, dictionary-encoded backends — M3DB, Mimir's blocks storage, VictoriaMetrics — push effective per-series overhead down by amortizing label-name/value strings across a shared dictionary instead of paying the full index cost per series per node. This is a direct driver of why fleet-scale operators migrate off single-node Prometheus at all (lesson 3 covers the migration itself); the *signal model* decision made here — what belongs on a label — determines how much that migration has to buy back.

Recording rules also carry a second-order cost most people miss: not just the storage of the output series, but the CPU/time cost of the rule *evaluation* itself, visible in `prometheus_rule_evaluation_duration_seconds`. A recording rule that pre-aggregates a cardinality bomb still has to *scan* the bomb every evaluation interval — collapsing storage cost without collapsing evaluation cost.

### Cardinality is a budget, and budgets need governance

Cardinality is a dollar figure, not just a RAM figure. At commercial vendor pricing (Datadog custom-metrics billing is the canonical example), a runaway high-cardinality metric can turn a five-figure monthly bill into six figures overnight — a genuine FinOps failure mode, not a hypothetical one. Datadog's own 2026 repricing of custom metrics ([Infinite Cardinality Metrics](https://www.datadoghq.com/blog/infinite-cardinality-metrics/)) — billing by metric *name* rather than unique series — is itself evidence that cardinality, not metric count, is what actually costs money; the vendor had to redesign pricing around the constraint this lesson teaches.

The 10³–10⁴ rule only holds in practice if someone enforces it *before* a label ships, which makes cardinality a governance problem before it's a technical one. The realistic mechanisms, in order of where they sit in the pipeline:

1. **CI-time linting** on metric/label names in the metric-definition PR — reject `user_id`, `request_id`, `gpu_uuid`-as-label patterns before merge.
2. **`metric_relabel_configs`** as the scrape-time backstop — drop or aggregate a label that got through review, without redeploying the source.
3. **Collector-side transform/admission processors** — in an OTel Collector pipeline, a transform processor that caps or rejects unbounded attribute values before they ever reach the metrics backend.

None of these are optional at fleet scale; the 10³–10⁴ rule is a policy, and policies without enforcement erode.

### Cardinality is time-varying, not static

The most realistic incident shape is not "someone added an obviously bad label." It's "a label that was bounded became unbounded." A `pod` or `version` label is fine at 200 stable pods; the same label is a bomb the moment deploy cadence shifts to per-deploy-per-canary pod churn, or a training job's `job_id`-adjacent pod names start rotating every few minutes under a scheduler doing aggressive bin-packing. Treat cardinality as a property you monitor over time (`prometheus_tsdb_head_series`, per-metric series counts) — not a one-time review checkbox — because the label that passed review six months ago may not be the label you have today.

### GPU-fleet tie

On the fleet the fatal high-cardinality labels are `gpu_uuid`, `job_id`, `pod`, and `mig_instance`. At 10k GPUs x up to 7 MIG instances x per-tenant labels, naive DCGM labeling is a cardinality bomb. The design decision to make *here* — which this lesson sets up and lesson 9 resolves — is: for each of those dimensions, is it a **bounded label** (keep it, e.g. a stable `node`/`gpu_index`), or is it an **exemplar/log reference** (drop from the label set, attach as an exemplar or emit to logs)? `gpu_uuid` in particular is stable per-device but 10k-wide fleet-wide; it is usually an identity you resolve out-of-band, not a metric label you carry on every series. (The SM_ACTIVE-vs-GPU_UTIL util-lie and MFU/goodput math live in the separate GPU-observability artifact — reference it, do not re-derive here.)

## Perspectives

**Systems-internals.** The ~1–3KB/series head-block cost is dominated by the labels index — a hashmap plus an inverted index per label pair — not the chunk of raw samples. That single fact determines where the fix has to live: the lever that moves the number is dropping label cardinality, not shortening the sample interval. Doubling the scrape interval barely helps a cardinality bomb at all, because it's operating on a different axis — I/O and sample count are time-bound, while RAM is series-count-bound. Diagnose a memory problem by asking "how many series" before you ask "how often do we sample."

**Economics.** Cardinality is a budget with a dollar figure attached, not just a RAM figure. At commercial vendor pricing, a runaway high-cardinality metric can turn a five-figure monthly observability bill into six figures overnight — a real FinOps failure mode that shows up in a finance review, not just an on-call page. Treat every proposed label as a line item, not a free technical choice.

**Org-design.** The 10³–10⁴ rule is toothless without enforcement. A staff engineer's actual leverage here isn't knowing the rule — it's building the mechanism that makes the rule stick: CI linting on metric/label definitions before merge, `metric_relabel_configs` as the scrape-time backstop when review fails, and Collector-side transform/admission processors as the pipeline-level safety net. Cardinality governance is an org-design artifact, not a wiki page.

**Failure-mode/incident.** The realistic incident is not "someone shipped an obviously unbounded label." It's "a label that was bounded became unbounded" — a `pod_id` or `version` label that was fine at 200 stable pods becomes a bomb the moment deploy cadence or scheduler behavior changes the churn rate underneath it. Model cardinality as a time-varying property of a label, and instrument for the *drift*, not just the initial review.

## Real-world use cases

- **Cloudflare, ["How Cloudflare runs Prometheus at scale"](https://blog.cloudflare.com/how-cloudflare-runs-prometheus-at-scale/).** 900+ Prometheus instances and roughly 4.9 billion active time series across the edge fleet. What it shows: at planet scale, staying alive requires organizational discipline layered on top of the technical fix — metric review and rule-linting (`pint`) as first-class parts of the pipeline, not an afterthought.
- **Uber, ["M3: Uber's Open Source, Large-scale Metrics Platform for Prometheus"](https://www.uber.com/en-IN/blog/m3/).** Uber built M3 because their existing metrics stack couldn't survive the cardinality and ingest rate at their scale (roughly 500M metrics/sec aggregated). What it shows: the signal-model choice has a real cost cliff — past a certain series-count and ingest-rate threshold, single-node/naive-federation Prometheus stops being viable and you're building (or buying) a distributed TSDB.
- **Datadog, ["Infinite Cardinality Metrics"](https://www.datadoghq.com/blog/infinite-cardinality-metrics/).** A 2026 vendor response repricing custom metrics by name rather than by unique series. What it shows: even a commercial vendor's billing model had to be redesigned around the fact that cardinality — not metric count — is what actually costs money; it's an implicit admission of the exact identity this lesson opens with.

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

   Then a **recording rule** pre-aggregates `sum by (tenant) (...)` for the per-tenant dashboard (200 output series), and tenant attribution for a *specific* hot GPU is reachable via an **exemplar** or a join against a low-churn `gpu_info{gpu, tenant}` mapping metric. Remember the recording rule's own evaluation cost: it still has to scan the 32,000 raw series every interval to produce those 200 rollups — cheaper to *store*, not free to *compute*.

Result: **32,000 raw series** for the per-GPU signal (well under a 256k budget) plus a couple hundred recording-rule series for the per-tenant view. From ~192M to ~32k — four orders of magnitude — by moving `model` and `tenant` off the hot label set. Compute both numbers yourself; the multiplication *is* the lesson.

## Practice

Feeds the [fleet observability design](../practice/fleet-observability/README.md).

Build a **cardinality budget worksheet** for the fleet's core GPU signals. For each of `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_DEV_SM_ACTIVE`, `DCGM_FI_DEV_FB_USED`, `DCGM_FI_DEV_POWER_USAGE`, and one XID-error event stream:

1. List every candidate label and estimate its prod distinct-value count at 10k GPUs / 7 MIG / N tenants.
2. Compute `series = ∏(distinct values)` for the naive labeling.
3. Classify each label: **bounded keep**, **recording-rule aggregate**, or **exemplar/log reference** — with the 10³–10⁴ rule as your cutoff and a one-line justification each.
4. Recompute the post-fix series count and check it lands under a stated fleet budget (e.g. 1M total active series for the GPU subsystem).
5. Note which questions now require an exemplar hop or a log query instead of a label filter — this is the cost you accepted.
6. For at least one label, note the *governance* mechanism that keeps your classification from silently reverting (CI lint rule, `metric_relabel_configs` entry, or Collector admission processor) — a worksheet with no enforcement plan is a wish list, not a budget.

Carry the label/exemplar decisions forward; lesson 9 turns them into the concrete DCGM relabel config.

## Common pitfalls

- **"Cardinality only matters for metrics."** It's the same failure mode wherever a dimension gets *indexed*: Loki stream labels, Elasticsearch mapping explosions, even trace backends' unbounded span-attribute indexing (see lesson 6 for the logging-pipeline version of this trap). The constraint generalizes past Prometheus.
- **"Just set a shorter retention and the cardinality problem goes away."** Retention controls disk usage for samples already written; it does nothing for the in-memory head-block RAM that OOMs Prometheus. A cardinality spike kills you in minutes, long before retention policy is even relevant.
- **"High cardinality is bad, full stop."** It's fine — often necessary — in logs and traces, where the whole point is unbounded per-event detail. It's specifically bad on an *indexed* dimension: a metric label, a Loki stream label. The problem is the index, not the cardinality in the abstract.
- **"Exemplars solve cardinality for free."** Exemplars have their own storage and retention limits — Prometheus keeps a small fixed-size buffer per series by default. They're cheap, and they're the right pattern for reaching expensive signals from cheap ones, but they are not free or unlimited.

## Self-check

- Why is series count, not sample rate or byte volume, the first-order cost driver for a metrics backend? **Answer:** Because each distinct series is a separate live entry in the TSDB head costing ~1–3 KB of RAM (dominated by the labels index, not the chunk) just to exist, and series count multiplies combinatorially as `∏(distinct label values)`; samples append cheaply to an existing series, so a high-cardinality label set explodes memory long before sample throughput does. Shortening the scrape interval barely moves this number — it's a different axis.
- A teammate wants to add `request_id` as a label on an HTTP request-rate metric "so we can find slow requests." What do you tell them, and where does that data belong? **Answer:** No — `request_id` is unbounded (far past the 10³–10⁴ cutoff), so it belongs as a span attribute on a trace (or a log field), not a metric label. Keep the metric bounded and wire an exemplar so the sampled slow request's trace_id rides along, giving you metric → exemplar → trace without indexing every request.
- State the cost/value inversion and the staff move it implies. **Answer:** Value-per-byte runs metrics > profiles > traces > logs while cost-per-byte runs the inverse; the staff move is to demote each question to the cheapest signal that still answers it and use exemplars to keep the expensive signals reachable only for the events that matter.
- A scrape target exposes ten metrics, each with its own label set. How do you combine their cardinality costs, and what's the common mistake? **Answer:** Cost is additive across metrics — `Σ over metrics of ∏(label cardinalities for that metric)` — because each metric name has its own independent series space. The common mistake is multiplying across metrics as if they shared one combinatorial space; that overstates cardinality wildly and can mask which specific metric is actually the offender.
- A label passed cardinality review six months ago and is still on the metric today. Is it still safe? **Answer:** Not necessarily — cardinality is a time-varying property of a label, not a static one. A label that was bounded (e.g. `pod` at 200 stable pods) can become a bomb if churn behavior changes underneath it (aggressive bin-packing, per-canary deploys). Monitor per-metric series counts over time rather than treating the initial review as permanent clearance.

## Connections & what's next

This lesson's cardinality budget is the constraint every later lesson inherits: lesson 2's recording rules exist to keep expensive aggregations cheap without changing the raw-series budget; lesson 3 covers what happens when a metrics system hits this constraint at genuine fleet scale (Thanos vs Mimir); lesson 6 applies the identical governance failure mode to log-stream labels; lesson 9 turns the bounded/exemplar classification made here into the concrete DCGM relabel config for a 10k-GPU fleet.

Next: [02 · Prometheus and PromQL](02-prometheus-and-promql.md) — the concrete query-semantics traps that violate the signal-fit and cost decisions made here.

## References & further reading

**Primary sources**
- [Prometheus storage docs](https://prometheus.io/docs/prometheus/latest/storage/) — the head-block/labels-index mechanics behind the 1–3KB/series figure.
- [Prometheus metric and label naming practices](https://prometheus.io/docs/practices/naming/)
- [OpenTelemetry signals concepts](https://opentelemetry.io/docs/concepts/signals/) — the collection-convergence framing referenced above.

**Real-world engineering blogs**
- Cloudflare, [How Cloudflare runs Prometheus at scale](https://blog.cloudflare.com/how-cloudflare-runs-prometheus-at-scale/)
- Uber, [M3: Uber's Open Source, Large-scale Metrics Platform for Prometheus](https://www.uber.com/en-IN/blog/m3/)
- Datadog, [Infinite Cardinality Metrics](https://www.datadoghq.com/blog/infinite-cardinality-metrics/)

**Deeper dives**
- Last9, [How to Manage High-Cardinality Metrics in Prometheus](https://last9.io/blog/how-to-manage-high-cardinality-metrics-in-prometheus/)
