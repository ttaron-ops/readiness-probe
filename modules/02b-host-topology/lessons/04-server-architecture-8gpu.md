---
lesson: "02b.4"
title: "The 8-GPU server: HGX/DGX H100 topology"
module: "02b"
concept: "The 8-GPU server: HGX/DGX H100 topology"
status: not-started
est_time: "7.5h"
prev: "03-pcie.md"
next: "05-topology-alignment-k8s.md"
artifacts: []
sources: 6
---
# 02b.4 · The 8-GPU server: HGX/DGX H100 topology

> **Concept.** The canonical 8x H100 node — drawn from memory — and how to map a rented instance's real GPUs onto it.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Where this fits

Lesson 03 gave you the single-link skill: read `LnkCap` vs `LnkSta`, walk a PCIe tree, tell a root port from a switch from an endpoint. That skill answers "is this one link healthy?" It does not answer "should this GPU's traffic even be on PCIe at all, and if it is, is it going to the *right* place?" This lesson supplies the reference topology that makes those questions answerable — the fixed, designed layout of a standard 8-GPU HGX/DGX node, including the second fabric (NVLink/NVSwitch) that doesn't exist on any machine you've run before this course. Everything you diagnose from here through lesson 08's capstone gets compared against the picture built in this lesson.

## Why this matters

Every diagnosis in this module bottoms out in one question: *is this GPU where it should be in the machine, and is it talking to the right neighbor over the right link?* You cannot answer that from `nvidia-smi topo -m` output alone — you need the **reference topology in your head** to compare against. When a rented "8x H100" box shows GPU5's NIC reachable only via `SYS` (crossing the socket interconnect) instead of `PXB` (a local switch hop), you only know that's wrong because you know GPU5 and its rail NIC are *supposed* to be co-located on socket 1. Without the reference, the output is just letters.

The stakes are concrete and measured, not hypothetical. CoreWeave's published H100 benchmark work reports **49.2% Model FLOPs Utilization (MFU) on a 128-H100 run — 18 points higher than a comparison baseline** — and MFU is precisely the metric a systemic topology defect *would* move, because it's an aggregate that (unlike per-GPU utilization%) can't be fooled by a stalled GPU that still shows busy SMs. A mis-mapped node runs distributed training with NIC traffic crossing UPI, or with a collective landing on the wrong NVSwitch path, and loses 20-40% of network throughput on every all-reduce — for the entire duration of a multi-week run, invisible to any per-GPU dashboard. At neocloud scale you accept fleets you didn't build; the reference-vs-real mapping is your acceptance test. Being able to sketch the HGX H100 node from memory — sockets, NVSwitches, retimers, rail-aligned NICs — is table-stakes senior platform fluency in this space, and it's a live whiteboard question in the interviews you're targeting.

## What's new here (calibration)

You know NUMA, PCIe link training (lesson 03), and how a single accelerator hangs off the host. What's new is the **whole node as a designed system** and the second interconnect fabric that doesn't exist on the machines you've run before:

- **NVLink and NVSwitch** as a fabric *parallel to* PCIe. GPU-to-GPU traffic does **not** ride PCIe on this class of node — it rides NVLink through dedicated NVSwitch silicon at ~7x PCIe bandwidth. This breaks an assumption from your on-prem background: GPU-GPU is **not** NUMA-bound here, even though GPU-CPU and GPU-storage still are.
- **The GPU-to-socket split** as a fixed design fact (GPU0-3 → socket 0, GPU4-7 → socket 1), not something the OS decides.
- **Rail alignment** — the 1:1 GPU:NIC pairing that makes multi-node training fast, why the NIC's *physical* placement relative to its GPU is the thing that matters, and why production nodes commonly run two entirely separate NIC fabrics rather than one.
- **Retimers instead of a PCIe switch on the baseboard** — a deliberate design choice you can now explain from lesson 03's signal-integrity discussion, and the reason MFU rather than per-GPU utilization is the metric that would catch a fleet-wide version of this mistake.

## Core concepts

