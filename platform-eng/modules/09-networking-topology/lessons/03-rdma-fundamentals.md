---
lesson: "09.3"
title: "RDMA fundamentals: kernel bypass, zero-copy, queue pairs"
module: "09"
concept: "RDMA fundamentals: kernel bypass, zero-copy, queue pairs"
status: not-started
est_time: "7h"
prev: "02-inter-node-fabric.md"
next: "04-ib-vs-roce-lossless.md"
artifacts: []
sources: 10
---
# 09.3 · RDMA fundamentals: kernel bypass, zero-copy, queue pairs

> **Concept.** RDMA is the deliberate deletion of the kernel datapath you learned in 01b — the NIC reads and writes application (and GPU) memory directly, so the CPU, socket buffers, netfilter, and conntrack are simply not on the wire. The win that matters for collectives isn't a faster average — it's a flat tail.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Where this fits

09.2 built the *shape* of the fabric above the NIC — radix, tiers, bisection, oversubscription ratios, and the routing behaviour that decides whether the capacity you bought can actually be reached. That answered "how many wires, arranged how." It did not answer what rides those wires, or why it matters that it is not TCP.

This lesson answers that. It takes the 01b kernel datapath you already traced — `send()` → `sk_buff` → TCP/IP → netfilter/conntrack → qdisc → NIC → softirq → `recv()` — and shows precisely which stages RDMA removes, what replaces them, and why removing *these particular* stages buys latency **stability** rather than merely lower average latency. That distinction is the whole point: a synchronous collective is a barrier, so 511 GPUs wait on the slowest one, and a jittery tail is a stalled step rather than an average inconvenience (09.2 §10 drew that as a timeline).

What this lesson does *not* cover is how the fabric itself becomes trustworthy enough for RDMA to run on. RDMA's reliable transport assumes packets essentially never get dropped to congestion, and Ethernet was not built that way. That is 09.4, and this lesson deliberately sets it up: everything here about hardware retransmission explains *why* a lossy fabric is catastrophic rather than merely slow.

## Why this matters

The interview question that separates a networking person from a platform engineer who runs GPU fabrics is *"why can't we just train on TCP?"* The wrong answer is "TCP is slow." The right answer names mechanism, and it has three parts you can defend with numbers:

- **CPU cost is measurable and large.** Microsoft's production measurements, published at SIGCOMM 2016, put sending at 40 Gb/s over 8 TCP connections at **6% of the aggregate CPU time of a 32-core Xeon E5-2690 server**, and receiving at 40 Gb/s at **12%** — with checksum offload, LSO, RSS and interrupt moderation all enabled. Scale that to 400 Gb/s per rail NIC and the arithmetic stops being an optimisation discussion: you would be spending most of a CPU socket per node moving bytes, and those are the cores your data loader needs.
- **The tail is the product, not the mean.** The same paper is explicit that kernel software latency can reach tens of milliseconds under load and that congestion drops, while rare, are not absent. A collective is a barrier; p99.9 on any one link becomes the step time for every GPU in the job.
- **Utilisation is money.** If TCP-class overhead and jitter drop fabric-bound collective efficiency from 95% to 70%, you are burning a quarter of the largest line item in a training budget, continuously.

This lesson is conceptual — you will not write verbs code — but "conceptual" here means you can stand at a whiteboard and draw both datapaths side by side, name each object the NIC holds, and say what happens on a dropped packet. Three claims you should be able to defend by the end: RDMA is **kernel bypass + zero-copy + transport offload**, which are three distinct wins; the win that matters for collectives is **tail stability**, and it comes specifically from deleting the *stateful, shared* kernel stages; and the **remote CPU is uninvolved** in a one-sided WRITE, a property TCP structurally cannot offer.

## What's new here (calibration)

You already know 01b's kernel datapath cold, 02b's PCIe and NVLink topology, 08's NCCL collectives, and 06's topology-aware scheduling. RDMA is introduced here strictly as *the substrate NCCL's inter-node transport rides on*. What is genuinely new:

- The **object model** — protection domain, memory region, queue pair, completion queue — and what each one exists to prevent, not just what it is called.
- **Memory registration internals**: pinning, the NIC's translation and protection tables, `lkey`/`rkey` as capability tokens, and what registration costs.
- The **per-message data path in hardware**: work-queue entry, doorbell, NIC-side DMA of the descriptor, scatter-gather, completion-queue entry, and the two ways to learn about completion.
- The **queue-pair state machine** (`RESET → INIT → RTR → RTS`) and exactly which parameters each transition programs — including the retransmission timers, which is where 09.4's story starts.
- **Reliability in silicon**: packet sequence numbers, ACK/NAK, the local-ACK-timeout formula, retry counters, and the genuinely important correction that classic RoCE NICs recovered from loss with **go-back-0**, not go-back-N, with a documented production consequence.
- **GPUDirect RDMA's mechanism and its ordering hazard**, including why a "flush" read exists at all.
- Real, annotated output from `ibstat`, `ibv_devinfo`, `show_gids` and `ib_write_bw`, with the formats taken from the tools' own source.

## Core concepts

### 1. What the kernel datapath actually costs, stage by stage

Before deleting something, price it. Take one 1 MB gradient shard moving from host A to host B over TCP, and label each stage with the *kind* of cost it imposes — because the kinds matter more than the magnitudes.

| Stage | Work | Cost class |
|---|---|---|
| `send()` syscall | mode switch, argument copy, socket lock | fixed, per message |
| copy into socket send buffer | 1 MB memcpy into `sk_buff` pages | **per byte** |
| TCP/IP | segmentation (or TSO), sequence numbers, checksums, congestion window | per packet, CPU |
| netfilter + conntrack | flow-tuple hash lookup and state update; NAT/overlay hooks | **stateful, shared, load-dependent** |
| qdisc + driver | enqueue, dequeue, ring descriptor write, doorbell | per packet |
| wire | serialisation, switch queueing | fabric |
| softirq `NET_RX` on B | drain the ring under a per-core budget shared with every other flow | **stateful, shared, load-dependent** |
| IP/TCP reassembly, conntrack lookup | reorder, ACK, checksum verify | stateful |
| copy to socket receive buffer, then to userspace | two more 1 MB memcpys | **per byte** |

The per-byte and per-packet costs explain the CPU numbers in the previous section. The rows marked *stateful, shared, load-dependent* explain the tail: their cost depends on **what else the machine is doing**. A conntrack table under pressure takes locks and misses cache; a softirq budget shared with a noisy neighbour defers your packets. Neither shows up in a quiet benchmark, and both show up exactly when the fabric is busy — that is, mid-collective.

**RDMA's value proposition, precisely stated:** it does not make these stages faster. It removes them, and it removes the *variance* along with the mean.

### 2. The three deletions, and why they are separable

"RDMA is fast" collapses three independent mechanisms. Keep them separate, because a deployment can have one without the others and the failure modes differ.

- **Kernel bypass** — the *control* transition disappears from the steady state. After setup, posting work is a userspace store to a mapped register. Deletes: syscalls, scheduler interaction, netfilter/conntrack, softirq.
- **Zero-copy** — the *payload* is never duplicated. The NIC DMAs out of and into the application's own registered buffers. Deletes: both socket-buffer copies, and with GPUDirect, both GPU↔host staging copies.
- **Transport offload** — reliability, ordering, segmentation and acknowledgement run in the NIC ASIC rather than in kernel software. Deletes: the TCP/IP stack's CPU cost on both ends.

You can have kernel bypass without zero-copy (a userspace stack that still copies), and zero-copy without transport offload (`sendfile`-style paths that still run TCP in software). RDMA is the case where all three hold at once, and each contributes a different part of the win: bypass and offload flatten the tail, zero-copy removes the per-byte CPU cost.

### 3. The object model — what the NIC holds, and what each object prevents

