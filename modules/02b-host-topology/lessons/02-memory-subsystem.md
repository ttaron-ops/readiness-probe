---
lesson: "02b.2"
title: "Memory subsystem as placement consequence"
module: "02b"
concept: "Memory subsystem as placement consequence"
status: not-started
est_time: "6h"
prev: "01-host-topology-tree.md"
next: "03-pcie.md"
artifacts: []
sources: 5
---

# 02b.2 · Memory subsystem as placement consequence

> **Concept.** Host memory bandwidth and latency are placement outcomes — which NUMA node a buffer lives on, and how many DDR channels are populated, decide how fast a GPU can be fed.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Where this fits

Lesson 01 gave you the static tree: sockets, NUMA nodes, and the fact that every PCIe device has exactly one home node, fixed at boot. That lesson answered *where* things live. This lesson answers *what it costs* when a buffer lives on the wrong branch — turning "the GPU's home node" from a fact you can read into a bandwidth number you can measure, quote, and defend in a cost review. Everything here assumes you can already produce the tree from lesson 01; if you can't yet name a GPU's home NUMA node from `nvidia-smi topo -m` without hesitating, that lesson isn't done.

## Why this matters

A GPU is only as fast as the path staging data into it, and that path bottoms out at host DRAM bandwidth — which you can silently halve by binding a pinned buffer to the wrong NUMA node or by shipping the box with half its memory channels populated. Neither shows up on a GPU dashboard: the accelerator reports high utilization while starved. The interview question that lands is "your H100 copy benchmark hits 28 GB/s instead of 55 — walk me through why," and the answer is almost always host-side placement or channel population, not the GPU. This lesson makes that failure measurable and quotable — the kind of finding ("we're leaving ~12% throughput on the table to a NUMA mis-bind, here's the benchmark") that gets a platform engineer believed in a room full of people staring at utilization graphs that all say 100%.

## What's new here (calibration)

From the Linux-internals module you already have local-vs-remote memory as a *kernel* concept and the cache hierarchy — none of that is re-taught here. On-prem you have sized DIMMs and known that "more channels = more bandwidth" as folklore.

The delta is treating **bandwidth and latency as placement consequences for an accelerator**, quantified and with the specific mechanisms named:
- **Local vs remote DRAM bandwidth as a hard, measured number**, not a distance code — and why a *pinned host buffer's* NUMA node changes GPU copy speed even though the GPU never touches DRAM directly.
- **DDR channel population as a bandwidth multiplier** you can lose at provisioning time — the difference between a 1-DIMM-per-channel full build and a half-populated box is not subtle.
- **Host DRAM vs GPU HBM as two tiers with a ~10× bandwidth gap**, which is *why* staging strategy (GPUDirect, NVLink) exists — you don't feed HBM from DRAM the naive way and expect it to keep up.
- **The specific mechanisms behind "pinned isn't enough"**: read-for-ownership (RFO) cost on writes, a second IOMMU/`SWIOTLB` bounce path independent of CUDA's own pageable-memory bounce buffer, and — for codebases using Unified Memory instead of explicit host/device buffers — the `cudaMemAdvise` hints that are UM's equivalent of `numactl --membind`.

## Core concepts

### Bandwidth vs latency, as placement
- **Latency** (~80–200 ns) dominates pointer-chasing and small, dependent accesses. Remote adds a fixed link-hop penalty.
- **Bandwidth** (streaming GB/s) dominates data staging — bulk host→device copies, dataloader→pinned-buffer memcpy. This is the accelerator-relevant axis.
Local DRAM bandwidth scales with **populated channels on that node**. Remote bandwidth is capped by the inter-socket link (lesson 01). So a remote buffer loses on both axes, but the *bandwidth* loss is what starves a GPU copy.

