---
lesson: "02b.8"
title: "Capstone — read one real GPU node end to end"
module: "02b"
concept: "Capstone — topology teardown"
status: not-started
est_time: "9h"
prev: "07-power-and-thermals.md"
next: null
artifacts: []
sources: 13
---

# 02b.8 · Capstone — read one real GPU node end to end

> **Concept.** Reconcile lstopo + lspci -tv + nvidia-smi topo -m + numactl --hardware into ONE coherent topology diagram for a real GPU node, predict its throughput failure modes, and measure the cost of a NUMA misalignment.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Where this fits

Every lesson in this module handed you one lens on the same machine: the NUMA tree (01–02), the PCIe fabric and its trained link state (03), the reference 8-GPU baseboard (04), Kubernetes' topology-aware placement (05), storage and GDS placement (06), and power/thermal throttling with its adjacent Xid/SXid fault layer (07). None of those lessons asked you to hold all of it in your head at once, on one real box, and decide what is actually true when two tools disagree.

That is this lesson. It closes the module by forcing the synthesis: reconcile four tools that each show a different *projection* of the same silicon into one diagram, predict where a job will bottleneck, then prove it with a measurement. It is deliberately procedural — exact commands in order, what each output means, how each finding lands in the deliverable — because the skill being certified is not "knowing about topology," it is **producing a defensible topology teardown of a machine you have never seen before, in one sitting, without looking anything up.**

## Why this matters

This is the whiteboard task. *"Draw me a DGX/HGX node and tell me where the bottleneck is"* is asked, in some form, in nearly every senior platform interview at a GPU-heavy shop. Not because anyone wants a memorised diagram, but because reconciling four tools that each show a different slice of the same hardware, and then reading off the consequences, is exactly the on-the-job diagnosis when a training job runs at half spec and the dashboards are green.

The failure class it prevents is expensive and silent. Every item below is a real percentage off a $2–4/GPU-hr rental, multiplied by every hour the job runs, and **none of them throws an error**:

| Defect | Cost | What reports it |
|---|---|---|
| GPU paired to a NIC across the socket link (`SYS`) | GPUDirect RDMA over UPI instead of a local PCIe switch — roughly halved effective RDMA bandwidth plus added latency | nothing |
| A ×16 link trained at ×4 (`LnkSta` ≠ `LnkCap`) | up to 4× host↔device bandwidth cap | nothing |
| Dataset NVMe on the wrong socket | one UPI hop on every data-loader read; bimodal step times | nothing |
| Pod admitted cross-NUMA under `best-effort` | remote memory on every H2D copy | nothing |
| A clock pinned at base by a power cap | ~20% fewer tensor FLOPs per second | `clocks.sm` vs `clocks.max.sm`, which nobody charts |
| A genuinely faulty component (Xid/SXid) | anything from throughput loss to a mid-run crash | `dmesg`, which nobody greps |

The capstone deliverable — a reconciled diagram, a measured aligned-versus-misaligned delta, and the exporter you would build — is the portfolio artifact that says you can find this class of waste. That is the cost/observability differentiator this whole track is built around.

The stakes compound at fleet scale. Meta's published data from training Llama 3 on a 16,384-GPU cluster reported **419 unexpected component failures over a 54-day run** — on the order of one every few hours — with GPU and HBM3 issues responsible for roughly half. At that cadence, manual per-incident reconciliation cannot be the production answer; it has to become the *foundation* skill automated fleet tooling is built on. Knowing both — how to do it yourself, and that real shops have automated it — is the difference between "I did this once on a rented box" and "I can build a first version of what production fleet-health systems run continuously."

## What's new here (calibration)

- **You already know each tool on its own.** After lessons 01–07 you can read each independently: what a NUMA node is, what `LnkCap` versus `LnkSta` means, how an 8-GPU HGX baseboard is wired, how Topology Manager merges hints, where NVMe sits, and how to decode a clocks-event bitmask. **None of the individual tools are new.**
- **What is genuinely new is that no single tool is authoritative for the whole node.** Each has a blind spot the others fill, and the skill being tested is *cross-checking* them into one picture and then *predicting* what it will do under load. You stop asking "what does this output mean" and start asking "these three disagree about GPU3's home node — which is right, and what breaks because of it."
- **New: identity mapping as a first-class step.** `lspci` calls a GPU `0000:9d:00.0`, `lstopo` calls it `cuda4`, `nvidia-smi` calls it `GPU4`, CUDA may enumerate it as device 0, and the kubelet knows it only by UUID. Nothing you say about "GPU4's link speed" means anything until that mapping exists.
- **New: a documented case of two NVIDIA-adjacent tools disagreeing** about the link class of the same edge on the same silicon.
- **New: the hardware fault as a distinct seventh hypothesis.** An Xid/SXid entry can produce a symptom identical to a placement bug and needs a repair ticket, not `numactl`.
- **New: the node-level scorecard.** Bandwidth, checkpoint time, power and cooling arithmetic computed together, so the diagram becomes a capacity statement rather than a picture.

## Core concepts

### 1. Four projections of one machine

Each tool sees the node through a different lens. The reason you need all four is that each one's blind spot is exactly another one's strength.

| Tool | Owns (trust it for this) | Blind to |
|---|---|---|
| `numactl --hardware` | NUMA node count, CPU→node mapping, memory per node, the SLIT distance matrix | **All** PCIe and device information. It cannot place a single GPU. |
| `lstopo` (hwloc) | The unified compute+I/O locality tree: `Machine → Package → NUMANode → caches → cores`, *and* `HostBridge → PCIBridge → device`. **The authoritative source for "which socket owns this device."** | The live NVLink mesh (NVLink is above PCIe and does not appear as a clean matrix); trained-under-load link state. |
| `lspci -tv` + `lspci -vvv` | The raw PCIe hierarchy (root complex → bridges/switches → endpoints) and — uniquely — **`LnkCap` vs `LnkSta`**, the trained speed and width, plus AER error counters. | NVLink entirely (it is not a PCIe link); friendly NUMA affinity; `cudaN`-style identity. |
| `nvidia-smi topo -m` | The GPU↔GPU and GPU↔NIC link-class matrix (`NV#`/`PIX`/`PXB`/`PHB`/`NODE`/`SYS`) plus per-GPU `CPU Affinity` and `NUMA Affinity`. | **NVMe devices — it lists none.** Flattens PCIe to a single letter, so a `PIX` link trained at ×4 looks fine. |

Two more belong in the same sweep, because their findings change the diagram's annotations: `nvidia-smi -q -d POWER,TEMPERATURE,CLOCK,PERFORMANCE` (plus `dcgmi dmon`) owns *whether the GPUs are delivering their rated clock and which constraint is binding*; `dmesg -T | grep -iE 'xid|sxid'` owns *whether any component has actually faulted*, which changes the remediation from "re-pin" to "replace".

### 2. `numactl --hardware` — the frame

```
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0-31 64-95
node 0 size: 515655 MB
node 0 free: 501004 MB
node 1 cpus: 32-63 96-127
node 1 size: 516086 MB
node 1 free: 508871 MB
node distances:
node   0    1
  0:  10   21
  1:  21   10
```

**Reads off:** two NUMA domains; node 0 owns logical CPUs 0–31 and 64–95 (the second range being the SMT siblings of the first — confirm with `lscpu -e`); ~504 GiB per node; a remote-access cost index of 21 against a local 10.

**The distance matrix is firmware-provided SLIT** (System Locality Information Table), where local is *defined* as 10 by convention. Treat `21` as an **index, not a measurement**: real measured latency deltas on two-socket servers run 1.5–1.8×, and SLIT captures nothing about contention, so under concurrent cross-socket load the effective penalty is worse. Same values per node at `/sys/devices/system/node/node0/distance`.

**Blind spot:** it knows nothing about PCIe. It is the frame you hang devices on, not the device map. Anyone answering "where does GPU3 live?" from `numactl` alone is guessing.

### 3. `lstopo` — the authoritative device→socket map

`lstopo` comes from **hwloc**. On a headless box use the text renderer:

```
$ lstopo-no-graphics --of console          # ASCII box drawing, terminal-width aware
$ lstopo-no-graphics --of txt              # plain text tree
$ lstopo --of svg > lstopo.svg             # the picture for the deliverable
$ lstopo-no-graphics --whole-io --of console   # include ALL I/O devices, not just the ones
                                              # hwloc thinks are interesting
```

`--whole-io` matters: by default hwloc filters some I/O devices out of the tree. On a GPU node you want everything.

```
$ lstopo-no-graphics --whole-io --of txt | head -40
Machine (2015GB total)
  Package L#0
    NUMANode L#0 (P#0 1007GB)
    L3 L#0 (105MB)
      ... 32 cores ...
    HostBridge L#0
      PCIBridge
        PCI 0000:1b:00.0 (3D)
          CoProc "cuda0"
          GPU "nvidia0"
      PCIBridge
        PCI 0000:1c:00.0 (InfiniBand)
          Net "ibp27s0"
          OpenFabrics "mlx5_0"
      PCIBridge
        PCI 0000:c1:00.0 (NVMExpress)
          Block(Disk) "nvme0n1"
  Package L#1
    NUMANode L#1 (P#1 1008GB)
    HostBridge L#8
      PCIBridge
        PCI 0000:9d:00.0 (3D)
          CoProc "cuda4"
          GPU "nvidia4"
      PCIBridge
        PCI 0000:9e:00.0 (InfiniBand)
          Net "ibp157s0"
          OpenFabrics "mlx5_4"
```

**Reads off — this is the spine of your whole diagram:** the home NUMA node of every GPU, NIC and NVMe, together with all three of that device's names: the PCI BDF (`0000:9d:00.0`), the CUDA ordinal (`cuda4`), and the OS/verbs name (`mlx5_4`, `nvme0n1`). **`lstopo` is the only tool that prints the BDF and the friendly name side by side**, which makes it your identity-mapping Rosetta stone.

Note the structural fact it makes visible: **a `HostBridge` belongs to exactly one `Package`/`NUMANode`.** That is the physical reason a PCIe device is local to exactly one socket — the root complex containing its root port is integrated into one CPU die, so its DMA reads and writes are issued by that socket's memory-side fabric. Reaching the other socket's DRAM, or a device on it, means traversing the inter-socket link.

**Blind spot:** `lstopo` shows the PCIe link *as capable/configured*, not as trained under load, and gives no usable view of the live NVLink mesh. Cross-check width and speed against `lspci -vvv`.

### 4. `lspci` — the tree, and the only place the *trained* link appears

```
$ lspci -tvD | head -20
-+-[0000:c0]-+-01.1-[c1]----00.0  KIOXIA Corporation NVMe SSD Controller
 |           +-03.1-[c2]----00.0  NVIDIA Corporation GH100 [H100 SXM5]
 |           \-03.2-[c3]----00.0  Mellanox Technologies MT2910 Family [ConnectX-7]
 \-[0000:00]-+-01.1-[01]----00.0  NVIDIA Corporation GH100 [H100 SXM5]
             \-01.2-[02]----00.0  Mellanox Technologies MT2910 Family [ConnectX-7]
```

`-D` prints full domain-qualified addresses, which you want when correlating with sysfs paths. Each `[0000:xx]` at the left margin is a **top-level host bridge — one root complex**. Devices under the same top-level bridge are on the same socket; different top-level bridges usually mean different sockets (confirm against `lstopo`, which labels the Package explicitly).

Then, per device, the two lines that matter most in this entire module:

```
$ sudo lspci -vvv -s 0000:c2:00.0 | grep -E 'LnkCap:|LnkSta:'
        LnkCap: Port #0, Speed 32GT/s, Width x16, ASPM not supported
        LnkSta: Speed 32GT/s, Width x16
```

Healthy: trained speed and width equal the capability. A degraded link looks like this:

```
        LnkCap: Port #0, Speed 32GT/s, Width x16, ASPM not supported
        LnkSta: Speed 8GT/s (downgraded), Width x4 (downgraded)
```

