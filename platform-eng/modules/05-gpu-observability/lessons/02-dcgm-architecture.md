---
lesson: "05.2"
title: "DCGM architecture and the cost of profiling"
module: "05"
concept: "DCGM architecture and the cost of profiling"
status: not-started
est_time: "6h"
prev: "01-lie-and-truth.md"
next: "03-dcgm-exporter-at-scale.md"
artifacts: []
sources: 17
---

# 05.2 · DCGM architecture and the cost of profiling

> **Concept.** DCGM's PROF metrics are a separate, hardware-counter-backed sampled subsystem with real costs — multiplexing, superuser privileges, and a bounded averaging window — and choosing scrape intervals against those costs is the fleet-scale judgment the interview probes.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Where this fits

Lesson 05.1 told you exactly which fields to trust — `SM_ACTIVE`, `SM_OCCUPANCY`,
`PIPE_TENSOR_ACTIVE`, `DRAM_ACTIVE` — which one (`GPU_UTIL`) to keep off every alert, and
sketched the two collection paths they arrive on. It stopped at the boundary of *how DCGM
is built*. This lesson goes inside: the process that owns the field cache, the object
model of fields and field groups and watches, what the profiling module does to the
silicon on pre-Hopper parts, why the hardware sometimes cannot give you everything you
asked for, and what happens when it can't.

You need this before 05.3, because dcgm-exporter's counter CSV is nothing but a field
group and its collect interval is nothing but a watch `updateFreq` — you cannot reason
about the config without the object model. You need it before 05.4, because the
attribution join depends on which *entity* a field is reported against, and that is a
DCGM concept. And you need it before 05.7, because `dcgmi profile --pause` exists for
exactly one reason and that reason is the subject of §6.

Everything below is checked against **DCGM 4.6.0** (`NVIDIA/DCGM`, July 2026) and
**dcgm-exporter 4.8.3** (`NVIDIA/dcgm-exporter`, image tag `4.6.0-4.8.3-distroless`),
reading `dcgmlib/dcgm_structs.h`, `dcgmlib/dcgm_fields.h`,
`dcgmlib/src/DcgmGpmManager.cpp`, `modules/core/DcgmModuleCore.cpp`,
`hostengine/src/HostEngineCommandLine.cpp`, `dcgmi/*`, `testing/python3/tests/test_prof.py`,
and the exporter's `internal/pkg/devicewatcher/`, `internal/pkg/collector/` and
`internal/pkg/dcgmerrors/`. Where a claim depends on GPU generation it is labelled.

## Why this matters

A senior platform interview at NVIDIA, CoreWeave or Datadog will not stop at "read
`SM_ACTIVE` instead of `GPU_UTIL`". The follow-up is: *"you want `SM_ACTIVE`,
`TENSOR_ACTIVE`, `DRAM_ACTIVE`, per-link NVLink and the FP64/32/16 pipes at 1-second
resolution on 5,000 GPUs. What breaks?"*

"Nothing, just scrape faster" is a failing answer. The correct one has four parts, and
each is a section below: **(1)** which fields can physically be read together is a property
of the GPU's counter layout and is exposed as *metric groups*; **(2)** on pre-Hopper
silicon DCGM will time-multiplex a bounded number of mutually-exclusive groups and
**refuses with `DCGM_ST_PROFILING_MULTI_PASS` (-38) beyond five of them**; **(3)** the
whole profiling path needs privileges NVML's device fields do not, and the failure is a
missing series rather than an error; **(4)** the averaging window of every activity ratio
*is* the watch interval, so the scrape-interval decision is bounded from one side by the
semantics you want and from the other by sampling cost.

The concrete cost of getting it wrong is a fleet-wide dashboard that is silently partial.
Every one of the four failure modes above produces the same visible symptom — a panel that
looks fine because the broken GPUs dropped out of the average — and none of them produces a
Prometheus scrape error. You will not find these by staring at PromQL.

## What's new here (calibration)

- You already deployed dcgm-exporter and wrote Prometheus scrape configs in module 04, and
  you understand scrape interval as an operational knob generically. **Not re-taught.**
- You already know from 05.1 which fields to trust and why field 203 lies. **Not
  re-taught.**
- New here:
  - **`nv-hostengine` as a process**: standalone vs embedded, the real CLI flags and
    defaults, the connection URI grammar, and what "the exporter embeds it" actually means
    for a second consumer on the same node.
  - **The object model**: entities, fields, *field groups*, *GPU groups*, and *watches*
    with `updateFreq` / `maxKeepAge` / `maxKeepSamples` — with the exact values
    dcgm-exporter passes.
  - **Metric groups and the `majorId`/`minorId` rule**, straight out of the header: same
    `majorId` ⇒ *cannot* be watched concurrently. This is the multiplexing mechanism, and it
    is visible in `dcgmi profile -l` output as `A.0` / `A.1`.
  - **The GPM discontinuity**: on Hopper and newer, all of 1001–1034 land in one metric
    group and multiplexing does not happen. The multiplexing story is a *pre-Hopper* story,
    and knowing that is the difference between reciting a rule and understanding it.
  - **The modular DCGM runtime**: eleven lazily-loaded modules, one of which is Profiling,
    and what "Failed to load" vs "Not loaded" vs "Paused" mean in `dcgmi modules -l`.
  - **The scrape-interval band, derived** rather than asserted.

## Core concepts

### 1. The problem DCGM exists to solve

Suppose you have no DCGM and want fleet GPU telemetry. Your options are NVML directly and
`nvidia-smi` parsing. Both break at scale for the same four reasons:

1. **NVML is a per-process library, not a service.** Every consumer that wants a metric
   opens its own NVML handle and polls the driver. Ten consumers on a node means ten
   independent polls of the same counters, ten sets of driver round-trips, and no shared
   cache. Some NVML calls are not cheap.
2. **Some metrics are not in NVML at all.** Everything in 05.1's Path B — SM breadth, warp
   occupancy, tensor-pipe activity, DRAM interface activity — comes from hardware
   performance monitors that require programming the counter units, which is a privileged
   operation that must be *coordinated*. Two processes cannot both own a counter bank.
3. **There is no history.** NVML tells you now. Anything that wants "what was `SM_ACTIVE`
   30 seconds ago" or "give me the last 100 samples" has to build its own ring buffer.
4. **Entities are not flat.** A MIG-partitioned A100 or H100 is a GPU containing GPU
   instances containing compute instances; an NVSwitch fabric has switches and links.
   "Field 1002 on this thing" needs a thing-identifier richer than a device index.

DCGM's answer is a **host engine**: one process per node that owns the driver handles,
programs the profiling hardware, keeps a time-indexed cache of watched fields per entity,
and answers queries from many clients over an RPC. Everything else in this lesson is a
consequence of that design.

### 2. `nv-hostengine`: standalone vs embedded

The engine ships two ways, and which one you are running determines who can talk to it.

**Standalone (daemon) mode.** `nv-hostengine` runs as its own long-lived process. Clients
— `dcgmi`, dcgm-exporter, your own Go/Python bindings — connect over the DCGM RPC. The real
flags and defaults, from `hostengine/src/HostEngineCommandLine.cpp`:

| Flag | Default | Meaning |
|---|---|---|
| `-p, --port PORT` | `5555` | TCP listen port |
| `-b, --bind-interface IP` | `127.0.0.1` | Interface to listen on; `ALL` binds everything |
| `-d, --domain-socket PATH` | `/tmp/nv-hostengine` | Unix domain socket. **Specifying this opens no TCP port at all.** |
| `-c, --vsock-cid CID` | bind any CID | vsock listener, for VM-to-host telemetry |
| `-n, --no-daemon` | off | Stay in the foreground (what you want in a container) |
| `--pid FILENAME` | `/var/run/nvhostengine.pid` | PID file enforcing single-instance |
| `--log-level LEVEL` | see build | `NONE`/`FATAL`/`ERROR`/`WARN`/`INFO`/`DEBUG` |
| `-t, --term` | — | Best-effort terminate a running host engine |

Note the mutual exclusions the parser enforces: you cannot specify both a Unix domain
socket and a vsock CID, and you cannot specify both a TCP interface and a Unix domain
socket. **The default bind of `127.0.0.1` is a real operational fact** — a standalone
host engine is loopback-only unless you deliberately widen it, which is why the
Kubernetes pattern is a DaemonSet with `hostNetwork` or a shared socket volume rather than
a Service.

**Embedded mode.** The engine is loaded *in-process* as `libdcgm.so`. There is no daemon,
no socket, no port. The consuming process **is** the host engine. In Go bindings this is
`dcgm.Init(dcgm.Embedded)`; standalone is `dcgm.Init(dcgm.Standalone, hostInfo, socketFlag)`.

**dcgm-exporter uses embedded mode by default.** From `internal/pkg/dcgmprovider/dcgm.go`,
the branch is a single boolean: if `config.UseRemoteHE` is set it calls the standalone
initialiser with `config.RemoteHEInfo`, otherwise it calls the embedded one. And
`UseRemoteHE` is set **only if the `-r/--remote-hostengine-info` flag was explicitly
provided** (`if c.IsSet(CLIRemoteHEInfo) { config.UseRemoteHE = true; ... }`) — the flag's
default value of `localhost:5555` is inert unless you pass it. There is no "auto-detect a
running host engine" behaviour. If you do not pass `-r`, you get an embedded engine even if
a perfectly good `nv-hostengine` is running on the node.

The `-r` argument accepts either a bare `HOST:PORT` or a DCGM URI:

```
  -r host123:5555                 # bare form, TCP
  -r tcp://host123:5555           # explicit TCP
  -r "[::1]:5555"                 # IPv6 needs brackets
  -r unix:///var/run/nvhe.sock    # Unix domain socket (note the three slashes)
  -r vsock://3:5555               # vsock, CID 3
```

Here is the structural picture. This is the diagram to be able to draw from memory.

