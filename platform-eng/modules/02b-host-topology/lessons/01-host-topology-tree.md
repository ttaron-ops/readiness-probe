---
lesson: "02b.1"
title: "The host as a topology tree"
module: "02b"
concept: "The host as a topology tree"
status: not-started
est_time: "5h"
prev: null
next: "02-memory-subsystem.md"
artifacts: []
sources: 12
---

# 02b.1 · The host as a topology tree

> **Concept.** A GPU host is a tree — sockets → NUMA domains → PCIe root complexes → endpoints — and every GPU, NIC, and NVMe hangs off exactly one branch, fixed at board-design and boot time.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Where this fits

The Linux-internals module (01b) gave you NUMA as a *kernel* mechanic: first-touch allocation, `numa_balancing`, page migration, the cost model behind `numactl --hardware`'s distance numbers. That was all about how the kernel behaves once memory exists. This lesson starts a different module with a different question: before any kernel policy runs, **where in the physical tree does a device already live, and who decided that?** The answer is fixed in silicon and firmware, not runtime-adjustable — and it is the one placement fact every later lesson in this module builds on. Lesson 02 turns a branch of this tree into a measured bandwidth number. Lesson 03 zooms into one edge of it and reads whether that edge trained to spec. Lesson 04 supplies the reference shape of the whole tree for an 8-GPU server. Lesson 05 makes Kubernetes enforce the alignment you learn to read by hand here. Get this tree wrong in your head and every downstream number looks like a mystery instead of an obvious consequence.

## Why this matters

The single most common silent-waste pattern on a GPU box is a job whose host memory, CPU threads, and network path all sit on the *wrong* branch of the tree relative to the GPU it feeds. The GPU shows 100% utilisation while every byte it needs crosses the inter-socket link first. No dashboard flags it; `nvidia-smi` says busy, DCGM says the card is hot and drawing power, and the job simply runs slower than the identical job on the node next to it.

The reason no dashboard flags it is structural, not an oversight. Utilisation counters measure *whether the SMs issued instructions*, not *whether the bytes those instructions consumed arrived on the short path*. A GPU stalled on a DMA that is queued behind an inter-socket hop still issues instructions — it issues the same instructions, just fewer times per second. There is no counter anywhere in the stack whose value changes when a buffer moves from the right NUMA node to the wrong one. The only thing that changes is throughput, and throughput has no absolute reference point unless you know what the machine should have delivered.

That is what this lesson gives you: the ability to look at a machine and say what it *should* deliver, so that the number it actually delivers becomes evidence. This is the literal job description at neocloud and hyperscaler infra teams — CoreWeave's hardware-engineering postings call out "troubleshoot complex GPU and PCIe failures," and the fleet-scale tooling Meta and others have published exists because misplacement and marginal links are common enough to justify dedicated automation. In an interview, "why is a PCIe device local to exactly one socket, and what does crossing the inter-socket link cost?" separates people who have run these boxes from people who have only read about them.

## What's new here (calibration)

You already hold the on-prem physical picture (sockets, DIMMs, expansion slots) and, from the Linux-internals module, the *kernel mechanics* of NUMA: `numa_balancing`, first-touch allocation, page migration, node-local vs remote memory. Nothing below re-teaches those.

The delta is **framing the host as a static placement tree that accelerators attach to**, not as a memory allocator's runtime behaviour:

- A **PCIe device has a home NUMA node** the same way a page does — but it's fixed in silicon at boot, not decided by first-touch. The device's "first touch" already happened when the board was seated and the firmware enumerated it.
- The tree has **levels that matter for accelerators** that on-prem intuition usually stops short of: socket → die/quadrant → NUMA domain → PCIe host bridge → root port → switch → endpoint → function. On-prem you cared about levels 1 and maybe 2; GPU placement lives at levels 4–7.
- **NUMA-node count is not socket count, and the mechanism differs by vendor.** AMD NPS (Nodes Per Socket) and Intel SNC (Sub-NUMA Clustering) both split one socket into multiple NUMA domains, but *how* they do it is physically different.
- **The affinity is declared by firmware, not measured at runtime.** The ACPI tables the kernel trusts (`_PXM`, SRAT, SLIT, HMAT) are static firmware output; they can be wrong, and nothing checks them against reality.
- **There are three overlapping distance planes on a GPU host, not one** — the cache-coherent CPU/memory plane, the PCIe plane, and (on SXM boxes) the NVLink plane. They have different topologies, and a device can be near on one and far on another.

## Core concepts

### 1. The problem: "the machine" is not one thing

Write a program on a laptop and you can pretend the machine is flat: one pool of CPU, one pool of RAM, some devices. Every access costs about the same. The abstraction holds because the hardware is small enough that it is nearly true.

On a two-socket GPU server it stops being true by roughly a factor of two on latency and a factor of five to ten on bandwidth, depending on which pair of components you pick. And unlike a cache miss — which the hardware handles transparently and which shows up in `perf` — the cost of talking to the far half of the machine is invisible to every counter that a monitoring stack scrapes.

The reason the abstraction breaks is physical and simple: **memory controllers and PCIe controllers are on the CPU die.** They have been since Nehalem (2008) on Intel and since the first Opteron (2003) on AMD. There is no external northbridge that everything talks to equally. A DIMM is wired to the memory controller of one specific package. A PCIe slot's 16 lanes are wired to the SerDes of one specific package. When a core on package 0 wants a cache line that package 1's memory controller owns, that request leaves package 0, crosses a serial inter-socket link, gets serviced, and comes back. Nothing in the ISA changes; only the time does.

So the correct model of a GPU host is not a pool. It is a **tree**, where the leaves are the things that do work (cores, DIMMs, GPUs, NICs, NVMe drives) and the internal nodes are the places where traffic is aggregated and where crossing costs something. Every performance question on this class of machine reduces to: *for this transfer, how far up the tree do the two endpoints' paths have to meet?*

That question has a name in the hwloc project's vocabulary — the **common ancestor** of two objects — and it is exactly what `nvidia-smi topo -m` prints, one letter code per pair.

### 2. The tree, level by level, with real names

Here is the complete vocabulary. Get these distinctions right; staff-level interviewers probe exactly the places where people conflate two adjacent levels.

| Level | Object | What it is physically | How Linux names it |
|---|---|---|---|
| 0 | **Machine** | The whole coherent domain the OS boots on | `lstopo`'s root; `Machine (2011GB total)` |
| 1 | **Package / Socket** | One physical CPU part in one physical socket | `lscpu`'s `Socket(s):`; `lstopo`'s `Package L#0` |
| 2 | **Die / Tile / CCD** | A silicon chiplet inside the package | `lstopo`'s `Die L#0` when firmware exposes it; `/sys/devices/system/cpu/cpuN/topology/die_id` |
| 3 | **NUMA node** | A set of cores + the memory controllers whose DRAM they reach without a hop | `numactl --hardware`; `/sys/devices/system/node/nodeN/` |
| 4 | **Memory controller / channel** | The DDR interface itself; one channel = one independent path to DRAM | not directly exposed; inferred from `dmidecode -t memory` |
| 5 | **PCIe Host Bridge** | The logical device Linux enumerates for one PCIe segment/domain | `lspci -tv`'s `-[0000:00]-` lines; ACPI `PNP0A08` devices |
| 6 | **Root Port** | One PCIe-facing port of the Root Complex; the top of one link | `lspci` class `PCI bridge`, one hop below a host bridge |
| 7 | **Switch** | Silicon with one upstream port and N downstream ports | `PCI bridge` entries with siblings underneath |
| 8 | **Endpoint** | The GPU, NIC or NVMe itself | `3D controller`, `Ethernet controller`, `Non-Volatile memory controller` |
| 9 | **Function** | One addressable function of an endpoint (`.0`, `.1`, …) | the trailing digit of the BDF |

Three of these get conflated constantly, so state the difference out loud:

- The **Root Complex** is the whole subsystem inside the CPU that bridges the coherent fabric to PCIe. It is not a wire and not a device you can point at in `lspci`. One exists per socket (and, under SNC/NPS, its ports are attributed to specific quadrants).
- A **Root Port** is one PCIe-facing interface of that Root Complex. It is the *upstream end of one link*. A modern server CPU has many — Sapphire Rapids exposes 80 PCIe Gen5 lanes per socket, organised as several root ports; EPYC Genoa exposes 128 lanes per socket in 1P (fewer in 2P, because some are re-tasked as xGMI).
- A **Host Bridge** is the ACPI/Linux object that owns one *bus-number range and address window*. It is what you see as `-[0000:00]-` or `-[0000:e0]-` at the left edge of `lspci -tv`.