**`(downgraded)` is the word you grep for.** Two independent things can degrade — speed (generation) and width (lane count) — and they multiply:

```
  PCIe per-direction bandwidth = GT/s × lanes × encoding_efficiency / 8

  Gen3 ×8 :  8 GT/s × 8  × (128/130) / 8  =  7.88 GB/s
  Gen4 ×16: 16 GT/s × 16 × (128/130) / 8  = 31.51 GB/s
  Gen5 ×16: 32 GT/s × 16 × (128/130) / 8  = 63.02 GB/s

  Gen3 ×8 vs Gen5 ×16 ratio = 63.02 / 7.88 = 8.0×  exactly
  (2× per generation × 2 generations = 4×, times 2× the width = 8×)
```

An eight-fold host-bandwidth cap, reported by nothing except this one command.

Capture **AER (Advanced Error Reporting) counters** in the same sweep, because a link that *retrains* under load looks healthy in a single `LnkSta` snapshot:

```
$ sudo lspci -vvv -s 0000:c2:00.0 | grep -A6 'Advanced Error Reporting'
        Capabilities: [148 v2] Advanced Error Reporting
                UESta:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- ...
                CESta:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+
                CEMsk:  RxErr+ BadTLP+ BadDLLP+ Rollover+ Timeout+ AdvNonFatalErr+
```

A trailing `+` on a `CESta` (correctable) or `UESta` (uncorrectable) bit means that error type has been observed. Correctable errors are recovered in hardware, so nothing fails — but a high rate means the link spends time on retries instead of payload, and it is the leading indicator of the link that drops to a lower speed next week. `dmesg | grep -i aer` shows the kernel's view. Meta open-sourced **PCIcrawler** to automate exactly this at fleet scale.

**Blind spots:** no NVLink, no friendly NUMA affinity, and GPUs appear as opaque vendor strings rather than `cuda0`. `lspci` tells you the *pipe width*, not the *GPU-to-GPU fabric*.

### 5. `nvidia-smi topo -m` — the accelerator-side matrix

```
$ nvidia-smi topo -m
        GPU0  GPU1  GPU2  GPU3  GPU4  GPU5  GPU6  GPU7  NIC0  NIC1  NIC4  CPU Affinity  NUMA Affinity
GPU0     X    NV18  NV18  NV18  NV18  NV18  NV18  NV18  PIX   PXB   SYS   0-31,64-95    0
GPU1    NV18   X    NV18  NV18  NV18  NV18  NV18  NV18  PXB   PIX   SYS   0-31,64-95    0
GPU4    NV18  NV18  NV18  NV18   X    NV18  NV18  NV18  SYS   SYS   PIX   32-63,96-127  1
NIC0    PIX   PXB   PXB   PXB   SYS   SYS   SYS   SYS    X    PIX   SYS
NIC4    SYS   SYS   SYS   SYS   PIX   PXB   PXB   PXB   SYS   SYS    X

Legend:

  X    = Self
  SYS  = Connection traversing PCIe as well as the SMP interconnect between NUMA nodes (e.g., QPI/UPI)
  NODE = Connection traversing PCIe as well as the interconnect between PCIe Host Bridges within a NUMA node
  PHB  = Connection traversing PCIe as well as a PCIe Host Bridge (typically the CPU)
  PXB  = Connection traversing multiple PCIe bridges (without traversing the PCIe Host Bridge)
  PIX  = Connection traversing at most a single PCIe bridge
  NV#  = Connection traversing a bonded set of # NVLinks
```

**The legend, best to worst, with what each means for the two things you actually care about — GPUDirect RDMA (GPU↔NIC) and NCCL (GPU↔GPU):**

| Code | Path | Verdict |
|---|---|---|
| `X` | self | — |
| `NV#` | a bonded set of `#` NVLinks | **best GPU↔GPU.** `NV18` on an H100 baseboard means all 18 NVLink4 links, ~900 GB/s bidirectional per GPU. |
| `PIX` | at most a **single** PCIe bridge | **best PCIe.** The ideal GPU↔NIC pairing for GPUDirect RDMA — the transaction is routed by one switch and never reaches the CPU. |
| `PXB` | **multiple** PCIe bridges, without traversing the host bridge | good. Acceptable GDR fallback. |
| `PHB` | traverses a **PCIe Host Bridge** (typically the CPU) | ok-ish. The root complex becomes a shared contention point, and peer-to-peer across it is platform-dependent. |
| `NODE` | same NUMA node, but **between** PCIe host bridges | poor. Two root complexes on one socket — happens on sub-NUMA-clustered or multi-IOD CPUs. |
| `SYS` | across NUMA nodes, over the **SMP interconnect** (UPI/xGMI) | **worst.** GDR traffic goes over the inter-socket link. |

**The rule that gets tested: `PIX`/`NV#` = local and fast; `NODE`/`SYS` = you crossed something you did not want to cross.** Each GPU should be paired to a NIC whose cell reads `PIX`, or at worst `PXB`. A GPU–NIC pair marked `SYS` will do GPUDirect RDMA over the socket interconnect, roughly halving effective RDMA bandwidth and adding latency — and NCCL notices: it will open fewer channels to a NIC it considers distant, or fall back to staging through host memory entirely.

**On the NVLink question the checkpoint asks:** GPU↔GPU traffic on an H100-generation HGX baseboard does **not** use PCIe at all — it goes over NVLink4, with each GPU's 18 links fanned across four third-generation NVSwitch chips, which is why every GPU-to-GPU cell reads `NV18`. No PCIe switch is required for peer traffic because peer traffic is not on PCIe. What *is* a board-design choice is whether a PCIe switch sits between a GPU and its paired NIC on the host side — and that choice determines whether `topo -m` reports `PIX` or `PHB` for the pair. **Read it off your own box; do not assume.**

Two companion commands belong in the same capture, because `topo -m` reports the NVLink connection's *class* but not its health:

```
$ nvidia-smi nvlink -s -i 0          # per-link state and rate
GPU 0: NVIDIA H100 80GB HBM3
         Link 0: 26.562 GB/s
         Link 1: 26.562 GB/s
         ...
         Link 17: 26.562 GB/s

$ nvidia-smi nvlink -e -i 0          # per-link error counters
GPU 0: NVIDIA H100 80GB HBM3
         Link 0: Replay Errors: 0
         Link 0: Recovery Errors: 0
         Link 0: CRC Errors: 0
```

Eighteen links at 26.562 GB/s each is 478 GB/s per direction, 956 GB/s bidirectional — the ~900 GB/s figure NVIDIA quotes, before protocol overhead. **A link listed as `inactive`, or a link count below 18, is an NVLink degradation that `topo -m` will still summarise as `NV#` with a smaller number** — so read the count, not just the code. Non-zero replay or CRC errors mean the link is retrying, which is the NVLink analogue of PCIe correctable errors.

**Blind spots:** it flattens PCIe into a single letter, so it will never tell you a `PIX` link trained at ×4 — `PIX` says "one bridge away," not "×16 and healthy." **It lists no NVMe devices at all.** And, as §7 shows, it can disagree with other NVIDIA-adjacent tooling about the same edge.

### 6. The reconciliation procedure

Reconcile in this order. Each step uses the tool that *owns* that fact, and each step's output feeds the next.

```
   THE RECONCILIATION PIPELINE — six steps, in this order, no shortcuts

  ┌─ STEP 1 ─ FRAME ────────────────────────────────────────────────────────┐
  │ numactl --hardware  +  lscpu -e                                         │
  │ → N NUMA nodes, CPU→node ranges, SMT sibling map, RAM/node, SLIT index  │
  │ OUTPUT: an empty two-column (or N-column) skeleton to hang devices on   │
  └────────────────────────────────┬────────────────────────────────────────┘
                                   ▼
  ┌─ STEP 2 ─ IDENTITY ─────────────────────────────────────────────────────┐
  │ lstopo --whole-io  +  nvidia-smi -q | grep 'Bus Id'  +  readlink on     │
  │ /sys/block/*/device/device  and /sys/class/infiniband/*/device          │
  │ → ONE TABLE: BDF ↔ cudaN ↔ GPU index ↔ UUID ↔ mlx5_N ↔ nvmeXn1         │
  │ OUTPUT: the Rosetta stone. NOTHING downstream is valid without it.      │
  └────────────────────────────────┬────────────────────────────────────────┘
                                   ▼
  ┌─ STEP 3 ─ PLACE ────────────────────────────────────────────────────────┐
  │ lstopo tree  → home NUMA node of every GPU, NIC and NVMe                │
  │ CROSS-CHECK against nvidia-smi topo -m's NUMA Affinity column           │
  │ and against /sys/bus/pci/devices/<bdf>/numa_node                        │
  │ → they must agree. If they don't: trust lstopo + sysfs, and FLAG IT.    │
  │ OUTPUT: every device placed in a column                                  │
  └────────────────────────────────┬────────────────────────────────────────┘
                                   ▼
  ┌─ STEP 4 ─ CLASSIFY THE EDGES ───────────────────────────────────────────┐
  │ nvidia-smi topo -m   → GPU↔GPU (NV#) and GPU↔NIC (PIX/PXB/PHB/SYS)     │
  │ nvidia-smi nvlink -s/-e → link count, rate, replay/CRC errors           │
  │ NVMe edges: derive from STEP 3 (topo -m has no NVMe rows)               │
  │ OUTPUT: every edge labelled with its class                               │
  └────────────────────────────────┬────────────────────────────────────────┘
                                   ▼
  ┌─ STEP 5 ─ VERIFY THE PIPES ─────────────────────────────────────────────┐
  │ lspci -vvv per device → LnkCap vs LnkSta; grep for '(downgraded)'       │
  │                       → AER CESta/UESta trailing '+'                     │
  │ → an edge needs BOTH a good class (step 4) AND a good width (step 5)    │
  │ OUTPUT: every edge annotated with trained speed × width + error state    │
  └────────────────────────────────┬────────────────────────────────────────┘
                                   ▼
  ┌─ STEP 6 ─ RULE OUT A FAULT, AND CHECK DELIVERY ─────────────────────────┐
  │ dmesg -T | grep -iE 'xid|sxid'   → any entry = HARDWARE TICKET, stop    │
  │ nvidia-smi -q -d POWER,TEMPERATURE,CLOCK,PERFORMANCE under load         │
  │ → clocks.sm / clocks.max.sm ratio + which event bit is set              │
  │ OUTPUT: is the node healthy, and is it delivering its rated clock?      │
  └─────────────────────────────────────────────────────────────────────────┘
                                   ▼
                   ONE DIAGRAM + ONE FAILURE-MODE PARAGRAPH
```

The order is not arbitrary. Step 2 before step 3 because you cannot place a device you cannot name. Step 4 before step 5 because the class tells you which edges are worth verifying in detail. Step 6 last because a hardware fault invalidates every other conclusion — if a GPU has fallen off the bus, its `topo -m` row is meaningless.

### 7. The three reconciliation traps

**Trap 1 — the tools disagree on identity.** `lspci` says `0000:9d:00.0`, `lstopo` says `cuda4`, `nvidia-smi topo -m` says `GPU4`, a CUDA program with `CUDA_VISIBLE_DEVICES` set may call it device 0, and the kubelet knows it only as `GPU-a1b2c3d4-...`. Build the mapping explicitly, once:

```
$ nvidia-smi --query-gpu=index,uuid,pci.bus_id,serial --format=csv
index, uuid, pci.bus_id, serial
0, GPU-c3d4e5f6-..., 00000000:1B:00.0, 1652xxxxxxxxx
4, GPU-a1b2c3d4-..., 00000000:9D:00.0, 1652yyyyyyyyy
```

Note that `nvidia-smi`'s `pci.bus_id` format is `00000000:9D:00.0` — an 8-digit domain and uppercase hex — while `lspci` prints `0000:9d:00.0`. **Normalise case and domain width before comparing, or your join silently produces zero matches.** And `nvidia-smi`'s `index` is *not* stable across reboots or driver reloads unless you have set `CUDA_DEVICE_ORDER=PCI_BUS_ID`; the UUID and the BDF are the stable identifiers.

