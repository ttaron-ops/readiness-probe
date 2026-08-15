# "Your GPU utilisation dashboard is lying to you" — Module 05 deliverable

The **flagship public artifact** — a dashboard + exhibit + PromQL pack built from *your
own* cluster. It's simultaneously the interview answer, the blog post, the input dataset
for Module 11, and the core query of your `gpu-cost-operator`.

> One GPU node, a weekend: GPU Operator + dcgm-exporter (with `SM_ACTIVE` enabled) +
> Prometheus/Grafana + vLLM. The argument is per-GPU — no multi-node needed.

> **Build it first without hardware.** The
> [fake GPU fleet lab](../../../04-gpu-on-kubernetes/practice/fake-gpu-fleet/README.md) gives you
> a synthetic DCGM exporter emitting the three workload shapes this dashboard must tell apart
> (training / inference / idle-but-allocated), plus injectable XIDs and stragglers. Develop every
> panel and PromQL query against it, then spend the rented-GPU weekend **validating** rather than
> debugging.

## Three components

### 1. Allocated-vs-utilised GPU-hours dashboard (per namespace) — the CFO panel
One panel, three series:
- **allocated GPU-hours** — from the pod-resources GPU count (Module 04's join),
- **SM-active GPU-hours** — `DCGM_FI_PROF_SM_ACTIVE`-weighted busy time,
- **the gap, in dollars** — `gap_gpu_hours × hourly_rate`.

If any GPU in your cluster is time-sliced or MPS-shared, add an explicit
**`unattributable_gpu_hours`** bucket (lesson 05.4) rather than silently crediting the
SM-active time to whichever pod's label happened to land last — device-level DCGM fields
can't distinguish co-tenants under those sharing modes.

### 2. The util-lie exhibit (screenshot)
`DCGM_FI_DEV_GPU_UTIL = 100%` beside `DCGM_FI_PROF_SM_ACTIVE ≈ 0.1` for the **same
GPU/pod**, captured from a **batch-1 decode workload you run yourself**. This single image
is the interview story and the blog's hero.

### 3. The PromQL query pack

```promql
# allocated-but-idle beyond 15 min (the headline query)
#   a GPU with a pod-resources allocation whose SM_ACTIVE has been ~0
avg_over_time(DCGM_FI_PROF_SM_ACTIVE[15m]) < 0.05
  and on(gpu, UUID) (DCGM_FI_DEV_FB_USED >= 0)   # scoped to allocated GPUs via the 04 join

# the util-lie detector
DCGM_FI_DEV_GPU_UTIL > 90 and DCGM_FI_PROF_SM_ACTIVE < 0.2

# per-namespace wasted GPU-hours (allocated − SM-active)
#   the divergence: rank namespaces by allocated GPUs, then by SM-active GPU-hours
```

(Finalize the exact label joins against your Module-04 exporter's `exported_pod` /
`namespace` labels.)

## The $ framing (for the blog + the CFO test)

- Industry avg GPU utilisation ~15% ⇒ ~85% of paid capacity idle — **a dated, directional
  2026 snapshot**, not a precise universal constant; state it as such.
- At $2–3/hr/H100 (also a 2026 snapshot — verify current pricing), a **10%
  fleet-utilisation improvement on 500 GPUs ≈ $0.9–1.3M/yr**.
- You are billed on **allocated** GPU-hours, not SM-active-hours — the gap is pure waste.

## Acceptance criteria (matches the [checkpoint](../../checkpoint.md))

- [ ] per-namespace allocated-vs-utilised dashboard renders, with the gap **in dollars**
- [ ] the `GPU_UTIL=100%` / `SM_ACTIVE≈0.1` exhibit screenshot exists from your own cluster
- [ ] the PromQL pack (allocated-but-idle, util-lie detector, per-namespace waste, divergence) works against your data
- [ ] a written blog draft ties the exhibit + query + $ number into the "your dashboard is lying" argument

## Guardrails

- Publishable-by-default — this is the sibling of Module 03's cost report; scrub real
  cluster/namespace/hostnames before posting.
- No cluster credentials or real cost rates committed (repo `.gitignore` guards these).
- Keep the query pack importable — it becomes the `gpu-cost-operator`'s core logic.
