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
sources: 12
---

# 02b.2 · Memory subsystem as placement consequence

> **Concept.** Host memory bandwidth and latency are placement outcomes — which NUMA node a buffer lives on, and how many DDR channels are populated, decide how fast a GPU can be fed.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Where this fits

Lesson 01 gave you the static tree and, at the end, a table of link ceilings: 450 GB/s over NVLink, 63 GB/s over PCIe Gen5 x16, ~38 GB/s over one UPI link, 307.2 GB/s to local DRAM. Those were *asserted* — computed from lane counts and clock rates, with a promise that this lesson would show where they come from and what fraction of them you actually get.

That is the job here. This lesson opens up the memory tier: how DRAM bandwidth is physically manufactured out of channels and banks, why populating half your DIMM slots halves it, how HBM gets an order of magnitude more, and — the part that matters operationally — why a *pinned host buffer's NUMA node* changes GPU copy throughput even though the GPU never executes a load instruction against host DRAM. Everything here assumes you can already produce the tree from lesson 01. If you cannot name a GPU's home NUMA node from `nvidia-smi topo -m` without hesitating, that lesson is not done.

## Why this matters

A GPU is a memory-bandwidth machine wearing a compute badge. The tensor cores are fast enough that on most real workloads the binding constraint is how quickly bytes arrive, not how quickly they are multiplied. That makes the entire supply chain — NVMe → host DRAM → PCIe → HBM → SM — a bandwidth problem at every hop, and the hop you are most likely to break by accident is the one in the middle.

You can silently halve host→device throughput two ways, and neither is visible from any GPU-side metric:

- **Bind a pinned buffer to the wrong NUMA node.** The DMA engine now drags every cache line across the inter-socket link before it reaches PCIe. Nothing errors. `nvidia-smi` reports the same utilisation. The copy just takes longer.
- **Ship the box with half its memory channels populated.** Same capacity on the inventory sheet, roughly half the bandwidth. This one is decided at purchase order time, by someone who was optimising DIMM cost, and it persists for the life of the machine.

The interview question that lands is *"your H100 copy benchmark hits 28 GB/s instead of 55 — walk me through why,"* and the answer is almost always host-side placement or channel population, not the GPU. This lesson makes that failure measurable and quotable — the kind of finding ("we're leaving 41% of host→device bandwidth to a NUMA mis-bind, here is the benchmark and here is the `numastat` proof") that gets a platform engineer believed in a room full of people staring at utilisation graphs that all read 100%.

## What's new here (calibration)

From the Linux-internals module you already have local-vs-remote memory as a *kernel* concept, first-touch placement, mempolicy modes, and the cache hierarchy. From on-prem work you have sized DIMMs and absorbed "more channels = more bandwidth" as folklore. None of that is re-taught.

The delta is treating **bandwidth and latency as placement consequences for an accelerator**, with the mechanisms named and the arithmetic shown:

- **How a channel actually produces bandwidth** — DDR5's split into two independent 32-bit sub-channels, burst length 16, and why that combination delivers exactly one 64-byte cache line per burst. This is the level at which "channels multiply, ranks mostly don't" stops being folklore.
- **Little's Law applied to memory** — why remote DRAM loses *bandwidth* and not just latency, even when the inter-socket link is nowhere near saturated. This is the single most useful mechanism in the lesson and almost nobody has it.
- **Host DRAM vs GPU HBM as two tiers with a ~10× gap**, derived from stack counts and pin rates rather than quoted, and what that gap forces about staging strategy.
- **Why a pinned buffer's node changes copy speed** — the DMA path in detail, why page-locking is required at all, and the fact that placement is decided by the *allocating thread's* affinity at allocation time, not by anything the GPU does.
- **Two independent bounce paths** — CUDA's own pageable-memory staging buffer and the kernel's `SWIOTLB` — which are at different layers and can both be active.
- **The modern CUDA API for NUMA-placed pinned memory** (`cudaMemLocationTypeHostNuma`), which replaces the `numactl`-wrapping trick with something a framework can call.

## Core concepts

### 1. The problem: the GPU's supply chain, tier by tier

Before any placement argument, get the shape of the pipeline in your head. Every byte a training step consumes travels some prefix of this path:

```
        THE SUPPLY CHAIN — capacity, bandwidth, and latency per tier
        (H100 SXM5 in a 2-socket Sapphire Rapids host; per-GPU figures)

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ TIER                     CAPACITY        BANDWIDTH          LATENCY          │
  ├──────────────────────────────────────────────────────────────────────────────┤
  │ SM registers             256 KB/SM       ~   100 TB/s       ~ 1 cycle        │
  │ Shared mem / L1          228 KB/SM       ~    30 TB/s       ~ 30 cycles      │
  │ L2 cache                  50 MB          ~    10 TB/s       ~200 cycles      │
  │                            │                                                 │
  │ ══════════ the GPU package boundary ══════════                              │
  │                            │                                                 │
  │ HBM3 (5 stacks)           80 GB          3 350 GB/s         ~500-700 ns      │
  │                            │                                                 │
  │ ══════════ PCIe Gen5 x16 — THE NARROWEST LINK IN THE CHAIN ══════════       │
  │                            │                63 GB/s          ~1-2 µs         │
  │                            │             (~50-55 achieved)   (DMA setup)     │
  │ ══════════ host coherent domain ══════════                                  │
  │                            │                                                 │
  │ Local DRAM (8ch DDR5-4800) 1 TB/socket     307 GB/s          ~ 80 ns         │
  │ Remote DRAM (via UPI)      1 TB            ~38 GB/s/link     ~140 ns         │
  │                            │                                                 │
  │ Local NVMe (Gen5 x4)      30 TB         ~14 GB/s/drive       ~80 µs          │
  │ Network (ConnectX-7)       —               50 GB/s           ~2-3 µs RTT     │
  └──────────────────────────────────────────────────────────────────────────────┘

   RATIOS THAT DECIDE EVERYTHING
     HBM : local DRAM            3350 : 307      ≈ 11 : 1
     local DRAM : PCIe Gen5 x16   307 :  63      ≈  4.9 : 1
     HBM : PCIe Gen5 x16         3350 :  63      ≈ 53 : 1
     local DRAM : one UPI link    307 :  38      ≈  8 : 1
     PCIe Gen5 x16 : one UPI link  63 :  38      ≈  1.7 : 1   ← why a wrong-node
                                                                buffer costs ~40%
```

*(HBM and PCIe figures from NVIDIA H100 specifications and PCIe Gen5 lane arithmetic; DRAM from channel arithmetic in §3; latencies are representative and vary with DIMM speed, rank count and load — measure yours.)*

Three readings that should already change how you think:

**HBM out-runs a whole socket's DRAM by about 11×.** You cannot stream a working set from host DRAM and keep an H100 busy — not even close, and not even before PCIe gets involved. The working set has to *reside* in HBM. PCIe is for prefetch and overlap, not for per-step feeding.

**PCIe Gen5 x16 is the narrowest link in the chain, at 63 GB/s.** It is 53× narrower than HBM and 4.9× narrower than local DRAM. Everything crossing it is precious, which is exactly why wasting any of it on a needless inter-socket detour is the most expensive placement mistake available on the box.

**One UPI link is only 1.7× narrower than PCIe Gen5 x16.** That ratio is why a wrong-node buffer costs roughly 40% rather than 90%: the DMA is not blocked, it is throttled to the narrower of the two links in series. If you have ever wondered why the observed penalty is a suspiciously consistent "about 40%", this is the arithmetic behind it.

### 2. Bandwidth, latency, and the law that connects them

Everyone knows latency and bandwidth are different. Almost nobody can say why *raising latency lowers bandwidth* on a system where the link is nowhere near full. The answer is Little's Law, and it is the highest-leverage idea in this lesson.

For any queueing system in steady state:

```
  concurrency  =  throughput  ×  latency
      L        =       λ      ×      W
```

Applied to memory: the number of memory requests a core can have *in flight* is fixed by hardware — the line-fill buffers / miss-status holding registers (MSHRs) per core, typically 10–16 outstanding cache-line misses on current x86 cores. So:

```
  achievable bandwidth  =  outstanding_requests × 64 B  ÷  latency

  One core, 12 outstanding misses, LOCAL DRAM at 80 ns:
     12 × 64 B ÷ 80e-9 s  =  768 B ÷ 80e-9  =  9.6 GB/s

  Same core, same 12 outstanding misses, REMOTE DRAM at 140 ns:
     12 × 64 B ÷ 140e-9 s =  768 B ÷ 140e-9 =  5.5 GB/s

  Ratio: 5.5 / 9.6 = 0.57  →  a 43% bandwidth loss from a 1.75× latency increase,
  with the inter-socket link at a small fraction of its capacity the whole time.
```

**That is the mechanism people miss.** The remote penalty is not primarily "the UPI link is congested." It is that *each individual requester* can keep fewer bytes in flight per unit time when each request takes longer to complete, and there is a hard hardware limit on how many requests it can have outstanding at once. You need ~1.75× as many concurrent requesters to recover the same bandwidth, and a dataloader thread cannot conjure extra MSHRs.

Three corollaries worth carrying:

- **Prefetching and streaming instructions help by raising concurrency**, not by lowering latency. That is why `MOVNTDQ`-style non-temporal stores and hardware prefetchers matter so much for memcpy-shaped work.
- **DMA engines are better at this than cores**, because they can have far more outstanding transactions. That is why a GPU's copy engine reading remote DRAM loses ~40% rather than the ~43% a single core loses — but it still loses, because the DMA engine's own outstanding-request limit (governed in part by PCIe `MaxReadRequest`, lesson 03) is also finite.
- **Under load the penalty compounds.** Queueing at the far memory controller raises W further, which lowers λ further. The measured penalty at 90% load is always worse than the idle-latency ratio predicts.

### 3. How DRAM bandwidth is actually manufactured

"More channels = more bandwidth" is true, but you cannot reason about half-populated boxes, 2DPC derating or sub-channels without the layer below it.

**The hierarchy inside the memory system, outermost to innermost:**

```
   MEMORY CONTROLLER (on the CPU die, one per group of channels)
      │
      ├── CHANNEL — an independent 64-bit data bus + its own command/address bus.
      │      Channels are the parallelism that matters. Two channels do two
      │      unrelated transfers at the same time.
      │      │
      │      ├── SUB-CHANNEL (DDR5 only) — the 64-bit channel is split into
      │      │      TWO independent 32-bit sub-channels (40-bit with ECC:
      │      │      32 data + 8 ECC), each with its own command/address bus.
      │      │      │
      │      │      └── RANK — one set of DRAM dies that respond together.
      │      │             Ranks SHARE the bus; they do not add bandwidth.
      │      │             They add capacity and a little bank-level parallelism.
      │      │             │
      │      │             └── BANK GROUP → BANK → ROW → COLUMN
      │      │                    A bank has one open row at a time. Hitting an
      │      │                    open row is fast; switching rows costs tRP+tRCD.
      │      │                    Banks are why sequential access beats random
      │      │                    even inside DRAM.

   THE DDR5 BURST, AND WHY THE SPLIT IS FREE PERFORMANCE

     DDR4:  one 64-bit channel × burst length 8   = 64 × 8  ÷ 8 = 64 bytes
            → one cache line per burst, but the WHOLE channel is committed to it.

     DDR5:  one 32-bit sub-channel × burst length 16 = 32 × 16 ÷ 8 = 64 bytes
            → one cache line per burst per sub-channel, and there are TWO
              sub-channels per DIMM, each with independent commands.
            → same 64 bits of data pins, TWO independent cache-line transfers
              in flight instead of one.

     Same peak bytes/second. Better utilisation, because two shorter,
     independent bursts tolerate bank conflicts and refresh far better than
     one long one. This is why BIOS and monitoring tools sometimes report an
     8-channel socket as "16 sub-channels" — same copper, finer grain.
```

**The peak-bandwidth formula**, which you should be able to write out cold:

```
  per-channel bandwidth (GB/s) = transfer_rate (MT/s) × bus_width (bytes)

  DDR5-4800, 64-bit channel:   4800 × 8 B = 38.4 GB/s
  DDR5-5600:                   5600 × 8 B = 44.8 GB/s
  DDR5-6400:                   6400 × 8 B = 51.2 GB/s
  MRDIMM-8800:                 8800 × 8 B = 70.4 GB/s

  per-socket = per-channel × populated_channels
```

Note that the 8 bytes is the *channel* width, not the sub-channel width — the two DDR5 sub-channels each move 4 bytes per transfer and together move 8, so the formula is unchanged from DDR4. ECC bits ride alongside and are not counted as payload.

**Peak memory bandwidth by platform**, all derived from that one formula:

| Platform | Channels/socket | DIMM speed (1DPC) | Per-channel | **Per-socket peak** |
|---|---|---|---|---|
| Intel Sapphire Rapids (Xeon 4th gen) | 8 | DDR5-4800 | 38.4 GB/s | **307.2 GB/s** |
| Intel Emerald Rapids (Xeon 5th gen) | 8 | DDR5-5600 | 44.8 GB/s | **358.4 GB/s** |
| Intel Granite Rapids (Xeon 6, P-core) | 12 | DDR5-6400 | 51.2 GB/s | **614.4 GB/s** |
| Intel Granite Rapids with MRDIMM | 12 | MRDIMM-8800 | 70.4 GB/s | **844.8 GB/s** |
| AMD EPYC Genoa (9004) | 12 | DDR5-4800 | 38.4 GB/s | **460.8 GB/s** |
| AMD EPYC Turin (9005) | 12 | DDR5-6000 | 48.0 GB/s | **576.0 GB/s** |

*(Channel counts and supported DIMM speeds from Intel and AMD platform documentation; the GB/s columns are the arithmetic above. Actual supported speed depends on the specific SKU and on DIMM population — see §4.)*

### 4. Channel population: the bandwidth you can lose with a purchase order

**Bandwidth scales with *populated channels*, not with capacity.** The memory controller can only stripe traffic across the channels that have DRAM on them. A channel with no DIMM contributes nothing.

The consequence, stated as concretely as possible:

```
  Target: 256 GB per socket on an 8-channel Sapphire Rapids socket.

  Option A: 8 × 32 GB, one DIMM per channel
     populated channels = 8
     peak bandwidth     = 8 × 38.4 GB/s = 307.2 GB/s
     capacity           = 256 GB

  Option B: 4 × 64 GB, one DIMM per channel on four channels
     populated channels = 4
     peak bandwidth     = 4 × 38.4 GB/s = 153.6 GB/s     ← HALF
     capacity           = 256 GB                          ← IDENTICAL

  Every inventory system, every `free -g`, every Kubernetes node capacity
  report shows these as the same machine. Only a bandwidth benchmark
  distinguishes them.
```

Option B is not hypothetical or rare. It is what you get when a purchasing decision optimises for DIMM count (fewer, larger DIMMs are often cheaper per GB and leave slots free for future expansion) without anyone modelling the bandwidth consequence. On a GPU host it is a serious defect: it halves the rate at which the dataloader can assemble batches and halves the rate at which pinned staging buffers can be filled.

**Two more population rules that bite:**

- **2DPC derating.** Putting two DIMMs on one channel adds capacity but electrically loads the bus. Platforms respond by dropping the supported speed grade — on Sapphire Rapids, 2DPC configurations commonly drop from DDR5-4800 to DDR5-4400. So `8 channels × 2 DIMMs` at 4400 MT/s gives `8 × 35.2 = 281.6 GB/s` for double the capacity: an 8% bandwidth loss for 2× capacity. Sometimes that is the right trade; make it deliberately.
- **Asymmetric population disables sub-NUMA modes.** Both Intel SNC and AMD NPS require symmetric channel population across the domains they create. Populate 5 of 8 channels and the firmware will refuse SNC, or produce badly-sized domains. This is one of the sneakier ways a "capacity-optimised" DIMM plan destroys the locality features lesson 01 taught you to use.

**Verifying population from the OS**, before you trust anyone's build sheet:

```
$ sudo dmidecode -t memory | grep -E 'Locator|Size|Speed|Configured' | head -20
        Locator: CPU0_DIMM_A1
        Size: 64 GB
        Speed: 4800 MT/s
        Configured Memory Speed: 4400 MT/s      ← DERATED. 2DPC or a slow DIMM.
        Locator: CPU0_DIMM_A2
        Size: No Module Installed               ← empty slot
...
$ sudo dmidecode -t memory | grep -c 'Size: [0-9]'     # populated slots
16
$ sudo dmidecode -t memory | grep -c 'No Module Installed'
16
```