### Why a pinned host buffer's NUMA node changes GPU copy speed
The GPU never reads host DRAM with load instructions — it **DMAs** over PCIe. The DMA engine sits behind the GPU's PCIe root complex on the GPU's home socket. When it copies from a **pinned** (page-locked) host buffer:
- **Buffer on the GPU's home node:** DMA target DRAM → root complex → GPU, no inter-socket hop. Bandwidth is limited by PCIe (Gen5 x16 ≈ 63 GB/s theoretical, ~50–55 GB/s achievable pinned).
- **Buffer on the *other* socket:** DMA must pull those cache lines across UPI/Infinity Fabric first, *then* over PCIe. The copy is now throttled by the inter-socket link, commonly landing at **50–70% of the local number**.

Two more reasons pinning matters at all: pageable memory forces CUDA to stage through an internal pinned bounce buffer (extra copy, lower BW), and only pinned memory allows true async overlap. But *given* pinned memory, the NUMA node is the remaining lever — and it's the one nobody watches, because of exactly how the allocation gets its node.

**The mechanism to be precise about:** `cudaHostAlloc`/`cudaMallocHost` determines its NUMA node by **first-touch, by the calling thread** — the same rule as any anonymous memory allocation, not a CUDA-specific one. This means a dataloader's *thread affinity at the moment of allocation*, not at the moment of use, decides pinned-buffer placement. If a thread allocates a pinned buffer while running on node 0, then the OS scheduler later migrates that thread to node 3, the buffer's pages do **not** move with it — they stay on node 0, and every subsequent DMA from that buffer to a node-3 GPU pays the inter-socket tax, invisibly, forever, until the process restarts. This is why a naive single-pool dataloader is a silent trap on multi-socket boxes: the allocation-time thread, not the steady-state thread, decides the fate of every transfer that follows.

**For Unified Memory codebases**, the equivalent lever exists but under a different name: `cudaMemAdviseSetPreferredLocation` and `cudaMemAdviseSetAccessedBy` are the CUDA-level hints for `cudaMallocManaged` allocations — functionally the Unified-Memory analog of `numactl --membind`. UM's automatic page migration follows the accessing processor, but its *host-side* backing still follows first-touch/hints the same as any other Linux memory — so if your codebase uses managed memory instead of explicit host/device buffers, this is the API surface to check instead of assuming the runtime "just handles it."

### DDR channels — why population is bandwidth
A memory controller exposes multiple **channels**; each populated channel is an independent, parallel path to DRAM. Per-socket bandwidth ≈ channels × per-channel rate:
- **DDR5-4800**: 4800 MT/s × 8 B ≈ **38.4 GB/s per channel**.
- **Sapphire/Emerald Rapids Xeon**: **8 channels/socket** → ~307 GB/s/socket fully populated.
- **AMD EPYC Genoa/Turin**: **12 channels/socket** → ~460 GB/s/socket at DDR5-4800 (higher with faster DIMMs).

Populate half the channels (e.g. 4 DIMMs on an 8-channel socket) and you get **~half the bandwidth** — the controller can only parallelize across populated channels. This is a provisioning-time footgun: a box shipped 8×32 GB on an 8-channel socket is full-bandwidth; the same 256 GB as 4×64 GB is half-bandwidth for identical capacity. It never appears in inventory ("256 GB, check") and never appears on a GPU dashboard — only a bandwidth benchmark reveals it. Also relevant: exceeding 1 DIMM-per-channel (2DPC) often drops the DIMMs to a lower speed grade, trading a little bandwidth for capacity.

A staff-level footnote worth knowing: **DDR5 splits each DIMM's 64-bit channel into two independent 32-bit sub-channels**, unlike DDR4's single 64-bit channel per DIMM. This is why some BIOS/monitoring tools report a socket's "8 channels" as "16 sub-channels" — it's the same physical population, described at a finer grain, not double the bandwidth.

### Host DRAM vs GPU HBM — the ~10× gap that shapes staging
| Tier | Bandwidth | Capacity | Role |
|------|-----------|----------|------|
| Host DRAM (per socket, DDR5) | ~300–460 GB/s | 100s of GB–TBs | staging, dataloader, spill |
| GPU HBM (H100 SXM, HBM3) | ~3.35 TB/s | 80 GB | compute working set |
| GPU HBM (H200, HBM3e) | ~4.8 TB/s | 141 GB | " |
| GPU HBM (B200, HBM3e) | ~8 TB/s | 192 GB | " |
| PCIe Gen5 x16 (the bridge between them) | ~63 GB/s | — | host↔device transfer |