**Trap 2 — `PIX` and `LnkSta` are orthogonal facts.** `PIX` counts *bridges traversed*. `LnkSta` reports *trained speed and width*. A link can be `PIX` and ×4-degraded, or `SYS` and perfectly ×16. **You need both codes for the same edge before you trust it.** This is the single most common way a teardown reaches a confidently wrong conclusion.

**Trap 3 — even NVIDIA's own tools can disagree about the same edge.** A production report on an 8×V100 system with Mellanox RDMA NICs found NCCL's own topology detection (`NCCL_DEBUG=INFO`, which prints the link class NCCL believes for each GPU–NIC pair) reporting `PIX`/`PXB` for a pair where `nvidia-smi topo -m` reported `PHB` — two official, NVIDIA-adjacent tools disagreeing about the same silicon (NVIDIA/nccl issue #246). This is direct evidence that "no single tool is authoritative" is not overcautious pedagogy. **The resolution is the discipline this lesson teaches: when two tools disagree, reconcile against a third source — the PCIe tree itself, via `lspci -tv` and the `lstopo` HostBridge structure — rather than trusting whichever tool you prefer by habit.**

Worth capturing NCCL's view deliberately, since it is the consumer that actually matters for distributed training:

```
$ NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=INIT,GRAPH ./all_reduce_perf -b 8 -e 128M -f 2 -g 8 2>&1 \
    | grep -E 'NET/IB|via NET|Channel|Trees|Rings'
node:1234:1234 [0] NCCL INFO NET/IB : Using [0]mlx5_0:1/IB [1]mlx5_1:1/IB ... 
node:1234:1234 [0] NCCL INFO Channel 00/24 :    0   1   2   3   4   5   6   7
node:1234:1234 [0] NCCL INFO Setting affinity for GPU 0 to 0-31,64-95
```

Two things to read: the **NIC list NCCL chose** (does it match your `PIX` pairings?) and the **channel count** (fewer channels to a NIC NCCL considers distant is a direct, observable consequence of a `SYS` pairing). `Setting affinity for GPU 0 to 0-31,64-95` is NCCL independently agreeing with `topo -m`'s CPU Affinity column — a free fourth opinion.

### 8. The diagnostic decision tree

The interview question is "GPU at 100% utilization, throughput ~half spec, no obvious error." Here is the full tree this module has been building, with one command per hypothesis. Work it top-down; each step either eliminates a cause or ends the investigation.

```
   "GPU AT 100% UTIL, THROUGHPUT ~HALF SPEC, NO ERROR"
   Seven hypotheses. One command each. Eliminate top-down; never guess.

   START
     │
     │  utilization.gpu only means "a kernel was resident in the sample
     │  window." It says nothing about clock, bandwidth, or health.
     ▼
  ┌─ H0 ─ IS THE HARDWARE EVEN HEALTHY? ────────────────────────────────────┐
  │ $ dmesg -T | grep -iE 'xid|sxid'                                        │
  │ ANY entry (esp. Xid 79 "fallen off the bus", Xid 48/63/64 ECC,           │
  │ Xid 13/31 app faults) → STOP. Cordon the node, file a repair ticket.    │
  │ Everything below assumes healthy silicon.                               │
  └──────────────────────┬──────────────────────────────────────────────────┘
                    clean │
                          ▼
  ┌─ H1 ─ IS IT DELIVERING ITS RATED CLOCK? ────────────────────────────────┐
  │ $ nvidia-smi --query-gpu=clocks.sm,clocks.max.sm,\                      │
  │     clocks_event_reasons.active,power.draw,enforced.power.limit,\        │
  │     temperature.gpu,temperature.memory --format=csv                     │
  │ ratio < 1 → the deficit is EXACTLY the ratio. Then decode the bit:      │
  │   0x04 SW Power Cap      → a DECISION. Was it yours?                    │
  │   0x20 SW Thermal        → a SYMPTOM. Facilities ticket.                │
  │   0x40 HW Thermal        → emergency. Page.                             │
  │   0x80 HW Power Brake    → PSU asserted the brake pin.                  │
  │   0x10 Sync Boost        → a PEER GPU is holding this one down.         │
  │   0x08 HW Slowdown alone → decompose with 0x40 / 0x80 first.            │
  └──────────────────────┬──────────────────────────────────────────────────┘
                    at boost │
                          ▼
  ┌─ H2 ─ DID THE PCIe LINK TRAIN CORRECTLY? ───────────────────────────────┐
  │ $ sudo lspci -vvv -s <bdf> | grep -E 'LnkCap:|LnkSta:'                  │
  │ '(downgraded)' on speed or width → the cap is the ratio of the products. │
  │ Gen3 ×8 on a Gen5 ×16 slot = 8.0× less host bandwidth.                  │
  │ Also: AER CESta with trailing '+' → the link is retrying under load.    │
  │ Cause: reseat, slot, BIOS bifurcation setting, cable, riser.            │
  └──────────────────────┬──────────────────────────────────────────────────┘
                     ×16 ok │
                          ▼
  ┌─ H3 ─ IS THE GPU↔NIC PATH LOCAL? ──────────────────────────────────────┐
  │ $ nvidia-smi topo -m     (+ NCCL_DEBUG=INFO for a second opinion)       │
  │ The NIC the job is actually using — is its cell PIX/PXB, or NODE/SYS?   │
  │ SYS → GDR over the socket interconnect ≈ half RDMA bandwidth + latency. │
  │ Fix: pin the job's HCA to its PIX NIC (NCCL_IB_HCA), or let Topology    │
  │ Manager single-numa-node place it.                                       │
  └──────────────────────┬──────────────────────────────────────────────────┘
                    PIX/PXB │
                          ▼
  ┌─ H4 ─ IS MEMORY LOCAL TO THE GPU? ─────────────────────────────────────┐
  │ $ numastat -p <pid>          → which node the pages actually live on    │
  │ $ cat /proc/<pid>/status | grep -i cpus_allowed_list                    │
  │ $ kubectl exec <pod> -- cat /sys/fs/cgroup/cpuset.{cpus,mems}.effective │
  │ Pages on the far node → every H2D copy crosses UPI. Fix: numactl        │
  │ --cpunodebind/--membind, or the kubelet policies from lesson 05.        │
  │ Watch for the TopologyInfo trap: policy satisfied, GPU still remote.    │
  └──────────────────────┬──────────────────────────────────────────────────┘
                    local │
                          ▼
  ┌─ H5 ─ IS THE DATA PATH LOCAL AND DEEP ENOUGH? ─────────────────────────┐
  │ $ readlink -f /sys/block/<dev>/device/device ; cat .../numa_node        │
  │ $ fio --direct=1 --ioengine=io_uring at iodepth 1 vs 32×4              │
  │ $ gdscheck -p    (use_compat_mode? filesystem supported? IOMMU?)       │
  │ Cross-socket drive → UPI hop per read. Shallow queue → device idle      │
  │ regardless of placement. CHECK QUEUE DEPTH FIRST — it's cheaper.        │
  └──────────────────────┬──────────────────────────────────────────────────┘
                     fine │
                          ▼
  ┌─ H6 ─ IS THE NVLINK MESH INTACT? ──────────────────────────────────────┐
  │ $ nvidia-smi nvlink -s -i <n>   → 18 links, all at 26.562 GB/s?        │
  │ $ nvidia-smi nvlink -e -i <n>   → replay / recovery / CRC errors       │
  │ Fewer active links → peer-to-peer capped; topo -m still says NV<k>.    │
  └──────────────────────┬──────────────────────────────────────────────────┘
                     fine │
                          ▼
     Not host topology. Now it is the application: kernel efficiency,
     collective algorithm choice, batch size, or a genuine algorithmic limit.
     THAT is when you reach for a framework profiler — not before.

   ── THE POINT ──────────────────────────────────────────────────────────
   utilization.gpu reads 100% through EVERY branch above. That is exactly
   why you reconcile the topology instead of trusting the dashboard.
```

### 9. From diagram to capacity statement — the node scorecard

A diagram that stops at "here is the layout" is half a deliverable. The other half is what the layout *implies*, in numbers. Four calculations turn the picture into a capacity statement, and every one of them is re-runnable with your own inputs.

**(a) Can the storage path keep the GPUs fed?**

```
  required_read_BW = N_gpus × samples_per_sec_per_gpu × bytes_per_sample

  Worked, video/multimodal training on this node:
    8 GPUs × 400 clips/s/GPU × 3.2 MB/clip        = 10.24 GB/s

  Available, from the diagram:
    Gen5 ×4 drive (confirmed by LnkSta)  15.75 GB/s theoretical, ~14 GB/s rated
    aligned    → the drive is the limit ✓  (10.24 / 14 = 73% of one drive)
    misaligned → your share of one UPI link: ~48 GB/s/dir, shared with ALL
                 coherence traffic; at a 50% share ≈ 24 GB/s for everything
                 crossing, with your 10.24 GB/s now competing for it. ✗

  Placement is load-bearing for THIS workload — and would not be for LLM
  pretraining on 2-byte tokens (8 × 40,000 tok/s × 2 B = 640 KB/s).
```

**(b) How long does a checkpoint take, and what does it cost in wall clock?**

```
  Standard mixed-precision Adam accounting (ZeRO, K=12):
    training state = 2Ψ (bf16 weights) + 2Ψ (bf16 grads) + 12Ψ (fp32 master
                     weights + momentum + variance) = 16 bytes/param
    resumable checkpoint (weights + optimizer, no grads) = 14 bytes/param

  70B-parameter model:
    weights-only bf16       70e9 × 2  = 140 GB
    full resumable          70e9 × 14 = 980 GB

  Write time, and cost as a % of wall clock checkpointing every 30 min (1800 s):
    one Gen5 ×4 NVMe @ 6.5 GB/s      980 / 6.5 = 151 s      →  8.4%
    8-drive RAID-0, host-limited 40  980 / 40  =  24.5 s    →  1.4%
    one 200 GbE NIC @ ~22 GB/s       980 / 22  =  44.5 s    →  2.5%
    one 25 GbE link @ ~2.8 GB/s      980 / 2.8 = 350 s      → 19.4%  ← a fifth
                                                                      of the cluster

  Placement multiplier: a cross-socket write path is capped by your share of one
  UPI link, not the array's rating — which can more than double the array case.
```

**(c) What does the node draw, and what does that mean per rack?**

```
  Anchor: NVIDIA publishes 10.2 kW max for DGX H100, from six 3,300 W
  Titanium PSUs in 4+2 redundancy (200-240 V, 16 A each).

  8 × H100 SXM5 @ 700 W (SKU TDP, hard)          = 5,600 W   (55%)
  2 × Xeon Platinum 8480C @ 350 W (SKU TDP)      =   700 W
  32 × DDR5 RDIMM @ ~7 W  (2 TB, estimate)       =  ~224 W
  8 × U.2 + 2 × M.2 NVMe  (estimate)             =  ~176 W
  8 × ConnectX-7 + 400G optics @ ~27 W (estimate)=  ~216 W
  2-4 × additional NIC/DPU @ ~30 W (estimate)    =   ~90 W
  ───────────────────────────────────────────────────────────
  itemised subtotal                              ≈ 7,006 W
  residual to the 10.2 kW datasheet figure       ≈ 3,194 W   (31%)
    = 4 × NVSwitch, PCIe switches/retimers, baseboard VRM losses,
      fans, PSU conversion loss (~4% at Titanium efficiency)

  THE LESSON: the GPUs are 55%, not 90%. Budgeting 8 × 700 W = 5.6 kW and
  being surprised when the rack trips is the classic capacity-planning error.

  Rack, from the DGX SuperPOD H100 reference (415 V 3-phase Wye, 32 A):
    S = √3 × 415 V × 32 A = 23.0 kVA;  P = 23.0 × PF 0.95 ≈ 21.8 kW/circuit
    3 circuits, N+1, no circuit above 50% of load:
      4 nodes × 10.2 kW = 40.8 kW IT load
      N-1: 2 × 21.8 = 43.6 kW ≥ 40.8 kW  ✓ (94% loaded)
      a 5th node: 51.0 kW → N-1 needs 25.5 kW/circuit = 117%  ✗ TRIPS
```

**(d) What ΔT does that heat load imply?**

```
  Q = V̇ · ρ · c_p · ΔT       ρ·c_p(air) = 1,206 J/(m³·K)
                              ρ·c_p(water) = 4,180,000 J/(m³·K)   → 3,470× ratio

  Air, one 10.2 kW node at ΔT = 15 K:
    V̇ = 10,200 / (1,206 × 15) = 0.564 m³/s = 1,195 CFM per node
  Air, the 40.8 kW rack:                    = 4,780 CFM per cabinet

  If containment delivers only 3,500 CFM, the achievable ΔT rises:
    ΔT = 40,800 / (1,206 × 1.652) = 20.5 K
    → inlet 27 °C becomes exhaust 47.5 °C, recirculated inlet climbs, and
      top-of-rack GPUs report 0x20 SW Thermal Slowdown FIRST.
      Nothing in GPU telemetry says "airflow." It says "margin is shrinking."

  Liquid, for contrast, a 120 kW GB200 NVL72 rack at ~130 L/min:
    ΔT = Q × 60 / (c_p × flow) = 120 × 60 / (4.18 × 130) = 13.2 K
  The same 120 kW with air at ΔT 15 K would need 14,050 CFM into one
  cabinet — which is why that generation is liquid-cooled by mandate.
```

Put all four on the deliverable's front page. **A reviewer who sees the diagram plus these four numbers knows immediately whether the node is provisioned coherently**, which is a materially stronger artifact than a diagram alone.

### 10. The exporter: turning the teardown into a monitored signal

The teardown is a one-time act. The production version runs continuously. Design the exporter as part of the deliverable, because "create alerts" is a literal line in the job description this module targets.

What to emit, at node start and then periodically:

```
# Static topology facts — emitted once at boot, as gauges with rich labels.
# These change only when hardware changes, which is exactly why a change is an alert.
node_gpu_nic_link_class{gpu="4",gpu_bdf="0000:9d:00.0",nic="mlx5_4",class="PIX"} 1
node_gpu_numa_node{gpu="4",gpu_bdf="0000:9d:00.0"} 1
node_nvme_numa_node{device="nvme0n1",bdf="0000:c1:00.0"} 0
node_gpu_device_plugin_numa_reported{gpu="4"} 1      # 0 = the lesson-05 TopologyInfo trap

# PCIe link health — the one thing no other exporter covers
node_pcie_link_speed_gts{bdf="0000:9d:00.0",kind="cap"} 32
node_pcie_link_speed_gts{bdf="0000:9d:00.0",kind="sta"} 32
node_pcie_link_width{bdf="0000:9d:00.0",kind="cap"} 16
node_pcie_link_width{bdf="0000:9d:00.0",kind="sta"} 16
node_pcie_link_downgraded{bdf="0000:9d:00.0"} 0
node_pcie_aer_correctable_total{bdf="0000:9d:00.0"} 0

# NVLink health
node_nvlink_active_links{gpu="4"} 18
node_nvlink_crc_errors_total{gpu="4",link="7"} 0
```

The alert rules that matter, in PromQL that would actually evaluate:

```promql
# 1. Any GPU↔NIC pair a job might use that is not local. Fires at rollout,
#    not after a training job's step-time histogram goes bimodal.
count by (instance) (node_gpu_nic_link_class{class=~"NODE|SYS"}) > 0

# 2. Any PCIe link trained below its capability. The 8× silent bandwidth cap.
max by (instance, bdf) (node_pcie_link_downgraded) > 0

# 3. The lesson-05 TopologyInfo trap, fleet-wide: a node claiming
#    single-numa-node whose device plugin published no NUMA affinity.
min by (instance) (node_gpu_device_plugin_numa_reported) == 0

# 4. Thermal margin, not absolute temperature — SKU-independent, so one rule
#    works across a mixed H100/B200 fleet. (DCGM field 153.)
min by (instance, gpu) (DCGM_FI_DEV_GPU_TEMP_MARGIN_CELSIUS) < 5

# 5. The UNCHOSEN constraints only (0x20 SW Thermal | 0x40 HW Thermal |
#    0x80 HW Power Brake). Deliberately excludes 0x04 SW Power Cap, or you
#    page on every correctly-capped GPU in the building. Do the bitmask test
#    in the exporter and emit a clean boolean — bitwise ops are awkward in
#    PromQL, and pushing the logic into the collector is the right call.
max by (instance, gpu) (node_gpu_unchosen_throttle) > 0

# 6. The FinOps metric: delivered clock as a fraction of rated, per rack.
#    A rack at 0.80 is delivering 80% of the FLOPs you are paying for.
avg by (rack) (DCGM_FI_DEV_SM_CLOCK / on(instance,gpu) group_left ()
               DCGM_FI_DEV_MAX_SM_CLOCK)
```

**Where your exporter sits relative to production systems** — name this out loud in an interview so you neither undersell nor oversell it:

| System | What it detects | Detection · alerting · remediation |
|---|---|---|
| Meta **PCIcrawler** (open source) | PCI/PCIe topology + AER error counters, fleet-wide | Detection |
| Meta **FBAR** | Broad failure taxonomy incl. thermal runaway; ~50× fewer training interruptions | Detection + automated remediation |
| Modal-style **DCGM + `dmesg` sweep** | Xid/SXid faults, thermal violations, sync-boost violations, >88 °C | Detection + alerting |
| Crusoe **AutoClusters** | Xid-79-class hardware failures | Detection + automated cordon/drain/replace |
| CoreWeave **SUNK** health probes | Failing nodes, evicted before they impact a running job | Detection + scheduling-level remediation |
| **Your** `node_gpu_nic_link_class` / `node_pcie_link_downgraded` exporter | GPU↔NIC link class, PCIe link degradation, device-plugin NUMA reporting | Detection (feeds alerting; remediation not built) |

The honest framing: your exporter covers a **slice none of the others explicitly cover** — static topology and PCIe link-training correctness as a monitored, alertable fact rather than a bring-up checklist item — while automated remediation is the next layer up and you have not built it.

## Perspectives

**Developer.** From the researcher's seat a topology defect and a hardware fault look identical: "the job is slow" or "the job crashed." The whole point of this capstone is that the *platform engineer* must tell them apart and route to the right fix — rebalance placement, adjust a power cap, or file a hardware ticket. The developer never sees the four-tool reconciliation and cannot self-serve past it.

**Operator / SRE.** This is the interview task rehearsed for real. But the *production* version runs the reconciliation continuously and automatically at fleet scale. The manual single-node version you are building is the foundation those systems rest on — and it is what lets you debug an automated detector's output when it is wrong.

**Cross-tool reconciliation.** NVIDIA/nccl #246 is real proof that even NVIDIA's own tools disagree on production hardware. That should *raise* confidence in this lesson's central claim: "no single tool is authoritative" is accuracy, and reconciling against a third source is a production-tested discipline.

**Economics / portfolio.** CoreWeave, Meta, Crusoe and Modal all publish exactly this kind of teardown as public engineering writing, so the deliverable genuinely resembles a valued external artifact — provided it contains a *measured* number, not just a diagram.

## Real-world use cases

- **Meta Engineering — "How Meta keeps its AI hardware reliable."** Describes **FBAR** (Facebook Auto Remediation), an explicit failure-type taxonomy (disks, CPUs, memory, switches, GPUs, ASICs, networks), thermal runaway as a named transient-error class, and a reported **~50× reduction in training-interruption rate**. **What it shows:** the clearest anchor for *why* fleet-scale reconciliation tooling exists, and the payoff for building it. `https://engineering.fb.com/2025/07/22/data-infrastructure/how-meta-keeps-its-ai-hardware-reliable/`
- **Meta Engineering — "How Facebook deals with PCIe faults."** Describes Meta's automated PCIe fault detect → diagnose → remediate → repair pipeline and their open-source **PCIcrawler** CLI for PCI/PCIe topology and AER inspection. **What it shows:** a direct production precedent for Part A — the automated version of the `lspci -tv` + `-vvv` + AER step. `https://engineering.fb.com/2021/06/02/data-center-engineering/how-facebook-deals-with-pcie-faults-to-keep-our-data-centers-running-reliably/`
- **Modal — "Keeping 20,000 GPUs healthy."** A full Xid/SXid fault taxonomy (including Xid 79, "GPU has fallen off the bus") and a `dcgmi` + `dmesg` passive health-check methodology run continuously at 20,000-GPU scale, with **>88 °C** as an operational red flag. **What it shows:** a model answer for Part B's "what exporter would you build," from a shop that runs it. `https://modal.com/blog/gpu-health`
- **Crusoe — "AutoClusters: Automated GPU Failure Remediation for AI Training Clusters."** Automatic cordon/drain/replace triggered on Xid-79-class failures, framed around minimising queue-wait time as "the largest controllable variable in cluster goodput." **What it shows:** the alert-design half of this capstone taken all the way to automated remediation, with an economic framing you can borrow. `https://www.crusoe.ai/resources/blog/autoclusters-minimizing-hardware-failures-in-large-gpu-clusters`
- **CoreWeave — "NVIDIA H100 GPU Benchmark Results."** Describes **SUNK** (Slurm on Kubernetes) topology-aware scheduling with health probes that evict failing nodes before they impact a running job, and reports **49.2% MFU on 128 H100s** (a 2025-era snapshot). **What it shows:** the reconciliation skill tied to a named production scheduler and to the single number it exists to defend. `https://www.coreweave.com/blog/nvidia-h100-gpu-benchmark-results-what-we-learned-from-large-scale-gpu-testing`
- **NVIDIA/nccl issue #246.** NCCL's own topology detection reporting `PIX`/`PXB` for a GPU–NIC pair where `nvidia-smi topo -m` reported `PHB`, on an 8×V100 system with Mellanox RDMA NICs. **What it shows:** direct production evidence for "no single tool is authoritative," and the reason §6 mandates a cross-check step. `https://github.com/NVIDIA/nccl/issues/246`

## Worked example

**A complete teardown of one node, start to finish.** Run this top to bottom on a rented box and you have the deliverable. Transcripts are representative of an 8-GPU, 2-socket HGX-class node; your values will differ, but every command and field name is real.

### Phase 0 — capture everything, once

Save this as `collect.sh` and run it as root. Capturing first and analysing second matters: you can re-analyse without re-renting, and the raw files are the evidence the deliverable asks for.

```bash
#!/usr/bin/env bash
# Topology teardown collector — module 02b capstone.
# Run as root. Produces ./raw/ with everything needed for offline analysis.
set -uo pipefail
mkdir -p raw && cd raw

# ── Frame: NUMA and CPU layout ───────────────────────────────────────────
numactl --hardware                                   > numactl.txt   2>&1
lscpu                                                > lscpu.txt     2>&1
lscpu -e=CPU,CORE,SOCKET,NODE,L3                     > lscpu-e.txt   2>&1
cat /sys/devices/system/node/node*/distance          > slit.txt      2>&1

# ── Locality tree: the authoritative device→socket map ───────────────────
lstopo-no-graphics --whole-io --of txt                > lstopo.txt    2>&1
lstopo-no-graphics --whole-io --of console            > lstopo-console.txt 2>&1
lstopo --whole-io --of svg                            > lstopo.svg    2>&1

# ── PCIe: tree, then per-device link state and AER ───────────────────────
lspci -tvD                                           > lspci-tree.txt 2>&1
lspci -Dnn                                           > lspci-ids.txt  2>&1
for bdf in $(lspci -Dn | awk '{print $1}'); do
  echo "===== $bdf  $(lspci -Ds "$bdf" | cut -d' ' -f2-)"
  lspci -vvv -s "$bdf" 2>/dev/null | grep -E \
    'LnkCap:|LnkSta:|LnkCap2:|LnkSta2:|DevSta:|CESta:|UESta:|NUMA node'
done                                                 > lspci-links.txt 2>&1

# ── sysfs ground truth for device→NUMA (no tool interpretation) ──────────
for d in /sys/bus/pci/devices/*/; do
  printf '%s vendor=%s class=%s numa_node=%s\n' \
    "$(basename "$d")" "$(cat "$d/vendor")" "$(cat "$d/class")" \
    "$(cat "$d/numa_node" 2>/dev/null)"
done                                                 > sysfs-numa.txt  2>&1

# ── GPU-side view ────────────────────────────────────────────────────────
nvidia-smi topo -m                                   > topo-m.txt      2>&1
nvidia-smi topo -mp                                  > topo-mp.txt     2>&1   # PCIe-only view
nvidia-smi --query-gpu=index,uuid,pci.bus_id,serial,name \
  --format=csv                                       > gpu-identity.csv 2>&1
nvidia-smi -q                                        > nvidia-smi-q.txt 2>&1
for i in $(nvidia-smi --query-gpu=index --format=csv,noheader); do
  echo "===== GPU $i"; nvidia-smi nvlink -s -i "$i"; nvidia-smi nvlink -e -i "$i"
done                                                 > nvlink.txt      2>&1

# ── Storage ──────────────────────────────────────────────────────────────
nvme list                                            > nvme-list.txt   2>&1
for n in /sys/block/nvme*n1; do
  [ -e "$n" ] || continue
  dev=$(basename "$n"); bdf=$(basename "$(readlink -f "$n/device/device")")
  printf '%s bdf=%s numa_node=%s hwq=%s sched=%s max_hw_sectors_kb=%s\n' \
    "$dev" "$bdf" "$(cat "$n/device/device/numa_node")" \
    "$(ls "$n/mq" | wc -l)" "$(cat "$n/queue/scheduler")" \
    "$(cat "$n/queue/max_hw_sectors_kb")"
  nvme smart-log "/dev/${dev%n1}"
done                                                 > nvme-detail.txt 2>&1

# ── Network device → NUMA ────────────────────────────────────────────────
for ib in /sys/class/infiniband/*; do
  [ -e "$ib" ] || continue
  printf '%s bdf=%s numa_node=%s\n' "$(basename "$ib")" \
    "$(basename "$(readlink -f "$ib/device")")" \
    "$(cat "$ib/device/numa_node" 2>/dev/null)"
done                                                 > ib-numa.txt     2>&1

# ── Fault layer and delivery state ───────────────────────────────────────
dmesg -T | grep -iE 'xid|sxid|aer|nvrm'              > kernel-gpu.txt  2>&1
nvidia-smi -q -d POWER,TEMPERATURE,CLOCK,PERFORMANCE > power-thermal.txt 2>&1

# ── Kubernetes-side, if this is a cluster node ───────────────────────────
if [ -f /var/lib/kubelet/config.yaml ]; then
  grep -E 'cpuManagerPolicy|memoryManagerPolicy|topologyManager|reserved' \
    /var/lib/kubelet/config.yaml                     > kubelet-policies.txt 2>&1
  cp /var/lib/kubelet/cpu_manager_state    cpu_manager_state.json    2>/dev/null
  cp /var/lib/kubelet/memory_manager_state memory_manager_state.json 2>/dev/null
fi

echo "Collected $(ls | wc -l) files into $(pwd)"
```

### Phase 1 — the frame

```
$ cat raw/numactl.txt
available: 2 nodes (0-1)
node 0 cpus: 0-31 64-95
node 0 size: 515655 MB
node 1 cpus: 32-63 96-127
node 1 size: 516086 MB
node distances:
node   0    1
  0:  10   21
  1:  21   10

$ head -4 raw/lscpu-e.txt
CPU CORE SOCKET NODE L3
0   0    0      0    0
1   1    0      0    0
64  0    0      0    0     ← CPU 64 shares CORE 0 with CPU 0: SMT siblings
```

**Frame established:** two sockets, two NUMA nodes, 128 logical CPUs on 64 physical cores, ~504 GiB per node, remote-access index 2.1×. CPUs 0–31 and 64–95 are the two SMT threads of socket 0's 32 cores.

### Phase 2 — identity (do not skip this)

```
$ cat raw/gpu-identity.csv
index, uuid, pci.bus_id, serial, name
0, GPU-1a2b..., 00000000:1B:00.0, 1652..., NVIDIA H100 80GB HBM3
1, GPU-2b3c..., 00000000:43:00.0, 1652..., NVIDIA H100 80GB HBM3
2, GPU-3c4d..., 00000000:52:00.0, 1652..., NVIDIA H100 80GB HBM3
3, GPU-4d5e..., 00000000:61:00.0, 1652..., NVIDIA H100 80GB HBM3
4, GPU-5e6f..., 00000000:9D:00.0, 1652..., NVIDIA H100 80GB HBM3
5, GPU-6f70..., 00000000:C3:00.0, 1652..., NVIDIA H100 80GB HBM3
6, GPU-7081..., 00000000:D1:00.0, 1652..., NVIDIA H100 80GB HBM3
7, GPU-8192..., 00000000:DF:00.0, 1652..., NVIDIA H100 80GB HBM3

$ grep -E 'CoProc|OpenFabrics|Block' raw/lstopo.txt | head
          CoProc "cuda0"            # under Package L#0, PCI 0000:1b:00.0
          OpenFabrics "mlx5_0"      #                     PCI 0000:1c:00.0
          Block(Disk) "nvme0n1"     #                     PCI 0000:c1:00.0
          CoProc "cuda4"            # under Package L#1, PCI 0000:9d:00.0
          OpenFabrics "mlx5_4"      #                     PCI 0000:9e:00.0
```

Build the one table everything else references. Note the case and domain-width normalisation — `nvidia-smi`'s `00000000:9D:00.0` and `lspci`'s `0000:9d:00.0` are the same device:

| GPU idx | BDF (normalised) | hwloc | NUMA (sysfs) | paired NIC | NIC BDF | local NVMe |
|---|---|---|---|---|---|---|
| 0 | `0000:1b:00.0` | `cuda0` | 0 | `mlx5_0` | `0000:1c:00.0` | `nvme0n1` |
| 1 | `0000:43:00.0` | `cuda1` | 0 | `mlx5_1` | `0000:44:00.0` | `nvme0n1` |
| 2 | `0000:52:00.0` | `cuda2` | 0 | `mlx5_2` | `0000:53:00.0` | `nvme1n1` |
| 3 | `0000:61:00.0` | `cuda3` | 0 | `mlx5_3` | `0000:62:00.0` | `nvme1n1` |
| 4 | `0000:9d:00.0` | `cuda4` | 1 | `mlx5_4` | `0000:9e:00.0` | `nvme2n1` |
| 5 | `0000:c3:00.0` | `cuda5` | 1 | `mlx5_5` | `0000:c4:00.0` | `nvme2n1` |
| 6 | `0000:d1:00.0` | `cuda6` | 1 | `mlx5_6` | `0000:d2:00.0` | `nvme3n1` |
| 7 | `0000:df:00.0` | `cuda7` | 1 | `mlx5_7` | `0000:e0:00.0` | `nvme3n1` |

**This answers the checkpoint's depth probe directly:** GPUs 0–3 share socket 0; GPU5 lives on socket 1 and its NIC is `mlx5_5`, attached to socket 1's root complex, which is why its `topo -m` cell reads `PIX`.

### Phase 3 — place, and cross-check three ways

```
$ grep -E '0x10de|0x15b3' raw/sysfs-numa.txt | head -6
0000:1b:00.0 vendor=0x10de class=0x030200 numa_node=0
0000:1c:00.0 vendor=0x15b3 class=0x020700 numa_node=0
0000:9d:00.0 vendor=0x10de class=0x030200 numa_node=1
0000:9e:00.0 vendor=0x15b3 class=0x020700 numa_node=1

$ grep -E 'NUMA Affinity' -A9 raw/topo-m.txt | awk '{print $1, $(NF)}'
GPU0 0
GPU1 0
GPU2 0
GPU3 0
GPU4 1
GPU5 1
GPU6 1
GPU7 1
```

Three independent sources — `lstopo`'s Package nesting, sysfs `numa_node`, and `topo -m`'s NUMA Affinity column — agree. **Record the agreement explicitly**; it is what licenses everything downstream. Had they disagreed, the resolution order is: sysfs and `lstopo` win (they read firmware-provided ACPI affinity directly), `topo -m` is flagged, and you note the discrepancy in the deliverable as a finding rather than papering over it. A `numa_node` of `-1` here would be the lesson-05 `TopologyInfo` trap in its raw form: the *hardware* affinity is unknown to the OS, so no Kubernetes policy can align to it.

### Phase 4 — classify the edges

```
$ cat raw/topo-m.txt
        GPU0 GPU1 GPU2 GPU3 GPU4 GPU5 GPU6 GPU7 mlx5_0 mlx5_3 mlx5_4 CPU Affinity   NUMA
GPU0     X   NV18 NV18 NV18 NV18 NV18 NV18 NV18  PIX    PXB    SYS   0-31,64-95     0
GPU3    NV18 NV18 NV18  X   NV18 NV18 NV18 NV18  PXB    PIX    SYS   0-31,64-95     0
GPU4    NV18 NV18 NV18 NV18  X   NV18 NV18 NV18  SYS    SYS    PIX   32-63,96-127   1
```

**Read it:** every GPU-to-GPU cell is `NV18` — a full 18-link NVLink4 mesh through the baseboard's NVSwitch chips, so NCCL's intra-node collectives are on the fast fabric regardless of PCIe layout. GPU0's `PIX` NIC is `mlx5_0`; GPU3's is `mlx5_3`; GPU4's is `mlx5_4`. GPU0↔`mlx5_4` is `SYS` — **never pair them for GPUDirect RDMA.**

Verify the mesh is actually intact, since `topo -m` reports the class rather than the health:

```
$ grep -c '26.562 GB/s' raw/nvlink.txt
144                                        # 8 GPUs × 18 links, all up ✓
$ grep -E 'Errors: [1-9]' raw/nvlink.txt
                                           # no output: zero replay/recovery/CRC ✓
```

144 links at 26.562 GB/s. Had this returned 138, two GPUs would be running with fewer links and `topo -m` would have quietly said `NV16` — a code you must read as a *number*, not a category.

### Phase 5 — verify the pipes

```
$ grep -B1 'downgraded' raw/lspci-links.txt
===== 0000:52:00.0  3D controller: NVIDIA Corporation GH100 [H100 SXM5]
        LnkSta: Speed 32GT/s, Width x8 (downgraded)

$ grep -A2 '===== 0000:52:00.0' raw/lspci-links.txt
===== 0000:52:00.0  3D controller: NVIDIA Corporation GH100 [H100 SXM5]
        LnkCap: Port #0, Speed 32GT/s, Width x16, ASPM not supported
        LnkSta: Speed 32GT/s, Width x8 (downgraded)
```

**One finding, and it is a big one.** GPU2 (`0000:52:00.0`, per the phase-2 table) negotiated ×8 on a ×16-capable Gen5 link:

```
  capable:  32 GT/s × 16 lanes × (128/130) / 8 = 63.02 GB/s per direction
  trained:  32 GT/s ×  8 lanes × (128/130) / 8 = 31.51 GB/s per direction
  → GPU2's host↔device bandwidth is capped at exactly HALF spec.
```

Note what did *not* catch this: `topo -m` still reports GPU2's NIC as `PIX`, `lstopo` shows a correct tree, `numactl` is silent, `nvidia-smi -q` shows a healthy GPU, and `dmesg` has no entry. **Only `lspci -vvv` found it.** Likely causes in order of probability: a mis-seated SXM module or riser, a BIOS PCIe bifurcation setting, or a marginal connector. That is a bring-up ticket, not a software fix.

Check the AER counters for links that are healthy in a snapshot but retrying under load:

```
$ grep -E 'CESta.*[A-Za-z]+\+' raw/lspci-links.txt | grep -v 'AdvNonFatalErr+$'
                                           # no output: no correctable errors beyond
                                           # the benign AdvNonFatalErr flag ✓
```

### Phase 6 — rule out a fault, and check delivery

```
$ grep -icE 'xid|sxid' raw/kernel-gpu.txt
0
```

Zero Xid/SXid entries. **This is the line that licenses the conclusion "GPU2's degraded link is a seating/configuration issue, not a failing part."** Had there been an Xid 79 on GPU2, the diagnosis would flip entirely: a device that intermittently drops off the bus can renegotiate at a lower width on re-enumeration, and the fix becomes a replacement rather than a reseat.

Then, under sustained load for ≥10 minutes (lesson 07 §6 explains why 60 seconds lies):

```
$ nvidia-smi --query-gpu=index,clocks.sm,clocks.max.sm,power.draw,\
enforced.power.limit,temperature.gpu,temperature.memory,clocks_event_reasons.active \
--format=csv,noheader,nounits
0, 1755, 1980, 699.8, 700.00, 74, 81, 0x0000000000000004
1, 1755, 1980, 699.4, 700.00, 73, 80, 0x0000000000000004
2, 1770, 1980, 698.9, 700.00, 72, 79, 0x0000000000000004
3, 1740, 1980, 699.9, 700.00, 76, 83, 0x0000000000000004
4, 1710, 1980, 699.7, 700.00, 79, 86, 0x0000000000000024   ← 0x04 | 0x20
5, 1725, 1980, 699.5, 700.00, 78, 85, 0x0000000000000004
6, 1740, 1980, 699.6, 700.00, 77, 84, 0x0000000000000004
7, 1755, 1980, 699.8, 700.00, 75, 82, 0x0000000000000004
```

**Two findings here.** All eight GPUs are power-capped (`0x04`) at ~1740 MHz against a 1980 MHz boost — a mean clock ratio of 0.877, so the node is delivering ~87.7% of its rated tensor throughput. That is expected on a 700 W part under a dense load and it is a *chosen* trade.

**GPU4 additionally shows `0x20` — `SW Thermal Slowdown`.** Its die is at 79 °C and its memory at 86 °C, the hottest on the board, and it is the only GPU whose clock is being pulled below what the power budget alone requires. That is *not* a chosen trade: it is a cooling finding, localised to one position in the chassis. Walk the lesson-07 thermal path from the facility loop upward, and check whether GPU4 sits in the airflow shadow of a neighbour.

### Phase 7 — the reconciled diagram

```
 ══════════════════════════════════════════════════════════════════════════════
  NODE gpu-07 — RECONCILED TOPOLOGY   (all four tools agree unless marked ⚠)
  2 × Xeon Platinum 8480C · 2 TB DDR5 · 8 × H100 SXM5 · 8 × CX-7 · 4 × NVMe
 ══════════════════════════════════════════════════════════════════════════════

  ┌──────── SOCKET 0 / NUMANode0 ────────┐  ┌──────── SOCKET 1 / NUMANode1 ────────┐
  │ cpu 0-31,64-95   (32 cores, SMT on)  │  │ cpu 32-63,96-127 (32 cores, SMT on)  │
  │ 1007 GB DDR5 · local dist 10         │  │ 1008 GB DDR5 · local dist 10         │
  │                                      │  │                                      │
  │  GPU0 ─PIX─▶ mlx5_0   [Gen5 x16 ok]  │  │  GPU4 ─PIX─▶ mlx5_4   [Gen5 x16 ok]  │
  │  GPU1 ─PIX─▶ mlx5_1   [Gen5 x16 ok]  │  │  GPU5 ─PIX─▶ mlx5_5   [Gen5 x16 ok]  │
  │  GPU2 ─PIX─▶ mlx5_2   [Gen5 x8  ⚠]   │  │  GPU6 ─PIX─▶ mlx5_6   [Gen5 x16 ok]  │
  │  GPU3 ─PIX─▶ mlx5_3   [Gen5 x16 ok]  │  │  GPU7 ─PIX─▶ mlx5_7   [Gen5 x16 ok]  │
  │                                      │  │                                      │
  │  nvme0n1 (0000:c1:00.0) Gen5 x4      │  │  nvme2n1 (0000:04:00.0) Gen5 x4      │
  │  nvme1n1 (0000:c5:00.0) Gen5 x4      │  │  nvme3n1 (0000:08:00.0) Gen5 x4      │
  └───────────────────┬──────────────────┘  └──────────────────┬───────────────────┘
                      │                                        │
                      └────────── UPI 2.0 ─────────────────────┘
                        ~48 GB/s/dir/link · SLIT 21 vs 10
                        shared with ALL coherence + remote traffic

  ┌─── NVLINK DOMAIN (spans both sockets; does NOT use PCIe) ────────────────┐
  │  GPU0..GPU7 fully meshed: NV18 on every pair                             │
  │  4 × 3rd-gen NVSwitch on the baseboard · 18 links/GPU × 26.562 GB/s      │
  │  = 478 GB/s per direction, ~900 GB/s bidirectional per GPU               │
  │  verified: 144/144 links up, 0 replay / 0 recovery / 0 CRC errors  ✓     │
  └──────────────────────────────────────────────────────────────────────────┘

  ── ANNOTATIONS ───────────────────────────────────────────────────────────
  ⚠ GPU2 (0000:52:00.0): LnkSta x8 vs LnkCap x16 → 31.5 GB/s vs 63.0 GB/s.
     Host↔device bandwidth HALVED. Found ONLY by lspci -vvv; topo -m still
     reports PIX, dmesg is clean, nvidia-smi -q shows a healthy GPU.
  ⚠ GPU4: clocks_event_reasons 0x24 = SW Power Cap | SW THERMAL SLOWDOWN.
     Die 79 °C / mem 86 °C — hottest on the board, and the only GPU pulled
     below the power-budget clock. Cooling finding, not a chosen trade.
  ⚠ GPU0↔mlx5_4 = SYS. Never use as a GDR pair. Same for any cross-socket
     GPU/NIC combination: pin NCCL_IB_HCA to the PIX partner.
  ⚠ A socket-0 job writing checkpoints to nvme2n1/nvme3n1 crosses UPI on
     every write. Point the writer at nvme0n1/nvme1n1.
  ✓ All 8 GPU↔NIC pairs are PIX (rail-aligned). Device→NUMA agrees across
     lstopo, sysfs numa_node, and topo -m NUMA Affinity.
  ✓ dmesg: zero Xid/SXid. No hardware fault. GPU2 is a seating/BIOS issue.
  ── DELIVERY ──────────────────────────────────────────────────────────────
     mean clocks.sm / clocks.max.sm across 8 GPUs = 0.877
     → node delivering ~87.7% of rated tensor throughput under this load,
       attributable to the 700 W cap (chosen) plus GPU4's thermal (not chosen)
 ══════════════════════════════════════════════════════════════════════════════
```

### Phase 8 — measure the cost of a misalignment

A diagram predicts; a measurement proves. Force the misalignment and record the delta.

```
# ── Host↔device bandwidth, aligned vs remote memory ──────────────────────
$ numactl --cpunodebind=1 --membind=1 ./bandwidthTest --device=4 --memory=pinned --htod
 Host to Device Bandwidth, Pinned memory
   Transfer Size (Bytes)  Bandwidth(GB/s)
   33554432               55.3

$ numactl --cpunodebind=0 --membind=0 ./bandwidthTest --device=4 --memory=pinned --htod
   33554432               31.7

# delta: 55.3 → 31.7 GB/s  =  −42.7%   on ONE numactl flag
```

```
# ── NCCL all-reduce, aligned NIC vs cross-socket NIC ─────────────────────
$ NCCL_IB_HCA=mlx5_4 mpirun -np 8 ./all_reduce_perf -b 8 -e 1G -f 2 -g 1
#  size(B)   count  type  time(us)  algbw(GB/s)  busbw(GB/s)
   1073741824  ...  float   4821.3      222.7       389.7

$ NCCL_IB_HCA=mlx5_0 mpirun -np 8 ./all_reduce_perf -b 8 -e 1G -f 2 -g 1
   1073741824  ...  float   4903.1      219.0       383.2
```

**Read the second result carefully, because it teaches something the first does not.** The NCCL delta is small — ~1.7% — and that is *correct*, not a failed experiment: an 8-GPU intra-node all-reduce runs on the **NVLink mesh**, which spans both sockets and does not touch PCIe or UPI at all. The NIC choice barely matters because the NIC is barely used. **NVLink-bound collectives hide NUMA misalignment.** Host-staged paths — `cudaMemcpy`, the data loader, checkpoint writes, and *multi-node* collectives that must egress through a NIC — do not.

That is the single most important methodological point in the whole capstone: **pick a benchmark whose bytes actually traverse the path you are testing.** Report both results and explain the difference; a candidate who only reports the NCCL number concludes, wrongly, that alignment does not matter.

```
# ── Storage, aligned vs cross-socket ─────────────────────────────────────
$ fio --name=r --filename=/mnt/nvme2/bench --rw=read --bs=1M --direct=1 \
      --ioengine=io_uring --numjobs=4 --iodepth=32 --runtime=60 --time_based \
      --cpus_allowed=32-39                         # socket 1 CPUs, socket 1 drive
  READ: bw=12.7GiB/s (13.6GB/s)

$ fio --name=r --filename=/mnt/nvme2/bench --rw=read --bs=1M --direct=1 \
      --ioengine=io_uring --numjobs=4 --iodepth=32 --runtime=60 --time_based \
      --cpus_allowed=0-7                           # socket 0 CPUs, socket 1 drive
  READ: bw=7.1GiB/s (7.6GB/s)

# delta: 12.7 → 7.1 GiB/s  =  −44%
```

**Then state which production metric would and would not have caught each:**

| Signal | GPU2 ×8 link | GPU4 thermal | remote memory | cross-socket NVMe |
|---|---|---|---|---|
| `nvidia-smi utilization.gpu` | no | no | no | no |
| DCGM `SM_ACTIVE` / `TENSOR_ACTIVE` | partially (lower, no cause) | partially | partially | yes (sags on the stall cadence) |
| `clocks.sm` vs `clocks.max.sm` | no | **yes** | no | no |
| `clocks_event_reasons.active` | no | **yes (0x20)** | no | no |
| DCGM PCIe byte counters | **yes** (throughput ceiling) | no | **yes** | no |
| per-NUMA memory-bandwidth counters | no | no | **yes** | **yes** |
| `lspci` `LnkSta` at bring-up | **yes** | no | no | no |
| step-time histogram bimodality | no | no | partially | **yes** |

The empty column in that table is `utilization.gpu` — **it catches none of the four**, which is the entire argument for the exporter in §10.

## Practice

**This is the module deliverable.** See [`../practice/topology-teardown/README.md`](../practice/topology-teardown/README.md). Acceptance is the [module checkpoint](../checkpoint.md) — you are done when you can defend every reconciliation from memory, unassisted.

**Part A — reconcile a real node.** Rent one real GPU node (an 8×A100/H100 HGX box is ideal; a 1–2 GPU two-socket box covers most of it). Run the `collect.sh` script from Worked example phase 0 and commit `raw/` as evidence. Then work phases 1–7 in order and produce **one reconciled topology diagram** — annotated `lstopo.svg`, or ASCII in the style of phase 7 — showing:

- sockets and NUMA nodes with CPU ranges and memory
- each GPU's home NUMA node, cross-checked three ways (`lstopo`, sysfs `numa_node`, `topo -m` NUMA Affinity), with agreement or disagreement stated explicitly
- each GPU↔NIC pairing with its link class; every `NODE`/`SYS` pair marked
- NVMe placement, derived from BDF + `numa_node` since `topo -m` does not list them
- the NVLink domain with its verified link count and error state
- every `(downgraded)` link marked, with the bandwidth ratio computed
- the `dmesg` Xid/SXid result — including a clean negative
- the mean `clocks.sm / clocks.max.sm` ratio under load and which event bits were set

Plus a **one-paragraph read of the failure modes**: given this picture, where does a distributed job bottleneck, and why?

**Part B — measure the cost of misalignment.** Run at least two of the three experiments from phase 8:

```bash
# host↔device bandwidth: the cleanest demonstration
numactl --cpunodebind=<gpu's node> --membind=<gpu's node> ./bandwidthTest --device=<n> --htod
numactl --cpunodebind=<other node>  --membind=<other node>  ./bandwidthTest --device=<n> --htod

# storage: aligned vs cross-socket CPUs against the same drive
fio --direct=1 --ioengine=io_uring --bs=1M --numjobs=4 --iodepth=32 \
    --cpus_allowed=<local node's cpus>  --filename=<drive on that node>/bench ...
fio --direct=1 --ioengine=io_uring --bs=1M --numjobs=4 --iodepth=32 \
    --cpus_allowed=<remote node's cpus> --filename=<same drive>/bench ...

# collectives: expect a SMALL delta intra-node, and explain why
NCCL_IB_HCA=<PIX nic>  ./all_reduce_perf -b 8 -e 1G -f 2 -g <ngpus>
NCCL_IB_HCA=<SYS nic>  ./all_reduce_perf -b 8 -e 1G -f 2 -g <ngpus>
```

Then write **one page** containing:

1. **The numbers.** Aligned versus misaligned, with the percentage delta, for every experiment you ran. If the NCCL delta was small, **explain why** (NVLink-bound collectives bypass PCIe and UPI entirely) rather than reporting it as a null result — that explanation is worth more than the number.
2. **Which production metric would and would not have caught it**, as a table like phase 8's. Be explicit that `utilization.gpu` catches none of it.
3. **The exporter you would build** — the metric names, the label sets, the PromQL alert rules, and the §10 comparison table placing your design against PCIcrawler, FBAR, Modal's DCGM sweep, Crusoe AutoClusters and CoreWeave SUNK, so you can defend out loud which slice of the production problem you cover and which slice is the next layer up.

**Part C — the node scorecard.** Compute all four §9 calculations with your own inputs: required read bandwidth for your workload shape, checkpoint write time and its wall-clock percentage, the node power budget against its published maximum, and the airflow or coolant ΔT the resulting heat load implies. Put them on the front page of `teardown.md`. State every assumption.

**Part D — the Kubernetes half.** If the node is in a cluster, capture the kubelet policies and the two manager state files, and demonstrate an admit case and a `TopologyAffinityError` reject case as in lesson 05. If it is not, write the config you *would* apply and the `reservedMemory` arithmetic that makes it valid.

**Acceptance:** `teardown.md` (Part A + Part C) and `misalignment.md` (Part B), plus `lstopo.svg` and the full `raw/` capture, laid out as the deliverable README describes. Then answer every item in the [module checkpoint](../checkpoint.md) cold.

## Common pitfalls

1. **Failing to map identity before drawing conclusions.** `lspci`'s `0000:9d:00.0`, `lstopo`'s `cuda4` and `nvidia-smi`'s `GPU4` must be confirmed as one device — and `nvidia-smi`'s `pci.bus_id` uses an 8-digit uppercase format that will not string-match `lspci`'s. Normalise, then join. `nvidia-smi`'s `index` is not stable across reboots without `CUDA_DEVICE_ORDER=PCI_BUS_ID`; the UUID and BDF are.
2. **Treating `PIX` as proof the link is healthy.** `PIX` is bridge-count; `LnkSta` is trained width and speed. The worked example's GPU2 is `PIX` *and* ×8-degraded.
3. **Reading `NV#` as a category instead of a number.** A GPU with two dead NVLinks reports `NV16`, which looks like just another code. Count links with `nvidia-smi nvlink -s`, check errors with `-e`.
4. **Benchmarking with a workload that does not use the path under test.** An intra-node NCCL all-reduce runs on the NVLink mesh and shows ~0% delta between an aligned and a cross-socket NIC — true, and proof of nothing about NUMA alignment. Use `bandwidthTest`, `fio`, or a real training step for host-staged paths.
5. **Characterising thermals with a short run.** Power binds in seconds; thermals bind in minutes. A 60-second capture reports a clock that has not yet fallen. Run ≥10 minutes.
6. **Skipping the `dmesg` check, or not recording the clean negative.** A faulty component produces symptoms identical to a placement bug and needs a repair ticket, not `numactl`. The negative result is what licenses every other conclusion.
7. **Trusting one tool when two disagree.** NVIDIA/nccl #246 documents exactly this. Reconcile against a third source — the PCIe tree via `lspci -tv` and `lstopo`'s HostBridge nesting — rather than picking a favourite.
8. **Expecting `nvidia-smi topo -m` to show storage.** It has no NVMe rows. Derive every drive's position from its BDF and `numa_node`, and *say how you derived it*.
9. **Reporting the diagram without a measured number.** The diagram is the easy half; the measurement, the metric-coverage table and the exporter design are what make it a portfolio artifact.
10. **Presenting this as the production end-state.** Meta, CoreWeave, Crusoe and Modal automate this reconciliation continuously. Say so, so you neither undersell the deliverable nor overclaim.
11. **Ignoring AER counters.** A link healthy in an `LnkSta` snapshot can be retrying constantly under load. `CESta` trailing `+` flags and `dmesg | grep -i aer` are the leading indicator for the link that trains down next week.
12. **Forgetting that the non-GPU 45% of node power is real.** "8 × 700 W = 5.6 kW" is the most common capacity-planning error in this domain, and it is the one that trips breakers.

## Self-check

**(a) Given this matrix, which NIC do you use for GPUDirect RDMA to GPU3, and why?**

```
       mlx5_0  mlx5_2  mlx5_3   NUMA Aff
GPU3    NODE    SYS     PIX       1
```

**Answer:** `mlx5_3`. Its cell is `PIX` — the GPU3→NIC path traverses at most a single PCIe bridge, the shortest possible GDR route, and both devices sit on GPU3's home NUMA node 1, so the transaction is routed by one switch and never reaches the CPU or the socket interconnect. `mlx5_0` is `NODE`: same NUMA node but a *different* PCIe host bridge, so an extra host-bridge hop and a shared contention point. `mlx5_2` is `SYS`: it crosses the SMP interconnect (UPI/xGMI), which roughly halves effective RDMA bandwidth and adds latency, and shares that link with all coherence traffic on the box. GDR wants the traffic never to touch the inter-socket link, so `PIX` wins, `PXB` is the acceptable fallback, and `NODE`/`SYS` are to be avoided. **And then check `lspci -vvv -s <mlx5_3 bdf>` for `LnkSta` versus `LnkCap`** — `PIX` says one bridge away, not "×16 and healthy."

**(b) Name one thing each of the four tools does not show.**

**Answer:** `numactl --hardware` — **no PCIe or device information at all**; it cannot place a single GPU, NIC or drive. `lstopo` — not the *live NVLink mesh* (NVLink sits above PCIe and does not appear as a usable matrix) and not the *trained-under-load* link state; it shows configured topology, not `LnkSta`. `lspci -vvv` — **no NVLink** (it is not a PCIe link), no friendly NUMA affinity in the output you would read by hand, and GPUs appear as opaque vendor strings rather than `cudaN`. `nvidia-smi topo -m` — it **flattens PCIe to a single letter**, so a `PIX` link trained at ×4 looks identical to a healthy one; it lists **no NVMe devices at all**; and per NVIDIA/nccl #246 it can disagree with NCCL's own topology detection about the same edge on the same silicon.

**(c) "GPU at 100% utilization, throughput ~half spec, no error." Enumerate the host-side causes and the one command that confirms or eliminates each.**

**Answer:** `utilization.gpu` only means a kernel was resident in the sample window — it says nothing about clock, bandwidth or health, and it reads 100% through every branch below. Work top-down:

- **Hardware fault** (a component actually broke): `dmesg -T | grep -iE 'xid|sxid'` → any entry, especially Xid 79 ("GPU has fallen off the bus"), means cordon the node and file a repair ticket. Check this **first**, because a fault invalidates every other reading.
- **Power/thermal throttle** (clock below rated): `nvidia-smi --query-gpu=clocks.sm,clocks.max.sm,clocks_event_reasons.active,power.draw,enforced.power.limit,temperature.gpu,temperature.memory --format=csv` → the deficit is exactly `clocks.sm / clocks.max.sm`, and the bit says which constraint binds: `0x04` SW Power Cap (a decision), `0x20` SW Thermal (a symptom), `0x40` HW Thermal (emergency), `0x80` HW Power Brake, `0x10` Sync Boost (a *peer* GPU is holding this one down), `0x08` a roll-up to decompose.
- **PCIe link trained low**: `sudo lspci -vvv -s <bdf> | grep -E 'LnkCap:|LnkSta:'` → grep for `(downgraded)`. Gen3 ×8 on a Gen5 ×16 slot is exactly 8× less host bandwidth. Also check `CESta` for correctable errors indicating a link that retries under load.
- **Cross-socket GPU↔NIC path** (GDR over UPI): `nvidia-smi topo -m` → is the NIC the job actually uses `PIX`/`PXB`, or `NODE`/`SYS`? Cross-check with `NCCL_DEBUG=INFO`, which prints the NICs NCCL chose and how many channels it opened to each.
- **Remote memory / wrong NUMA binding**: `numastat -p <pid>` for where the pages actually are, plus `cat /proc/<pid>/status | grep -i cpus_allowed_list`, or in Kubernetes `kubectl exec -- cat /sys/fs/cgroup/cpuset.{cpus,mems}.effective`. Watch for the lesson-05 `TopologyInfo` trap: the policy can report success while the GPU is still remote.
- **Cross-socket or under-driven NVMe**: `readlink -f /sys/block/<dev>/device/device` plus its `numa_node` for placement, and an `fio --direct=1 --ioengine=io_uring` sweep at `iodepth=1` versus `32×4` to separate placement from concurrency. **Check queue depth first** — it is cheaper to test and more often the answer.
- **NVLink degraded**: `nvidia-smi nvlink -s -i <n>` for the link count and rate, `-e` for replay/recovery/CRC errors. Fewer than 18 active links caps peer-to-peer while `topo -m` still shows an `NV#` code.

Only after all seven are eliminated is it an application problem — and *then* you reach for a framework profiler.

**(d) Why is a PCIe device local to exactly one socket, and what does crossing the inter-socket link cost?**

**Answer:** Because the PCIe root complex containing the device's root port is **integrated into one CPU die**. `lstopo` makes this structural: a `HostBridge` is nested under exactly one `Package`/`NUMANode`. Every DMA read or write that device issues is serviced by that socket's memory-side fabric, and every MMIO access to it is routed there. Reaching the other socket's DRAM — or a device on the other socket — means traversing the inter-socket link. The cost has three parts. **Latency:** the SLIT matrix reports a relative index (`21` versus a local `10`), and real measured deltas on two-socket servers run 1.5–1.8×. **Bandwidth:** one UPI 2.0 link carries roughly 48 GB/s per direction, against per-socket DDR5 bandwidth of ~307 GB/s on a Sapphire Rapids box — about 6× narrower. **Contention:** that link is shared with all cache-coherence traffic, every other process's remote accesses, and any cross-socket peer-to-peer device traffic, so the effective penalty under load is worse than the static index suggests. A GPU wanting 63 GB/s of Gen5 ×16 host bandwidth simply cannot get it across a contended UPI link.

**(e) A GPU trained at Gen3 ×8 versus Gen5 ×16 — what is the bandwidth ratio, and how do you detect it?**

**Answer:** Exactly **8×**. Per-direction PCIe bandwidth is `GT/s × lanes × encoding_efficiency / 8`; both Gen3 and Gen5 use 128b/130b encoding, so Gen3 ×8 gives `8 × 8 × (128/130) / 8 = 7.88 GB/s` and Gen5 ×16 gives `32 × 16 × (128/130) / 8 = 63.02 GB/s`. The ratio decomposes cleanly: 2× per generation over two generations is 4×, times 2× the lane count is 8×. Detect it with `sudo lspci -vvv -s <bdf> | grep -E 'LnkCap:|LnkSta:'` and compare the two lines — the kernel appends the literal word `(downgraded)` to a `LnkSta` field that is below its `LnkCap`. **No other tool reports this**: `nvidia-smi topo -m` still shows the same relationship code, `lstopo` shows a correct tree, `numactl` is silent, `nvidia-smi -q` shows a healthy GPU, and `dmesg` has no entry. Pair the check with the AER `CESta` counters, since a link with a high correctable-error rate is the one that will train down next.

**(f) Which GPUs share socket 0 on a standard HGX H100 node, where does GPU5's NIC attach, and why is there no PCIe switch carrying GPU–GPU traffic?**

**Answer:** On the standard 4/4 split, **GPUs 0–3 are on socket 0** and GPUs 4–7 on socket 1 — confirm on your own box with `lstopo`'s Package nesting, sysfs `numa_node`, and `topo -m`'s NUMA Affinity column, all three of which must agree. **GPU5 is on socket 1, so its rail-aligned NIC is `mlx5_5` on socket 1's root complex**, which is exactly why `topo -m` reports `PIX` for that pair; pairing GPU5 with a socket-0 NIC would read `SYS`. As for the switch: **GPU↔GPU traffic on an H100-generation HGX baseboard does not use PCIe at all.** It runs on NVLink4 — 18 links per GPU at 26.562 GB/s each, ~900 GB/s bidirectional — fanned across four third-generation NVSwitch chips on the baseboard, which is why every GPU-to-GPU cell reads `NV18`. No PCIe switch is needed to carry peer traffic because peer traffic is not on PCIe; PCIe is only the host path and the NIC path. Whether a PCIe switch sits between a GPU and its paired NIC on the *host* side is an OEM board-design choice, and it is precisely that choice which determines whether a pair reads `PIX` or `PHB` — read it off your own machine.

**(g) You measure a 0% delta on an intra-node NCCL all-reduce between an aligned NIC and a cross-socket NIC. Did the experiment fail?**

**Answer:** No — it produced a correct and important result that is easy to misread. An 8-GPU **intra-node** all-reduce runs on the NVLink mesh, which spans both sockets and touches neither PCIe nor the inter-socket link, so the NIC is barely used and the NIC's link class barely matters. **NVLink-bound collectives structurally hide NUMA misalignment.** The misalignment is real and costly on every *host-staged* path: `cudaMemcpy` host↔device (the phase-8 measurement showed 55.3 → 31.7 GB/s, −43%, on one `numactl` flag), the data loader (12.7 → 7.1 GiB/s, −44%), checkpoint writes, and **multi-node** collectives that must egress through a NIC. The methodological lesson is the one to state out loud: **pick a benchmark whose bytes actually traverse the path you are testing**, report both results, and explain the difference. A candidate who reports only the NCCL number concludes, wrongly, that alignment does not matter.

**(h) At 16,384 GPUs, roughly how often does a training cluster hit an unexpected component failure, and why does that make manual reconciliation impractical?**

**Answer:** On the order of one every few hours. Meta's published data from training Llama 3 on a 16,384-GPU cluster reported **419 unexpected component failures over a 54-day run**, roughly half attributed to GPU and HBM3 issues. (Cite cautiously — Meta's own figure as compiled by secondary coverage, not independently re-derived here.) At that cadence a human running four tools per incident cannot keep pace, which is why production shops build continuous automated versions: Meta's **PCIcrawler** (PCIe topology + AER) and **FBAR** (~50× fewer training interruptions), Modal's `dcgmi` + `dmesg` sweeps, Crusoe's **AutoClusters** cordon/drain/replace on Xid-79-class failures, and CoreWeave's **SUNK** health probes. The manual skill remains the prerequisite: **you cannot validate or debug an automated detector's output if you cannot do the reconciliation by hand** — nor design the detector at all without knowing which tool owns which fact.

