---
lesson: "02b.2"
title: "Memory subsystem as placement consequence"
module: "02b"
concept: "Memory subsystem as placement consequence"
status: not-started
est_time: "3h"
artifacts: []
---

# 02b.2 · Memory subsystem as placement consequence

> **Concept.** Host memory bandwidth and latency are placement outcomes — which NUMA node a buffer lives on, and how many DDR channels are populated, decide how fast a GPU can be fed.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Why this matters
A GPU is only as fast as the path staging data into it, and that path bottoms out at host DRAM bandwidth — which you can silently halve by binding a pinned buffer to the wrong NUMA node or by shipping the box with half its memory channels populated. Neither shows up on a GPU dashboard: the accelerator reports high utilization while starved. The interview question that lands is "your H100 copy benchmark hits 28 GB/s instead of 55 — walk me through why," and the answer is almost always host-side placement or channel population, not the GPU. This lesson makes that failure measurable and quotable.

## What's new for you
From the Linux-internals module you already have local-vs-remote memory as a *kernel* concept and the cache hierarchy — none of that is re-taught here. On-prem you have sized DIMMs and known that "more channels = more bandwidth" as folklore.

The delta is treating **bandwidth and latency as placement consequences for an accelerator**, quantified:
- **Local vs remote DRAM bandwidth as a hard number**, not a distance code — and why a *pinned host buffer's* NUMA node changes GPU copy speed even though the GPU never touches DRAM directly.
- **DDR channel population as a bandwidth multiplier** you can lose at provisioning time — the difference between a 1-DIMM-per-channel full build and a half-populated box is not subtle.
- **Host DRAM vs GPU HBM as two tiers with a ~10× bandwidth gap**, which is *why* staging strategy exists — you don't feed HBM from DRAM the naive way and expect it to keep up.

## Core notes

### Bandwidth vs latency, as placement
- **Latency** (~80–200 ns) dominates pointer-chasing and small, dependent accesses. Remote adds a fixed link-hop penalty.
- **Bandwidth** (streaming GB/s) dominates data staging — bulk host→device copies, dataloader→pinned-buffer memcpy. This is the accelerator-relevant axis.
Local DRAM bandwidth scales with **populated channels on that node**. Remote bandwidth is capped by the inter-socket link (lesson 01). So a remote buffer loses on both axes, but the *bandwidth* loss is what starves a GPU copy.

### Why a pinned host buffer's NUMA node changes GPU copy speed
The GPU never reads host DRAM with load instructions — it **DMAs** over PCIe. The DMA engine sits behind the GPU's PCIe root complex on the GPU's home socket. When it copies from a **pinned** (page-locked) host buffer:
- **Buffer on the GPU's home node:** DMA target DRAM → root complex → GPU, no inter-socket hop. Bandwidth is limited by PCIe (Gen5 x16 ≈ 63 GB/s theoretical, ~50–55 GB/s achievable pinned).
- **Buffer on the *other* socket:** DMA must pull those cache lines across UPI/Infinity Fabric first, *then* over PCIe. The copy is now throttled by the inter-socket link, commonly landing at **50–70% of the local number** — the exact gap you'll measure below.
Two more reasons pinning matters at all: pageable memory forces CUDA to stage through an internal pinned bounce buffer (extra copy, lower BW), and only pinned memory allows true async overlap. But *given* pinned memory, the NUMA node is the remaining lever — and it's the one nobody watches. `cudaHostAlloc` allocates on whatever node the calling thread first-touches, so an unpinned loader thread on the wrong socket poisons the buffer's placement.

### DDR channels — why population is bandwidth
A memory controller exposes multiple **channels**; each populated channel is an independent, parallel path to DRAM. Per-socket bandwidth ≈ channels × per-channel rate:
- **DDR5-4800**: 4800 MT/s × 8 B ≈ **38.4 GB/s per channel**.
- **Sapphire/Emerald Rapids Xeon**: **8 channels/socket** → ~307 GB/s/socket fully populated.
- **AMD EPYC Genoa/Turin**: **12 channels/socket** → ~460 GB/s/socket at DDR5-4800 (higher with faster DIMMs).
Populate half the channels (e.g. 4 DIMMs on an 8-channel socket) and you get **~half the bandwidth** — the controller can only parallelize across populated channels. This is a provisioning-time footgun: a box shipped 8×32 GB on an 8-channel socket is full-bandwidth; the same 256 GB as 4×64 GB is half-bandwidth for identical capacity. It never appears in inventory ("256 GB, check") and never appears on a GPU dashboard — only a bandwidth benchmark reveals it. Also relevant: exceeding 1 DIMM-per-channel (2DPC) often drops the DIMMs to a lower speed grade, trading a little bandwidth for capacity.