```
 ╔═══════════════════════════════════════════════════════════════════════════════════╗
 ║  MODE A — EMBEDDED  (dcgm-exporter default)                                       ║
 ╚═══════════════════════════════════════════════════════════════════════════════════╝

   ┌─ pod: nvidia-dcgm-exporter ─────────────────────────────┐
   │                                                          │
   │   dcgm-exporter (Go)                                     │
   │      │  dcgm.Init(dcgm.Embedded)                         │
   │      ▼                                                   │
   │   ┌────────────────────── libdcgm.so ─────────────────┐  │
   │   │  MODULE TABLE (lazy-loaded)                        │  │
   │   │    0 Core       ← always loaded                    │  │
   │   │    1 NvSwitch   4 Health    7 Diag                 │  │
   │   │    2 VGPU       5 Policy    8 Profiling ◀── ★      │  │
   │   │    3 Introspect 6 Config    9 SysMon  10 MnDiag    │  │
   │   ├────────────────────────────────────────────────────┤  │
   │   │  FIELD CACHE   key: (entityGroup, entityId, field) │  │
   │   │                val: ring of (timestamp, value)     │  │
   │   ├────────────────────────────────────────────────────┤  │
   │   │  WATCH TABLE   updateFreq / maxKeepAge /           │  │
   │   │                maxKeepSamples, per watcher         │  │
   │   └───────┬─────────────────────────────┬──────────────┘  │
   │           │ NVML                        │ GPM / DCP       │
   └───────────┼─────────────────────────────┼─────────────────┘
               ▼                             ▼
        ┌──────────────┐          ┌────────────────────────┐
        │ NVIDIA driver│          │ GPU performance monitors│
        └──────────────┘          └────────────────────────┘
                                       ▲
   ┌─ pod: my-health-checker ────┐     │  ✗ CONTENTION: a second embedded
   │  its own libdcgm.so         │─────┘     engine wants the same counters.
   │  its own module table       │           This is DCGM issue #287.
   │  its own field cache        │
   └─────────────────────────────┘

 ╔═══════════════════════════════════════════════════════════════════════════════════╗
 ║  MODE B — STANDALONE  (one engine, many clients)                                  ║
 ╚═══════════════════════════════════════════════════════════════════════════════════╝

   ┌─ DaemonSet: nvidia-dcgm ────────────────────────────────┐
   │  nv-hostengine -n --port 5555 --bind-interface 127.0.0.1│
   │  ┌──────────────────────────────────────────────────┐   │
   │  │  ONE module table · ONE field cache · ONE watch  │   │
   │  │  table with per-watcher entries.                  │   │
   │  │  Watches are UNIONED: the engine samples at the   │   │
   │  │  MINIMUM updateFreq any watcher asked for, and    │   │
   │  │  retains to the MAXIMUM maxKeepAge.               │   │
   │  └────────────────────┬─────────────────────────────┘   │
   └───────────────────────┼─────────────────────────────────┘
                    ▲      │      ▲                    ▲
      -r tcp://…5555│      │      │ dcgmi --host …     │ your Go client
   ┌────────────────┴──┐   │   ┌──┴───────────────┐ ┌──┴──────────────┐
   │  dcgm-exporter    │   │   │ dcgmi dmon / diag│ │ health checker  │
   └───────────────────┘   │   └──────────────────┘ └─────────────────┘
                           ▼
                  ONE owner of the profiling counters.
                  No contention. This is the fix for #287.
```

**Why this is a real decision and not trivia.** `NVIDIA/DCGM` issue #287, "How to correctly
use the nv-hostengine on Kubernetes", is exactly this: a user discovered they were running
two embedded engines on one node — one inside dcgm-exporter, one inside a custom
health-check container — recognised it as wrong, and asked whether to consolidate onto a
standalone engine reachable over `tcp://localhost:5555` with host networking, or a Unix
socket mounted into both pods, and whether the engine should be a host process or a
container. As of writing, the issue is open with no maintainer answer. The design guidance
you need is derivable from this lesson:

| Situation | Mode | Why |
|---|---|---|
| One consumer per node (exporter only) | **Embedded** | No socket, no extra pod, no lifecycle coupling. This is why it is the default. |
| Exporter **plus** anything else touching PROF fields | **Standalone** | Only one owner of the counters; watches are unioned instead of contended. |
| You run `dcgmi diag` / `dcgmi dmon` on production nodes | **Standalone** | Otherwise every ad-hoc `dcgmi` spins up its own engine. |
| VM guests reporting to a host collector | **Standalone + vsock** | `-c CID` exists for precisely this. |
| Multi-tenant nodes with untrusted workloads | **Standalone + Unix socket** | Socket file permissions are your access control; no TCP port is opened at all. |

The GPU Operator ships the standalone engine as an operand (`nvidia-dcgm` DaemonSet) but
leaves it **disabled by default** (`dcgm.enabled=false`), because the default single-consumer
case is served by the exporter's embedded engine. Turning it on and *not* also pointing the
exporter at it with `-r` gives you the worst of both: a running daemon nobody uses and an
embedded engine still owning the counters.

### 3. The modular runtime, and how to see it

DCGM is not monolithic. `dcgm_structs.h` defines eleven module IDs, and modules are
**lazily loaded** — a module is not resident until something asks for a capability it
provides:

| ID | Name | Loaded when |
|---|---|---|
| 0 | `Core` | always |
| 1 | `NvSwitch` | NVSwitch fields watched |
| 2 | `VGPU` | vGPU fields watched |
| 3 | `Introspection` | introspection API used |
| 4 | `Health` | `dcgmi health` / health watches |
| 5 | `Policy` | policy API used |
| 6 | `Config` | `dcgmi config` |
| 7 | `Diag` | `dcgmi diag` |
| **8** | **`Profiling`** | **any `DCGM_FI_PROF_*` field watched on a non-GPM GPU** |
| 9 | `SysMon` | CPU/system fields |
| 10 | `MnDiag` | multi-node diagnostics |

Module status is one of seven values, and the distinction between three of them is the
first thing to check when PROF fields are missing:

```console
$ dcgmi modules --list
+-----------+--------------------+------------------------------------------------+
| List Modules                                                                     |
| Status: Success                                                                  |
+===========+====================+================================================+
| Module ID | Name               | State                                          |
+-----------+--------------------+------------------------------------------------+
| 0         | Core               | Loaded                                         |
| 1         | NvSwitch           | Not loaded                                     |
| 2         | VGPU               | Not loaded                                     |
| 3         | Introspection      | Not loaded                                     |
| 4         | Health             | Not loaded                                     |
| 5         | Policy             | Not loaded                                     |
| 6         | Config             | Not loaded                                     |
| 7         | Diag               | Not loaded                                     |
| 8         | Profiling          | Loaded                                         |
| 9         | SysMon             | Not loaded                                     |
| 10        | MnDiag             | Not loaded                                     |
+-----------+--------------------+------------------------------------------------+
```

(Column widths 11 / 20 / 50 and the exact state strings come from `dcgmi/Module.cpp`; the
transcript is representative of a healthy single-consumer node, not a specific capture.)

Read the `State` column carefully — the three failure states mean different things:

- **`Not loaded`** — nobody has asked yet. On a node where nothing has ever watched a PROF
  field, `Profiling: Not loaded` is *normal*, not broken.
- **`Failed to load`** — DCGM tried and could not. This is `NVIDIA/dcgm-exporter` issue #22
  verbatim: a user on a Tesla K80 saw every PROF field empty, `dcgmi` reporting *"This
  request is serviced by a module of DCGM that is not currently loaded"*
  (`DCGM_ST_MODULE_NOT_LOADED`, -33), and `dcgmi modules -l` showing `Profiling: Failed to
  load` while Core and NvSwitch loaded fine. Kepler predates the profiling hardware; there
  is nothing to fix. **On Volta through Ampere this state also appears when the proprietary
  DCP component is missing** — dcgm-exporter's README notes that *"for Ampere and earlier
  generation GPUs, profiling metrics depend on the datacenter-gpu-manager-4-proprietary
  package"* (shipped inside the container image, but a real gap in hand-rolled installs).
- **`Denylisted`** — someone ran `dcgmi modules --denylist Profiling`, or the deployment
  did. A module on the denylist cannot be loaded until the engine restarts.
- **`Paused`** — someone ran `dcgmi profile --pause`. See §6.

### 4. The object model: entities, fields, field groups, GPU groups, watches

This is the vocabulary the rest of the module is written in. Five nouns, and they compose.

**Entity.** A thing a field can be reported against. `dcgm_field_entity_group_t` includes
`DCGM_FE_GPU` (a whole physical GPU), `DCGM_FE_GPU_I` (a MIG *GPU instance*),
`DCGM_FE_GPU_CI` (a MIG *compute instance*), `DCGM_FE_SWITCH`, `DCGM_FE_LINK` (an
individual NVLink), `DCGM_FE_CPU`, `DCGM_FE_CPU_CORE`. Every field declares the entity
level it is natively reported at; in `dcgm_fields.cpp`, `SM_ACTIVE`, `SM_OCCUPANCY`,
`GR_ENGINE_ACTIVE` and `PIPE_TENSOR_ACTIVE` are registered at `DCGM_FE_GPU_CI`, while
`DRAM_ACTIVE` is registered at `DCGM_FE_GPU_I`. **This is why MIG attribution works for SM
fields and why bandwidth is instance-scoped** — it is baked into the field metadata, and
lesson 05.4 depends on it.

**Field.** An integer ID with a type (`DCGM_FT_INT64`, `_DOUBLE`, `_STRING`, `_TIMESTAMP`,
`_BINARY`), a tag (`sm_active`), a five-character short name for `dcgmi dmon` (`SMACT`), a
display width, and a unit string. 05.1's table is the subset you care about.

**Field group.** A named, server-side set of field IDs, created with `dcgmFieldGroupCreate()`
and referred to by handle thereafter. You do not enumerate fields on every call; you create
the group once and watch *it*. The exporter's `newFieldGroupSimple()` does exactly this,
deduplicating the field list first.

**GPU group.** A named set of *entities*. `DCGM_GROUP_ALL_GPUS` is the built-in
everything-group; the exporter creates its own (`gpu-collector-group-<random>`) and adds
the entities its device-selection flags matched.

**Watch.** The cross product: *this field group* × *this GPU group*, sampled at
`updateFreq` microseconds, retained for `maxKeepAge` seconds or `maxKeepSamples` samples.
The call is `dcgmWatchFieldsWithGroupEx(fieldGroup, group, updateFreq, maxKeepAge,
maxKeepSamples)`.

The exact values dcgm-exporter passes, from `internal/pkg/devicewatcher/`:

```go
// internal/pkg/devicewatcher/const.go
const (
    maxKeepAge     = 600.0 // seconds — how long DCGM keeps samples for these fields
    maxKeepSamples = 0     // 0 = no per-field sample-count limit
)

// internal/pkg/devicewatcher/device_watcher.go
func watchFieldGroupSimple(group dcgm.GroupHandle, field dcgm.FieldHandle, updateFreq int64) error {
    return dcgmprovider.Client().WatchFieldsWithGroupEx(field, group, updateFreq, maxKeepAge, maxKeepSamples)
}

// …called with the watch group's interval converted to microseconds:
err = watchFieldGroupSimple(group, fieldGroup, fieldWatchGroup.IntervalMSec*1000)
```

And `IntervalMSec` defaults to the exporter's `--collect-interval`, whose default is
**30000 ms**:

```go
// pkg/cmd/app.go
&cli.IntFlag{
    Name:    CLICollectInterval,
    Value:   30000,
    Usage:   "Interval of time at which point metrics are collected. Unit is milliseconds (ms).",
    EnvVars: []string{"DCGM_EXPORTER_INTERVAL"},
},
```

**Write that number down: out of the box, every DCGM field dcgm-exporter serves is sampled
by the engine every 30 seconds and retained for 600 seconds.** Not 1 second. Combined with
05.1 §5, this means the default `SM_ACTIVE` you scrape is *the mean over the preceding
~30 seconds*, and DCGM is holding 20 samples of history per field per entity.

Since 4.8.x the exporter can split that into per-field watch intervals via YAML, which
matters for §7's cost argument:

```yaml
# --config-file / DCGM_EXPORTER_CONFIG_FILE
version: 1
metrics:
  file: /etc/dcgm-exporter/default-counters.csv
collection:
  interval: 30s                  # default for anything not matched below
  watchGroups:
    - name: fast-thermals        # cheap NVML fields: sample often
      interval: 5s
      fields:
        - DCGM_FI_DEV_GPU_TEMP
        - DCGM_FI_DEV_POWER_USAGE
    - name: slow-nvlink-prm      # expensive/rarely-changing: sample rarely
      interval: 5m
      fields:
        - DCGM_FI_DEV_NVLINK_PPCNT_*
```

Field names glob-match; anything unmatched uses `collection.interval`; a field matching
more than one named group is a **startup error**, and a watch group matching zero fields
is also rejected. YAML edits require a restart (only the CSV metric file hot-reloads).

Now the temporal picture — what actually happens between a watch being armed and a
Prometheus sample landing:

```
  ONE WATCH, END TO END.  updateFreq = 30 s, Prometheus scrape = 30 s.

  t (s)    0        30        60        90       120       150
           │         │         │         │         │         │
  ENGINE   │         │         │         │         │         │
  samples  ●─────────●─────────●─────────●─────────●─────────●
           S0        S1        S2        S3        S4        S5
           │         │         │         │         │         │
           │  ┌──────┴──────┐  │         │         │         │
           │  │ metric =    │  │         │         │         │
           │  │ f(S0,S1)    │  │         │         │         │
           │  │ = mean over │  │         │         │         │
           │  │ [0,30]      │  │         │         │         │
           │  └─────────────┘  │         │         │         │
           │                   │         │         │         │
  CACHE    ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
           └─ maxKeepAge = 600 s ⇒ 20 samples retained ─────────▶
           (GPM sample map prunes at 2×interval + min(interval,1 s) = 61 s)

  SCRAPE      ▲          ▲          ▲          ▲          ▲
              │          │          │          │          │
           t=5        t=35       t=65       t=95      t=125
           reads      reads      reads      reads      reads
           BLANK      f(S0,S1)   f(S1,S2)   f(S2,S3)   f(S3,S4)
           (no        ──────────────────────────────────────────
           baseline    each scrape returns the value computed from
           yet)        the most recent completed interval — so a
                       Prometheus sample at t is describing [t-35, t-5].

  ─────────────────────────────────────────────────────────────────────────
  CONSEQUENCE 1  The first scrape after exporter start has NO PROF series.
  CONSEQUENCE 2  Scraping faster than updateFreq re-reads the SAME cached
                 value. You get duplicate points, not resolution.
  CONSEQUENCE 3  Scraping slower than updateFreq throws away completed
                 intervals. A 5-minute scrape of a 30 s watch shows you
                 30 s out of every 300 s — 10% duty-cycle sampling, with
                 the other 90% never observed by anyone.
  ─────────────────────────────────────────────────────────────────────────
```

Consequence 3 is the one that costs money. **A bursty training step is invisible if your
scrape interval exceeds your watch interval.** The GPU is busy for 20 seconds and idle for
280; the watch computed six separate 30-second means; Prometheus kept one of them, chosen
by scrape phase, not by content. Whether your "average utilisation" number reads 0.9 or
0.05 is then luck.

### 5. Metric groups: which fields the silicon can read together

Now the constraint that makes profiling different from every other exporter you have run.

The physical situation: the GPU's SM partitions expose performance counters through a fixed
number of hardware collection slots, and each slot holds **one counter-select register at a
time**. A "metric" is not a counter — `PIPE_TENSOR_ACTIVE` is derived from tensor-pipe issue
counters, `SM_OCCUPANCY` from a resident-warp accumulator, and on some architectures those
derivations read the same physical bank. Two metrics that need the same bank programmed
differently **cannot both be collected in one pass**, because the bank cannot hold two
selects simultaneously. That is a silicon property, and it changes between generations.

DCGM surfaces it as **metric groups**, and the rule is stated exactly in `dcgm_structs.h`:

```c
typedef struct
{
    unsigned short majorId;   //!< Major ID of this metric group. Metric groups with the same majorId
                              //!< cannot be watched concurrently with other metric groups with the
                              //!< same majorId
    unsigned short minorId;   //!< Minor ID of this metric group. This distinguishes metric groups
                              //!< within the same major metric group from each other
    unsigned int numFieldIds;
    unsigned short fieldIds[DCGM_PROF_MAX_FIELD_IDS_PER_GROUP_V2];  // 64
} dcgmProfMetricGroupInfo_v2;
```

with `DCGM_PROF_MAX_NUM_GROUPS_V2 = 10` and `DCGM_PROF_MAX_FIELD_IDS_PER_GROUP_V2 = 64`.

**Read the rule twice, because it is counter-intuitive: *different* `majorId` means the
groups CAN be watched together; the *same* `majorId` with different `minorId` means they
CANNOT.** `minorId` enumerates the mutually-exclusive alternatives within one hardware
resource. So `A.0` and `A.1` are the pair that will multiplex; `A.0` and `B.0` are free.

`dcgmi profile -l` prints exactly this, one row per field, with `majorId` rendered as a
letter (`'A' + majorId`) and `minorId` as the number after the dot. Columns are 16 / 10 / 54
characters wide (`dcgmi/DcgmiProfile.cpp`):

```console
$ dcgmi profile -l -i 0
+----------------+----------+------------------------------------------------------+
| Group.Subgroup | Field ID | Field Tag                                            |
+================+==========+======================================================+
| A.0            | 1001     | gr_engine_active                                     |
| A.0            | 1002     | sm_active                                            |
| A.0            | 1003     | sm_occupancy                                         |
| A.0            | 1004     | tensor_active                                        |
| A.0            | 1005     | dram_active                                          |
| A.0            | 1006     | fp64_active                                          |
| A.0            | 1007     | fp32_active                                          |
| A.0            | 1008     | fp16_active                                          |
| A.0            | 1009     | pcie_tx_bytes                                        |
| A.0            | 1010     | pcie_rx_bytes                                        |
| A.0            | 1011     | nvlink_tx_bytes                                      |
| A.0            | 1012     | nvlink_rx_bytes                                      |
| A.0            | 1013     | tensor_imma_active                                   |
| A.0            | 1014     | tensor_hmma_active                                   |
| A.0            | 1015     | tensor_dfma_active                                   |
| A.0            | 1016     | integer_active                                       |
| B.0            | 1040     | nvlink_l0_tx_bytes                                   |
| B.0            | 1041     | nvlink_l0_rx_bytes                                   |
| …              | …        | …                                                    |
| C.0            | 1084     | sm_cycles_elapsed_total                              |
| C.0            | 1085     | sm_cycles_active_total                               |
+----------------+----------+------------------------------------------------------+
```

**That output is from an H100 — a GPM GPU — and its shape is the whole point.** Every
activity ratio is in group `A.0`. There is no `A.1`. **Nothing multiplexes.**

Why: on Hopper and newer, DCGM does not go through the profiling module for these fields at
all. `DcgmModuleCore::ProcessProfGetMetricGroups()` checks `EntityPairSupportsGpm()` first
and, for a GPM GPU, *synthesises* exactly three metric groups in-line rather than asking the
profiling module:

- **group 0 (`A.0`)** — every field ID from `DCGM_FI_PROF_FIRST_ID` (1001) through
  `DCGM_FI_PROF_NVOFA_UTIL_1_RATIO` (1034), contiguously.
- **group 1 (`B.0`)** — `DCGM_FI_PROF_NVLINK_L0_TX_BYTES` (1040) through
  `DCGM_FI_PROF_PEERMEM_CACHE_MISS` (1083), plus the two per-link `dcgm_link_t`-keyed fields.
- **group 2 (`C.0`)** — the cumulative cycle counters, `DCGM_FI_PROF_SM_CYCLES_ELAPSED_TOTAL`
  (1084) through `DCGM_FI_PROF_FP16_CYCLES_ACTIVE_TOTAL`.

All three have `minorId = 0` and distinct `majorId`s, so by the header's own rule **all
three can be watched concurrently.** The mechanism behind that is 05.1 §5:
`nvmlGpmSampleGet()` takes one snapshot of *every* GPM metric at once, so there is nothing
to trade off. NVML documents GPM as *"For Hopper or newer fully supported devices"*.

On **Volta, Turing and Ampere**, the request is routed to the proprietary profiling module,
which reports the *real* per-architecture layout — and there the same-`majorId` case exists.
The canonical documented example, from NVIDIA's DCGM profiling documentation: *SM Activity
and SM Occupancy cannot be collected together with Tensor Utilization on a V100, but can be
on a T4.* On such a GPU the same `dcgmi profile -l` shows something like:

```console
$ dcgmi profile -l -i 0          # representative pre-GPM layout (V100-class)
+----------------+----------+------------------------------------------------------+
| Group.Subgroup | Field ID | Field Tag                                            |
+================+==========+======================================================+
| A.0            | 1002     | sm_active                                            |
| A.0            | 1003     | sm_occupancy                                         |
| A.1            | 1004     | tensor_active                                        |   ◀── same major 'A'
| A.1            | 1008     | fp16_active                                          |       as A.0 ⇒ EXCLUSIVE
| B.0            | 1005     | dram_active                                          |
| C.0            | 1009     | pcie_tx_bytes                                        |
| C.0            | 1010     | pcie_rx_bytes                                        |
+----------------+----------+------------------------------------------------------+
```

Watch `1002` and `1004` together on that GPU and you have asked for `A.0` **and** `A.1`.
DCGM does not refuse — it time-multiplexes, programming `A.0` for a while, then `A.1`, and
statistically sampling. NVIDIA's documentation is explicit that DCGM *"supports automatic
multiplexing of metrics by statistically sampling the requested metrics and performing the
groupings internally,"* and that this *"may be transparent to users."*

**But the multiplexing is bounded.** DCGM's own test suite (`testing/python3/tests/test_prof.py`)
encodes the limit as a constant and tests both sides of it:

```python
DLG_MAX_METRIC_GROUPS = 5

# positive: watching up to DLG_MAX_METRIC_GROUPS exclusive groups succeeds
for i in range(min(len(mpFieldIds), DLG_MAX_METRIC_GROUPS)):
    ...
    dcgmGroup.samples.WatchFields(fieldGroup, 1000000, 3600.0, 0)

# negative: beyond that, WatchFields raises
for i in range(DLG_MAX_METRIC_GROUPS + 1, len(mpFieldIds) + 1):
    ...
    with test_utils.assert_raises(
            dcgm_structs.dcgmExceptionClass(dcgm_structs.DCGM_ST_PROFILING_MULTI_PASS)):
        dcgmGroup.samples.WatchFields(fieldGroup, 1000000, 3600.0, 0)
```

`DCGM_ST_PROFILING_MULTI_PASS` is **-38**, documented in `dcgm_structs.h` as *"The requested
profiling metrics cannot be collected in a single pass"*. So the honest, complete answer to
"what happens if I ask for everything?" is a three-branch answer:

```
   HOW MANY MUTUALLY-EXCLUSIVE (same-majorId) METRIC GROUPS DID YOU WATCH?

        1 group                2..5 groups                  >5 groups
           │                       │                            │
           ▼                       ▼                            ▼
   ┌───────────────┐   ┌──────────────────────────┐   ┌──────────────────────┐
   │ SINGLE PASS   │   │ MULTIPLEXED              │   │ REFUSED              │
   │               │   │                          │   │                      │
   │ counters      │   │ engine rotates the       │   │ WatchFields returns  │
   │ programmed    │   │ counter-select registers │   │ DCGM_ST_PROFILING_   │
   │ once, read    │   │ between groups and       │   │   MULTI_PASS  (-38)  │
   │ every         │   │ statistically samples.   │   │                      │
   │ interval.     │   │ Each field's EFFECTIVE   │   │ dcgm-exporter logs   │
   │               │   │ sample rate falls by ~N. │   │ the failure and the  │
   │ Full          │   │ Values are estimates,    │   │ whole watch is not   │
   │ resolution.   │   │ noisier at short         │   │ established — you    │
   │               │   │ intervals.               │   │ lose ALL of them.    │
   └───────────────┘   └──────────────────────────┘   └──────────────────────┘

   On Hopper+ (GPM): every ratio field is majorId 0, so you are ALWAYS in
   the left box no matter how many you ask for.
```

That last line is the piece almost nobody knows, and it is what makes the difference
between reciting "profiling multiplexes" and being able to answer *"does it multiplex on
our H100 fleet?"* — no, it does not, and you can prove it with one `dcgmi profile -l`.

**The practical rule, restated:** do not carry a "safe PROF field set" across a hardware
refresh. Run `dcgmi profile -l` on each GPU model in the fleet, group the rows by the letter
before the dot, and count distinct letters-with-multiple-subgroups. That count, not the
number of fields, is what determines whether you are multiplexing.

### 6. Privileges, pausing, and the three ways PROF fields go missing

Three separate mechanisms make `DCGM_FI_PROF_*` disappear, and they look identical on a
dashboard. Distinguishing them is most of the operational value of this lesson.

**(a) Privileges.** Programming performance counters is privileged. The classic failure is
`NVIDIA/dcgm-exporter` issue #34: on an A100 node with MIG enabled on one device, the
container failed with *"CacheManager Init Failed. Error: -17"* and *"Error starting
nv-hostengine: DCGM initialization error"*, having printed *"dcgm-exporter doesn't have
sufficient privileges to expose profiling metrics. To get profiling metrics with
dcgm-exporter, use `--cap-add SYS_ADMIN`."*

Modern versions have internalised this. The exporter's Helm chart ships:

```yaml
# deployment/values.yaml
securityContext:
  runAsNonRoot: false
  runAsUser: 0
  capabilities:
    add: ["SYS_ADMIN"]  # Required for profiling metrics (DCGM_FI_PROF_*)
    drop: ["ALL"]
  allowPrivilegeEscalation: false
  # Note: For non-root without profiling metrics, use:
  # runAsNonRoot: true, runAsUser: 1000, and remove SYS_ADMIN from capabilities.add
```

and the exporter now classifies the DCGM status codes into operator-facing hints
(`internal/pkg/dcgmerrors/dcgm_errors.go`):

| DCGM status | Exporter's hint |
|---|---|
| `DCGM_ST_NO_PERMISSION` | *"DCGM does not have permission… Verify container/device permissions, root/euid, and required capabilities such as CAP_SYS_ADMIN."* |
| `DCGM_ST_REQUIRES_ROOT` (-29) | *"This DCGM operation requires root. Run dcgm-exporter or nv-hostengine with the required root privileges."* |
| `DCGM_ST_PAUSED` | *"Resume DCGM or restart nv-hostengine before scraping metrics."* |
| `DCGM_ST_CONNECTION_NOT_VALID` | *"Verify nv-hostengine is running and restart dcgm-exporter after the DCGM connection recovers."* |
| `DCGM_ST_MODULE_NOT_LOADED` (-33) | routed through the same classifier — check `dcgmi modules -l` |

**The operational consequence:** any Pod Security Standard, OPA policy or hardened base
that strips `SYS_ADMIN` silently converts your fleet from "GPU observability" to "GPU
thermometry". Device fields keep flowing. Every honest field goes away. Nothing alerts.

**(b) Deliberate pause.** DCGM and external profilers cannot both own the counters, so DCGM
exposes an explicit yield:

```console
$ dcgmi profile --pause
Successfully paused profiling.

$ ncu --set full ./my_kernel          # Nsight Compute now owns the counters
...

$ dcgmi profile --resume
Successfully resumed profiling.
```

The flags are exclusive-or with `-l` in the CLI parser, and the help text names the reason
outright: *"Pause DCGM profiling in order to run NVIDIA developer tools like nvprof, nsight
compute, or nsight systems."* While paused, the Profiling module reports state `Paused` in
`dcgmi modules -l`, PROF fields stop producing fresh values, and any client call that needs
them can return `DCGM_ST_PAUSED`.

**This cuts both ways, and it is lesson 05.7's hinge.** A data scientist who runs
`ncu`/`nsys` on a shared node — with or without pausing DCGM first — takes the counters
away from your fleet telemetry for the duration. A gap in `SM_ACTIVE` on one node for
eleven minutes at 14:30 is frequently a human with a profiler, not a hardware fault.

**(c) Stale watches, and the exporter's self-repair.** Since 4.8.x dcgm-exporter watches
its own profiling streams for staleness. From `internal/pkg/collector/gpu_collector.go`:

```go
// resolveProfilingStaleWindows maps watched profiling fields to twice their watch interval.
// …
windows[fieldID] = 2 * time.Duration(watchGroup.IntervalMSec) * time.Millisecond
```

Per `(entity, field)` stream it remembers the last DCGM value timestamp. If the timestamp
has not advanced for **2× the watch interval**, or DCGM returns a status the exporter
classifies as a repairable watch-lifecycle error, it declares the watch stale, **discards
the profiling metrics from that scrape** (keeping the non-profiling ones), re-arms the DCGM
watch, and retries once. A rate-limiting cooldown of the same 2× window prevents repeated
failures from churning watches.

Two things follow. First, with the default 30 s interval, a wedged profiling watch produces
a **60-second minimum gap** in your PROF series before repair even starts. Second — and this
is the important one — **the repair path deliberately emits a scrape with device fields and
no PROF fields.** So even the healthy self-healing behaviour manifests as intermittent
absence. Alert on absence *sustained over several scrapes*, not on a single missing sample,
or you will page on the exporter fixing itself.

Putting all three together, the diagnostic order that actually saves time:

```
  "DCGM_FI_PROF_SM_ACTIVE is missing / gappy for some GPUs"
        │
        ├─▶ Is it missing for ALL GPUs on the node, since exporter start?
        │      └─▶ dcgmi modules -l  →  Profiling: "Failed to load"?
        │             ├─ pre-Volta silicon           → nothing to fix (issue #22)
        │             └─ Volta–Ampere, hand install  → missing proprietary DCP package
        │          →  exporter logs mention CAP_SYS_ADMIN / DCGM_ST_REQUIRES_ROOT?
        │             └─ securityContext dropped the capability (issue #34)
        │          →  is the field even in the counters CSV?      → lesson 05.3
        │
        ├─▶ Is it missing for ONE node, for a bounded window?
        │      └─▶ dcgmi modules -l  →  Profiling: "Paused"?
        │             └─ somebody is running ncu/nsys           → §6(b)
        │
        ├─▶ Is it gappy every ~60 s across many nodes?
        │      └─▶ exporter logs "profiling watch repair"       → §6(c)
        │             └─ usually a symptom of contention: two engines? (§2)
        │
        └─▶ Are values present but noisy / implausible at short intervals?
               └─▶ dcgmi profile -l: two subgroups under one letter?
                      └─ multiplexing on pre-GPM silicon         → §5
```

### 7. Deriving the interval band

