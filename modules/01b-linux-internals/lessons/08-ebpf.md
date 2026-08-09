---
lesson: "01b.8"
title: "eBPF"
module: "01b"
concept: "eBPF"
status: not-started
est_time: "7h"
artifacts: []
---

# 01b.8 · eBPF

> **Concept.** eBPF is a sandboxed virtual machine inside the kernel: you attach small verified programs to kernel and userspace hooks, they run at those events, and they ship data out through maps — turning the kernel into a programmable, production-safe observability datapath.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Why this matters
"eBPF" is now a line item in job descriptions at exactly the companies you are targeting: CoreWeave and NVIDIA want kernel-level GPU/host telemetry, Datadog's agent ships eBPF for network and universal service monitoring, Cilium/Hubble is the CNI layer on a large share of managed Kubernetes. When a JD says "experience with eBPF-based observability," the filter is whether you can talk about the *conceptual model* — programs, hooks, maps, the verifier — not whether you memorized `bpftrace` syntax. On a fleet, the practical payoff is that you can answer questions that are otherwise unanswerable without a redeploy: "which container is issuing the 4KB random reads hammering this NVMe," "who is calling `execve` with `curl | sh`," "why does p99 latency for this service spike every 30s" — live, on a production node, at overhead low enough to leave running. This is the single fastest-rising signal in the platform/SRE hiring market, and it is a genuine capability multiplier, not résumé decoration.

## From using to understanding
As an operator you have used tools that are eBPF underneath without naming it: `cilium hubble observe`, the Datadog network map, `bpftop`, maybe `execsnoop` or `opensnoop`. You treat them as magic boxes that emit events. You know `strace` shows syscalls and is dangerous on a hot process because it stops the target on every call via ptrace.

What you are learning now is the machine under those tools. eBPF is not a tracer — it is a **safe, in-kernel execution environment**. You write a program in a restricted C (or a bpftrace/DSL script), a userspace loader compiles it to eBPF bytecode and hands it to the kernel via the `bpf()` syscall, the **verifier** proves it cannot crash or hang the kernel, the **JIT** turns it into native machine code, and it gets attached to a **hook** so it fires whenever that event happens — a syscall entry, a kernel function, a userspace function, a network packet, a perf counter overflow. Unlike `strace`, there is no context switch to a tracer process per event and no stopping the target: the program runs *in kernel context at the hook* and aggregates in-kernel, so you can trace a million-events-per-second path and only copy summaries to userspace. Once you see it as "a verified VM I attach to hooks," `strace` vs `bpftrace`, BCC vs libbpf, and why Hubble sees things iptables logging cannot all become obvious.

## Core notes

### The conceptual model: programs, hooks, maps, verifier, JIT
An eBPF program is bytecode for a small RISC-like virtual machine (11 registers, a 512-byte stack, its own instruction set). The lifecycle:

1. **You attach a program to a hook.** The hook is *where* the program runs. The families you care about for observability:
   - **kprobes / kretprobes** — dynamic instrumentation of (almost) any kernel function entry/return, e.g. `vfs_read`, `tcp_sendmsg`. Powerful but unstable: function names and arguments can change between kernel versions.
   - **tracepoints** — static, kernel-maintained instrumentation points with a stable ABI (`syscalls:sys_enter_read`, `sched:sched_switch`, `block:block_rq_issue`). Prefer these over kprobes when one exists — they don't break across kernel upgrades. `bpftrace -l 'tracepoint:*'` lists them.
   - **uprobes / uretprobes** — the same dynamic instrumentation for *userspace* functions in a binary or library, e.g. a Go runtime function, `libc:malloc`, a TLS read before encryption. This is how you trace application internals without recompiling.
   - **USDT** — statically-defined tracepoints compiled into userspace apps (used by the JVM, Node, PostgreSQL).
   - **perf events / PMU** — sample on a hardware counter overflow (cycles, cache misses) or a timer; this is what powers CPU profiling and off-CPU sampling.
   - **network hooks** — XDP (earliest, at the driver, for line-rate drop/redirect), tc/`clsact` ingress-egress, cgroup/socket hooks. This is the datapath Cilium is built on.

