---
lesson: "A03.6"
title: "Logging pipelines"
module: "A-03"
concept: "cost/value & label cardinality"
status: not-started
est_time: "4 hrs"
prev: "05-distributed-tracing.md"
next: "07-slos-and-alerting.md"
artifacts: ["loki-label-schema", "logql-straggler-query", "loki-vs-elk-decision-memo"]
sources: 8
---

# A03.6 · Logging pipelines

> **Concept.** Logs are the most-expensive-per-value signal — a last resort; the Loki-vs-ELK fork and label cardinality decide whether the pipeline survives at fleet scale.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 05 built the sampling-economics lens: decide what to keep by outcome, and pay a cost proportional to what you must buffer to decide. This lesson carries that same lens over to logs, where the equivalent decision is not "which trace do I keep" but "which fields do I even index" — and where the cost curve is driven by cardinality rather than buffering. Logs sit at the expensive end of the signal-cost gradient this module has walked since lesson 01 (metrics → traces → logs), and this is the lesson that makes that cost concrete enough to defend in a design review.

## Why this matters

Logs are where observability budgets go to die. They're the highest cost-per-unit-value signal you run, and at fleet scale a single bad labeling decision — one high-cardinality field promoted to an index dimension — takes down the ingest path for everyone. The staff move is not "ship logs better"; it's to demote logs to a last resort (anything a metric or an exemplar-linked trace can answer should never be a log), and then to make the two architectural decisions that determine whether the pipeline is affordable: the Loki-vs-ELK fork, and where you draw the cardinality line. On a GPU fleet, per-rank per-step logging is a firehose that will bankrupt a naive pipeline in days.

## What's new here (calibration)

- Not "what is structured logging" or "what does the ship→parse→index→store pipeline look like" — you run this daily. The delta is the cost model behind the Loki-vs-ELK fork and the precise mechanics of where cardinality bites.
- Loki's label model is not a new thing to learn — it's Prometheus's cost model applied to logs. Recognizing that isomorphism means the cardinality discipline from lesson 01 transfers almost line-for-line, rather than needing to be relearned.
- ELK is not "the expensive legacy option" — there's a genuine, defensible case for choosing it (ad hoc forensic search over fields nobody predicted needing to search), and a staff engineer should be able to make that case confidently, not apologetically.
- GPU-fleet-specific: NCCL debug logs create a distinct duplication problem (every rank logging the same collective event within milliseconds) that a generic sampling strategy doesn't fully solve — the durable fix is extracting the signal as a metric, not sampling the logs harder.

## Core concepts

**Logs are a last resort.** Order signals by cost-per-value: metrics (cheap, aggregate) → exemplar-linked traces (targeted, one representative) → logs (expensive, per-event). Anything answerable by a metric or a trace you can navigate to via an exemplar should **not** be a log. Logs earn their place only for high-cardinality per-event forensic detail you genuinely need to grep after the fact. Treat every log line as a line item.

**The Loki vs ELK architectural fork.** This is the decision that sets your cost curve.
- **Loki:** indexes *only labels*, stores the log body as compressed chunks in object storage, and has **no full-text inverted index**. Result: often 80%+ storage reduction vs ELK (some migration reports cite 75-90%), and a Prometheus-shaped label model that unifies with your metrics. Search is a two-stage process: (1) chunks are selected cheaply by label matchers (index-driven), then (2) a linear scan/regex/JSON-parse runs over the line content within those chunks (expensive). A tight label selector keeps stage 2 small and fast; a wide-open query pushes a large scan and is slow — you pay for search at query time, not ingest/storage time.
- **ELK / Elasticsearch:** builds a full inverted index over fields. Rich, fast, arbitrary full-text search — but far higher storage and much heavier operations. Choose ELK when investigative full-text search *is* the primary workload. This is not a lesser fallback choice: for security/forensics use cases, the requirement is often genuinely ad hoc querying across fields nobody predicted needing to search (an IOC hash, an unusual header value) — a label-indexed system structurally can't serve that well no matter how the label schema is tuned, because you can't pre-declare a label for a field you didn't know mattered. Choose Loki when logs are mostly label-scoped tail-and-filter and cost is the constraint — the common platform case.
- **Migration cost heuristic.** Rough rule of thumb from reported migrations: either direction (ELK→Loki or the reverse) tends to run 1-2 weeks of engineering effort for a moderate-sized pipeline, and ELK→Loki migrations commonly report 75-90% storage/cost reduction. Useful as a first-order estimate in a design doc, not a guarantee.

