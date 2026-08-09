---
lesson: "09.3"
title: "RDMA fundamentals: kernel bypass, zero-copy, queue pairs"
module: "09"
concept: "RDMA fundamentals: kernel bypass, zero-copy, queue pairs"
status: not-started
est_time: "5h"
artifacts: []
---
# 09.3 · RDMA fundamentals: kernel bypass, zero-copy, queue pairs

> **Concept.** RDMA is the deliberate deletion of the kernel datapath you learned in 01b — the NIC reads and writes application memory directly, so the CPU, socket buffers, netfilter, and conntrack are simply not on the wire.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Why this matters

At 40+ clusters you already know the interview question that separates a "networking person" from a platform engineer who runs GPU fabrics: *"Why can't we just train on TCP?"* The wrong answer is "TCP is slow." The right answer names mechanism — and the mechanism is that a synchronous collective (all-reduce over NCCL, module 08) is bottlenecked by the **slowest** and **most jittery** link in the ring, not the average one. A collective is a barrier: 511 GPUs wait for the 512th. TCP's tail latency — the p99.9 that spikes when a conntrack bucket rehashes or a softirq gets stolen by another core — becomes *your* step time. RDMA exists to make the tail flat.

Two dollars-and-cents consequences you will own:

- **Utilization is money.** A100/H100/B200 time is the single largest line item in a training budget. If TCP overhead and jitter drop fabric-bound collective efficiency from 95% to 70%, you are lighting ~25% of a nine-figure GPU spend on fire. RDMA is not a nicety; it is the thing that keeps `$/token` defensible.
- **CPU is not free either.** A single 400G NIC saturated with TCP can consume many host cores just doing copies and interrupt handling — cores you wanted for the data loader. RDMA gives them back.

This lesson is conceptual. You will not write verbs code. You will be able to stand at a whiteboard and trace exactly which of your 01b kernel stages RDMA removes, and why removing them buys latency *stability*, not just latency.

## What's new here (contrast with 01b)

In 01b you traced a packet through the Linux kernel datapath: `send()` syscall → copy into the socket send buffer (`sk_buff`) → TCP/IP stack builds segments → `netfilter` hooks (`nf_conntrack` creates/looks up a flow tuple, NAT, firewall) → qdisc → driver `ndo_start_xmit` → NIC DMA. On receive: NIC → softirq (`NET_RX`) → IP/TCP reassembly → conntrack lookup → socket receive buffer → copy to userspace on `recv()`. Every stage is CPU work, and several stages are **stateful and contended** (conntrack table, socket locks, per-core softirq budget).

**RDMA deletes that entire diagram.** Not optimizes — deletes. There is no `sk_buff`. There is no conntrack tuple. There is no softirq on the hot path. There is no copy into or out of a socket buffer. The RDMA-capable NIC (an "RNIC" — ConnectX-7, ConnectX-8, or a BlueField-3 DPU) has read/write access to registered regions of application (and, with GPUDirect RDMA, GPU HBM) memory, and it moves bytes from one machine's registered memory to another's **without the remote CPU being involved at all**.

The mental model flip:

| 01b kernel datapath (TCP) | RDMA |
|---|---|
| CPU orchestrates every packet | CPU posts a descriptor, then leaves |
| Kernel owns buffers; app copies in/out | NIC touches app/GPU memory directly (zero-copy) |
| Stateful middle: conntrack, NAT, firewall | Stateless on the wire; no flow table |
| Interrupt/softirq per batch of packets | Completion posted to a queue; poll or event |
| Transport (TCP) runs in kernel software | Transport (reliable delivery, ordering) runs in NIC hardware |

The phrase to memorize for interviews: **"RDMA is kernel bypass plus zero-copy plus transport offload."** Those are three distinct wins and a senior candidate separates them.

## Core notes

### 1. Kernel bypass — the control path leaves the kernel once

"Bypass" does not mean the kernel is never involved. Setup still goes through it: you open a device, **register a memory region (MR)** — which pins the pages so they can't be swapped and programs the NIC's on-board translation (MTT/MPT) so the NIC can translate virtual → physical and enforce access rights — and you create queues. This is expensive and done **once**, at connection/buffer-pool setup. After that, the *data path* is a direct userspace→NIC conversation via a memory-mapped "doorbell." The syscall (`send()`/`recv()`) that dominated your 01b trace is gone from steady state.

Consequence: no per-message trip across the syscall boundary, no scheduler interaction, no copy into kernel-owned memory.

### 2. Zero-copy — the bytes are never duplicated

