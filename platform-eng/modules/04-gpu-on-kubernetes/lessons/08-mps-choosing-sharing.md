---
lesson: "04.8"
title: "MPS and choosing a sharing mode"
module: "04"
concept: "MPS and choosing a sharing mode"
status: not-started
est_time: "7h"
prev: "07-time-slicing-attribution.md"
next: "09-dra-driver-and-quotas.md"
artifacts: []
sources: 5
---

# 04.8 · MPS and choosing a sharing mode

> **Concept.** MPS shares a GPU by running clients concurrently on the SMs (spatial sharing, one shared CUDA context, one shared fault domain) — the third point in the MIG / time-slicing / MPS decision triangle, and it is marked *experimental* in NVIDIA's own device plugin as of this writing, which is itself part of the senior answer about when to use it.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Lesson 06 gave you MIG: hardware walls, distinct UUIDs, clean attribution. Lesson 07 gave you time-slicing: no walls at all, shared UUID, attribution that only works via a DCGM fallback. This lesson completes the triangle with MPS (Multi-Process Service) — a third mechanism that shares the *same* attribution problem as time-slicing (no distinct UUID, same DCGM fallback) but a *different* performance and risk profile (concurrent execution instead of turns, and a worse fault domain than either of the other two). By the end of this lesson you'll have the full three-way decision table the module README names as an explicit interview probe — "MIG vs time-slicing vs MPS — mechanism/isolation/when" — answerable cold.

## Why this matters

You now have two of the three sharing modes in hand: MIG (walls in silicon, distinct UUIDs, real isolation) and time-slicing (turns, no isolation, collapsed attribution). MPS is the third, and it's the one most engineers can't place correctly — they either don't know it exists, or they know the name but can't say precisely how it differs mechanistically from time-slicing. It sits *between* the other two on some axes and *outside* both on others, and picking it — or refusing it — is a one-question sorting exercise a senior platform engineer must answer without hesitation in a design review.

The reason it matters for cost/efficiency work specifically: MPS is the mode that actually *raises aggregate throughput* on small concurrent jobs, because it lets kernels from different processes run at the same time on different SMs instead of waiting their turn. A cost-per-useful-work story that only knows time-slicing leaves real efficiency on the table on inference fleets running many small, under-filling models. But MPS also carries the sharpest edge of the three modes — one fatal CUDA error takes down the shared MPS server and **every** client with it — and, as of this writing, NVIDIA's own device plugin still labels MPS sharing **experimental**. Knowing that caveat, and being able to answer "is MPS production-ready?" with genuine nuance rather than a flat yes or no, is itself part of the interview-grade answer here. Owning the clean three-way mental model — including its honest limits — is what turns "we share GPUs" into a defensible engineering policy instead of a slogan.

## What's new here (calibration)

This module's README calibration (skip device-plugin gRPC mechanics, the DRA object model, Topology Manager internals, silicon-level MIG — all owned by modules 02/02b/03) applies here unchanged. Lessons 06 and 07 already built the "sharing strategies share the `nvidia.com/gpu` resource name but differ underneath" frame and the DCGM-fallback attribution path. This lesson does not re-derive either. What it adds:

- **The mechanistic contrast with time-slicing** — concurrent SM occupancy via one shared CUDA context, versus time-slicing's whole-device context switch — stated precisely enough to survive a "walk me through the difference" interview question.
- **The confirmed maturity status** — MPS sharing has been marked experimental in NVIDIA's device plugin since v0.15.0, and is explicitly unsupported on MIG-enabled devices — a fact that tempers this lesson's throughput story with a real production-readiness caveat.
- **The completed three-way decision table**, now that all three modes (MIG, time-slicing, MPS) have been covered, as the single artifact this whole three-lesson arc has been building toward.
- **A real production adopter's motivation for choosing MPS** — Databricks's account of scaling small-LLM inference, which grounds "jobs that under-fill the GPU" in an actual number (a 15 GB model on an 80 GB card).

## Core concepts

### 1 — Enabling MPS via the device plugin

MPS is configured through the same sharing block as time-slicing, selecting the `mps` strategy. With the GPU Operator:

```yaml
# mps-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mps-config
  namespace: gpu-operator
data:
  any: |-
    version: v1
    flags:
      migStrategy: none
    sharing:
      mps:
        resources:
          - name: nvidia.com/gpu
            replicas: 4          # up to 4 concurrent MPS clients per GPU
```

