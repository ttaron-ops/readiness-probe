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
sources: 7
---

# 02b.1 · The host as a topology tree

> **Concept.** A GPU host is a tree — sockets → NUMA domains → PCIe root complexes → endpoints — and every GPU, NIC, and NVMe hangs off exactly one branch, fixed at board-design and boot time.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Where this fits

The Linux-internals module (01b) gave you NUMA as a *kernel* mechanic: first-touch allocation, `numa_balancing`, page migration, the cost model behind `numactl --hardware`'s distance numbers. That was all about how the kernel behaves once memory exists. This lesson starts a different module with a different question: before any kernel policy runs, **where in the physical tree does a device already live, and who decided that?** The answer is fixed in silicon and firmware, not runtime-adjustable — and it is the one placement fact every later lesson in this module (memory bandwidth, PCIe link health, 8-GPU server layout, Kubernetes topology alignment) builds on top of. Get this tree wrong in your head, and every downstream number you measure will look like a mystery instead of an obvious consequence.

## Why this matters

The single most common silent-waste pattern on a GPU box is a job whose host memory, CPU threads, and network path all sit on the *wrong* branch of the tree relative to the GPU it feeds — the GPU shows 100% utilization while every byte it needs crosses the inter-socket link first. No dashboard flags it; `nvidia-smi` says busy. This is the literal job description at neocloud and hyperscaler infra teams: CoreWeave's hardware-engineering postings call out "troubleshoot complex GPU and PCIe failures," and Meta's own infra-engineering writing (cited below) exists because misplacement and marginal links are common enough at fleet scale to justify dedicated automated tooling. In an interview, "why is a PCIe device local to exactly one socket, and what does crossing the inter-socket link cost?" separates people who have run these boxes from people who have only read about them. You already know kernel NUMA; this lesson makes you able to draw the host from memory so placement decisions become reflexive, not something you discover after a training run underperforms.

## What's new here (calibration)

You already hold the on-prem physical picture (sockets, DIMMs, expansion slots) and, from the Linux-internals module, the *kernel mechanics* of NUMA: `numa_balancing`, first-touch allocation, page migration, node-local vs remote memory. Nothing below re-teaches those.

The delta is **framing the host as a static placement tree that accelerators attach to**, not as a memory allocator's runtime behavior:
- A **PCIe device has a home NUMA node** the same way a page does — but it's fixed in silicon at boot, not decided by first-touch. The GPU's "first touch" already happened when the board was seated in a slot and the BIOS enumerated it.
- The tree has **levels that matter for accelerators** that on-prem intuition usually stops short of: socket → NUMA domain → PCIe root complex → root port → endpoint (GPU/NIC/NVMe). On-prem you cared about levels 1 and maybe 2; GPU placement lives at levels 3–4.
- **NUMA-node count is not socket count, and the mechanism differs by vendor.** AMD NPS (Nodes Per Socket) and Intel SNC (Sub-NUMA Clustering) both split one socket into multiple NUMA domains, but *how* they do it is physically different — a genuinely new, staff-level distinction most on-prem backgrounds never needed.
- **The affinity is declared by firmware, not measured at runtime.** The ACPI tables the kernel trusts (`_PXM`, SLIT) are static BIOS output; they can be wrong, and nothing checks them against reality.

## Core concepts

### The tree, top to bottom
```
Host
├── Socket 0  (a physical CPU package)
│   ├── NUMA node 0  ── memory controllers → local DRAM channels
│   │   └── PCIe root complex(es) → root port(s) → GPU0, NIC0, NVMe0
│   └── NUMA node 1  (present only if SNC/NPS splits the socket)
│       └── PCIe root complex(es) → GPU1, NIC1
└── Socket 1
    ├── NUMA node 2 → GPU2, NIC2 ...
    └── NUMA node 3 → GPU3 ...
        └────── inter-socket link (UPI / Infinity Fabric) joins the halves
```
Every leaf (GPU, NIC, NVMe) reaches CPU and DRAM by climbing to its root complex, into its NUMA node's memory controllers for *local* DRAM, or across the inter-socket link for *remote* DRAM. "Local to a socket" means: reachable without crossing that link.

