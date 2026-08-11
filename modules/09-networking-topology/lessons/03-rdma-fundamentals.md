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
sources: 6
---
# 09.3 · RDMA fundamentals: kernel bypass, zero-copy, queue pairs

> **Concept.** RDMA is the deliberate deletion of the kernel datapath you learned in 01b — the NIC reads and writes application (and GPU) memory directly, so the CPU, socket buffers, netfilter, and conntrack are simply not on the wire. The win that matters for collectives isn't a faster average — it's a flat tail.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Where this fits

Lesson 02 built the *shape* of the fabric above the NIC — leaf/spine/aggregation tiers, bisection bandwidth, and why rail-optimized designs let you oversubscribe the spine for free. That answered "how many wires, arranged how." It did not answer "what actually rides those wires, and why does it matter that it isn't TCP." This lesson answers that: it takes the 01b kernel datapath you already traced — `send()` → `sk_buff` → TCP/IP → conntrack → softirq → `recv()` — and shows exactly which stages RDMA removes and why removing them buys latency *stability*, not just speed. That matters because a synchronous collective (NCCL all-reduce, module 08) is a barrier: 511 GPUs wait on the slowest one, so a jittery tail is a stalled step, not an average inconvenience.

What this lesson does **not** yet cover is *how the fabric itself becomes trustworthy enough for RDMA to run on* — RDMA's reliable transport assumes packets essentially never get dropped to congestion, and Ethernet was not built that way. That's the subject of lesson 04, and it's the direct sequel to a fact this lesson sets up: Meta runs production RoCE fabrics with the network-level congestion-control layer (DCQCN) *disabled*, replacing it with library-level coordination between NCCL and the RDMA transport. You can't understand why that trade is even conceivable without first understanding, at the mechanism level, what RDMA is and isn't doing on your behalf — which is this lesson's job.

## Why this matters

At 40+ clusters you already know the interview question that separates a "networking person" from a platform engineer who runs GPU fabrics: *"Why can't we just train on TCP?"* The wrong answer is "TCP is slow." The right answer names mechanism — and the mechanism is that a synchronous collective is bottlenecked by the **slowest** and **most jittery** link in the ring, not the average one. TCP's tail latency — the p99.9 that spikes when a conntrack bucket rehashes or a softirq gets stolen by another core — becomes *your* step time. RDMA exists to make the tail flat.

Two dollars-and-cents consequences you will own:

- **Utilization is money.** A100/H100/B200 time is the single largest line item in a training budget. If TCP overhead and jitter drop fabric-bound collective efficiency from 95% to 70%, you are lighting ~25% of a nine-figure GPU spend on fire. RDMA is not a nicety; it is the thing that keeps `$/token` defensible.
- **CPU is not free either.** A single 400G NIC saturated with TCP can consume many host cores just doing copies and interrupt handling — cores you wanted for the data loader. RDMA gives them back.

This lesson is conceptual. You will not write verbs code. You will be able to stand at a whiteboard and trace exactly which of your 01b kernel stages RDMA removes, and why removing them buys latency *stability*, not just latency. Three claims you should be able to defend by the end:

- RDMA is **kernel-bypass + zero-copy + transport-offload** — three distinct wins, not one.
- The win that matters for collectives is **tail stability**, and it comes specifically from deleting the *stateful, shared* kernel stages (conntrack, softirq), not from a faster mean.
- The **remote CPU is uninvolved** in a one-sided WRITE — a property TCP structurally cannot offer.

## What's new here (calibration)

You already know 01b's kernel datapath cold — this lesson leans on that trace rather than re-teaching it. You also already know 02b's `nvidia-smi topo -m` and the NVLink/PCIe topology matrix, 08's NCCL collective algorithms, and 06's topology-aware scheduling — so RDMA is introduced here strictly as *the substrate NCCL's inter-node transport rides on*, not as a standalone networking primitive to master from scratch. What's genuinely new:

- The **QP/CQ/MR object model** — the hardware abstractions that let userspace talk to the NIC safely without the kernel arbitrating every message.
- **One-sided (WRITE/READ) vs two-sided (SEND/RECV)** semantics, and why the one-sided case is the one that makes "the remote CPU does nothing" true.
- The precise **boundary of kernel bypass** — what still goes through the kernel (setup) vs what doesn't (steady-state data path) — a distinction that's a favorite interview trap.
- RDMA is taught here as a **conceptual deletion of the 01b datapath**, not as verbs-coding; you will not write `ibv_post_send()`, you will be able to say what it does and why.

## Core concepts

### 1. Kernel bypass — the control path leaves the kernel once

"Bypass" does not mean the kernel is never involved. Setup still goes through it: you open a device, **register a memory region (MR)** — which pins the pages so they can't be swapped and programs the NIC's on-board translation (MTT/MPT) so the NIC can translate virtual → physical and enforce access rights — and you create queues. This is expensive and done **once**, at connection/buffer-pool setup. After that, the *data path* is a direct userspace→NIC conversation via a memory-mapped "doorbell." The syscall (`send()`/`recv()`) that dominated your 01b trace is gone from steady state.

Consequence: no per-message trip across the syscall boundary, no scheduler interaction, no copy into kernel-owned memory. This is also the seed of the first pitfall below: bypass is a *steady-state* property, not an *always* property.

### 2. Zero-copy — the bytes are never duplicated

In TCP the same 1 MB is copied at least twice: userspace → socket send buffer on the sender, socket receive buffer → userspace on the receiver (plus DMA to/from the NIC). With RDMA the NIC DMAs **straight out of** the sender's registered buffer and **straight into** the receiver's registered buffer. With **GPUDirect RDMA**, "registered buffer" is GPU HBM itself — the NIC reads/writes the GPU over PCIe/NVLink-adjacent paths and the bytes never touch host DRAM or a bounce buffer. This is why RDMA + GPUDirect is the substrate NCCL rides on for inter-node all-reduce.

### 3. Queue pairs (QP) — the connection abstraction that makes bypass safe

A **queue pair** is the unit of an RDMA connection: a **Send Queue (SQ)** and a **Receive Queue (RQ)**. You post **Work Requests (WRs / WQEs — work queue entries)** onto a queue: "WRITE these 1 MB from local MR X to remote address Y," or "here is a buffer, RECV into it." The NIC consumes WQEs and executes them.

Why does kernel-bypass *need* the QP? Because you have removed the kernel as the arbiter that used to (a) multiplex many connections onto one NIC, (b) enforce isolation between processes, and (c) hold the connection state. The QP is the hardware object that restores all three: it is per-connection state living in the NIC, bound to specific memory regions with specific access rights (keys — `lkey`/`rkey`), so one process's WQE physically cannot address another's memory. The kernel programs the QP once at setup; thereafter the NIC enforces the boundary the kernel used to. **No QP, no safe bypass** — you'd have userspace poking a shared NIC with no isolation. This is the rebuttal to "RDMA is just a fast socket": removing the kernel as arbiter still requires *some* hardware-enforced isolation, and the QP/MR/key model is that enforcement, implemented in silicon instead of software.

QP transport service types matter for the interview: **RC (Reliable Connected)** — the NIC guarantees in-order, reliable, ACK'd delivery in hardware; this is what large-message collectives use. **UD (Unreliable Datagram)** and others exist but RC is the one that "replaces TCP's reliability, in silicon."

### 4. Completion queues (CQ) — asynchrony without interrupts

When the NIC finishes a WQE it posts a **Completion Queue Entry (CQE)** to a **Completion Queue (CQ)**. The CPU learns of completion by **polling** the CQ (busy-poll: lowest latency, burns a core) or by arming an **event** (a doorbell/interrupt when a CQE lands: lower CPU cost, slightly higher latency). Contrast 01b: TCP completion is a softirq you don't control, sharing a per-core budget with every other flow on that core. Here completion is a data structure *you* drain on *your* schedule.