### Host DRAM vs GPU HBM — the ~10× gap that shapes staging
| Tier | Bandwidth | Capacity | Role |
|------|-----------|----------|------|
| Host DRAM (per socket, DDR5) | ~300–460 GB/s | 100s of GB–TBs | staging, dataloader, spill |
| GPU HBM (H100 SXM, HBM3) | ~3.35 TB/s | 80 GB | compute working set |
| GPU HBM (H200, HBM3e) | ~4.8 TB/s | 141 GB | " |
| GPU HBM (B200, HBM3e) | ~8 TB/s | 192 GB | " |
| PCIe Gen5 x16 (the bridge between them) | ~63 GB/s | — | host↔device transfer |
HBM out-runs a full socket's DRAM by roughly **10×**, and the PCIe bridge between them is ~5–7× slower still than DRAM. The consequences you must internalize: (1) you cannot stream from DRAM and keep HBM busy — the working set must **resident** in HBM, with PCIe used for prefetch/overlap, not per-step feeding; (2) because PCIe is the scarcest link, wasting any of it on a needless UPI detour (wrong-node buffer) is the most expensive placement mistake on the box; (3) this is why NVLink/NVSwitch (GPU↔GPU at 900 GB/s on H100) and GPUDirect (NIC/NVMe DMA straight into HBM, bypassing host DRAM) exist — to keep the slow host tier off the critical path.

### Why GPUDirect exists (and what it removes from the tree)
The default staging path for network or storage data into a GPU is: NIC/NVMe → host DRAM → PCIe → HBM. That routes everything through the slowest tier (host DRAM) and burns PCIe twice. **GPUDirect RDMA** lets the NIC DMA straight into HBM, and **GPUDirect Storage** lets NVMe DMA straight into HBM — both bypassing host DRAM and the CPU. The catch: it only works when the NIC/NVMe and the GPU share a PCIe path *without* crossing the inter-socket link. A GPU whose RDMA NIC is on the far socket cannot do a clean GPUDirect transfer — the DMA falls back through host memory or crosses UPI, quietly erasing the feature's benefit. So the placement tree from lesson 01 is a *precondition* for the memory-path optimizations, not an independent concern.

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
This is the module's whole point: the waste is real but off-dashboard. A 41% host→device bandwidth loss on a GPU whose per-step time is copy-bound translates directly into lower samples/sec at unchanged GPU-utilization %. If that GPU-hour costs (pick your neocloud rate) and the copy is on the critical path 30% of each step, you can state the loss as a throughput and dollar figure — a claim no `nvidia-smi`, DCGM, or Grafana panel makes for you, because they measure occupancy, not whether the occupied cycles were fed on the fast path. The engineer who ships that number ("we're leaving ~12% throughput on the table to a NUMA mis-bind, here's the benchmark") is the one who gets believed. Wire the local-vs-remote copy benchmark into node-bringup validation and you catch half-populated boxes and mis-affinitized NICs *before* they bill for a quarter.

### Reading a STREAM result (the GPU-free fallback)
STREAM reports four kernels; **Triad** (`a[i] = b[i] + q*c[i]`, 2 reads + 1 write) is the one to quote as sustained bandwidth:
```
Function    Best Rate MB/s
Copy:           288400.1
Scale:          287110.5
Add:            291020.7
Triad:          290560.3
```
~290 GB/s local on an 8-channel DDR5-4800 socket ≈ 94% of the ~307 GB/s spec — healthy population. Re-run under `numactl --membind=<remote>` and Triad drops toward the inter-socket-link ceiling. Two subtleties: STREAM must be built with arrays ≫ LLC (it warns if not) or you measure cache, not DRAM; and write-heavy kernels can look slower because a store first reads the line (read-for-ownership) unless the compiler emits non-temporal stores — so Triad, not Copy, is the honest streaming number.

### Why the benchmark under-reads real training (and still matters)
`bandwidthTest` measures a single synchronous stream; real training overlaps copy with compute across multiple streams and may exceed the single-stream figure, or hide it entirely behind compute. So the *absolute* number isn't your training throughput — but the **local-vs-remote ratio is**: whatever the workload's copy sensitivity, a wrong-node buffer scales its host-transfer cost by that same ~1.4–2×. Use the benchmark for the *delta*, not as a throughput prediction, and you avoid the trap of dismissing placement because "the copy is overlapped anyway" — overlap hides latency, not a halved bandwidth ceiling.

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
   ~32 GB/s — a **41% drop** with zero change to the GPU or the copy size. The DMA is now dragging every line across Infinity Fabric before it reaches PCIe. This is the number that never shows on a dashboard.