## Connections & what's next

This capstone is the module's synthesis point. Lesson 01's NUMA tree gives the frame; lesson 02's bandwidth hierarchy says why placement matters in bytes/sec; lesson 03's `LnkCap`/`LnkSta` is the check that found GPU2; lesson 04's reference layout is the "what should this look like" baseline; lesson 05's Topology Manager is the Kubernetes lever that *fixes* a misalignment, plus the `TopologyInfo` trap that makes a fix look applied when it is not; lesson 06's queue-depth arithmetic and GDS mode checks extend locality to storage; lesson 07's clocks-event bitmask, power budget and cooling ΔT supply the power/thermal hypothesis and the Xid/SXid fault layer. This lesson proves you can hold all seven at once, on a real machine, under interview pressure.

**This closes Module 02b.** The module's goal — understanding the machine underneath the accelerator well enough to place and debug GPU workloads — is what this capstone certifies. From here, the fluency is a direct prerequisite for the four modules the README lists as unlocked by 02b:

- **Module 03 (`gpu-hardware`)** — you now have the host-side placement vocabulary to go inside the GPU die itself (SM architecture, tensor cores, the on-package memory hierarchy) without re-deriving NUMA and PCIe fundamentals.
- **Module 04 (`gpu-on-kubernetes`)** — lesson 05's Topology Manager and device-plugin material, plus this lesson's exporter design, are the foundation for scheduling GPU workloads correctly in production clusters; 04 builds the operational layer on top.
- **Module 09 (`networking-topology`)** — the GPU↔NIC pairing and rail-alignment concepts from lessons 04 and 08 extend directly into multi-node network design: the same `PIX`/`SYS` logic one level up, across the fabric connecting many nodes, where the NCCL channel-count effects you observed intra-node become the dominant factor.
- **Module 10 (`bare-metal-lifecycle`)** — the `collect.sh` sweep and the acceptance-testing mindset in this capstone are exactly what a bare-metal bring-up process runs on every new node before it enters a fleet. Your collector is a first draft of a real acceptance test.