### The canonical HGX H100 8-GPU node

Commit this diagram to memory. It is the same board (`HGX H100 SXM5 8-GPU`, NVIDIA part 935-24287-...) inside a DGX H100 and inside every OEM/neocloud "8x H100 SXM" server:

```
            Socket 0 (CPU0)                         Socket 1 (CPU1)
          Xeon Sapphire Rapids  <====== UPI ======>  Xeon Sapphire Rapids
             |   |   |   |                               |   |   |   |
         PCIe Gen5 x16 (via retimers)              PCIe Gen5 x16 (via retimers)
             |   |   |   |                               |   |   |   |
          GPU0 GPU1 GPU2 GPU3                        GPU4 GPU5 GPU6 GPU7
            \   \   |   /  \______                 __/  \   |   /   /
             \___\__|__/___ all-to-all NVLink ____/______\__|__/___/
                    ||||        (900 GB/s/GPU)        ||||
                 [ NVSwitch0  NVSwitch1  NVSwitch2  NVSwitch3 ]   <- 4x NVSwitch
                        (non-blocking all-to-all fabric)

   Rail-aligned NICs: GPU0..7 each pair 1:1 with a ConnectX-7 (400 Gb/s NDR)
```

The load-bearing facts:

- **Two sockets, GPUs split 4/4.** GPU0-3 attach (via PCIe) to CPU **socket 0**; GPU4-7 to **socket 1**. This split is the whole reason NUMA matters on these boxes: a process feeding GPU5 should run on socket 1's cores and allocate from socket 1's memory, or every host↔GPU copy crosses UPI. This is the NVIDIA reference design — some OEM boards vary the layout, so always confirm with `nvidia-smi topo -m`'s NUMA Affinity column rather than assuming 4/4 from memory alone.
- **4x NVSwitch, all-to-all NVLink.** The eight GPUs are fully connected through **four third-gen NVSwitch** chips on the baseboard. Every GPU reaches every other GPU at the full **900 GB/s** aggregate NVLink bandwidth (H100: **18 NVLink-4 links × 50 GB/s bidirectional** = 900 GB/s), non-blocking — no pair contends with another pair. This is ~7x the ~63+63 GB/s a PCIe Gen5 x16 link offers. For scale, the generation-over-generation arithmetic is a common whiteboard follow-up: **A100 = 12 NVLink-3 links × 50 GB/s = 600 GB/s**; **H100 = 18 NVLink-4 links × 50 GB/s = 900 GB/s**; **B200/GB200 = NVLink 5, ~1.8 TB/s per GPU**.
- **8x PCIe Gen5 x16 host stubs, via retimers, no baseboard PCIe switch.** Each GPU exposes one independent Gen5 x16 link up to its CPU socket. Because the baseboard-to-CPU-tray trace is too long for a clean 32 GT/s signal, each stub goes through **PCIe Gen5 retimers** (lesson 03). Crucially there is **no PCIe switch merging the GPUs** on the baseboard — the GPUs do not share a PCIe path, and they don't need one, because GPU-GPU traffic uses NVLink instead. A switch here would only add shared-bandwidth contention for a path that carries none of the GPU-GPU traffic.
- **1:1 rail-aligned ConnectX-7 NICs, plus a separate storage fabric.** Eight **ConnectX-7** (or BlueField-3) 400 Gb/s NDR InfiniBand/Ethernet ports, one per GPU, each placed so the GPU and its NIC sit under the same PCIe complex — enabling **GPUDirect RDMA** GPU→NIC→wire without crossing the CPU or UPI. Production nodes typically add a *second*, separate NIC fabric purely for storage and management traffic, specifically so a training job's collective (all-reduce) traffic never contends on the wire with data-loading or checkpoint I/O — CoreWeave calls this out explicitly as a named dual-fabric design pattern in their own benchmark writeup (see Real-world use cases below).

### NVLink/NVSwitch vs PCIe — what rides where

This is the mental model that prevents the invisible-loss failure:

