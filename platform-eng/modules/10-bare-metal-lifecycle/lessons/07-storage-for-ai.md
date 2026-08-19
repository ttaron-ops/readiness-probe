---
lesson: "10.7"
title: "Storage for AI: parallel filesystems, tiering, and feeding the GPUs"
module: "10"
concept: "Storage for AI: parallel filesystems, tiering, and feeding the GPUs"
status: not-started
est_time: "7h"
prev: "06-hardware-health-remediation-rma.md"
next: "08-capex-vs-cloud-and-lb.md"
artifacts: []
sources: 12
---

# 10.7 · Storage for AI: parallel filesystems, tiering, and feeding the GPUs

> **Concept.** Size a storage hierarchy — local NVMe scratch, a parallel filesystem for datasets, object storage for weights and checkpoints — to the aggregate GB/s a GPU count demands, and wire it in through CSI so a slow data path never starves six-figure silicon.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Where this fits

Lesson 06 closed the hardware health loop: detect a sick node in minutes, cordon and drain
it without losing the job, decide reset versus RMA, and feed the failure rate back into
procurement and spares. That loop protects goodput from *broken* hardware. This lesson
protects goodput from **perfectly healthy** hardware that is starved anyway — a GPU with a
green DCGM health check, zero Xids and full thermal margin, sitting at 40% utilisation
because the dataloader is blocked on a read.

It also supplies a number lesson 06 borrowed on credit. The Young/Daly checkpoint interval
`Δ* = sqrt(2·δ·M)` used there takes `δ`, the checkpoint write time, as an input. Ninety
seconds was assumed. Ninety seconds for a 70-billion-parameter model is a *storage design
decision* with a price tag, and this lesson is where you compute it, size the tier that
delivers it, and see what it costs. With provisioning (05) and health (06) covered, this is
the last purely technical lesson before the capstone turns all of it into dollars.

## Why this matters

Storage failure in an AI cluster does not look like storage failure. There is no I/O
error, no exception, no alert. There is a GPU utilisation number that is lower than it
should be, and a training run that finishes later than the plan said. That is the most
expensive shape a problem can have, because nothing in the system is complaining.

The magnitude is not speculative. Meta reported measuring **56% of GPU cycles stalled
waiting for training data** before rebuilding its storage stack — more than half of every
paid-for GPU-hour, at hyperscaler scale, buying nothing. The fix was not "buy a faster
filesystem": it took a purpose-built Data PreProcessing Service, a distributed caching
layer using spare GPU-host memory that reached an average **80% cache hit rate**, and
metadata latency brought down to **1–2 ms**, on top of the exabyte-scale Tectonic
filesystem they already had.

Read that 56% as a multiplier rather than a percentage. If a GPU-hour costs you $2.50 and
56% of it is spent waiting, your effective cost per *useful* GPU-hour is
`2.50 / 0.44 = $5.68` — you have roughly doubled your unit cost without changing a single
line item in the budget. That is the same arithmetic lesson 08 will run on utilisation,
and it is why the storage line and the utilisation line in a capex model are not
independent.

At a neocloud, storage throughput per GPU is a number customers benchmark before they
sign, and getting it wrong is a churn event. It is also where on-prem experience is a
genuine differentiator: managed clouds hand you FSx for Lustre or Filestore as opaque
endpoints with a throughput SKU. On bare metal you choose the filesystem, size the metadata
tier, lay out the NVMe, own the GPUDirect path, and answer for the result.

## What's new here (calibration)

Lesson 02b.6 taught you the host-side data path: NVMe protocol and queue pairs, blk-mq,
Little's law for queue depth, PCIe placement (`PIX`/`PXB`/`PHB`/`SYS`), and GPUDirect
Storage with its two independently-failing layers. Module 04 covered GPU scheduling;
module 09 covered the fabric. None of that is re-taught.

What is new is everything above one host:

- **Turning a GPU count into a GB/s target from primary data**, not a rule of thumb —
  derived directly from MLCommons' published MLPerf Storage workload configurations and
  cross-checked against submitted results.
- **Checkpoint sizing that is arithmetic, not a guess**: the exact bytes-per-parameter
  accounting, why it is 14 and not 16, and how it reconciles with MLCommons' published
  checkpoint sizes for 8B / 70B / 405B / 1T models.
- **What a parallel filesystem actually does** — striping, the aggregate-bandwidth
  formula, and the separate metadata path that is the real scaling ceiling — with Lustre's
  concrete defaults read out of its source so you can reason about the tuning knobs rather
  than memorising vendor advice.
- **The metadata wall as a calculation**: how many metadata operations per second a
  training job generates, why that number and not bandwidth decides your filesystem, and
  the two mitigations that actually work.
- **The tier model with capacity, bandwidth, latency and $/TB attached**, so "which tier"
  becomes an economic decision rather than an aesthetic one.
- **Caching as a first-class tier**, because Meta's 80% hit rate is not a detail — it is a
  5× reduction in the bandwidth the backing store has to deliver.
- **CSI expression** of all of it, so requesting the right tier is one line in a PVC.

## Core concepts

### 1. The only question storage has to answer

A training step is a loop: fetch a batch, compute on it, exchange gradients, repeat. The
fetch and the compute can overlap — that is what prefetching is for — so storage only has
to be fast enough that the fetch for step `n+1` completes before the compute for step `n`
does. Everything in this lesson is in service of that one inequality.

MLCommons formalises it as **Accelerator Utilisation (AU)**:

```
  AU  =  total_compute_time / total_benchmark_running_time  × 100%
```

If every I/O operation is hidden behind compute, `total_compute_time` equals
`total_running_time` and AU is 100%. MLPerf Storage requires **AU ≥ 90%** for 3D U-Net and
ResNet-50, **≥ 85%** for RetinaNet and **≥ 70%** for CosmoFlow to record a valid result
(`mlcommons/storage`, `training/README.md` and the workload configs, commit `ce28a98`,
read 2026-08-18). Those thresholds are the industry's answer to "how much stall is
acceptable," and they are a much better target than "make it fast."

The demand side of the inequality is completely determined by three numbers you can look
up for your workload: **bytes per sample**, **samples per batch**, and **seconds of compute
per batch**.

```
  per-accelerator read demand (B/s) = batch_size × bytes_per_sample / compute_time_per_batch
```

That is it. Everything else — parallel filesystems, caching, GPUDirect, tiering — is
machinery for satisfying that number at a given cost. Which means the first thing you do,
before you shop, is compute it.

### 2. From GPU count to GB/s, derived from primary data

The old rule of thumb is "about 1 GB/s per GPU." It is wrong by more than an order of
magnitude in both directions depending on workload, and you can prove that from
MLCommons' own published workload configurations, which specify exactly the three numbers
the formula needs. The table below is computed from
`mlcommons/storage/configs/dlio/workload/*.yaml` at commit `ce28a98`:

| Workload | Accelerator | `batch_size` | `record_length_bytes` | `computation_time` (s) | **Derived per-accelerator demand** | Min AU |
|---|---|---|---|---|---|---|
| **3D U-Net** (medical imaging, `.npz`) | B200 | 7 | 146,600,628 (~146.6 MB avg) | 0.162 | **6.33 GB/s** | 90% |
| **3D U-Net** | H100 | 7 | 146,600,628 | 0.323 | **3.18 GB/s** | 90% |
| **CosmoFlow** (cosmology, TFRecord) | H100 | 1 | 2,828,486 (~2.83 MB) | 0.00350 | **0.81 GB/s** | 70% |
| **ResNet-50** (image classification, TFRecord) | H100 | 400 | 114,660 (~115 KB/image) | 0.224 | **0.20 GB/s** | 90% |
| **RetinaNet** (object detection, JPEG) | B200 | 24 | 322,957 (~323 KB) | 0.04755 | **0.16 GB/s** | 85% |
| **LLM pre-training** on pre-tokenised shards | any | — | 2 B/token | — | **~0.0001 GB/s** (see below) | — |

Worked, for the first row, so you can re-run it with your own numbers:

```
  7 samples/batch × 146,600,628 B/sample = 1,026,204,396 B per batch
  ÷ 0.162 s of compute per batch          = 6,334,595,037 B/s
                                          = 6.33 GB/s per accelerator
```

**The spread is 40×**, from 0.16 GB/s for RetinaNet to 6.33 GB/s for 3D U-Net on a B200 —
and it is 40× because bytes-per-sample varies by three orders of magnitude while compute
time varies by one. Add LLM pre-training and the spread becomes five orders of magnitude:
at 40,000 tokens/s/GPU and 2 bytes per token on disk, an LLM pre-training job reads
**80 KB/s per GPU** (lesson 02b.6 derives this). **"GB/s per GPU" is not a property of the
GPU. It is a property of the ratio between how big a sample is and how long the GPU takes
to consume it.**

The other structural fact in that table: **a faster accelerator makes the storage problem
harder in exact proportion.** The 3D U-Net rows differ only in `computation_time` — 0.323 s
on H100, 0.162 s on B200 — and the bandwidth demand doubles. A storage tier sized for an
H100 fleet is half-sized for the Blackwell refresh, and nothing about the dataset changed.

**Cross-check against submitted results.** Derived numbers are only as good as the model,
so verify against what vendors actually achieved in MLPerf Storage v2.0 (August 2025):
Hammerspace reported **420.8 GB/s supporting 140 simulated H100 accelerators at 96.4%
AU**, and Western Digital's OpenFlex Data24 reported **106.5 GB/s saturating 36 simulated
H100s**. That is `420.8/140 = 3.01` and `106.5/36 = 2.96` GB/s per H100 — against the
3.18 GB/s derived from the config. Agreement within 6%, from two independent submitters,
using a method you can re-run.

**Aggregate for a fleet:**