### 5. The verbs (know the names, not the code)

The operations ("verbs"): **RDMA WRITE** (push into remote memory, remote CPU uninvolved — the workhorse), **RDMA READ** (pull from remote memory), **SEND/RECV** (two-sided: receiver must pre-post a RECV; used for control/handshake), and **atomics** (fetch-add, compare-swap). One-sided WRITE/READ are the "remote CPU does nothing" superpower; SEND/RECV is the two-sided fallback that looks most like sockets — the receiver's CPU must have posted a matching RECV WQE in advance, so it *is* aware traffic is expected, just not involved in moving the bytes.

### 6. GPUDirect RDMA — why the GPU is a first-class RDMA peer

The reason RDMA matters *specifically* for GPU clusters is **GPUDirect RDMA**: the RNIC is given a direct PCIe path to peer into GPU HBM, so the GPU's memory is registered as an MR and the NIC DMAs into/out of it with **no bounce through host DRAM and no CPU orchestration**. Without it, every inter-node collective byte would take the TCP-style detour HBM→DRAM→NIC→wire→NIC→DRAM→HBM (the two staging copies in the worked example below). With it, the path is HBM→NIC→wire→NIC→HBM. This is the substrate NCCL's inter-node transport rides on, and it's why "RDMA" and "GPU networking" are effectively synonyms at this tier — the win is not just kernel-bypass for the CPU, it's cutting the CPU and host DRAM out of the *GPU-to-GPU* path entirely. (BlueField-3 DPUs can additionally offload the transport and even parts of the collective onto the NIC itself.)

GPUDirect RDMA is **not automatic just because RDMA is present** — it additionally requires the PCIe-topology same-root-complex condition you learned in 02b (`PIX`/`PXB` good, `SYS` bad in `nvidia-smi topo -m`), extended fabric-wide to "same rail." Lesson 05 covers that trace in full; the point to hold here is that RDMA and GPUDirect RDMA are related but distinct guarantees — RDMA gets bytes host-to-host without the CPU; GPUDirect additionally gets them GPU-to-GPU without a DRAM detour, and it has its own precondition.

### 7. Memory registration — the cost you pay once, and why it exists

Registration (`ibv_reg_mr`) is not bookkeeping; it does two load-bearing things:

- **Pins the pages.** The NIC will DMA to physical addresses asynchronously, so the pages must not move or be swapped out underneath it. Pinning removes them from the pageable pool — which is why registered memory is a **scarce, finite resource** you pool and reuse, not allocate per message. On a multi-tenant node, every tenant's pinned buffers subtract from the same finite pinned-memory budget; registering on the hot path would also reintroduce a syscall and a page-table walk, defeating the entire point.
- **Programs the NIC's translation and protection tables** (MTT/MPT): virtual→physical mappings plus access rights, keyed by `lkey` (local) and `rkey` (remote). The `rkey` is what a peer must present in a WRITE/READ WQE; it's the capability token that makes one-sided access safe.

For GPUDirect the registered region is GPU HBM (via `nv_peer_mem` / DMA-BUF), and the same pin+translate contract applies to device memory. **Interview-ready framing:** RDMA moves *all* the expensive, stateful, kernel-touching work — connection setup, memory pinning, address translation programming — to a **one-time setup phase**, leaving the per-message path as pure silicon.

### 8. RC vs UD — pick the transport that matches the message

- **RC (Reliable Connected):** hardware-guaranteed reliable, in-order, ACK'd delivery; supports one-sided WRITE/READ. One QP per peer, so state scales with the number of peers — fine for the structured, all-to-few communication of a collective. This is the default for large-message GPU collectives.
- **UD (Unreliable Datagram):** connectionless, one QP talks to many, but no reliability and small messages only — used for control/discovery, not bulk data.

