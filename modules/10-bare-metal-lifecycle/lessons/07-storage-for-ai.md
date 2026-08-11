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
sources: 8
---

# 10.7 · Storage for AI: parallel filesystems, tiering, and feeding the GPUs

> **Concept.** Size a storage hierarchy — local NVMe scratch, a parallel filesystem for datasets, object storage for weights and checkpoints — to the aggregate GB/s a GPU count demands, and wire it in through CSI so a slow data path never starves six-figure silicon.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Where this fits

Lesson 06 closed the **hardware health loop**: detect a sick node in minutes, cordon/drain it without losing the job, RMA it, and feed the failure back into procurement. That loop protects goodput from *broken* hardware. This lesson protects goodput from **perfectly healthy** hardware that is starved anyway — a GPU with a green DCGM health check and zero XIDs, sitting at 40% `GPU-Util` because the dataloader thread is blocked on a read. Same symptom the module has been chasing since lesson 05 (idle six-figure silicon), different subsystem: not the compute node, but everything that feeds it bytes. With provisioning (01–05) and health (06) covered, this is the last purely *technical* lesson before the module's capstone turns all of it into a dollar figure.

## Why this matters

An H100 costs on the order of $2–3/GPU-hr to rent or own (2026 snapshot; you'll model it precisely in 10.8). If the data path can't keep the GPU's SMs busy, you are paying that rate to idle. A slow filesystem shows up not as an error but as `GPU-Util` sitting at 40% while `dataloader` threads block on I/O — the single most common way a bare-metal AI cluster silently wastes half its capex.

That is not a hypothetical framing — it is Meta's own number. In its 2026 storage-infrastructure retrospective, Meta reported measuring **56% of GPU cycles stalled waiting for training data** before it rebuilt its storage stack ([Meta engineering blog, 2026](https://engineering.fb.com/2026/07/01/data-infrastructure/metas-ai-storage-blueprint-at-scale/); corroborated independently by the [ZenML LLMOps database](https://www.zenml.io/llmops-database/reimagining-storage-infrastructure-for-ai-model-training-at-scale) and [SDxCentral](https://www.sdxcentral.com/news/meta-rebuilds-its-ai-storage-stack-from-the-ground-up-to-stop-gpus-sitting-idle/)). More than half of Meta's GPU-hours, at Meta's scale, were being spent waiting on I/O rather than computing. If you take nothing else from this lesson, take that number: at a shop the size of Meta, "storage starves GPUs" is not a tail risk, it is the default state until someone engineers against it.

At a neocloud, storage throughput per GPU is a line item customers benchmark before they sign; getting it wrong is a churn event. This is also where on-prem knowledge is a differentiator. Managed clouds hand you EFS/FSx-Lustre/GCS as opaque endpoints. On bare metal you choose the filesystem, size the metadata tier, lay out NVMe, and own the GPUDirect path. That is the job at CoreWeave/NVIDIA-class shops.

## What's new here (calibration)

Module 04 (GPU-on-Kubernetes) covered device plugins, MIG, and scheduling GPUs. Networking (09) covered the fabric. Neither covered **where the bytes the GPU eats come from**. New here:

- **The GPU is fed by a hierarchy, not one store.** Four distinct data classes (scratch, datasets, checkpoints, weights) with different access patterns, each belonging on a different tier.
- **Parallel filesystems** (WEKA, Lustre, GPFS/Spectrum Scale, VAST) — how they stripe a single namespace across many storage nodes to hit hundreds of GB/s aggregate, and why **metadata-server design** (WEKA's distributed metadata vs Lustre's classic single-MDS) is the scaling ceiling.
- **Throughput sizing as arithmetic**: turning a GPU count into a target aggregate GB/s and a checkpoint-burst bandwidth — and why the industry now treats checkpoint-burst sizing as a *standardized, benchmarked* problem, not a back-of-envelope guess.
- **GPUDirect Storage (GDS)** — DMA from NVMe/NIC straight into GPU HBM, bypassing the CPU bounce buffer.
- **CSI integration** — expressing all of this as StorageClasses so pods request the right tier by name.

## Core concepts

### The four data classes and where each belongs
| Data class | Access pattern | Belongs on | Why |
|---|---|---|---|
| **Scratch / cache** | ephemeral, per-node, random + streaming, highest IOPS | **local NVMe** on the GPU node (`local` PV / `emptyDir` on NVMe) | lowest latency, no network hop; it's throwaway, so no need for shared durability |
| **Training datasets** | read-mostly, shared by every node, high sequential aggregate | **parallel filesystem** (WEKA/Lustre/VAST/GPFS) | one namespace, hundreds of GB/s spread across all readers; random-access to shards |
| **Checkpoints** | bursty large sequential **writes**, then cold | **parallel FS hot tier → object storage** | write burst needs peak bandwidth; old checkpoints tier down to cheap object |
| **Model weights / artifacts** | write-once, read-many, versioned, durable | **object storage** (S3/MinIO/Ceph RGW) | cheap, durable, versioned; pulled to NVMe cache at job start |

The rule: **scratch is local and disposable; datasets are shared and hot; checkpoints burst then cool; weights are durable and cheap.** Putting datasets on object storage starves the GPUs; putting checkpoints only on object storage makes the write barrier the bottleneck; putting scratch on the shared FS wastes parallel-FS bandwidth on throwaway bytes.

### Sizing throughput: from GPU count to GB/s
The estimate is a Fermi calculation, not a spec-sheet lookup. Two independent budgets:

**1. Steady-state dataset read bandwidth.** Per-GPU appetite depends on the workload:
- **LLM pre-training** is compute-heavy relative to bytes: tokenized data is tiny, so per-GPU read demand is often **low — ~0.1–0.5 GB/s/GPU**. Storage is rarely the bottleneck here *except at checkpoint time*.
- **Image/video/multimodal training** with on-the-fly decode is byte-hungry: **~1–4 GB/s/GPU** is common; large-frame video can exceed it.
- The rule of thumb the vendors quote for a mixed AI cluster is roughly **~1 GB/s per GPU** of sustained read as a planning number (2026 snapshot). For 64 GPUs that's **~64 GB/s**; a **512-GPU cluster lands in the ~400–600 GB/s** band — which is why big clusters buy WEKA/VAST rather than a single NFS box.

**2. Checkpoint write burst — usually the sizing driver.** A checkpoint writes the full model + optimizer state. For a model with `P` parameters trained in mixed precision, the checkpoint (fp32 master weights + Adam moments m,v) is roughly:
```
bytes ≈ P × (4 weights + 4 grad-less + 4 m + 4 v)  ≈ 16 bytes/param   (fp32 optimizer state)
```
A 70B model ⇒ ~**1.1 TB** per checkpoint (sharded across ranks). If you want that written in, say, **30 s** so GPUs resume fast, you need **~1.1 TB / 30 s ≈ 37 GB/s** of *write* burst — often **larger than your steady read demand**, and it hits all at once every N steps. Size the hot tier for `max(steady_read, checkpoint_burst)`, and decide **checkpoint cadence** as a tradeoff: more frequent = less lost work on a failure (MTBF on a big fleet is hours, not days), but more bandwidth tax and more GPU stall. Async/sharded checkpointing (write to local NVMe, drain to FS in background) decouples the burst from GPU stall and is how large shops hide it.

This is not a made-up scenario. As of August 2025, **MLCommons standardized it**: the industry-consortium benchmark suite added the [MLPerf Storage v2.0 Checkpointing workload](https://mlcommons.org/2025/08/storage-2-checkpointing/) specifically to measure a storage system's checkpoint-write and checkpoint-restore bandwidth for large models. When an industry consortium ships a benchmark for a specific sizing scenario, that is your strongest evidence it is a real, recurring, expensive problem across the industry — not a one-off you're overthinking.

### Parallel filesystems — the aggregate-bandwidth machines
A parallel FS presents **one POSIX namespace** but stripes file data across many storage servers, so `N` clients reading get `N`-way aggregate bandwidth instead of one server's NIC.

| FS | Architecture | Metadata | Notes (2025/2026) |
|---|---|---|---|
| **WEKA** | software-defined, NVMe, own network FS protocol; tiers to S3 | **distributed metadata** across all nodes | scales metadata with capacity; strong small-file/mixed; common at GPU neoclouds |
| **Lustre** | OSS/OST data servers + **MDS/MDT** metadata | classically **single active MDS** (DNE adds more, but it's bolt-on) | huge sequential bandwidth, cheap per TB; **MDS is the classic bottleneck** on metadata-heavy (many-small-file) workloads |
| **GPFS / IBM Spectrum Scale** | shared-disk, distributed metadata | distributed | mature, enterprise HPC; strong but license-heavy |
| **VAST** | disaggregated DASE, all-flash, NFS/RDMA | distributed, shared everything | simple ops, good metadata scaling; popular in newer AI DCs |
| **BeeGFS** | OSS-style, distributable metadata | can add metadata servers | cheap/OSS, common in smaller HPC |

A GPU-hosting neocloud's own practitioner comparison of exactly these three ([WhiteFiber — "Storage for AI Workloads: Ceph, VAST, and WEKA"](https://www.whitefiber.com/blog/ai-storage-ceph-vast-weka)) is a good companion read once you know which axis (metadata scaling, GPUDirect RDMA support, cost/TB) matters for your workload — it walks the same tradeoffs from the operator side, on hardware they actually rent out.

**Why metadata design matters at scale (self-check b).** Training reads millions of small files (image shards, tokenized samples): every `open()`/`stat()`/`readdir()` is a metadata op. On **Lustre with a single MDS**, all of that funnels through one server — the data path can be at 500 GB/s while the job crawls because the MDS is pegged at 100% CPU serving `stat`s. **WEKA/VAST distribute metadata** across all nodes, so metadata IOPS scale with the cluster and the small-file wall moves out. This is why "what filesystem?" is really "what's your file-count and access pattern?": big sequential files (video, sharded tensors) forgive a single MDS; billions of tiny files do not. Mitigations on Lustre: DNE (multiple MDTs), or packing samples into large shards (WebDataset/tar, TFRecord) so the metadata op count collapses.

### GPUDirect Storage (GDS)
Normally a read goes NVMe/NIC → **CPU bounce buffer in system RAM** → GPU HBM, burning CPU cycles and PCIe bandwidth twice. **GPUDirect Storage** lets the storage DMA straight into GPU memory over PCIe/NVLink, bypassing the CPU bounce. Accessed via the **cuFile** API (and integrated in DALI, and in WEKA/VAST/Lustre clients that support it). Single-stream GDS delivers on the order of **~10–12 GB/s** (2026 snapshot); it matters most when the CPU-bounce copy is your ceiling — large sequential reads into HBM, or checkpoint restore. It needs a compatible NIC/NVMe, MOFED, and a client that speaks it. Don't assume it's free: verify the whole path (`gdscheck`) or you'll silently fall back to the bounce buffer.

### CSI integration — expressing tiers as StorageClasses
Everything above becomes **StorageClasses** so pods request a tier by name:
- **Local NVMe scratch** — the [local-static-provisioner](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner) or `local` volumes with `volumeBindingMode: WaitForFirstConsumer` (bind the PVC only once the pod is scheduled, so it lands on a node that has the NVMe). Or plain `emptyDir` on an NVMe-backed kubelet root for pure throwaway.
- **Parallel FS** — vendor CSI driver (WEKA CSI, VAST CSI, Lustre via the [aws-fsx-csi-driver](https://github.com/kubernetes-sigs/aws-fsx-csi-driver) or a generic Lustre CSI, GPFS CSI). Usually `ReadWriteMany`, one big shared PVC mounted by every training pod.
- **Object storage** — not a filesystem mount; accessed via S3 SDK, or surfaced through the [container-object-storage-interface (COSI)](https://github.com/kubernetes-sigs/container-object-storage-interface) / a MinIO gateway. Weights/checkpoints live here.

## Perspectives

**Developer / ML-engineer view.** From inside a training script, storage is invisible until it isn't: a `DataLoader` with too few worker threads or a dataset mounted on the wrong tier shows up as low `GPU-Util` with no error, no stack trace, nothing to `grep`. The fix is rarely code — it's usually "this dataset is on the wrong tier" or "checkpoint writes are synchronous and blocking the training loop." Knowing the four-tier model above turns a mysterious slowdown into a one-line diagnosis.

**Operator / platform-engineer view.** You don't get to blame the framework. You choose the filesystem, size the metadata tier, provision the NVMe scratch capacity per node, and wire the CSI StorageClasses so requesting the right tier is a one-line PVC, not tribal knowledge. Your job is to turn the Fermi-calculation sizing above into a procurement decision *before* the cluster is built — retrofitting a parallel FS under a live 64-GPU training job is far more expensive than sizing it correctly up front.

**Hardware / kernel view.** The bytes travel a real physical path: NVMe → PCIe/NIC → (network fabric, see module 09) → PCIe → GPU HBM, or with GDS, a shorter DMA path that skips the CPU bounce buffer entirely. Whether that path is fast depends on things invisible from the training script: NIC generation, whether RDMA/GPUDirect RDMA is enabled between storage and compute nodes, and whether the storage client actually negotiated the GDS path rather than silently falling back. This is the same fabric-quality problem module 09 covered for compute traffic — storage traffic rides the same physical network and competes for the same bandwidth.

**Economics view.** Every GB/s of storage bandwidth you under-provision converts directly into idle GPU-hours, and idle GPU-hours are the single input that dominates the crossover model you'll build in 10.8 — utilisation is the denominator of the owned-$/GPU-hr equation, and a storage stall is exactly the kind of utilisation-killer that equation is sensitive to. Meta's 56% figure, read through that lens, is not a storage statistic — it's roughly a 2x cost multiplier on every GPU-hour paid for during the stalled period.

## Real-world use cases

- **Meta — "Meta's AI Storage Blueprint at Scale" (2026).** Meta measured **56% of GPU cycles stalled waiting for training data**, with exabytes of training data living in its **Tectonic** distributed filesystem. It built a new **Data PreProcessing Service (DPP)** plus a tiered-caching object-storage layer specifically to eliminate the stalls. What it shows: at hyperscaler scale, the "GPU idle because storage is slow" problem is the *default* state, not an edge case, and it takes a dedicated engineering program to fix. [engineering.fb.com](https://engineering.fb.com/2026/07/01/data-infrastructure/metas-ai-storage-blueprint-at-scale/) (canonical Meta engineering blog URL; cross-referenced, not independently fetched this session, by [ZenML's LLMOps database](https://www.zenml.io/llmops-database/reimagining-storage-infrastructure-for-ai-model-training-at-scale) and [SDxCentral](https://www.sdxcentral.com/news/meta-rebuilds-its-ai-storage-stack-from-the-ground-up-to-stop-gpus-sitting-idle/)).
- **Meta — "Building Meta's GenAI Infrastructure" (2024).** An earlier, complementary piece describing the hardware/network/storage build-out for Meta's GenAI training clusters — a good theory-to-practice pairing for this lesson's sizing worked example below, showing what the storage tier looks like *before* the 2026 rebuild that produced the 56% number. What it shows: the storage architecture a hyperscaler ships on day one still needs a from-the-ground-up rebuild two years later as model/dataset scale grows — sizing isn't a one-time exercise. [engineering.fb.com](https://engineering.fb.com/2024/03/12/data-center-engineering/building-metas-genai-infrastructure/) (canonical URL, not independently fetched this session; also discussed at [Hammerspace's guest-post writeup](https://hammerspace.com/guest-post-building-metas-genai-infrastructure/)).
- **WhiteFiber — "Storage for AI Workloads: Ceph, VAST, and WEKA."** A GPU-hosting neocloud's own comparison of the three storage backends this lesson's table covers, with GPUDirect RDMA specifics from an operator who runs all three for tenants. What it shows: the metadata/architecture tradeoffs in the table above aren't academic — they're what a neocloud actually weighs when deciding which backend to offer customers. [whitefiber.com](https://www.whitefiber.com/blog/ai-storage-ceph-vast-weka) (canonical URL, not independently fetched this session).
- **WEKA — customer performance claims (vendor-marketing, flag accordingly).** WEKA's own site reports an anonymized customer's training runs dropping from **40–80 hours to 4–6 hours**, claims **1.8 TB/s** bandwidth in a single rack, and up to **20x GPU-utilization improvement**. What it shows: even discounting for vendor framing, the *shape* of the claim — a multi-x reduction in wall-clock training time purely from a storage-tier swap, no compute change — matches the mechanism this lesson teaches (storage stalls are large enough that fixing them alone changes wall-clock time by multiples). Treat the specific numbers as a vendor's best case, not an industry average. [weka.io](https://www.weka.io/article/why-storage-architecture-is-the-new-bottleneck-for-hpc-and-ai-teams) (canonical URL, not independently fetched this session; **vendor-published, not independently audited**).

## Worked example — sizing storage for a 64-GPU H100 cluster (mixed workload)
8 nodes × 8 H100 = 64 GPUs. Assume a **mixed** shop: some multimodal training (byte-hungry) plus LLM fine-tuning of up to a 70B model.

1. **Steady dataset read.** Planning number ~1 GB/s/GPU ⇒ **~64 GB/s aggregate** target from the parallel FS. Multimodal-heavy days push higher; LLM-only days far lower.
2. **Checkpoint burst.** 70B fine-tune, fp32 optimizer state ⇒ ~16 B/param ⇒ **~1.1 TB** per checkpoint. Target a 30 s write ⇒ **~37 GB/s write burst**. Cadence: checkpoint every ~30 min (fleet MTBF on 8 nodes is days, but a 30-min cadence caps lost work at 30 min for ~1 min of amortized write tax). Use **async checkpointing**: ranks write shards to **local NVMe first** (see below), then a background drain copies to the FS/object tier — so the 37 GB/s never stalls the GPUs. This is exactly the pattern the MLPerf Storage v2.0 checkpointing benchmark measures: write bandwidth and restore bandwidth under a realistic large-model shard pattern, not a synthetic sequential-write test.
3. **Sizing driver** = `max(64 GB/s read, 37 GB/s burst)` ≈ **~64 GB/s**, but provision the FS hot tier to **~80–100 GB/s** to absorb read+drain concurrency. A single VAST/WEKA cluster of a handful of NVMe nodes hits this comfortably; a single NFS box does not.
4. **Local NVMe scratch.** Each node: size scratch to hold the largest local checkpoint shard + dataset cache. ~2× the per-node checkpoint shard (1.1 TB / 8 ≈ 140 GB) plus dataset working set ⇒ provision **~3.5 TB usable NVMe/node** as scratch.
5. **Object tier.** Weights + cold checkpoints on S3/MinIO/Ceph — size for retained-checkpoint history (e.g. keep last 10 ⇒ ~11 TB) + weight artifacts; cheap and durable, no bandwidth target beyond restore.

StorageClass sketch (three tiers):
```yaml
# 1) Hot shared datasets + checkpoint hot tier — parallel FS (example: WEKA CSI)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pfs-hot
provisioner: csi.weka.io
parameters:
  filesystemGroupName: gpu-hot
  capacityEnforcement: "HARD"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: Immediate
mountOptions: ["readahead_kb=8192"]   # tune for large sequential reads
---
# 2) Local NVMe scratch — bind only where the disk physically is
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nvme-scratch
provisioner: kubernetes.io/no-provisioner   # local-static-provisioner backs it
volumeBindingMode: WaitForFirstConsumer      # schedule pod first, then bind local PV
reclaimPolicy: Delete
```
```yaml
# Training pod: shared dataset RWX + fast local scratch
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: datasets }
spec:
  accessModes: ["ReadWriteMany"]
  storageClassName: pfs-hot
  resources: { requests: { storage: 50Ti } }
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: scratch }
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: nvme-scratch
  resources: { requests: { storage: 3500Gi } }
```
Weights/checkpoint-cold: **no PVC** — the training container reads/writes S3 via SDK to `s3://weights/...` and `s3://ckpt-cold/...`.

## Practice — feeds the deliverable
**Size the storage tier for a 64-GPU cluster and write the CSI/StorageClass config.** Deliver:
1. **A one-page sizing sheet** with your own assumptions: target **aggregate GB/s** (show the per-GPU × 64 arithmetic and your workload mix), **checkpoint size** (state your model size and the 16 B/param math), **checkpoint cadence and burst GB/s**, and the **hot-NVMe-scratch capacity per node** vs the **object-storage** retention. State explicitly which of steady-read vs checkpoint-burst is your sizing driver.
2. **The StorageClass + PVC YAML** for three tiers (parallel-FS hot, local NVMe scratch with `WaitForFirstConsumer`, object via SDK) — a working config, not pseudocode.
3. One sentence on **metadata**: given your file layout (many small files vs packed shards), whether a single-MDS Lustre would bottleneck you, and your mitigation.

**Acceptance:** a 64-GPU storage sizing sheet (aggregate GB/s + checkpoint burst + tiering) **and** a three-tier StorageClass/PVC config, committed into the deliverable folder ([`practice/capex-vs-cloud/`](../practice/capex-vs-cloud/README.md)). It must show the GB/s arithmetic and name the sizing driver — the 10.8 capex model will use your storage line as one of its opex inputs.

## Common pitfalls

- **Sizing off a spec sheet instead of a workload Fermi calc.** "This filesystem does 200 GB/s" tells you nothing about whether *your* 64 GPUs need 64 GB/s or 6.4 GB/s — always compute per-GPU appetite × GPU count first, then shop.
- **Treating checkpoint-burst as an afterthought.** It's easy to size only for steady-state reads and get blindsided by a 37 GB/s write spike every 30 minutes. This is now an industry-recognized, *benchmarked* problem — MLPerf Storage v2.0 shipped a checkpointing workload in August 2025 specifically because enough shops hit this — not a corner case you're overthinking.
- **Ignoring metadata-server architecture until it bites.** A single-MDS Lustre setup looks fine in a proof-of-concept with a few large files; it falls over once real training data means millions of small shards. Ask about metadata scaling *before* you commit to a filesystem, not after the job queue backs up.
- **Assuming GPUDirect Storage is "just on."** GDS requires a compatible NIC/NVMe, MOFED, and a client that actually negotiated the cuFile path. Without verifying with `gdscheck`, you can be silently running the slower CPU-bounce path while believing you configured GDS — the failure is invisible in `nvidia-smi`.
- **Misplacing a data class onto the wrong tier.** Datasets on object storage starve GPUs (too slow, too much per-object overhead); scratch on the shared parallel FS wastes bandwidth every other job needs; checkpoints living only on object storage make every checkpoint write a slow, blocking network call instead of a fast local write + async drain.

## Self-check
**(a) Roughly what aggregate GB/s do you need to keep 64 H100s fed, and how do you estimate it?**
**Answer:** Two budgets, take the max. **Steady read** ≈ per-GPU appetite × 64; at the ~1 GB/s/GPU planning rule that's **~64 GB/s** (lower for LLM-only, higher for multimodal decode). **Checkpoint burst** ≈ (params × ~16 B for fp32 optimizer state) ÷ target write time — e.g. a 70B model ⇒ ~1.1 TB ÷ 30 s ≈ **~37 GB/s** of write, often the real driver, hitting all at once every N steps. Provision the hot tier to `max(read, burst)` plus headroom (~80–100 GB/s), and use async checkpointing to keep the burst off the GPU critical path.

**(b) Why does metadata-server design (WEKA distributed vs Lustre MDS) matter at scale?**
**Answer:** Training on millions of small files makes `open`/`stat`/`readdir` — metadata ops, not data — the hot path. A **single active MDS (classic Lustre)** funnels all of them through one server; it pegs at 100% CPU and the job crawls even while the data path is nearly idle. **Distributed metadata (WEKA/VAST/GPFS)** scales metadata IOPS with the cluster, moving the small-file wall out. Mitigations on Lustre: DNE multi-MDT, or pack samples into large shards (WebDataset/tar, TFRecord) to collapse the metadata op count.

**(c) Where do model weights, datasets, checkpoints, and scratch each belong, and why?**
**Answer:** **Scratch** → local NVMe on the GPU node: lowest latency, disposable, no shared-durability need. **Datasets** → parallel FS: one shared namespace read by every node at hundreds of GB/s aggregate. **Checkpoints** → parallel-FS hot tier for the write burst, then tier down to object storage as they cool. **Weights** → object storage: cheap, durable, versioned, write-once/read-many, pulled to NVMe cache at job start. Misplacing any one (datasets on object, checkpoints only on object, scratch on the shared FS) either starves the GPUs or wastes the expensive shared bandwidth.

**(d) Meta measured 56% of GPU cycles stalled on storage. Why is that number more damning than it first looks, and what did fixing it require?**
**Answer:** 56% stalled means more than half of every paid-for GPU-hour, at hyperscaler scale, was buying nothing — a rough 2x effective cost multiplier during the stalled period, on top of six-figure-per-node hardware. It required more than "buy a faster filesystem": Meta built a dedicated **Data PreProcessing Service (DPP)** to eliminate the stalls and a **tiered-caching object-storage layer**, on top of its existing exabyte-scale **Tectonic** filesystem — i.e. a purpose-built preprocessing/caching layer, not just bigger disks. The lesson for a smaller fleet: budget engineering time for the data-loading path itself, not only for filesystem procurement.

## Connections & what's next

This lesson closes the technical half of the module's arc. Lesson 06 proved that a *broken* node stops paying for itself if you don't catch it fast; this lesson proves the same thing about a *healthy* node starved of bytes — both failure modes reduce to the same number, GPU-hours paid for but not used. Storage also touches lessons 04 and 09 directly: the CSI StorageClasses here sit alongside the GPU device-plugin scheduling from module 04, and every GB/s moved between a parallel-FS node and a GPU node rides the same network fabric module 09 built.

Next is **10.8, the module's capstone**: the capex-vs-cloud crossover model. Everything this lesson taught — the storage hardware you provisioned, the bandwidth you sized, and the idle-GPU cost of getting it wrong — becomes an input line in that model. A parallel-FS cluster is real capex and real opex (power, maintenance, an FTE fraction); a storage-induced stall is a direct hit to the **utilisation** term that dominates the owned-$/GPU-hr equation. Carry the sizing sheet you build in this lesson's Practice section straight into 10.8's model as your storage cost/utilisation input.

## References & further reading

**Primary sources**
- **NVIDIA GPUDirect Storage design guide** — <https://docs.nvidia.com/gpudirect-storage/design-guide/index.html> — the authoritative GDS/cuFile reference for the DMA-into-HBM path; use `gdscheck` to verify the whole chain before trusting it.
- **Kubernetes CSI / local-static-provisioner** — <https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner> — the mechanics behind the `nvme-scratch` StorageClass and `WaitForFirstConsumer` local binding in the practice config.
- **MLCommons — "Announcing the MLPerf Storage v2.0 Checkpointing Workload"** (Aug 2025) — <https://mlcommons.org/2025/08/storage-2-checkpointing/> — read for the industry-standardized checkpoint-burst benchmark; confirms checkpoint sizing is a recognized, measured problem, not an invented scenario.

**Real-world engineering blogs**
- **Meta — "Meta's AI Storage Blueprint at Scale" (2026)** — <https://engineering.fb.com/2026/07/01/data-infrastructure/metas-ai-storage-blueprint-at-scale/> — what it shows: the 56% GPU-stall number and the DPP/tiered-caching fix; the headline citation for this lesson's "why this matters."
- **Meta — "Building Meta's GenAI Infrastructure" (2024)** — <https://engineering.fb.com/2024/03/12/data-center-engineering/building-metas-genai-infrastructure/> — what it shows: the earlier hardware/network/storage build-out that the 2026 piece rebuilt on top of.
- **WhiteFiber — "Storage for AI Workloads: Ceph, VAST, and WEKA"** — <https://www.whitefiber.com/blog/ai-storage-ceph-vast-weka> — what it shows: a neocloud operator's own comparison of the three backends, with GPUDirect RDMA specifics.
- **WEKA — "Why Storage Architecture Is the New Bottleneck for HPC and AI Teams"** (vendor-published) — <https://www.weka.io/article/why-storage-architecture-is-the-new-bottleneck-for-hpc-and-ai-teams> — what it shows: a customer story claiming 40–80hr → 4–6hr training runs and 1.8 TB/s/rack; cite as illustrative of the mechanism, not as an audited benchmark.

**Deeper dives**
- **Spheron — "Parallel File Systems for AI (WEKA/Lustre/BeeGFS) guide"** — <https://www.spheron.network/blog/parallel-file-systems-ai-gpu-cloud-wekaio-lustre-beegfs-guide/> — the best single map of the parallel-FS landscape and the metadata/architecture tradeoffs behind self-check (b).
