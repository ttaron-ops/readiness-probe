---
lesson: "02b.6"
title: "NVMe and storage placement for GPU data paths"
module: "02b"
concept: "NVMe and storage placement for GPU data paths"
status: not-started
est_time: "6h"
prev: "05-topology-alignment-k8s.md"
next: "07-power-and-thermals.md"
artifacts: []
sources: 8
---
# 02b.6 · NVMe and storage placement for GPU data paths
> **Concept.** Local NVMe (or the NIC serving remote storage) that does not share the GPU's PCIe switch/root complex contends for the same PCIe path the GPU feeds on, and the stall shows up as "GPU idle" that no utilization metric explains — and GPUDirect Storage only pays off when that placement is correct.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Where this fits

Lesson 05 gave you a Kubernetes-level guarantee: with the right kubelet policies, a pod's CPUs, memory, and GPU are provably co-located on one NUMA node. That guarantee says nothing about a fourth resource every training job depends on — the drive (or network path) the dataset lives on. A pod can pass every check in lesson 05's worked example — `single-numa-node`, admitted, CPUs and GPU on the same NUMA node — and still run at half the throughput its FLOPs budget predicts, because the NVMe controller feeding it sits across the socket boundary. This lesson closes that last placement gap by extending the exact same PCIe-topology reading skill from lessons 03 and 04 to storage, and it introduces the one software mechanism, GPUDirect Storage, whose entire value proposition depends on getting that placement right. It's also the last piece before lesson 07's power/thermal factors and the lesson 08 capstone, where you'll reconcile GPU, CPU, NIC, and NVMe placement into one diagram on a real node.

## Why this matters

You already know the GPU is the expensive part. What the utilization dashboard never tells you is that a GPU spends real wall-clock time waiting for the next batch to arrive from disk — and while it waits, `nvidia-smi` can still print `GPU-Util: 100%` because that counter reports "a kernel was resident in the last sample window," not "the SM array did useful work." A training step that is 30% data-loader-bound is 30% paid-for-but-undelivered compute. On an 8×H100 box at neocloud rates (~$2–3/GPU-hr, a 2026 snapshot), a persistent 20% data stall across a 512-GPU cluster is a low-six-figure annual line item that appears nowhere in your monitoring.

The root cause is almost always placement: the NVMe drive holding the dataset (or the read cache in front of an object store) sits on the *wrong* PCIe root complex, so every byte it reads crosses the inter-socket link (UPI/xGMI) or contends on the same switch the GPU is using for peer-to-peer and NIC traffic. The fix is free — it's a scheduling/placement decision, not new hardware — which is exactly why it's one of the highest-ROI findings a platform engineer can bring to a cost review: no capital spend, just correct placement. But you can only make the fix if you can read the topology. That is this lesson.

## What's new here (calibration)

You've read `lspci` a hundred times for NIC and HBA troubleshooting, and you know NVMe's queue-depth advantage over SATA at a generic block-I/O level, so we skip re-deriving those. New here:

- **New:** treating *storage* as a first-class citizen on the GPU's PCIe path — not just "somewhere on the host," but explicitly on the GPU's switch, on the GPU's root complex, or across the socket, with wildly different achievable bandwidth in each case.
- **New:** reading `nvidia-smi topo -m` for GPU↔NIC↔NVMe relationships, not just GPU↔GPU.
- **New:** **GPUDirect Storage (GDS)** — the DMA path that lets an NVMe controller write straight into GPU HBM, and the crucial detail that GDS can silently run in a slower *compatibility mode* even when your code calls the fast-path API correctly.
- **New:** generalizing the local-NVMe placement question to network-attached storage (NVMe-oF, parallel filesystems) — the more common real-world case at cluster scale, where "which drive" becomes "which NIC serves this GPU's storage traffic."

## Core concepts

### The three placement cases

For a GPU reading training data off a local drive, the byte path is one of:

1. **Same PCIe switch.** NVMe and GPU both hang off the same PCIe switch chip (common in DGX/HGX-class boxes where a PLX/Broadcom switch fans out lanes). Traffic goes NVMe → switch → GPU without ever touching the CPU's root port or the inter-socket link. Lowest latency, full drive bandwidth, and the *only* topology where GPUDirect Storage reaches its peak. `nvidia-smi topo -m` prints `PIX` (single PCIe bridge) or `PXB` (multiple bridges) for this pair.