Removing the kernel from the datapath removes an arbiter that was doing three jobs: multiplexing one NIC across many processes, enforcing memory isolation, and holding connection state. RDMA does not eliminate the need for those jobs — it moves them into silicon, and the object model *is* that relocation. Every object below exists to prevent a specific disaster.

| Object | Created by | What it holds | What it prevents |
|---|---|---|---|
| **Device context** (`ibv_open_device`) | once per process | the process's handle on the HCA | unmediated access to the device |
| **Protection domain** (`ibv_alloc_pd`) | once per process | the association group for MRs, QPs, AHs | a QP touching a memory region it was never associated with |
| **Memory region** (`ibv_reg_mr`) | per buffer, at setup | pinned pages, address translation, access rights, `lkey`/`rkey` | the NIC DMA-ing to a page that moved, or to memory the peer was not granted |
| **Queue pair** (`ibv_create_qp`) | per connection | send queue, receive queue, transport type, peer identity, timers | one process's work requests reaching another's connection |
| **Completion queue** (`ibv_create_cq`) | per QP or shared | completion entries | the CPU having to be interrupted to learn anything |
| **Address handle** (`ibv_create_ah`) | for datagram transports | destination path attributes | needing a connection for one-shot sends |

The **queue pair** is the load-bearing one. It is the RDMA connection: a **send queue** and a **receive queue** living in NIC-visible memory, plus per-connection state — transport service type, peer queue-pair number, path MTU, retransmission timers, and the protection domain that scopes which memory regions its work requests may name. Because the kernel programs the QP once and the NIC enforces it thereafter, a process physically cannot address another process's memory: its work requests carry `lkey`s valid only within its own protection domain, and a remote peer's WRITE must present the `rkey` it was explicitly given.

**This is the rebuttal to "RDMA is just a fast socket."** A socket's isolation is enforced by the kernel on every syscall. RDMA's is enforced by the NIC on every work-queue entry, checked against state the kernel programmed exactly once. Same guarantee, different enforcement point, radically different per-message cost.

### 4. Memory registration: the cost you pay once, and what it buys

`ibv_reg_mr(pd, addr, length, access)` is not bookkeeping. It does three things, each of which is required for the datapath to be safe:

1. **Pins the pages.** The NIC will DMA to physical addresses asynchronously and without coordination, so the pages must not be swapped, migrated or freed underneath it. Pinning removes them from the pageable pool, which makes registered memory a **finite, node-wide resource** — you pool and reuse MRs, you do not register per message.
2. **Programs the NIC's translation and protection tables.** The HCA keeps its own virtual-to-physical mapping (historically the memory translation table and memory protection table) so it can accept a *virtual* address in a work request and resolve it without the CPU. Registration is what populates those entries.
3. **Mints the keys.** Registration returns an `lkey` (local key), which local work requests must present, and an `rkey` (remote key), which a peer must present in a one-sided WRITE or READ. The `rkey` is a **capability token**: possession of it, plus the virtual address, is exactly the authority to write that region and nothing else.

The `access` argument is a bitmask that decides what the region will tolerate. NCCL's own registrations show the real vocabulary in use — `IBV_ACCESS_LOCAL_WRITE` for a receive-side scratchpad, `IBV_ACCESS_REMOTE_WRITE | IBV_ACCESS_LOCAL_WRITE | IBV_ACCESS_REMOTE_READ` for a FIFO the peer both writes and reads, and `IBV_ACCESS_REMOTE_ATOMIC` where atomics are used (all visible in `src/transport/net_ib/connect.cc` of NCCL v2.31.2). The principle: **grant the narrowest access the protocol needs**, because an `rkey` with `REMOTE_WRITE` is a standing permission for a peer to modify your memory with no further checks.

Two refinements you should know exist, because they change the cost model:

- **On-demand paging (ODP)** lets the NIC take a page fault and have the kernel populate the mapping, removing the hard pinning requirement at the cost of a fault on the datapath. It converts a registration-time cost into a rare runtime cost.
- **DMA-BUF registration** is how GPU memory becomes an MR on modern stacks: the GPU driver exports the allocation as a DMA-BUF file descriptor and the RDMA stack imports it, replacing the older out-of-tree peer-memory shim. Same pin-and-translate contract, applied to device memory.

**The framing to carry into an interview:** RDMA moves *all* the expensive, stateful, kernel-touching work — connection setup, page pinning, translation programming, key exchange — into a one-time setup phase, leaving the per-message path as pure silicon. That is also why "kernel bypass" never meant "the kernel is not involved."

### 5. The data path in hardware, and the two paths side by side

Here is the mechanism, step by step, for one RDMA WRITE:

1. The application builds a **work request** — opcode, a scatter-gather list of `{addr, length, lkey}` triples, and for a WRITE the remote address and `rkey` — and the provider library writes it as a **work-queue entry (WQE)** into the send queue's memory.
2. It rings the **doorbell**: a store to a memory-mapped NIC register saying "the send queue has new entries." On Mellanox/NVIDIA hardware small WQEs can be written *directly* into the doorbell region as a write-combined burst (the "BlueFlame" path), saving the NIC a DMA read to fetch the descriptor.
3. The NIC fetches (or already has) the WQE, walks the scatter-gather list, translates each address through the MR's entries, and issues PCIe **memory reads** to gather the payload — from host DRAM, or from GPU HBM if the region is GPU memory.
4. It segments to the path MTU, adds transport headers with the next **packet sequence number**, computes CRCs, and transmits.
5. The remote NIC validates the `rkey` and the target range, and issues PCIe **memory writes** straight into the destination buffer. For a one-sided WRITE, no receive-side work request was consumed and no remote CPU was involved at any point.
6. The remote NIC acknowledges in hardware; on receiving the ACK the sending NIC writes a **completion-queue entry (CQE)** — status, opcode, byte count, and the work request's user-supplied ID — into the send CQ.
7. The application learns about completion either by **polling** the CQ (`ibv_poll_cq` in a loop: lowest latency, burns a core) or by arming the CQ for an **event** (`ibv_req_notify_cq` plus a completion channel: an interrupt-driven wakeup, cheaper on CPU, higher latency). Bulk-transfer code typically also uses **CQ moderation** — one completion per N work requests — to cut completion overhead; perftest's default is `CQ Moderation : 100`, one CQE per 100 posted requests.

Now the comparison that is the point of the lesson:

```
  KERNEL TCP PATH                          RDMA VERBS PATH
  (1 MB, GPU→GPU, no GPUDirect)            (1 MB, GPU→GPU, GPUDirect)

  ┌─ USERSPACE ─────────────────┐          ┌─ USERSPACE ────────────────┐
  │ app buffer                   │          │ app posts WQE into SQ mem  │
  │   │ cudaMemcpy D2H  ①        │          │   │ store to doorbell  ①   │
  │   ▼                          │          │   ▼  (no syscall)          │
  │ pinned host staging buffer   │          └──────────┬─────────────────┘
  │   │ send()  ── SYSCALL ──▶   │                     │ MMIO
  ├───┼─ KERNEL ─────────────────┤          ┌──────────▼─────────────────┐
  │   ▼ copy ②                   │          │        HCA (silicon)       │
  │ sk_buff / socket send buffer │          │  fetch WQE                 │
  │   ▼                          │          │  translate via MR (lkey)   │
  │ TCP: segment, seq, cksum,    │          │  DMA-read payload  ────────┼──┐
  │      congestion window       │          │  segment to path MTU       │  │
  │   ▼                          │          │  add BTH + PSN, CRC        │  │
  │ netfilter + CONNTRACK  ★     │          │  transmit                  │  │
  │   ▼                          │          └────────────────────────────┘  │
  │ qdisc → driver → ring        │                     │                    │
  └───┼──────────────────────────┘          ┌──────────▼─────────────┐      │
      │ DMA                                  │  PCIe peer-to-peer     │      │
      ▼                                      │  read straight out of  │◀─────┘
    [ NIC ] ──────── wire ────────▶          │  GPU HBM (no bounce)   │
                                             └────────────────────────┘
  ┌─ RECEIVER, KERNEL ──────────┐                      │ wire
  │ IRQ → SOFTIRQ NET_RX  ★     │                      ▼
  │ IP/TCP reassembly, ACK      │          ┌────────────────────────────┐
  │ CONNTRACK lookup  ★         │          │  remote HCA validates rkey │
  │ copy ③ → socket rcv buffer  │          │  DMA-writes into GPU HBM   │
  │ recv() ── SYSCALL ──▶       │          │  hardware ACK              │
  │ copy ④ → userspace          │          │  posts CQE on the SENDER   │
  │ cudaMemcpy H2D ⑤ → GPU HBM  │          └────────────────────────────┘
  └─────────────────────────────┘             REMOTE CPU: did nothing.
                                              Not interrupted, no copy,
  ★ = stateful, shared, LOAD-DEPENDENT        may not know it happened.
      → these are the tail-latency sources
```