| Traffic | Path | NUMA-bound? |
|---------|------|-------------|
| GPU ↔ GPU (same node) | NVLink → NVSwitch → NVLink | **No** — bypasses CPU/PCIe entirely |
| GPU ↔ host memory | PCIe Gen5 x16 → CPU root complex | **Yes** — bound to the GPU's socket |
| GPU ↔ NIC (GPUDirect RDMA) | PCIe, GPU→NIC under shared complex | **Yes** — must stay rail-local |
| GPU ↔ NVMe (GPUDirect Storage) | PCIe, GPU→drive | **Yes** — bound by PCIe placement |

So: **GPU-GPU all-reduce inside one node is not affected by NUMA** — it's on the NVSwitch fabric. But the moment data leaves the GPU for host RAM, a NIC, or a disk, PCIe and NUMA locality govern it. A team that "fixed" a throughput problem by pinning GPU-GPU collectives to NVLink but left the NIC on the wrong socket fixed the half that was already fine.

One more layer of precision worth having: NVSwitch's "all-to-all, non-blocking" guarantee is a **per-domain** property, not an unconditional one. Within one HGX baseboard the domain is 8 GPUs. At rack scale (see below), the domain grows to include multiple chassis connected through *external* NVSwitch trays — the guarantee still holds, but only within whatever the current domain boundary is. Knowing the domain size for the hardware generation in front of you is part of the fluency.

### Rail alignment

In a multi-node GPU cluster, the network is organized into **rails**: NIC *k* on every node connects to leaf switch *k* (rail *k*). "Rail-aligned" means GPU *k*'s traffic egresses on NIC *k*, which lands on rail *k*'s switch — so a collective (e.g. NCCL ring/tree all-reduce) between GPU5s across many nodes stays on a single rail and never has to hop between leaf switches. Two placement requirements make this work, and both must hold — one without the other still loses the throughput:

1. **Within the node:** GPU *k* and NIC *k* must be co-located under a shared PCIe complex so GPUDirect RDMA goes GPU→NIC directly (a `PIX`/`PXB` hop), not across the root complex or UPI (`PHB`/`SYS`). GPU5 is a socket-1 GPU, so **GPU5's NIC must be a socket-1 NIC** on GPU5's PCIe complex — attaching it to a socket-0 slot forces its RDMA across UPI.
2. **Across the fabric:** NIC *k* cables to rail *k*'s leaf switch, consistently on every node in the fleet. A perfectly-aligned single node still loses if the cabling to the leaf switches isn't consistent across the fleet — within-node alignment is necessary but not sufficient.

Break either and inter-node collective bandwidth collapses even though every GPU looks 100% busy — the exact invisible-loss pattern this module targets. NCCL is topology-aware and auto-detects the NVLink/PCIe/IB paths at initialization — but "auto-detects" means it trusts what the drivers and system tools report. As lesson 03's real-world example showed (NCCL issue #246), NCCL's own topology log and `nvidia-smi topo -m` can genuinely disagree; when debugging a slow all-reduce, check `NCCL_DEBUG=INFO`'s topology output against `topo -m` rather than trusting either alone.

### Where it's going: Grace-Blackwell GB200

Briefly, so you can speak to the direction. The **GB200** pairs Blackwell GPUs with **Grace** CPUs over **NVLink-C2C** (a coherent CPU↔GPU link, ~900 GB/s, replacing the PCIe stub for the CPU-GPU path), and scales NVLink out of the box: **NVLink 5** (~1800 GB/s/GPU) and the **GB200 NVL72** rack connects **72 GPUs** into one NVLink domain via external NVSwitch trays, so "all-to-all NVLink" now spans a whole rack, not just eight GPUs in a chassis. This is architecturally distinct from simply scaling up the 8-GPU chassis — know the term and the shape of the change even if depth isn't required yet. The operational lens is the same — trained links, socket/rail placement, GPU-GPU-on-NVLink vs everything-else-on-PCIe — but the NVLink domain gets much bigger and the CPU joins it coherently.

## Perspectives