And the distinction that decides bandwidth: a **switch shares its upstream link** among everything below it, whereas **bifurcation** (splitting a root port's 16 lanes into two independent x8 links) does not — it partitions lanes rather than multiplexing traffic. Lesson 03 goes into both; here, just hold on to the fact that "two devices under the same box in the diagram" can mean either "they share bandwidth" or "they don't", depending on whether that box is a switch or a bifurcated root port.

### 3. The tree, drawn

This is the structural picture to hold in your head. It is a two-socket, four-NUMA-node, eight-GPU host — the shape of most rented "8× H100 SXM" instances — drawn the way `lstopo` orders things, with capacities and link types labelled on the edges.

```
 ════════════════════════════════════════ MACHINE (2 TiB total) ════════════════════════════════════════

 ┌──────────────────── Package L#0 (socket 0) ────────────────────┐  ┌──────── Package L#1 (socket 1) ───────┐
 │                                                                │  │                                       │
 │  ┌─── NUMANode L#0 ───┐        ┌─── NUMANode L#1 ───┐          │  │  NUMANode L#2       NUMANode L#3      │
 │  │ cores  0-27        │        │ cores 28-55        │          │  │  cores 56-83        cores 84-111      │
 │  │      112-139       │        │      140-167       │          │  │      168-195            196-223       │
 │  │ 512 GiB            │        │ 512 GiB            │          │  │  512 GiB             512 GiB          │
 │  │ 4× DDR5-4800 ch    │        │ 4× DDR5-4800 ch    │          │  │  4 ch                4 ch             │
 │  │ = 153.6 GB/s       │        │ = 153.6 GB/s       │          │  │                                       │
 │  └─────────┬──────────┘        └─────────┬──────────┘          │  │      │                   │            │
 │            │  ← SNC=2 splits ONE socket into two NUMA nodes    │  │      │                   │            │
 │            └──────────┬───────────────────┘                    │  │      └─────────┬─────────┘            │
 │                       │                                        │  │                │                      │
 │              ┌────────┴────────┐  Root Complex (on die)        │  │       ┌────────┴────────┐             │
 │              │ HostBridge      │                               │  │       │ HostBridge      │             │
 │              │  [0000:00]      │  bus 00-7f                    │  │       │  [0000:80]      │  bus 80-ff  │
 │              └───┬──────┬──────┘                               │  │       └───┬──────┬──────┘             │
 │       root port  │      │  root port                          │  │           │      │                    │
 │        Gen5 x16  │      │  Gen5 x16                           │  │           │      │                    │
 │            ┌─────┴──┐ ┌─┴──────┐                              │  │      ┌────┴───┐ ┌┴───────┐            │
 │            │ PCIe   │ │ PCIe   │  ← Broadcom Gen5 switches     │  │      │ PCIe   │ │ PCIe   │            │
 │            │ switch │ │ switch │     (OEM HGX boards; see L04) │  │      │ switch │ │ switch │            │
 │            │  SW0   │ │  SW1   │                               │  │      │  SW2   │ │  SW3   │            │
 │            └┬─┬─┬─┬─┘ └┬─┬─┬─┬─┘                              │  │      └┬─┬─┬─┬─┘ └┬─┬─┬─┬─┘            │
 │             │ │ │ │    │ │ │ │                                │  │       │ │ │ │    │ │ │ │              │
 │           GPU0│ │NVMe GPU2│ │NVMe                             │  │     GPU4 │ │     GPU6 │ │              │
 │             NIC0 GPU1   NIC2 GPU3                             │  │       NIC4 GPU5      NIC6 GPU7        │
 │                  NIC1        NIC3                             │  │            NIC5           NIC7        │
 └────────────────────────────────┬───────────────────────────────┘  └───────┬───────────────────────────────┘
                                  │                                          │
                                  └────────── UPI / Infinity Fabric ─────────┘
                                     2–4 links · SLIT distance 21 vs 10
                                     ~38–48 GB/s per direction per link

 ─────────── SECOND FABRIC, SAME GPUs, COMPLETELY DIFFERENT TOPOLOGY (see lesson 04) ───────────
   GPU0..GPU7 also each carry 18 NVLink-4 ports into 4 NVSwitches on the SXM baseboard.
   That graph is a full mesh: every GPU reaches every other at 450 GB/s per direction,
   and it does NOT pass through any box drawn above. Two networks, one set of leaves.
```

Read the diagram as a set of claims you can check on a real machine:

- **Every leaf has exactly one path upward.** GPU5 reaches host DRAM by climbing SW3 → root port → host bridge `[0000:80]` → socket 1's coherent fabric → NUMANode L#3's memory controllers. There is no second route. If the buffer it wants lives on NUMANode L#0, the path continues across UPI.
- **The height at which two leaves meet is their cost.** GPU4 and NIC4 meet at SW2 — one switch hop, no root complex involved, so a GPUDirect RDMA transfer between them never enters the CPU. GPU4 and NIC0 meet at the *machine* level, above both host bridges: every byte crosses UPI.
- **A NUMA node is a memory-side grouping, a host bridge is an I/O-side grouping, and they need not be 1:1.** In the drawing SNC=2 gives socket 0 two NUMA nodes but one host bridge. That asymmetry is real and it is where people's mental models break — see §6.
- **The NVLink plane is drawn separately because it *is* separate.** Nothing about the PCIe tree constrains GPU↔GPU bandwidth on an SXM node. Conversely, a perfect NVLink mesh tells you nothing about whether the host feed path is sane.

### 4. Who builds the tree, and when — the boot-time causal chain

The tree is not discovered by Linux from first principles. It is *declared*, by firmware, in a specific order, and Linux transcribes it. Understanding this order tells you exactly which lies you can be told and by whom.

```
  TIME ──────────────────────────────────────────────────────────────────────────▶

  [t0] POWER ON
        │
        │  Board traces already fix which SerDes on which package serve which
        │  slot/connector. Nothing after this point can change it.
        ▼
  [t1] PLATFORM FIRMWARE (BIOS/UEFI) — PCIe ENUMERATION
        │  Depth-first walk from each host bridge:
        │    · write bus number into each bridge's PRIMARY / SECONDARY /
        │      SUBORDINATE bus registers as it descends
        │    · read each function's Vendor/Device ID at every dev.fn
        │    · size each BAR (write 1s, read back the mask), assign MMIO
        │    · TRAIN each link — LTSSM negotiates speed & width (lesson 03)
        │  Output: the bus numbering you will later read out of a BDF.
        ▼
  [t2] FIRMWARE PUBLISHES ACPI TABLES
        │  SRAT  — which CPUs and which memory ranges belong to which
        │          "proximity domain" (= what Linux will call a NUMA node)
        │  SLIT  — an N×N matrix of *relative* distances between domains,
        │          10 = local by convention
        │  HMAT  — optional: actual latency/bandwidth attributes per
        │          initiator→target pair (newer firmware; CXL made it matter)
        │  DSDT/ ── per-device `_PXM` methods: "this PCI device is in
        │  SSDT     proximity domain N"
        ▼
  [t3] KERNEL BOOT — acpi_numa / PCI subsystem
        │  · parse SRAT → create NUMA nodes, set node→cpu and node→memory maps
        │  · parse SLIT → fill node_distance[][]
        │  · re-walk PCI (trusting firmware's bus numbers unless
        │    pci=realloc / assign-busses is passed)
        │  · for each PCI device, evaluate `_PXM` → dev->numa_node
        ▼
  [t4] SYSFS IS POPULATED — the API everything else reads
        │  /sys/devices/system/node/nodeN/{cpulist,meminfo,distance}
        │  /sys/bus/pci/devices/<bdf>/{numa_node,local_cpulist,local_cpus}
        ▼
  [t5] USERSPACE READS SYSFS
        │  numactl --hardware   ← node/distance view
        │  lscpu                ← cpu→node view
        │  lstopo               ← merges both + PCI tree into one object graph
        │  nvidia-smi topo -m   ← driver's own view + sysfs affinity columns
        │
        │  EVERY ONE OF THESE IS DOWNSTREAM OF FIRMWARE. If [t2] lied,
        │  all five tools repeat the lie, consistently and confidently.
        ▼
  [t6] WORKLOAD RUNS — and pays the real, physical cost from [t0],
        which may or may not be what [t2] said it would be.
```

Two consequences fall straight out of this chain, and both are the kind of thing that gets asked as a follow-up question.

**First: SLIT distances are declarations, not measurements.** The `21` you read from `numactl --hardware` is a number a firmware engineer typed. It is defined by ACPI as a *relative* figure with local normalised to 10, so 21 means "firmware asserts remote costs about 2.1× local." It is not sampled at boot, it does not account for load, and on broken firmware — early server BIOS revisions, several hypervisor BIOS implementations, some cloud instance types — it is simply wrong. Every NUMA-aware allocator in the kernel, including `numa_balancing` and the default `MPOL_LOCAL` behaviour, trusts it anyway. When a measurement and the SLIT disagree, believe the measurement.

**Second: `numa_node = -1` is a firmware defect indicator, not a "don't care" default.** The kernel writes `-1` (`NUMA_NO_NODE`) when no `_PXM` was found for the device. On a genuinely single-socket box that is harmless because there is only one answer. On a multi-socket box it means the allocator has no idea where this device is, so DMA buffers for it can land anywhere, and the scheduler will not steer its interrupt handlers or its consumers to the right cores. It is latent waste, waiting for a careless allocation:

```
$ cat /sys/bus/pci/devices/0000:17:00.0/numa_node
0
$ cat /sys/bus/pci/devices/0000:17:00.0/local_cpulist
0-27,112-139
$ cat /sys/bus/pci/devices/0000:17:00.0/local_cpus
0000ffff,ffff0000,0fffffff
```

`numa_node` is the node ID. `local_cpulist` is the human-readable set of CPUs on that node — this is the file to feed to `taskset`/`numactl` when you want a thread on the same node as a device. `local_cpus` is the same set as a hex bitmask, which is what older tooling parses. All three come from the same `_PXM` evaluation at [t3]; if `numa_node` is `-1`, `local_cpulist` will be the full CPU list, which reads like "this device is close to everything" and means "firmware didn't say."

There is a fourth ACPI table worth knowing by name even though you will rarely read it directly: **HMAT** (Heterogeneous Memory Attribute Table). Where SLIT gives a single relative integer, HMAT can give real latency (ns) and bandwidth (MB/s) figures per initiator/target pair, and it distinguishes read from write. It became important with CXL-attached memory, where "distance 21" is far too coarse a description. Linux surfaces it under `/sys/devices/system/node/nodeN/access0/initiators/{read_bandwidth,read_latency,write_bandwidth,write_latency}` when firmware provides it. If those files exist on your box, they are strictly better information than the SLIT.

### 5. Why a PCIe device is local to exactly one node

The short answer is the one from §1: the root complex is on the die, and the lanes are wired. But the mechanism deserves one more level of detail, because it explains both why you cannot change it and why the *reported* answer can be wrong while the physical answer is right.

A PCIe link is a set of differential lane pairs running between two SerDes blocks. One end is inside a CPU package; the other is in the GPU, NIC, drive or switch. The board designer decided at layout time which package's SerDes drive which connector. There is no crossbar in between — you cannot re-home a slot in firmware any more than you can re-solder it from the BIOS menu. (Bifurcation, lesson 03, changes how a root port's 16 lanes are *grouped*, not which package they come from.)

So "GPU3 is local to NUMA node 1" is a statement about copper. What varies is whether firmware tells the OS the truth about that copper:

