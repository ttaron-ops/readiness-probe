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
sources: 12
---
# 02b.6 · NVMe and storage placement for GPU data paths
> **Concept.** Local NVMe (or the NIC serving remote storage) that does not share the GPU's PCIe switch/root complex contends for the same PCIe path the GPU feeds on, and the stall shows up as "GPU idle" that no utilization metric explains — and GPUDirect Storage only pays off when that placement is correct.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Where this fits

Lesson 05 gave you a Kubernetes-level guarantee: with the right kubelet policies, a pod's CPUs, memory, and GPU are provably co-located on one NUMA node. That guarantee says nothing about a fourth resource every training job depends on — the drive, or the network path, the dataset lives on. **There is no Topology Manager hint provider for block devices.** Nothing in Kubernetes will align storage for you; `single-numa-node` will happily admit a perfectly aligned pod whose dataset sits on an NVMe controller across the socket boundary. This lesson closes that last placement gap by extending the same PCIe-topology reading skill from lessons 03 and 04 to storage, teaching enough of the NVMe protocol and the Linux block layer that you can tell a *placement* stall from a *queue-depth* stall, and introducing the one software mechanism — GPUDirect Storage — whose entire value proposition depends on getting that placement right. It's the last piece before lesson 07's power and thermal factors and the lesson 08 capstone, where you reconcile GPU, CPU, NIC and NVMe placement into one diagram on a real node.

## Why this matters

You already know the GPU is the expensive part. What the utilization dashboard never tells you is that a GPU spends real wall-clock time waiting for the next batch to arrive from disk — and while it waits, `nvidia-smi` can still print `GPU-Util: 100%`, because that counter reports "at least one kernel was resident during the sample window," not "the SM array did useful work." A training step that is 30% data-loader-bound is 30% paid-for-but-undelivered compute.

The root cause is usually one of two things, and they need different fixes:

- **Placement.** The NVMe holding the dataset (or the NIC fronting the object store) sits on the *wrong* PCIe root complex, so every byte crosses the inter-socket link (Intel UPI / AMD Infinity Fabric) or contends on the same switch the GPU is using for peer-to-peer and RDMA traffic. Fix: free. It is a scheduling decision, not a purchase.
- **Concurrency.** The reader is issuing one I/O at a time. An NVMe SSD is a *parallelism* machine before it is a *speed* machine; a single-threaded, queue-depth-1 reader will leave a Gen5 drive 90% idle no matter where it sits. Fix: also free, but a different change entirely.

Diagnosing the wrong one wastes a week. That is why this lesson teaches the protocol before the topology: once you understand that a modern NVMe controller has thousands of independent queues and that Linux gives one submission/completion pair to each CPU core, both the "why does queue depth matter" and the "why does placement matter" answers fall out of the same picture.

The economics repeat lesson 05's. At roughly $2–3/GPU-hr (a 2026 snapshot — plug your own), a persistent 20% data stall across a 512-GPU fleet is a low-seven-figure annual line item that appears in no dashboard, and it is one of the highest-ROI findings a platform engineer can bring to a cost review precisely because the fix requires no capital spend.

## What's new here (calibration)

You've read `lspci` a hundred times for NIC and HBA troubleshooting, and you know NVMe beats SATA at a hand-wave level, so we skip re-deriving those. Genuinely new here:

- **New:** the NVMe queue-pair model at operational depth — submission and completion queues as circular buffers in *host* memory, doorbell registers in the controller's BAR0, the 64-byte SQE / 16-byte CQE, the phase tag, `CAP.MQES`, the doorbell stride, and why the admin queue is QID 0 with a hard 4096-entry cap while I/O queues go to 65535.
- **New:** how Linux maps that onto blk-mq — one hardware queue per CPU by default, `nvme.io_queue_depth` (default 1024), `write_queues`/`poll_queues`, `scheduler=none` — and how to compute the queue depth your target bandwidth actually requires with Little's law, instead of guessing.
- **New:** reading `nvme id-ctrl` and `nvme smart-log` field by field, including the two units that trip everyone (`data_units_read` is in thousands of 512-byte units; `temperature` is Kelvin) and the fields that turn a drive into a *thermal* story that connects to lesson 07.
- **New:** treating *storage* as a first-class citizen on the GPU's PCIe path — on the GPU's switch, on its root complex, or across the socket, with materially different achievable bandwidth in each case.
- **New:** **GPUDirect Storage** — the peer-to-peer DMA path that lets an NVMe controller write straight into GPU HBM — its two independently failing layers (`cuFile` userspace, `nvidia-fs` kernel), the `allow_compat_mode` setting that makes it silently slow rather than loudly broken, and `gdscheck -p`.
- **New:** worked capacity math you can re-run — read bandwidth for N GPUs at a given samples/sec, and checkpoint write time for a model with P parameters.
- **New:** generalising to network-attached storage and **NVMe over Fabrics**, where "which drive" becomes "which NIC."

## Core concepts

### 1. The problem NVMe was invented to solve

AHCI, the interface SATA drives speak, was designed around a spinning disk: **one command queue, 32 entries deep**, and a register-based submission path that required several MMIO round-trips per command. That was fine when the media took 5–10 ms to answer. It is catastrophic when the media answers in 80 µs, because the *interface* becomes the bottleneck: one queue means one lock and one CPU serialising submissions for the whole device, and 32 outstanding commands is nowhere near enough to keep a device with dozens of independent NAND channels busy.

NVMe's design answer is: **make the queues plural, put them in host DRAM, and make the doorbell the only MMIO write.**

Everything else in this section follows from those three decisions.

### 2. The queue-pair model, precisely

An NVMe controller exposes **queue pairs**. Each pair is one **Submission Queue (SQ)** and one **Completion Queue (CQ)**, both circular buffers allocated by the *host* in ordinary system memory. The controller reads commands out of the SQ by DMA and writes results into the CQ by DMA. The only registers involved in the fast path are the **doorbells**.

