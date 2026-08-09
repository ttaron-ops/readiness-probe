---
lesson: "10.7"
title: "Storage for AI: parallel filesystems, tiering, and feeding the GPUs"
module: "10"
concept: "Storage for AI: parallel filesystems, tiering, and feeding the GPUs"
status: not-started
est_time: "5h"
artifacts: []
---

# 10.7 · Storage for AI: parallel filesystems, tiering, and feeding the GPUs

> **Concept.** Size a storage hierarchy — local NVMe scratch, a parallel filesystem for datasets, object storage for weights and checkpoints — to the aggregate GB/s a GPU count demands, and wire it in through CSI so a slow data path never starves six-figure silicon.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Why this matters
An H100 costs on the order of $2–3/GPU-hr to rent or own (2026 snapshot; you'll model it precisely in 10.8). If the data path can't keep the GPU's SMs busy, you are paying that rate to idle. A slow filesystem shows up not as an error but as `GPU-Util` sitting at 40% while `dataloader` threads block on I/O — the single most common way a bare-metal AI cluster silently wastes half its capex. At a neocloud, storage throughput per GPU is a line item customers benchmark before they sign; getting it wrong is a churn event.

This is also where on-prem knowledge is a differentiator. Managed clouds hand you EFS/FSx-Lustre/GCS as opaque endpoints. On bare metal you choose the filesystem, size the metadata tier, lay out NVMe, and own the GPUDirect path. That is the job at CoreWeave/NVIDIA-class shops.

## What's new here
Module 04 (GPU-on-Kubernetes) covered device plugins, MIG, and scheduling GPUs. Networking (09) covered the fabric. Neither covered **where the bytes the GPU eats come from**. New here:

- **The GPU is fed by a hierarchy, not one store.** Four distinct data classes (scratch, datasets, checkpoints, weights) with different access patterns, each belonging on a different tier.
- **Parallel filesystems** (WEKA, Lustre, GPFS/Spectrum Scale, VAST) — how they stripe a single namespace across many storage nodes to hit hundreds of GB/s aggregate, and why **metadata-server design** (WEKA's distributed metadata vs Lustre's classic single-MDS) is the scaling ceiling.
- **Throughput sizing as arithmetic**: turning a GPU count into a target aggregate GB/s and a checkpoint-burst bandwidth.
- **GPUDirect Storage (GDS)** — DMA from NVMe/NIC straight into GPU HBM, bypassing the CPU bounce buffer.
- **CSI integration** — expressing all of this as StorageClasses so pods request the right tier by name.

## Core notes

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

### Parallel filesystems — the aggregate-bandwidth machines
A parallel FS presents **one POSIX namespace** but stripes file data across many storage servers, so `N` clients reading get `N`-way aggregate bandwidth instead of one server's NIC.

| FS | Architecture | Metadata | Notes (2026) |
|---|---|---|---|
| **WEKA** | software-defined, NVMe, own network FS protocol; tiers to S3 | **distributed metadata** across all nodes | scales metadata with capacity; strong small-file/mixed; common at GPU neoclouds |
| **Lustre** | OSS/OST data servers + **MDS/MDT** metadata | classically **single active MDS** (DNE adds more, but it's bolt-on) | huge sequential bandwidth, cheap per TB; **MDS is the classic bottleneck** on metadata-heavy (many-small-file) workloads |
| **GPFS / IBM Spectrum Scale** | shared-disk, distributed metadata | distributed | mature, enterprise HPC; strong but license-heavy |
| **VAST** | disaggregated DASE, all-flash, NFS/RDMA | distributed, shared everything | simple ops, good metadata scaling; popular in newer AI DCs |
| **BeeGFS** | OSS-style, distributable metadata | can add metadata servers | cheap/OSS, common in smaller HPC |

**Why metadata design matters at scale (self-check b).** Training reads millions of small files (image shards, tokenized samples): every `open()`/`stat()`/`readdir()` is a metadata op. On **Lustre with a single MDS**, all of that funnels through one server — the data path can be at 500 GB/s while the job crawls because the MDS is pegged at 100% CPU serving `stat`s. **WEKA/VAST distribute metadata** across all nodes, so metadata IOPS scale with the cluster and the small-file wall moves out. This is why "what filesystem?" is really "what's your file-count and access pattern?": big sequential files (video, sharded tensors) forgive a single MDS; billions of tiny files do not. Mitigations on Lustre: DNE (multiple MDTs), or packing samples into large shards (WebDataset/tar, TFRecord) so the metadata op count collapses.

### GPUDirect Storage (GDS)
Normally a read goes NVMe/NIC → **CPU bounce buffer in system RAM** → GPU HBM, burning CPU cycles and PCIe bandwidth twice. **GPUDirect Storage** lets the storage DMA straight into GPU memory over PCIe/NVLink, bypassing the CPU bounce. Accessed via the **cuFile** API (and integrated in DALI, and in WEKA/VAST/Lustre clients that support it). Single-stream GDS delivers on the order of **~10–12 GB/s** (2026 snapshot); it matters most when the CPU-bounce copy is your ceiling — large sequential reads into HBM, or checkpoint restore. It needs a compatible NIC/NVMe, MOFED, and a client that speaks it. Don't assume it's free: verify the whole path (`gdscheck`) or you'll silently fall back to the bounce buffer.

### CSI integration — expressing tiers as StorageClasses
Everything above becomes **StorageClasses** so pods request a tier by name:
- **Local NVMe scratch** — the [local-static-provisioner](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner) or `local` volumes with `volumeBindingMode: WaitForFirstConsumer` (bind the PVC only once the pod is scheduled, so it lands on a node that has the NVMe). Or plain `emptyDir` on an NVMe-backed kubelet root for pure throwaway.
- **Parallel FS** — vendor CSI driver (WEKA CSI, VAST CSI, Lustre via the [aws-fsx-csi-driver](https://github.com/kubernetes-sigs/aws-fsx-csi-driver) or a generic Lustre CSI, GPFS CSI). Usually `ReadWriteMany`, one big shared PVC mounted by every training pod.
- **Object storage** — not a filesystem mount; accessed via S3 SDK, or surfaced through the [container-object-storage-interface (COSI)](https://github.com/kubernetes-sigs/container-object-storage-interface) / a MinIO gateway. Weights/checkpoints live here.

## Worked example — sizing storage for a 64-GPU H100 cluster (mixed workload)
8 nodes × 8 H100 = 64 GPUs. Assume a **mixed** shop: some multimodal training (byte-hungry) plus LLM fine-tuning of up to a 70B model.

1. **Steady dataset read.** Planning number ~1 GB/s/GPU ⇒ **~64 GB/s aggregate** target from the parallel FS. Multimodal-heavy days push higher; LLM-only days far lower.
2. **Checkpoint burst.** 70B fine-tune, fp32 optimizer state ⇒ ~16 B/param ⇒ **~1.1 TB** per checkpoint. Target a 30 s write ⇒ **~37 GB/s write burst**. Cadence: checkpoint every ~30 min (fleet MTBF on 8 nodes is days, but a 30-min cadence caps lost work at 30 min for ~1 min of amortized write tax). Use **async checkpointing**: ranks write shards to **local NVMe first** (see below), then a background drain copies to the FS/object tier — so the 37 GB/s never stalls the GPUs.
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

**Acceptance:** a 64-GPU storage sizing sheet (aggregate GB/s + checkpoint burst + tiering) **and** a three-tier StorageClass/PVC config, committed into the deliverable folder. It must show the GB/s arithmetic and name the sizing driver.

## Self-check
**(a) Roughly what aggregate GB/s do you need to keep 64 H100s fed, and how do you estimate it?**
**Answer:** Two budgets, take the max. **Steady read** ≈ per-GPU appetite × 64; at the ~1 GB/s/GPU planning rule that's **~64 GB/s** (lower for LLM-only, higher for multimodal decode). **Checkpoint burst** ≈ (params × ~16 B for fp32 optimizer state) ÷ target write time — e.g. a 70B model ⇒ ~1.1 TB ÷ 30 s ≈ **~37 GB/s** of write, often the real driver, hitting all at once every N steps. Provision the hot tier to `max(read, burst)` plus headroom (~80–100 GB/s), and use async checkpointing to keep the burst off the GPU critical path.

**(b) Why does metadata-server design (WEKA distributed vs Lustre MDS) matter at scale?**
**Answer:** Training on millions of small files makes `open`/`stat`/`readdir` — metadata ops, not data — the hot path. A **single active MDS (classic Lustre)** funnels all of them through one server; it pegs at 100% CPU and the job crawls even while the data path is nearly idle. **Distributed metadata (WEKA/VAST/GPFS)** scales metadata IOPS with the cluster, moving the small-file wall out. Mitigations on Lustre: DNE multi-MDT, or pack samples into large shards (WebDataset/tar, TFRecord) to collapse the metadata op count.

**(c) Where do model weights, datasets, checkpoints, and scratch each belong, and why?**
**Answer:** **Scratch** → local NVMe on the GPU node: lowest latency, disposable, no shared-durability need. **Datasets** → parallel FS: one shared namespace read by every node at hundreds of GB/s aggregate. **Checkpoints** → parallel-FS hot tier for the write burst, then tier down to object storage as they cool. **Weights** → object storage: cheap, durable, versioned, write-once/read-many, pulled to NVMe cache at job start. Misplacing any one (datasets on object, checkpoints only on object, scratch on the shared FS) either starves the GPUs or wastes the expensive shared bandwidth.

## Resources
1. **Spheron — Parallel File Systems for AI (WEKA/Lustre/BeeGFS) guide** — https://www.spheron.network/blog/parallel-file-systems-ai-gpu-cloud-wekaio-lustre-beegfs-guide/ — **Deep.** Best single map of the parallel-FS landscape and the metadata/architecture tradeoffs behind self-check (b).
2. **NVIDIA GPUDirect Storage design guide** — https://docs.nvidia.com/gpudirect-storage/design-guide/index.html — **Skim to deep.** The authoritative GDS/cuFile reference for the DMA-into-HBM path; use `gdscheck` to verify the whole chain before trusting it.
3. **Kubernetes CSI / local-static-provisioner** — https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner — **Skim.** The mechanics behind `nvme-scratch` and `WaitForFirstConsumer` local binding in the practice config.