**Read `Configured Memory Speed`, not `Speed`.** `Speed` is what the DIMM is rated for; `Configured Memory Speed` is what the controller actually clocked it at. A gap between them means derating — 2DPC, a mixed-speed population, or a BIOS setting.

### 5. What you actually measure, and why it is below peak

Peak is `channels × rate × width`. Measured is always lower, for reasons worth naming because each one tells you something different when the gap is unusually large.

**STREAM** is the standard host-DRAM benchmark. It runs four kernels over three large arrays and reports the best sustained rate for each:

```
$ OMP_NUM_THREADS=56 numactl --cpunodebind=0 --membind=0 ./stream
-------------------------------------------------------------
Array size = 400000000 (elements), Offset = 0 (elements)
Memory per array = 3051.8 MiB (= 2.98 GiB).
Total memory required = 9155.3 MiB (= 8.94 GiB).
-------------------------------------------------------------
Function    Best Rate MB/s  Avg time     Min time     Max time
Copy:          255120.4     0.025241     0.025086     0.025498
Scale:         253880.1     0.025375     0.025209     0.025644
Add:           271044.8     0.035590     0.035418     0.035812
Triad:         270512.6     0.035671     0.035488     0.035901
```

*(Representative transcript for an 8-channel DDR5-4800 socket; your numbers will differ.)*

**Quote Triad.** `Triad` is `a[i] = b[i] + q*c[i]` — two reads and one write per element, which is the closest of the four to a realistic streaming mix. Here 270.5 GB/s against a 307.2 GB/s peak is **88% of peak**, which is a healthy, fully-populated 8-channel socket. If it read ~150 GB/s you would be looking at Option B from §4.

Why the missing 12%:

- **Read-for-ownership (RFO).** Under MESI, a plain store to a cache line the core does not already own must first *read* that line into cache so it can be modified coherently. So a "write" of 64 bytes costs 64 bytes of read traffic plus 64 bytes of eventual writeback — the DRAM sees up to 3 lines of traffic for what the program thinks is 2 reads and 1 write. This is why `Copy` (1 read, 1 write) reads *lower* than `Add` (2 reads, 1 write) despite moving less data: `Copy`'s ratio of RFO overhead to useful work is worse. Non-temporal stores (`MOVNTDQ`, `_mm_stream_*`) bypass RFO by writing straight to memory without allocating the line, which is why some benchmarks compiled with streaming-store flags report numbers that look impossibly close to (or above) the read-only peak.
- **Read/write turnaround.** The DDR bus is bidirectional; switching between read and write bursts costs idle cycles (`tWTR`, `tRTP`). A pure-read stream gets closer to peak than a mixed one.
- **Refresh.** DRAM rows must be refreshed; during refresh a bank is unavailable. This costs a small, steady percentage.
- **Bank conflicts and row misses.** Sequential access keeps rows open; random access pays `tRP + tRCD` on every miss.

**The rule of thumb to internalise: a healthy, fully-populated socket measures 80–90% of peak on STREAM Triad.** Below ~70% and you should suspect population, derating, or that your arrays fit in the LLC.

Two things that will make you measure the wrong thing:

1. **Arrays smaller than the last-level cache.** Sapphire Rapids has up to 105 MB of L3 per socket. STREAM's own guidance is arrays at least 4× LLC. If you measure 900 GB/s, you measured cache.
2. **Not pinning.** Without `numactl --cpunodebind=N --membind=N` the threads and pages smear across nodes and you measure an average of local and remote.

For the local-vs-remote *matrix*, Intel MLC is the better tool because it does the cross product for you:

```
$ sudo ./mlc --latency_matrix
                Numa node
Numa node            0       1
       0          81.4   139.7
       1         140.2    80.9

$ sudo ./mlc --bandwidth_matrix
                Numa node
Numa node            0       1
       0        272841.3   38104.6
       1         37982.1  273560.2
```

*(Representative; run it yourself.)* Read this against the SLIT: firmware said `21` vs `10`, i.e. 2.1×. Measured idle latency says 139.7/81.4 = **1.72×**. Measured bandwidth says 272841/38104 = **7.2×**. **The SLIT's single integer describes neither of the two things you care about.** Remote bandwidth here is capped near one UPI link's effective rate (§1), exactly as the arithmetic in lesson 01 predicted.

### 6. HBM: where the 11× comes from

HBM is not "faster DRAM." It is the same DRAM technology with a radically different interface, and understanding the difference tells you why the gap is structural rather than a matter of NVIDIA buying better parts.

A DDR5 channel is 64 bits wide, running at 4800 MT/s, over PCB traces to a socketed DIMM several centimetres away. Trace length and connector parasitics cap both the width (you cannot route thousands of traces across a motherboard) and the rate.

An **HBM stack** is a tower of DRAM dies bonded on top of a base logic die with **through-silicon vias**, sitting on a silicon interposer millimetres from the GPU die. Because the connection is inside the package, it can be **1024 bits wide per stack**. It runs *slower per pin* than DDR5 — but width wins overwhelmingly.

```
  per-stack bandwidth = interface_width (bits) × pin_rate (Gbps) ÷ 8

  H100 SXM5:  5 active HBM3 stacks × 1024 bits = 5120-bit bus
              3350 GB/s ÷ 5120 bits × 8 = 5.23 Gbps/pin
              per stack: 3350 ÷ 5 = 670 GB/s

  Compare one DDR5-4800 channel: 64 bits × 4.8 Gbps ÷ 8 = 38.4 GB/s
  One HBM3 stack ≈ 17.4 DDR5-4800 channels, at a similar pin rate.
  The entire advantage is 1024 bits vs 64 bits of interface width.
```

| GPU | HBM generation | Stacks × width | Capacity | **Bandwidth** | Derived pin rate |
|---|---|---|---|---|---|
| A100 80GB SXM | HBM2e | 5 × 1024 = 5120-bit | 80 GB | 2 039 GB/s | ~3.2 Gbps |
| H100 SXM5 | HBM3 | 5 × 1024 = 5120-bit | 80 GB | 3 350 GB/s | ~5.2 Gbps |
| H100 NVL / PCIe (94 GB) | HBM3 | 6 × 1024 = 6144-bit | 94 GB | 3 900 GB/s | ~5.1 Gbps |
| H200 SXM | HBM3e | 6 × 1024 = 6144-bit | 141 GB | 4 800 GB/s | ~6.3 Gbps |
| B200 | HBM3e | 8 × 1024 = 8192-bit | 180–192 GB | 8 000 GB/s | ~7.8 Gbps |

*(Capacities and bandwidths from NVIDIA datasheets; stack counts and pin rates derived from the published bus widths. B200 capacity was announced as 192 GB and appears as 180 GB in shipping datasheets — check the revision for the SKU you are quoting. A100 and H100 both ship with one stack disabled for yield, which is why 80 GB is 5 × 16 GB rather than 6.)*

The operational consequences of the 11× gap, stated as rules:

1. **The working set must reside in HBM.** You cannot stream it from host DRAM. At 307 GB/s host DRAM feeds an H100 at 9% of the rate its own memory does — and that is before PCIe cuts it to 63 GB/s.
2. **PCIe is for prefetch and overlap, not for feeding.** Anything on the per-step critical path that crosses PCIe is a 53× bandwidth cliff relative to HBM.
3. **This is *why* NVLink and GPUDirect exist.** NVLink (450 GB/s per direction on H100) keeps GPU↔GPU traffic off the host tier entirely. GPUDirect RDMA and GPUDirect Storage let a NIC or NVMe DMA straight into HBM, skipping host DRAM and the CPU. Both are engineering responses to the same gap.

### 7. Does it fit, and at what read bandwidth? — the arithmetic

Here is the calculation you will do most often in practice: given a model, does it fit, and if it does not, what does the alternative cost?