```
  aggregate_read_BW = N_gpus × per_gpu_demand / cache_hit_factor

  where cache_hit_factor = 1 / (1 − hit_rate), i.e. the reduction in load the
  backing store sees because some reads are served from a cache tier (§9).

  A 64-GPU H100 fleet, 3D-U-Net-shaped imaging workload, no cache:
      64 × 3.18 GB/s                              = 203.5 GB/s   ← from the PFS
  Same fleet, with an 80% cache hit rate (Meta's figure):
      64 × 3.18 × (1 − 0.80)                      =  40.7 GB/s   ← from the PFS
  Same fleet, LLM pre-training on tokenised shards:
      64 × 0.00008 GB/s                           =   0.005 GB/s ← irrelevant
```

Three orders of magnitude between the first and third lines, on the same hardware. **Size
for your workload mix, and state the mix, or the number is meaningless.**

### 3. Checkpoint burst: the term that usually dominates

Steady-state reads are continuous and overlappable. Checkpoints are the opposite: the
entire model and optimizer state has to be durable, all at once, every `Δ` seconds.

**The bytes-per-parameter accounting.** For mixed-precision training with Adam, the
standard decomposition (ZeRO, Rajbhandari et al., SC20) is: `2Ψ` bytes of bf16/fp16
weights, `2Ψ` of bf16/fp16 gradients, and `KΨ` of optimizer state with **K = 12** for
Adam — 4 bytes of fp32 master weights, 4 of first moment `m`, 4 of second moment `v`.
That is **16 bytes per parameter of live training state**. But a *resumable checkpoint*
does not persist gradients, which are recomputed on the next backward pass:

```
  live training state      = 2Ψ (weights) + 2Ψ (grads) + 12Ψ (optimizer) = 16 B/param
  resumable checkpoint     = 2Ψ (weights)              + 12Ψ (optimizer) = 14 B/param
  weights-only (inference) = 2Ψ                                          =  2 B/param
```

**This is a correction to an earlier version of this lesson, which used 16 B/param and
produced ~1.1 TB for a 70B model.** The correct figure is 14, and it is confirmed by
MLCommons' own published checkpoint sizes — their `llama3_8b_checkpoint` config states
"Total model+optimizer: 15 GB + 90 GB = 105 GB" for an 8.05B-parameter model, which is
exactly `2 B/param` of fp16 weights plus `12 B/param` of fp32 optimizer state.

Their full table (`mlcommons/storage/checkpointing/README.md`, commit `ce28a98`):

| Model | Hidden dim | Layers | Parallelism (TP×PP×DP) | Processes | ZeRO | **Checkpoint size** | In bytes |
|---|---|---|---|---|---|---|---|
| 8B | 4,096 | 32 | 1×1×8 | 8 | 3 | 105 GiB | 113 GB |
| 70B | 8,192 | 80 | 8×1×8 | 64 | 3 | **912 GiB** | **979 GB** |
| 405B | 16,384 | 126 | 8×32×2 | 512 | 1 | 5.29 TiB | 5.82 TB |
| 1T | 25,872 | 128 | 8×64×2 | 1,024 | 1 | 18 TiB | 19.8 TB |