```bash
kubectl create -f mps-config.yaml
kubectl patch clusterpolicy/cluster-policy -n gpu-operator --type merge \
  -p '{"spec":{"devicePlugin":{"config":{"name":"mps-config","default":"any"}}}}'
```

The operator stands up the MPS control daemon per GPU (as a managed `nvidia-cuda-mps-control -d` process) and steers client CUDA processes at it via `CUDA_MPS_PIPE_DIRECTORY`. Confirm the node now advertises MPS sharing:

```bash
$ kubectl get node <node> -o json \
    | jq '.metadata.labels["nvidia.com/gpu.sharing-strategy"]'
"mps"

$ kubectl describe node <node> | grep nvidia.com/gpu:
  nvidia.com/gpu:  4        # 4 concurrent MPS clients over 1 physical GPU
```

Like time-slicing, allocation is still uniform — MPS **does not** give each client a distinct UUID (only MIG does). So the attribution hole from lesson 07 applies identically: **you still cannot bill from allocation counts; you fall back to DCGM per-pod utilization.** MPS changes throughput and isolation, not the attribution story.

**Important, directly confirmed against the plugin's own source** ([NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin), v0.17.1 README): MPS sharing has been marked **"experimental"** since device-plugin **v0.15.0**, and is **explicitly documented as not supported on devices with MIG enabled**. Both facts matter operationally: (1) "experimental" from the vendor that owns the component is not a formality — treat MPS as a mode you adopt with eyes open on a fleet where a regression is tolerable, not as a default; (2) the MIG-incompatibility is absolute at the device level — you pick MIG *or* MPS for a given physical GPU, never both simultaneously, so a fleet using MIG for isolated tenants and MPS for a shared pool needs that split enforced at the node-pool level, not the GPU level.

### 2 — Why MPS raises throughput where time-slicing does not

Consider four inference pods each running a small model whose kernels occupy ~25% of the SMs. Under **time-slicing** they run one at a time — the GPU is ~25% busy in each slice, and aggregate throughput is roughly one job's worth plus context-switch overhead. Under **MPS**, a single **control daemon** (the MPS server) funnels all clients' work into **one shared CUDA context**, so kernels from different processes execute *spatially concurrently* across the SM array — the four kernels dispatch together and pack onto the SMs concurrently, and the GPU approaches ~100% occupancy with aggregate throughput approaching ~4× the single-job rate (until you hit an SM, memory-bandwidth, or framebuffer ceiling).

The rule of thumb: **MPS wins exactly when individual jobs under-fill the GPU and run concurrently.** If each job already saturates the SMs, MPS buys nothing over time-slicing (they now contend for the same busy SMs) and you've taken on the shared-fault-domain risk for no throughput gain.

MPS can also cap each client's SM fraction via `CUDA_MPS_ACTIVE_THREAD_PERCENTAGE` (Volta+) — a soft compute-provisioning knob time-slicing simply doesn't have, since time-slicing has no notion of partial occupancy, only turns. Memory, however, is still **not** hard-partitioned under MPS: a client can request a cap via `CUDA_MPS_PINNED_DEVICE_MEM_LIMIT`, but this is a cooperative limit the client can be built to respect, not a hardware wall the GPU enforces — mechanistically no different from time-slicing's uncapped framebuffer sharing.

### 3 — The fatal-error blast radius — the disqualifier

Because every client shares one CUDA context under the MPS server, an unrecoverable fault in *any* client — illegal memory access, an uncorrectable ECC event, a hang — can corrupt the shared context and bring down the **MPS control daemon and all connected clients simultaneously**. There is no per-client fault boundary. (Volta+ MPS improved error containment for some client-side faults, but a fatal fault on the device remains fleet-fatal for that GPU's MPS clients.) This is a **broader** blast radius than time-slicing's, not a narrower one — time-slicing's fault propagation is bad (a neighbor's crash can still degrade the shared device), but MPS adds a second, architectural failure point: the shared server process itself, which doesn't exist under time-slicing at all.

This is why MPS is for **trusted, cooperative** workloads only — your own services sharing a box — and never for hostile or arms-length multi-tenancy, where one tenant's bug becomes every tenant's outage.