```
  WEIGHT FOOTPRINT
    bytes = parameters × bytes_per_parameter
      BF16 / FP16 : 2 B      FP8 : 1 B      INT4 : 0.5 B

    Llama-3-70B at BF16 : 70e9 × 2 B = 140 GB
    Llama-3-70B at FP8  : 70e9 × 1 B =  70 GB

  DOES IT FIT?
    H100 SXM5   80 GB : BF16 140 GB → NO.  FP8 70 GB → yes, 10 GB spare.
    H200 SXM   141 GB : BF16 140 GB → technically yes, ~1 GB spare — i.e. NO,
                        because KV cache, activations and the CUDA context
                        need room too. Budget ~10-15% overhead minimum.
    B200       180 GB : BF16 140 GB → yes, 40 GB for KV cache.

  IF IT FITS: how fast can one forward pass read the weights?
    Autoregressive decode is memory-bound: every token reads every weight once.
      t_min = weights_bytes ÷ HBM_bandwidth
      140 GB ÷ 3350 GB/s = 41.8 ms    (if it fit on an H100)
      140 GB ÷ 4800 GB/s = 29.2 ms    (H200)
      140 GB ÷ 8000 GB/s = 17.5 ms    (B200)
    → upper bound on decode rate = 1/t_min = 23.9 / 34.2 / 57.1 tokens/s
      for batch size 1. (Real systems batch, which amortises the weight read
      across many sequences — that is the entire economic argument for batching.)

  IF IT DOES NOT FIT: stream the weights from host DRAM over PCIe
      140 GB ÷ 55 GB/s (achieved Gen5 x16) = 2.55 s per forward pass
      → 0.39 tokens/s.  61× slower than the H100-resident case.

    And if the host buffer is on the WRONG NUMA NODE:
      140 GB ÷ 32 GB/s = 4.38 s per forward pass
      → 0.23 tokens/s.  A further 1.7× loss, from a placement decision.
```

**That 61× is the number that makes "does it fit" the first question in any capacity conversation**, and the further 1.7× is the number that makes NUMA placement the second one. Both are pure arithmetic over published bandwidths; neither requires a benchmark to argue.

Re-run it with your own inputs: the only quantities are parameter count, bytes per parameter, and the bandwidth of whichever tier the weights actually live in.

### 8. Why a pinned host buffer's NUMA node changes GPU copy speed

The GPU never issues a load instruction against host DRAM. It **DMAs**. So why does a host-side placement decision change GPU-visible throughput? Because the DMA engine is a device on the PCIe tree, it sits behind the GPU's root complex on the GPU's socket, and where it has to fetch from is a property of the buffer, not of the GPU.

**First, why pinning is required at all.** A DMA engine addresses *physical* memory and cannot take a page fault. Ordinary (pageable) userspace memory can be swapped out, migrated by compaction, or remapped at any moment — its physical address is not stable, so it is not a legal DMA target. Page-locking (`cudaHostAlloc`, `cudaMallocHost`, or `cudaHostRegister` on existing memory; `mlock`-class behaviour underneath) pins the pages so their physical frames cannot move for the lifetime of the registration. Given that, three things become possible that were not:

- The DMA engine can read the buffer directly, with no intermediate copy.
- The copy can be truly asynchronous (`cudaMemcpyAsync` on a stream), because the CPU is not needed to shepherd it.
- The transfer can use the GPU's copy engines concurrently with compute, which is what makes prefetch overlap work at all.

**If the source is pageable, CUDA bounces it.** The runtime maintains its own internal pinned staging buffer; a `cudaMemcpy` from pageable memory becomes *pageable → CUDA's pinned staging buffer → PCIe → device*. That extra CPU-driven memcpy is why pageable H2D typically measures around half of pinned H2D, and why it cannot overlap with compute. This is the first of the two bounce paths in this lesson.

**Now the placement question.** `cudaHostAlloc` does not choose a NUMA node. It maps pages; **the node is decided by first-touch, by whichever thread first writes to each page** — the same rule as any anonymous mapping, not a CUDA-specific one. So:

```
   THE TWO PATHS A BATCH TAKES — and where the node is decided

   t0  loader thread starts.  Scheduler happens to run it on a core of node 0.
   t1  thread calls cudaHostAlloc(&buf, 4 GiB, cudaHostAllocDefault)
         → virtual mapping created. NO physical pages yet. NO node chosen yet.
   t2  thread (or CUDA, during registration) first-touches the pages
         → EVERY page is allocated on NODE 0, permanently, and pinned there.
   t3  scheduler migrates the thread to node 3 (load balancing, or the app
       pins it later after reading nvidia-smi topo -m)
         → the THREAD moved. The PAGES DID NOT. They are pinned; migration
           and compaction cannot touch them. They are on node 0 forever.
   t4  every cudaMemcpyAsync from buf to GPU5 (home node 3) now runs:

   ┌──────────────────────────── ALIGNED (what you want) ────────────────────────┐
   │  buf on node 3, GPU5 on node 3                                              │
   │                                                                             │
   │   DRAM(node3) ──307 GB/s──▶ mem ctrl ──▶ IOMMU ──▶ root cplx ──▶ PCIe Gen5  │
   │                                                              63 GB/s ──▶ GPU5│
   │   bottleneck = PCIe.  measured pinned H2D ≈ 50-55 GB/s.                     │
   │   UPI crossings: 0                                                          │
   └─────────────────────────────────────────────────────────────────────────────┘

   ┌────────────────────────── MISALIGNED (the default) ─────────────────────────┐
   │  buf on node 0, GPU5 on node 3                                              │
   │                                                                             │
   │   DRAM(node0) ──▶ mem ctrl(sock0) ──UPI ~38 GB/s──▶ sock1 ──▶ root cplx     │
   │                                                        ──▶ PCIe ──▶ GPU5    │
   │   bottleneck = UPI, in SERIES with PCIe.                                    │
   │   measured pinned H2D ≈ 30-33 GB/s   (≈ 40% loss)                           │
   │   UPI crossings: 1 per byte, contending with every other cross-socket flow  │
   └─────────────────────────────────────────────────────────────────────────────┘

   NOTHING in this picture is visible to nvidia-smi, DCGM, or any Grafana panel.
   The only observable is that step time is longer.
```

**The trap in one sentence: the allocating thread's affinity at allocation time, not the using thread's affinity at steady state, decides the fate of every transfer that follows.** A dataloader that allocates one big pinned pool in `main()` before spawning workers puts the entire pool on whichever node `main()` happened to run on, and no amount of later pinning moves it.

**Three ways to control it, in increasing order of how modern the codebase is:**

```bash
# 1. Wrap the process. Blunt but universal; binds CPUs and memory together.
numactl --cpunodebind=3 --membind=3 python train.py --local-rank 5
```

```c
/* 2. Bind the allocating thread, then allocate. Works with any CUDA version. */
#include <numa.h>
struct bitmask *nm = numa_allocate_nodemask();
numa_bitmask_setbit(nm, 3);
numa_bind(nm);                       /* CPUs + memory policy for this thread */
cudaHostAlloc(&buf, bytes, cudaHostAllocDefault);
memset(buf, 0, bytes);               /* force first-touch NOW, on node 3 */
```

```c
/* 3. Ask CUDA directly. CUDA 12+ exposes host-NUMA as a memory location type,
      so a framework can place a pinned pool without shelling out to numactl. */
cudaMemPoolProps props = {0};
props.allocType     = cudaMemAllocationTypePinned;
props.location.type = cudaMemLocationTypeHostNuma;
props.location.id   = 3;                       /* the OS NUMA node id */
cudaMemPool_t pool;
cudaMemPoolCreate(&pool, &props);
cudaMallocFromPoolAsync(&buf, bytes, pool, stream);
```

Option 3 is the one to reach for in new code and the one to look for when auditing a framework: `cudaMemLocationTypeHostNuma` with a `location.id` set to the GPU's home node is the explicit, per-allocation statement of intent. It is also the API a device plugin or runtime can call on your behalf, which is where lesson 05 picks the thread back up.

**Verify, always.** Intent is not placement:

```
$ numastat -p $(pgrep -f 'train.py.*local-rank 5')

Per-node process memory usage (in MBs) for PID 48213
                           Node 0          Node 1          Node 2          Node 3           Total
                  --------------- --------------- --------------- --------------- ---------------
Huge                         0.00            0.00            0.00            0.00            0.00
Heap                        18.42            0.00            0.00           64.11           82.53
Stack                        0.09            0.00            0.00            0.31            0.40
Private                   4102.77            2.15            0.00          210.88         4315.80
                  --------------- --------------- --------------- --------------- ---------------
Total                     4121.28            2.15            0.00          275.30         4398.73
```