Check the 70B row against the formula: `70e9 params × 14 B = 980 GB = 912.6 GiB`. Exact.
(Note MLCommons' table is in binary units; misreading "912 GB" costs you 7%.)

**Turning size into a bandwidth requirement:**

```
  checkpoint_write_BW = checkpoint_bytes / target_write_seconds

  70B model, want the write to cost 60 s of a 30-minute interval (3.3% overhead):
      979 GB / 60 s                          = 16.3 GB/s of WRITE
  Want it in 20 s (1.1% overhead):
      979 GB / 20 s                          = 49.0 GB/s
  405B model, 60 s:
      5,820 GB / 60 s                        = 97.0 GB/s
```

**And the restore side, which people forget.** MLPerf Storage v2.0 measures write *and*
read for exactly this reason: recovery time `ρ` in lesson 06's Young/Daly formula is
dominated by reading the checkpoint back. Worse, MLCommons requires that the read be
performed by *different* hosts than wrote it, and that any "remapping" time a storage
system needs before another host can read a freshly-written checkpoint be **measured and
added to the recovery time**. If your storage has single-writer semantics with a
promotion step, that step is on the critical path of every job restart.

**Which term is the sizing driver?** Compare, for a 64-GPU H100 fleet:

```
  steady read (3D U-Net shape, no cache) : 203.5 GB/s   continuous
  steady read (LLM pre-train)            :   0.005 GB/s continuous
  checkpoint write (70B, 60 s target)    :  16.3 GB/s   bursty, every 30 min
  checkpoint restore (70B, 60 s target)  :  16.3 GB/s   bursty, on failure

  Imaging shop  → steady read dominates by 12×. Size for read.
  LLM shop      → checkpoint burst dominates by 3,000×. Size for the burst.
```

That is the actual reason the "1 GB/s per GPU" rule of thumb is useless: it produces a
number that is 12× too small for one shop and 3,000× too big for the other, and neither of
them will notice until the cluster is built.

**Async and sharded checkpointing is how you decouple the burst from the GPU.** Each rank
writes its shard to **local NVMe first** — a Gen5 ×4 drive sustains on the order of 6 GB/s
of write (lesson 02b.6), and a per-node shard of a 70B checkpoint at DP=8 is
`979/8 = 122 GB`, so the on-GPU-critical-path portion is `122/6 ≈ 20 s` — and a background
process drains it to the shared tier afterwards. The GPUs resume after the local write; the
shared tier sees a smoothed drain rather than a spike. The cost is local NVMe capacity
(§10) and the risk that a node dies between the local write and the drain, which is why the
drain target has to be durable and the retention policy has to keep the previous good
checkpoint until the new one has landed.

### 4. The storage tiers, with numbers attached

```
   THE HIERARCHY — capacity, bandwidth, latency and cost per TB
   (August 2026 snapshot; every $ figure is volatile — see the note below)

   ┌──────────────────────────────────────────────────────────────────────────────┐
   │ ① GPU HBM                                                       80–192 GB/GPU│
   │    ~3.4 TB/s per H100 · latency ~hundreds of ns                              │
   │    holds: the active batch, weights, activations                             │
   │    $/TB: not purchasable separately (it is most of the GPU's price)          │
   └────────────────────────────▲─────────────────────────────────────────────────┘
        PCIe Gen5 ×16 ≈ 64 GB/s │  or GDS peer-to-peer DMA (§11)
   ┌────────────────────────────┴─────────────────────────────────────────────────┐
   │ ② HOST DRAM / PAGE CACHE                             1–2 TB per node         │
   │    ~307 GB/s per socket (DDR5 ×8, lesson 02b.6) · latency ~80 ns             │
   │    holds: prefetch buffers, page cache, the distributed cache tier (§9)      │
   │    $/TB: ~$3–6K (server DDR5 RDIMM, and it competes with model memory)       │
   └────────────────────────────▲─────────────────────────────────────────────────┘
        PCIe Gen5 ×4 per drive  │
   ┌────────────────────────────┴─────────────────────────────────────────────────┐
   │ ③ LOCAL NVMe SCRATCH                          8–120 TB per node              │
   │    14 GB/s read / ~6 GB/s write per Gen5 ×4 drive (lesson 02b.6)             │
   │    latency ~80 µs · 2.7M random read IOPS on a CM7-class part                │
   │    holds: checkpoint shards pre-drain, dataset cache, spill                  │
   │    $/TB: ~$95–110 capex, raw (enterprise U.2, 2026 — NAND pricing is         │
   │          unusually volatile right now; re-quote before you budget)           │
   └────────────────────────────▲─────────────────────────────────────────────────┘
        storage fabric — 200/400G  │ RDMA or TCP; shares the physical network with
        per client                 │ collectives unless you built a separate rail
   ┌────────────────────────────┴─────────────────────────────────────────────────┐
   │ ④ PARALLEL FILESYSTEM (Lustre / GPFS / WEKA / VAST / BeeGFS)                 │
   │    1–100+ PB · 100 GB/s–1 TB/s aggregate (scales with server count)          │
   │    latency ~200 µs–2 ms · POSIX namespace, RWX from every node               │
   │    holds: training datasets, the checkpoint hot tier                         │
   │    $/TB-month: ~$25–70 as a managed SKU; self-built ≈ NVMe capex ÷ usable    │
   │          fraction, + servers + fabric + support                              │
   └────────────────────────────▲─────────────────────────────────────────────────┘
        HTTP(S), often over the same fabric │
   ┌────────────────────────────┴─────────────────────────────────────────────────┐
   │ ⑤ OBJECT STORAGE (S3 / MinIO / Ceph RGW)                                     │
   │    effectively unbounded · 10s of GB/s aggregate, ~10–100 ms first-byte      │
   │    no POSIX; GET/PUT/multipart; strong durability, versioning, lifecycle     │
   │    holds: model weights, cold checkpoints, raw dataset of record             │
   │    $/TB-month: S3 Standard ~$23 (first 50 TB) → ~$21 (>500 TB);              │
   │          Standard-IA ~$12.50 + retrieval; Glacier Deep Archive ~$0.99        │
   │    ⚠ egress is priced separately and can exceed storage — see lesson 11.7    │
   └──────────────────────────────────────────────────────────────────────────────┘

   THE RATIO THAT ORGANISES ALL OF IT:
     bandwidth per dollar falls roughly 10× per level going down;
     capacity per dollar rises roughly 10–100× per level going down;
     latency rises 1,000× from ② to ⑤.
   So the design question is never "which tier" — it is "what fraction of accesses
   can I serve from level k, so that level k+1 only has to supply the misses?"
   That fraction IS the cache_hit_factor in §2's formula, and it is the single
   cheapest lever in the entire stack.
```

Every dollar figure above is an **August 2026 snapshot** and should be re-quoted before it
goes in a budget. NAND flash pricing in particular is in an unusual state — multiple trade
reports through 2026 describe sharp increases driven by AI demand — so the `$95–110/TB`
NVMe figure is the one most likely to be stale by the time you read this.

**The four data classes, and why each lands where it does:**

| Data class | Access pattern | Belongs on | Why, mechanically |
|---|---|---|---|
| **Scratch / cache** | ephemeral, per-node, mixed random and streaming, highest IOPS | **local NVMe** (③) | Lowest latency and no network hop; it is throwaway so it needs no shared durability, and putting it on ④ spends bandwidth every other node also wants. |
| **Training datasets** | read-mostly, read by every node, high aggregate sequential | **parallel FS** (④), fronted by a cache | One namespace, aggregate bandwidth that scales with server count, RWX so every pod mounts the same path. Object storage (⑤) is too slow per-object for random shard access unless the reader streams large contiguous ranges. |
| **Checkpoints** | bursty large sequential writes, then cold; read only on failure | **local NVMe (③) → parallel FS hot tier (④) → object (⑤)** | The burst needs peak write bandwidth that only ③ can absorb without disturbing readers; the drain to ④ makes it durable and readable by *other* hosts; ageing to ⑤ makes retention cheap. |
| **Model weights / artifacts** | write-once, read-many, versioned, must be durable | **object** (⑤) | Cheap, durable, versioned, lifecycle-managed. Pulled to ③ at job start, where the read is a one-off. |

The failure mode for each misplacement is specific: **datasets on object storage** starves
GPUs on per-object overhead and first-byte latency; **checkpoints written only to object
storage** puts a 10–100 ms-latency store on the GPU critical path every `Δ` seconds;
**scratch on the parallel filesystem** burns shared bandwidth on bytes nobody else will
ever read; **weights on the parallel filesystem** wastes your most expensive $/TB on
write-once data and gives up versioning.

### 5. What a parallel filesystem actually is

A parallel filesystem presents one POSIX namespace but splits two things across many
servers: **file data** and **file metadata**. Understanding that they are separate paths is
the whole lesson, because they scale differently and fail differently.

Take Lustre, which names the parts most explicitly:

```
                        ┌─────────────────────────────────────────────┐
   CLIENTS (GPU nodes)  │ every node mounts /lustre — one namespace    │
   ┌────┐ ┌────┐ ┌────┐ └───────┬─────────────────────────┬───────────┘
   │ n0 │ │ n1 │ │ nN │         │ METADATA path            │ DATA path
   └─┬──┘ └─┬──┘ └─┬──┘         │ open/stat/readdir/create │ read/write
     │      │      │            │ (small, latency-bound,   │ (large, bandwidth-
     └──────┴──────┴────────────┤  synchronous)            │  bound, streaming)
                                ▼                          ▼
                     ┌──────────────────┐      ┌──────────────────────────────┐
                     │  MDS  (server)   │      │  OSS 0 │ OSS 1 │ … │ OSS M   │
                     │   ├─ MDT 0       │      │   OST0 │  OST2 │   │  OST2M  │
                     │   └─ MDT 1 (DNE) │      │   OST1 │  OST3 │   │  OST2M+1│
                     └──────────────────┘      └──────────────────────────────┘
                       holds: names, inodes,     holds: the file's data OBJECTS,
                       permissions, LAYOUT       striped across OSTs per the
                       (which OSTs hold which    layout the MDS handed back
                       stripes of this file)

   ONE READ, IN ORDER:
     1. client → MDS   : "open /lustre/ds/shard-0004.tar"    (one RPC, ~0.3–2 ms)
        MDS  → client  : inode + LAYOUT (stripe_count, stripe_size, OST list)
     2. client → OSTs  : parallel bulk RPCs to each OST holding a stripe
        OSTs → client  : data, at aggregate = min(client NIC,
                                                  stripe_count × per-OST BW,
                                                  Little's-law limit — see §6)
     3. no further MDS involvement for the rest of the file

   THE CONSEQUENCE: bandwidth scales with the number of OSTs you stripe across;
   metadata throughput scales with the number of MDTs — and those are two different
   purchases. A cluster can be at 500 GB/s on the data path while every job crawls,
   because one MDS is pegged at 100% CPU serving stat() calls.
```

The aggregate-bandwidth formula that falls out:

```
  per-file BW  = min( client_NIC_BW,
                      stripe_count × per_OST_BW,
                      in_flight_bytes / RPC_latency )      ← Little's law, §6

  cluster BW   = min( Σ client_NIC_BW,
                      Σ OST_BW,
                      Σ storage-fabric bisection BW )
```

Note the third term in the cluster line. **Storage traffic rides a physical network**, and
unless you built a separate storage rail it is the same fabric carrying your NCCL
collectives (module 09). A 200 GB/s checkpoint burst landing in the middle of an all-reduce
does not politely queue — it competes, and the symptom is a training step that intermittently
takes twice as long with nothing wrong on either subsystem individually.

### 6. Lustre's real knobs and their real defaults

These are verified by reading the Lustre source, not vendor advice (`lustre/lustre-release`,
commit `4322543`, read 2026-08-18):

| Knob | Where | Default | Max | What it does |
|---|---|---|---|---|
| `stripe_count` (`lfs setstripe -c`) | per file or directory | **1** | `-1` = all OSTs | How many OSTs a file's data is spread across. **A default-striped file gets exactly one OST's bandwidth.** |
| `stripe_size` (`lfs setstripe -S`) | per file or directory | **4 MiB** | — | Bytes written to one OST before moving to the next. |
| `max_rpcs_in_flight` | `osc.*.max_rpcs_in_flight` (`OBD_MAX_RIF_DEFAULT`) | **8** | 256 (OSC), 512 (OBD) | Concurrent bulk RPCs per client-OST connection. This is the `N` in Little's law. |
| `max_dirty_mb` | `osc.*.max_dirty_mb` (`OSC_MAX_DIRTY_DEFAULT`) | **2000 MB** | 2048 MB | Dirty write-back cache per client-OST connection. |
| RPC size | negotiated `ocd_brw_size`; `DT_DEF_BRW_SIZE` | **4 MiB** | **64 MiB** (`PTLRPC_MAX_BRW_SIZE` = `1 << (LNET_MTU_BITS 20 + PTLRPC_BULK_OPS_BITS 6)`) | Bytes per bulk RPC. `max_pages_per_rpc` is this divided by page size — 1,024 pages at 4 MiB / 4 KiB. |

**`stripe_count` defaulting to 1 is the single most consequential default in Lustre**, and
it is the one that produces "we bought a 500 GB/s filesystem and get 3 GB/s." A file
written with the default layout lives entirely on one OST, so a single reader gets one
OST's bandwidth no matter how many you own. It is the right default for a filesystem full
of small files (striping a 100 KB file across 16 OSTs just multiplies the metadata and RPC
overhead by 16 for no gain), and exactly wrong for a directory of multi-gigabyte dataset
shards.

```
  Set the layout on the DIRECTORY, before you write into it — layout is inherited
  at creation and cannot be changed on an existing file without a migrate.

  # dataset shards: large sequential, want wide striping
  $ lfs setstripe -c 16 -S 4M /lustre/datasets/imagenet-webdataset
  $ lfs getstripe -d /lustre/datasets/imagenet-webdataset
  stripe_count:  16 stripe_size:   4194304 pattern:  raid0 stripe_offset: -1

  # checkpoints: each rank writes its own file, wide stripes, big RPCs
  $ lfs setstripe -c 32 -S 16M /lustre/checkpoints

  # a tree of millions of small files: leave stripe_count at 1, and stripe the
  # DIRECTORY across MDTs instead (DNE) so metadata ops spread
  $ lfs setdirstripe -c 4 /lustre/datasets/many-small-files

  # verify what a file actually got
  $ lfs getstripe /lustre/checkpoints/step-12000/rank-0007.pt
       obdidx    objid   objid    group
           11    98317   0x18009  0
           27    98318   0x1800a  0
           …
```

**Little's law on the client side** decides whether you reach the stripe-count ceiling. From
lesson 02b.6, `outstanding_bytes = throughput × latency`:

```
  in_flight_bytes = max_rpcs_in_flight × RPC_size = 8 × 4 MiB = 33.5 MB   (defaults)

  achievable per-OSC throughput = in_flight_bytes / RPC_round_trip_latency
      at 1 ms  RTT :  33.5 MB / 0.001 s  = 33.5 GB/s   ← not the limit
      at 5 ms  RTT :  33.5 MB / 0.005 s  =  6.7 GB/s
      at 20 ms RTT :  33.5 MB / 0.020 s  =  1.7 GB/s   ← now it is the limit

  So: on a low-latency RDMA fabric the defaults are fine, and on a congested or
  high-latency path they are the ceiling. Raising max_rpcs_in_flight to 32 with
  16 MiB RPCs gives 537 MB in flight — 26.8 GB/s even at a 20 ms RTT.

  $ lctl set_param osc.*.max_rpcs_in_flight=32
  $ lctl set_param osc.*.max_pages_per_rpc=4096      # 16 MiB at 4 KiB pages
  $ lctl get_param osc.*.max_rpcs_in_flight
```

**The diagnostic order that follows:** if you are not getting the bandwidth you paid for,
check `lfs getstripe` first (are you on one OST?), then `max_rpcs_in_flight` and RPC size
(are you deep enough for the latency?), then the fabric. Reversing that order wastes days,
because "the filesystem is slow" and "you are using 1/16th of the filesystem" look
identical from a client.

Other filesystems expose the same two ideas under different names.
**IBM Storage Scale (GPFS)** distributes metadata rather than concentrating it, and its
client-side knobs are `pagepool` (data cache per node; default is the *smaller* of one
third of physical memory or 4 GiB), `maxFilesToCache` (default 4,000) and `maxStatCache`
(default the larger of 1,000 or 4× `maxFilesToCache`), all set with `mmchconfig`, with
IBM's guidance to keep their combined footprint at or below 50% of node memory. On a GPU
node with 2 TB of RAM, a 4 GiB `pagepool` default is a rounding error against what the
node could cache — this is the GPFS equivalent of Lustre's `stripe_count: 1`.
*(IBM's documentation domain is blocked by this session's egress proxy; these defaults come
from search extracts of the IBM Storage Scale tuning pages and are labelled accordingly —
verify against `mmlsconfig` on your own cluster before relying on them.)*

### 7. The metadata wall, as arithmetic

Bandwidth is the number people size for. Metadata is the number that decides whether the
filesystem works.

Every sample read from an individual file costs at least an `open()`, and depending on the
loader also a `stat()`; every epoch shuffle costs a `readdir()` over the dataset; every
checkpoint costs a `create()` per rank per shard. These are small, synchronous,
latency-bound operations, and on a classic single-MDT Lustre they all funnel through one
server.

```
  METADATA DEMAND, for a fleet reading individual sample files

      metadata_ops/s ≈ N_gpus × samples_per_sec_per_gpu × ops_per_sample

  ResNet-50-shaped, individual JPEGs, 64 H100s:
      samples/s/GPU = batch_size / computation_time = 400 / 0.224 = 1,786
      ops_per_sample = 2 (open + stat; more if the loader also fstats or seeks)

      64 × 1,786 × 2                                   = 228,600 metadata ops/s

  A well-tuned single Lustre MDT serves on the order of 10⁴–10⁵ ops/s depending on
  the operation mix, hardware and whether the working set fits in the MDS's cache.
  228,600 ops/s is, at best, at the very top of that range and more likely 2–20×
  beyond it.

  → The job is metadata-bound. The OSTs are nearly idle. Every dashboard shows a
    healthy filesystem. The GPUs sit at 40%.
```

Now the same fleet reading **packed shards** — WebDataset `.tar`, TFRecord, or any format
that concatenates thousands of samples into one large file:

```
  shard size 1 GiB, 9,700 samples per shard (at ~115 KB/sample)
      shards consumed/s = 64 × 1,786 / 9,700                = 11.8 shards/s
      ops_per_shard ≈ 2 (one open, one close; the reads are sequential within)

      11.8 × 2                                              = 24 metadata ops/s

  → a 9,500× reduction in metadata load, for the same bytes and the same samples.
```

**That is the entire small-file problem and its entire solution.** Packing samples into
large shards converts a metadata-bound workload into a bandwidth-bound one, and
bandwidth-bound is the problem parallel filesystems are actually good at. It costs you
random access within a shard (you shuffle at the shard level plus a buffer, not
globally) — which is why WebDataset and TFRecord both ship a shuffle-buffer abstraction.

The three mitigations, in order of how much they buy:

1. **Pack into shards** (WebDataset/tar, TFRecord, Parquet, or a mmap-able binary format).
   Orders of magnitude. Do this first, always.
2. **Distribute metadata.** Lustre DNE (`lfs setdirstripe -c N`) spreads a directory's
   entries across multiple MDTs; published DNE measurements show file-create throughput
   improving substantially per added MDS, with diminishing returns beyond about two MDTs
   per MDS node. WEKA, VAST and GPFS distribute metadata by architecture rather than as an
   add-on, which is the real reason they show up in GPU-cluster shortlists.
3. **Data-on-MDT** (`lfs setstripe -E 1M -L mdt`), which stores small files directly on the
   MDT so the read costs no OST round trip at all. Helps when you genuinely cannot repack.

**The question to ask a filesystem vendor is therefore not "how many GB/s."** It is *"how
many metadata operations per second, at what file-count, with what mix of open/stat/create,
and what happens to that number as I add capacity?"* A vendor whose metadata throughput is
flat in capacity has a wall in it, and you will find it at exactly the scale where you
stopped being able to move.

### 8. The filesystem landscape

| Filesystem | Data layout | Metadata | Access | Where it fits |
|---|---|---|---|---|
| **Lustre** | Objects striped across OSTs on OSS servers | Classically one active MDT; DNE adds more, and striped directories spread entries | POSIX via a kernel client (LNet over RDMA or TCP) | Highest bandwidth per dollar at large scale; open source; the small-file wall is real and DNE is a mitigation, not an architecture change. The default `stripe_count: 1` is a trap. |
| **IBM Storage Scale (GPFS)** | Shared-disk, blocks striped across all NSDs | Distributed across nodes with a token manager | POSIX, plus NFS/SMB/S3 gateways | Mature, strong metadata behaviour, extensive tiering and policy engine; licence cost is the objection. |
| **WEKA** | Software-defined over NVMe, own protocol, tiers to object | Distributed across all nodes | POSIX (kernel client), NFS, S3 | Designed for mixed and small-file workloads; common at GPU neoclouds. *Vendor performance claims should be treated as best case.* |
| **VAST Data** | Disaggregated shared-everything over NVMe/QLC | Distributed, shared-everything | NFS (incl. nconnect/RDMA), S3 | Simple operations, good metadata scaling, popular in newer AI datacentres. |
| **BeeGFS** | Objects across storage targets | Metadata servers can be added | POSIX | Cheap, open, easy to stand up; common in smaller HPC and as a scratch tier. |
| **Ceph (CephFS / RGW)** | RADOS objects with CRUSH placement | MDS cluster for CephFS | POSIX (CephFS), S3 (RGW), block (RBD) | One system for file, object and block; usually chosen for operational consolidation rather than peak AI throughput. |

**Read that table as a metadata-architecture table, because that is the axis that
differentiates them.** Everyone can put bytes on NVMe and stripe them. The distinguishing
question is whether metadata throughput grows when you add hardware, and Lustre is the only
row where the honest answer is "with effort."

**On vendor numbers.** Storage vendors publish impressive aggregate figures, and they are
usually real — on the benchmark, on the configuration, with the client count they used.
MLPerf Storage exists precisely so those claims land on a common basis with a stated
accelerator-utilisation floor. **Prefer an MLPerf submission over a datasheet, and prefer
your own workload over an MLPerf submission** — and when you cite a vendor's own marketing
figure, label it as such rather than laundering it into a planning number.

### 9. Caching is a tier, and it is the cheapest one

Return to §2's formula. The backing store only has to supply the *misses*:

```
  BW_from_backing_store = N_gpus × per_gpu_demand × (1 − cache_hit_rate)

  64 H100s, 3D-U-Net shape, 3.18 GB/s each:
      hit rate 0%   →  203.5 GB/s   from the parallel filesystem
      hit rate 50%  →  101.8 GB/s
      hit rate 80%  →   40.7 GB/s   ← Meta's reported average hit rate
      hit rate 95%  →   10.2 GB/s
```

**An 80% hit rate is a 5× reduction in the filesystem you have to buy.** That is a larger
lever than any tuning knob in this lesson, and it is why Meta's rebuild centred on a
caching layer rather than a faster backing store. Three cache layers exist, and they are
not alternatives — they compose:

1. **The Linux page cache**, free and automatic, sized by whatever host DRAM is not
   otherwise used. On a GPU node with 1–2 TB of RAM this is not small. It works only for
   buffered reads — `O_DIRECT` and GDS bypass it by design — which is a real tension: the
   fast path for large sequential reads is also the path that defeats caching for repeated
   small reads. Choose per data class, not per cluster.
2. **Node-local NVMe as a cache tier**, populated at job start or on first read. This is
   what "pull the dataset to local scratch" means when it is done properly: a read-through
   cache with an eviction policy, not a manual `cp`.
3. **A distributed cache across nodes**, which is what Meta built — using spare GPU-host
   memory across the fleet as one pool, so a shard fetched by node 7 can be served to node
   23 over the fabric instead of re-fetched from the backing store. This is the layer that
   gets you from ~50% to ~80%, because it turns `N` independent caches into one cache `N`
   times the size.

The precondition for any of it is **reuse**. A hit rate is a property of the access
pattern: multi-epoch training over a dataset that fits in the cache tier gets high hit
rates almost for free; a single pass over a dataset far larger than the cache gets none,
and no amount of cache hardware changes that. Compute
`dataset_size / total_cache_capacity` before you budget for a cache tier — if it is much
greater than one and you make a single pass, you are buying nothing.

### 10. Local NVMe: capacity sizing, and the one number that decides it

Local NVMe (tier ③) does three jobs: absorb the checkpoint write burst, cache the dataset
working set, and hold spill. Size it as the sum:

```
  per-node NVMe capacity ≥
        K × (checkpoint_bytes / DP_degree)      ← K ≥ 2: keep the previous good
                                                   checkpoint until the new one has
                                                   drained and been verified
      + dataset_working_set_per_node            ← what the cache tier holds locally
      + spill_and_headroom                      ← logs, container images, ~15%

  70B model, DP = 8 → per-node checkpoint shard = 979 GB / 8      = 122 GB
      K = 2                                                      = 245 GB
    + dataset working set (say 2 TB of a packed shard set)        = 2,000 GB
    + 15% headroom                                               =   337 GB
    ──────────────────────────────────────────────────────────────────────
      per-node NVMe usable                                        ≈ 2.6 TB

  Round up for RAID/overprovisioning and the fact that NVMe write performance
  degrades near full: provision ~4 TB usable per node.
```

And the bandwidth check, from lesson 02b.6's numbers: a Gen5 ×4 drive is rated ~14 GB/s
read and sustains on the order of 6 GB/s write. One drive absorbs the 122 GB shard in
`122/6 ≈ 20 s`; an 8-drive array with the host-side limit around 40 GB/s does it in
about 3 s. **That is the number that sets `δ` in lesson 06's checkpoint-interval formula**,
and it is a purchasing decision: whether a checkpoint costs 20 seconds or 3 seconds of GPU
time, every 30 minutes, forever.

Placement matters as much as capacity, and lesson 02b.6 has the mechanism: an NVMe under
the same PCIe switch as the GPU (`PIX`/`PXB`) delivers full drive bandwidth and is the only
topology where GPUDirect Storage reaches its peak; across the root complex (`PHB`) it is
platform-dependent; across sockets (`SYS`) every byte crosses the ~48 GB/s-per-direction
inter-socket link shared with all cache-coherence traffic, and GDS falls back to the CPU
bounce path.

### 11. GPUDirect Storage, and the trap

Without GDS, a read into GPU memory is two copies: storage DMAs into a host bounce buffer,
then `cudaMemcpy` moves it to HBM. GDS lets the NVMe or NIC DMA **directly into the GPU's
BAR-mapped HBM**, eliminating one full copy, the CPU cycles that drove it, and the host
DRAM bandwidth it consumed — at H100-class rates a 14 GB/s bounce path costs 28 GB/s of
host DRAM bandwidth (one write plus one read) out of roughly 307 GB/s per socket, per
stream.

The measured difference, using `gdsio` from the GDS package (lesson 02b.6's captured
figures on a correctly-placed drive):

```
  $ gdsio -f /mnt/nvme/testfile -d 0 -w 8 -s 16G -i 1M -x 0   # GDS fast path
  Throughput: 12.914 GiB/sec, Avg_Latency: 619.42 usecs

  $ gdsio -f /mnt/nvme/testfile -d 0 -w 8 -s 16G -i 1M -x 1   # CPU bounce path
  Throughput:  6.201 GiB/sec, Avg_Latency: 1289.90 usecs
```

**The trap: "GDS is enabled" is not the claim "GDS is on the fast path."** `cuFile` has a
compatibility mode (`allow_compat_mode` in `/etc/cufile.json`) that, when the fast path is
unavailable — mount not GDS-capable, `nvidia-fs.ko` not loaded, unsupported filesystem,
IOMMU in translation mode, wrong PCIe topology — silently stages through the CPU bounce
buffer **and returns success**. Your code is correct, your calls succeed, and you have none
of the benefit. Verify the whole chain:

```
  $ /usr/local/cuda/gds/tools/gdscheck -p
   GDS release version: 1.16.0.x
   … supported GDS filesystems / drivers …
   IOMMU: disabled
   GPU index 0 NVIDIA H100 80GB HBM3 bar:1 bar size (MiB):131072 supports GDS
   properties.max_direct_io_size_kb : 16384
```

Three lines to read: does the *mount type* appear as supported; does every GPU say
`supports GDS`; and what is the IOMMU state — many platforms only deliver the peer-to-peer
fast path with the IOMMU off or in passthrough (`iommu=pt`, which is exactly what lesson
05's iPXE kernel line sets). Note `max_direct_io_size_kb: 16384`: cuFile splits requests
larger than 16 MiB, so a 64 MiB `cuFileRead` is four operations, and your effective queue
depth is four times what you thought.

GDS matters most where the CPU bounce is genuinely your ceiling — large sequential reads
into HBM and checkpoint restore. It matters least for small random reads, where latency
and metadata dominate and the copy is not the bottleneck.

### 12. Expressing the tiers through CSI

All of the above becomes StorageClasses, so a pod requests a tier by name:

```yaml
# ── Tier ④: shared parallel filesystem, RWX, for datasets and checkpoint-hot ──
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pfs-hot
provisioner: csi.weka.io                 # or the Lustre / GPFS / VAST CSI driver
parameters:
  filesystemGroupName: gpu-hot
  capacityEnforcement: "HARD"
reclaimPolicy: Retain                    # datasets outlive the PVC. Delete is a
                                         # data-loss footgun on a shared FS.
allowVolumeExpansion: true
volumeBindingMode: Immediate             # a shared FS is reachable from everywhere,
                                         # so there is nothing to wait for
mountOptions:
  - "readahead_kb=8192"                  # large sequential reads; tune with §6's
                                         # Little's-law numbers, not by feel
---
# ── Tier ③: node-local NVMe scratch ──────────────────────────────────────────
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nvme-scratch
provisioner: kubernetes.io/no-provisioner  # backed by local-static-provisioner,
                                           # which discovers mounts under a
                                           # configured discovery directory
volumeBindingMode: WaitForFirstConsumer    # ← MANDATORY. See below.
reclaimPolicy: Delete
```

**`volumeBindingMode: WaitForFirstConsumer` on a local volume is not optional.** With
`Immediate`, the PV binds as soon as the PVC is created — before the scheduler has picked a
node — so the scheduler is then *constrained* to the node that happens to own that PV,
regardless of GPU availability, topology or taints. On a GPU cluster that means a pod
requesting 8 GPUs and local scratch can be pinned to a node with no free GPUs and sit
`Pending` forever. `WaitForFirstConsumer` inverts the order: schedule the pod first,
considering all its constraints, then bind a local PV on the node that was chosen.

```yaml
# ── The training pod's claims ────────────────────────────────────────────────
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: datasets }
spec:
  accessModes: ["ReadWriteMany"]          # every rank mounts the same namespace
  storageClassName: pfs-hot
  resources: { requests: { storage: 200Ti } }
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: scratch }
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: nvme-scratch
  resources: { requests: { storage: 4Ti } }   # from §10's capacity sizing
```

**Tier ⑤ gets no PVC.** Object storage is not a filesystem and mounting it as one
(`s3fs`, `goofys`) reintroduces POSIX semantics the protocol cannot cheaply provide —
every `stat()` becomes a HEAD request, every rename becomes a copy-and-delete. The
training container talks S3 through an SDK, or through the Container Object Storage
Interface (COSI) if you want the bucket lifecycle managed by Kubernetes. Weights come from
`s3://weights/...`; cold checkpoints age out to `s3://ckpt-cold/...` under a lifecycle
policy.

And the grace period that ties this back to lesson 06:

```yaml
spec:
  terminationGracePeriodSeconds: 300      # ≥ the checkpoint write time from §3.
                                          # The default of 30 s SIGKILLs the rank
                                          # mid-write and the drain in lesson 06
                                          # destroys the job it was trying to save.
```

## Perspectives

**Developer / ML-engineer view.** From inside a training script, storage is invisible until
it is not: a `DataLoader` with too few workers, a dataset on the wrong tier, or a
synchronous checkpoint in the training loop all present as low GPU utilisation with no
error and no stack trace. The two knobs that live in the code are prefetch depth (NVIDIA
DALI's `prefetch_queue_depth` defaults to **2**, verified in
`dali/python/nvidia/dali/pipeline.py`) and worker count — and lesson 02b.6's Little's-law
arithmetic tells you whether either is deep enough. Everything else is a platform decision
wearing a code costume.

**Operator / platform-engineer view.** You do not get to blame the framework. You choose
the filesystem, size the metadata tier, set the stripe layout on the directories before
anyone writes into them, provision NVMe per node, and wire the StorageClasses so
requesting the right tier is one line rather than tribal knowledge. The highest-leverage
thing you do is the sizing calculation in §2 and §3 *before* procurement — retrofitting a
parallel filesystem under a live 64-GPU cluster is enormously more expensive than sizing it
correctly once.

**Hardware / kernel view.** The bytes travel a real physical path: NVMe or NIC → PCIe →
(fabric) → PCIe → GPU HBM, with GDS shortening it by one copy. Whether it is fast depends
on things invisible from the training script — PCIe topology relative to the GPU, IOMMU
mode, whether the storage client negotiated RDMA, and whether cuFile silently fell back to
compat mode. And storage traffic shares the physical fabric with collectives, so the
storage and network designs are one design.

**Economics view.** Every GB/s you under-provision converts directly into idle GPU-hours,
and idle GPU-hours are the denominator of lesson 08's owned-$/GPU-hr equation. Read Meta's
56% through that lens and it is not a storage statistic — it is roughly a 2× multiplier on
the effective cost of every GPU-hour during the stalled period. Conversely, the cache tier
is the cheapest bandwidth in the stack: an 80% hit rate is a 5× reduction in the parallel
filesystem you have to buy, which at $25–70/TB-month on a multi-petabyte tier is a
six-figure annual line item.

## Real-world use cases

- **Meta — "Meta's AI Storage Blueprint at Scale" (2026).** Meta measured **56% of GPU
  cycles stalled waiting for training data** and rebuilt the stack: a Data PreProcessing
  Service, a distributed caching layer built from unused GPU-host memory reaching an
  **average 80% cache hit rate**, and **metadata accessible within 1–2 ms**, all layered on
  the existing exabyte-scale **Tectonic** filesystem. **What it shows:** at hyperscaler
  scale, "storage starves GPUs" is the default state until an engineering programme fixes
  it — and the fix was a caching and preprocessing architecture, not faster disks. The
  80% hit rate is the single most reusable number in this lesson because it is the term
  that divides the bandwidth you must buy.
  <https://engineering.fb.com/2026/07/01/data-infrastructure/metas-ai-storage-blueprint-at-scale/>
  *(`engineering.fb.com` is blocked by this session's egress proxy and the post was not
  fetched. The figures above come from multiple independent secondary reports that agree —
  [SDxCentral](https://www.sdxcentral.com/news/meta-rebuilds-its-ai-storage-stack-from-the-ground-up-to-stop-gpus-sitting-idle/),
  [TechRadar Pro](https://www.techradar.com/pro/a-disk-in-a-planet-scale-computer-meta-has-so-many-expensive-gpus-that-its-buying-ssds-to-kill-idle-time)
  — and are labelled accordingly.)*
- **MLCommons — MLPerf Storage v2.0, and the checkpointing workload added August 2025.**
  The benchmark defines accelerator utilisation with per-workload floors (90% for 3D U-Net
  and ResNet-50, 85% RetinaNet, 70% CosmoFlow), publishes the exact workload configurations
  this lesson derives its per-accelerator bandwidth table from, and added a checkpointing
  workload with published checkpoint sizes for 8B/70B/405B/1T models plus a requirement
  that checkpoint *reads* be performed by different hosts than wrote them, with any
  remapping time measured and added to recovery. **What it shows:** checkpoint sizing is a
  standardised, benchmarked industry problem, not an invented scenario — and the
  configurations give you primary data to size against instead of a rule of thumb.
  Verified by reading the repository (`mlcommons/storage`, commit `ce28a98`, read
  2026-08-18).
- **MLPerf Storage v2.0 submitted results as a reality check.** Hammerspace reported
  **420.8 GB/s supporting 140 simulated H100 accelerators at 96.4% AU**, and Western
  Digital's OpenFlex Data24 reported **106.5 GB/s saturating 36 simulated H100s** on the
  3D U-Net workload; YanRong reported a peak of **513 GB/s**. **What it shows:** two
  independent submitters land at 3.01 and 2.96 GB/s per H100, within 6% of the 3.18 GB/s
  derived from the workload config in §2 — which is how you know the derivation method is
  sound rather than plausible. *(Results pages were not fetched directly; figures from
  [MLCommons' results announcement coverage](https://blocksandfiles.com/2025/08/05/storage-arrays-get-faster-in-2nd-version-of-mlperf-storage-benchmark/)
  and [YanRong's 3D U-Net result coverage](https://blocksandfiles.com/2025/08/08/yanrong-excels-at-3d-u-net-in-mlperf-storage-benchmark/).)*
- **Lustre's `stripe_count: 1` default as an industry-wide footgun.** The filesystem that
  underpins a large fraction of the world's HPC bandwidth ships with a per-file default
  that confines each file to a single storage target. **What it shows:** the most expensive
  storage bugs are configuration defaults that are correct for the original workload
  (millions of small HPC files) and exactly wrong for the new one (multi-gigabyte dataset
  shards). Verified from source (`lustre/lustre-release`, `Documentation/man1/lfs-setstripe.1`,
  commit `4322543`) rather than from a tuning blog, because tuning blogs disagree and the
  man page does not.

## Worked example: sizing storage for a 64-GPU H100 cluster

Eight nodes × 8 H100 = 64 GPUs. A **mixed** shop: multimodal/imaging training some days,
70B fine-tuning others. Every number below is derived, not asserted, so you can re-run it.

**Step 1 — steady-state read demand, per workload, with the mix stated.**

```
  Imaging days (3D-U-Net-shaped, ~146 MB samples, H100):
      per-GPU  = 7 × 146,600,628 / 0.323                    = 3.18 GB/s
      64 GPUs                                               = 203.5 GB/s

  Detection/classification days (RetinaNet-shaped, ~323 KB samples, H100-adjusted):
      per-GPU  ≈ 0.16 GB/s  (B200 config; H100 is ~half again slower ⇒ ~0.11)
      64 GPUs                                               = ~7.0 GB/s

  LLM fine-tune days (pre-tokenised shards):
      per-GPU  ≈ 0.00008 GB/s
      64 GPUs                                               = 0.005 GB/s

  Design point: the imaging day, because a tier sized for it covers the others.
      target from the backing store, NO cache                = 203.5 GB/s
```

**Step 2 — apply the cache tier, because this is where the money is.**

```
  Dataset of record: 400 TB in object storage.
  Working set for a given campaign: ~40 TB of packed shards.
  Cache capacity available: 8 nodes × 4 TB local NVMe                = 32 TB
                          + 8 nodes × ~1 TB spare host DRAM          =  8 TB
                          ────────────────────────────────────────────────
                                                                     = 40 TB

  working_set / cache_capacity = 40/40 = 1.0  → multi-epoch training over this
  campaign should reach a high hit rate. Assume a conservative 70% (Meta reports
  an 80% average with a purpose-built distributed cache; assume less than a
  hyperscaler until you have measured your own).

      from the parallel FS = 203.5 × (1 − 0.70)              = 61.1 GB/s
```

**Step 3 — checkpoint burst, both directions.**

```
  70B fine-tune, resumable checkpoint = 70e9 × 14 B          = 979 GB  (912 GiB)
  DP = 8 ⇒ per-node shard = 979 / 8                          = 122 GB

  Synchronous straight to the shared FS, 60 s target:
      979 GB / 60 s                                          = 16.3 GB/s write
      … and 60 s of every 1,800 s is 3.3% of GPU time, forever.

  Async, local-first (the design we choose):
      write to node-local NVMe at ~6 GB/s (one Gen5 ×4 drive):
          122 GB / 6 GB/s                                    = 20.3 s  ← ON the
                                                                GPU critical path
      with an 8-drive array, host-limited to ~40 GB/s:
          122 GB / 40 GB/s                                   =  3.1 s  ← better
      background drain to the PFS, spread over the next 10 min:
          979 GB / 600 s                                     =  1.6 GB/s
                                                                (negligible)

  δ for lesson 06's Young/Daly formula = 3.1 s with the array, 20.3 s with one
  drive. That single procurement choice changes the optimal checkpoint interval by
  sqrt(20.3/3.1) = 2.6×.
```

**Step 4 — the sizing driver, named explicitly.**

```
  max( 61.1 GB/s steady read after cache , 1.6 GB/s checkpoint drain ) = 61.1 GB/s

  Provision the PFS hot tier at ~80–100 GB/s to absorb read + drain concurrency,
  cache-miss storms at epoch boundaries, and the restore path (which is NOT
  smoothed: a restart reads 979 GB as fast as it can).

  Restore check: 979 GB / 100 GB/s = 9.8 s of pure transfer, plus MLPerf's
  cross-host remapping caveat if your FS has single-writer semantics. Budget ρ
  in lesson 06's formula accordingly.

  SIZING DRIVER: steady-state read on imaging days, after caching.
  Without the cache tier it would be 203.5 GB/s — a 3.3× more expensive
  filesystem for the identical workload.
```

**Step 5 — metadata, which decides the filesystem choice.**

```
  If samples are individual files:
      64 GPUs × (7 / 0.323) samples/s/GPU × 2 ops/sample     = 2,774 ops/s
      (3D U-Net's huge samples make this small — the dangerous case is the
       classification workload:)
      64 × (400 / 0.224) × 2                                 = 228,600 ops/s
                                                               ← single-MDT wall

  If samples are packed into 1 GiB WebDataset shards (~9,700 samples each):
      64 × 1,786 / 9,700 shards/s × 2 ops                    = 24 ops/s

  DECISION: pack the classification dataset into shards. With shards, a
  single-MDT Lustre is comfortable and the cheaper filesystem is viable. Without
  them, distributed metadata (WEKA/VAST/GPFS) or Lustre DNE across ≥4 MDTs is
  mandatory — and that is a procurement difference, not a tuning one.
```

**Step 6 — capacity and cost, per tier.**

```
  ③ local NVMe : 8 nodes × 4 TB usable                = 32 TB
                 capex at ~$100/TB raw, ~1.3× for OP  ≈ $4.2K   (drives only)
  ④ parallel FS: 300 TB usable at 80–100 GB/s
                 managed SKU at $25–70/TB-month       ≈ $7.5K–21K /month
                                                      ≈ $90K–252K /year
  ⑤ object     : 400 TB dataset of record
                 + 10 retained 70B checkpoints (9.8 TB)
                 ≈ 410 TB at S3 Standard $21–23/TB-mo ≈ $8.6K–9.4K /month
                 age checkpoints > 30 d to Standard-IA ($12.50/TB-mo) or
                 Deep Archive ($0.99/TB-mo) — on 9.8 TB that is $122 → $10/month,
                 which is small here but is the pattern that matters at 10× scale

  TOTAL STORAGE OPEX  ≈ $16K–30K / month  for a 64-GPU fleet
  as a share of the fleet's ~$129K/month all-in (lesson 08)   ≈ 12–23%
```

That last line is the one to carry into lesson 08: **storage is a double-digit percentage
of a GPU fleet's total cost of ownership**, and it is the line most often estimated by
analogy rather than computed.

**The complete manifest set** for that design is the CSI configuration in §12 — `pfs-hot`
at 200 TiB RWX, `nvme-scratch` at 4 TiB per node with `WaitForFirstConsumer`, object
accessed by SDK, and `terminationGracePeriodSeconds: 300` so lesson 06's drain does not
kill the job it is trying to save.

## Practice — feeds the deliverable

**Size the storage tier for a 64-GPU cluster and write the CSI configuration.** Deliver
into [`practice/capex-vs-cloud/`](../practice/capex-vs-cloud/README.md):

1. **A sizing sheet with every number derived.** Not "1 GB/s per GPU." Show:
   (a) per-accelerator read demand computed as
   `batch_size × bytes_per_sample / compute_time` for *your* workload mix, with the
   inputs stated; (b) the aggregate for your GPU count; (c) the cache hit rate you assume
   and the resulting backing-store demand; (d) the checkpoint size from
   `params × 14 B/param` with your model size, and the write bandwidth for your target
   write time; (e) `max(steady, burst)` with the **sizing driver named explicitly**; and
   (f) the restore-path bandwidth, which is the term everyone forgets.
2. **The metadata calculation**, in the form of §7: ops/s for your dataset layout as
   individual files versus packed shards, and a one-line decision about whether a
   single-MDT filesystem is viable for you. This is the calculation that decides which
   filesystem you can buy.
3. **A three-tier StorageClass + PVC set** — parallel-FS RWX, local NVMe with
   `WaitForFirstConsumer` (and one sentence on why `Immediate` breaks GPU scheduling), and
   object via SDK — plus the `terminationGracePeriodSeconds` that covers your checkpoint
   write time, cross-referenced to lesson 06's drain.
4. **The cost table**, per tier, with capacity × $/TB and the monthly total, expressed as a
   percentage of the fleet total you will compute in lesson 08. Flag every price as a
   dated snapshot.
5. **One tuning artifact.** Either `lfs setstripe`/`lfs getstripe` output showing a
   directory layout you set deliberately and the file that inherited it, or a `gdscheck -p`
   capture with the IOMMU line and the supported-filesystem list read out. The point is to
   demonstrate you can verify a claim about the data path rather than trust it.

**Hardware-free path.** All of the arithmetic runs on paper. For the artifacts: a
single-node Lustre or BeeGFS on loopback devices in a VM is enough to exercise
`lfs setstripe`/`getstripe` and see layout inheritance; `fio` with `--ioengine=libaio
--direct=1` and varying `--iodepth` reproduces the Little's-law curve from §6 on any disk;
and `mlpstorage` (from `mlcommons/storage`) will run the DLIO-based training and
checkpointing workloads against any mount point without a GPU, which is the closest you can
get to a real measurement on a VM.

**Acceptance:** a sizing sheet whose every number is derived from stated inputs, a metadata
calculation with a filesystem decision attached, a three-tier CSI configuration, a cost
table flagged as a snapshot, and one verification artifact — all committed, with the
storage cost and the utilisation impact carried forward as named inputs to lesson 08's
model.

## Common pitfalls

- **Sizing from a rule of thumb instead of the workload.** *Symptom:* a filesystem that is
  12× too small or 3,000× too big. *Mechanism:* per-GPU demand is
  `batch × bytes_per_sample / compute_time`, and bytes-per-sample varies by three orders of
  magnitude across real workloads. MLCommons' published configs span 0.16 to 6.33 GB/s per
  accelerator. Compute yours first, then shop.
- **Using 16 bytes per parameter for checkpoint size.** *Symptom:* a checkpoint tier
  over-sized by 14%, or an interval calculation that is subtly wrong. *Mechanism:* 16 B/param
  is *live training state* including gradients; a resumable checkpoint persists weights plus
  optimizer state only, which is **14 B/param**. MLCommons' published sizes confirm it —
  and note their table is in **GiB**, so "912 GB" misread as decimal is another 7% error.
- **Forgetting that a faster GPU makes the storage problem harder.** *Symptom:* a storage
  tier that was adequate for H100s and starves the Blackwell refresh. *Mechanism:* demand
  is inversely proportional to `computation_time`; MLCommons halves it from H100 to B200
  for the same workload, which exactly doubles the bandwidth requirement with no change to
  the dataset.
- **Leaving `stripe_count` at the Lustre default.** *Symptom:* "we bought 500 GB/s and get
  3 GB/s." *Mechanism:* the default is **1**, so each file lives on one OST and a reader
  gets one OST's bandwidth. Set the layout on the *directory* before writing into it; an
  existing file's layout cannot be changed without a migrate.
- **Ignoring metadata until it bites.** *Symptom:* GPUs at 40% while the OSTs are idle and
  every storage dashboard is green. *Mechanism:* training on millions of individual files
  makes `open`/`stat` the hot path, and a single MDT saturates in the 10⁴–10⁵ ops/s range
  while a 64-GPU classification job can demand 228,600 ops/s. Pack into shards; that is a
  ~10,000× reduction for identical bytes.
- **Assuming GPUDirect Storage is on because you configured it.** *Symptom:* half the
  expected throughput, invisible in every metric. *Mechanism:* `cuFile`'s
  `allow_compat_mode` silently stages through a CPU bounce buffer and returns success when
  the fast path is unavailable — wrong PCIe topology, IOMMU in translation mode,
  unsupported mount. Verify with `gdscheck -p` and read the IOMMU line.
- **Binding local volumes with `Immediate`.** *Symptom:* GPU pods stuck `Pending` on nodes
  with no free GPUs. *Mechanism:* `Immediate` binds the PV before the scheduler runs, which
  then pins the pod to that node regardless of GPU availability. `WaitForFirstConsumer`
  schedules first and binds second.
- **A `terminationGracePeriodSeconds` shorter than a checkpoint write.** *Symptom:* lesson
  06's remediation drain destroys the job it was supposed to save. *Mechanism:* the default
  is 30 seconds; a 122 GB per-node shard takes 20 s on one drive and far longer on a shared
  filesystem. Set the grace period from the number you computed, not from the default.
- **Mounting object storage as a filesystem.** *Symptom:* pathological latency on directory
  listings and renames. *Mechanism:* S3 has no directories and no atomic rename; a FUSE
  layer emulates them with HEAD/LIST/copy-delete sequences. Use the SDK, or COSI, and keep
  POSIX workloads on a POSIX filesystem.
- **Treating the storage network as separate from the compute fabric.** *Symptom:*
  intermittent step-time spikes that correlate with nothing on either subsystem alone.
  *Mechanism:* unless you built a separate storage rail, a checkpoint drain and an
  all-reduce contend for the same links. Model total fabric demand, not per-subsystem
  demand.

## Self-check

**(a) Roughly what aggregate GB/s keeps 64 H100s fed, and how do you get the number?**
**Answer:** There is no single answer, and producing one without naming the workload is the
mistake. Compute per-accelerator demand as
`batch_size × bytes_per_sample / compute_time_per_batch`. From MLCommons' published
configurations that gives 3.18 GB/s per H100 for 3D U-Net (7 × 146.6 MB / 0.323 s),
0.81 GB/s for CosmoFlow, 0.20 GB/s for ResNet-50 and roughly 0.0001 GB/s for LLM
pre-training on pre-tokenised shards — a five-order-of-magnitude spread. For 64 H100s on
the imaging workload that is `64 × 3.18 = 203.5 GB/s` from the backing store with no cache,
or `203.5 × (1 − 0.8) = 40.7 GB/s` at Meta's reported 80% cache hit rate. Then take the max
against the checkpoint burst (§3) and name which one is your driver. The derivation
cross-checks against MLPerf Storage v2.0 submissions: Hammerspace at 420.8 GB/s for 140
H100s and Western Digital at 106.5 GB/s for 36 H100s are 3.01 and 2.96 GB/s per GPU, within
6% of the derived figure.

**(b) Why does metadata-server design decide your filesystem choice, and what is the
arithmetic?**
**Answer:** Because a parallel filesystem has two independent paths — data (striped across
storage targets, bandwidth-bound, scales with target count) and metadata (`open`, `stat`,
`readdir`, `create`; small, synchronous, latency-bound, scales with metadata-server count).
Training on individual sample files makes metadata the hot path:
`N_gpus × samples/s/GPU × ops/sample`, which for 64 H100s at ResNet-50 rates is
`64 × 1,786 × 2 = 228,600 ops/s`. A single Lustre MDT serves on the order of 10⁴–10⁵ ops/s,
so the job is metadata-bound while the OSTs sit idle and every bandwidth dashboard looks
healthy. Packing samples into 1 GiB shards drops it to ~24 ops/s — a ~10,000× reduction for
the same bytes. So: pack first; then, if you still cannot, either distribute metadata by
architecture (WEKA, VAST, GPFS) or use Lustre DNE striped directories across several MDTs,
with Data-on-MDT for genuinely unpackable small files. The vendor question is not "how many
GB/s" but "how many metadata ops/s, and does that number grow when I add capacity?"

**(c) Where do scratch, datasets, checkpoints and weights each belong, and what breaks if
you misplace one?**
**Answer:** **Scratch → local NVMe:** lowest latency, no network hop, disposable, so it
needs no shared durability; put it on the parallel FS and you spend shared bandwidth on
bytes nobody else will read. **Datasets → parallel filesystem, fronted by a cache:** one
RWX namespace at aggregate bandwidth that scales with server count; put them on object
storage and per-object overhead plus 10–100 ms first-byte latency starves the GPUs.
**Checkpoints → local NVMe first, drained to the parallel FS hot tier, aged to object:**
the burst needs peak write bandwidth only local NVMe can absorb without disturbing readers,
the drain makes it durable and readable by other hosts (which MLPerf explicitly requires
for recovery), and ageing makes retention cheap; write them synchronously to object storage
and you put a 10–100 ms store on the GPU critical path every interval. **Weights → object
storage:** cheap, durable, versioned, lifecycle-managed, write-once/read-many, pulled to
local NVMe at job start; keeping them on the parallel FS wastes your most expensive $/TB on
write-once data and forfeits versioning.

**(d) A 70B model checkpoints every 30 minutes. How big is the checkpoint, how long does
the write take, and what does that imply for the tier?**
**Answer:** A resumable checkpoint is weights plus optimizer state and *not* gradients:
`2Ψ` bytes of bf16 weights plus `12Ψ` of fp32 Adam state (master weights, `m`, `v`) =
**14 bytes per parameter**. For 70B that is `70e9 × 14 = 979 GB`, which matches MLCommons'
published 912 GiB for their 70B configuration exactly. Written synchronously to a shared
filesystem in a 60-second target, that is `979/60 = 16.3 GB/s` of write, and 60 s of every
1,800 s is 3.3% of GPU time permanently. Written asynchronously with sharding at DP=8, each
node writes `979/8 = 122 GB` to local NVMe — 20.3 s on a single Gen5 ×4 drive at ~6 GB/s of
sustained write, or 3.1 s on an 8-drive array host-limited to ~40 GB/s — and a background
process drains it to the shared tier at a negligible 1.6 GB/s over the following ten
minutes. That local-write time *is* `δ` in lesson 06's `Δ* = sqrt(2δM)`, so choosing one
drive over an array changes the optimal checkpoint interval by `sqrt(20.3/3.1) = 2.6×`.
Do not forget the restore side: recovery reads the full 979 GB, unsmoothed, and MLPerf
requires that any cross-host "remapping" delay before a different host can read a
freshly-written checkpoint be counted in recovery time.

**(e) Meta measured 56% of GPU cycles stalled on storage. Why is that worse than it sounds,
and what fixed it?**
**Answer:** 56% stalled means more than half of every paid-for GPU-hour bought nothing —
at $2.50/GPU-hr the effective cost per *useful* GPU-hour becomes `2.50/0.44 = $5.68`, a
2.3× multiplier that appears nowhere in the budget. It is worse than a bandwidth shortfall
because there is no error, no alert and no failed request; the only symptom is a
utilisation number and a schedule that slips. The fix was architectural rather than
hardware: a Data PreProcessing Service, a distributed cache built from unused GPU-host
memory reaching an **average 80% hit rate**, and metadata brought down to **1–2 ms** — on
top of the exabyte-scale Tectonic filesystem they already had. The transferable lesson is
the hit rate: since the backing store only supplies the misses,
`BW = N × per_gpu_demand × (1 − hit_rate)`, an 80% hit rate is a **5× reduction in the
filesystem you have to buy**. That is the cheapest bandwidth in the stack, and it is why
the first storage design question is "what fraction of accesses can I serve from a cheaper
tier," not "how fast is the expensive one."

**(f) You are told GPUDirect Storage is enabled and throughput is unchanged. What do you
check, in what order?**
**Answer:** First `gdscheck -p`, and read three specific things: whether your mount type
appears in the supported-filesystem list, whether every GPU line says `supports GDS`, and
the `IOMMU:` line — many platforms only deliver the peer-to-peer fast path with the IOMMU
disabled or in passthrough (`iommu=pt`). Second, check whether `allow_compat_mode` is
`true` in `/etc/cufile.json`, because compat mode silently stages through a CPU bounce
buffer **and returns success**, so working code with successful calls is fully consistent
with no benefit. Third, check topology: peer-to-peer DMA is fast when the NVMe and the GPU
share a PCIe switch (`PIX`/`PXB` in `nvidia-smi topo -m`), platform-dependent across a root
complex (`PHB`), and not a supported fast path across sockets (`SYS`) — so a placement
problem presents as a GDS problem. Fourth, measure rather than infer: `gdsio -x 0` versus
`-x 1` on the same file gives you the fast-path and bounce-path numbers side by side (on a
correctly-placed drive, roughly 12.9 versus 6.2 GiB/s). And note `max_direct_io_size_kb`
(16,384 by default): cuFile splits larger requests, so a 64 MiB read is four operations and
your effective queue depth is four times what you assumed.

## Connections & what's next

This lesson closes the technical half of the module's arc. Lesson 06 showed that a *broken*
node stops paying for itself if you do not catch it fast; this lesson shows the same is
true of a *healthy* node starved of bytes. Both reduce to one accounting line: GPU-hours
paid for and not used.

It also settles a debt to lesson 06. The `δ` in `Δ* = sqrt(2δM)` is not a constant — it is
`per_node_checkpoint_shard / local_write_bandwidth`, and §10 shows that one procurement
choice moves it by 6.5× and the optimal checkpoint interval by 2.6×. The
`terminationGracePeriodSeconds` that makes lesson 06's drain safe is derived here too. And
storage traffic shares the physical fabric with the collectives module 09 designed, so the
storage design and the network design are the same design viewed twice.

Next is **10.8, the module's capstone**: the capex-versus-cloud crossover model. Everything
here becomes an input line. The parallel filesystem is real capex and real opex — §6 of the
worked example puts it at 12–23% of a 64-GPU fleet's total cost of ownership, which is far
too large to estimate by analogy. The cache tier is capital that *reduces* another line by
5×, which is the shape of argument a capex model exists to make. And a storage-induced
stall is a direct subtraction from the **utilisation** term in the denominator of owned
$/GPU-hr — the term lesson 08 will show dominates every other assumption in the model.
Carry the sizing sheet and the cost table from this lesson's Practice straight into 10.8.

## References & further reading

**Primary sources (verified against upstream source this session)**

- **MLCommons MLPerf Storage** — <https://github.com/mlcommons/storage> — read 2026-08-18
  at commit `ce28a98`. Source for the Accelerator Utilisation definition and per-workload
  minimums (`training/README.md`); the workload configurations this lesson derives its
  per-accelerator bandwidth table from (`configs/dlio/workload/{unet3d_b200,retinanet_b200,cosmoflow_h100,resnet50_h100}.yaml`
  — `batch_size`, `record_length_bytes`, `computation_time`); and the checkpointing
  workload (`checkpointing/README.md`) with the model table (8B 105 GiB, 70B 912 GiB, 405B
  5.29 TiB, 1T 18 TiB), the enforced `fsync`, the 10-write/10-read submission protocol, the
  cache-clearing rule, and the requirement that checkpoint reads be performed by different
  hosts with any remapping time added to recovery. **Correction applied from this source:**
  an earlier version of this lesson used 16 bytes/parameter and reported ~1.1 TB for a 70B
  checkpoint; the correct resumable-checkpoint figure is **14 B/param** (weights + optimizer,
  no gradients), giving 979 GB / 912 GiB, which matches MLCommons' published table exactly.
  The earlier "~1 GB/s per GPU" planning rule has also been replaced by the derived
  per-workload table.
- **Lustre** — <https://github.com/lustre/lustre-release> — read 2026-08-18 at commit
  `4322543`. Source for the defaults in §6: `stripe_count` default **1** and `stripe_size`
  default **4 MiB** (`Documentation/man1/lfs-setstripe.1`); `OBD_MAX_RIF_DEFAULT` = **8**
  with `OSC_MAX_RIF_MAX` = 256 and `OBD_MAX_RIF_MAX` = 512, and
  `OSC_MAX_DIRTY_DEFAULT` = **2000 MB** with a 2048 MB maximum (`lustre/include/obd.h`);
  and the bulk-RPC sizes `DT_DEF_BRW_SIZE` = 4 MiB and `PTLRPC_MAX_BRW_SIZE` = 64 MiB, from
  `LNET_MTU_BITS` = 20 and `PTLRPC_BULK_OPS_BITS` = 6 (`lustre/include/lustre_net.h`,
  `include/uapi/linux/lnet/lnet-types.h`). *`doc.lustre.org` and `wiki.lustre.org` are
  blocked by this session's egress proxy and were not read; everything above is from the
  source tree.*
- **NVIDIA DALI** — <https://github.com/NVIDIA/DALI> — read 2026-08-18. Source for the
  `prefetch_queue_depth` default of **2** (`dali/python/nvidia/dali/pipeline.py`), the
  loader-side knob that decides whether the data pipeline is deep enough to hide storage
  latency behind compute.
- **NVIDIA GPUDirect Storage design guide and `cuFile` reference** —
  <https://docs.nvidia.com/gpudirect-storage/> — *`docs.nvidia.com` is blocked by this
  session's egress proxy and was not fetched.* The GDS mechanics, the `allow_compat_mode`
  behaviour, `gdscheck -p` output fields, `max_direct_io_size_kb` = 16,384, and the
  `gdsio -x 0` versus `-x 1` measurements (12.914 versus 6.201 GiB/s) are carried forward
  from [lesson 02b.6](../../02b-host-topology/lessons/06-storage-nvme.md), which verified
  them against that source when it was reachable.
- **IBM Storage Scale (GPFS) configuration and tuning** — <https://www.ibm.com/docs/en/storage-scale> —
  *`ibm.com` is blocked by this session's egress proxy and was not fetched.* The defaults
  quoted in §6 (`pagepool` = the smaller of one third of physical memory or 4 GiB;
  `maxFilesToCache` = 4,000; `maxStatCache` = the larger of 1,000 or 4× `maxFilesToCache`;
  keep their sum ≤ 50% of node memory) come from search extracts of IBM's tuning
  documentation and are **labelled as unverified against the primary source** — check
  `mmlsconfig` on your own cluster before relying on them.
- **Kubernetes CSI / `sig-storage-local-static-provisioner`** —
  <https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner> — the mechanics
  behind the `nvme-scratch` StorageClass and the `WaitForFirstConsumer` binding behaviour
  described in §12.
- **Container Object Storage Interface (COSI)** —
  <https://github.com/kubernetes-sigs/container-object-storage-interface> — the
  Kubernetes-native way to manage bucket lifecycle for tier ⑤ without pretending object
  storage is a filesystem.

**Real-world engineering**

- **Meta — "Meta's AI Storage Blueprint at Scale" (2026)** —
  <https://engineering.fb.com/2026/07/01/data-infrastructure/metas-ai-storage-blueprint-at-scale/>
  — **what it shows:** the 56% GPU-stall measurement, the Data PreProcessing Service, the
  distributed cache built from unused GPU-host memory at an average 80% hit rate, and 1–2 ms
  metadata latency, on top of Tectonic. *Domain blocked by this session's egress proxy and
  not fetched; figures taken from independent secondary reports —*
  <https://www.sdxcentral.com/news/meta-rebuilds-its-ai-storage-stack-from-the-ground-up-to-stop-gpus-sitting-idle/>
  *and*
  <https://www.techradar.com/pro/a-disk-in-a-planet-scale-computer-meta-has-so-many-expensive-gpus-that-its-buying-ssds-to-kill-idle-time>
  *— which agree on the 56% and 80% figures.*
- **MLPerf Storage v2.0 results coverage (August 2025)** —
  <https://blocksandfiles.com/2025/08/05/storage-arrays-get-faster-in-2nd-version-of-mlperf-storage-benchmark/>
  and
  <https://blocksandfiles.com/2025/08/08/yanrong-excels-at-3d-u-net-in-mlperf-storage-benchmark/>
  — **what they show:** submitted 3D U-Net figures (Hammerspace 420.8 GB/s / 140 simulated
  H100s at 96.4% AU; Western Digital OpenFlex Data24 106.5 GB/s / 36 H100s; YanRong 513 GB/s
  peak) that independently corroborate the ~3 GB/s-per-H100 figure derived from the
  workload configs. *Results pages on `mlcommons.org` are blocked by this session's egress
  proxy; these are trade-press reports of them.*
- **Meta — "Building Meta's GenAI Infrastructure" (2024)** —
  <https://engineering.fb.com/2024/03/12/data-center-engineering/building-metas-genai-infrastructure/>
  — **what it shows:** the storage and network build-out that the 2026 rebuild sat on top
  of, i.e. that a hyperscaler's day-one storage architecture still needed a ground-up
  rebuild two years later. Sizing is not a one-time exercise. *Domain blocked; not fetched
  and not relied upon for any specific figure here.*

**Deeper dives**

- **Lesson 02b.6 — Storage and NVMe on the host** —
  [`../../02b-host-topology/lessons/06-storage-nvme.md`](../../02b-host-topology/lessons/06-storage-nvme.md)
  — the layer beneath this one: NVMe queue pairs and doorbells, blk-mq, Little's law for
  queue depth, the three PCIe placements and their ceilings, `nvme-cli` field by field, and
  the GDS verification procedure. Read §5 and §8 there before tuning anything in §6 or §11
  here.
- **AWS S3 pricing** — <https://aws.amazon.com/s3/pricing/> — the tier ⑤ price anchors used
  in §4 and the worked example (Standard $0.023/GB-month for the first 50 TB tapering to
  $0.021 above 500 TB; Standard-IA $0.0125 plus retrieval; Glacier Deep Archive $0.00099).
  *`aws.amazon.com` is blocked by this session's egress proxy; figures are from search
  extracts of the pricing page and multiple 2026 pricing guides that agree. Re-quote before
  budgeting, and read [lesson 11.7](../../11-gpu-cost-economics/lessons/07-neocloud-vs-hyperscaler.md)
  for why egress, not storage, is often the larger line.*
