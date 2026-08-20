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
sources: 11
---

# 04.8 · MPS and choosing a sharing mode

> **Concept.** MPS shares a GPU by running clients concurrently on the SMs (spatial sharing, one shared CUDA context, one shared fault domain) — the third point in the MIG / time-slicing / MPS decision triangle, and it is marked *experimental* in NVIDIA's own device plugin as of this writing, which is itself part of the senior answer about when to use it.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)
>
> **Advanced Learning** — [MPS and the Decision Triangle](../../../learning/08-mps-choosing-sharing.html): the full MIG / time-slicing / MPS comparison matrix, and the ceilings that bound the fair-share estimate. Optional visual companion; this lesson stays the source of truth.

## Where this fits

Lesson 06 gave you MIG: hardware walls, distinct UUIDs, an exact 1:1 attribution join. Lesson 07 gave you time-slicing, and — more importantly — gave you the *mechanical* statement of what sharing breaks: the per-pod **scheduling** identity survives (the device plugin's annotated `GPU-<uuid>::N` IDs come through the pod-resources API intact), while the per-pod **measurement** identity does not (DCGM programs counters per physical device, so one number gets fanned out across every holder, and a naive `sum` over-reports by exactly the number of holders).

This lesson completes the triangle with MPS. Its attribution *conclusion* is identical to time-slicing's — same annotated IDs, same device-level counters, same fan-out — but the mechanism underneath differs on every axis that matters, and one of those differences is genuinely good news for attribution: **MPS enforces per-client compute and memory ceilings, which turns "fair share" from a guess into a bounded estimate.** That is the sharpest reason to know MPS well, and almost nobody says it.

By the end you will have the three-way decision table the module README names as an explicit interview probe — "MIG vs time-slicing vs MPS — mechanism/isolation/when" — answerable cold, with the attribution consequence of each mode attached.

## Why this matters

MPS is the sharing mode most engineers misplace. They either do not know it exists, or they know the name and reach for the wrong mechanism when asked how it differs from time-slicing. It matters for three concrete reasons.

**It is the only mode that reclaims wasted SM capacity.** Time-slicing packs the *time* axis: there are no idle GPU-seconds between turns beyond switch overhead. But each turn still wastes whatever fraction of the SM array the resident job does not need. A 15 GB model on an 80 GB card typically uses a small fraction of the SMs, and under time-slicing that fraction is all the GPU ever does at any instant. MPS packs the *occupancy* axis, letting kernels from different processes execute concurrently on different SMs. On workloads that under-fill the device, that is a direct throughput multiplier, and throughput per dollar is the entire subject of module 11.

**It changes the shape of the attribution estimate.** Under time-slicing the shares are unbounded — one pod can consume 98% of the device's work and your fair-share estimate will be off by a factor of four. Under MPS the control daemon sets a *default active thread percentage* and a *default pinned device memory limit* per client, both enforced by the driver. Those are ceilings, not reservations, so a client can still under-use its share — but it cannot exceed it. Fair share therefore becomes an *upper-bounded* estimator: you can state the worst-case error rather than shrug at it. That is the difference between a number a FinOps lead will accept and one they will not.

**Its risk profile is genuinely different, and the common summary of it is wrong.** "MPS has the worst fault domain" is the received wisdom and it is too crude. Volta and later changed both the work-submission path and the address-space model, so a fatal fault is contained to the set of clients sharing the offending GPU rather than every client of the server across all GPUs. That is *narrower* than pre-Volta. It is still, however, an all-clients-on-that-GPU kill, plus a second failure mode time-slicing does not have at all: the control daemon and per-user server are processes that can die, hang, or be left in a fault state. Getting that nuance right — narrower than the folklore, still disqualifying for untrusted tenants — is what a design review is actually testing.

And the maturity caveat is load-bearing, not a footnote. MPS sharing is marked **experimental** in NVIDIA's device plugin as of v0.15.0 and remains so through the current v0.19.x line, is **not supported on MIG-enabled devices** by that plugin, and in the DRA driver (lesson 09) MPS support sits behind an **alpha, default-off feature gate**. "Is MPS production-ready?" has a real answer with three clauses in it, and delivering it cleanly is the interview.

## What's new here (calibration)

This module's README calibration (skip device-plugin gRPC mechanics, the DRA object model, Topology Manager internals, silicon-level MIG — modules 02/02b/03) applies unchanged. Lessons 06 and 07 built the "same resource name, different attribution regime" frame and the DCGM fan-out mechanism; neither is re-derived. What this lesson adds:

- **The MPS architecture as three distinct processes** — control daemon, per-user server, clients — with the pre-Volta versus Volta+ difference stated at the level of *who submits work to the hardware* and *whose page tables are in play*. "One shared context" is the pre-Volta model and is the single most common inaccuracy about MPS.
- **The complete control interface**: every `nvidia-cuda-mps-control` command, every environment variable with its default, and the client-count limits per compute capability — reproduced from the `nvidia-cuda-mps-control(1)` manual page rather than paraphrased.
- **What the Kubernetes MPS control daemon actually computes**, read out of its source: `100/replicas` active thread percentage, `totalMemory/replicas` pinned memory limit in MiB, compute mode forced to `EXCLUSIVE_PROCESS`, the pipe/log directory layout, and the two environment variables injected into every client.
- **The attribution upgrade nobody mentions** — enforced ceilings turn fair share into a bounded estimator — with the error bound derived.
- **The correction on MPS + MIG.** The incompatibility is a *device-plugin* limitation, not a hardware one: the DRA driver models MPS on MIG devices explicitly (`MigDeviceSharing` with an `MpsConfig`). This matters because "MPS and MIG are mutually exclusive" is the kind of absolute claim that gets falsified in a follow-up question.
- **Field numbers from named benchmarks**, with the win conditions and the loss conditions both stated, because a mode that only ever helps is a mode you have not measured.

## Core concepts

### 1 — The problem: time-slicing packs time, never occupancy

Restate lesson 07's timeline, then add the axis it leaves on the table.

```
  FOUR INFERENCE PODS, EACH NEEDING ~25% OF THE SM ARRAY

  TIME-SLICING (replicas: 4) — pack the TIME axis
                0ms      2ms      4ms      6ms      8ms
  ctx A          ▓▓▓▓▓▓▓▓ ........ ........ ........ ▓▓▓▓▓▓▓▓
  ctx B          ........ ▓▓▓▓▓▓▓▓ ........ ........ ........
  ctx C          ........ ........ ▓▓▓▓▓▓▓▓ ........ ........
  ctx D          ........ ........ ........ ▓▓▓▓▓▓▓▓ ........

  SM array   ┌──────────────────────────────────────────────┐
  (132 SMs)  │ ██████████████ 33 SMs busy (one tenant's work)│
             │ ·············································· │
             │ ············ 99 SMs IDLE, EVERY INSTANT ····· │
             └──────────────────────────────────────────────┘
  aggregate throughput ≈ 1× a single pod  (+ switch overhead)
  device GR_ENGINE_ACTIVE ≈ 1.00   ← "busy" and yet 75% wasted

  MPS (replicas: 4) — pack the OCCUPANCY axis
                0ms      2ms      4ms      6ms      8ms
  client A       ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓
  client B       ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓
  client C       ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓
  client D       ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓
                 └─ all four resident SIMULTANEOUSLY ─┘

  SM array   ┌──────────────────────────────────────────────┐
  (132 SMs)  │ ███████ A │ ███████ B │ ███████ C │ ███████ D │
             │  ~33 SMs  │  ~33 SMs  │  ~33 SMs  │  ~33 SMs  │
             └──────────────────────────────────────────────┘
  aggregate throughput → up to ~4× a single pod
  device GR_ENGINE_ACTIVE ≈ 1.00   ← SAME NUMBER, 4× the work

  ⚠ NOTE THE LAST TWO LINES. The device-level utilisation metric is
    ~1.00 in BOTH cases. It cannot tell you which of these two
    pictures you are looking at. That is why lesson 07's warning
    about device counters applies here verbatim — and why occupancy
    (DCGM_FI_PROF_SM_OCCUPANCY) is the field that distinguishes them.
```

That last observation is worth internalising. `DCGM_FI_PROF_GR_ENGINE_ACTIVE` ("ratio of time the graphics engine is active") saturates the moment *anything* is running. `DCGM_FI_PROF_SM_ACTIVE` ("ratio of cycles an SM has at least 1 warp assigned", averaged over SMs) and `DCGM_FI_PROF_SM_OCCUPANCY` ("ratio of number of warps resident on an SM") are the fields that see the difference between one tenant on 25% of the array and four tenants on 100% of it. If your dashboard only carries `GR_ENGINE_ACTIVE`, switching a pool from time-slicing to MPS will produce **no visible change** while quadrupling throughput — and you will have no evidence the change worked.

**The rule that falls out: MPS wins exactly when individual jobs under-fill the GPU and run concurrently.** If each job already saturates the SM array, MPS buys nothing over time-slicing — the clients now contend for the same busy SMs, you pay context-management overhead, and you have accepted a shared fault domain for zero throughput. Check the precondition (`SM_OCCUPANCY` well below its ceiling under exclusive allocation) before reaching for MPS.

### 2 — MPS architecture: three processes, and what changed at Volta

MPS is not one thing. It is three:

| Process | Binary | Lifetime | Role |
|---|---|---|---|
| **Control daemon** | `nvidia-cuda-mps-control -d` | long-lived, one per node (or per resource) | Accepts control commands on a pipe; spawns/reaps servers; holds the *default* provisioning settings that new servers inherit |
| **Server** | `nvidia-cuda-mps-server` | one per user (uid) per daemon | Owns the GPU-side resources shared by that user's clients; funnels client connections; holds the CUDA context(s) |
| **Client** | your process | your workload | A normal CUDA process that discovers the pipe via `CUDA_MPS_PIPE_DIRECTORY` and transparently becomes an MPS client |

By default a daemon runs **one server per uid**, which is the isolation boundary MPS was originally designed around. `nvidia-cuda-mps-control` can be started in multi-user mode to allow one server to serve clients with different uids — the DRA driver exposes this as `MpsConfig.multiUser` — but that deliberately removes a boundary, so treat it as an explicit decision.

Now the part most summaries get wrong. **Pre-Volta and Volta+ MPS are architecturally different systems:**

```
  PRE-VOLTA MPS (SM 3.5 – 6.x)
  ────────────────────────────
   client A ──┐
   client B ──┤─ pipe ─▶ MPS SERVER ──▶ [ ONE shared CUDA context ]
   client C ──┘         (proxies every        │  ONE GPU address
                         work submission)     │  space, PARTITIONED
                                              ▼  between clients
                                        ┌───────────┐
                                        │  the GPU  │
                                        └───────────┘
   • Every kernel launch traverses the server → proxy latency.
   • Clients share one GPU virtual address space, carved into
     partitions. A stray pointer can reach a neighbour's data.
   • A fatal fault is reported to ALL clients of that server.

  VOLTA AND LATER (SM 7.0+) — this is what you are running
  ─────────────────────────────────────────────────────────
   client A ─── own GPU address space ──┐
   client B ─── own GPU address space ──┤──▶ hardware work queues
   client C ─── own GPU address space ──┘        │  DIRECTLY.
                    ▲                            ▼
                    │                     ┌───────────┐
              MPS SERVER (setup,          │  the GPU  │
              resource mgmt, provisioning)└───────────┘
                    ▲
                  pipe
                    │
              CONTROL DAEMON

   • Clients submit work directly to the GPU without passing
     through the server → the proxy latency is gone from the
     data path. (NVIDIA MPS documentation, "Volta MPS".)
   • Each client owns its own GPU address space → memory
     PROTECTION between clients. Not a memory *partition* by
     itself, but client A cannot read client B's allocations.
   • A fatal GPU exception is contained to the set of clients
     sharing the offending GPU — narrower than pre-Volta, but
     still an all-clients-on-that-device kill.
```

Two corrections fall out of that diagram, and both are commonly stated wrong — including in the previous version of this lesson:

- **"MPS funnels all clients' work into one shared CUDA context"** describes pre-Volta MPS. On Volta and later, clients own separate GPU address spaces and submit directly to the hardware. Say "a shared server process that provisions and arbitrates, with per-client address spaces" and you are describing the hardware you actually have.
- **"MPS gives no memory isolation"** is too strong on two counts. Volta+ gives memory *protection* (separate address spaces — a neighbour cannot read your allocations), and the pinned-device-memory limit gives a driver-*enforced* cap on how much device memory a client can allocate (§4). What MPS does not give is a *hardware memory partition* the way MIG does: there is still one physical framebuffer, one memory controller, one L2, and the limits are policy set on the server rather than walls in silicon.

Client-count limits, from the `nvidia-cuda-mps-control(1)` manual page: **compute capability SM 3.5 through SM 6.0 is limited to 16 clients per GPU at a time; SM 7.0 raises that to 48.** Treat 48 as the practical ceiling on modern datacenter parts and verify against your driver's documentation for the newest architectures rather than assuming the number scaled again.

### 3 — The control interface, completely

You will debug MPS through this interface, so know it. `nvidia-cuda-mps-control` accepts commands either interactively or by piping a single command to it — the latter is what automation does:

```bash
# Start the daemon (per node / per resource)
$ export CUDA_MPS_PIPE_DIRECTORY=/run/nvidia/mps/nvidia.com~gpu/pipe
$ export CUDA_MPS_LOG_DIRECTORY=/run/nvidia/mps/nvidia.com~gpu/log
$ nvidia-cuda-mps-control -d

# One-shot command form (how the k8s control daemon drives it)
$ echo get_default_active_thread_percentage | nvidia-cuda-mps-control
100.0

$ echo get_server_list | nvidia-cuda-mps-control
28941

$ echo get_client_list 28941 | nvidia-cuda-mps-control
30112 30119 30204

$ echo "set_default_active_thread_percentage 25" | nvidia-cuda-mps-control
25.0

$ echo "set_default_device_pinned_mem_limit 0 20G" | nvidia-cuda-mps-control

# Orderly teardown of daemon + all servers
$ echo quit | nvidia-cuda-mps-control
```

The full command set, from the manual page:

| Command | What it does |
|---|---|
| `get_server_list` | PIDs of all running MPS servers |
| `get_server_status <PID>` | Operational state of one server |
| `start_server -uid <UID>` | Start a server for a given user |
| `shutdown_server <PID> [-f]` | Stop a server; waits for clients to disconnect unless `-f` |
| `get_client_list <PID>` | PIDs of all clients connected to that server |
| `get_device_client_list <PID>` | Devices, and the client PIDs that enumerated each |
| `get_default_active_thread_percentage` | Current default (the shipped default is **100**) |
| `set_default_active_thread_percentage <pct>` | Set the default for **subsequently spawned** servers |
| `get_active_thread_percentage <PID>` | The limit in force on one server |
| `set_active_thread_percentage <PID> <pct>` | Change it for one server; affects **new clients only** |
| `get_default_device_pinned_mem_limit <dev>` | Current default pinned-memory limit for a device |
| `set_default_device_pinned_mem_limit <dev> <value>` | Set it; value is an integer plus `G` or `M` |
| `set_device_pinned_mem_limit <PID> <dev> <value>` | Override for one server; **new clients only** |
| `get_default_client_priority` / `set_default_client_priority <p>` | `0` = normal, `1` = below-normal |
| `terminate_client <server_PID> <client_PID>` | Cancel a specific client's outstanding GPU work |
| `ps [-p <PID>]` | Snapshot of current client processes |
| `quit [-t <TIMEOUT>]` | Shut down the daemon and all servers |

Three behaviours in that table are operational traps:

1. **`set_default_*` affects only *future* servers.** If a server is already running for your uid, changing the default does nothing to it. You must `shutdown_server` (or restart the daemon) for a new default to take effect. This is why the Kubernetes control daemon sets the defaults *immediately after* starting the daemon and before any client connects — see §5.
2. **`set_active_thread_percentage <PID>` affects only *new* clients of that server.** Existing clients keep the percentage they were admitted with. Provisioning is therefore admission-time, not continuous.
3. **A default set via the control interface is lost on `quit`.** Nothing persists. Any provisioning policy has to be re-applied on every daemon start, which is exactly why it belongs in a supervising process rather than a runbook.

The environment variables, with their defaults, also from the manual page:

| Variable | Default | Set on | Effect |
|---|---|---|---|
| `CUDA_MPS_PIPE_DIRECTORY` | `/tmp/nvidia-mps` | daemon **and** client | Where the control/server pipes live. A client finds MPS by finding this directory. |
| `CUDA_MPS_LOG_DIRECTORY` | `/var/log/nvidia-mps` | daemon | `control.log` and `server.log` land here. Your first debugging stop. |
| `CUDA_VISIBLE_DEVICES` | — | daemon or client | Restricts visible devices; accepts index or UUID |
| `CUDA_MPS_ACTIVE_THREAD_PERCENTAGE` | — | daemon, client, or per-context | On the daemon: the default for all spawned servers. On a client: constrains that client, and **cannot exceed** the daemon's value. |
| `CUDA_MPS_PINNED_DEVICE_MEM_LIMIT` | — | daemon or client | Caps device memory available to client processes. Valid from **CUDA 11.5**; images built against earlier CUDA ignore it. Format is per-device, e.g. `0=20G,1=20G`. |
| `CUDA_MPS_ENABLE_PER_CTX_DEVICE_MULTIPROCESSOR_PARTITIONING` | — | client | Allows the active-thread percentage to vary per CUDA context inside one client instead of being fixed for the process |
| `CUDA_MPS_CLIENT_PRIORITY` | — | client | Default client priority at startup |
| `CUDA_DEVICE_MAX_CONNECTIONS` | — | client | Preferred number of host→device work-submission connections. Under MPS these are a shared resource, so raising it per client reduces how many clients fit. |

The "cannot exceed the daemon's value" semantics on `CUDA_MPS_ACTIVE_THREAD_PERCENTAGE` is the one that makes provisioning trustworthy: a client can voluntarily take *less* than its allotment but cannot grant itself more. That asymmetry is what makes the ceiling a real bound rather than a suggestion, and it is the basis of the attribution argument in §6.

### 4 — Provisioning: what the two knobs actually mean

**Active thread percentage.** On Volta and later, this limits the number of SMs a client's work may occupy. Three properties matter:

- It is a **limit, not a reservation.** Setting 25% for four clients does not guarantee each client 25%; it guarantees none can exceed ~25%. If three clients are idle, the fourth still cannot use more than its 25% — so aggregate throughput under aggressive partitioning can be *lower* than under no partitioning when load is skewed. This is the single most surprising MPS behaviour in practice, and it is why "just set 100/N" is not automatically the right policy.
- Its **granularity is whole SMs.** A percentage is converted to an SM count and rounded, so on a 132-SM H100, 25% is ~33 SMs and the achievable percentages are quantised at ~0.76% steps. Do not design a policy around 1% differences.
- It **binds at client admission.** A client that connects while the limit is 50% keeps 50% even if you change the server's setting afterward.

**Pinned device memory limit.** Set via `set_default_device_pinned_mem_limit <dev> <value>` or `CUDA_MPS_PINNED_DEVICE_MEM_LIMIT`, this caps how much device memory a client may allocate. It is **enforced by the driver**: allocations beyond the cap fail with an out-of-memory error rather than succeeding and squeezing a neighbour. That is materially different from time-slicing, where the framebuffer is an uncapped free-for-all (lesson 07 §4).

The honest framing of what that buys, in three sentences: it is enforcement, not partitioning. There is still one physical framebuffer and one memory controller; the limit is arithmetic the driver applies to allocation requests, not an address-range wall; and it covers device memory allocations, so you should verify empirically that the specific allocation paths your framework uses (unified memory, CUDA arrays, IPC handles, external memory imports) are all subject to it on your CUDA version before you promise a tenant a hard cap. Also note the CUDA 11.5 floor: a client image built against CUDA 11.4 or earlier simply ignores the environment-variable form.

The comparison with MIG, precisely:

| | Time-slicing | MPS (Volta+) | MIG |
|---|---|---|---|
| Compute limit per client | none | active-thread % (limit, admission-time, SM-quantised) | dedicated SMs (partition) |
| Compute is a *reservation* | no | **no** — a cap only | **yes** |
| Memory cap per client | none | driver-enforced pinned-memory limit | dedicated framebuffer slice |
| Memory *protection* between clients | separate contexts | separate GPU address spaces | separate partitions |
| Memory bandwidth / L2 partitioned | no | no | **yes** |
| Reconfiguration cost | plugin restart | daemon restart | **node drain + GPU reset** |

The last row is the trade in one line: MPS gives you most of MIG's provisioning story with none of MIG's reconfiguration cost, and none of MIG's hardware guarantees.

### 5 — Enabling MPS in Kubernetes, and what the control daemon computes

The device-plugin config is the same shape as time-slicing's, selecting `mps` instead of `timeSlicing`. The two are **mutually exclusive** and the chosen mode applies to *all* GPUs on the node — the plugin does not support per-GPU sharing configuration (NVIDIA k8s-device-plugin README, *Shared Access to GPUs*).

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
      migStrategy: none          # MPS sharing is NOT supported by this
                                 # plugin on MIG-enabled devices.
      plugin:
        deviceIDStrategy: uuid
    sharing:
      mps:
        # As with time-slicing: true advertises nvidia.com/gpu.shared
        # instead of nvidia.com/gpu; false keeps the name and appends
        # "-SHARED" to the node's gpu.product label.
        renameByDefault: false
        resources:
          - name: nvidia.com/gpu     # the ONLY supported resource for MPS,
                                     # and only full GPUs
            devices: all
            replicas: 4              # 4 concurrent MPS clients per GPU
```

```bash
kubectl create -f mps-config.yaml
kubectl patch clusterpolicy/cluster-policy -n gpu-operator --type merge \
  -p '{"spec":{"devicePlugin":{"config":{"name":"mps-config","default":"any"}}}}'
```

Confirm the node's advertised state:

```bash
$ kubectl get node <node> -o json \
    | jq '.metadata.labels | with_entries(select(.key|test("nvidia.com")))'
{
  "nvidia.com/gpu.count": "1",
  "nvidia.com/gpu.replicas": "4",
  "nvidia.com/gpu.product": "NVIDIA-A100-SXM4-80GB-SHARED",
  "nvidia.com/gpu.sharing-strategy": "mps",
  "nvidia.com/mps.capable": "true"
}

$ kubectl describe node <node> | grep 'nvidia.com/gpu:'
  nvidia.com/gpu:  4        # 4 concurrent MPS clients over 1 physical GPU
```

`nvidia.com/mps.capable` is the documented label meaning "devices on this node are configured for MPS" — a cleaner selector for your operator than string-matching the product name.

**Now the part that makes MPS's attribution story different, read out of the plugin's source** (`cmd/mps-control-daemon/mps/daemon.go` in NVIDIA/k8s-device-plugin). The `nvidia-mps-control-daemon` DaemonSet does not just start `nvidia-cuda-mps-control -d`. On startup it:

1. Sets the GPU compute mode to `EXCLUSIVE_PROCESS` (constant `computeModeExclusiveProcess = computeMode("EXCLUSIVE_PROCESS")`). That is what forces every CUDA process on the node through MPS rather than letting some processes create their own contexts directly.
2. Creates the pipe and log directories, applying the SELinux label `system_u:object_r:container_file_t:s0` to the pipe directory so unprivileged workload containers can open it. The layout is per-resource: `<mps-root>/<resource-name>/pipe`, `<mps-root>/<resource-name>/log`, plus a `.started` sentinel and a shared `<mps-root>/shm`. With the GPU Operator, `<mps-root>` is a hostPath — commonly `/run/nvidia/mps`.
3. Launches `nvidia-cuda-mps-control -d`.
4. Computes and applies the per-client provisioning, immediately, before any client can connect:

```go
// Pinned device-memory limit, per device index:
limits[index] = fmt.Sprintf("%vM", totalMemory/replicas/1024/1024)
// …applied as:
d.EchoPipeToControl(fmt.Sprintf("set_default_device_pinned_mem_limit %s %s", index, limit))

// Active thread percentage:
replicasPerDevice := len(m.Devices()) / len(m.Devices().GetUUIDs())
return fmt.Sprintf("%d", 100/replicasPerDevice)
// …applied as:
d.EchoPipeToControl(fmt.Sprintf("set_default_active_thread_percentage %s", threadPercentage))
```

5. Exports exactly two environment variables into client containers:

```go
"CUDA_MPS_PIPE_DIRECTORY": d.PipeDir(),
"CUDA_MPS_LOG_DIRECTORY":  d.LogDir(),
```

6. Tails `control.log` into its own stdout, and health-checks the daemon by sending `get_default_active_thread_percentage` on an interval — a liveness probe that actually exercises the pipe rather than just checking a PID.

So on a single A100-80GB with `replicas: 4`, the plugin sets each client's ceiling to **25% active threads** and **~20 350 MiB pinned device memory** (80 GB reported by NVML, divided by 4, converted to MiB). Verify on your own node:

```bash
$ kubectl exec -n gpu-operator ds/nvidia-mps-control-daemon -c mps-control-daemon -- \
    sh -c 'echo get_default_active_thread_percentage | nvidia-cuda-mps-control'
25.0
$ kubectl exec -n gpu-operator ds/nvidia-mps-control-daemon -c mps-control-daemon -- \
    sh -c 'echo "get_default_device_pinned_mem_limit 0" | nvidia-cuda-mps-control'
20350M
```

*(Representative output — the exact container name and MiB figure depend on your Operator version and the card's NVML-reported total memory. The commands are the ones the daemon itself uses.)*

The structural picture, with the two ceilings marked:

```
  MPS ON KUBERNETES — WHO SETS WHAT

  ┌────────────────────────────────────────────────────────────────┐
  │ ConfigMap: sharing.mps.resources[0].replicas = 4               │
  └───────────────────────────┬────────────────────────────────────┘
              ┌───────────────┴────────────────┐
              ▼                                ▼
  ┌───────────────────────┐      ┌──────────────────────────────────┐
  │ nvidia-device-plugin  │      │ nvidia-mps-control-daemon (DS)   │
  │ advertises 4 devices: │      │  computeMode = EXCLUSIVE_PROCESS │
  │  GPU-abc::0 …::1      │      │  set_default_active_thread_      │
  │  GPU-abc::2 …::3      │      │      percentage 100/4  = 25      │
  │                       │      │  set_default_device_pinned_mem_  │
  │ Allocate() strips ::N │      │      limit 0  80GB/4 ≈ 20350M    │
  │  → NVIDIA_VISIBLE_    │      │  nvidia-cuda-mps-control -d      │
  │    DEVICES=GPU-abc    │      └──────────────┬───────────────────┘
  └───────────┬───────────┘                     │ pipe dir + log dir
              │                                 │ injected as env
              ▼                                 ▼
  ┌────────────────────────────────────────────────────────────────┐
  │ workload container                                             │
  │   CUDA_MPS_PIPE_DIRECTORY=/run/nvidia/mps/nvidia.com~gpu/pipe  │
  │   CUDA_MPS_LOG_DIRECTORY=/run/nvidia/mps/nvidia.com~gpu/log    │
  │   → CUDA runtime detects MPS, connects as a client,            │
  │     admitted with ceiling 25% SMs / 20350 MiB                  │
  └────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
  ┌────────────────────────────────────────────────────────────────┐
  │ THE SILICON — one GPU, one UUID, one framebuffer.              │
  │ DCGM keys metrics here. SAME FAN-OUT AS TIME-SLICING:          │
  │   GR_ENGINE_ACTIVE{UUID="GPU-abc",pod="a"} 0.98                │
  │   GR_ENGINE_ACTIVE{UUID="GPU-abc",pod="b"} 0.98  ← identical   │
  └────────────────────────────────────────────────────────────────┘
```

**On MPS + MIG — the correction.** The device plugin says MPS sharing "is currently not supported on devices with MIG enabled", and that is authoritative *for the device plugin*. It is not a hardware law. CUDA MPS can run inside a MIG compute instance, and the DRA driver models exactly that: its API has a distinct `MigDeviceSharing` type carrying an `MpsConfig`, alongside `GpuSharing` for full GPUs (`api/nvidia.com/resource/v1beta1/sharing.go` in kubernetes-sigs/dra-driver-nvidia-gpu). So the correct statement is: **the device-plugin path forces you to choose MIG *or* MPS per node; the DRA path models MPS on top of MIG devices** — subject to that driver's own alpha `MPSSupport` gate (lesson 09). Saying "they are mutually exclusive, full stop" is the version of this answer that falls over on the follow-up.

### 6 — Attribution under MPS: same fan-out, better estimate

Everything lesson 07 said about the metric applies here without modification, because it is a property of DCGM and the hardware, not of the sharing mode:

- Replicas are advertised as annotated IDs (`GPU-<uuid>::N`), so pod-resources gives you a distinct ID per pod and a correct co-residency set.
- DCGM programs counters per physical device, so `DCGM_FI_PROF_GR_ENGINE_ACTIVE` and `DCGM_FI_DEV_FB_USED` are single device values.
- `dcgm-exporter` with `--kubernetes-virtual-gpus` duplicates that single value across every holder. `sum` over-reports by exactly the number of holders. This is the same failure documented in `dcgm-exporter` #587 and #642 (lesson 07).

But one thing *is* different, and it is the reason to prefer MPS over time-slicing when you must estimate:

```
  WHY THE SAME ESTIMATOR IS BETTER UNDER MPS

  Under TIME-SLICING, a pod's true share s_i is UNBOUNDED in [0,1]:
     one greedy pod can take ~all the work.
     fair-share error  err_i = u × (1/N − s_i)
     worst case (s_1 = 1, N = 4):  |err_1| = 0.75 u   → 75% wrong

  Under MPS with active-thread % = p (= 100/N by default),
  a client's compute share is CAPPED by the driver:
     s_i ≤ p / Σp = p / (N·p) = 1/N     ← the ceiling IS the fair share
     and memory is capped at  M/N       ← driver-enforced

  So:  s_i ∈ [0, 1/N]     (not [0, 1])
       fair-share error   err_i = u × (1/N − s_i)  ∈  [0, u/N]
       ─────────────────────────────────────────────────────────
       Fair share can now only OVER-charge, never under-charge,
       and by at most u/N — i.e. at most one Nth of one GPU-hour.

  Concretely, N=4, u=0.98, $4.00/GPU-hr:
       max per-pod error = $4.00 × 0.98 / 4 = $0.98/hr
       and the SIGN is known: the idle pod is over-charged, the
       busy pod is charged at most its ceiling, never more.
```

That is a genuinely different epistemic position. Under time-slicing, "fair share" is an estimate with an unbounded, unsigned error. Under MPS with default provisioning it is an estimate with a **known bound and a known sign**, which is something you can write in a chargeback policy: *"tenants on MPS pools are billed at their provisioned ceiling; a tenant that under-uses its ceiling is over-charged by at most 1/N of a GPU-hour, and the remedy is to request a smaller ceiling."* That is a defensible policy. It is also a policy with the right incentive gradient, unlike time-slicing's, because asking for a smaller ceiling now lowers your bill.

Two caveats that must ride along:

1. **The ceiling is not a reservation.** A pod billed at its ceiling may have been unable to use it because a neighbour saturated the memory bus or the L2 — neither of which MPS partitions. So the bound is on *compute occupancy*, not on delivered throughput. Do not promise proportional performance.
2. **Per-PID data is still better if you have it.** Everything in lesson 07 §8 applies: probe `nvmlDeviceGetProcessUtilization` and `nvmlDeviceGetComputeRunningProcesses_v3` on your SKU, and note that MPS changes the process topology — the work is submitted by your client PIDs on Volta+, but on some driver/architecture combinations per-process accounting may attribute to the *server* process instead. **Re-verify per-PID resolution under MPS specifically; a field that resolved per-pod under time-slicing is not guaranteed to resolve per-pod under MPS on the same hardware.** Record which you got.

The PromQL is identical to lesson 07's, and the label is what changes:

```promql
# Deduplicate before aggregating — mandatory under any sharing mode.
DCGM_FI_PROF_GR_ENGINE_ACTIVE
  / on(UUID) group_left()
    count by (UUID) (DCGM_FI_PROF_GR_ENGINE_ACTIVE)

# What the exporter emits (lesson 10):
#   attribution="shared-estimate"           under time-slicing
#   attribution="shared-capped-estimate"    under MPS  ← bounded error
#   attribution="per-pid"                   where NVML resolves
#   attribution="exact"                     MIG / whole GPU
```

### 7 — The fault domain, precisely

This is the axis that disqualifies MPS for untrusted tenancy, and it is worth stating exactly rather than reaching for "worst of the three".

```
  FATAL FAULT PROPAGATION — CAUSAL CHAIN UNDER MPS

  t0  client B launches a kernel with an illegal address
      (or hits an uncorrectable ECC error on the device)

  t1  the GPU raises a fatal exception attributable to B's context

  t2  ── Volta+ containment ──
      the exception is contained to the set of clients sharing
      the OFFENDING GPU. Clients of the same server using only
      OTHER GPUs are unaffected. (Pre-Volta: every client of
      that server dies, regardless of which GPU it used.)

  t3  the MPS server enters a fault state; all clients sharing
      that GPU have their CUDA calls fail and are terminated.
      In Kubernetes terms: every pod holding a replica of that
      physical GPU dies at once.

  t4  the control daemon reaps the faulted server. A new server
      is spawned on the next client connection, INHERITING THE
      CURRENT DEFAULTS — which is why the provisioning has to be
      re-applied by a supervising process, not set once by hand.

  ── the second failure mode, which time-slicing does not have ──

  A client that launches a kernel that never terminates can
  require a FORCED SHUTDOWN of the MPS server, because the
  server creates and issues GPU work on behalf of its clients
  (nvidia-cuda-mps-control(1)). And terminating a client without
  synchronising its outstanding GPU work can leave the server and
  the other clients in an undefined state — hangs, unexpected
  failures, or corruption. So `kubectl delete pod` on an MPS
  client is not a clean operation by itself.
```

That last paragraph is the operationally expensive one, and it has a direct consequence for your platform: **pod deletion is not a safe primitive on an MPS pool unless your workload drains its GPU work on `SIGTERM`.** Note that the DRA driver's own demo manifests do exactly this — `trap 'exit 0' TERM; <work> & wait` — which is a small detail that tells you the authors hit this. Set a `terminationGracePeriodSeconds` long enough for your longest kernel plus a synchronise, and handle `SIGTERM` in your serving code.

Compared with the other two modes:

- **vs. time-slicing:** MPS's *device-fatal* propagation is comparable (both kill everything on the device), its *containment across GPUs* is better on Volta+ (scoped to the offending device rather than the whole server), and it adds a second, process-level failure surface — the daemon and server — that time-slicing does not have at all. Net: not simply "worse", but strictly *more* failure modes to operate.
- **vs. MIG:** no contest. A MIG instance is a hardware fault boundary; a fault in one instance is contained to that instance. That is why MIG is the answer for untrusted or SLO-bearing tenancy.

The conclusion is unchanged from the folklore even though the mechanism is more nuanced: **MPS is for trusted, cooperative workloads — your own services sharing a box — and never for hostile or arms-length multi-tenancy.**

### 8 — The three-way decision matrix

This is the artifact the 06/07/08 arc has been building toward. Read it as a decision table, not a feature list.

```
  ISOLATION / FAULT / MEMORY / ATTRIBUTION — THE COMPARISON MATRIX

                     │ TIME-SLICING  │ MPS (Volta+)      │ MIG
  ───────────────────┼───────────────┼───────────────────┼──────────────────
  Shares the …       │ TIME axis     │ OCCUPANCY axis    │ SILICON
  Mechanism          │ context       │ concurrent kernels│ hardware
                     │ preemption    │ via a shared      │ partitions
                     │               │ server + per-     │ (SM + FB + L2
                     │               │ client addr space │  + mem ctrl)
  ───────────────────┼───────────────┼───────────────────┼──────────────────
  Compute isolation  │ NONE          │ CAP (active-      │ RESERVATION
                     │               │ thread %, admis-  │ (dedicated SMs)
                     │               │ sion-time, not a  │
                     │               │ reservation)      │
  Memory cap         │ NONE          │ driver-ENFORCED   │ dedicated FB
                     │ (free-for-all)│ pinned-mem limit  │ slice
  Memory protection  │ per-context   │ per-client GPU    │ per-partition
                     │               │ address space     │
  Bandwidth / L2     │ shared        │ shared            │ PARTITIONED
                     │               │                   │
  Fault domain       │ the DEVICE    │ the DEVICE, plus  │ the INSTANCE
                     │               │ daemon+server as  │
                     │               │ extra failure     │
                     │               │ surfaces          │
  ───────────────────┼───────────────┼───────────────────┼──────────────────
  Distinct device    │ NO (annotated │ NO (annotated     │ YES
   UUID per tenant   │  ::N only)    │  ::N only)        │  (MIG-xxxx)
  DCGM metric        │ device-wide   │ device-wide       │ PER INSTANCE
   granularity       │ (fans out N×) │ (fans out N×)     │ (1:1)
  Attribution        │ estimate,     │ estimate, error   │ EXACT
   accuracy          │ error UNBOUND │ ≤ u/N and signed  │
  Naive sum() error  │ ×N            │ ×N                │ none
  ───────────────────┼───────────────┼───────────────────┼──────────────────
  Concurrency gain   │ none          │ YES for under-    │ yes, capped by
                     │ (serialised)  │ filling jobs      │ fixed slice size
  Reconfigure cost   │ plugin restart│ daemon restart    │ DRAIN + reset
  Clients per GPU    │ replicas (no  │ 16 (SM 3.5–6.0)   │ ≤ 7 instances
                     │  hard limit)  │ 48 (SM 7.0+)      │
  Maturity           │ GA, ubiquitous│ EXPERIMENTAL in   │ GA, ubiquitous
                     │               │ device plugin     │
                     │               │ since v0.15.0;    │
                     │               │ alpha gate in the │
                     │               │ DRA driver        │
  Works with MIG     │ yes (mixed    │ NOT via device    │ —
                     │  strategy)    │ plugin; YES via   │
                     │               │ DRA driver        │
  ───────────────────┴───────────────┴───────────────────┴──────────────────

  ONE-LINE DECISION:
    need isolation, an SLO, or a defensible per-tenant bill  → MIG
    trusted + jobs UNDER-FILL the GPU + want throughput      → MPS
    trusted + just need to overcommit an idle GPU cheaply    → time-slicing
    untrusted tenants                                        → MIG or nothing
```

### 9 — When MPS wins, when it loses, with numbers

A mode that only ever helps is a mode you have not measured. Two published accounts, both worth knowing because they disagree about scope in a useful way:

**Databricks, "Scaling Small LLMs with NVIDIA MPS."** Two identical inference engines on one H100 with MPS enabled, against a single-instance baseline, using the Qwen2.5 family. The reported conclusion is narrow and specific: MPS delivers meaningful throughput wins for **very small models (≈3B parameters and below) at short-to-medium context (under ~2k tokens)**, and for prefill-dominated workloads at that scale. Outside that envelope the win evaporates — which is exactly what §1 predicts, because a larger model or a longer context fills more of the SM array and leaves less headroom for a concurrent client. *(Located and corroborated via search this session; the blog itself is blocked by this environment's egress proxy, so treat the parameter and context thresholds as needing a spot-check before you quote them.)*

**Pebble, "NVIDIA MPS vs. Dedicated GPU Allocation for LLM Inference."** A controlled benchmark on Qwen3-4B-FP8 on A100-80GB in GKE with **60% SM allocation and 50% GPU memory per process**, two processes per GPU. The headline result: **~50% cost reduction for ~7.5% throughput loss** per process. Note the SM allocation sums to 120%, deliberately over-provisioned — because the active-thread percentage is a cap, not a reservation, over-provisioning lets a busy client borrow idle SMs while still bounding worst-case interference. *(Same fetch caveat.)*

Read those two together and the engineering rule is:

```
  MPS DECISION FLOW

  1. Under exclusive allocation, what is this job's SM_OCCUPANCY?
        ≥ ~60%  ──▶ MPS buys little. Do not take the fault-domain
                    risk. Use exclusive, or MIG if you need packing.
        < ~40%  ──▶ candidate. Continue.

  2. Do the jobs run CONCURRENTLY (overlapping request windows)?
        no  ──▶ time-slicing is simpler and equivalent here.
        yes ──▶ continue.

  3. Are all co-tenants TRUSTED and cooperative, and can whoever
     owns their SLOs sign off on a shared fault domain?
        no  ──▶ MIG. Stop.
        yes ──▶ continue.

  4. Does the fleet also need MIG on the same nodes?
        yes, and you are on the device plugin ──▶ separate node
             pools; the plugin cannot do MIG + MPS on one node.
        yes, and you are on DRA ──▶ possible via MigDeviceSharing
             with MpsConfig, behind the driver's alpha MPSSupport
             gate. Pilot it; do not assume it.
        no  ──▶ continue.

  5. Set the ceilings deliberately. Default is 100/N SMs and
     totalMem/N. Consider OVER-provisioning SMs (e.g. 60% each
     for 2 clients) so idle capacity is borrowable, while keeping
     the memory limit at or below totalMem/N so a neighbour
     cannot OOM you.

  6. Measure SM_OCCUPANCY and aggregate throughput before and
     after. GR_ENGINE_ACTIVE will NOT move — it is ~1.0 either
     way. If you only track that field you cannot prove the win.
```

Step 5 is where most MPS deployments are left on the table. The device plugin's defaults (`100/N`, `totalMem/N`) are a safe, symmetric starting point, not an optimum. If your workload mix is bursty and trusted, over-provisioning the compute ceiling while keeping the memory ceiling strict gets you most of the throughput of unpartitioned sharing with the neighbour-OOM protection that time-slicing lacks — which is arguably the best isolation-per-unit-of-configuration on offer anywhere in this triangle.

## Perspectives

**Throughput-engineer view.** MPS is bin-packing in the SM-occupancy dimension. Time-slicing already packs the time dimension well — there is little idle GPU-time between turns — but every turn wastes the SM capacity the resident job does not need, and for small-model inference that waste is the majority of the device. The mental shift is from "how do I keep the GPU busy?" to "how do I keep the *SMs* busy?", and the metric that answers the second question is `DCGM_FI_PROF_SM_OCCUPANCY`, not `GR_ENGINE_ACTIVE`. A team that optimises against the wrong one of those two fields will conclude time-slicing and MPS are equivalent, because on that field they are.

**Reliability / blast-radius view.** Count failure surfaces, not adjectives. Time-slicing has one shared resource a bad neighbour can wreck: the physical GPU. MPS has three: the GPU, the per-user server process, and the control daemon. The daemon is supervised (the k8s control daemon health-checks it over the pipe on an interval), which is good — but a supervised process that must be restarted still means all clients on that node reconnect, and reconnection re-runs admission with whatever the current defaults are. Add to that the "terminating a client without synchronising outstanding GPU work can leave the server and other clients in an undefined state" property, and MPS is a mode whose *operational* surface is meaningfully larger than time-slicing's even where its *fault containment* is better.

**FinOps / attribution view.** MPS is the only sharing mode where the estimate has a proof attached. Under time-slicing you say "we split by fair share and the error is whatever the load skew happens to be." Under MPS with enforced ceilings you say "we bill at the provisioned ceiling; the error is bounded by 1/N of a GPU-hour, it can only over-charge, and a tenant who wants a lower bill can request a lower ceiling." The second statement survives a chargeback dispute. It also has the correct incentive gradient — under time-slicing, asking for more replicas lowers your per-replica share and therefore your bill; under ceiling-based MPS billing, asking for more capacity raises it. **When a sharing mode's billing model has a perverse incentive gradient, that is a design bug in the platform, not an accounting detail** — and this is the one place in the triangle where you get to fix it without moving to MIG.

**Real-production-adopter view.** Both published accounts above narrow MPS's win envelope rather than widening it: small models, short contexts, prefill-heavy, two processes rather than eight. That pattern is worth generalising — MPS's benefit is proportional to the *headroom* each job leaves, and headroom shrinks fast as models and contexts grow. An MPS policy written for a 1.5B classifier will not survive that team shipping an 8B model, so the policy needs to be re-derived when the workload changes, and your dashboard needs to make the occupancy headroom visible so someone notices.

**Comparative decision-maker view.** In a design review nobody has time to re-derive three mechanisms. What they need is the §8 matrix, and — more than the matrix — the two rows at the bottom of it that most comparisons omit: **reconfiguration cost** and **attribution accuracy**. Those two rows are what turn a mechanism comparison into a platform decision, because they are the rows that price the operational and financial consequences of the choice rather than just describing the silicon.

## Real-world use cases

- **NVIDIA k8s-device-plugin README, *Shared Access to GPUs* → *With CUDA MPS*.** **Fetched and read this session at release v0.19.2.** The vendor's own maturity statement, quoted precisely: "As of v0.15.0 of the device plugin, MPS support is considered experimental", and "Sharing with MPS is currently not supported on devices with MIG enabled", and the scope limit "the only supported resource available for MPS are `nvidia.com/gpu` resources and only with full GPUs". It also states the substantive difference from time-slicing: MPS "does space partitioning and allows memory and compute resources to be explicitly partitioned and enforces these limits per workload", with each client limited to "an equal fraction (1/10)" — in that example's terms — of the device's memory and compute. **Correction to the previous version of this lesson:** the plugin's current release line is v0.19.x, not v0.17.1.
- **NVIDIA k8s-device-plugin, `cmd/mps-control-daemon/mps/daemon.go` and `root.go`.** **Fetched and read this session.** The implementation of everything in §5: `computeModeExclusiveProcess`, the `set_default_device_pinned_mem_limit`/`set_default_active_thread_percentage` commands with their exact format strings, the `totalMemory/replicas/1024/1024` and `100/replicasPerDevice` arithmetic, the `<root>/<resource>/{pipe,log,.started}` layout, the SELinux label `system_u:object_r:container_file_t:s0`, the two injected environment variables, and the `get_default_active_thread_percentage` health check. If you want to know what your cluster is actually doing to your GPUs, this file is the answer.
- **`nvidia-cuda-mps-control(1)` manual page.** **Fetched and read this session.** The complete command reference and environment-variable table in §3, the `set_default_*` "affects only the next server" semantics, the client limits (16 for SM 3.5–6.0, 48 for SM 7.0), the CUDA 11.5 floor on `CUDA_MPS_PINNED_DEVICE_MEM_LIMIT`, and the two operational warnings quoted in §7 — that a runaway kernel may require a forced server shutdown, and that terminating a client without synchronising its GPU work can leave the server and other clients in an undefined state.
- **Databricks, "Scaling Small LLMs with NVIDIA MPS."** A production platform's measured envelope: two identical inference engines on one H100, Qwen2.5 family, with the win confined to very small models (≈≤3B) at short-to-medium context. The value of this reference is that it is a *negative* result outside a narrow band, which is far more useful for policy than a success story. *(Corroborated via search this session; the domain is blocked by this environment's proxy — spot-check the thresholds.)*
- **Pebble, "NVIDIA MPS vs. Dedicated GPU Allocation for LLM Inference."** Qwen3-4B-FP8 on A100-80GB in GKE, 60% SM and 50% memory per process, two processes per GPU → ~50% cost reduction for ~7.5% throughput loss. The deliberate 120%-sum SM over-provisioning is the practical illustration of "the active-thread percentage is a cap, not a reservation". *(Same fetch caveat.)*
- **kubernetes-sigs/dra-driver-nvidia-gpu, `api/nvidia.com/resource/v1beta1/sharing.go` and `pkg/featuregates/featuregates.go`.** **Fetched and read this session.** The `GpuSharing` / `MigDeviceSharing` split that proves MPS-on-MIG is modelled (correcting the "mutually exclusive, full stop" claim), the full `MpsConfig` field set (`defaultActiveThreadPercentage`, `defaultPinnedDeviceMemoryLimit`, `defaultPerDevicePinnedMemoryLimit`, `multiUser`), and the fact that `MPSSupport` and `TimeSlicingSettings` are registered as **Alpha, `Default: false`** driver feature gates — i.e. the DRA path to MPS is opt-in and pre-beta. That last fact is the honest counterweight to "DRA fixes sharing configuration".

## Worked example — the same GPU, three modes, one reconciliation

Setup: one A100-80GB, four identical inference pods each serving a ~3B model whose kernels occupy roughly 25% of the SM array under exclusive allocation. Node rate $16.00/hr for 4 GPUs → **$4.00 per physical GPU-hour**. Requests are concurrent (overlapping windows), and all four pods belong to trusted internal services.

### Step 1 — throughput under each mode

| Mode | Config | Per-pod r/s | Aggregate r/s | `GR_ENGINE_ACTIVE` | `SM_OCCUPANCY` |
|---|---|---|---|---|---|
| Exclusive (baseline, 1 pod) | `replicas: 1` | 240 | 240 | 0.98 | 0.24 |
| Time-slicing | `replicas: 4` | ~58 | **~232** | 0.99 | 0.24 |
| MPS, default ceilings | `replicas: 4` → 25% SMs, 20 350 MiB each | ~205 | **~820** | 0.99 | 0.91 |
| MPS, over-provisioned | 4 clients × 40% SMs, mem still 20 350 MiB | ~215 | **~860** | 0.99 | 0.94 |
| MIG | 4 × `2g.20gb` | ~225 | **~900** | per-instance | per-instance |

*(These are illustrative figures consistent with the published accounts in §9, not captured measurements. Measure on your own SKU and workload before quoting any of them — the whole point of §9 is that MPS's win envelope is narrow and workload-specific.)*

Read the table by column, not by row:

- **`GR_ENGINE_ACTIVE` is ~0.99 in every software-sharing row.** It cannot distinguish 232 r/s from 860 r/s. This is the single most important operational lesson in the table.
- **`SM_OCCUPANCY` moves from 0.24 to 0.91.** That is the field that proves the change worked, and the field your dashboard must carry before you attempt this migration.
- **Time-slicing aggregate ≈ one pod's throughput.** 232 vs a 240 baseline: the ~3% shortfall is context-switch overhead. Time-slicing did exactly what it promises — it kept the GPU busy — and delivered no additional work.
- **MPS is ~3.5× time-slicing here, and MIG is slightly ahead of MPS.** MIG wins on raw throughput because its partitions are hardware and there is no server in the path; MPS wins on flexibility, because a `2g.20gb` MIG slice cannot lend idle SMs to a busy neighbour and an MPS ceiling can (when over-provisioned).

### Step 2 — cost per pod under each mode

The allocated figure is the same in all three software cases, because allocation is uniform:

```
  allocated cost/pod/hr = $4.00 / 4 = $1.00       (all three modes)
```

The *utilised* figure and its honesty label differ. Take the MPS over-provisioned row, and suppose per-PID `smUtil` comes back as 38 / 36 / 12 / 4 percent for pods a/b/c/d.

```
  MODE A — TIME-SLICING, fair share (attribution="shared-estimate")
    device u = 0.99 ;  N = 4
    per-pod utilised share = 0.99/4 = 0.2475
    per-pod utilised cost  = $4.00 × 0.2475 = $0.99   (all four identical)
    total = $3.96 = $4.00 × 0.99   ✓ reconciles
    ERROR BOUND: none. If one pod did 95% of the work it is
                 under-billed by up to 0.75 × $3.96 ≈ $2.97/hr.

  MODE B — MPS, billed at the provisioned ceiling
                                (attribution="shared-capped-estimate")
    ceiling per pod = 40% SMs, but Σ ceilings = 160% > 100%, so
    normalise by the sum:  0.40 / 1.60 = 0.25 of the device
    per-pod utilised cost = $4.00 × 0.99 × 0.25 = $0.99
    total = $3.96   ✓ reconciles
    ERROR BOUND: signed and bounded. A pod cannot exceed its
                 ceiling, so it can only be OVER-charged, by at
                 most $0.99/hr (one Nth of one GPU-hour).

  MODE C — MPS, per-PID smUtil (attribution="per-pid")
    Σ smUtil = 38 + 36 + 12 + 4 = 90
      share(a) = 38/90 = 0.4222   cost = $4.00 × 0.99 × 0.4222 = $1.672
      share(b) = 36/90 = 0.4000   cost =                        $1.584
      share(c) = 12/90 = 0.1333   cost =                        $0.528
      share(d) =  4/90 = 0.0444   cost =                        $0.176
                        ──────                                 ───────
                        1.0000                          total = $3.960 ✓

    Error that Mode B introduced, per pod:
      pod a  $0.99 vs $1.672  →  under-billed $0.68  (41%)
      pod b  $0.99 vs $1.584  →  under-billed $0.59  (38%)
      pod c  $0.99 vs $0.528  →  over-billed  $0.46  (88%)
      pod d  $0.99 vs $0.176  →  over-billed  $0.81  (463%)

  MODE D — MIG, 4 × 2g.20gb (attribution="exact")
    Four distinct MIG UUIDs; DCGM reports per instance; pod-resources
    reports the same MIG UUID. No fan-out, no estimate.
      pod_share = 2/7 of compute per instance (2 compute slices of 7)
      allocated cost/pod/hr = $4.00 × (2/7) = $1.143
        …and 4 × 2g.20gb consumes 8 compute slices > 7 available, so
        this configuration is NOT VALID on a 7-slice card — you would
        run 3 × 2g.20gb (6 slices) + 1 × 1g.10gb, i.e. UNEVEN shares.
      That stranding is MIG's real cost, and it is why the exact
      attribution comes with a capacity-planning tax.
```

That last block is the honest MIG footnote most comparisons skip: exactness is purchased with a fixed partition menu, and a fixed menu strands capacity whenever your tenant count does not tile it. **The attribution-accuracy column and the stranded-capacity column move in opposite directions**, which is precisely the trade a platform engineer is paid to make.

### Step 3 — the reconciliation check

Both identities from lesson 07 must hold, and under MPS there is a third worth asserting:

```
  IDENTITY 1 — allocation conservation (exact)
    Σ_pods allocated_share + unallocated_share = 1.00 per physical GPU
    Here: 4 × 0.25 + 0 = 1.00                                    ✓
    In GPU-hours over window W:
      Σ attributed GPU-hours + idle GPU-hours = (#GPUs) × W

  IDENTITY 2 — utilisation conservation (within scrape jitter)
    Σ_pods utilised_share = device utilisation u
    Mode C: (0.4222+0.4000+0.1333+0.0444) × 0.99 = 0.99          ✓
    PromQL form, should sit near 1 for every UUID:
      sum by (UUID) (gpu_util_attributed)
        / on(UUID) group_left()
          max by (UUID) (DCGM_FI_PROF_GR_ENGINE_ACTIVE)

  IDENTITY 3 — MPS ceiling conservation (the new one)
    For every pod on an MPS device:
      per-PID measured share  ≤  normalised ceiling share + ε
    Mode C vs Mode B: a=0.4222 vs 0.25 → VIOLATION by 0.17.
    A violation means one of three things, all worth alerting on:
      (a) the ceilings were never applied (daemon restarted and the
          defaults were not re-set — see §3 trap 3, §7 t4);
      (b) the client connected before the ceiling was set;
      (c) your per-PID sampling window does not align with the
          DCGM window and you are comparing apples to oranges.
    Assert it. It is the cheapest possible detector for "MPS
    provisioning silently stopped working", which is otherwise
    invisible — throughput just quietly gets less fair.
```

Identity 3 is the reason to bother computing both Mode B and Mode C even if you only bill from one of them: the *disagreement* between the ceiling and the measurement is a health signal about your MPS provisioning, and nothing else in the stack reports it.

### Step 4 — the caveats that ship with the number

1. **Fault domain.** All four pods are one fault domain. An illegal-address error in any of them puts the MPS server into a fault state and kills all four. Every cost report for an MPS pool should carry the co-residency set alongside the dollars, for the same reason lesson 07 argued it: the bill and the incident are the same physical fact.
2. **Deletion is not free.** `kubectl delete pod` on an MPS client without a `SIGTERM` drain can leave the server and siblings in an undefined state. Ship a `SIGTERM` handler and a `terminationGracePeriodSeconds` that covers your longest kernel.
3. **The ceiling is not a reservation, and bandwidth is not partitioned.** A pod billed at its ceiling may not have been able to use it because a neighbour saturated HBM. Do not promise proportional performance from a proportional bill.
4. **Verify per-PID resolution under MPS specifically.** MPS changes the process topology. A DCGM or NVML field that resolved per-pod under time-slicing on this exact hardware may attribute to the server process under MPS. Probe it, record the result with driver version and SKU, and let the `attribution` label tell the truth.

## Practice

For the [Per-pod GPU attribution](../practice/per-pod-attribution/README.md) deliverable. A single sub-$1/hr GPU rental is sufficient for all of this except the MIG row (which lesson 06 already covered on an A100/H100 session).

1. **Enable MPS and capture the provisioning.** Apply the `mps` ConfigMap from §5, patch the ClusterPolicy, and confirm `nvidia.com/gpu.sharing-strategy: mps`, `nvidia.com/mps.capable: true`, and N allocatable over one physical GPU. Then — the part that matters — `kubectl exec` into the `nvidia-mps-control-daemon` pod and query the daemon directly for `get_default_active_thread_percentage` and `get_default_device_pinned_mem_limit 0`. Verify they equal `100/replicas` and `totalMemory/replicas` in MiB. Record the numbers and the device-plugin image tag.

2. **Prove the memory limit is enforced (the difference from time-slicing).** Run a single pod on the MPS device and allocate device memory in a loop past the computed ceiling. Capture the out-of-memory failure at the ceiling rather than at the card's physical capacity. Then repeat the identical test under time-slicing (lesson 07's config) and capture the allocation succeeding well past `totalMemory/replicas`. That contrast — same code, two modes, two failure points — is the artifact that proves MPS does space partitioning and time-slicing does not.

3. **Throughput comparison on the occupancy axis.** Run the same set of N small concurrent jobs (each sized to under-fill the GPU) under time-slicing, then under MPS, on the same physical GPU. Record aggregate throughput **and both** `DCGM_FI_PROF_GR_ENGINE_ACTIVE` and `DCGM_FI_PROF_SM_OCCUPANCY` for each mode. The acceptance bar is that you can show `GR_ENGINE_ACTIVE` barely moving while `SM_OCCUPANCY` and aggregate throughput both jump — that pairing is the proof, and either metric alone is not.

4. **Tune the ceilings and show it matters.** Re-run the MPS case with the active-thread percentage over-provisioned (e.g. `set_default_active_thread_percentage 40` for 4 clients, sum 160%) after restarting the daemon so the new default takes effect. Compare aggregate throughput against the default `100/N`. Document whether over-provisioning helped on your workload and by how much. Note explicitly that you had to restart the daemon, and why (`set_default_*` affects only subsequently spawned servers).

5. **Document the fault domain, and optionally demonstrate it.** Write the causal chain from §7 in your own words: fatal exception → Volta+ containment to clients sharing the offending GPU → server fault state → all co-resident clients terminated → daemon reaps and respawns with current defaults. If you are willing to break the node, force a CUDA illegal-address error in one client and capture the sibling clients failing together, plus the `control.log`/`server.log` excerpt from `CUDA_MPS_LOG_DIRECTORY`. Then verify Identity 3 from the worked example still holds after the respawn — i.e. that the ceilings came back.

6. **Verify the maturity and compatibility claims yourself, with versions.** Check your installed device-plugin release's README (or `docs/`) for the MPS experimental statement and the MIG-incompatibility statement, and record the version you checked. Separately, note that the DRA driver's `MPSSupport` gate is alpha/default-off, so the DRA path to MPS requires explicitly enabling it. Version-stamp both findings; the whole point is that this status moves.

7. **Write the decision matrix.** Produce the §8 matrix as your own artifact — mechanism, three isolation axes, memory-bandwidth partitioning, fault domain, distinct UUID, DCGM granularity, attribution accuracy with the error bound, reconfiguration cost, client limits, maturity, MIG compatibility, and when-to-use. This is the artifact the 06/07/08 arc has been building toward and the cold-answer reference for the module's interview probe.

**Acceptance:** (a) the daemon-queried ceilings matching `100/replicas` and `totalMemory/replicas`; (b) the two-mode memory-limit contrast showing enforcement under MPS and no enforcement under time-slicing; (c) an aggregate-throughput comparison for under-filling concurrent jobs with **both** `GR_ENGINE_ACTIVE` (flat) and `SM_OCCUPANCY` (moved) as evidence; (d) a written fault-domain causal chain; (e) a version-stamped note on MPS's experimental status and the device-plugin-vs-DRA MIG-compatibility difference; and (f) the completed three-way decision matrix including the attribution-accuracy row with its error bound.

## Common pitfalls

1. **Describing MPS as "one shared CUDA context".** That is pre-Volta MPS. On Volta and later, clients submit work directly to the hardware without passing through the server and each client owns its own GPU address space. Getting this wrong makes you unable to answer the natural follow-up ("so what did Volta change?") and leads you to understate MPS's memory protection.
2. **Saying MPS gives no memory isolation.** It gives memory *protection* (separate per-client address spaces) and a driver-*enforced* pinned-device-memory cap. What it does not give is a hardware memory partition, or partitioned memory bandwidth and L2. State the three separately; collapsing them into "no isolation" is the error, and so is collapsing them into "it's like MIG".
3. **Treating the active-thread percentage as a reservation.** It is a cap, applied at client admission, quantised to whole SMs. Setting `100/N` guarantees nobody exceeds their share; it does not guarantee anybody gets it, and under skewed load it can reduce aggregate throughput below unpartitioned sharing. Over-provisioning the sum above 100% is often the better policy for trusted, bursty tenants.
4. **Changing a default and expecting it to apply.** `set_default_active_thread_percentage` and `set_default_device_pinned_mem_limit` affect only *subsequently spawned* servers, `set_active_thread_percentage <PID>` affects only *new clients*, and every default is lost on `quit`. If provisioning "stopped working" after a node event, this is almost always why — and Identity 3 in the worked example is how you detect it.
5. **Claiming MPS and MIG are mutually exclusive, full stop.** The exclusivity is a *device-plugin* limitation, documented as such. The DRA driver models MPS on MIG devices via `MigDeviceSharing` with an `MpsConfig`. Say "the device-plugin path forces one or the other per node; the DRA path models both, behind an alpha gate."
6. **Deploying MPS for jobs that already saturate the GPU.** If each job needs the whole SM array, MPS buys nothing over time-slicing and you have taken on a shared fault domain plus a daemon and a server for zero upside. Measure `SM_OCCUPANCY` under exclusive allocation first; below ~40% is a candidate, above ~60% is not.
7. **Proving the migration worked with `GR_ENGINE_ACTIVE`.** It is ~1.0 before and after. You will conclude nothing changed while throughput quadrupled — or, worse, conclude that time-slicing was already fine. Carry `SM_ACTIVE` and `SM_OCCUPANCY`.
8. **Reusing lesson 07's verified per-PID field without re-verifying it under MPS.** The attribution *conclusion* is the same, but the process topology is not. A field that resolved per-pod under time-slicing may attribute to the MPS server process instead. Re-probe, and change the `attribution` label if the answer changed.
9. **Deleting MPS client pods without a drain.** Terminating a client without synchronising its outstanding GPU work can leave the server and other clients in an undefined state. `trap 'exit 0' TERM` plus a real `SIGTERM` handler and an adequate grace period is not defensive over-engineering here; it is the documented requirement.
10. **Treating "experimental" as either disqualifying or meaningless.** Both extremes fail the interview. The defensible position: the vendor labels device-plugin MPS experimental and the DRA driver gates it alpha-off, published production accounts exist but with narrow measured envelopes, therefore you deploy it deliberately on a carved-out trusted pool with the fault-domain risk signed off — not as a fleet default.

## Self-check

- **MPS vs time-slicing — what is the mechanistic difference, precisely?** *Answer:* They pack different axes. Time-slicing multiplexes *contexts in time*: the GPU's channel scheduler preempts one context and restores another, so clients take turns and a client owns the whole SM array during its slice — which means an under-filling job leaves most of the array idle at every instant. MPS shares *occupancy*: a control daemon spawns a per-user server that provisions and arbitrates, and on Volta and later the clients submit work **directly to the hardware work queues** (not through the server, which is the pre-Volta model) with **each client owning its own GPU address space**. Kernels from different processes are therefore resident and executing on different SMs simultaneously. Consequence: four jobs each needing 25% of the SMs run at ~1× aggregate under time-slicing and up to ~4× under MPS. Consequence for observability: `GR_ENGINE_ACTIVE` is ~1.0 in both cases and cannot tell them apart; `SM_ACTIVE`/`SM_OCCUPANCY` can.

- **What does MPS actually enforce per client, how, and what does it not enforce?** *Answer:* Two ceilings, both set on the server via the control daemon and both enforced by the driver. (1) `CUDA_MPS_ACTIVE_THREAD_PERCENTAGE` / `set_default_active_thread_percentage` caps the SMs a client's work may occupy — a *limit*, not a reservation, quantised to whole SMs, bound at client admission, and a client-set value can only narrow it below the daemon's value, never exceed it. (2) `CUDA_MPS_PINNED_DEVICE_MEM_LIMIT` / `set_default_device_pinned_mem_limit` caps device-memory allocation; requests beyond it fail rather than squeezing a neighbour. Valid from CUDA 11.5 in its environment-variable form. It does **not** enforce a hardware memory partition, memory-bandwidth or L2 partitioning, or any performance guarantee — and it is not a reservation, so a pod may be unable to use its ceiling because a neighbour saturated HBM. In Kubernetes the device plugin's MPS control daemon computes these as `100/replicas` percent and `totalMemory/replicas` MiB, and also forces compute mode to `EXCLUSIVE_PROCESS` so all CUDA processes go through MPS.

- **Why is MPS's attribution *estimate* better than time-slicing's, even though the metric problem is identical?** *Answer:* The fan-out is the same — annotated `GPU-<uuid>::N` IDs above the plugin, one device-level DCGM counter below it, duplicated per holder, `sum` over-reporting by exactly N. What differs is the *support* of the unknown share. Under time-slicing a pod's true share `s_i` can be anything in [0,1], so the fair-share error `u × (1/N − s_i)` is unbounded and unsigned; a single greedy pod can be under-billed by 0.75 of a GPU-hour at N=4. Under MPS with enforced ceilings, `s_i ≤ ceiling_i / Σceilings`, so with symmetric default ceilings `s_i ≤ 1/N` — the ceiling *is* the fair share. The error is therefore bounded by `u/N` and **can only over-charge**. That converts the estimate into a billable policy ("we bill at your provisioned ceiling; over-charge is at most 1/N of a GPU-hour; request a lower ceiling to lower your bill") with a correct incentive gradient, which time-slicing's fair share does not have.

- **Why is MPS disqualified for hostile or arms-length multi-tenancy, and what exactly is the fault behaviour?** *Answer:* Because the fault domain is the device plus two extra process-level failure surfaces, and none of it is a hardware boundary. A fatal GPU exception raised by one client is, on Volta and later, contained to the set of clients sharing the *offending GPU* — narrower than pre-Volta, where every client of that server died regardless of which GPU it used — but it still puts the server into a fault state and terminates every client on that device, i.e. every co-resident pod dies together. Beyond that, a client launching a kernel that never terminates can require a forced shutdown of the server (because the server issues GPU work on the clients' behalf), and terminating a client without synchronising its outstanding GPU work can leave the server and other clients in an undefined state — hangs, unexpected failures, or corruption. So one tenant's bug becomes every co-tenant's outage, and `kubectl delete pod` is not by itself a clean operation. Untrusted or SLO-bearing tenants need MIG's hardware fault boundary.

- **Is MPS production-ready — what do you actually tell a hiring manager?** *Answer:* Three clauses, in this order. (1) *Vendor status:* NVIDIA's device plugin has marked MPS sharing experimental since v0.15.0 and still does in the current v0.19.x line, restricts it to `nvidia.com/gpu` full GPUs, and does not support it on MIG-enabled devices; in the DRA driver, `MPSSupport` is an alpha, default-off feature gate. (2) *Field evidence:* it is genuinely used in production, but the published envelopes are narrow — Databricks reports wins confined to very small models at short-to-medium context on an H100; a controlled Pebble benchmark reports ~50% cost reduction for ~7.5% throughput loss at two processes per A100-80GB with 60% SM / 50% memory each. (3) *Therefore:* deploy it deliberately on a node pool carved out for trusted, concurrent, under-filling workloads, with the fault-domain risk signed off by whoever owns those tenants' SLOs, with `SM_OCCUPANCY` on the dashboard to prove it worked and Identity 3 asserted to prove the ceilings are still applied — not as a fleet default, and not dismissed either.

- **Are MPS and MIG mutually exclusive?** *Answer:* Not at the hardware level — the exclusivity is a device-plugin limitation. NVIDIA's k8s-device-plugin documents that "sharing with MPS is currently not supported on devices with MIG enabled", so on the device-plugin path you route MIG tenants and MPS tenants to different node pools. But CUDA MPS can run inside a MIG compute instance, and the NVIDIA DRA driver models this explicitly: its API carries a `MigDeviceSharing` type with an `MpsConfig`, distinct from `GpuSharing` for full GPUs. So the accurate answer is "mutually exclusive per node on the device-plugin path; modelled and supported on the DRA path, behind that driver's alpha `MPSSupport` gate — pilot it rather than assuming it."

- **What are the client-count limits, and why do they matter for capacity planning?** *Answer:* From the `nvidia-cuda-mps-control(1)` manual page: compute capability SM 3.5 through SM 6.0 is limited to 16 clients per GPU at a time; SM 7.0 raises that to 48. They matter because they are an independent ceiling on `replicas` that has nothing to do with memory or SMs — you can be well within your framebuffer budget and still fail to admit a client. They also interact with `CUDA_DEVICE_MAX_CONNECTIONS`: work-submission connections are a shared resource under MPS, so raising it per client reduces how many clients fit. Verify the limit for the newest architectures against your driver's documentation rather than assuming 48 scaled again.

## Connections & what's next

This closes the three-lesson sharing arc. MIG (06) partitions silicon and gives exact attribution at the price of a drain and a fixed partition menu that strands capacity. Time-slicing (07) packs time for free and gives an unbounded estimate. MPS (08) packs occupancy, enforces bounded ceilings, and gives a *bounded, signed* estimate — the best attribution available without hardware partitioning — at the price of a shared fault domain and two extra process-level failure surfaces. The §8 matrix is the cold answer to the module's interview probe and the branch logic your capstone cost operator runs off `nvidia.com/gpu.sharing-strategy`, which takes exactly the values `none`, `time-slicing`, and `mps`.

From here the module moves from *sharing one GPU* to *requesting devices properly*. Lesson 09 covers the DRA driver, where the sharing decision stops being a node-wide ConfigMap and becomes a per-claim `GpuConfig` — `sharing.strategy: MPS` with `mpsConfig.defaultActiveThreadPercentage` and `defaultPinnedDeviceMemoryLimit` set *by the workload that wants them*, which is exactly the per-workload ceiling policy §6 argued for. It also introduces DRA's consumable-capacity model, which is the first mechanism in this module that gives a *shared* device a per-share identity in the API — the closest thing to a structural fix for the fan-out. Carry this lesson's conclusions forward unchanged: DRA changes who asks for what and how it is fenced, not the fact that a time-sliced or MPS-shared physical GPU reports one DCGM number.

Next: **[04.9 · DRA driver (real install) + multi-tenancy quotas](09-dra-driver-and-quotas.md)** — from device-plugin ConfigMaps to `ResourceClaim`/`DeviceClass` objects, and namespace-level GPU quotas.

## References & further reading

**Primary sources**

- [NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) — **fetched and read this session at release v0.19.2.** The `sharing.mps` schema, the MPS-experimental statement ("As of v0.15.0 of the device plugin, MPS support is considered experimental"), the MIG-incompatibility statement, the full-GPU-only scope limit, the mutual exclusivity with time-slicing, the "same sharing method applies to all GPUs on a node" constraint, and the space-partitioning description ("allows memory and compute resources to be explicitly partitioned and enforces these limits per workload"). **Correction:** the current release line is v0.19.x; the earlier version of this lesson cited v0.17.1, which is a stale number that appears in the README's own install examples.
- [NVIDIA/k8s-device-plugin — `cmd/mps-control-daemon/mps/daemon.go`, `cmd/mps-control-daemon/mps/root.go`](https://github.com/NVIDIA/k8s-device-plugin) — **fetched and read this session.** Everything in §5: `computeModeExclusiveProcess`, the exact control-command format strings, the `totalMemory/replicas/1024/1024` MiB and `100/replicasPerDevice` arithmetic, the `<root>/<resource>/{pipe,log,.started}` plus shared `shm` layout, the SELinux label, the two injected environment variables, and the `get_default_active_thread_percentage` health check.
- [`nvidia-cuda-mps-control(1)` manual page](https://man.archlinux.org/man/extra/nvidia-utils/nvidia-cuda-mps-control.1.en) — **fetched and read this session.** The complete command reference and environment-variable defaults in §3; the "affects only the next server / only new clients" semantics; client limits (16 clients per GPU for SM 3.5–6.0, 48 for SM 7.0); the CUDA 11.5 floor on `CUDA_MPS_PINNED_DEVICE_MEM_LIMIT`; and the two operational warnings in §7 about runaway kernels requiring forced server shutdown and unsynchronised client termination leaving the server in an undefined state.
- [NVIDIA Multi-Process Service (MPS) documentation](https://docs.nvidia.com/deploy/mps/) — the authority on the Volta architectural change quoted in §2: "Volta MPS clients submit work directly to the GPU without passing through the MPS server" and "each Volta MPS client owns its own GPU address space instead of sharing GPU address space with all other MPS clients", plus the error-containment scoping (a fatal exception from a Volta MPS client is contained to the subset of GPUs shared with the faulting client). *(This domain is blocked by the current environment's egress proxy; the quoted statements were corroborated across multiple independent sources this session. Read the MPS overview directly when you have access — sections on provisioning and error containment are the ones that matter.)*
- [kubernetes-sigs/dra-driver-nvidia-gpu — `api/nvidia.com/resource/v1beta1/sharing.go`, `gpuconfig.go`, `pkg/featuregates/featuregates.go`](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu) — **fetched and read this session.** The `GpuSharing` / `MigDeviceSharing` split (proving MPS-on-MIG is modelled), the full `MpsConfig` field set, the `TimeSliceInterval` enum with its integer mapping (`Default=0, Short=1, Medium=2, Long=3` — the same values as `nvidia-smi compute-policy --set-timeslice`), and the registration of `MPSSupport` and `TimeSlicingSettings` as **Alpha, `Default: false`** driver feature gates.
- [NVIDIA/dcgm-exporter — `etc/default-counters.csv`](https://github.com/NVIDIA/dcgm-exporter) — **fetched and read this session.** The exact field names and help text relied on in §1 and §9: `DCGM_FI_PROF_GR_ENGINE_ACTIVE` ("Ratio of time the graphics engine is active"), `DCGM_FI_PROF_SM_ACTIVE` ("The ratio of cycles an SM has at least 1 warp assigned"), `DCGM_FI_PROF_SM_OCCUPANCY` ("The ratio of number of warps resident on an SM").

**Real-world engineering accounts**

- Databricks, ["Scaling Small LLMs with NVIDIA MPS"](https://www.databricks.com/blog/scaling-small-llms-nvidia-mps) — two identical inference engines on one H100 against a single-instance baseline, Qwen2.5 family; wins confined to very small models (≈≤3B) at short-to-medium context (<~2k tokens) and prefill-dominated work. Read it for the *negative* result outside that band, which is what makes it useful for policy. *(Not independently fetched — domain blocked by this environment's proxy; corroborated via search. Spot-check the thresholds.)*
- Pebble, ["NVIDIA MPS vs. Dedicated GPU Allocation for LLM Inference"](https://www.gopebble.com/case-studies/nvidia-mps-vs-dedicated-gpu-allocation-for-llm-inference/) — Qwen3-4B-FP8 on A100-80GB in GKE, 60% SM and 50% memory per process, two processes per GPU, ~50% cost reduction for ~7.5% throughput loss. Note the deliberate 120%-sum SM over-provisioning. *(Same fetch caveat.)*

**Deeper dives**

- NVIDIA GPU Operator docs, ["GPU Sharing"](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html) — the `sharing.mps` ConfigMap and ClusterPolicy wiring reference behind §5. *(Domain blocked by this environment's proxy; the ClusterPolicy `spec.devicePlugin.config.{name,default}` path used here was verified directly against the CRD in NVIDIA/gpu-operator v26.3.3.)*
- [Lesson 07 — Time-slicing and the attribution trap](07-time-slicing-attribution.md) — read first if you have not. The annotated-device-ID mechanism, the DCGM fan-out, the naive-`sum` arithmetic, and the per-PID fallback menu are all worked there and reused here without re-derivation.
- [Lesson 06 — MIG operations](06-mig-operations.md) — the third column of §8's matrix, and the source of the stranded-capacity arithmetic in the worked example's Mode D.