Now the number the interview wants: what do you set `--collect-interval` and the Prometheus
scrape interval to, fleet-wide, and *why*.

**Upper bound — semantics.** From §4, the activity ratio is the mean over the watch
interval. If `scrape_interval > updateFreq`, Prometheus keeps one interval in every
`scrape_interval / updateFreq` and never sees the rest. To observe every second of GPU time
you need:

```
    scrape_interval  ≤  updateFreq
```

and to make each stored sample describe a contiguous, non-overlapping slice of wall-clock
time, you want them **equal**. This is the opposite of the usual Prometheus instinct
(exporters normally expose an instantaneous value that can be sampled at any rate); DCGM's
PROF fields are *interval statistics*, so scrape and watch must be aligned.

**Lower bound — signal.** Setting `updateFreq` very small does not make the number more
informative, it makes it noisier, for two reasons. On the GPM path, DCGM must retain enough
samples to find a baseline, and it prunes at `2 × maxUpdateInterval + min(maxUpdateInterval,
1 s)`; below ~1 s the jitter slack dominates the interval itself and lost samples become
likely. On the multiplexing path (§5), a field that is only programmed a fraction of the
time is being *statistically estimated*, and shortening the interval reduces the number of
observations per estimate. **~1 s is the practical floor.**

**Lower bound — cost.** Every watch interval costs a driver round trip per entity per watch
group. Sizing it for a node:

```
  8 GPUs/node × 26 enabled counters (dcgm-exporter's default CSV, lesson 05.3)
      = 208 (entity, field) pairs per node

  At updateFreq = 30 s :  208 / 30    ≈    6.9 engine samples/s/node
  At updateFreq =  1 s :  208 / 1     ≈  208   engine samples/s/node   ← 30×

  Across a 65-node / 520-GPU fleet:
      30 s :   ~450 samples/s fleet-wide
       1 s : ~13,500 samples/s fleet-wide
```

The exporter's own Helm chart sizes the pod at `requests: 100m CPU / 128Mi` and
`limits: 200m CPU / 512Mi`. Thirty-fold more sampling on the same limit is how you get a
CPU-throttled exporter whose scrapes start timing out — and the chart's ServiceMonitor sets
`scrapeTimeout: 25s` against a 30 s interval, so there is not much headroom to lose.

**Upper bound — Prometheus.** Series count is unchanged by interval, but sample rate is
linear in it, and Prometheus stores 1–2 bytes per sample after compression (Prometheus
storage docs: *"Prometheus stores an average of only 1-2 bytes per sample"*, with
`needed_disk_space = retention_time_seconds × ingested_samples_per_second × bytes_per_sample`).
For the same fleet with 26 counters:

```
  520 GPUs × 26 counters = 13,520 series

  at 30 s scrape :  13,520 / 30 ≈    451 samples/s
                    451 × 15 d × 86,400 s × 2 B ≈  1.17 GB
  at  5 s scrape :  13,520 /  5 ≈  2,704 samples/s
                    2,704 × 15 d × 86,400 s × 2 B ≈  7.0 GB
  at  1 s scrape :  13,520 /  1 ≈ 13,520 samples/s
                    13,520 × 15 d × 86,400 s × 2 B ≈ 35.0 GB
```

Disk is not the binding constraint at this scale — a 35 GB delta is nothing. **The binding
constraint is engine CPU and exporter scrape latency, not TSDB storage**, which is worth
knowing because the instinctive objection ("that'll blow up Prometheus") is wrong here and
the real objection ("that'll throttle the exporter and time out the scrape") is right.

**The recommendation, and its justification:**

| Fleet posture | `--collect-interval` | Prometheus scrape | Why |
|---|---|---|---|
| **Default / cost & capacity work** | `30000` (30 s) | `30s` | Aligned. One contiguous 30 s mean per stored point. Cheap. Adequate for allocated-vs-utilised GPU-hours, which integrate over hours. |
| **Efficiency debugging on a subset** | `5000` (5 s) | `5s` | Resolves individual training steps. Apply to a node subset via a second exporter DaemonSet with a distinct ServiceMonitor, not fleet-wide. |
| **Incident / live investigation** | — | — | Don't change the fleet. Use `dcgmi dmon -d 1000` on the node; it is a separate watch that goes away when you Ctrl-C. |
| **Mixed** | `30s` default + `watchGroups` | `30s` | Keep thermals/power at 5 s and NVLink error counters at 5 m via YAML `watchGroups`, PROF at 30 s. Best cost/signal ratio. |

**The sentence to say in the interview:** *"Prometheus scrape interval must equal the DCGM
watch `updateFreq`, because the PROF fields are interval means rather than instantaneous
gauges — scraping faster duplicates points, scraping slower discards whole intervals. We
run both at 30 s fleet-wide, which is dcgm-exporter's default, and drop a second DaemonSet
at 5 s onto the nodes we're actively debugging. Sub-second is pointless: on Hopper the GPM
sample retention is `2×interval + jitter slack`, and on Ampere and older you're
statistically estimating multiplexed counters, so shortening the interval buys noise."*

## Perspectives

**Developer.** A developer running `dcgmi dmon` on a devbox sees clean, unbroken numbers,
because they are the only client, they are root, and their single field group fits one
pass. Every mechanism in this lesson — contention, multiplexing, permission gaps,
pause/resume, watch staleness — is invisible until a *second* consumer exists. It is a
textbook "works on my node" trap, and it is why the fleet story looks nothing like a
single-GPU debugging session. The developer-facing implication: if you are about to run
`ncu` or `nsys` on a shared node, `dcgmi profile --pause` first and `--resume` after, or you
will silently punch a hole in someone's dashboard and never know.

**Operator.** Engine mode and watch interval are fleet-wide configuration with non-obvious
failure modes, and *none of them surface as a Prometheus scrape error*. The operator's job
is to (a) decide embedded vs standalone once, based on how many consumers exist per node;
(b) align scrape and watch interval; (c) run `dcgmi profile -l` on each GPU model in the
fleet, once, and record whether any letter has multiple subgroups; and (d) alert on the
*absence* of PROF series relative to device series, because absence is the shared symptom
of every failure in §6.

**Hardware.** The counter-bank constraint is not a software limitation DCGM could fix with
more threads. A bank holds one counter-select register; two metrics deriving from the same
bank are genuinely exclusive. The interesting part is that Hopper's GPM **changed the
shape of the problem** rather than adding capacity: instead of programming banks per
requested metric, GPM snapshots the whole monitoring state at once and lets software
difference two snapshots. That is why 1001–1034 collapse into a single metric group on
Hopper and why the multiplexing question quietly stops being a question on modern silicon —
while remaining live on the A100 fleet you almost certainly still operate.

**Economics.** Two costs sit here, and only one is obvious. The obvious one: over-sampling
PROF fields fleet-wide multiplies engine CPU by the interval ratio, and at thousands of
GPUs a 30× increase runs a 200m-CPU-limited exporter into throttling and scrape timeouts.
The non-obvious and much larger one: **a silently half-collected fleet costs you the entire
value of the module.** A dashboard missing `SM_ACTIVE` on 40% of GPUs — because those nodes
lost `SYS_ADMIN` in a hardening pass — under-reports waste in exact proportion, and the
waste it fails to report is the six-to-seven-figure line item the capstone exists to
surface. Getting the interval band right once, fleet-wide, is cheaper than re-deriving it
per team; getting the *absence alert* right is worth more than either.

## Real-world use cases

- **`NVIDIA/DCGM` issue #287 — "How to correctly use the nv-hostengine on Kubernetes."**
  A user discovers they are running two embedded host engines on one node — one inside
  dcgm-exporter, one inside a custom health-check container — recognises this as wrong, and
  asks two concrete questions: TCP on `localhost:5555` with host networking versus a mounted
  Unix socket, and whether `nv-hostengine` should be a host process or a dedicated
  container. The issue is open with no maintainer response.
  **What it shows:** the embedded-vs-standalone decision is a live production question with
  no vendor-blessed answer, which means *you* are expected to be able to reason it out. §2's
  table is that reasoning. It also shows the failure is discovered by *inspection*, not by
  an alert — nothing in either container reported a problem.

- **`NVIDIA/dcgm-exporter` issue #34 — profiling privileges.** dcgm-exporter fails to start
  on an A100/MIG node with `CacheManager Init Failed. Error: -17` and
  `Error starting nv-hostengine: DCGM initialization error`, having warned
  *"dcgm-exporter doesn't have sufficient privileges to expose profiling metrics. To get
  profiling metrics with dcgm-exporter, use `--cap-add SYS_ADMIN`."*
  **What it shows:** the privilege requirement is a real first-encounter production failure,
  and the fix is now baked into the shipped Helm chart's `securityContext`. The historical
  value is the *warning text*, because that string is what you grep exporter logs for when a
  hardened cluster re-creates the bug. Note the image tag in the report (`2.3.1-2.6.1`) —
  this is a 2021-era report against a much older stack; the mechanism survives, the specific
  init error does not.

- **`NVIDIA/dcgm-exporter` issue #22 — "Profiling metrics not being collected."** On a
  Tesla K80, `dcgmi` reports *"This request is serviced by a module of DCGM that is not
  currently loaded"* and `dcgmi modules -l` shows `Profiling: Failed to load` while Core and
  NvSwitch load normally.
  **What it shows:** the module table is the diagnostic surface, and `Failed to load` vs
  `Not loaded` is a meaningful distinction — the first says DCGM tried and the hardware or
  package could not support it, the second says nothing has asked yet. It also anchors the
  generation boundary: the profiling path is Volta and newer.

