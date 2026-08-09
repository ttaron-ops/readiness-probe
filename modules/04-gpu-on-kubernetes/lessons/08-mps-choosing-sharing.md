---
lesson: "04.8"
title: "MPS and choosing a sharing mode"
module: "04"
concept: "MPS and choosing a sharing mode"
status: not-started
est_time: "5h"
artifacts: []
---
# 04.8 · MPS and choosing a sharing mode
> **Concept.** MPS shares a GPU by running clients concurrently on the SMs (spatial, shared memory, shared fault domain); consolidate MIG vs time-slicing vs MPS into one mechanism/isolation/when-to-use decision table.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Why this matters

You now have two of the three sharing modes in hand: MIG (walls in silicon,
distinct UUIDs, real isolation) and time-slicing (turns, no isolation, collapsed
attribution). MPS is the third, and it is the one most people cannot place. It
sits *between* the other two on every axis, and picking it — or refusing it — is a
one-question sorting exercise that a senior platform engineer must answer without
hesitation in a design review.

The reason it matters for your differentiator: MPS is the mode that actually
*raises throughput* on small concurrent jobs, because it lets kernels from
different processes run **at the same time on different SMs** instead of waiting
their turn. If your cost-per-useful-work story only knows time-slicing, you will
leave real efficiency on the table on inference fleets — and you will also miss
the sharp edge that disqualifies MPS for untrusted tenants: one fatal CUDA error
takes down the shared MPS server and **every** client with it. Owning the clean
three-way mental model is what turns "we share GPUs" into a defensible policy.

## What's new here

- **Concurrent execution, not turns.** Time-slicing context-switches the whole
  GPU between clients. MPS runs a single **control daemon** (the MPS server) that
  funnels all clients' work into **one shared CUDA context**, so kernels from
  different processes execute *spatially concurrently* across the SMs. Small
  kernels that each leave the GPU 80% idle now overlap instead of serialising.
- **Optional compute provisioning.** MPS can cap each client's SM fraction via
  `CUDA_MPS_ACTIVE_THREAD_PERCENTAGE` (Volta+), giving soft compute partitioning
  — a knob time-slicing does not have. Memory is still **not** hard-partitioned
  (a client can request a limit via `CUDA_MPS_PINNED_DEVICE_MEM_LIMIT`, but this
  is a cooperative cap, not a hardware wall).
- **A new, worse fault domain.** All clients share one context and one server.
  A fatal CUDA error (illegal address, uncorrectable ECC, a client crash inside
  the shared context) can poison the MPS server and **kill all clients at once**.
  This is a *broader* blast radius than time-slicing's, not a narrower one.
- **The device-plugin exposes it as `mps` sharing.** Same ConfigMap shape as
  time-slicing, different `renameByDefault`/strategy — the plugin launches and
  manages the `nvidia-cuda-mps-control` daemon for you.

## Core notes — the meat

### 1 — Enabling MPS via the device plugin

MPS is configured through the same sharing block as time-slicing, selecting the
`mps` strategy. With the GPU Operator:

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

The operator stands up the MPS control daemon per GPU (as a managed
`nvidia-cuda-mps-control -d` process) and steers client CUDA processes at it via
`CUDA_MPS_PIPE_DIRECTORY`. Confirm the node now advertises MPS sharing:

```bash
$ kubectl get node <node> -o json \
    | jq '.metadata.labels["nvidia.com/gpu.sharing-strategy"]'
"mps"

$ kubectl describe node <node> | grep nvidia.com/gpu:
  nvidia.com/gpu:  4        # 4 concurrent MPS clients over 1 physical GPU
```

Like time-slicing, allocation is still uniform — MPS **does not** give each client
a distinct UUID (only MIG does). So the attribution trap from 04.7 applies
identically: **you still cannot bill from allocation counts; you fall back to DCGM
per-pod utilisation.** MPS changes throughput and isolation, not the attribution
story.

### 2 — Why MPS raises throughput where time-slicing does not

