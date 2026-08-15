---
lesson: "01b.8"
title: "eBPF — the observability substrate"
module: "01b"
concept: "eBPF"
status: not-started
est_time: "8h"
prev: "07-networking-datapath-conntrack.md"
next: "09-perf-ftrace-use.md"
artifacts: []
sources: 8
---

# 01b.8 · eBPF — the observability substrate

> **Concept.** eBPF is a sandboxed virtual machine inside the kernel: you attach small verified programs to kernel and userspace hooks, they run at those events, and they ship data out through maps — turning the kernel into a programmable, production-safe observability datapath.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

Lesson 07 ended by naming the mechanism without fully explaining it: Cilium's socket-LB rewrites a connection's destination "at connect time" using "a BPF program attached to a socket hook," and keeps state in "its own eBPF conntrack map." You were asked to take that on faith. This lesson removes the faith requirement. It builds eBPF from first principles — programs, hooks, maps, the verifier, the JIT — so that "Cilium is eBPF-based" and "Hubble sees Kubernetes identity that iptables logging can't" become mechanisms you can verify with `bpftool` and reason about from scratch, not vendor claims you repeat. This is also, deliberately, the deepest lesson in the module: eBPF is the connective tissue between everything else you've learned (cgroups, PSI, memory, networking) and the ability to *observe* it live, in production, without a redeploy. Next: **09 — perf / ftrace / USE method**, which gives you the second half of the diagnostic toolkit — CPU profiling and static kernel tracing — and shows how bpftrace one-liners, `perf`, and the USE method compose into the systematic debugging flow that's the actual interview signal for this role.

## Why this matters

"eBPF" is now a line item in job descriptions at exactly the companies you are targeting: CoreWeave and NVIDIA want kernel-level GPU/host telemetry, Datadog's agent ships eBPF for network and universal service monitoring, Cilium/Hubble is the CNI layer on a large share of managed Kubernetes. When a JD says "experience with eBPF-based observability," the filter is whether you can talk about the *conceptual model* — programs, hooks, maps, the verifier — not whether you memorized `bpftrace` syntax. On a fleet, the practical payoff is that you can answer questions that are otherwise unanswerable without a redeploy: "which container is issuing the 4KB random reads hammering this NVMe," "who is calling `execve` with `curl | sh`," "why does p99 latency for this service spike every 30s" — live, on a production node, at overhead low enough to leave running. It's also worth knowing that this isn't one vendor's bet: the **eBPF Foundation**, hosted under the Linux Foundation with members including Meta, Google, Isovalent, Microsoft, and Netflix, exists specifically because multiple large, competing infrastructure organizations independently converged on eBPF as their observability substrate. That's the strongest possible signal this is durable infrastructure, not a fad to wait out. This is the single fastest-rising signal in the platform/SRE hiring market, and it is a genuine capability multiplier, not résumé decoration.

## What's new here (calibration)

Per the module README's calibration, you already know the *surface* of eBPF-powered tooling — you've used `cilium hubble observe`, the Datadog network map, `bpftop`, maybe `execsnoop` or `opensnoop` — and you know `strace` shows syscalls and is dangerous on a hot process because it stops the target on every call via ptrace. None of that surface familiarity is re-taught. What's genuinely new here:

- The **machine underneath** those tools: verifier, JIT, hooks, maps, as a mechanistic model rather than "magic box that emits events."
- **Why** eBPF is cheap where `strace` is ruinous — in-kernel aggregation, no per-event context switch — not just that it is.
- **BTF and CO-RE** as the specific mechanism that lets one compiled binary run across kernel versions, and **sleepable programs** as the deliberate exception to "can't block."
- The **toolchain trade-offs** (bpftrace / BCC / libbpf+CO-RE) as an engineering decision you'd defend in a design review, not trivia.
- The **GPU-specific wrinkle**: uprobing CUDA/NCCL symbols requires host-side debug symbol/BTF availability that's often missing in minimal container images — a real practical failure mode, not a footnote.