4.1 GB of `Private` on node 0 for a rank that is supposed to live on node 3 is the pinned pool, sitting on the wrong socket. That output *is* the evidence for the finding; a claim without it is an opinion.

### 9. The second bounce path: IOMMU and SWIOTLB

You can get pinning right, get NUMA placement right, and still measure below the arithmetic. The next hypothesis is a bounce that happens a layer below CUDA entirely.

**SWIOTLB** ("software I/O TLB") is the kernel's DMA bounce-buffer allocator. It reserves a chunk of physically contiguous low memory at boot — **64 MB by default**, adjustable with the `swiotlb=<slabs>` kernel parameter — and copies DMA payloads through it whenever a buffer cannot be handed to a device directly. Historically this served 32-bit-DMA-capable devices on 64-bit hosts; today it also serves confidential-computing memory encryption and untrusted-device scenarios where the IOMMU cannot or will not map the buffer directly.

When SWIOTLB is in the path, every DMA byte is copied by the CPU into the bounce region and then DMA'd out of it. That is the same structural cost as CUDA's pageable-memory bounce, at a different layer, and both can be active at once.

Diagnosis:

```
$ dmesg | grep -iE 'iommu|swiotlb'
[    0.000000] Command line: ... intel_iommu=on iommu=pt
[    0.512394] DMAR: IOMMU enabled
[    1.882910] software IO TLB: mapped [mem 0x00000000b3000000-0x00000000b7000000] (64MB)
[    2.104773] DMAR: Intel(R) Virtualization Technology for Directed I/O

$ cat /sys/kernel/debug/swiotlb/io_tlb_used     # slabs currently in use (debugfs)
0
```

`iommu=pt` (passthrough) installs an identity mapping for host-owned devices, so the IOMMU translates nothing for them and the fast path is restored. Without it, `intel_iommu=on` in strict mode adds an IOTLB invalidation on every unmap, which is a per-transfer cost that shows up as reduced small-transfer throughput rather than as a bounce. The kernel logging `software IO TLB: mapped ...` at boot is normal — it means the region was reserved, not that it is being used. `io_tlb_used` climbing, or `swiotlb buffer is full` messages in `dmesg`, means it is actually in the path.

**The two bounce paths, stated side by side so you do not confuse them:**

| | CUDA pageable bounce | Kernel SWIOTLB bounce |
|---|---|---|
| Layer | CUDA runtime, userspace | kernel DMA API |
| Trigger | source/destination is pageable, not pinned | IOMMU cannot map the buffer directly (address limits, encryption, untrusted device) |
| Fix | use `cudaHostAlloc` / `pin_memory=True` | `iommu=pt`, or fix the addressing constraint |
| Evidence | pinned vs pageable `bandwidthTest` gap | `dmesg` SWIOTLB messages, `io_tlb_used` > 0 |

### 10. GPUDirect: deleting the host tier from the path

Given §1's ratios, the obvious optimisation is to stop routing data through host DRAM at all. That is what the GPUDirect family does.

```
   DEFAULT PATH — network data into a GPU
     wire ─▶ NIC ─▶ PCIe ─▶ root complex ─▶ HOST DRAM ─▶ root complex ─▶ PCIe ─▶ HBM
                                             ▲                                    
                              two PCIe crossings, two DRAM crossings,
                              CPU involved in the handoff

   GPUDirect RDMA
     wire ─▶ NIC ─▶ PCIe switch crossbar ─▶ HBM
                                             
                              one PCIe hop. Host DRAM never touched.
                              CPU never involved.

   GPUDirect Storage (GDS)
     NVMe ─▶ PCIe switch crossbar ─▶ HBM        (same shape, storage instead of net)
```

The mechanism: the GPU exposes a window of its HBM through a BAR (PCIe base address register), and the peer-memory kernel module (`nvidia-peermem` for RDMA verbs) lets the NIC's driver register that BAR range as a legal DMA target. The NIC then writes incoming payload straight into HBM using ordinary PCIe memory writes routed by the switch.

**The precondition is topological, and it is exactly the tree from lesson 01.** NVIDIA's own GPUDirect RDMA documentation states the requirement as the two devices sharing the same root complex, and notes that platform bugs make some same-root-complex configurations perform badly anyway. In practice:

- **`PIX` or `PXB`** between GPU and NIC → the switch can route the transfer. GPUDirect works and is fast.
- **`PHB` / `NODE`** → the transfer must go up into the root complex and back down. It may work; it is materially slower on most platforms.
- **`SYS`** → different sockets. GPUDirect RDMA is not a supported fast path; the stack falls back to staging through host memory, and you pay every cost in the "default path" above *plus* a UPI crossing.
- **ACS enabled on the switch** (lesson 01 §11) → the hierarchy says `PIX` but the routing forces the transaction up to the root complex anyway. NVIDIA's GPUDirect and GDS guidance is explicit that ACS must be disabled for the switch-local path to be taken.

**So the placement tree is a precondition for the memory-path optimisations, not an independent concern.** A node with perfect DIMM population, perfect pinning and a NIC on the wrong socket gets none of the benefit.

### 11. Turning the gap into a number nobody else has

This is the module's whole point: the waste is real, large, and off-dashboard. The chain of reasoning that converts a measurement into a claim:

1. **Measure the ratio, not the absolute.** Run the same pinned H2D copy under `--membind=<home>` and `--membind=<far>`. The absolute number depends on transfer size, driver version and whether anything else is on the link; the *ratio* is a property of the topology and is stable.
2. **Prove the placement.** `numastat -p` output showing the pages where you claimed they are. Without this, a sceptic can say the benchmark was noisy.
3. **Bound the blast radius.** Not every byte crosses PCIe. Estimate the fraction of step time that is copy-bound (Nsight Systems, or the crude version: run with an artificially tiny batch and see how step time scales). A 40% bandwidth loss on a path that is 30% of step time is a ~12% throughput loss, not a 40% one — say so, because overclaiming is how findings get dismissed.
4. **Multiply by the fleet and the rate.** GPU-hours × the loss fraction × the hourly rate. This is the only step that requires no engineering and the only one anyone remembers.
5. **Name the metric that would and would not have caught it.** `DCGM_FI_PROF_DRAM_ACTIVE` and SM occupancy would not — both stay high. A scheduled local-vs-remote copy benchmark at node bring-up would. That contrast is the recommendation.

The reason this is worth doing carefully is that step 3 is where most people's version of this argument falls apart. "Overlap hides it anyway" is the standard objection, and it is half right: **overlap hides latency; it does not undo a halved bandwidth ceiling.** If the copy takes twice as long, it can no longer be fully hidden behind a compute phase that has not got longer. Say that explicitly and the objection evaporates.

## Perspectives

**Developer.** Pinned memory is a framework detail most ML engineers never touch — `DataLoader(pin_memory=True)` hides it, and it does the right thing for the common case (single GPU, single socket). The developer's job is knowing *when* the framework default is wrong: multi-GPU on a multi-socket box, where one shared pinned pool feeds GPUs on both sockets. The check is one line of `numastat`, and the fix is per-rank allocation with the allocating thread already bound. For Unified Memory codebases the equivalent lever is `cudaMemAdvise` with `cudaMemAdviseSetPreferredLocation` — same intent, different API surface.

**Operator.** Bandwidth validation belongs in node acceptance, not in mid-run debugging. STREAM Triad against `channels × rate × 8 B`, plus a pinned local-vs-remote `bandwidthTest` pair, is a five-minute script that catches half-populated boxes, derated DIMMs and mis-affinitised NICs on day zero rather than in week six. Store the numbers per node and diff them; a node that regresses after a firmware update is a signal you will otherwise never see.

**Hardware.** The 11× HBM:DRAM and 4.9× DRAM:PCIe gaps are architectural constants that shape every design decision above them. NVLink exists because GPU↔GPU traffic could not be allowed onto PCIe. GPUDirect exists because the host DRAM tier could not be allowed on the per-step path. Even MIG's NUMA-localisation behaviour is the same argument one level down, inside the GPU package. These are not independent features; they are four responses to one ratio.