Consider four inference pods each running a small model whose kernels occupy ~25%
of the SMs. Under **time-slicing** they run one at a time — the GPU is ~25% busy
in each slice, and aggregate throughput is roughly one job's worth plus
context-switch overhead. Under **MPS** all four kernels dispatch into the shared
context and pack onto the SM array concurrently — the GPU approaches ~100%
occupancy and aggregate throughput can approach ~4× the single-job rate (until you
hit an SM, memory-bandwidth, or framebuffer ceiling).

The rule of thumb: **MPS wins exactly when individual jobs under-fill the GPU and
run concurrently.** If each job already saturates the SMs, MPS buys nothing over
time-slicing (they now contend for the same busy SMs) and you have taken on the
shared-fault-domain risk for no throughput gain.

### 3 — The fatal-error blast radius

The disqualifier. Because every client shares one CUDA context under the MPS
server, an unrecoverable fault in *any* client — illegal memory access, an
uncorrectable ECC event, a hang — can corrupt the shared context and bring down
the **MPS control daemon and all connected clients simultaneously**. There is no
per-client fault boundary. (Volta+ MPS improved error containment for some
client-side faults, but a fatal fault on the device remains fleet-fatal for that
GPU's MPS clients.) This is why MPS is for **trusted, cooperative** workloads only
— your own services sharing a box — and never for hostile or arms-length
multi-tenancy, where one tenant's bug becomes every tenant's outage.

### 4 — The three-way decision table (the consolidation)

| Axis | **Time-slicing** | **MPS** | **MIG** |
|------|------------------|---------|---------|
| Mechanism | Share by **taking turns** — GPU context-switches between clients | Share by **running concurrently on the SMs** — one shared context via the MPS server | Share by **building walls in silicon** — fixed HW partitions |
| Compute isolation | None (whole GPU per turn) | Soft — optional SM % cap (`ACTIVE_THREAD_PERCENTAGE`) | Hard — dedicated SMs per instance |
| Memory isolation | **None** — full FB, uncapped | Soft/cooperative cap only, shared FB | **Hard** — dedicated framebuffer slice |
| Fault isolation | None — fault hits all co-residents | **Worst** — one fatal error kills MPS server + all clients | **Hard** — fault contained to the instance |
| Distinct GPU UUID? | **No** (same physical UUID) | **No** (same physical UUID) | **Yes** (per-instance UUID) |
| Cost attribution | DCGM per-pod util (allocation dead) | DCGM per-pod util (allocation dead) | **Allocation counts work** (per-UUID) |
| Concurrency gain | No (serialised turns) | **Yes** for under-filling jobs | Yes, but capped by fixed slice size |
| Use when | Bursty/idle **trusted** dev & low-QPS inference; simplest overcommit | Many **small concurrent trusted** jobs that under-fill the GPU and need throughput | **Untrusted/regulated** multi-tenancy or hard SLOs needing real isolation + clean billing |

Read it as one decision: **need isolation or clean per-tenant billing → MIG.**
**Trusted, jobs under-fill the GPU, want throughput → MPS.** **Trusted, just need
to overcommit an idle GPU cheaply → time-slicing.**

## Worked example — MPS vs time-slicing throughput on one GPU

Setup: one A100-40GB, four identical pods each serving a small model whose kernel
occupies ~25% of the SMs; measure aggregate requests/sec over 60 s under each
mode (worked placeholders — measure on your SKU).

| Mode | Per-job r/s | Aggregate r/s | GPU SM occupancy | Note |
|------|-------------|---------------|------------------|------|
| Time-slicing (`replicas:4`) | ~240 | **~250** | ~25% avg | Jobs serialise; ~1× a single job + switch overhead |
| MPS (`replicas:4`) | ~210 | **~840** | ~92% avg | Kernels overlap on the SMs; ~3.5× time-slicing |

MPS delivers ~3.4× aggregate throughput here **because the jobs under-fill the
GPU**. The per-job rate dips slightly under MPS (SM contention), but aggregate —
the number finance cares about for $/useful-work — jumps. Now the caveat you must
attach: this GPU is a **single shared fault domain**. If one of the four pods hits
an illegal-address error, expect all four to fail together — a trade you accept
only because these are your own trusted services. And attribution is unchanged
from 04.7: all four pods report the same physical UUID, so you split the GPU's
$/hr by DCGM per-pod SM-active time, not by the (uniform) allocation.

## Practice — MPS throughput + decision table (feeds the deliverable)

For the [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)
deliverable:

1. **Enable MPS.** Apply the `mps` sharing ConfigMap, patch the ClusterPolicy,
   and confirm `nvidia.com/gpu.sharing-strategy: mps` with N allocatable replicas
   over one physical GPU. Capture the node labels.
2. **Throughput comparison.** Run the same set of several small concurrent jobs
   (each sized to under-fill the GPU) first under time-slicing (`replicas: N`),
   then under MPS (`replicas: N`), on the same physical GPU. Record aggregate
   throughput (r/s or tok/s) and DCGM `DCGM_FI_PROF_SM_ACTIVE` for each mode.
3. **Document the fault-domain risk.** In writing, state the "one fatal CUDA error
   kills the MPS control daemon and all clients" property and that it disqualifies
   MPS for untrusted tenants. Optionally demonstrate by forcing a CUDA illegal
   access in one client and observing sibling clients fail.
4. **Write the decision table.** Produce the MIG / time-slicing / MPS table
   (mechanism, isolation ×3, distinct UUID, attribution, when-to-use) as the
   operator's mode-selection reference.

**Acceptance:** an aggregate-throughput comparison showing MPS > time-slicing for
under-filling concurrent jobs on the same GPU (with the SM-occupancy evidence),
**and** a written MIG/time-slicing/MPS decision table that names, for each mode,
its mechanism, its three isolation properties, whether it yields a distinct UUID,
and its attribution basis. These consolidate the sharing comparison for the
deliverable.

## Self-check

**(a) MPS vs time-slicing — what's the mechanistic difference (concurrent SMs vs
time turns)?**
**Answer:** Time-slicing context-switches the **whole GPU** between clients — they
run one at a time, taking turns, so an under-filling job leaves the GPU mostly
idle during its slice. MPS runs an **MPS control daemon** that funnels all clients
into **one shared CUDA context**, letting their kernels execute **spatially
concurrently across the SMs** at the same instant. Turns vs concurrency: that is
why MPS can pack four 25%-SM jobs onto one GPU while time-slicing serialises them.