The reason RC "replaces TCP" cleanly: everything TCP did in kernel software (retransmit, ordering, ACK, flow state) RC does in the NIC ASIC, so you keep TCP's guarantees without TCP's CPU tax or its drop-and-recover tail. **What RC does not do**, and this is the seam to lesson 04: it guarantees reliable delivery of packets that *arrive*, but it does nothing to stop the fabric from dropping packets under congestion in the first place. RC's reliability is a transport-layer property; the fabric being loss-free is a separate, physical-layer/link-layer property that IB and RoCE achieve through entirely different mechanisms — that's the whole subject of the next lesson. Conflating "RC guarantees reliable delivery" with "the fabric is lossless" is a common and costly interview mistake.

### 9. The numbers (anchor them)

- **Latency:** RDMA half-round-trip is roughly **1–2 µs** small-message; a tuned kernel TCP path is tens of µs, and — the point — its *tail* is far worse and load-dependent. InfiniBand switch hop latency is ~**100–130 ns**.
- **Bandwidth:** a ConnectX-7 does **400 Gb/s** (NDR-class), ConnectX-8 does **800 Gb/s** (XDR-class), at near-line-rate with near-zero host CPU.
- **CPU:** RDMA large transfers run at **~0% host CPU** on the data path; the equivalent TCP saturating 400G burns multiple cores.
- **Copies:** TCP GPU→GPU = **4 payload copies** (2 GPU↔DRAM staging + 2 socket buffers); RDMA+GPUDirect = **0** (NIC DMAs HBM↔HBM).

(These figures are current as of 2026 for the ConnectX-7/8 generation; treat them as a snapshot that moves with silicon generations, not a permanent constant.)

## Perspectives

**Developer.** If you write training code against NCCL, RDMA is almost entirely invisible — you call `all_reduce()` and never see a QP or a WQE. It surfaces only when something goes wrong: a fallback from GPUDirect RDMA to host-staged TCP shows up as a mysterious throughput cliff, and the only way to see it is `NCCL_DEBUG=INFO` init-log tokens (`[GDRDMA]` / `via NET/IB/0/GDRDMA`, covered in lesson 05). The developer's mental model can stay "NCCL handles it" — but a senior developer knows *which log line* proves it actually happened.

**Operator.** Registered memory is pinned and **finite**, not free or instant. On a shared, multi-tenant node, every job's pinned buffers eat into the same physical-memory budget, and pinning pressure can starve other tenants or force a job into smaller buffer pools than it wants. Watching pinned-memory usage across a fleet — not just GPU memory — is part of running RDMA at scale, and it's a resource class most platform engineers don't think to monitor until it bites them.

**Kernel / hardware.** The QP/CQ/MR model exists because removing the kernel as arbiter doesn't remove the *need* for an arbiter — it just moves the job into silicon. A NIC handing raw DMA access to userspace with no isolation mechanism would be a security and correctness disaster (one process corrupting another's memory). The QP, bound to specific MRs with specific `lkey`/`rkey` rights, is that hardware-enforced isolation. This is the rebuttal to "RDMA is just a fast socket" — a socket's isolation is enforced by the kernel per-syscall; RDMA's is enforced by the NIC per-WQE, checked against state the kernel programmed once.

**Economics.** GPU-hours dominate training cost at hyperscaler scale — a 24K-GPU cluster represents a nine-figure capital and power commitment running continuously. At that scale, even a *small* percentage loss in collective efficiency from TCP-class tail latency (a few points of all-reduce slowdown) is a large *absolute* dollar figure, because it's a few points of a nine-figure number, every day the cluster runs. This is why RDMA isn't a "nice to have" networking optimization at this tier — it's a line item large enough that a platform engineer is expected to reason about it in dollars, not just microseconds.

## Real-world use cases