- **dcgm-exporter's own Helm chart as a spec.** `deployment/values.yaml` at 4.8.3 encodes
  most of this lesson's conclusions as defaults: `securityContext.capabilities.add:
  ["SYS_ADMIN"]` with the inline justification, `serviceMonitor.interval: 30s` matched to the
  30,000 ms collect interval, `scrapeTimeout: 25s`, `resources.limits: 200m CPU / 512Mi`,
  and `honorLabels: false`.
  **What it shows:** the vendor's own defaults are aligned scrape-and-watch at 30 s with
  profiling privileges on — a useful sanity check that the reasoning in §7 lands where the
  people who wrote the code landed.

- **NVIDIA's documented V100-vs-T4 asymmetry.** NVIDIA's DCGM profiling documentation states
  that SM Activity and SM Occupancy cannot be collected together with Tensor Utilization on
  a V100 but can be on a T4, and that DCGM handles the over-subscribed case by *"automatic
  multiplexing of metrics by statistically sampling the requested metrics and performing the
  groupings internally"*, which *"may be transparent to users"*.
  **What it shows:** the co-residency question is answered by silicon, not by DCGM version,
  and it is answered differently on two GPUs from adjacent generations. This is why the
  correct operational answer is "run `dcgmi profile -l`", not "here is the safe set".

## Worked example

**Scenario.** A 65-node fleet: 40 nodes of 8×A100-SXM4 (older training cluster), 25 nodes of
8×H100-SXM5 (new). A platform engineer enables the full PROF field set fleet-wide at
`--collect-interval 1000` because "we want 1-second resolution for the efficiency dashboard".
Within a day: the H100 dashboards look great, the A100 dashboards are noisy and implausible,
and about a fifth of exporter pods are restarting. Diagnose it end to end.

**Step 1 — establish what was asked for.** The custom counters CSV enables the full
1001–1016 range plus PCIe and NVLink:

```
DCGM_FI_PROF_GR_ENGINE_ACTIVE,   gauge, Ratio of time the graphics engine is active.
DCGM_FI_PROF_SM_ACTIVE,          gauge, The ratio of cycles an SM has at least 1 warp assigned.
DCGM_FI_PROF_SM_OCCUPANCY,       gauge, The ratio of number of warps resident on an SM.
DCGM_FI_PROF_PIPE_TENSOR_ACTIVE, gauge, Ratio of cycles any tensor pipe is active.
DCGM_FI_PROF_DRAM_ACTIVE,        gauge, Ratio of cycles the device memory interface is active.
DCGM_FI_PROF_PIPE_FP64_ACTIVE,   gauge, Ratio of cycles the fp64 pipes are active.
DCGM_FI_PROF_PIPE_FP32_ACTIVE,   gauge, Ratio of cycles the fp32 pipes are active.
DCGM_FI_PROF_PIPE_FP16_ACTIVE,   gauge, Ratio of cycles the fp16 pipes are active.
DCGM_FI_PROF_PCIE_TX_BYTES,      gauge, PCIe transmit rate (bytes/sec).
DCGM_FI_PROF_PCIE_RX_BYTES,      gauge, PCIe receive rate (bytes/sec).
DCGM_FI_PROF_NVLINK_TX_BYTES,    gauge, Aggregate NVLink transmit rate (bytes/sec).
DCGM_FI_PROF_NVLINK_RX_BYTES,    gauge, Aggregate NVLink receive rate (bytes/sec).
```

**Step 2 — rule out privileges first, because it is the cheapest check and its symptom
overlaps with everything else.**

```console
$ kubectl -n gpu-operator get ds nvidia-dcgm-exporter \
    -o jsonpath='{.spec.template.spec.containers[0].securityContext}{"\n"}'
{"capabilities":{"add":["SYS_ADMIN"],"drop":["ALL"]},"runAsUser":0}

$ kubectl -n gpu-operator logs ds/nvidia-dcgm-exporter | grep -iE 'SYS_ADMIN|REQUIRES_ROOT|NO_PERMISSION'
(no output)
```

Capability present, no permission errors. Not §6(a).

**Step 3 — ask the hardware what it can co-collect.** This is the step people skip.

```console
# On an H100 node
$ dcgmi profile -l -i 0 | awk 'NR>4 {print $2}' | sort -u
A.0
B.0
C.0

# On an A100 node
$ dcgmi profile -l -i 0 | awk 'NR>4 {print $2}' | sort -u
A.0
A.1
B.0
C.0
D.0
```

There it is. The H100s report three metric groups, all with distinct major IDs — every
watched field is single-pass, which is the GPM behaviour from §5. The A100s report a letter
(`A`) with two subgroups, so the requested field set spans mutually-exclusive groups and is
being multiplexed. **Same config, same exporter, same DCGM version, two different answers,
decided by silicon.**

**Step 4 — quantify the multiplexing.** Look at the A100 layout in detail and count which
of the requested fields fall where:

```console
$ dcgmi profile -l -i 0
+----------------+----------+------------------------------------------------------+
| Group.Subgroup | Field ID | Field Tag                                            |
+================+==========+======================================================+
| A.0            | 1001     | gr_engine_active                                     |
| A.0            | 1002     | sm_active                                            |
| A.0            | 1003     | sm_occupancy                                         |
| A.1            | 1004     | tensor_active                                        |
| A.1            | 1006     | fp64_active                                          |
| A.1            | 1007     | fp32_active                                          |
| A.1            | 1008     | fp16_active                                          |
| B.0            | 1005     | dram_active                                          |
| C.0            | 1009     | pcie_tx_bytes                                        |
| C.0            | 1010     | pcie_rx_bytes                                        |
| D.0            | 1011     | nvlink_tx_bytes                                      |
| D.0            | 1012     | nvlink_rx_bytes                                      |
+----------------+----------+------------------------------------------------------+
```

(Representative of a pre-GPM layout; run it on your own hardware — the grouping is a
per-generation, per-driver fact.)

Two exclusive groups under `A` ⇒ the counters serving `A.0` and `A.1` are programmed
alternately. At `updateFreq = 1 s`, each group is programmed roughly half the wall-clock
time, so each field's estimate is built from about **500 ms of observation per 1-second
reported interval**, and the reported value for the half it was not watching is
extrapolated. That is where "noisy and implausible" comes from: `SM_ACTIVE` and
`TENSOR_ACTIVE` on the same A100 are describing *disjoint* halves of each second, so their
ratio is not a meaningful quantity at that resolution.

**Step 5 — explain the restarts.** Work the sampling load:

```
  A100 node, 8 GPUs × 12 PROF fields  = 96 (entity, field) pairs
  plus the ~14 enabled NVML device fields × 8 = 112 pairs
                                       ────────────────────────
                                        208 pairs

  at updateFreq 1 s   →  208 engine samples/s/node
  at updateFreq 30 s  →  ~6.9 engine samples/s/node       (30× less)

  Chart resources:  requests 100m CPU / 128Mi
                    limits   200m CPU / 512Mi
```

At 200 millicores the exporter has one fifth of a core. Multiplexed sampling on the A100s
adds counter-reprogramming work on top of the raw sample rate. The pods hit the CPU limit,
scrapes slow past the chart's `scrapeTimeout: 25s`, the liveness/readiness path degrades,
pods restart. **The restarts are on the A100 nodes only, which is the tell** — same
config, more work per sample because of the multiplexing.

**Step 6 — the fix, stated as a config diff and a justification.**

```yaml
# values.yaml
arguments: []                 # leave embedded mode: one consumer per node
extraEnv:
  - name: DCGM_EXPORTER_INTERVAL
    value: "30000"            # back to 30 s, aligned with the ServiceMonitor
serviceMonitor:
  interval: 30s
  scrapeTimeout: 25s
```

and a counters CSV trimmed to the fields that answer a question someone asks:

```
DCGM_FI_PROF_SM_ACTIVE,          gauge, The ratio of cycles an SM has at least 1 warp assigned.
DCGM_FI_PROF_SM_OCCUPANCY,       gauge, The ratio of number of warps resident on an SM.
DCGM_FI_PROF_PIPE_TENSOR_ACTIVE, gauge, Ratio of cycles any tensor pipe is active.
DCGM_FI_PROF_DRAM_ACTIVE,        gauge, Ratio of cycles the device memory interface is active.
```

On the A100s this is `A.0` (1002, 1003), `A.1` (1004) and `B.0` (1005) — still two exclusive
groups, still multiplexed, but only two rather than the earlier span, and at 30 s each field
gets ~15 s of observation per reported interval instead of ~500 ms, which is enough for the
estimate to be stable. On the H100s it is one group and single-pass. **You have not
eliminated multiplexing on the A100s — you cannot, it is the silicon — you have made the
sampling interval long enough that the statistical estimate is trustworthy.**

**Step 7 — record the fleet fact, so nobody re-derives it.** Commit a short table alongside
the values file:

| GPU model | Distinct majorIds | Multi-subgroup letters | Multiplexes? | Notes |
|---|---|---|---|---|
| H100-SXM5 | A, B, C | none | **No** | GPM: 1001–1034 all in A.0 |
| A100-SXM4 | A, B, C, D | A (A.0/A.1) | **Yes** | tensor + pipe fields exclusive with SM fields |

That table is the artifact. It is two lines, it took one command per model to produce, and
it converts "profiling multiplexes, be careful" into an answer.

## Practice

On a rented GPU with DCGM installed, feeding the *"why our scrape config is what it is"*
appendix of ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md).
If you have no hardware, the
[fake GPU fleet lab](../../04-gpu-on-kubernetes/practice/fake-gpu-fleet/README.md) will let
you build the queries and the config, but §5 and §6(b) genuinely require real silicon.

1. **Enumerate.** `dcgmi discovery -l` to list GPUs and their UUIDs. `dcgmi modules -l` to
   see which modules are loaded; note that `Profiling` is probably `Not loaded` before you
   watch anything, and check again after step 3.

2. **Map the metric groups — the load-bearing step.** `dcgmi profile -l -i 0`. Record the
   full output. Then reduce it:
   `dcgmi profile -l -i 0 | awk 'NR>4 {print $2}' | sort | uniq -c`. **Write down whether any
   letter has more than one subgroup.** That single fact determines whether multiplexing is
   a concern on this hardware, and it is the first thing to check on every new GPU model
   your fleet acquires.

3. **Verify privileges before anything else.** Confirm the container or your shell can
   actually read PROF fields — `dcgmi dmon -e 1002 -c 3` should print numbers, not `N/A`.
   If it prints `N/A`, stop and fix that; a permissions gap and a multiplexing artefact
   produce confusingly similar-looking data.

4. **Watch and observe.** `dcgmi dmon -e 1002,1003,1004,1005 -d 1000` while a real workload
   runs. Note the columns are the registered short names (`SMACT`, `SMOCC`, `TENSO`,
   `DRAMA`) and values print to three decimals; the header reprints every 14 rows.

5. **Force the multi-pass refusal (pre-GPM hardware only).** If step 2 showed multiple
   subgroups under multiple letters, watch enough mutually-exclusive groups to exceed five
   and confirm you get `DCGM_ST_PROFILING_MULTI_PASS` (-38) rather than degraded data. On
   Hopper+ this is not reproducible — **and confirming that it is not reproducible is itself
   the result to record.**

6. **Pause and resume.** `dcgmi profile --pause`, confirm `dcgmi modules -l` shows
   `Profiling: Paused` and that `dcgmi dmon -e 1002` stops producing fresh values, then
   `dcgmi profile --resume` and confirm recovery. Time the gap.

7. **Measure the interval effect directly.** Run the exporter at
   `DCGM_EXPORTER_INTERVAL=30000` with a 30 s scrape, then at `5000` with a 5 s scrape,
   against the same bursty workload (a training loop with a deliberate 20-s-busy /
   40-s-idle cycle is ideal). Compare `avg_over_time(DCGM_FI_PROF_SM_ACTIVE[10m])` between
   the two. Then run the pathological case — 30 s watch, 5 min scrape — and show the number
   is now a function of scrape phase.

8. **Consolidate onto a standalone engine (if you can).** Start
   `nv-hostengine -n --port 5555`, run the exporter with `-r tcp://localhost:5555`, and
   confirm `dcgmi dmon` and the exporter can both read PROF fields concurrently without
   either degrading. Compare with the same test using the embedded default.

