---
id: "01b"
title: "Linux systems internals"
notion: "https://app.notion.com/p/3b33abaeb823812a8e94cf07ea623410"
phase: "Phase 0 · Months 1–3 (parallel with Go)"
effort: "~65 hrs ≈ 6 weeks @ 10–12 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: []
unlocks: []
started: null
completed: null
---

# 🐧 01b — Linux systems internals

> **Goal.** Move from **using** Linux to **understanding** the kernel mechanisms
> underneath — because at GPU-fleet scale the money-losing failures live below the
> container runtime, and nothing above the kernel explains them.

- **Notion page:** https://app.notion.com/p/3b33abaeb823812a8e94cf07ea623410
- **Phase:** Phase 0 · runs **parallel with Go** (not sequentially gated) · **Est. effort:** ~65 hrs ≈ 6 weeks
- **Deliverable:** [Anatomy of a Container](practice/anatomy-of-a-container/) — a
  publishable teardown + a bpftrace/PSI diagnostic toolkit.

## Why this module, and to what bar

The GPU/neocloud tier tests **kernel-mechanism reasoning + performance debugging
under saturation**, not Linux admin:

- **CoreWeave** — *Systems Engineer, Kernel*: "kernel-level observability … kernel readiness for production workloads"; *Staff SWE Compute*: "observability stacks (Prometheus/PromQL) … operating large fleets of GPU servers."
- **Datadog** — *Sr SWE, Linux Kernel / GPU Monitoring*: "at the intersection of **eBPF, the Linux kernel, and GPU infrastructure** … investigating production incidents."
- **NVIDIA** — Base OS/Kernel: "GDB, kdump, and **eBPF tracing** to debug multiprocessor systems."

**Recurring interview questions:** "why is a pod throttled at 40% CPU?" (CFS quota) ·
"trigger + read an OOM kill from the kernel log" · "D-state processes" · "TCP
retransmits without drops" (conntrack) · "how does eBPF work" — all rewarding a
systematic **USE-method** diagnostic flow over memorized commands.

## Calibrated to your background — what we skip

You've operated Linux for years, so we **skip**: shell/pipes/permissions, package
managers, distro tours, beginner systemd units, vim, bash scripting, networking
basics, LFCS/LPIC. We start at the **kernel-mechanism** layer — and reframe tunables
from "which sysctl" to "what subsystem this sysctl steers."

## Lessons

Anchored on **cgroups + namespaces = "what a container really is."** Spine = L3, L8, L9.

| # | Lesson | Hrs | Fleet tie-in |
|---|--------|-----|--------------|
| 01 | [Processes, scheduling & the run queue](lessons/01-processes-and-scheduling.md) | 6 | D-state, load-vs-cores, NCCL core isolation |
| 02 | [Namespaces — half of what a container is](lessons/02-namespaces.md) | 7 | build a container by hand, GPU device visibility |
| 03 | [cgroups v2 + K8s resource enforcement](lessons/03-cgroups-v2-and-k8s-enforcement.md) (anchor) | 9 | limits/QoS → `cpu.max`/`memory.max`, cpuset/NUMA pinning |
| 04 | [PSI — saturation the right way](lessons/04-psi.md) | 5 | data-loader stalls the GPU step |
| 05 | [Memory management & the OOM killer](lessons/05-memory-and-oom.md) | 7 | attribute a kill from `dmesg`, checkpoint-loss economics |
| 06 | [Hugepages / THP / NUMA](lessons/06-hugepages-thp-numa.md) | 5 | GPU/NIC affinity, TLB/throughput |
| 07 | [Networking datapath & conntrack](lessons/07-networking-datapath-conntrack.md) | 6 | NCCL/egress blows conntrack |
| 08 | [eBPF — the observability substrate](lessons/08-ebpf.md) | 8 | Cilium/Hubble, Datadog-style tracing, BTF/CO-RE |
| 09 | [perf / ftrace / USE method](lessons/09-perf-ftrace-use.md) | 7 | how you answer debugging interviews |
| 10 | [systemd-as-cgroup-manager + block I/O](lessons/10-systemd-cgroups-and-block-io.md) | 5 | delegation, io PSI, iocost |

Total ≈ **65 hrs ≈ 6 weeks** — deliberately lighter than the controllers module,
runs parallel to Go.

## Resource spine

- **Brendan Gregg — *Systems Performance* (2nd ed)** + **USE method** + ***BPF
  Performance Tools*** — the mental model, methodology, and eBPF cookbook.
- **Kernel docs** — cgroup-v2 & PSI (authoritative interface reference).
- **bpftrace one-liner tutorial** + **Cilium/Hubble docs** — the eBPF datapath.
- **Julia Evans — *How Containers Work*** — fast, correct on-ramp to namespaces/cgroups.
- (TLPI = reference-only, 2010; EEVDF replaced CFS in kernel 6.6 but kept the
  bandwidth-control interface, so the throttling lesson holds.)

## Deliverable & checkpoint

- Build **[Anatomy of a Container](practice/anatomy-of-a-container/)** across the lessons.
- The [**checkpoint**](checkpoint.md) is the gate — you can locate any container's
  ns+cgroup state by hand, explain an OOM from the kernel log, and diagnose a pressured
  node via PSI.

## How to work this module

1. Lessons in order; do every **Practice** on a laptop/VM/container — they compound
   into the teardown + toolkit deliverable.
2. Keep a running lab log; capture the bpftrace snippets and flame graph as you go.
3. Answer the [checkpoint](checkpoint.md) from memory; flip `status` and update the
   Notion tracking row when the deliverable is done.