In TCP the same 1 MB is copied at least twice: userspace → socket send buffer on the sender, socket receive buffer → userspace on the receiver (plus DMA to/from the NIC). With RDMA the NIC DMAs **straight out of** the sender's registered buffer and **straight into** the receiver's registered buffer. With **GPUDirect RDMA**, "registered buffer" is GPU HBM itself — the NIC reads/writes the GPU over PCIe/NVLink-adjacent paths and the bytes never touch host DRAM or a bounce buffer. This is why RDMA + GPUDirect is the substrate NCCL rides on for inter-node all-reduce.

### 3. Queue pairs (QP) — the connection abstraction that makes bypass safe

A **queue pair** is the unit of an RDMA connection: a **Send Queue (SQ)** and a **Receive Queue (RQ)**. You post **Work Requests (WRs / WQEs — work queue entries)** onto a queue: "WRITE these 1 MB from local MR X to remote address Y," or "here is a buffer, RECV into it." The NIC consumes WQEs and executes them.

Why does kernel-bypass *need* the QP? Because you have removed the kernel as the arbiter that used to (a) multiplex many connections onto one NIC, (b) enforce isolation between processes, and (c) hold the connection state. The QP is the hardware object that restores all three: it is per-connection state living in the NIC, bound to specific memory regions with specific access rights (keys — `lkey`/`rkey`), so one process's WQE physically cannot address another's memory. The kernel programs the QP once at setup; thereafter the NIC enforces the boundary the kernel used to. **No QP, no safe bypass** — you'd have userspace poking a shared NIC with no isolation.

QP transport service types matter for the interview: **RC (Reliable Connected)** — the NIC guarantees in-order, reliable, ACK'd delivery in hardware; this is what large-message collectives use. **UD (Unreliable Datagram)** and others exist but RC is the one that "replaces TCP's reliability, in silicon."

### 4. Completion queues (CQ) — asynchrony without interrupts

When the NIC finishes a WQE it posts a **Completion Queue Entry (CQE)** to a **Completion Queue (CQ)**. The CPU learns of completion by **polling** the CQ (busy-poll: lowest latency, burns a core) or by arming an **event** (a doorbell/interrupt when a CQE lands: lower CPU cost, slightly higher latency). Contrast 01b: TCP completion is a softirq you don't control, sharing a per-core budget with every other flow on that core. Here completion is a data structure *you* drain on *your* schedule.

### 5. The verbs (know the names, not the code)

The operations ("verbs"): **RDMA WRITE** (push into remote memory, remote CPU uninvolved — the workhorse), **RDMA READ** (pull from remote memory), **SEND/RECV** (two-sided: receiver must pre-post a RECV; used for control/handshake), and **atomics** (fetch-add, compare-swap). One-sided WRITE/READ are the "remote CPU does nothing" superpower; SEND/RECV is the two-sided fallback that looks most like sockets.

### 6. Memory registration — the cost you pay once, and why it exists

Registration (`ibv_reg_mr`) is not bookkeeping; it does two load-bearing things:

- **Pins the pages.** The NIC will DMA to physical addresses asynchronously, so the pages must not move or be swapped out underneath it. Pinning removes them from the pageable pool — which is why registered memory is a *scarce* resource you pool and reuse, not allocate per message. Registering on the hot path would reintroduce a syscall and a page-table walk, defeating the entire point.
- **Programs the NIC's translation and protection tables** (MTT/MPT): virtual→physical mappings plus access rights, keyed by `lkey` (local) and `rkey` (remote). The `rkey` is what a peer must present in a WRITE/READ WQE; it's the capability token that makes one-sided access safe.

For GPUDirect the registered region is GPU HBM (via `nv_peer_mem` / DMA-BUF), and the same pin+translate contract applies to device memory. **Interview-ready framing:** RDMA moves *all* the expensive, stateful, kernel-touching work — connection setup, memory pinning, address translation programming — to a **one-time setup phase**, leaving the per-message path as pure silicon.

### 7. RC vs UD — pick the transport that matches the message

- **RC (Reliable Connected):** hardware-guaranteed reliable, in-order, ACK'd delivery; supports one-sided WRITE/READ. One QP per peer, so state scales with the number of peers — fine for the structured, all-to-few communication of a collective. This is the default for large-message GPU collectives.
- **UD (Unreliable Datagram):** connectionless, one QP talks to many, but no reliability and small messages only — used for control/discovery, not bulk data.

The reason RC "replaces TCP" cleanly: everything TCP did in kernel software (retransmit, ordering, ACK, flow state) RC does in the NIC ASIC, so you keep TCP's guarantees without TCP's CPU tax or its drop-and-recover tail.