**Economics.** Bandwidth lost to mis-pinning or under-populated channels is *pure* waste: it costs nothing to detect (a benchmark) and nothing to fix (a `numactl` flag, an allocator change, or a line in the next provisioning spec). That asymmetry — free to find, free to fix, expensive to ignore — is what makes it one of the highest-ROI findings a platform engineer can bring to a cost review. Contrast a genuine capacity problem, where the fix is a purchase order.

## Real-world use cases

- **Ronak Nathani — "Keeping GPU Workloads NUMA-Local in Kubernetes"** ([ronaknathani.com](https://ronaknathani.com/blog/2026/05/keeping-gpu-workloads-numa-local-in-kubernetes/)). The module's required deep read. The substance: it walks the same local-vs-remote argument from the Kubernetes scheduler's side, cites PyTorch's own performance-tuning guidance to bind training processes to a single NUMA node, and reports markedly worse p99 tail latency for inference pods whose CPU allocation spanned both sockets versus pods confined to one. What it shows: the "wrong-node buffer" problem this lesson quantifies with a copy benchmark has a directly measured production cost, in a latency metric that product teams already watch — which is a much easier sell than a bandwidth number.

- **NVIDIA — MIG with NUMA node localisation** ([developer.nvidia.com](https://developer.nvidia.com/blog/accelerating-data-processing-with-nvidia-multi-instance-gpu-and-numa-node-localization)). The substance: on a bandwidth-bound stencil kernel, configuring MIG instances so their memory is localised to the GPU's internal memory partitions produced a large speedup at a fixed power limit, with no change to the compute kernel at all. What it shows: the local-vs-remote argument is fractal. The same reasoning you apply to sockets applies inside the GPU package, and in both cases the win comes from *placement*, not from doing less work.

- **Meta Engineering — "How Meta trains large language models at scale"** ([engineering.fb.com](https://engineering.fb.com/2024/06/12/data-infrastructure/training-large-language-models-at-scale-meta/)). The substance: Meta describes the data-loading and storage-to-GPU path as a first-class scaling constraint at cluster scale, not an afterthought — the staging pipeline gets architectural attention alongside the collective communication. What it shows: the pipeline this lesson benchmarks on one node is the same pipeline that governs throughput on a 24k-GPU cluster; the difference is that at scale nobody can debug it by hand, so it has to be validated by construction.

- **NVIDIA GPUDirect RDMA documentation** ([docs.nvidia.com](https://docs.nvidia.com/cuda/gpudirect-rdma/)). The substance: NVIDIA states the same-root-complex requirement directly, warns that chipset bugs make some technically-valid configurations perform badly or not at all, and documents the peer-memory module registration path. What it shows: the topological precondition in §10 is a vendor-stated constraint, not a folk rule — which matters when you are arguing that a NIC needs to physically move.

## Worked example

**Goal.** Measure the local-vs-remote host→device bandwidth delta for one GPU on a two-socket, 8-GPU node, prove the placement, cross-check channel population, and record the result for the deliverable.

**Step 1 — find the GPU's home node.**

```
$ nvidia-smi topo -m | awk '/^GPU5/ {print $1, $(NF-1), $NF}'
GPU5    56-83,168-195    2
```

GPU5 homes on NUMA node 2 (socket 1). On this box nodes {0,1} are socket 0 and {2,3} are socket 1, so node 0 is the far socket.

**Step 2 — local copy: buffer on the GPU's node.**

```
$ numactl --cpunodebind=2 --membind=2 \
    ./bandwidthTest --device=5 --memory=pinned --mode=quick --htod --dtoh
[CUDA Bandwidth Test] - Starting...
Running on...
 Device 5: NVIDIA H100 80GB HBM3
 Quick Mode

 Host to Device Bandwidth, 1 Device(s)
 PINNED Memory Transfers
   Transfer Size (Bytes)        Bandwidth(GB/s)
   33554432                     54.2

 Device to Host Bandwidth, 1 Device(s)
 PINNED Memory Transfers
   Transfer Size (Bytes)        Bandwidth(GB/s)
   33554432                     51.9
```

54.2 GB/s HtoD against a 63.0 GB/s Gen5 x16 theoretical ceiling = **86% of link peak**. That is the healthy number and the target. The missing 14% is PCIe protocol overhead — packet framing, headers, LCRC, DLLP acknowledgements and flow-control credits — which lesson 03 breaks down line by line.

**Step 3 — remote copy: same GPU, buffer forced onto the far socket.**

```
$ numactl --cpunodebind=0 --membind=0 \
    ./bandwidthTest --device=5 --memory=pinned --mode=quick --htod --dtoh
 Host to Device Bandwidth, 1 Device(s)
   33554432                     31.9

 Device to Host Bandwidth, 1 Device(s)
   33554432                     30.1
```

**31.9 GB/s — a 41% drop**, with no change to the GPU, the transfer size, or the code. The DMA engine is now dragging every cache line across the inter-socket link before it reaches PCIe, and the result lands almost exactly where §1's arithmetic said it would: bounded by one UPI 2.0 link's ~38 GB/s effective rate, minus the concurrency loss from §2.

```
  Predicted:  min(PCIe 63 GB/s, UPI ~38 GB/s) = 38 GB/s, less Little's-Law
              concurrency loss on the DMA engine's outstanding reads
  Observed:   31.9 GB/s
  Ratio to local: 31.9 / 54.2 = 0.589 → 41.1% loss
```

**Step 4 — prove the placement, do not assume it.**

```
$ numactl --cpunodebind=0 --membind=0 ./bandwidthTest --device=5 --memory=pinned &
$ numastat -p $! | sed -n '1,12p'

Per-node process memory usage (in MBs) for PID 51447
                           Node 0          Node 1          Node 2          Node 3           Total
                  --------------- --------------- --------------- --------------- ---------------
Private                   4098.11            0.00            1.94            0.00         4100.05
                  --------------- --------------- --------------- --------------- ---------------
Total                     4121.06            0.06            2.11            0.02         4123.25
```

4.1 GB on node 0, essentially nothing on node 2. The buffer really is where `--membind` put it; the 41% gap is the inter-socket tax, reproducible.

**Step 5 — cross-check channel population so you can rule out a half-built box.**

```
$ OMP_NUM_THREADS=28 numactl --cpunodebind=2 --membind=2 ./stream | grep Triad
Triad:         270512.6     0.035671     0.035488     0.035901

$ sudo dmidecode -t memory | grep -c 'Size: [0-9]'
16
$ sudo dmidecode -t memory | grep 'Configured Memory Speed' | sort -u
        Configured Memory Speed: 4800 MT/s
```

16 populated slots on a 2-socket, 8-channel-per-socket board = 1 DIMM per channel, all at the rated 4800 MT/s. Peak per socket = `8 × 4800 × 8 B = 307.2 GB/s`; measured Triad = 270.5 GB/s = **88% of peak**. Channels are fully populated and undreated, so the remote gap is *purely* placement.

Had Triad read ~150 GB/s you would be looking at two independent defects stacked — half-populated channels *and* a placement bug — and the correct report says so, because they have different owners and different fixes.

**Step 6 — record the finding in the form that survives review.**

```
NODE: gpu-h100-047
GPU5 home NUMA node: 2 (socket 1)

  pinned H2D, --membind=2 (local) : 54.2 GB/s   [86% of Gen5 x16 peak]
  pinned H2D, --membind=0 (remote): 31.9 GB/s   [-41.1%]
  placement proof                 : numastat -p, 4098 MB on node 0
  STREAM Triad, node 2, 28 threads: 270.5 GB/s  [88% of 307.2 GB/s peak]
  DIMM population                 : 16/32 slots, 1 DPC, 4800 MT/s configured
                                    → fully populated, not derated

  CONCLUSION: channels are healthy. The 41% delta is attributable entirely to
  buffer placement. Predicted from topology: min(63, ~38) GB/s → observed 31.9.

  BLAST RADIUS: dataloader H2D is ~28% of step time (Nsight Systems).
                0.41 × 0.28 = ~11.5% throughput loss on this rank.
                In a synchronous data-parallel job the slowest rank sets the
                step time, so this is ~11.5% for the whole job.

  METRIC THAT WOULD HAVE CAUGHT IT: none in production. SM occupancy and
                DRAM-active both stayed at their normal values throughout.
                Recommend: local-vs-remote copy pair in node acceptance.
```

**Step 7 — scale the rule to the whole box.** Placement is *per-GPU*. On this node GPUs 0–3 home on socket 0 and 4–7 on socket 1. A dataloader that allocates one big pinned pool on node 0 feeds GPUs 0–3 on the fast path and GPUs 4–7 at the remote number — **half your GPUs silently run 41% slower on their host feed, and the synchronous collective makes that everyone's problem.** The correct pattern is per-GPU pinned buffers allocated by a thread already bound to that GPU's home node (or `cudaMemPoolCreate` with `cudaMemLocationTypeHostNuma` set per rank). Frameworks that shard by `local_rank` and allocate per rank get this right by construction; a naive single-pool loader does not. That is the bug the deliverable exists to catch.

## Practice

On a rented multi-GPU instance (or a multi-socket CPU box for the GPU-free fallback):

1. **Establish the peak you should be getting.** From `dmidecode -t memory`, count populated channels and read `Configured Memory Speed`. Compute `channels × rate × 8 B` per socket and write it down before measuring anything.
2. **Measure host DRAM.** Build STREAM with arrays at least 4× your LLC size. Run it under `numactl --cpunodebind=N --membind=N` for each node. Record Triad and compute the percentage of the peak from step 1. Then run it again with `--membind=<far node>` and record the drop.
3. **Measure host↔device.** Build `bandwidthTest` from `cuda-samples`. For one GPU, run the pinned HtoD/DtoH pair under `--cpunodebind=<home> --membind=<home>`, then again under `--membind=<far>`. Record both numbers and the percentage delta.
4. **Prove placement.** Capture `numastat -p <pid>` for each run. A finding without this is not a finding.
5. **Compare pinned to pageable** on the local case (`--memory=pageable` vs `--memory=pinned`) so you can see the CUDA bounce path's cost separately from the NUMA path's cost. Two different mechanisms, two different numbers.
6. **Check the IOMMU.** `dmesg | grep -iE 'iommu|swiotlb'`. Record whether `iommu=pt` is set and whether SWIOTLB is merely reserved or actually in use (`/sys/kernel/debug/swiotlb/io_tlb_used`). Note it as a candidate explanation if your measured numbers come in below what the placement arithmetic predicts.
7. **Do the fit arithmetic.** For a model you care about, compute weight bytes at BF16 and FP8, decide whether it fits in the GPU you measured, and compute the minimum per-forward-pass weight-read time at that GPU's HBM bandwidth — and at your measured remote PCIe bandwidth, as the "it does not fit" alternative.

**Acceptance:** a recorded **local-vs-remote bandwidth pair with the percentage delta** (e.g. "54.2 vs 31.9 GB/s, −41%"), the `numastat` proof of buffer placement for both runs, one sentence stating whether channels are fully populated *backed by the STREAM number against your computed peak*, and the model-fit arithmetic with units carried. Saved for the Topology Teardown.

## Common pitfalls

1. **"`pin_memory=True` means I'm safe."** *Symptom:* a correctly page-locked buffer that still copies at the remote rate. *Mechanism:* pinning fixes the *physical* address; it says nothing about *which node's* physical memory. The node was chosen by first-touch, by the allocating thread, at allocation time — and because the pages are pinned, nothing can migrate them afterwards. Pinning actively makes a misplacement permanent.

2. **Quoting `bandwidthTest`'s absolute number as a throughput prediction.** *Symptom:* a real placement bug dismissed because "the copy is overlapped anyway." *Mechanism:* overlap hides *latency* behind compute; it cannot hide a halved *bandwidth ceiling*, because the copy now takes longer than the compute phase it was meant to hide behind. Use the local-vs-remote ratio as your evidence and multiply it by a measured copy-bound fraction of step time.

3. **Assuming identical capacity means identical bandwidth.** *Symptom:* two "256 GB" nodes with a 2× difference in dataloader throughput. *Mechanism:* 4×64 GB populates four channels; 8×32 GB populates eight. The controller stripes across populated channels only. This is a provisioning-time defect that no inventory system, `free -g`, or Kubernetes node capacity report will ever surface.

4. **Reading `Speed` instead of `Configured Memory Speed` in `dmidecode`.** *Symptom:* computed peak that the machine can never reach. *Mechanism:* `Speed` is the DIMM's rating; `Configured Memory Speed` is what the controller clocked it at after 2DPC derating, mixed-population downclocking, or a conservative BIOS setting. Always compute peak from the configured value.

5. **Measuring cache and calling it DRAM.** *Symptom:* STREAM Triad reading 700+ GB/s on an 8-channel socket. *Mechanism:* the arrays fit in the LLC (up to ~105 MB on Sapphire Rapids, ~256 MB+ on some EPYC parts). STREAM warns about this; heed the warning and size arrays at ≥4× LLC.

6. **Ignoring SWIOTLB as an independent second bounce.** *Symptom:* pinning and NUMA both fixed, copy still below the arithmetic. *Mechanism:* the kernel is bouncing the DMA through its reserved low-memory region because the IOMMU cannot map the buffer directly. It lives at a completely different layer from CUDA's pageable bounce and both can be active. Check `dmesg` and `io_tlb_used`, and check whether `iommu=pt` is set.

7. **Treating remote bandwidth loss as pure link congestion.** *Symptom:* "the UPI link is only at 20% utilisation, so remote memory can't be the problem." *Mechanism:* Little's Law. Each requester can hold a fixed number of misses in flight; raising per-request latency by 1.75× lowers per-requester throughput by the same factor regardless of how idle the link is. Link utilisation is the wrong observable.

## Self-check

- **Why does a pinned host buffer's NUMA node change GPU copy speed, when the GPU never executes a load against host DRAM?**
  **Answer:** The GPU's copy engine DMAs the buffer over PCIe from its own socket's root complex. If the buffer is on the GPU's home node the path is `DRAM → memory controller → root complex → PCIe → HBM` and the bottleneck is PCIe (63 GB/s theoretical, ~50–55 GB/s achieved pinned). If the buffer is on the other socket, the DMA's reads must be serviced by the far memory controller and returned across the inter-socket link, putting a ~38 GB/s-effective UPI link *in series* with the 63 GB/s PCIe link. `min(63, 38)` less concurrency losses lands around 30–33 GB/s, a ~40% drop. The buffer's node was fixed by first-touch, by whichever thread first wrote the pages, at allocation time — and page-locking then makes that placement permanent, because pinned pages cannot be migrated.

- **Name two distinct mechanisms that bounce a DMA through an intermediate buffer even when the target is correctly placed.**
  **Answer:** (1) **CUDA's internal pinned staging buffer**, used when the *host side* of a `cudaMemcpy` is pageable rather than page-locked — the runtime copies pageable → its own pinned region → PCIe, which roughly halves throughput and prevents true async overlap. (2) **The kernel's SWIOTLB**, a 64 MB-by-default reserved low-memory region the DMA API copies through when the IOMMU cannot map the buffer directly (address-width limits, memory encryption, untrusted device). They sit at different layers — CUDA runtime vs kernel DMA API — and can both be active at once. Evidence for the first is a pinned-vs-pageable `bandwidthTest` gap; for the second, `dmesg` SWIOTLB messages and a non-zero `/sys/kernel/debug/swiotlb/io_tlb_used`.

- **Why is Triad, not Copy, the honest number from a STREAM run?**
  **Answer:** `Copy` is one read and one write per element. Under MESI, a store to a line the core does not already own triggers a **read-for-ownership** — the line is read into cache before it can be modified — so a "write" costs a read plus an eventual writeback. `Copy`'s ratio of RFO overhead to useful work is therefore the worst of the four kernels, which is why it reads *lower* than `Add` despite moving less data. `Triad` (`a[i] = b[i] + q*c[i]`, two reads and one write) is closest to a realistic streaming mix and is the figure STREAM's own guidance recommends. Beware benchmarks compiled with non-temporal stores (`MOVNTDQ`, `_mm_stream_*`): those bypass RFO entirely by never allocating the destination line, which produces write numbers that look impossibly high relative to the read-only peak.

- **You have 256 GB on an 8-channel socket as 4×64 GB DIMMs. What bandwidth do you expect versus 8×32 GB, and why?**
  **Answer:** Roughly half — `4 × 38.4 = 153.6 GB/s` versus `8 × 38.4 = 307.2 GB/s` at DDR5-4800. Each populated channel is an independent 64-bit data bus with its own command/address bus; the memory controller can only stripe traffic across the channels that have DRAM on them. Capacity is identical, so no inventory system, `free -g`, or Kubernetes node report distinguishes them; only a bandwidth benchmark does. Verify population with `dmidecode -t memory` (count `Size:` entries and read `Configured Memory Speed`, not `Speed`), and validate with STREAM Triad against your computed peak — a healthy socket lands at 80–90%.

- **What is the HBM-to-host-DRAM ratio, where does it come from, and what does it dictate?**
  **Answer:** About **11:1** — H100 SXM5 HBM3 at 3 350 GB/s versus 307.2 GB/s for an 8-channel DDR5-4800 socket. The gap is *interface width*, not clock: an HBM stack is 1024 bits wide because it is bonded to the GPU through TSVs and a silicon interposer inside the package, while a DDR5 channel is 64 bits because that is what you can route across a motherboard to a socketed DIMM. H100's five active stacks give a 5120-bit bus at ~5.2 Gbps/pin. The consequences: the working set must *reside* in HBM rather than be streamed from DRAM; PCIe (63 GB/s, 53× narrower than HBM) is for prefetch and overlap only; and NVLink and GPUDirect exist specifically to keep the slow host tier off the per-step critical path.

- **Why does remote memory lose *bandwidth* rather than only latency, even when the inter-socket link is far from saturated?**
  **Answer:** Little's Law: `concurrency = throughput × latency`. A core can hold only a fixed number of outstanding cache misses (typically 10–16 line-fill buffers / MSHRs). At 12 outstanding misses of 64 bytes each, local DRAM at 80 ns yields `12 × 64 B ÷ 80 ns = 9.6 GB/s` per core; remote DRAM at 140 ns yields `12 × 64 B ÷ 140 ns = 5.5 GB/s` — a 43% bandwidth loss purely from the latency increase, with the link nowhere near capacity. Recovering the lost bandwidth would require ~1.75× as many concurrent requesters, which a fixed thread count cannot provide. This is why link utilisation is the wrong observable, and why the penalty gets *worse* under load: queueing raises latency further, which lowers per-requester throughput further.

- **What is the topological precondition for GPUDirect RDMA, and what happens when it is not met?**
  **Answer:** The GPU and the NIC must share a root complex — in practice, be siblings under the same PCIe switch, which `nvidia-smi topo -m` reports as `PIX` or `PXB`. The mechanism is that the GPU exposes a window of HBM through a PCIe BAR, the peer-memory module registers that range as a legal DMA target, and the NIC writes payload into it with ordinary PCIe memory writes routed by the switch. At `PHB`/`NODE` the transaction must go up into the root complex and back down, which works but is materially slower on most platforms. At `SYS` — different sockets — it is not a supported fast path and the stack falls back to staging through host DRAM, paying two PCIe crossings, two DRAM crossings and a UPI crossing. Separately, ACS enabled on the switch forces even a `PIX` pair's transactions upstream to the root complex, so the label can be right and the path still wrong.

## Connections & what's next

This lesson turned lesson 01's static tree into a measured cost: the same NUMA-affinity read now has a bandwidth number attached to it, derived from channel arithmetic rather than asserted, and you have the tooling (`bandwidthTest`, STREAM, MLC, `numastat`, `dmidecode`) to reproduce it on any box.

**Lesson 03** picks up exactly where §5's "86% of link peak" left off. Every bandwidth ceiling in this lesson assumed a healthy PCIe link that trained to Gen5 x16; lesson 03 teaches you to verify that assumption from `LnkCap` vs `LnkSta`, breaks down where the missing 14% goes (encoding, TLP framing, DLLP acknowledgements, `MaxPayload`/`MaxReadRequest`), and shows you the early-warning signals that precede a link silently downtraining. **Lesson 04** applies this lesson's per-GPU placement rule at whole-node scale and introduces the NVLink plane properly. **Lesson 05**'s Kubernetes Memory Manager is the cluster-scheduled version of the `numactl --membind` trick you just ran by hand. **Lesson 06** reuses the GPUDirect precondition from §10 for NVMe and GPUDirect Storage.

## References & further reading

**Primary sources**

- **NVIDIA CUDA C++ Best Practices Guide — pinned memory and async transfers** — [docs.nvidia.com](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/#pinned-memory) — the rationale for page-locking, the pageable-memory bounce, and why only pinned memory permits `cudaMemcpyAsync` overlap.
- **NVIDIA CUDA Runtime API — memory management** — [docs.nvidia.com](https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__MEMORY.html) — `cudaHostAlloc`, `cudaHostRegister`, `cudaMemPoolCreate`, and the `cudaMemLocationTypeHostNuma` location type used in §8's option 3.
- **NVIDIA GPUDirect RDMA documentation** — [docs.nvidia.com](https://docs.nvidia.com/cuda/gpudirect-rdma/) — the same-root-complex requirement, the peer-memory registration path, and the platform caveats behind §10.
- **Linux kernel — DMA and swiotlb** — [docs.kernel.org/core-api/swiotlb.html](https://docs.kernel.org/core-api/swiotlb.html) — what SWIOTLB is, when the DMA API falls back to it, the `swiotlb=` parameter, and the debugfs counters.
- **JEDEC DDR5 (JESD79-5) and vendor DDR5 technical notes** — e.g. [Micron's DDR5 new-features white paper](https://assets.micron.com/adobe/assets/urn:aaid:aem:5ea148c8-e3fe-489e-8489-99b1b9cdcd3c/renditions/original/as/ddr5-new-features-white-paper.pdf) — the two-32-bit-sub-channel structure, burst length 16, and the 40-bit (32 data + 8 ECC) sub-channel width used in §3.
- **Intel and AMD platform documentation** — 4th/5th Gen Xeon Scalable datasheets and the AMD EPYC 9004 tuning guides — channel counts, supported DIMM speeds, 2DPC derating, and the SNC / NPS population requirements referenced in §4.

**Tools**

- **STREAM (John McCalpin)** — [cs.virginia.edu/stream](https://www.cs.virginia.edu/stream/) — the reference host-DRAM benchmark; read the FAQ on array sizing versus cache before quoting any number from it.
- **Intel Memory Latency Checker (MLC)** — [intel.com](https://www.intel.com/content/www/us/en/download/736633/intel-memory-latency-checker-intel-mlc.html) — `--latency_matrix` and `--bandwidth_matrix` produce the measured node×node tables in §5, which are the honest replacement for the SLIT's single integer.
- **NVIDIA `cuda-samples` — `bandwidthTest`, `p2pBandwidthLatencyTest`** — [github.com/NVIDIA/cuda-samples](https://github.com/NVIDIA/cuda-samples) — the H2D/D2H pinned-vs-pageable measurements used throughout, and the P2P matrix that exposes the ACS problem from lesson 01.

**Real-world engineering**

- **Ronak Nathani — "Keeping GPU Workloads NUMA-Local in Kubernetes"** — [ronaknathani.com](https://ronaknathani.com/blog/2026/05/keeping-gpu-workloads-numa-local-in-kubernetes/) — production p99 latency cost for cross-socket pods, and the PyTorch tuning-guide recommendation this lesson's practice echoes.
- **NVIDIA Developer Blog — MIG and NUMA node localisation** — [developer.nvidia.com](https://developer.nvidia.com/blog/accelerating-data-processing-with-nvidia-multi-instance-gpu-and-numa-node-localization) — a measured speedup on a bandwidth-bound kernel from correct memory localisation alone.
- **Meta Engineering — "How Meta trains large language models at scale"** — [engineering.fb.com](https://engineering.fb.com/2024/06/12/data-infrastructure/training-large-language-models-at-scale-meta/) — the staging pipeline treated as a first-class scaling constraint at cluster scale.

**Deeper dives**

- **Brendan Gregg, *Systems Performance*, 2nd ed.** — [brendangregg.com](https://www.brendangregg.com/systems-performance-2nd-edition-book.html) — targeted skim: the memory and NUMA chapters, for the methodology of turning a benchmark into evidence rather than an anecdote. The USE method applied to memory is the discipline behind §11's five steps.