HBM out-runs a full socket's DRAM by roughly **10×**, and the PCIe bridge between them is ~5–7× slower still than DRAM. The consequences you must internalize: (1) you cannot stream from DRAM and keep HBM busy — the working set must **reside** in HBM, with PCIe used for prefetch/overlap, not per-step feeding; (2) because PCIe is the scarcest link, wasting any of it on a needless inter-socket detour (wrong-node buffer) is the most expensive placement mistake on the box; (3) this is why NVLink/NVSwitch (GPU↔GPU at 900 GB/s on H100) and GPUDirect (NIC/NVMe DMA straight into HBM, bypassing host DRAM) exist — to keep the slow host tier off the critical path.

### Why GPUDirect exists (and what it removes from the tree)
The default staging path for network or storage data into a GPU is: NIC/NVMe → host DRAM → PCIe → HBM. That routes everything through the slowest tier (host DRAM) and burns PCIe twice. **GPUDirect RDMA** lets the NIC DMA straight into HBM, and **GPUDirect Storage** lets NVMe DMA straight into HBM — both bypassing host DRAM and the CPU. The catch: it only works when the NIC/NVMe and the GPU share a PCIe path *without* crossing the inter-socket link. A GPU whose RDMA NIC is on the far socket cannot do a clean GPUDirect transfer — the DMA falls back through host memory or crosses the inter-socket link, quietly erasing the feature's benefit. So the placement tree from lesson 01 is a *precondition* for the memory-path optimizations, not an independent concern.

### The second bounce path nobody checks: IOMMU / SWIOTLB
Even a correctly pinned, correctly NUMA-placed buffer can still be slow for a reason that has nothing to do with NUMA: if the IOMMU is running in strict/translate mode without a fast passthrough path for that device, the kernel can additionally route the DMA through a **`SWIOTLB`** (software I/O translation lookaside buffer) bounce region. This is a **second, independent bounce path**, distinct from CUDA's own internal pageable-memory bounce buffer mentioned above — one lives at the kernel/IOMMU layer, the other at the CUDA runtime layer, and either can be active at once. If you've fixed pinning and NUMA placement and a copy is still slower than the arithmetic predicts, `dmesg` and the IOMMU's mode (`intel_iommu=on,strict` vs passthrough, or the AMD equivalent) is the next thing to check — not a fact most engineers who "know NUMA" think to look for.

### Bandwidth ceilings you should be able to recite
- **PCIe Gen4 x16** ≈ 32 GB/s/dir theoretical, ~26 GB/s pinned achievable.
- **PCIe Gen5 x16** ≈ 63 GB/s/dir theoretical, ~50–55 GB/s pinned achievable.
- **NVLink (H100, via NVSwitch)** ≈ 900 GB/s aggregate GPU↔GPU — ~14× a Gen5 x16 link, which is *why* multi-GPU collectives use NVLink and touch host DRAM as little as possible.
- **Per-socket DDR5-4800**: ~307 GB/s (8-ch Xeon) / ~460 GB/s (12-ch EPYC).
Chain them: HBM (~3.35 TB/s) ≫ DRAM (~300–460 GB/s) ≫ PCIe (~55 GB/s). Every arrow is a potential starvation point, and the cheapest one to accidentally halve is the DRAM→PCIe handoff, via a wrong-node buffer.

### The commands
- **`numactl --membind=N --cpunodebind=N <cmd>`** — force a benchmark's memory and threads onto node N. The core tool for local-vs-remote measurement.
- **CUDA `bandwidthTest`** (from `cuda-samples`): `bandwidthTest --memory=pinned --mode=range` reports HtoD/DtoH GB/s. Wrap in `numactl` to control the host buffer's node.
- **`numastat -p <pid>`** — per-node page counts for a running process; proves where a buffer actually landed vs where you intended.
- **STREAM** (`stream.c`, Triad) — pure host DRAM bandwidth, the GPU-free fallback; run under `numactl` local vs remote and read the Triad MB/s.
- **`mbw`** or `sysbench memory` — quick sanity BW checks when STREAM isn't built.

