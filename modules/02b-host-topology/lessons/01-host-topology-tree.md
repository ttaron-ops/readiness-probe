---
lesson: "02b.1"
title: "The host as a topology tree"
module: "02b"
concept: "The host as a topology tree"
status: not-started
est_time: "2.5h"
artifacts: []
---

# 02b.1 · The host as a topology tree

> **Concept.** A GPU host is a tree — sockets → NUMA domains → PCIe root complexes — and every GPU, NIC, and NVMe hangs off exactly one branch.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Why this matters
The single most common silent-waste pattern on a GPU box is a job whose host memory, CPU threads, and network path all sit on the *wrong* branch of the tree relative to the GPU it feeds — the GPU shows 100% utilization while every byte it needs crosses the inter-socket link first. No dashboard flags it; `nvidia-smi` says busy. In an interview, "how does a PCIe device end up local to one NUMA node, and how do you prove it?" separates people who have run these boxes from people who have read about them. You already know kernel NUMA; this lesson makes you able to draw the host from memory so placement decisions are reflexive.

## What's new for you
You already hold the on-prem physical picture (sockets, DIMMs, expansion slots) and, from the Linux-internals module, the *kernel mechanics* of NUMA: `numa_balancing`, first-touch allocation, page migration, node-local vs remote memory. Nothing below re-teaches those.

The delta is **framing the host as a static placement tree that accelerators attach to**, not as a memory allocator's runtime behavior:
- A **PCIe device has a home NUMA node** the same way a page does — but it's fixed in silicon at boot, not decided by first-touch. The GPU's "first touch" already happened when the board was seated in a slot.
- The tree has **three levels that matter for accelerators**: socket → NUMA domain → PCIe root complex → endpoint (GPU/NIC/NVMe). On-prem you cared about levels 1 and maybe 2. GPU placement lives at levels 3–4.
- **NUMA-node count is not socket count.** AMD NPS (Nodes Per Socket) and Intel SNC (Sub-NUMA Clustering) split one socket into 2 or 4 NUMA domains, each owning a subset of memory controllers *and* a subset of PCIe lanes. This is the piece on-prem intuition gets wrong.

## Core notes

### The tree, top to bottom
```
Host
├── Socket 0  (a physical CPU package)
│   ├── NUMA node 0  ── memory controllers → local DRAM channels
│   │   └── PCIe root complex(es) → GPU0, NIC0, NVMe0
│   └── NUMA node 1  (present only if SNC/NPS splits the socket)
│       └── PCIe root complex(es) → GPU1, NIC1
└── Socket 1
    ├── NUMA node 2 → GPU2, NIC2 ...
    └── NUMA node 3 → GPU3 ...
        └────── inter-socket link (UPI / Infinity Fabric) joins the halves
```
Every leaf (GPU, NIC, NVMe) reaches CPU and DRAM by climbing to its root complex, into its NUMA node's memory controllers for *local* DRAM, or across the inter-socket link for *remote* DRAM. "Local to a socket" means: reachable without crossing that link.

### Why a PCIe device is local to exactly one node
The PCIe root complex is **physically integrated into the CPU die** (has been since Sandy Bridge / the first EPYC). The lanes a slot uses are wired to controllers inside one specific socket — and, under SNC/NPS, to one specific quadrant of that die. A GPU in that slot is electrically closer to that socket's memory controllers; reaching any other socket's DRAM means a hop over UPI/Infinity Fabric. There is no configuration that makes a seated card "belong" to two nodes — the wiring is fixed. The kernel reads this affinity from ACPI (`_PXM` method) at boot and exposes it:
```
$ cat /sys/bus/pci/devices/0000:17:00.0/numa_node
0
```
A value of `-1` means the platform/BIOS did not publish affinity (common on some single-socket boards or misconfigured firmware) — the kernel then treats the device as equidistant, and the scheduler/allocator can place its buffers anywhere, which is itself a latent-waste bug.