```
   PHYSICAL FACT                    DECLARED FACT                 WHAT YOU OBSERVE
   ─────────────                    ─────────────                 ────────────────
   lanes wired to pkg 0   ──✓──▶   _PXM = 0 in SSDT   ──▶   numa_node 0.  Correct.

   lanes wired to pkg 0   ──✗──▶   no _PXM emitted    ──▶   numa_node -1. Kernel
                                                             treats device as
                                                             equidistant; buffers
                                                             land anywhere; the
                                                             physical cost is still
                                                             paid, silently.

   lanes wired to pkg 1   ──✗──▶   _PXM = 0 (BIOS bug) ──▶   numa_node 0.  Every tool
                                                             agrees, every tool is
                                                             wrong, and a "correct"
                                                             pinning makes it worse.
```

That third row is rare but it is why the module's capstone (lesson 08) insists on *reconciling four tools* rather than trusting one. Two of the four (`lstopo`, `numactl`) read the same sysfs values and cannot disagree with each other about `_PXM`. But `lspci`'s bus numbering comes from [t1] rather than [t2], and `nvidia-smi topo -m`'s link classification comes from the NVIDIA driver walking the PCIe hierarchy itself. When the bus-number story and the `_PXM` story point at different sockets, firmware is lying to one of them.

### 6. NUMA nodes vs sockets: SNC and NPS

The most common wrong assumption on a modern server is `NUMA nodes == sockets`. Both vendors ship features that break it, for the same reason and by different means.

**The reason.** A modern server die is not a single blob with one memory controller. Sapphire Rapids is four tiles bonded with EMIB; Genoa is up to twelve core chiplets around a central I/O die. Cores are physically closer to some memory controllers than others *within one package*. Presenting the whole package as one NUMA node averages that away — a thread gets uniform, mediocre latency because the address space is interleaved across all controllers. Presenting sub-package domains lets a well-pinned thread get the short path, at the price of a smaller memory pool per domain and a harder scheduling problem.