## Core concepts

### 1. The conceptual model: programs, hooks, maps, verifier, JIT

An eBPF program is bytecode for a small RISC-like virtual machine (11 registers, a 512-byte stack, its own instruction set). The lifecycle:

1. **You attach a program to a hook.** The hook is *where* the program runs. The families you care about for observability:
   - **kprobes / kretprobes** — dynamic instrumentation of (almost) any kernel function entry/return, e.g. `vfs_read`, `tcp_sendmsg`. Powerful but unstable: function names and arguments can change between kernel versions.
   - **tracepoints** — static, kernel-maintained instrumentation points with a stable ABI (`syscalls:sys_enter_read`, `sched:sched_switch`, `block:block_rq_issue`). Prefer these over kprobes when one exists — they don't break across kernel upgrades. `bpftrace -l 'tracepoint:*'` lists them.
   - **uprobes / uretprobes** — the same dynamic instrumentation for *userspace* functions in a binary or library, e.g. a Go runtime function, `libc:malloc`, a TLS read before encryption. This is how you trace application internals without recompiling.
   - **USDT** — statically-defined tracepoints compiled into userspace apps (used by the JVM, Node, PostgreSQL).
   - **perf events / PMU** — sample on a hardware counter overflow (cycles, cache misses) or a timer; this is what powers CPU profiling and off-CPU sampling.
   - **network hooks** — XDP (earliest, at the driver, for line-rate drop/redirect), tc/`clsact` ingress-egress, cgroup/socket hooks. This is the datapath Cilium is built on — and specifically the mechanism behind lesson 07's socket-LB: a program on the `connect(2)`/`sendmsg` cgroup/socket hook.

2. **The program runs and writes to maps.** A **map** is a kernel data structure — hash, array, per-CPU array, LRU hash, ring buffer, stack-trace map — that both the eBPF program (in kernel) and userspace read/write. Maps are how state survives between invocations and how data crosses the kernel/userspace boundary. The key move is **aggregate in-kernel**: instead of sending every event up, increment a per-key counter or a histogram bucket in a map and let userspace read the summary periodically. That is why eBPF is cheap where `strace` is ruinous. `BPF_MAP_TYPE_RINGBUF` (per-event streaming) and `BPF_MAP_TYPE_PERCPU_HASH` (lock-free aggregation) are the two you'll meet most.

3. **The verifier proves it safe *before* it loads.** This is the crux of "why is it safe to run arbitrary code in the kernel." The verifier does a static analysis — it walks every possible execution path and checks: the program **terminates** (originally: no loops at all; modern kernels allow *bounded* loops the verifier can prove finite, and `bpf_loop()` helpers), it never dereferences an **unchecked or out-of-bounds pointer**, it never reads **uninitialized memory** or leaks kernel pointers to unprivileged userspace, it stays within the stack and instruction-count limits (historically 4096 insns, now up to 1M for privileged), and it only calls a **fixed allowlist of helper functions** (`bpf_map_lookup_elem`, `bpf_probe_read_kernel`, `bpf_ktime_get_ns`, ...). It cannot call arbitrary kernel functions (except a curated set of "kfuncs"), cannot allocate unbounded memory, and cannot block or sleep — except in explicitly **sleepable** program types (see §5). If any path fails the proof, `bpf()` returns and the program never runs. So safety is **guaranteed by construction at load time**, not by trapping faults at runtime. The cost is real: "the verifier rejected my program" is the eBPF developer's daily pain, and its analysis is why loops and memory access are so constrained.

4. **The JIT compiles bytecode to native instructions.** Once verified, the JIT translates eBPF bytecode into native machine code (x86-64, arm64) so it runs at near-native speed rather than being interpreted. Verified + JITed is what makes production-always-on tracing viable.

Mental model to keep: **verifier = "will this be safe?", JIT = "make it fast", maps = "how do I get data out", hook = "when does it run".**