### What crosses the inter-socket link, and what it costs
The link (Intel **UPI** — Ultra Path Interconnect; AMD **Infinity Fabric** / xGMI between sockets) carries: remote DRAM reads/writes, cache-coherence traffic, and any PCIe→remote-DRAM DMA (a GPU on socket 0 doing DMA into a buffer that lives in socket 1's DRAM). Rough current numbers:
- **Local DRAM latency** ~80–110 ns; **remote** ~130–200 ns. Budget roughly +50–100% latency per hop.
- **Bandwidth**: a Sapphire/Emerald Rapids UPI 2.0 link runs 16–20 GT/s; a socket has 2–4 links, giving low-hundreds of GB/s aggregate — but a *single* stream to remote DRAM is throttled by the link long before it would saturate local DRAM. Practically, remote-bound bandwidth often lands at **50–70% of local**, and worse when the link is also carrying coherence traffic.
The key mental model: the link is a shared, finite bridge. Local DRAM bandwidth scales with channels populated on that node; remote bandwidth is capped by the bridge no matter how many channels the far socket has.

### Reading the tree with real tools
- **`lscpu`** → the CPU side of the tree: sockets, cores-per-socket, threads, and the `NUMA node(N) CPU(s):` lines mapping logical CPUs to nodes. This tells you SNC/NPS is on when you see more NUMA nodes than sockets.
- **`numactl --hardware`** → the memory side: node count, `size`/`free` per node (memory-per-node), and the **distance matrix** (SLIT). `10` = local, `21` = one hop remote; asymmetric or multi-tier values (`11/12/21/22`) reveal SNC/NPS or multi-hop fabrics.
- **`lstopo` / `hwloc-ls`** (hwloc package) → renders the whole tree including which PCIe bus and thus which GPU/NIC hangs off each node. `lstopo --output-format txt` or `lstopo topo.png`. This is the single best "draw it for me" command.
- **`nvidia-smi topo -m`** → the accelerator view: a matrix of GPU↔GPU and GPU↔NIC links (NV#/PIX/PXB/PHB/NODE/SYS) plus a **NUMA Affinity** and **CPU Affinity** column per GPU. `SYS` between two GPUs = traffic crosses the inter-socket link; `NODE` = same NUMA node via PCIe host bridge; `NV#` = NVLink, bypassing PCIe entirely.
- **`ls -l /sys/class/net/<nic>/device/numa_node`** and the PCIe `numa_node` file above → confirm NIC/NVMe home node.

### Decoding `nvidia-smi topo -m`
The matrix legend is the vocabulary of the accelerator tree — memorize it:
- **`NV#`** — NVLink, *N* bricks (H100 = up to 18 links via NVSwitch, ~900 GB/s aggregate). Bypasses PCIe and the host tree entirely for GPU↔GPU.
- **`PIX`** — single PCIe switch hop; **`PXB`** — multiple PCIe switches (bridges); **`PHB`** — via the PCIe Host Bridge (the root complex), same NUMA node.
- **`NODE`** — within one NUMA node but across PCIe host bridges; **`SYS`** — across the inter-socket link (the SMP/UPI path). **`SYS` between two GPUs, or between a GPU and its NIC, is the red flag** — every P2P or DMA byte pays the socket crossing.
A well-built 8-GPU box shows `NV#` between all GPUs and `PIX`/`PXB` (never `SYS`) from each GPU to its paired NIC. `SYS` in the GPU-to-NIC column is the classic "GPU fed from the wrong socket" defect.

### NVLink and the host tree are separate planes
NVLink/NVSwitch gives GPUs a fast mesh *among themselves*, independent of the PCIe tree — but the host memory path (dataloaders, pinned staging buffers, CPU-side preprocessing) still climbs the PCIe/NUMA tree. A box can have perfect all-NVLink GPU↔GPU topology and still be crippled by every GPU's host buffer sitting on the wrong socket. Never let a clean `nvidia-smi topo -m` GPU-to-GPU block lull you: the host-feed path is a different question, answered by the NUMA-affinity column plus where the buffers actually land.

### SNC / NPS is a deliberate BIOS tradeoff
Sub-NUMA Clustering (Intel) and Nodes-Per-Socket (AMD `NPS1/2/4`) split a socket into more NUMA domains, each owning a slice of memory controllers *and* PCIe lanes. Benefit: lower local latency and tighter GPU-to-memory affinity (a GPU homes on a quarter-socket, not a half). Cost: smaller per-node memory pools and more ways to mis-bind. For GPU hosts the common choice is NPS4/SNC-on for latency-sensitive HPC, or NPS1/SNC-off for simpler placement and larger contiguous pools. Either way you must *read the actual node count at runtime* — never assume node==socket, and never hardcode node numbers across a fleet whose BIOS settings may differ.

### The read that matters
Cross-reference three things and you have the placement map: (1) `nvidia-smi topo -m` NUMA-affinity column → each GPU's home node; (2) `numactl --hardware` → memory available on that node; (3) NIC `numa_node` → whether the GPU's feeding NIC shares its node. When a GPU's home node, its NIC, and the CPUs pinned to its job all match, the tree is aligned. Any mismatch is a UPI crossing you're paying for on every transfer.

### IOMMU / ACS can break the tree you think you have
Even when two GPUs sit on the same switch (`PIX`), PCIe **Access Control Services (ACS)** can force their peer-to-peer traffic *up to the root complex and back down* — or block P2P entirely — so a topologically-local pair behaves like a remote one. This is common on servers with ACS enabled for IOMMU isolation (required for VFIO passthrough/security). Symptom: `nvidia-smi topo -m` shows `PIX`/`PXB` but P2P bandwidth tests read PCIe-through-root numbers. Check with `lspci -vvv | grep -i acsctl` and validate P2P with `p2pBandwidthLatencyTest` (cuda-samples). The lesson: the *drawn* tree is necessary but not sufficient — ACS and IOMMU grouping decide whether the short path is actually usable.

### The four commands, and what each answers
| Command | Answers |
|---------|---------|
| `lscpu` | Sockets, cores, threads, CPU→NUMA-node map (reveals SNC/NPS) |
| `numactl --hardware` | Node count, memory-per-node, distance matrix |
| `lstopo` / `hwloc-ls` | The full tree — which GPU/NIC/NVMe hangs off each node |
| `nvidia-smi topo -m` | GPU↔GPU / GPU↔NIC link type + per-GPU NUMA & CPU affinity |

### Common misreads to avoid
- **node == socket.** Wrong whenever SNC/NPS is on. Always confirm from `numactl --hardware`.
- **`numa_node = -1` is harmless.** It means the platform published no affinity; the allocator then scatters buffers freely — a latent placement bug, not a non-issue.
- **A clean GPU-GPU NVLink matrix means the box is aligned.** It says nothing about the host-feed path.
- **`lstopo` on a VM tells the truth.** Under virtualization the guest may see a flattened or fabricated topology; trust it only on bare metal or with verified passthrough/topology hints.

## Worked example
**Scenario.** A 2-socket Sapphire Rapids box, 8× H100 SXM, jobs report full GPU utilization but training throughput is ~15% under a sibling cluster. Trace the tree.

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

4. **Confirm the buffer's node.** The training process was started without pinning; `numastat -p <pid>` shows most of its pages on node 0 (first-touch happened on the loader thread, which the scheduler parked on socket 0). Fix: pin the loader and its pinned host buffers to node 3 (`numactl --cpunodebind=3 --membind=3`) and move the NIC feeding GPU4 to a socket-1 NIC. The 15% gap was pure UPI tax — invisible to every utilization dashboard.

## Practice
On a multi-socket box (or a rented multi-GPU instance — e.g. an 8×GPU node on a neocloud), produce the **host topology sketch** for the Topology Teardown deliverable.
1. Run `numactl --hardware` and `lscpu`; capture output.
2. If a GPU is present, run `nvidia-smi topo -m` and `cat /sys/bus/pci/devices/<bdf>/numa_node` for each accelerator and NIC.
3. **Sketch by hand** (then digitize): sockets as boxes, NUMA nodes inside them, memory-per-node labeled on each, and an arrow from each PCIe root complex to the GPU/NIC/NVMe it owns — with each leaf drawn *under its correct NUMA node*.
4. Mark the inter-socket link and label any GPU whose feeding NIC is on a different node.

**Acceptance:** a saved diagram + the raw command output, in which every accelerator and NIC is placed under its true NUMA node, and node count matches `numactl --hardware`. You can reproduce the sketch from memory without re-running the tools.

## Self-check
**(a) Why is a PCIe device "local" to exactly one socket/NUMA node?**
**Answer:** Because the PCIe root complex is integrated into the CPU die and the slot's lanes are physically wired to one socket's (and, under SNC/NPS, one quadrant's) memory controllers. The kernel reads this fixed affinity from ACPI `_PXM` at boot and publishes it as the device's `numa_node`. Reaching any other node's DRAM requires a hop over the inter-socket link, so exactly one node is the no-hop home.

**(b) What crosses the inter-socket link (UPI / Infinity Fabric) and what does it cost?**
**Answer:** Remote DRAM reads/writes, cache-coherence traffic, and PCIe DMA targeting remote DRAM. Cost: roughly +50–100% latency per hop (local ~80–110 ns vs remote ~130–200 ns) and a single-stream bandwidth ceiling set by the shared link — typically landing remote bandwidth at ~50–70% of local, degrading further under coherence contention.

**(c) How many NUMA nodes does your box have and how do you know from `numactl --hardware`?**
**Answer:** Read the `available: N nodes` line and the distance matrix. The count of nodes (and whether it exceeds the socket count from `lscpu`) tells you if SNC/NPS is on; the matrix (`10` local, `11` same-socket other domain, `21` cross-socket) confirms which nodes share a socket.

## Resources
1. **Frank Denneman — "Understanding Multi-GPU Topologies Within a Single Host"** — https://frankdenneman.ai/2026-03-27-Understanding-Multi-GPU-Topologies-Within-a-Single-Host/ — *deep.* The canonical placement-first treatment of exactly this tree, with `nvidia-smi topo -m` reads and NVLink-vs-PCIe-vs-SYS paths. Read in full; it is the backbone of this lesson.
2. **`lstopo` / hwloc** — https://www.open-mpi.org/projects/hwloc/ — *skim/hands-on.* The tool that renders the tree for you; install it and diff its picture against your hand sketch. Best way to catch a leaf you placed under the wrong node.
3. **Drepper — "What Every Programmer Should Know About Memory," §5 (NUMA)** — https://www.akkadia.org/drepper/cpumemory.pdf — *skim, 5 min only.* A refresher on the mechanics you already have; the concepts are timeless but the 2007 latency/bandwidth numbers are stale — trust your box's `numactl` output over the paper's figures.