**Read the star-marked rows as the thesis.** The stages TCP adds that RDMA deletes are precisely the *load-dependent, shared, stateful* ones. That is why the win is a flat tail rather than a smaller mean, and it is why an average-latency benchmark understates RDMA's value for barrier-synchronous work by a wide margin.

One more consequence, which matters enormously in Kubernetes and shows up again in 09.6: **there is no flow tuple to inspect**. Conntrack, stateful firewalls, service meshes and overlay encapsulation all operate on state RDMA never creates. RDMA and the standard pod-network security model are not merely in tension; they are describing different objects.

### 6. The queue-pair state machine — how a connection is actually built

A QP is not usable when created. It is walked through a state machine, and each transition programs a specific set of attributes into the NIC. This is where the parameters that govern retransmission behaviour get set, so it is also the hinge into 09.4.

```
   TIME ──────────────────────────────────────────────────────────────▶

   HOST A                          out-of-band                HOST B
   ──────                          (TCP socket, MPI,          ──────
                                    rdma_cm, or NCCL's
                                    bootstrap)
   ibv_open_device
   ibv_alloc_pd
   ibv_reg_mr(buf)  ──► lkey/rkey
   ibv_create_cq
   ibv_create_qp    ──► QPN                                    …same…
        │
        │  state = RESET   (exists, cannot do anything)
        ▼
   modify_qp → INIT
        programs: port number, pkey index, access flags
        │         (now legal to post RECVs, not to send)
        ▼
   ┌───────────────── EXCHANGE ────────────────┐
   │  QPN, PSN, LID (IB) or GID (RoCE),        │
   │  rkey, virtual address, MTU               │◀──── the ONLY thing
   └───────────────────┬───────────────────────┘      that needs an
        │                                              out-of-band channel
        ▼
   modify_qp → RTR   ("ready to receive")
        programs: path MTU, destination QPN, remote PSN,
                  address vector (LID or GID + hop limit + traffic class),
                  max_dest_rd_atomic, min_rnr_timer
        ▼
   modify_qp → RTS   ("ready to send")
        programs: local PSN, TIMEOUT, RETRY_CNT, RNR_RETRY,
                  max_rd_atomic
        │
        ▼
   ═══════════ steady state: post WQE, ring doorbell, reap CQE ═══════════
        │
        ├─ error (protection violation, retry exhausted, remote down)
        ▼
      ERR   — all outstanding WQEs flush with error status;
              the QP must be reset and rebuilt.  There is no
              "reconnect" in the TCP sense: a broken RC QP is dead.
```

The values in the RTS transition are the ones to memorise, because they are the fabric's failure-recovery policy expressed as three integers. NCCL v2.31.2 sets them explicitly (`src/transport/net_ib/connect.cc`): `qpAttr.timeout = NCCL_IB_TIMEOUT` (**default 20**), `qpAttr.retry_cnt = NCCL_IB_RETRY_CNT` (**default 7**), `qpAttr.rnr_retry = 7`, `qpAttr.max_rd_atomic = 1`. What they mean:

- **`timeout`** is an exponent, not a duration. The IBTA-defined local ACK timeout is `4.096 µs × 2^timeout`. At the default of 20 that is `4.096 µs × 1,048,576 ≈ **4.3 seconds**` before the NIC decides an unacknowledged packet is lost. That number is deliberately enormous, and the reason is the subject of 09.4: on a properly lossless fabric a missing ACK means something is badly wrong, not that a packet was dropped in passing, so the transport waits rather than retransmitting into congestion.
- **`retry_cnt`** is how many times it will retry after that timeout before failing the QP — 7 is the maximum. So a totally dead path takes roughly `7 × 4.3 s ≈ 30 s` to surface as an error, which is exactly why a job with a failed link often appears to hang before it fails.
- **`rnr_retry`** governs "receiver not ready" — the two-sided case where a SEND arrives and no RECV work request was posted. 7 means "retry indefinitely" on most hardware.

**The operational consequence is worth stating plainly:** these defaults are tuned for a fabric that does not drop packets. On a fabric that does, they turn a lossy link into a multi-second stall rather than a fast failure, and every GPU in the job waits.

### 7. Transport services: what RC buys and what it costs

The QP's transport service type decides reliability, connection model, and which operations are legal. perftest's own option list names the full set — `RC`, `UC`, `UD`, `XRC`, `DC`, `SRD` — of which these matter for GPU collectives:

| Service | Connected? | Reliable / ordered? | One-sided WRITE/READ? | State cost | Used for |
|---|---|---|---|---|---|
| **RC** (reliable connected) | yes, one QP per peer | yes, in hardware | yes | O(peers) QPs per process | **the default for large-message GPU collectives** |
| **UC** (unreliable connected) | yes | no retransmission | WRITE only | O(peers) | niche; loss-tolerant bulk |
| **UD** (unreliable datagram) | no, one QP to many | no | no | O(1) | discovery, control, small messages |
| **XRC** (extended reliable connected) | yes, shared receive across processes | yes | yes | reduced QP count per node | many-process-per-node MPI |
| **DC** (dynamically connected) | connection established per message in hardware | yes | yes | O(1) contexts | large-scale fabrics where O(peers) QPs is prohibitive |

**RC is what "replaces TCP" cleanly.** Everything TCP did in kernel software — sequencing, retransmission, acknowledgement, connection state — RC does in the NIC ASIC. You keep TCP's guarantees without TCP's CPU tax and without its per-packet software path.

Two consequences of the "one QP per peer" property are worth holding. First, **QP state scales with the number of peers**, which is fine for a collective's structured, few-neighbours communication pattern and is why XRC and DC exist for topologies where it isn't. Second — and this is the link back to 09.2 §6 — **one QP means one ECMP path**, because the source UDP port that provides hash entropy on a RoCE fabric is chosen per QP. The transport-level choice and the fabric-level load balancing are the same decision seen from two ends.

### 8. What RC does *not* do — the seam into 09.4

RC guarantees that packets which arrive are delivered exactly once, in order, and acknowledged. **It does nothing to stop the fabric from dropping packets in the first place**, and its recovery behaviour when that happens is where the entire next lesson comes from.

The recovery mechanism in classic RDMA transports is a **go-back-N** retransmission: on detecting a gap the receiver NAKs, and the sender resumes from the missing packet, discarding and re-sending everything after it. That is already brutal at 400 Gb/s, because "everything after it" is a large window of in-flight data.

**The correction worth knowing, because it is frequently mis-stated:** the NICs in Microsoft's production RoCEv2 deployment implemented **go-back-0**, not go-back-N — on a NAK the sender restarted the *entire message* from packet 0. Their published experiment is the clearest possible demonstration of why that matters. They configured a switch to drop any packet whose IP ID had a least-significant byte of a fixed value, giving a deterministic **1-in-256 drop rate**, and ran 4 MB RDMA transfers (about 4,000 packets each). Application goodput went to **zero**: with one packet in every 256 dropped, one of each message's first 256 packets was always lost, the sender always restarted from packet 0, and the link ran at line rate forever while making no progress. They call this **RDMA transport livelock**, and the fix was to work with the vendor to implement go-back-N.

