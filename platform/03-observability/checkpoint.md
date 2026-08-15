# 🔭 Checkpoint — Observability engineering

The **completion gate**. Prove it with the [fleet observability design](practice/fleet-observability/)
and answer cold. You've passed when you can, **unaided**:

## Pass criteria

- [ ] **1 · Fix three broken PromQL panels** and explain *why each is wrong*: `rate()` on a gauge
      (DCGM_FI_DEV_GPU_UTIL), `avg(p99)` across instances, and a `histogram_quantile` whose p99
      lands in a `[…,+Inf]` bucket.
- [ ] **2 · Derive an MWMBR alert set** from first principles — the two-window AND, the tiers
      (14.4×/1h+5m, 6×/6h+30m, 1×/3d+6h), and the multiplier math
      `budget_fraction = burn_rate × (alert_window / SLO_window)`.
- [ ] **3 · Defend a signal-choice tradeoff** out loud — "why is `gpu_uuid`/`request_id` a span
      attribute or exemplar, not a metric label" — with the cardinality cost.
- [ ] **4 · Size a fleet metrics system** — for 4,000 GPU nodes, name the fall-over mode
      (head-block cardinality → RAM → OOM → slow WAL replay), the two alerts that catch it, and
      the scrape-shard + Mimir/Thanos + downsampling design.
- [ ] **5 · Alert on goodput, not utilisation** — define a GPU goodput-regression SLO
      (achieved/expected tokens-per-sec), its wasted-GPU-hours error budget, and the straggler
      query (per-rank step-time `> 1.3 × median`).
- [ ] **6 · Make tracing pay off** — head vs tail sampling, why all spans of a trace must reach one
      collector (the `loadbalancing` exporter), and the exemplar link from a latency panel to a trace.
- [ ] **7 · Serve the question the hot path can't** — explain why `$/useful-GPU-hour by team, last
      quarter` is unanswerable in PromQL (cardinality *and* the missing join to a time-versioned
      rate card), design the two-path tee, and name where a 200-node fleet should **stop** on the
      escalation ladder (Prometheus → +downsampling → one ClickHouse → lakehouse).

## Depth probes (answer cold)

- [ ] Why does `rate()` on a gauge produce garbage — what does `rate` assume?
- [ ] Why is `avg(p99)` meaningless, and what's the correct fleet-percentile query?
- [ ] What actually kills a Prometheus at scale — disk or RAM? Name the metric.
- [ ] Thanos vs Mimir — the architectural difference and when each wins.
- [ ] Why must `tail_sampling` run on the gateway tier, not the agent DaemonSet?
- [ ] Loki label cardinality vs a metric label cardinality — same bomb or different?
- [ ] On-CPU vs off-CPU profiling — which reveals a training thread stuck on `cudaStreamSynchronize`?
- [ ] Which DCGM labels are safe as metric labels at 10k GPUs, and which must become exemplars?

## Interview-readiness proxy

- [ ] You have a fleet observability design doc with a real cardinality budget + relabel configs.
- [ ] You have a deployable burn-rate alert set for both a service SLO and a GPU goodput SLO.
- [ ] You can whiteboard "design observability for a 4,000-GPU fleet" end to end.

## Fail signal

- [ ] Quotes utilisation as health · `rate()`s a gauge · `avg`s percentiles · puts high-cardinality
      IDs in metric labels · treats tracing as "turn it on and store everything."

## Answers / notes

_Record answers as you close each lesson; link the fleet-observability design for items 1–6._