2. **Same root complex, different switch/port.** NVMe and GPU are under the same CPU socket's root complex but traverse the CPU's internal PCIe fabric to meet. Fine for host-staged I/O; adds the host root port as a shared contention point. `topo -m` shows `PHB` (traverses a PCIe Host Bridge).

3. **Cross-socket / cross-root-complex.** The NVMe is on socket 0's root complex, the GPU on socket 1's. Every read crosses the CPU-to-CPU interconnect — Intel **UPI** or AMD **Infinity Fabric (xGMI)** — which is shared with all cross-socket memory traffic and is bandwidth-limited (UPI is on the order of ~20 GB/s per link; a Gen5 x4 NVMe alone can push ~14 GB/s of reads, and a Gen5 x16 GPU wants ~64 GB/s). `topo -m` shows `SYS` (traverses the SMP interconnect between NUMA nodes). This is the case that silently caps your data loader and shows up as GPU idle.

The asymmetry is the whole point: in case 3 the drive's rated bandwidth is real, but you cannot deliver it to *that* GPU without stealing from every other cross-socket transfer on the box.

### Why a misplaced disk reads as "GPU idle"

The data-loader pipeline is: NVMe read → page cache / CPU DRAM (bounce buffer) → (decode/augment on CPU) → `cudaMemcpy` H2D → GPU HBM → step. If the NVMe read leg is throttled by a cross-socket hop, batches arrive late. The GPU finishes step *N*, has no batch *N+1* ready, and blocks. During that block:

- `nvidia-smi` utilization may still read high if you sample coarsely, or drop to a sawtooth if you sample fast — but averaged it looks "busy enough" that nobody investigates.
- SM occupancy / `DCGM` `SM_ACTIVE` and `TENSOR_ACTIVE` tell the true story: they sag on the same cadence as the stall. Most teams don't chart those.
- The trainer's own step-time histogram shows a bimodal distribution: fast steps (batch was ready) and slow steps (waited on I/O). That bimodality is the fingerprint.

So the observable symptom is: high-cost GPU, "looks utilized," step time worse than the FLOPs budget predicts, and a data-loader that is CPU/PCIe-starved. The tell that it's *placement* and not slow code: the same loader on a GPU that shares the drive's root complex runs fine.

### Queue depth and parallelism (briefly)

NVMe's advantage over SATA is many deep queues (up to 64K queues × 64K entries in spec; real drives expose dozens). A single-threaded, QD-1 reader will leave a Gen5 SSD 80% idle no matter where it sits — the drive needs multiple outstanding requests to keep its internal channels busy. So before you blame topology, confirm the loader is actually issuing parallel I/O: `num_workers` in a PyTorch `DataLoader`, `O_DIRECT` + `io_uring` or `libaio` with QD ≥ 16–32, or a prefetch depth ≥ 2. Topology sets the *ceiling*; queue depth determines whether you reach it. A shallow queue makes even a perfectly placed drive look slow, which is why you check both — and why "I moved the drive to the right root complex and nothing changed" is a queue-depth question, not proof placement doesn't matter.

### GPUDirect Storage (GDS): mechanism and the two layers that can fail independently

GDS removes the **CPU bounce buffer** from the read path. Without it: NVMe → system DRAM → GPU (two copies, CPU orchestrates each, DRAM bandwidth and a CPU core burned per stream). With GDS: the NVMe controller DMAs **directly into GPU HBM** over PCIe.

GDS has two layers, and a failure can originate in either one:

- **`cuFile`** — the userspace API applications call (`cuFileRead`/`cuFileWrite`), part of the CUDA userspace stack. An application-level GDS bug shows up here, as a cuFile API error.
- **`nvidia-fs`** — the *kernel* module that actually performs the peer-to-peer DMA registration between the block device and the GPU's PCIe BAR memory. A kernel-side registration failure shows up in `dmesg`/`nvidia-fs` logs, not in the application's cuFile return codes.

What GDS removes when it's working:
- the extra copy through pinned host memory (the bounce buffer),
- the CPU cycles that drive that copy,
- the system-DRAM bandwidth that copy consumes.

What GDS **requires to actually help**: the NVMe and the target GPU must share a **PCIe switch** (or at minimum the same root complex) so the peer-to-peer DMA doesn't have to be brokered across the CPU root port or the inter-socket link. If the drive is cross-socket from the GPU, GDS either falls back to a staged/compatibility path or delivers a fraction of its potential — you've enabled the feature and gotten none of the win.