**Label cardinality is the same bomb as in metrics.** In Loki, every unique label-set is a **stream**. Put `trace_id`, `pod_hash`, `request_id`, or any unbounded field in a *label* and you detonate stream count — ingest and query collapse, exactly as unbounded metric labels blow up a TSDB. The discipline is identical: labels are for *bounded, low-cardinality* dimensions you select on; high-cardinality data goes in the **log line**, retrieved with a LogQL line filter (`| json | ...`), never as a label. In ELK the same failure wears a different mask: **dynamic mapping** on unbounded JSON keys causes mapping/field explosion, destabilizing the cluster. Both systems die of cardinality; the symptom differs (stream explosion vs mapping explosion) — but the failure modes differ in *character*, not just name, which matters operationally (see below).

**Loki's stream limit: fail-fast, not silent.** Loki enforces per-tenant/per-ingester stream limits (historically on the order of 5,000-10,000 streams per tenant per ingester, depending on version and config) as a hard wall. When you hit it, writes are rejected outright ("maximum active stream limit exceeded") — a loud, visible failure. Contrast this with Prometheus, where an unbounded-cardinality spike tends to fail catastrophically via ingester OOM rather than a clean rejection. The Loki failure mode is objectively better operationally (you get a clear signal and can roll back the offending label), even though the root cause — cardinality discipline — is identical across both systems.

**Chunk compaction cost cuts both ways.** It's tempting to assume only *high-volume* streams are a memory risk. Not so: Loki's compressed-chunk-per-stream model means a stream with low, sparse write volume still holds an open, uncompacted, memory-resident chunk until the flush interval elapses. A schema with many low-volume streams (e.g. one stream per rarely-active tenant/job combination) can pressure ingester memory just as much as fewer high-volume ones — cardinality is the risk variable, not raw log volume.

**Cost-control levers.**
- **Structure at the source** so downstream doesn't re-parse.
- **Sample** high-volume/low-value logs — keep 100% of ERROR, sample INFO/DEBUG.
- **Drop/redact fields at the collector** before storage (Vector/Fluent Bit transforms) so you never pay to store what you'll never query.
- **Route by value:** audit logs → cold object storage / long retention; debug logs → short retention / hot then expire.
- **Enforce a schema** with a bounded set of indexed fields — this is what prevents dynamic-mapping explosion in ES and stream explosion in Loki. For ELK specifically, disabling dynamic mapping is necessary but not sufficient — you must also define an explicit, bounded mapping schema up front, which requires knowing your field set in advance. Turning dynamic mapping off without a schema behind it just changes the failure from "silent explosion" to "silent field drops."
- **Push line filters early.** In LogQL, a `|= "text"` filter placed before `| json` reduces the volume that hits the expensive parse stage, since filtering happens on raw line bytes before the JSON parser runs.