### 2. BTF and CO-RE — how one compiled binary runs across kernel versions

**BTF (BPF Type Format)** is type metadata — struct layouts, field names, field offsets, function signatures — describing the kernel's own data structures, generated at kernel build time and embedded in the running kernel. You can inspect it directly:

```
$ ls /sys/kernel/btf/vmlinux
/sys/kernel/btf/vmlinux
$ bpftool btf dump file /sys/kernel/btf/vmlinux format c | grep -A5 '^struct task_struct'
```

That command prints something close to the real C struct definitions the running kernel was compiled with — BTF is, concretely, a compact encoding of exactly that information, queryable at runtime.

Why this matters: before BTF, a program reading `task->pid` had to be compiled against a specific kernel's headers, because the byte offset of `pid` inside `struct task_struct` can shift between kernel versions and configs. That meant you needed a compiler and matching headers *on every target machine* (BCC's approach) or a separate compiled binary per kernel version. **CO-RE (Compile Once – Run Everywhere)** solves this: the compiler emits the program with *relocatable* field-access instructions instead of hardcoded offsets, and at load time the loader reads the *target kernel's* BTF, compares it against the BTF the program was compiled against, and patches in the correct offsets for the box it's actually running on. The practical result: you compile a libbpf program once, ship a single binary, and it correctly reads kernel structs on a kernel version you never tested against — as long as that kernel exports BTF (default on any reasonably modern distro kernel). This is the specific mechanism, not just the name, behind "libbpf + CO-RE is the production path" in §6.

### 3. Maps in more depth — the data plane

The map type you choose is a performance decision, and interviewers who know eBPF probe it:
- **`BPF_MAP_TYPE_HASH` / `PERCPU_HASH`** — keyed aggregation. The **per-CPU** variant gives each CPU its own copy so the program never takes a lock on the hot path; userspace sums the copies at read time. This is how you count "syscalls by pid" at millions/sec without cache-line contention.
- **`BPF_MAP_TYPE_ARRAY` / `PERCPU_ARRAY`** — fixed-size, index-keyed; used for histograms (bucket index → count) and global config/flags.
- **`BPF_MAP_TYPE_LRU_HASH`** — bounded hash that evicts least-recently-used entries; used when the key space is unbounded (per-connection state) and you must cap memory.
- **`BPF_MAP_TYPE_RINGBUF`** — the modern per-event channel to userspace (single MPSC ring shared across CPUs, replacing the older per-CPU `PERF_EVENT_ARRAY` perf buffer). Use it when you genuinely need every event (a stream of `execve`s), not an aggregate.
- **`BPF_MAP_TYPE_STACK_TRACE`** — stores captured stack traces keyed by an id; the profiling/off-CPU tools use it to fold thousands of stacks in-kernel.

The design rule that makes eBPF cheap: **aggregate in a per-CPU map and read summaries; stream through a ring buffer only when you must.** Sending every event to userspace is the anti-pattern that makes a tracer expensive — see Common pitfalls below.

### 4. Attach vs load, and pinning

Two distinct steps often conflated: **loading** a program (verify + JIT, `bpf(BPF_PROG_LOAD)`) and **attaching** it to a hook (`bpf_link`). A loaded-but-unattached program does nothing; detaching leaves it loaded. Programs and maps are normally torn down when the loader process exits (refcount to zero) — unless you **pin** them into the bpffs virtual filesystem (`/sys/fs/bpf`), which keeps them alive independent of any process. This is how a CNI or agent keeps its datapath running across restarts, and how `bpftool prog show` / `bpftool map dump` let you inspect what's currently loaded on a node — a useful "what eBPF is even running here?" first move on an unfamiliar box.

### 5. Cost, overhead, and safety envelope