**Terminology precision, because staff interviewers probe exactly this:** the **Root Complex** is the whole subsystem that connects the CPU/memory to the PCIe fabric — one exists per socket (or per NUMA quadrant under SNC/NPS). A **Root Port** is one of the Root Complex's PCIe-facing interfaces; you see one per top-level bus in `lspci -tv`. The **Host Bridge** is the logical device Linux enumerates for each root-complex domain. Don't conflate these three — "the root complex" is not a single wire, it's a subsystem with multiple ports hanging off it.

### Why a PCIe device is local to exactly one node
The PCIe root complex is **physically integrated into the CPU die** (has been since Sandy Bridge / the first EPYC). The lanes a slot uses are wired to controllers inside one specific socket — and, under SNC/NPS, to one specific quadrant of that die. A GPU in that slot is electrically closer to that socket's memory controllers; reaching any other socket's DRAM means a hop over UPI/Infinity Fabric. There is no configuration that makes a seated card "belong" to two nodes — the wiring is fixed.

The mechanism, precisely: the platform BIOS publishes a **Proximity Domain** object (ACPI `_PXM` method) per device in the DSDT/SSDT tables. At boot, the kernel's PCI and `acpi_numa` code reads `_PXM` for each device and populates `/sys/bus/pci/devices/<bdf>/numa_node`:
```
$ cat /sys/bus/pci/devices/0000:17:00.0/numa_node
0
```
A value of `-1` means the platform/BIOS did not publish affinity for that device — the kernel then treats it as equidistant, and the allocator/scheduler can place its buffers anywhere, which is itself a latent-waste bug, not a "single-socket, so it's fine" signal (a multi-socket box with a buggy or incomplete `_PXM` entry produces the same `-1`).

The node-*to*-node distance numbers you read from `numactl --hardware` come from a separate, related ACPI table: the **SLIT (System Locality Information Table)**. This is worth being precise about because it's a common staff-level trap: **SLIT is a static, firmware-declared cost model, not something measured at runtime.** The `21` you read for a cross-socket hop is BIOS's stated opinion about relative cost, not a live latency measurement. On rare but real broken firmware (early server BIOS revisions, some VM BIOS implementations) the SLIT is simply wrong, and every NUMA-aware allocator in the kernel — including `numa_balancing` — silently trusts it anyway.

**A second, related but separate wrinkle:** Linux tracks CPU topology and memory topology through separate sysfs trees for the *same* NUMA node — `/sys/devices/system/node/nodeN/cpulist` for which logical CPUs belong to node N, and `/sys/devices/system/node/nodeN/meminfo` for that node's memory. `lscpu`'s `NUMA node(N) CPU(s):` line and `numactl --hardware`'s `size`/`free` describe the same domain through these two different trees — worth knowing so that if one tool ever seems to disagree with another, you know where each number actually comes from before assuming a bug.

