---
lesson: "02b.6"
title: "NVMe and storage placement for GPU data paths"
module: "02b"
concept: "NVMe and storage placement for GPU data paths"
status: not-started
est_time: "3h"
artifacts: []
---
# 02b.6 · NVMe and storage placement for GPU data paths
> **Concept.** Local NVMe that does not share the GPU's PCIe switch/root complex contends for the same PCIe path the GPU feeds on, and the stall shows up as "GPU idle" that no utilization metric explains.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Why this matters

You already know the GPU is the expensive part. What the utilization dashboard never tells you is that a GPU spends real wall-clock time waiting for the next batch to arrive from disk — and while it waits, `nvidia-smi` can still print `GPU-Util: 100%` because that counter reports "a kernel was resident in the last sample window," not "the SM array did useful work." A training step that is 30% data-loader-bound is 30% paid-for-but-undelivered compute. On an 8×H100 box at neocloud rates (~$2–3/GPU-hr), a persistent 20% data stall across a 512-GPU cluster is a low-six-figure annual line item that appears nowhere in your monitoring.

The root cause is almost always placement: the NVMe drive holding the dataset (or the read cache in front of an object store) sits on the *wrong* PCIe root complex, so every byte it reads crosses the inter-socket link (UPI/xGMI) or contends on the same switch the GPU is using for peer-to-peer and NIC traffic. The fix is free — it's a scheduling/placement decision, not new hardware. But you can only make it if you can read the topology. That is this lesson.

## What's new for you

You know PCIe lanes, root complexes, and NUMA from the earlier lessons in this module, and you've read `lspci` a hundred times for NIC and HBA troubleshooting. What is new:

- **You know:** PCIe topology, root complexes hang off sockets, NUMA locality matters for CPU memory. `lspci -tv`, `nvidia-smi`. Queue depth as a generic block-I/O concept.
- **What's new:** Treating *storage* as a first-class citizen on the GPU's PCIe path — the drive is not just "somewhere on the host," it is either on the GPU's switch, on the GPU's root complex, or across the socket, and those three cases have wildly different achievable bandwidth. Reading `nvidia-smi topo -m` for GPU↔NIC↔NVMe relationships (not just GPU↔GPU). The idea that a data-loader stall is a *topology* bug, not a code bug. And **GPUDirect Storage (GDS)** — a DMA path that lets the NVMe controller write straight into GPU HBM, removing the CPU bounce buffer entirely, with a hard placement requirement to actually pay off.

## Core notes

### The three placement cases

For a GPU reading training data off a local drive, the byte path is one of:

1. **Same PCIe switch.** NVMe and GPU both hang off the same PCIe switch chip (common in DGX/HGX-class boxes where a PLX/Broadcom switch fans out lanes). Traffic goes NVMe → switch → GPU without ever touching the CPU's root port or the inter-socket link. Lowest latency, full drive bandwidth, and the *only* topology where GPUDirect Storage reaches its peak. `nvidia-smi topo -m` prints `PIX` (single PCIe bridge) or `PXB` (multiple bridges) for this pair.

2. **Same root complex, different switch/port.** NVMe and GPU are under the same CPU socket's root complex but traverse the CPU's internal PCIe fabric to meet. Fine for host-staged I/O; adds the host root port as a shared contention point. `topo -m` shows `PHB` (traverses a PCIe Host Bridge).

3. **Cross-socket / cross-root-complex.** The NVMe is on socket 0's root complex, the GPU on socket 1's. Every read crosses the CPU-to-CPU interconnect — Intel **UPI** or AMD **Infinity Fabric (xGMI)** — which is shared with all cross-socket memory traffic and is bandwidth-limited (UPI is on the order of ~20 GB/s per link; a Gen5 x4 NVMe alone can push ~14 GB/s of reads, and a Gen5 x16 GPU wants ~64 GB/s). `topo -m` shows `SYS` (traverses the SMP interconnect between NUMA nodes). This is the case that silently caps your data loader and shows up as GPU idle.

The asymmetry is the whole point: in case 3 the drive's rated bandwidth is real, but you cannot deliver it to *that* GPU without stealing from every other cross-socket transfer on the box.

### Why a misplaced disk reads as "GPU idle"

The data-loader pipeline is: NVMe read → page cache / CPU DRAM (bounce buffer) → (decode/augment on CPU) → cudaMemcpy H2D → GPU HBM → step. If the NVMe read leg is throttled by a cross-socket hop, batches arrive late. The GPU finishes step *N*, has no batch *N+1* ready, and blocks. During that block:

- `nvidia-smi` utilization may still read high if you sample coarsely, or drop to a sawtooth if you sample fast — but averaged it looks "busy enough" that nobody investigates.
- SM occupancy / `DCGM` `SM_ACTIVE` and `TENSOR_ACTIVE` tell the true story: they sag on the same cadence as the stall. Most teams don't chart those.
- The trainer's own step-time histogram shows a bimodal distribution: fast steps (batch was ready) and slow steps (waited on I/O). That bimodality is the fingerprint.

So the observable symptom is: high-cost GPU, "looks utilized," step time worse than the FLOPs budget predicts, and a data-loader that is CPU/PCIe-starved. The tell that it's *placement* and not slow code: the same loader on a GPU that shares the drive's root complex runs fine.

### Queue depth and parallelism (briefly)

NVMe's advantage over SATA is many deep queues (up to 64K queues × 64K entries in spec; real drives expose dozens). A single-threaded, QD-1 reader will leave a Gen5 SSD 80% idle no matter where it sits — the drive needs multiple outstanding requests to keep its internal channels busy. So before you blame topology, confirm the loader is actually issuing parallel I/O: `num_workers` in a PyTorch `DataLoader`, `O_DIRECT` + `io_uring` or `libaio` with QD ≥ 16–32, or a prefetch depth ≥ 2. Topology sets the *ceiling*; queue depth determines whether you reach it. A shallow queue makes even a perfectly placed drive look slow, which is why you check both.