4. **Confirm placement, not luck:**
   ```
   $ numactl --cpunodebind=3 --membind=3 numastat -p $(pgrep bandwidthTest)
   Node 3: 4096 MB   Node 0: 2 MB
   ```
   The buffer really is on node 3. The 41% gap is the UPI/IF tax, reproducible.

5. **Cross-check the channel story.** Full-node STREAM Triad under `numactl --membind=0` reads ~290 GB/s; the vendor spec for 8×DDR5-4800 is ~307 GB/s — channels are fully populated, so the remote gap is purely placement, not a half-built box. Had Triad read ~150 GB/s, you'd suspect half-populated channels *on top of* the placement issue. **Record: local 54 GB/s, remote 32 GB/s, delta 41%, STREAM-local 290 GB/s** for the Topology Teardown.

### Scaling the rule to an 8-GPU box
Placement is *per-GPU*: in an 8-GPU, 2-socket node, GPUs 0–3 typically home on socket 0 and 4–7 on socket 1. A dataloader that allocates one big pinned pool on node 0 feeds GPUs 0–3 on the fast path and GPUs 4–7 across UPI — so half your GPUs silently run the remote number. The correct pattern is *per-GPU pinned buffers bound to that GPU's home node* (one loader worker pinned per node, or `cudaHostAlloc` from a thread already `numactl`-bound to the GPU's node). Frameworks that shard by `local_rank` and pin per rank get this for free; a naive single-pool loader does not. This is the exact bug the deliverable is meant to catch and quantify.

## Practice
On a rented single-GPU (or multi-GPU) instance, or a multi-socket CPU box for the fallback:
1. Build/obtain `bandwidthTest` (`cuda-samples`) — or STREAM if no GPU.
2. Identify the GPU's home NUMA node (`nvidia-smi topo -m`).
3. Run the pinned HtoD/DtoH copy under `numactl --cpunodebind=<home> --membind=<home>`, then again under `--membind=<far node>`. For the fallback, run STREAM Triad `--membind=local` vs `--membind=remote`.
4. Verify placement with `numastat -p <pid>`.
5. Optionally run a full-node STREAM Triad and compare to the vendor's channels × 38.4 GB/s figure to check channel population.

**Acceptance:** a recorded **local-vs-remote bandwidth pair and the percentage delta** (e.g. "54 vs 32 GB/s, −41%"), plus the `numastat` proof of buffer placement, saved for the deliverable. One sentence stating whether channels are fully populated, backed by the STREAM number.

## Self-check
**(a) What is the measured local-vs-remote memory-bandwidth gap on your box, and why does it exist?**
**Answer:** Report your own pair (typically a 30–50% drop, e.g. 54→32 GB/s). It exists because the remote buffer forces the GPU's DMA (or a CPU stream) to traverse the inter-socket link (UPI / Infinity Fabric) before reaching PCIe/the core; that shared link's single-stream ceiling is well below the local DRAM/PCIe path, and it also carries coherence traffic.

**(b) Why does a pinned host buffer's NUMA node change GPU copy speed?**
**Answer:** The GPU DMAs the buffer over PCIe from its home socket's root complex. If the buffer is on the GPU's home node, the copy is limited only by PCIe (~55 GB/s Gen5 x16). If it's on the other socket, the DMA must pull the data across the inter-socket link first, throttling the copy to ~50–70% of local. The GPU never load/stores DRAM directly, but its DMA source location decides the path.

**(c) Why does half-populating memory channels cost you bandwidth?**
**Answer:** Each populated channel is an independent parallel path to DRAM; per-socket bandwidth ≈ channels × per-channel rate (~38.4 GB/s at DDR5-4800). The controller can only stripe across populated channels, so 4-of-8 populated yields ~half the bandwidth of 8-of-8 — at identical capacity, and invisible in inventory. Only a bandwidth benchmark exposes it.

## Resources
1. **Frank Denneman — GPU host memory / multi-GPU topology writeups** — https://frankdenneman.ai/2026-03-27-Understanding-Multi-GPU-Topologies-Within-a-Single-Host/ — *deep.* Ties buffer placement and PCIe/NVLink paths to measured throughput; the placement-first framing this lesson extends into bandwidth numbers.
2. **Brendan Gregg — *Systems Performance*, 2nd ed., Memory chapter** — https://www.brendangregg.com/systems-performance-2nd-edition-book.html — *targeted skim.* For the methodology: NUMA effects, memory-bandwidth measurement, and using STREAM/`numastat` as evidence rather than guessing. Read the memory-bandwidth and NUMA sections only.
3. **NVIDIA CUDA `bandwidthTest` + pinned-memory / GPUDirect docs** — https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/#pinned-memory — *hands-on.* The tool and the rationale for pinned buffers, async overlap, and why host-buffer placement gates host↔device throughput.