- **Queue identifiers.** The pair with **QID 0** is the **Admin queue** — one per controller, used for `Identify`, `Get Log Page`, `Create I/O Submission Queue`, firmware update, format, and so on. Its size is capped at **4096 entries** for both SQ and CQ. I/O queue identifiers run **1 to 65535**, so the spec allows up to 65,535 I/O queue pairs. Real controllers advertise far fewer (dozens to a few hundred); `Identify Controller` and the `Set Features`/`Number of Queues` negotiation tell you what you actually got.
- **Entry sizes.** An NVM-command-set submission queue entry (SQE) is **64 bytes**; a completion queue entry (CQE) is **16 bytes**. So a 1024-entry SQ costs 64 KiB of host memory and its CQ costs 16 KiB. Multiply by the number of queues when you wonder where the driver's memory went: 128 queues × 1024 entries ≈ 10 MiB per drive.
- **Queue depth.** `CAP.MQES` (Maximum Queue Entries Supported, in the controller's Capabilities register) is a **0's-based** field: the real maximum is `MQES + 1`. Linux uses `min(CAP.MQES + 1, io_queue_depth)`.
- **Doorbells.** The controller's BAR0 MMIO region contains a doorbell array starting at **offset 0x1000**. For queue `qid`, the SQ Tail doorbell is at index `qid * 2 * stride` and the CQ Head doorbell at `qid * 2 * stride + stride`, where `stride = 1 << CAP.DSTRD` in dwords. In the Linux driver this is literally `dev->dbs = dev->bar + 4096;` and `nvmeq->q_db = &dev->dbs[qid * 2 * dev->db_stride];`.

**The command lifecycle**, step by step:

1. The host writes a 64-byte SQE into the SQ at its private tail index. The SQE carries an opcode (`01h` write, `02h` read for the NVM command set), a **Command Identifier (CID)**, the namespace ID, the starting LBA, the length, and either **PRP** entries or an **SGL** describing the data buffer's physical pages.
2. The host advances its tail and **writes the new tail value to the SQ Tail doorbell**. This is the *only* MMIO write in the fast path — one 4-byte store, no read-back, no round trip. Multiple commands can be written and then published with a single doorbell write, which is how batching works.
3. The controller sees the doorbell, DMAs the SQE(s) out of host memory, executes them, and DMAs the data to or from the buffers the PRP/SGL pointed at.
4. The controller writes a 16-byte CQE into the CQ. The CQE carries the CID it is completing, a status field, and a **phase tag** bit. The host does not need a doorbell read to know a new entry arrived: the controller flips the phase tag each time the queue wraps, so the host simply polls the next CQE slot and compares its phase bit to the expected value. This is what makes polled (interrupt-free) completion possible.
5. If interrupts are enabled, the controller also raises the **MSI-X vector** bound to that CQ. One vector per queue is the normal arrangement, which is what lets completions be delivered to the *same core* that submitted.
6. The host processes completions and **writes the CQ Head doorbell** to tell the controller those slots are free.

```
   NVMe QUEUE PAIRS, DOORBELLS, AND WHERE EACH BYTE ACTUALLY LIVES

   HOST DRAM (allocated by the driver)              CONTROLLER (BAR0 MMIO)
   ────────────────────────────────────             ──────────────────────
                                                    0x0000  CAP  (MQES, DSTRD, …)
   core 0 ┌─ SQ 1 ─────────────────┐                0x0008  VS
          │ head↑        tail↓     │                0x0014  CC
          │ [SQE][SQE][SQE][ ][ ]  │ 64 B each      0x001C  CSTS
          └────────────────────────┘                0x0024  AQA
          ┌─ CQ 1 ─────────────────┐                0x0028  ASQ  (admin SQ base addr)
          │ [CQE][CQE][ ][ ][ ]    │ 16 B each      0x0030  ACQ  (admin CQ base addr)
          │  phase tag flips on wrap│               ─────────────────────────
          └────────────────────────┘                0x1000  ┌ DOORBELL ARRAY ┐
                                                            │ SQ0 Tail (admin)│
   core 1 ┌─ SQ 2 ─┐  ┌─ CQ 2 ─┐                            │ CQ0 Head        │
          └────────┘  └────────┘                            │ SQ1 Tail  ◀─────┼── ONE 4-byte
              …           …                                 │ CQ1 Head        │   MMIO write
   core N ┌─ SQ N+1┐  ┌─ CQ N+1┐                            │ SQ2 Tail        │   publishes a
          └────────┘  └────────┘                            │ CQ2 Head        │   whole batch
                                                            │       …         │
   QID 0 = admin pair (≤4096 entries)                       └─────────────────┘
   QID 1..65535 = I/O pairs                       index = qid*2*stride (+stride for CQ)
                                                  stride = 1 << CAP.DSTRD

   ── ONE READ, END TO END ──
   ① host writes SQE at SQ tail        (host DRAM store, no MMIO)
   ② host writes SQ Tail doorbell      (MMIO store — the only one)
   ③ controller DMA-reads the SQE      (PCIe read from host DRAM)
   ④ controller DMA-writes the data    (PCIe write → destination buffer)
   ⑤ controller DMA-writes the CQE     (PCIe write to host DRAM)
   ⑥ controller raises MSI-X           (or host polls the phase tag)
   ⑦ host writes CQ Head doorbell      (MMIO store, frees the slots)

   ⚠ EVERY ONE of ③④⑤ is a PCIe transaction that must traverse whatever
     bridges, switches and inter-socket links sit between this controller
     and the memory the buffer lives in.  That path is what §5 is about.
```

The crucial observation for the rest of this lesson: **step ④ writes to whatever physical memory the PRP/SGL pointed at.** Normally that is host DRAM on some NUMA node. With GPUDirect Storage it is a GPU's BAR-mapped HBM. In both cases the controller is a PCIe requester issuing writes to a target, and the cost of that write is entirely determined by the PCIe path between the two devices.

### 3. How Linux drives it: blk-mq, one queue per core

The Linux block layer's multi-queue design (blk-mq) exists to remove the single global request-queue lock that made the old block layer a bottleneck above ~1M IOPS. The structure:

- **Software queues** — one per CPU. A thread submitting I/O touches only its own CPU's queue, so submissions never contend across cores.
- **Hardware queues** — mapped onto the device's real queues. For NVMe, **each hardware queue is exactly one SQ/CQ pair**, and the MSI-X vector for that CQ is affinitised to the same CPU. Submission, completion and interrupt handling all land on one core; the CQE the controller writes is DMA'd into memory that core's caches already own.
- **Scheduler** — NVMe devices default to `none`. The elevator's job was to merge and reorder for a seek-limited spinning disk; on a device with no seek penalty and thousands of queues, reordering costs more than it saves.

How many queues does Linux create? From `drivers/nvme/host/pci.c`:

```c
static unsigned int nvme_max_io_queues(struct nvme_dev *dev)
{
    return blk_mq_num_possible_queues(0) + dev->nr_write_queues + dev->nr_poll_queues;
}
```

Default: **one I/O queue per possible CPU**, plus optional dedicated write and poll queue sets, capped by what the controller and the available MSI-X vectors allow. On a 128-thread box with a controller that supports it, you get 128 I/O queue pairs.

Queue depth comes from the same file:

```c
static unsigned int io_queue_depth = 1024;          /* nvme.io_queue_depth */
#define NVME_PCI_MIN_QUEUE_SIZE 2
#define NVME_PCI_MAX_QUEUE_SIZE 4095
...
dev->q_depth   = min_t(u32, NVME_CAP_MQES(dev->ctrl.cap) + 1, io_queue_depth);
dev->db_stride = 1 << NVME_CAP_STRIDE(dev->ctrl.cap);
dev->dbs       = dev->bar + 4096;
```

Three module parameters worth knowing, all settable as `nvme.<name>=` on the kernel command line:

| Parameter | Default | Effect |
|---|---|---|
| `io_queue_depth` | 1024 (min 2, max 4095) | Entries per I/O SQ/CQ. Rarely needs changing; deeper queues mostly buy queueing latency, not throughput. |
| `write_queues` | 0 | Carve out a separate set of queues for writes so a burst of writes cannot head-of-line-block reads. Relevant for checkpoint-heavy training. |
| `poll_queues` | 0 | Queues serviced by **polling** instead of interrupts. Burns a core but removes interrupt and context-switch latency — the lowest-latency mode available, used with `io_uring`'s `IORING_SETUP_IOPOLL` and `O_DIRECT`. |

Inspect the result on a live system:

```
$ ls /sys/block/nvme0n1/mq/ | wc -l
128                                  # 128 hardware queues = 128 SQ/CQ pairs

$ cat /sys/block/nvme0n1/queue/scheduler
[none] mq-deadline kyber bfq

$ cat /sys/block/nvme0n1/queue/nr_requests
1023                                 # per-hw-queue tag depth exposed to the block layer

$ cat /sys/block/nvme0n1/queue/io_poll
0                                    # 1 when poll queues are configured

$ cat /sys/block/nvme0n1/queue/max_hw_sectors_kb
2048                                 # derived from Identify Controller MDTS

$ cat /sys/block/nvme0n1/queue/rotational
0
```

`max_hw_sectors_kb` is worth a pause: it comes from the controller's **MDTS** (Maximum Data Transfer Size) field, reported as a power of two in units of the minimum memory page size. `MDTS = 5` on a 4 KiB page controller means `2^5 × 4 KiB = 128 KiB` maximum per command. Any read larger than that gets split by the block layer into multiple commands — which is fine, but it means "one 1 MiB read" is really eight commands and eight completions, and your effective queue depth is eight times what you thought.

### 4. Queue depth is not a vibe — compute it

The single most common wrong diagnosis in this area is blaming topology when the reader is simply not asking for enough work at once. Little's law settles it:

```
  outstanding_requests  =  throughput (IOPS)  ×  latency (seconds)
```

**Case A — large sequential reads, the shape a shard-based data loader produces.**

```
  Target                          14.0 GB/s   (one Gen5 ×4 drive's rated seq read)
  Request size                    128 KiB  = 131,072 B
  Device read latency at that size  ≈ 80 µs   (measure yours with fio --output-format=json)

  required IOPS      = 14.0e9 / 131,072            = 106,812 IOPS
  outstanding needed = 106,812 × 80e-6             = 8.5 requests

  → a queue depth of ~9 is the *theoretical minimum*. Run QD 16–32 to absorb
    jitter and keep the pipe full during completion processing.
    At QD 1 you would get: 1 / 80e-6 × 131,072 = 1.6 GB/s — 12% of the drive.
```

**Case B — small random reads, the shape a record-level loader produces.**

```
  Target                          2.7M IOPS  (CM7-series rated random read)
  Request size                    4 KiB
  Latency                         ≈ 80 µs

  outstanding needed = 2.7e6 × 80e-6               = 216 requests

  → 216 in flight. With 128 hardware queues that is under 2 per queue, which is
    exactly why NVMe wants many shallow queues rather than one deep one: the
    concurrency has to come from many *cores* submitting, or the submitting core
    becomes the bottleneck long before the media does.
```

**Case C — what a naïve PyTorch `DataLoader` actually issues.** `num_workers=4` with buffered (non-`O_DIRECT`) reads gives you at most 4 outstanding read syscalls, plus whatever the kernel's readahead adds. Against Case A's requirement of 9, and Case B's of 216, you can see the shape of the problem immediately.

The knobs, in the order to try them:

| Knob | Where | Effect on outstanding I/O |
|---|---|---|
| `num_workers` | `torch.utils.data.DataLoader` | Multiplies outstanding syscalls by worker count. First thing to raise. |
| `prefetch_factor` | same | Batches queued ahead per worker (default 2). Deepens the pipeline, hides latency spikes. |
| `pin_memory=True` | same | Staging buffers are page-locked, so the H2D copy is a real DMA rather than a driver-side bounce copy. Costs pinned memory; required for `cudaMemcpyAsync` overlap. |
| readahead | `blockdev --setra` / `/sys/block/*/queue/read_ahead_kb` | Kernel-side prefetch for buffered sequential reads. Irrelevant under `O_DIRECT`. |
| `io_uring` + `O_DIRECT` | application | Submit N requests with one syscall; bypass the page cache entirely. The only way to reach Case B's 216 outstanding from a small number of threads. |
| `poll_queues` | `nvme.poll_queues=N` | Removes interrupt latency at the cost of a spinning core. Last resort, and only with `IORING_SETUP_IOPOLL`. |

**Rule to remember: topology sets the ceiling, queue depth determines whether you reach it.** Check queue depth *first*, because it is cheaper to test and because "I moved the drive and nothing changed" is almost always a queue-depth story.

### 5. The three placement cases

Now the topology. For a GPU consuming data from a local drive, the byte path is one of three shapes, and `nvidia-smi topo -m`'s legend names them:

| Case | Path | `topo -m` code | Consequence |
|---|---|---|---|
| **1. Same PCIe switch** | NVMe → switch → GPU. Never touches the CPU's root port or the inter-socket link. | `PIX` (at most one bridge) or `PXB` (multiple bridges, still below the host bridge) | Lowest latency, full drive bandwidth, and **the only topology where GPUDirect Storage reaches its peak** — peer-to-peer DMA stays inside the switch. |
| **2. Same root complex, different port** | NVMe → root complex → GPU. Traffic traverses the CPU's internal PCIe fabric. | `PHB` (traverses a PCIe Host Bridge) | Fine for host-staged I/O. Adds the root port as a shared contention point with every other device under it. P2P across a root complex is supported on some CPUs and not others — this is where GDS quietly falls back on some platforms. |
| **3. Cross-socket** | NVMe on socket 0's root complex, GPU on socket 1's. | `SYS` (traverses the SMP interconnect) | Every byte crosses UPI/Infinity Fabric, sharing a ~48 GB/s-per-direction link with all cache-coherence traffic and every other remote access on the box. This is the case that silently caps your loader. |

The asymmetry is the whole point: in case 3 the drive's rated bandwidth is entirely real, but you cannot deliver it to *that* GPU without stealing from every other cross-socket transfer on the machine.

```
   THE SAME 14 GB/s DRIVE, THREE PLACEMENTS, THREE CEILINGS

  ┌────────────── SOCKET 0 / NUMA 0 ──────────────┐ ┌──── SOCKET 1 / NUMA 1 ────┐
  │  DDR5 ×8  ~307 GB/s                           │ │  DDR5 ×8  ~307 GB/s       │
  │      │                                        │ │      │                    │
  │  ┌───┴─────────── root complex ───────────┐   │ │  ┌───┴──── root complex ─┐│
  │  │        │                    │          │   │ │  │        │              ││
  │  │   ┌────┴─────┐         ┌────┴────┐     │   │ │  │   ┌────┴────┐         ││
  │  │   │ PCIe sw  │         │ PCIe sw │     │   │ │  │   │ PCIe sw │         ││
  │  │   └─┬──┬──┬──┘         └──┬───┬──┘     │   │ │  │   └──┬───┬──┘         ││
  │  │  GPU0 │ nvme0          GPU1  nvme2     │   │ │  │   GPU4  nvme3         ││
  │  │    mlx5_0                              │   │ │  │                       ││
  │  └────────────────────────────────────────┘   │ │  └───────────────────────┘│
  └───────────────────────┬───────────────────────┘ └────────────┬──────────────┘
                          └──────────── UPI 2.0 ─────────────────┘
                            ~48 GB/s/dir/link, SHARED with all
                            coherence + remote-memory traffic

  ── CASE 1  GPU0 ← nvme0   (PIX: same switch) ────────────────────────────────
     path: nvme0 ──▶ switch ──▶ GPU0                    hops below host bridge: 1
     host-staged ceiling: drive rating           14.0 GB/s
     GDS (peer-to-peer) : switch-local DMA       14.0 GB/s  ✓ full benefit
     CPU cost           : doorbell + completion only

  ── CASE 2  GPU0 ← nvme2   (PHB: same socket, different switch) ──────────────
     path: nvme2 ──▶ switch ──▶ ROOT COMPLEX ──▶ switch ──▶ GPU0
     host-staged ceiling: drive rating, minus root-port contention
     GDS                : platform-dependent; may fall back to compat mode
     shared with        : every other device under that root complex

  ── CASE 3  GPU0 ← nvme3   (SYS: cross-socket) ───────────────────────────────
     path: nvme3 ──▶ sw ──▶ RC1 ──▶ UPI ──▶ RC0 ──▶ sw ──▶ GPU0
     host-staged ceiling: min(14.0 GB/s, your share of one UPI link)
                          at a 50% share ≈ 24 GB/s aggregate for ALL cross-
                          socket traffic — your 14 GB/s stream is competing
     GDS                : peer-to-peer across the SMP interconnect is not a
                          supported fast path — expect compat-mode fallback
     symptom            : "GPU idle", bimodal step times, no error anywhere
```

### 6. Why a misplaced disk reads as "GPU idle"

The host-staged data-loader pipeline is:

```
  NVMe read → page cache or O_DIRECT buffer in host DRAM
            → (decode / augment / collate on CPU workers)
            → pinned staging buffer
            → cudaMemcpyAsync H2D over PCIe
            → GPU HBM
            → step N+1
```

If the NVMe leg is throttled by a cross-socket hop, batch *N+1* arrives after step *N* finishes and the GPU blocks. During that block:

- **`nvidia-smi`'s `utilization.gpu`** may still read high, because it reports the fraction of the sample window during which one or more kernels were executing — and the framework typically leaves *something* resident. Sample it fast enough and you see a sawtooth; average it and it looks "busy enough" that nobody investigates.
- **DCGM `SM_ACTIVE` (`DCGM_FI_PROF_SM_ACTIVE`) and `PIPE_TENSOR_ACTIVE`** sag on the same cadence as the stall. These tell the truth. Most teams do not chart them.
- **The trainer's own step-time histogram goes bimodal** — a fast mode (batch was prefetched) and a slow mode (waited on I/O). That bimodality is the fingerprint, and it is visible without any GPU telemetry at all.

```
   TIMELINE — WHY THE SAME UTILIZATION NUMBER HIDES A 40% LOSS
   (one GPU, five steps; ▓ = SMs doing work, · = SMs stalled on data)

   ALIGNED   (loader keeps ahead; prefetch depth 2 always satisfied)
     loader  [read N+1 ][read N+2 ][read N+3 ][read N+4 ][read N+5 ]
     H2D       [cp N+1]  [cp N+2]  [cp N+3]  [cp N+4]  [cp N+5]
     GPU     ▓▓▓▓▓▓▓▓▓▓|▓▓▓▓▓▓▓▓▓▓|▓▓▓▓▓▓▓▓▓▓|▓▓▓▓▓▓▓▓▓▓|▓▓▓▓▓▓▓▓▓▓
     step ms      100        100        100        100        100
     nvidia-smi utilization.gpu ≈ 100%    DCGM SM_ACTIVE ≈ 0.95

   MISALIGNED (reads cross UPI; each read takes 1.7× as long)
     loader  [ read N+1     ][ read N+2     ][ read N+3     ]
     H2D          [cp N+1]        [cp N+2]        [cp N+3]
     GPU     ▓▓▓▓▓▓▓▓▓▓········|▓▓▓▓▓▓▓▓▓▓········|▓▓▓▓▓▓▓▓▓▓·······
     step ms      100    +70        100    +70        100   +70
     nvidia-smi utilization.gpu ≈ 100%  ← STILL, because a kernel is
                                          resident during most windows
     DCGM SM_ACTIVE ≈ 0.58              ← the truth
     step-time histogram: bimodal at 100 ms and 170 ms  ← the fingerprint

   Throughput loss: 70/170 = 41% of wall clock, on a $2-3/GPU-hr asset,
   with the primary utilization metric reading unchanged.
```

The tell that separates *placement* from *slow code*: run the identical loader against a GPU that shares the drive's root complex. If it runs fine there, it is placement. If it is equally slow, go back to §4 and count outstanding I/Os.

### 7. Reading the drive: `nvme-cli` field by field

Two commands carry almost everything you need. Learn the fields, not just the commands.

**`nvme list`** — the inventory, and the mapping from block device to PCI address:

```
$ nvme list
Node          SN                   Model                    Namespace  Usage                      Format           FW Rev
------------- -------------------- ------------------------ ---------- -------------------------- ---------------- --------
/dev/nvme0n1  Y0X0A00ZT1L8         KIOXIA KCMYXRUG3T84               1   3.84  TB /   3.84  TB      512   B +  0 B  0104
/dev/nvme1n1  Y0X0A00ZT1M2         KIOXIA KCMYXRUG3T84               1   3.84  TB /   3.84  TB      512   B +  0 B  0104

$ readlink -f /sys/block/nvme0n1/device/device
/sys/devices/pci0000:c0/0000:c0:01.1/0000:c1:00.0
$ cat /sys/block/nvme0n1/device/device/numa_node
0
```

That `readlink` is the step people skip and then regret: it is how you map `/dev/nvme0n1` to a **BDF** (`0000:c1:00.0`) so you can look it up in `lspci -tv` and in `nvidia-smi topo -m`. Without it, every statement about "the drive's topology" is a guess.

**`nvme id-ctrl /dev/nvme0 -H`** — Identify Controller. The fields that matter operationally:

| Field | Meaning | Why you care |
|---|---|---|
| `vid` / `ssvid` | PCI Vendor ID / Subsystem Vendor ID | Cross-check against `lspci` when a chassis has drives from two vendors. |
| `sn` / `mn` / `fr` | Serial Number / Model Number / Firmware Revision, all ASCII | `fr` is the first thing to compare across a fleet when one node is slow. |
| `mdts` | **Maximum Data Transfer Size**, a power of two in units of the minimum page size (`CAP.MPSMIN`) | Sets `max_hw_sectors_kb`. Determines how many commands a large read becomes. `mdts=5` on a 4 KiB-page controller = 128 KiB per command. |
| `sqes` / `cqes` | Submission / Completion Queue Entry Size, min and max as power-of-two nibbles | Confirms the 64 B / 16 B entry sizes the driver assumes. |
| `nn` / `mnan` | Number of Namespaces / Maximum Number of Allowed Namespaces | Multi-namespace drives partitioned for different tenants. |
| `oncs` | Optional NVM Commands Supported | Whether the drive does Write Zeroes, Dataset Management (i.e. usable TRIM/discard), Compare, and so on. |
| `vwc` | Volatile Write Cache present | If there is a volatile cache, `fsync`/FUA behaviour and power-loss semantics change. |
| `awun` | Atomic Write Unit Normal, in logical blocks, 0's-based | The write size guaranteed atomic. Matters for checkpoint integrity. |
| `wctemp` / `cctemp` | **Warning** and **Critical** Composite Temperature Thresholds, in Kelvin | The two numbers the SMART log's `warning_temp_time` and `critical_comp_time` counters are measured against. This is the hook into lesson 07. |
| `tnvmcap` / `unvmcap` | Total / Unallocated NVM Capacity, in **bytes** | Over-provisioning check. |
| `hmpre` / `hmmin` | Host Memory Buffer preferred / minimum size, in 4 KiB units | Consumer drives borrow host DRAM for their FTL. If a "server" drive reports these, you bought the wrong SKU. |
| `cntrltype` | Controller Type (I/O, discovery, administrative) | On an NVMe-oF fabric you will meet discovery controllers. |
| `kas` | Keep Alive Support granularity | Fabrics only; the keep-alive timer that decides how fast a dead path is noticed. |

**`nvme smart-log /dev/nvme0`** — SMART / Health Information, Log Identifier `02h`. The full field list with the units that trip people:

```
$ nvme smart-log /dev/nvme0
Smart Log for NVME device:nvme0 namespace-id:ffffffff
critical_warning                    : 0
temperature                         : 42 C
available_spare                     : 100%
available_spare_threshold           : 10%
percentage_used                     : 3%
endurance group critical warning summary: 0
data_units_read                     : 2,914,882,143
data_units_written                  : 411,203,884
host_read_commands                  : 28,441,092,113
host_write_commands                 : 3,880,214,772
controller_busy_time                : 41,229
power_cycles                        : 37
power_on_hours                      : 14,802
unsafe_shutdowns                    : 4
media_errors                        : 0
num_err_log_entries                 : 0
Warning Temperature Time            : 0
Critical Composite Temperature Time : 0
Temperature Sensor 1                : 42 C
Temperature Sensor 2                : 51 C
Thermal Management T1 Trans Count   : 0
Thermal Management T2 Trans Count   : 0
Thermal Management T1 Total Time    : 0
Thermal Management T2 Total Time    : 0
```

| Field | Exact definition | Read it as |
|---|---|---|
| `critical_warning` | Bitmask. bit0 available spare below threshold · bit1 temperature above/below a threshold · bit2 NVM reliability degraded · bit3 media placed in read-only mode · bit4 volatile memory backup device failed · bit5 Persistent Memory Region read-only/unreliable | **Anything non-zero is a page.** Decode the bit, don't just note "non-zero". |
| `temperature` | **Composite** temperature, stored in **Kelvin** in the log (nvme-cli converts to °C for display). Implementation-defined combination of controller and namespace sensors — not necessarily any physical point. | Compare against `wctemp`/`cctemp` from `id-ctrl`. |
| `available_spare` / `available_spare_threshold` | Normalised percentage 0–100 of remaining spare blocks, and the threshold below which an AEN fires. | Spare falling is the real wear signal, ahead of `percentage_used`. |
| `percentage_used` | Vendor estimate of consumed endurance. 100 = rated endurance consumed. **Allowed to exceed 100**, and values above 254 are reported as 255. Updated once per power-on hour. | An estimate, not a measurement. Trend it; don't alert on a single reading. |
| `data_units_read` / `data_units_written` | Number of 512-byte units, **reported in thousands and rounded up**. A value of 1 means 1–1000 units of 512 B. | **Bytes = value × 1000 × 512.** Getting this wrong by 1000× is the classic mistake. |
| `host_read_commands` / `host_write_commands` | Count of completed read/write commands. | Divide `data_units_*` by these to get the average I/O size the host is actually issuing — a direct check on your loader's request shape. |
| `controller_busy_time` | Minutes during which at least one command was outstanding on an I/O queue (specifically: a command was published by an SQ Tail doorbell write and its CQE has not yet been posted). | Divide by `power_on_hours × 60` for a duty-cycle percentage. A drive at 5% busy is not your bottleneck. |
| `power_cycles` / `power_on_hours` / `unsafe_shutdowns` | Self-explanatory. `unsafe_shutdowns` increments when power is lost without a `CC.SHN` shutdown notification. | Rising `unsafe_shutdowns` in a fleet means a power or firmware problem, not a drive problem. |
| `media_errors` | Occurrences of unrecovered data-integrity errors — uncorrectable ECC, CRC failure, LBA tag mismatch. | Non-zero on a young drive is a replacement conversation. |
| `num_err_log_entries` | Lifetime count of Error Information log entries. | Follow up with `nvme error-log`. |
| `warning_temp_time` / `critical_comp_time` | **Minutes** spent at or above `wctemp` (but below `cctemp`), and at or above `cctemp`. Both read 0 if the corresponding threshold is 0. | **This is thermal throttling, measured in wall-clock minutes.** A non-zero `warning_temp_time` on a data drive is a cooling finding, exactly like lesson 07's GPU thermal slowdown. |
| `temp_sensor[1..8]` | Per-sensor temperature in Kelvin; a sensor that is not implemented reports 0. | Sensor 2 running 10 °C above sensor 1 usually means the controller ASIC, not the NAND, is the hot spot. |
| `thm_temp1/2_trans_count`, `thm_temp1/2_total_time` | Host-Controlled Thermal Management: how many times the controller dropped to a lower-power state to shed heat, and how many **seconds** it spent there. | The NVMe equivalent of `SW Thermal Slowdown`. Non-zero means the drive is *already* trading throughput for temperature. |

**`nvme show-regs /dev/nvme0 -H`** — the controller registers, when you want to confirm the queue model on the actual hardware:

```
$ nvme show-regs /dev/nvme0 -H
cap     : 3c033fff
        Maximum Queue Entries Supported (MQES): 4096
        Contiguous Queues Required (CQR): 1
        Doorbell Stride (DSTRD): 0
        ...
vs      : 20000
        NVMe specification 2.0
cc      : 460001
        I/O Completion Queue Entry Size (IOCQES): 16 bytes
        I/O Submission Queue Entry Size (IOSQES): 64 bytes
        Enable (EN): 1
csts    : 1
        Ready (RDY): 1
```

Read across: `MQES: 4096` means the controller supports 4096-entry queues, so Linux's `q_depth` will be `min(4096, 1024) = 1024` at the default `io_queue_depth`. `DSTRD: 0` means a stride of 1 dword — doorbells are 4 bytes apart, so SQ1's tail doorbell is at BAR0 offset `0x1000 + 2*4 = 0x1008`. `IOSQES: 64 / IOCQES: 16` confirms the entry sizes.

### 8. GPUDirect Storage: the mechanism, and the two layers that fail independently

**What it removes.** Without GDS, a read into GPU memory is two copies:

```
  NVMe ──DMA──▶ pinned host bounce buffer ──cudaMemcpy──▶ GPU HBM
                ↑                            ↑
                consumes host DRAM bandwidth │ consumes a CPU core to orchestrate
```

With GDS, the NVMe controller's DMA engine writes **directly into the GPU's BAR-mapped HBM**:

```
  NVMe ──DMA (peer-to-peer over PCIe)──▶ GPU HBM
```

Eliminated: one full copy, the CPU cycles that drove it, and the host-DRAM bandwidth it consumed. At H100-class rates that third item is not incidental — a bounce path moving 14 GB/s costs 28 GB/s of host DRAM bandwidth (one write, one read) out of ~307 GB/s per socket, per stream.

**Two layers, two independent failure surfaces:**

| Layer | What it is | How a failure looks |
|---|---|---|
| **`cuFile` / `libcufile`** | The userspace API applications call — `cuFileDriverOpen`, `cuFileHandleRegister`, `cuFileBufRegister`, `cuFileRead`/`cuFileWrite`, plus the batch and async variants. Part of the CUDA userspace stack. | A `CUfileError_t` return from the API. Application-visible. |
| **`nvidia-fs.ko`** | The kernel module that performs the peer-to-peer DMA registration between the block device and the GPU's PCIe BAR memory, hooking the filesystem's `O_DIRECT` path. | A kernel-side registration failure in `dmesg` / the `nvidia-fs` stats. **Not** visible in the application's `cuFile` return codes. |

**The trap: "GDS is enabled" is not the claim "GDS is on the fast path."** `cuFile` has an explicit **compatibility mode**, controlled by `allow_compat_mode` in `/etc/cufile.json`. When it is `true` and the fast path is unavailable — mount not GDS-capable, `nvidia-fs.ko` not loaded, unsupported filesystem, unsupported peer-to-peer path — the library **silently stages through a CPU bounce buffer and returns success**. Your code is correct, your calls succeed, and you have none of the benefit. As of GDS 1.16 the `gdscheck` output was extended to name which of the modes (`p2pdma`, `nvfs`, `compat`) `libcufile` supports, precisely because this ambiguity kept biting people.

```
$ /usr/local/cuda/gds/tools/gdscheck -p
 GDS release version: 1.16.0.x
 nvidia_fs version:  2.24 libcufile version: 2.12
 Platform: x86_64
 ============
 ENVIRONMENT:
 ============
 =====================
 DRIVER CONFIGURATION:
 =====================
 NVMe               : Supported
 NVMeOF             : Supported
 SCSI               : Unsupported
 ScaleFlux CSD      : Unsupported
 NVMesh             : Unsupported
 DDN EXAScaler      : Unsupported
 LUSTRE             : Unsupported
 NFS                : Unsupported
 WekaFS             : Supported
 Userspace RDMA     : Unsupported
 --Mellanox PeerDirect : Enabled
 --rdma library        : Not Loaded
 --rdma devices        : Not configured
 =====================
 CUFILE CONFIGURATION:
 =====================
 properties.use_compat_mode : true
 properties.gds_rdma_write_support : true
 properties.max_direct_io_size_kb : 16384
 properties.max_device_cache_size_kb : 131072
 properties.max_device_pinned_mem_size_kb : 33554432
 properties.posix_pool_slab_size_kb : 4 1024 16384
 =====================
 GPU INFO:
 =====================
 GPU index 0 NVIDIA H100 80GB HBM3 bar:1 bar size (MiB):131072 supports GDS
 GPU index 4 NVIDIA H100 80GB HBM3 bar:1 bar size (MiB):131072 supports GDS
 =====================
 IOMMU: disabled
 Platform verification succeeded
```

Read it in three passes:

1. **Driver configuration** — is the *filesystem type* under your mount supported at all? `NVMe: Supported` covers a local ext4/xfs on a raw NVMe; `NFS: Unsupported` means an NFS-backed dataset will always run in compat mode no matter how good the topology is.
2. **`properties.use_compat_mode : true`** — the fallback is armed. That is usually what you want for robustness, but it also means **a failure of the fast path will never surface as an error**. If you are benchmarking, flip it to `false` temporarily so that a fast-path failure becomes a loud `cuFile` error rather than a quiet slowdown.
3. **GPU INFO / IOMMU** — every GPU must report `supports GDS`, and note the IOMMU line. With the IOMMU enabled in translation mode, peer-to-peer transactions need IOMMU mappings for both the source and target device; many platforms only support GDS's fast path with the IOMMU off or in passthrough (`iommu=pt`). This is a very common reason a topologically perfect setup still runs compat.

`properties.max_direct_io_size_kb : 16384` is worth noting too: cuFile splits requests larger than this, so a 64 MiB `cuFileRead` becomes four 16 MiB operations — the same "your effective queue depth is a multiple of what you thought" effect as MDTS.

**What GDS requires from topology.** Peer-to-peer DMA means the NVMe controller issues PCIe writes addressed to the GPU's BAR. For that to be fast, the two devices should sit under a **common PCIe switch** (case 1 in §5), because the switch can route the transaction directly between downstream ports without it ever reaching the CPU. Under a common root complex (case 2) it depends on whether the CPU supports and enables P2P forwarding across root ports; across sockets (case 3) it is not a supported fast path. So: **`PIX`/`PXB` = GDS can be fast. `PHB` = platform-dependent. `SYS` = expect compat mode.**

**Diagnostic order when GDS throughput does not move after a placement fix:**

1. `gdscheck -p` — is the fast path even available for this GPU and this mount type? Is `use_compat_mode` masking a failure?
2. `lsmod | grep nvidia_fs` and `dmesg | grep -i nvidia-fs` — did the kernel module load, and did the peer-to-peer registration succeed? This layer fails without any `cuFile` error.
3. `cat /proc/driver/nvidia-fs/stats` — the module's own counters distinguish reads that took the direct path from reads that were staged.
4. Check the IOMMU state (`dmesg | grep -i -e DMAR -e IOMMU`, or the `IOMMU:` line in `gdscheck`).
5. Only then question the topology.

**Benchmarking it.** `gdsio`, shipped with the GDS package, is the reference tool and lets you compare the same workload with and without the direct path:

```
# -x 0 : storage → GPU memory via GDS (the fast path)
# -x 1 : storage → CPU memory, then CPU → GPU (the bounce path)
$ gdsio -f /mnt/data0/testfile -d 0 -w 8 -s 8G -i 1M -x 0 -I 1
IoType: READ XferType: GPUD Threads: 8 DataSetSize: 8388608(KiB) IOSize: 1024(KiB)
Throughput: 12.914 GiB/sec, Avg_Latency: 619.42 usecs ops: 8192 total_time 0.619 secs

$ gdsio -f /mnt/data0/testfile -d 0 -w 8 -s 8G -i 1M -x 1 -I 1
IoType: READ XferType: CPUONLY Threads: 8 DataSetSize: 8388608(KiB) IOSize: 1024(KiB)
Throughput: 6.201 GiB/sec, Avg_Latency: 1289.90 usecs ops: 8192 total_time 1.290 secs
```

Representative numbers, not captured from a specific machine — but the *shape* is the finding: run this pair on a `PIX` GPU↔drive path and on a `SYS` path, and the delta between the four results is your entire storage-placement argument, quantified.

### 9. Worked capacity math

Two calculations every platform engineer sizing a GPU node should be able to do on a whiteboard.

**(a) How much read bandwidth do N GPUs need?**

```
  required_read_BW  =  N_gpus × samples_per_sec_per_gpu × bytes_per_sample
                       ─────────────────────────────────────────────────────
                                     cache_hit_factor
```

`cache_hit_factor` accounts for re-reads served from the page cache; for a dataset far larger than RAM it is 1.

*Case 1 — image classification, ImageNet-scale JPEGs.* ImageNet-1k's training set is ~1.28M images in ~140 GB, so ~110 KB per image on average.

```
  N = 8 GPUs
  throughput = 3,000 images/s/GPU        ← ASSUMPTION: plug your own measured rate
  sample size = 110 KB (average JPEG)
  cache hits: none (dataset >> RAM)

  8 × 3,000 × 110e3  =  2.64 GB/s

  → one Gen5 ×4 drive (14 GB/s rated) is 5× oversized for this.
    Placement barely matters. Even a cross-socket path has headroom.
```

*Case 2 — LLM pretraining on pre-tokenised shards.* Tokens are `uint16` or `uint32` on disk; 2 bytes/token is typical.

```
  N = 8 GPUs
  throughput = 40,000 tokens/s/GPU       ← ASSUMPTION
  sample size = 2 bytes/token

  8 × 40,000 × 2  =  640 KB/s

  → six hundred kilobytes per second. LLM pretraining is essentially never
    read-bandwidth bound. If someone tells you their LLM pretraining job is
    storage-bound on the *read* path, look at checkpointing (below) instead.
```

*Case 3 — video or high-resolution multimodal, decoded on the host.*

```
  N = 8 GPUs
  throughput = 400 clips/s/GPU           ← ASSUMPTION
  sample size = 3.2 MB (a short pre-decoded clip)

  8 × 400 × 3.2e6  =  10.24 GB/s

  → now you need ~73% of one Gen5 ×4 drive's rated read bandwidth, sustained.
    A cross-socket path sharing a ~48 GB/s UPI link with everything else on
    the box will not deliver this reliably. Placement is now load-bearing,
    and so is queue depth: 10.24e9 / 131,072 = 78,125 IOPS, × 80 µs latency
    = 6.3 outstanding requests minimum, so QD 16+ across ≥4 workers.
```

**The lesson from running all three:** whether storage placement matters is a property of your *bytes per sample*, not of your GPU count. Do this arithmetic before you argue about topology — and note the second-order effect the three cases share: whatever the read bandwidth is, it must be delivered *while* the GPU is also pulling gradients over the NIC and pushing checkpoints back out.

**(b) How long does a checkpoint take?**

Use the standard mixed-precision Adam accounting (ZeRO, Rajbhandari et al., SC20): for a model with Ψ parameters, training state is `2Ψ` (bf16/fp16 weights) + `2Ψ` (bf16/fp16 gradients) + `KΨ` optimizer state with **K = 12** for Adam (4 B fp32 master weights + 4 B momentum + 4 B variance) = **16 bytes per parameter**. A resumable checkpoint normally persists weights plus optimizer state but not gradients: **14 bytes per parameter**.

```
  Model: 70B parameters, Adam, mixed precision

  weights-only (bf16)        : 70e9 × 2  =  140 GB
  full resumable checkpoint  : 70e9 × 14 =  980 GB

  ── written to one Gen5 ×4 NVMe at 6.5 GB/s sustained write ──
     980 / 6.5        = 151 s   per checkpoint
  ── written to an 8-drive RAID-0 array, host-limited to ~40 GB/s ──
     980 / 40         = 24.5 s
  ── written over a single 200 GbE storage NIC at 22 GB/s effective ──
     980 / 22         = 44.5 s
  ── written over a single 25 GbE link at 2.8 GB/s effective ──
     980 / 2.8        = 350 s   (5 min 50 s)  ← this is how jobs die

  ── as a fraction of wall clock, checkpointing every 30 minutes ──
     one Gen5 drive   : 151 / 1800   =  8.4%  of training time
     8-drive array    :  24.5 / 1800 =  1.4%
     25 GbE           : 350 / 1800   = 19.4%   ← one fifth of the cluster,
                                                 burned writing checkpoints

  ── and the placement question ──
     If that write path crosses UPI, the effective write bandwidth is your
     share of one inter-socket link, not the array's rating. On the 8-drive
     case that can turn 24.5 s into well over a minute — a 3-4% wall-clock
     regression that no dashboard attributes to storage placement.
```

Sanity-check the write direction against the same SMART fields: `data_units_written × 1000 × 512` should grow by ≈980 GB per checkpoint on the node doing the writing. If it grows by 8× that, someone configured every rank to write a full copy instead of a shard.

### 10. Generalising: network-attached storage and NVMe over Fabrics

Local NVMe is the right first mental model, but at real cluster scale training data usually lives on a parallel filesystem (Lustre, WekaFS, VAST, DDN) or an S3-compatible object store, reached over the network. **The placement question does not disappear — it moves.** Instead of "which NVMe is on this GPU's switch," it becomes:

> **Which NIC serves this GPU's storage traffic, and is that NIC on the GPU's switch?**

The `PIX`/`PXB`/`PHB`/`NODE`/`SYS` legend applies unchanged, just to a storage-serving NIC instead of a local drive. On a rail-aligned 8-GPU node this is a real design decision: the compute fabric NICs are already paired to GPUs by the baseboard layout, so the storage NIC is often the odd one out, and it is easy for it to end up on the wrong root complex.

**NVMe over Fabrics (NVMe-oF)** is the protocol that carries the NVMe command set across a network — over RDMA (RoCE or InfiniBand), TCP, or Fibre Channel — so a remote namespace appears as a local `/dev/nvmeXnY`. The queue-pair model from §2 survives intact: an NVMe-oF host creates queue pairs against a remote controller, and each pair maps onto a fabric connection (an RDMA queue pair, or a TCP socket). The doorbell writes become fabric capsules, but the concurrency model, the queue depth arithmetic in §4, and the `nvme-cli` fields in §7 all still apply. `gdscheck`'s `NVMeOF: Supported` line means the GDS fast path can reach remote namespaces too — the NIC DMAs straight into GPU HBM.

The commands change slightly:

```
$ nvme list-subsys
nvme-subsys0 - NQN=nqn.2019-08.org.example:storage-array-01
\
 +- nvme0 rdma traddr=10.20.0.11,trsvcid=4420 live
 +- nvme1 rdma traddr=10.20.0.12,trsvcid=4420 live      # multipath, two portals

$ nvme discover -t rdma -a 10.20.0.11 -s 4420
$ nvme connect -t rdma -a 10.20.0.11 -s 4420 -n nqn.2019-08.org.example:ns1
```

Two paths `live` in `list-subsys` means native NVMe multipath is active — and it is worth knowing which of those two portals is reached through which NIC, because if one is on the GPU's root complex and one is not, your effective bandwidth depends on which path the multipath policy picked. Check with `nvme list-subsys -o json` and correlate against `nvidia-smi topo -m`.

**Do not stop at "the local drive is fine."** If your production storage is remote, check the storage-serving NIC's relationship to each GPU exactly as you would a compute NIC's.

## Perspectives

**Developer.** A `DataLoader` author sees `num_workers`, `prefetch_factor`, `pin_memory` — the §4 knobs — and nothing else. GDS/cuFile is normally invoked by the framework or a storage vendor's plugin, not hand-written. The developer's realistic job is picking a GDS-aware data-loading backend and setting the concurrency high enough to reach the hardware ceiling; they cannot see or fix placement.

**Operator.** Storage-to-GPU placement belongs on the node acceptance test alongside PCIe link state (lesson 03): capture `nvidia-smi topo -m`, the per-NVMe `numa_node`, and `gdscheck -p` at bring-up and diff against the reference layout. Finding it at bring-up costs minutes; finding it because a step-time histogram went bimodal costs a week.

**Kernel / driver.** Almost everything in §2 and §3 is the kernel getting out of the way: one queue per core so submissions never contend, `scheduler=none` because reordering is pure overhead on a seek-free device, MSI-X affinity so the completion interrupt lands on the submitting core, and `O_DIRECT`/`io_uring` so the page cache is optional rather than mandatory. GDS is the logical endpoint of that trajectory — the last remaining copy, removed.

**Storage vendor.** WEKA, VAST and DDN all built GDS integrations because the CPU bounce-buffer path becomes the binding constraint at H100-class GPU bandwidth. Their public benchmark wars are a market-tested confirmation of this lesson's core claim: the limiting factor is usually the *path*, not the media.

**Economics.** This is the cheapest fix in the module — no capital spend, no code change, just a scheduling or mount decision. The 41% wall-clock loss in the §6 timeline, at $2–3/GPU-hr across a fleet, is invisible to every default dashboard.

## Real-world use cases

- **NVIDIA GPUDirect Storage documentation — compatibility mode and `gdscheck`.** The GDS troubleshooting and configuration guides document `allow_compat_mode` in `/etc/cufile.json`, the conditions that trigger fallback (mount not GDS-enabled, `nvidia-fs.ko` unavailable, unsupported filesystem), and — from GDS 1.16 — a `gdscheck` output extended to name which of `p2pdma`, `nvfs` and `compat` `libcufile` supports. **What it shows:** the "silently slow" failure mode is not folklore, it is a documented, configurable behaviour, and NVIDIA extended their own tooling because operators kept being unable to tell the two apart. `https://docs.nvidia.com/gpudirect-storage/troubleshooting-guide/index.html`
- **VAST Data — "NVIDIA GPU Direct Storage: The VAST Data Story."** A storage vendor's account of implementing GDS in a production filesystem. **What it shows:** what GDS integration looks like from the storage side, and why a vendor invests in the direct-DMA path rather than accepting the bounce buffer. `https://www.vastdata.com/blog/nvidia-gpu-direct-for-storage-gds-the-vast-data-story`
- **Blocks & Files — "WekaIO races out of the blocks in GPUDirect storage race" (2020).** An early independently-tested head-to-head GDS benchmark on a DGX-2 with InfiniBand-attached storage: **97.9 GB/s (WekaIO) and 92.6 GB/s (VAST)**. **What it shows:** real measured GDS throughput at the point GDS became production-viable. Cite it explicitly as a **2020, DGX-2-era snapshot** — it is a historical marker, not a current expectation. `https://blocksandfiles.com/2020/11/03/gpudirect-storage-race/`
- **Oracle Cloud Infrastructure Blog — WEKA on OCI H100 compute.** Reports **192 GiB/s sequential-read GDS throughput on a single client node**. **What it shows:** how far the high end has moved since 2020 — and note *how* it gets there: that figure is roughly 3× a single Gen5 ×16 link's ~63 GB/s ceiling, so it is necessarily aggregated across multiple GPUs and NICs on the node, not one link. That aggregation only works if every GPU↔NIC pair is correctly placed, which is this lesson's whole argument. `https://blogs.oracle.com/cloud-infrastructure/accelerate-ai-performance-weka-converged-storage`
- **CoreWeave Docs — "About CoreWeave Storage."** Describes CoreWeave's AI Object Storage serving data directly to GPU nodes, including **Tensorizer** loading tensors from S3/HTTP endpoints toward GPU memory at up to 2 GB/s per GPU across large fleets. **What it shows:** the neocloud-scale, network-attached version of this lesson's placement problem — and a concrete per-GPU bandwidth target to size against. `https://docs.coreweave.com/docs/products/storage`

## Worked example

An 8-GPU host. Locate storage relative to the GPUs, trace one data path, confirm the GDS mode, and measure the difference. Transcripts are representative; the commands and field names are real.

### Step 1 — inventory and map identity across tools

```
$ nvme list
Node          SN             Model                  Namespace  Usage                  Format         FW Rev
/dev/nvme0n1  Y0X0A00ZT1L8   KIOXIA KCMYXRUG3T84            1  3.84 TB / 3.84 TB      512 B + 0 B    0104
/dev/nvme1n1  Y0X0A00ZT1M2   KIOXIA KCMYXRUG3T84            1  3.84 TB / 3.84 TB      512 B + 0 B    0104

$ for n in /sys/block/nvme*n1; do
    dev=$(basename $n)
    bdf=$(basename $(readlink -f $n/device/device))
    echo "$dev  bdf=$bdf  numa_node=$(cat $n/device/device/numa_node)  hwq=$(ls $n/mq | wc -l)"
  done
nvme0n1  bdf=0000:c1:00.0  numa_node=0  hwq=128
nvme1n1  bdf=0000:04:00.0  numa_node=1  hwq=128
```

Two drives, one per socket, 128 hardware queues each. **Write the BDFs down** — every later statement depends on them.

### Step 2 — the PCIe tree

```
$ lspci -tv | grep -iE 'nvme|nvidia|mellanox|pci bridge'
-+-[0000:c0]-+-01.1-[c1]----00.0  NVIDIA Corporation GH100 [H100 SXM5]
 |           +-01.2-[c2]----00.0  KIOXIA Corporation NVMe SSD Controller
 |           \-03.1-[c3]----00.0  Mellanox Technologies MT2910 [ConnectX-7]
 \-[0000:00]-+-01.1-[01]----00.0  NVIDIA Corporation GH100 [H100 SXM5]
             +-05.1-[04]----00.0  KIOXIA Corporation NVMe SSD Controller
             \-05.2-[05]----00.0  Mellanox Technologies MT2910 [ConnectX-7]
```

Read it: `[0000:c0]` and `[0000:00]` are two separate top-level host bridges — two root complexes, one per socket. Under `c0`: a GPU at `c1`, an NVMe at `c2`, a NIC at `c3`. Under `00`: a GPU at `01`, an NVMe at `04`, a NIC at `05`. Structurally, each socket has a matched GPU/NVMe/NIC set.

Now check the *pipes*, not just the tree — a `PIX` relationship over a link that trained at half width is still half bandwidth (lesson 03):

```
$ for bdf in 0000:c1:00.0 0000:c2:00.0 0000:04:00.0; do
    echo "== $bdf"; sudo lspci -vvv -s $bdf | grep -E 'LnkCap:|LnkSta:'; done
== 0000:c1:00.0
        LnkCap: Port #0, Speed 32GT/s, Width x16, ASPM not supported
        LnkSta: Speed 32GT/s, Width x16
== 0000:c2:00.0
        LnkCap: Port #0, Speed 32GT/s, Width x4, ASPM not supported
        LnkSta: Speed 32GT/s, Width x4
== 0000:04:00.0
        LnkCap: Port #0, Speed 32GT/s, Width x4, ASPM not supported
        LnkSta: Speed 16GT/s (downgraded), Width x4
```

`nvme1n1` (`0000:04:00.0`) trained at **Gen4 speed on a Gen5-capable link** — `16GT/s (downgraded)`. Its ceiling is ~7.9 GB/s instead of ~15.75 GB/s, before any topology consideration. `nvme smart-log` will show nothing wrong; only `lspci -vvv` catches it.

### Step 3 — the GPU-aware view

```
$ nvidia-smi topo -m
        GPU0    GPU1    NIC0    NIC1    CPU Affinity  NUMA Affinity
GPU0     X      SYS     PXB     SYS     0-47          0
GPU1    SYS      X      SYS     PXB     48-95         1
NIC0    PXB     SYS      X      SYS
NIC1    SYS     PXB     SYS      X

Legend:
  X    = Self
  SYS  = Connection traversing PCIe as well as the SMP interconnect between
         NUMA nodes (e.g., QPI/UPI)
  NODE = Connection traversing PCIe as well as the interconnect between PCIe
         Host Bridges within a NUMA node
  PHB  = Connection traversing PCIe as well as a PCIe Host Bridge (typically the CPU)
  PXB  = Connection traversing multiple PCIe bridges (without traversing the
         PCIe Host Bridge)
  PIX  = Connection traversing at most a single PCIe bridge
  NV#  = Connection traversing a bonded set of # NVLinks
```

**`nvidia-smi topo -m` does not list NVMe devices.** That is its blind spot, and it is why step 1's BDF mapping matters: you resolve the NVMe's position from `lspci -tv` and `numa_node`, then reason about it using the same legend. Doing that here:

- **GPU0 (`c1`, NUMA 0) ↔ nvme0n1 (`c2`, NUMA 0)** — same root complex, sibling ports under it, no host-bridge traversal in the tree. Effectively `PXB`. **This is GPU0's local drive.**
- **GPU0 ↔ nvme1n1 (`04`, NUMA 1)** — different root complex, different socket. **`SYS`.** Every read crosses UPI.
- **GPU0 ↔ NIC0** — `PXB`, confirmed by the matrix. GPU0's rail-aligned NIC, and the right one for both GPUDirect RDMA and NVMe-oF storage traffic destined for GPU0.
- Mirror everything for GPU1 / nvme1n1 / NIC1.

Conclusion for the deliverable: *GPU0's local drive is nvme0n1 (same root complex as GPU0, and sharing it with NIC0 — a GDS-ready arrangement for both local and fabric-attached reads). GPU0 reading nvme1n1 is a cross-socket `SYS` path and must be avoided. Additionally, nvme1n1's PCIe link trained at Gen4 instead of Gen5 — a separate, independent defect worth a reseat/BIOS check.*

### Step 4 — confirm GDS is on the fast path, not just correctly placed

```
$ /usr/local/cuda/gds/tools/gdscheck -p | grep -E 'NVMe |use_compat_mode|IOMMU|supports GDS'
 NVMe               : Supported
 properties.use_compat_mode : true
 GPU index 0 NVIDIA H100 80GB HBM3 bar:1 bar size (MiB):131072 supports GDS
 GPU index 1 NVIDIA H100 80GB HBM3 bar:1 bar size (MiB):131072 supports GDS
 IOMMU: disabled

$ lsmod | grep nvidia_fs
nvidia_fs             311296  0

$ cat /proc/driver/nvidia-fs/stats | head -12
GDS Version: 1.16.0.x
NVFS statistics(ver: 4.0)
NVFS Driver(version: 2.24)
Mellanox PeerDirect Supported: True
IO stats: Enabled, peer IO stats: Enabled
Logging level: info

Active Shadow-Buffer (MiB): 0
Active Process: 1
Reads   : n=1048576 ok=1048576 err=0 readMiB=1048576 io_state_err=0
Reads   : Bandwidth(MiB/s)=13224 Avg-Latency(usec)=594
Writes  : n=0 ok=0 err=0 writeMiB=0 io_state_err=0
```

`Reads: err=0` and a real `Bandwidth` figure mean the direct path is genuinely being used. If reads were falling back, you would see them counted as shadow-buffer activity here while `cuFileRead` kept returning success.

Now do the comparison that produces a number:

```
# GPU0 reading its LOCAL drive, GDS fast path
$ gdsio -f /mnt/data0/bench -d 0 -w 8 -s 16G -i 1M -x 0 -I 1
Throughput: 13.02 GiB/sec, Avg_Latency: 601.3 usecs

# GPU0 reading the CROSS-SOCKET drive, GDS requested
$ gdsio -f /mnt/data1/bench -d 0 -w 8 -s 16G -i 1M -x 0 -I 1
Throughput: 5.44 GiB/sec, Avg_Latency: 1438.7 usecs

# Same cross-socket drive, explicitly bounce-buffered (to separate the two effects)
$ gdsio -f /mnt/data1/bench -d 0 -w 8 -s 16G -i 1M -x 1 -I 1
Throughput: 5.11 GiB/sec, Avg_Latency: 1531.2 usecs
```

Read the three together, because the *pattern* is the diagnosis:

- Local + GDS: 13.02 GiB/s — near the drive's rating minus the Gen5 ×4 link's practical overhead. Good.
- Cross-socket + GDS requested: 5.44 GiB/s — **58% slower**.
- Cross-socket + explicit bounce: 5.11 GiB/s — **essentially the same as "GDS requested."** That near-equality is the signature of compat-mode fallback: asking for the fast path bought you 6%, which means you did not get the fast path.

To turn that inference into proof, temporarily disable the fallback and re-run:

```
$ sudo jq '.properties.allow_compat_mode = false' /etc/cufile.json | sudo tee /etc/cufile.json.new
$ sudo mv /etc/cufile.json.new /etc/cufile.json
$ gdsio -f /mnt/data1/bench -d 0 -w 8 -s 16G -i 1M -x 0 -I 1
cuFileRead failed : GPUDirect Storage not supported on current file / device combination
```

**Now** the failure is loud instead of silent. Restore `allow_compat_mode` to `true` afterwards — you want the fallback in production, you just do not want it to be invisible while you are measuring.

### Step 5 — rule out queue depth before blaming topology

Prove the loader is actually asking for enough work, using the drive's own counters:

```
$ nvme smart-log /dev/nvme0 | grep -E 'data_units_read|host_read_commands|controller_busy_time'
data_units_read                     : 2,914,882,143
host_read_commands                  : 28,441,092,113
controller_busy_time                : 41,229
```

```
  bytes read      = 2,914,882,143 × 1000 × 512      = 1.492e15 B  = 1.49 PB
  average I/O size = 1.492e15 / 28,441,092,113      = 52,461 B    ≈ 51 KiB

  busy fraction    = controller_busy_time (minutes) / (power_on_hours × 60)
                   = 41,229 / (14,802 × 60)         = 4.6%
```

Two findings in four lines. The average read is **51 KiB**, not the 1 MiB the framework config claims — something is splitting requests (check `max_hw_sectors_kb` against MDTS, and whether the loader is reading many small records rather than shards). And the controller is **busy only 4.6% of its powered life**, so over the drive's whole history it has been nowhere near saturated. Those two numbers together say "concurrency problem," and would have saved you from a topology hunt.

Confirm live with `fio`, which lets you sweep queue depth directly:

```
$ fio --name=seq --filename=/mnt/data0/bench --rw=read --bs=1M --direct=1 \
      --ioengine=io_uring --numjobs=1 --iodepth=1 --runtime=30 --time_based
  READ: bw=1623MiB/s (1702MB/s), io=47.6GiB, run=30001msec

$ fio --name=seq --filename=/mnt/data0/bench --rw=read --bs=1M --direct=1 \
      --ioengine=io_uring --numjobs=4 --iodepth=32 --runtime=30 --time_based
  READ: bw=12.7GiB/s (13.6GB/s), io=381GiB, run=30002msec
```

QD 1 gets **13%** of what QD 128 (4 jobs × 32) gets, on the identical drive with the identical placement. Publish that pair alongside the topology finding; it is the fastest way to shut down the "is it really the topology?" objection.

## Practice

Feeds the [Topology Teardown](../practice/topology-teardown/README.md) deliverable.

1. **Inventory and map identity.** Run `nvme list`, and for every namespace resolve its BDF and NUMA node with `readlink -f /sys/block/<dev>/device/device` and `cat .../numa_node`. Record the hardware queue count (`ls /sys/block/<dev>/mq | wc -l`), the scheduler, and `max_hw_sectors_kb`. Produce a table: device → BDF → NUMA node → hw queues.
2. **Read the controller.** For one drive, capture `nvme id-ctrl /dev/nvmeX -H` and record `mdts`, `sqes`, `cqes`, `wctemp`, `cctemp`, `oncs`, `vwc`, and `tnvmcap`. Convert `mdts` into a byte figure and check it against `max_hw_sectors_kb`. Capture `nvme show-regs /dev/nvmeX -H` and confirm `MQES`, `DSTRD`, `IOSQES`, `IOCQES`.
3. **Read the health log, with units.** Capture `nvme smart-log` for each drive. Convert `data_units_read`/`data_units_written` to bytes (`× 1000 × 512`), divide by `host_read_commands`/`host_write_commands` to get the average I/O size, and compute the busy fraction from `controller_busy_time` and `power_on_hours`. Explicitly note `warning_temp_time`, `critical_comp_time` and the thermal-management counters — a non-zero value there is a cooling finding that belongs in lesson 07's section of the teardown.
4. **Place storage on the topology.** Run `lspci -tv` and `nvidia-smi topo -m`. For **each GPU**, name its local NVMe (same root complex) and flag every GPU→NVMe pairing that is cross-socket. Since `topo -m` does not list NVMe rows, state explicitly how you derived each NVMe's position (from `lspci -tv` plus `numa_node`) — that reasoning is part of the deliverable. Also check `LnkCap` vs `LnkSta` on every NVMe, GPU and NIC and flag anything `(downgraded)`.
5. **Separate concurrency from placement.** Run `fio` with `--direct=1 --ioengine=io_uring` at `iodepth=1` and at `iodepth=32 --numjobs=4` against the *same* drive. Record both. Then compute, with Little's law, the outstanding-request count your target bandwidth requires at your request size and measured latency, and compare it to what your real data loader issues.
6. **Check the GDS mode, not just the topology.** If GDS matters for your workload, run `gdscheck -p` and record: the driver-configuration line for your filesystem type, `properties.use_compat_mode`, the `IOMMU:` line, and whether every GPU reports `supports GDS`. Then run `gdsio -x 0` (GDS) and `-x 1` (bounce) against a local drive and a cross-socket drive — four numbers. If the GDS and bounce numbers for the cross-socket drive are within ~10% of each other, that is compat-mode fallback; prove it by temporarily setting `allow_compat_mode: false` and capturing the resulting error.
7. **Do the capacity math for your workload.** Compute the required read bandwidth for your GPU count, samples/sec and bytes/sample, and the checkpoint write time for your model size at your measured write bandwidth (state whether you are checkpointing weights-only at 2 B/param or a full resumable state at ~14 B/param). Express checkpoint time as a percentage of wall clock at your checkpoint interval.
8. **If your data is network-attached**, repeat step 4 for the storage-serving NIC instead of a local drive, and capture `nvme list-subsys` if you are on NVMe-oF. Note which GPU(s) each storage NIC is local to, and whether multipath is selecting a portal reached through the right NIC.

**Acceptance:** a note in the Topology Teardown that states, per GPU: where its storage sits (relationship code, which drive or NIC is local, and how you derived it), the `gdscheck -p` verdict if GDS applies, and any cross-socket or link-degraded path. Plus the queue-depth sweep showing you eliminated concurrency as an explanation, the SMART-derived average I/O size and busy fraction, and the two capacity calculations with your own numbers. Paste the raw `nvidia-smi topo -m`, `lspci -tv`, `nvme smart-log` and `fio` outputs as evidence.

## Common pitfalls

1. **Blaming topology before counting outstanding I/Os.** A QD-1 single-threaded reader leaves a Gen5 drive ~87% idle regardless of placement (the §10 `fio` pair proves it on your own hardware). Run the queue-depth sweep first; it is cheaper and it is more often the answer.
2. **Assuming "GDS enabled" means the fast path is active.** With `allow_compat_mode: true` — the default — `cuFile` silently stages through a bounce buffer and returns success. `gdscheck -p`, `/proc/driver/nvidia-fs/stats`, and a temporary `allow_compat_mode: false` are how you find out. A `cuFile` return code will never tell you.
3. **Forgetting the IOMMU.** A topologically perfect `PIX` GPU↔NVMe pair can still run compat mode because peer-to-peer DMA needs IOMMU mappings for both devices, and many platforms only support the GDS fast path with the IOMMU disabled or in passthrough. Check the `IOMMU:` line in `gdscheck -p` before re-cabling anything.
4. **Misreading `data_units_read` by a factor of 1000.** The NVMe spec reports it in **thousands** of 512-byte units. `bytes = value × 1000 × 512`. Skip the 1000 and you will conclude a drive that has read 1.5 PB has read 1.5 TB, and size your cache tier accordingly.
5. **Treating `percentage_used` as a measurement.** It is a vendor *estimate*, allowed to exceed 100, capped in reporting at 255, and updated once per power-on hour. Trend it; alert on `critical_warning` bits and `available_spare` instead.
6. **Trusting `nvidia-smi topo -m` for storage.** It has no NVMe rows at all. You must resolve each drive's BDF and NUMA node from sysfs and place it on the tree yourself. Anyone who claims a GPU↔NVMe relationship code straight out of `topo -m` has not run it.
7. **Confusing a relationship code with a healthy link.** `PIX` counts *bridges traversed*; `LnkSta` reports *trained speed and width*. A `PIX` NVMe whose link trained at Gen4 ×2 is close and slow. Check both for every edge you care about — the worked example above found exactly that.
8. **Assuming a "misplaced NVMe" only ever means a local drive.** With network-attached storage the equivalent defect is a storage-serving NIC on the wrong root complex, or an NVMe-oF multipath policy that picked the portal reached through the far NIC. Same failure shape, different device, easy to miss out of habit.
9. **Ignoring the write path.** Read bandwidth sizing (§9a) and checkpoint write time (§9b) are different calculations with different answers, and a job can be comfortably read-provisioned while spending 20% of its wall clock writing checkpoints over the wrong link.
10. **Skipping `nvme.write_queues` on checkpoint-heavy jobs.** With a single queue set, a burst of checkpoint writes head-of-line-blocks the read stream feeding the next step. A dedicated write queue set is a one-line kernel parameter.

## Self-check

**(a) Explain the NVMe queue-pair model precisely enough to say what an SQ Tail doorbell write actually does.**

**Answer:** A queue pair is one Submission Queue and one Completion Queue, both circular buffers the *host* allocates in system memory. QID 0 is the Admin pair (capped at 4096 entries); I/O queue IDs run 1–65535. To issue a command the host writes a 64-byte SQE (opcode, Command Identifier, namespace, LBA, length, and PRP/SGL pointers to the data buffer's physical pages) into the SQ at its private tail index, then writes the new tail value to that queue's **SQ Tail doorbell** — a 4-byte MMIO store into the controller's BAR0 doorbell array at offset `0x1000 + qid*2*stride` dwords, where `stride = 1 << CAP.DSTRD`. That single store is the *only* MMIO transaction in the fast path, and it can publish many SQEs at once, which is how batching works. The controller then DMA-reads the SQE, DMA-moves the data to or from the buffer, DMA-writes a 16-byte CQE carrying the CID, a status field and a **phase tag** (flipped on each wrap, so the host can poll for new completions without any read from the device), optionally raises the CQ's MSI-X vector, and the host writes the CQ Head doorbell to free the slots.

**(b) A GPU shows 100% utilization and a training step time 70% above the FLOPs budget. Give the two competing hypotheses and the command that separates them.**

**Answer:** Hypothesis 1 is **placement**: the drive or storage NIC feeding this GPU is on the wrong root complex, so every read crosses UPI/Infinity Fabric and cannot be delivered at rated bandwidth. Hypothesis 2 is **concurrency**: the loader is issuing too few outstanding I/Os to keep the device busy, and the placement is irrelevant. Separate them with a queue-depth sweep on the *same* drive: `fio --direct=1 --ioengine=io_uring --bs=1M` at `--iodepth=1` versus `--numjobs=4 --iodepth=32`. If the deep run reaches near the drive's rating, the hardware path is fine and the loader is the problem. If both runs are capped at the same low number, the path is the problem — confirm by resolving the drive's BDF and NUMA node from sysfs and comparing against the GPU's NUMA affinity in `nvidia-smi topo -m`. Corroborate historically with `nvme smart-log`: `controller_busy_time / (power_on_hours × 60)` gives the drive's lifetime duty cycle, and `data_units_read × 1000 × 512 / host_read_commands` gives the average I/O size the host has actually been issuing.

**(c) What does GPUDirect Storage remove from the data path, what topology does it need, and how do you prove it is actually active?**

**Answer:** It removes the CPU bounce buffer — the staging copy through pinned host DRAM — letting the NVMe (or the NIC, for NVMe-oF and GDS-aware filesystems) DMA directly into GPU HBM through `nvidia-fs.ko` under the `cuFile` API. That eliminates one full copy, the CPU cycles that drove it, and the host-DRAM bandwidth it consumed (a bounce path moving 14 GB/s costs ~28 GB/s of DRAM bandwidth, one write plus one read). It needs the two devices close enough for peer-to-peer PCIe DMA: a shared PCIe switch (`PIX`/`PXB`) is the reliable case, a shared root complex (`PHB`) is platform-dependent, and cross-socket (`SYS`) is not a supported fast path. Proving it is active takes three checks, because "GDS enabled" and "GDS fast" are different claims: `gdscheck -p` (is the filesystem type supported, does every GPU report `supports GDS`, what is the IOMMU state, and is `use_compat_mode` armed); `/proc/driver/nvidia-fs/stats` (are reads counted with `err=0` and a real bandwidth figure, or as shadow-buffer activity); and the decisive test — set `allow_compat_mode: false` in `/etc/cufile.json` and re-run, so a fast-path failure becomes a `cuFileRead` error instead of a silent slowdown.

**(d) Size the read bandwidth for 8 GPUs at 400 clips/s/GPU with 3.2 MB clips, and say whether placement matters.**

**Answer:** `8 × 400 × 3.2e6 = 10.24 GB/s` sustained, assuming no page-cache hits (dataset ≫ RAM). That is ~73% of one Gen5 ×4 drive's 14 GB/s rating — so yes, placement is load-bearing: a cross-socket path shares a ~48 GB/s-per-direction UPI link with all coherence traffic and every other remote access on the box, and will not deliver 10 GB/s reliably. Queue depth is also load-bearing at this rate: at 128 KiB requests that is `10.24e9 / 131,072 = 78,125` IOPS, and at ~80 µs device latency Little's law gives `78,125 × 80e-6 ≈ 6.3` outstanding requests as the bare minimum — so at least QD 16 spread across four or more workers. Contrast with LLM pretraining on 2-byte tokens at 40,000 tokens/s/GPU: `8 × 40,000 × 2 = 640 KB/s`, where placement is irrelevant on the read path. **Whether storage placement matters is a property of bytes-per-sample, not of GPU count.**

**(e) How long does a full resumable checkpoint of a 70B-parameter Adam mixed-precision model take, and what is the wall-clock cost?**

**Answer:** Using the ZeRO accounting — `2Ψ` bf16 weights + `2Ψ` bf16 gradients + `12Ψ` Adam optimizer state (fp32 master weights + momentum + variance) = 16 bytes/param of training state, of which a resumable checkpoint persists weights plus optimizer state ≈ **14 bytes/param** — a 70B model is `70e9 × 14 = 980 GB`. At one Gen5 ×4 drive's ~6.5 GB/s sustained write that is **151 s**; across an 8-drive RAID-0 host-limited to ~40 GB/s, **24.5 s**; over a single 25 GbE link at ~2.8 GB/s effective, **350 s**. Checkpointing every 30 minutes, those are **8.4%**, **1.4%** and **19.4%** of wall clock respectively. And if the write path crosses UPI, the effective bandwidth is your share of one inter-socket link rather than the array's rating — which can double the array case, a 3–4% regression that no dashboard attributes to storage placement. Weights-only bf16 checkpoints are `70e9 × 2 = 140 GB`, seven times cheaper, which is why "save weights every N steps, save full state every M steps" is the standard compromise.

**(f) `nvidia-smi topo -m` shows GPU0 with `PXB` to NIC0. What does that tell you about GPU0's NVMe, and how do you actually find out?**

**Answer:** **Nothing.** `nvidia-smi topo -m` has no NVMe rows at all — it enumerates GPUs and NICs only. To place a drive you resolve its identity yourself: `readlink -f /sys/block/nvme0n1/device/device` gives the BDF, `cat /sys/block/nvme0n1/device/device/numa_node` gives its socket, and `lspci -tv` shows which top-level host bridge and which bridges sit between it and the GPU's BDF. Compare that against the GPU's `NUMA Affinity` column from `topo -m`. Same top-level bridge and a sibling port below it means same root complex (effectively `PXB`); a different top-level bridge means `SYS`. Then, separately, check `lspci -vvv -s <bdf> | grep -E 'LnkCap|LnkSta'` on the drive, because a correctly placed drive on a link that trained at Gen4 ×2 is still only a quarter of the bandwidth you budgeted for, and no topology tool will mention it.

## Connections & what's next

This lesson generalises lesson 05's core claim — physical co-location is a policy decision, not a given — from Kubernetes' CPU/memory/GPU alignment to a fourth resource that Kubernetes has **no mechanism to align at all**. The `PIX`/`PXB`/`PHB`/`NODE`/`SYS` reading skill from lessons 03 and 04 transfers unchanged; what is new is that you must resolve the device's position yourself, from sysfs and `lspci`, because the GPU-side tool does not report it. The economics are identical to the NUMA-misalignment story: invisible in coarse utilization metrics, free to fix once diagnosed.

Two threads run straight into lesson 07. The first is thermal: `nvme smart-log`'s `warning_temp_time`, `critical_comp_time` and `thm_temp1/2_total_time` counters are the storage equivalent of a GPU's `SW Thermal Slowdown` bit — a device trading throughput for temperature, measured in wall-clock minutes, in a chassis where the GPUs are already the dominant heat source. The second is power: eight NVMe drives at 20–25 W each under sustained load is 160–200 W of a node's budget that has to come out of the same PDU circuit as the GPUs. **Lesson 07** covers power and thermals as measurable undelivered FLOPs, completing the "GPU at 100% util, throughput half spec" diagnostic tree that started with PCIe link state in lesson 03. All of it converges in the **lesson 08 capstone**, where you reconcile GPU, CPU, NIC and NVMe placement — plus power and thermal state — into one topology diagram on a real node.

## References & further reading

**Primary sources**

- **NVM Express Base Specification** (NVM Express, Inc.) — `https://nvmexpress.org/specifications/` — the authority on the queue-pair model: SQ/CQ as host-memory circular buffers, the QID-0 admin pair with its 4096-entry cap, I/O queue IDs 1–65535, the 64-byte SQE / 16-byte CQE, the phase tag, the doorbell array at BAR0 offset 0x1000 with stride `1 << CAP.DSTRD`, and `CAP.MQES` as a 0's-based maximum.
- **`linux-nvme/libnvme` — `src/nvme/types.h`** — `https://github.com/linux-nvme/libnvme/blob/master/src/nvme/types.h` — the definitive per-field documentation for `struct nvme_smart_log` and `struct nvme_id_ctrl`, including the two units this lesson calls out: `data_units_read`/`data_units_written` are 512-byte units **reported in thousands and rounded up**, and `temperature`/`temp_sensor[]` are **Kelvin**. Also the `enum nvme_smart_crit` bit definitions behind `critical_warning`.
- **`linux-nvme/nvme-cli` documentation** — `https://github.com/linux-nvme/nvme-cli/tree/master/Documentation` — command reference for `nvme list`, `id-ctrl`, `smart-log`, `show-regs`, `list-subsys`, `discover`, `connect`. `show-regs` mmaps the controller's PCI resource to read CAP/VS/CC/CSTS/AQA/ASQ/ACQ directly.
- **Linux kernel — `drivers/nvme/host/pci.c`** — `https://github.com/torvalds/linux/blob/master/drivers/nvme/host/pci.c` — the authority for the Linux-side numbers used here: `io_queue_depth` default **1024** (min 2, max 4095), `q_depth = min(CAP.MQES + 1, io_queue_depth)`, `dbs = bar + 4096`, `q_db = &dbs[qid * 2 * db_stride]`, and `nvme_max_io_queues()` = one queue per possible CPU plus `write_queues` plus `poll_queues`.
- **Linux kernel — "Multi-Queue Block IO Queueing Mechanism (blk-mq)"** — `https://docs.kernel.org/block/blk-mq.html` — per-CPU software queues, hardware queues, and why NVMe defaults to the `none` scheduler.
- **NVIDIA — GPUDirect Storage Installation and Troubleshooting Guide** — `https://docs.nvidia.com/gpudirect-storage/troubleshooting-guide/index.html` — `allow_compat_mode` / `use_compat_mode` in `/etc/cufile.json`, the conditions that trigger fallback, `gdscheck -p` output structure, `/proc/driver/nvidia-fs/stats`, and the IOMMU requirements for peer-to-peer DMA.
- **NVIDIA — GDS cuFile API Reference** — `https://docs.nvidia.com/gpudirect-storage/api-reference-guide/index.html` — the `cuFileDriverOpen` / `cuFileHandleRegister` / `cuFileBufRegister` / `cuFileRead` call sequence and the separation between the `cuFile` userspace layer and the `nvidia-fs` kernel layer.
- **PyTorch — `torch.utils.data.DataLoader`** — `https://pytorch.org/docs/stable/data.html` — `num_workers`, `prefetch_factor`, `pin_memory`; the knobs that determine how many outstanding I/Os your loader actually issues, which is what §4's Little's-law arithmetic is measured against.
- **Rajbhandari et al., "ZeRO: Memory Optimizations Toward Training Trillion Parameter Models" (SC20)** — `https://arxiv.org/abs/1910.02054` — the `2Ψ + 2Ψ + KΨ` mixed-precision Adam memory accounting with **K = 12** (fp32 master weights + momentum + variance), the basis for the 14–16 bytes-per-parameter checkpoint arithmetic in §9b.

**Real-world engineering**

- **Blocks & Files — "WekaIO races out of the blocks in GPUDirect storage race" (2020)** — `https://blocksandfiles.com/2020/11/03/gpudirect-storage-race/` — **what it shows:** an early independently-tested GDS benchmark (97.9 GB/s WekaIO, 92.6 GB/s VAST) on DGX-2-era hardware. Cite as a **2020 snapshot**, not a current expectation.
- **Oracle Cloud Infrastructure Blog — WEKA on OCI H100 compute** — `https://blogs.oracle.com/cloud-infrastructure/accelerate-ai-performance-weka-converged-storage` — **what it shows:** 192 GiB/s sequential-read GDS throughput on a single client node, roughly 3× a single Gen5 ×16 link's ceiling and therefore necessarily aggregated across several correctly-placed GPU/NIC pairs.
- **CoreWeave Docs — "About CoreWeave Storage"** — `https://docs.coreweave.com/docs/products/storage` — **what it shows:** a neocloud's production network-attached storage architecture and a concrete per-GPU bandwidth target (Tensorizer, up to 2 GB/s per GPU) to size a fabric against.