**Developer.** NCCL is topology-aware and auto-detects NVLink/PCIe/IB paths at initialization, which mostly hides this whole lesson from application code. But "auto-detects" is not "always correct" — a developer debugging a mysteriously slow all-reduce should know to check NCCL's own topology log (`NCCL_DEBUG=INFO`) against `nvidia-smi topo -m`, the same cross-tool reconciliation habit lesson 03 and lesson 08 both build.

**Operator.** The reference topology in this lesson is the acceptance-test baseline for any rented or leased 8-GPU box. The operator's job is proving a specific SKU matches the reference — socket split, NVLink domain, rail alignment — not re-deriving the reference from scratch on every new node.

**Hardware.** Retimers, not a baseboard PCIe switch, exist purely because of Gen5 signal-integrity physics over the baseboard-to-CPU-tray trace length — a direct callback to lesson 03's retimer discussion, now applied at the whole-node scale. The absence of a GPU-side PCIe switch is itself a hardware design statement: the designers moved GPU-GPU traffic entirely off PCIe rather than trying to make PCIe carry it.

**Economics / scale.** Rail alignment is what makes *multi-node* training economical. A single node's perfect internal topology is necessary but not sufficient — the network's rail design (matching NIC *k* to leaf switch *k* across every node in the fleet) is what lets a training job scale past one chassis without the per-node topology correctness going to waste. CoreWeave's own MFU numbers are the aggregate proof point: topology correctness is what a percentage-point MFU swing is actually measuring.

## Real-world use cases

- **CoreWeave — "NVIDIA H100 GPU Benchmark Results: What We Learned from Large-Scale GPU Testing."** Reports **49.2% MFU on 128 H100 GPUs, 18 points higher than a comparison baseline**, describes CoreWeave's dual-fabric architecture (NVIDIA Quantum InfiniBand for all-reduce, a separate BlueField-DPU-offloaded Ethernet fabric for storage, explicitly to prevent contention between the two traffic classes), and cites SUNK (Slurm on Kubernetes) topology-aware scheduling with health probes evicting failing nodes before they impact a job. What it shows: concrete production evidence tying node-level topology correctness to a measured, fleet-level metric (MFU), and names the dual-fabric pattern this lesson's core-concepts section describes. https://www.coreweave.com/blog/nvidia-h100-gpu-benchmark-results-what-we-learned-from-large-scale-gpu-testing
- **NADDOD — "High-Performance GPU Server Hardware Topology and Cluster Networking-2: A Deep Dive into 8-Card A100/A800 and H100 Host Configurations."** The clearest practitioner walkthrough of the 8-GPU baseboard, NVSwitch fabric, PCIe stubs, and rail-aligned NICs. What it shows: an independent, detailed confirmation of the reference topology diagrammed above, from a networking-vendor's practitioner perspective rather than NVIDIA's own marketing material. https://www.naddod.com/blog/high-performance-gpu-server-hardware-topology-and-cluster-networking-2
- **Frank Denneman — "Topology-Aware Multi-GPU VM Placement"** (Part 11 of "Architecting AI Infrastructure"). Covers vSphere Device Groups and NVIDIA vGPU Manager/Fabric Manager partitioning to preserve the NVLink-domain shape a multi-GPU VM needs. What it shows: the same 8-GPU baseboard reality from a virtualization angle — what happens to this topology once it's carved up for VMs rather than run bare-metal. https://frankdenneman.ai/2026-03-31-Topology-Aware-Multi-GPU-VM-Placement/
- **NVIDIA Developer Blog — "Introducing NVIDIA HGX H100: An Accelerated Server Platform for AI and High-Performance Computing."** The primary-source confirmation of the 900 GB/s NVLink figure, the 4-NVSwitch count, and the platform's PCIe Gen5 host connectivity. What it shows: the authoritative numbers this lesson's reference diagram is built from, straight from the vendor. https://developer.nvidia.com/blog/introducing-nvidia-hgx-h100-an-accelerated-server-platform-for-ai-and-high-performance-computing

## Worked example