Three things to take from that:

1. **A tiny loss rate is not a small slowdown for RDMA; it can be total collapse.** TCP was designed around loss (fast retransmit, SACK, a congestion window that treats loss as signal). RDMA transports were designed around its absence.
2. **The recovery semantics are a NIC property, and they have changed.** "RDMA uses go-back-N" is a claim about a particular generation; newer hardware adds selective retransmission. Say which behaviour you mean.
3. **This is *why* the PFC/ECN/DCQCN edifice exists.** Not to make the fabric faster — to drive the congestion-drop probability toward zero so that recovery almost never triggers. That is 09.4 in one sentence, and it is a much better answer than "RoCE needs lossless Ethernet because RDMA is sensitive to loss."

### 9. GPUDirect RDMA: the GPU as a first-class peer, and its ordering hazard

**GPUDirect RDMA** is what makes RDMA specifically valuable in a GPU cluster: the GPU's HBM is registered as a memory region (via DMA-BUF on current stacks), so the HCA issues PCIe reads and writes directly against GPU memory and the bytes never touch host DRAM. The path collapses from `HBM → DRAM → NIC → wire → NIC → DRAM → HBM` to `HBM → NIC → wire → NIC → HBM`, deleting two full PCIe traversals and two staging copies per transfer.

It is **not automatic just because RDMA is present**. It carries the 02b same-root-complex precondition, extended fabric-wide as "same rail" in 09.1 — and the enforcement is explicit in the collective library. NCCL enables GDR only when the GPU-to-NIC path type is `PXB` or closer (`PATH_P2C` on C2C-capable systems), and prints `NET/IB : GPU Direct RDMA Enabled|Disabled for HCA n` at init to tell you which it chose. A GPU whose only NIC is across the root complex silently gets the staged path, and 09.1 §5 computed what that costs: roughly 22 GB/s instead of 46–48 GB/s, paid by every rank in the job because a collective is a barrier.

There is also a genuine ordering hazard that explains an otherwise mysterious knob. PCIe writes into GPU memory are **posted** — the NIC considers them complete when they are issued, not when they are visible to the GPU's SMs. So a receiver that is told "the data has landed" (by a completion, or by a flag the sender WROTE last) can in principle read the flag before the payload writes have drained. The remedy is a **flush**: after the completion, the receiving side issues a small read that forces the earlier posted writes to complete before the read returns. NCCL implements exactly this — it registers a small host-memory region for the purpose and performs a flush read on the GDR path — and exposes `NCCL_GDR_FLUSH_DISABLE` to turn it off on hardware where it is unnecessary. If you ever meet a bug where a GPU reads a stale buffer immediately after an RDMA completion, this is the mechanism to reason about, and it is a good example of the general rule that **zero-copy moves ordering responsibility to you**.

### 10. Reading the tools: what a healthy RDMA endpoint looks like

Four commands cover almost everything you will be asked to check. The formats below are taken from the tools' own source (`rdma-core`'s `infiniband-diags/ibstat.c` and `libibverbs/examples/devinfo.c`, `mlnx-tools`' `show_gids`, and `perftest`'s `src/perftest_parameters.c`); the values are illustrative rather than captured from a specific machine.

**(a) `ibstat` — is the port up, at what rate, and on which link layer?**

```
$ ibstat mlx5_0
CA 'mlx5_0'
	CA type: MT4129
	Number of ports: 1
	Firmware version: 28.39.1002
	Hardware version: 0
	Node GUID: 0xa088c20300f1a2b4
	System image GUID: 0xa088c20300f1a2b4
	Port 1:
		State: Active
		Physical state: LinkUp
		Rate: 400
		Base lid: 26
		LMC: 0
		SM lid: 3
		Capability mask: 0xa651e848
		Port GUID: 0xa088c20300f1a2b4
		Link layer: InfiniBand
```

- **`State: Active` / `Physical state: LinkUp`** — the two-level check. `LinkUp` with `State: Initializing` is the classic "cable is fine, the subnet manager has not configured this port" signature: the physical link trained, but no SM assigned it a LID. On a fabric with no running SM every port sits in `Initializing` forever.
- **`Rate: 400`** — the port rate in Gb/s, computed as *lane rate × width*. The full generation table is 09.4's material; here, note that a port reading `Rate: 200` on hardware you bought as 400G means it trained at half width or a lower generation, and the fabric will not tell you twice.
- **`Base lid` / `SM lid`** — the local identifier this port was assigned, and the LID of the subnet manager that assigned it. Both are InfiniBand concepts; on RoCE they are absent and addressing uses GIDs instead.
- **`Link layer: InfiniBand` vs `Ethernet`** — the single most important line, because it tells you which of the next two lessons applies to this port. `Ethernet` means RoCE, which means GIDs, DSCP/PFC configuration, and everything in 09.4.

**(b) `show_gids` — which GID index should the job use?** On RoCE this is a real operational question, because a port has several GIDs (link-local and global, RoCEv1 and RoCEv2, IPv4-mapped and IPv6) and choosing the wrong one produces a connection that fails or a path that is not routable.

```
$ show_gids
DEV     PORT    INDEX   GID                                     IPv4            VER     DEV
---     ----    -----   ---                                     ------------    ---     ---
mlx5_0  1       0       fe80:0000:0000:0000:a288:c2ff:fe03:f1a2                 v1      ibp13s0f0
mlx5_0  1       1       fe80:0000:0000:0000:a288:c2ff:fe03:f1a2                 v2      ibp13s0f0
mlx5_0  1       2       0000:0000:0000:0000:0000:ffff:0a1f:0402  10.31.4.2       v1      ibp13s0f0
mlx5_0  1       3       0000:0000:0000:0000:0000:ffff:0a1f:0402  10.31.4.2       v2      ibp13s0f0
n_gids_found=4
```

Read it as: indices 0/1 are the link-local GID derived from the MAC (not routable off-subnet); indices 2/3 are the IPv4-mapped GID carrying the interface's actual address; the `VER` column distinguishes **RoCEv1** (raw Ethernet, layer 2 only) from **RoCEv2** (UDP/IP, routable — the one you want). So on this port the correct choice for a routed fabric is **index 3**. That is why `NCCL_IB_GID_INDEX` exists, and why its NCCL default is `-1`, meaning "detect it" — NCCL also defaults `NCCL_IB_ROCE_VERSION_NUM` to `2` so that its detection prefers a RoCEv2 GID. On a fabric where autodetection picks wrong, pinning the index is the fix, and this table is how you learn which one to pin.

**(c) `ibv_devinfo -v` — what does this HCA support?** The verbose form prints the device capabilities and per-port attributes: `max_qp`, `max_qp_wr`, `max_cq`, `max_mr`, `max_mr_size`, `max_sge`, `active_mtu`, `max_mtu`, `active_speed`, `active_width`, `phys_state`, plus the GID table. The two fields worth checking habitually are **`active_mtu`** (a small path MTU inflates per-packet overhead and message rate requirements) and **`max_qp_wr`** (the send-queue depth ceiling, which bounds how much you can keep in flight).

**(d) `ib_write_bw` — is the path actually delivering?** This is the ground-truth bandwidth test, and the transcript below uses perftest's exact banner and result formats:

```
$ ib_write_bw -d mlx5_0 -x 3 -s 65536 -F --report_gbits 10.31.4.7
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF		Device         : mlx5_0
 Number of qps   : 1		Transport type : IB
 Connection type : RC		Using SRQ      : OFF
 PCIe relax order: ON		Lock-free      : OFF
 ibv_wr* API     : ON		Using Enhanced Reorder      : OFF
 TX depth        : 128
 CQ Moderation   : 100
 Mtu             : 4096[B]
 Link type       : Ethernet
 GID index       : 3
 Max inline data : 0[B]
 rdma_cm QPs	 : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0x0000 QPN 0x0113 PSN 0x9c4e2d RKey 0x1803ee VAddr 0x007f2a4c000000
 remote address: LID 0x0000 QPN 0x0129 PSN 0x3b17aa RKey 0x1804f2 VAddr 0x007f9b18000000
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 65536      5000             389.42             388.77             0.741456
---------------------------------------------------------------------------------------
```

Line by line, because every one of these fields is a thing that can be wrong:

- **`Number of qps : 1` and `Connection type : RC`** — one queue pair, reliable connected. Remember 09.2 §6: one QP is one ECMP path, so a single-QP benchmark measures one path's health, not the fabric's aggregate capacity. To exercise multiple paths, raise `-q`.
- **`Mtu : 4096[B]`** — the negotiated path MTU, the maximum of the `ibv_mtu` enum. A port showing 1024 here on a fabric you believe is configured for 4 KB has an MTU mismatch somewhere, and the cost is a 4× increase in packets for the same bytes.
- **`Link type : Ethernet` with `GID index : 3`** — this is RoCE, using the RoCEv2 IPv4-mapped GID from the `show_gids` table above. On an InfiniBand port these read `IB` and the GID line is absent, with `LID` populated instead.
- **`CQ Moderation : 100`** — one completion per 100 work requests. This is a bandwidth test; a latency test would use 1.
- **`TX depth : 128`** — how many work requests may be outstanding. Too small a depth cannot keep a 400G pipe full: at 64 KB per request, 128 outstanding requests is 8 MB in flight, which is ample here but would not be at 800G with a longer RTT.
- **The result row.** 388.77 Gb/s average on a 400 Gb/s link is **97% of line rate**, which is what a healthy NDR path with GPUDirect and a 4 KB MTU should look like. Convert and sanity-check: `388.77 Gb/s ÷ 8 = 48.6 GB/s`; `48.6 GB/s ÷ 65,536 B = 741,000 messages/s = 0.741 Mpps`, matching the `MsgRate` column exactly — a useful arithmetic check that the numbers are self-consistent.
- **What a *bad* run looks like:** ~50% of line rate suggests a PCIe link at half width (check `LnkSta`, 02b.3) or a single-lane path issue; ~25 GB/s on a 400G link is the fingerprint of a lost GPUDirect path staging through host memory (09.1 §5); and a low `MsgRate` with small `-s` values points at completion or doorbell overhead rather than at the wire.

### 11. The numbers to anchor

Treat all of these as ConnectX-7/Quantum-2-generation figures, and state the generation whenever you quote them.

| Quantity | Value | Provenance |
|---|---|---|
| RDMA half-round-trip, small message | ~1–2 µs | generation-typical; `ib_write_lat`/`ib_send_lat` measure it directly |
| InfiniBand switch hop | ~100–130 ns | vendor-published switch latency |
| Achieved bandwidth, 400G NDR, large messages | ~46–48 GB/s (≈385–390 Gb/s), ~97% of line | `ib_write_bw`-class measurement |
| Host CPU on the RDMA data path | ≈0% for large transfers; one core if busy-polling completions | polling is a *choice*, not a requirement |
| Host CPU for TCP at 40 Gb/s, 8 connections | 6% of a 32-core Xeon E5-2690 sending, 12% receiving | measured, Microsoft SIGCOMM 2016 |
| Payload copies, GPU→GPU | TCP: 4 (2 socket buffers + 2 GPU↔DRAM staging). RDMA + GPUDirect: 0 | §5 diagram |
| Local ACK timeout at NCCL defaults | `4.096 µs × 2^20` ≈ 4.3 s, up to 7 retries | NCCL v2.31.2 `IB_TIMEOUT`=20, `IB_RETRY_CNT`=7 |

## Perspectives

**Developer.** RDMA is invisible from training code — you call `all_reduce()` and never see a queue pair. It surfaces in exactly two ways: as an init-time log line proving which HCA and which path each rank chose, and as a throughput cliff when that choice was wrong. The habit worth building is reading the `NCCL_DEBUG=INFO` network block once per new SKU and confirming `GPU Direct RDMA Enabled` for every rank, because a silent fallback to host staging looks like "the cluster is a bit slow today" for weeks.

**Operator.** Registered memory is pinned and finite, and it is a resource class most platform teams do not monitor until it bites. Every job's registered buffers subtract from the same physical-memory budget on a shared node; pinning pressure can force a job into smaller buffer pools, or fail registration outright. Add pinned/locked memory to the same dashboard as GPU memory, and remember that registration is a *setup-time* cost — a job that registers on the hot path has reintroduced the syscall RDMA exists to remove.

**Kernel / hardware.** The object model is a security architecture, not an API convenience. Handing userspace direct DMA access to a shared NIC without isolation would be a correctness and security disaster; the protection domain plus MR keys are what make it safe, enforced per work-queue entry in silicon against state the kernel programmed once. When you are asked "how is this safe?", the answer is the `rkey`: a capability token scoped to one region, one protection domain, and one set of access rights.

**Economics.** Two lines on the same balance sheet. RDMA gives back the CPU cores that TCP would spend on copies and interrupts — at 400 Gb/s per rail NIC that is a material fraction of a socket per node, and those cores are exactly what the input pipeline needs. And it flattens the tail, which at barrier-synchronous scale converts directly into GPU-hours: a few points of collective efficiency on a nine-figure cluster is a large absolute number, every day it runs.

## Real-world use cases

