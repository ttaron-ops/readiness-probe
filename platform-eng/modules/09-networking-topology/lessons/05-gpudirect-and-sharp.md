---
lesson: "09.5"
title: "GPUDirect over the fabric + SHARP"
module: "09"
concept: "GPUDirect and SHARP"
status: not-started
est_time: "7h"
prev: "04-ib-vs-roce-lossless.md"
next: "06-k8s-multi-nic.md"
artifacts: []
sources: 14
---

# 09.5 · GPUDirect over the fabric + SHARP

> **Concept.** GPUDirect RDMA extends the same-root-complex rule across the fabric so a NIC DMAs straight into a peer GPU's HBM, and SHARP moves the all-reduce sum into the switch ASIC.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Where this fits

**04** answered the fabric-technology question — InfiniBand vs RoCEv2, and *why* each is (or must be engineered to be) lossless. That lesson treated the fabric as a pipe and asked "does it drop packets." This lesson asks two different questions about the same pipe. First: **once bytes are moving losslessly, whose memory do they come out of?** GPUDirect RDMA answers that — it takes the CPU and host DRAM out of the inter-node data path entirely, the same way NVLink P2P takes them out of the intra-node path (02b). Second: **does anything have to move at all?** SHARP answers that — for reduction-shaped collectives it deletes most of the movement by doing the arithmetic inside the switch.

Both are levers you pull only after the lossless-fabric question from 04 is settled; you cannot GPUDirect or SHARP your way out of a fabric that drops packets. Lesson 03 named GPUDirect RDMA and told you it "has its own precondition." This lesson is that precondition, in full: the kernel modules, the PCIe topology walk the driver actually performs, the ACS bit that silently breaks it, and the exact log line that proves it engaged. **06** then covers how Kubernetes gets a pod a rail-aligned NIC in the first place — this lesson is the reason that plumbing is worth building.

## Why this matters

The failure mode this lesson exists to prevent does not throw an error. When GPUDirect RDMA does not engage, NCCL stages every byte through a pinned host buffer, the job keeps running, every rank reports success, and throughput is quietly wrong. On an 8-rail 400 Gb/s node the arithmetic in §9 shows it is not a 20% tax — the host DRAM bandwidth required to stage eight rails at line rate exceeds what a dual-socket server has, so the node simply cannot reach its rated inter-node bandwidth at all. You pay for the whole world size to wait on the slowest rank, and the only visible symptom is a step time that does not match the number in the paper.

Across 40+ clusters that failure is not hypothetical: it is one ACS bit set by a BIOS default, one node SKU whose PCIe layout differs from the last one, one `nvidia_peermem` that failed to load after a kernel bump. Being able to prove GPUDirect engaged — from `nvidia-smi topo -m`, one `lsmod`, and one grep of the NCCL init log — is the difference between a fabric you can defend in a procurement review and one that wastes a third of a 512-GPU reservation. Two of the module's checkpoint probes sit here: *"what breaks GPUDirect RDMA if the GPU and NIC are on different root complexes"* and *"which collective shapes benefit most from SHARP"*. Both want mechanism, not a slide.

## What's new here (calibration)

- You already know 02b's rule — GPU↔GPU peer-to-peer needs a shared PCIe root complex — and 08's NCCL collective model. Lesson 01 of this module already promoted that rule to "same rail" and gave you the `topo -m` legend. **None of that is re-taught.**
- **New: the kernel plumbing.** Two mutually-exclusive-in-practice mechanisms let a NIC address GPU memory — the legacy **peer-memory client** (`nvidia_peermem`) and the modern **dma-buf** path (`ibv_reg_dmabuf_mr`). You will see the actual call sequence, the 64 KiB page granularity it imposes, and the three sysfs paths NCCL probes to decide which one is available.
- **New: the exact engage conditions, and the silent fallback when they are not met.** Four distinct causes, four distinct signatures. This is the highest-value content in the lesson — the fallback is silent by design, and knowing which of the four you have is what turns a two-week mystery into a ten-minute check.
- **New: NCCL's own decision procedure**, read out of its source rather than paraphrased — `ncclTopoCheckGdr`, the numeric `PATH_*` ladder, the real default (`PXB`, not "conservative"), and the `NCCL_IB_HCA` grammar including `^` and `=`.
- **New: SHARP as a hardware reduction engine**, with the byte and latency arithmetic derived rather than asserted, the exact ops and datatypes the NCCL plugin can offload (fewer than you think), and the distinction from **NVLink SHARP / NVLS**, which shares a name but not a mechanism and is the one most people have actually measured.

## Core concepts

### 1. The problem: a NIC cannot see GPU memory

Start with why this needs a mechanism at all. An RDMA NIC does not copy bytes the way a CPU does. When you post a send, the NIC's DMA engine issues PCIe memory-read transactions against **physical addresses** it was given at registration time, asynchronously, with no software in the loop. For that to work, three things must be true of the buffer:

1. **It must have a stable physical address.** The NIC will read it microseconds or milliseconds later; the pages cannot move or be swapped out underneath it.
2. **The NIC must be able to address it.** The address must be routable across the PCIe fabric from the NIC's position in the tree.
3. **The NIC's translation tables must contain it.** RDMA NICs hold their own translation and protection tables (MTT/MPT on `mlx5`), programmed at `ibv_reg_mr` time and keyed by `lkey`/`rkey` (lesson 03 §7).

Ordinary host memory satisfies all three trivially: `ibv_reg_mr` pins the pages, walks the page tables, and programs the NIC. GPU memory satisfies **none** of them by default. HBM is not in the kernel's page tables. It has no `struct page`. `get_user_pages()` — the call that underpins normal registration — returns nothing useful for a `cudaMalloc`'d pointer. So the default answer is the one everyone starts with: copy it to somewhere the NIC *can* see.

That is the **bounce path**, and it is what you get when GPUDirect does not engage. Both paths, drawn against the same hardware:

```
  NON-GDR (HOST-BOUNCED) SEND                      GDR SEND
  ═══════════════════════════                      ════════

   ┌────────┐                                       ┌────────┐
   │  GPU   │  HBM3e ~3.35 TB/s                     │  GPU   │
   │  HBM   │                                       │  HBM   │
   └───┬────┘                                       └───┬────┘
       │ ① copy engine, PCIe x16                        │
       ▼                                                │  ① PCIe peer-to-peer
   ┌────────────┐   ② over the ROOT PORT                │     read, stays inside
   │ PCIe switch│──────────────┐                        │     the PCIe switch
   └────────────┘              ▼                        ▼
       ▲                  ┌─────────┐              ┌────────────┐
       │ ③ read back      │  CPU +  │              │ PCIe switch│
       │   over the       │  DRAM   │              └─────┬──────┘
       │   ROOT PORT      │ (pinned │                    │
       │                  │  bounce)│                    │
   ┌───┴────┐             └─────────┘                ┌───┴────┐
   │  NIC   │                                        │  NIC   │
   └───┬────┘                                        └───┬────┘
       │ ④                                               │ ②
       ▼                                                 ▼
     WIRE  400 Gb/s = 50 GB/s                          WIRE  400 Gb/s

  payload crosses the root port  2×          payload crosses the root port  0×
  host DRAM traffic  2 × payload             host DRAM traffic  0
  CPU copy-engine sync per chunk  yes        CPU on the data path  no
  extra latency per 512 KiB chunk ~26 µs     extra latency  0
  (both sides; see §9 for the arithmetic)
```

Note carefully what does *not* change: the payload crosses the GPU's own x16 link once either way, and the NIC's own x16 link once either way. **What doubles is the root-port link and host DRAM** — and on an 8-rail node those are shared by all eight rails, which is why the bounce path collapses at scale rather than merely costing 2×.

### 2. GPUDirect RDMA: making HBM a legitimate DMA target

GPUDirect RDMA (GDR) solves the three requirements from §1 directly instead of dodging them.

**Addressability comes from a PCIe BAR.** Every PCIe device advertises Base Address Registers — windows of its own memory that the host maps into the system physical address space so other devices can address them. NVIDIA GPUs expose a large window (conventionally called BAR1) through which a region of HBM can be made visible to the PCIe fabric. When a NIC issues a read to an address inside that window, the PCIe fabric routes the transaction to the GPU, and the GPU services it out of HBM. That is the entire trick: **HBM gets a physical address that a peer device can put on the bus.**

BAR1 is a finite resource and it is per-GPU. Read the actual size on the hardware in front of you rather than assuming:

```console
$ nvidia-smi -q | grep -A 4 "BAR1 Memory Usage"
    BAR1 Memory Usage
        Total                             : 131072 MiB
        Used                              : 6 MiB
        Free                              : 131066 MiB
```

*(Representative output from an H100-class SXM board. The number varies by product, by whether the platform enables above-4G decoding, and by resizable-BAR support — which is exactly why you read it instead of quoting it.)* If BAR1 is small relative to the buffers you register — some virtualised and consumer platforms cap it at 256 MiB — large registrations fail or fall back, and that is a real, checkable cause of "GDR worked in the lab and not in the VM."

**Pinning and translation come from a kernel shim.** The RDMA subsystem cannot pin GPU pages itself, so NVIDIA supplies a module that teaches it how. There are two generations of that shim, and which one you have decides several things you will care about.

### 3. Path A — the peer-memory client (`nvidia_peermem`)

The original mechanism extends `ib_core` with a **peer memory client** registry. A driver that owns non-system memory registers a `struct peer_memory_client` with callbacks; when `ibv_reg_mr` is handed an address `ib_core` cannot resolve, it offers the range to each registered client in turn, and the client that says "mine" takes over pinning and mapping.

`nvidia_peermem` is that client for GPU memory. Verified against the driver source (`kernel-open/nvidia-peermem/nvidia-peermem.c`, open-gpu-kernel-modules 610.57.04), the sequence is:

```
  REGISTRATION — ibv_reg_mr(pd, gpu_ptr, len, IBV_ACCESS_LOCAL_WRITE|...)
  ══════════════════════════════════════════════════════════════════════
   user space          ib_core             nvidia_peermem            nvidia.ko
   ──────────          ───────             ──────────────            ─────────
   ibv_reg_mr ──────▶  cannot resolve
                       the range in the
                       page tables
                            │
                            │ offer to each registered
                            │ peer_memory_client
                            ▼
                       acquire() ──────▶  round the range
                                          DOWN/UP to 64 KiB
                                          (GPU_PAGE_SHIFT 16)
                                               │
                                               ├─▶ nvidia_p2p_get_pages() ──▶ pin,
                                               │   build nvidia_p2p_page_table
                                               │   of 64 KiB PTEs
                                               └─▶ nvidia_p2p_put_pages()
                                                   "yes, mine" (probe only)
                            │
                            ▼
                       dma_map() ──────▶  nvidia_p2p_dma_map_pages(pdev, pt)
                                          ── THIS is where the GPU driver
                                             decides the NIC may reach it;
                                             see §5 for the topology check
                                               │
                                               ▼
                                          dma_mapping->dma_addresses[]
                                               │
                            ◀──────────────────┘ returned as an sg_table
                            │
                            ▼
                       program mlx5 MTT/MPT with those bus addresses,
                       hand back lkey / rkey
  ══════════════════════════════════════════════════════════════════════
  DATA PATH — from here the NIC DMAs to those bus addresses with no
  software involvement at all, until the buffer is freed.

  TEARDOWN — if CUDA frees the memory while it is still registered,
  nvidia.ko invokes the free_callback, which calls
  mem_invalidate_callback() into ib_core to revoke the MR first.
  That callback is the reason the module exists rather than a simple
  pin: GPU memory can vanish under the NIC, and someone has to tell
  the NIC before it does.
```

Three consequences worth holding:

- **64 KiB granularity is a hard constraint, not a tuning choice.** `GPU_PAGE_SHIFT` is 16 in the driver; `nvidia_p2p_get_pages` documents that the start address must be 64 KiB-aligned and the length a multiple of 64 KiB, and `nv_dma_map` refuses outright with `-EINVAL` and the log line `nv_dma_map -- assumption of 64KB pages failed` if the returned page table is any other size. Registration rounds outward to that grain, so a small registration always pins at least 64 KiB.
- **The invalidation callback is the interesting engineering.** Host pinning is a promise the memory will not move. GPU memory can be freed by the application at any time, so the peer-memory contract instead says: *if it goes away, I will tell you first*. `nvidia_peermem` sets `PEER_MEM_INVALIDATE_UNMAPS` so `ib_core` knows it does not need to call unmap/put during that callback.
- **The module registers under the name `nv_mem`.** Since the persistent-API default, the "nc" (non-callback, persistent-pages) client also registers as `nv_mem` rather than `nv_mem_nc`, so libraries find one name. This is why the sysfs paths NCCL probes look historical.

**How to check it is loaded**, which is exactly what NCCL does:

```console
$ lsmod | grep -E 'nvidia_peermem|nv_peer_mem'
nvidia_peermem         16384  0
nvidia              12345678  N nvidia_peermem,nvidia_uvm

$ ls /sys/module/nvidia_peermem/version
/sys/module/nvidia_peermem/version
```

NCCL's `ncclIbGdrSupport()` (`src/transport/net_ib/gdr.cc`) returns success if **any** of these exist:

| Path probed | Meaning |
|---|---|
| `/sys/module/nvidia_peermem/version` | modern in-driver `nvidia_peermem` (driver ≥ 465) |
| `/sys/kernel/mm/memory_peers/nv_mem/version` | legacy out-of-tree `nv_peer_mem` |
| `/sys/kernel/mm/memory_peers/nv_mem_nc/version` | legacy non-callback variant |

A separate probe, `ncclIbPeerMemSupport()`, checks **only** `/sys/module/nvidia_peermem/version` — some newer transports (GDAKI) do not accept the legacy client. So "peermem is loaded" and "modern peermem is loaded" are two different questions, and NCCL asks both.

### 4. Path B — dma-buf, the modern route

`nvidia_peermem` requires an `ib_core` patched with the peer-memory symbols, which historically meant MLNX_OFED / DOCA-OFED rather than an inbox kernel. The kernel community's answer is **dma-buf**: a generic in-tree framework for one driver to export a buffer and another to import and DMA to it. The GPU driver exports a `dma_buf` file descriptor for a memory range; the RDMA driver imports it.

The user-space call is a distinct verb, not an overload of `ibv_reg_mr` (`rdma-core`, `libibverbs/verbs.h`):

```c
struct ibv_mr *ibv_reg_dmabuf_mr(struct ibv_pd *pd,
                                 uint64_t offset,   /* offset within the dma-buf   */
                                 size_t   length,
                                 uint64_t iova,     /* virtual base for lkey/rkey; */
                                                    /* must share offset's page    */
                                                    /* offset                      */
                                 int      fd,       /* the dma-buf file descriptor */
                                 int      access);
```

The man page restricts `access` to `IBV_ACCESS_LOCAL_WRITE`, `REMOTE_WRITE`, `REMOTE_READ`, `REMOTE_ATOMIC` and `RELAXED_ORDERING` — the flags that make sense for memory with no `struct page`.

**How NCCL probes for it** is worth copying, because it is the cheapest possible feature test and you can reproduce it: it allocates a protection domain and calls `ibv_reg_dmabuf_mr` with a deliberately invalid `fd` of `-1` and zero length. A kernel with dma-buf MR support fails with `EBADF` (bad descriptor — the call reached the real implementation); a kernel without it fails with `EOPNOTSUPP` or `EPROTONOSUPPORT` (the call was never implemented). NCCL treats the latter two as "no dma-buf" (`src/transport/net_ib/gdr.cc`).

The GPU side has its own gate. NCCL requires **CUDA ≥ 11.7 in both the toolkit and the driver**, and queries `cuDeviceGetAttribute(CU_DEVICE_ATTRIBUTE_DMA_BUF_SUPPORTED)` per device; `NCCL_DMABUF_ENABLE` defaults to `1` and turns the whole path off when set to `0` (`src/init.cc`). Success prints:

```
NCCL INFO DMA-BUF is available on GPU device 0
```

**Which path do you actually get?** NCCL sets `ptrSupport` on the device from both probes independently (`src/transport/net_ib/init.cc`): `NCCL_PTR_CUDA` if peermem is present, `NCCL_PTR_DMABUF` if dma-buf is present, and it can hold both. When both are available it prefers dma-buf for the registration (`resources->useDmaBuf = useGdr && dmaBufSupport && (ptrSupport & NCCL_PTR_DMABUF)`). **Either one is sufficient** — which is why "we removed `nvidia_peermem` and GDR still works" is a real and confusing observation on a modern stack, and why checking `lsmod` alone is not a complete diagnosis.

| | peer-memory client (`nvidia_peermem`) | dma-buf |
|---|---|---|
| Where it lives | NVIDIA module + peer-mem-patched `ib_core` | in-tree kernel framework |
| User-space call | ordinary `ibv_reg_mr` on a GPU pointer | `ibv_reg_dmabuf_mr` with an fd |
| Needs OFED/DOCA-OFED | yes, in practice | no |
| Grain | 64 KiB (`GPU_PAGE_SHIFT` 16) | dma-buf sg entries |
| Revocation | driver `free_callback` → `ib_core` invalidate | dma-buf move-notify / importer ops |
| Minimum CUDA | any | 11.7 toolkit **and** driver |
| NCCL probe | three sysfs paths | `ibv_reg_dmabuf_mr(fd=-1)` errno |
| Who loads it | **GPU Operator**, not Network Operator | nothing to load |

That last row catches people out on Kubernetes. The NVIDIA Network Operator's own README states it plainly: from driver v465 the GPU driver ships `nvidia_peermem` built in, and **the GPU Operator manages loading it**. So a cluster whose Network Operator is perfectly healthy can still have no peermem, because that is the other operator's job. Lesson 06 revisits this split.

### 5. The topology condition, as the driver actually implements it

Lessons 02b and 01 gave you the rule — same root complex, expressed at fabric scale as "same rail." Here is what the driver literally does, because the shape of the check explains the shape of the failures. From `kernel-open/nvidia/nv-pci.c` (610.57.04):

```
  nv_grdma_pci_topology_supported(gpu, peer_nic)
  ══════════════════════════════════════════════
        │
        ├─ platform is CACHE-COHERENT (Grace-Hopper / Grace-Blackwell, C2C)?
        │      └──▶ ALLOW.  The NIC reaches GPU memory over C2C; PCIe
        │                   topology is irrelevant on these boxes.
        │
        ├─ GPU is in a PASSTHROUGH VM?
        │      └──▶ ALLOW by default. The hypervisor owns the real topology
        │           and the driver cannot see it.  (This is also why ACS
        │           problems show up here and cannot be fixed from inside.)
        │
        ├─ chipset flagged NV_FLAG_PCI_P2P_UNSUPPORTED_CHIPSET?
        │   (set at RmInitAdapter when the platform is not known to be
        │    both PCIe-peer-read and peer-write capable)
        │      └──▶ ALLOW ONLY IF nv_pci_has_common_pci_switch(gpu, nic)
        │                                   │
        │                                   ▼
        │            walk pci_upstream_bridge() from the GPU;
        │            for each ancestor, walk pci_upstream_bridge()
        │            from the NIC; return true on the first common
        │            ancestor bridge.  No common ancestor → refuse.
        │
        └─ otherwise ──▶ ALLOW.
```

Three things fall out of reading it rather than reciting the rule:

- **"Same root complex" is really "has a common upstream PCIe bridge."** The check is a literal two-nested-loop ancestor walk. On an HGX board where GPU*k* and NIC*k* hang off the same PCIe switch, the common ancestor is that switch and the answer is yes at depth 1 — which is precisely why `topo -m` reports `PXB` for that pair and why `PXB` is the level you care about.
- **Cross-socket is not a special case in the code.** GPU on socket 0 and NIC on socket 1 simply have no common PCIe bridge — their ancestor chains terminate in two different root complexes joined by UPI, which is not a PCIe bridge. The walk returns false. `SYS` in the matrix is the user-visible name for "the ancestor walk will fail."
- **The check is gated on the chipset flag**, so on a server platform the driver has validated it will not run the walk at all. This is why the same misconfiguration behaves differently on a workstation and on an HGX node, and why "it worked on my dev box" transfers badly.

The dma-buf path applies the same check at `attach` time (`nv-dmabuf.c`), logging `NVRM: dma-buf attach failed: PCI topology not supported for dma-buf` and returning `-ENOTSUPP`. That is one of the few places this failure is *loud*, and it is worth knowing the exact string.

### 6. ACS — the bit that silently converts P2P into a bounce

Everything in §5 can pass and GPUDirect can still be broken, by a PCIe feature you did not configure.

**Access Control Services** is a PCIe capability on switch downstream ports, intended for IO virtualisation. With ACS *source validation* and *P2P request redirect* enabled, a switch port that receives a peer-to-peer transaction does not forward it laterally to a sibling port. It **redirects it upstream to the root complex**, so the IOMMU can validate that device A is permitted to touch device B's memory. That is the correct behaviour for a multi-tenant virtualised host. It is also fatal for GPUDirect: the transaction that was supposed to stay inside the PCIe switch now makes a round trip to the CPU, and — depending on IOMMU configuration — is either enormously slower or rejected outright.

The rejection surfaces as an RDMA completion error, and NCCL specifically pattern-matches it (`src/transport/net_ib/common.h`):

```
NCCL INFO HINT: In many cases this error indicates that ACS is enabled,
               which breaks the GPU Direct RDMA protocol.
NCCL INFO HINT: To confirm, set NCCL_NET_GDR_LEVEL=0; if that resolves it,
               disable ACS following your vendor documentation.
```

That hint fires on completion status `IBV_WC_LOC_PROT_ERR` — local protection error. **If you see `IBV_WC_LOC_PROT_ERR` on a GPU buffer, suspect ACS before you suspect your code.** (For reference, NCCL prints two sibling hints: `IBV_WC_WR_FLUSH_ERR` suggests NICs on the same node cannot reach each other, testable with `ib_write_bw`; `IBV_WC_RETRY_EXC_ERR` suggests `NCCL_IB_TIMEOUT` is too low — its default of 20 is roughly 30 seconds before giving up.)

Checking and clearing ACS on the bridges between a GPU and its NIC:

```console
# Which bridges sit above the devices? (0000:xx:00.0 = GPU, mlx5 = NIC)
$ lspci -tv | head -20

# Read the ACS control register on a bridge:
$ sudo lspci -vvv -s 0000:16:00.0 | grep -i ACSCtl
        ACSCtl: SrcValid+ TransBlk- ReqRedir+ CmpltRedir+ UpstreamFwd+ EgressCtrl- DirectTrans-
#               ^^^^^^^^^^          ^^^^^^^^^^^^^^^^^^^^
#               a '+' on SrcValid / ReqRedir / CmpltRedir means P2P is
#               being forced up to the root complex.  You want '-'.

# Clear bits 0x2f (SrcValid, TransBlk, ReqRedir, CmpltRedir, UpstreamFwd)
# on one bridge.  Offset is capability-dependent; setpci resolves it:
$ sudo setpci -s 0000:16:00.0 ECAP_ACS+0x6.w=0000
```

Three operational notes. First, this does not survive a reboot — put it in a systemd unit or, better, disable ACS in BIOS. Second, **inside a virtual machine you cannot do this at all**: ACS is required for IO virtualisation, and the fix is to enable **ATS** (Address Translation Services) on the network adapter instead, so the NIC caches translations and the redirect becomes cheap. Third, disabling ACS weakens an isolation boundary; on a multi-tenant node that is a security decision, not just a performance one. Say so when you propose it.

### 7. NCCL's decision procedure, from the source

NCCL does not just ask "is GDR possible." It asks "is the GPU close enough to this NIC that GDR is worth using," and the answer is a comparison against a numeric ladder. From `src/include/graph.h` and `src/graph/paths.cc` (NCCL v2.31.2-1):

| Value | Name | Meaning |
|---:|---|---|
| 0 | `LOC` | same device |
| 1 | `NVL` | traversing NVLink |
| 2 | `NVB` | NVLink via an intermediate GPU |
| 3 | `C2C` | over a coherent CPU–GPU link (Grace) |
| 4 | `PIX` | at most a single PCIe bridge |
| 5 | `PXB` | multiple PCIe bridges, **no host bridge crossed** |
| 6 | `P2C` | GPU→CPU over C2C, then CPU→NIC over PCIe |
| 7 | `PXN` | reached via an intermediate GPU's NIC (rail-local aggregation) |
| 8 | `PHB` | through the PCIe host bridge (the CPU) |
| 9 | `SYS` | across the SMP interconnect between sockets (UPI/xGMI) |
| 10 | `NET` | through the network |
| 11 | `DIS` | disconnected |

`ncclTopoCheckGdr()` then runs, in order:

1. **Capability gate.** If the NIC did not advertise `NCCL_PTR_CUDA`/`NCCL_PTR_DMABUF` (i.e. neither peermem nor dma-buf is present), or the GPU reports no GDR support, return "disabled" immediately. *No warning is printed.*
2. **Read gate (sends only).** `NCCL_NET_GDR_READ` defaults to `-2` (auto). Under auto, on GPUs with compute capability **< 8.0** (pre-Ampere) and no NVLink to another local GPU, **GDR reads are disabled** while GDR writes stay on. The historical reason is that reading GPU memory over PCIe is much slower than writing it on those parts, so staging the send was genuinely faster. On Ampere and later this gate does not fire.
3. **Distance gate.** `netGdrLevel` defaults to **`PATH_PXB` (5)** — or `PATH_P2C` (6) on coherent platforms, since `NCCL_NET_GDR_C2C` defaults to 1. If `NCCL_NET_GDR_LEVEL` is set it replaces that default entirely. The GPU→NIC distance from the topology graph is compared: `distance > netGdrLevel` → **GDR off, silently**. If the path type is `PXN`, the distance of the *intermediate* GPU is used instead.
4. **Mode.** On a coherent platform where the path still goes through PCIe, the mode is forced to `ncclTopoGdrModePci`; otherwise `ncclTopoGdrModeDefault`.

> **Correct the common claim here.** The default GDR level is `PXB`, which is exactly the GPU↔paired-NIC distance on a standard HGX board. NCCL's default therefore **enables** GDR on the rail-aligned pair — it is not "conservative and excludes the path you want." What the default does exclude is `PHB` and `SYS`, i.e. anything that crosses the host bridge or the socket interconnect, which is the right call. Set `NCCL_NET_GDR_LEVEL` explicitly when you need to *widen* the rule for a degraded node, or to pin the behaviour so a future NCCL version cannot change it under you — not because the default is wrong.

`NCCL_NET_GDR_LEVEL` accepts either a level name from the table above (up to `SYS`) or a legacy integer, mapped `0→LOC, 1→PIX, 2→PXB, 3→PHB, 4→SYS, 5→SYS`. Setting it to `0` (`LOC`) disables GDR entirely — which is the A/B test the ACS hint asks you to run.

**`NCCL_IB_HCA` grammar**, from `src/transport/net_ib/init.cc` and `src/misc/utils.cc`, because the details matter and are rarely documented together:

```
  NCCL_IB_HCA=[^][=]<entry>[,<entry>...]
                     entry := prefix[:port[:rail[:plane]]]

  mlx5_3          prefix match  → matches mlx5_3, mlx5_30, mlx5_3x
  =mlx5_3         exact match   → matches only mlx5_3
  mlx5_3:1        device mlx5_3, port 1 (empty or absent port = any)
  mlx5_0,mlx5_1   a list
  ^mlx5_0         EXCLUDE everything matching; the ^ inverts the whole list
```

The `=` prefix is the one that bites: without it, `NCCL_IB_HCA=mlx5_1` on a node with `mlx5_1` and `mlx5_10` selects both. NCCL logs its own reading of the variable, so you never have to guess:

```
NCCL INFO NCCL_IB_HCA set to =mlx5_3:1
```

### 8. The silent fallback — four causes, four signatures

This is the section to remember. When any gate in §7 fails, NCCL sets `useGdr = 0` and continues. There is no warning, no error, no metric. The connection is established, the collective completes, the numbers are wrong. The **only** first-class evidence is the connection log line, whose format string is (`src/transport/net.cc`):

```
Channel %02d/%d : %d[%d] -> %d[%d] [send] via NET/%s/%d%s%s%s
                                                      ▲  ▲  ▲
                                        "/GDRDMA" ────┘  │  └── "/Shared"
                                        "(PCI)"  ────────┘
```

So on a healthy node you expect, per channel, per rank:

```
NCCL INFO Channel 00/0 : 3[3] -> 11[3] [send] via NET/IB/3/GDRDMA
NCCL INFO Channel 00/0 : 11[3] -> 3[3] [receive] via NET/IB/3/GDRDMA/flush=None
```

and on a broken one, the same lines with the `/GDRDMA` token simply **absent**:

```
NCCL INFO Channel 00/0 : 3[3] -> 11[3] [send] via NET/IB/3
```

The one-line check, which belongs in every job's smoke test:

```bash
NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=INIT,NET NCCL_DEBUG_FILE=/tmp/nccl.%h.%p.log \
  ./all_reduce_perf -b 512M -e 512M -g 8
grep -c 'GDRDMA' /tmp/nccl.*.log     # 0 means every rank fell back
```

Now the four causes, and how to tell them apart:

| # | Cause | What proves it | Fix |
|---|---|---|---|
| 1 | **No kernel shim.** Neither peermem nor dma-buf available. Gate 1 in §7. | `lsmod \| grep nvidia_peermem` empty **and** no `DMA-BUF is available on GPU device` line in the log | load `nvidia_peermem` (GPU Operator's job), or get CUDA ≥ 11.7 + a dma-buf-capable `mlx5` |
| 2 | **Too far.** GPU↔NIC distance exceeds the GDR level. Gate 3. | `nvidia-smi topo -m` shows `PHB`/`SYS`/`NODE` for that GPU–NIC pair; the pairing is wrong or the pod got the wrong VF | pin `NCCL_IB_HCA` to the rail-aligned device; if genuinely cross-socket, fix placement (lesson 06), don't widen the level |
| 3 | **ACS.** All gates pass, the MR registers, the transfer errors. | `IBV_WC_LOC_PROT_ERR` plus NCCL's ACS hint; `lspci -vvv \| grep ACSCtl` shows `SrcValid+ ReqRedir+` | clear ACS on the bridges (§6), or enable ATS inside a VM |
| 4 | **Pre-Ampere send-side gate.** GDR on receive, not on send. Gate 2. | asymmetric log: `[receive] ... /GDRDMA` present, `[send] ... ` without it, on a GPU with compute capability < 8.0 | usually correct behaviour; override with `NCCL_NET_GDR_READ=1` only after benchmarking |

Cause 2 has a nasty sibling worth calling out separately, because it is the one Kubernetes creates: **the topology is fine but the process is pointed at the wrong NIC.** K8s hands the pod VF `mlx5_5` while the container's env pins `NCCL_IB_HCA=mlx5_3`; or the pod gets a GPU on NUMA 1 and a VF on NUMA 0. Both produce cause-2's signature on a node whose `topo -m` looks perfect. That is lesson 06's entire subject.

**A note on GDRCopy, because it is constantly confused with GDR.** `NCCL_GDRCOPY_ENABLE` defaults to **0** and does something different: GDRCopy maps a small piece of GPU memory into the *CPU's* address space, write-combined, so the CPU proxy thread can poke a FIFO head/tail word directly instead of via a CUDA call. In NCCL it is used for a single `sizeof(uint64_t)` sync word, and only when the proxy is in the same process and on the same device (`src/transport/net.cc`). It is a **control-path latency** optimisation with no bearing on whether payload bytes bypass host memory. `NCCL_GDRCOPY_SYNC_ENABLE` (default 1) and `NCCL_GDRCOPY_FLUSH_ENABLE` (default 0) only have effect once the library is initialised. Turning `NCCL_GDRCOPY_ENABLE=1` on will not fix a missing `/GDRDMA` token, and benchmark it before adopting it — it is off by default upstream for a reason.

### 9. Worked math: what GDR is actually worth

Two calculations. The second one is the one that ends arguments.

**(a) Added latency per chunk.** NCCL's `SIMPLE` protocol runs a pipeline of `NCCL_STEPS = 8` chunks through a per-connection buffer of `NCCL_BUFFSIZE`, which defaults to 4 MiB — so the unit of transfer is **512 KiB** (`src/init.cc`, `src/include/device.h`). Assume a root-complex path that sustains 40 GB/s (conservative for PCIe Gen5 ×16 carrying other traffic; substitute your measured number):

```
  chunk                       = 512 KiB = 524,288 B
  extra root-port crossings   = 1 on send (HBM→DRAM), 1 on recv (DRAM→HBM)
                                + the NIC's read/write of DRAM on each side

  time per extra crossing     = 524,288 B ÷ 40e9 B/s   = 13.1 µs
  added latency, both sides   = 2 × 13.1 µs            = 26.2 µs

  for comparison, on the same chunk:
  wire time at 400 Gb/s       = 524,288 × 8 ÷ 400e9    = 10.5 µs
  fabric latency (2 IB hops
    @ ~130 ns + endpoint)     ≈ 2 µs

  GDR chunk latency           ≈ 10.5 + 2               = 12.5 µs
  bounced chunk latency       ≈ 12.5 + 26.2            = 38.7 µs   (3.1×)
```

Pipelining hides much of that under steady state, which is why the bounce path does not look 3× worse on a bandwidth benchmark. It does not hide it on the *first* chunk of every collective, which is what latency-sensitive small all-reduces are made of.

**(b) The bandwidth argument, which pipelining cannot hide.** A 400 Gb/s NIC at line rate moves 50 GB/s. Each direction of the bounce path makes host DRAM carry that payload **twice** — once written by the GPU's copy engine, once read by the NIC:

```
  per rail, one direction   : 50 GB/s payload → 100 GB/s of DRAM bandwidth
  8-rail HGX node, one dir  : 8 × 100 GB/s    = 800 GB/s of DRAM bandwidth

  a dual-socket server with 8 channels of DDR5-4800 per socket has
      4800 MT/s × 8 B × 8 ch = 307 GB/s per socket
                             = 614 GB/s for the whole node — and that is the
                               theoretical peak, before the training process
                               uses any of it for anything else.

  800 GB/s required  >  614 GB/s available   ⇒ the node CANNOT reach its
  rated 3.2 Tb/s of inter-node bandwidth on the bounce path, at any
  pipeline depth, no matter how the software is tuned.
```

**That is the sentence to carry into a design review.** GDR is not a 20% optimisation on an 8-rail node; it is the difference between the node's rated bandwidth being achievable and being arithmetically impossible. Re-run it with your own DRAM configuration and NIC count — the conclusion survives a wide range of inputs, which is what makes it a good argument.

### 10. GPUDirect Storage, in one paragraph

The same idea applied to the storage side. With the **cuFile** API and the `nvidia-fs` kernel module, a read from local NVMe or a GDS-aware network filesystem DMAs directly into GPU memory instead of landing in a page-cache buffer and then being `cudaMemcpy`'d. It matters on exactly two paths in a training cluster: reloading a multi-hundred-gigabyte optimizer state after a failure, and streaming dataset shards fast enough to keep the GPUs fed. The same ACS constraint applies for the same reason — a P2P transaction from an NVMe device to GPU memory is redirected upstream by ACS just as a NIC's is. **Provenance:** the NVIDIA GPUDirect Storage documentation was unreachable from this environment, so this paragraph is deliberately limited to what follows from the same PCIe mechanism described above plus the module and API names; verify cuFile specifics against the vendor docs before designing against them.

### 11. SHARP: moving the sum into the switch

**The problem, stated as arithmetic.** Take a ring all-reduce of `S` bytes over `N` ranks. The ring runs a reduce-scatter followed by an all-gather: `N−1` steps of `S/N` bytes each, twice. So every rank sends — and receives —

```
  bytes_per_rank = 2 · (N−1)/N · S
```

which is the `2(N−1)/N` factor `nccl-tests` multiplies algorithmic bandwidth by to get bus bandwidth (`src/all_reduce.cu`: `factor = 2*(nranks-1)/nranks`). For `N = 64` that is `1.969 · S`. **Each rank puts roughly twice its own buffer on the wire, and the fabric carries roughly `2S` per participant, forever, no matter how big the cluster gets.**

The latency term is worse. Ring all-reduce is `2(N−1)` *sequential* dependent steps. At even 2 µs per step, `N = 512` gives 2.0 ms of pure serialisation before a single byte of payload time is counted. This is why NCCL switches to a Tree algorithm for small messages at large scale: a binary tree is `2·log₂(N)` steps, 18 instead of 1,022 at `N = 512`. But every interior node of that tree is **a GPU**, which must receive, run a reduction kernel, and re-send — so each of those 18 hops costs a full NIC round trip plus kernel launch.

**SHARP replaces the interior nodes with switch silicon.** The InfiniBand switch ASIC contains arithmetic units. A **SHARP aggregation tree** is provisioned over the fabric by an aggregation manager; each switch acting as an interior node of that tree is an **aggregation node**. Endpoints send their contribution once, up the tree. Each aggregation node waits for its children, performs the reduction in hardware at line rate, and forwards a single reduced buffer upward. The root produces the final result and it is multicast back down.

```
  RING ALL-REDUCE, N=64 ranks            SHARP, N=64 ranks on one leaf tier
  ═══════════════════════════            ══════════════════════════════════

   r0 ──S/64──▶ r1 ──S/64──▶ r2 ──▶ …          ┌──────────────────────┐
    ▲                                │         │  switch ASIC          │
    └────────────── … ───────────────┘         │  ┌────────────────┐   │
                                               │  │ aggregation    │   │
   2·(N−1) = 126 sequential steps              │  │ node: ALU tree │   │
   each rank sends 2·63/64·S = 1.969 S         │  │ SUM/MAX/MIN    │   │
   each rank receives             1.969 S      │  └───▲────────┬───┘   │
   fabric carries per endpoint    3.94  S      └──────┼────────┼───────┘
                                            ▲ S each  │        │ S each ▼
                                        ┌───┴───┬─────┴──┬ … ──┴────┐
                                        │ GPU0  │ GPU1   │   GPU63  │
                                        └───────┴────────┴──────────┘
                                        2 hops.  each rank sends S,
                                        receives S.  fabric carries
                                        per endpoint  2 S.

   TRAFFIC RATIO  ring : SHARP  =  2·(N−1)/N  →  1.969× at N=64
                                                 1.984× at N=128
                                                 1.75×  at N=8
                                                 1.00×  at N=2   ← no gain

   LATENCY RATIO  126 sequential endpoint steps
                  vs 2 switch hops at ~100–130 ns each
                  plus one endpoint round trip.
```

Two things this diagram is telling you that the prose usually does not:

- **The byte saving asymptotes at exactly 2×, and it is 1× at N=2.** That is not a marketing rounding of "up to 2×" — it is `2(N−1)/N`, and it explains why the plugin's `NCCL_SHARP_GROUP_SIZE_THRESH` defaults to **2** (below that there is provably nothing to gain) and why NCCL's `NCCL_COLLNET_NODE_THRESHOLD` defaults to **2 nodes** before CollNet is considered at all.
- **The latency saving is unbounded in `N`, and it is the bigger win.** Replacing `2(N−1)` endpoint steps — or even `2·log₂(N)` GPU-mediated tree hops — with two switch-ASIC hops is what makes a small gradient sync at 4,096 ranks tolerable.

**How NCCL reaches it.** SHARP is not built into NCCL. It arrives through the **CollNet** transport, implemented by an external plugin (`nccl-rdma-sharp-plugins`, which builds `libnccl-net.so` against the SHARP library from HPC-X/DOCA). The moving parts:

```
  training process
      │  ncclAllReduce()
      ▼
  NCCL core ── picks algorithm: RING | TREE | COLLNET_DIRECT | COLLNET_CHAIN | NVLS
      │        (requires config.collnetEnable == 1  AND  nNodes >= 2)
      ▼
  CollNet plugin (libnccl-net-sharp)
      │  sharp_coll_init / sharp_coll_comm_init / sharp_coll_do_allreduce
      ▼
  SHARP library ──▶ aggregation manager on the fabric ──▶ switch ASICs
```

The gates you must satisfy, each verifiable:

| Gate | Where it lives | Default / value |
|---|---|---|
| `NCCL_COLLNET_ENABLE=1` | NCCL core, re-read at every `commInit` | unset ⇒ off |
| a CollNet plugin is loadable | `libnccl-net.so` on the library path | none shipped with NCCL |
| ≥ 2 nodes in the communicator | `NCCL_COLLNET_NODE_THRESHOLD` | **2** |
| ≥ `NCCL_SHARP_GROUP_SIZE_THRESH` ranks | plugin | **2** |
| not disabled per-communicator | `NCCL_SHARP_DISABLE` | unset |
| the fabric has SHARP trees provisioned | aggregation manager + subnet manager | site configuration |

If any of these fails, **NCCL falls back to Ring/Tree without erroring** — the same silent-degradation pattern as GDR. The plugin at least logs when it declines:

```
NCCL INFO SHARP: Group size:1 is less than threshold:2. fallback to non-sharp
NCCL INFO SHARP rank 0/64 initialized on mlx5_0            ← this is success
NCCL INFO Connected CollNet
```

**What SHARP can and cannot offload.** This is narrower than most descriptions admit, and it is checkable in the plugin source (`src/sharp_plugin.c`):

| NCCL reduction op | Offloaded? |
|---|---|
| `ncclSum` | yes → `SHARP_OP_SUM` |
| `ncclMax` | yes → `SHARP_OP_MAX` |
| `ncclMin` | yes → `SHARP_OP_MIN` |
| `ncclProd`, `ncclAvg`, custom `ncclRedOp_t` | **no** — `SHARP_OP_NULL`, falls back |

| NCCL datatype | Offloaded? |
|---|---|
| `ncclFloat16`, `ncclFloat32`, `ncclFloat64` | yes |
| `ncclInt32`, `ncclUint32`, `ncclInt64`, `ncclUint64` | yes |
| `ncclBfloat16`, `ncclInt8`, `ncclUint8` | **only if the fabric reports v3 datatype support** — the plugin calls `sharp_coll_caps_query()` and checks `support_mask.dtypes`; if absent it warns `SHARP int8,uint8,bfloat16 Datatypes not supported` and those types fall back |
| `ncclFloat8*`, others | no |

**That bfloat16 row is the one that matters in 2026.** Mixed-precision training reduces gradients in bf16. On a fabric whose SHARP generation does not advertise the v3 datatypes, your bf16 all-reduce gets *nothing* from a SHARP-provisioned fabric while an fp32 all-reduce on the same fabric gets the full benefit — and the only evidence is one `WARN` line at init. `NCCL_SHARP_V3_DATATYPES` (default 2) controls whether the plugin even attempts the query.

Two more real limits from the same source: on **SHARP 3.6 and older** the plugin caps `maxCollBytes` at **512 MiB** per collective, so larger buffers are chunked; and the plugin sets `SHARP_COLL_ENABLE_SAT=1` by default, enabling **Streaming Aggregation** — the mode that pipelines large payloads through the tree rather than requiring the whole buffer to land before reducing. `NCCL_SHARP_DISABLE_REDUCE_SCATTER` and `NCCL_SHARP_DISABLE_ALLGATHER` (both default 0) let you switch off the reduce-scatter and all-gather offloads independently, which is the first thing to try when SHARP is implicated in a correctness or performance regression.

**Which shapes win.** Reduction-shaped collectives only: all-reduce, reduce, broadcast, barrier, and — with newer plugin/SHARP versions — reduce-scatter and all-gather. Traffic that is fundamentally point-to-point never enters an aggregation tree and gets nothing: **all-to-all in expert-parallel MoE, and pipeline-parallel sends.** As MoE-shaped architectures take a larger share of production training, a cluster procured on SHARP's gains for a dense model can find those gains simply absent for the workload it ends up running. That is a procurement argument, not a footnote, and lesson 07 prices it.

### 12. NVLink SHARP (NVLS) is a different thing with the same name

Almost every measured "SHARP speedup" you will find online is **NVLink SHARP**, not InfiniBand SHARP. NVLS performs the reduction in the **NVSwitch** inside a node (or an NVL domain), using multicast groups; NCCL exposes it as the `NVLS` and `NVLSTree` algorithms, gated by `NCCL_NVLS_ENABLE`. It shares the acronym and the idea. It does not touch the scale-out fabric.

The distinction is worth holding precisely because the NVLS numbers are the ones that are easy to reproduce. Measured on an 8×H200 node with NCCL 2.27.7 at a 16 GiB payload (stas00 *ml-engineering*, network chapter, dated 2026-08):

| collective | algorithm | busbw | vs 450 GB/s unidirectional spec |
|---|---|---:|---:|
| all-reduce, `NCCL_NVLS_ENABLE=1` | NVLS | 480.0 GB/s | 107% |
| all-reduce, `NCCL_NVLS_ENABLE=0` | Ring | 367.2 GB/s | 82% |
| all-gather | Ring | 361.4 GB/s | 80% |
| reduce-scatter | Ring | 362.9 GB/s | 81% |

A 1.31× gain, and a reported bandwidth **above the physical wire rate** — which is the tell. `busbw` is computed by multiplying by `2(N−1)/N`, a factor that assumes the ring's traffic pattern. Once the reduction happens in the switch, that factor is no longer describing the wire, so the number stops being a wire measurement and becomes a throughput-equivalent. **When `busbw` exceeds the link spec, you are looking at in-network reduction, not a faster link.**

The same source records that the GPU-count threshold for NCCL to select NVLS is not portable: NVLS was chosen above 4 GPUs on that H200 node, but only above 5 GPUs on an 8×B200 `p6-b200.48xlarge` (NCCL 2.27.7, measured 2026-08-09), where the full-node gain was 1.23×. Measure on your own SKU rather than carrying a threshold across generations.

Tie back to **08**: neither SHARP changes *that* you call `all_reduce`. They change which algorithm NCCL selects underneath and where the arithmetic executes. Your bucket sizing and compute/communication overlap still decide whether the collective is on the critical path at all.

## Perspectives

**Developer.** From inside a training script both mechanisms are invisible on the happy path — GDR is "NCCL got faster" and SHARP is an algorithm NCCL may or may not pick. What changes is your debugging instinct. "Did GDR actually engage" becomes a first question with a definite answer (`grep GDRDMA`), not a guess, and "is `busbw` above the wire rate" becomes a diagnostic rather than a puzzle. The one thing a developer genuinely controls is the reduction op and dtype: a custom `ncclRedOp_t` or an `ncclAvg` silently forfeits SHARP offload, so if you need SHARP, sum and divide rather than averaging.

**Operator.** GDR-level tuning is **per-SKU, not per-fleet**. A value correct for an H100 HGX board (paired ConnectX-7 at `PXB`) can be silently wrong on the next node generation with a different PCIe layout, a Grace-coherent platform where `P2C` is the relevant level, or a virtualised node where ACS cannot be cleared. Nothing pages you when it is wrong; throughput just drops. The durable practice is a per-SKU validation gate: `topo -m` read once, `lsmod`/dma-buf probe once, a two-node `all_reduce_perf` with `grep -c GDRDMA`, all recorded against the SKU, re-run whenever the node image or driver moves.

**Hardware.** SHARP is silicon, not a protocol trick. There are arithmetic units inside the InfiniBand switch ASIC and they consume die area and power. That is exactly why it is InfiniBand-only: RoCE and commodity Ethernet switches — including NVIDIA's own Spectrum-X line — optimise routing, buffering and congestion, not arithmetic. When someone says "we will get SHARP-like gains on RoCE with better tuning," the hardware answer is no: there is no ALU in the path to do the sum. The honest RoCE counter-argument is different and better — put the reduction in the NVSwitch (NVLS) where you already own the silicon, and keep the scale-out fabric carrying fewer, larger, already-reduced buffers.

**Economics.** GDR is not a cost lever, it is a correctness gate: §9(b) shows the node cannot hit its rated bandwidth without it, so a fabric priced on 3.2 Tb/s per node and deployed without GDR is a fabric you overpaid for by construction. SHARP *is* a cost lever — up to `2(N−1)/N ≈ 2×` fewer bytes on every link for reduction-shaped traffic, which either lets a more oversubscribed (cheaper) spine hit the target step time or raises effective bandwidth on the fabric you already own. But the lever only pulls on workloads whose collectives are reductions in a supported op and dtype. Lesson 07 turns both of these into per-GPU-hour numbers.

## Real-world use cases

- **NCCL's own ACS hint (`src/transport/net_ib/common.h`, v2.31.2-1).** *What it shows:* the ACS-breaks-GPUDirect failure is common enough that NVIDIA hard-coded a diagnostic for it into the completion-error path, keyed on `IBV_WC_LOC_PROT_ERR`, complete with the A/B test (`NCCL_NET_GDR_LEVEL=0`) to confirm it. *Why it matters:* it is the single most reliable way to distinguish "GDR is off" (no `/GDRDMA` token, no error) from "GDR is on but the fabric rejects the transaction" (token present, completions failing). Two different problems, two different fixes.

- **The pre-Ampere GDR-read gate (`src/graph/paths.cc`).** *What it shows:* NCCL disables GPUDirect **reads** — i.e. the send side — by default on GPUs with compute capability below 8.0 that have no local NVLink peer, while leaving GDR writes enabled. *Why it matters:* it is a concrete, still-shipping example of "GPUDirect engaged" not being a boolean. On a V100 or T4 fleet you can legitimately see `/GDRDMA` on every receive line and none on the sends, and that is the library working as designed rather than a misconfiguration. It also explains why an inherited "GDR is broken on our older nodes" ticket is often not a bug.

- **The `nvidia_peermem` invalidation callback (`kernel-open/nvidia-peermem/nvidia-peermem.c`, 610.57.04).** *What it shows:* the whole module exists because GPU memory can be freed while a NIC still holds a mapping to it, so the driver calls back into `ib_core` to revoke the memory region before releasing the pages, and advertises `PEER_MEM_INVALIDATE_UNMAPS` so `ib_core` knows not to unmap during that callback. *Why it matters:* it is the mechanism behind the advice "register once and reuse" — a buffer that is allocated and freed per iteration drags a kernel-side revoke and a NIC translation-table reprogram along with it, which is invisible in profilers that only watch the GPU.

- **The `nccl-rdma-sharp-plugins` capability query (`src/sharp_plugin.c`).** *What it shows:* the plugin explicitly queries the fabric for int8/uint8/bfloat16 support and warns rather than failing when it is absent, then converts those NCCL datatypes to `SHARP_DTYPE_NULL` so the collective falls back. *Why it matters:* it is a real production trap — a bf16 mixed-precision trainer on a SHARP-provisioned fabric can receive zero SHARP benefit, with a single `WARN` line as the only evidence, while the fp32 benchmark you validated the fabric with gets the full gain.

## Worked example

**Node under test:** HGX H100, 8×GPU, 8×ConnectX-7 (400 Gb/s), rail-optimized, dual-socket. Trimmed `nvidia-smi topo -m`:

```
        GPU0  GPU1  GPU2  GPU3  ...  NIC0  NIC1  NIC2  NIC3  CPUAffinity  NUMA
GPU0     X    NV18  NV18  NV18       PXB   SYS   SYS   SYS   0-47         0
GPU1    NV18   X    NV18  NV18       SYS   PXB   SYS   SYS   0-47         0
GPU2    NV18  NV18   X    NV18       SYS   SYS   PXB   SYS   48-95        1
GPU3    NV18  NV18  NV18   X         SYS   SYS   SYS   PXB   48-95        1
```

**Step 1 — read the pairing.** `NICk` is `mlx5_k`. GPU0↔NIC0 is `PXB` (multiple PCIe bridges, no host bridge crossed — the paired rail); GPU0↔NIC1/2/3 are `SYS` (across the socket interconnect). By the diagonal, GPU*k*'s rail NIC is `mlx5_k`. GPU0/1 are on NUMA 0, GPU2/3 on NUMA 1.

**Step 2 — check the distance against NCCL's ladder.** `PXB` is level 5. NCCL's default `netGdrLevel` is also `PATH_PXB` = 5, so `distance > netGdrLevel` is false and GDR is permitted on the paired rail *by default*. `SYS` is level 9 > 5, so GDR on any non-paired NIC is refused by default — correctly.

**Step 3 — verify the kernel shim exists.** Either is sufficient:

```console
$ ls /sys/module/nvidia_peermem/version && echo "peermem: yes"
/sys/module/nvidia_peermem/version
peermem: yes
$ grep -c "DMA-BUF is available" /tmp/nccl.*.log     # the other path
8
```

**Step 4 — the pod env for a rank pinned to GPU3.**

```yaml
env:
  # Exact match (leading '='), so mlx5_3 does not also select mlx5_30 on a
  # denser node. Port 1 named explicitly.
  - { name: NCCL_IB_HCA,        value: "=mlx5_3:1" }
  # Pin the level rather than inheriting it. PXB is also the current default,
  # so this is a stability guarantee, not a change in behaviour: it stops a
  # future NCCL default from silently widening or narrowing the rule.
  - { name: NCCL_NET_GDR_LEVEL, value: "PXB" }
  # Relaxed ordering lets the NIC's PCIe writes complete out of order, which
  # materially helps GDR write throughput on many root complexes.
  - { name: NCCL_IB_PCI_RELAXED_ORDERING, value: "1" }
  # Bootstrap/out-of-band only. Keep NCCL's rendezvous off the IB rail so a
  # slow control path cannot be mistaken for a slow fabric.
  - { name: NCCL_SOCKET_IFNAME, value: "eth0" }
  # Prove it, every run.
  - { name: NCCL_DEBUG,         value: "INFO" }
  - { name: NCCL_DEBUG_SUBSYS,  value: "INIT,NET,GRAPH" }
```

Why `PXB` and not `PIX`: the paired path measures `PXB`, so `PIX` (level 4) would be *stricter* than the hardware and would disable GDR on the very link you want. Why not `SYS`: that would permit GDR out `mlx5_0..2` for GPU3 — a cross-socket path that defeats rail alignment and, on many platforms, will not actually work (§5). If a degraded node forces a GPU onto a non-paired NIC, `SYS` is the knob and you accept the loss knowingly.

**Step 5 — prove it engaged.**

```console
$ grep -E 'Channel 00/0 .*(send|receive)' /tmp/nccl.node01.*.log | head -2
NCCL INFO Channel 00/0 :  3[3] -> 11[3] [send] via NET/IB/3/GDRDMA
NCCL INFO Channel 00/0 : 11[3] ->  3[3] [receive] via NET/IB/3/GDRDMA/flush=None

$ grep -c GDRDMA /tmp/nccl.*.log
128            # non-zero on every rank; zero anywhere is the alarm
```

*(Transcript is representative of the log format strings in `src/transport/net.cc`, not a captured run.)* Read it line by line: `NET/IB` is the transport, `/3` is the NCCL net device index (which maps to `mlx5_3` given the `NCCL_IB_HCA` pin — confirm with the `NCCL_IB_HCA set to` line), `/GDRDMA` is the token that proves the payload path bypasses host memory, and `flush=None` says NCCL determined no explicit PCIe read-flush is needed after the DMA, which it can do from Hopper onward.

**Step 6 — the counterfactual, quantified.** From §9(b): eight rails at 400 Gb/s staged through host memory would demand ~800 GB/s of DRAM bandwidth against ~614 GB/s theoretical peak on a dual-socket DDR5-4800 node. The bounce path cannot deliver the node's rated 3.2 Tb/s at any pipeline depth. State it that way in a review rather than "GDR makes it faster."

**Step 7 — decide on SHARP for this workload.** Suppose the job is a 512-GPU (64-node) data-parallel run reducing bf16 gradients.

```
  ring traffic per rank at N=64 nodes : 2 · 63/64 · S = 1.969 S
  SHARP traffic per rank              :                1.000 S
  reduction in link bytes             : 49.2 %

  ring latency term  : 2·(64−1) = 126 sequential endpoint steps
  SHARP latency term : 2 switch hops (~130 ns each) + 1 endpoint round trip

  BUT: the payload is bf16.  Check first:
    $ grep -E 'SHARP.*Datatypes|SHARP rank' /tmp/nccl.*.log
    NCCL INFO SHARP rank 0/64 initialized on mlx5_0
    # no "Datatypes not supported" warning ⇒ v3 datatypes present ⇒ bf16 offloads
```

If that warning *is* present, the correct conclusion is not "SHARP is broken" but "SHARP does not apply to this dtype on this fabric" — and the options are to reduce in fp32 (doubling the bytes to halve them again, usually a wash) or to stop counting SHARP in the fabric's justification for this workload.

## Practice

Feeds the deliverable's **HCA-pinning and GDR-verification decision**.

1. Using the `topo -m` above (or your own node's real output), build a per-GPU table with four columns: **GPU index · rail NIC (`mlx5_*`) · topo code for that pair · NUMA node**. Do GPU0 through GPU3 at minimum.
2. For each GPU, write the pod env block: `NCCL_IB_HCA` (using exact-match `=` syntax), `NCCL_NET_GDR_LEVEL` justified from the topo code and the numeric ladder in §7, and `NCCL_IB_PCI_RELAXED_ORDERING`.
3. Write the **GDR verification procedure** as four checks a runbook could execute in order, and for each one state the exact command and the exact output that means "pass":
   (a) is a kernel shim present — and name *both* acceptable answers;
   (b) is the GPU–NIC distance within the level;
   (c) did the connection log carry `/GDRDMA` on **both** send and receive lines;
   (d) are there `IBV_WC_LOC_PROT_ERR` completions.
4. For each of the four silent-fallback causes in §8, write one sentence giving the distinguishing evidence and the fix. Then write one more sentence on the Kubernetes variant of cause 2 — right topology, wrong NIC in the pod — and what you would check first.
5. Compute, for **your** target cluster's node count `N`, the SHARP link-traffic reduction `2(N−1)/N` and state in one sentence whether the workload's dominant collective is reduction-shaped or point-to-point-shaped. If it is a bf16 workload, name the log line you would check before counting the benefit.
6. Redo the §9(b) DRAM arithmetic with your own node's NIC count, NIC rate and memory configuration, and write the one-sentence conclusion.

**Acceptance:** a per-GPU pinning table plus a four-step GDR verification runbook with exact commands and pass criteria, a four-cause fallback triage table, and the two calculations (SHARP traffic ratio, bounce-path DRAM demand) with your own inputs — dropped into the deliverable. It is done when someone could paste your env block into a pod spec on that topology, run your four checks, and know within five minutes whether GDR engaged and, if not, which of the four causes it is.

## Common pitfalls

- **Treating "GDR is off" and "GDR errors" as the same problem.** *Symptom:* two completely different failures both described as "GPUDirect isn't working." *Mechanism:* if a gate in `ncclTopoCheckGdr` fails, NCCL never attempts the peer DMA — no error, no `/GDRDMA` token, a working but slow job. If the gates pass and ACS redirects the transaction, the DMA *is* attempted and fails with `IBV_WC_LOC_PROT_ERR`. *Fix:* check for the token first; only if it is present should you go looking for completion errors, and vice versa.

- **Checking only `lsmod | grep nvidia_peermem`.** *Symptom:* "peermem isn't loaded, so GDR is impossible" — on a cluster where GDR is working fine. *Mechanism:* NCCL accepts **either** a peer-memory client **or** dma-buf, and prefers dma-buf when both exist. A modern inbox-kernel stack with CUDA ≥ 11.7 needs no peermem at all. *Fix:* check both — the three sysfs paths and the `DMA-BUF is available on GPU device` log line.

- **Believing NCCL's GDR default is too conservative for HGX.** *Symptom:* people set `NCCL_NET_GDR_LEVEL=SYS` "to make sure GDR turns on," which permits cross-socket GDR paths that are slower than the bounce they replace and may not work at all. *Mechanism:* the default is `PATH_PXB` (5), which is exactly the paired-rail distance on a standard HGX board; it excludes `PHB` (8) and `SYS` (9) deliberately. *Fix:* set the level explicitly to `PXB` for stability, not to `SYS` for permissiveness. Widen only for a specific, understood, degraded placement.

- **Confusing GDRCopy with GPUDirect RDMA.** *Symptom:* `NCCL_GDRCOPY_ENABLE=1` set in production as a performance measure, with no measured effect. *Mechanism:* GDRCopy maps a GPU page into the CPU's address space so the CPU can write an 8-byte FIFO sync word directly; it is a control-path latency optimisation used only when the proxy is same-process and same-device. It is off by default upstream and touches no payload bytes. *Fix:* leave it at its default unless you have an A/B measurement on your own workload showing a win.

- **Quoting an NVLS measurement as an InfiniBand SHARP result.** *Symptom:* "SHARP gives us 1.3×" cited in a fabric procurement argument, where the measurement came from a single-node `all_reduce_perf`. *Mechanism:* NVLink SHARP reduces in the NVSwitch inside a node and never touches the scale-out fabric; InfiniBand SHARP reduces in the IB switch and is what a fabric purchase buys. They share a name and nothing else. *Fix:* state which one, and state the node count — a single-node number is definitionally NVLS.

- **Assuming SHARP offloads your reduction.** *Symptom:* a SHARP-provisioned fabric showing no improvement on a bf16 or `ncclAvg` workload. *Mechanism:* the NCCL SHARP plugin offloads only `SUM`, `MAX` and `MIN`; `ncclAvg`, `ncclProd` and custom ops map to `SHARP_OP_NULL` and fall back. bf16/int8/uint8 offload only when the fabric reports v3 datatype support, which the plugin discovers via `sharp_coll_caps_query()` and reports as a single `WARN`. *Fix:* grep the init log for the datatype warning and for `SHARP rank N/M initialized` before crediting SHARP with anything.

- **Reading `busbw` as a wire measurement once in-network reduction is on.** *Symptom:* a reported bandwidth above the link's physical spec, treated as either a bug or a miracle. *Mechanism:* `nccl-tests` multiplies algorithmic bandwidth by `2(N−1)/N` for all-reduce, a factor that describes the ring's traffic pattern. When the switch does the reduction, that pattern no longer holds and the product stops describing bytes on the wire. *Fix:* compare like with like — run the same size with the in-network path disabled (`NCCL_NVLS_ENABLE=0` for NVLS, `NCCL_SHARP_DISABLE=1` for IB SHARP) and report the ratio.

## Self-check

- **What actually breaks GPUDirect RDMA when the GPU and NIC are on different root complexes, and how does the driver decide?**
  **Answer:** GDR is a PCIe peer-to-peer DMA: the NIC issues memory transactions against physical addresses inside the GPU's BAR window. For those to be routed laterally, the two devices need a common upstream PCIe bridge. The NVIDIA driver implements this literally — `nv_pci_has_common_pci_switch()` walks `pci_upstream_bridge()` from the GPU and, for each ancestor, walks the same chain from the NIC, returning true on the first common bridge. Devices on different root complexes have no common PCIe bridge; their chains terminate in two root complexes joined by UPI/xGMI, which is not a bridge, so the walk fails. That check is gated on `NV_FLAG_PCI_P2P_UNSUPPORTED_CHIPSET` (set when the platform is not known to be both PCIe-peer-read and peer-write capable) and is skipped entirely on cache-coherent Grace platforms and in passthrough VMs. `SYS` in `nvidia-smi topo -m` is the user-visible name for "this walk will fail"; NCCL never even attempts it, because `SYS` (level 9) exceeds the default GDR level `PXB` (5).

- **Name the two kernel mechanisms that let a NIC register GPU memory, and how NCCL detects each.**
  **Answer:** (1) The **peer-memory client**: `nvidia_peermem` registers a `peer_memory_client` with `ib_core`; when `ibv_reg_mr` gets an unresolvable address, `ib_core` offers it to each client, and `nvidia_peermem` claims it, calls `nvidia_p2p_get_pages` to pin (rounding to 64 KiB, `GPU_PAGE_SHIFT` 16) and `nvidia_p2p_dma_map_pages` to obtain bus addresses, then hands back an `sg_table` for the MTT/MPT. It also registers a `free_callback` that invalidates the MR if CUDA frees the memory. NCCL detects it by testing for `/sys/module/nvidia_peermem/version`, `/sys/kernel/mm/memory_peers/nv_mem/version`, or `/sys/kernel/mm/memory_peers/nv_mem_nc/version`. (2) **dma-buf**: the GPU driver exports a `dma_buf` fd and user space calls `ibv_reg_dmabuf_mr(pd, offset, length, iova, fd, access)`. NCCL detects it by calling that verb with `fd = -1` and zero length — `EOPNOTSUPP`/`EPROTONOSUPPORT` means unsupported, any other errno (e.g. `EBADF`) means the path exists — and additionally requires CUDA ≥ 11.7 with `CU_DEVICE_ATTRIBUTE_DMA_BUF_SUPPORTED` on the device, logging `DMA-BUF is available on GPU device N`. Either is sufficient; NCCL prefers dma-buf when both are present. On Kubernetes, `nvidia_peermem` is loaded by the **GPU** Operator, not the Network Operator.

- **What does ACS do, why does it break GPUDirect, and what is the exact symptom?**
  **Answer:** PCIe Access Control Services, when source-validation and P2P-request-redirect are enabled on a switch's downstream ports, prevents the switch from forwarding a peer-to-peer transaction laterally to a sibling port and instead redirects it upstream to the root complex so the IOMMU can validate it. That is correct for virtualisation isolation and fatal for GDR: the transaction that should have stayed inside the PCIe switch takes a round trip to the CPU, and depending on IOMMU state is either far slower or rejected. The symptom is an RDMA completion with status `IBV_WC_LOC_PROT_ERR`, for which NCCL prints a hard-coded hint naming ACS and suggesting `NCCL_NET_GDR_LEVEL=0` as the A/B confirmation. Check with `lspci -vvv -s <bridge> | grep ACSCtl` and look for `SrcValid+ ReqRedir+ CmpltRedir+`; clear with `setpci -s <bridge> ECAP_ACS+0x6.w=0000` on every bridge between the pair (not reboot-persistent — prefer BIOS). Inside a VM you cannot disable ACS at all; enable ATS on the adapter instead.

- **Derive SHARP's reduction in link traffic for an all-reduce over N ranks, and say where the bigger win actually is.**
  **Answer:** A ring all-reduce is a reduce-scatter of `N−1` steps of `S/N` bytes followed by an all-gather of the same, so each rank sends and receives `2(N−1)/N · S` — the factor `nccl-tests` uses to convert algorithmic to bus bandwidth. SHARP sends each rank's buffer once up the aggregation tree and multicasts the result once back down, so each rank sends and receives `S`. The traffic ratio is therefore exactly `2(N−1)/N`: 1.00× at N=2, 1.75× at N=8, 1.969× at N=64, 1.984× at N=128 — asymptotically 2×, never more. The *bigger* win is latency: ring is `2(N−1)` sequential dependent steps (1,022 at N=512); a GPU-mediated tree is `2·log₂(N)` hops but every interior node is a GPU doing a NIC round trip plus a kernel; SHARP's interior nodes are switch ASICs at roughly 100–130 ns per hop with a tree depth of one or two on a high-radix fabric. That is why SHARP's headline benefit shows up on small, frequent, large-world-size reductions rather than on bandwidth-bound big buffers. It also explains the two thresholds in the code: `NCCL_SHARP_GROUP_SIZE_THRESH` = 2 and `NCCL_COLLNET_NODE_THRESHOLD` = 2, below which there is provably nothing to gain.

- **A 512-GPU bf16 training run on a SHARP-provisioned InfiniBand fabric shows no improvement over the same run with `NCCL_SHARP_DISABLE=1`. Give three candidate explanations, in the order you would check them.**
  **Answer:** (1) **SHARP never engaged at all.** `NCCL_COLLNET_ENABLE=1` must be set, a CollNet plugin must be loadable, and the communicator must span ≥ `NCCL_COLLNET_NODE_THRESHOLD` (default 2) nodes; without all three NCCL silently uses Ring/Tree. Check for `SHARP rank R/N initialized on <dev>` and `Connected CollNet` in the init log — absence of those lines settles it. (2) **The datatype is not offloadable on this fabric.** bf16 (with int8/uint8) offloads only when `sharp_coll_caps_query()` reports v3 datatype support; otherwise the plugin logs `SHARP int8,uint8,bfloat16 Datatypes not supported`, maps the type to `SHARP_DTYPE_NULL`, and every collective falls back while the fabric still reports itself SHARP-capable. Grep for that warning. (3) **The collective is not reduction-shaped, or the op is not offloadable.** SHARP handles `SUM`/`MAX`/`MIN` only — `ncclAvg` and `ncclProd` fall back — and offloads nothing for all-to-all or pipeline-parallel sends, so a run whose time is dominated by expert-parallel all-to-all will show no change no matter how healthy SHARP is. A fourth, lower-probability check: on SHARP ≤ 3.6 the plugin caps a single collective at 512 MiB, so very large buffers are chunked and the per-collective gain is diluted.

## Connections & what's next

GDR is the fabric-scale continuation of 02b's PCIe peer-to-peer rule and of lesson 01's rail promotion, and it is the concrete mechanism underneath every inter-node NCCL send from 08 — lesson 03 named it and deferred the preconditions to here, and this lesson supplied them: the kernel shim, the driver's ancestor walk, the ACS bit, and NCCL's numeric distance ladder. SHARP extends 04's InfiniBand-vs-RoCE argument with the one capability RoCE has no equivalent for, and §11's `2(N−1)/N` is the number lesson **07** converts into dollars — SHARP as a byte-reduction lever that changes how much bisection you have to buy at the spine.

This lesson assumed a pod could simply *have* a rail-aligned NIC to pin `NCCL_IB_HCA` against. **06** is the reality check. The default CNI gives a pod exactly one interface on the cluster pod network; getting a second, RDMA-capable, NUMA-aligned device into the pod's namespace takes Multus, an SR-IOV stack, and the Network Operator — and every one of those layers can produce the §8 cause-2 signature on a node whose `topo -m` is perfect. Read 06 as "everything in this lesson, but now the NIC is a schedulable Kubernetes object that has to survive admission control."

## References & further reading

**Primary sources — read directly and relied upon**

1. **NVIDIA NCCL, source tree at v2.31.2-1** — https://github.com/NVIDIA/nccl — cloned and read. The GDR decision procedure (`src/graph/paths.cc`: `ncclTopoCheckGdr`, `ncclGetLevel`, `NCCL_PARAM(NetGdrRead, "NET_GDR_READ", -2)`, `NCCL_PARAM(NetGdrC2c, "NET_GDR_C2C", 1)`); the `PATH_*` ladder (`src/include/graph.h`); the connection log format strings and GDRCopy parameters (`src/transport/net.cc`); `NCCL_BUFFSIZE` default 4 MiB and `NCCL_COLLNET_NODE_THRESHOLD` default 2 (`src/init.cc`); `NCCL_STEPS` = 8 (`src/include/device.h`); the `NCCL_IB_HCA` grammar (`src/transport/net_ib/init.cc`, `src/misc/utils.cc`). **Correction to the previous version of this lesson:** the default GDR level is `PATH_PXB`, which *permits* GDR on the HGX paired rail; the earlier claim that the default excludes that path was wrong.

2. **NCCL, `src/transport/net_ib/gdr.cc` and `common.h`** — same tree. The three peermem sysfs probe paths, the `ibv_reg_dmabuf_mr(fd=-1)` capability test and its `EOPNOTSUPP`/`EPROTONOSUPPORT` semantics, and the hard-coded ACS hint on `IBV_WC_LOC_PROT_ERR` including the `NCCL_IB_TIMEOUT` default of 20.

3. **NVIDIA open-gpu-kernel-modules, tag 610.57.04** — https://github.com/NVIDIA/open-gpu-kernel-modules — cloned and read. `kernel-open/nvidia-peermem/nvidia-peermem.c` for the peer-memory client registration, `GPU_PAGE_SHIFT` 16, the `nvidia_p2p_get_pages` → `nvidia_p2p_dma_map_pages` sequence, the `nv_dma_map` 64 KiB assertion and the invalidation callback; `nv-p2p.h` for the documented 64 KiB alignment requirement and the persistent-API flags; `kernel-open/nvidia/nv-pci.c` for `nv_grdma_pci_topology_supported()` and `nv_pci_has_common_pci_switch()`; `kernel-open/nvidia/nv-dmabuf.c` for the dma-buf attach-time topology check and its `-ENOTSUPP` message.

4. **`nccl-rdma-sharp-plugins`, `src/sharp_plugin.c`** — https://github.com/Mellanox/nccl-rdma-sharp-plugins — cloned and read. `NCCL_SHARP_GROUP_SIZE_THRESH` default 2, `NCCL_SHARP_DISABLE`, `NCCL_SHARP_V3_DATATYPES` default 2, `NCCL_SHARP_DISABLE_REDUCE_SCATTER` / `_ALLGATHER`, the `SHARP_COLL_ENABLE_SAT=1` default, the 512 MiB `maxCollBytes` cap on SHARP ≤ 3.6, the `typeConvert`/`opConvert` tables that define exactly which ops and datatypes offload, and the `sharp_coll_caps_query()` datatype gate.

5. **rdma-core, `libibverbs/verbs.h` and `libibverbs/man/ibv_reg_mr.3`** — https://github.com/linux-rdma/rdma-core — cloned and read. The `ibv_reg_dmabuf_mr()` signature, the `iova` page-offset requirement, and the restricted set of access flags it accepts.

6. **`nccl-tests`, `src/all_reduce.cu`** — https://github.com/NVIDIA/nccl-tests — cloned and read. `busBw = algBw × 2(N−1)/N` for all-reduce, versus `(N−1)/N` for all-gather and reduce-scatter — the factor behind §11's derivation and the "busbw above wire rate" tell.

7. **NVIDIA Network Operator, `README.md`** — https://github.com/Mellanox/network-operator — cloned and read. The statement that from driver v465 `nvidia_peermem` ships inside the GPU driver and **the GPU Operator manages loading it** — the division of labour that catches people out on Kubernetes.

8. **stas00, *ml-engineering*, network chapter** — https://github.com/stas00/ml-engineering — cloned and read (commit 139708e). The measured NVLS-vs-Ring tables on 8×H200 (480.0 vs 367.2 GB/s at 16 GiB, NCCL 2.27.7) and 8×B200 `p6-b200.48xlarge` (838.0 vs 682.2 GB/s, measured 2026-08-09), the differing NVLS GPU-count thresholds, the ACS/ATS note for virtual machines, and the `2N` → `N+1` framing of SHARP's send count. A practitioner source with dated, reproducible measurements — read it for what an actual A/B looks like.

**Optional depth — could not be fetched from this environment, and no claim in this lesson rests on them**

9. **NVIDIA GPUDirect RDMA developer documentation** — https://docs.nvidia.com/cuda/gpudirect-rdma/ — `docs.nvidia.com` is blocked by this environment's egress proxy. **Not relied upon.** The BAR-mapping and `nvidia_p2p_*` mechanics in §2–§3 were verified against the driver source (entry 3) instead. Read the vendor page for the userspace-API framing and the supported-platform matrix.

10. **NVIDIA GPUDirect Storage overview guide** — https://docs.nvidia.com/gpudirect-storage/overview-guide/index.html — blocked. **Not relied upon**; §10 is deliberately limited to what follows from the PCIe mechanism plus the `nvidia-fs`/cuFile names, and says so.

11. **NCCL environment-variable reference** — https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html — blocked. **Not relied upon.** Every default and value semantic quoted in this lesson was read from the NCCL source (entries 1–2). Consult the page for prose descriptions, but note that the source is the authority when they disagree.

12. **NVIDIA SHARP user manual** — https://networking-docs.nvidia.com/sharpum/300 — blocked. **Not relied upon.** The aggregation-tree model in §11 is derived from the collective arithmetic plus the plugin's actual API usage (entry 4). Read it for the aggregation-manager operational model (`sharp_am`, tree provisioning, tenancy) which this lesson does not attempt to specify.

13. **Lambda — "Introducing NVIDIA SHARP on Lambda 1CC"** — https://lambda.ai/blog/nvidia-sharp-on-lambda-1cc — blocked. **Not relied upon.** An earlier version of this lesson quoted a "45–63% all-reduce improvement across 16–1,500 GPUs" figure from it; that figure could not be verified here and has been removed rather than repeated. The verified numbers in §12 are NVLS measurements from entry 8, and they are a different mechanism — do not substitute one for the other.

14. **CoreWeave — NCCL configuration reference and GPUDirect RDMA how-to** — https://docs.coreweave.com/products/networking/hpc-interconnect/nccl-configuration-reference — blocked. **Not relied upon.** An earlier version of this lesson attributed a "do not enable `NCCL_GDRCOPY_ENABLE`" caution to it. The substance is preserved from a source that could be read: NCCL's own default for that parameter is `0` (`src/init.cc`, "GDRCOPY support: Off by default"), and the source shows it affects only an 8-byte control-path sync word.