**GPU tie.** Training and inference logs are firehoses — per-rank, per-step, across thousands of GPUs. Keep `node`, `rank`, `job`, `tenant` as **bounded Loki labels**; push `step`, `loss`, `trace_id` into the **log line** (filter with LogQL), never into labels. NCCL debug logs (`NCCL_DEBUG=INFO`) on a large synchronous job are near-duplicate across every rank simultaneously — every rank logs substantively the same collective event within milliseconds of every other rank. A naive per-rank stream design pays for N× redundant storage of what is, informationally, one event. Sampling by rank is one lever, but the durable fix is extracting straggler/timing signal as a *metric* (see lesson 09's NCCL Inspector/NIXT tooling) so the raw log stream itself can be dropped to near-zero retention rather than merely sampled down. See the separately built GPU-observability artifact (DCGM / util-lie / MFU) for the metric side that should absorb most of what naive pipelines log.

## Perspectives

**Data-model.** Loki's "index only labels" design directly mirrors Prometheus's label model — Loki is best understood as Prometheus's cost model applied to logs, not as a separate system with its own rules to learn from scratch. The same cardinality discipline from lesson 01 (bounded dimensions in the index, everything else in the payload) transfers almost line-for-line. If you've internalized why an unbounded Prometheus label is dangerous, you already understand why an unbounded Loki label is dangerous — recognize the isomorphism rather than re-deriving it.

**Compliance/security.** ELK's inverted index isn't just "faster search" as a generic property — for security and forensics workloads, the actual requirement is often ad hoc querying across fields nobody predicted needing to search: an IOC hash showing up in an unexpected header, an unusual field value that only matters in hindsight. A label-indexed system structurally can't serve that well, no matter how carefully the label schema is tuned, because the whole point is that you didn't know to index it. This is a genuine, confident "pick ELK" case — not a consolation prize for teams that couldn't get Loki's cardinality discipline right.

**Migration-cost.** Treat the Loki-vs-ELK decision as having a roughly quantifiable switching cost, not a purely qualitative one: reported migrations in either direction tend to run 1-2 weeks of engineering effort, and ELK→Loki migrations commonly report 75-90% cost reduction. Citing a rough number in a design doc is more persuasive — and more honest about the tradeoff — than an unqualified "Loki is cheaper."

**Fleet-scale/GPU-specific.** NCCL debug logs at `NCCL_DEBUG=INFO` on a large synchronous job are a distinct failure shape from ordinary high-volume logging: it's not that any one rank logs a lot, it's that every rank logs the *same* event at the *same* moment. A naive per-rank stream design multiplies storage by rank count for what is substantively one collective-operation event. This is a case where deduplication or sampling-by-rank is the right first lever, but the durable fix — treated in lesson 09 — is to stop logging the event at all and emit a metric instead.

## Real-world use cases

- **Cloudflare — "An overview of Cloudflare's logging pipeline."** https://blog.cloudflare.com/an-overview-of-cloudflares-logging-pipeline/ — One of Cloudflare's largest data pipelines, handling millions of log events per second from every edge server. A genuine fleet-scale logging architecture reference, useful for grounding the cost-control levers above in a real production system.
- **Cloudflare — "Adopting OpenTelemetry for our logging pipeline."** https://blog.cloudflare.com/adopting-opentelemetry-for-our-logging-pipeline/ — Shows the collection side moving to OTel while storage/query remains a separate concern. Reinforces the point from lesson 04 that OTel unifies collection, not storage — true for logs specifically, not just metrics and traces.
- **University of Toronto CS (cks) sysadmin blog — "Grafana Loki and what can go wrong with label cardinality."** https://utcc.utoronto.ca/~cks/space/blog/sysadmin/GrafanaLokiCardinalityProblem — A practitioner's first-hand account of hitting Loki's stream-cardinality wall in a real operated deployment. Useful as ground truth for what the failure actually looks like from the operator's chair, not just in the docs.

## Worked example

A LogQL query and label schema for a 4k-GPU training fleet.

**Correct — bounded labels, high-cardinality in the line, filter pushed early:**
```logql
{job="llama-70b-pretrain", tenant="research", rank=~"1[0-9]"}
  |= "loss"
  | json
  | loss > 5.0
```
Labels: `{job, node, rank, tenant}`. Stream count ≈ jobs × nodes × ranks × tenants — bounded and knowable (e.g. 1 job × 500 nodes × 8 ranks × 1 tenant ≈ 4,000 streams). `step`, `loss`, `trace_id` live in the JSON body, filtered at query time. The `|= "loss"` line filter runs before `| json`, so lines that can't possibly match are discarded on raw bytes before paying the JSON-parse cost — the cheap-then-expensive ordering that LogQL's two-stage cost model rewards.

**Cardinality bomb — `trace_id` promoted to a label:**
```logql
{job="llama-70b-pretrain", trace_id="a1b2c3..."}   # WRONG
```
Now stream count ≈ 4,000 × (unique trace_ids) → effectively unbounded; every distinct request mints a new stream. Ingest backs up against Loki's per-tenant stream limit (commonly cited in the 5,000-10,000/ingester range) and starts rejecting writes outright — a loud failure, at least, rather than a silent one — while queries against the ballooning index time out. **Compute both ways:** bounded schema = ~4k streams; add `trace_id` at ~10⁶ traces/day and you're at millions of streams/day — the same detonation as an unbounded metric label. The fix is one character of judgment: `trace_id` is a line field, not a label.

**NCCL duplication, sized.** A 500-node, 8-rank job running `NCCL_DEBUG=INFO` logs each collective (allreduce, broadcast) once per rank — 4,000 near-identical log lines per collective event, differing mainly in rank ID and a timestamp within milliseconds. At one collective per training step and a step every few seconds, that's tens of thousands of near-duplicate lines per minute for what is one event from an operational standpoint. Sampling by rank (e.g. keep rank 0 plus stragglers) cuts this but still stores an arbitrary sample; extracting straggler/timing signal as a metric (lesson 09) lets you drop the raw NCCL log stream to near-zero retention entirely.

## Practice

Design the logging layer of the fleet observability system. Specify: (a) the Loki-vs-ELK choice with the workload justification, including the migration-cost heuristic if a switch is proposed; (b) the Loki label schema for a multi-tenant GPU fleet and the stream-count math (bounded vs the `trace_id`-as-label bomb); (c) the collector-side cost levers (sampling rules per level, field drops/redaction, value-based routing to hot/cold retention, early line-filter placement in LogQL queries you'd standardize); (d) which currently-logged signals (e.g. NCCL straggler timing) you'd promote to metrics instead, and why sampling alone isn't the durable fix for the NCCL duplication case; (e) how you'd size and alert on Loki's per-tenant stream limit before it silently starts rejecting writes. State the failure mode each choice defends against. <feeds [fleet observability design](../practice/fleet-observability/README.md)>

## Common pitfalls

- **"Structured logging solves the label-cardinality problem."** It doesn't — JSON logging makes fields available for filtering, but says nothing about which fields are *promoted to indexed labels* versus left in the body. Cardinality discipline is a separate decision layered on top of structuring; structuring alone is necessary but not sufficient.
- **"Loki has no full-text search, so it can't do ad hoc queries at all."** It can — `|=`/`|~` line filters do brute-force scans over label-selected chunks. It's slow for wide-open queries across many streams, not impossible; the real constraint is that it doesn't have an *index* over line content, so the scan cost is linear in the selected chunk volume.
- **"NCCL debug logs are a logging problem; just sample them."** Sampling reduces volume but keeps you sampling an event that's fundamentally duplicated across every rank. The durable fix is extracting straggler/timing signal as a metric (lesson 09's NCCL Inspector/NIXT tooling) so the log stream itself can be dropped to near-zero retention, rather than merely thinned.
- **"ELK's dynamic mapping is just a config default you can turn off."** Disabling it is necessary but not sufficient — you must also define an explicit, bounded mapping schema up front, which requires knowing your field set in advance. Without that schema behind it, you've just moved the failure from "silent field explosion" to "silent field drops for anything not pre-declared."
- **"A low-volume stream can't be a memory problem."** It can — Loki holds a stream's uncompacted chunk in ingester memory until the flush interval regardless of write rate, so many low-volume, high-cardinality streams pressure memory just as much as fewer high-volume ones. Cardinality, not raw log rate, is the risk variable.

## Self-check

- Why is `trace_id` catastrophic as a Loki label but fine in the log line? **Answer:** Every unique label-set is a Loki stream, so promoting an unbounded field like `trace_id` to a label makes stream count effectively unbounded — one new stream per request — which collapses ingest and query, the same bomb as unbounded metric labels. In the log line it's just data, retrieved by a LogQL line filter (`| json | ...`) after label selection, with zero cardinality cost.
- When do you actually choose ELK over Loki despite the cost? **Answer:** When investigative full-text search is the *primary* workload — you need fast arbitrary queries over fields nobody predicted needing to search (security forensics, compliance discovery), not label-scoped tail-and-filter. Loki has no inverted index and scans chunks brute-force after label selection, so wide-open full-text search is slow; ELK's inverted index is what you're paying the higher storage and ops cost for. This is a genuine, confidently-defensible choice, not a fallback.
- Same cardinality failure, two masks — name each, and which fails more gracefully? **Answer:** In Loki it's **stream explosion**: unbounded label values multiply stream count until a per-tenant/per-ingester limit rejects writes outright — a loud, fail-fast failure. In Elasticsearch it's **mapping/field explosion**: dynamic mapping over unbounded JSON keys creates unbounded indexed fields, destabilizing the cluster — closer to Prometheus's OOM-style fail-catastrophic mode. Same root cause (unbounded cardinality in an indexed dimension), but Loki's failure is operationally cleaner to detect and roll back.
- Why can't sampling alone fix the NCCL debug-log duplication problem? **Answer:** Every rank in a synchronous collective logs substantively the same event within milliseconds of every other rank — the redundancy is structural, not a volume problem sampling addresses. Sampling by rank reduces stored volume but still stores an arbitrary slice of a duplicated signal; the durable fix is extracting straggler/timing information as a metric so the raw log stream can be dropped to near-zero retention instead of merely thinned.
- Why does pushing a LogQL line filter (`|=`) before `| json` matter for query cost? **Answer:** LogQL has a two-stage cost model: chunks are selected cheaply by label matchers, then content within those chunks is scanned — regex/line filters run on raw bytes, but `| json` parsing is comparatively expensive. Placing `|=` before `| json` discards non-matching lines on the cheap raw-byte pass, so fewer lines reach the expensive parse stage.

## Connections & what's next

This lesson carries the sampling-economics lens from lesson 05's tracing tail-sampling over to logs, and its cardinality discipline is a direct restatement of lesson 01's signal-model constraint applied to a new indexing structure. The NCCL-duplication and metric-extraction points here set up lesson 09's fleet-scale GPU/ML synthesis. Next: [07 · SLOs and alerting](07-slos-and-alerting.md), which builds multi-window multi-burn-rate alerting from first principles — the layer that decides which of these signals (metrics, exemplar-linked traces, or last-resort logs) actually pages someone.

## References & further reading

**Primary sources**
- Grafana Loki, labels documentation — https://grafana.com/docs/loki/latest/get-started/labels/
- Grafana Loki, architecture documentation — https://grafana.com/docs/loki/latest/get-started/architecture/
- Grafana Loki, LogQL log queries documentation — https://grafana.com/docs/loki/latest/query/log_queries/

**Real-world engineering blogs**
- Cloudflare, "An overview of Cloudflare's logging pipeline" — https://blog.cloudflare.com/an-overview-of-cloudflares-logging-pipeline/
- Cloudflare, "Adopting OpenTelemetry for our logging pipeline" — https://blog.cloudflare.com/adopting-opentelemetry-for-our-logging-pipeline/
- University of Toronto CS (cks) sysadmin blog, "Grafana Loki and what can go wrong with label cardinality" — https://utcc.utoronto.ca/~cks/space/blog/sysadmin/GrafanaLokiCardinalityProblem

**Deeper dives**
- SigNoz, "Loki vs Elasticsearch" comparison — https://signoz.io/blog/loki-vs-elasticsearch/