**Goal: map a rented "8x H100 SXM" instance onto the reference and verify rail alignment.**

**1. Pull the GPU-GPU + GPU-NIC matrix:**

```
$ nvidia-smi topo -m
        GPU0 GPU1 GPU2 GPU3 GPU4 GPU5 GPU6 GPU7 NIC0 NIC1 ... NIC5 ...  CPU Affinity  NUMA
GPU0     X   NV18 NV18 NV18 NV18 NV18 NV18 NV18 PXB  SYS      SYS       0-47,96-143    0
GPU5    NV18 NV18 NV18 NV18 NV18  X   NV18 NV18 SYS  SYS      PXB       48-95,144-191  1
...
NIC5     SYS  SYS  SYS  SYS  SYS  PXB  SYS  SYS   X
```

**Read it against the reference:**

- **GPU-GPU is `NV18` for every pair** — all-to-all NVLink, 18 links, through the 4 NVSwitches. Matches the reference: GPU-GPU is not on PCIe, not NUMA-bound. Good.
- **GPU0's CPU affinity is `0-47,96-143`, NUMA node 0; GPU5's is `48-95,144-191`, NUMA node 1.** Confirms the split: GPU0-3 → socket 0, GPU4-7 → socket 1. If a scheduler pins a GPU5 job to cores 0-47, every host copy crosses UPI — a placement bug you can now name.
- **GPU5 ↔ NIC5 is `PXB`** (a couple of PCIe bridge hops, staying local) while GPU5 ↔ NIC0 is `SYS` (crosses the socket interconnect). So **NIC5 is GPU5's rail-aligned NIC** — correct. GPUDirect RDMA for GPU5 will go GPU5→NIC5 locally.

**2. Confirm the physical hierarchy with `lspci -tv`:** find NIC5 and GPU5 and verify they share an upstream bridge / sit under the same socket-1 root complex (high BDF, e.g. `c0`/`e0` domain), matching the `PXB` reading. This is the lesson 03 tree skill applied to confirm the `topo -m` label.

**3. Spot the failure it would catch:** on a *mis-cabled* node you'd instead see GPU5 ↔ NIC5 as `SYS`, and some other NIC as GPU5's local one. That means GPU5's "rail" NIC is on the wrong socket — inter-node all-reduce for GPU5 crosses UPI on every node, silently. The reference-vs-real mapping is what surfaces it; no per-GPU utilization metric will.

**4. Translate the defect into the metric an interviewer will ask for.** If GPU5's rail is misaligned, expect roughly the 20-40% collective-bandwidth loss this lesson's stakes section quotes, on every all-reduce that touches GPU5. Stated in CoreWeave's own terms: that's the kind of defect that shows up as a multi-point MFU drop across the run — translate it concretely as "X points of MFU lost, at $Y/GPU-hr, over a Z-week run" rather than leaving it as an abstract percentage. That framing is what a staff-level, economics-minded interview question is actually asking for.

**Outcome:** a table — reference expectation vs. observed `topo -m` label — for all 8 GPUs and their NICs, with rail alignment confirmed or flagged, and the cost of any flagged misalignment stated in fleet-metric terms.

## Practice

Feeds the **Topology Teardown** deliverable.

1. **Draw the reference from memory first**, before touching a machine: 2 sockets, GPU0-3/GPU4-7 split, 4 NVSwitches with all-to-all NVLink at 900 GB/s, 8 Gen5 x16 retimer stubs (no baseboard PCIe switch), 8 rail-aligned ConnectX-7. Save the sketch. Cross-check it against the NADDOD and NVIDIA HGX references below and correct any gaps.
2. **Capture the real node:** `nvidia-smi topo -m > topo.txt` and `lspci -tv > tree.txt` on a rented 8x H100 (or A100) instance.
3. **Build the reference-vs-real mapping table.** Columns: `GPU | expected socket/NUMA | observed CPU affinity + NUMA | expected rail NIC | observed local NIC (topo label) | aligned? (Y/N)`. Verify GPU-GPU is `NV18`/all-to-all, the 4/4 socket split holds, and each GPU's local NIC is `PIX`/`PXB` (not `SYS`).
4. **Flag divergences:** any GPU whose local NIC shows `SYS`/`PHB`, any wrong socket affinity, any GPU-GPU pair not on NVLink. State the throughput consequence of each, and where you can, translate it into the MFU-style framing from the worked example.