### Turning the gap into a number nobody else has
This is the module's whole point: the waste is real but off-dashboard. A 41% host→device bandwidth loss on a GPU whose per-step time is copy-bound translates directly into lower samples/sec at unchanged GPU-utilization %. If that GPU-hour costs (pick your neocloud rate) and the copy is on the critical path 30% of each step, you can state the loss as a throughput and dollar figure — a claim no `nvidia-smi`, DCGM, or Grafana panel makes for you, because they measure occupancy, not whether the occupied cycles were fed on the fast path. Wire the local-vs-remote copy benchmark into node-bringup validation and you catch half-populated boxes and mis-affinitized NICs *before* they bill for a quarter.

### Reading a STREAM result (the GPU-free fallback)
STREAM reports four kernels; **Triad** (`a[i] = b[i] + q*c[i]`, 2 reads + 1 write) is the one to quote as sustained bandwidth:
```
Function    Best Rate MB/s
Copy:           288400.1
Scale:          287110.5
Add:            291020.7
Triad:          290560.3
```
~290 GB/s local on an 8-channel DDR5-4800 socket ≈ 94% of the ~307 GB/s spec — healthy population. Re-run under `numactl --membind=<remote>` and Triad drops toward the inter-socket-link ceiling.

Two subtleties, both worth naming precisely:
1. STREAM must be built with arrays ≫ LLC (it warns if not) or you measure cache, not DRAM.
2. Write-heavy kernels can look artificially slow because of **read-for-ownership (RFO)**: under the MESI cache-coherence protocol, a plain store to a cache line the CPU doesn't already own triggers an implicit read of that line first. This is exactly *why* Copy's number is untrustworthy relative to Triad's, and it's also why some memory benchmarks report impossibly-high write numbers when compiled with flags that emit **non-temporal stores** (`MOVNTDQ` / `_mm_stream_*`) — those bypass RFO entirely by never bringing the destination line into cache. So Triad, not Copy, is the honest streaming number, and now you know the CPU-level mechanism that makes it so.

### Why the benchmark under-reads real training (and still matters)
`bandwidthTest` measures a single synchronous stream; real training overlaps copy with compute across multiple streams and may exceed the single-stream figure, or hide it entirely behind compute. So the *absolute* number isn't your training throughput — but the **local-vs-remote ratio is**: whatever the workload's copy sensitivity, a wrong-node buffer scales its host-transfer cost by that same ~1.4–2× factor. Use the benchmark for the *delta*, not as a throughput prediction, and you avoid the trap of dismissing placement because "the copy is overlapped anyway" — overlap hides latency, not a halved bandwidth ceiling.

## Perspectives

**Developer.** Pinned memory and `cudaHostAlloc` are framework details most ML engineers never touch directly — PyTorch's `DataLoader(pin_memory=True)` hides it. The developer's job is knowing *when* to override the framework default (multi-GPU, multi-socket boxes) and how to verify the framework actually pinned correctly per rank, not necessarily writing the allocation call by hand.

**Operator.** Bandwidth validation belongs in node bring-up and acceptance testing, not left to be discovered mid-training-run. STREAM plus `bandwidthTest` local-vs-remote should be a repeatable script run on every new rented instance, the same way you'd smoke-test disk or network before trusting a box.

**Hardware.** The ~10× HBM:DRAM and ~5–7× DRAM:PCIe gaps are architectural constants that shape every staging decision above them — GPUDirect, NVLink, and even MIG's NUMA localization inside the GPU package exist *because* of this gap, not as independent features.

**Economics.** Bandwidth loss from mis-pinning or under-populated channels is *pure waste* — it costs nothing to detect (a benchmark) and nothing to fix (a `numactl`/`cudaHostAlloc` flag or a provisioning-spec change), which makes it one of the highest-ROI findings a platform engineer can bring to a cost review. Contrast this with, say, a genuine hardware upgrade: the fix here is a config change, not a purchase order.

## Real-world use cases