### GPUDirect Storage (GDS)

GDS removes the **CPU bounce buffer** from the read path. Without it: NVMe → system DRAM → GPU (two copies, CPU orchestrates each, DRAM bandwidth and a CPU core burned per stream). With GDS: the NVMe controller DMAs **directly into GPU HBM** over PCIe, coordinated by the `nvidia-fs` kernel driver and cuFile API — one copy, no CPU staging, no extra DRAM traffic.

What GDS removes:
- the extra copy through pinned host memory (the bounce buffer),
- the CPU cycles that drive that copy,
- the system-DRAM bandwidth that copy consumes.

What GDS **requires to actually help**: the NVMe and the target GPU must share a **PCIe switch** (or at minimum the same root complex) so the peer-to-peer DMA doesn't have to be brokered across the CPU root port or the inter-socket link. If the drive is cross-socket from the GPU, GDS either falls back to a staged/compatibility path or delivers a fraction of its potential — you've enabled the feature and gotten none of the win. This is why GDS is a *topology* feature as much as a software one: it turns your case-1 placement into a genuinely CPU-free path, and does little for case 3.

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

Conclusion for the deliverable note: *GPU0's local drive is NVMe0 (PXB, shared root complex, and NVMe0 shares a switch with NIC0 — GDS-ready). GPU0 reading NVMe1 is a cross-socket (`SYS`) path and must be avoided. Mirror for GPU1↔NVMe1.* That single mapping is what prevents the silent data stall.

## Practice

Feeds the **Topology Teardown** deliverable.

1. Run `lspci -tv` on the target host. Identify every NVMe controller and every GPU, and note which root complex (top-level bus, e.g. `[0000:c0]`) each hangs off. If you're on a laptop/dev box with no discrete GPU, run it against a cloud GPU instance or capture the output from a DGX/HGX node.
2. Run `nvidia-smi topo -m`. For **each GPU**, record the relationship (`PIX`/`PXB`/`PHB`/`SYS`) to every NVMe and to the NIC(s). Note the CPU Affinity and NUMA node columns.
3. Cross-reference: for each GPU, name its *local* NVMe (best relationship code) and flag any GPU→NVMe pairing that is `SYS` (cross-socket) or `PHB` — those are the data paths that will stall.
4. If GDS matters for your workload, check `PIX`/`PXB` between the intended GPU and drive, and (optionally) confirm the stack with `gdscheck -p` from the GDS tools.

**Acceptance:** a note in the Topology Teardown that states, for each GPU, where its storage sits (relationship code + which drive is local), and explicitly identifies any cross-socket/cross-root-complex data path on the box. One or two sentences per GPU plus the raw `topo -m` output pasted in.

## Self-check

**(a)** Is your NVMe on the GPU's root complex, and how do you tell?

**Answer:** Two independent checks. Structurally, `lspci -tv` shows both devices hanging off the same top-level host-bridge bus (e.g. both under `[0000:c0]`) — different top-level buses means different root complexes, typically different sockets. Functionally, `nvidia-smi topo -m` prints the relationship: `PIX` (same switch) or `PXB`/`PHB` (same host/root complex, more hops) means same root complex; `SYS` means the path crosses the SMP/NUMA interconnect to a *different* root complex. `SYS` is the answer you don't want for a data drive.

**(b)** What production symptom does a misplaced data disk produce, and why does it look like "GPU idle"?

**Answer:** The GPU periodically stalls waiting for the next batch because reads cross the UPI/xGMI link and can't be delivered at full drive bandwidth. Step times become bimodal (fast when the batch was prefetched, slow when it waited on I/O), and measured throughput falls short of the FLOPs budget. It reads as "GPU idle" because during the wait the SM array does no work — `DCGM SM_ACTIVE`/`TENSOR_ACTIVE` sag — even though coarse `nvidia-smi` utilization can still show high (it reports kernel residency, not useful work). The cost story: you pay full GPU rate for wall-clock time the accelerator spends blocked on a fixable placement bug.

**(c)** What does GPUDirect Storage remove from the data path, and what placement does it require?

**Answer:** GDS removes the CPU bounce buffer — the staging copy through pinned system DRAM — letting the NVMe controller DMA directly into GPU HBM via `nvidia-fs`/cuFile. That eliminates one copy, the CPU cycles driving it, and the system-DRAM bandwidth it consumed. To actually pay off it requires the NVMe and target GPU to share a PCIe switch (`PIX`/`PXB`), ideally the same root complex; if they're cross-socket (`SYS`) the peer-to-peer DMA is brokered across the CPU root port / inter-socket link and GDS falls back or delivers a fraction of its benefit.

## Resources

1. **NVIDIA GPUDirect Storage — Configuration & Overview** — the authoritative source on the DMA path, `nvidia-fs`, cuFile, and `gdscheck`. https://docs.nvidia.com/gpudirect-storage/configuration-guide/index.html
2. **`lspci(8)` man page** — the `-t` (tree) and `-v`/`-vvv` flags for reading root complexes and bridges; pair with `nvidia-smi topo -m`. https://man7.org/linux/man-pages/man8/lspci.8.html
3. **A data-loader-bottleneck writeup** — e.g. PyTorch's `DataLoader` performance/prefetch guidance (`num_workers`, `pin_memory`, prefetch factor) to distinguish a shallow-queue software stall from a topology stall before you blame placement. https://pytorch.org/docs/stable/data.html