2. **The program runs and writes to maps.** A **map** is a kernel data structure — hash, array, per-CPU array, LRU hash, ring buffer, stack-trace map — that both the eBPF program (in kernel) and userspace read/write. Maps are how state survives between invocations and how data crosses the kernel/userspace boundary. The key move is **aggregate in-kernel**: instead of sending every event up, increment a per-key counter or a histogram bucket in a map and let userspace read the summary periodically. That is why eBPF is cheap where `strace` is ruinous. `BPF_MAP_TYPE_RINGBUF` (per-event streaming) and `BPF_MAP_TYPE_PERCPU_HASH` (lock-free aggregation) are the two you'll meet most.

3. **The verifier proves it safe *before* it loads.** This is the crux of "why is it safe to run arbitrary code in the kernel." The verifier does a static analysis — it walks every possible execution path and checks: the program **terminates** (originally: no loops at all; modern kernels allow *bounded* loops the verifier can prove finite, and `bpf_loop()` helpers), it never dereferences an **unchecked or out-of-bounds pointer**, it never reads **uninitialized memory** or leaks kernel pointers to unprivileged userspace, it stays within the stack and instruction-count limits (historically 4096 insns, now up to 1M for privileged), and it only calls a **fixed allowlist of helper functions** (`bpf_map_lookup_elem`, `bpf_probe_read_kernel`, `bpf_ktime_get_ns`, ...). It cannot call arbitrary kernel functions (except a curated set of "kfuncs"), cannot allocate unbounded memory, cannot block or sleep (except in explicitly sleepable program types). If any path fails the proof, `bpf()` returns and the program never runs. So safety is **guaranteed by construction at load time**, not by trapping faults at runtime. The cost is real: "the verifier rejected my program" is the eBPF developer's daily pain, and its analysis is why loops and memory access are so constrained.

4. **The JIT compiles bytecode to native instructions.** Once verified, the JIT translates eBPF bytecode into native machine code (x86-64, arm64) so it runs at near-native speed rather than being interpreted. Verified + JITed is what makes production-always-on tracing viable.

Mental model to keep: **verifier = "will this be safe?", JIT = "make it fast", maps = "how do I get data out", hook = "when does it run".**

### Maps in more depth — the data plane
The map type you choose is a performance decision, and interviewers who know eBPF probe it:
- **`BPF_MAP_TYPE_HASH` / `PERCPU_HASH`** — keyed aggregation. The **per-CPU** variant gives each CPU its own copy so the program never takes a lock on the hot path; userspace sums the copies at read time. This is how you count "syscalls by pid" at millions/sec without cache-line contention.
- **`BPF_MAP_TYPE_ARRAY` / `PERCPU_ARRAY`** — fixed-size, index-keyed; used for histograms (bucket index → count) and global config/flags.
- **`BPF_MAP_TYPE_LRU_HASH`** — bounded hash that evicts least-recently-used entries; used when the key space is unbounded (per-connection state) and you must cap memory.
- **`BPF_MAP_TYPE_RINGBUF`** — the modern per-event channel to userspace (single MPSC ring shared across CPUs, replacing the older per-CPU `PERF_EVENT_ARRAY` perf buffer). Use it when you genuinely need every event (a stream of `execve`s), not an aggregate.
- **`BPF_MAP_TYPE_STACK_TRACE`** — stores captured stack traces keyed by an id; the profiling/off-CPU tools use it to fold thousands of stacks in-kernel.

The design rule that makes eBPF cheap: **aggregate in a per-CPU map and read summaries; stream through a ring buffer only when you must.** Sending every event to userspace is the anti-pattern that makes a tracer expensive.