The trap worth naming explicitly: **"GDS enabled" is not the same claim as "GDS is on the fast path."** GDS can silently run in **compatibility mode** — functionally correct, staged through a bounce buffer under the hood, just not faster — without the application ever seeing an error. The `gdscheck -p` tool (shipped with the GDS package) reports per-mount, per-GPU support status and tells you whether a given GPU/mount pair is actually getting the direct DMA path or has quietly fallen back. If a placement fix doesn't move the needle on GDS throughput, `gdscheck -p` plus a `dmesg`/`nvidia-fs` log check is the next diagnostic step — one confirms whether the fast path is active at all, the other tells you whether the failure is a kernel-level registration problem distinct from an application-level GDS misuse.

### Generalizing beyond local NVMe: network-attached storage

Local NVMe is the right starting mental model, but at real cluster scale, training data more often lives on a parallel filesystem (Lustre, WekaFS, VAST, an S3-compatible object store) reached over the network, not a drive plugged into the same chassis. The placement question doesn't disappear — it moves. Instead of "which NVMe is on this GPU's switch," the question becomes **"which NIC serves this GPU's storage traffic, and is that NIC on the GPU's switch?"** The same `PIX`/`PXB`/`PHB`/`SYS` legend applies, just to a storage-serving NIC instead of a local drive; **NVMe-oF (NVMe over Fabrics)** is the protocol that extends the local NVMe command set across that network fabric so the placement-sensitivity carries through end to end. If your production storage is remote, don't stop at "the local drive is fine" — check the storage-serving NIC's relationship to the GPU exactly as you'd check a compute NIC's.

### Concrete throughput numbers, for scale

Two dated snapshots worth knowing, both real production benchmarks:

- **2020, DGX-2 era:** an independently tested (Microsoft Research) head-to-head GDS benchmark reported **97.9 GB/s** for WekaIO and **92.6 GB/s** for VAST Data, both over InfiniBand-attached storage feeding GDS on a DGX-2. Dated — cite as a 2020 snapshot, not a current number.
- **2026-era:** Oracle Cloud Infrastructure's benchmark with WEKA reports **192 GiB/s sequential-read GDS throughput on a single client node** with OCI H100 GPUs — roughly 3× a single Gen5 x16 GPU link's ~63 GB/s ceiling, achieved by aggregating across multiple GPUs/NICs on the node rather than one link.

The point of citing both isn't the exact numbers — it's the trend: GDS throughput at the high end has scaled roughly 2× over six years as storage vendors compete specifically on "how close to raw drive/network bandwidth can we deliver straight into HBM." That competition exists *because* the CPU-bounce-buffer path becomes the bottleneck at H100-class GPU bandwidth — a direct commercial expression of this lesson's core claim.

## Perspectives

**Developer.** A `DataLoader` author sees only `num_workers`, `pin_memory`, `prefetch_factor` — the queue-depth discussion above. GDS/cuFile is typically invoked by the framework or a storage vendor's plugin, not written by hand; the developer's job is choosing a GDS-aware data-loading library or backend, not implementing the DMA path themselves.

**Operator.** Storage-to-GPU placement is a fleet acceptance-test item exactly like PCIe link state (lesson 03): verify `nvidia-smi topo -m` and `gdscheck -p` at node bring-up, not after a training job's step-time histogram already looks bimodal in production.

**Storage vendor.** WEKA and VAST both built their GDS integrations specifically because the CPU-bounce-buffer path becomes the bottleneck at H100-class GPU bandwidth. Their public benchmarks exist because storage vendors compete commercially on delivering bandwidth as close to raw drive/network speed into HBM as possible — a direct, market-tested confirmation of this lesson's core claim that placement, not raw drive speed, is usually the limiting factor.

**Economics.** This lesson's own framing — 30% data-stall equals 30% paid-for-but-undelivered compute — is the right lens, and it's worth stating explicitly that a misplaced or under-provisioned storage path is one of the *cheapest* fixes in the whole module: no new hardware, just a placement or scheduling change, yet one of the most impactful line items at fleet scale.

## Real-world use cases