**Intel SNC (Sub-NUMA Clustering)** partitions the on-die mesh, its LLC slices, and its memory controllers into 2 or 4 logical domains. On Sapphire Rapids the BIOS exposes SNC2 and SNC4 alongside the default non-SNC modes (Intel's platform docs name the default clustering as Quadrant/Hemisphere — mesh partitioning for latency *without* exposing extra NUMA nodes to the OS). SNC requires symmetric memory population; with lopsided DIMM population the firmware will refuse to enable it or will produce badly-sized domains. The important mechanical point: **SNC is a partition of a fabric, not of silicon.** The mesh is one physical interconnect; SNC draws lines on it and maps address ranges accordingly.

**AMD NPS (Nodes Per Socket)** partitions along the package's physical quadrant structure. Genoa's I/O die has four quadrants, each owning three of the twelve DDR5 channels and a share of the PCIe lanes. `NPS1` interleaves memory across all 12 channels and presents one node per socket. `NPS2` presents two nodes of six channels each. `NPS4` presents four nodes of three channels each, matching the quadrants. There is also `NPS0` — available only in 2P — which interleaves across *both* sockets and presents the entire two-socket system as a single NUMA node. NPS0 is the option that makes a GPU box unmanageable: it deliberately destroys the locality information you are trying to use.

| | Intel SNC | AMD NPS |
|---|---|---|
| What is partitioned | the on-die mesh + LLC slices + memory controllers | package quadrants (I/O-die regions) and their channels |
| Physical basis | logical partition of one fabric | follows chiplet / quadrant layout |
| Typical settings | off (Quadrant/Hemisphere), SNC2, SNC4 | NPS0 (2P only), NPS1, NPS2, NPS4 |
| Channels per node (typical 2-socket) | 8 ch socket → 4/node at SNC2 | 12 ch socket → 6/node at NPS2, 3/node at NPS4 |
| Requirement | symmetric DIMM population | symmetric DIMM population |
| Result on `numactl` | 4 nodes on a 2-socket box at SNC2 | 8 nodes on a 2-socket box at NPS4 |

The distance matrix is how you tell which one you are looking at, and how many hops the firmware thinks exist. A two-socket box with SNC2 or NPS2 reports a three-tier matrix:

```
$ numactl --hardware
available: 4 nodes (0-3)
node 0 cpus: 0-27 112-139
node 0 size: 515084 MB
node 0 free: 498120 MB
node 1 cpus: 28-55 140-167
node 1 size: 515084 MB
node 1 free: 501773 MB
node 2 cpus: 56-83 168-195
node 2 size: 515084 MB
node 2 free: 502991 MB
node 3 cpus: 84-111 196-223
node 3 size: 515084 MB
node 3 free: 503002 MB
node distances:
node   0   1   2   3
  0:  10  11  21  21
  1:  11  10  21  21
  2:  21  21  10  11
  3:  21  21  11  10
```

Read it top to bottom:

- **4 nodes, and `lscpu` will say `Socket(s): 2`** → sub-socket partitioning is on.
- **`10` on the diagonal** — local, by ACPI convention.
- **`11` between 0↔1 and 2↔3** — same package, other sub-domain. Cheap: the request stays on one die's mesh, it just does not hit the nearest controller. This is the tier that does not exist on a plain two-node box.
- **`21` between {0,1} and {2,3}** — across the inter-socket link. Nodes 0 and 1 are socket 0; nodes 2 and 3 are socket 1.
- **The pairing in the matrix is how you recover the socket grouping** without `lscpu`: group nodes by which ones are `11` apart.

`numactl --hardware` will not tell you *which* mechanism produced the four nodes. `lscpu | grep -E 'Vendor|Model name|Socket'` will, because Intel and AMD parts are not confusable.

One more trap in this section: **more NUMA nodes does not imply more PCIe host bridges.** SNC2 on Sapphire Rapids gives you four NUMA nodes on a two-socket box but the PCIe lanes are still attributed to the socket's root complex; a device's `_PXM` will point at one of the two nodes within its socket, and which one is a firmware decision that varies by platform. So the count of nodes in `numactl` and the count of host-bridge lines in `lspci -tv` are independent facts, and expecting them to match is a reliable way to convince yourself a healthy machine is broken.

### 7. The inter-socket link: what crosses it and what it costs

Everything that has to get from one package to the other rides one link family: **UPI** (Ultra Path Interconnect) on Intel, **Infinity Fabric / xGMI** on AMD. What rides it:

1. **Remote DRAM reads and writes** — a core on socket 0 touching a line whose home controller is on socket 1.
2. **Cache-coherence traffic** — snoops, snoop responses, directory updates. This is *not* proportional to your data movement; it is proportional to *sharing*, and it consumes link bandwidth even when no data moves.
3. **PCIe DMA to remote memory** — a GPU on socket 1 DMA-ing into a buffer that lives in socket 0's DRAM. The DMA engine issues reads that the far controller services.
4. **Peer-to-peer PCIe across sockets** — a GPU on socket 0 writing to a NIC on socket 1. Some platforms support this poorly or not at all; NVIDIA's own guidance is that cross-socket P2P is not a supported fast path.

**The numbers, with the arithmetic shown.**

Intel UPI, Sapphire Rapids generation: UPI 2.0 at **16 GT/s**, links are **24 lanes** wide, and a socket has up to **4 links** (Intel's SPR platform documentation; Emerald Rapids raises the rate to 20 GT/s and Granite Rapids/Xeon 6 to 24 GT/s with up to 6 links).

```
raw per link per direction = 16 GT/s × 24 lanes ÷ 8 bits/byte
                           = 384 Gbit/s ÷ 8
                           = 48 GB/s

payload efficiency: Intel publishes UPI 1.0 as 10.4 GT/s × 20 lanes = 26 GB/s raw,
                    quoted as 20.8 GB/s per direction → 20.8 / 26 = 0.80
                    (this 0.80 is inferred from Intel's own two published numbers;
                     Intel does not publish a separate efficiency figure for UPI 2.0)

effective per link per direction ≈ 48 GB/s × 0.80 ≈ 38 GB/s
2-socket board with 3 links populated ≈ 115 GB/s per direction, aggregate
```

AMD xGMI, Genoa generation: up to **4 xGMI links** between sockets, each **16 lanes**, commonly configured at **32 Gbps per lane** on EPYC 9004 (BIOS-selectable; some platforms allow lower for power). Vendors expose a 3-link option that frees 32 PCIe lanes for slots.

```
raw per link per direction = 32 Gbit/s/lane × 16 lanes ÷ 8 = 64 GB/s
4 links                    = 256 GB/s per direction, raw
3 links                    = 192 GB/s per direction, raw, + 32 extra PCIe lanes
```

**Now compare against local DRAM, because the ratio is the whole point.**

```
Sapphire Rapids socket, 8 channels of DDR5-4800, 1 DIMM per channel:
  per channel  = 4800 MT/s × 8 B/transfer = 38.4 GB/s
  per socket   = 8 × 38.4 GB/s            = 307.2 GB/s   (peak, read+write combined)

EPYC Genoa socket, 12 channels of DDR5-4800:
  per socket   = 12 × 38.4 GB/s           = 460.8 GB/s

Ratio, local DRAM : one UPI link  =  307.2 : 38  ≈  8 : 1
Ratio, local DRAM : 3 UPI links   =  307.2 : 115 ≈  2.7 : 1
```

**So the inter-socket link is not "a bit slower memory." It is a pipe roughly three to eight times narrower than the local memory system, shared by every core on the socket, every device DMA that has to cross, and all coherence traffic.** A single streaming thread will not saturate it; sixteen threads doing remote streaming absolutely will, and when they do, the *latency* seen by everyone else on that link climbs too, because queueing.

Latency, for completeness: on current two-socket Xeon and EPYC platforms, idle local DRAM latency measures around **75–95 ns** and idle remote around **130–150 ns** — call it **1.5–1.8×**, consistent with the SLIT's 21/10 being a rough but honest index. Measure yours with Intel MLC's `--idle_latency` / `--latency_matrix` rather than trusting either the SLIT or this paragraph; the ratio is stable across platforms but the absolute numbers move with DIMM speed and rank count.

### 8. Reading the socket off a BDF — and why it is only a heuristic

Every PCIe function has an address written `DDDD:BB:DD.F`: a 16-bit **domain** (also called a segment), an 8-bit **bus**, a 5-bit **device**, and a 3-bit **function** — `0000:e3:00.0`.

At [t1] in the boot chain, firmware assigns bus numbers by a depth-first walk. Each host bridge's ACPI `_CRS` declares a *bus-number range* it owns, and firmware hands out numbers from that range as it descends. Because two host bridges cannot both own bus 0 in the same domain, platforms carve up the 0–255 bus space between them. That is why on a two-socket box you routinely see socket 0's devices in a low bus range and socket 1's in a high one:

```
$ lspci -D | grep -iE 'nvidia|mellanox'
0000:18:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB]
0000:2a:00.0 Infiniband controller: Mellanox Technologies MT2910 Family [ConnectX-7]
0000:3a:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB]
0000:5d:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB]
0000:9a:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB]
0000:ab:00.0 Infiniband controller: Mellanox Technologies MT2910 Family [ConnectX-7]
0000:ba:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB]
0000:db:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB]
```

Buses `18`–`5d` are below `0x80`; buses `9a`–`db` are above. On the overwhelming majority of two-socket x86 platforms that means the first four GPUs are on socket 0's host bridge (`[0000:00]`, owning buses `00`–`7f`) and the last four are on socket 1's (`[0000:80]`, owning `80`–`ff`).

Use this as a **five-second first guess, then confirm.** It is a heuristic, not a rule, for three concrete reasons:

- The split point is a firmware choice. Four-socket boxes carve into quarters; some platforms use separate PCI *domains* (`0000:`, `0001:`) per socket rather than bus ranges, in which case the bus numbers restart and the heuristic inverts.
- Adding or removing a switch changes how many bus numbers get consumed on each branch, so the same board with different risers produces different numbers.
- `pci=assign-busses` or a hotplug reservation policy will renumber everything.

The confirmation is one file: `/sys/bus/pci/devices/<bdf>/numa_node`. The heuristic is worth learning anyway because it lets you sanity-check that file — if the bus numbers say socket 1 and `_PXM` says node 0 on a two-socket box, you have found the firmware bug from §5's third row.

### 9. The five tools, and exactly what each one knows

Each tool reads a different part of the boot chain. Knowing which part is how you resolve disagreements instead of picking a favourite.

| Tool | Reads | Knows about | Blind to |
|---|---|---|---|
| `lscpu` | `/sys/devices/system/cpu/` | sockets, dies, cores, threads, CPU→node map | anything on PCIe |
| `numactl --hardware` | `/sys/devices/system/node/` | node count, memory per node, SLIT distances | which device is on which node |
| `lspci -tv` / `-D` | PCI config space + sysfs | the real PCIe hierarchy, bus numbering, link state | NUMA (unless you add `-nn` + sysfs lookups) |
| `lstopo` | both sysfs trees + PCI | the *merged* object graph — the only single view | firmware lies; it faithfully reproduces them |
| `nvidia-smi topo -m` | NVIDIA driver's own walk + sysfs | GPU↔GPU and GPU↔NIC common-ancestor class, NVLink, per-GPU CPU/NUMA affinity | non-NVIDIA endpoints it wasn't told about |

**`lscpu` — the CPU side.**

```
$ lscpu | grep -E 'Model name|Socket|Core|Thread|NUMA'
Model name:                      Intel(R) Xeon(R) Platinum 8480C
Thread(s) per core:              2
Core(s) per socket:              56
Socket(s):                       2
NUMA node(s):                    4
NUMA node0 CPU(s):               0-27,112-139
NUMA node1 CPU(s):               28-55,140-167
NUMA node2 CPU(s):               56-83,168-195
NUMA node3 CPU(s):               84-111,196-223
```

56 cores × 2 sockets × 2 threads = 224 logical CPUs. Four NUMA nodes over two sockets → SNC2. The CPU-list format is the giveaway for hyperthread numbering: Linux enumerates all first-siblings (0–111) before all second-siblings (112–223), so `0-27,112-139` is 28 physical cores and their 28 siblings, not 56 distinct cores. Pin to `0-27` if you want one thread per core; pin to the whole list if you want both siblings.

**`numactl --hardware` — the memory side.** Covered in §6. The one extra thing to read is `free` per node: a node with much less free memory than its siblings is where something already landed, and is the node your next first-touch will *not* want to spill from.

**`lstopo` — the merged view, and the single best "draw it for me" command.** Install `hwloc`. Then:

```
$ lstopo --output-format console --no-caches --whole-io
Machine (2011GB total)
  Package L#0
    NUMANode L#0 (P#0 503GB)
    L3 L#0 (105MB)
      Core L#0
        PU L#0 (P#0)
        PU L#1 (P#112)
      ...
    HostBridge L#0
      PCIBridge
        PCIBridge
          PCI 10de:2330 (3D)
            GPU(OpenCL) "opencl0d0"
            CoProc(CUDA) "cuda0"
          PCI 15b3:1021 (InfiniBand)
            Net "ibp24s0"
            OpenFabrics "mlx5_0"
      PCIBridge
        PCI 144d:a80a (NVMExpress)
          Block(Disk) "nvme0n1"
  Package L#1
    NUMANode L#2 (P#2 503GB)
    ...
```

Flags worth knowing, because the defaults hide the parts you care about:

- `--output-format console` (or `txt`) prints the tree as text; `lstopo topo.png`/`topo.pdf`/`topo.svg` renders an image; `--output-format xml` dumps a machine-readable graph you can diff across nodes at bring-up.
- `--no-caches` collapses the L1/L2/L3 lines so the PCI tree is readable.
- `--whole-io` shows *all* I/O devices, not just the ones hwloc thinks are interesting; without it, some bridges are merged away and the hierarchy looks shallower than it is.
- `-p` / `--physical` prints OS/physical indices instead of hwloc's logical `L#` numbering. **This matters more than it looks.** `NUMANode L#2 (P#2 …)` means "hwloc's second NUMA node, which the OS calls node 2." When they diverge — and they do on machines with a node that has memory but no CPUs, or CPUs but no memory — every `numactl` argument you write must use the `P#` number.
- `--filter` controls what is kept: `--filter cache:none`, `--filter io:all`, etc.

The killer feature is `lstopo --output-format xml > node.xml` at bring-up, stored per node. A `diff` of two nodes' XML is a complete, machine-checkable statement of "these two boxes are or are not the same shape," which is exactly the acceptance test §Perspectives argues for.

**`lspci -tv` — the ground truth for hierarchy.** Indentation is depth; the leftmost `-[0000:00]-` entries are host bridges.

```
$ lspci -tv
-+-[0000:c0]-+-01.1-[c1-c6]----00.0-[c2-c6]--+-04.0-[c3]----00.0  NVIDIA GH100 [H100 SXM5]
 |           |                               \-08.0-[c5]----00.0  Mellanox MT2910 [ConnectX-7]
 |           \-02.0-[c7]----00.0  Samsung NVMe SSD
 +-[0000:80]-+-01.1-[81-86]----00.0-[82-86]--+-04.0-[83]----00.0  NVIDIA GH100 [H100 SXM5]
 |           |                               \-08.0-[85]----00.0  Mellanox MT2910 [ConnectX-7]
 \-[0000:00]-+-00.0  Intel Device 09a2
             +-01.1-[01-06]----00.0-[02-06]--+-04.0-[03]----00.0  NVIDIA GH100 [H100 SXM5]
             |                               \-08.0-[05]----00.0  Mellanox MT2910 [ConnectX-7]
             \-1f.0  Intel C741 LPC Controller
```

Read one branch character by character:

- `-[0000:00]-` — host bridge for domain 0, bus 0. This is socket 0's root complex as Linux sees it.
- `+-01.1-` — device `01`, function `1` on bus `00`: a **root port**. Its class is `PCI bridge`.
- `[01-06]` — the bus-number range behind that bridge (its secondary through subordinate bus registers from [t1]). Six buses reserved for what is below.
- `----00.0-` — a single device on bus `01`: the **upstream port of a PCIe switch**.
- `[02-06]` — the switch's own downstream bus range.
- `+-04.0-[03]----00.0 NVIDIA` — a **downstream port** of the switch at `02:04.0`, behind which bus `03` holds the GPU at `03:00.0`.
- `\-08.0-[05]----00.0 Mellanox` — a *sibling* downstream port with the NIC behind it.

**Those last two lines are the load-bearing observation of this whole lesson.** The GPU and the NIC are siblings under one switch. Their common ancestor is the switch, not the root complex. A DMA between them can be routed by the switch without ever entering the CPU — that is what makes GPUDirect RDMA cheap for this pair, and it is exactly what `nvidia-smi topo -m` will report as `PIX` or `PXB`.

`lspci -PP -s <bdf>` prints the full path from host bridge to device on one line, which is handy in scripts:

```
$ lspci -PP -s 03:00.0
00:01.1/01:00.0/02:04.0/03:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB]
```

**`nvidia-smi topo -m` — the accelerator view.** See §10.

**sysfs — the tiebreaker.** When two tools disagree, these files are what three of them were reading anyway:

```
$ cat /sys/bus/pci/devices/0000:03:00.0/numa_node          # GPU's node
0
$ cat /sys/class/net/ibp24s0/device/numa_node               # NIC's node, via netdev
0
$ cat /sys/class/nvme/nvme0/device/numa_node                # NVMe's node
0
$ cat /sys/devices/system/node/node0/distance
10 11 21 21
$ readlink -f /sys/class/net/ibp24s0/device                 # NIC → its BDF
/sys/devices/pci0000:00/0000:00:01.1/0000:01:00.0/0000:02:08.0/0000:05:00.0
```

That last `readlink` is the highest-value one-liner in the section: the path *is* the tree. `0000:00:01.1` (root port) → `0000:01:00.0` (switch upstream) → `0000:02:08.0` (switch downstream) → `0000:05:00.0` (the NIC). Do the same for the GPU and compare prefixes; the length of the shared prefix is the height of the common ancestor.

### 10. Decoding `nvidia-smi topo -m`

This matrix is the accelerator-native summary of everything above. Each cell is a **classification of the common ancestor** of a pair of devices. Memorise the codes; you will be asked to read a matrix cold.

| Code | Meaning | Where the paths meet | Approximate cost |
|---|---|---|---|
| `X` | self | — | — |
| `NV#` | a bonded set of **#** NVLinks | the NVLink fabric — **not** the PCIe tree at all | highest bandwidth on the box; `NV18` on H100 = 18 links |
| `PIX` | at most **one** PCIe bridge | one switch downstream-port pair | best PCIe case; P2P stays in the switch |
| `PXB` | **multiple** PCIe bridges, without touching the host bridge | a switch further up, or a multi-stage switch | still no CPU involvement; slightly more hops |
| `PHB` | traverses a **PCIe Host Bridge** (i.e. the CPU's root complex) | the root complex | P2P must go up into the CPU and back down |
| `NODE` | across PCIe host bridges **within one NUMA node** | the fabric inside one NUMA domain | on-die hop, no socket crossing |
| `SYS` | across the **inter-socket** interconnect (UPI / xGMI) | the machine root | worst case: socket crossing, and P2P may be unsupported |

The ordering from best to worst is `NV#` > `PIX` > `PXB` > `PHB` > `NODE` > `SYS`. Two subtleties people get wrong:

- **`PHB` and `NODE` are both "the CPU is involved," and `NODE` is the worse of the two.** `PHB` means one host bridge was traversed; `NODE` means the path went between *different* host bridges that happen to belong to the same NUMA node. The extra hop is real.
- **`NV#` describes GPU↔GPU only.** A GPU↔NIC cell can never be `NV#` on standard hardware; the NIC is not on the NVLink fabric. (Grace-Hopper and GB200 change the CPU side of this with NVLink-C2C, but not the NIC side.)

A representative matrix from a two-socket, 8-GPU SXM node — this is the shape you should expect and the shape lesson 04 will make you draw from memory:

```
$ nvidia-smi topo -m
        GPU0  GPU1  GPU2  GPU3  GPU4  GPU5  GPU6  GPU7  NIC0  NIC1  NIC2  NIC3  CPU Affinity      NUMA Affinity
GPU0     X    NV18  NV18  NV18  NV18  NV18  NV18  NV18  PIX   NODE  SYS   SYS   0-27,112-139      0
GPU1    NV18   X    NV18  NV18  NV18  NV18  NV18  NV18  NODE  PIX   SYS   SYS   0-27,112-139      0
GPU2    NV18  NV18   X    NV18  NV18  NV18  NV18  NV18  NODE  NODE  SYS   SYS   28-55,140-167     1
GPU3    NV18  NV18  NV18   X    NV18  NV18  NV18  NV18  NODE  NODE  SYS   SYS   28-55,140-167     1
GPU4    NV18  NV18  NV18  NV18   X    NV18  NV18  NV18  SYS   SYS   PIX   NODE  56-83,168-195     2
GPU5    NV18  NV18  NV18  NV18  NV18   X    NV18  NV18  SYS   SYS   NODE  PIX   56-83,168-195     2
GPU6    NV18  NV18  NV18  NV18  NV18  NV18   X    NV18  SYS   SYS   NODE  NODE  84-111,196-223    3
GPU7    NV18  NV18  NV18  NV18  NV18  NV18  NV18   X    SYS   SYS   NODE  NODE  84-111,196-223    3
NIC0    PIX   NODE  NODE  NODE  SYS   SYS   SYS   SYS    X    NODE  SYS   SYS
NIC1    NODE  PIX   NODE  NODE  SYS   SYS   SYS   SYS   NODE   X    SYS   SYS
NIC2    SYS   SYS   SYS   SYS   PIX   NODE  NODE  NODE  SYS   SYS    X    NODE
NIC3    SYS   SYS   SYS   SYS   NODE  PIX   NODE  NODE  SYS   SYS   NODE   X

Legend:
  X    = Self
  SYS  = Connection traversing PCIe as well as the SMP interconnect between NUMA nodes (e.g., QPI/UPI)
  NODE = Connection traversing PCIe as well as the interconnect between PCIe Host Bridges within a NUMA node
  PHB  = Connection traversing PCIe as well as a PCIe Host Bridge (typically the CPU)
  PXB  = Connection traversing multiple PCIe bridges (without traversing the PCIe Host Bridge)
  PIX  = Connection traversing at most a single PCIe bridge
  NV#  = Connection traversing a bonded set of # NVLinks
```

*(Representative transcript — the exact NIC count, code mix and affinity ranges differ per SKU. Capture your own; the reading method is what transfers.)*

Six things to read off it, in order:

1. **The whole GPU×GPU block is `NV18`.** Every GPU pair reaches every other over 18 bonded NVLink-4 links. That block being uniform is the signature of an NVSwitch-based baseboard: no pair is privileged. If you saw a mix of `NV18` and `NV4`/`SYS` in that block, you would be looking at a direct-attach (bridge-connected) board or a broken fabric, not an NVSwitch one.
2. **The NUMA Affinity column splits 0,0,1,1,2,2,3,3.** Four NUMA nodes, two GPUs each — consistent with SNC2 on a two-socket box, GPUs 0–3 on socket 0 and 4–7 on socket 1.
3. **`PIX` appears exactly once per GPU in the NIC columns.** GPU0↔NIC0, GPU1↔NIC1, GPU4↔NIC2, GPU5↔NIC3. Those are the switch-sibling pairs from the `lspci -tv` reading in §9 — the pairs where GPUDirect RDMA is a straight shot.
4. **`NODE` between GPU0 and NIC1** means they are on the same NUMA node but under different host bridges — usable, one extra hop, not a fault.
5. **`SYS` between GPU0 and NIC2** is the red flag class. Every byte crosses the inter-socket link, and NVIDIA does not treat cross-socket P2P as a supported fast path, so GPUDirect RDMA will typically be *disabled* for that pair rather than merely slow — the traffic falls back to staging through host memory.
6. **The matrix is symmetric, and the CPU Affinity column is the actionable output.** `0-27,112-139` for GPU0 is precisely the argument to give `taskset`/`numactl --cpunodebind` for a process feeding GPU0.

Two more invocations worth knowing:

```
$ nvidia-smi topo -m -p        # same matrix but PCI-path-based classification only
$ nvidia-smi topo -p2p rw      # per-pair peer-to-peer read/write capability: OK / CNS / NS
```

`topo -p2p` is the one that catches the ACS problem in §11 — a pair can be `PIX` in the matrix and still report `CNS` ("chipset not supported") for P2P.

### 11. IOMMU and ACS: when the drawn tree is not the usable tree

The tree tells you where a short path *exists*. Whether traffic is *allowed* to take it is a separate question, decided by PCIe **Access Control Services (ACS)**.

The problem ACS solves: in a virtualised system, if two devices assigned to different guests sit under the same switch, the switch could route a transaction from one directly to the other, never passing it to the IOMMU for translation and permission checking. That is a VM-escape-shaped hole. ACS closes it by making the switch's downstream ports *redirect* peer-to-peer transactions upstream to the root complex, where the IOMMU can inspect them.

The performance consequence: a GPU→NIC transfer between two switch siblings, which the topology says should be one switch hop, instead goes **switch → root port → root complex → root port → switch → NIC**. It still works; it is now bounded by the root complex's P2P path instead of the switch's crossbar, and on many platforms the root-complex P2P path is dramatically slower or simply unsupported. `nvidia-smi topo -m` still says `PIX`, because `PIX` describes the hierarchy, not the routing policy.

Diagnosis:

```
$ sudo lspci -vvv -s 02:04.0 | grep -A2 'Access Control Services'
    Capabilities: [xxx v1] Access Control Services
        ACSCap: SrcValid+ TransBlk- ReqRedir+ CmpltRedir+ UpstreamFwd+ EgressCtrl- DirectTrans-
        ACSCtl: SrcValid+ TransBlk- ReqRedir+ CmpltRedir+ UpstreamFwd+ EgressCtrl- DirectTrans-
```

`ACSCap` is what the port *can* do; `ACSCtl` is what is *enabled*. `ReqRedir+` and `CmpltRedir+` in `ACSCtl` on a downstream port are what force P2P upstream. Confirm the functional consequence rather than reasoning from the bits alone:

```
$ nvidia-smi topo -p2p rw          # look for CNS/NS instead of OK on PIX-labelled pairs
$ ./p2pBandwidthLatencyTest        # cuda-samples: compare P2P=Enabled vs Disabled matrices
```

**State the trade-off honestly, because "just disable ACS" is bad advice delivered as good advice.** Turning ACS off on the switch restores the short path and typically restores full P2P bandwidth. It also removes the isolation guarantee that makes VFIO passthrough safe, and it collapses IOMMU groups — devices that were individually assignable become assignable only as a group. On a bare-metal, single-tenant training node that is usually an acceptable trade. On a multi-tenant node that hands GPUs to customer VMs, it is not. Know which kind of node you are on before touching it.

Related: the IOMMU's own mode matters even without ACS. `intel_iommu=on` with strict invalidation adds an IOTLB flush on every unmap; `iommu=pt` (passthrough) sets up an identity mapping for host-owned devices and removes most of that cost. Lesson 02 picks up the second-order effect (SWIOTLB bounce buffers) that shows up when the IOMMU cannot map a DMA directly.

### 12. Putting a cost on each path

Here is the mechanical payoff. For any transfer on a GPU host, classify the pair, then read the ceiling off the table. Numbers are per direction and are *link* ceilings — achievable throughput is lower (lesson 02 measures the gap).

| Pair, and how they meet | Ceiling per direction | Where the number comes from |
|---|---|---|
| GPU↔GPU over NVLink-4 (`NV18`) | **450 GB/s** | 18 links × 25 GB/s/dir (NVIDIA quotes 900 GB/s bidirectional) |
| GPU↔GPU over PCIe Gen5 x16 through a switch (`PIX`) | 63 GB/s | 32 GT/s × 16 lanes × 128/130 ÷ 8 |
| GPU↔host DRAM, local node | 63 GB/s | same link; bounded by PCIe, not DRAM |
| GPU↔host DRAM, remote node (`SYS` path for DMA) | ~38 GB/s and falling under load | one UPI 2.0 link's effective rate |
| GPU↔NIC same switch (`PIX`), GPUDirect RDMA | min(63 GB/s PCIe, 50 GB/s NIC) | ConnectX-7 at 400 Gb/s = 50 GB/s |
| GPU↔NIC across sockets (`SYS`) | GPUDirect typically disabled → host-staged | two PCIe crossings + one UPI crossing |
| Core↔local DRAM | 307.2 GB/s per socket | 8 ch × DDR5-4800 × 8 B |
| Core↔remote DRAM | ~38 GB/s per link, ~115 GB/s over 3 | UPI arithmetic in §7 |

**Worked: what does one GPU→GPU transfer of 1 GiB cost on each path?**

```
Payload: 1 GiB = 1,073,741,824 B

(a) NVLink-4, 18 links, 450 GB/s per direction
    t = 1.0737e9 B ÷ 450e9 B/s = 2.39 ms

(b) PCIe Gen5 x16 through a shared switch, 63 GB/s
    t = 1.0737e9 ÷ 63e9        = 17.0 ms          → 7.1× slower than NVLink

(c) Same, but the two GPUs are on different sockets, so the transfer is
    staged through host memory (GPU0 → DRAM0 → UPI → DRAM2 → GPU4):
      leg 1  GPU0 → DRAM0 over PCIe Gen5      1.0737e9 ÷ 63e9  = 17.0 ms
      leg 2  DRAM0 → DRAM2 over one UPI link  1.0737e9 ÷ 38e9  = 28.3 ms
      leg 3  DRAM2 → GPU4 over PCIe Gen5      1.0737e9 ÷ 63e9  = 17.0 ms
      serial total (no overlap)                                = 62.3 ms
      with perfect pipelining, bounded by the slowest leg      = 28.3 ms
                                                    → 12–26× slower than NVLink
```

That spread — 2.4 ms to 62 ms for the same logical operation — is the entire reason this module exists. And note which quantity moved: not the GPU, not the code, not the data. **Only which branch of the tree the two endpoints happened to sit on.**

**Worked: why a NIC on the wrong root complex halves effective bandwidth.**

Take GPU4 (socket 1, node 2) and a 400 Gb/s ConnectX-7 that was cabled into a socket-0 slot. The intended path and the actual path:

```
INTENDED (NIC on GPU4's switch, `PIX`):
   wire → NIC → switch crossbar → GPU4 HBM
   one PCIe hop, no host DRAM touched, no UPI.
   ceiling = min(NIC 50 GB/s, PCIe Gen5 x16 63 GB/s) = 50 GB/s

ACTUAL (NIC on socket 0, `SYS`):
   GPUDirect RDMA is not used across sockets. The path becomes:
   wire → NIC → PCIe → socket-0 root complex → DRAM (node 0)     [PCIe leg 1]
        → UPI → socket-1 root complex                            [UPI leg]
        → PCIe → GPU4 HBM                                        [PCIe leg 2]

   Each 400 Gb/s of inbound wire traffic now consumes:
     50 GB/s of socket-0 PCIe write bandwidth
     50 GB/s of UPI bandwidth in one direction  ← out of ~38 GB/s effective
     50 GB/s of socket-1 PCIe read bandwidth
     plus 100 GB/s of DRAM traffic on node 0 (write then read)

   The UPI link cannot supply 50 GB/s. It supplies ~38 GB/s at best, and it is
   shared with every other cross-socket flow on the box. Effective NIC throughput
   collapses to the UPI share: ~38 GB/s in the best case, and in a multi-rail node
   where two or more misplaced NICs contend, ~19 GB/s each.
```

So "halves" is not a figure of speech and not a rule of thumb — it is `min(link ceilings along the path)` with the inter-socket link inserted into a path that was designed not to contain it, divided by however many flows are sharing it. **A wrong-socket NIC does not add latency to a fast path; it replaces the fast path with a different, narrower one.**

## Perspectives

**Developer.** You never see the tree from application code. CUDA presents "GPU 0..N"; PyTorch presents `cuda:0`. The tree becomes visible only when you `numactl`-wrap a process, set `CUDA_VISIBLE_DEVICES`, or call `cudaDeviceGetPCIBusId`. The characteristic failure is treating the logical GPU index as if it implied physical placement — index 0 is whatever the driver enumerated first, and `CUDA_DEVICE_ORDER` defaults to `FASTEST_FIRST`, not `PCI_BUS_ID`, so the enumeration order can differ from the BDF order on heterogeneous boxes. If you are writing rank→device→NUMA binding logic, set `CUDA_DEVICE_ORDER=PCI_BUS_ID` first so the mapping is stable across reboots and driver versions.

**Operator / SRE.** The tree is an acceptance-test artefact, not a curiosity. `lstopo --output-format xml`, `lspci -Dnnvv`, `numactl --hardware` and `nvidia-smi topo -m` captured at bring-up and diffed against the vendor reference is a five-minute job that catches: SNC/NPS flipped between firmware revisions, a riser populated differently on one node in a batch, a NIC in the wrong slot, ACS enabled where the fleet expects it off. All of those are invisible later and all of them are cheap now. The failure mode this prevents is discovering, six weeks into a training run, that node 47 is shaped differently from the other 63.

**Hardware / kernel.** This is the one placement decision in the whole module that the kernel cannot fix later. Pages can be migrated; interrupts can be re-steered; a device's home node cannot change without a reboot, and its physical wiring cannot change without a screwdriver. Everything the kernel offers — mempolicy, IRQ affinity, XPS/RPS, Topology Manager — is downstream compensation for a decision made at [t0].

**Economics.** A `numa_node = -1` or an unnoticed SNC-on box costs nothing by itself; it costs money only in combination with careless placement. That is why this lesson is a *precondition* for lessons 02 and 05's dollar arguments rather than a dollar argument on its own. The economic framing that does land here is the acceptance-test one: capturing topology at bring-up is a fixed, tiny cost that converts an unbounded, undetectable class of loss into a bounded, detectable one.

## Real-world use cases

- **Meta Engineering — fleet hardware reliability at scale** ([engineering.fb.com](https://engineering.fb.com/2020/12/09/data-center-engineering/how-facebook-keeps-its-large-scale-infrastructure-hardware-up-and-running/)). The substance: Meta describes running automated, continuous hardware validation across a fleet large enough that "someone will notice" is not a detection strategy. The relevant lesson for you is not the specific tooling but the shape of the argument — at fleet scale, topology capture has to be a scheduled, diffable, machine-checked artefact, because the class of fault it catches produces no alert and no error, only a slower node that looks identical to a fast one. This is the motivation for the `lstopo --output-format xml` bring-up capture in the Practice section.

- **NVIDIA/nccl issue #246 — two official NVIDIA tools disagreeing about the same box** ([github.com/NVIDIA/nccl/issues/246](https://github.com/NVIDIA/nccl/issues/246)). On an 8× V100 node with Mellanox RDMA NICs, NCCL's own topology detection logged `PIX`/`PXB` for GPU–NIC pairs where `nvidia-smi topo -m` reported `PHB`. The substance: the two tools classify the *same* hierarchy through different code paths — NCCL walks sysfs and builds its own graph, `nvidia-smi` uses the driver's classification — and they can round the same physical layout to different labels. What it shows: never treat one tool's label as the topology. Walk the `lspci -tv` hierarchy or the sysfs device path when the labels matter, which is exactly the reconciliation skill lesson 08 grades.

- **NVIDIA MIG + NUMA-node localisation** ([developer.nvidia.com](https://developer.nvidia.com/blog/accelerating-data-processing-with-nvidia-multi-instance-gpu-and-numa-node-localization)). NVIDIA documents that on multi-die GPUs the *same* local-vs-remote argument this lesson makes about the host applies one level down, inside the GPU package: MIG instances localised to the GPU's internal memory partitions outperform non-localised ones on bandwidth-bound kernels, with no change to the compute. What it shows: "the tree" is fractal. The reasoning you are learning about sockets and root complexes reappears at the die level inside an accelerator, and will reappear again at rack level in lesson 04's NVL72 discussion.

- **The `_PXM`/SRAT mechanism as documented by ACPI and Linux** ([uefi.org](https://uefi.org/specifications), [docs.kernel.org](https://docs.kernel.org/driver-api/cxl/platform/acpi/srat.html)). The substance: ACPI §17 defines proximity domains, the SRAT affinity structures (processor, memory, and — newer — *generic initiator*, which is how a device that is itself a memory initiator, like a CXL accelerator, gets its own domain), and SLIT's normalised-to-10 distance convention. HMAT extends this with real bandwidth/latency attributes per initiator-target pair. What it shows: everything Linux reports about NUMA is a transcription of these tables, which is why "the firmware is wrong" is a real, non-exotic root cause.

## Worked example

**Scenario.** A rented two-socket Sapphire Rapids box, 8× H100 SXM, 4 InfiniBand NICs. Jobs report full GPU utilisation but training throughput is ~15% under a sibling node in the same cluster. Nothing is logged. Trace the tree.

**Step 1 — shape of the CPU side.**

```
$ lscpu | grep -E 'Model name|Socket\(s\)|Core\(s\) per socket|NUMA'
Model name:                      Intel(R) Xeon(R) Platinum 8480C
Core(s) per socket:              56
Socket(s):                       2
NUMA node(s):                    4
NUMA node0 CPU(s):               0-27,112-139
NUMA node1 CPU(s):               28-55,140-167
NUMA node2 CPU(s):               56-83,168-195
NUMA node3 CPU(s):               84-111,196-223
```

Four NUMA nodes over two sockets → **SNC2 is enabled**. First hypothesis recorded: the sibling node may have SNC off, in which case every `--membind=0` in the job's launch script means something different on the two machines. Note it; do not act on it yet.

**Step 2 — memory side and the hop structure.**

```
$ numactl --hardware | tail -8
node distances:
node   0   1   2   3
  0:  10  11  21  21
  1:  11  10  21  21
  2:  21  21  10  11
  3:  21  21  11  10
```

Confirms SNC2 and gives the grouping: **{0,1} = socket 0, {2,3} = socket 1.** The `11` tier is the cheap intra-socket hop; `21` is the UPI crossing.

**Step 3 — where does each GPU live, and where is its NIC?**

```
$ nvidia-smi topo -m | awk 'NR==1 || /^GPU/ {print $1, $(NF-1), $NF}'
        CPU Affinity  NUMA Affinity
GPU0    0-27,112-139        0
GPU1    0-27,112-139        0
GPU2    28-55,140-167       1
GPU3    28-55,140-167       1
GPU4    56-83,168-195       2
GPU5    56-83,168-195       2
GPU6    84-111,196-223      3
GPU7    84-111,196-223      3

$ for n in 0 1 2 3; do
>   printf 'NIC%s -> node %s\n' "$n" "$(cat /sys/class/net/ibp$((24+n*8))s0/device/numa_node)"
> done
NIC0 -> node 0
NIC1 -> node 0
NIC2 -> node 0        ← expected node 2
NIC3 -> node 2
```

**There it is.** NIC2 is on socket 0. It should be on socket 1 alongside GPU4/GPU5. Cross-check against the matrix:

```
$ nvidia-smi topo -m | grep -E '^GPU4|^GPU5'
GPU4  ... NIC0  NIC1  NIC2  NIC3
GPU4      SYS   SYS   SYS   PIX      56-83,168-195   2
GPU5      SYS   SYS   SYS   NODE     56-83,168-195   2
```

GPU4's only `PIX` partner is NIC3, and GPU5 has *no* `PIX` partner at all — its intended rail NIC (NIC2) reads `SYS`. On the reference layout every GPU should have exactly one `PIX` or `PXB` NIC.

**Step 4 — confirm from the hierarchy, not just the label.**

```
$ readlink -f /sys/class/net/ibp40s0/device      # NIC2
/sys/devices/pci0000:00/0000:00:03.1/0000:28:00.0/0000:29:10.0/0000:2a:00.0

$ readlink -f /sys/bus/pci/devices/0000:9a:00.0  # GPU5
/sys/devices/pci0000:80/0000:80:01.1/0000:98:00.0/0000:99:04.0/0000:9a:00.0
```

The two paths share **no** prefix beyond the machine root: NIC2 hangs off host bridge `pci0000:00`, GPU5 off `pci0000:80`. Their common ancestor is the machine. The `SYS` label is correct and the cause is physical placement, not a firmware mislabel. Bus numbers agree too — `2a` is below `0x80`, `9a` is above.

**Step 5 — quantify before you escalate.**

```
Design intent for GPU5:  GPUDirect RDMA over NIC2, ceiling = min(400 Gb/s NIC, Gen5 x16)
                       = min(50 GB/s, 63 GB/s) = 50 GB/s

Actual for GPU5:         cross-socket → GPUDirect RDMA not used → host-staged path
                         bounded by one UPI 2.0 link ≈ 38 GB/s effective,
                         shared with NIC0/NIC1 traffic and all coherence traffic.
                         Under concurrent load, measured share ≈ 20–25 GB/s.

Loss on GPU5's inbound path ≈ 50 → ~22 GB/s  ≈ 56% of design bandwidth.
One GPU of eight is on a half-speed feed. For a synchronous collective,
the slowest rank sets the step time, so a 1/8 defect is a whole-job defect.
```

That last line is the sentence that turns a topology observation into an argument: **in a synchronous data-parallel job, one misplaced rank taxes all of them**, because every all-reduce waits for the straggler. A 15% end-to-end regression from one badly placed NIC out of four is entirely consistent with the arithmetic.

**Step 6 — the fix, and the fix you can actually apply.** The correct fix is physical: move NIC2 to a socket-1 slot. Until that happens, the mitigations are (a) pin GPU5's process to node 2 and let NCCL route GPU5's inter-node traffic through a rail-local peer over NVLink — this is what NCCL's **PXN** ("PCI × NVLink") path does, hopping GPU5 → NVLink → GPU4 → NIC3 → wire rather than crossing UPI; and (b) exclude GPU5 from the job. Record the finding, the arithmetic, and the measurement in the teardown — that trio is the deliverable, not the diagram alone.

## Practice

Produce the **host topology sketch** for the Topology Teardown deliverable. Work on a multi-socket box, or a rented multi-GPU instance from a neocloud. If you have neither, a two-socket CPU-only server still exercises steps 1–4 and 6.

1. **Capture the raw evidence, all of it, into files you keep.**
   ```
   lscpu                                   > topo/lscpu.txt
   numactl --hardware                      > topo/numactl.txt
   lspci -Dnnvv                            > topo/lspci-vv.txt     # needs root
   lspci -tv                               > topo/lspci-tree.txt
   lstopo --output-format console --no-caches --whole-io > topo/lstopo.txt
   lstopo --output-format xml              > topo/lstopo.xml       # the diffable artefact
   nvidia-smi topo -m                      > topo/nvsmi-topo.txt   # if GPUs present
   for d in /sys/bus/pci/devices/*; do
     printf '%s %s\n' "${d##*/}" "$(cat $d/numa_node)"
   done                                    > topo/pci-numa.txt
   ```
2. **Guess before you look.** From `lspci -D` alone, write down which socket you think each GPU and NIC is on, using the bus-number heuristic from §8. *Then* check `topo/pci-numa.txt` and the `nvidia-smi topo -m` NUMA Affinity column. Record how many you got right — the point is to calibrate how much you can trust the heuristic on this platform.
3. **Sketch by hand, then digitise.** Sockets as boxes; NUMA nodes inside them with memory size labelled on each; host bridges under the nodes; switches under the host bridges; every GPU, NIC and NVMe drawn *under its correct branch*. Draw the inter-socket link between the sockets and label it with the link type and your computed per-direction bandwidth from §7's arithmetic.
4. **Overlay the NVLink plane in a second colour or a separate panel.** It must be visibly a different network, not another edge in the PCIe tree.
5. **Annotate every mismatch.** Any device whose `numa_node` is `-1`; any GPU whose closest NIC is `SYS`; any GPU with no `PIX`/`PXB` NIC at all; any place where the bus-number heuristic disagreed with `_PXM`.
6. **Verify one path from the hierarchy, not the label.** Pick one GPU–NIC pair, `readlink -f` both sysfs device paths, and show that the length of their shared prefix matches the code `nvidia-smi topo -m` gave them.
7. **Compute one cost.** For your worst-classified pair, write the arithmetic from §12: which links are on the path, each one's ceiling, and therefore the ceiling of the path.

**Acceptance:** a saved diagram plus the raw command output, in which (a) every accelerator, NIC and NVMe is placed under its true NUMA node, (b) the node count matches `numactl --hardware`, (c) the NVLink plane is drawn as a separate network, (d) at least one GPU–NIC pair's classification is independently confirmed from the sysfs device paths, and (e) one path cost is derived with units carried. You can reproduce the sketch from memory without re-running the tools.

## Common pitfalls

1. **Assuming `numactl --hardware`'s node count equals the PCIe host-bridge count.** SNC/NPS partition the memory fabric; they do not necessarily create matching PCIe domains. A two-socket SNC2 box shows four NUMA nodes and two host bridges, and both numbers are correct. *Symptom:* you conclude the machine is misconfigured because `lspci -tv` shows "only two roots." *Mechanism:* memory-side and I/O-side groupings are declared by different ACPI structures and need not be 1:1.

2. **Treating `numa_node = -1` as "single-socket, so it's fine."** *Symptom:* a device with no affinity on a two-socket box, dismissed as benign. *Mechanism:* `-1` means firmware emitted no `_PXM`, so the kernel treats the device as equidistant from every node. The physical cost of reaching it is unchanged; only the OS's ability to avoid that cost is gone. Cross-check against `lscpu`'s socket count before assuming it does not matter.

3. **Believing a clean NVLink block implies a clean host tree.** *Symptom:* "all `NV18`, topology is fine." *Mechanism:* the GPU×GPU block describes a completely different fabric from the one that feeds host data in. A perfect NVSwitch mesh and a NIC on the wrong socket coexist happily and the matrix shows both, in different columns.

4. **Forgetting ACS can defeat a topologically short path.** *Symptom:* `PIX` in the matrix, but `p2pBandwidthLatencyTest` reports numbers consistent with routing through the root complex, or `nvidia-smi topo -p2p rw` reports `CNS`. *Mechanism:* ACS redirection on the switch's downstream ports forces P2P transactions upstream so the IOMMU can inspect them. The hierarchy is unchanged; the routing is not. Disabling ACS fixes the bandwidth *and* removes VFIO isolation — know which node you are on.

5. **Reading SLIT distances as measurements.** *Symptom:* quoting "remote is 2.1× local" from `numactl`. *Mechanism:* SLIT is a static firmware table normalised so local = 10. It is an assertion, not a sample, it ignores load entirely, and it is wrong on some platforms. Measured idle ratios run 1.5–1.8×; measured *loaded* ratios are worse than either number. Use MLC or STREAM.

6. **Trusting `CUDA_VISIBLE_DEVICES` index order to match BDF order.** *Symptom:* per-rank NUMA binding logic that is correct on one node and wrong on another. *Mechanism:* CUDA's default `CUDA_DEVICE_ORDER=FASTEST_FIRST` sorts by capability, not bus address. Set `CUDA_DEVICE_ORDER=PCI_BUS_ID` before you derive anything from a device index.

7. **Comparing `lstopo`'s `L#` indices to `numactl` node numbers.** *Symptom:* binding to "node 1" from an `lstopo` reading and landing on the wrong node. *Mechanism:* hwloc uses its own dense logical numbering (`L#`) and prints the OS index separately as `P#`. Always run `lstopo -p`, or read the `P#` value, before feeding a number to `numactl`.

## Self-check

- **Why is a PCIe device "local" to exactly one socket / NUMA node?**
  **Answer:** Because the PCIe root complex is integrated into the CPU die and a slot's lanes are physically wired to one package's SerDes — and, under SNC/NPS, attributed to one sub-domain of that package. Nothing in firmware or the OS can re-home them; only re-cabling can. The kernel learns the mapping at boot by evaluating each device's ACPI `_PXM` method (backed by the SRAT's proximity-domain definitions) and publishes it as `/sys/bus/pci/devices/<bdf>/numa_node`, with `local_cpulist` giving the matching CPU set. Reaching any other node's DRAM from that device requires a hop over the inter-socket link, so exactly one node is the no-hop home.

- **What crosses the inter-socket link, and what does it cost — with the arithmetic?**
  **Answer:** Four traffic classes: remote DRAM reads/writes, cache-coherence traffic (snoops and responses, proportional to *sharing* rather than to your data movement), PCIe DMA targeting remote DRAM, and cross-socket peer-to-peer PCIe (often unsupported rather than merely slow). Cost, Sapphire Rapids generation: UPI 2.0 at 16 GT/s × 24 lanes ÷ 8 = **48 GB/s raw per direction per link**; applying the 0.80 payload efficiency implied by Intel's own UPI 1.0 figures (20.8 GB/s quoted against 26 GB/s raw) gives **≈38 GB/s effective**. Against a socket's 8 × DDR5-4800 = **307.2 GB/s** of local DRAM bandwidth, that is roughly **8:1** for a single link, or **2.7:1** with three links populated. Latency: ~75–95 ns local vs ~130–150 ns remote, a 1.5–1.8× ratio, worse under load because of queueing.

- **What is the mechanistic difference between AMD NPS and Intel SNC, and what does each produce?**
  **Answer:** AMD NPS partitions along the package's *physical* quadrant structure — Genoa's I/O die has four quadrants, each owning three of the twelve DDR5 channels and a share of PCIe lanes, so NPS4 exposes four nodes per socket that map onto real silicon boundaries. NPS0 (2P only) goes the other way and interleaves across both sockets, presenting one node for the whole system. Intel SNC is a *logical* partition of a single on-die mesh: the mesh, its LLC slices and its memory controllers are carved into 2 or 4 address-range domains with no chiplet boundary involved. Both require symmetric DIMM population, both produce more NUMA nodes than sockets, and both are read the same way from `numactl --hardware`'s three-tier distance matrix (`10` local / `11` same-socket other domain / `21` cross-socket).

- **You see `numa_node = -1` for a GPU on a dual-socket box. What does that tell you, and what does it not?**
  **Answer:** It tells you the platform emitted no `_PXM` for that device, so the kernel set `NUMA_NO_NODE` and treats it as equidistant from every node — buffers land wherever first-touch puts them, IRQ affinity has no preferred set, and `local_cpulist` will read as the full CPU list. It does *not* tell you the box is single-socket, and it does not tell you placement is irrelevant: the physical hop cost is unchanged, only the OS's ability to avoid it is gone. Confirm the socket count with `lscpu`, then derive the true home from the bus-number range and from which host bridge the sysfs device path descends from.

- **Given `0000:e3:00.0` versus `0000:17:00.0` on a two-socket box, how would you guess the socket, and how do you confirm it?**
  **Answer:** Firmware assigns bus numbers depth-first from ranges declared per host bridge in ACPI `_CRS`, and two host bridges in one domain cannot share bus numbers, so platforms typically split 0–255 between them — commonly `00`–`7f` for socket 0 and `80`–`ff` for socket 1. `17` is below `0x80` (socket 0), `e3` is above (socket 1). Confirm with `/sys/bus/pci/devices/<bdf>/numa_node`, or definitively with `readlink -f /sys/bus/pci/devices/<bdf>` and reading which `pci0000:XX` host bridge the path starts from. Treat the heuristic as a first guess only: it breaks on four-socket boxes, on platforms that use separate PCI domains per socket, and after `pci=assign-busses`.

- **Two GPUs show `PIX` in `nvidia-smi topo -m` but a P2P bandwidth test reports root-complex-class numbers. What is happening?**
  **Answer:** PCIe Access Control Services is enabled on the switch's downstream ports with `ReqRedir`/`CmpltRedir` set in `ACSCtl`, which forces peer-to-peer transactions upstream to the root complex so the IOMMU can translate and permission-check them. The *hierarchy* is still one switch hop — which is all `PIX` describes — but the *routing* now goes switch → root port → root complex → root port → switch. Confirm with `lspci -vvv | grep -A2 'Access Control Services'` and `nvidia-smi topo -p2p rw` (look for `CNS`). Disabling ACS restores the short path and simultaneously removes the isolation guarantee VFIO passthrough depends on, and collapses IOMMU groups — acceptable on a single-tenant bare-metal node, not on a multi-tenant one.

- **What is the common-ancestor idea, and how does it map onto the `topo -m` codes?**
  **Answer:** Every device is a leaf in a tree; a transfer between two leaves must be routed by their lowest common ancestor. The higher that ancestor sits, the more shared, contended and slow the path. `PIX` = the ancestor is a single PCIe bridge (one switch). `PXB` = the ancestor is further up but still below the host bridge. `PHB` = the ancestor is the root complex itself. `NODE` = the ancestor is the fabric between two host bridges inside one NUMA node. `SYS` = the ancestor is the machine root, reached across the inter-socket link. `NV#` sits outside this ordering entirely, because NVLink is a second fabric whose graph does not contain any of these nodes.

## Connections & what's next

This tree is the map every other lesson in the module places numbers onto. **Lesson 02** takes "the GPU's home node" and turns it into a measured host↔device bandwidth delta, using the same NUMA-affinity read you just learned, and adds the memory-side detail (channels, DDR5 sub-channels, HBM tiers) that the §7 arithmetic assumed. **Lesson 03** zooms into one edge of this tree — the root-complex-to-endpoint link — and teaches you to read whether it actually trained to the speed and width §12's ceilings assume. **Lesson 04** gives you the reference shape of the whole tree for an 8-GPU HGX/DGX node, including the NVLink plane drawn properly. **Lesson 05**'s Kubernetes Topology Manager exists to *enforce*, at pod-admission time, exactly the alignment you are learning to read by hand here. **Lesson 06** reuses the `PIX`/`PXB`/`SYS` reasoning for NVMe and GPUDirect Storage. And **lesson 08**'s capstone is this lesson's four-tool reconciliation, done once on a real, unfamiliar node, under time pressure.

## References & further reading

**Primary sources**

- **ACPI Specification, §17 — NUMA Architecture Platforms** — [uefi.org/specifications](https://uefi.org/specifications) — the canonical definition of proximity domains, the SRAT affinity structures (processor, memory, generic initiator), the SLIT's normalise-local-to-10 convention, and the `_PXM` object. Read §17 if you ever need to argue that a platform's tables are wrong.
- **Linux kernel — SRAT / ACPI platform documentation** — [docs.kernel.org/driver-api/cxl/platform/acpi/srat.html](https://docs.kernel.org/driver-api/cxl/platform/acpi/srat.html) — how Linux consumes SRAT/SLIT/HMAT, including the generic-initiator affinity structures that matter for CXL and for devices that are themselves memory initiators.
- **hwloc / `lstopo`** — [open-mpi.org/projects/hwloc](https://www.open-mpi.org/projects/hwloc/) and [github.com/open-mpi/hwloc](https://github.com/open-mpi/hwloc) — the object model this lesson's vocabulary borrows (Machine / Package / Die / NUMANode / Core / PU / HostBridge / PCIBridge / PCI / OS device) and the `--output-format`, `--whole-io`, `-p`, `--filter` flags used in Practice.
- **`lspci(8)` / pciutils** — [github.com/pciutils/pciutils](https://github.com/pciutils/pciutils) — the flags that matter here: `-tv` (tree), `-D` (show domains), `-PP` (full path), `-nn` (numeric + name), `-vvv` (capability blocks, needs root), `-s` (select).
- **`numactl(8)` / `numastat(8)`** — [github.com/numactl/numactl](https://github.com/numactl/numactl) — `--hardware`'s node/distance output and the mempolicy flags lesson 02 uses to force placement.
- **NVIDIA `nvidia-smi` manual, `topo` subcommand** — bundled with the driver (`man nvidia-smi`) — the authoritative legend for `X`/`NV#`/`PIX`/`PXB`/`PHB`/`NODE`/`SYS`, plus `topo -p` and `topo -p2p rw`. The legend text quoted in §10 is what the tool prints.

**Real-world engineering**

- **NVIDIA/nccl issue #246** — [github.com/NVIDIA/nccl/issues/246](https://github.com/NVIDIA/nccl/issues/246) — NCCL's topology detection and `nvidia-smi topo -m` reporting different codes (`PIX`/`PXB` vs `PHB`) for the same GPU–NIC pairs on an 8× V100 node. The concrete case for reconciling tools rather than trusting one.
- **Meta Engineering — fleet hardware reliability** — [engineering.fb.com](https://engineering.fb.com/2020/12/09/data-center-engineering/how-facebook-keeps-its-large-scale-infrastructure-hardware-up-and-running/) — the scale argument for automated, diffable topology capture at bring-up rather than manual inspection on demand.
- **NVIDIA Developer Blog — MIG and NUMA node localisation** — [developer.nvidia.com](https://developer.nvidia.com/blog/accelerating-data-processing-with-nvidia-multi-instance-gpu-and-numa-node-localization) — the same local-vs-remote argument replayed inside the GPU package, showing the reasoning is fractal across levels.
- **Frank Denneman — "Understanding Multi-GPU Topologies Within a Single Host"** — [frankdenneman.ai](https://frankdenneman.ai/2026-03-27-Understanding-Multi-GPU-Topologies-Within-a-Single-Host/) — a placement-first treatment of this tree with real `nvidia-smi topo -m` reads; the module's spine resource, and good calibration for what a complete topology write-up looks like.
- **NVIDIA GPUDirect RDMA documentation** — [docs.nvidia.com/cuda/gpudirect-rdma/](https://docs.nvidia.com/cuda/gpudirect-rdma/) — states the same-root-complex requirement and the ACS caveat directly; the primary source behind §11.

**Deeper dives**

- **Ulrich Drepper, "What Every Programmer Should Know About Memory," §5 (NUMA)** — [akkadia.org/drepper](https://www.akkadia.org/drepper/cpumemory.pdf) — a five-minute skim only. The concepts are timeless; the 2007 latency and bandwidth figures are stale by an order of magnitude, so trust your own `numactl` and MLC output over the paper's numbers.
- **Intel Memory Latency Checker (MLC)** — [intel.com](https://www.intel.com/content/www/us/en/download/736633/intel-memory-latency-checker-intel-mlc.html) — `--latency_matrix` and `--bandwidth_matrix` produce the measured node×node numbers that let you check the SLIT rather than believe it. Used in lesson 02's practice.