Before moving on, answer every item in the [module checkpoint](../checkpoint.md) cold and unassisted. That, not this lesson's self-check, is the actual gate.

## References & further reading

**Primary sources**

- **`lstopo` / hwloc manual** — `https://www.open-mpi.org/projects/hwloc/doc/` — the authoritative locality-tree tool. Read for the output-format flags (`--of console/txt/svg`), `--whole-io` (needed to see every I/O device rather than hwloc's filtered default), and how it labels `Package` / `NUMANode` / `HostBridge` / `PCIBridge` / `CoProc` / `Net` / `OpenFabrics` / `Block`. The only tool that prints a device's PCI BDF and its friendly name side by side, which is what makes it the identity-mapping Rosetta stone.
- **`lspci(8)` manual** — `https://man7.org/linux/man-pages/man8/lspci.8.html` — `-tvD` for the domain-qualified tree, `-vvv` for the capability dump containing `LnkCap`/`LnkSta` and the AER `CESta`/`UESta` registers, `-Dnn` for numeric IDs. The literal `(downgraded)` annotation on a `LnkSta` field is the string this whole module's PCIe-health check greps for.
- **`nvidia-smi` reference** — `https://docs.nvidia.com/deploy/nvidia-smi/index.html` — `topo -m` (and `topo -mp` for a PCIe-only view) with the `X`/`NV#`/`PIX`/`PXB`/`PHB`/`NODE`/`SYS` legend, `nvlink -s`/`-e` for per-link state and error counters, `-q -d` query groups, and the `--query-gpu` field names including `clocks_event_reasons.active` (with `clocks_throttle_reasons.active` as a deprecated alias).
- **NVIDIA GPUDirect RDMA documentation** — `https://docs.nvidia.com/cuda/gpudirect-rdma/` — the mechanism behind *why* `SYS` is expensive: RDMA directly between NIC and GPU memory over PCIe, and what crossing the root complex or the socket boundary does to it. Read the supported-systems and PCIe-topology sections.
- **NVIDIA DCGM — `dcgm_fields.h`** — `https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/dcgm_fields.h` — the field IDs the §10 exporter and alert rules reference, notably `DCGM_FI_DEV_SM_CLOCK` (100), `DCGM_FI_DEV_CLOCKS_EVENT_REASONS` (112), `DCGM_FI_DEV_GPU_TEMP_MARGIN_CELSIUS` (153), and the 160–164 power-limit hierarchy.
- **NVIDIA DGX H100 documentation and DGX SuperPOD H100 data-center design guide** — `https://docs.nvidia.com/dgx/dgxh100-user-guide/` and `https://docs.nvidia.com/dgx-superpod/design-guides/dgx-superpod-data-center-design-h100/latest/electrical.html` — the anchors for §9's arithmetic: 10.2 kW maximum system power from six 3,300 W Titanium PSUs in 4+2 redundancy, 5–30 °C operating inlet, and at rack level 415 V three-phase Wye at 32 A giving ~21.8 kW per circuit with three rPDUs in an N+1 scheme at 4 DGX H100 (>40 kW) per rack.

**Real-world engineering**

- **Meta Engineering — "How Meta keeps its AI hardware reliable"** — `https://engineering.fb.com/2025/07/22/data-infrastructure/how-meta-keeps-its-ai-hardware-reliable/` — **what it shows:** FBAR automated remediation, an explicit failure taxonomy including thermal runaway, and a reported ~50× reduction in training-interruption rate. The clearest anchor for why fleet-scale reconciliation tooling exists.
- **Meta Engineering — "How Facebook deals with PCIe faults to keep our data centers running reliably"** — `https://engineering.fb.com/2021/06/02/data-center-engineering/how-facebook-deals-with-pcie-faults-to-keep-our-data-centers-running-reliably/` — **what it shows:** PCIcrawler, an open-source CLI automating PCI/PCIe topology plus AER inspection — the direct production precedent for this capstone's Part A tooling.
- **Modal — "Keeping 20,000 GPUs healthy"** — `https://modal.com/blog/gpu-health` — **what it shows:** the Xid/SXid taxonomy in production, Xid 79 by name, and the `dcgmi` + `dmesg` passive fleet health-check pattern with a >88 °C operational threshold.
- **Crusoe — "AutoClusters: Automated GPU Failure Remediation for AI Training Clusters"** — `https://www.crusoe.ai/resources/blog/autoclusters-minimizing-hardware-failures-in-large-gpu-clusters` — **what it shows:** automated cordon/drain/replace on Xid-class failures, framed around queue-wait time as the largest controllable lever on cluster goodput.
- **CoreWeave — "NVIDIA H100 GPU Benchmark Results: What We Learned from Large-Scale GPU Testing"** — `https://www.coreweave.com/blog/nvidia-h100-gpu-benchmark-results-what-we-learned-from-large-scale-gpu-testing` — **what it shows:** SUNK topology-aware scheduling with health probes, and 49.2% MFU on 128 H100s as the fleet-level number this diagnostic chain protects (2025-era snapshot).

**Deeper dives**

- **NVIDIA/nccl issue #246** — `https://github.com/NVIDIA/nccl/issues/246` — **what it shows:** a real production case of NCCL's own topology detection reporting `PIX`/`PXB` where `nvidia-smi topo -m` reported `PHB` for the same GPU–NIC edge on an 8×V100 system. Direct evidence for this lesson's "no single tool is authoritative" claim and the reason §6 mandates a cross-check step.
- **Tom's Hardware coverage of Meta's Llama 3 training-failure data** — `https://www.tomshardware.com/tech-industry/artificial-intelligence/faulty-nvidia-h100-gpus-and-hbm3-memory-caused-half-of-the-failures-during-llama-3-training-one-failure-every-three-hours-for-metas-16384-gpu-training-cluster` — **what it shows:** a secondary-sourced summary of Meta's own published 16,384-GPU cluster failure numbers (419 failures over 54 days, roughly half GPU/HBM3). Use for the fleet-scale stakes framing, and cite it as secondary.