### 8. The numbers (anchor them)

- **Latency:** RDMA half-round-trip is roughly **1–2 µs** small-message; a tuned kernel TCP path is tens of µs, and — the point — its *tail* is far worse and load-dependent. InfiniBand switch hop latency is ~**100–130 ns**.
- **Bandwidth:** a ConnectX-7 does **400 Gb/s** (NDR-class), ConnectX-8 does **800 Gb/s** (XDR-class), at near-line-rate with near-zero host CPU.
- **CPU:** RDMA large transfers run at **~0% host CPU** on the data path; the equivalent TCP saturating 400G burns multiple cores.
- **Copies:** TCP GPU→GPU = **4 payload copies** (2 GPU↔DRAM staging + 2 socket buffers); RDMA+GPUDirect = **0** (NIC DMAs HBM↔HBM).

## Worked example — trace a 1 MB GPU→GPU transfer, TCP vs RDMA

Setup: GPU on host A must deliver a 1 MB gradient shard to a GPU on host B (one leg of an all-reduce). Trace every stage; name what RDMA deletes.

**(a) Over TCP (the 01b datapath, in full):**

1. **Copy GPU → host DRAM.** No GPUDirect, so CUDA copies 1 MB out of HBM into a pinned host staging buffer over PCIe.
2. **`send()` syscall.** Userspace → kernel transition; scheduler and syscall entry cost.
3. **Copy into socket send buffer.** Kernel copies the 1 MB into `sk_buff`s (copy #2 of the payload).
4. **TCP/IP stack.** Segmentation (or TSO deferral), sequence numbers, checksums, congestion-window bookkeeping — all CPU.
5. **netfilter + conntrack.** `nf_conntrack` looks up / updates the flow tuple; firewall and any NAT/overlay (Kubernetes!) hooks run. Stateful, lock-touching, cache-miss-prone under load.
6. **qdisc / driver.** Enqueue, `ndo_start_xmit`, NIC DMAs the segments out.
7. **Wire → NIC B.** Ethernet, subject to switch buffering and drops → retransmits.
8. **Softirq (`NET_RX`) on host B.** NIC raises interrupt; softirq drains the ring, sharing the core's budget with every other flow.
9. **IP/TCP reassembly + conntrack lookup** on B. Reorder, ACK, checksum-verify.
10. **Copy into socket receive buffer**, then **copy to userspace on `recv()`** (payload copies #3 and #4).
11. **Copy host DRAM → GPU B** over PCIe.

Cost: 4 payload copies, 2 syscalls, 2 conntrack touches, 1 softirq you don't schedule, full software transport on both ends. The killer isn't the mean — it's that steps 5, 8, 9 are **load-dependent and shared**, so p99.9 balloons exactly when the fabric is busy, i.e., mid-collective.

**(b) Over RDMA + GPUDirect (what actually deletes):**

*Setup, done once (not per-message):* register the GPU HBM regions as MRs on both hosts (pin + program NIC translation), create an RC QP between A and B, exchange QP numbers and `rkey`s over a one-time out-of-band handshake.

*Per 1 MB transfer:*

1. Host A posts one **RDMA WRITE WQE** to the QP's send queue via a memory-mapped doorbell: "1 MB from local GPU MR → remote GPU address Y, `rkey` Z." **~No syscall.**
2. **RNIC A DMAs the 1 MB straight out of GPU A's HBM** (GPUDirect — no GPU→DRAM copy, step A1 above is gone).
3. Wire → **RNIC B DMAs the 1 MB straight into GPU B's HBM.** RNIC B handles reliable delivery/ordering/ACK **in hardware**.
4. RNIC A posts a **CQE**; A's CPU sees completion by polling the CQ.

**What RDMA deleted, named explicitly:** the two `send()`/`recv()` **syscalls**; **all four payload copies** (both socket buffers *and* both GPU↔DRAM staging copies — zero-copy end to end); the entire **kernel TCP/IP stack**; **netfilter + conntrack** on both ends (no flow tuple exists — this is why RDMA and stateful firewalls/overlays are fundamentally in tension, a real Kubernetes design constraint); the **softirq** receive path; and the remote CPU's involvement entirely (host B's CPU did **nothing** — it wasn't interrupted, it didn't copy, it may not even know the WRITE happened until it reads its own memory). Result: ~2 copies-worth of PCIe DMA and one wire crossing, at ~0% CPU, with a latency whose tail doesn't depend on how busy the host's cores or conntrack table are.

## Practice (feeds the deliverable)

Produce the **TCP-vs-RDMA datapath trace** for the Network Architecture Read deliverable.

**Task.** In prose (a table plus two short paragraphs is fine), trace the same 1 MB GPU→GPU transfer twice — once over TCP, once over RDMA+GPUDirect — for your own reference cluster. For the TCP path, name **every** 01b kernel stage on the hot path: `send()`/`recv()` syscalls, both socket-buffer copies, the GPU↔host staging copies, the TCP/IP stack, netfilter/**conntrack**, the **softirq** receive path, and the NIC DMA. For the RDMA path, name what is deleted and what remains (WQE post, doorbell, NIC DMA out of/into HBM, hardware reliable transport, CQE completion), and mark which stages are **once-only setup** (MR registration, QP creation, key exchange) vs **per-message**.

**Acceptance criteria:**
- Both paths trace the *same* 1 MB transfer, so they're comparable.
- Every TCP hot-path stage from 01b is named (syscall, ≥1 socket-buffer copy, staging copy, TCP/IP, conntrack/netfilter, softirq, NIC DMA) — a reviewer can check them off.
- The RDMA trace explicitly separates **setup (once)** from **data path (per-message)** and states that the **remote CPU is uninvolved**.
- You name at least one **stability** consequence (a stage whose *tail* latency is load-dependent under TCP and simply absent under RDMA), not just a throughput number.

Keep it to one page. This trace is the backbone reviewers use to check you understand *mechanism*, not vocabulary.

## Self-check

**(a) During an RDMA WRITE, what does the host CPU do — and what does it NOT do?**

**Answer:** On the **sender**, the CPU does exactly one small thing: it builds a Work Request and rings the NIC's doorbell (a memory-mapped write), then it's free — it later reaps a CQE to know the WRITE landed. It does **not** copy the payload (the NIC DMAs straight from the registered buffer / GPU HBM), does **not** run a transport stack (the RNIC does reliability, ordering, ACKs in hardware), and does **not** enter the kernel per message. On the **receiver**, the CPU does **nothing at all** — no interrupt, no softirq, no copy; the bytes appear in its memory. That "remote side is uninvolved" property is the essence of one-sided RDMA.

**(b) Why is RDMA latency stable in a way a conntrack'd TCP path isn't? (Tie to 01b.)**

**Answer:** TCP's tail latency comes from **stateful, shared, load-dependent** stages on the 01b hot path: the `nf_conntrack` table (hash lookups that miss cache and take locks, worse as connection count and traffic rise), the per-core softirq budget (your receive processing competes with every other flow on that core and can be deferred), and socket-buffer contention. Those are exactly the stages RDMA **deletes** — no flow table to consult, no softirq on the data path, no shared kernel buffers. What remains (a WQE, a DMA, a wire hop, a hardware ACK) is fixed-cost silicon whose latency doesn't grow with host load. Collectives are barriers, so it's the *tail* that sets step time — and RDMA flattens the tail by removing the stateful stages that produce it.

**(c) What is a queue pair, and why does kernel-bypass need it?**

**Answer:** A queue pair is the RDMA connection object — a Send Queue plus a Receive Queue in the NIC — onto which you post Work Requests and which holds the per-connection state (peer, transport type like RC, and the memory regions/keys it may touch). Kernel-bypass needs it because bypassing the kernel removes the arbiter that used to multiplex the NIC across processes, enforce memory isolation, and hold connection state. The QP restores all three **in hardware**: the kernel programs it once at setup, binding it to specific MRs with specific `lkey`/`rkey` access rights, and thereafter the NIC enforces that a process's WQEs can only touch its own registered memory. Without a QP there is no safe, isolated way for userspace to drive a shared NIC directly.

## Resources

1. **RDMA / InfiniBand for Kubernetes GPU networking — conceptual primer.** Ties RDMA concepts to the K8s + GPU context you operate in, including why RDMA and pod-network overlays/conntrack are in tension. https://www.kubenatives.com/p/rdma-infiniband-kubernetes-gpu-networking
2. **RDMA verbs primer (skim for concepts only).** Use it to solidify QP/CQ/MR/WQE and the WRITE/READ/SEND-RECV verbs; do **not** get pulled into coding the API — you want the object model, not `ibv_post_send()` syntax. (RDMAmojo / the "RDMA Aware Networks Programming User Manual" are the canonical references.)
3. **Module 01b — the kernel datapath RDMA deletes (reference, don't re-read fully).** Your own conntrack/netfilter/softirq/socket-buffer trace is the contrast that makes this lesson land; keep it open when you write the Practice trace.