### 4 — Is MPS production-ready? The honest, senior answer

This is worth stating as its own point because it's a real interview question, and the honest answer is more nuanced than either extreme. MPS is not vaporware — NVIDIA ships it, the device plugin wires it up cleanly, and (per §5 below) at least one large production AI platform has published a account of running it in anger. But it carries two concrete, vendor-confirmed caveats that a "yes, it's production ready" answer glosses over:

1. It is labeled **experimental** in the device plugin as of v0.15.0 — not deprecated, not beta-with-a-timeline, just experimental, with the maturity implications that label carries from any vendor: expect rougher edges, less operational tooling maturity, and a higher bar for your own validation before trusting it for anything with an SLO.
2. It is **incompatible with MIG on the same device** — so it cannot be your only sharing strategy on a fleet that also needs MIG's hard isolation for some tenants; the two are mutually exclusive per-GPU, and your platform needs to route workloads to the right node pool accordingly.

The defensible interview answer: "MPS is real, it's used in production by at least one major AI platform for exactly the throughput reason this lesson describes, but it's still labeled experimental by NVIDIA and can't coexist with MIG on the same device — so I'd deploy it deliberately, on a node pool carved out for trusted concurrent workloads, with the fault-domain risk explicitly signed off by whoever owns those tenants' SLOs, not as a blanket default."

### 5 — The three-way decision table (the consolidation)

| Axis | **Time-slicing** | **MPS** | **MIG** |
|------|------------------|---------|---------|
| Mechanism | Share by **taking turns** — GPU context-switches between clients | Share by **running concurrently on the SMs** — one shared context via the MPS server | Share by **building walls in silicon** — fixed HW partitions |
| Compute isolation | None (whole GPU per turn) | Soft — optional SM % cap (`ACTIVE_THREAD_PERCENTAGE`) | Hard — dedicated SMs per instance |
| Memory isolation | **None** — full FB, uncapped | Soft/cooperative cap only, shared FB | **Hard** — dedicated framebuffer slice |
| Fault isolation | None — fault hits all co-residents | **Worst** — one fatal error kills MPS server + all clients | **Hard** — fault contained to the instance |
| Distinct GPU UUID? | **No** (same physical UUID) | **No** (same physical UUID) | **Yes** (per-instance UUID) |
| Cost attribution | DCGM per-pod util (allocation dead) | DCGM per-pod util (allocation dead) | **Allocation counts work** (per-UUID) |
| Concurrency gain | No (serialised turns) | **Yes** for under-filling jobs | Yes, but capped by fixed slice size |
| Maturity | GA, widely used | **Experimental** since device-plugin v0.15.0; **incompatible with MIG on the same device** | GA, widely used |
| Use when | Bursty/idle **trusted** dev & low-QPS inference; simplest overcommit | Many **small concurrent trusted** jobs that under-fill the GPU and need throughput | **Untrusted/regulated** multi-tenancy or hard SLOs needing real isolation + clean billing |

Read it as one decision: **need isolation or clean per-tenant billing → MIG. Trusted, jobs under-fill the GPU, want throughput, and you accept an experimental component → MPS. Trusted, just need to overcommit an idle GPU cheaply → time-slicing.**

## Perspectives

**Throughput-engineer view.** MPS's win condition is a packing problem: many small kernels that each leave most of the SM array idle can be interleaved spatially instead of serialized temporally. The mental model is bin-packing in the *SM-occupancy* dimension rather than the *time* dimension — time-slicing already "packs" in time (no wasted GPU-seconds between turns beyond switch overhead), but each turn still wastes SM capacity if the job doesn't need the whole device. MPS is the mode that reclaims that wasted SM capacity.

**Reliability/blast-radius view.** "Worse fault domain than time-slicing" is the fact that disqualifies MPS from any use case involving untrusted or arms-length tenants, and it's disqualifying regardless of how good the throughput numbers look. A reliability engineer's framing: time-slicing has one shared resource (the physical GPU) that a bad neighbor can degrade; MPS has two (the physical GPU, plus the MPS server process itself), and a crash in the second one is a harder, faster failure than anything time-slicing produces.