### Attach vs load, and pinning
Two distinct steps often conflated: **loading** a program (verify + JIT, `bpf(BPF_PROG_LOAD)`) and **attaching** it to a hook (`bpf_link`). A loaded-but-unattached program does nothing; detaching leaves it loaded. Programs and maps are normally torn down when the loader process exits (refcount to zero) — unless you **pin** them into the bpffs virtual filesystem (`/sys/fs/bpf`), which keeps them alive independent of any process. This is how a CNI or agent keeps its datapath running across restarts, and how `bpftool prog show` / `bpftool map dump` let you inspect what's currently loaded on a node — a useful "what eBPF is even running here?" first move on an unfamiliar box.

### Cost, overhead, and safety envelope
eBPF is cheap but not free: each event pays the cost of the hook mechanism (a kprobe is an INT3/optimized-jump trampoline; a tracepoint is a cheaper static branch) plus your program's instructions. A tight counting probe on a hot function adds tens of nanoseconds; a probe that captures full stacks or does per-event ring-buffer output on a million-events/sec path *can* matter — measure before leaving something on permanently. Two safety caveats beyond the verifier: kprobes on a very hot function multiply that per-event cost across the whole system, and `bpf_probe_read` on userspace/kernel memory can fail (returns an error, doesn't fault) if the page isn't present — your program must handle that. Privilege: loading tracing programs needs `CAP_BPF` + `CAP_PERFMON` (or root); this is why agents run privileged and why unprivileged eBPF has been progressively locked down for security.

### Fleet/GPU tie-in
For your target roles this is the concrete hook: eBPF is how you attribute host-side behavior to GPU workloads without touching the training code. uprobes on the CUDA runtime or NCCL, tracepoints on the block and network layers correlated to the cgroup/container issuing them, and off-CPU stacks (next lesson) that reveal a trainer blocked on `nfs`/`io_uring` rather than compute — all of it is the "the GPU is 20% utilized, *why*" investigation, answered from the host with no redeploy. That question ("expensive accelerator sitting idle, find the host-side stall") is precisely the cost/observability problem these teams hire for.

### The three toolchains: bpftrace vs BCC vs libbpf
All three produce eBPF programs; they differ in effort, portability, and control.

- **bpftrace** — a high-level tracing language (awk/DTrace-like) for **ad-hoc, interactive investigation**. One-liners and short scripts; it compiles your script to eBPF on the fly. Reach for it first at 2am on a node: fast to write, no build step. Ceiling: it is a tracing DSL, not a way to ship a daemon. `bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }'`.
- **BCC (BPF Compiler Collection)** — a Python (or Lua) framework that embeds C for the kernel side and compiles it **at runtime on the target** with an in-process LLVM/Clang. Ships the classic tool suite (`execsnoop`, `opensnoop`, `biolatency`, `tcplife`, `runqlat`). Good for reusable tools with rich userspace logic. Downsides: heavyweight (LLVM + kernel headers on every host), slow startup, and runtime compilation is fragile across a fleet. Historically the standard; now largely a **legacy** path — many BCC tools have been rewritten to libbpf.
- **libbpf + CO-RE** — the **production** path. You write the kernel side in C, compile it **once** ahead of time to a BPF object, and it runs across kernel versions thanks to **CO-RE (Compile Once – Run Everywhere)**: BTF (BPF Type Format, kernel type metadata) plus compiler relocations let the loader fix up struct field offsets at load time, so you don't need kernel headers or a compiler on the target. This is what Cilium, the Datadog agent, and Pixie ship. `bpftool` is the companion CLI (inspect loaded programs/maps, dump BTF, generate skeletons). Rust has an equivalent ecosystem (**Aya**, libbpf-rs).

Rule of thumb: **bpftrace to investigate, libbpf/CO-RE to ship, BCC only for its existing tool catalog.**