- **CoreWeave — "Use GPUDirect RDMA with InfiniBand."** A practitioner-facing how-to for wiring RDMA into a real cloud GPU workload on Kubernetes: requesting `rdma/ib` as a schedulable resource, installing NCCL + OFED in the pod image, and setting the NCCL/UCX env vars that select the InfiniBand device. It grounds the QP/MR/verbs vocabulary of this lesson in something you'd actually type into a pod spec. https://docs.coreweave.com/docs/products/networking/hpc-interconnect/use-gpudirect-rdma
- **Meta Engineering — "RoCE networks for distributed AI training at scale" (2024).** A real hyperscaler's account of running RDMA's one-sided semantics in production for multiple years at 24K+ GPU scale — and, notably, doing so with the network-level congestion-control layer (DCQCN) turned off, substituting library-level coordination instead. It's the direct forward-link to lesson 04: this lesson gives you the RDMA mechanism Meta is riding on; lesson 04 explains the fabric-level trade they made around it. https://engineering.fb.com/2024/08/05/data-center-engineering/roce-network-distributed-ai-training-at-scale/

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

**Side-by-side stage count (the whiteboard summary):**

```
                       TCP (per 1 MB)        RDMA+GPUDirect (per 1 MB)
  syscalls                 2 (send/recv)         0  (doorbell write)
  payload copies           4                     0  (HBM↔HBM DMA)
  kernel TCP/IP stack      yes (both ends)       none (transport in NIC)
  netfilter/conntrack      2 touches             none  (no flow tuple)
  softirq receive path     yes (host B)          none
  remote CPU involved      yes (drains, copies)  NO — receiver does nothing
  transport/reliability    kernel software       NIC hardware (RC QP)
  once-only setup          connect()             MR register + QP + key xchg
  tail-latency drivers     conntrack, softirq,   fixed silicon; no
                           socket-buffer contention  load-dependent stages
```

Read the last row as the thesis of the whole lesson: the stages TCP adds are precisely the *load-dependent, shared, stateful* ones — and those, not the mean, are what a barrier-synchronous collective pays for.

## Practice (feeds the deliverable)

Produce the **TCP-vs-RDMA datapath trace** for the Network Architecture Read deliverable.