**Real production adopter view.** Databricks has published an account of using MPS to scale small-LLM inference on GKE with A100-80GB GPUs, motivated by the concrete observation that a 15 GB model sitting on an 80 GB GPU leaves more than 80% of the card's memory idle — exactly the "jobs under-fill the GPU" precondition this lesson's decision table names as MPS's win condition. *(Cited from search discovery; not independently fetched this session due to a proxy block on the domain — treat the specific numbers as needing a spot-check, but the shape of the story — MPS for small-model inference consolidation — is corroborated by the vendor's own device-plugin docs describing exactly this use case.)*

**Comparative decision-maker view.** Someone sitting in a design review doesn't need to re-derive MIG vs time-slicing vs MPS from first principles every time — they need the three-way table above as a lookup, and the ScaleOps comparison post covers the same three-way ground independently, which is useful as a second source when you want to sanity-check your own table against someone else's before presenting it.

## Real-world use cases

- **["Scaling Small LLMs with NVIDIA MPS" — Databricks](https://www.databricks.com/blog/scaling-small-llms-nvidia-mps).** A real production account of using MPS on GKE with A100-80GB GPUs to serve multiple small LLMs concurrently, motivated by a 15 GB model leaving most of an 80 GB card's memory unused under exclusive allocation — the canonical "jobs under-fill the GPU" MPS use case, from a company actually running it. *(Not independently fetched this session — the research proxy blocked the domain. Corroborated via search; treat exact figures as needing a spot-check before quoting them verbatim in an interview.)*
- **["Kubernetes GPU Sharing" — ScaleOps](https://scaleops.com/blog/kubernetes-gpu-sharing/).** A side-by-side practitioner comparison of time-slicing, MPS, and MIG mechanisms and trade-offs — a second, independent source for the decision table in this lesson, useful for cross-checking. *(Not independently fetched this session — proxy-blocked; corroborated via search.)*
- **[NVIDIA/k8s-device-plugin README — "Shared Access to GPUs"](https://github.com/NVIDIA/k8s-device-plugin) (v0.17.1).** The primary source confirming MPS's experimental status (since v0.15.0) and its explicit incompatibility with MIG-enabled devices — the vendor's own maturity statement, directly fetched and verified this session.

## Worked example — MPS vs time-slicing throughput on one GPU

Setup: one A100-40GB, four identical pods each serving a small model whose kernel occupies ~25% of the SMs; measure aggregate requests/sec over 60 s under each mode (worked placeholders, dated 2026 — measure on your own SKU before quoting these numbers anywhere real).

| Mode | Per-job r/s | Aggregate r/s | GPU SM occupancy | Note |
|------|-------------|---------------|-------------------|------|
| Time-slicing (`replicas:4`) | ~240 | **~250** | ~25% avg | Jobs serialise; ~1× a single job + switch overhead |
| MPS (`replicas:4`) | ~210 | **~840** | ~92% avg | Kernels overlap on the SMs; ~3.5× time-slicing |

MPS delivers ~3.4× aggregate throughput here **because the jobs under-fill the GPU** — the same precondition that motivated Databricks's published use case. The per-job rate dips slightly under MPS (SM contention), but aggregate — the number finance cares about for $/useful-work — jumps.

Now the caveats you must attach, both load-bearing:

1. **Fault domain.** This GPU is a single shared fault domain. If one of the four pods hits an illegal-address error, expect all four to fail together — a trade you accept only because these are your own trusted services.
2. **Attribution is unchanged from lesson 07.** All four pods report the same physical UUID, so you split the GPU's $/hr by DCGM per-pod SM-active time, not by the (uniform) allocation — and per lesson 07's finding, you verify that your chosen DCGM field actually resolves per-pod on this hardware before trusting the split.

## Practice

For the [Per-pod GPU attribution](../practice/per-pod-attribution/README.md) deliverable:

1. **Enable MPS.** Apply the `mps` sharing ConfigMap, patch the ClusterPolicy, and confirm `nvidia.com/gpu.sharing-strategy: mps` with N allocatable replicas over one physical GPU. Capture the node labels. (A sub-$1/hr single-GPU rental is sufficient for this.)
2. **Throughput comparison.** Run the same set of several small concurrent jobs (each sized to under-fill the GPU) first under time-slicing (`replicas: N`, reusing lesson 07's config), then under MPS (`replicas: N`), on the same physical GPU. Record aggregate throughput (r/s or tok/s) and DCGM `DCGM_FI_PROF_SM_ACTIVE` for each mode.
3. **Document the fault-domain risk.** In writing, state the "one fatal CUDA error kills the MPS control daemon and all clients" property and that it disqualifies MPS for untrusted tenants. Optionally demonstrate by forcing a CUDA illegal-address error in one client and observing sibling clients fail.
4. **Verify the experimental/MIG-incompatibility claims yourself.** Check the installed device-plugin version's README or release notes for the MPS experimental flag and the MIG-incompatibility statement; note the version you tested against, since this status may change.
5. **Write the decision table.** Produce the MIG / time-slicing / MPS table (mechanism, isolation ×3, distinct UUID, attribution, maturity, when-to-use) as the operator's mode-selection reference — this is the artifact the whole three-lesson arc (06/07/08) has been building toward.

**Acceptance:** an aggregate-throughput comparison showing MPS > time-slicing for under-filling concurrent jobs on the same GPU (with the SM-occupancy evidence), **and** a written MIG/time-slicing/MPS decision table that names, for each mode, its mechanism, its three isolation properties, whether it yields a distinct UUID, its attribution basis, and its maturity status. These consolidate the sharing comparison for the deliverable and double as your cold-answer reference for the module's interview probe.

## Common pitfalls

1. **Assuming MPS gives you memory isolation because it has a "memory limit" knob.** `CUDA_MPS_PINNED_DEVICE_MEM_LIMIT` is a cooperative cap the client process can be built to respect — it is not a hardware wall the GPU enforces the way a MIG partition's dedicated framebuffer slice is. A misbehaving or malicious client can still exceed it.
2. **Trying to run MPS and MIG together on the same physical GPU.** They are explicitly, mutually incompatible per NVIDIA's own device-plugin documentation — a device is either MIG-partitioned or MPS-shared (or time-sliced, or exclusive), never MIG-plus-MPS simultaneously. Route MIG tenants and MPS tenants to different node pools.
3. **Treating "MPS is still labeled experimental" as a reason to dismiss it outright, or as a non-issue to ignore.** Both extremes are wrong for a senior answer. The honest position is: it's real, it's used in production (see Databricks), but the experimental label means you validate it yourself and deploy it deliberately rather than defaulting to it — this nuance is itself frequently the actual interview question.
4. **Reusing the time-slicing attribution formula from lesson 07 without re-verifying the DCGM field resolves per-pod under MPS specifically.** The mechanism differs (concurrent execution vs turns) even though the attribution *conclusion* (allocation is dead, fall back to DCGM utilization) is the same — don't assume a field verified under time-slicing automatically behaves identically under MPS on the same hardware.
5. **Deploying MPS for jobs that already saturate the GPU.** If each job needs the whole SM array, MPS buys no throughput over time-slicing — you've added a shared fault-domain risk for zero upside. Check the "does this job under-fill the GPU" precondition before reaching for MPS.

## Self-check

- MPS vs time-slicing — what's the mechanistic difference (concurrent SMs vs time turns)? **Answer:** Time-slicing context-switches the whole GPU between clients — they run one at a time, taking turns, so an under-filling job leaves the GPU mostly idle during its slice. MPS runs an MPS control daemon that funnels all clients into one shared CUDA context, letting their kernels execute spatially concurrently across the SMs at the same instant. Turns vs concurrency: that's why MPS can pack four 25%-SM jobs onto one GPU while time-slicing serialises them.
- When does MPS beat both time-slicing and MIG? **Answer:** When you have many small, trusted, concurrent jobs that individually under-fill the GPU and you want aggregate throughput. MPS beats time-slicing because those jobs overlap on the SMs instead of serialising (higher occupancy, higher aggregate throughput) — Databricks's small-LLM-inference use case is a real example. It beats MIG because MIG's fixed partition boundaries would waste capacity (a fixed slice can't lend idle SMs to a busy neighbor, and small jobs may not tile the fixed slice sizes efficiently), whereas MPS lets work fill whatever SMs are free. The precondition is trust — MPS's shared fault domain is acceptable only among cooperative workloads.
- Why is MPS disqualified for hostile/untrusted multi-tenancy? **Answer:** All clients share one CUDA context and one MPS server, and there is no memory hard-wall and no fault boundary between them. A fatal CUDA error in any single client — illegal address, uncorrectable ECC, a crash inside the shared context — can poison the MPS control daemon and take down every client on that GPU simultaneously. One tenant's bug (or deliberate abuse) becomes an outage for all co-residents, so MPS is safe only for trusted, cooperative workloads; untrusted tenants need MIG's hardware isolation.
- Is MPS production-ready — what would you actually tell a hiring manager? **Answer:** Yes and no, precisely stated: NVIDIA's device plugin has marked MPS sharing "experimental" since v0.15.0, and it's explicitly unsupported on MIG-enabled devices, so it's not a mature, boring default the way time-slicing or MIG are. But it is genuinely used in production — Databricks has published an account of running it for small-LLM inference consolidation — so the correct answer is "usable in production for trusted, throughput-sensitive workloads, deployed deliberately with the experimental status and fault-domain risk explicitly acknowledged," not a flat yes or no.

## Connections & what's next

This lesson closes the three-lesson sharing-strategy arc that started in lesson 06: MIG gives hardware isolation and clean allocation-based billing; time-slicing (lesson 07) gives free overcommit with zero isolation and a DCGM-fallback attribution story; MPS gives concurrent-execution throughput with the worst fault domain of the three and the same DCGM-fallback attribution story as time-slicing. The three-way decision table above is the single artifact that answers the module README's cold interview probe on this topic, and it's also the mode-selection logic your capstone cost operator needs to branch on via the `nvidia.com/gpu.sharing-strategy` node label.

From here the module moves from *sharing one GPU* to *scheduling many* — lesson 09 covers the DRA driver's claim-based model and multi-tenancy quotas, which is where "which sharing mode does this workload get" starts being expressed as a schedulable object (`ResourceClaim`/`DeviceClass`) instead of a ConfigMap-driven device-plugin flag. Carry this lesson's attribution conclusions forward unchanged — DRA doesn't retroactively fix time-slicing's or MPS's shared-UUID problem, it changes how the *request* for a shared or partitioned device gets expressed and scheduled.

Next: **[04.9 · DRA driver (real install) + multi-tenancy quotas](09-dra-driver-and-quotas.md)** — from device-plugin ConfigMaps to `ResourceClaim`/`DeviceClass` objects, and namespace-level GPU quotas.

## References & further reading

**Primary sources**
- [NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) (confirmed v0.17.1) — read for the authoritative MPS sharing ConfigMap schema, and the exact confirmation that MPS has been "experimental" since v0.15.0 and is unsupported on MIG-enabled devices.
- [NVIDIA Multi-Process Service (MPS) documentation](https://docs.nvidia.com/deploy/mps/) — authoritative on the control daemon, shared context, `ACTIVE_THREAD_PERCENTAGE`, memory limits, and error/containment behavior underlying this lesson's mechanism section. *(Not independently fetched this session — proxy-blocked; treat exact flag semantics as needing a spot-check against your CUDA/driver version.)*

**Real-world engineering blogs**
- Databricks, ["Scaling Small LLMs with NVIDIA MPS"](https://www.databricks.com/blog/scaling-small-llms-nvidia-mps) — a real production adopter's account of MPS for small-model inference consolidation on GKE/A100-80GB; the concrete "15 GB model, 80 GB card" motivation behind this lesson's throughput story. *(Not independently fetched — proxy-blocked; corroborated via search.)*
- ScaleOps, ["Kubernetes GPU Sharing"](https://scaleops.com/blog/kubernetes-gpu-sharing/) — an independent practitioner comparison of MIG, time-slicing, and MPS; useful as a cross-check on this lesson's decision table. *(Not independently fetched — proxy-blocked; corroborated via search.)*

**Deeper dives**
- NVIDIA GPU Operator docs, ["GPU Sharing"](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html) — the `sharing.mps` ConfigMap and ClusterPolicy wiring reference behind the config shown in this lesson. *(Not independently fetched this session — proxy-blocked; spot-check field names against your installed Operator version.)*
- Lesson [07 — Time-slicing and the attribution hole](07-time-slicing-attribution.md) — read first if you haven't: the DCGM-fallback attribution mechanism this lesson reuses without re-deriving is fully worked there, including the dcgm-exporter#642 evidence for why you must verify per-pod resolution empirically.