### eBPF as the production observability datapath
This is the part interviewers probe, because it separates "I ran a bpftrace one-liner once" from "I understand why the industry moved here."
- **Cilium + Hubble** — Cilium is a CNI that implements pod networking, load balancing, and network policy in eBPF at tc/XDP/socket hooks instead of iptables. **Hubble** is its observability layer: because the eBPF programs sit *in the datapath at the socket and packet level*, Hubble sees every flow with **Kubernetes identity already attached** — source/destination pod, namespace, service, L7 verb (HTTP method+path, gRPC method, DNS query) — and whether policy *allowed or dropped* it, with the reason. `hubble observe --namespace foo --verdict DROPPED` shows drops labeled by identity and policy.
- **Datadog agent** — uses eBPF for Network Performance Monitoring and Universal Service Monitoring: it hooks the kernel TCP/UDP path (`tcp_sendmsg`, `tcp_close`, retransmit tracepoints) to attribute bytes, retransmits, and RTT to processes/containers without sidecars or app changes.
- **Pixie** — auto-instruments a cluster with eBPF (including uprobes on TLS libraries to capture *decrypted* HTTP/2/gRPC) for no-instrumentation service maps and request tracing.

The through-line: eBPF gives you **kernel-truth with application/orchestrator identity**, gathered in the datapath, with no code changes and low enough overhead to run continuously.

## Worked example: finding the busiest syscall, then who is behind it
A node is sluggish; CPU shows high `%sys` (kernel time) in `top`, so the cost is in syscalls, not user code. Question one: **which syscall dominates?**

```
# Count every syscall entry system-wide for a few seconds, then Ctrl-C.
$ sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[args.id] = count(); }'
Attaching 1 probe...
^C
@[257]: 40213      # ... which id is 257?
@[0]:   88190
@[1]:   90934
```

`raw_syscalls:sys_enter` fires for *all* syscalls with a numeric `id`; that's the cheapest possible probe (one hook, in-kernel counting). Ids 0/1/257 are `read`/`write`/`openat` on x86-64. To skip the id→name lookup, count by comm+name using the per-syscall tracepoints for the top candidate, or use `ksym`/the `syscall` builtin. The reads (`@[0]`) win. Question two: **who is issuing the reads, and how big are they?**

```
# Which processes issue read(), and the size distribution of the returned bytes.
$ sudo bpftrace -e '
  tracepoint:syscalls:sys_exit_read /args.ret > 0/ {
    @count[comm] = count();
    @bytes[comm] = sum(args.ret);
    @sizes = hist(args.ret);
  }'
^C
@count[fluent-bit]: 12043
@count[trainer]:    401200
@bytes[trainer]:    1643212800
@sizes:
[16, 32)          210 |                                        |
[4K, 8K)       390100 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[8K, 16K)       10800 |@                                       |
```

Reading the output: `trainer` is doing 400K reads averaging ~4KB — a tight small-read loop (the `hist` power-of-two histogram peaks in `[4K,8K)`). `args.ret` is the syscall's return value (bytes actually read); the `/args.ret > 0/` filter drops errors and EOFs so the histogram isn't polluted. The `hist()` builtin aggregates buckets **in-kernel in a map** — you never copied 400K events to userspace, which is exactly why this is safe to run on a busy box where `strace -c` would grind the process to a halt. Conclusion in one line: *PID group `trainer` is CPU-`%sys`-bound on 4KB reads; the fix is buffering/readahead or a larger block size, not more CPU.* That is a fleet answer produced live, non-invasively, in under a minute.