eBPF is cheap but not free: each event pays the cost of the hook mechanism (a kprobe is an INT3/optimized-jump trampoline; a tracepoint is a cheaper static branch) plus your program's instructions. A tight counting probe on a hot function adds tens of nanoseconds; a probe that captures full stacks or does per-event ring-buffer output on a million-events/sec path *can* matter — measure before leaving something on permanently (the Netflix case study below is a concrete instance of exactly this discipline). Two safety caveats beyond the verifier: kprobes on a very hot function multiply that per-event cost across the whole system, and `bpf_probe_read` on userspace/kernel memory can fail (returns an error, doesn't fault) if the page isn't present — your program must handle that. Privilege: loading tracing programs needs `CAP_BPF` + `CAP_PERFMON` (or root); this is why agents run privileged and why unprivileged eBPF has been progressively locked down for security.

**Sleepable programs are the deliberate exception to "can't block or sleep."** Most eBPF programs run in an atomic, interrupt-like context where blocking would be unsafe — no mutexes, no sleeping helpers. But some program types are explicitly marked **sleepable** (the `.sleepable`/`BPF_F_SLEEPABLE` flag), notably LSM (Linux Security Module) hooks and some fentry/fexit/uprobe attachments, which run in a context where sleeping is safe and can call helpers like `bpf_copy_from_user` that may block on a page fault. The trade-off is extra RCU/trampoline overhead versus a non-sleepable program — you opt into sleepable only when the hook genuinely needs it (e.g., a security-policy decision that must read userspace memory that might not be resident).

### 6. The three toolchains: bpftrace vs BCC vs libbpf

All three produce eBPF programs; they differ in effort, portability, and control.

- **bpftrace** — a high-level tracing language (awk/DTrace-like) for **ad-hoc, interactive investigation**. One-liners and short scripts; it compiles your script to eBPF on the fly. Reach for it first at 2am on a node: fast to write, no build step. Ceiling: it is a tracing DSL, not a way to ship a daemon. `bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }'`.
- **BCC (BPF Compiler Collection)** — a Python (or Lua) framework that embeds C for the kernel side and compiles it **at runtime on the target** with an in-process LLVM/Clang. Ships the classic tool suite (`execsnoop`, `opensnoop`, `biolatency`, `tcplife`, `runqlat`). Good for reusable tools with rich userspace logic. Downsides: heavyweight (LLVM + kernel headers on every host), slow startup, and runtime compilation is fragile across a fleet. Historically the standard; now largely a **legacy** path — many BCC tools have been rewritten to libbpf.
- **libbpf + CO-RE** — the **production** path. You write the kernel side in C, compile it **once** ahead of time to a BPF object, and it runs across kernel versions thanks to CO-RE (see §2): BTF plus compiler relocations let the loader fix up struct field offsets at load time, so you don't need kernel headers or a compiler on the target. This is what Cilium, the Datadog agent, and Pixie ship. `bpftool` is the companion CLI (inspect loaded programs/maps, dump BTF, generate skeletons). Rust has an equivalent ecosystem (**Aya**, libbpf-rs).

Rule of thumb: **bpftrace to investigate, libbpf/CO-RE to ship, BCC only for its existing tool catalog.**

### 7. eBPF as the production observability datapath

This is the part interviewers probe, because it separates "I ran a bpftrace one-liner once" from "I understand why the industry moved here." **Cilium + Hubble** is the spine example: Cilium is a CNI that implements pod networking, load balancing, and network policy in eBPF at tc/XDP/socket hooks instead of iptables (this is the same mechanism lesson 07 leaned on). **Hubble** is its observability layer: because the eBPF programs sit *in the datapath at the socket and packet level*, Hubble sees every flow with **Kubernetes identity already attached** — source/destination pod, namespace, service, L7 verb (HTTP method+path, gRPC method, DNS query) — and whether policy *allowed or dropped* it, with the reason. `hubble observe --namespace foo --verdict DROPPED` shows drops labeled by identity and policy. The Real-world use cases section below covers three more production instances of this same through-line — Netflix's scheduler-hook observability, Datadog's TCP-path service monitoring, and Pixie's TLS-uprobe service mapping — each a different application of the identical idea: **kernel-truth with application/orchestrator identity, gathered in the datapath, with no code changes and low enough overhead to run continuously.**