- **Microsoft, RDMA over Commodity Ethernet at Scale (SIGCOMM 2016).** [Paper PDF](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/11/rdma_sigcomm2016.pdf) — read directly for this lesson. Substance used here: the measured TCP CPU cost (6% of a 32-core Xeon E5-2690 sending at 40 Gb/s over 8 connections, 12% receiving); the **RDMA transport livelock** experiment (a deterministic 1-in-256 drop rate on 4 MB messages, ~4,000 packets each, produced *zero* application goodput at full line rate) and its root cause, **go-back-0** loss recovery in the NIC, fixed by moving to go-back-N. This is the single best public demonstration that RDMA's loss sensitivity is qualitative, not quantitative.
- **NCCL as the production consumer.** The NCCL source (v2.31.2) is itself a worked example of everything in §§3–9: RC queue pairs (`IBV_QPT_RC`), explicit access-flag choices per registration, `NCCL_IB_TIMEOUT`/`NCCL_IB_RETRY_CNT` on the RTS transition, a GDR flush region with `NCCL_GDR_FLUSH_DISABLE`, and PCIe relaxed ordering under `NCCL_IB_PCI_RELAXED_ORDERING`. Reading `src/transport/net_ib/` is the fastest way to see which verbs features a real collective library actually depends on.
- **CoreWeave — GPUDirect RDMA on Kubernetes.** [Use GPUDirect RDMA with InfiniBand](https://docs.coreweave.com/docs/products/networking/hpc-interconnect/use-gpudirect-rdma) — what it shows: the operational form of this lesson's object model, from requesting an RDMA device as a schedulable resource to the environment variables that select the HCA. The bridge from "what is a queue pair" to "what goes in a pod spec," which 09.6 develops.

## Worked example — trace one 1 MB GPU→GPU transfer, both ways, then price it

**Setup.** A GPU on host A must deliver a 1 MB gradient shard to a GPU on host B — one leg of a rail-local all-reduce on the 400G fabric from 09.1. Trace every stage, count what each path costs, and then use the counts to explain a measurement.

**(a) Over TCP, with no GPUDirect — the 01b datapath in full.**

1. **Copy GPU → host DRAM.** `cudaMemcpy` moves 1 MB out of HBM into a pinned host staging buffer over PCIe. *Payload copy #1.*
2. **`send()` syscall.** Mode switch, socket lock, argument validation.
3. **Copy into the socket send buffer.** The kernel copies the payload into `sk_buff` pages. *Payload copy #2.*
4. **TCP/IP.** Segmentation (or TSO deferral), sequence numbers, checksums, congestion-window bookkeeping — all CPU, all per packet.
5. **netfilter + conntrack.** Flow-tuple lookup and state update, plus any NAT or overlay hooks. In a Kubernetes environment this is not optional and not cheap.
6. **qdisc → driver → NIC ring.** Enqueue, dequeue, descriptor write, doorbell; the NIC DMAs the segments out.
7. **Wire → NIC on host B**, subject to switch queueing and, if the fabric drops, retransmission.
8. **IRQ → softirq `NET_RX` on B.** The receive path drains the ring under a per-core budget shared with every other flow on that core.
9. **IP/TCP reassembly, ACK generation, conntrack lookup** on B.
10. **Copy into the socket receive buffer**, then **copy to userspace on `recv()`**. *Payload copies #3 and #4.*
11. **Copy host DRAM → GPU B** over PCIe. *Payload copy #5 — or #4 if you count the two staging copies as one pair.*

**Total: 4 payload copies through host memory plus 2 PCIe staging traversals, 2 syscalls, 2 conntrack touches, 1 softirq you do not schedule, and a full software transport on both ends.** The mean cost of that is bad. The *variance* is worse, and it comes from exactly three stages: conntrack, the softirq budget, and socket-buffer contention.

**(b) Over RDMA + GPUDirect.**

*Setup, once — not per message:*

- `ibv_open_device` / `ibv_alloc_pd` on both hosts.
- `ibv_reg_mr` over the GPU HBM regions (via DMA-BUF), returning `lkey` and `rkey`. Pages pinned, NIC translation programmed.
- `ibv_create_cq`, `ibv_create_qp` (type `IBV_QPT_RC`).
- Exchange `{QPN, PSN, GID or LID, rkey, virtual address}` over a one-time out-of-band channel.
- `modify_qp` through `INIT → RTR → RTS`, programming path MTU, peer identity, and the timeout/retry policy from §6.

*Per 1 MB transfer:*

1. Host A builds one **WQE** — opcode `RDMA_WRITE`, an SGE pointing at the GPU buffer with its `lkey`, and the peer's remote address plus `rkey` — and **rings the doorbell**. No syscall.
2. **HCA A DMA-reads the 1 MB directly out of GPU A's HBM** (peer-to-peer PCIe through the switch crossbar), segments it to the 4 KB path MTU (256 packets), stamps sequence numbers, and transmits.
3. **HCA B validates the `rkey`** and DMA-writes the payload straight into GPU B's HBM. It acknowledges in hardware.
4. **HCA A posts a CQE**; A's thread reaps it by polling (or by an armed event). With CQ moderation, one CQE may cover many WQEs.

**What was deleted, named explicitly:** both syscalls; all four host-memory payload copies *and* both GPU↔DRAM staging copies; the entire kernel TCP/IP stack; netfilter and conntrack on both ends (there is no flow tuple to match — which is why RDMA and stateful pod networking are in structural tension); the softirq receive path; and the remote CPU entirely. Host B was not interrupted, did not copy anything, and may not know the write happened until it reads its own memory.

**(c) Put numbers on it.** Take the measurement from §10(d) — 388.77 Gb/s average, 48.6 GB/s — and check that the model explains it:

```
   1 MB at 48.6 GB/s of achieved bandwidth        = 20.6 µs of wire time
   plus one half-RTT of latency (~1-2 µs)         ≈ 22 µs total
   packets at 4 KB MTU: 1,048,576 / 4,096         = 256 packets
   host CPU consumed:  one WQE post + one CQE reap ≈ hundreds of ns

   TCP equivalent, same wire, no GPUDirect:
   wire time is similar, but add
     • 4 × 1 MB of memcpy through host memory
     • 2 PCIe staging traversals (~2 × 1 MB at ~54 GB/s ≈ 39 µs)
     • per-packet CPU on both ends
     • conntrack + softirq, whose cost depends on the rest of the machine
   → the mean roughly doubles or worse; the p99.9 is unbounded by anything
     you control.
```

**(d) Now use it to diagnose.** Three observed results on the same 400G link, and what each one means:

| Observation | Diagnosis | Where to look |
|---|---|---|
| `ib_write_bw` = 388 Gb/s, NCCL inter-node bus bandwidth ≈ 45 GB/s | healthy | nothing to do |
| `ib_write_bw` = 388 Gb/s, but NCCL reports ≈ 22 GB/s | the *link* is fine; the *GPU-to-NIC path* is not — GPUDirect is off and traffic is staging through host DRAM | `NET/IB : GPU Direct RDMA Disabled for HCA n` in the NCCL init log; `topo -m` for a `PHB`/`NODE`/`SYS` pairing (09.1 §5) |
| `ib_write_bw` = 195 Gb/s | the link itself is at half capacity — PCIe trained at x8, or the port at half width | `lspci -vv` `LnkSta` (02b.3); `ibstat` `Rate:` |
| `ib_write_bw` fine at 64 KB, collapses at 256 B | message-rate bound, not bandwidth bound: per-WQE and per-CQE overhead dominates | raise CQ moderation, batch work requests, or use inline sends |

That table is the practical payoff of the whole lesson: **each symptom maps to a specific stage of the mechanism**, and you can name the stage rather than guessing.

## Practice (feeds the deliverable)

Produce the **TCP-vs-RDMA datapath trace** for the Network Architecture Read deliverable.

**Task.** Trace the same 1 MB GPU→GPU transfer twice — once over TCP, once over RDMA with GPUDirect — for your own reference cluster, and then attach the evidence that proves which path your cluster is actually using.

**Requirements / acceptance:**

1. **Both paths trace the same transfer**, so they are comparable, and each stage is numbered.
2. **Every TCP hot-path stage from 01b is named** and a reviewer can check them off: `send()`/`recv()` syscalls, both socket-buffer copies, both GPU↔host staging copies, the TCP/IP stack, netfilter/conntrack, the softirq receive path, and the NIC DMA.
3. **The RDMA trace separates setup from data path.** Setup: device open, protection domain, `ibv_reg_mr` (state what it pins and what keys it mints), CQ and QP creation, the out-of-band exchange, and the `INIT → RTR → RTS` transitions. Data path: WQE post, doorbell, NIC DMA out of HBM, segmentation to the path MTU, hardware ACK, CQE.
4. **State explicitly what the remote CPU does** in the one-sided WRITE case, and why that is structurally impossible for TCP.
5. **Name at least one stability consequence**, not just a throughput number: pick a stage whose *tail* latency is load-dependent under TCP and absent under RDMA, and say why it matters for a barrier-synchronous collective.
6. **Attach real evidence from the machines you can reach**: `ibstat` for one HCA (annotate `State`, `Physical state`, `Rate`, `Link layer`), `show_gids` if the port is Ethernet (say which GID index a RoCEv2 job should use and why), and an `ib_write_bw` run if you have two endpoints — or an `ib_write_bw` loopback plus a note about what a two-node run would add.
7. **State the retransmission policy your stack is configured with**: the QP `timeout` and `retry_cnt` values in force (NCCL's defaults are 20 and 7), convert `timeout` into seconds using `4.096 µs × 2^timeout`, and say what that implies for how a link failure surfaces to the job.

Keep it to one page plus transcripts. This trace is the backbone reviewers use to check you understand mechanism rather than vocabulary.

## Common pitfalls

- **"Kernel bypass means the kernel is never involved."** False, and it is the single most common RDMA misconception. Device open, protection-domain allocation, memory registration and queue-pair creation are all kernel operations — they pin pages, program NIC translation tables and allocate device resources. Bypass is a property of the *steady-state, per-message* path after a one-time setup phase that goes straight through the kernel.
- **Registering memory on the hot path.** Registration pins pages and programs hardware tables; doing it per message reintroduces exactly the syscall and page-walk costs RDMA exists to remove, and it churns a finite node-wide resource. Register a pool at startup and reuse it. If you genuinely cannot know the buffers in advance, the mechanisms to reach for are on-demand paging or a registration cache, not per-message `ibv_reg_mr`.
- **Assuming GPUDirect RDMA works whenever RDMA works.** It carries an additional precondition — the GPU and NIC must be close enough in the PCIe tree — and the collective library enforces it with a threshold (NCCL: `PXB` or closer). A cross-root-complex pairing silently falls back to host staging, costs roughly half the bandwidth, and raises no error. The proof is one log line, not an assumption.
- **Conflating RC's reliability with a lossless fabric.** RC guarantees that packets which arrive are delivered once, in order, and acknowledged — a transport property enforced in the NIC. It says nothing about whether the fabric drops packets, and its recovery from a drop is coarse: classic implementations restart from the beginning of the message (go-back-0) or from the lost packet (go-back-N), either of which is a throughput cliff rather than a graceful degradation. Losslessness is a separate, link-layer property, and it is 09.4's entire subject.
- **Quoting "RDMA uses go-back-N" as a timeless fact.** Microsoft's production NICs used **go-back-0**, which is precisely why a 1-in-256 drop rate produced *zero* goodput rather than a modest slowdown; go-back-N was the fix, and newer hardware adds selective retransmission. Always say which generation and which behaviour you mean.
- **Forgetting the ordering hazard of zero-copy into GPU memory.** PCIe writes are posted, so a completion or a flag can become visible before the payload has drained into HBM. That is why a flush read exists on the GDR path (`NCCL_GDR_FLUSH_DISABLE` controls it). Zero-copy does not remove ordering work; it moves the responsibility to the application.
- **Benchmarking one queue pair and calling it "the fabric."** `ib_write_bw` with `-q 1` measures one QP, which on a RoCE fabric is one ECMP path (09.2 §6). A healthy single-path result tells you the link and the endpoints are fine; it tells you nothing about whether the fabric can spread a real collective across its uplinks.
- **Treating the huge default timeout as a bug.** `4.096 µs × 2^20 ≈ 4.3 s` with 7 retries looks absurd next to a TCP RTO until you remember it was chosen for a fabric where a missing ACK means something is broken, not congested. The right response to "our job hangs for 30 seconds then fails" is to find the broken link, not to shrink the timer — though knowing that the timer is why it hangs for 30 seconds is what lets you say that.

## Self-check

**(a) During an RDMA WRITE, what does the host CPU do — and what does it not do?**

**Answer:** On the **sender**, the CPU does exactly two small things: it writes a work-queue entry describing the transfer (opcode, scatter-gather list with `lkey`, remote address with `rkey`) and rings the NIC's doorbell with a store to a mapped register. Later it reaps a completion-queue entry to learn the write landed. It does **not** copy the payload (the HCA DMAs it straight from the registered buffer, which with GPUDirect is GPU HBM), does **not** run a transport stack (sequencing, retransmission and acknowledgement are in the ASIC), and does **not** enter the kernel per message. On the **receiver**, the CPU does *nothing at all* for a one-sided WRITE — no interrupt, no softirq, no copy, no work request consumed; the bytes simply appear in its memory. That property is what "one-sided" means, and TCP cannot offer it because the receiving kernel is definitionally in the path.

**(b) Why is RDMA latency stable in a way a conntrack'd TCP path is not?**

**Answer:** Because the stages RDMA deletes are precisely the *stateful, shared, load-dependent* ones. Under TCP the tail comes from `nf_conntrack` (hash lookups that miss cache and take locks, worse as connection count and traffic rise), the per-core softirq budget (your receive processing competes with every other flow on that core and can be deferred), and socket-buffer contention. Their cost is a function of what else the machine is doing, which is why they blow up exactly when the fabric is busy — mid-collective. What RDMA leaves behind is a WQE, a DMA, a wire crossing and a hardware ACK, all fixed-cost silicon whose latency does not grow with host load. Since a collective is a barrier, the step time is set by the tail rather than the mean, so removing the variance is worth more than removing the average.

**(c) What is a queue pair, and why does kernel bypass require one?**

**Answer:** A queue pair is the RDMA connection object: a send queue and a receive queue in NIC-visible memory, plus the per-connection state — transport service type (RC, UD, …), the peer's queue-pair number, path MTU, addressing, retransmission timers, and the protection domain that scopes which memory regions its work requests may name. Bypass requires it because removing the kernel from the datapath removes the arbiter that used to multiplex the NIC across processes, enforce memory isolation and hold connection state. The QP restores all three in hardware: the kernel programs it once at setup, binding it to a protection domain, and thereafter the NIC checks every work-queue entry against that state — a process's WQEs can only reference its own registered memory via `lkey`, and a peer can only write memory whose `rkey` it was explicitly given. Without a QP there is no safe way for userspace to drive a shared NIC directly.

**(d) What is the difference between "kernel bypass" and "the kernel is never involved"?**

**Answer:** They are different claims and interviewers use the gap to separate mechanism from marketing. Kernel bypass describes the steady-state data path: once a QP exists and memory is registered, each message goes from userspace to the NIC through a doorbell write with no syscall. "The kernel is never involved" is false: `ibv_open_device`, `ibv_alloc_pd`, `ibv_reg_mr` (which pins pages and programs the HCA's translation and protection tables) and `ibv_create_qp`/`create_cq` are all kernel operations, performed once at setup, and they are expensive — which is exactly why they are done once. The correct framing is that RDMA relocates all stateful, kernel-touching work into a setup phase and leaves the per-message path as silicon.

**(e) A switch drops one packet in 256. What happens to an RDMA transfer, and why is the answer not "it retransmits that packet"?**

**Answer:** It depends on the NIC's loss-recovery semantics, and the classic answer is much worse than a single retransmission. Microsoft's production RoCEv2 NICs implemented **go-back-0**: on a NAK the sender restarted the entire message from packet 0. With 4 MB messages (about 4,000 packets at their MTU) and a deterministic 1-in-256 drop rate, one of the first 256 packets was always lost, so the sender restarted forever and application goodput was **zero** while the link ran at line rate — a livelock, not a slowdown. Go-back-N (retransmit from the lost packet onward) is the improved behaviour and is still a throughput cliff at 400 Gb/s, because "onward" is a large in-flight window. The general point: RDMA transports were designed on the assumption that congestion loss does not happen, so their recovery is coarse. That assumption is what 09.4's PFC/ECN/DCQCN machinery exists to make true.

**(f) What does memory registration actually do, and why is registered memory scarce?**

**Answer:** Three things. It **pins** the pages so they cannot be swapped, migrated or freed while the NIC is DMA-ing to them asynchronously. It **programs the HCA's address-translation and protection tables** so the NIC can resolve a virtual address in a work request without CPU help and enforce the access rights attached to that region. And it **mints the keys** — an `lkey` that local work requests must present and an `rkey` that a remote peer must present in a one-sided operation, making the `rkey` a capability token for exactly that region with exactly those rights. It is scarce because pinning removes pages from the pageable pool: on a shared node every job's registered buffers subtract from the same physical-memory budget, so registration is a startup-time, pooled resource rather than a per-message allocation. On-demand paging relaxes the hard pin at the cost of datapath faults, and GPU memory is registered through DMA-BUF under the same pin-and-translate contract.

**(g) Your NCCL job runs at 22 GB/s per rank between nodes, but `ib_write_bw` on the same pair reports 388 Gb/s. What is wrong, and how do you confirm it?**

**Answer:** The link is healthy and the GPU-to-NIC path is not: 22 GB/s is the fingerprint of GPUDirect RDMA being disabled and the transfer staging through pinned host memory, capped by the root complex or the inter-socket link rather than by the wire. `ib_write_bw` does not catch it because by default it moves *host* memory, so it exercises the NIC and the fabric while bypassing the GPU-to-NIC question entirely. Confirm it in two places: the NCCL init log, which prints `NET/IB : GPU Direct RDMA Disabled for HCA n` for the affected device, and `nvidia-smi topo -m`, which will show that GPU's chosen NIC at `PHB`, `NODE` or `SYS` rather than `PIX`/`PXB` — past the threshold NCCL requires. The fix is placement or cabling, not tuning; NCCL's PXN path can recover much of the bandwidth by routing over NVLink to a GPU that owns a good NIC.

## Connections & what's next

This lesson is the mechanism layer beneath three others. **09.2's fabric shape** is what RDMA rides across, and §7's "one QP is one path" is the same fact as 09.2 §6's flow-entropy argument seen from the transport side. **09.5's GPUDirect and SHARP** extends §9's same-root-complex rule fabric-wide and then changes the problem entirely by doing the reduction in the switch instead of in a ring of RDMA WRITEs. **09.6's Kubernetes multi-NIC** is how a pod gets access to the object model described in §3 — and §5's "there is no flow tuple" is precisely why the default CNI, built around conntrack and overlays, cannot deliver it. Module 08's NCCL collectives are the caller of everything here: NCCL decides *what* to communicate and in what pattern, RDMA decides *how one point-to-point transfer physically happens* underneath that pattern.

The next lesson, **09.4 (InfiniBand vs RoCEv2 and engineered losslessness)**, is the direct sequel to §8. This lesson established that RC's reliability guarantee covers packets that arrive and says nothing about drops — and that recovery from a drop is a cliff rather than a graceful degradation. 09.4 is entirely about the upstream problem: how InfiniBand makes drops impossible by construction with credit-based link-level flow control, how RoCEv2 manufactures the same property on Ethernet with PFC and ECN/DCQCN, exactly how those mechanisms work at the buffer level, how they fail (head-of-line blocking, pause storms, deadlock), and how to choose between the two fabrics for a real procurement. Carry three things forward: the QP timeout and retry defaults, the go-back-0 livelock, and the fact that "reliable transport" and "lossless fabric" are different claims.

## References & further reading

**Source-access note for this pass.** Vendor and publisher domains (docs.nvidia.com, arxiv.org, dl.acm.org, IEEE) are blocked by this environment's egress proxy. The sources marked **verified** below were fetched and read directly while writing; entries marked **not re-verified in this pass** carry their attribution from the previous version of this lesson and should be treated as citations to check. The IBTA specification itself was not reachable, so IBTA-defined semantics (the `4.096 µs × 2^timeout` local-ACK-timeout formula, the transport service definitions) are stated as spec-defined and cross-checked against the verbs implementations named below rather than against the specification text.

**Verified against source in this pass**

- **rdma-core** (`github.com/linux-rdma/rdma-core`) — the userspace RDMA stack. Verified here: `enum ibv_mtu` (256/512/1024/2048/4096) in `libibverbs/verbs.h`; the exact `ibstat` output fields and their order (CA type, firmware/hardware version, node and system-image GUID; per port: State, Physical state, Rate, Base lid, LMC, SM lid, Capability mask, Port GUID, Link layer) in `infiniband-diags/ibstat.c`, including the port-state and physical-state string tables; the `ibv_devinfo` capability and GID printing in `libibverbs/examples/devinfo.c`. Read for: what the tools actually print, as opposed to what documentation says they print.
- **linux-rdma/perftest** (`github.com/linux-rdma/perftest`) — the `ib_*_bw` / `ib_*_lat` benchmarks. Verified here: the result header `#bytes #iterations BW peak[Gb/sec] BW average[Gb/sec] MsgRate[Mpps]` (`RESULT_FMT_G`), the banner fields and their tab layout, the address line format `local address: LID %#04x QPN %#06x PSN %#06x RKey %#08x VAddr %#016llx` (`BASIC_ADDR_FMT` and `RDMA_FMT` in `src/perftest_communication.h`), the supported connection types `RC/XRC/UC/UD/DC/SRD`, and defaults including `CQ Moderation = 100` and a default BW message size of 64 KB with 1,000 iterations. Read for: constructing a transcript whose shape is exactly right.
- **NCCL v2.31.2** (`github.com/NVIDIA/nccl`) — the production consumer of all of this. Verified here: `IBV_QPT_RC` queue pairs and the access-flag combinations used per registration in `src/transport/net_ib/connect.cc`; the RTS attributes `timeout` (from `NCCL_IB_TIMEOUT`, default **20**), `retry_cnt` (from `NCCL_IB_RETRY_CNT`, default **7**), `rnr_retry = 7`, `max_rd_atomic = 1`; `NCCL_IB_GID_INDEX` default `-1` and `NCCL_IB_ROCE_VERSION_NUM` default `2`; `NCCL_GDR_FLUSH_DISABLE`; `NCCL_IB_PCI_RELAXED_ORDERING` default `2`; and the GDR path-distance threshold (`PATH_PXB`, or `PATH_P2C` with C2C) plus the `GPU Direct RDMA Enabled|Disabled for HCA` log format in `src/graph/paths.cc` and `src/graph/topo.cc`.
- **mlnx-tools** (`show_gids`) — verified here: the script's column layout (`DEV PORT INDEX GID IPv4 VER DEV`), that it enumerates `/sys/class/infiniband/<dev>/ports/<port>/gids/*` and skips all-zero GIDs. Read for: reproducing the GID table an operator actually sees.
- **Guo et al., Microsoft, [RDMA over Commodity Ethernet at Scale](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/11/rdma_sigcomm2016.pdf), SIGCOMM 2016** — fetched and read directly. Verified here: the TCP CPU measurements (40 Gb/s over 8 connections costing 6% of a 32-core Xeon E5-2690's aggregate CPU on send and 12% on receive, with offloads enabled); the RDMA transport livelock experiment (switch configured to drop packets by IP ID for a deterministic 1-in-256 drop rate, 4 MB messages segmented into ~4,000 packets, zero application goodput) and its root cause in **go-back-0** recovery, fixed by moving to go-back-N; and RoCEv2's encapsulation with a fixed destination UDP port 4791 and a per-QP randomised source port.

**Cited, not re-verified in this pass**

- NVIDIA, [RDMA Aware Networks Programming User Manual](https://docs.nvidia.com/networking/display/rdmaawareprogrammingv17/rdma-aware+programming+overview) — the canonical narrative reference for the verbs object model (protection domains, memory regions, queue pairs, completion queues, work requests). Read for the model, not to memorise the C API.
- Meta / SIGCOMM 2024, [RDMA over Ethernet for Distributed Training at Meta Scale](https://engineering.fb.com/wp-content/uploads/2024/08/sigcomm24-final246.pdf) — cited for how RDMA's one-sided semantics interact with collective-library design at 24K+ GPU scale; the congestion-control substitution it describes is 09.4's material.
- CoreWeave, [Use GPUDirect RDMA with InfiniBand](https://docs.coreweave.com/docs/products/networking/hpc-interconnect/use-gpudirect-rdma) — the operational form: requesting an RDMA resource, image requirements, and the environment variables that select the device.

**Deeper dives**

- RDMAmojo (Dotan Barak) — long-form posts on individual verbs, queue-pair state transitions, and completion semantics; the best free reference when you need the detail behind one call. https://www.rdmamojo.com/
- kubenatives, [RDMA / InfiniBand for Kubernetes GPU networking](https://www.kubenatives.com/p/rdma-infiniband-kubernetes-gpu-networking) — ties the object model to the Kubernetes context, including why RDMA and conntrack-based pod networking are in tension. Not re-verified in this pass.