- **VAST Data — "NVIDIA GPU Direct Storage: The VAST Data Story"** — a storage vendor's account of implementing GDS in production storage. What it shows: what GDS integration looks like from the storage side, and why a vendor invests in the direct-DMA path rather than accepting a CPU bounce buffer. `https://www.vastdata.com/blog/nvidia-gpu-direct-for-storage-gds-the-vast-data-story`
- **Blocks & Files — "WekaIO races out of the blocks in GPUDirect storage race"** (2020) — an early, independently-tested head-to-head GDS benchmark (WekaIO vs. VAST, DGX-2 era, InfiniBand-attached storage). What it shows: real, measured GDS throughput numbers at the point GDS was newly production-viable — cite as a 2020 snapshot. `https://blocksandfiles.com/2020/11/03/gpudirect-storage-race/`
- **Oracle Cloud Infrastructure Blog — "Accelerate AI Model Performance with WEKA's NeuralMesh Axon and OCI GPU Compute"** — a current-generation (2026-era) production number, 192 GiB/s sequential-read GDS throughput on a single H100 client node. What it shows: how far GDS throughput has scaled since the 2020 DGX-2-era numbers above, and what "good" looks like today. `https://blogs.oracle.com/cloud-infrastructure/accelerate-ai-performance-weka-converged-storage`
- **CoreWeave Docs — "About CoreWeave Storage"** — describes CoreWeave's AI Object Storage serving data directly to GPU nodes, including via **Tensorizer** (CoreWeave's model-serialization tool) loading tensors from S3/HTTP endpoints toward GPU memory at up to 2 GB/s per GPU across large fleets. What it shows: a neocloud-specific, network-attached-storage version of this lesson's placement problem, at genuinely large scale. `https://docs.coreweave.com/docs/products/storage`

## Worked example

Suppose an 8-GPU host. Locate storage relative to the GPUs and trace one data path.

Start with the PCIe tree:

```
$ lspci -tv | grep -iE 'nvme|nvidia|pci bridge' 
-+-[0000:c0]-+-01.1-[c1]----00.0  NVIDIA Corporation GH100 [H100 SXM5]
 |           \-03.1-[c2]----00.0  Samsung NVMe SSD Controller PM9A3
 \-[0000:00]-+-01.1-[01]----00.0  NVIDIA Corporation GH100 [H100 SXM5]
             \-05.1-[04]----00.0  Samsung NVMe SSD Controller PM9A3
```

Read it: bus `0000:c0` and bus `0000:00` are two separate root complexes (two host bridges — one per socket on a 2-socket box). The GPU at `c1` and the NVMe at `c2` are both under `[0000:c0]` — **same root complex**. Good. But note the GPU at `01` under `[0000:00]` and the NVMe at `04` also under `[0000:00]` — that pairing is local too. The trap would be a training job pinned to the `c0`-side GPU reading a dataset staged on the `00`-side NVMe: that is a cross-socket path.

Now confirm with the GPU-aware view:

```
$ nvidia-smi topo -m
        GPU0    GPU1    NIC0    NVMe0   NVMe1   CPU Affinity  NUMA
GPU0     X      SYS     PXB     PXB     SYS     0-47          0
GPU1    SYS      X      SYS     SYS     PXB     48-95         1
NIC0    PXB     SYS      X      PIX     SYS
NVMe0   PXB     SYS     PIX      X      SYS
NVMe1   SYS     PXB     SYS     SYS      X
```

Read the legend: `PIX` = single PCIe bridge (same switch), `PXB` = multiple bridges on the same host, `PHB` = through a host bridge, `SYS` = across the SMP/NUMA interconnect. Trace the meat:

- **GPU0 ↔ NVMe0 = PXB.** Same root complex, close. GPU0's data loader should read from NVMe0. Note NVMe0 ↔ NIC0 = `PIX` — the drive and the NIC share a switch, which is exactly the sweet spot for GDS-over-network-fetched data or NVMe-over-Fabrics staging.
- **GPU0 ↔ NVMe1 = SYS.** Cross-socket. If GPU0 reads from NVMe1, every batch crosses the UPI/xGMI link. This is the misplacement to flag.
- **CPU Affinity / NUMA columns** confirm GPU0 lives on NUMA 0 (cores 0–47), GPU1 on NUMA 1. So "correct" placement is: GPU0 job → cores 0–47 → NVMe0; GPU1 job → cores 48–95 → NVMe1.

Conclusion for the deliverable note: *GPU0's local drive is NVMe0 (PXB, shared root complex, and NVMe0 shares a switch with NIC0 — GDS-ready). GPU0 reading NVMe1 is a cross-socket (`SYS`) path and must be avoided. Mirror for GPU1↔NVMe1.*

**Now confirm GDS is actually on the fast path, not just correctly placed:**

```
$ gdscheck -p
 GPU0    NVMe0 mount: /mnt/data0     GDS: Supported
 GPU0    NVMe1 mount: /mnt/data1     GDS: Unsupported (compatibility mode)
```

