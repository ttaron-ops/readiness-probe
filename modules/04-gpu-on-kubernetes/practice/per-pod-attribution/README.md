# Per-pod GPU attribution — Module 04 deliverable

Extend the [`gpu-cost-operator`](../../../02-kubernetes-controllers/practice/gpu-cost-operator/)
(or a companion exporter) to do **real per-pod GPU attribution** on a live single-node
GPU cluster — the piece that makes your cost operator actually measure GPU spend. Built
across this module's lessons on a rented GPU.

> This mirrors exactly how **dcgm-exporter** joins the kubelet pod-resources API to GPU
> metrics — you're building the same UUID→pod bridge, then adding the cost layer.

## What it does

1. **The mapper** (from L3) — a Go client to the kubelet **pod-resources** socket
   (`/var/lib/kubelet/pod-resources/kubelet.sock`, `List`) that builds a live
   **device-UUID → pod → namespace** map and exposes it as Prometheus labels.
2. **The join** — join that map to **DCGM utilisation** (or `nvidia-smi`/NVML for a
   minimal version) to emit a per-pod GPU-cost/efficiency metric:
   `per_pod_gpu_cost = utilisation × node_hourly_rate ÷ device_count`.
3. **The two hard cases** (the whole point):
   - **MIG device** — distinct MIG-UUID per slice → **clean 1:1 attribution**.
   - **Time-sliced device** — all replicas share **one physical UUID** (many pods : 1
     device) → allocation-based billing is **impossible**; fall back to **per-PID
     utilisation** from DCGM. Document why.
4. **Freshness** — keep the map current as pods churn (watch/informer over poll).

## Plus: the GPU Operator failure-mode log

A written `failure-mode-log.md` — the "interview gold" — assembled from the break/fix
work in L2 (crash-loops), L5 (driver upgrade), and L6 (MIG reconfigure). For each:
**symptom → `kubectl`/log evidence → root cause → fix → prevention.**

## Suggested layout

```
per-pod-attribution/
├── cmd/gpu-attributor/main.go   # pod-resources client + metrics server
├── internal/
│   ├── podresources/            # kubelet socket client (List)
│   ├── attribute/               # UUID→pod→ns map + MIG vs time-sliced handling
│   └── metrics/                 # per-pod GPU-cost GaugeVec
├── failure-mode-log.md          # the break/fix log (L2/L5/L6)
└── README.md                    # run instructions + the time-sliced caveat
```

## Acceptance criteria (matches the [checkpoint](../../checkpoint.md))

- [ ] `List`s the pod-resources socket and emits a live UUID→pod→namespace mapping as Prometheus labels
- [ ] joins to DCGM (or NVML) and emits a per-pod GPU-cost/efficiency metric
- [ ] handles a **MIG** device (clean 1:1) and a **time-sliced** device (shared UUID → per-PID fallback), with the caveat documented
- [ ] the map stays fresh as pods are created/deleted
- [ ] `failure-mode-log.md` has ≥5 real break/fix entries (symptom → evidence → root cause → fix → prevention)

## Guardrails

- Runs on one rented GPU; the pod-resources socket needs a hostPath mount + the right RBAC.
- No cluster credentials, kubeconfigs, or real cost rates committed (repo `.gitignore` guards these).
- The mapper core drops back into the capstone `gpu-cost-operator` — keep it importable.