**Acceptance:** a written note capturing (i) the `dcgmi profile -l` metric-group table for
every GPU model you have access to, with the multiplexes-yes/no verdict; (ii) confirmation
that privileges were verified, and what the failure looked like when you removed them;
(iii) your chosen fleet `--collect-interval` **and** Prometheus scrape interval **with the
derivation** — the alignment argument on the upper side, the retention/estimation and
CPU-cost arguments on the lower; and (iv) the measured before/after from step 7 showing what
a misaligned scrape does to a bursty workload's reported average. That note is the "why our
scrape config is what it is" appendix, and it is the section of the blog post that persuades
readers you actually ran this rather than repeating it.

## Common pitfalls

1. **Assuming DCGM always silently multiplexes.** It multiplexes up to
   `DLG_MAX_METRIC_GROUPS = 5` mutually-exclusive groups and then **refuses** with
   `DCGM_ST_PROFILING_MULTI_PASS` (-38). **Symptom of the refusal:** the watch is not
   established at all, so you lose *every* field in that field group, not just the excess
   ones. **Mechanism:** DCGM's own test suite asserts both branches — success up to five
   groups, `assert_raises` beyond.

2. **Carrying a "safe co-resident field set" across a hardware refresh.** Co-residency is a
   property of the generation's counter layout. **Symptom:** a config that produced clean
   data on T4s produces noisy estimates on V100s, or vice versa. **Fix:** `dcgmi profile -l`
   per GPU model; group by the letter; a letter with multiple subgroups is your answer.

3. **Assuming multiplexing is still a concern on Hopper and newer.** On GPM GPUs, DCGM
   synthesises exactly three metric groups — 1001–1034 in `A.0`, the NVLink/C2C/cache range
   in `B.0`, the cumulative cycle counters in `C.0` — all with `minorId = 0` and distinct
   `majorId`, so all three can be watched together. **Mechanism:** `nvmlGpmSampleGet()`
   snapshots every metric in one call; there is nothing to trade off. Claiming otherwise in
   an interview about an H100 fleet is a tell.

4. **Scraping Prometheus faster than the DCGM watch interval.** **Symptom:** a "1-second"
   dashboard that is visibly a staircase — the same value repeated thirty times. **Mechanism:**
   DCGM returns the cached value from the last completed interval; the exporter has nothing
   newer to give you. You bought thirty times the samples and zero extra information.

5. **Scraping slower than the watch interval.** **Symptom:** the same query returns wildly
   different averages on different days. **Mechanism:** each stored point describes one
   `updateFreq` window; the rest of the wall clock is discarded. A 30 s watch on a 5 min
   scrape observes 10% of time, chosen by scrape phase.