The `PXB` topology alone tells you GDS *can* be fast for GPU0↔NVMe0 — `gdscheck -p` confirms it *is*. If GPU0 were instead reading `/mnt/data1` (the cross-socket NVMe1), `gdscheck -p` would show the mount falling back to compatibility mode: functionally correct, silently slow, no error raised. That's the concrete difference between "GDS enabled" and "GDS fast" — one topology check, one tool call, and you know which one you actually have.

## Practice

Feeds the **Topology Teardown** deliverable.

1. Run `lspci -tv` on the target host. Identify every NVMe controller and every GPU, and note which root complex (top-level bus, e.g. `[0000:c0]`) each hangs off. If you're on a laptop/dev box with no discrete GPU, run it against a cloud GPU instance or capture the output from a DGX/HGX node.
2. Run `nvidia-smi topo -m`. For **each GPU**, record the relationship (`PIX`/`PXB`/`PHB`/`SYS`) to every NVMe and to the NIC(s). Note the CPU Affinity and NUMA node columns.
3. Cross-reference: for each GPU, name its *local* NVMe (best relationship code) and flag any GPU→NVMe pairing that is `SYS` (cross-socket) or `PHB` — those are the data paths that will stall.
4. If GDS matters for your workload, check `PIX`/`PXB` between the intended GPU and drive, and confirm the stack with `gdscheck -p` — record whether each mount reports supported GDS or compatibility-mode fallback, not just whether the topology looks right.
5. If your training data is network-attached rather than local NVMe, repeat step 2's relationship check for the storage-serving NIC instead of a local drive, and note which GPU(s) it's local to.

**Acceptance:** a note in the Topology Teardown that states, for each GPU, where its storage sits (relationship code + which drive or NIC is local + `gdscheck -p` result if GDS applies), and explicitly identifies any cross-socket/cross-root-complex data path on the box. One or two sentences per GPU plus the raw `topo -m` output pasted in.

## Common pitfalls

1. **Blaming topology before confirming queue depth.** A shallow-queue reader (QD-1, single-threaded) leaves even a perfectly-placed Gen5 drive 80% idle. Check `num_workers`/`io_uring` queue depth ≥ 16–32 before concluding the bottleneck is placement.
2. **Assuming "GDS enabled" means the fast path is active.** GDS can silently run in compatibility/fallback mode — the cuFile calls still succeed, nothing errors, it's just staged through a bounce buffer under the hood. `gdscheck -p` is the way to confirm the fast path is genuinely in use, not just whether the application's API calls return success.
3. **Forgetting storage is often network-attached, not local NVMe, at real cluster scale.** The local-NVMe framing is the right pedagogical starting point, but a staff engineer should immediately generalize to "which NIC serves this GPU's storage path" for parallel-filesystem-backed training data — the same `PIX`/`PXB`/`SYS` logic, applied to a different device.
4. **Trusting `nvidia-smi topo -m` as a complete picture without cross-referencing `lspci`.** The tool shows relationship codes between named rows, but doesn't hand you a friendly device inventory — you still need `lspci -tv`/`lspci -D` to confirm exactly which PCI address is which physical drive before trusting the topology matrix's labels.
5. **Assuming a "misplaced NVMe" only ever means a local-drive placement mistake.** With network-attached storage, the equivalent misplacement is a storage-serving NIC on the wrong root complex — same failure shape, different device, and easy to miss if you only ever check local drives out of habit.

## Self-check

- **Is your NVMe on the GPU's root complex, and how do you tell?**
  **Answer:** Two independent checks. Structurally, `lspci -tv` shows both devices hanging off the same top-level host-bridge bus (e.g. both under `[0000:c0]`) — different top-level buses means different root complexes, typically different sockets. Functionally, `nvidia-smi topo -m` prints the relationship: `PIX` (same switch) or `PXB`/`PHB` (same host/root complex, more hops) means same root complex; `SYS` means the path crosses the SMP/NUMA interconnect to a *different* root complex. `SYS` is the answer you don't want for a data drive.
- **What production symptom does a misplaced data disk produce, and why does it look like "GPU idle"?**
  **Answer:** The GPU periodically stalls waiting for the next batch because reads cross the UPI/xGMI link and can't be delivered at full drive bandwidth. Step times become bimodal (fast when the batch was prefetched, slow when it waited on I/O), and measured throughput falls short of the FLOPs budget. It reads as "GPU idle" because during the wait the SM array does no work — `DCGM SM_ACTIVE`/`TENSOR_ACTIVE` sag — even though coarse `nvidia-smi` utilization can still show high (it reports kernel residency, not useful work). The cost story: you pay full GPU rate for wall-clock time the accelerator spends blocked on a fixable placement bug.
