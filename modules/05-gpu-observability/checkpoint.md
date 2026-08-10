# 📊 Checkpoint — 05 · GPU observability and telemetry

The **completion gate**. Prove it with ["Your GPU dashboard is lying to you"](practice/gpu-dashboard-lie/)
built from your own cluster, and answer the probes cold. You've passed when you can, **unaided**:

## Pass criteria

- [ ] **1 · One sentence, no "utilisation".** State what `DCGM_FI_DEV_GPU_UTIL` measures —
      accept: *"the fraction of the sample window during which at least one kernel was
      resident on the GPU."*
- [ ] **2 · Alert on the right metric.** Given an idle-cost goal, choose `SM_ACTIVE` (idle)
      vs `TENSOR_ACTIVE`/`DRAM_ACTIVE` (efficiency) and justify **never** alerting on `GPU_UTIL`.
- [ ] **3 · XIDs.** Name 5+ and correctly split **cordon** (48/63/64/79/94/95) from
      **log / user-bug** (13/31/43).
- [ ] **4 · TTFT vs TPOT.** Define both, say which continuous batching improves vs degrades,
      and why **total request-latency p99 is the wrong streaming SLO**.
- [ ] **5 · The query.** Write the **allocated-but-idle-beyond-N-minutes** PromQL from memory,
      using the module-04 pod-resources join.
- [ ] **6 · The CFO test.** Explain the utilisation trap and the allocated-vs-utilised gap
      **with a dollar figure** in 2 minutes to a non-engineer — state the ~15%-utilised
      industry baseline and your $/GPU-hr rate as **dated, directional 2026 snapshots**,
      not universal constants.
- [ ] **7 · Artifact.** The per-namespace gap dashboard renders, and the
      `GPU_UTIL=100%` / `SM_ACTIVE≈0.1` exhibit exists from your own cluster.

## Depth probes (answer cold)

- [ ] Why does a batch-1 LLM decode read 100% `GPU_UTIL`?
- [ ] `SM_ACTIVE=0.9` with `TENSOR_ACTIVE=0.1` — what's the workload doing wrong?
- [ ] Why can't you sample every `PROF_*` field at 1s on one GPU? (multiplexing)
- [ ] Why is `SM_ACTIVE` missing from the default DCGM Grafana dashboard?
- [ ] Does a custom dcgm-exporter counters CSV extend or replace the default set?
- [ ] Why can't you attribute per-pod under time-slicing, in metric terms?
- [ ] XID 48 then XID 63 — what do you do?
- [ ] Rising queue-depth while TPOT stays flat — what does it tell you?
- [ ] `SM_ACTIVE` high, `TENSOR_ACTIVE` low — which profiling tool next?

## Interview-readiness proxy

- [ ] You hold the `GPU_UTIL`=100% / `SM_ACTIVE`≈0 screenshot from your own cluster.
- [ ] You have a dashboard showing allocated-vs-utilised GPU-hours per namespace, gap in dollars.

## Answers / notes

_Record answers as you close each lesson; link the dashboard + exhibit + query pack for items 1–7._
