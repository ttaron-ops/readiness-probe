# 🗺️ Roadmap — 12–15 months to Senior GPU-Platform Engineer

**Target:** Senior Platform Engineer where GPU fleet operations are mandatory.
**Budget:** ~10–12 focused hrs/week. **Target interview window:** months 10–12.

Three tracks run in parallel (see [README](../README.md)):
**A — Platform excellence**, **B — GPU specialization**, **C — Evidence & interview**.
The sequencing rule: **Go and the two long-runway platform modules (distributed
systems, networking) go early; the flagship controller starts the moment Go is
solid and then runs continuously; interviewing ramps in the back third.**

Rough hour split across the whole plan: **~55% Track B, ~25% Track A, ~20% Track C**
— but Track C's hours are mostly *the same hours* as B, because you build the
controller out of what you're learning.

---

## Phase 0 · Foundations — Months 1–3
*The make-or-break stretch. Go is the gate; nothing downstream ships without it.*

- **B:** [01 Go](../platform-eng/modules/01-go-for-infra/README.md) to real competence · [01b Linux internals](../platform-eng/modules/01b-linux-internals/README.md) in parallel · begin [02 K8s controllers](../platform-eng/modules/02-kubernetes-controllers/README.md) once Go basics land.
- **A:** begin [A1 Distributed systems & design](../platform-eng/platform/01-distributed-systems-and-design/README.md) — long-runway, pairs well with light-focus days.
- **C:** no code yet. **Have the open-source conversation with your manager now** — agreeing a licence before writing significant capstone code is far easier than retrofitting one.

**Milestone M3:** you can write a basic reconcile loop and read Kubernetes source without flinching · DDIA-level distributed-systems fluency building · Go checkpoint passed.

---

## Phase 1 · Controller + GPU base — Months 3–6

- **B:** finish [02 controllers](../platform-eng/modules/02-kubernetes-controllers/README.md) (operator with finalizers, status, envtest) · [02b Host topology](../platform-eng/modules/02b-host-topology/README.md) · [03 GPU hardware](../platform-eng/modules/03-gpu-hardware/README.md) · start [04 GPU on Kubernetes](../platform-eng/modules/04-gpu-on-kubernetes/README.md).
- **A:** [A2 Platform networking](../platform-eng/platform/02-platform-networking/README.md).
- **C:** **Capstone Stage 1 — Metrics MVP** (~month 4, once Go is comfortable): watch pods requesting `nvidia.com/gpu`, join DCGM + node instance types, compute allocated-vs-utilised GPU-hours, export Prometheus metrics. Draft blog post #1 ("your GPU dashboard is lying to you") from the utilisation-trap screenshot.

**Milestone M6:** GPU-on-k8s operational on a cluster you built · Metrics MVP producing allocated-vs-utilised numbers in Skyro · post #1 drafted.

---

## Phase 2 · GPU depth + evidence — Months 6–9

- **B:** [05 GPU observability](../platform-eng/modules/05-gpu-observability/README.md) · [06 Scheduling & capacity](../platform-eng/modules/06-scheduling-capacity/README.md) (Kueue) · [07 Inference serving](../platform-eng/modules/07-inference-serving/README.md) (vLLM).
- **A:** [A3 Observability engineering](../platform-eng/platform/03-observability/README.md) — deliberately paired with B05; lesson A3.9 bridges straight into GPU/ML telemetry.
- **C:** **Capstone Stage 2 — CRDs & reconcile loop** (`GPUCostPolicy`, `WorkloadCost`, `Budget`) · **inference benchmark weekend** (cost-per-million-tokens curve, FP8 saving) · **publish posts #1 and #2**.

**Milestone M9:** controller has CRDs and runs in Skyro production · 2 posts published · you can hold an inference-economics conversation cold.

---

## Phase 3 · Cost differentiator + interview ramp — Months 9–12

- **B:** [11 GPU cost & unit economics](../platform-eng/modules/11-gpu-cost-economics/README.md) — formalise your differentiator.
- **A:** run the [interview-refresh](../platform-eng/platform/interview-refresh/README.md) pages (IaC, CI/CD, cloud, security, SRE, k8s ops) · [A1 design rehearsal](../platform-eng/platform/01-distributed-systems-and-design/lessons/09-design-rehearsal.md) timed reps.
- **C:** **Capstone Stage 3 — fractional attribution** (MIG + time-sliced from SM occupancy) · **2 mock system-design interviews** with a working engineer · **behavioral / staff-signal stories** written · **post #3**.
- **Start applying and interviewing around month 10–11 — do not wait until "finished".**

**Milestone M12:** readiness gate mostly green · actively interviewing.

---

## Phase 4 · Close — Months 12–15
*Buffer. Convert momentum into an offer.*

- **Evidence:** get the capstone **adopted by ≥1 org other than Skyro** — adoption outweighs stars.
- **B stretch (only as a target employer demands):** [08 Distributed training](../platform-eng/modules/08-distributed-training/README.md) · [09 Networking & topology](../platform-eng/modules/09-networking-topology/README.md) · [10 Bare metal & lifecycle](../platform-eng/modules/10-bare-metal-lifecycle/README.md).
- Continue interviewing to offer; use the buffer for slipped modules.

---

## Readiness gate

Don't start interviewing seriously until most of these are true; don't wait for
*all* of them before *applying*.

- [ ] **Go / controllers:** you can scaffold and ship an operator (finalizers, status, envtest) unaided.
- [ ] **GPU on k8s:** you can install, debug and upgrade the GPU Operator on a cluster you built.
- [ ] **Observability:** a dashboard showing allocated-vs-utilised GPU-hours per namespace, plus the `GPU_UTIL` 100% / `SM_ACTIVE` ~0% screenshot from your own cluster.
- [ ] **Cost:** you can produce a cost-per-million-tokens figure from raw metrics for a workload you operate.
- [ ] **Platform depth:** distributed-systems and networking checkpoints passed; sharp areas refreshed.
- [ ] **System design:** ≥2 mock interviews done; you can design a multi-tenant GPU platform end to end.
- [ ] **Behavioral:** 4 staff-signal stories ready (unpopular decision, cross-team adoption, being wrong, driving a process).
- [ ] **Evidence public:** controller open-sourced and run by ≥1 external org; 2–3 posts with real readership.

## What determines whether you hit 12 or 15 months

1. **Consistency of the 10–12 hrs.** This is the single biggest variable, and it's a calendar problem, not an ability one.
2. **Building, not reading.** Every module ends in an artifact — ideally one that also advances the capstone.
3. **Go not dragging.** Protect Phase 0; don't start module 04 while shaky on reconcile loops.