- **What does GPUDirect Storage remove from the data path, and what placement does it require?**
  **Answer:** GDS removes the CPU bounce buffer — the staging copy through pinned system DRAM — letting the NVMe controller DMA directly into GPU HBM via `nvidia-fs`/cuFile. That eliminates one copy, the CPU cycles driving it, and the system-DRAM bandwidth it consumed. To actually pay off it requires the NVMe and target GPU to share a PCIe switch (`PIX`/`PXB`), ideally the same root complex; if they're cross-socket (`SYS`) the peer-to-peer DMA is brokered across the CPU root port / inter-socket link and GDS falls back or delivers a fraction of its benefit.
- **Your GDS-enabled job's storage read throughput doesn't budge when you fix the NUMA placement — what would you check next?**
  **Answer:** First, `gdscheck -p` for that specific GPU/mount pair — GDS can be quietly running in compatibility mode (functionally correct, staged through a bounce buffer, not actually using the direct-DMA path) with no application-visible error, in which case the NUMA fix alone won't change throughput. Second, if `gdscheck -p` says GDS is supported, check `dmesg`/`nvidia-fs` logs for a kernel-level DMA registration failure distinct from any cuFile-level API error, since the two layers (`cuFile` userspace, `nvidia-fs` kernel) can fail independently.

## Connections & what's next

This lesson generalizes lesson 05's core claim — "physical co-location is a policy decision, not a given" — from Kubernetes' CPU/memory/GPU alignment to a fourth resource, storage, and shows the same `PIX`/`PXB`/`PHB`/`SYS` reading skill from lessons 03 and 04 applies directly. The economics are identical to the NUMA-misalignment story: the failure is invisible in coarse utilization metrics and free to fix once diagnosed. Next, **lesson 07** covers the last host-side factor this module's checkpoint asks you to diagnose — power and thermal throttling — completing the "GPU at 100% util, throughput half spec" diagnostic tree that started with PCIe link state in lesson 03. All of it converges in the **lesson 08 capstone**, where you reconcile GPU, CPU, NIC, and NVMe placement — plus power/thermal state — into one topology diagram on a real node, exactly the skill this lesson's worked example and practice tasks rehearse.

## References & further reading

**Primary sources**
- NVIDIA — "GPUDirect Storage Configuration & Overview" — `https://docs.nvidia.com/gpudirect-storage/configuration-guide/index.html` — the authoritative source on the DMA path, `nvidia-fs`, cuFile, and `gdscheck`.
- `lspci(8)` man page — `https://man7.org/linux/man-pages/man8/lspci.8.html` — the `-t` (tree) and `-v`/`-vvv` flags for reading root complexes and bridges; pair with `nvidia-smi topo -m`.
- PyTorch — `DataLoader` documentation — `https://pytorch.org/docs/stable/data.html` — `num_workers`, `pin_memory`, and prefetch guidance to distinguish a shallow-queue software stall from a topology stall before you blame placement.

**Real-world engineering blogs**
- VAST Data — "NVIDIA GPU Direct Storage: The VAST Data Story" — `https://www.vastdata.com/blog/nvidia-gpu-direct-for-storage-gds-the-vast-data-story` — what it shows: a storage vendor's account of building GDS support in production.
- Blocks & Files — "WekaIO races out of the blocks in GPUDirect storage race" (2020) — `https://blocksandfiles.com/2020/11/03/gpudirect-storage-race/` — what it shows: an early, independently-tested GDS benchmark; dated snapshot (2020, DGX-2 era).
- Oracle Cloud Infrastructure Blog — "Accelerate AI Model Performance with WEKA's NeuralMesh Axon and OCI GPU Compute" — `https://blogs.oracle.com/cloud-infrastructure/accelerate-ai-performance-weka-converged-storage` — what it shows: current-generation (2026) GDS throughput at the high end, for scale contrast against the 2020 numbers.
- CoreWeave Docs — "About CoreWeave Storage" — `https://docs.coreweave.com/docs/products/storage` — what it shows: a neocloud's production network-attached-storage architecture and its GPU-facing throughput target.

**Deeper dives**
- WEKA — "GPUDirect Storage: How it Works and More" — `https://www.weka.io/learn/glossary/gpu/what-is-gpudirect-storage/` — a vendor explainer good for going deeper on the DMA mechanism itself.