### What crosses the inter-socket link, and what it costs
The link (Intel **UPI** — Ultra Path Interconnect; AMD **Infinity Fabric** / xGMI between sockets) carries: remote DRAM reads/writes, cache-coherence traffic, and any PCIe→remote-DRAM DMA (a GPU on socket 0 doing DMA into a buffer that lives in socket 1's DRAM). Rough current numbers:
- **Local DRAM latency** ~80–110 ns; **remote** ~130–200 ns. Budget roughly +50–100% latency per hop.
- **Bandwidth**: a Sapphire/Emerald Rapids UPI 2.0 link runs 16–20 GT/s; a socket has 2–4 links, giving low-hundreds of GB/s aggregate — but a *single* stream to remote DRAM is throttled by the link long before it would saturate local DRAM. Practically, remote-bound bandwidth often lands at **50–70% of local**, and worse when the link is also carrying coherence traffic.

The key mental model: the link is a shared, finite bridge. Local DRAM bandwidth scales with channels populated on that node; remote bandwidth is capped by the bridge no matter how many channels the far socket has. (Lesson 02 turns this into a measured, quotable number.)

### AMD NPS vs Intel SNC — same symptom, different mechanism
Both features split one socket into multiple NUMA domains, and both are worth knowing apart at staff depth because they are **physically different**, not just differently named:
- **AMD NPS (Nodes Per Socket)** splits by **memory-controller-die placement on the physical package**. EPYC is a chiplet architecture — several Core Complex Dies (CCDs) surrounding a central I/O die. NPS4 exposes each of four CCD-adjacent I/O-die quadrants as its own NUMA node — it follows the actual physical chiplet layout.
- **Intel SNC (Sub-NUMA Clustering)** is a **logical partition of a monolithic or tiled die's mesh interconnect** into sub-domains. It's not a physical chiplet split — it's the mesh fabric being carved into regions in firmware/microcode.
Functionally, both hand you "more NUMA nodes than sockets, each owning a slice of memory controllers and PCIe lanes" — but one reflects physical package boundaries and the other reflects a logical fabric partition. This is a genuine differentiator in a staff-level conversation, and it also explains why the *number* of extra nodes you get can differ even between vendors offering similar core counts.

### PCIe domain numbers — reading the socket straight off the BDF
Every PCIe device has a Bus:Device.Function (BDF) address, prefixed with a **PCI domain**: `0000:e3:00.0`. On single-root-complex boxes the domain is always `0000` and only the bus number varies. On boxes with **multiple independent root complexes** (i.e., most dual-socket servers), Linux commonly assigns each root complex's devices a distinct bus range — for example bus `00-1f` for socket 0's root complex and bus `e0-ff` for socket 1's — purely because BDF address space can't be shared across two independent root complexes without collision. In practice this means you can often *guess* which socket a device sits on just by looking at the top nibble of its bus number in `lspci -D`, before you ever cross-check against `nvidia-smi topo -m`'s NUMA Affinity column. This is exactly the skill the module's capstone (lesson 08) drills — rehearse it here first.

### Reading the tree with real tools
- **`lscpu`** → the CPU side of the tree: sockets, cores-per-socket, threads, and the `NUMA node(N) CPU(s):` lines mapping logical CPUs to nodes. This tells you SNC/NPS is on when you see more NUMA nodes than sockets.
- **`numactl --hardware`** → the memory side: node count, `size`/`free` per node, and the **distance matrix** (SLIT). `10` = local, `21` = one hop remote; asymmetric or multi-tier values (`11/12/21/22`) reveal SNC/NPS or multi-hop fabrics.
- **`lstopo` / `hwloc-ls`** (hwloc package) → renders the whole tree including which PCIe bus, and thus which GPU/NIC, hangs off each node. `lstopo --output-format txt` or `lstopo topo.png`. This is the single best "draw it for me" command.
- **`nvidia-smi topo -m`** → the accelerator view: a matrix of GPU↔GPU and GPU↔NIC links (`NV#`/`PIX`/`PXB`/`PHB`/`NODE`/`SYS`) plus a **NUMA Affinity** and **CPU Affinity** column per GPU. `SYS` between two GPUs means traffic crosses the inter-socket link; `NODE` means same NUMA node via PCIe host bridge; `NV#` means NVLink, bypassing PCIe entirely.
- **`ls -l /sys/class/net/<nic>/device/numa_node`** and the PCIe `numa_node` file above → confirm NIC/NVMe home node.
- **`lspci -D`** → read the BDF domain/bus directly, the fast pre-check described above.

### Decoding `nvidia-smi topo -m`
The matrix legend is the vocabulary of the accelerator tree — memorize it:
- **`NV#`** — NVLink, *N* bricks (H100 = up to 18 links via NVSwitch, ~900 GB/s aggregate). Bypasses PCIe and the host tree entirely for GPU↔GPU.
- **`PIX`** — single PCIe switch hop; **`PXB`** — multiple PCIe switches (bridges); **`PHB`** — via the PCIe Host Bridge (the root complex), same NUMA node.
- **`NODE`** — within one NUMA node but across PCIe host bridges; **`SYS`** — across the inter-socket link. **`SYS` between two GPUs, or between a GPU and its NIC, is the red flag** — every P2P or DMA byte pays the socket crossing.
A well-built 8-GPU box shows `NV#` between all GPUs and `PIX`/`PXB` (never `SYS`) from each GPU to its paired NIC. `SYS` in the GPU-to-NIC column is the classic "GPU fed from the wrong socket" defect.

### NVLink and the host tree are separate planes
NVLink/NVSwitch gives GPUs a fast mesh *among themselves*, independent of the PCIe tree — but the host memory path (dataloaders, pinned staging buffers, CPU-side preprocessing) still climbs the PCIe/NUMA tree. A box can have perfect all-NVLink GPU↔GPU topology and still be crippled by every GPU's host buffer sitting on the wrong socket. Never let a clean `nvidia-smi topo -m` GPU-to-GPU block lull you: the host-feed path is a different question, answered by the NUMA-affinity column plus where the buffers actually land.

### The read that matters
Cross-reference three things and you have the placement map: (1) `nvidia-smi topo -m` NUMA-affinity column → each GPU's home node; (2) `numactl --hardware` → memory available on that node; (3) NIC `numa_node` → whether the GPU's feeding NIC shares its node. When a GPU's home node, its NIC, and the CPUs pinned to its job all match, the tree is aligned. Any mismatch is an inter-socket-link crossing you're paying for on every transfer.

### IOMMU / ACS can break the tree you think you have
Even when two GPUs sit on the same switch (`PIX`), PCIe **Access Control Services (ACS)** can force their peer-to-peer traffic *up to the root complex and back down* — or block P2P entirely — so a topologically-local pair behaves like a remote one. This is common on servers with ACS enabled for IOMMU isolation (required for VFIO passthrough/security). Symptom: `nvidia-smi topo -m` shows `PIX`/`PXB` but P2P bandwidth tests read PCIe-through-root numbers. Check with `lspci -vvv | grep -i acsctl` and validate P2P with `p2pBandwidthLatencyTest` (cuda-samples). Note the tradeoff explicitly: disabling ACS system-wide to unlock the short path weakens VFIO/IOMMU device isolation — a real security tradeoff, not a free performance win. The lesson: the *drawn* tree is necessary but not sufficient — ACS and IOMMU grouping decide whether the short path is actually usable.

### The four commands, and what each answers
| Command | Answers |
|---------|---------|
| `lscpu` | Sockets, cores, threads, CPU→NUMA-node map (reveals SNC/NPS) |
| `numactl --hardware` | Node count, memory-per-node, distance matrix (SLIT) |
| `lstopo` / `hwloc-ls` | The full tree — which GPU/NIC/NVMe hangs off each node |
| `nvidia-smi topo -m` | GPU↔GPU / GPU↔NIC link type + per-GPU NUMA & CPU affinity |

## Perspectives

**Developer.** You never see the tree from application code — CUDA and PyTorch just present "GPU 0..N." The tree only becomes visible when you `numactl`-wrap a process or set `CUDA_VISIBLE_DEVICES`/`taskset`. The developer's failure mode is trusting the logical GPU index as if it implied anything about physical placement; index 0 is whatever the driver enumerated first, not necessarily the GPU closest to your dataloader thread.

**Operator/SRE.** The tree is an acceptance-test object, not a one-time curiosity. Every rented or leased node should have its tree captured and diffed against the vendor reference at bring-up, because BIOS defaults — SNC/NPS on or off, ACS enabled, PCIe bifurcation — vary silently across firmware revisions from the same vendor, even on nominally identical SKUs.

**Hardware/kernel.** The tree is fixed at board-design and BIOS-boot time, not at OS runtime — this is the one placement decision in the whole module the kernel cannot change or migrate around later (unlike page migration, which can move memory after the fact). Once the BIOS has published `_PXM` and enumerated the bus, that device's home node is set until reboot.

**Economics.** A `numa_node=-1` device or an unnoticed SNC-on box doesn't cost anything by itself — it costs money only when combined with careless buffer placement. That's why this lesson is a *precondition* for lessons 02 and 05's dollar arguments, not a dollar argument on its own: you can't quantify a placement mistake you haven't first learned to see.

## Real-world use cases

- **Frank Denneman — "Understanding Multi-GPU Topologies Within a Single Host"** ([frankdenneman.ai](https://frankdenneman.ai/2026-03-27-Understanding-Multi-GPU-Topologies-Within-a-Single-Host/)) — the module's spine resource. Walks PCIe, NVLink domains, and 4-GPU vs 8-GPU NVLink splits from a placement-first lens; shows what a correctly-reasoned topology writeup looks like end to end.
- **NVIDIA Developer Blog — "Accelerating Data Processing with NVIDIA Multi-Instance GPU and NUMA Node Localization"** ([developer.nvidia.com](https://developer.nvidia.com/blog/accelerating-data-processing-with-nvidia-multi-instance-gpu-and-numa-node-localization)) — shows that modern NVIDIA GPUs have internal NUMA-like behavior between GPU dies, and that NUMA-node-localized MIG configuration yields large speedups on a memory-bandwidth-bound kernel. What it shows: the exact local-vs-remote argument this lesson makes about the *host*, replayed one level down — inside the GPU package itself.
- **dev.to (Daya Shankar) — "How PCIe, NVLink, and NUMA Topology Affect GPU Scheduling Outcomes"** ([dev.to](https://dev.to/daya-shankar/how-pcie-nvlink-and-numa-topology-affect-gpu-scheduling-outcomes-l52)) — a practitioner walkthrough connecting GPU/NIC placement and NUMA config to how frameworks build communication paths. What it shows: a second, more implementation-flavored angle on the same tree.
- **Meta Engineering — "How Facebook keeps its large-scale infrastructure hardware up and running"** ([engineering.fb.com](https://engineering.fb.com/2020/12/09/data-center-engineering/how-facebook-keeps-its-large-scale-infrastructure-hardware-up-and-running/)) — a fleet-reliability piece. What it shows: the scale motivation for treating topology capture as an automated, repeatable acceptance-test step rather than a manual one-off (a theme that returns in lesson 03's PCIe fault-detection story).

## Worked example

**Scenario.** A 2-socket Sapphire Rapids box, 8× H100 SXM. Jobs report full GPU utilization but training throughput is ~15% under a sibling cluster. Trace the tree.

1. **Shape of the tree.**
   ```
   $ lscpu | grep -E 'Socket|NUMA'
   Socket(s):            2
   NUMA node(s):         4
   NUMA node0 CPU(s):    0-15,64-79
   NUMA node1 CPU(s):    16-31,80-95
   NUMA node2 CPU(s):    32-47,96-111
   NUMA node3 CPU(s):    48-63,112-127
   ```
   4 NUMA nodes on 2 sockets → **SNC=2 (Sub-NUMA Clustering on)**. Each socket is two NUMA domains. This is the trap: people assume node==socket and bind to node 0 for a GPU that actually homes on node 3.

2. **Memory per node.**
   ```
   $ numactl --hardware
   available: 4 nodes (0-3)
   node 0 size: 257000 MB   node 1 size: 257000 MB
   node 2 size: 257000 MB   node 3 size: 257000 MB
   node distances:
       0   1   2   3
   0: 10  11  21  21
   1: 11  10  21  21
   2: 21  21  10  11
   3: 21  21  11  10
   ```
   Reads cleanly: `10` local, `11` same-socket-other-SNC-domain (cheap), `21` across-socket (the UPI hop). Nodes 0/1 = socket 0; nodes 2/3 = socket 1.

3. **Where does GPU4 live?**
   ```
   $ nvidia-smi topo -m
        GPU4  ... CPU Affinity  NUMA Affinity
   GPU4       ... 48-63,112-127        3
   $ cat /sys/class/net/eth2/device/numa_node
   0
   ```
   GPU4 homes on **node 3 (socket 1)**. Its data-loading NIC `eth2` homes on **node 0 (socket 0)**. Every batch the NIC receives lands in socket-0 DRAM, then crosses UPI to reach GPU4's root complex on socket 1. The GPU is "busy" — waiting on staged data that took the slow path.

4. **Cross-check via BDF domain before trusting the tool.**
   ```
   $ lspci -D | grep -i nvidia
   0000:17:00.0 GPU0 ...
   0000:31:00.0 GPU1 ...
   0000:e0:00.0 GPU4 ...
   0000:e3:00.0 GPU5 ...
   ```
   GPU0/1 sit in the low bus range (socket 0's root complex); GPU4/5 sit in the high `e0-ff` range (socket 1's). This matches the NUMA Affinity column above *before* you even ran `nvidia-smi` — the same read you'll use as a fast sanity check in the lesson 08 capstone.

5. **Confirm the buffer's node.** The training process was started without pinning; `numastat -p <pid>` shows most of its pages on node 0 (first-touch happened on the loader thread, which the scheduler parked on socket 0). Fix: pin the loader and its pinned host buffers to node 3 (`numactl --cpunodebind=3 --membind=3`) and move the NIC feeding GPU4 to a socket-1 NIC. The 15% gap was pure inter-socket-link tax — invisible to every utilization dashboard.

## Practice

On a multi-socket box (or a rented multi-GPU instance — e.g. an 8×GPU node on a neocloud), produce the **host topology sketch** for the Topology Teardown deliverable.
1. Run `numactl --hardware` and `lscpu`; capture output.
2. If a GPU is present, run `nvidia-smi topo -m` and `cat /sys/bus/pci/devices/<bdf>/numa_node` for each accelerator and NIC. Also run `lspci -D | grep -i nvidia` and derive each GPU's socket from the bus-number prefix *before* checking the NUMA Affinity column — then verify you guessed right.
3. **Sketch by hand** (then digitize): sockets as boxes, NUMA nodes inside them, memory-per-node labeled on each, and an arrow from each PCIe root complex to the GPU/NIC/NVMe it owns — with each leaf drawn *under its correct NUMA node*.
4. Mark the inter-socket link and label any GPU whose feeding NIC is on a different node.

**Acceptance:** a saved diagram + the raw command output, in which every accelerator and NIC is placed under its true NUMA node, and node count matches `numactl --hardware`. You can reproduce the sketch from memory without re-running the tools.

## Common pitfalls

1. **Assuming `numactl --hardware` node count equals PCIe root complex count.** SNC/NPS can produce more NUMA nodes than there are physical root complexes if a die's mesh is logically partitioned without a matching physical PCIe split. Always confirm PCIe grouping via `lstopo`/`lspci`, not just the NUMA node count.
2. **Treating `numa_node = -1` as "single-socket, so it's fine."** It equally appears on multi-socket boxes with buggy or incomplete `_PXM` entries. Always cross-check against `lscpu` socket count before assuming it's benign.
3. **Believing NVLink topology cleanliness implies host-tree cleanliness.** A perfect all-NVLink GPU↔GPU matrix says nothing about where host buffers, NICs, or CPU threads sit.
4. **Forgetting IOMMU/ACS can defeat a topologically-short path.** A `PIX`-labeled pair can still route P2P traffic up to the root complex and back if ACS forces it — and disabling ACS system-wide trades away VFIO/IOMMU isolation, so it's not a free fix.
5. **Assuming SLIT distances are measured, not declared.** Engineers sometimes treat `numactl --hardware`'s `21` as an empirically-measured latency ratio; it's actually a static firmware table that can be wrong or merely approximate.

## Self-check

- Why is a PCIe device "local" to exactly one socket/NUMA node? **Answer:** Because the PCIe root complex is integrated into the CPU die and the slot's lanes are physically wired to one socket's (and, under SNC/NPS, one quadrant's) memory controllers. The kernel reads this fixed affinity from ACPI `_PXM` at boot and publishes it as the device's `numa_node`. Reaching any other node's DRAM requires a hop over the inter-socket link, so exactly one node is the no-hop home.
- What crosses the inter-socket link (UPI / Infinity Fabric) and what does it cost? **Answer:** Remote DRAM reads/writes, cache-coherence traffic, and PCIe DMA targeting remote DRAM. Cost: roughly +50–100% latency per hop (local ~80–110 ns vs remote ~130–200 ns) and a single-stream bandwidth ceiling set by the shared link — typically landing remote bandwidth at ~50–70% of local, degrading further under coherence contention.
- What's the mechanistic difference between how AMD NPS and Intel SNC create extra NUMA nodes? **Answer:** AMD NPS follows physical chiplet boundaries — it exposes each CCD-adjacent I/O-die quadrant on the package as its own node. Intel SNC is a logical partition of a monolithic/tiled die's mesh interconnect into sub-domains — no physical chiplet split exists. Same symptom (more nodes than sockets), physically different mechanism.
- You see `numa_node = -1` for a GPU on a dual-socket box — what does that tell you, and what does it *not* tell you? **Answer:** It tells you the platform/BIOS didn't publish `_PXM` affinity for that device, so the kernel treats it as equidistant and buffers can land anywhere. It does *not* tell you the box is single-socket or that placement doesn't matter — it's a latent bug waiting for a careless allocation, not a benign default.
- Given only a device's BDF domain/bus prefix (e.g. `0000:e3:00.0` vs `0000:17:00.0`), how would you guess which socket it's on, and what would you use to confirm it? **Answer:** On dual-root-complex boxes, Linux commonly assigns each socket's root complex a distinct bus range (e.g. `00-1f` for socket 0, `e0-ff` for socket 1), so a high vs low bus-number prefix is a fast first guess. Confirm with `nvidia-smi topo -m`'s NUMA Affinity column or the device's `/sys/bus/pci/devices/<bdf>/numa_node` file.

## Connections & what's next

This tree is the map every other lesson in the module places numbers onto. Lesson 02 takes the "home node" concept from this lesson and turns it into a measured bandwidth delta — you'll use the exact same `nvidia-smi topo -m` NUMA-affinity read to set up that experiment. Lesson 03 (PCIe) zooms into one edge of this tree — the root-complex-to-endpoint link itself — and teaches you to read its trained speed and width. Lesson 05's Kubernetes Topology Manager exists to *enforce*, at pod-admission time, exactly the alignment you're learning to read by hand here. And lesson 08's capstone is this lesson's four-tool reconciliation exercise, done once on a real, unfamiliar node, under time pressure.

## References & further reading

**Primary sources**
- ACPI Specification — SLIT (System Locality Information Table) and the `_PXM` method — [uefi.org/specifications](https://uefi.org/specifications) — canonical mechanism reference for the firmware tables this lesson's affinity reads ultimately trace back to; not independently re-fetched this session, cited as the canonical spec home.
- hwloc / `lstopo` project — [open-mpi.org/projects/hwloc](https://www.open-mpi.org/projects/hwloc/) — read for: the tool that renders the whole tree for you; install it and diff its picture against your hand sketch.

**Real-world engineering blogs**
- Frank Denneman, "Understanding Multi-GPU Topologies Within a Single Host" — [frankdenneman.ai](https://frankdenneman.ai/2026-03-27-Understanding-Multi-GPU-Topologies-Within-a-Single-Host/) — what it shows: the canonical placement-first treatment of this exact tree, with real `nvidia-smi topo -m` reads.
- NVIDIA Developer Blog, "Accelerating Data Processing with NVIDIA Multi-Instance GPU and NUMA Node Localization" — [developer.nvidia.com](https://developer.nvidia.com/blog/accelerating-data-processing-with-nvidia-multi-instance-gpu-and-numa-node-localization) — what it shows: the same local-vs-remote argument, one level down, inside the GPU package.
- Meta Engineering, "How Facebook keeps its large-scale infrastructure hardware up and running" — [engineering.fb.com](https://engineering.fb.com/2020/12/09/data-center-engineering/how-facebook-keeps-its-large-scale-infrastructure-hardware-up-and-running/) — what it shows: the fleet-scale motivation for automated topology capture at node bring-up.
- dev.to (Daya Shankar), "How PCIe, NVLink, and NUMA Topology Affect GPU Scheduling Outcomes" — [dev.to](https://dev.to/daya-shankar/how-pcie-nvlink-and-numa-topology-affect-gpu-scheduling-outcomes-l52) — what it shows: an implementation-flavored practitioner angle on the same tree.

**Deeper dives**
- Drepper, "What Every Programmer Should Know About Memory," §5 (NUMA) — [akkadia.org/drepper](https://www.akkadia.org/drepper/cpumemory.pdf) — skim only, 5 minutes; the concepts are timeless but the 2007 latency/bandwidth numbers are stale — trust your own `numactl` output over the paper's figures.