**Acceptance:** a committed reference-vs-real mapping in the deliverable — your from-memory HGX H100 diagram plus a table binding all 8 real GPUs (and their NICs) to the reference, with socket split, GPU-GPU-on-NVLink, and per-GPU rail alignment each explicitly confirmed or flagged. A fully-aligned node is a valid result; the mapping must prove it with the `topo -m` labels and NUMA affinities, not assert it.

## Common pitfalls

1. **Assuming "8x H100" boxes from different vendors are topologically identical.** They're built on the same reference HGX baseboard, but host CPU choice (Intel vs AMD), NIC count/placement, and storage NIC design can differ — always verify a new SKU against the reference rather than assuming it.
2. **Believing NVSwitch means literally "no bottleneck, ever."** NVSwitch is non-blocking *for the traffic patterns it's designed for* (all-to-all within its domain) — a workload with pathological access patterns or contention at the domain boundary can still see uneven latency. "Non-blocking" describes a topology property of the fabric, not a performance guarantee under every possible traffic pattern.
3. **Conflating within-node rail alignment with the network's rail-optimized design.** They're related but distinct: within-node rail alignment is a PCIe-placement fact about one server; the network's rail-optimized topology is a separate switch/cabling design choice that must *also* be correct across the whole fleet for the within-node work to pay off. Getting one right without the other still loses the throughput.
4. **Assuming the CPU-GPU split is symmetric across every vendor's 8-GPU board.** The 4/4 split is the NVIDIA reference design; some OEM boards vary. Always confirm with `nvidia-smi topo -m`'s NUMA Affinity column rather than assuming 4/4 from memory alone.
5. **Treating GB200 NVL72 as "just a bigger HGX box."** The rack-scale NVLink domain (72 GPUs, external NVSwitch trays, coherent Grace-CPU NVLink-C2C) is architecturally distinct from scaling up the 8-GPU chassis — know the term and the shape of the change even if full depth isn't required yet.

## Self-check

- Which GPUs share socket 0 on a standard HGX H100 node? **Answer:** **GPU0, GPU1, GPU2, GPU3.** The eight GPUs are split 4/4 across the two CPU sockets — GPU0-3 attach to socket 0, GPU4-7 to socket 1 — which you confirm via each GPU's CPU-affinity/NUMA-node column in `nvidia-smi topo -m`. This governs GPU↔host and GPU↔NIC locality; GPU↔GPU is unaffected because it rides NVLink.
- Where should the training NIC for GPU5 attach, and why? **Answer:** On a **socket-1 PCIe complex, co-located with GPU5** — GPU5 is a socket-1 GPU (GPU4-7 → socket 1), so its rail NIC (NIC5) must sit under the same PCIe complex, showing `PIX`/`PXB` to GPU5 in `topo -m`. That keeps **GPUDirect RDMA** GPU5→NIC5 local (no CPU/UPI hop) and keeps GPU5's traffic on its **rail** so inter-node all-reduces between GPU5s stay on one leaf switch. Attaching it to a socket-0 slot forces RDMA across UPI and breaks rail alignment — a silent inter-node bandwidth loss.
- Why is there no PCIe switch on the HGX baseboard, and what carries GPU-GPU traffic instead? **Answer:** Because GPU-GPU traffic doesn't use PCIe — it rides **NVLink through the four NVSwitches** as a non-blocking all-to-all fabric at **900 GB/s per GPU (~7x PCIe Gen5)**. A baseboard PCIe switch would only add shared-bandwidth contention and buy nothing for the GPU-GPU path. Instead each GPU gets an **independent PCIe Gen5 x16 stub to its CPU socket** (carried over **retimers** for signal integrity, not a switch), used only for GPU↔host, GPU↔NIC, and GPU↔storage — the paths that genuinely need PCIe.
- What production metric would show a systemic rail-misalignment bug across a whole fleet, and roughly what would you expect it to do? **Answer:** **MFU (Model FLOPs Utilization)** — an aggregate metric that (unlike per-GPU utilization%) reflects whether delivered compute matches theoretical peak. A systemic rail-misalignment bug would show up as a meaningful multi-point MFU drop, comparable in scale to the 18-point (49.2% vs. a lower baseline) swing CoreWeave published between a well-tuned and a less-tuned 128-H100 run.
- Name the architectural reason production 8-GPU nodes often carry two separate NIC fabrics rather than one. **Answer:** To separate compute/collective RDMA traffic (all-reduce over InfiniBand) from storage and management traffic (a separate Ethernet/DPU-offloaded fabric), so storage or checkpoint I/O never contends on the wire with a training job's collective communication. CoreWeave names this explicitly as their dual-fabric design pattern.