## Practice (feeds the deliverable toolkit)
Work the [bpftrace one-liner tutorial](https://bpftrace.org/tutorial-one-liners) top to bottom, then run these four on a real box (a dev node or a lab VM with a couple of workloads; `sudo` and a recent kernel with BTF required). Read each output, don't just run it:

```
# 1. Count syscalls by process — find the noisiest process by syscall volume.
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# 2. Histogram of read() sizes — spot small-read / large-read patterns.
sudo bpftrace -e 'tracepoint:syscalls:sys_exit_read /args.ret>0/ { @ = hist(args.ret); }'

# 3. Trace every execve system-wide — catch what is being launched (supply chain / cron / shell-outs).
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%-16s %-6d %s\n", comm, pid, str(args.filename)); }'

# 4. Count page faults by process — find memory-churning processes.
sudo bpftrace -e 'software:page-faults:1 { @[comm] = count(); }'
```

Then answer for your box: **which syscall is busiest system-wide** (from #1 aggregated by id, or reason from #2/#3)? Capture the reasoning in a comment.

**Acceptance (into the deliverable's diagnostic toolkit):** commit **3+ working bpftrace snippets** to the toolkit directory, each with a one-line comment stating the *fleet question it answers*, e.g.:
- `# Which container/process is generating the most syscalls right now? (find the noisy neighbor)`
- `# Are reads small-and-frequent (fix: buffering/readahead) or large? (I/O sizing)`
- `# What is being exec'd across the node? (unexpected shell-outs, cron surprises, supply-chain)`
- `# Which process is churning memory via page faults? (allocation pressure before OOM)`

Each snippet must run clean on your kernel and its output must be legible without you at the keyboard.

## Self-check
**(a) Why is eBPF safe to run in the kernel — what does the verifier guarantee and forbid?**
**Answer:** Safety is proven statically at load time by the verifier before the program ever executes. It walks all execution paths and *guarantees*: the program terminates (no unbounded loops — originally none, now only verifier-provable bounded loops), every memory access is within known bounds (no unchecked/out-of-range pointer dereference, no reading uninitialized memory), it stays within stack and instruction-count limits, and it cannot leak kernel pointers to unprivileged userspace. It *forbids*: calling arbitrary kernel functions (only a fixed allowlist of helpers/kfuncs), unbounded memory allocation, and blocking/sleeping in non-sleepable program types. If any path fails the proof, the `bpf()` syscall rejects it and it never loads. So it's safe *by construction*, not by catching faults at runtime; the JIT then compiles the verified bytecode to native code.

**(b) bpftrace vs BCC vs libbpf — when do you reach for each?**
**Answer:** **bpftrace** for ad-hoc interactive investigation — one-liners and short scripts, no build step, the 2am-on-a-node tool; ceiling is that it's a tracing DSL, not a shippable daemon. **BCC** is the Python+embedded-C framework that compiles on the target at runtime via in-process LLVM; use it mainly for its existing tool catalog (`execsnoop`, `biolatency`, etc.), but it's heavyweight (needs LLVM/headers on every host), slow to start, and now largely legacy. **libbpf + CO-RE** is the production path: compile the C program once ahead of time, and BTF + compiler relocations let one binary run across kernel versions with no headers or compiler on the target — this is what Cilium, the Datadog agent, and Pixie ship. Investigate with bpftrace, ship with libbpf/CO-RE, use BCC only for its tools.

**(c) What can Hubble/eBPF see about pod-to-pod traffic that iptables logging cannot?**
**Answer:** iptables logging operates on IPs, ports, and packet counts at netfilter chains — it has no notion of Kubernetes identity and no application-layer visibility, and on churny clusters the IPs are ephemeral and meaningless by the time you read the log. Hubble's eBPF programs sit in the datapath at the socket/tc/XDP layer *with Kubernetes identity attached*, so it reports flows labeled by source/destination **pod, namespace, and service**, the **L7 verb** (HTTP method+path, gRPC method, DNS query) for allowed protocols, and the **policy verdict** — allowed or dropped and *which* policy caused it — all correlated per-flow. It sees decrypted L7 semantics and orchestrator identity that raw iptables logs fundamentally cannot express.

## Resources
1. **[bpftrace one-liner tutorial](https://bpftrace.org/tutorial-one-liners)** — the fastest path from zero to useful; work every step. Start here.
2. **Brendan Gregg — [bpftrace intro](https://www.brendangregg.com/blog/2019-08-19/bpftrace.html)** and, for depth, **[BPF Performance Tools](https://www.brendangregg.com/bpf-performance-tools-book.html)** — the canonical treatment of the model and the tool catalog; the book is the reference you'll keep.
3. **[ebpf.io](https://ebpf.io/)** and **[Cilium + Hubble docs](https://docs.cilium.io/)** — the production-datapath story (verifier, maps, CO-RE) and how eBPF observability actually ships in Kubernetes.