### 8. bpftrace language in one screen (so the practice makes sense)

Every bpftrace program is a list of `probe /filter/ { action }` blocks:
- **probe** — `type:target`, e.g. `tracepoint:syscalls:sys_enter_openat`, `kprobe:vfs_read`, `uprobe:/lib/x86_64-linux-gnu/libc.so.6:malloc`, `profile:hz:99` (timed sampling), `interval:s:1` (once/sec). `BEGIN`/`END` fire at start/exit.
- **filter** — an optional predicate: `/pid == 1234/`, `/comm == "nginx"/`, `/args.ret > 0/`. The block runs only when it's true.
- **action** — statements. Builtins you'll use constantly: `comm` (process name), `pid`/`tid`, `args.<field>` (probe arguments), `retval`, `nsecs`, `kstack`/`ustack` (kernel/user stack), `str()` (read a char* into a string).
- **maps** — variables prefixed `@`. `@x = count()`, `@x = sum(v)`, `@x = hist(v)` (power-of-two histogram), `@x = lhist(v, min, max, step)` (linear). Keyed maps: `@[comm] = count()` aggregates per process. bpftrace prints all `@` maps automatically at exit.

That's ~90% of the one-liners you'll ever write: pick a probe, optionally filter, aggregate into a keyed map. Latency timing is the one two-block idiom worth memorizing — record a start timestamp keyed by tid on the entry probe, subtract on the return probe:

```
kprobe:vfs_read  { @start[tid] = nsecs; }
kretprobe:vfs_read /@start[tid]/ { @us = hist((nsecs - @start[tid]) / 1000); delete(@start[tid]); }
```

## Perspectives