6. **Running two embedded engines on one node.** dcgm-exporter embeds by default and does
   *not* auto-detect a running `nv-hostengine`; `UseRemoteHE` is only set when `-r` is
   explicitly passed. **Symptom:** contention over the profiling counters that neither
   process reports. **Fix:** exactly one owner — standalone engine plus `-r` for every
   client (DCGM issue #287).

7. **Treating a missing PROF series as an idle GPU.** Blank/unsupported/not-permissioned
   values are *omitted* by the exporter, not zeroed. **Symptom:** `avg by (namespace)` looks
   healthy because the broken GPUs left the average. **Fix:** the `unless` absence alert from
   05.1 Rule 5, evaluated over several scrapes so the exporter's own watch-repair
   (2× interval) does not page you.

8. **Forgetting that an external profiler steals the counters.** `ncu` or `nsys` on a shared
   node takes the performance monitors, with or without `dcgmi profile --pause`.
   **Symptom:** an unexplained multi-minute gap in `SM_ACTIVE` on one node. **Fix:** correlate
   gaps against node process history before opening a hardware ticket, and teach the team the
   pause/resume etiquette.

9. **Assuming `maxKeepAge` is the PROF retention horizon.** dcgm-exporter passes
   `maxKeepAge = 600.0` s, but the GPM sample map prunes on a *derived* horizon,
   `2 × maxUpdateInterval + min(maxUpdateInterval, 1 s)` — 61 s at a 30 s interval.
   **Consequence:** you cannot query DCGM for "SM_ACTIVE ten minutes ago" and get a GPM-backed
   answer; that history lives in Prometheus, not in the engine.

## Self-check

- **Standalone vs embedded — which does dcgm-exporter use by default, and when do you
  override?** *Answer:* embedded, in-process via `libdcgm.so`. The exporter calls
  `dcgm.Init(dcgm.Embedded)` unless `-r/--remote-hostengine-info` was *explicitly set* — the
  flag's `localhost:5555` default is inert, and there is no auto-detection of a running
  daemon. Override to standalone whenever a second consumer on the node needs PROF fields —
  another exporter, a health-check sidecar, or routine `dcgmi diag`/`dmon` — so that one
  engine owns the counters and the watches are unioned rather than contended. The engine's
  own defaults are port 5555, bind `127.0.0.1`, PID file `/var/run/nvhostengine.pid`, and a
  Unix socket at `/tmp/nv-hostengine` if you pass `-d` (which suppresses the TCP listener
  entirely).

- **Why can't you sample every `PROF_*` field at 1 s on one GPU?** *Answer:* it depends on
  the GPU, and the honest answer names both branches. On **pre-Hopper** silicon the counter
  banks hold one counter-select at a time, so metrics deriving from the same bank land in
  metric groups sharing a `majorId`, and the header rule is that same-`majorId` groups
  *cannot be watched concurrently*. DCGM time-multiplexes up to `DLG_MAX_METRIC_GROUPS = 5`
  such groups, statistically sampling each, and refuses beyond that with
  `DCGM_ST_PROFILING_MULTI_PASS` (-38). Each multiplexed field then gets only a fraction of
  each interval as real observation, so shortening the interval to 1 s makes each estimate
  worse. On **Hopper and newer** the premise is false: GPM snapshots all metrics in one
  `nvmlGpmSampleGet()` call, DCGM reports 1001–1034 as a single group `A.0`, and there is no
  multiplexing to avoid — the remaining limits are engine CPU and the fact that GPM
  retention is only `2×interval + jitter slack`. Run `dcgmi profile -l` to find out which
  world you are in.

- **What is `majorId`/`minorId`, and how do you read `dcgmi profile -l`?** *Answer:*
  `dcgmi profile -l` prints one row per supported PROF field as `Group.Subgroup | Field ID |
  Field Tag`, where the letter is `'A' + majorId` and the number after the dot is `minorId`.
  Per `dcgm_structs.h`, *metric groups with the same `majorId` cannot be watched concurrently
  with other groups of the same `majorId`* — so `A.0` and `A.1` are mutually exclusive and
  will multiplex, while `A.0` and `B.0` are free to co-reside. The practical procedure is:
  list, take the letters, and look for any letter appearing with more than one subgroup.

- **What privilege does the profiling path need, and why is the failure hard to see?**
  *Answer:* it programs hardware performance counters, which is privileged — in a container,
  `CAP_SYS_ADMIN` (the exporter's Helm chart ships
  `capabilities: add: ["SYS_ADMIN"]` with the comment *"Required for profiling metrics
  (DCGM_FI_PROF_*)"*), or root for a standalone `nv-hostengine`. It is hard to see because
  the failure is *silent by construction*: DCGM returns a blank/`NOT_PERMISSIONED` value,
  dcgm-exporter drops blanks rather than exporting zeros, so the series simply does not
  appear. NVML device fields keep flowing the whole time, so the dashboard looks alive.
  DCGM status codes `DCGM_ST_NO_PERMISSION` and `DCGM_ST_REQUIRES_ROOT` (-29) are what the
  exporter's classifier maps to the CAP_SYS_ADMIN hint in the logs.

- **What sets the averaging window of `SM_ACTIVE`, and what is it by default?** *Answer:*
  the watch's `updateFreq`. DCGM computes the ratio by differencing two GPM samples separated
  by that interval, so the value is a mean over exactly that window. dcgm-exporter passes
  `--collect-interval` (default **30,000 ms**) as the watch interval, with `maxKeepAge = 600.0`
  s and `maxKeepSamples = 0`, so out of the box each scraped `SM_ACTIVE` is a ~30-second
  mean. It is a knob, not a constant: change it and you change what the number *means*, not
  just how fresh it is.

- **Why must the Prometheus scrape interval equal the watch interval?** *Answer:* because
  PROF fields are interval statistics, not instantaneous gauges. Scraping faster re-reads the
  same cached value — duplicate points, no extra information. Scraping slower discards whole
  completed intervals: a 30 s watch scraped every 5 minutes shows you 30 s out of every
  300 s, a 10% duty cycle whose content is decided by scrape phase rather than by the
  workload. For a bursty training job that is the difference between reporting 0.9 and 0.05
  for the same hour.

- **Three PROF series vanish from one node for eleven minutes, then return. Rank your
  hypotheses.** *Answer:* (1) somebody ran `ncu`/`nsys` on the node, with or without
  `dcgmi profile --pause` — external profilers and DCGM cannot both own the counters, and
  eleven minutes is a very human duration; check `dcgmi modules -l` for `Profiling: Paused`
  and the node's process history. (2) The exporter's stale-watch repair fired — it declares a
  profiling stream stale after 2× the watch interval, discards profiling metrics from that
  scrape, re-arms and retries, which shows as absence; look for repair messages in the
  exporter log. (3) A second embedded engine appeared on the node (a health-check sidecar
  rolled out) and is contending. (4) The exporter restarted — the first scrape after a
  restart has no PROF series because GPM has no baseline sample yet. Note that a *permission*
  change would not fit the shape: that produces permanent absence from process start, not an
  eleven-minute window.

- **`dcgmi modules -l` shows `Profiling: Not loaded` on a node with healthy GPUs. Is
  something broken?** *Answer:* almost certainly not. DCGM modules are lazily loaded; the
  Profiling module is loaded on first demand for a `DCGM_FI_PROF_*` field on a non-GPM GPU.
  `Not loaded` means nothing has asked yet. The states that indicate a real problem are
  `Failed to load` (hardware too old, or the proprietary DCP component missing on
  Volta–Ampere), `Denylisted` (someone ran `dcgmi modules --denylist`), and `Paused` (someone
  ran `dcgmi profile --pause`).

## Connections & what's next

This lesson turns 05.1's "trust these fields" into something you can actually operate: you
now know who owns the counters, what a field group and a watch are, what the exporter passes
for `updateFreq`/`maxKeepAge`, why co-residency is a silicon question answered by
`dcgmi profile -l`, and what each of the three disappearance mechanisms looks like. Lesson
05.3 sits directly on top: dcgm-exporter's counter CSV *is* the field group this lesson
described, its `--collect-interval` *is* the watch `updateFreq`, and the audit it performs —
`SM_ACTIVE` and `SM_OCCUPANCY` ship commented out — is a field-group decision made by
someone else on your behalf, partly because of the costs derived here. 05.3 also takes the
cardinality arithmetic that §7 only touched (sample rate) and works the series-count side
properly. Lesson 05.4 then depends on §4's entity model: `SM_ACTIVE` is registered at
`DCGM_FE_GPU_CI` and `DRAM_ACTIVE` at `DCGM_FE_GPU_I`, which is exactly why MIG preserves
per-tenant attribution and device-level sharing does not. Lesson 05.5 reuses the module
model for the health path, and 05.7's profiling-escalation ladder is a direct consequence of
§6(b): DCGM and Nsight compete for the same counters, which is *why* `dcgmi profile --pause`
exists at all.

## References & further reading

**Primary sources**

- `NVIDIA/DCGM` — `dcgmlib/dcgm_structs.h` (DCGM 4.6.0) — https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/dcgm_structs.h — read for: the `dcgmProfMetricGroupInfo_v2` struct and its `majorId`/`minorId` co-residency rule (§5); `DCGM_PROF_MAX_NUM_GROUPS_V2 = 10` and `DCGM_PROF_MAX_FIELD_IDS_PER_GROUP_V2 = 64`; the status codes `DCGM_ST_REQUIRES_ROOT` (-29), `DCGM_ST_MODULE_NOT_LOADED` (-33), `DCGM_ST_PROFILING_NOT_SUPPORTED` (-36), `DCGM_ST_PROFILING_LIBRARY_ERROR` (-37) and `DCGM_ST_PROFILING_MULTI_PASS` (-38); and the eleven module IDs with their seven status values.
- `NVIDIA/DCGM` — `modules/core/DcgmModuleCore.cpp`, `ProcessProfGetMetricGroups()` — https://github.com/NVIDIA/DCGM/blob/master/modules/core/DcgmModuleCore.cpp — read for: the `EntityPairSupportsGpm()` branch and the three synthesised GPM metric groups. This is the primary evidence that Hopper+ does not multiplex the activity ratios. *Correction vs. the previous version of this lesson: multiplexing is a pre-GPM behaviour, not a universal one.*
- `NVIDIA/DCGM` — `testing/python3/tests/test_prof.py` — https://github.com/NVIDIA/DCGM/blob/master/testing/python3/tests/test_prof.py — read for: `DLG_MAX_METRIC_GROUPS = 5`, the definition of a multi-pass field set as "same `majorId`, different `minorId`", and the `assert_raises(DCGM_ST_PROFILING_MULTI_PASS)` negative test. *Correction: DCGM does not always silently multiplex — beyond five exclusive groups the watch is refused.*
- `NVIDIA/DCGM` — `hostengine/src/HostEngineCommandLine.cpp` — https://github.com/NVIDIA/DCGM/blob/master/hostengine/src/HostEngineCommandLine.cpp — read for: every `nv-hostengine` flag and default in §2's table, including the `127.0.0.1` bind default and the socket/vsock mutual exclusions.
- `NVIDIA/DCGM` — `dcgmi/DcgmiProfile.cpp`, `dcgmi/Module.cpp`, `dcgmi/DeviceMonitor.cpp`, `dcgmi/CommandLineParser.cpp` — https://github.com/NVIDIA/DCGM/tree/master/dcgmi — read for: the `dcgmi profile -l` column layout and the `'A' + majorId` rendering; the `dcgmi modules -l` column widths and state strings; `dcgmi dmon`'s `-d` default of 1000 ms, three-decimal formatting, `N/A` for blanks and 14-row header repeat; and the `--pause`/`--resume` help text naming nvprof/Nsight.
- `NVIDIA/DCGM` — `dcgmlib/src/DcgmGpmManager.cpp` — https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/src/DcgmGpmManager.cpp — read for: `GetDerivedMaxSampleAge()` = `2 × maxUpdateInterval + min(maxUpdateInterval, 1 s)`, and the baseline-lookup logic that makes the watch interval the averaging window.
- `NVIDIA/dcgm-exporter` — `internal/pkg/devicewatcher/` — https://github.com/NVIDIA/dcgm-exporter/tree/main/internal/pkg/devicewatcher — read for: `maxKeepAge = 600.0`, `maxKeepSamples = 0`, and `watchFieldGroupSimple()` converting the watch group's `IntervalMSec` to microseconds.
- `NVIDIA/dcgm-exporter` — `internal/pkg/dcgmprovider/dcgm.go` and `pkg/cmd/app.go` — https://github.com/NVIDIA/dcgm-exporter/tree/main/internal/pkg/dcgmprovider — read for: the embedded-vs-standalone branch, the fact that `UseRemoteHE` is set only when `-r` is explicitly passed, the `--collect-interval` default of 30,000 ms, and the remote-hostengine URI grammar in the flag's usage string.
- `NVIDIA/dcgm-exporter` — `internal/pkg/collector/gpu_collector.go` — https://github.com/NVIDIA/dcgm-exporter/blob/main/internal/pkg/collector/gpu_collector.go — read for: `resolveProfilingStaleWindows()` (2× the watch interval) and the repair-and-retry path that produces intermittent PROF absence.
- `NVIDIA/dcgm-exporter` — `internal/pkg/dcgmerrors/dcgm_errors.go` — https://github.com/NVIDIA/dcgm-exporter/blob/main/internal/pkg/dcgmerrors/dcgm_errors.go — read for: the exact operator-facing hint strings for `DCGM_ST_NO_PERMISSION`, `DCGM_ST_REQUIRES_ROOT`, `DCGM_ST_PAUSED` and `DCGM_ST_CONNECTION_NOT_VALID`. These are the strings to grep for in exporter logs.
- `NVIDIA/dcgm-exporter` — `README.md` and `deployment/values.yaml` (4.8.3) — https://github.com/NVIDIA/dcgm-exporter — read for: the `collection.watchGroups` YAML schema with its glob matching and one-group-per-field rule; the `securityContext.capabilities.add: ["SYS_ADMIN"]` default; `serviceMonitor.interval: 30s` / `scrapeTimeout: 25s`; pod resource requests and limits; and the note that Ampere-and-earlier profiling depends on the `datacenter-gpu-manager-4-proprietary` package.
- NVIDIA DCGM documentation — profiling and feature overview — https://docs.nvidia.com/datacenter/dcgm/latest/dcgm-api/dcgm-api-profiling.html and https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html — read for: the prose description of automatic multiplexing by statistical sampling, and the documented V100-vs-T4 co-residency example (SM Activity / SM Occupancy vs Tensor Utilization).

**Real-world engineering issues**

- `NVIDIA/DCGM` issue #287, "How to correctly use the nv-hostengine on Kubernetes" — https://github.com/NVIDIA/DCGM/issues/287 — what it shows: two embedded engines on one node, discovered by inspection rather than by any alert; the TCP-vs-Unix-socket and host-process-vs-container questions stated by a real operator, still unanswered upstream.
- `NVIDIA/dcgm-exporter` issue #34 — https://github.com/NVIDIA/dcgm-exporter/issues/34 — what it shows: the `--cap-add SYS_ADMIN` warning text and an A100/MIG init failure; the origin of the capability now shipped by default in the Helm chart. Note this is a 2021-era report against image `2.3.1-2.6.1`; the mechanism persists, the specific init error does not.
- `NVIDIA/dcgm-exporter` issue #22 — https://github.com/NVIDIA/dcgm-exporter/issues/22 — what it shows: `Profiling: Failed to load` in `dcgmi modules -l` on pre-Volta hardware, and `DCGM_ST_MODULE_NOT_LOADED` surfacing as *"This request is serviced by a module of DCGM that is not currently loaded"*. The module table as a diagnostic surface.

**Deeper dives**

- Prometheus — storage and configuration docs — https://prometheus.io/docs/prometheus/latest/storage/ and https://prometheus.io/docs/prometheus/latest/configuration/configuration/ — go deeper on: `needed_disk_space = retention_time_seconds × ingested_samples_per_second × bytes_per_sample` with 1–2 bytes/sample (used in §7's arithmetic), the 15 d default retention, and the `scrape_interval` (1 m) / `scrape_timeout` (10 s) / `honor_labels` (false) defaults.
- DigitalOcean — "How to Enable GPU Metrics on GPU Droplets with DCGM" — https://docs.digitalocean.com/products/droplets/how-to/gpu/enable-metrics/ — go deeper on: a major GPU cloud's customer-facing `nv-hostengine` setup, as a worked example of the standalone deployment decision at provider scale.
</content>