**(b) When does MPS beat both time-slicing and MIG?**
**Answer:** When you have **many small, trusted, concurrent jobs that individually
under-fill the GPU** and you want aggregate throughput. MPS beats time-slicing
because those jobs overlap on the SMs instead of serialising (higher occupancy,
higher aggregate throughput). It beats MIG because MIG's fixed partition
boundaries would waste capacity (a 1g.10gb slice can't lend idle SMs to a busy
neighbour, and small jobs may not tile the fixed slice sizes efficiently), whereas
MPS lets work fill whatever SMs are free. The precondition is **trust** — MPS's
shared fault domain is acceptable only among cooperative workloads.

**(c) Why is MPS disqualified for hostile/untrusted multi-tenancy?**
**Answer:** All clients share **one CUDA context and one MPS server**, and there is
**no memory hard-wall and no fault boundary** between them. A fatal CUDA error in
any single client — illegal address, uncorrectable ECC, a crash inside the shared
context — can poison the MPS control daemon and **take down every client on that
GPU simultaneously**. One tenant's bug (or deliberate abuse) becomes an outage for
all co-residents, so MPS is safe only for trusted, cooperative workloads;
untrusted tenants need MIG's hardware isolation.

## Resources

1. **ScaleOps — Kubernetes GPU Sharing.** Side-by-side of time-slicing, MPS, and
   MIG mechanisms and trade-offs — the decision-table source.
   https://scaleops.com/blog/kubernetes-gpu-sharing/
2. **NVIDIA Multi-Process Service (MPS) documentation.** Authoritative on the
   control daemon, shared context, `ACTIVE_THREAD_PERCENTAGE`, memory limits, and
   error/containment behaviour. https://docs.nvidia.com/deploy/mps/
3. **NVIDIA GPU Operator — GPU Sharing.** The `sharing.mps` ConfigMap and
   ClusterPolicy wiring that stands up the managed MPS daemon.
   https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html