- **NVIDIA Developer Blog — "Accelerating Data Processing with NVIDIA Multi-Instance GPU and NUMA Node Localization"** ([developer.nvidia.com](https://developer.nvidia.com/blog/accelerating-data-processing-with-nvidia-multi-instance-gpu-and-numa-node-localization)) — the same source as lesson 01, read here for its bandwidth angle: NUMA-node-localized MIG configuration on a memory-bandwidth-bound kernel (a lattice-QCD stencil) shows a large speedup at a fixed power limit purely from correct memory localization, with no change to the compute itself.
- **Ronak Nathani — "Keeping GPU Workloads NUMA-Local in Kubernetes"** ([ronaknathani.com](https://ronaknathani.com/blog/2026/05/keeping-gpu-workloads-numa-local-in-kubernetes/)) — the module's required deep read. Cites PyTorch's own performance-tuning guidance to bind training processes to a single NUMA node, and reports markedly higher p99 tail latency for inference pods whose CPUs spanned both sockets versus pods pinned to one socket. What it shows: a concrete, production-measured cost for exactly the "wrong-node buffer" problem this lesson quantifies with `bandwidthTest`.
- **Meta Engineering — "How Meta trains large language models at scale"** ([engineering.fb.com](https://engineering.fb.com/2024/06/12/data-infrastructure/training-large-language-models-at-scale-meta/)) — covers Meta's training infrastructure including data-loading and storage-to-GPU paths at scale. What it shows: the staging-pipeline discussion in this lesson generalizes to production LLM training infrastructure, not just a single-node benchmark exercise.

## Worked example

**Goal.** Measure the local-vs-remote host→device bandwidth delta for one GPU and record it for the deliverable.

1. **Find the GPU's home node** (from lesson 01):
   ```
   $ nvidia-smi topo -m | grep GPU0
   GPU0 ... NUMA Affinity: 0
   ```
   GPU0 homes on node 0. On this 2-socket EPYC box, node 3 is the far socket.

2. **Local copy — buffer on the GPU's node:**
   ```
   $ numactl --cpunodebind=0 --membind=0 ./bandwidthTest --memory=pinned --dtoh --htod
   Host to Device Bandwidth, Pinned: 54210.3 MB/s
   Device to Host Bandwidth, Pinned: 51880.7 MB/s
   ```
   ~54 GB/s HtoD — near the PCIe Gen5 x16 practical ceiling. This is the target.

3. **Remote copy — same GPU, buffer forced onto the far socket:**
   ```
   $ numactl --cpunodebind=3 --membind=3 ./bandwidthTest --memory=pinned --dtoh --htod
   Host to Device Bandwidth, Pinned: 31940.6 MB/s
   Device to Host Bandwidth, Pinned: 30110.2 MB/s
   ```
   ~32 GB/s — a **41% drop** with zero change to the GPU or the copy size. The DMA is now dragging every line across the inter-socket link before it reaches PCIe. This is the number that never shows on a dashboard.

4. **Confirm placement, not luck:**
   ```
   $ numactl --cpunodebind=3 --membind=3 numastat -p $(pgrep bandwidthTest)
   Node 3: 4096 MB   Node 0: 2 MB
   ```
   The buffer really is on node 3. The 41% gap is the UPI/Infinity Fabric tax, reproducible.

5. **Cross-check the channel story.** Full-node STREAM Triad under `numactl --membind=0` reads ~290 GB/s; the vendor spec for 8×DDR5-4800 is ~307 GB/s — channels are fully populated, so the remote gap is purely placement, not a half-built box. Had Triad read ~150 GB/s, you'd suspect half-populated channels *on top of* the placement issue. **Record: local 54 GB/s, remote 32 GB/s, delta 41%, STREAM-local 290 GB/s** for the Topology Teardown.

6. **The Unified Memory equivalent, one line.** If the codebase uses `cudaMallocManaged` instead of explicit `cudaHostAlloc` buffers, the same pin is expressed as:
   ```c
   cudaMemAdvise(ptr, size, cudaMemAdviseSetPreferredLocation, /* host NUMA node */ 0);
   ```
   Recognize this as the modern-code-path equivalent of the `numactl --membind` trick above — the lever is the same, the API surface is different.

### Scaling the rule to an 8-GPU box
Placement is *per-GPU*: in an 8-GPU, 2-socket node, GPUs 0–3 typically home on socket 0 and 4–7 on socket 1. A dataloader that allocates one big pinned pool on node 0 feeds GPUs 0–3 on the fast path and GPUs 4–7 across the inter-socket link — so half your GPUs silently run the remote number. The correct pattern is *per-GPU pinned buffers bound to that GPU's home node* (one loader worker pinned per node, or `cudaHostAlloc` from a thread already `numactl`-bound to the GPU's node — remember, allocation-time thread affinity is what decides this). Frameworks that shard by `local_rank` and pin per rank get this for free; a naive single-pool loader does not. This is the exact bug the deliverable is meant to catch and quantify.

## Practice

On a rented single-GPU (or multi-GPU) instance, or a multi-socket CPU box for the fallback:
1. Build/obtain `bandwidthTest` (`cuda-samples`) — or STREAM if no GPU.
2. Identify the GPU's home NUMA node (`nvidia-smi topo -m`).
3. Run the pinned HtoD/DtoH copy under `numactl --cpunodebind=<home> --membind=<home>`, then again under `--membind=<far node>`. For the fallback, run STREAM Triad `--membind=local` vs `--membind=remote`.
4. Verify placement with `numastat -p <pid>`.
5. Optionally run a full-node STREAM Triad and compare to the vendor's channels × 38.4 GB/s figure to check channel population.
6. If the box has IOMMU enabled, check its mode (`dmesg | grep -i iommu`) and note whether it's in strict/translate mode — a candidate second-bounce explanation if your measured numbers come in lower than the placement arithmetic alone predicts.

**Acceptance:** a recorded **local-vs-remote bandwidth pair and the percentage delta** (e.g. "54 vs 32 GB/s, −41%"), plus the `numastat` proof of buffer placement, saved for the deliverable. One sentence stating whether channels are fully populated, backed by the STREAM number.

## Common pitfalls

1. **"`pin_memory=True` in PyTorch means I'm safe."** It ensures page-locking, not correct NUMA placement — the pinned pool's node is still first-touch-determined by whichever thread allocated it, at allocation time, not at steady state.
2. **Trusting `bandwidthTest`'s absolute number as a training-throughput predictor.** It's a genuinely common mistake to dismiss a real placement bug because "the copy is overlapped anyway" — overlap hides latency, it doesn't undo a halved bandwidth ceiling. Use the local-vs-remote *ratio*, not the absolute number, as your evidence.
3. **Assuming identical DIMM capacity means identical bandwidth.** 4×64 GB and 8×32 GB give the same 256 GB but very different bandwidth on an 8-channel socket — and this is a **provisioning-time**, not runtime, defect that inventory and asset-tracking systems will never flag.
4. **Ignoring IOMMU/SWIOTLB as a second, independent bounce path.** A pinned, correctly-NUMA-placed buffer can still be slower than expected if IOMMU strict mode forces a `SWIOTLB` bounce — check `dmesg`/IOMMU mode as the next hypothesis when the "obvious" fix doesn't fully close the gap.
5. **Reading STREAM Triad numbers without confirming array size ≫ LLC.** If the arrays fit in cache, you're measuring cache bandwidth, not DRAM bandwidth — a very real source of "my numbers look too good to be true" confusion.

## Self-check

- Why does a pinned host buffer's NUMA node change GPU copy speed even though the GPU never touches DRAM directly? **Answer:** The GPU DMAs the buffer over PCIe from its home socket's root complex. If the buffer is on the GPU's home node, the copy is limited only by PCIe (~55 GB/s Gen5 x16). If it's on the other socket, the DMA must pull the data across the inter-socket link first, throttling the copy to ~50–70% of local. The buffer's node is fixed by first-touch at *allocation* time by the calling thread, not by anything the GPU does.
- Name two distinct mechanisms that can bounce a DMA transfer through an intermediate buffer even when the target memory is pinned. **Answer:** (1) CUDA's own internal bounce buffer, used when the *source* is pageable rather than pinned memory; (2) a kernel-level `SWIOTLB` bounce, triggered when the IOMMU is in strict/translate mode without a fast passthrough path — independent of whether CUDA-side pinning is correct.
- Why is Triad, not Copy, the honest number to quote from a STREAM run? **Answer:** Copy is a pure write, and a plain store to a cache line the CPU doesn't own triggers a read-for-ownership (RFO) under MESI before the write completes — inflating or distorting the apparent cost unless non-temporal stores are used. Triad's read-heavy mix (2 reads, 1 write) is more representative of real streaming bandwidth and is the number the STREAM benchmark's own documentation recommends citing.
- You have 256 GB on an 8-channel socket as 4×64 GB DIMMs — what bandwidth fraction do you expect versus 8×32 GB, and why? **Answer:** Roughly half. Each populated channel is an independent parallel path to DRAM; with only 4 of 8 channels populated, the memory controller can only stripe traffic across 4 paths instead of 8, so measured bandwidth lands near 50% of the fully-populated figure at identical total capacity — invisible to any inventory system, visible only to a bandwidth benchmark.
- What is the approximate bandwidth ratio between GPU HBM and host DRAM, and why does that gap dictate the whole staging strategy (GPUDirect, NVLink)? **Answer:** Roughly 10×: H100 HBM3 runs ~3.35 TB/s versus ~300–460 GB/s per host socket. Because HBM so vastly outruns DRAM, the working set must reside in HBM rather than being streamed from DRAM per step; GPUDirect and NVLink exist specifically to keep the comparatively slow host-DRAM tier off the critical compute path.

## Connections & what's next

This lesson turned lesson 01's static tree into a measured cost — the same NUMA-affinity read now has a bandwidth number attached to it, and you have the tooling (`bandwidthTest`, STREAM, `numastat`) to reproduce it on any box. Lesson 03 (PCIe) picks up exactly where the bandwidth ceiling table here left off: it teaches you to read whether a link actually *trained* at the speed/width this lesson assumed, because every number above silently assumed a healthy PCIe link. Lesson 04's 8-GPU server architecture applies the per-GPU placement rule from this lesson's closing section at whole-node scale, and lesson 05's Kubernetes Memory Manager is the automated, cluster-scheduled version of the `numactl --membind` trick you just ran by hand.

## References & further reading

**Primary sources**
- NVIDIA CUDA C++ Best Practices Guide, pinned-memory / GPUDirect section — [docs.nvidia.com](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/#pinned-memory) — read for: the rationale for pinned buffers, async overlap, and why host-buffer placement gates host↔device throughput.

**Real-world engineering blogs**
- NVIDIA Developer Blog, "Accelerating Data Processing with NVIDIA Multi-Instance GPU and NUMA Node Localization" — [developer.nvidia.com](https://developer.nvidia.com/blog/accelerating-data-processing-with-nvidia-multi-instance-gpu-and-numa-node-localization) — what it shows: a large, measured speedup from NUMA-correct memory localization on a bandwidth-bound kernel.
- Ronak Nathani, "Keeping GPU Workloads NUMA-Local in Kubernetes" — [ronaknathani.com](https://ronaknathani.com/blog/2026/05/keeping-gpu-workloads-numa-local-in-kubernetes/) — what it shows: a production p99-latency cost for cross-socket pods, and the PyTorch tuning-guide recommendation this lesson's practice echoes.
- Meta Engineering, "How Meta trains large language models at scale" — [engineering.fb.com](https://engineering.fb.com/2024/06/12/data-infrastructure/training-large-language-models-at-scale-meta/) — what it shows: this lesson's staging-pipeline concerns at real production LLM-training scale.

**Deeper dives**
- Brendan Gregg, *Systems Performance*, 2nd ed. — [brendangregg.com](https://www.brendangregg.com/systems-performance-2nd-edition-book.html) — targeted skim: the memory-bandwidth and NUMA sections only, for methodology on using STREAM/`numastat` as evidence rather than guessing.