## Connections & what's next

This lesson is the reference picture the rest of the module measures against. Lesson 03 supplied the single-link vocabulary (`LnkCap`/`LnkSta`, `PIX`/`PXB`/`PHB`/`SYS`) this lesson applies at node scale. Lesson 05 takes the socket/NUMA split established here and asks how to make Kubernetes *guarantee* — not just hope for — GPU-CPU-memory alignment on a scheduler that is otherwise topology-blind. Lesson 06 reuses the same PCIe-locality reasoning for NVMe placement and GPUDirect Storage. Lesson 08's capstone is where you reconcile all four tools (`lstopo`, `lspci -tv`, `nvidia-smi topo -m`, `numactl --hardware`) against a real node and this lesson's reference diagram simultaneously.

Next: **lesson 05** moves from "can I read and draw the topology" to "can I make Kubernetes respect it" — Topology Manager, CPU Manager, and Memory Manager, and the difference between what each policy *guarantees* versus what it merely *attempts*.

## References & further reading

**Primary sources**
- NVIDIA Developer Blog, "Introducing NVIDIA HGX H100" — https://developer.nvidia.com/blog/introducing-nvidia-hgx-h100-an-accelerated-server-platform-for-ai-and-high-performance-computing — authoritative source for the 900 GB/s NVLink / 4-NVSwitch numbers and the platform's PCIe Gen5 host connectivity.
- NVIDIA, DGX-1 System Architecture whitepaper (topology/NVLink section) — https://images.nvidia.com/content/pdf/dgx1-v100-system-architecture-whitepaper.pdf — read only for the enduring structure (socket split, NVLink fabric, GPU-NIC pairing); **ignore the V100-era performance numbers**, which the H100 figures above supersede.

**Real-world engineering blogs**
- CoreWeave, "NVIDIA H100 GPU Benchmark Results: What We Learned from Large-Scale GPU Testing" — https://www.coreweave.com/blog/nvidia-h100-gpu-benchmark-results-what-we-learned-from-large-scale-gpu-testing — what it shows: measured MFU numbers and the dual-fabric (compute/storage) architectural pattern in production.
- NADDOD, "High-Performance GPU Server Hardware Topology and Cluster Networking-2" — https://www.naddod.com/blog/high-performance-gpu-server-hardware-topology-and-cluster-networking-2 — what it shows: an independent, detailed practitioner walkthrough of the 8-GPU baseboard confirming this lesson's reference diagram.
- Frank Denneman, "Topology-Aware Multi-GPU VM Placement" — https://frankdenneman.ai/2026-03-31-Topology-Aware-Multi-GPU-VM-Placement/ — what it shows: the same topology preserved (or broken) once it's virtualized rather than run bare-metal.

**Deeper dives**
- Frank Denneman, "Understanding Multi-GPU Topologies Within a Single Host" (Part 10, prerequisite reading to the Part 11 piece above) — https://frankdenneman.ai/2026-03-27-Understanding-Multi-GPU-Topologies-Within-a-Single-Host/