**Task.** In prose (a table plus two short paragraphs is fine), trace the same 1 MB GPU→GPU transfer twice — once over TCP, once over RDMA+GPUDirect — for your own reference cluster. For the TCP path, name **every** 01b kernel stage on the hot path: `send()`/`recv()` syscalls, both socket-buffer copies, the GPU↔host staging copies, the TCP/IP stack, netfilter/**conntrack**, the **softirq** receive path, and the NIC DMA. For the RDMA path, name what is deleted and what remains (WQE post, doorbell, NIC DMA out of/into HBM, hardware reliable transport, CQE completion), and mark which stages are **once-only setup** (MR registration, QP creation, key exchange) vs **per-message**.

**Acceptance criteria:**
- Both paths trace the *same* 1 MB transfer, so they're comparable.
- Every TCP hot-path stage from 01b is named (syscall, ≥1 socket-buffer copy, staging copy, TCP/IP, conntrack/netfilter, softirq, NIC DMA) — a reviewer can check them off.
- The RDMA trace explicitly separates **setup (once)** from **data path (per-message)** and states that the **remote CPU is uninvolved**.
- You name at least one **stability** consequence (a stage whose *tail* latency is load-dependent under TCP and simply absent under RDMA), not just a throughput number.

Keep it to one page. This trace is the backbone reviewers use to check you understand *mechanism*, not vocabulary.

## Common pitfalls

- **"Kernel bypass means the kernel is never involved."** False — MR registration and QP creation *are* syscalls, and they're expensive (pinning pages, programming NIC translation tables). Only the steady-state, per-message data path bypasses the kernel. Confusing "bypass" with "never touches the kernel" is the single most common RDMA misconception in interviews.
- **Forgetting registered memory is pinned and finite.** It's not a free, instant allocation — pinning removes pages from the swappable pool and consumes a scarce, per-node resource. Registering per-message (instead of pooling and reusing MRs) reintroduces the exact syscall/page-walk cost RDMA exists to avoid, and over-registering on a shared node can starve other tenants.
- **Assuming GPUDirect RDMA "just works" whenever RDMA is present.** It has an additional precondition beyond plain RDMA: the NIC and GPU must share a PCIe root complex (or at worst one switch) — the same 02b same-root-complex rule, extended fabric-wide as "same rail" in lesson 05. Cross-root-complex GPU↔NIC pairs silently fall back to host-staged copies, and nothing crashes to tell you.
- **Conflating RC's reliable-delivery guarantee with "the fabric is lossless."** RC (Reliable Connected) guarantees that packets which *arrive* are delivered in order and acknowledged — a transport-layer property enforced in the NIC ASIC. It says nothing about whether the fabric drops packets to congestion in the first place; that's an entirely separate, link/network-layer property that InfiniBand and RoCEv2 achieve through different mechanisms, which is the whole subject of the next lesson.

## Self-check

**(a) During an RDMA WRITE, what does the host CPU do — and what does it NOT do?**

**Answer:** On the **sender**, the CPU does exactly one small thing: it builds a Work Request and rings the NIC's doorbell (a memory-mapped write), then it's free — it later reaps a CQE to know the WRITE landed. It does **not** copy the payload (the NIC DMAs straight from the registered buffer / GPU HBM), does **not** run a transport stack (the RNIC does reliability, ordering, ACKs in hardware), and does **not** enter the kernel per message. On the **receiver**, the CPU does **nothing at all** — no interrupt, no softirq, no copy; the bytes appear in its memory. That "remote side is uninvolved" property is the essence of one-sided RDMA.

**(b) Why is RDMA latency stable in a way a conntrack'd TCP path isn't? (Tie to 01b.)**

**Answer:** TCP's tail latency comes from **stateful, shared, load-dependent** stages on the 01b hot path: the `nf_conntrack` table (hash lookups that miss cache and take locks, worse as connection count and traffic rise), the per-core softirq budget (your receive processing competes with every other flow on that core and can be deferred), and socket-buffer contention. Those are exactly the stages RDMA **deletes** — no flow table to consult, no softirq on the data path, no shared kernel buffers. What remains (a WQE, a DMA, a wire hop, a hardware ACK) is fixed-cost silicon whose latency doesn't grow with host load. Collectives are barriers, so it's the *tail* that sets step time — and RDMA flattens the tail by removing the stateful stages that produce it.

**(c) What is a queue pair, and why does kernel-bypass need it?**

**Answer:** A queue pair is the RDMA connection object — a Send Queue plus a Receive Queue in the NIC — onto which you post Work Requests and which holds the per-connection state (peer, transport type like RC, and the memory regions/keys it may touch). Kernel-bypass needs it because bypassing the kernel removes the arbiter that used to multiplex the NIC across processes, enforce memory isolation, and hold connection state. The QP restores all three **in hardware**: the kernel programs it once at setup, binding it to specific MRs with specific `lkey`/`rkey` access rights, and thereafter the NIC enforces that a process's WQEs can only touch its own registered memory. Without a QP there is no safe, isolated way for userspace to drive a shared NIC directly.

**(d) What's the difference between "kernel bypass" and "the kernel is never involved"?**

**Answer:** They are not the same claim, and interviewers use this to separate people who understand the mechanism from people who've memorized the marketing line. "Kernel bypass" describes the *steady-state data path*: once a QP is created and memory is registered, each message goes straight from userspace to the NIC via a doorbell, with no syscall. "The kernel is never involved" is false — MR registration and QP creation *are* kernel operations (they pin pages, program translation tables, and allocate NIC resources), and they happen through the kernel exactly once, at setup. The correct framing: bypass removes the kernel from the *per-message* path after a one-time setup phase that still goes through it.

**(e) Why does Meta's DCQCN-disabled RoCE fleet still depend on everything in this lesson?**

**Answer:** Meta replaced the network-level congestion-control algorithm (DCQCN) with receiver-driven admission co-designed into the NCCL collective library — that's a lesson-04 topic, about *how the fabric avoids congestion*. But underneath that substitution, every byte still moves as a one-sided RDMA WRITE over a hardware-reliable RC queue pair, registered memory, GPUDirect into HBM — the entire mechanism this lesson covers. Turning off DCQCN changes *who decides when to send* (the collective library instead of a NIC-level algorithm reacting to ECN marks); it does not change *how a send physically happens* once permission is granted. RDMA fundamentals are the substrate both designs — DCQCN-based and library-driven — are built on.

## Connections & what's next

This lesson is the mechanism layer underneath three other lessons in the module: 02's fabric shape is *what RDMA rides across*; 05's GPUDirect/SHARP trace is *the same-root-complex rule from §6 extended fabric-wide, plus what happens when a switch does the reduction instead of a ring of RDMA WRITEs*; 06's Kubernetes multi-NIC lesson is *how a pod gets access to the QP/MR machinery this lesson describes* (why the default CNI, which knows nothing about RDMA devices, can't do this). Module 08's NCCL collectives are the *caller* of everything here — NCCL decides *what* to communicate and *in what pattern*; RDMA decides *how a single point-to-point transfer physically happens* underneath that pattern.

The immediate next step is lesson 04: **InfiniBand vs RoCEv2 and engineered losslessness.** This lesson established that RC's reliable-delivery guarantee is not the same thing as "the fabric never drops packets" — RC handles packets that arrive, but a congestion drop still triggers a coarse, throughput-collapsing retransmit. The next lesson is entirely about how InfiniBand and RoCEv2 solve that *upstream* problem — one intrinsically (credit-based flow control), one by engineering it on top of ordinary Ethernet (PFC/ECN/DCQCN) — and about the real production trade Meta made when they decided library-level coordination could substitute for part of that engineering.

## References & further reading

**Primary sources**
- NVIDIA/Mellanox — *RDMA Aware Networks Programming User Manual.* The canonical verbs/QP/CQ/MR reference — read for the object model (queue pairs, memory regions, work requests, completion queues), not to memorize the C API. https://docs.nvidia.com/networking/display/rdmaawareprogrammingv17/rdma-aware+programming+overview
- Meta / SIGCOMM 2024 — *"RDMA over Ethernet for Distributed Training at Meta Scale"* (paper PDF). Read for how RDMA's one-sided semantics interact with real collective-library design at 24K+ GPU scale. https://engineering.fb.com/wp-content/uploads/2024/08/sigcomm24-final246.pdf

**Real-world engineering blogs**
- CoreWeave — *"Use GPUDirect RDMA with InfiniBand."* What it shows: a practitioner how-to wiring RDMA into a real cloud GPU workload on Kubernetes — grounds the QP/MR/verbs vocabulary operationally. https://docs.coreweave.com/docs/products/networking/hpc-interconnect/use-gpudirect-rdma
- Meta Engineering — *"RoCE networks for distributed AI training at scale."* What it shows: a hyperscaler running RDMA in production for years at extreme scale, and the direct forward-bridge to lesson 04's congestion-control story. https://engineering.fb.com/2024/08/05/data-center-engineering/roce-network-distributed-ai-training-at-scale/

**Deeper dives**
- RDMAmojo (Dotan Barak) — a long-running blog with deep individual posts on queue pairs, completion queues, memory registration, and verbs mechanics, for whenever you want to go past the object-model level. https://www.rdmamojo.com/
- kubenatives — *"RDMA / InfiniBand for Kubernetes GPU networking."* Ties RDMA concepts to the K8s + GPU context you operate in, including why RDMA and pod-network overlays/conntrack are in tension. https://www.kubenatives.com/p/rdma-infiniband-kubernetes-gpu-networking