**Kernel-mechanism view.** Everything else in this lesson is a consequence of three primitives: the verifier (proves safety at load time, by construction, not by trapping faults at runtime), the JIT (compiles verified bytecode to native code so it's fast enough to leave on), and maps (the only channel across the kernel/userspace boundary, which is why *how* you aggregate is the whole performance story). If you can explain why a program that dereferences an unchecked pointer never loads, and why a per-CPU hash map avoids a hot-path lock, you understand eBPF's core.

**Operator/SRE view.** The discipline that actually matters day to day is **bpftrace-first, ship-with-libbpf**: reach for a bpftrace one-liner to investigate an unfamiliar incident live (no build step, disposable), and only invest in a compiled libbpf/CO-RE program once you know you need something running continuously across the fleet. Treating this the other way around — building a compiled tool before you've even confirmed the hypothesis with a one-liner — is a common and expensive mistake. The bpftrace one-liner tutorial exists precisely to make the fast path fast.

**GPU-fleet-specific view.** eBPF is how you attribute host-side behavior to GPU workloads without touching the training code: uprobes on the CUDA runtime or NCCL, tracepoints on the block and network layers correlated to the cgroup/container issuing them, and off-CPU stacks (next lesson) that reveal a trainer blocked on `nfs`/`io_uring` rather than compute — all of it is the "the GPU is 20% utilized, *why*" investigation, answered from the host with no redeploy. But there's a practical wrinkle worth knowing before you try it: uprobing a symbol in `libcuda.so` or an NCCL shared library requires the **host** to have matching debug symbols or BTF-equivalent type information for the *container's* runtime version, and minimal GPU container images frequently strip exactly that information to save space. In practice this means "attach a uprobe to `ncclAllReduce`" can fail not because the mechanism is wrong, but because the symbol table you need isn't available anywhere the host can see it — a real operational gap you have to plan for (matching debug packages, or `-symbols` image variants) rather than a footnote.

**Industry-adoption/economics view.** The **eBPF Foundation**, established under the Linux Foundation, counts Meta, Google, Isovalent, Microsoft, and Netflix among its members — direct competitors in infrastructure who otherwise rarely standardize on shared tooling. That convergence is itself evidence worth citing in an interview: eBPF isn't a niche tracing trick one vendor is pushing, it's an industry-standardized observability substrate that multiple hyperscalers independently decided was worth building shared governance around. When you're arguing for investing engineering time in eBPF-based tooling instead of a proprietary agent, "this is foundation-governed, multi-vendor infrastructure, not a single company's bet" is a legitimate part of that economic case.

## Real-world use cases

- **[Netflix TechBlog — "Noisy Neighbor Detection with eBPF"](https://netflixtechblog.com/noisy-neighbor-detection-with-ebpf-64b1f4b3bbdd)** — Netflix built noisy-neighbor detection on eBPF scheduler hooks, and built their own `bpftop` tool specifically to measure and guarantee sub-600ns overhead per hook *before* shipping the tracer fleet-wide. What it shows: "measure before leaving something on permanently" (§5) is not academic advice — it's exactly the discipline a hyperscaler applied before trusting an always-on eBPF tracer in production.
- **[Datadog — "Universal Service Monitoring"](https://www.datadoghq.com/blog/universal-service-monitoring-datadog/)** — Datadog hooks the kernel TCP/UDP path (`tcp_sendmsg`, retransmit tracepoints) with eBPF to build a zero-instrumentation service map. What it shows: kernel-truth (real bytes, real retransmits) combined with orchestrator identity, with no code changes to the monitored services — the through-line from §7 applied at fleet scale.
- **[AWS Open Source Blog — "Gathering insights on Kubernetes applications, services, and network traffic with Pixie"](https://aws.amazon.com/blogs/opensource/gathering-insights-on-kubernetes-applications-services-and-network-traffic-with-pixie/)** — Pixie's eBPF auto-instrumentation includes uprobes on TLS libraries to capture *decrypted* HTTP/2/gRPC traffic. What it shows: uprobes (§1) aren't limited to your own code — hooking a userspace TLS library function is how you see plaintext application traffic without touching app code or terminating TLS anywhere.

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

**Feeds:** [Anatomy of a Container](../practice/anatomy-of-a-container/README.md) — commit these snippets directly into its diagnostic toolkit directory.

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

Then answer for your box: **which syscall is busiest system-wide** (from #1 aggregated by id, or reason from #2/#3)? Capture the reasoning in a comment. As a stretch task, run `bpftool prog list` and `bpftool map list` on your box and identify anything already loaded and pinned (a CNI, an observability agent) — practice the "what eBPF is even running here" first move from §4.

**Acceptance (into the deliverable's diagnostic toolkit):** commit **3+ working bpftrace snippets** to the toolkit directory, each with a one-line comment stating the *fleet question it answers*, e.g.:
- `# Which container/process is generating the most syscalls right now? (find the noisy neighbor)`
- `# Are reads small-and-frequent (fix: buffering/readahead) or large? (I/O sizing)`
- `# What is being exec'd across the node? (unexpected shell-outs, cron surprises, supply-chain)`
- `# Which process is churning memory via page faults? (allocation pressure before OOM)`

Each snippet must run clean on your kernel and its output must be legible without you at the keyboard.

## Common pitfalls

1. **Treating eBPF as "just a fancier strace."** The fundamental difference isn't syntax, it's architecture: `strace` stops the target on every syscall via ptrace and pays a context switch to a tracer process per event; eBPF aggregates *in the kernel* with a map and only copies out summaries. That's why eBPF is safe to leave running on a production box under real load and `strace -c` is not.
2. **Assuming kprobes are as stable across kernel versions as tracepoints.** kprobe targets are internal kernel function names and signatures — they can change or disappear entirely between kernel releases with no deprecation warning. Tracepoints have a stable, kernel-maintained ABI. Prefer a tracepoint whenever one exists for what you need; treat a kprobe as version-pinned instrumentation you'll need to revisit.
3. **Forgetting that a LOADED program isn't necessarily ATTACHED, and that an unpinned program dies when the loader exits.** Loading (verify+JIT) and attaching (to a hook via a `bpf_link`) are separate steps; and unless a program/map is pinned into bpffs (`/sys/fs/bpf`), it's torn down when the process that loaded it exits. A program that "was working a minute ago" after you closed your terminal is usually this.
4. **Assuming unprivileged users can load arbitrary eBPF.** Most tracing program types require `CAP_BPF` + `CAP_PERFMON` (or root); unprivileged eBPF has been progressively locked down for security reasons. Don't design a workflow that assumes a non-privileged service account can attach a kprobe.
5. **Choosing a RINGBUF/streaming design when a PERCPU_HASH aggregate would do.** Streaming every event to userspace "just in case you need the detail later" is the classic mistake that turns a cheap tracer into the very overhead problem you were trying to diagnose. Default to in-kernel aggregation (counts, histograms, per-key sums); reach for a ring buffer only when you genuinely need every individual event.

## Self-check

- **Why is eBPF safe to run in the kernel — what does the verifier guarantee and forbid?**
  **Answer:** Safety is proven statically at load time by the verifier before the program ever executes. It walks all execution paths and *guarantees*: the program terminates (no unbounded loops — originally none, now only verifier-provable bounded loops), every memory access is within known bounds (no unchecked/out-of-range pointer dereference, no reading uninitialized memory), it stays within stack and instruction-count limits, and it cannot leak kernel pointers to unprivileged userspace. It *forbids*: calling arbitrary kernel functions (only a fixed allowlist of helpers/kfuncs), unbounded memory allocation, and blocking/sleeping in non-sleepable program types. If any path fails the proof, the `bpf()` syscall rejects it and it never loads. So it's safe *by construction*, not by catching faults at runtime; the JIT then compiles the verified bytecode to native code.

- **bpftrace vs BCC vs libbpf — when do you reach for each?**
  **Answer:** **bpftrace** for ad-hoc interactive investigation — one-liners and short scripts, no build step, the 2am-on-a-node tool; ceiling is that it's a tracing DSL, not a shippable daemon. **BCC** is the Python+embedded-C framework that compiles on the target at runtime via in-process LLVM; use it mainly for its existing tool catalog (`execsnoop`, `biolatency`, etc.), but it's heavyweight (needs LLVM/headers on every host), slow to start, and now largely legacy. **libbpf + CO-RE** is the production path: compile the C program once ahead of time, and BTF + compiler relocations let one binary run across kernel versions with no headers or compiler on the target — this is what Cilium, the Datadog agent, and Pixie ship. Investigate with bpftrace, ship with libbpf/CO-RE, use BCC only for its tools.

- **What can Hubble/eBPF see about pod-to-pod traffic that iptables logging cannot?**
  **Answer:** iptables logging operates on IPs, ports, and packet counts at netfilter chains — it has no notion of Kubernetes identity and no application-layer visibility, and on churny clusters the IPs are ephemeral and meaningless by the time you read the log. Hubble's eBPF programs sit in the datapath at the socket/tc/XDP layer *with Kubernetes identity attached*, so it reports flows labeled by source/destination **pod, namespace, and service**, the **L7 verb** (HTTP method+path, gRPC method, DNS query) for allowed protocols, and the **policy verdict** — allowed or dropped and *which* policy caused it — all correlated per-flow. It sees decrypted L7 semantics and orchestrator identity that raw iptables logs fundamentally cannot express.

- **What is BTF, and how does it enable CO-RE ("compile once, run everywhere")?**
  **Answer:** BTF (BPF Type Format) is compact type metadata — struct layouts, field names and offsets, function signatures — describing the kernel's own data structures, embedded in the running kernel and exposed at `/sys/kernel/btf/vmlinux` (inspectable via `bpftool btf dump`). Before BTF, a program reading a kernel struct field needed a hardcoded byte offset, which meant compiling against the exact target kernel's headers. CO-RE instead has the compiler emit *relocatable* field-access instructions; at load time, the loader compares the program's BTF against the target kernel's BTF and patches in the correct offsets for that specific kernel. The result: one compiled libbpf binary runs correctly across kernel versions it was never built against, as long as the target exports BTF — no on-target compiler or matching headers required.

- **Why can't eBPF programs normally block or sleep, and what's the exception?**
  **Answer:** Most eBPF program types run in an atomic, interrupt-like execution context (e.g., directly inside a kprobe or tracepoint handler), where blocking would be unsafe — there's no scheduler-safe way to suspend that context, so sleeping helpers and mutexes are disallowed by the verifier. The exception is **sleepable programs**, explicitly marked with the `.sleepable`/`BPF_F_SLEEPABLE` flag — used by LSM hooks and some fentry/fexit/uprobe attachments — which run in a context where sleeping is safe and can call helpers like `bpf_copy_from_user` that may block (e.g., on a page fault). The trade-off is extra RCU/trampoline overhead, so you opt into sleepable only when the specific hook genuinely requires it.

## Connections & what's next

This lesson is the mechanism behind lesson 07's Cilium datapath (socket-LB, BPF conntrack map) and the general tool that will power lesson 09's diagnostic flow. More broadly, it's the connective tissue for the whole module: cgroups (03), PSI (04), memory/OOM (05), hugepages/NUMA (06), and the network datapath (07) are all subsystems you can now *observe directly* with a bpftrace one-liner instead of inferring from `/proc` snapshots — a kprobe on `try_charge` or a tracepoint on `sched_switch` turns any of those earlier lessons' abstractions into a live trace. It's also the direct foundation of the module's deliverable: the bpftrace snippets you commit here are the toolkit's core.

Next: **09 — perf / ftrace / USE method**, which adds CPU profiling (`perf record`/flame graphs) and static kernel tracing (`ftrace`) to the eBPF toolchain you just built, and assembles all of it — PSI, eBPF, perf, ftrace — into the systematic USE-method diagnostic flow that's the actual thing being tested when an interviewer asks "walk me through how you'd debug a slow node."

## References & further reading

**Primary sources**
- [bpftrace one-liner tutorial](https://bpftrace.org/tutorial-one-liners) — the fastest path from zero to useful; work every step before the Practice section.
- [ebpf.io](https://ebpf.io/) — the primary conceptual home for eBPF: verifier, maps, CO-RE, and the broader ecosystem/foundation context, read for the canonical vocabulary.

**Real-world engineering blogs**
- [Netflix TechBlog — "Noisy Neighbor Detection with eBPF"](https://netflixtechblog.com/noisy-neighbor-detection-with-ebpf-64b1f4b3bbdd) — what it shows: measuring and bounding per-hook overhead (their own `bpftop` tool, sub-600ns) before shipping an eBPF tracer fleet-wide.
- [Datadog — "Universal Service Monitoring"](https://www.datadoghq.com/blog/universal-service-monitoring-datadog/) — what it shows: hooking the kernel TCP/UDP path to build a zero-instrumentation service map.
- [AWS Open Source Blog — "Gathering insights on Kubernetes applications, services, and network traffic with Pixie"](https://aws.amazon.com/blogs/opensource/gathering-insights-on-kubernetes-applications-services-and-network-traffic-with-pixie/) — what it shows: uprobes on TLS libraries to capture decrypted HTTP/2/gRPC without touching app code.

**Deeper dives**
- Brendan Gregg — [bpftrace intro](https://www.brendangregg.com/blog/2019-08-19/bpftrace.html) and, for depth, [BPF Performance Tools](https://www.brendangregg.com/bpf-performance-tools-book.html) — the canonical treatment of the model and the tool catalog; the book is the reference you'll keep.
- [Cilium + Hubble docs](https://docs.cilium.io/) — the production-datapath story (verifier, maps, CO-RE) and how eBPF observability actually ships in Kubernetes; read alongside lesson 07's Cilium section.
