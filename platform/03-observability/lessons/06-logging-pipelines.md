---
lesson: "A03.6"
title: "Logging pipelines"
module: "A-03"
concept: "cost/value & label cardinality"
status: not-started
est_time: "3 hrs"
artifacts: ["loki-label-schema", "logql-straggler-query"]
---

# A03.6 · Logging pipelines

> **Concept.** Logs are the most-expensive-per-value signal — a last resort; the Loki-vs-ELK fork and label cardinality decide whether the pipeline survives at fleet scale.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Why this matters

Logs are where observability budgets go to die. They're the highest cost-per-unit-value signal you run, and at fleet scale a single bad labeling decision — one high-cardinality field promoted to an index dimension — takes down the ingest path for everyone. The staff move is not "ship logs better"; it's to demote logs to a last resort (anything a metric or an exemplar-linked trace can answer should never be a log), and then to make the two architectural decisions that determine whether the pipeline is affordable: the Loki-vs-ELK fork, and where you draw the cardinality line. On a GPU fleet, per-rank per-step logging is a firehose that will bankrupt a naive pipeline in days.

## Core notes

**Skip (you already know):** structured JSON logging; the ship→parse→index→store pipeline shape; that Fluent Bit / Vector are the agents; that logs are expensive in the abstract.

**Logs are a last resort.** Order signals by cost-per-value: metrics (cheap, aggregate) → exemplar-linked traces (targeted, one representative) → logs (expensive, per-event). Anything answerable by a metric or a trace you can navigate to via an exemplar should **not** be a log. Logs earn their place only for high-cardinality per-event forensic detail you genuinely need to grep after the fact. Treat every log line as a line item.

**The Loki vs ELK architectural fork.** This is the decision that sets your cost curve.
- **Loki:** indexes *only labels*, stores the log body as compressed chunks in object storage, and has **no full-text inverted index**. Result: often 80%+ storage reduction vs ELK, and a Prometheus-shaped label model that unifies with your metrics. Search is a brute-force scan over the chunks selected by the label matchers (grep-like), so a tight label selector is fast and a wide-open query is slow — you pay for search at query time, not ingest/storage time.
- **ELK / Elasticsearch:** builds a full inverted index over fields. Rich, fast, arbitrary full-text search — but far higher storage and much heavier operations. Choose ELK when investigative full-text search *is* the primary workload (security forensics, compliance discovery) and you're willing to pay for it. Choose Loki when logs are mostly label-scoped tail-and-filter and cost is the constraint — the common platform case.

**Label cardinality is the same bomb as in metrics.** In Loki, every unique label-set is a **stream**. Put `trace_id`, `pod_hash`, `request_id`, or any unbounded field in a *label* and you detonate stream count — ingest and query collapse, exactly as unbounded metric labels blow up a TSDB. The discipline is identical: labels are for *bounded, low-cardinality* dimensions you select on; high-cardinality data goes in the **log line**, retrieved with a LogQL line filter (`| json | ...`), never as a label. In ELK the same failure wears a different mask: **dynamic mapping** on unbounded JSON keys causes mapping/field explosion, destabilizing the cluster. Both systems die of cardinality; the symptom differs (stream explosion vs mapping explosion).

**Cost-control levers.**
- **Structure at the source** so downstream doesn't re-parse.
- **Sample** high-volume/low-value logs — keep 100% of ERROR, sample INFO/DEBUG.
- **Drop/redact fields at the collector** before storage (Vector/Fluent Bit transforms) so you never pay to store what you'll never query.
- **Route by value:** audit logs → cold object storage / long retention; debug logs → short retention / hot then expire.
- **Enforce a schema** with a bounded set of indexed fields — this is what prevents dynamic-mapping explosion in ES and stream explosion in Loki.

**GPU tie.** Training and inference logs are firehoses — per-rank, per-step, across thousands of GPUs. Keep `node`, `rank`, `job`, `tenant` as **bounded Loki labels**; push `step`, `loss`, `trace_id` into the **log line** (filter with LogQL), never into labels. NCCL debug logs (`NCCL_DEBUG=INFO`) are enormous — sample them and route to cold storage, and extract straggler/collective-timing signals as *metrics* rather than keeping the raw firehose hot. See the separately built GPU-observability artifact (DCGM / util-lie / MFU) for the metric side that should absorb most of what naive pipelines log.

## Worked example

A LogQL query and label schema for a 4k-GPU training fleet.

**Correct — bounded labels, high-cardinality in the line:**
```logql
{job="llama-70b-pretrain", tenant="research", rank=~"1[0-9]"}
  | json
  | loss > 5.0
```
Labels: `{job, node, rank, tenant}`. Stream count ≈ jobs × nodes × ranks × tenants — bounded and knowable (e.g. 1 job × 500 nodes × 8 ranks × 1 tenant ≈ 4,000 streams). `step`, `loss`, `trace_id` live in the JSON body, filtered at query time.

**Cardinality bomb — `trace_id` promoted to a label:**
```logql
{job="llama-70b-pretrain", trace_id="a1b2c3..."}   # WRONG
```
Now stream count ≈ 4,000 × (unique trace_ids) → effectively unbounded; every distinct request mints a new stream. Ingest backs up, the index balloons, queries time out. **Compute both ways:** bounded schema = ~4k streams; add `trace_id` at ~10⁶ traces/day and you're at millions of streams/day — the same detonation as an unbounded metric label. The fix is one character of judgment: `trace_id` is a line field, not a label.

## Practice

Design the logging layer of the fleet observability system. Specify: (a) the Loki-vs-ELK choice with the workload justification; (b) the Loki label schema for a multi-tenant GPU fleet and the stream-count math (bounded vs the `trace_id`-as-label bomb); (c) the collector-side cost levers (sampling rules per level, field drops/redaction, value-based routing to hot/cold retention); (d) which currently-logged signals (e.g. NCCL straggler timing) you'd promote to metrics instead. State the failure mode each choice defends against. <feeds [fleet observability design](../practice/fleet-observability/README.md)>

## Self-check

- Why is `trace_id` catastrophic as a Loki label but fine in the log line? **Answer:** Every unique label-set is a Loki stream, so promoting an unbounded field like `trace_id` to a label makes stream count effectively unbounded — one new stream per request — which collapses ingest and query, the same bomb as unbounded metric labels. In the log line it's just data, retrieved by a LogQL line filter (`| json | ...`) after label selection, with zero cardinality cost.
- When do you actually choose ELK over Loki despite the cost? **Answer:** When investigative full-text search is the *primary* workload — you need fast arbitrary queries over the log body (security forensics, compliance discovery) rather than label-scoped tail-and-filter. Loki has no inverted index and scans chunks brute-force, so wide-open full-text search is slow; ELK's inverted index is what you're paying the higher storage and ops cost for. If logs are mostly label-scoped and cost is the constraint, Loki wins.
- Same cardinality failure, two masks — name each. **Answer:** In Loki it's **stream explosion**: unbounded label values multiply the number of streams until ingest/query collapse. In Elasticsearch it's **mapping/field explosion**: dynamic mapping over unbounded JSON keys creates unbounded indexed fields, destabilizing the cluster. Same root cause (unbounded cardinality in an indexed dimension), different symptom.

## References

- https://grafana.com/docs/loki/latest/get-started/labels/
- https://grafana.com/docs/loki/latest/get-started/architecture/
- https://signoz.io/blog/loki-vs-elasticsearch/
- https://grafana.com/docs/loki/latest/query/log_queries/
